## 标准 Raft 实现的 ReadIndex 流程

在 Raft 中，ReadIndex 的标准处理步骤如下：

- **步骤-1**：Leader 检查自己当前 Term 的日志是否已提交。如果没有提交，则放弃读取并返回，等待当前 Term 的日志提交。
- **步骤-2**：Leader 将当前的 CommitIndex 作为 ReadIndex。
- **步骤-3**：向 quorum 发送心跳消息，确认自己是唯一的 Leader。
- **步骤-4**：等待 AppliedIndex 达到或超过 ReadIndex，然后对 StateMachine 进行读取操作。

通过以上步骤，这个流程保证了一个 read 操作发生的时间（墙上时钟时间）之前已经被 read 的数据也一定能在这次 read 中被读到，也就是 Linearizable read。

我们来看看线性一致性读取是如何被保证的：

## 线性一致性读的简单证明

当当前节点（Leader）接收到一个 read 请求 `read_1` 时，假设墙上时钟时间是 `time_1`，Leader 的 Term 是 `term_1`。

假设有另一个 read 请求 `read_0`，发生在之前的某个时间，即 `time_0`：`time_0 < time_1`，
那么读流程要能保证 `read_1` 一定能读到所有 `read_0` 读到的状态，以保证线性一致性。

再假定 `read_0` 读的 StateMachine，包含了从 `(0, 0)` 到 `(term_0, index_0)` 这一系列日志的状态，
也就是说 `(term_0, index_0)` 这条日志是 `read_0` 看到的最后一条日志。那么其中的 `term_0` 就有3种情况：

- **case-gt**：`term_0 > term_1`：为避免这种不可处理的情况发生，Leader 在时间 `t` 向一个 quorum 发送 heartbeat 请求，以确认在时间范围 `(0, t)` 内都没有更高的 Term；而显然 `t` 在收到读请求 `read_1` 之后，即 `t >= time_1`，从而保证了：在收到读请求 `read_1` 的时刻 `time_1`，没有其他读取者读到更大 Term 的日志（**步骤-3**）。
- **case-eq**：`term_0 == term_1`：对于这种情况，读操作 `read_0` 一定是在当前节点执行的读操作；而我们又知道由于 Raft 协议规定只有已经提交的日志才能被读取，所以 `read_0` 读到的数据一定是当前 CommitIndex 之前的，即 `index_0 <= CommitIndex`；这种情况下要保证 linearizable read，也就是 `read_1` 看到所有 `read_0` 看到的状态，就要求 `read_1` 读时的 StateMachine 至少要包含到 CommitIndex 这个位置的日志。
- **case-lt**：`term_0 < term_1`：对于此情况，因为 Raft 保证了当前 Leader 建立时一定包含所有已经提交的日志，所以 `index_0 < NoopIndex`，这里 `NoopIndex` 是 Leader 建立时写入的 noop 日志的索引；在这种情况下要保证 linearizable read，就要求 `read_1` 读时的 StateMachine 至少要包含到 `NoopIndex` 的日志。

根据以上分析，**case-gt** 被排除，而当 **case-lt** 满足后，也就是 `NoopIndex` 提交后（**步骤-1**），就只需考虑 **case-eq** 了（**步骤-2**），也就是等 StateMachine apply 到至少 CommitIndex 再读（**步骤-4**），就可以保证 `read_1` 一定看到 `read_0` 看到的状态，即 Linearizable read。