---
title: "从一次 database space exceeded 聊起：彻底搞懂 etcd 在 K8s 中的角色"
date: 2026-04-24
draft: true
tags: ["Kubernetes", "etcd", "Raft", "MVCC", "中文"]
summary: "管理 200+ 集群时 CR 大量累积触发 etcd space quota，所有写操作报 mvcc: database space exceeded。排查过程揭开 etcd 的全部秘密：Raft 共识如何保证一致性、MVCC 如何实现乐观并发控制、watch 如何驱动整个 K8s 事件循环，以及 compaction 和 defrag 为什么要分两步。"
weight: 4
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## 引子：一次让整个集群瘫痪的报错

某天凌晨，监控告警疯狂响起。200+ 托管集群的 Hub 集群上，所有写操作开始报错：

```
etcdserver: mvcc: database space exceeded
```

kubectl apply 失败，新 Pod 调度不了，HPA 扩容停滞。整个集群进入「只读」状态——读操作勉强正常，但任何写入都被拒绝。

排查发现，数百个集群的 `ManagedCluster`、`ManifestWork` 等 CR 大量累积，加上历史 revision 从未清理，etcd 数据库体积膨胀到超过默认的 2GB 空间配额（`--quota-backend-bytes`），触发了自我保护机制。

这次事故让我意识到：**要真正运维好 Kubernetes，必须彻底搞懂 etcd。** 它不是一个「装好就不用管」的组件，而是整个集群的大脑。

---

## 核心定位：etcd 是 K8s 的「大脑」

在 Kubernetes 架构中，etcd 的定位非常清晰：

- **唯一的持久化存储**：集群中所有资源对象（Pod、Service、Deployment、ConfigMap、CRD...）的期望状态和实际状态，全部存储在 etcd 中。
- **唯一的客户端：API Server**。kubelet、Controller Manager、Scheduler 都不会直接访问 etcd，它们全部通过 API Server 间接读写。API Server 是 etcd 的「守门人」。

```
kubelet ──┐
scheduler ─┤──→ API Server ──→ etcd
controller ─┘
```

这意味着：
1. etcd 挂了 = 集群失忆，所有控制面操作停止
2. etcd 慢了 = 整个集群反应迟钝
3. etcd 的一致性保证 = K8s 状态的正确性保证

那么，etcd 是如何保证数据一致性的？答案是 **Raft 共识算法**。

---

## Raft 共识算法：如何让多个节点达成一致

### 三种角色

Raft 协议中，每个 etcd 节点在任意时刻只会处于三种角色之一：

| 角色 | 职责 |
|------|------|
| **Leader** | 接收所有写请求，将日志复制到 Follower，驱动共识流程 |
| **Follower** | 被动接收 Leader 的日志复制和心跳，不主动发起请求 |
| **Candidate** | Follower 超时未收到心跳后转变为 Candidate，发起选举 |

正常运行时，集群中只有 **一个 Leader**，其余全是 Follower。

### 写入流程

一次完整的写操作如何在 Raft 集群中完成？以 3 节点集群为例：

```
               ┌──────────────────────────────────────────────────────────┐
               │                    写入流程时序                          │
               └──────────────────────────────────────────────────────────┘

  Client          Leader            Follower-1          Follower-2
    │                │                   │                   │
    │─── PUT key ───→│                   │                   │
    │                │                   │                   │
    │                │── Append Log ──→  │                   │
    │                │── Append Log ──────────────────────→  │
    │                │                   │                   │
    │                │                   │                   │
    │                │←─ ACK ───────────│                   │
    │                │←─ ACK ─────────────────────────────  │
    │                │                   │                   │
    │                │  [多数确认, commit]  │                   │
    │                │                   │                   │
    │                │── Commit 通知 ──→ │                   │
    │                │── Commit 通知 ──────────────────────→ │
    │                │                   │                   │
    │←── 写入成功 ──│                   │                   │
    │                │                   │                   │
```

关键步骤：

1. **Client 发送写请求到 Leader**（如果发到 Follower，会被转发到 Leader）
2. **Leader 将操作写入本地 WAL 日志**（Write-Ahead Log），并行发送 AppendEntries RPC 给所有 Follower
3. **Follower 写入自己的 WAL 日志**，返回 ACK
4. **Leader 收到多数派确认**（包括自己，3 节点中需要 2 个确认）后，将日志标记为 committed
5. **Leader apply 到状态机**（即 boltdb），返回成功给 Client
6. **Leader 通过后续心跳通知 Follower commit**，Follower 也 apply 到本地状态机

