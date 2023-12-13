## [常见问题解答](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#faq)

#### [Openraft 和标准 Raft 有什么区别？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-are-the-differences-between-openraft-and-standard-raft)

- 可选地，一个任期内可以建立多个领导者，以减少选举冲突。请参阅：std 模式和 adv 模式领导者 ID：[`leader_id`](https://docs.rs/openraft/latest/openraft/docs/data/leader_id/index.html)；
- Openraft 存储已提交的日志 ID：请参阅：[`RaftLogStorage::save_committed()`](https://docs.rs/openraft/latest/openraft/docs/faq/`crate::storage::RaftLogStorage::save_committed`)；
- Openraft 优化了 `ReadIndex`：无需进行 `blank log` 检查：[`Linearizable Read`](https://docs.rs/openraft/latest/openraft/docs/faq/`crate::docs::protocol::read`)；
- 重新启动的领导者将尽可能保持领导者状态；
- 不支持单步成员变更，只支持联合变更。

#### [为什么日志 ID 是 `(term, node_id, log_index)` 的元组？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#why-is-log-id-a-tuple-of-term-node_id-log_index)

在标准 Raft 中，日志 ID 是 `(term, log_index)`，而在 Openraft 中，日志 ID 是 `(term, node_id, log_index)`，以最小化选举冲突的机会。这样，在每个任期内可以选举出多个领导者，最后一个领导者是有效的。有关详细信息，请参阅：[`leader-id`](https://docs.rs/openraft/latest/openraft/docs/data/leader_id/index.html)。

#### [如何初始化一个集群？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#how-to-initialize-a-cluster)

有两种方法可以初始化 Raft 集群，假设有三个节点，`n1, n2, n3`：

1. 单步方法：在任意一个节点上调用 `Raft::initialize()`，并使用包含所有 3 个节点配置的参数，例如 `n2.initialize(btreeset! {1,2,3})`。
2. 增量方法：首先，在 `n1` 上调用 `Raft::initialize()`，配置中包含 `n1` 本身，例如 `n0.initialize(btreeset! {1})`。然后，使用 `Raft::change_membership()` 在 `n1` 上添加 `n2` 和 `n3` 到集群中。

采用第二种方法可以灵活地从单节点集群开始进行测试，并随后将其扩展为三节点集群，以用于生产环境部署。

#### [运行单节点服务有什么问题吗？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#are-there-any-issues-with-running-a-single-node-service)

完全没有问题。

只运行一个节点的集群是测试或设置集群的初始步骤的标准方法。

单个节点的功能与集群模式完全相同。它将始终保持 `Leader` 状态，不会转换为 `Candidate` 或 `Follower` 状态。

#### [如何安全地从集群 `{1, 2, 3}` 中移除节点 2？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#how-to-remove-node-2-safely-from-a-cluster-1-2-3)

调用 `Raft::change_membership(btreeset!{1, 3})` 以将节点 2 从集群中排除。然后清除节点 2 的数据。**绝对不要**修改/删除任何仍在 Raft 集群中的节点的数据，除非你知道自己在做什么。

#### [节点重新启动时需要执行哪些操作？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-actions-are-required-when-a-node-restarts)

不需要任何操作。不需要调用 [`add_learner()`](https://docs.rs/openraft/latest/openraft/raft/struct.Raft.html#method.add_learner) 或 [`change_membership()`](https://docs.rs/openraft/latest/openraft/raft/struct.Raft.html#method.change_membership)。

Openraft 在 [`Membership`](https://docs.rs/openraft/latest/openraft/struct.Membership.html) 中维护了集群中所有节点（包括投票者和非投票者）的成员配置。当 `follower` 或 `learner` 重新启动时，领导者将自动重新建立复制。

#### [数据丢失时会发生什么？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-will-happen-when-data-gets-lost)

Raft 假设存储介质（即磁盘）是安全可靠的。

如果这个假设被违反，例如 Raft 日志丢失或快照损坏，就无法保证可预测的结果。换句话说，结果行为是**未定义的**。

#### [Openraft 是否对配置错误的集群具有弹性？](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#is-openraft-resilient-to-incorrectly-configured-clusters)

不，Openraft 和标准 Raft 一样，无法识别集群配置错误。

常见的错误是将错误的网络地址分配给节点。在这种情况下，如果该节点成为领导者，它将尝试将日志复制到自己。这将导致 Openraft 崩溃，因为复制消息只能由追随者接收。

```text
thread 'main' panicked at openraft/src/engine/engine_impl.rs:793:9:
assertion failed: self.internal_server_state.is_following()
```

[ⓘ](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#)

```rust
// openraft/src/engine/engine_impl.rs:793
pub(crate) fn following_handler(&mut self) -> FollowingHandler<C> {
    debug_assert!(self.internal_server_state.is_following());
    // ...
}
```