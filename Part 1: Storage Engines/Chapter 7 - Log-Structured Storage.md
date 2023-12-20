# Chapter 7: Log-Structured Storage

In this chapter, we will focus on LSM Trees.

## LSM Trees

LSM Tree uses buffering + append-only storage to achieve sequential writes.

**Why is it called LSM Tree?**
- The name comes from log-structured filesystems, which write all modifications on disk in a log-like file.
- "Merge" in LSM Trees == Tree contents are merged using an approach like merge-sort.

**Write performance and throughput**
- Files are immutable.
- So, insert/update/delete operations don't need to locate data on disk.
- This improves write performance and throughput.

**Reads**
- Reads and writes don't intersect, thanks to the design.
- This simplifies concurrent access.

## LSM Tree Structure

- Memtable: Memory-resident mutable component to buffer data records, and serves as a target for read and write operations.
- Write-Ahead Log (WAL): This is required to guarantee durability of data records. Data records are appended to the log before committing to the memtable.
- Disk-resident components: Used only for reads. We will call these "tables" in this chapter (as a shortcut for disk-resident tables).

### Two-component LSM Tree

This has two components:
- Memtable
- Only one disk-resident component, comprised of immutable segments.

Memory-resident tree contents are flushed on disk in parts.

### Multicomponent LSM Trees

Alternative design: Having more than one disk-resident table.

**Compaction:** To keep the number of disk-resident tables under control, LSM Trees periodically merge the disk-resident tables. This process is called compaction.

Memtable:
-  In this case, entire memtable is flushed to disk, because we are allowed to have more than one disk-resident table. This simplifies compaction process.
- Memtable flushes can be triggered by size, or periodically.

**Memtable Flush - The Process:**
- **Memtable Switch:**
  - A new memtable is allocated.
  - The switch is performed atomically to ensure consistency.
  - The new memtable becomes the target for all new writes.

- **Flushing State:**
  - The old memtable, now in a flushing state, retains its contents.
  - It remains available for reads until its contents are fully flushed to disk.

- **Disk-Resident Table:**
  - The flushed data is written to disk as a new table (e.g., SSTable).
  - The old memtable is discarded.

- **Availability for Reads:**
  - Once fully flushed, the disk-resident table becomes available for reads.

- **After Flushing:**
  - The write-ahead log can be trimmed.
  - The log section holding operations-applied-to-the-flushed-memtable can be discarded.

The responsibilities of the different components involved in this process are very nicely summarised in the book. I am copy pasting it here for reference:

- **Current Memtable:** Receives writes and serves reads.

- **Flushing Memtable:** Available for reads.

- **On-Disk Flush Target:** Does not participate in reads, as its contents are incomplete.

- **Flushed Tables:** Available for reads as soon as the flushed memtable is discarded.

- **Compacting Tables:** Currently merging disk-resident tables.

- **Compacted Tables:** Created from flushed or other compacted tables.

## Updates and Deletes

Updates: Just append, and let it reconcile the redundant records during reads.

Deletes: We need to use tombstones to mark the deleted records.

**What is a tombstone?**
- It is a special "delete entry".
- Tombstones help prevent resurrection of deleted records during reads.
- We can also have range tombstones, which are used to mark deletion of a range of records.

## LSM Tree Lookups

**Merge-Iteration**

- Contents of disk-resident tables are sorted.
- This allows us to use multiway merge-sort algorithm (which uses a priority queue such as a min-heap).

Since the disk-resident tables are sorted, we can be sure that only the latest version for a key would be eventually included in the merged result.

**Reconciliation**

To be able to reconcile the redundant records, we need to have a way to identify the latest version of a record (the version that has to take precedence).

Data records hold metadata necessary for this, such as timestamps.

## Maintenance in LSM Trees

- Periodic compaction is needed to reduce the growing number of disk-resident tables.
- Compaction picks multiple disk-resident tables, iterates over their entire contents using the (merge + reconciliation) algorithms, and writes out the results into the newly created table.

**Leveled Compaction:**

- Diks-resident tables separated into levels with target sizes.
- Level-0 tables from flushed memtable contents.
- Higher-level tables created through compaction of overlapping key ranges.
- Exponential size growth between levels, freshest data in lower levels.

**Size-Tiered Compaction:**

- Tables grouped by size, not levels.
- Level 0 holds smallest tables, incremented recursively with compaction.
- Larger tables promoted, smaller tables demoted.

**Other Compaction Strategies:**

- Apache Cassandra's time window compaction strategy for time-series workloads.
- Considers write timestamps and allows dropping files with expired data without rewriting contents.

## Read, Write, and Space Amplification

We have 3 problems when storing data on disk in an immutable manner:
- **Read amplification:** Resulting from a need to address multiple tables to retrieve data.
- **Write amplification:** Caused by continuous rewrites by the compaction process.
- **Space amplification:** Arising from storing multiple records associated with the same key.

## RUM Conjecture

RUM == Read, Update, and Memory overheads

RUM Conjecture states that reducing 2/3 of the overheads leads to the third one increasing.

- B-Trees excel in read efficiency but incur higher update and memory overheads.
- LSM Trees optimize updates and tend to have lower memory overhead but may have higher read overheads.

## Implementation Details

This section covers how the disk-resident tables are implemented.

**Sorted String Tables**

Sorted String Tables == SSTables == data records are sorted in the key order.

SSTables generally have 2 components:
1. Index files: Holds keys and offsets of the corresponding data records in the data files. Implemented using B-Trees or hash tables.
2. Data files: Holds the data records in key order.

## Bloom Filters

