---
layout: post
title: "MIT 6.824 lab3-CD raft persistence and snapshot"
subtitle: "Raft共识算法--持久化与日志快照"
date: 2025-11-3
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

# 持久化与快照说明

持久化：为了使server重启后可以恢复原来的状态，按照Figure2，保存currentTerm，votedFor和log[]三个状态。

快照：为了防止log无限增大占用太多空间，采用快照机制，并持久化在磁盘中。

# 快照带来的问题

由于引入快照，物理索引和逻辑索引不再统一

eg.

快照前：

logic index :  0  1  2  3  4
physic index:  0  1  2  3  4
log entry   : c1 c2 c3 c4 c5

当对索引2处做snapshot后：

logic index : 3  4
physic index: 0  1
log entry   :c4 c5

可以看到，在snapshot后，物理index和逻辑index之间差了一个首entry的逻辑索引值。因此，需要将逻辑索引保存在log entry中。

在代码中凡是涉及到index操作的（如log读取等），都需要对其做修正。

---

快照带来的另一个问题是：在leader和follower的日志不统一时，follower会发送conflictIndex供leader修改prevIndex。当prevIndex小于leader的第一个log entry的逻辑索引时，代表此index处的日志条目已经被快照了。此时leader应该向follower发送InstallSnapshot RPC来更新follower的信息。

# 实现

## 架构优化

上一篇blog提到，sendEntries的函数耦合太严重造成锁上的性能问题，这个函数可以分为几大逻辑：

1. 判断是否需要发送快照以及构造发送AppendEntries和InstallSnapshot的参数
2. 实际的发送RPC操作
3. 处理rpc回复结果

此外，将心跳与实际append操作进行解耦，方便不同粒度的控制rpc发送频次
所以我们将原来的一个函数拆成这样:

