---
layout: post
title: "MIT 6.824 lab4 C kvraft snapshot"
subtitle: "采用snapshot的kv服务"
date: 2025-11-18
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

## 实验背景

在完成了 Part A 和 Part B 后，我们已经有了一个可以正确工作的分布式键值存储系统。但是存在一个严重的问题：随着系统运行时间增长，Raft 日志会无限增长。这带来两个问题：

1. **内存占用过大**：日志条目越来越多，占用大量内存
2. **重启时间长**：服务器重启时需要重放完整的日志来恢复状态，日志越长重启越慢

Part C 的任务就是实现快照机制来解决这些问题。

## 快照机制原理

快照的核心思想很简单：定期将状态机的完整状态保存下来，然后丢弃快照之前的所有日志。

比如系统执行了 1000 条命令后，我们创建一个快照保存当前状态，然后就可以丢弃前 1000 条日志。这样：
- 日志大小被限制在合理范围内
- 重启时只需加载快照，不需要重放所有历史命令

## 实现步骤详解

### 第一步：实现 KVServer 的快照功能

KVServer 需要实现两个方法：`Snapshot()` 和 `Restore()`。

#### Snapshot() 方法

这个方法需要序列化服务器的所有状态。

```go
func (kv *KVServer) Snapshot() []byte {
    w := new(bytes.Buffer)
    e := labgob.NewEncoder(w)
    
    // 编码所有需要持久化的状态
    e.Encode(kv.data)        // 键值数据
    e.Encode(kv.versions)    // 版本信息
    e.Encode(kv.lastApplied) // 去重：每个客户端最后处理的序列号
    e.Encode(kv.lastResults) // 去重：每个客户端的最后响应缓存
    
    return w.Bytes()
}
```

**关键点**：必须包含去重信息（lastApplied 和 lastResults）。

考虑这个场景：
1. 客户端发送 Put 请求（seqNum=5）
2. 服务器执行并缓存了结果
3. 创建快照并重启
4. 客户端重试请求（因为响应丢失）
5. 如果快照没有包含去重信息，服务器会重复执行这个请求

所以去重信息必须持久化，否则重启后会破坏精确一次语义。

#### Restore() 方法

这个方法从快照恢复状态：

```go
func (kv *KVServer) Restore(data []byte) {
    if len(data) == 0 {
        return
    }
    
    r := bytes.NewBuffer(data)
    d := labgob.NewDecoder(r)
    
    var restoredData map[string]string
    var restoredVersions map[string]rpc.Tversion
    var restoredLastApplied map[int64]int64
    var restoredLastResults map[int64]any
    
    d.Decode(&restoredData)
    d.Decode(&restoredVersions)
    d.Decode(&restoredLastApplied)
    d.Decode(&restoredLastResults)
    
    // 恢复所有状态
    kv.data = restoredData
    kv.versions = restoredVersions
    kv.lastApplied = restoredLastApplied
    kv.lastResults = restoredLastResults
}
```

**注意事项**：
- 检查快照是否为空（len(data) == 0）
- 解码顺序必须和编码顺序一致
- 使用临时变量先解码，验证成功后再赋值

### 第二步：在 RSM 中添加快照管理

RSM 需要做三件事：
1. 启动时恢复快照
2. 监控日志大小，决定何时创建快照
3. 处理来自 Raft 的快照消息

#### 启动时恢复快照

在 `MakeRSM()` 中添加：

```go
func MakeRSM(...) *RSM {
    rsm := &RSM{
        // ... 初始化字段 ...
        persister: persister,  // 保存 persister 引用
    }
    
    // 创建 Raft 实例
    rsm.rf = raft.Make(servers, me, persister, rsm.applyCh)
    
    // 启动时恢复快照
    snapshot := persister.ReadSnapshot()
    if len(snapshot) > 0 {
        rsm.sm.Restore(snapshot)
    }
    
    // 启动 applier goroutine
    go rsm.reader()
    
    return rsm
}
```

**重要**：必须在启动 reader goroutine 之前恢复快照。这样可以确保状态机在处理新命令前已经处于正确的状态。

#### 监控日志大小并触发快照

在 reader() 中处理完每条命令后检查：

```go
func (rsm *RSM) reader() {
    for msg := range rsm.applyCh {
        if msg.CommandValid {
            // ... 执行命令 ...
            
            rsm.mu.Lock()
            rsm.lastApplied = msg.CommandIndex
            rsm.mu.Unlock()
            
            // 检查是否需要快照
            if rsm.maxraftstate != -1 && 
               rsm.rf.PersistBytes() >= rsm.maxraftstate*9/10 {
                
                rsm.mu.Lock()
                snapshot := rsm.sm.Snapshot()
                snapshotIndex := rsm.lastApplied
                rsm.mu.Unlock()
                
                rsm.rf.Snapshot(snapshotIndex, snapshot)
            }
        }
    }
}
```

