# Chapter 6: B-Tree Variants

In this chapter, we discuss some of the variants of B-Trees.

## Copy-on-Write

Copy-on-Write B-Trees use copy-on-write technique to ensure data integrity during concurrent operations.

- Whenever a page has to be updated, its contents are copied, the copy is updated. This way a paralle tree hierarchy is created.
- Old/Original/Initial versions of tree still remain accessible to readers (that happen concurrently with the writers).
- Once the writer is done with the update, it updates the root pointer to point to the new tree.

Downsides:
- Requires more space.
- Requires more processor time.

Advantages:
- Readers require no synchronization.
- Writers are not blocked by readers.
- Crash cannot leave the tree in a corrupted state, because we point to the "copy" only after the update is complete.

## Implementing Copy-on-Write: LMDB

- LMDB == Lightning Memory-Mapped Database == a storage engine that uses copy-on-write.
- LMDB is a key-value store used by OpenLDAP.
- LMDB holds only 2 versions of the tree at any given time.

## Abstracting Node Updates

Few ways to represent a node in-memory:
1. Direct Access to Cached Version or Native Memory Manipulation
2. Materializing Nodes into Language-Native Structures
3. Wrapper Object Approach in Managed Memory Languages

## Lazy B-Trees

Lazy B-Trees use update-friendly in-memory structures to buffer updates and propagate them with a delay.

## WiredTiger

WiredTiger is the default storage engine for MongoDB.

B-Tree nodes are materialized in memory upon paging in, storing updates until a flush is triggered.

**Page Structure:**
- In WiredTiger, a clean page initially consists of an index constructed from the on-disk page image.
- Updates are saved into the update buffer, accessed during reads to merge with the original on-disk page contents.

**Reconciliation Process:**

- Before in-memory pages are persisted, they undergo a reconciliation process.
- During flush, update buffer contents are reconciled with page contents and overwritten on disk; large pages are split if needed.

**Implementation Details:**:

- Update buffers in WiredTiger are implemented using skiplists, offering a concurrency profile advantage.
- Clean and dirty pages both have in-memory versions referencing a base image on disk, with dirty pages additionally having an update buffer.

## Lazy-Adaptive Tree

- LA-Tree groups nodes into subtrees and attaches an update buffer to each subtree for batching operations.
- Update buffers track operations on the subtree top node and its descendants.

**Insertion:**

- When inserting a data record, a new entry is added to the root node update buffer.
- When the buffer is full, changes are copied and propagated to buffers in lower tree levels recursively until reaching leaf nodes.

**Batched Updates for Efficiency:**

- LA-Tree optimizes update time by performing batched updates, minimizing disk accesses and structural changes.
- Splits and merges propagate in batches to higher levels, reducing the need for separate updates on individual pages.

## FD-Trees

FD-Trees == Flash-Drive Trees

Idea here is to group the "updates that are targeted different nodes" together, by using append-only storage and merge processes.

**FD-Tree Design:**

- Small mutable head tree and multiple immutable sorted runs.
- Minimizes the surface area where random write I/O is required.

**Insertion:**

- Head tree acts as a buffer in-place for any updates.
- When the head tree is full, the contents are transferred to the immutable run.
- Runs are merged with the next level if they exceed a threshold.

## Fractional Cascading

Goal: To reduce the cost of locating an item in the cascade of sorted arrays.

Approach: Build bridges between the different levels, to facilitate starting the search from the closest match on the previous level.

For example, if we have multiple sorted arrays:
```
  A1 = [12, 24, 32, 34, 39]
  A2 = [22, 25, 28, 30, 35]
  A3 = [11, 16, 24, 26, 30]
```

We can bridge the gaps by pulling every alternate element to an upper level:
```
  A1 = [12, 24, 25, 30, 32, 34, 39]
  A2 = [16, 22, 25, 26, 28, 30, 35]
  A3 = [11, 16, 24, 26, 30]
```

## Logarithmic Runs

An FD-Tree == fractional cascading + logarithmically sized sorted "runs"

`logarithmically sized sorted "runs"` == immutable sorted arrays with sizes increasing by a factor of k, which are created by merging the previous level with the current one.

