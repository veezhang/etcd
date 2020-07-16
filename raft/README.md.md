# Raft library

Raft is a protocol with which a cluster of nodes can maintain a replicated state machine.
The state machine is kept in sync through the use of a replicated log.
For more details on Raft, see "In Search of an Understandable Consensus Algorithm"
(https://raft.github.io/raft.pdf) by Diego Ongaro and John Ousterhout.

Raft 是分布式集群通过日志复制维护状态一致的协议 state machine replicate。

This Raft library is stable and feature complete. As of 2016, it is **the most widely used** Raft library in production, serving tens of thousands clusters each day. It powers distributed systems such as etcd, Kubernetes, Docker Swarm, Cloud Foundry Diego, CockroachDB, TiDB, Project Calico, Flannel, Hyperledger and more.

Raft 稳定，特征全，广泛的Raft库。

Most Raft implementations have a monolithic design, including storage handling, messaging serialization, and network transport. This library instead follows a minimalistic design philosophy by only implementing the core raft algorithm. This minimalism buys flexibility, determinism, and performance.

大部分 Raft 都是高聚合设计，包括：存储处理，消息序列化，网络传输。相反，该库通过仅实现核心 Raft 算法来遵循简约的设计理念。 这种极简主义具有灵活性，确定性和性能。

To keep the codebase small as well as provide flexibility, the library only implements the Raft algorithm; both network and disk IO are left to the user. Library users must implement their own transportation layer for message passing between Raft peers over the wire. Similarly, users must implement their own storage layer to persist the Raft log and state.

为了使代码库较小并提供灵活性，该库仅实现Raft算法。网络和磁盘IO都留给用户。用户必须实现自己的传输层在 Raft 之间传递消息。 同样，用户必须实现自己的存储层才能保留Raft日志和状态。

In order to easily test the Raft library, its behavior should be deterministic. To achieve this determinism, the library models Raft as a state machine.  The state machine takes a `Message` as input. A message can either be a local timer update or a network message sent from a remote peer. The state machine's output is a 3-tuple `{[]Messages, []LogEntries, NextState}` consisting of an array of `Messages`, `log entries`, and `Raft state changes`. For state machines with the same state, the same state machine input should always generate the same state machine output.

为了轻松测试 Raft 库，其行为应具有确定性。 为了实现这种确定性，库将 Raft 建模为状态机。状态机以`Message`作为输入。消息可以是本地计时器更新，也可以是从远程对端发送的网络消息。状态机的输出是一个三元组的`{[]Messages, []LogEntries, NextState}`，由一系列`Messages`，`log entries`和`Raft state changes`组成。对于状态相同的状态机，相同的状态机输入应始终生成相同的状态机输出。

A simple example application, _raftexample_, is also available to help illustrate how to use this package in practice: https://github.com/etcd-io/etcd/tree/master/contrib/raftexample

还提供了一个简单的示例应用程序 raftexample 来帮助说明如何使用此库。

# Features

This raft implementation is a full feature implementation of Raft protocol. Features includes:

实现了 Raft 协议的全功能实现。

- Leader election
- Log replication
- Log compaction
- Membership changes
- Leadership transfer extension
- Efficient linearizable read-only queries served by both the leader and followers
  - leader checks with quorum and bypasses Raft log before processing read-only queries
  - followers asks leader to get a safe read index before processing read-only queries
- More efficient lease-based linearizable read-only queries served by both the leader and followers
  - leader bypasses Raft log and processing read-only queries locally
  - followers asks leader to get a safe read index before processing read-only queries
  - this approach relies on the clock of the all the machines in raft group

- Leader 选举
- 日志复制
- 日志压缩
- 成员变更
- Leader转移
- leader 和 follower 都可以高效地线性化只读查询
  - leader 使用仲裁进行检查，并在处理只读查询之前绕过Raft日志
  - follower 请求 leader 在处理只读查询之前获得安全的读取索引
- leader 和 follower 都可以使用更有效的基于租约的线性化只读查询
  - leader 绕过 Raft 日志并在本地处理只读查询
  - follower 请求 leader 在处理只读查询之前获得安全的读取索引
  - 这种方法依赖于 Raft 组中所有机器的时钟

This raft implementation also includes a few optional enhancements:

该 raft 实现还包括一些可选的增强功能：

- Optimistic pipelining to reduce log replication latency
- Flow control for log replication
- Batching Raft messages to reduce synchronized network I/O calls
- Batching log entries to reduce disk synchronized I/O
- Writing to leader's disk in parallel
- Internal proposal redirection from followers to leader
- Automatic stepping down when the leader loses quorum
- Protection against unbounded log growth when quorum is lost

- 优化 pipeline 来减少日志复制延迟
- 日志复制的流控制
- 批量处理 Raft 消息以减少同步的网络 I/O 调用
- 批处理日志条目以减少磁盘同步的 I/O
- 并发写入 leader 磁盘
- 内部提案从 follower 重定向到 leader
- 当 leader 失去仲裁时自动下台
- 防止仲裁丢失时日志不受限制的增长

## Notable Users

- [cockroachdb](https://github.com/cockroachdb/cockroach) A Scalable, Survivable, Strongly-Consistent SQL Database
- [dgraph](https://github.com/dgraph-io/dgraph) A Scalable, Distributed, Low Latency, High Throughput Graph Database
- [etcd](https://github.com/etcd-io/etcd) A distributed reliable key-value store
- [tikv](https://github.com/pingcap/tikv) A Distributed transactional key value database powered by Rust and Raft
- [swarmkit](https://github.com/docker/swarmkit) A toolkit for orchestrating distributed systems at any scale.
- [chain core](https://github.com/chain/chain) Software for operating permissioned, multi-asset blockchain networks

## Usage

The primary object in raft is a Node. Either start a Node from scratch using raft.StartNode or start a Node from some initial state using raft.RestartNode.

Raft 中的主要对象是一个节点。使用 raft.StartNode 从头启动 Node 或使用 raft.RestartNode 从某个初始状态启动 Node 。

To start a three-node cluster
```go
  storage := raft.NewMemoryStorage()
  c := &raft.Config{
    ID:              0x01,
    ElectionTick:    10,
    HeartbeatTick:   1,
    Storage:         storage,
    MaxSizePerMsg:   4096,
    MaxInflightMsgs: 256,
  }
  // Set peer list to the other nodes in the cluster.
  // Note that they need to be started separately as well.
  // 启动多节点的时候设置 peer 列表为其他节点
  n := raft.StartNode(c, []raft.Peer{{ID: 0x02}, {ID: 0x03}})
```

Start a single node cluster, like so:
```go
  // Create storage and config as shown above.
  // Set peer list to itself, so this node can become the leader of this single-node cluster.
  // 启动单节点的时候，设置 peer 列表为自身，以便能成为 leader
  peers := []raft.Peer{{ID: 0x01}}
  n := raft.StartNode(c, peers)
```

To allow a new node to join this cluster, do not pass in any peers. First, add the node to the existing cluster by calling `ProposeConfChange` on any existing node inside the cluster. Then, start the node with an empty peer list, like so:

允许新节点加入集群，不要设置 peer 列表。首先在已经存在的集群中任何存在的节点上调用 `ProposeConfChange` 来添加这个节点。然后以空 peer 列表启动这个节点。

```go
  // Create storage and config as shown above.
  n := raft.StartNode(c, nil)
```

To restart a node from previous state:

从之前的状态重启节点

```go
  storage := raft.NewMemoryStorage()

  // Recover the in-memory storage from persistent snapshot, state and entries.
  // 从持久快照，状态和日志 entries 恢复 storage
  storage.ApplySnapshot(snapshot)
  storage.SetHardState(state)
  storage.Append(entries)

  c := &raft.Config{
    ID:              0x01,
    ElectionTick:    10,
    HeartbeatTick:   1,
    Storage:         storage,
    MaxSizePerMsg:   4096,
    MaxInflightMsgs: 256,
  }

  // Restart raft without peer information.
  // Peer information is already included in the storage.
  // 重启 raft 节点，不需要配置 peer 信息，在 storage 已经包含了。
  n := raft.RestartNode(c)
```

After creating a Node, the user has a few responsibilities:

启动节点后，用户有以下职责：

First, read from the Node.Ready() channel and process the updates it contains. These steps may be performed in parallel, except as noted in step 2.

首先，从 Node.Ready() channel 读取并处理其包含的更新。这些步骤可以并行执行，除非步骤 2 中另有说明。

1. Write Entries, HardState and Snapshot to persistent storage in order, i.e. Entries first, then HardState and Snapshot if they are not empty. If persistent storage supports atomic writes then all of them can be written together. Note that when writing an Entry with Index i, any previously-persisted entries with Index >= i must be discarded.
按顺序将 Entry ，HardState 和 Snapshot 写入持久性存储，即，先写入 Entry ，然后其写入 HardState 和 Snapshot 。如果持久性存储支持原子写入，那么所有这些都可以一起写入。 请注意，在编写索引为 i 的 Entry 时，必须丢弃索引 >= i 的所有先前存在的 Entry 。

2. Send all Messages to the nodes named in the To field. It is important that no messages be sent until the latest HardState has been persisted to disk, and all Entries written by any previous Ready batch (Messages may be sent while entries from the same batch are being persisted). To reduce the I/O latency, an optimization can be applied to make leader write to disk in parallel with its followers (as explained at section 10.2.1 in Raft thesis). If any Message has type MsgSnap, call Node.ReportSnapshot() after it has been sent (these messages may be large). Note: Marshalling messages is not thread-safe; it is important to make sure that no new entries are persisted while marshalling. The easiest way to achieve this is to serialise the messages directly inside the main raft loop.
将所有 Messages 发送到 To 字段中命名的节点。重要的是，直到最新的 HardState 持久化到磁盘上、先前 Ready 的批处理的所有 Entry 被写入（正在持久化的批次的消息可能被发送），都不要发送任何消息。 为了减少 I/O 延迟，可以优化让 leader 和 follower 并行地写入磁盘（如 Raft 论文中的10.2.1节所述）。如果任何消息的类型为 MsgSnap ，请在发送消息后调用 Node.ReportSnapshot() （这些消息可能很大）。注意：Marshalling 消息不是线程安全的； 重要的是要确保在 Marshalling 时不持久任何新 Entry 。实现此目的最简单的方法是直接在主 Raft 循环内序列化消息。

3. Apply Snapshot (if any) and CommittedEntries to the state machine. If any committed Entry has Type EntryConfChange, call Node.ApplyConfChange() to apply it to the node. The configuration change may be cancelled at this point by setting the NodeID field to zero before calling ApplyConfChange (but ApplyConfChange must be called one way or the other, and the decision to cancel must be based solely on the state machine and not external information such as the observed health of the node).
应用 Snapshot 和 CommittedEntries 到 state machine 。如果任何已提交的 Entry 的类型为 EntryConfChange ，则调用 Node.ApplyConfChange() 将其应用于节点。 此时可以通过在调用 ApplyConfChange 之前将 NodeID 字段设置为零来取消配置更改（但是 ApplyConfChange 必须以一种或另一种方式调用，并且取消决定必须仅基于状态机而不是外部信息，例如 观察到的节点健康状况）。

4. Call Node.Advance() to signal readiness for the next batch of updates. This may be done at any time after step 1, although all updates must be processed in the order they were returned by Ready.
调用 Node.Advance() 表示已准备好进行下一批更新。尽管必须按照 Ready 返回的顺序处理所有更新，但是可以在步骤1之后的任何时间完成此操作。

Second, all persisted log entries must be made available via an implementation of the Storage interface. The provided MemoryStorage type can be used for this (if repopulating its state upon a restart), or a custom disk-backed implementation can be supplied.

其次，通过 Storage 接口的实现必须能访问使所有持久日志 Entry 。提供的 MemoryStorage 类型可以用于此目的（如果在重新启动时重新填充其状态），或者可以提供定制的磁盘支持的实现。

Third, after receiving a message from another node, pass it to Node.Step:

再次，从另一个节点收到 message 后，将其传递给 Node.Step ：

```go
	func recvRaftRPC(ctx context.Context, m raftpb.Message) {
		n.Step(ctx, m)
	}
```

Finally, call `Node.Tick()` at regular intervals (probably via a `time.Ticker`). Raft has two important timeouts: heartbeat and the election timeout. However, internally to the raft package time is represented by an abstract "tick".

最后，以固定的时间间隔（可能通过 `time.Ticker` ）调用 `Node.Tick()` 。 Raft 有两个重要的超时：心跳和选举超时。但是，在 Raft 包内部，时间用抽象的 tick 表示。

The total state machine handling loop will look something like this:

总的状态机处理循环如下所示：

```go
  for {
    select {
    case <-s.Ticker:
      // Finally
      n.Tick()
    case rd := <-s.Node.Ready():
      // First . 1
      saveToStorage(rd.HardState, rd.Entries, rd.Snapshot)
      // First . 2
      send(rd.Messages)
      // First . 3
      if !raft.IsEmptySnap(rd.Snapshot) {
        processSnapshot(rd.Snapshot)
      }
      for _, entry := range rd.CommittedEntries {
        process(entry)
        if entry.Type == raftpb.EntryConfChange {
          var cc raftpb.ConfChange
          cc.Unmarshal(entry.Data)
          s.Node.ApplyConfChange(cc)
        }
      }
      // First . 4
      s.Node.Advance()
    case <-s.done:
      return
    }
  }
```

To propose changes to the state machine from the node to take application data, serialize it into a byte slice and call:

节点获取的应用数据，提交到 state machine ，将其序列化为 byte slice 并调用：

```go
	n.Propose(ctx, data)
```

If the proposal is committed, data will appear in committed entries with type raftpb.EntryNormal. There is no guarantee that a proposed command will be committed; the command may have to be reproposed after a timeout.

如果提案已提交，则数据将以 raftpb.EntryNormal 类型出现在已提交的 Entry 中。无法保证会执行提案；可能必须在超时后重新提出。

To add or remove node in a cluster, build ConfChange struct 'cc' and call:

要添加或删除集群中的节点，请构建 ConfChange 结构 'cc' 并调用：

```go
	n.ProposeConfChange(ctx, cc)
```

After config change is committed, some committed entry with type raftpb.EntryConfChange will be returned. This must be applied to node through:

配置更改提交后，将返回一些类型为 raftpb.EntryConfChange 的已提交 entry 。这必须通过以下方式应用于节点：

```go
	var cc raftpb.ConfChange
	cc.Unmarshal(data)
	n.ApplyConfChange(cc)
```

Note: An ID represents a unique node in a cluster for all time. A
given ID MUST be used only once even if the old node has been removed.
This means that for example IP addresses make poor node IDs since they
may be reused. Node IDs must be non-zero.

注意：ID 始终代表集群中的唯一节点。 即使删除了旧节点，给定的 ID 也必须仅使用一次。节点 ID 必须为非零。

## Implementation notes

This implementation is up to date with the final Raft thesis (https://github.com/ongardie/dissertation/blob/master/stanford.pdf), although this implementation of the membership change protocol differs somewhat from that described in chapter 4. The key invariant that membership changes happen one node at a time is preserved, but in our implementation the membership change takes effect when its entry is applied, not when it is added to the log (so the entry is committed under the old membership instead of the new). This is equivalent in terms of safety, since the old and new configurations are guaranteed to overlap.

该实现是最新的 Raft 最终论文（https://github.com/ongardie/dissertation/blob/master/stanford.pdf） ，成员变更协议与第4章中描述的有所不同。关键不变的是：成员变更一次发生在一个节点上，但是在我们的实现中，成员变更在应用 Entry 时才生效，而不是在将其添加到日志时生效（因此该 Entey 是在旧成员而不是在新成员下提交的）。 就安全性而言，这是等效的，因为保证了新旧配置的重叠。

To ensure there is no attempt to commit two membership changes at once by matching log positions (which would be unsafe since they should have different quorum requirements), any proposed membership change is simply disallowed while any uncommitted change appears in the leader's log.

为了确保没有尝试通过匹配日志位置来一次提交两个成员变更（这是不安全的，因为它们具有不同的仲裁，这是不安全的），当 leader 有未提交的日志时，任何提议的成员变更都将被禁止。

This approach introduces a problem when removing a member from a two-member cluster: If one of the members dies before the other one receives the commit of the confchange entry, then the member cannot be removed any more since the cluster cannot make progress. For this reason it is highly recommended to use three or more nodes in every cluster.

当从两个成员的集群中删除一个成员时，这种方法会带来一个问题：如果一个成员在另一个成员接收提交的 confchange entry 之前就已经退出，则该成员将无法再删除，因为该集群无法处理。 因此，强烈建议在每个群集中使用三个或更多节点。
