# Chapter 13: Distributed Transactions

- Distributed transactions are a way to ensure that a set of operations are performed atomically across multiple nodes in a cluster.
- Transaction atomicity == Either all the results of a transaction become visible, or none of them become visible.
- To ensure atomicity, transactions need to be recoverable. (so that they can be rolled back if they fail)

# Making Operations Appear Atomic

To make multiple operations appear atomic, we use algorithms belonging to "atomic commitment" category.

Atomic Commitment:
- A transaction will not commit if any of the participants vote against it.
- We discuss two algorithms for atomic commitment:
  - Two-phase commit (2PC)
  - Three-phase commit (3PC)

## Two-Phase Commit

- Two-phase commit (also known as 2PC) has 2 phases.
- Phase 1: Prepare
  - Coordinator notifies all participants about the new transaction by sending the proposed value.
  - Participants respond/vote with a yes/no answer.
  - If any participant says no, the coordinator aborts the transaction.
  - If all participants say yes, the coordinator moves to phase 2.
- Phase 2: Commit/Abort
  - Coordinator sends a commit request to all the participants.
  - Participants respond with an acknowledgement.
  - If any participant fails to respond, the coordinator aborts the transaction.
  - If all participants respond, the coordinator commits the transaction.

**Cohort Failures in 2PC**

Following cases:
1. Cohort fails before voting: Coordinator will abort the transaction, as it needs all votes to be positive.
2. Cohort fails after voting:
  - If it had voted "YES": When it recovers, it has to find out the actual outcome of the vote before serving values correctly, as the coordinator might have aborted the commit due to other cohorts' decisions.
  - If it had voted "NO": It doesn't have to do anything, since it wouldn't have had precommitted the transaction anyway, so it can release any locks it had acquired.

**Coordinator Failures in 2PC**

Following cases:
1. Coordinator fails before proposing a value: Nothing needs to be "fixed" as the transaction was never proposed to the cohorts.
2. Coordinator fails after collecting votes: 
  - If none of the cohorts have received any result from coordinator: Cohorts are left blocked. This is called the "blocking" issue with 2PC.
  - If a subset of the cohorts have not received the result: They contact peers, since the decision (if received) would be unanimous.

**Databases that use 2PC:** MySQL, PostgreSQL, MongoDB, etc.