**Addressing Items in Sorted Runs:**

FD-Trees adapt fractional cascading for addressing items in sorted runs.

**Reduced Search Cost:**

Head elements from lower levels act as pointers to higher levels.
Reduces search cost in lower-level trees by leveraging previous searches from higher levels.

## Bw-Trees

3 problems that we need to solve:
1. Write amplification
2. Space amplification
3. Complexity in solving concurrency problems + dealing with latches

Solution: Bw-Trees!

**Bw-Tree (Buzzword-Tree):**
- Uses append-only storage to batch updates to various nodes that are linked together in chains, optimizing write operations.
- Achieves a lock-free structure by linking nodes into chains and employing an in-memory data structure that enables pointer installation between nodes with a single compare-and-swap operation.

## Update Chains

Bw-Tree adopts an update chain strategy, separating base and delta nodes in a linked list, allowing individual updates without rewriting nodes, but requires traversing all delta nodes during reads to reconstruct the node state, akin to LA-Trees.

## Taming Concurrency with Compare-and-Swap

Algorithm to update a Bw-Tree node:
1. Locate the target logical leaf node by traversing the tree from root to leaf, utilizing virtual links in the mapping table to access target base nodes or the latest delta nodes in the update chain.
2. Create a new delta node with a pointer to the identified base node (or the latest delta node) from step 1.
3. Update the mapping table with a pointer to the newly created delta node, ensuring the algorithm tracks the most recent modifications in the update chain.

## Structural Modification Operations

Bw-Tree also requires splits and merges. We call them Structural Modification Operations (SMOs).

**Split SMOs in Bw-Tree:** Start by consolidating logical contents of the splitting node, creating a new page to the right of the split point. Two-step process:
1. Split: Append a special split delta node to the splitting node with a midpoint separator key and a link to the new logical sibling node.
2. Parent Update: Add the new sibling node as a direct child to the parent node, bypassing the need to traverse through the splitting node, completing the split process.

**Merge SMOs in Bw-Tree:**
1. Remove Sibling: Create a special remove delta node appended to the right sibling, marking it for deletion and initiating the merge process.
2. Merge: Generate a merge delta node on the left sibling, pointing to the contents of the right sibling and integrating it logically.
3. Parent Update: Remove the link to the right sibling from the parent node to complete the merge, making the right sibling's contents accessible from the left sibling.

## Consolidation and Garbage Collection

Epoch-Based Reclamation technique:
1. Bw-Trees use an Epoch-Based Reclamation technique to separate threads that might have encountered a specific node from those that couldn't.
2. Nodes and deltas removed from the mapping table during consolidations are preserved until every reader that started during the same or earlier epoch finishes, ensuring safe garbage collection afterward as later readers are guaranteed not to have seen those nodes.

## Cache-Oblivious B-Trees

**Cache-Oblivious B-Trees:**
- Asymptotically optimal B-Tree performance without tuning parameters, adapting to diverse memory hierarchies.

**Two-Level Memory Hierarchy:**
- Traditional B-Trees for disk-resident pages and a limited-size page cache, with data transfer in blocks.

**Simplicity in Parameterization:**
- Design simplicity with only two parameters (page cache and disk) and cache-aware block transfers.

**Cache-Oblivious Algorithms:**
- Optimal performance without platform-specific parameters, maximizing work at the highest cache level.

## van Emde Boas Layout

**van Emde Boas Layout and Dynamic Operations:**

- Cache-oblivious B-Trees use van Emde Boas layout for a static B-Tree and a packed array structure for dynamic operations.
- The van Emde Boas layout ensures contiguous memory storage, and the packed array, with density-based gaps, allows efficient inserts and updates.

**Challenges and Future Considerations:**

- On-disk structures mimic main memory layouts, but challenges include potential impacts from abstracted cache loading and comparable block transfer complexity to cache-aware counterparts.
- Future improvements may arise with the prevalence of more efficient nonvolatile byte-addressable storage devices.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

1. Andy Pavlo has a really good paper on implementing Bw-trees: [Link](https://www.cs.cmu.edu/~huanche1/publications/open_bwtree.pdf)
