---
layout: post
title: "MIT 6.824 lab4 B K/V service"
subtitle: "使用raft实现的key/value服务"
date: 2025-11-18
author: 渚汐
catalog: true
tags:
  - 分布式系统
---

本次实验的目标是构建一个基于 Raft 的分布式键值存储服务（KVRaft）。这个服务需要保证线性一致性，并能够在网络分区、服务器崩溃等各种故障场景下正确运行。

实验的核心挑战在于：
- 如何在 Raft 之上实现精确一次（exactly-once）语义
- 如何处理客户端重试带来的重复请求
- 如何正确实现版本控制机制
- 如何在各种故障场景下保持系统的正确性

信息流动过程如下：
```
客户端 -> KVServer.Get/Put (RPC) -> RSM.Submit -> Raft -> 
    -> 日志复制到多数节点 -> 应用到状态机 -> KVServer.DoOp -> 返回结果
```

### 服务器端设计

KVServer 的核心数据结构包括：

```go
type KVServer struct {
    me   int
    rsm  *rsm.RSM
    mu   sync.Mutex
    
    // 存储层
    data     map[string]string
    versions map[string]rpc.Tversion
    
    // 去重层
    lastApplied map[int64]int64  // clientId -> 最后处理的 seqNum
    lastResults map[int64]any    // clientId -> 缓存的响应
}
```

存储层负责维护实际的键值数据和版本号。去重层则用于实现精确一次语义，防止重复请求被多次执行。

### 客户端设计

客户端需要为每个请求分配唯一标识：

```go
type Clerk struct {
    clientId   int64  // 客户端唯一标识
    seqNum     int64  // 单调递增的序列号
    lastLeader int    // 上次成功的 leader 索引
}
```

每个请求都携带 `(clientId, seqNum)` 二元组，服务器通过这个二元组来识别和去重。

## 核心实现细节

### DoOp 方法：状态机的执行逻辑

DoOp 是整个系统的核心，它在每个服务器上以相同的顺序执行相同的操作：

```go
func (kv *KVServer) DoOp(req any) any {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    
    switch args := req.(type) {
    case rpc.PutArgs:
        // 去重检查
        if lastSeq, exists := kv.lastApplied[args.ClientId]; exists {
            if args.SeqNum <= lastSeq {
                if args.SeqNum == lastSeq {
                    // 精确重复，返回缓存结果
                    return kv.lastResults[args.ClientId]
                }
                // 旧请求，返回错误避免重复执行
                return rpc.PutReply{Err: rpc.ErrVersion}
            }
        }
        
        // 版本检查
        if kv.versions[args.Key] != args.Version {
            reply := rpc.PutReply{Err: rpc.ErrVersion}
            if args.SeqNum > lastSeq {
                kv.lastApplied[args.ClientId] = args.SeqNum
                kv.lastResults[args.ClientId] = reply
            }
            return reply
        }
        
        // 执行更新
        kv.data[args.Key] = args.Value
        kv.versions[args.Key]++
        reply := rpc.PutReply{Err: rpc.OK}
        
        if args.SeqNum > lastSeq {
            kv.lastApplied[args.ClientId] = args.SeqNum
            kv.lastResults[args.ClientId] = reply
        }
        return reply
    }
}
```

关键点：
- 所有修改都在锁保护下进行，确保原子性
- 去重检查必须在执行操作之前完成
- 只有当 `seqNum > lastSeq` 时才更新缓存，避免缓存被旧请求污染

### RPC 处理：连接客户端和状态机

RPC 处理器作为桥梁，将客户端请求提交给 Raft：

```go
func (kv *KVServer) Put(args *rpc.PutArgs, reply *rpc.PutReply) {
    if kv.killed() {
        reply.Err = rpc.ErrWrongLeader
        return
    }
    
    err, value := kv.rsm.Submit(*args)
    
    if err == rpc.ErrWrongLeader {
        reply.Err = rpc.ErrWrongLeader
        return
    }
    
    if result, ok := value.(rpc.PutReply); ok {
        *reply = result
    }
}
```

`rsm.Submit` 会将请求提交给 Raft，等待其被复制和应用，然后返回 DoOp 的执行结果。

### 客户端重试逻辑

客户端需要处理各种错误情况并正确重试：

```go
func (ck *Clerk) Put(key string, value string, version rpc.Tversion) rpc.Err {
    ck.seqNum++
    args := rpc.PutArgs{
        Key:      key,
        Value:    value,
        Version:  version,
        ClientId: ck.clientId,
        SeqNum:   ck.seqNum,
    }
    
    serverIndex := ck.lastLeader
    isFirstAttempt := true
    
    for {
        var reply rpc.PutReply
        ok := ck.clnt.Call(ck.servers[serverIndex], "KVServer.Put", &args, &reply)
        
        if ok {
            switch reply.Err {
            case rpc.OK:
                ck.lastLeader = serverIndex
                return rpc.OK
            case rpc.ErrVersion:
                if isFirstAttempt {
                    return rpc.ErrVersion
                } else {
                    return rpc.ErrMaybe
                }
            case rpc.ErrWrongLeader:
                isFirstAttempt = false
                serverIndex = (serverIndex + 1) % len(ck.servers)
                time.Sleep(10 * time.Millisecond)
            }
        } else {
            isFirstAttempt = false
            serverIndex = (serverIndex + 1) % len(ck.servers)
            time.Sleep(10 * time.Millisecond)
        }
    }
}
```

