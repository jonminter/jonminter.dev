+++
title = "Hands-on Learning: Raft & Consensus Algorithms (Part 1 - What is consensus?)"
draft = false
[taxonomies]
tags = ["distributed systems", "consensus", "raft", "rust" ]
+++

## Why?

So I've been reading a lot lately about distributed systems and the building blocks used to build distributed systems from the ground up. This series of posts is going to focus on consensus algorithms. And on one algorithm in particular, the Raft consensus algorithm.

I will walk you through the Raft white paper and how to actually implement it.

My goals in attempting to write my own version of Raft are:

- Develop a deeper, more practical understanding on Raft & consensus algorithms. For me things tend to gel in my mind better when I do vs just raad.
- Learn Rust: I am choosing to implement this using Rust mostly because I've long been intrigued by the Rust language and this exercise presents a non-trivial problem to solve with Rust.
- Share my learnings with others that might read this

### Table of Contents

This will be an ongoing series of posts and new posts will be added to the table of contents as they are published:

- **Part 1 - Raft & Distributed Consensus**
- [Part 2 - Leader Election in Raft](/posts/raft-consensus-part2-leader-election)
- [Part 3 - Does this actually work?](/posts/raft-consensus-part2-leader-election)

## What do we mean by "consensus"?

So first off what does consensus mean in the context of distributed systems?

A distributed system is a software system that consists of multiple machines working together that appear to the client to be one single machine. Some examples could be a database system that has read replicas, a sharded key value store, or even a system of interconnected microservices.

When your program spans multiple machines it introduces some additional challenges that you don't have to worry about when compared to a single machine. You are for example forced to communicate between machines over networks, where even the most reliable networks cannot guarantee 100% delivery rate of messages and where latency of delivering those messages cannot be completely predictable.

Given the inherent complexities there should be a good reason to choose to build a distributed system. Just like choosing to write a concurrent program for a single machine system we should have a good a reason to justify the extra effort and risk of bugs when building a distributed system.

Some reasons might be:

- It is impractical/impossible to veritically scale the system on a single machine. Maybe we have high throughput demands or need massive storage capacity that makes it difficult to use a single machine. Or maybe the costs of procuring a server large enough to meet our needs is prohibitive
- These systems need to be able to function in the face of unreliable networks or hardware failures. If you need a highly avaiable, resilient system you may not have a choice but to develop a distributed system

However, with multiple machines receiving requests from clients we open the door to conflicting concurrent requests and need a way to allow our system to return consistent results in the face of these conflicts.

Consensus algorithms provide building blocks for us to build these types of resilient, consistent distributed systems.

## The Problem

To illustrate the problem at hand, let's take the example of a key-value database that exposes a simple API with two possible commands:

- GET(key) -> value
- PUT(key, value)

Let's suppose we have a cluster of 3 machines that our clients connect to. Suppose two clients send PUT requests that around the same time both for the same key but have a different value. How do we reconcile that? Or suppose a client sends a request to server 1 `PUT(foo, 3)` and later a different (or the same) client issues a `GET(foo)` request to a different server in the cluster. How can we be sure that client sees the most up to date value of `foo`?

Or what happens if a client issues a `PUT(foo, 3)` command to a server in the cluster and that server's hard disk fails, have we lost that update?

## Solutions

One possible solution for consistency issues is to make one node responsible for determining the order of writes in the system. Many RDBMS systems offer a configuration to have a primary node and some number of replica nodes for example.

Also having one or more replicas can make our system more available in the face of failure as the probability of all machines in the cluster being unavailable at the same time is lower than if we had only one machine.

However we still have a couple of problems:

- We still have a single point of failure for writes. If our master goes down we lose the ability to update our database
- Recovery from node failures is not fully automatic. We can put a load balancer in front of our cluster to spread the load across machines and this could remove a failing server from rotation if it saw that requests to the bad node were failing. We could also failover and promote a replica to replace a failing master node but there will probably be some downtime while that happens and there may be some risk of data loss as well

There exists consensus/replication algorithms that can allow a cluster to work around node failures to both primary and replica nodes,and remove the single point of failure for writes.

### Paxos

Until fairly recently the Paxos consensus algorithm has been one of the main go to algorithms to use as a basis for consensus in many distributed systems. The Paxos algorithm was developed by Leslie Lamport. He first published a paper on Paxos in 1989 and provided formal mathematical proof that Paxos satisfied the properties needed to guarantee consensus/failure handling within a distributed system.