```go
// buildOutbound constructs either snapshot or append args for a given follower.
// withEntries controls whether to include log entries (append) or send an empty heartbeat.
// Returns: needSnapshot, snapshotArgs, appendArgs, valid (still leader in term when built).
func (rf *Raft) buildOutbound(server int, term int, withEntries bool) (bool, InstallSnapshotArgs, AppendEntriesArgs, bool) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.state != Leader || rf.currentTerm != term {
		return false, InstallSnapshotArgs{}, AppendEntriesArgs{}, false
	}

	first := rf.getFirstLog()
	prevLogIndex := rf.nextIndex[server] - 1
	// Need snapshot
	if prevLogIndex < first.Index {
		snap := InstallSnapshotArgs{
			Term:             term,
			LeaderId:         rf.me,
			LastIncludeIndex: first.Index,
			LastIncludeTerm:  first.Term,
			Data:             rf.persister.ReadSnapshot(),
		}
		return true, snap, AppendEntriesArgs{}, true
	}

	prevLogTerm := rf.log[prevLogIndex-first.Index].Term
	leaderCommit := rf.commitIndex

	args := AppendEntriesArgs{
		Term:         term,
		LeaderID:     rf.me,
		PrevLogIndex: prevLogIndex,
		PrevLogTerm:  prevLogTerm,
		LeaderCommit: leaderCommit,
	}
	if withEntries && rf.nextIndex[server] <= rf.getLastIndex() {
		args.Entries = rf.log[rf.nextIndex[server]-first.Index:]
	} else {
		args.Entries = []LogEntry{}
	}
	return false, InstallSnapshotArgs{}, args, true
}

// handleAppendReply updates follower progress and handles term changes.
// Returns (done) whether replication to this follower can stop.
func (rf *Raft) handleAppendReply(server int, term int, args AppendEntriesArgs, reply AppendEntriesReply) (done bool) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.state != Leader || rf.currentTerm != term {
		return true
	}

	if reply.Term > rf.currentTerm {
		rf.currentTerm = reply.Term
		rf.state = Follower
		rf.voteFor = -1
		rf.persist(false, nil)
		go rf.sendHeartbeat(false)
		return true
	}

	if reply.Success {
		if args.PrevLogIndex != rf.nextIndex[server]-1 {
			return true
		}
		if len(args.Entries) > 0 {
			rf.nextIndex[server] = args.PrevLogIndex + len(args.Entries) + 1
			rf.matchIndex[server] = rf.nextIndex[server] - 1
		}
		return true
	}

	// conflict resolution
	if reply.ConflictIndex <= rf.getFirstLog().Index {
		rf.nextIndex[server] = rf.getFirstLog().Index
	} else {
		rf.nextIndex[server] = reply.ConflictIndex
	}
	return false
}

// replicateToPeer keeps trying to bring one follower up to date.
// withEntries=true sends entries; false sends heartbeat-only (but may still install snapshot if needed).
func (rf *Raft) replicateToPeer(server int, withEntries bool, term int) {
	localWithEntries := withEntries
	for !rf.killed() {
		needSnap, snapArgs, appArgs, ok := rf.buildOutbound(server, term, localWithEntries)
		if !ok {
			return
		}

		if needSnap {
			reply := InstallSnapshotReply{}
			if rf.sendInstallSnapshot(server, &snapArgs, &reply) {
				rf.mu.Lock()
				if rf.state != Leader || rf.currentTerm != term {
					rf.mu.Unlock()
					return
				}
				if reply.Term > rf.currentTerm {
					rf.currentTerm = reply.Term
					rf.state = Follower
					rf.voteFor = -1
					rf.persist(false, nil)
					rf.mu.Unlock()
					rf.sendHeartbeat(false)
					return
				}
				// successful install snapshot advances indices
				rf.nextIndex[server] = snapArgs.LastIncludeIndex + 1
				rf.matchIndex[server] = snapArgs.LastIncludeIndex
				rf.mu.Unlock()
				// after snapshot, try append entries
				localWithEntries = true
				continue
			}
			time.Sleep(50 * time.Millisecond)
			continue
		}

		// AppendEntries path
		reply := AppendEntriesReply{}
		if rf.sendAppendEntries(server, &appArgs, &reply) {
			done := rf.handleAppendReply(server, term, appArgs, reply)
			if done {
				return
			}
			// If this was a heartbeat and we hit a conflict, escalate to append
			localWithEntries = true
			continue
		}
		// RPC failed; retry
		time.Sleep(50 * time.Millisecond)
		continue
	}
}

// broadcastHeartbeat sends heartbeats (no log entries) to all followers.
func (rf *Raft) broadcastHeartbeat() {
	rf.mu.Lock()
	term := rf.currentTerm
	rf.mu.Unlock()
	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		go rf.replicateToPeer(i, false, term)
	}
}

// broadcastAppend sends append entries (with log entries when available) to all followers.
func (rf *Raft) broadcastAppend() {
	rf.mu.Lock()
	term := rf.currentTerm
	rf.mu.Unlock()
	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		go rf.replicateToPeer(i, true, term)
	}
}
```

通过逻辑的解耦，降低了锁的开销，使整体性能尤其是3B测试和3D测试的性能提高了50%以上。

## RPC handler

注意在InstallSnapshot RPC中也需要处理term不匹配导致的退位问题

另外，需要处理当snapshot造成日志需要截断时的操作

