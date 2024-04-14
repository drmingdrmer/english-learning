# Raft (Not) Almighty: How to Make It More Robust

[Original Article](https://dev.to/tarantool/raft-notalmighty-how-to-make-it-more-robust-3a11)

[![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/mt0wckqsxaqv041ulfw3-20240131104950569.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--sRlhwn_T--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mt0wckqsxaqv041ulfw3.png)

Everyone loves Raft. It is widely believed that using the Raft algorithm in a distributed system means that the system will work perfectly. Specifically:

1. As long as the majority of nodes in the cluster are active and connected to each other, the cluster will be writable (with a pause during leader election).
2. If the leader is functioning properly and connected to the majority of nodes, the cluster will always be writable.
3. If the leader crashes, a new leader will be "quickly" elected - whatever that means.

In reality, simply following the Raft specification ([raft.pdf](https://raft.github.io/raft.pdf) or Diego Ongaro's doctoral dissertation version) is not enough to achieve all of the above. Even etcd has suffered from this, leading to the [2020 Cloudflare outage](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/).

My name is Sergey Petrenko, and I have been working on replication in Tarantool for 4 years. Today, I want to tell you about the weaknesses of the Raft algorithm and how to overcome them. This article is a free paraphrase of my presentation with Boris Stepanenko at [Hydra 2022](https://hydraconf.com/talks/f7082c470b8f4ddd894a940f892f40fb/). If you are not familiar with Raft, I recommend reading [my article](https://dzone.com/articles/raft-in-tarantool-how-it-works-and-how-to-use-it) first.

Let's start from a distance. Suppose you have a system that uses Raft and you want to use it in a production environment. We have all heard, more than once, that Raft guarantees that there will be only one leader in each term and that it can maintain performance even with less than half of the nodes in the cluster failing. In other words, if the leader is alive, it will be able to process entries, and if there is no leader, a new leader will be elected. Besides these guarantees, it seems that we don't need anything else for the system to run smoothly in the long term. Is this really the case? We are about to find out.

First, let's take a closer look at the claim that the cluster can keep running as long as the majority of servers are active and connected to each other. Misunderstanding this claim ultimately led to the Cloudflare outage.

## Pre-Vote

Let's start with an example to see how a Raft cluster behaves when there is partial loss of connectivity.

Suppose our cluster consists of three nodes: A, B, and C. A is the leader, and B and C are its replicas; it is now the 2022nd term. What happens if the connection between A and C is lost? Within the election timeout, if the leader does not send a heartbeat signal, server C will consider the leader lost and start an election in the 2023rd term.

Server B is the only server to which server C can send a RequestVote request.

Once server B receives a request with a larger term, it also increments its own term. However, it may not vote for C because if the leader wrote some data after the connection was lost between A and C, B may have more up-to-date data.

Upon receiving the next message from the leader (still server A), server B reports that the term has increased, forcing server A to resign. The cluster is left without a leader, so an election begins. The cluster is not writable until a leader is elected.

If there are no new entries after the self-connection is lost, once server B receives a RequestVote from server C, it will vote for server C.

If this happens, we get stuck in an infinite loop:

1. Server C cannot see the leader.
2. When the election timeout expires, server C starts a new election.
3. The increase in term is conveyed to server A through server B.
4. Server B votes for server C, so
5. Server A resigns.
6. A and C swap places and repeat from step 1 in the new term.

Clients connected to a cluster with a flickering leader may not be able to write anything at all. Once an active leader is elected and the client tries to perform a write operation, the leader resigns without even knowing where to forward the write request, as it cannot directly see the new leader. This situation continues until, miraculously, at least one entry can be written or until the connection between A and C is restored. This is exactly what happened with Cloudflare and etcd. External systems reasonably assume that if they cannot write anything to the etcd cluster, it means that most of the nodes have failed and emergency measures should be taken. Just 6 minutes of etcd being unwritable led to over 6 hours of downtime.

You might think that some of Raft's guarantees are being violated here. In reality, they are not.

At any given moment, the leader is connected to the majority of nodes (including itself), and the leader itself does not fail. However, our expectations are shattered. The cluster cannot function properly.

But the problem is that Raft never promised to work properly in this situation. Its promise is "if the majority of nodes are normal and connected to each other, a leader will be chosen." It never mentioned that the leader is permanent. And the guarantee of choosing a leader is not actually being violated. In the example above, it happens approximately once per election timeout. In practice, Raft's guarantees are not sufficient.

But that's only half of the trouble.

Let's go back to the 2022nd term and assume that in the cluster of nodes A, B, and C, node C loses all communication with the other nodes. Server A is the leader, still connected to server B, and able to handle write requests. So far, so good.

Server C, just like in the previous example, starts a new election every election timeout. Of course, server C cannot win any election. It simply cannot get a majority of votes. However, its term keeps increasing indefinitely, and at the moment of connection recovery, server A sees a larger term again and resigns. The cluster becomes read-only again, at least for one round of election, but without any apparent reason.

## Raft's Solution

This problem of "disruptive" servers is actually mentioned by Raft's author, Diego Ongaro, in his doctoral dissertation. However, he associates them with configuration changes.

The solution he proposes is a new phase of the election called Pre-Vote. The idea is that each server sends a simulated vote request, called PreVote, to everyone before the actual election. This request contains the same fields as a regular RequestVote, and the voters respond to it according to the same rules as RequestVote. The only difference is that voters do not increment their term upon receiving this request and will deny or simply ignore it if they still see a leader in the current term.

Only when a candidate receives confirmation from a majority of voters that they are ready to vote for it, the candidate will increment its term and start the actual election.

Let's check if PreVote can really save the Cloudflare incident. Consider the same situation where C loses connection with the leader but remains connected to the other replicas. Although C can send PreVote requests to B, B will deny the request because it still sees the leader. C will not be able to get a majority of votes - its PreVote request will only receive 2 denials.

In the case of complete loss of connection and subsequent recovery, PreVote also helps. Indeed, server C will not receive any responses to its PreVote requests, so it will not start an election.

The only downside of this solution for us is that it violates backward compatibility. Raft has been running in Tarantool for some time, and there are no PreVote requests in the protocol. It could certainly be added, but old servers do not know how to handle the new requests. We have to introduce new logic on the sender side: if the server is old, we will not send PreVote requests to it and assume that it will definitely respond. We don't like this redundancy and want to get rid of the extra code that exists to support old versions. A better solution is to extend one of the existing requests. Old servers will simply ignore the new fields in the existing requests, so no additional logic is needed. That's the way we chose.

## Our Solution

"In Tarantool, servers are smart enough not to start an election when it is unnecessary," we thought, and we did Pre-Vote in our own way. We let all replicas tell each other whether they see the current leader. After collecting information about who sees the leader and who doesn't, you can decide whether to start an election. If anyone sees the leader or if there is no connection to the majority of nodes, an election will not be initiated.

The advantage of our solution is its backward compatibility. We don't need to introduce new requests and send them to old servers in a special way, and it works fine even if there are different versions of Tarantool servers in the cluster.

The downside is that we don't compare the leader's log with the logs of the voters, which means that we ignore updated data in the voters' logs when starting an election. This may not be a bad thing: as long as someone sees the leader, no election will occur. When no one sees the leader anymore, the election will start anyway. It doesn't matter if the election ends in one or more rounds, thanks to our another modification, Split-Vote detection. Now, let's talk about it.

## Split-Vote Detection

This time, I don't have a scary case like Cloudflare or other famous companies, but I hope it's still interesting.

You may know that not every round of election results in a leader. For example, if several nodes in the cluster notice the leader's absence at the same time, they will independently start an election before receiving each other's RequestVote requests. The votes of the remaining replicas may be scattered among several candidates, so that no candidate can win a majority of votes. We call this situation Split-Vote.

Raft handles this problem by randomizing the election timeout. In each server and each round, a new random value slightly different from the configured value is chosen. This increases the chance that only one candidate has time to start the next round of election, while other candidates receive its RequestVote request before their own election timeout expires. If Raft did not implement this, the election might never end: all servers would synchronously restart and vote for themselves. But there is still room for optimization: each additional round of election is an election timeout of a read-only cluster.

## Our Solution

In practice, rounds that end in a tie are a waste of time. If a leader has already been elected, it may have been elected at the beginning of the round (approximately within the time it takes for packets to be exchanged with the farthest server). According to Raft's specification, the election timeout should be much longer than this.

Also, unlike the classic Raft implementation, in Tarantool, servers not only send vote information to the candidate they voted for but also to all cluster members. This means that each server can keep track of the number of votes each candidate receives. When it sees that no candidate can win in the current round, the server speeds up the start of a new round. The time from detecting a tie to starting a new round averages 0.05 * the election timeout. This is because we choose a random delay within the range of election timeout / 10 to restart the election. We do this for the same purpose as Raft: to avoid new ties. Therefore, in the best case, we save about 0.9 * the election timeout on each round of tie. This improvement is very noticeable: in the original Raft, electing a leader in two rounds of election would take about the election timeout + a fraction of the election timeout. However, in our implementation, two rounds of such elections only take a small fraction of the election timeout. This is faster than the time it takes for the original Raft to realize that a tie has occurred.

In the worst case, when two ties occur in a row, we handle it faster. For us, each round of tie actually has no cost. Additionally, we do not increase the probability of a second tie in any way: in this case, the random delay to restart the election is generated within the same range as in regular cases.

## CheckQuorum

Raft ensures that there are never two leaders within the same term. However, it does not address the possibility of having two leaders at the same time, across different terms. This is a situation we want to avoid because clients connected to the old leader may not be aware of the existence of a new leader. Until the old leader actually steps down, there is no reason for them to look for a new leader. The old leader remains writable, and any synchronous transactions will be rolled back if they fail to receive confirmation from a majority of replicas.

For us, the scenario of having two leaders at the same time is critical because Tarantool supports both synchronous and asynchronous replication. Each leader is writable, and if you write asynchronous transactions on the old leader, clients may ignore the existence of the new leader. This can lead to some clients contacting the new leader while others contact the old leader, resulting in a split brain situation. To avoid this, the old leader must step down and become read-only in a situation where it could potentially be replaced.

The possibility of having two leaders only arises when the old leader loses connection to a majority of servers in the cluster. To win an election, a candidate must receive a majority of votes. The reason the old leader is unaware of the election is because it is not connected to a majority of nodes. If it is connected to at least one server that votes for the new leader, it will immediately step down. It is impossible for the old leader to be connected to a majority of nodes that did not vote, while the new leader is connected to a majority of voting nodes.

Therefore, we want to ensure that the leader steps down immediately when it loses connection to a majority of nodes in the cluster. If the current leader's connection is lost, it is highly likely that a majority has already chosen a new leader.

Another reason for CheckQuorum is that without it, relying solely on PreVote can result in the cluster being locked: the current leader cannot write anything, and replicas cannot start an election. Let's consider an example: suppose we have a cluster consisting of five servers, with D as the leader. Then a series of unusual events occur: first, server E crashes, then the connection between A and D is interrupted for some reason, and finally, the connection between C and D is interrupted for another reason.

As a result, even though server D is still the leader, it cannot commit any synchronous transactions because it cannot obtain three confirmations. Server B will not start an election because it still sees leader D. Servers A and C will not start an election because server B tells them that the leader is still alive.

CheckQuorum helps address this issue by forcing the leader to step down when it loses connection to a majority of nodes in the cluster, allowing one of the replicas to take its place.

It is also necessary to determine when the leader should step down. In the case of a blocked cluster, it doesn't really matter. The main concern here is that the leader eventually steps down, allowing its replicas to start an election.

If only synchronous replication is involved, everything works fine. The speed at which the leader steps down is also not important. The old leader cannot confirm any synchronous transactions after the connection is lost.

If asynchronous transactions are involved, some measures must be taken to ensure consistency.

The minimum requirement is that the old leader must step down strictly before the replicas can elect a new leader. This is to ensure that the cluster does not have two writable nodes at the same time. We call this pattern strict CheckQuorum.

But that's not enough. We cannot guarantee that no asynchronous transactions will be written after a connection failure. Detecting a connection failure takes some time, however short. This means that the old leader may write transactions that the new leader does not have. When the connection is eventually restored, the transactions will reach the other nodes, and the consistency of all the content written by the new leader may be compromised. Therefore, that is our ultimate goal.

The ultimate goal is that after the connection is restored, the new leader and all its replicas should not apply any transactions recorded by the previous leader. We will discuss this issue in more detail in the next chapter, but for now, let's discuss how to achieve the minimum requirement.

Both the leader and the replicas monitor the state of their connections through heartbeat signals. If no heartbeat signal is received from a server within 4 times the replication timeout, the connection is considered lost. The heartbeat of a replica is a response to the leader's heartbeat and is only sent after receiving the leader's heartbeat.

As you can see, the leader restarts the connection timer after the last heartbeat exchange later than the replicas. In the worst case, the replica's last heartbeat will arrive at the leader exactly when the timeout expires. The leader does not know when the replica sent its heartbeat, and its timeout may have already expired.

If the replica can maintain a fast election in addition to being able to perform a fast election, then the old leader will step down too late.

Therefore, to ensure that the leader steps down strictly before a majority of nodes start a new election, the leader's timeout needs to be half of the replica's timeout. That's what we need to do.

## Split Brain Detection

Tarantool allows you to configure a quorum. This is convenient in many cases; for example, you can emergency unlock a cluster where a majority of nodes have failed. But it is also dangerous: if the quorum is less than N / 2 + 1, it is possible to have two disconnected leaders, either within the same term or across different terms. Both leaders can independently commit synchronous transactions and write asynchronous transactions. If the two leaders reconnect after working independently in the cluster for some time, one leader's changes will overwrite the other's. To prevent this from happening, you need to detect transactions from competing leaders and terminate the connection to the node that sent them without applying them.

## PROMOTE Entry

The appearance of a new leader is signaled by a PROMOTE entry. It contains the term of the elected leader, the ID of the leader, the ID of the previous leader, and the last LSN received by the previous leader. This information is sufficient to construct a linear leadership history from the first term to the last term. When the cluster is functioning normally, each incoming PROMOTE matches the information known to the node. The term should be the largest among all PROMOTE entries so far, the ID of the previous leader must match the ID in the previous PROMOTE, and the LSN of the previous leader must match the LSN of the last confirmed transaction.

If any of the above conditions are not met, a split brain has occurred.

We also need to detect the case where the old leader continues to commit transactions after the new leader has appeared in the cluster. In fact, any transaction from a node that did not send the last PROMOTE is an indicator of a split brain.

This final example also solves the problem of our strict CheckQuorum: now any transaction (synchronous or asynchronous) from the old leader will result in a disconnection from it and will not be applied, thus maintaining the data consistency of the new leader and its replicas. Therefore, the old leader cannot affect the state of the new leader and its replicas.

## Lessons Learned

The classic version of the Raft algorithm does not provide complete cluster operability in the case of partial connection loss. To address this issue, two improvements have been made: PreVote and CheckQuorum.

Our variant of the classic implementation allows for faster elections and detects deadlocks, although additional modifications are required to ensure consistency: strict CheckQuorum and split brain detection.

You can download Tarantool from the [official website](http://www.tarantool.io/en/download/os-installation/docker-hub/?utm_source=dev&utm_medium=referral&utm_campaign=2022) and seek help in [our Telegram chat](http://t.me/tarantool?utm_source=dev&utm_medium=referral&utm_campaign=2022).