It has some similarities to Viewstamped Replication a protocol for consensus and replication proposed by Barbar Liskov and Brian Oki in 1988. Viewstamped replication is built on the concept of a replicated state machines, where the state machine has some initial state and a historical log of events that should be applied to the state machine. Both VR and Paxos have the concept of primary/replica or leader/follower nodes. VR uses a round robin process of selecting a leader from the available nodes where Paxos uses a more sophisticated leader election process.

Although Paxos has formal proofs it has been said that it is a difficult algorithm to understand, and not always the most practical for building real world systems.

I won't go into detail on how Paxos works for two reasons. Partly because I am one of those people who finds it hard to wrap my head around Paxos and I would like to devote the space in the post to discuss an alternative the Raft protocol in more detail. I've linked to a few more thorough Paxos explanations in the Additinal Reading section at the bottom.

These challenges that system builders had with understanding and implementing Paxos led to the development of the Raft consensus algorithm where one of the primary goals was understandability.

### Raft

Raft is a newer consensus algorithm developed in 2014 by Diego Ongaro & John Ousterhout. It is built on the same concepts of leader/follower nodes and a replicated state machine like Paxos. It is designed to be simpler and easier to both understand and implement when compared to Paxos.

In Raft nodes in the system can have one of 3 states:

- Leader - responsible for handling writes and replicating writes to follower nodes
- Follower - handles read requests from clients
- Candidate - when the cluster does not have a recognized leader, one or more nodes enter candidate mode and one of the candidates are selected as the leader

We will see how these nodes transition states and coordinate in the following sections. For the visual learners there is an excellent visual demonstration of the Raft protocol called [The Secret Lives of Data](https://thesecretlivesofdata.com/raft/) that I would highly recommend watching.

### Leader election

With Raft leader election is fairly straightforward.

- Initially all nodes are in follower mode.
- There is an election timeout where followers wait for a heartbeat from the leader and if they do not receive a heartbeat they enter candidate mode.
- In candidate mode a node votes for itself and then sends messages to the other cluster members requesting votes.
- The candidate that receives votes from the majority of the members of the cluster will become the leader.
- Once it becomes the leader it sends out heartbeat messages to notify the rest of the cluster that it has become the leader. Raft uses the same RPC message for replication and heartbeating. The leader sends an empty replication RPC as a heartbeat if no write request has been received within the heartbeat timeout. Followers will only enter candidate mode if they don't receive the heartbeat before the timeout
- To mitigate a potential deadlock where there are too many nodes in candidate mode each node uses a randomly chosen timeout each election cycle between 150 - 300ms.

### Replication

All writes go through the leader node. Any node can receive a write request but it will forward the request to be processed by the leader node.

- The leader first writes the request to a write ahead log that is persisted to the servers disk. This means that even if the server process crashes before it finishes processing the write request it can pick up where it left off when it restarts.
- The leader then sends out requests to the other cluster members to append the write request to their log
- Once it receives successful responses from a majority of the cluster it can respond to the client with a success response. This means that the request has been replicated to a majority of the cluster
- The leader can now apply the message to it's internal state machine
- Once the leader has applied the message the followers can do the same
- If a follower is not up to date it will coordinate with the follower to receive any messages it is missing

### Node failures

A Raft cluster of N nodes can tolerate up to N/2 - 1 nodes failing and still be able to make progress.

- If a leader node dies it won't be able to send heartbeat messages. When this happens a new leader will be chosen and eventually the write will succeed if retried
- If a follower dies it the client can try the same request against a different node in the cluster
- If a client does not receive a successful response from the node it sent the request to, it should retry the request with another node in the cluster until it receives a successful response.

### Additional considerations

- Since Raft is built on top of a replicated state machine with a log eventually the log will grow too large if the system is up long enough as a machine will have finite resources such as memory and disk. Raft provides a mechanism for compacting the log while maintaining consistency/availability.
- To build a true highly avaible system we need to be able to add/remove nodes from the cluster. For example if a node dies and can't be recovered we may need to add a new machine to replace it. Or we may need to increase the number of replicas to be able to handle more read throughput. We would want to be able to do this without causing any downtime or consistency issues. Raft also allows for safe cluster membership changes.

We will explore these in more detail in later posts in this series.

## Next

Now that we understand the problem of consensus within distributed systems and how we can use the Raft algorithm to solve this problem, we will implement our version of Raft starting first with leader election in the next post -> [Part 2 - Leader Election in Raft](/posts/raft-consensus-part2-leader-election)

## Additional Reading

- [Paxos Made Simple - Leslie Lamport](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Paxos Explanation - Martin Fowler](https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html)
- [Raft Whitepaper - Diego Ongaro & John Ousterhout](https://raft.github.io/raft.pdf)
- [The Secret Life of Data - Raft Visualization](https://thesecretlivesofdata.com/raft/)