```go
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
	rf.mu.Lock()

	// term handling
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		rf.mu.Unlock()
		return
	}
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.voteFor = -1
		rf.state = Follower
		rf.persist(false, nil)
	}
	reply.Term = rf.currentTerm
	// receiving valid RPC resets election timer
	rf.electionTimer.Reset(rf.getNewElectionTimeout())

	// If snapshot is outdated (we already have equal/newer snapshot), ignore
	if args.LastIncludeIndex <= rf.getFirstLog().Index {
		rf.mu.Unlock()
		return
	}

	// Rebuild log with snapshot sentinel and an optional suffix to retain
	firstIdx := rf.getFirstLog().Index
	lastIdx := rf.getLastIndex()

	var newLog []LogEntry
	if args.LastIncludeIndex <= lastIdx {
		// There may be a suffix to keep
		cut := args.LastIncludeIndex - firstIdx
		// Guard: cut could be out of bound if state is inconsistent
		if cut >= 0 && cut < len(rf.log) && rf.log[cut].Term == args.LastIncludeTerm {
			// keep suffix strictly after snapshot index
			suffix := rf.log[cut+1:]
			newLog = make([]LogEntry, 1+len(suffix))
			newLog[0] = LogEntry{Index: args.LastIncludeIndex, Term: args.LastIncludeTerm}
			copy(newLog[1:], suffix)
		} else {
			// terms conflict; drop everything and start fresh from sentinel
			newLog = []LogEntry{{Index: args.LastIncludeIndex, Term: args.LastIncludeTerm}}
		}
	} else {
		// Snapshot goes beyond our last log; drop everything
		newLog = []LogEntry{{Index: args.LastIncludeIndex, Term: args.LastIncludeTerm}}
	}
	rf.log = newLog

	// bump commitIndex/lastApplied to at least snapshot index
	if rf.commitIndex < args.LastIncludeIndex {
		rf.commitIndex = args.LastIncludeIndex
	}
	if rf.lastApplied < args.LastIncludeIndex {
		rf.lastApplied = args.LastIncludeIndex
	}

	// Persist new state and snapshot atomically
	rf.persist(true, &args.Data)

	// deliver snapshot to service outside the lock
	msg := raftapi.ApplyMsg{
		SnapshotValid: true,
		Snapshot:      args.Data,
		SnapshotTerm:  args.LastIncludeTerm,
		SnapshotIndex: args.LastIncludeIndex,
	}
	rf.mu.Unlock()
	rf.applyCh <- msg // send message out the critical section!
}
```

# 最终性能

