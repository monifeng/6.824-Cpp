## Lab2A

### 0、实验要求（gpt翻译）

#### 01 介绍

**请先研究Raft算法**

![img](https://pic2.zhimg.com/v2-dd21de5dec6e2325d01bd70c84e88595_r.jpg)



#### Lab 2A实验要求

的主题是“Raft一致性算法的基本实现”。

在该实验中，学生需要实现Raft协议的领导者选举和心跳机制，并处理来自客户端的请求。具体要求如下：

1. 实现Raft的领导者选举机制。在实现中需要满足Raft协议的规范，即在选举过程中设置超时机制，实现PreVote机制等。
2. 实现Raft的心跳机制。领导者需要定期向其他节点发送心跳信号，以维持自己的地位。
3. 处理来自客户端的请求。领导者需要维护一个日志，并将客户端的请求转化为日志项，通过Raft协议保证日志的一致性。
4. 实现集群成员变更的支持。支持添加或删除节点，以及处理在节点宕机后重新加入集群的情况。
5. 编写测试代码，验证实现的正确性。在测试过程中需要模拟节点宕机、网络分区等情况，以测试Raft协议的可靠性和一致性。

实验完成后，学生需要撰写实验报告，介绍自己的设计思路、实现细节、遇到的问题及解决方案等，并在提交代码的同时，将实验报告提交至对应的GitHub仓库。

实现Raft领导者选举和心跳（不带日志条目的AppendEntries RPC）。第2A部分的目标是选举出一个单一的领导者，如果没有故障，领导者仍将保持领导者的地位，并且在旧领导者失败或从旧领导者往返的数据包丢失时，新的领导者将接管。运行go test -run 2A来测试你的2A代码。





### 1、Lab2A的实现

#### 01 RequestVoteArgs和RequestVoteReply

Args保存了发起投票的follwer的log信息，term信息（详见Raft paper的图二）



#### 02 完善RequestVote函数

该函数就是投票函数，需要严格按照paper中讲的步骤来进行设计。

在这里理一下投票的流程：

1. 所有server都是follower状态（随机初始化超时事件50~350）

2. 某个leader超时以后就默认没有leader，直接开始选举

3. 开始选举，自增term，切换到condidate状态，立刻投票给自己，然后发送RequestVote RPC给其他所有节点

   condidate会一直维持该状态直到：

   - 赢得选举（过半投票）
   - 新的leader已经出现（退为follower）
   - 在规定时间后，不输不赢

   下面三点都是这三条的扩充与完善

4. 如果出现了leader，就会立刻向其他所有的服务器节点发送心跳信息，确认并阻止其他选举。

5. 如果在condidate等待期间，新的leader的任期号不小于自己，就会承认leader并退化，如果小于自己的term，不管该心跳信息。

6. 没有candidate赢得过半选票，但等待时间已经结束，此时就term++，重启选举流程（Raft 算法使用随机选举超时时间的方法来确保很少发生选票瓜分的情况，就算发生也能很快地解决。每个 candidate 在开始一次选举的时候会重置一个随机的选举超时时间，然后一直等待直到选举超时）



##### 如果出现已经投票的情况，直接返回false

判断已经投票：在同一个term内（不同term投票无所谓），并且voteFor不为-1（初始化为-1了）



#### 04 定时器

使用协程中的time.sleep来实现定时，然后在定时结束后，进行操作的处理，例如检查rf的状态。



#### 05 heartBeat			





#### 0x 一些细节

实验指导中说明了：test可能是每秒几十次心跳，所以选举超时时间应该稍长一些，我个人设定为了300-600ms（论文中为150-300ms）



### 3、 遇到的问题

#### 01 ticker中如何实现一旦获得过半选票， 立刻成为leader

因为ticker是一个定时器，每次触发都需要运行完一个ticker周期，很难做到“立刻成为leader”这种操作，

```go
for {
			select {
			case <-time.After(time.Duration(300+(rand.Int63()%150)) * time.Millisecond):
				rf.mu.Lock()
				var st = rf.state
				rf.mu.Unlock()
				if st == "follower" {
					// 开始选举
					rf.mu.Lock()
					rf.currentTerm++
					rf.state = "candidate"
					rf.voteNum++
					rf.votedFor = rf.me
					// voteN := rf.voteNum
					rqArgs := RequestVoteArgs{
						term:         rf.currentTerm,
						candidateId:  rf.me,
						lastLogIndex: len(rf.log) - 1,
						lastLogTerm:  rf.currentTerm,
					}
					rf.mu.Unlock()
					// 没有超过半数，继续选举
					rqReply := RequestVoteReply{
						term:        -1,
						voteGranter: false,
					}
					for i := range rf.peers {
						if i == rf.me {
							continue
						} else {
							ok := rf.sendRequestVote(i, &rqArgs, &rqReply)
							if !ok {
								continue
								// TODO: 服务器错误应该做什么
							}
						}
					}
				} else if st == "candidate" { // st == "follower"
					rf.mu.Lock()
					voteN := rf.voteNum
					votefor := rf.votedFor
					me := rf.me
					rf.mu.Unlock()

					if voteN >= len(rf.peers)/2 && votefor == me {
						rf.mu.Lock()
						rf.state = "leader"
						rf.persister.raftstate = []byte(rf.state)
						
						rf.mu.Unlock()
					} // voteN >= len(rf.peers) / 2
				} // st == "candidate"
			}
		}
```

op的状态：

- -1：收到了ae，继续作为follower运行
- 0 ：初始状态
- 1：开始选举
- 2：leader状态，发送心跳信息
- 3：超时了，可以回到初始状态重新开始选举
- 4：被其他更新的leader取代了



#### 02 选举不出leader

debug了很久，发现是一个很简单的错误，在循环发送rpc的时候，把rpc以外的内容包含在内了，导致一直出现follower超时然后重新选举的情况，正常情况是votecond会阻塞，但也有其他错误原因，比如错误地唤醒了计时线程，但计时线程只有三种情况被唤醒：

- 收到了当前leader的AE（过期leader不应该重置定时器）
- 转变为candidate，并开始一次选举
- 作为follower投票给了其它人

投票阶段计时器协程tThread：就是初始阶段的计时器一样，设立一个随机时间，如果到时了，那么就尝试将opSelect设置为3，若opSelect为-1或4表明节点已经不是candidate，投票阶段已经结束了，那么就直接退出即可，否则是将opSelect设置为3并通知ticker协程重新开一起一次投票。

因为一旦在收到足够选票过后，就会转换为leader并且唤醒后面的进程然后开始给所有进程发送心跳，根本不会等待计时线程结束，



### 结果展示

![image-20230522202507745](https://gitee.com/moni_world/pic_bed/raw/master/img/image-20230522202507745.png)