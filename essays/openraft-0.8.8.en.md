## [FAQ](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#faq)

#### [What are the differences between Openraft and standard Raft?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-are-the-differences-between-openraft-and-standard-raft)

- Optionally, In one term there could be more than one leaders to be established, in order to reduce election conflict. See: std mode and adv mode leader id: [`leader_id`](https://docs.rs/openraft/latest/openraft/docs/data/leader_id/index.html);
- Openraft stores committed log id: See: [`RaftLogStorage::save_committed()`](https://docs.rs/openraft/latest/openraft/docs/faq/`crate::storage::RaftLogStorage::save_committed`);
- Openraft optimized `ReadIndex`: no `blank log` check: [`Linearizable Read`](https://docs.rs/openraft/latest/openraft/docs/faq/`crate::docs::protocol::read`).
- A restarted Leader will stay in Leader state if possible;
- Does not support single step memebership change. Only joint is supported.

#### [Why is log id a tuple of `(term, node_id, log_index)`?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#why-is-log-id-a-tuple-of-term-node_id-log_index)

In standard Raft log id is `(term, log_index)`, in Openraft he log id `(term, node_id, log_index)` is used to minimize the chance of election conflicts. This way in every term there could be more than one leaders elected, and the last one is valid. See: [`leader-id`](https://docs.rs/openraft/latest/openraft/docs/data/leader_id/index.html) for details.

#### [How to initialize a cluster?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#how-to-initialize-a-cluster)

There are two ways to initialize a raft cluster, assuming there are three nodes, `n1, n2, n3`:

1. Single-step method: Call `Raft::initialize()` on any one of the nodes with the configuration of all 3 nodes, e.g. `n2.initialize(btreeset! {1,2,3})`.
2. Incremental method: First, call `Raft::initialize()` on `n1` with configuraion containing `n1` itself, e.g., `n0.initialize(btreeset! {1})`. Subsequently use `Raft::change_membership()` on `n1` to add `n2` and `n3` into the cluster.

Employing the second method provides the flexibility to start with a single-node cluster for testing purposes and subsequently expand it to a three-node cluster for deployment in a production environment.

#### [Are there any issues with running a single node service?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#are-there-any-issues-with-running-a-single-node-service)

Not at all.

Running a cluster with just one node is a standard approach for testing or as an initial step in setting up a cluster.

A single node functions exactly the same as cluster mode. It will consistently maintain the `Leader` status and never transition to `Candidate` or `Follower` states.

#### [How to remove node-2 safely from a cluster `{1, 2, 3}`?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#how-to-remove-node-2-safely-from-a-cluster-1-2-3)

Call `Raft::change_membership(btreeset!{1, 3})` to exclude node-2 from the cluster. Then wipe out node-2 data. **NEVER** modify/erase the data of any node that is still in a raft cluster, unless you know what you are doing.

#### [What actions are required when a node restarts?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-actions-are-required-when-a-node-restarts)

None. No calls, e.g., to either [`add_learner()`](https://docs.rs/openraft/latest/openraft/raft/struct.Raft.html#method.add_learner) or [`change_membership()`](https://docs.rs/openraft/latest/openraft/raft/struct.Raft.html#method.change_membership) are necessary.

Openraft maintains the membership configuration in [`Membership`](https://docs.rs/openraft/latest/openraft/struct.Membership.html) for for all nodes in the cluster, including voters and non-voters (learners). When a `follower` or `learner` restarts, the leader will automatically re-establish replication.

#### [What will happen when data gets lost?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#what-will-happen-when-data-gets-lost)

Raft operates on the presumption that the storage medium (i.e., the disk) is secure and reliable.

If this presumption is violated, e.g., the raft logs are lost or the snapshot is damaged, no predictable outcome can be assured. In other words, the resulting behavior is **undefined**.

#### [Is Openraft resilient to incorrectly configured clusters?](https://docs.rs/openraft/latest/openraft/docs/faq/index.html#is-openraft-resilient-to-incorrectly-configured-clusters)

No, Openraft, like standard raft, cannot identify errors in cluster configuration.

A common error is the assigning a wrong network addresses to a node. In such a scenario, if this node becomes the leader, it will attempt to replicate logs to itself. This will cause Openraft to panic because replication messages can only be received by a follower.

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
