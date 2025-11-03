---
layout: post
title: "MIT 6.824 lab3-B raft log replica"
subtitle: "Raft共识算法--日志复制"
date: 2025-10-28
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

原本实现,性能问题,性能分析工具使用,结构优化

# 日志复制简介

日志复制的过程很简单:leader收到客户端请求,追加到日志中,并行发送给所有follower。收到多数成功复制回复后，commit日志。

其中有一些细节需要注意：

1. follower因为各种原因没有收到rpc或者回复成功，则leader持续发送
2. leader commit一条日志条目时，也会连带commit之前的日志条目
3. leader维护最高的已知将被commit的日志条目的index，用这个信息通知follower commit其自己的日志条目
4. leader发送的rpc包含对应follower的nextIndex的前一个索引（previndex），用来判断日志一致性
5. 若follower和leader不一致，则强制要求follower与leader保持一致。
6. leader只能commit当前任期的日志条目

# 具体实现

## leader发送rpc

主要在于sendEntries函数，用来实现entry的发送和处理rpc回复逻辑。同时用作发送心跳（entry为0）.

首先记录下开始发送时的状态，防止在发送过程中发生了超时而导致任期更换，状态变更。

然后起多个go routine并行发送rpc，当rpc回复成功后，检查回复有效性，比如如果回复的term更高则意味着当前leader已经为旧leader，应该退位为follower等。回复有效，则更新此follower的nextIndex和matchIndex信息。

若rpc回复中显示失败，则意味着prevIndex不匹配，需要更新后重试，这里有一个优化策略是，在rpc reply中加上冲突位置的index和term信息，可以让leader一次性找到冲突位置并发送后续所有日志条目，减小rpc开销。

代码如下：

```go
func (rf *Raft) sendEntries(entries []LogEntry) {
	rf.mu.Lock()

	term := rf.currentTerm
	commitIndex := rf.commitIndex
	nextIndexSnapshot := make([]int, len(rf.nextIndex))
	copy(nextIndexSnapshot, rf.nextIndex)
	rf.mu.Unlock()

	//for synchronization

	for i := range rf.peers {
		if i == rf.me {
			continue
		}

		go func(server int) {
			rf.mu.Lock()

			if rf.state != Leader {
				logp(dWarn, rf.me, rf.currentTerm, rf.state, "is not leader anymore stop sending AppendEntries RPC to S%d", server)
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
				LeaderCommit: commitIndex,
				Entries:      entries,
			}

			if nextIndexSnapshot[server] <= rf.getLastIndex() {
				args.Entries = rf.log[nextIndexSnapshot[server]:]
			} else {
				args.Entries = []LogEntry{}
			}

			reply := AppendEntriesReply{}

			if rf.sendAppendEntries(server, &args, &reply) {
				rf.mu.Lock()
				logp(dLeader, rf.me, rf.currentTerm, rf.state, "send AppendEntries RPC to S%d success", server)

				if rf.state != Leader || rf.currentTerm != term {
					rf.mu.Unlock()
					rf.sendHeartbeat(false)
					return
				}

				if reply.Term > rf.currentTerm {
					logp(dLeader, rf.me, rf.currentTerm, rf.state, "is out of term,term %d -> term %d", rf.currentTerm, reply.Term)
					rf.currentTerm = reply.Term
					rf.state = Follower
					rf.voteFor = -1
					rf.mu.Unlock()
					rf.sendHeartbeat(false)
					return
				}
				if reply.Success {
					logp(dAppend, rf.me, rf.currentTerm, rf.state, "AppendEntries success on S%d", server)
					if args.PrevLogIndex != rf.nextIndex[server]-1 {
						rf.mu.Unlock()
						return
					}
					if len(args.Entries) > 0 {
						rf.nextIndex[server] = args.PrevLogIndex + len(args.Entries) + 1
						rf.matchIndex[server] = rf.nextIndex[server] - 1
					}
					rf.mu.Unlock()
					return
				} else {
					logp(dAppend, rf.me, rf.currentTerm, rf.state, "AppendEntries fail on S%d,with nextIndex %d PrevLogIndex %d PrevLogTerm %d", server, rf.nextIndex[server], prevLogIndex, prevLogTerm)

					if reply.ConflictTerm == -1{
						rf.nextIndex[server] = reply.ConflictIndex
					} else {
						foundConflictTerm := false
						for i := args.PrevLogIndex;i>=0;i-- {
							if rf.log[i].Term == reply.ConflictTerm {
								rf.nextIndex[server] = i + 1
								foundConflictTerm = true
								break
							}
						}
						if !foundConflictTerm {
							rf.nextIndex[server] = reply.ConflictIndex
						}
					}
					if rf.nextIndex[server] < 1 {
						rf.nextIndex[server] = 1
					}
					rf.mu.Unlock()
				}
			}
		}(i)
	}
}
```

当然这个函数极为丑陋，内部耦合度过高，在后续处理snapshot时，因为对reply的处理没有进行抽象，导致大量的重复代码。此外，此函数没有实现重试机制，仅靠心跳触发，导致等待时间过长。而且即使加了重试机制，由于go routine内部频繁对锁进行操作，导致性能很低。这个问题在snapshot处得到了优化。