**设计决策**：

1. **阈值设置**：使用 90% 阈值而不是 100%
   - 原因：Raft 状态可能在检查和快照之间继续增长
   - 留出 10% 缓冲避免超限

2. **快照时机**：在每条命令应用后检查
   - 优点：及时响应，不会严重超限
   - 缺点：会有一些检查开销
   - 替代方案：周期性检查，但可能延迟响应

3. **并发控制**：
   - 在锁内获取快照和索引，保证一致性
   - 在锁外调用 Raft，避免长时间持锁

#### 处理快照消息

当服务器作为 follower 收到 leader 的 InstallSnapshot RPC 时，Raft 会通过 applyCh 发送快照消息：

```go
func (rsm *RSM) reader() {
    for msg := range rsm.applyCh {
        // 处理快照消息
        if msg.SnapshotValid {
            rsm.mu.Lock()
            rsm.sm.Restore(msg.Snapshot)
            rsm.lastApplied = msg.SnapshotIndex
            rsm.mu.Unlock()
            continue
        }
        
        // 处理普通命令
        if msg.CommandValid {
            // ...
        }
    }
}
```

**关键点**：
- 收到快照后立即恢复，丢弃当前状态
- 更新 lastApplied 到快照索引
- 不需要通知等待的 Submit()，因为快照来自 leader，本地没有等待者

### 第三步：理解快照与日志的交互

快照机制涉及三个组件的协作：

```
KVServer (状态机)
    ↓ 调用 Snapshot()
RSM (协调层)
    ↓ 调用 rf.Snapshot()
Raft (共识层)
    → 保存快照并截断日志
```

**时序图**：

```
时间线：
T1: 应用命令 1-100
T2: RSM 检测日志过大
T3: RSM.mu.Lock()
T4: snapshot = sm.Snapshot()  ← 获取状态快照
T5: index = lastApplied       ← 记录索引
T6: RSM.mu.Unlock()
T7: rf.Snapshot(index, snapshot) ← 调用 Raft
T8: Raft 保存快照，丢弃索引 ≤ index 的日志
```

**一致性保证**：

快照必须对应一个确定的日志索引。具体来说：
- 快照包含执行前 N 条命令后的状态
- Raft 会丢弃前 N 条日志
- 如果丢弃了第 N+1 条，状态和日志就不一致了

这就是为什么要在锁内同时获取快照和 lastApplied：

```go
rsm.mu.Lock()
snapshot := rsm.sm.Snapshot()      // 状态
snapshotIndex := rsm.lastApplied   // 对应的索引
rsm.mu.Unlock()
// 这两者必须配对
```

如果不加锁，可能发生：
```
线程 A: snapshot = sm.Snapshot()     // 包含命令 1-100
线程 B: 应用命令 101
线程 B: lastApplied = 101
线程 A: index = lastApplied           // index=101，但快照只到100
线程 A: rf.Snapshot(101, snapshot)   // 不一致！
```

## 性能考虑

### 快照开销

创建快照需要：
1. 序列化整个状态机（O(n)，n = 键值对数量）
2. Raft 持久化快照（O(n)，磁盘 I/O）

如果状态很大，快照会阻塞服务。优化方案：
- 使用增量快照（更复杂）
- 后台异步快照（需要 copy-on-write）
- 本实验采用简单方案：在锁内快速序列化

### 快照频率

太频繁：
- CPU 和 I/O 开销大
- 性能下降

太稀疏：
- 日志可能超限
- 重启时恢复慢

本实验的 90% 阈值是一个平衡。

### 内存占用

快照会占用额外内存：
- Raft 持有快照副本
- 传输时可能有多个副本

优化：使用流式传输（Raft 的 InstallSnapshot 支持）

## 与 Raft Part D 的关系

Lab 3D 实现了 Raft 的快照支持，包括：
- `Snapshot(index, snapshot)`：保存快照并截断日志
- `CondInstallSnapshot()`：原子地安装快照
- InstallSnapshot RPC：leader 向 follower 传输快照

Lab 4C 在此基础上构建应用层的快照逻辑：
- 决定何时快照（应用层策略）
- 生成快照内容（应用层数据）
- 恢复快照（应用层逻辑）

两者分工明确：
- Raft：管理快照的存储和传输
- RSM/KVServer：管理快照的内容和时机

## 总结

Lab 4C 的核心是实现一个完整的快照机制，包括：

1. **状态序列化**：Snapshot() 和 Restore()
2. **触发策略**：监控日志大小，90% 阈值
3. **一致性保证**：快照和索引配对，加锁保护
4. **恢复流程**：启动时加载快照，运行时接收快照

关键挑战：
- 确定快照内容（不要忘记去重信息）
- 保证一致性（快照和索引配对）
- 正确加锁（避免并发问题）

实现后的收益：
- 日志大小可控
- 重启时间短
- 系统可以长期运行
