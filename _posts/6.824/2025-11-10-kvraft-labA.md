---
layout: post
title: "MIT 6.824 lab4 A raft与client的中间层-rsm"
subtitle: "replicated state machine"
date: 2025-11-10
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

## 引言

在分布式系统中，如何确保多个副本保持一致性是一个核心挑战。本文将详细介绍如何基于 Raft 共识算法实现一个通用的 Replicated State Machine (RSM) 层，这是 MIT 6.5840 (原 6.824) Lab 4A 的核心内容。

## 实验背景

### 问题场景

在之前的lab中,我们已经实现了一个robust的Raft服务,但是还没有解决raft如何与client进行交互的问题,lab4的第一部分需要实现一个中间层,封装来自client的请求并发送给raft,同时进行一系列异常的处理等工作.

### RSM 的作用

RSM 充当服务层（如 KV 存储）和 Raft 之间的中间层：

```
┌─────────────┐
│  KV Service │  ← 业务逻辑
├─────────────┤
│     RSM     │  ← 今天的主角
├─────────────┤
│    Raft     │  ← 共识算法
└─────────────┘
```

RSM 的核心职责是：
1. 将客户端操作提交到 Raft
2. 从 Raft 接收已提交的操作并执行
3. 将执行结果返回给调用方
4. 检测并处理 leader 变更

## 设计思路

### 核心挑战

实现 RSM 面临三个主要挑战：

**1. 请求-响应匹配**
- 多个并发的 `Submit()` 调用同时提交操作
- 如何确保每个调用收到正确的响应？

**2. Leader 变更检测**
- Leader 提交操作后可能失去 leadership
- 如何检测操作是否被新 leader 的操作覆盖？

**3. 资源清理**
- 超时、节点关闭等异常场景
- 如何避免 goroutine 和内存泄漏？

### 数据结构设计

#### Op：操作包装器

```go
type Op struct {
    ClientId int64  // 提交节点的 ID
    SeqId    int64  // 操作的唯一序列号
    Req      any    // 实际的客户端请求
}
```

`ClientId + SeqId` 组合提供全局唯一标识：
- ClientId = 节点索引（如 0, 1, 2）
- SeqId = 原子递增的序列号（1, 2, 3, ...）

#### Result：结果封装

```go
type Result struct {
    Err   rpc.Err  // OK 或 ErrWrongLeader
    Value any      // DoOp() 的返回值
}
```

#### RSM 状态管理

```go
type RSM struct {
    // ... 基础字段 ...
    
    nextSeqId  int64                 // 原子递增的序列号生成器
    pending    map[int64]chan Result // SeqId → 结果通道
    seqToIndex map[int64]int         // SeqId → Raft index
    indexToSeq map[int]int64         // Raft index → SeqId (关键！)
}
```

**关键设计：双向映射**

- `seqToIndex`：知道操作的 SeqId，查找它在 Raft 日志中的位置
- `indexToSeq`：知道日志索引，查找我们期望在这个位置看到的操作

`indexToSeq` 是检测 leader 变更的关键：
```
期望: indexToSeq[10] = seqId_A
实际: applyCh 返回 index=10, seqId_B

如果 seqId_A ≠ seqId_B → Leader 变更！
```

## 核心实现

### Submit()：提交操作

`Submit()` 是 RSM 的对外接口，负责提交操作并等待结果。

#### 关键步骤

**1. 生成唯一标识**
```go
seqId := atomic.AddInt64(&rsm.nextSeqId, 1)
op := Op{
    ClientId: int64(rsm.me),
    SeqId:    seqId,
    Req:      req,
}
```

**2. 提交到 Raft**
```go
index, term, isLeader := rsm.rf.Start(op)
if !isLeader {
    return rpc.ErrWrongLeader, nil
}
```

**3. 注册等待通道**
```go
rsm.mu.Lock()
resultChan := make(chan Result, 1)
rsm.pending[seqId] = resultChan
rsm.seqToIndex[seqId] = index
rsm.indexToSeq[index] = seqId  // 记录期望
rsm.mu.Unlock()
```

**4. 等待结果（带超时和 term 检查）**
```go
ticker := time.NewTicker(50 * time.Millisecond)
timeout := time.NewTimer(2 * time.Second)

for {
    select {
    case result := <-resultChan:
        return result.Err, result.Value
        
    case <-ticker.C:
        currentTerm, isStillLeader := rsm.rf.GetState()
        if !isStillLeader || currentTerm != term {
            // 清理并返回错误（注意不删除 indexToSeq）
            rsm.mu.Lock()
            delete(rsm.pending, seqId)
            delete(rsm.seqToIndex, seqId)
            rsm.mu.Unlock()
            return rpc.ErrWrongLeader, nil
        }
        
    case <-timeout.C:
        // 超时处理（同样不删除 indexToSeq）
        return rpc.ErrWrongLeader, nil
    }
}
```