> 为什么是「多数派」？因为只要多数节点存活并达成一致，即使少数节点故障，集群仍然能正常工作。这就是 **Quorum 机制**。

### Leader 选举

当 Follower 在 **election timeout**（通常 1000-1500ms 随机值）内没有收到 Leader 的心跳（默认 100ms 间隔），它会：

1. 将自己的 term（任期号）+1
2. 转变为 Candidate
3. 先投自己一票，然后向所有其他节点发送 RequestVote RPC
4. 如果收到多数票，成为新 Leader
5. 如果收到其他节点的心跳且 term >= 自己，退回 Follower
6. 如果选举超时（没人赢），term+1 重新选举

> **为什么 election timeout 要随机？** 避免所有 Follower 同时超时、同时发起选举导致「分票」僵局。随机化大幅降低了冲突概率。

### 容错能力

| 集群节点数 | 可容忍故障数 | Quorum |
|-----------|------------|--------|
| 3 | 1 | 2 |
| 5 | 2 | 3 |
| 7 | 3 | 4 |

生产环境推荐 **3 或 5 节点**。为什么不用更多？因为节点越多，写入延迟越高（需要等待更多 ACK），且收益递减。

---

## MVCC：乐观并发控制的基石

### 全局递增的 Revision

etcd 使用 **多版本并发控制（MVCC）** 来管理数据。每次写操作（不论修改哪个 key）都会让全局的 **revision** 递增 1：

```
操作                          revision
─────────────────────────────────────
PUT /a = "hello"                1
PUT /b = "world"                2
PUT /a = "hello2"               3
DELETE /b                       4
```

注意：revision 是 **全局唯一、单调递增** 的，不是每个 key 独立计数。每个 key 还有自己的 **version**（从 1 开始，每次修改该 key 时 +1，删除后重置）。

### Kubernetes 的 resourceVersion

**K8s 中每个资源对象的 `resourceVersion` 字段，就是 etcd 的 revision。**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  resourceVersion: "384920"   # ← 这就是 etcd 的 revision
```

这个映射关系看似简单，却是整个 K8s 并发控制和 watch 机制的基础。

### 乐观并发控制（Optimistic Concurrency Control）

多个 Controller 或 API Server 实例同时修改同一个资源时，K8s 如何避免冲突？答案是基于 `resourceVersion` 的乐观锁。

```
  API Server A              etcd               API Server B
      │                       │                      │
      │── GET /pods/nginx ──→│                      │
      │←─ {rv: 100, ...} ───│                      │
      │                       │                      │
      │                       │←── GET /pods/nginx ──│
      │                       │──→ {rv: 100, ...} ──│
      │                       │                      │
      │── PUT /pods/nginx ──→│                      │
      │   (if rv == 100)      │                      │
      │←─ OK {rv: 101} ─────│                      │
      │                       │                      │
      │                       │←── PUT /pods/nginx ──│
      │                       │    (if rv == 100)     │
      │                       │──→ CONFLICT! ────────│
      │                       │    (当前 rv=101≠100)   │
      │                       │                      │
      │                       │←── GET /pods/nginx ──│
      │                       │──→ {rv: 101, ...} ──│
      │                       │                      │
      │                       │←── PUT /pods/nginx ──│
      │                       │    (if rv == 101)     │
      │                       │──→ OK {rv: 102} ────│
