# Chapter 10: Leader Election

Problem: Synchronization can be costly because of significant communication overhead.

Solution: Leader election.

Leader == a process that is responsible for coordinating the actions of the other processes.

Liveness of election algorithm
- Guarantees that most of the time, there will be a leader in the system.
- Election will eventually terminate and a leader will be elected.

Safety:
- Ideally, only one process should be elected as the leader.

When is leader election triggered?
- When the system initializes.
- When the existing leader fails.

Some algorithms (ZAB, Multi-Paxos, Raft) use temporary leaders to reduce the number of messages required to reach consensus.

## Bully Algorithm

- Leader election algorithm.
- Uses process ranks to elect a leader.
  - Each process is assigned a unique rank.
  - During the election, the process with the highest rank is elected as the leader.
- Called "bully" because the process with the highest rank "bullies" the other processes into accepting it as the leader.
- Also called monarchial leader election.

Election happens when:
- one of the processes notices that there's no leader. (it was never initialized)
- existing leader has stopped responding to requests.

Election happens in 3 steps:
1. Process (that noticed the absence of a leader) sends election messages to all processes with higher ranks.
2. If no response is received, the process declares itself as the leader, and notifies all other lower ranked processes.
3. If a response is received, the process notifies the highest-ranked process that responded to do step 2.

How to write a nested numbered list in markdown?
1. Step 1
    1. Step 1.1
    2. Step 1.2


Problems:
1. Safety: In the presence of network partitions, multiple leaders can be elected.
    1. We can end in a situation where nodes get split into independent subnets, and each subnet elects its own leader. This situation is called a split brain.
2. Strong preference for higher ranked processes. This can lead to a situation if the higher-ranked process keeps failing shortly after winning the election.

## Next-In-Line Failover

- Each elected leader provides a list of failover candidate nodes.
- When a process detects that the leader has failed, it starts a new election round by contacting the highest-ranked failover candidate from the list.
- If the failover candidate is not available, the process contacts the next highest-ranked candidate. If it is available, it becomes the new leader.
- If the process itself is the highest-ranked candidate, it becomes the new leader and notifies all other processes.

This improves upon the Bully algorithm by reducing the number of messages required to elect a new leader.

## Candidate/Ordinary Optimization

This is another optimization to reduce the number of messages required to elect a new leader.

In this, we split the nodes into 2 subsets:
1. Candidate nodes: These are the nodes that can become the leader.
2. Ordinary nodes

Process: Ordinary process initiates an election round by sending an election message to all candidate nodes, picks the highest-ranked candidate that responds, and notifies all other processes.

## Invitation Algorithm

- Invitation Algorithm allows processes to invite other processes to join their groups.
- By definition, it allows multiple leaders to coexist in the system.

✨ **Side learning:**
The Invitation Algorithm provides a protocol for forming groups of available processors within partitions, and then creating larger groups as failed processors are returned to service or network partitions are rectified. ([Source](https://cseweb.ucsd.edu/classes/sp16/cse291-e/applications/ln/lecture6.html#:~:text=The%20Invitation%20Algorithm%20provides%20a,or%20network%20partitions%20are%20rectified.))

Steps:
- Each process starts as a leader of a new group, which contains only itself.
- Group leaders contact peers to invite them to join their groups.
  - If the peer process is a leader, both the groups are merged.
  - If the peer process is not a leader, it responds with the group-leader-ID, allowing communication between the two group leaders, and leading to merging of the groups.

Generally, leader of a larger group becomes the leader of the merged group. This way, the number of messages required to elect a leader is reduced, as processes from the smaller group have to be notified about the new leader.

## Ring Algorithm

- All nodes are arranged in a ring.
- All nodes are aware of their neighbors (predecessor and successor).

Steps:
- Each node starts an election round by sending an election message to its successor.
- If the successor is not available, the node skips it and sends the message to the next node, until it finds a node that responds.
- Nodes keep collecting the "live node set". They add themselves to the set before passing it to their successor.
- The whole ring is traversed once, until the initiator receives the message back with the "live node set".
- When the message reaches the initiator, the highest-ranked node in the "live node set" is elected as the leader.
- The initiator notifies all other nodes about the new leader.

## Pending Doubts

1. It was mentioned that some algorithms (ZAB, Multi-Paxos, Raft) use temporary leaders to reduce the number of messages required to reach consensus. Does this simply mean they re-elect a leader every now and then? If yes, how does it help?

[Update] Got a couple of answers by folks in the reading group discussion thread:

Answer 1: I think we'll get to Raft in another chapter, but yes, they do elect a new leader every so often. A leader (A) won't itself step down until it sees another node (B) claiming to be a leader for a more recent election than (A) is aware of. A follower won't attempt to become a leader until it hasn't heard from that leader within a period of time. The leader is responsible for sending out periodic heartbeats to attempt to avoid followers attempting to become leaders.

Answer 2: Contributing a bit to answer 1 - By utilizing the leader, the algorithms you mention put more pressure on one node but can reduce the number of messages exchanged to reach consensus.
The leader decides the sequence number and replicates that, so it does not need additional round-trips between the members.
There are also a few leaderless (or, in contrast, multi-leader) algorithms out there.
Each employs different techniques to reduce the round-trip of messages to deal with no dedicated node ordering everything and usually has a worst-case for contention.
I believe we'll look at some of that in the following chapters.

2. In Candidate/Ordinary Optimization, can the candidate nodes not initiate an election round? If yes, why not?

[Update] Got an answer by someone in the reading group discussion thread:

From my understanding, the candidates never initiate an election. All the responsibility is on the ordinary set.
I believe an election does not start if the FD doesn't suspect a leader, and the delta parameter would help in false-positive cases.
With the candidates acting reactively to ordinary requests, the number of false positives also reduces since it has fewer nodes starting the election.
This is only an assumption on my part, but it seems a trade-off. A more stable algorithm, but it would take longer to identify and replace a failed leader.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.