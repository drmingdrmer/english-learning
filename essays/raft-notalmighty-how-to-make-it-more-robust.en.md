# Raft (Not) Almighty: How to Make It More Robust

[Original Article](https://dev.to/tarantool/raft-notalmighty-how-to-make-it-more-robust-3a11)

[#raft](https://dev.to/t/raft)[#quorum](https://dev.to/t/quorum)[#algorithms](https://dev.to/t/algorithms)[#programming](https://dev.to/t/programming)

Everyone loves Raft. It is widely believed that using this algorithm in a distributed system means that the system will work well. Specifically:

1. As long as the majority of nodes in the cluster are active and connected to each other, the cluster can perform write operations (with a pause during leader election).
2. If the leader is functioning properly and connected to the majority of nodes, the cluster will always be able to perform write operations.
3. If the leader crashes, a new leader will be "quickly" selected, regardless of what "quickly" means.

In reality, simply following the Raft specification (https://raft.github.io/raft.pdf or Diego Ongaro's doctoral dissertation version) is not enough to achieve all of the above. Even etcd suffered a loss because of this, resulting in the [2020 Cloudflare outage](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/).

My name is Sergey Petrenko, and I have been working on replication in Tarantool for 4 years. Today, I want to tell you about the weaknesses of the Raft algorithm and how to overcome them. This article is a free paraphrase of my and Boris Stepanenko's presentation at Hydra 2022. If you are not familiar with Raft, I recommend reading [my article](https://dzone.com/articles/raft-in-tarantool-how-it-works-and-how-to-use-it) first.

[![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/mt0wckqsxaqv041ulfw3-20240131104950569.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--sRlhwn_T--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mt0wckqsxaqv041ulfw3.png)

Let's start from afar. Suppose you have a system that uses Raft and you want to use it in a production environment. We have all heard, more than once, that Raft guarantees only one leader per term and does not lose performance as long as less than half of the nodes in the cluster are lost. In other words, if the leader is active, it will be able to process entries, and if there is no leader, a new leader will be selected. It seems that, apart from these guarantees, we don't need anything else for the system to run smoothly. Is that really the case? We are about to find out.

First, let's take a closer look at the claim that the cluster can keep running as long as the majority of servers are active and connected to each other. Misunderstanding this claim ultimately led to the Cloudflare outage.

## Pre-Vote

Let's start with an example. We will look at the behavior of a Raft cluster in the case of partial connection loss.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/ajhkj5j9k9cwbdt6jxtu-20240131104950473.gif)

Suppose our cluster consists of three nodes: A, B, and C. A is the leader. B and C are its replicas; this is the 2022nd term. What happens when the connection between A and C is lost? Since the leader does not send heartbeats during the election timeout, server C considers the leader lost and starts an election in the 2023rd term.

Server B is the only server to which server C can send a RequestVote request.

Once server B receives a request with a larger term, it also increments its own term. However, it may not vote for C because if the leader wrote some entries after the connection between A and C was lost, B may have more up-to-date data.

When server B receives the next message from the leader (still server A), it reports an increased term, forcing server A to resign. The cluster has no leader, so an election begins. The cluster is not writable until a leader is chosen.

If there have been no new entries since the self-connection loss, once server B receives a RequestVote from server C, it will vote for server C.

[![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/kbg4lpz8u1qzrftjygy9-20240131104950437.png)](https://res.cloudinary.com/practicaldev/image/fetch/s--ech3-gSA--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kbg4lpz8u1qzrftjygy9.png)

If this happens, we are stuck in an infinite loop:

1. Server C cannot see the leader.
2. When the election timeout expires, server C starts a new election.
3. The increase in term is conveyed through server B.
4. Server B votes for server C, so
5. Server A resigns.
6. A and C swap places and repeat from step 1, but in a new term.

Clients connected to a cluster with such a flickering leader may not be able to write anything at all. Once a temporary leader is elected and the client tries to perform a write operation, the leader resigns without even knowing where to forward the write request, as it cannot directly see the new leader. All of these situations will continue to occur until at least one entry can be created or until the connection between A and C is restored. This is exactly what happened to Cloudflare with etcd. The external system reasonably assumed that if it cannot write anything to the etcd cluster, it means that most of the nodes in it have failed and immediate action needs to be taken. The fact that etcd was not writable for just 6 minutes resulted in an incident that lasted over 6 hours.

You might think that some of Raft's guarantees are being violated here. In fact, they are not.

At any given moment, the leader is connected to the majority of nodes (including itself), and the leader itself does not fail. However, our expectations have been shattered. The cluster cannot function properly.

But the problem is, Raft never promised operationality in this situation. Its promise is "if the majority of nodes are functioning and connected to each other, a leader will be chosen." There is no mention of the leader being permanent. And the guarantee of choosing a leader is not actually being violated. In the example above, it happens about once per election timeout. In practice, Raft's guarantee is not sufficient.

But that's only half of the problem.

Let's go back to the 2022nd term and assume that in the cluster of nodes A, B, and C, node C has lost all communication with the other nodes. Server A is the leader, it is still connected to server B, and it can handle write requests. So far, so good.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/zu6dftzc89uqcri8nd0m-20240131104950525.gif)

Just like in the previous example, server C will start a new election at each election timeout. Of course, server C cannot win any election. It simply cannot get a majority of votes. However, its term will keep increasing, and at the moment of connection recovery, server A resigns again because it sees a larger term. The cluster enters a state where it can only be read, and there will be at least one round of election.

## Raft's Solution

This problem of "disruptive" servers is actually mentioned by Raft's author, Diego Ongaro, in his doctoral dissertation. However, he mentions it in relation to configuration changes.

The solution he proposes is a new phase of elections called Pre-Vote. The idea is that each server sends a simulated vote request, called PreVote, to everyone before the actual election. This request contains the same fields as a regular RequestVote, and the voters respond to it according to the same rules as RequestVote. The only difference is that voters do not increment their term upon receiving this request, and if they still see a leader in the current term, they respond negatively or simply ignore the request.

Only when confirmation is received from a majority of voters that they are ready to vote for the candidate, the candidate increases its term and starts the actual election.

Let's see if PreVote can really save the Cloudflare incident. Consider the same situation where C loses connection with the leader but not with the other replicas. Although C can send PreVote requests to B, B will respond negatively because it still sees the leader. C will not be able to get a majority of votes - there will be 2 negative responses to its PreVote request.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/vp6lymwnb8lvt8iwep5j-20240131104950473.gif)

In the case of complete loss of connection and subsequent recovery, PreVote also helps. Indeed, server C will not receive any positive responses to its PreVote requests, so it will not start an election.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/2p8t0b8xi7lzwzck5ohu-20240131104950604.gif)

The only downside of this solution for us is that it violates backward compatibility. Raft has been running in Tarantool for some time, and there are no PreVote requests in the protocol. It can certainly be added, but old servers do not know how to handle the new requests. We need to introduce new logic on the sender side: if the server is old, we do not send PreVote requests and assume that it will definitely respond positively. We don't like this redundancy and want to get rid of the extra code that exists to support old versions. For us, a better solution is to extend one of the existing requests. Old servers simply ignore the new field in the existing request, so no additional logic is needed. This is the way we chose.

## Our Solution

"In Tarantool, servers know that they won't start an election unnecessarily," we thought, and we implemented Pre-Vote in our own way. We let all replicas inform other replicas whether they see the current leader or not. Once we have collected information about who sees the leader and who doesn't, we can decide whether to start an election. If anyone sees the leader or there is no connection to the majority of nodes, an election will not be initiated.

The advantage of our solution is its backward compatibility. We don't need to introduce new requests and send them to old servers in a special way if there are different versions of Tarantool servers in the cluster.

The downside is that we don't compare the leader's log with the logs of the voters, which means that an election will start even if the voters' logs contain more up-to-date data. This is not necessarily a bad thing: as long as someone sees the leader, an election will not take place. When no one sees it anymore, the election will start anyway. Whether it ends in one or more rounds doesn't matter, thanks to another modification we made, Split-Vote detection. Now, let's talk about it.

## Split-Vote Detection

This time, I don't have a terrifying case from Cloudflare or other famous companies, but I hope it's still interesting.

You may know that not every round of election results in a leader being elected. For example, if several nodes in the cluster notice the leader loss at the same time, they will independently start elections before receiving RequestVote requests from each other. The votes of the remaining replicas may be scattered among several candidates, so that no candidate can win a majority of votes. We call this situation Split-Vote.

Raft handles this problem by randomizing the election timeout. In each server and each round, a new random value is chosen that is slightly different from the configured timeout. This increases the chance that only one candidate has time to start the next round of elections, while other candidates receive RequestVote requests from it before their own election timeout expires. If Raft did not implement this feature, elections might never end: all servers would synchronously restart and vote for themselves. But there is still room for optimization: each additional round of election is an election timeout time when the read-only cluster is down.

## Our Solution

In practice, rounds that end in a tie are a waste of time. If a leader has already been elected, it may have been elected at the start of the round. According to Raft's specification, the election timeout should be much longer than that.

Also, unlike the classic Raft implementation, in Tarantool, servers not only send voting information to the candidate they voted for but also to all cluster members. This means that each server can track the number of votes each candidate receives. When it is seen that no candidate can win in the current round, servers speed up the start of a new round. From the detection of a tie to the start of a new round, on average, it takes 0.05 * the election timeout time. This is because we choose a random delay within the range of election timeout time / 10 to restart the election. We do this for the same purpose as Raft: to avoid new ties. Therefore, in the best case, we save about 0.9 * the election timeout time on each round with a tie. This improvement is very noticeable: in the original Raft, electing a leader in two rounds meant spending a fraction of the election timeout time + a fraction of the election timeout time. However, with our implementation, electing a leader in two such rounds takes less than a fraction of the election timeout time. This is faster than the time it takes for the original Raft to realize that a tie has occurred.

In the worst case scenario, when two consecutive ties occur, we handle it faster. For us, each round of a tie has almost no cost. Additionally, we do not increase the probability of a second tie in any way: in this case, the random delay for restarting the election is generated within the same range as in the usual case.

## CheckQuorum

Raft ensures that there will not be two leaders within one term. However, it does not address the possibility of having two leaders at the same time: in the old term and the new term. We definitely want to avoid this situation. The reason is that clients connected to the old leader may not be aware of the existence of a new leader. They have no reason to look for a new leader until the old leader actually resigns. The old leader is still writable, and synchronous transactions will roll back if they fail to receive confirmation from a majority of replicas.

For us, the presence of two leaders at the same time can be critical because Tarantool supports not only synchronous replication but also asynchronous replication. Each leader is writable, and if you write asynchronous transactions on the old leader, clients may ignore the existence of the new leader. This will result in some clients contacting the new leader while others contact the old leader. We will encounter the problem of split-brain. To avoid this situation, the old leader must resign and enter a read-only state when it is likely to be replaced.

The situation of having two leaders at the same time can only occur when the old leader loses connection with a majority of servers in the cluster. In order to win an election, you must obtain a majority of votes. The reason the old leader is unaware of the election is because it is not connected to a majority of nodes. If it is connected to at least one server that votes for the new leader, it will immediately resign. It is also impossible for the old leader to have no votes from a majority of nodes while the new leader is connected to a majority of voting nodes.

Therefore, we want to ensure that the leader resigns immediately after losing connection with a majority of nodes in the cluster. If the current leader's connection is lost, it is highly likely that a majority has already chosen another leader.

There is another reason for CheckQuorum: if only PreVote is used, the cluster may be locked: the current leader cannot write anything, and replicas cannot start an election. Let's look at an example: suppose we have a cluster consisting of five servers, with D being the leader. Then a series of unusual events occur: first, server E crashes, then the connection between A and D is interrupted for some reason, and finally, the connection between C and D is interrupted for another reason.

As a result, although server D is still the leader, it cannot commit synchronous transactions because it cannot obtain three confirmations. Server B will not start an election because it still sees leader D. Servers A and C will not start an election because server B tells them that the leader is active.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/oizii764s48dq1tm3c49-20240131104950552.gif)

CheckQuorum helps address this issue by forcing the leader that loses connection with a majority of the cluster to resign and allowing one of the replicas to take its place.

![Image Description](raft-notalmighty-how-to-make-it-more-robust.en.assets/hch89oq8nf174a838uj4-20240131104950558.gif)

Now the decision needs to be made as to when the leader should resign. It is not actually important for the blocked cluster example. The main issue here is that the leader will eventually resign, allowing its replicas to start an election.

If only synchronous replication is involved, everything will be fine. The speed of resignation is also not important. The old leader cannot confirm any synchronous transactions after the connection is lost.

If asynchronous transactions are involved, some measures must be taken to ensure consistency.

The **minimum requirement** is that the old leader must resign **strictly** before the replicas start an election. This is to ensure that the cluster does not have two writable nodes at the same time. We call this pattern strict CheckQuorum.

But that's not enough. We cannot guarantee that no asynchronous transactions will be written after a connection failure. In any case, it takes some time, however short, to detect a connection failure. This means that the old leader may write transactions that the new leader does not have. When the connection is eventually restored, the transactions will be propagated to other nodes, and the consistency of all the content written by the new leader may be compromised. Therefore, that is our ultimate goal.

**Ultimate goal:** After the connection is restored, the new leader and all its replicas must not apply transactions recorded by the previous leader. We will discuss this issue in more detail in the next chapter, but for now, let's discuss how to achieve the minimum requirement.

Both the leader and the replicas monitor the state of the connection through heartbeat. If no heartbeat is received from a server within 4 times the replication timeout, the connection is considered lost. The heartbeat of a replica is a response to the heartbeat of the primary server and is only sent after receiving the heartbeat from the primary server.

As you can see, the leader restarts the connection timer after the last heartbeat exchange later than the replicas. In the worst case scenario, the last heartbeat from a replica will arrive at the leader exactly when the timeout expires. The leader does not know when the replica sent its heartbeat, and its timeout may have already expired.

If the replica can also start an election in addition to being able to perform a fast election, then the old leader will resign too late.

Therefore, to ensure that the leader strictly resigns before a majority of nodes start a new election, the leader's timeout needs to be half of the replica's timeout. That's what we need to do.

## Split-Brain Detection

Tarantool allows you to configure a quorum. This is convenient in many cases; for example, you can emergency unlock a cluster where a majority of nodes have failed. But it is also dangerous: if the quorum is less than N / 2 + 1, there may be two leaders that cannot connect. Whether in the same term or in different terms. Both of these leaders can independently confirm synchronous transactions and write asynchronous transactions. If the two leaders recover their connection after working in the cluster for some time, one leader's changes will overwrite the other's changes. To prevent this from happening, you need to detect transactions from competing leaders and terminate the connection with the node that sent them without applying them.

## PROMOTE Entry

The appearance of a new leader is indicated by the PROMOTE entry. It contains the term of the elected leader, the ID of the leader, the ID of the previous leader, and the last LSN received by the previous leader. This information is sufficient to construct a linear leadership history from the first term to the last term. When the cluster is functioning normally, each incoming PROMOTE is matched against the information known to the node. The term should be the largest among all PROMOTE entries so far, the ID of the previous leader must match the ID in the previous PROMOTE entry, and the LSN of the previous leader must match the LSN of the last confirmed transaction by the previous leader.

If any of the above conditions are not met, a split-brain occurs.

We also need to detect the case where the old leader continues to confirm transactions after the new leader appears in the cluster. In fact, any transaction from a node that has not sent the last PROMOTE is an indicator of split-brain.

This final example also solves the problem we had in strict CheckQuorum: now any transaction from the old leader (including synchronous and asynchronous transactions) will result in a disconnection from it and will not be applied, thus maintaining the data consistency of the new leader and its replicas. Therefore, the old leader cannot affect the state of the new leader and its replicas.

## Lessons Learned

The classic version of the Raft algorithm does not provide complete cluster operability in the case of partial connection loss. To address this issue, two improvements have been made: PreVote and CheckQuorum.

Variants of the classic implementation allow for faster elections and detection of ties, although additional modifications are required to ensure consistency: strict CheckQuorum and Split-Brain Detection.

You can download Tarantool from the [official website](http://www.tarantool.io/en/download/os-installation/docker-hub/?utm_source=dev&utm_medium=referral&utm_campaign=2022) and seek help in [our Telegram chat](http://t.me/tarantool?utm_source=dev&utm_medium=referral&utm_campaign=2022).