```

流程解读：

1. **A 和 B 同时读取** `/pods/nginx`，都拿到 `resourceVersion: 100`
2. **A 先写入成功**，etcd 将 revision 更新为 101
3. **B 尝试写入**，携带条件 `if rv == 100`，但 etcd 发现当前 rv 已经是 101，返回冲突
4. **B 重新读取**最新版本（rv: 101），合并修改后重试写入

这就是 K8s 中经常看到的 `409 Conflict` 错误的根源。

### 为什么选择乐观锁？

| 考量 | 乐观锁 | 悲观锁 |
|------|--------|--------|
| 读写比例 | K8s 读远多于写，乐观锁在无冲突时零开销 | 每次读都要加锁，开销大 |
| 分布式锁管理 | 不需要额外的分布式锁服务 | 需要锁管理器，增加复杂度 |
| 死锁风险 | 无死锁可能 | 需要死锁检测/超时 |
| Controller 模式 | Controller 天然有 reconcile 重试循环，冲突后自动重试 | 锁持有期间其他 Controller 阻塞 |

K8s 的设计哲学是 **level-triggered**（基于期望状态，而非事件驱动），Controller 本身就会不断 reconcile。冲突只是触发一次额外的 reconcile，完美契合。

---

## Watch 机制：驱动 K8s 的事件循环

如果说 etcd 是 K8s 的大脑，那么 **watch** 就是它的神经系统。

### 基于 Revision 的 Watch

etcd 的 watch 基于 revision 实现：客户端告诉 etcd「从 revision N 开始，告诉我这个 key（或前缀）的所有变更」。底层使用 **gRPC 双向流**，服务端持续推送事件。

```go
// 伪代码：从 revision 100 开始 watch 所有 pod
watcher := client.Watch(ctx, "/registry/pods/", clientv3.WithRev(100))
for resp := range watcher {
    for _, event := range resp.Events {
        // event.Type: PUT 或 DELETE
        // event.Kv.ModRevision: 变更时的 revision
    }
}
```

### API Server 的 watchCache

如果每个 kubelet、Controller 都直接 watch etcd，etcd 会被压垮。所以 **API Server 充当了聚合层**：

```
Controller-1 ─┐                              
Controller-2 ─┤── watch ──→ API Server ── 单个 watch ──→ etcd
kubelet-1    ─┤            (watchCache)
kubelet-2    ─┘
```

API Server 内部维护了 **watchCache**：
- 对 etcd 建立 **一个** watch 连接（per 资源类型）
- 将收到的事件缓存在内存环形缓冲区中
- 所有客户端的 watch 请求都从 watchCache 提供服务

**resourceVersion 的映射**：
- 客户端 watch 时携带 `resourceVersion`，API Server 在 watchCache 中查找对应位置
- 如果请求的 resourceVersion 还在缓冲区内，直接从缓存回放
- 如果已经被淘汰（太旧），返回 `410 Gone`，客户端需要重新 LIST

---

## 性能瓶颈与优化

etcd 作为整个集群的核心存储，性能问题会被放大到全局。以下是 5 个常见的性能瓶颈及应对策略：

### 1. 大 Value 问题

etcd 默认限制单个请求大小为 1.5MB。当存储大型 ConfigMap、Secret 或 CRD 时：

- **现象**：读写延迟飙升，etcd 内存占用异常
- **优化**：
  - 避免在 ConfigMap/Secret 中存放大文件
  - 拆分大对象为多个小对象
  - 考虑将大数据放在外部存储（如 S3），etcd 只存引用

### 2. 频繁写入 / 磁盘 I/O

etcd 的写入性能严重依赖磁盘 I/O，特别是 **fsync 延迟**：

- **现象**：`etcd_disk_wal_fsync_duration_seconds` 持续偏高
- **优化**：
  - **必须使用 SSD**，HDD 在生产环境不可接受
  - 使用专用磁盘，不与其他 I/O 密集型应用共享
  - 云环境选择高 IOPS 的存储类型（如 AWS gp3 或 io2）

### 3. Watch 连接过多

大规模集群中，大量 Controller + 大量资源类型 = 海量 watch 连接：

- **现象**：API Server 内存暴涨，etcd gRPC 连接数告警
- **优化**：
  - 确保 API Server 的 watchCache 正常工作（`--watch-cache-sizes`）
  - 避免 Controller 对全量资源做无过滤的 watch
  - 使用 label selector 或 field selector 缩小 watch 范围

### 4. 数据增长 / Compaction / Defrag

这正是我们开头事故的根因：

- **Compaction**（压缩）：删除旧的 revision 历史。etcd 默认保留所有历史版本，K8s 的 `--etcd-compaction-interval`（默认 5 分钟）会定期清理
- **Defrag**（碎片整理）：compaction 只是标记空间可复用，**不会释放磁盘空间**。需要 defrag 来真正回收
- **优化**：
  - 确认 auto-compaction 在正常运行
  - 定期执行 defrag（注意：defrag 会阻塞读写，应逐节点操作）
  - 监控 `etcd_mvcc_db_total_size_in_bytes`

### 5. 大规模节点心跳

当集群节点数达到数千时，Node 的 Lease 续约和状态更新会产生大量写入：

- **现象**：etcd 写入 QPS 随节点数线性增长
- **优化**：
  - 调整 `--node-status-update-frequency`（默认 10s）
  - 使用 Node Lease 机制（K8s 1.14+ 默认启用）减少 Node 对象的更新频率
  - 考虑分片或联邦架构分散压力

---

## 面试追问

### Q1: etcd 如何备份和恢复？

```bash
# 备份
etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/server.crt \
  --key=/etc/etcd/server.key

