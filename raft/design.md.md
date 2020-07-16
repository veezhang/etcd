## Progress

Progress represents a follower’s progress in the view of the leader. Leader maintains progresses of all followers, and sends `replication message` to the follower based on its progress.

Progress （进度？） 表示 leader 记录 follower 的 progress 。Leader 负责维护所有 follower 的 progress ，并根据其 progress 发送 `replication message` 给 follower 。

`replication message` is a `msgApp` with log entries.

`replication message` 是携带日志 Entry 的 `msgApp` 。

A progress has two attribute: `match` and `next`. `match` is the index of the highest known matched entry. If leader knows nothing about follower’s replication status, `match` is set to zero. `next` is the index of the first entry that will be replicated to the follower. Leader puts entries from `next` to its latest one in next `replication message`.

Progress 具有连个属性，`match` 和 `next` 。`match` 是已知最高匹配的 entry 索引。如果 Leader 对 follower 的复制状态一无所知，`match` 设置为 0 。 `next` 是将被复制到 follower 的第一个 entry 的索引。 Leader 在下一个 `replication message` 中将 entry 从 `next` 放到最新的 entry 下发。

A progress is in one of the three state: `probe`, `replicate`, `snapshot`.

Progress 有三个状态 `probe` (探测), `replicate` (复制), `snapshot` (快照)。

```
                            +--------------------------------------------------------+          
                            |                  send snapshot                         |          
                            |                                                        |          
                  +---------+----------+                                  +----------v---------+
              +--->       probe        |                                  |      snapshot      |
              |   |  max inflight = 1  <----------------------------------+  max inflight = 0  |
              |   +---------+----------+                                  +--------------------+
              |             |            1. snapshot success                                    
              |             |               (next=snapshot.index + 1)                           
              |             |            2. snapshot failure                                    
              |             |               (no change)                                         
              |             |            3. receives msgAppResp(rej=false&&index>lastsnap.index)
              |             |               (match=m.index,next=match+1)                        
receives msgAppResp(rej=true)                                                                   
(next=match+1)|             |                                                                   
              |             |                                                                   
              |             |                                                                   
              |             |   receives msgAppResp(rej=false&&index>match)                     
              |             |   (match=m.index,next=match+1)                                    
              |             |                                                                   
              |             |                                                                   
              |             |                                                                   
              |   +---------v----------+                                                        
              |   |     replicate      |                                                        
              +---+  max inflight = n  |                                                        
                  +--------------------+                                                        
```

When the progress of a follower is in `probe` state, leader sends at most one `replication message` per heartbeat interval. The leader sends `replication message` slowly and probing the actual progress of the follower. A `msgHeartbeatResp` or a `msgAppResp` with reject might trigger the sending of the next `replication message`.

当 follower 的 progress 处于 `probe` 状态时，leader 每心跳间隔最多发送一个 `replication message` 。 leader 缓慢地发送 `replication message` 并探查跟随者的实际 progress 。 带有拒绝的 `msgHeartbeatResp` 或 `msgAppResp` 可能触发下一个 `replication message` 的发送。

When the progress of a follower is in `replicate` state, leader sends `replication message`, then optimistically increases `next` to the latest entry sent. This is an optimized state for fast replicating log entries to the follower.

当 follower 的 progress 处于 `replicate` 状态时，leader 发送 `replication message` ，然后乐观地将 `next` 增加到已发送的最新 entry 。 这是优化点：将日志 entry 快速复制到 follower 。

When the progress of a follower is in `snapshot` state, leader stops sending any `replication message`.

当 follower 的 progress 处于 `snapshot` 状态时，leader 停止发送 `replication message` 。

A newly elected leader sets the progresses of all the followers to `probe` state with `match` = 0 and `next` = last index. The leader slowly (at most once per heartbeat) sends `replication message` to the follower and probes its progress.

新当选的 leader 将所有 follower 的 progress 设置为 `probe` 状态，其中 `match` = 0 ，`next` =最后索引。 leader （每个心跳最多一次）缓慢地向 follower 发送 `replication message` 并探测其 progress 。

A progress changes to `replicate` when the follower replies with a non-rejection `msgAppResp`, which implies that it has matched the index sent. At this point, leader starts to stream log entries to the follower fast. The progress will fall back to `probe` when the follower replies a rejection `msgAppResp` or the link layer reports the follower is unreachable. We aggressively reset `next` to `match`+1 since if we receive any `msgAppResp` soon, both `match` and `next` will increase directly to the `index` in `msgAppResp`. (We might end up with sending some duplicate entries when aggressively reset `next` too low.  see open question)

当 follower 回复不拒绝的 `msgAppResp` 时， progress 将变为 `replicate` ，这意味着它与已发送的索引匹配。此时，leader 开始快速将日志 entry 流式传输到 follower 。当 follower 回复拒绝 `msgAppResp` 或网络不通时，progress 将回落到 `probe` 。 我们会主动将设置 `next` = `match`+1 ，因为如果我们很快收到任何 `msgAppResp` ，则 `match` 和 `next` 都将直接增加到 `msgAppResp` 中的 `index` 。 （当将 `next` 重置得太低时，我们可能最终会发送一些重复的entry 。）

A progress changes from `probe` to `snapshot` when the follower falls very far behind and requires a snapshot. After sending `msgSnap`, the leader waits until the success, failure or abortion of the previous snapshot sent. The progress will go back to `probe` after the sending result is applied.

当 follower 落后很远并且需要快照时， progress 将从 `probe` 更改为 `snapshot` 。 发送完 `msgSnap` 后， leader 将等待，直到成功发送，失败或中止上次发送的快照。 应用发送结果后，progress 将返回到 `probe` 。

### Flow Control 流控制

1. limit the max size of message sent per message. Max should be configurable.
Lower the cost at probing state as we limit the size per message; lower the penalty when aggressively decreased to a too low `next`
限制每条消息发送的最大消息大小。 Max 应该是可配置的。
降低探测状态的成本，因为我们限制了每条消息的大小； 当积极降低到太低的 `next` 时，降低惩罚。

2. limit the # of in flight messages < N when in `replicate` state. N should be configurable. Most implementation will have a sending buffer on top of its actual network transport layer (not blocking raft node). We want to make sure raft does not overflow that buffer, which can cause message dropping and triggering a bunch of unnecessary resending repeatedly.
处于 `replicate` 状态时，限制发送中消息的数量 < N 。 N 应该是可配置的。 大多数实现在实际网络传输层上面有一个缓冲区（不阻塞 raft 节点）。我们要确保 Raft 不溢出该缓冲区，这可能导致消息丢失并触发大量不必要的重发。
