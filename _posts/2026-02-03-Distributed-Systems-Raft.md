---
title: "Understanding Raft"
published: false
---

# Understanding Raft

I have recently taken on a personal task of delving deeper into understanding distributed systems via a practical approach. 

My main materials of reference comes from the famous [MIT Distributed Systems](https://pdos.csail.mit.edu/6.824/schedule.html) series, with lectures also available on youtube.

As part of that process, I have started attempting the labs and reading the papers hope to document down some of my key learnings, as well as confusion points that I may have with respect to the things that I've learnt.

I envision this to be a multi-part series and will strive to log down points that were interesting or difficult for me to understand, so as to serve as a future reference.

# Overview

## What is Raft?
In distributed systems, the key challenge is to be able to replicate state across several state machines in a cluster, so as to allow for backups (replicas) or maybe even high availability across different regions. 

There are 2 key ways to do so, either via state transfer or replicated state (to insert link to timestamp to MIT lecture). State transfers are usually very expensive to perform hence replicating state is usually the preferred way of doing so.  In order to replicate state, a log or series of commands is usually stored so that when a state machine with blank slate is introduced, it can be replicated by running the series of commands in this log and get up to speed.

To this end, Raft is one of such techniques that aims to achieve consensus algorithm when managing replicated logs, used to perform replications in state.

## What is Paxos?
Paxos is the predecessor to Raft, in which it is also a consensus algorithm. It is famously known to be difficult to grok, which therefore spurred the invention of new conesus algorithms like Raft.

## Applications of Raft
TBC

## Raft Mechanisms
TBC

## Other References
* (Raft Extended paper)[https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf]
* (Eli Bendersky's Blog on implementing Raft)[https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/] seems to be a good resource, though I have not spent too much time going through it yet
* (Raft Consensus Algorithm)[https://www.youtube.com/watch?v=vYp4LYbnnW8]
	- A good overview of the algorithm as explained by the original authors
* https://raft.github.io/ - as demonstrated in the original authors' youtube video with a nice animation to demonstrate how it works