# 恢复（需要停止 etcd，所有节点都要操作）
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name=etcd-0 \
  --data-dir=/var/lib/etcd-restore \
  --initial-cluster="etcd-0=https://10.0.0.1:2380,etcd-1=https://10.0.0.2:2380,etcd-2=https://10.0.0.3:2380" \
  --initial-advertise-peer-urls=https://10.0.0.1:2380
```

关键点：
- snapshot 是某一时刻的全量快照，包含所有 key-value 数据
- 恢复时会创建新的 cluster ID 和 member ID，避免与旧集群冲突
- 恢复后所有节点需要使用相同的 snapshot 文件

### Q2: 为什么 K8s 选 etcd 而不是 ZooKeeper 或 Consul？

| 维度 | etcd | ZooKeeper | Consul |
|------|------|-----------|--------|
| 共识协议 | Raft（更易理解和实现） | ZAB（复杂） | Raft |
| Watch 机制 | 基于 revision 的持续 watch | 一次性 watch，需反复注册 | 长轮询，效率较低 |
| API | gRPC，高效的二进制协议 | 自定义 TCP 协议 | HTTP |
| 数据模型 | 扁平 key-value | 树形 ZNode | key-value + 服务发现 |
| 功能定位 | 专注一致性 KV 存储 | 偏重协调和锁服务 | 偏重服务发现 |

etcd 的 watch 基于 revision 可以可靠地回放历史变更，这对 K8s 的 Controller 模式至关重要。ZooKeeper 的一次性 watch 在高并发下容易丢事件。

### Q3: etcd revision 和 K8s resourceVersion 是什么关系？

**直接映射**。API Server 将 etcd 的 `ModRevision`（key 最后一次修改时的全局 revision）作为该资源对象的 `resourceVersion` 返回给客户端。

```
etcd key: /registry/pods/default/nginx
etcd ModRevision: 384920