**为什么 term 变化时不删除 `indexToSeq[index]`？**

因为操作可能已经被提交到 Raft 日志中，reader goroutine 需要这个映射来检测并清理。

### reader()：处理已提交的操作

reader 是一个长期运行的 goroutine，从 Raft 的 `applyCh` 读取已提交的命令并执行。

#### 核心逻辑

```go
func (rsm *RSM) reader() {
    for msg := range rsm.applyCh {
        if !msg.CommandValid {
            continue
        }
        
        op := msg.Command.(Op)
        index := msg.CommandIndex
        
        // 1. 执行命令（所有节点都执行）
        result := rsm.sm.DoOp(op.Req)
        
        rsm.mu.Lock()
        
        // 2. 检查是否有等待者
        expectedSeqId, hasWaiting := rsm.indexToSeq[index]
        
        if hasWaiting {
            // 3. 验证命令匹配（Leader 变更检测！）
            if expectedSeqId == op.SeqId && op.ClientId == int64(rsm.me) {
                //  匹配成功：这是本节点提交的预期命令
                if ch, exists := rsm.pending[op.SeqId]; exists {
                    ch <- Result{Err: rpc.OK, Value: result}
                    close(ch)
                }
            } else {
                //  不匹配：Leader 变更，命令被覆盖
                if ch, exists := rsm.pending[expectedSeqId]; exists {
                    ch <- Result{Err: rpc.ErrWrongLeader, Value: nil}
                    close(ch)
                }
            }
            
            // 4. 清理资源
            delete(rsm.pending, expectedSeqId)
            delete(rsm.seqToIndex, expectedSeqId)
            delete(rsm.indexToSeq, index)
        } else {
            // 没有等待者：可能是其他节点的操作，或已超时
            // 清理可能遗留的 indexToSeq
            delete(rsm.indexToSeq, index)
        }
        
        rsm.mu.Unlock()
    }
    
    // applyCh 关闭后清理所有等待者
    rsm.cleanupOnShutdown()
}
```

#### Leader 变更检测示例

```
场景：节点 A 是 leader，提交操作 op1 到 index=10

节点 A:
  Submit() → Start(op1) → index=10
  注册 indexToSeq[10] = op1.SeqId
  
[A 失去 leadership，B 成为新 leader]

节点 B: 
  在 index=10 提交 op2
  
节点 A:
  applyCh ← msg{index=10, command=op2}
  reader: 
    expectedSeqId = indexToSeq[10] = op1.SeqId
    actualSeqId = op2.SeqId
    op1.SeqId ≠ op2.SeqId → 检测到 Leader 变更！
    返回 ErrWrongLeader 给等待 op1 的 Submit()
```

## 关键问题与解决

### 问题 1：并发竞态导致重复操作

**现象**：测试中发现 counter 变成 54 而不是预期的 50。

**根因**：Raft 的 `Start()` 方法存在并发 bug：

```go
// 原始实现（有问题）
rf.mu.Lock()
rf.log = append(rf.log, entry)
rf.mu.Unlock()

rf.broadcastAppend()

rf.mu.Lock()
index := rf.getLastIndex()  // 可能已被其他线程修改！
rf.mu.Unlock()
return index, term, isLeader
```

两个并发的 `Start()` 调用可能返回相同的 index，导致 `indexToSeq` 映射被覆盖。

**修复**：在追加日志时立即捕获 index

```go
rf.mu.Lock()
entry := LogEntry{Index: rf.getLastIndex() + 1, ...}
rf.log = append(rf.log, entry)

// 立即捕获 index 和 term（仍持有锁）
index := entry.Index
term := rf.currentTerm
rf.mu.Unlock()

rf.broadcastAppend()
return index, term, isLeader
```


## 设计亮点

1. **双向映射机制**：通过 `indexToSeq` 优雅地检测 leader 变更
2. **超时保护**：2 秒超时 + term 轮询，避免永久阻塞
3. **资源清理策略**：谁创建谁清理，reader 负责最终清理
4. **并发安全**：原子操作 + 互斥锁 + buffered channel

## 经验教训

1. **并发 bug 难以发现**：Raft `Start()` 的 bug 只在高并发场景下才会触发
2. **日志分析很重要**：通过添加详细日志快速定位问题
3. **边界条件要考虑**：超时、关闭、leader 变更都要妥善处理
4. **测试要充分**：连续运行多次才能发现偶发性问题

## 总结

实现 RSM 的核心在于：
1. 设计合理的数据结构来追踪请求和响应
2. 使用双向映射检测 leader 变更
3. 处理好各种边界条件和异常场景
4. 保证并发安全和资源清理