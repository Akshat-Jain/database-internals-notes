# Chapter 11: Replication and Consistency

Consistency models explain the visibility semantics and system behavior in the presence of multiple copies of data.

Fault tolerance == ability to continue operating despite the failure of some of the components. This is achieved by replicating data across multiple nodes.

Data replication == Introduction of multiple copies of data across multiple nodes, so that the system can continue to operate even if some of the nodes fail.

If we have data replicated across multiple nodes, we need to ensure that the data is consistent across all the nodes. Data can go inconsistent since every write has to be propagated to all the nodes.

## Achieving Availability

Aim: high availability, low downtime.

Solution: Introduce redundancy and replication.

New problem (based on the above solution): How to keep the replicas consistent?

## Infamous CAP

CAP == Consistency, Availability, Partition tolerance.

Availability == ability of the system to serve a response for every request successfully.

Partition tolerance == ability of the system to continue operating despite a network partition in the system (nodes divided into multiple subnets).

CAP theorem == It is impossible for a distributed system to simultaneously provide all three guarantees (consistency, availability, partition tolerance).

Partition tolerance is unavoidable in a distributed system. So, we have to choose between consistency and availability == either (Consistency + Partition tolerance) or (Availability + Partition tolerance).

✨ **Side learning:**
Good resource for intuitive understanding of why both Consistency and Availability can't be achieved simultaneously in the presence of a network partition: [Arpit Bhayani's video on CAP Theorem](https://www.youtube.com/watch?v=--YbYCfMnxc)

Instead of providing a binary choice between consistency and availability, we can provide relaxed guarantees for both. We can define 2 tunable metrics - Harvest and Yield.

**Harvest and Yield:**
- Harvest:How complete is the data that is returned? (example: 99 rows being returned instead of 100 rows, 100 being the expected number of rows)
- Yield: Number of requests that were completed successfully, compared to the total number of attempted requests.

## Shared Memory

From client's perspective, the system appears to be a single machine with a single shared memory.

Register == a single unit of storage that can be read and written to.

Shared memory in distributed systems == an array of such registers.

3 types of registers:
1. Safe registers: Reads may return arbitrary values within the range of values being concurrently written to the register.
2. Regular registers: Reads can return only the value written by the most recent completed write, or the value being written by the current overlapping write operation.
3. Atomic registers: These guarantee linearizability. Everyw rite operation has a single point in time:
      1. before which every read operation returns an old value
      2. after which every read operation returns a new value.

✨ **Side learning:**
This StackOverflow answer explains the difference in a very nice way with excellent example: [Link](https://stackoverflow.com/questions/8871633/whats-the-difference-between-safe-regular-and-atomic-registers)

## Ordering

In a distributed system with, it's unclear what would be the outcome of a read(x) operation when there are concurrent read(x) and write(x) operations. It becomes even more complicated when there are replicated copies of data involved.

We define "consistency models" to explain the visibility semantics and system behavior in such situations.

## Consistency Models

We need models to define the semantics in the face of concurrent operations.

Consistency models:
- provide semantics and guarantees.
- describes what expectations the clients should have about the system behavior.
- describes which state invariants are acceptable --- OR --- describes "allowable relationships" between copies of data placed on replicas.

## Strict Consistency

- Any write by any process is instantly available for subsequent reads by any process.
- This is just theoretical. It's not possible to achieve this in practice.

## Linearizability

- Linearizability is the strongest single-object, single-operation consistency model.
- Writes becomes visible to all readers "exactly once" between the invocation and response of a write operation --- AND --- no client can observe intermediate state of a write operation.
- Effects become visible in a way that makes them appear sequential.

**Linearization point:** The point in time at which a write operation becomes visible to all readers.

**Cost of linearizability:**
- Linearizability is expensive to implement. It requires coordination between replicas.
- In concurrent programming: Achievable using compare-and-swap (CAS) operations.
- In distributed systems: Requires coordination and ordering. Achievable using consensus protocols.

## Sequential Consistency

Achieving linearizability is expensive. So, we relax the guarantees to achieve sequential consistency which still provides rather strong consistency guarantees.

- Sequential consistency: Allows ordering operations as if they were executed in some sequential order, but in the same order for all processes.

Different from linearizability in the following way: Under linearizability, the operation has to become visible within the operation duration. Under sequential consistency, the operation can become visible after the operation duration.

## Causal Consistency

Under this model:
- all processes have to see causally related operations in the same order.
- concurrent writes with no causal relationship can be observed in different orders on different processes.

In a causally consistent system, we get session guarantees for the application: monotonic reads and writes, read-your-writes, writes-follow-reads.

Implementation: Via logical clocks like vector clocks.

**Vector clocks:** Vector clock is a structure for establishing the partial ordering of events in a distributed system, detecting causality violations, and resolving divergence between the event chains.

- Processes maintain vectors of logical clocks, with one clock per process.
- Every clock starts at the initial value.
- The value of a clock is incremented every time a new event occurs (for example, a write occurs).
- On receiving a vector clock from another process, a process updates its local vector to the highest clock values per process from the received vectors.

✨ **Side learning:** Vector clocks allow us to detect whether 2 events are:
1. causally related: If the vector clock of one event (V(e1)) is less than or equal to the vector clock of the other event (V(e2)). That is, `V(e1) <= V(e2)`.
2. concurrent: If `NOT (V(e1) <= V(e2)) AND NOT (V(e2) <= V(e1))`.

Source: [Blog on Vector Clocks](https://medium.com/big-data-processing/vector-clocks-182007060193)

## Session Models

Session models == thinking about the DB state from client's perspective.

- Client should be able to reason about the state of the database based on the operations it has performed.
- Client should read its own writes. (read-own-writes)

## Eventual Consistency

Eventual consistency == updates propagate through the system asynchronously. The system eventually reaches a consistent state.

## Tunable Consistency

Eventually consistent systems usually implement tunable consistency models, using 3 variables:
1. Replication factor N: Number of nodes that will store a copy of data.
2. Write consistency W: Number of nodes that must acknowledge a write before it is considered successful.
3. Read consistency R: Number of nodes that have to respond to a read operation to succeed.

For example:
- If the system is using a consistency model having `(R + W > N)`, then the system guarantees that the read operation will always return the most recent write.
- This is because there's always an overlap between the nodes that are used for reading and writing.

Other aspects:
- Write-heavy systems can use `W = 1` and `R = N`.
- Read-heavy systems can use `W = N` and `R = 1`.

**Quorums**: A consistency level having `[N/2] + 1` nodes. Or simply put, a majority of nodes.

## Witness Replicas

Problem: Using replication increases storage costs because we need to store multiple copies of data.

Solution: Use witness replicas.

- Instead of storing copies of data on each replica, we split replicas into "copy replicas" and "witness replicas".
- Copy replicas store the actual data.
- Under normal situations, witness replicas store only a record indicating that the write has been performed.
- If number of copy replicas falls below a threshold (cannot achieve quorum), witness replicas can be promoted to copy replicas, and temporarily store the actual data.
- When the copy replicas recover, the witness replicas can be demoted back to witness replicas, OR the recovered replicas can become witness replicas.

Witness replicas are used in popular databases like Cassandra and Spanner.

## Strong Eventual Consistency and CRDTs

Strong eventual consistency:
- It is a middle group between eventual consistency and linearizability.
- This allows relaxed consistency guarantees, but not as much as eventual consistency.
- Under this model, updates are allowed to propagate late or out of order. But when all updates are propagated, conflicts between them are resolved and they are merged to form a consistent state.
- Operations need to preserve additional state that can be used to resolve conflicts. Example: Using CRDTs.

CRDTs:
- Conflict-free Replicated Data Types.
- These are special data structures that are replicated across multiple nodes.
- CRDTs are useful in eventually consistent systems, since it allows the replica states to temporarily diverge.
- CRDTs allow us to reconstruct the state of the system by merging the states of all the replicas.

Simplest example of CRDT: Operation-based Commutaive Replicated Data Types (CmRDTs). For CmRDTs to work, the operations need to be:
1. Side-effect free.
2. Commutative: It shouldn't matter whether x is merged into y or y is merged into x.
3. Causally ordered

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

1. A real life example of consistency came up at work in the context of a batch job that has multiple readers pulling from a GCS bucket: https://cloud.google.com/storage/docs/consistency