→ K8s Pod nginx 的 metadata.resourceVersion = "384920"
```

LIST 操作返回的 `resourceVersion` 是当前 etcd 的最新 revision，代表这次 LIST 的「快照点」。

### Q4: 一个 etcd 节点挂了，集群读写是否正常？

**3 节点集群挂 1 个：读写都正常。** 因为剩下 2 个节点满足多数派（quorum = 2）。

但有细节：
- 如果挂的是 **Leader**，会触发重新选举，期间（通常 1-2 秒）写操作会短暂中断
- 如果挂的是 **Follower**，对客户端完全透明
- 如果 2 个节点挂了，剩 1 个节点无法形成 quorum，**读写都不可用**

### Q5: revision 和 key version 的区别？

| 属性 | revision | key version |
|------|----------|-------------|
| 作用域 | 全局，所有 key 共享 | 单个 key 独立 |
| 递增规则 | 任何 key 的任何写操作都会 +1 | 该 key 被修改时 +1 |
| 删除后 | 继续递增 | 重置为 0（重新 PUT 后从 1 开始） |
| K8s 映射 | resourceVersion | 无直接映射 |

示例：
```
PUT /a = 1    → revision=1, /a version=1
PUT /b = 2    → revision=2, /b version=1
PUT /a = 3    → revision=3, /a version=2
DELETE /a     → revision=4
PUT /a = 5    → revision=5, /a version=1（重新计数）
```

### Q6: 为什么 K8s 用乐观锁不用悲观锁？

核心原因：

1. **读多写少**：K8s 的读写比例极高（大量 LIST/GET vs 少量 UPDATE），乐观锁在无冲突时零额外开销
2. **无需锁管理器**：悲观锁需要分布式锁服务（如 ZooKeeper），引入额外的复杂度和故障点
3. **无死锁**：乐观锁不持有锁，不存在死锁问题
4. **天然契合 Controller 模式**：Controller 的 reconcile 循环本身就是「读取 → 计算 → 写入 → 冲突则重试」，乐观锁是最自然的选择

### Q7: linearizable read vs serializable read

etcd 支持两种一致性级别的读操作：

| 类型 | 一致性 | 性能 | 实现 |
|------|--------|------|------|
| **Linearizable**（默认） | 强一致，读到最新数据 | 较慢，需要经过 Leader 确认 | 读请求需要 Leader 确认自己仍是 Leader（通过一轮心跳） |
| **Serializable** | 可能读到过期数据 | 快，任何节点本地读取 | 直接从本地状态机读取，不需要共识 |

K8s API Server 默认使用 **Serializable read**（通过 `--etcd-servers` 直连），因为 API Server 有 watchCache 补偿，且对读的实时性要求可以容忍微小延迟。

### Q8: compaction 和 defrag 为什么要分两步？

这是一个常见的困惑，也是开头事故的关键知识点：

**Compaction（压缩历史版本）：**
- 删除指定 revision 之前的所有历史版本
- 操作是 **逻辑删除**，在 boltdb 中将旧数据标记为可覆盖
- 不阻塞正常读写，可以在线执行
- etcd 中由 `--auto-compaction-retention` 控制

**Defrag（碎片整理）：**
- 将 boltdb 文件重新整理，真正释放被标记的空间
- 操作是 **物理删除**，需要重写整个数据库文件
- **会阻塞该节点的读写**，必须逐节点执行
- 需要手动触发：`etcdctl defrag --endpoints=...`

为什么不合并？因为 compaction 需要频繁执行（默认 5 分钟），而 defrag 开销大且会阻塞服务。如果每次 compaction 都做 defrag，etcd 会频繁不可用。分开设计让日常清理（compaction）不影响服务，磁盘回收（defrag）可以在维护窗口执行。

### Q9: 两个 API Server 同时写同一个 key，etcd 怎么处理？

etcd 提供 **mini-transaction**（也叫 Txn），支持 CAS（Compare-And-Swap）语义：

```go
// 伪代码：只有当 mod_revision == 100 时才更新
txnResp, err := client.Txn(ctx).
    If(clientv3.Compare(clientv3.ModRevision(key), "=", 100)).
    Then(clientv3.OpPut(key, newValue)).
    Else(clientv3.OpGet(key)).
    Commit()
```

API Server 的更新操作在底层就是一个 mini-transaction：

1. 比较 `mod_revision` 是否等于客户端传入的 `resourceVersion`
2. 如果相等，执行 PUT（Then 分支）
3. 如果不等，返回当前值（Else 分支），上层转化为 `409 Conflict`

因为 Raft 保证了所有写操作的线性顺序，两个并发 Txn 一定有先后。先到的成功，后到的在 If 比较时发现 revision 已变，走 Else 分支。**不需要加锁，全靠 revision 对比。**

---

## 实战场景

### 场景 1：云环境磁盘延迟引发 Leader 选举风暴

**现象**：etcd 日志中频繁出现 Leader 变更，集群不稳定，API 响应时间飙升。

**根因**：云环境使用了网络附加存储（如 AWS EBS gp2），在 I/O 突发消耗完后，磁盘延迟从 <1ms 突增到 >100ms。etcd Leader 写 WAL 日志时 fsync 超时，无法按时发送心跳，Follower 认为 Leader 已死，发起选举。

**关键指标**：
```bash
# WAL fsync 延迟（应 < 10ms）
etcd_disk_wal_fsync_duration_seconds