```
=== RUN   TestInitialElection3A
Test (3A): initial election (reliable network)...
  ... Passed --  time  3.0s #peers 3 #RPCs    54 #Ops    0
--- PASS: TestInitialElection3A (3.02s)
=== RUN   TestReElection3A
Test (3A): election after network failure (reliable network)...
  ... Passed --  time  4.5s #peers 3 #RPCs   131 #Ops    0
--- PASS: TestReElection3A (4.51s)
=== RUN   TestManyElections3A
Test (3A): multiple elections (reliable network)...
  ... Passed --  time  5.5s #peers 7 #RPCs   474 #Ops    0
--- PASS: TestManyElections3A (5.52s)
=== RUN   TestBasicAgree3B
Test (3B): basic agreement (reliable network)...
  ... Passed --  time  0.7s #peers 3 #RPCs    18 #Ops    0
--- PASS: TestBasicAgree3B (0.66s)
=== RUN   TestRPCBytes3B
Test (3B): RPC byte count (reliable network)...
  ... Passed --  time  1.5s #peers 3 #RPCs    50 #Ops    0
--- PASS: TestRPCBytes3B (1.53s)
=== RUN   TestFollowerFailure3B
Test (3B): test progressive failure of followers (reliable network)...
  ... Passed --  time  4.9s #peers 3 #RPCs   127 #Ops    0
--- PASS: TestFollowerFailure3B (4.89s)
=== RUN   TestLeaderFailure3B
Test (3B): test failure of leaders (reliable network)...
  ... Passed --  time  5.0s #peers 3 #RPCs   221 #Ops    0
--- PASS: TestLeaderFailure3B (4.99s)
=== RUN   TestFailAgree3B
Test (3B): agreement after follower reconnects (reliable network)...
  ... Passed --  time  5.5s #peers 3 #RPCs   124 #Ops    0
--- PASS: TestFailAgree3B (5.51s)
=== RUN   TestFailNoAgree3B
Test (3B): no agreement if too many followers disconnect (reliable network)...
  ... Passed --  time  3.5s #peers 5 #RPCs   218 #Ops    0
--- PASS: TestFailNoAgree3B (3.50s)
=== RUN   TestConcurrentStarts3B
Test (3B): concurrent Start()s (reliable network)...
  ... Passed --  time  0.7s #peers 3 #RPCs    22 #Ops    0
--- PASS: TestConcurrentStarts3B (0.70s)
=== RUN   TestRejoin3B
Test (3B): rejoin of partitioned leader (reliable network)...
  ... Passed --  time  4.1s #peers 3 #RPCs   167 #Ops    0
--- PASS: TestRejoin3B (4.10s)
=== RUN   TestBackup3B
Test (3B): leader backs up quickly over incorrect follower logs (reliable network)...
  ... Passed --  time 17.1s #peers 5 #RPCs  3716 #Ops    0
--- PASS: TestBackup3B (17.13s)
=== RUN   TestCount3B
Test (3B): RPC counts aren't too high (reliable network)...
  ... Passed --  time  2.1s #peers 3 #RPCs    60 #Ops    0
--- PASS: TestCount3B (2.06s)
=== RUN   TestPersist13C
Test (3C): basic persistence (reliable network)...
  ... Passed --  time  4.0s #peers 3 #RPCs    80 #Ops    0
--- PASS: TestPersist13C (3.95s)
=== RUN   TestPersist23C
Test (3C): more persistence (reliable network)...
  ... Passed --  time 12.5s #peers 5 #RPCs   385 #Ops    0
--- PASS: TestPersist23C (12.55s)
=== RUN   TestPersist33C
Test (3C): partitioned leader and one follower crash, leader restarts (reliable network)...
  ... Passed --  time  1.8s #peers 3 #RPCs    42 #Ops    0
--- PASS: TestPersist33C (1.83s)
=== RUN   TestFigure83C
Test (3C): Figure 8 (reliable network)...
  ... Passed --  time 32.9s #peers 5 #RPCs  1304 #Ops    0
--- PASS: TestFigure83C (32.90s)
=== RUN   TestUnreliableAgree3C
Test (3C): unreliable agreement (unreliable network)...
  ... Passed --  time  3.6s #peers 5 #RPCs  1419 #Ops    0
--- PASS: TestUnreliableAgree3C (3.59s)
=== RUN   TestFigure8Unreliable3C
Test (3C): Figure 8 (unreliable) (unreliable network)...
  ... Passed --  time 30.1s #peers 5 #RPCs 20544 #Ops    0
--- PASS: TestFigure8Unreliable3C (30.13s)
=== RUN   TestReliableChurn3C
Test (3C): churn (reliable network)...
  ... Passed --  time 16.2s #peers 5 #RPCs  9140 #Ops    0
--- PASS: TestReliableChurn3C (16.15s)
=== RUN   TestUnreliableChurn3C
Test (3C): unreliable churn (unreliable network)...
  ... Passed --  time 16.1s #peers 5 #RPCs  3029 #Ops    0
--- PASS: TestUnreliableChurn3C (16.11s)
=== RUN   TestSnapshotBasic3D
Test (3D): snapshots basic (reliable network)...
  ... Passed --  time  3.4s #peers 3 #RPCs   516 #Ops    0
--- PASS: TestSnapshotBasic3D (3.36s)
=== RUN   TestSnapshotInstall3D
Test (3D): install snapshots (disconnect) (reliable network)...
  ... Passed --  time 38.7s #peers 3 #RPCs  1673 #Ops    0
--- PASS: TestSnapshotInstall3D (38.70s)
=== RUN   TestSnapshotInstallUnreliable3D
Test (3D): install snapshots (disconnect) (unreliable network)...
  ... Passed --  time 42.5s #peers 3 #RPCs  2014 #Ops    0
--- PASS: TestSnapshotInstallUnreliable3D (42.49s)
=== RUN   TestSnapshotInstallCrash3D
Test (3D): install snapshots (crash) (reliable network)...
  ... Passed --  time 29.1s #peers 3 #RPCs  1217 #Ops    0
--- PASS: TestSnapshotInstallCrash3D (29.07s)
=== RUN   TestSnapshotInstallUnCrash3D
Test (3D): install snapshots (crash) (unreliable network)...
  ... Passed --  time 31.5s #peers 3 #RPCs  1562 #Ops    0
--- PASS: TestSnapshotInstallUnCrash3D (31.49s)
=== RUN   TestSnapshotAllCrash3D
Test (3D): crash and restart all servers (unreliable network)...
  ... Passed --  time  9.8s #peers 3 #RPCs   331 #Ops    0
--- PASS: TestSnapshotAllCrash3D (9.80s)
=== RUN   TestSnapshotInit3D
Test (3D): snapshot initialization after crash (unreliable network)...
  ... Passed --  time  2.9s #peers 3 #RPCs    90 #Ops    0
--- PASS: TestSnapshotInit3D (2.91s)
PASS
ok      6.5840/raft1    333.975s
```