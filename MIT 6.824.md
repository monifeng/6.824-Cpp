# MIT 6.824

> MIT 6.824分布式系统的课程以及lab实现
>
> 本文章仅为记录，并不考虑别人的阅读体验，如果在阅读过程中觉得吃力或疑惑，你可以选择阅读原教程和原文章。
>
> 如果在阅读过程中发现错误或疑问，可以评论或私信，看到一定及时回复！

原课程地址：[MIT 6.824](https://pdos.csail.mit.edu/6.824/schedule.html)

B站中文翻译视频：[2020 MIT 6.824 分布式系统](https://www.bilibili.com/video/BV1R7411t71W/?share_source=copy_web&vd_source=e1e24c47f1c7ed2a5d313e7001d78147) 

翻译材料：[MIT 6.824](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-01-introduction/1.1-fen-bu-shi-xi-tong-de-qu-dong-li-he-tiao-zhan-drivens-and-challenges)

## MapReduce（论文）

> 原文：[MapReduce](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)
>
> 中文译文（个人觉得比较好的）: [谷歌三大核心技术（二）Google MapReduce中文版](https://cloud.tencent.com/developer/article/1981246?shareByChannel=link&sharedUid=10535947#content)

### 1、 概要

通过 **map** 方法实现一个值对另一个值的映射，用  **reduce** 方法来合并所有的中间值，以获得最初的 *value* ；

### 2、 Introduction

分布式系统产生原因：越来越大量的数据输入（TB），但是需要在一个合理的时间范围内返回结果，一台机器并是解决的办法（成本，效率），更好的办法是并行且分布式的发布给很多台电脑来执行处理，然后返回正确的结果（光是这里已经能感受到问题很多，难度很大，包括但不限于多线程情况下的各种问题）

个人读下来的问题有：

- 如何并行执行？

- 如何分散数据？

- 如何让数据执行后结合并返回正确结果？

- 如果任何一台机器出现问题，其他机器怎么办或者如何得知其他机器的执行情况？

  这里个人猜测是：sockets，signal，可以借鉴TCP协议的可靠传输机制；



解决诸多复杂问题的办法：

应用MapReduce处理数据：

1. 利用map操作来获取一个键值对的集合，在所有具有相同key值的value使用reduce操作，来合并中间的数据，得到一个想要的结果的目的。
2. 因为Mapreduce的键值对特性，自带 **再次执行** 的功能，提供了初级的容灾方案。

### 3、 编程模型

Mapreduce原理： *利用一个输入键值对集合，来获取一个输出键值对集合* 。

用户自定义 **map** 函数，接收一个输入的键值对集合，MapReduce库把所有具有相同key值的中间value集合在一起然后传输给reduce；

用户自定义 **reduce** 函数，接收一个key值，和一个value值的集合，一般每次调用返回0或1个value，这样就解决了value集合太大无法放入内存的麻烦（每次迭代一个然后运算后返回？）

#### 01 单词统计伪代码示例

略。



#### 02 类型

尽管在前面例子的伪代码中使用了以字符串表示的输入输出值，但是在概念上，用户定义的Map和Reduce函数都有相关联的类型：

```c++
 map(k1,v1) -> list(k2,v2)
 reduce(k2,list(v2)) ->list(v2) 
```



#### 03 更多例子

略



### 4、 实现

![image-20230426224220737](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230426224220737.png)

​										                          																																（图1）

#### 01 概括

将数据分割成M个数据片段，并行执行在多台机器上，map调用，并将产生的中间key值分散到多个机器上执行；

步骤与**（图1）**一一对应（也许不会全写）：

1. 将数据分片为M个，然后用户程序在机群中创建大量副本；
2. 有一个特殊的 **master** 程序，其他都是worker，由master来分配任务，一共由M个map任务和R个reduce任务；（联想到了Reactor模式，用来处理大并发连接，可见这种模式的思想非常值得学习）；
3. worker从输入中读取相关片段，调用map函数，然后将输出的中间kv set缓存到内存中；
4. 然后缓存中的kv pair通过分区函数分成R个区域，然后周期性地写到磁盘上，缓存的位置返回给master，master再发送地址到Reduce worker；
5. Reduce worker接收到master发来的数据存储信息后，使用RPC从map worker读取信息，对key排序，让相同key的信息聚合在一起（数据量太大则从外部排序），**必须排序！** 
6. Reduce worker遍历排序后中间数据，对于每个唯一的key值，Reduce worker将所有中间value值的集合传递给reduce函数；
7. 所有任务执行完毕后，master唤醒用户函数，返回结果。



#### 02 Master数据结构

Master数据结构中存储了Map和Reduce任务的状态（空闲，工作，完成），以及Worker机器的状态（非空闲）。

像一个管道一样（Muduo中的Channel？）将Map执行完的数据存储，然后将存储位置传递给Reduce。



#### 03 容错

设计初衷是成百上千的机器一同工作，其中任何一台机器都有可能出错，所以需要容错机制。

##### Worker故障

Master周期性的ping每个worker（轮询的思想），如果一定时间没有回应，将该Worker标记为失效，所有失效的Worker完成的Map任务被重定义为初始的空闲状态（ **必须重新执行，因为map的结果在这台机器上， 已经无法访问** ），之后这些任务安排给其他的worker。同时所有正在运行的 map 和 reduce 任务重置为空闲状态，等待调度。

这个机制可以处理大规模的worker失效，即使很多机器失效，所需要做的工作也仅仅是将他们做的工作重新做一遍，然后再继续向下执行。



> 如果存在一种极端情况，绝大部分机器都已失效，此时从硬件层次考虑也许是更好的选择。因为软件的提升是有瓶颈的。



##### master失效

简单的解决办法是：周期性存储master状态， 如果master失效，就恢复到上一个 check point 的状态，但是这种办法是非常复杂的，因为只有一个master；我们的处理方式是：直接中止操作，然后根据这个状态重新执行MapReduce操作。



#### 04 失效处理

Map和Reduce都是输入确定性的函数，系统在任何情况下的输出和所有程序没有出现任何错误，顺序的执行产生的输出是一样的。

这依赖于M，R操作都是原子性提交，每个任务完成后，将输出存储到私有的临时文件中。每个Reduce任务生成一个这样的文件，而每个map任务则生成R个这样的文件，如果map完成，就信息传递给master，然后master存储（多次传输也只保留一次），然后将该临时文件的地址发送给reduce。

如果一个reduce在多个机器上执行，依赖于底层重命名的原子性，同时间只有一个Reduce任务产生的数据。

如果M，R是不确定性函数，有一个较弱的机制：

有M和Ri这样的任务，e（Ri）代表Ri已经提交的执行过程， e（Ri）分别读取M，就形成了较弱的失效处理。



#### 05 存储位置

网络带宽是一个匮乏的资源，所以我们尽量让各个任务在本地执行。实现方式是，通过把输入数据分割成需要多Blocks（64M），然后拷贝存储在多个机器上（一般3个），master在分配任务时，尽量分配到本地的任务，如果没有，就分配到就近的任务，这样让大部分的工作不占用网络带宽。（思想类似于多线程中，减少临界区的概念？学了多线程，发现其中的思想非常有意思，非常实用且简洁优雅。）



#### 06 任务粒度

为了提高负载均衡的能力，比较理想的状态是，M和R比worker机器多得多，每台机器都能执行大量不同的M，R操作，失效机器的M，R都可以在其他机器上执行。

但实际上M和R有一定的限制：

时间复杂度：O（M+R）次唤醒；

空间复杂度：O（M*R），这里影响较小，因为每个M+R差不多一个字节就足够。



R一般由用户指定，更倾向于指定M值，使得每个任务处理在16~64M之间，这样也正         *符合05存储位置* 那个地方的内存优化策略，然后把R设为我们想要的Worker机器数量小的倍数，一个常见的例子：

- M 200000, R 5000, worker 2000;



#### 07 备用任务

影响MapReduce总执行时间最多的一般是 “落伍者” ，一台机器出现故障导致速度非常慢，有一个通用的机制来解决问题：

当MapReduce接近完成的时候，master将任务调用给备用任务进程来执行剩下的，处于处理中的任务（没错，即使正在执行，也会调用给备用进程再重新执行），无论备用还是worker先处理完，都会标记为处理完，并加入master的数据结构。



### 5、技巧

#### 01 分区函数

缺省的分区函数采用hash， hash生成一个平均的分区，对于特定的，可以采取特定属性的hash。

#### 02 顺序保证

保证给定分区中的数据时按照 **key** 的增量顺序来处理，保证了输出文件的有序性。

个人思考：map读取出来后，保存在一个vector中是否可行呢，因为map的查找时间为O（1），然后用move函数来插入到一个vector中，每个key的value集合用一个vector来保存，但是问题在于怎么保存key呢？直接用数组名来保存是否可行？

#### 03 Combiner函数

Map函数在输出时可能存在大量的重复输出，“the 1”这样的键值对，如果让这种重复数据进入网络是非常浪费资源的。较好的处理方式是，在本地先进行一次合并，变成 {the， n}这样的键值对，然后存储到中间文件，再由Reduce来处理



#### 04 输入输出类型

通过一个Reader接口，来读取不同的数据类型（数据库，内存，文件）。



#### 05 副作用

看文章暂时没看出副作用是什么，但是提供了一个获得原子且幂等的结果的方法：

输出一个临时文件，然后对临时文件进行重命名操作。



#### 06 跳过损坏的记录

用户程序会导致MapReduce出现bug，任务直接crash，常规做法是解决bug然后重启MapReduce，但实际处理方式是，用信号的方式，发一个UDP包给master，master标记该条需要跳过（UDP包的原因：无连接，尽全力交付，很轻易地就能发送，即使丢失在下一次执行时也可以找到，但不管怎么样都存在丢失的可能）。



#### 07 本地执行

本地执行的版本专门为DEBUG而生，个人猜测是由多线程来实现的（又必须得提到Reactor模式了^_^）。



#### 08 状态信息

master使用一个嵌入式的HTTP服务器(Jetty)，监控各种信息：执行进度，中间数据，输入输出字节数，方便用户来跟踪错误信息和分配资源（不就是日志系统吗）。猜测这些信息是写在content-text里的。



#### 09 计数器

用户自定义一个计数器，每次执行Map和Reduce操作就会让计数器++（毫无疑问是原子性的）。

```c++
Counter* uppercase;
uppercase = GetCounter("uppercase");
map(String name, String contents):
for each word w in contents:
if (IsCapitalized(w)):
uppercase->Increment();
EmitIntermediate(w, "1");
```

周期性的传递给master，附加在ping包中（防失效的轮询机制），然后master进行汇总，

当整个MapReduce完成后，用户汇总。



### 6、性能，经验，相关工作

本节可能并不会全部记录，请参看原文。



### 7、总结

MapReduce封装了并行处理，容错处理（标记失效，重新分配），数据本地化优化（排序实现），负载均衡。



## Lab1

### 1、概要

完成两个类和两个函数的实现：

- master，worker；
- 两个函数：map，reduce；

master调用worker，他们之间通过rpc来通信。输入个文件，map输出中间文件，worker输出文件。





![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230428163702025.png)



### 2、Master设计

#### 01 分析

Master的任务：

- 控制worker执行map和reduce操作；
- 当worker失效时将他的任务拷贝并分配给其他任务；



在paper中提到：

- 存储每个Map和Reduce的状态；
- Worker机器的标识；
- 存储了Map任务产生的R个中间文件的存储位置和大小；





### 2、worker的设计

01 分析

worker在本地表现为一种工作的线程，我自己的实现方法是：创建线程，在线程中分别执行map和work。



### 3、 Map和Reduce

#### 01 键值对



#### 02 枚举类状态

根据前文分析，需要使用到各个worker的状态，随设计出枚举类 state：

```c++
enum class State
{
    Free , Executing, Complete
};
```

分别表示空闲，执行中，完成三种状态。



#### 03 Map函数

输入键值对（仅针对word count程序）

- key：文件名
- value：文本内容

输出中间文件

- key：单词
- value：单词

我对map函数的设计参考了paper的附录A，输入的文本内容，按照单词来划分（空格到下一个空格）；



#### 04 获取文本的过程

 获取文本内容的调用顺序是：`getText()` -> 取得KeyVal{file_name, content} -> `mapFun()` 

-> `split()` -> vector<string> {"word"}  -> (**返回mapFun**)vector<KeyVal> {"word", "1"}...

### 04 Reduce函数

基本原理和map差不多，可能分割字符串的时候稍微有一些复杂，不再赘述思路，实在想不通可以参考一下源码。

反思：对于文件IO这类基本操作还不是很熟悉，需要查资料和参考别人的代码。





### 4、分配任务的机制

master如何分配任务给worker？worker又如何通知maser任务已完成？涉及到了多线程的知识，这里并不是简单的一分一发。

我预想的实现方式是：

一个vector存储map任务，一个vector存储reduce任务，每运行一个worker，就来申请一个任务，优先执行map任务





### 5、遇到的问题

#### 01 如何判断任务是否超时

方法：从worker线程中detach一个线程，专门用来计时，预设时间到达后，直接返回并检查任务状态，如果任务在预设时间内并未完成，就需要重新分配任务给另一个worker。

实现：

- 分离线程，到时间后回收线程并检查；
- 用epoll来监听timerfd；（暂未实现，只是一个想法）
- 条件变量；（暂未实现，只是一个想法）



在master.cpp中的`waitMapTask()` ，作者的定时器方式是构造一个线程，让线程执行等待时间的任务：

```c++
    pthread_create(&tid, NULL, waitTime, &op);
    pthread_join(tid, &status);  //join方式回收实现超时后解除阻塞
```

如果这样阻塞，和单线程的效率似乎是差不多的（甚至可能更差，涉及到线程上下文切换）？

因为在执行`waitTime()` 的时候，主线程就会一直阻塞。

我的想法是用epoll或者poll来通过定时事件触发（暂未实现），请问这样会不会更好呢？



#### 02 互斥锁的使用和临界区的界定

例如master.cpp中的 `assignMap()` ，

```c++
string Master::assignMap()
{
    if (mapDone()) return "empty";
    char *task = "";
    // 临界区
    {
        lock_.lock();
        if (!files_.empty()) 
        {
            task = files_.back();
            files_.pop_back();
            runningMaps_.push_back(string(task));
            lock_.unlock();
            return string(task);
        }
        //这里的解锁是必须要的，否则如果已经读完了files_,直接跳过if语句中的解锁，就会导致一直阻塞，无法返回empty();
        lock_.unlock();                             
    }
    return "empty";
}
```

这是一个给worker调用的rpc函数，由mapWorker来调用该函数，第一次写该函数，我并没有在if语句外加上`unlock()` 导致一些bug出现的让人百思不得其解，如果多次调用worker，并不会返回empty，而是直接阻塞，后来才发现，如果files_（即任务队列）为空, 原来的unlock并不生效，就会导致永不解锁的情况。

我的建议是：**用大括号来划出临界区，每次写完检查临界区是否完整的加锁解锁** 。

同时临界区应当尽可能地小，因为否则会因为互斥锁而导致性能降低，如源代码中的 `mapDone() `函数 中，有一段非常经典的缩小临界区的方法；

```c++
    // 临界区,需加锁
    {
        lock_.lock();
        sz = finishedMaps_.size();
        lock_.unlock();
    }
```

这段代码让临界区仅有一行，执行非常迅速，仅需获取 finishedMaps_ 的大小，比较放在临界区外面再进行，这也是从muduo项目中学到的非常有用的技法。



#### 03 如何通知worker所有任务已经完成？

可以用rpc调用一个 `done()` 函数，done返回一个bool值，如果所有任务执行完成（完成任务队列的大小与输入文件的数量相等），就返回true，此时woker在执行完自己的任务后就会相继关闭。

否则就会一直循环等待，如果某个worker超时，就会把任务退还给master，然后再重新分配。

同时符合了MapReduce的容错机制和muduo中的one loop per thread思想（从muduo中受益良多，非常建议大家学习一下）。



#### 04 让不同map处理的相同单词写在同一文件

初见时感觉非常困惑，因为map函数并不会进行单词的合并；后来研究了一下官方的worker文件发现，非常简单：

- 使用iHash函数，让相同单词分配到相同的哈希值，然后相同哈希值的单词由同一个reduce函数来处理；



#### 05 写入文件时遇到的errno 13

open函数在写文件时遇到了 **permission denied** 的情况；

原因：如果用open来创建了文件，就必须指定第三个参数，也就是文件权限的设定；

一般文件为0644，文件夹为0755（用户可读可执行，拥有者全权）；

```c++
// 如果指定O_CREAT，必须指定第三个参数，第三个参数的含义是：用户可读写，组内用户和其他用户只读
int fd = open(path.c_str(), O_WRONLY | O_APPEND | O_CREAT, 0644);  
```

然后解决了问题。



#### 06 超时重传（容错机制）

基本思路就是：分离一个线程专门用来计时，在计时线程内分离线程，用cond或者epoll来实现计时器，当计时器返回时，检查任务是否完成，如果没有完成，就需要将任务交给下一个线程。

分离线程并执行如下函数：

```c++
void* setTimer(void* fd)
{
    int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC);
    if (tfd == -1)
    {
        handle_error("timerfd_create");
    }
    struct itimerspec new_val;
    new_val.it_interval.tv_nsec = 0;
    new_val.it_interval.tv_sec = 0;

    new_val.it_value.tv_nsec = 0;
    new_val.it_value.tv_sec = WAIT_TIME;

    if ((timerfd_settime(tfd, 0, &new_val, NULL)) == -1)
    {
        handle_error("timerfd_settime");
    }

    // printf("timer started\n");
    
    int efd = epoll_create1(EPOLL_CLOEXEC);
    if (efd == -1) 
    {
        handle_error("epoll_create1");
    }

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = tfd;
    
    epoll_ctl(efd, EPOLL_CTL_ADD, tfd, &ev);

    const int maxEpollNum = 1;
    struct epoll_event events[maxEpollNum];

    while(1)
    {
        int nfd = epoll_wait(efd, events, maxEpollNum, -1); // 设为一直阻塞，直到事件发生（3s）
        if (nfd > 0)
        {
            for (int i = 0; i < nfd; ++i)
            {
                if (events[i].data.fd == tfd)
                {
                    uint64_t exp;
                    int ret = read(tfd, &exp, sizeof(uint64_t));
                    if (ret == -1)
                    {
                        handle_error("read");
                    }
                    // TODO: 进行超时判断的逻辑处理；
          			cout << "执行超时判断" << endl;
                }
            }
        }
    }
}
```

分离线程后，阻塞的会是这个计时器线程，主线程会继续派发任务，但是因为main函数中的 `done()` 判断，会一直等待所有任务执行完毕，才会结束主线程。

执行结果如下图，可以看出在所有定时器结束之前，所有任务就已经完成并退出主程序了（计时器线程是 `detch` 的）。所以两次执行的server端打印不同。

![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230505195845950.png)

此时需要人为制造bug，来验证我们的计时线程超时判断是否生效：

```c++
        // 人造bug
        {
            g_lock.lock();
            if (g_disableMap == 1 || g_disableMap == 3 || g_disableMap == 5)
            {   
                g_disableMap++;
                g_lock.unlock();
                cout << "map " << g_disableMap << " is sleep in" << pthread_self() << endl;
                // while(1){
                //     sleep(2);
                // }
                break;
            }
            else g_disableMap++;
            g_lock.unlock();
        }
```

运行结果正确，但是不知道为什么服务端会中断...debug半天没找出原因，只能作罢，综合来说这是一个有些小瑕疵的代码，而且经过测试，推荐使用 `pthread_join` 来创建一个计时器，因为作用和效率差不多，但pthread明显好写很多。

在  ` mapWait` 种加上：

```c++
    pthread_t tid;
    char op = 'm';
    pthread_create(&tid, NULL, waitTime, &op);
    pthread_join(tid, NULL);  //join方式回收实现超时后解除阻塞
```

`waitTime()` 定义

```c++
void* Master::waitTime(void* arg){
    char* op = (char*)arg;
    if(*op == 'm'){
        sleep(MAP_TASK_TIMEOUT);
    }else if(*op == 'r'){
        sleep(REDUCE_TASK_TIMEOUT);
    }
}
```

结果展示：	![image-20230505231631238](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230505231631238.png)



### 6、总结

仅靠网上的参考视频，参考资料，非常难。前面我描述的遇到的问题，很多一笔带过，但其实带给我了我非常大的麻烦，尤其是最后的DEBUG阶段，面对如此昂长且没有章法的代码，一次次编译，一次次运行，真的非常头疼，并且中途遇见了很多学过但记得并不牢靠的知识点，也去网上查资料重新学习（epoll，timerd，read...）。

我写的代码非常冗长，并且因为没有官方测试程序，全靠我自己编写单元测试（代码中非常多的cout），来一步一步完成，在这个过程中，真的收获非常大，推荐大家自己独立去写一写（我自己都没做到，实在有些地方想不通，debug不到，只能去参考别人的源码）。

学会了用AddressSanitilizer去debug（解决多线程和空指针问题），太好用了，太好用了，太好用了，考虑出一篇博客，记录真实的debug过程。

**最后祝大家愉快的coding，能从中有所收获！**