✨ **Side learning 1:** An AWESOME blog which explains the failure cases of 2PC in a very detailed way: [Link to blog](https://www.alibabacloud.com/blog/tech-insights---two-phase-commit-protocol-for-distributed-transactions_597326)

I highly recommend checking out the above blog. It helped me understand the different failure cases of 2PC in a much better way.

✨ **Side learning 2:** I was confused on how or why PostgreSQL would need 2PC as that's meant for distributed transactions. Here's why:
- https://www.endpointdev.com/blog/2006/05/postgresql-supports-two-phase-commit/
- https://www.postgresql.org/docs/10/sql-prepare-transaction.html

## Three-Phase Commit

Problems with 2PC: The "blocking" issue.

Solution: Introduce timeouts in 2PC.

Problem with introducing timeouts: The "inconsistency" issue.

**What is the "inconsistency" issue?** If the coordinator goes down when it receives a failure from a participant and intends to send an abort request to other participants, the corresponding distributed transaction is supposed to fail. However, some participants may commit it due to timeout.

Solution: Introduce a third phase in 2PC.

**3 phases of 3PC:**
- Phase 1: Propose
  - Coordinator sends a proposed value to all participants, and collects their votes.
  - If any participant votes no, the coordinator aborts the transaction.
  - If all participants vote yes, the coordinator moves to phase 2.
- Phase 2: Prepare
  - Coordinator notifies participants about the vote results.
  - If any participant votes no, the coordinator aborts the transaction by sending an abort request to all participants.
  - If all participants vote yes, the coordinator sends a "prepare" request to all participants. Think of "prepare" request as coordinator telling the participants that everyone has voted yes, so you should prepare yourself to commit the transaction. Next, we move to phase 3.
- Phase 3: Commit/Abort
  - If the coordinator receives a "prepared" acknowledgement from all participants, it sends a "commit" request to all participants. If the coordinator fails before sending the commit request, the participants will commit the transaction by themselves after the timeout as they are aware that everyone has voted yes.
  - If any participant fails to respond with the "prepared" acknowledgement, the coordinator aborts the transaction by sending an abort request to all participants. If the coordinator fails before sending the abort request, the participants will abort the transaction by themselves after the timeout.

**Coordinator Failures in 3PC**

- The worst-case scenario for 3PC is a network partition.
- In this case, the coordinator and some of the participants are on one side of the partition, and the remaining participants are on the other side.
- The coordinator and the participants on one side of the partition can communicate with each other, but they cannot communicate with the participants on the other side of the partition.
- This results in a split-brain situation, where the participants on one side of the partition proceed with a commit, while the participants on the other side of the partition abort the transaction.
- This leaves the system in an inconsistent state.

**Why is 3PC not used in practice?**
- Larger message overhead than 2PC.
- Does not work well in the presence of network partitions.

✨ **Side learning:**
- This StackOverflow answer explains the difference between 2PC and 3PC in a very good way: https://stackoverflow.com/a/55834191
- This blog by TiKV is also good: https://tikv.org/deep-dive/distributed-transaction/distributed-algorithms/

## Distributed Transactions with Calvin

- Calvin is a transaction scheduling system in distributed systems.
- It has 4 components, in order of a transaction lifecycle:
  - Sequencer: The sequencer determines the order in which transactions are executed, and establishes a global transaction input sequence.
  - Scheduler: The scheduler is responsible for scheduling transactions for execution.
  - Worker: The worker thread is managed by the scheduler, and is responsible for executing a transaction.
  - Storage: Worker threads read and write data from the storage layer.

✨ **Side learning:**

- The core idea of calvin is as follows: "When multiple machines need to agree on how to handle a particular transaction, they do it outside of transactional boundaries — that is, before they acquire locks and begin executing the transaction."
- Source: https://medium.com/@balrajasubbiah/calvin-fast-distributed-transactions-for-partitioned-database-systems-17b1aee15e09

I feel the above blog is an easier-to-understand text than the book itself. I highly recommend reading the above blog.

## Distributed Transactions with Spanner

- Spanner is a distributed database system developed by Google.
- It uses 2PC on consensus groups per partition/shard.

Spanner offers 3 operation types:
1. Read-only transactions: These are lock-free transactions that can be executed on any replica.
2. Read-write transactions: These require locks, pessimistic concurrency control, and presence of leader replica.
3. Snapshot reads: These are lock-free reads that can be executed on any replica.

**As mentioned above, Spanner uses 2PC on "consensus groups per partition/shard". Why?**
- 2PC algorithm requires presence of all participants to commit a transaction. This hurts availability of the system.
- Solution: Spanner uses "Paxos groups" instead of individual nodes as the participants in 2PC. This allows 2PC to continue even if some members of the Paxos group are down. This improves availability of the system.

Paxos is covered in greater detail in Chapter 14, so hopefully this would become more intuitive after reading that chapter.

## Database Partitioning

Partitioning: A logical division of data into smaller segments.

Example:
- A database with 1000 users can be partitioned into 10 partitions, each containing 100 users.
- A database with 1000 locations can be partitioned on the zip code.

How do partitions work?
- The individual segments can reside on different replica sets. This is called "sharding". Each replica set acts as a source of truth for a subset of the data.
- Database systems have to store a routing key, which is used to determine which partition a read/write request should be routed to. This is generally done by mapping a hash value to a node ID (which is essentially the partition).

Problem: Adding and removing nodes from a cluster would require rebalancing of the data (depending on the partitioning scheme / hash function used).

Solution:
- Use consistent hashing.
- Each partition / node is assigned a hash value which is mapped to a "ring" of hash values.
- This helps because a change in the number of partitions / nodes would only affect the partitions / nodes immediately before and after the changed partition / node.

## Distributed Transactions with Percolator

- Percolator is a library that implements a transactional API on top of the distributed database Bigtable.
- Each transaction has to consult a clusterwise timestamp oracle twice:
  - to get timestamp for start of transaction
  - to get timestamp during commit
- Writes are buffered and committed using client-driven 2PC.

## Coordination Avoidance

Invariant Confluence: Property of a system that ensures that 2 different states --- which adhere to invariant rules, but are diverged --- can be merged into a single state, and the invariant rules would still hold in the merged state.

**RAMP (Read-Atomic Multi Partition)**
- RAMP transactions use MVCC and metadata of in-flight operations to fetch missing updates from other nodes.
- Writes in RAMP are made visible using 2PC.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.
