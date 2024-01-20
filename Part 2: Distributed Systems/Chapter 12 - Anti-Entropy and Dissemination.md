# Chapter 12: Anti-Entropy and Dissemination

More than the data records, we want the cluster-wide metadata changes to be propagated to all the nodes in the cluster quickly and reliably. This chapter discusses the anti-entropy and dissemination protocols that are used to achieve this.

Cluster-wide metadata: Which nodes are joining or leaving, node states, failures, schema changes, etc

3 ways:
1. Notification broadcast from one process to all other processes
2. Periodic peer-to-peer information exchange
3. Cooperative broadcast, where receivers help forward the message to other nodes

Entropy == measure of disorder in the system == message of state divergence between nodes (in distributed systems context)

Anti-entropy == attempting to lower the convergency time bounds in eventually consistency systems.

Anti-entropy measures can be of 2 ways:
1. Background processes: Using structures like Merkle trees to detect divergence and then using a background process to reconcile the differences
2. Foreground processes: Piggybacking on the normal read/write operations to detect divergence and reconcile the differences. Example: hinted handoff, read-repair, etc

## Read Repair

- Detects divergence during read operations
- Coordinator node sends read request to all replicas. If replicas send back different values, then the coordinator node picks the most recent value and sends it to all replicas which reported older values.

2 types of read repair:
1. Blocking read repair: Client request has to wait until the coordinator node has repaired all the replicas == we sacrifice availability for consistency
2. Asynchronous read repair: Client request does not have to wait for the coordinator node to repair all the replicas, coordinator does that in the background asynchronously == we sacrifice consistency for availability

## Digest Reads

This is an optimization to the read repair process.
- Coordinator issues only one full read request to one replica, and issues "digest requests" to all other replicas.
- Digest request returns a hash of the data (instead of the actual data), which is much smaller than the actual data.
- Coordinator compares the hash values from all replicas.
  - If they are the same, then the replicas are in sync.
  - If they are different, then the coordinator issues a full read repair request to all replicas which reported a different hash value.

## Hinted Handoff

- This is a write-repair mechanism.
- If a node fails to acknowledge a write request, then a special record (called hint) is stored by either the coordinator or by one of the replicas. This hint is then "replayed" to the failed node when it comes back online.

## Merkle Trees

Problem with read-repair: They only fix inconsistencies on the data that was queried. We still need a background process to fix inconsistencies on the data that is not being queried.

We need an efficient way to detect inconsistencies in the data. Merkle trees are one such way.

Merkle trees:
- They are composed of a hashed representation of the data, structured in the form of a tree of hashes.
- Lowest level of the tree contains the hashes of the data records.
- Higher levels of the tree contain the hashes of the lower-level hashes.

Good aspect: It's efficient to detect inconsistencies in the data, by comparing the hashes of the data records (going from the root node to the leaf nodes).

Bad aspect: Any write operation would cause the hashes to be re-computed recursively for the entire subtree. This is expensive.

## Bitmap Version Vectors

- Each node keeps a per-peer log of operations that have occurred locally or were replicated.
- During anti-entropy, logs are compared, and missing data is replicated to the target node.

## Gossip Dissemination

Objective: To spread information from one process to the rest of the cluster.

Some terms:
- Infective process: The process that holds the information to be spread.
- Susceptible process: The process that has not yet received the information.
- Removed process: The infected process that are not spreading information anymore after a period of spreading the information.

Since we're just spreading information randomly, and relying on probabilities for the information to reach all nodes of the cluster, we need a metric that helps us figure out when we can stop spreading the information. This is done using "loss of interest" function == the degree with which we've lost interest in spreading the information.

## Gossip Mechanics

Message redundancy == metric that measures the overhead incurred due to repeated delivery. Redundancy is inevitable since it's probabilistic.

Latency: Amount of time it takes for the system to reach convergence. This is not the same as the time it takes for the information to reach all nodes. This is because information would still continue to spread (for some time) even after all nodes have received the information.

- Processes periodically select `f` peers at random. `f` is a configurable parameter called fanout.
- Then they share the information with the selected peers.
- Over time, nodes notice that they are receiving the same information multiple times.
- "Interest loss" is calculated using multiple approaches:
  - probabilistically (computing probability of stopping the propagation for each process on every step)
  - using a fixed threshold (counting the number of received duplicates, and stopping when the threshold exceeds a certain value)

## Overlay Networks

Problem with random selection of nodes: It involves a lot of redundant communication. 

Solution: We can reduce this by using a structured overlay network.

- We construct an overlay network of nodes, and then use the overlay network to spread information.
- Nodes in the system can form "spanning trees".
- Information is then spread using the spanning trees.

## Hybrid Gossip

- Push/lazy-push multicast trees (also called Plumtrees) make a trade-off between the overhead of gossip and the latency of spanning trees.
- Nodes send full messages to a subset of their peers, and send only the "message ID" to the rest of the peers.
- If the receiving node has not seen the message before (corresponding to the message ID), then it requests the full message from its peers.

## Partial Views

Broadcasting messages to all known peers is expensive. We can reduce this by using partial views.

- We maintain a partial view of the cluster, and only send messages to the nodes in the partial view.
- The partial view is periodically refreshed using gossip.

Example: Hybrid Partial View (HyParView) protocol.

✨ **Side learning:**

Really good article that explains HyParView: https://www.bartoszsypytkowski.com/hyparview/

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.