# 心跳发送间隔（应稳定在 100ms 左右）
etcd_network_peer_round_trip_time_seconds
```

**解决方案**：
- 切换到高性能存储（gp3 + 预配置 IOPS，或本地 NVMe SSD）
- 确保 etcd 数据目录使用专用磁盘
- 适当调大 `--heartbeat-interval` 和 `--election-timeout`（但会增加故障检测时间）

### 场景 2：数据库膨胀触发 Space Quota — 集群进入只读

这就是文章开头的真实事故。展开讲：

**事件链**：
1. 200+ 集群的 CR（ManagedCluster、ManifestWork 等）大量创建和更新
2. 每次更新生成新的 revision，旧版本占用空间
3. Auto-compaction 在运行，但 **defrag 从未执行过**
4. boltdb 文件持续增长（即使 compaction 标记了可回收空间，物理文件不缩小）
5. 文件大小超过 `--quota-backend-bytes`（默认 2GB），触发 alarm
6. etcd 设置 `NOSPACE` alarm，拒绝所有写操作

**恢复步骤**：
```bash
# 1. 确认 alarm 状态
etcdctl alarm list

# 2. 执行 compaction（压缩到当前 revision）
rev=$(etcdctl endpoint status --write-out="json" | jq '.[0].Status.header.revision')
etcdctl compaction $rev

# 3. 逐节点执行 defrag
etcdctl defrag --endpoints=https://etcd-0:2379
etcdctl defrag --endpoints=https://etcd-1:2379
etcdctl defrag --endpoints=https://etcd-2:2379

# 4. 解除 alarm
etcdctl alarm disarm

# 5. 验证写入恢复
etcdctl put /health ok
```

**预防措施**：
- 监控 `etcd_mvcc_db_total_size_in_bytes` 和 `etcd_mvcc_db_total_size_in_use_in_bytes`
- 当两者差值过大时，说明碎片过多，需要 defrag
- 设置告警：db size > quota 的 80% 时报警
- 定期在维护窗口执行 defrag

### 场景 3：Watch 事件丢失 — 410 Gone

**现象**：Controller 日志中出现 `too old resource version`，watch 断开后 re-LIST 导致 API Server 负载突增。

**根因**：API Server 的 watchCache 使用环形缓冲区存储事件。当事件产生速度超过 Controller 消费速度，旧事件被覆盖。Controller 尝试用过期的 `resourceVersion` 恢复 watch 时，API Server 发现该 revision 已不在缓存中。

**时间线**：
```
Controller watch (rv=1000) ──→ 正常接收事件
                              ··· 大量事件涌入 ···
watchCache 缓冲区: [rv=5000 ... rv=8000]
                              rv=1000 已被淘汰
Controller 断线重连 (rv=4999) ──→ 410 Gone
Controller 被迫 re-LIST ──→ 全量数据加载
```

**解决方案**：
- 增大 watchCache 大小：`--watch-cache-sizes=pods#1000,nodes#500`
- 检查 Controller 的事件处理是否有阻塞（如 reconcile 中的长时间操作）
- 使用 `reflector_watch_duration_seconds` 监控 watch 连接寿命

### 场景 4：大对象 / 海量小对象导致 etcd 读取缓慢

**现象**：`etcdctl get` 某些 key 耗时数秒，LIST 操作超时。

**大对象问题**：
- 某些 CRD status 中嵌入了完整的集群状态，单个对象达到数百 KB
- etcd 的 value 通过 boltdb 存储，大 value 需要跨多个 page 读取
- 影响同一个 boltdb page 上的其他 key 的读取性能

**海量小对象问题**：
- 数万个 Event 对象频繁创建和过期
- 大量 Lease 对象占用存储
- LIST 操作需要序列化所有对象，内存和 CPU 开销大

**解决方案**：
- CRD 设计时控制 status 大小，考虑使用 subresource 或外部存储
- 缩短 Event 的 TTL（`--event-ttl`，默认 1h）
- 使用分页 LIST（`limit` + `continue` token）避免全量加载
- 监控 `etcd_request_duration_seconds` 按操作类型分析瓶颈

---

## 总结

回到那次 `database space exceeded` 事故：根因并不复杂——缺少 defrag 导致物理空间无法回收。但理解这个问题的「为什么」，需要串联 etcd 的整套知识体系：

1. **Raft** 保证了分布式一致性，但对磁盘 I/O 有硬性要求
2. **MVCC** 用 revision 实现了高效的乐观并发控制，但历史版本会持续积累
3. **Watch** 基于 revision 驱动了整个 K8s 的事件循环，但依赖 watchCache 的健康运行
4. **Compaction** 清理历史，**Defrag** 回收空间——两步缺一不可

etcd 不是一个「装好就不用管」的组件。理解它的工作原理，才能在出问题时快速定位，在设计系统时避开陷阱。