关键的语义：
- 第一次尝试收到 ErrVersion：确定操作未执行，返回 ErrVersion
- 重试后收到 ErrVersion：无法确定操作是否执行（可能第一次成功了但响应丢失），返回 ErrMaybe

## Bug 查找与解决过程

### 问题发现

运行完整测试套件时，TestConcurrent4B 测试出现间歇性失败：

```
Fatal: Reliable: Wrong number of puts: server 127 clnts &{155 4}
```

这表示服务器的版本号是 127，但客户端报告了 155 次成功的 Put 操作和 4 次 Maybe 结果。两者相差 28 次，这意味着有些 Put 被客户端计数为成功，但实际上并未在服务器上执行。

### 调试策略

首先启用详细日志，记录每次 DoOp 的执行情况：

```go
DPrintf("[Server %d DoOp] Put: clientId=%d seqNum=%d key=%s value=%s version=%d", 
    kv.me, clientId, seqNum, args.Key, args.Value, args.Version)
```

多次运行测试，捕获失败的日志文件进行分析。

### 第一个发现：旧请求的存在

在日志中发现了大量"OLD REQUEST"的警告：

```
[Server 3 DoOp] Put: OLD REQUEST detected! seqNum=49 < lastSeq=50
[Server 4 DoOp] Put: OLD REQUEST detected! seqNum=49 < lastSeq=50
```

这说明确实存在 `seqNum < lastSeq` 的情况。这是因为 Raft 会将所有提交的日志条目复制到所有服务器，即使某个请求在客户端看来已经完成，它仍会在所有服务器上被应用。

最初的代码有一个严重缺陷：

```go
// 错误的实现
kv.lastApplied[clientId] = seqNum  // 无条件更新
kv.lastResults[clientId] = reply
```

如果先处理 seqNum=19，再处理延迟到达的 seqNum=17，会导致 lastApplied 从 19 退化到 17，破坏了去重机制。

修复方法是只在 seqNum 更大时才更新缓存：

```go
if lastSeq, exists := kv.lastApplied[clientId]; !exists || seqNum > lastSeq {
    kv.lastApplied[clientId] = seqNum
    kv.lastResults[clientId] = reply
}
```

### 第二个发现：客户端 ID 冲突

修复了缓存更新问题后，测试仍然失败。继续分析日志，发现了关键线索：

```
[Server 0 DoOp] Put: clientId=1763449207747363000 seqNum=1 value={"Id":0,...}
[Server 0 DoOp] Put: clientId=1763449207747363000 seqNum=1 value={"Id":1,...}
```

相同的 clientId 和 seqNum，但是 value 中的 Id 不同。这说明有两个不同的客户端使用了相同的 clientId。

追查客户端 ID 的生成代码：

```go
// 问题代码
clientId: time.Now().UnixNano()
```

当多个 Clerk 在同一纳秒内并发创建时，它们会获得相同的时间戳，导致 clientId 冲突。这是问题的根本原因。

### 最终解决方案

使用加密随机数生成器来确保 clientId 的唯一性：

```go
func nrand() int64 {
    var buf [8]byte
    rand.Read(buf[:])
    return int64(binary.LittleEndian.Uint64(buf[:]))
}

func MakeClerk(...) kvtest.IKVClerk {
    ck := &Clerk{
        clientId: nrand(),
        // ...
    }
    return ck
}
```

`crypto/rand` 提供的随机数质量足够高，即使并发创建成千上万个客户端也不会产生冲突。


## 经验总结

### 分布式系统中的 ID 生成

在分布式系统中，生成全局唯一 ID 是一个常见需求。几种常见方案：

1. 时间戳：简单但在高并发下会冲突
2. 时间戳 + 计数器：需要额外的并发控制
3. UUID：字符串形式占用空间大
4. 加密随机数：本实验采用的方案，简单可靠

### 去重机制的复杂性

实现去重比预想的复杂：
- 需要考虑请乱序到达的情况
- 缓存更新需要谨慎，避免被旧数据污染
- 理想情况下应该为每个 seqNum 都缓存结果，但这需要考虑内存管理

本实验采用了只缓存最后一个结果的简化方案，配合"拒绝旧请求"的策略来确保正确性。

### 调试分布式系统的方法

1. 结构化日志：记录关键的状态转换和决策点
2. 可重现性：通过种子控制随机性，便于复现问题
3. 日志分析：使用 grep、awk 等工具分析大量日志
4. 分离测试：先跑单个测试，发现问题后再跑完整套件

### Raft 上构建服务的关键点

1. 线性化读：即使是读操作也需要通过 Raft 达成共识（本实验的要求）
2. 精确一次语义：通过客户端 ID + 序列号实现去重
3. 错误处理：区分"肯定失败"和"可能成功"（ErrVersion vs ErrMaybe）
4. 状态一致性：DoOp 在所有服务器上执行相同的操作序列