## leader参数更新

前文提到，leader检测到某一个entry的replica数目达到大多数之后才可以提交。这个检查逻辑不应该放在go routine中进行，否则增大代码耦合度导致难以优化。相反，此逻辑可以抽象出来，放在主循环或者单独的routine进行处理，定期检查matchIndex中大于当前commitIndex的数量，若满足，则说明可以提交，更新commitIndex即可。注意，此go routine仅进行是否可提交的判断任务。

而真正的提交操作，则放在另一个routine中。这里体现了leader中设置`lastApplied`和`commmitIndex`两个参数的用途：

1. 前者标识真正应用到状态机的最后一个index
2. 后者标识现在最后一个可以被commit的index，但是不一定被应用到状态机中了

代码片段如下：

```go
func (rf *Raft) applyLogEntry() {
	for !rf.killed() {
		rf.mu.Lock()
		if rf.commitIndex > rf.lastApplied {
			for rf.lastApplied < rf.commitIndex {
				rf.lastApplied += 1
				rf.commitEntries([]LogEntry{rf.log[rf.lastApplied]})
				msg := raftapi.ApplyMsg{
					CommandValid:  true,
					Command:       rf.log[rf.lastApplied].Command,
					CommandIndex:  rf.lastApplied,
					SnapshotValid: false,
				}
				logp(dCommit, rf.me, rf.currentTerm, rf.state, "apply log entry to applyCh")
				rf.applyCh <- msg
			}
		}
		rf.mu.Unlock()
		time.Sleep(10 * time.Millisecond)
	}
}

// below are logic to update commitIndex,which is in ticker()
for i := rf.commitIndex + 1; i <= rf.getLastIndex(); i++ {
				if rf.log[i].Term == rf.currentTerm {
					counts := 1
					for j := range rf.peers {
						if j != rf.me && rf.matchIndex[j] >= i {
							counts += 1
						}
					}
					if counts >= rf.majority {
						rf.commitIndex = i
					}
				}
			}
			rf.mu.Unlock()
			time.Sleep(10 * time.Millisecond)
```

## AppendEntries RPC handler

按照论文Figure 2来实现就好

为了实现prevIndex不匹配时的优化，更新reply的rpc定义为：

```go
type AppendEntriesReply struct {
	Term    int
	Success bool
	ConflictIndex int
	ConflictTerm  int
}
```

然后在发生冲突处设置两个字段，具体为：

```go
// 如果超出了自己的lastIndex，说明落后太多了，直接返回自己应该需要的下一个index即可
	if args.PrevLogIndex > rf.getLastIndex() {
		reply.ConflictIndex = len(rf.log)
		reply.ConflictTerm = -1
		return
	}
// 如果是Term不匹配，则直接找到当前Term的第一个index后返回
	if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
		logp(dAppend, rf.me, rf.currentTerm, rf.state, "log[PrevLogIndex].Term != args.PrevLogTerm,refuse")
		reply.ConflictTerm = rf.log[args.PrevLogIndex].Term

		conflictIndex := args.PrevLogIndex
		for conflictIndex > 0 && rf.log[conflictIndex].Term == reply.ConflictTerm {
			conflictIndex -= 1
		}
		reply.ConflictIndex = conflictIndex 
		return
	}

//日志处理逻辑
//遍历发来的日志条目
	for i := 0; i < len(args.Entries); i++ {
		index := args.PrevLogIndex + 1 + i
    //如果超出当前index，则直接追加到后面即可
		if index > rf.getLastIndex() {
			logp(dAppend,rf.me,rf.currentTerm,rf.state,"index %d > rf.getLastIndex() %d, append the rest entries %v", index, rf.getLastIndex(), args.Entries[i:])
			rf.log = append(rf.log, args.Entries[i:]...)
			break
		}
    //如果在相同index处term不匹配，则直接截断后使用leader发来的日志条目
		if rf.log[index].Term != args.Entries[i].Term {
			logp(dAppend, rf.me, rf.currentTerm, rf.state, "Term not the same at the same log index,truncate")
			rf.log = rf.log[:index]
			rf.log = append(rf.log, args.Entries[i:]...)
			logp(dAppend,rf.me,rf.currentTerm,rf.state,"after truncate, log is %v", rf.log)
			break
		}
	}

//最后根据leader的commit信息来更新自己的
	if args.LeaderCommit > rf.commitIndex {
		logp(dAppend, rf.me, rf.currentTerm, rf.state, " args.LeaderCommit %d > rf.commitIndex %d ,update to %d", args.LeaderCommit, rf.commitIndex, min(args.LeaderCommit, rf.getLastIndex()))
		rf.commitIndex = min(args.LeaderCommit, rf.getLastIndex())
	}
```

lab3 B容易出的错误是，忘记更新nextIndex和matchIndex，commitIndex，以及搞不清这几个参数的具体含义和作用，导致瞎写一气，啥也不是。
这个版本的代码处理速度非常慢，使用pprof工具分析查看可知，对锁的频繁操作造成了大量的等待时间，后续对其进行了架构的优化。

![alt text](/img/post_img/raft-3b-pprof.png)