---
layout: post
title: "MIT 6.824 lab3-A raft leader election"
subtitle: "Raft共识算法--领导者选举"
date: 2025-10-26
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

# 一些介绍和牢骚

lab3要求手写实现一个raft共识算法，part A从领导者选举开始

先回顾下领导者选举的过程:

1. 集群启动，所有server均为follower状态
2. 一个server A 的election timeout超时，转为candidate状态
3. 它增加自己的term号，投票给自己，并行向其他server申请投票

一个选举过程在以下三种情况下结束:

1. A得到大多数server的投票，成为leader
2. A收到来自更高term的心跳信息或者请求投票信息，转换为follower
3. election timeout，没有选出leader，进入下一轮投票阶段

在具体实现过程中，关键是严格按照Figure 2的要求来做，尤其是关于election timeout什么时候重置的问题，如果设置不当，很容易造成选举频繁发生。

此外，还有一个关键点是，如何在go中实现这个election timeout。对于非leader，它需要一只在后台计时，且要在收到心跳后重置。转换为leader后，又要停止计时。lab的初始框架中给的是使用`time.Sleep()`，这个用起来很麻烦，因为sleep是阻塞式的，不是很直观的符合raft的逻辑，这也耗费了我大量的时间。

最后，按照这个计时特点，选择在raft结构体中设置两个timer计时器来分别做election timeout和heartbeat。

# 具体实现

第一次接触共识算法时，很容易陷入细节，又忽略细节。需要在细节的海洋中艰难地对着日志信息不断试错，找出不符合论文的点，发现bug，修正bug，修正逻辑，重构代码。

本实验官方推荐6h，而我花了18h。很大一部分原因在于没有慢下来按照论文的要求来实现，有想当然的部分，还有病急乱求医，乱问ai，乱改代码造成的逻辑混乱等问题。

把Figure 2贴在这里以示警示：

![raft figure 2](/img/post_img/raft-figure2.png)

## Raft结构体

按照论文来即可：

```go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *tester.Persister   // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	// Your data here (3A, 3B, 3C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.
	state       Status
	currentTerm int
	voteFor     int
	log         []LogEntry

	commitIndex int
	lastApplied int

	nextIndex  []int
	matchIndex []int

	majority int

	//helper status
	rng *rand.Rand
	electionTimer  *time.Timer
	heartbeatTimer *time.Timer
}
```

## 初始化

同样，按照论文的要求，初始化相应字段：

```go
rf := &Raft{
		peers:          peers,
		persister:      persister,
		me:             me,
		state:          Follower,
		currentTerm:    0,
		voteFor:        -1,
		log:            make([]LogEntry, 1),
		commitIndex:    0,
		lastApplied:    0,
		nextIndex:      make([]int, len(peers)),
		matchIndex:     make([]int, len(peers)),
		rng:            rand.New(rand.NewSource(time.Now().UnixNano())),
		heartbeatTimer: time.NewTimer(heartbeatInterval),
	}
	rf.heartbeatTimer.Stop()
	rf.electionTimer = time.NewTimer(rf.getNewElectionTimeout())
	rf.majority = len(rf.peers)/2 + 1
	for i := range peers {
		rf.nextIndex[i] = rf.getLastIndex() + 1
		rf.matchIndex[i] = 0
	}
```

需要注意的是，raft中，log的下标从1开始，0作为dummy
另，heartbeat初始时需要置为暂停，只有在当选leader后才启动。

## ticker

这里是整个raft运行的主逻辑，通过检测不同的定时器或事件，来执行相应的逻辑：

```go
func (rf *Raft) ticker() {
	for !rf.killed() {
		select {
		// election timeout
		case <-rf.electionTimer.C:
			logp(dTimer, rf.me, rf.currentTerm, rf.state, "election timeout!")
			rf.election()
		case <-rf.heartbeatTimer.C:
			rf.mu.Lock()
			if rf.state == Leader {
				logp(dTimer, rf.me, rf.currentTerm, rf.state, "sending heartbeat!")
				rf.heartbeatTimer.Reset(heartbeatInterval)
				rf.mu.Unlock()
				rf.heartBeat()
			} else {
				logp(dWarn, rf.me, rf.currentTerm, rf.state, "not a leader should not give a heartbeat!")
				rf.mu.Unlock()
			}
		default:
			time.Sleep(5 * time.Millisecond)
		}
	}
}
```

这里在heartbeat里写了关于leader的逻辑判断，其实应该是多余的。但是为了防止不知道哪里出错导致预期之外的行为发生，先留着，给个warning。

## election

election按照论文逻辑来，需要注意的是，在进入election和发送rpc之间，很可能有多个term已经过去了，也可能有很多其他的事情发生了。因此，不能假定状态在函数执行过程中保持不变。而且这一点需要做检查：如该candidate在选举过程中，其term没有发生改变。

首先，填充rpc参数：

```go
  rf.mu.Lock()
	rf.state = Candidate
	rf.currentTerm += 1
	rf.voteFor = rf.me
	votes := 1
	rf.electionTimer.Reset(rf.getNewElectionTimeout())
	currentTerm := rf.currentTerm
	lastLogIndex := rf.getLastIndex()
	lastLogTerm := rf.getLastLogTerm()
	rf.mu.Unlock()

	args := RequestVoteArgs{
		Term:         currentTerm,
		CandidateId:  rf.me,
		LastLogIndex: lastLogIndex,
		LastLogTerm:  lastLogTerm,
	}
```

