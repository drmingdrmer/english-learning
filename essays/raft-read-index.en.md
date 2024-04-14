## Standard Implementation Process for Raft Read Index

The standard process for handling read indexes in Raft is as follows:

- **Step 1**: The leader checks if the logs for the current term have been committed. If they haven't been committed, it aborts the read operation, returns, and waits for the logs to be committed for the current term.
- **Step 2**: The leader sets the current CommitIndex as the read index.
- **Step 3**: The leader sends heartbeats to a quorum to confirm its leadership.
- **Step 4**: Before executing the read operation on the state machine, it waits for the AppliedIndex to reach or exceed the read index.

Through these steps, this process ensures that any data that has been read before the time of the read operation can also be read in this read operation, thus ensuring linearizability.

Let's take a look at how linearizability is ensured:

## Simple Proof of Linearizability

When the current node (leader) receives a read request `read_1`, let's assume the current wall clock time is `time_1`, and the leader's term is `term_1`.

Let's assume there was another read request `read_0` that occurred earlier, at time `time_0`: `time_0 < time_1`.
In order to ensure linearizability, the read process must ensure that `read_1` can read all the states that `read_0` read.

Let's assume that the state machine read by `read_0` includes a series of log states from `(0, 0)` to `(term_0, index_0)`, where `(term_0, index_0)` is the last log seen by `read_0`. In this case, for `term_0`, there are three possibilities:

- **case-gt**: `term_0 > term_1`: To avoid this unhandled case, the leader sends heartbeat requests to a quorum at time `t` to confirm that there are no higher terms within the time range `(0, t)`; obviously, `t` is after the receipt of the read request `read_1`, i.e., `t >= time_1`, ensuring that at the moment `time_1` when the read request `read_1` is received, no other reader reads logs with higher terms (**Step 3**).
- **case-eq**: `term_0 == term_1`: In this case, the read operation `read_0` must be executed by the current node; we also know that according to the Raft protocol, only committed logs can be read, so the data read by `read_0` must be before the current CommitIndex, i.e., `index_0 <= CommitIndex`; to ensure linearizability in this case, i.e., to ensure that `read_1` sees all the states that `read_0` sees, the state machine needs to include logs up to at least the CommitIndex at the moment of `read_1`.
- **case-lt**: `term_0 < term_1`: In this case, since Raft guarantees that the current leader includes all committed logs upon its establishment, `index_0 < NoopIndex`, where `NoopIndex` is the index of the empty operation log written by the leader upon its establishment; to ensure linearizability in this case, the state machine needs to include logs up to at least `NoopIndex` at the moment of `read_1`.

Based on the analysis above, excluding **case-gt**, when **case-lt** is satisfied, i.e., after `NoopIndex` is committed (**Step 1**), only **case-eq** needs to be considered (**Step 2**), which is waiting for the state machine to apply logs up to at least the CommitIndex before reading (**Step 4**). This ensures that `read_1` will definitely see the states that `read_0` saw, i.e., linearizability.