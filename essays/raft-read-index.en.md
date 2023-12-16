## Standard Raft Implementation of ReadIndex Process

The standard process for handling ReadIndex in Raft is as follows:

- **Step 1**: The leader checks if its current term log has been committed. If not, it abandons the read, returns, and waits for the log to be committed in the current term.
- **Step 2**: The leader takes the current CommitIndex as the ReadIndex.
- **Step 3**: The leader sends a heartbeat to the quorum to confirm itself as the only leader.
- **Step 4**: It waits for the AppliedIndex to reach or exceed the ReadIndex before performing a read operation on the StateMachine.

Through the above steps, this process ensures that:
**Any data that has been read before the time of a read operation (wall clock time) can also be read in this read operation**, which guarantees linearizable read.

Let's take a look at how linearizable read is ensured:

## Simple Proof of Linearizable Read

When the current node (leader) receives a read request `read_1`, assuming the wall clock time is `time_1` and the leader's term is `term_1`;

Suppose there is another read request `read_0` that occurred at a previous time, i.e., `time_0`: `time_0 < time_1`,
Then the read process must ensure that `read_1` can read all the states that `read_0` has read to guarantee linearizable read.

Let's assume that the StateMachine read by `read_0` includes a series of log states from `(0, 0)` to `(term_0, index_0)`, which means `(term_0, index_0)` is the last log seen by `read_0`.
In this case, there are three possibilities for `term_0`:

- **case-gt**: `term_0 > term_1`: To avoid this unprocessable situation, the leader sends a heartbeat request to a quorum at time `t` to confirm that there is no higher term in the time range `(0, t)`; obviously, `t` is after receiving the read request `read_1`, i.e., `t >= time_1`, thus ensuring that at the moment `time_1` when the read request `read_1` is received, no other reader reads a log with a higher term (**Step 3**).
- **case-eq**: `term_0 == term_1`: In this case, the read operation `read_0` must be performed by the current node; and we also know that according to the Raft protocol, only committed logs can be read, so the data read by `read_0` must be before the current CommitIndex, i.e., `index_0 <= CommitIndex`; to ensure linearizable read in this case, which means `read_1` sees all the states that `read_0` sees, it requires the StateMachine at the time of `read_1` to include at least the log up to the CommitIndex.
- **case-lt**: `term_0 < term_1`: In this case, because Raft guarantees that the current leader contains all the logs that have been committed when it is established, `index_0 < NoopIndex`, where `NoopIndex` is the index of the noop log written when the leader is established; to ensure linearizable read in this case, it requires the StateMachine at the time of `read_1` to include at least the log up to `NoopIndex`.

Based on the above analysis, **case-gt** is excluded, and when **case-lt** is satisfied, i.e., after `NoopIndex` is committed (**Step 1**), only **case-eq** needs to be considered (**Step 2**), which means waiting for the StateMachine to apply up to at least the CommitIndex before reading (**Step 4**), which ensures that `read_1` will definitely see the state that `read_0` sees, i.e., linearizable read.