这里没有直接使用`defer rf.mu.unlock()`，因为后面有个循环，虽然是循环是异步的，但还是控制一下锁的粒度。

此外，要首先记录下发起选举时，当前server的term等信息。

接下来，同时向不同的server发起投票请求：

```go
	// request votes
	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		go func(server int) {
			reply := RequestVoteReply{}
			logp(dVote, rf.me, rf.currentTerm, rf.state, "ask %d for votes", server)
			if rf.sendRequestVote(server, &args, &reply) {
				rf.mu.Lock()
				// first ensure that the server is in the same term and is still a candidate
				if rf.currentTerm == currentTerm && rf.state == Candidate {
					if reply.VoteGranted {
						votes += 1
						if votes >= rf.majority {
							logp(dLeader, rf.me, rf.currentTerm, rf.state, "become a leader by %d votes", votes)
							rf.state = Leader
							for j := range rf.peers {
								rf.nextIndex[j] = rf.getLastIndex() + 1
								rf.matchIndex[j] = 0
							}
							rf.electionTimer.Stop()
							rf.mu.Unlock()
							rf.sendHeartbeat(true)
							rf.heartBeat() //since the heart beat interval is 100ms,we send heartbeat once immediately 
							return
						} else if reply.Term > rf.currentTerm {
							logp(dVote, rf.me, rf.currentTerm, rf.state, "find a new leader %d with a higher term,change from Term %d to Term %d", server, rf.currentTerm, reply.Term)
							rf.currentTerm = reply.Term
							rf.state = Follower
							rf.mu.Unlock()
							return
						}
					}
				}
				rf.mu.Unlock() // here!!!!
			}
		}(i)
	}
```

代码中有几个细节：

1. 收到rpc回复后，首先要检查是否还是原来的term，以及是否还是candidate，因为这两个状态决定了当前执行发送投票和处理结果的go routine是否还应该继续的问题。
2. 当投票数大于大多数后，当选为leader，停止自己的election timer
3. 启动心跳，并立刻发送一轮心跳
4. 当回复是没有收到投票时，如果原因是对方的term大于自己，按照Figure 2的server规则，任何server，如果收到大于自己term的RPC，应该更新自己的term，转换为follower。

## heartbeat

当选leader后，需要定期向其他server发送心跳来维护自己的leader地位：

发送heartbeat的本质是发送`AppendEntries RPC`,因此，需要做一切追加日志所需的检查操作。只不过日志体为空。当前的检查等操作还不是很完善，将来可能还需要修改。

```go
func (rf *Raft) heartBeat() {
	rf.mu.Lock()
	
	term := rf.currentTerm
	commitIndex := rf.commitIndex
	nextIndexSnapshot := make([]int, len(rf.nextIndex))
	copy(nextIndexSnapshot, rf.nextIndex)
	rf.mu.Unlock()
	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		go func(server int) {
			rf.mu.Lock()
			if rf.state != Leader {
				logp(dWarn, rf.me, rf.currentTerm, rf.state, "is not leader anymore stop hearbeat to S%d", server)
				rf.mu.Unlock()
				return
			}
			prevLogIndex := nextIndexSnapshot[server] - 1
			prevLogTerm := rf.log[prevLogIndex].Term
			rf.mu.Unlock()
			args := AppendEntriesArgs{
				Term:         term,
				LeaderID:     rf.me,
				PrevLogIndex: prevLogIndex,
				PrevLogTerm:  prevLogTerm,
				Entries:      []LogEntry{},
				LeaderCommit: commitIndex,
			}

			reply := AppendEntriesReply{}

			logp(dLeader, rf.me, term, Leader, "send the heartbeat to S%d", server)
			if rf.sendAppendEntries(server, &args, &reply) {
				rf.mu.Lock()
				logp(dLeader, rf.me, rf.currentTerm, rf.state, "send heartbeat to S%d success", server)
				if reply.Term > rf.currentTerm {
					logp(dLeader, rf.me, rf.currentTerm, rf.state, "is out of term,term %d -> term %d", rf.currentTerm, reply.Term)
					rf.currentTerm = reply.Term
					rf.state = Follower
					rf.voteFor = -1
					rf.mu.Unlock()
					rf.sendHeartbeat(false)
					return
				}
				rf.mu.Unlock()
			}
		}(i)
	}
}
```

进入函数后，先复制进入heartbeat函数时server的各种状态，在向各个server发送heartbeat时都应该使用发起heartbeat时的状态填充参数。而不应该使用rf.xxx，同样是因为这些内容很可能在发送RPC的过程中已经改变。

对于回复，仍然检查term。如果发现更高的term，说明有server发起选举了，自己很可能已经不是server了，需要退位。停止发送心跳。

## RPC

RequestVote RPC按照Figure 2一字不落地实现即可。需要注意两个问题：

1. 如果自己是leader，且对方term高于自己，则需要停止发送心跳
2. 只有将票投出去之后，才重置`election timeout`

2来自原文：

> follower:
>
> ...
>
> if election timeout elapses without receiving AppendEntries RPC from current leader or granting vote to candidate:convert to candidate

AppendEntries RPC在之后的lab中还需要进一步完善，届时介绍。