The root cause of "Read Amplification" problem in LSM Trees is that we need to check multiple disk-resident tables for a complete read operation. This can be improved by using Bloom Filters.

- Bloom Filters are probabilistic data structures that can tell us whether a key is present in a set or not.
- They either tell us that a key is either "absent" or "possibly present".
- A bloom filter uses a bit array and a set of hash functions.
- A "set bit" / "1 bit" means that the key is "possibly present". A "clear bit" / "0 bit" means that the key is "absent".

For example, if we have a bloom filter with 10 bits, and 2 hash functions, and we have the following:

HashFunction1("key1") = 3
HashFunction2("key1") = 7

Inserting "key1" into the bloom filter would result in the following:
```
[0, 0, 0, 1, 0, 0, 0, 1, 0, 0]
```

Similarly, for "key2", let's say we have:

HashFunction1("key2") = 1
HashFunction2("key2") = 4

Inserting "key2" into the bloom filter would result in the following:
```
[0, 1, 0, 1, 1, 0, 0, 1, 0, 0]
```

Now, for "key3", let's say we have:
HashFunction1("key3") = 1
HashFunction2("key3") = 5

To check if "key3" is present in the bloom filter, we would do the following:
- Check if the bit at index 1 is set. It is set, so we move on to the next step. This is because we have a "possibly present" result, since it's not guaranteed that the bit was set because of "key3".
- Check if the bit at index 5 is set. It is not set, so we can conclude that "key3" is not present in the bloom filter.

In the context of LSM Trees, each disk-resident table can have a bloom filter attached to it, which can be used to check if a key is "possibly present" in the table or "not present". This can help us avoid reading the table if the key is not present in it.

## Skiplist

Skiplist is like a linked list, but with additional pointers that allow us to skip some elements. A skiplist consists of a series of nodes of a different height, building linked hierarchies allowing to skip ranges of items.

For example:

```
|  | --------------------------------> |    |
|  | ----------> | 5 | --------------> | 10 |
|  | -> | 3 | -> |   | -> | 7 | -----> |    |
```

If we have to search for key 7:
1. We follow the pointer on the highest level to node 10.
2. Since search key (7) is smaller than 10, we follow the next-level pointer from the previous node, that is, to node 5.
3. We follow the highest level pointer from node 5 to node 10.
4. Since search key (7) is smaller than 10, we follow the next-level pointer from the previous node, that is, to node 7.

## Disk Access

- LSM Trees leverage the page cache for efficient disk access, and in-memory data is made immutable, eliminating the need for locks.
- Reference counting ensures that actively used pages are not removed from memory during compaction.
- Unlike traditional structures, LSM Trees allow data records to span across page boundaries.

## Compression

- Compression in LSM Trees: 
  - Utilizing compression for storage efficiency.
- Handling Compressed Pages:
  - Addressing challenges due to smaller, non-aligned sizes.
  - Indirection layer for offset and size mapping.
- Read Process and Compression Information:
  - Sequential appending of compressed pages during compaction and flush.
  - Storage of compression information in a separate file segment.
  - Lookup of compressed page offset and size during reads, followed by decompression.

## Unordered LSM Storage

Unordered LSM Storage == Store data records in an unordered manner.

They don't require a separate log file, and allow us to reduce cost of writes by storing data records in the order of insertion.

## Bitcask

Bitcask is an example of unordered LSM storage.
- It does not use memtables for buffering.
- It stores data records directly in logfiles.
- It uses a data structure called "keydir" that holds references to the latest data records for each key. This is done to make the values searchable.
- During a "write", the data record is appended to the logfile, and the keydir is updated.
- During compaction, contents of the logfiles are merged into a new logfile, and the keydir is updated with new pointers to the relocated data records.

Advantages: Simple. Great point query performance.

Disadvantages: Does not support range queries.

## WiscKey

WiskKey keeps the keys sorted in LSM trees, and data records in unordered append-only files called vLogs (value logs).

This helps compaction process to be significantly more efficient, as the keys are typically much smaller than the data records.

## Concurrency in LSM Trees

We have the following synchronization points in LSM Trees:
- Memtable switch: After this, all writes go to the new memtable, and the old memtable is available for reads.
- Flush finalization: Replaces the old memtable with a flushed disk-resident table.
- Write-ahead log truncation: Discards the log section holding operations-applied-to-the-flushed-memtable.

## Log Stacking

Log stacking == Stacking multiple log-structured systems on top of each other.

This can run into problems like write amplification, fragmentation, and poor performance.

## Flash Translation Layer

Log-structured storage is used on SSDs to amortize I/O costs by writing data in batches, which result in smaller number of operations, and hence lesser number of times the garbage collection is triggered.

This also helps SSD wear leveling because of the reduced number of writes.

## Filesystem Logging

Having multiple layers of log-structured storage on top of each other could cause writing records that have sizes not aligned with the underlying hardware page size.

This results in fragmentation.

To reduce this interleaving, some database vendors recommend keeping the logs on a separate device.

## LLAMA and Mindful Stacking

LLAMA == Latch-free, log-structured, access-method aware storage subsystem

Stacking LLAMA and Bw-Tree demonstrates the benefits of coordination between software layers because of the awareness of the underlying storage subsystem, and hence logical node consolidation.

Conclusion: Stacking, when considered carefully, yields efficiency benefits.

## Open-Channel SSDs

Alternative to stacking software layers is to skip them all and directly use the hardware.

## Pending Doubts

NA at the moment.

## Things to Read

NA at the moment.

## Reading group discussion

NA at the moment.
