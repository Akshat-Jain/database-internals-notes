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

2. In Candidate/Ordinary Optimization, can the candidate nodes not initiate an election round? If yes, why not?

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.