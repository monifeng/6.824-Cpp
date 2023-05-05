# Lab1

> 仅为记录，并不为他人阅读体验做考虑，如果阅读不适，请去看官方资料。

## 1、概要

完成两个类和两个函数的实现：

- master，worker；
- 两个函数：map，reduce；

master调用worker，他们之间通过rpc来通信。输入个文件，map输出中间文件，worker输出文件。





![](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230428163702025.png)



## 2、Master设计

### 01 分析

Master的任务：

- 控制worker执行map和reduce操作；
- 当worker失效时将他的任务拷贝并分配给其他任务；



在paper中提到：

- 存储每个Map和Reduce的状态；
- Worker机器的标识；
- 存储了Map任务产生的R个中间文件的存储位置和大小；





## 2、worker的设计

### 01 分析

worker在本地表现为一种工作的线程，我自己的实现方法是：创建线程，在线程中分别执行map和work。



## 3、 Map和Reduce

### 01 键值对



### 02 枚举类状态

根据前文分析，需要使用到各个worker的状态，随设计出枚举类 state：

```c++
enum class State
{
    Free , Executing, Complete
};
```

分别表示空闲，执行中，完成三种状态。



### 03 Map函数

输入键值对（仅针对word count程序）

- key：文件名
- value：文本内容

输出中间文件

- key：单词
- value：单词

我对map函数的设计参考了paper的附录A，输入的文本内容，按照单词来划分（空格到下一个空格）；



### 04 获取文本的过程

 获取文本内容的调用顺序是：`getText()` -> 取得KeyVal{file_name, content} -> `mapFun()` 

-> `split()` -> vector<string> {"word"}  -> (**返回mapFun**)vector<KeyVal> {"word", "1"}...

### 04 Reduce函数

基本原理和map差不多，可能分割字符串的时候稍微有一些复杂，不再赘述思路，实在想不通可以参考一下源码。

反思：对于文件IO这类基本操作还不是很熟悉，需要查资料和参考别人的代码。





## 4、分配任务的机制

master如何分配任务给worker？worker又如何通知maser任务已完成？涉及到了多线程的知识，这里并不是简单的一分一发。

我预想的实现方式是：

一个vector存储map任务，一个vector存储reduce任务，每运行一个worker，就来申请一个任务，优先执行map任务





## 5、遇到的问题

### 01 如何判断任务是否超时

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



### 02 互斥锁的使用和临界区的界定

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



### 03 如何通知worker所有任务已经完成？

可以用rpc调用一个 `done()` 函数，done返回一个bool值，如果所有任务执行完成（完成任务队列的大小与输入文件的数量相等），就返回true，此时woker在执行完自己的任务后就会相继关闭。

否则就会一直循环等待，如果某个worker超时，就会把任务退还给master，然后再重新分配。

同时符合了MapReduce的容错机制和muduo中的one loop per thread思想（从muduo中受益良多，非常建议大家学习一下）。



### 04 让不同map处理的相同单词写在同一文件

初见时感觉非常困惑，因为map函数并不会进行单词的合并；后来研究了一下官方的worker文件发现，非常简单：

- 使用iHash函数，让相同单词分配到相同的哈希值，然后相同哈希值的单词由同一个reduce函数来处理；



### 05 写入文件时遇到的errno 13

open函数在写文件时遇到了 **permission denied** 的情况；

原因：如果用open来创建了文件，就必须指定第三个参数，也就是文件权限的设定；

一般文件为0644，文件夹为0755（用户可读可执行，拥有者全权）；

```c++
// 如果指定O_CREAT，必须指定第三个参数，第三个参数的含义是：用户可读写，组内用户和其他用户只读
int fd = open(path.c_str(), O_WRONLY | O_APPEND | O_CREAT, 0644);  
```

然后解决了问题。



### 06 超时重传（容错机制）

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



## 6、总结

仅靠网上的参考视频，参考资料，非常难。前面我描述的遇到的问题，很多一笔带过，但其实带给我了我非常大的麻烦，尤其是最后的DEBUG阶段，面对如此昂长且没有章法的代码，一次次编译，一次次运行，真的非常头疼，并且中途遇见了很多学过但记得并不牢靠的知识点，也去网上查资料重新学习（epoll，timerd，read...）。

我写的代码非常冗长，并且因为没有官方测试程序，全靠我自己编写单元测试（代码中非常多的cout），来一步一步完成，在这个过程中，真的收获非常大，推荐大家自己独立去写一写（我自己都没做到，实在有些地方想不通，debug不到，只能去参考别人的源码）。

**最后祝大家愉快的coding，能从中有所收获！**
