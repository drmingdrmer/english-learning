# Raft is not almighty: How to make it more robust

Everyone loves Raft. It is widely believed that using this algorithm in a distributed system means that the system will be very good. Specifically:

1. As long as the majority of nodes in the cluster are active and connected to each other, the cluster will be writable (with a pause for leader election).
2. If the leader is running normally and connected to the majority of nodes, the cluster will always be writable.
3. If the leader crashes, a new leader will be "quickly" elected - whatever that means.

In reality, simply following the Raft specification (https://raft.github.io/raft.pdf or Diego Ongaro's doctoral dissertation version) is not enough to achieve all of the above. Even etcd has suffered from this, leading to the Cloudflare outage in 2020 (https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/).

My name is Sergey Petrenko, and I have been working on replication in Tarantool for 4 years. Today, I want to tell you about the weaknesses of the Raft algorithm and how to overcome them. This article is a free adaptation of my and Boris Stepanenko's talk at Hydra 2022. If you are not familiar with Raft, I recommend reading [my article](https://dzone.com/articles/raft-in-tarantool-how-it-works-and-how-to-use-it).

![Image description](https://res.cloudinary.com/practicaldev/image/fetch/s--sRlhwn_T--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mt0wckqsxaqv041ulfw3.png)

Let's start from afar. Suppose you have a system that uses Raft and you want to use it in production. We have all heard that using Raft guarantees at most one leader per term and maintains performance even with less than half of the nodes in the cluster lost. That is, if the leader is active, it will be able to process entries, and if there is no leader, a new leader will be elected. Besides these guarantees, it seems that we don't need anything else for the system to run smoothly in the long term. Is this true? We are about to find out.

First, let's take a closer look at the claim that the cluster will keep running as long as the majority of servers are active and connected to each other. Misunderstanding this claim ultimately led to the Cloudflare outage.

## Pre-voting

Let's start with an example. We will look at the behavior of a Raft cluster in the case of partial connection loss.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/ajhkj5j9k9cwbdt6jxtu-20240131104950473.gif)

Suppose our cluster consists of three nodes: A, B, and C. A is the leader. B and C are its replicas; this is the 2022nd term. What happens if the connection between A and C is lost? During the election timeout, if the leader does not send a heartbeat signal, server C will consider the leader lost and start an election in 2023.

Server B is the only server to which server C can send a RequestVote request.

Once server B receives a request with a larger term, it also increments its term. However, it may not vote for C because if the leader wrote some entries after the connection between A and C was lost, B may have more up-to-date data.

In response to the next message from the leader (still server A), server B reports that the term has increased, forcing server A to resign. The cluster has no leader, so an election begins. The cluster is not writable until a leader is chosen.

If there are no new entries after the self-connection is lost, once server B receives a RequestVote from server C, it will vote for server C.

![Image description](https://res.cloudinary.com/practicaldev/image/fetch/s--ech3-gSA--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kbg4lpz8u1qzrftjygy9.png)

If this happens, we are stuck in an infinite loop:

1. Server C cannot see the leader.
2. When the election timeout expires, server C starts a new election.
3. The increase in term is conveyed to server A through server B.
4. Server B votes for server C, so
5. Server A resigns.
6. A and C swap places and repeat from step 1 in the new term.

Clients connected to a cluster with such a flickering leader may not be able to write anything at all. Once an active leader is elected and the client tries to perform a write operation, the leader resigns without even knowing where to forward the write request because it cannot directly see the new leader. All of this will continue to happen until at least one entry can be created or until the connection between A and C is restored. This is exactly what happened with Cloudflare and etcd. External systems reasonably assume that if they cannot write anything to the etcd cluster, it means that most of the nodes in it have failed and emergency measures need to be taken. The inability to write in etcd for just 6 minutes resulted in an incident lasting over 6 hours.

You might think that some of Raft's guarantees are violated here. In fact, they are not.

At every moment, the leader is connected to the majority of nodes (including itself), and the leader itself has not failed. However, our expectations are shattered. The cluster cannot function properly.

But the problem is that Raft never promised to function properly in this situation. Its promise is "if the majority of nodes are functioning and connected to each other, a leader will be chosen." There is no mention of the leader being permanent. And the guarantee of choosing a leader is not actually violated. In the example above, it happens approximately once per election timeout. The result is that, in practice, Raft's guarantees are not sufficient.

But that's only half of the trouble.

Let's go back to the 2022nd term and assume that in our cluster of nodes A, B, and C, node C loses all communication with the others. Server A is the leader, it is still connected to server B, and can handle write requests. So far, so good.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/zu6dftzc89uqcri8nd0m-20240131104950525.gif)

Just like in the previous example, server C will start a new election at each election timeout. Of course, server C cannot win any election. It simply cannot get a majority of votes. However, its term will keep increasing, and at the moment of connection recovery, server A will resign as soon as it sees a larger term. The cluster becomes read-only again, having gone through at least one round of elections for no apparent reason.

## Raft's solution

This problem of "disruptive" servers is actually mentioned by Raft's author, Diego Ongaro, in his doctoral dissertation. However, he mentions them in relation to configuration changes.

The solution he proposes is a new phase of the election called Pre-Vote. The idea is that each server sends a simulated vote request, called PreVote, to everyone before the actual election. This request contains the same fields as a regular RequestVote, and the voter responds to it according to the same rules as a RequestVote. The only difference is that the voter does not increment its term upon receiving this request and will reject or simply ignore the request if it still sees a leader in the current term.

The candidate will only promote its term and start a real election once it receives confirmation from a majority of voters that they are ready to vote for it.

Let's check if PreVote can really save Cloudflare from the impact of this incident. Consider the same situation where C loses connection with the leader but not with the other replicas. Although C can send PreVote requests to B, B will reject it because it still sees the leader. C will not be able to get a majority of votes - 2 responses to PreVote.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/vp6lymwnb8lvt8iwep5j-20240131104950473.gif)

In the case of complete loss of connection and subsequent recovery, PreVote also helps. Indeed, server C will not receive any responses to its PreVote requests, so it will not start an election.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/2p8t0b8xi7lzwzck5ohu-20240131104950604.gif)

The advantage of this solution is its backward compatibility. We do not need to introduce a new request and send it to old servers in a special way if there are different versions of Tarantool servers in the cluster.

The drawback is that we do not compare the leader's log with the logs of the voters, which means that we may start an election even if the voters' logs contain more up-to-date data. This is not necessarily a bad thing: as long as someone sees the leader, no election will take place. When no one sees it anymore, the election will start anyway. It doesn't matter if the election ends in one or more rounds, thanks to our other modification, Split-Vote detection. Now, let's talk about it.

## Split-Vote detection

This time, I don't have a scary case from Cloudflare or other famous companies, but I hope it's still interesting.

You may know that not every round of elections results in a leader. For example, if several nodes in the cluster notice the leader loss at the same time, they will independently start elections before receiving RequestVote requests from each other. The votes of the remaining replicas may be scattered among several candidates, so that no candidate can win a majority of votes. We call this situation a split vote.

Raft handles this problem by randomizing the election timeout. In each server and each round, a new random value slightly different from the configured value is chosen. This increases the chance that only one candidate has time to start the next round of elections, while other candidates receive RequestVote requests from it before their own election timeout expires. If Raft did not have this feature, elections might never end: all servers would restart in sync and vote for themselves. But there is still room for optimization: each additional election time is a part of the election timeout when the read-only cluster is down.

## Our solution

"In Tarantool, servers know that they won't start an election when it's unnecessary," we thought, and we did Pre-Vote in our own way. We made all replicas tell each other whether they see the current leader. After collecting information about who sees the leader and who doesn't, it can be decided whether to start an election. If anyone sees the leader or if there is no connection to the majority of nodes, an election will not be initiated.

The advantage of our solution is its backward compatibility. We don't need to introduce a new request and send it to old servers in a special way if there are different versions of Tarantool servers in the cluster.

The drawback is that we don't compare the leader's log with the logs of the voters, which means that we may start an election even if the voters' logs contain more up-to-date data. This is not necessarily a bad thing: as long as someone sees the leader, no election will take place. When no one sees it anymore, the election will start anyway. It doesn't matter if the election ends in one or more rounds, thanks to our other modification, Split-Vote detection. Now, let's talk about it.

In the worst case scenario, when two consecutive ties occur, we handle it faster. For us, each round of a tie has almost no cost. Additionally, we do not increase the probability of a second tie in any way: in this case, the random delay for restarting the election is generated within the same range as in the usual case.

## CheckQuorum

Raft ensures that there will not be two leaders within one term. However, it does not specify the possibility of two leaders existing at the same time: in the old term and the new term. We certainly want to avoid this situation. The reason is that clients connected to the old leader may not be aware of the existence of a new leader. They have no reason to look for a new leader until the old leader actually resigns. The old leader is still writable, and synchronous transactions will be rolled back if they are not confirmed by a majority of replicas.

For us, the existence of two leaders at the same time can be crucial because Tarantool supports not only synchronous replication but also asynchronous replication. Each leader is writable, and if you write asynchronous transactions on the old leader, clients may ignore the existence of the new leader. This will result in some clients contacting the new leader while others contact the old leader. We will encounter the problem of split brain. To avoid this situation, the old leader must resign and enter a read-only state when it is possible to be replaced.

The possibility of having two leaders only exists when the old leader loses connection with a majority of servers in the cluster. In fact, to win an election, you must obtain a majority of votes, and the only possibility for the old leader to be unaware of the election is if it is not connected to a majority of nodes. If it is connected to at least one server that voted for the new leader, it will immediately resign. It is also impossible for the old leader to be connected to a majority of nodes but receive no votes, while the new leader is connected to a majority of votes.

Therefore, we want to ensure that the leader resigns immediately when it loses connection with a majority of the cluster. If the current leader's connection is lost, it is likely that a majority has already chosen another leader.

There is another reason for CheckQuorum: if only PreVote is used, the cluster may be locked: the current leader cannot write anything, and replicas cannot start an election. Let's look at an example: suppose we have a cluster consisting of five servers, with D as the leader. Then a series of unusual events occur: first, server E crashes, then the connection between A and D is lost for some reason, and finally, the connection between C and D is lost for another reason.

As a result, although server D is still the leader, it cannot commit synchronous transactions because it cannot obtain three confirmations. Server B will not start an election because it still sees leader D. Servers A and C will also not start an election because server B tells them that the leader is active.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/oizii764s48dq1tm3c49-20240131104950552.gif)

CheckQuorum helps to solve this problem by making the leader who loses connection with a majority of the cluster resign and allowing one of the replicas to take its place.

![Image description](raft-notalmighty-how-to-make-it-more-robust.en.assets/hch89oq8nf174a838uj4-20240131104950558.gif)

Now we need to decide when the leader should resign. For the blocked cluster example, this is not important. The main issue here is that the leader eventually resigns, allowing its replicas to start an election.

If only synchronous replication is involved, everything will be fine. The speed of resignation is also not important. The old leader cannot confirm any synchronous transactions after the connection is lost.

If asynchronous transactions are involved, you need to take some measures to ensure consistency.

The minimum requirement is that the old leader must resign strictly before the replicas start an election. This is to ensure that there are not two writable nodes simultaneously in the cluster. We call this pattern strict CheckQuorum.

But that's not enough. We cannot guarantee that no asynchronous transactions will be written after a connection failure. In any case, it takes some time, however short, to detect a connection failure. This means that the old leader may write a transaction that the new leader does not have. When the connection is eventually restored, the transaction will reach the other nodes, and the consistency of all the content written by the new leader may be compromised. Therefore, this is our ultimate goal.

The ultimate goal: after the connection is restored, the new leader and all its replicas must not apply the transactions recorded by the previous leader. We will discuss this issue in more detail in the next chapter, but for now, let's discuss how to achieve the minimum requirement.

Both the leader and the replicas monitor the status of the connection through heartbeat signals. If no heartbeat signal is received from a server within 4 times the replication timeout, the connection is considered lost. The heartbeat of the replicas is a response to the heartbeat of the primary server and is only sent after receiving the heartbeat from the primary server.

As you can see, the leader restarts the connection timer later than the replicas after the last heartbeat exchange. In the worst case, the last heartbeat from the replicas will arrive at the leader exactly when the timeout expires. The leader does not know when the replicas sent the heartbeat, and its timeout may have already expired.

If the replicas are still able to perform a fast election, the old leader will resign too late.

Therefore, to ensure that the leader strictly resigns before a majority of nodes start a new election, the leader's timeout needs to be half of the replicas' timeout. That's what we need to do.

## Split-Brain Detection

Tarantool allows you to configure a quorum. This is convenient in many cases; for example, you can emergency unlock a cluster with a majority of nodes failed. But it is also dangerous: if the quorum is less than N / 2 + 1, there may be two disconnected leaders. Whether in the same term or in different terms. Both of these leaders can independently confirm synchronous transactions and write asynchronous transactions. If the two leaders reconnect after working in the cluster for some time, one of their changes will overwrite the other's changes. To prevent this from happening, you need to detect transactions from competing leaders and terminate the connection with the node that sent them without applying them.

## PROMOTE Entry

The appearance of a new leader is indicated by a PROMOTE entry. It contains the term in which the leader was elected, the ID of the leader, the ID of the previous leader, and the last LSN received by the previous leader. This information is sufficient to construct a linear leadership history from the first term to the last term. When the cluster is running normally, each incoming PROMOTE matches the information known to the node. The term should be the largest among all PROMOTE entries so far, the ID of the previous leader must match the ID in the previous PROMOTE, and the LSN of the previous leader must match the LSN of the last confirmed transaction by the previous leader.

If any of the above conditions are not met, a split brain occurs.

We also need to detect the case where the old leader continues to confirm transactions after the new leader appears in the cluster. In fact, any transaction from a node that did not send the last PROMOTE is an indicator of split brain.

The last example also solves the problem of our strict CheckQuorum: now any transaction from the old leader (including synchronous and asynchronous) will cause the connection with it to be terminated and will not be applied, thus maintaining the data consistency of the new leader and its replicas. Therefore, the old leader cannot affect the state of the new leader and its replicas.

## Lessons Learned

The specification version of the Raft algorithm does not provide complete cluster operability in the case of partial connection loss. To address this issue, the following two improvements were made: PreVote and CheckQuorum.

The improvements to the specification version allow for faster elections and detection of ties, although additional modifications are needed to ensure consistency: strict CheckQuorum and Split-Brain Detection.

You can [download Tarantool](http://www.tarantool.io/en/download/os-installation/docker-hub/?utm_source=dev&utm_medium=referral&utm_campaign=2022) from the official website and get help in our Telegram chat [here](http://t.me/tarantool?utm_source=dev&utm_medium=referral&utm_campaign=2022).