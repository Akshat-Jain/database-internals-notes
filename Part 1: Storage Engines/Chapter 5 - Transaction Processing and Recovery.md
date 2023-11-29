# Chapter 5: Transaction Processing and Recovery

Transaction: Indivisible unit of work that either succeeds completely or fails completely.

✨ **Side learning:** [Transactions are not locks](https://www.benburwell.com/posts/transactions-are-not-locks/)

**ACID properties**

- Atomicity: All or nothing. Either all the operations in a transaction succeed, or none of them do.
- Consistency: The database is always in a valid state. This is the most weakly defined property, because it's the only property that is controller by the user and not by the database itself.
- Isolation: Multiple transactions that are running concurrently should not interfere with each other (as if no other transaction is running at the same time).
- Durability: Once a transaction has completed successfully, the changes it has made to the database are permanent. This allows the database to recover from failures.

A bunch of different components come together to implement transactions. These are:
1. Transaction manager: Coordinates, schedules, and tracks transactions.
2. Lock manager: Prevents concurrent access to data. If the lock is exclusive, then only one transaction can access the data at a time, and other transactions have to wait for the lock to be released. If the lock is shared, then multiple transactions can read the data at the same time.
3. Page cache: Intermediary between the database's storage engine and the persistent storage (disk). It manages caching of pages in memory.
4. Log manager: Writes all the changes to the database to a log file. This is used for recovery in case of a crash.

## Buffer Management

Goal is to reduce the number of accesses to the disk, because disk access is slow. For this, pages are cached in memory.

Virtual Disk: A read from virtual disk accesses physical storage (disk) only if the page is not in the page cache. If the page is in the buffer pool, then it is read from there.

Virtual Disk / Page Cache / Buffer Pool: All these terms are used interchangeably. We will use the term page cache for the rest of the notes.

Some other phrases that are widely used:
1. "Paged in": When a page is read from the disk (because it is not in cache), it is said to be paged in.
2. Dirty page: A page that has been modified in memory but not yet written to disk.
3. "flushed to disk": When a page is written to disk, it is said to be flushed to disk.
4. "evicted": When a page is removed from the page cache, it is said to be evicted.

## Caching Semantics

When storage engine requests the page:
1. If the page is in the page cache, then it is returned.
2. If the page is not in the page cache, then it is read from the disk and returned.

Until the storage engine hands over the page to the page cache, the page is said to be "refenced". Once the page is handed over to the page cache, the page is said to be "unreferenced".

The page cache can be told to avoid evicting a page by "pinning" it.

## Cache Eviction

Page cache has a limited capacity. When the page cache is full, it needs to evict some pages to make space for new pages.

1. Dirty pages need to be flushed to disk before they can be evicted.
2. Referenced pages cannot be evicted while some thread is still using them.

Postgres has a background flush writer that cycles over the dirty pages that are likely to be evicted, and flushes them to disk. This is because triggering a flush on every eviction might be bad for performance.

Durability: If the database crashes, all the dirty pages that have not been flushed to disk are lost. To ensure durability, the checkpoint process coordinates the page cache and the write-ahead log (WAL).

Which log records can be discarded from WAL? Only the ones which correspond to operations applied to cached pages that have been flushed to disk. This is because the changes are already on disk, so they can be recovered from there in the case of a crash.

## Locking Pages in Cache

Pinning: Locking pages in the cache so that they cannot be evicted.

Which pages would we want to pin? The ones that we know we will need to access again soon.

In the context of B-Trees, we would want to pin the pages that are on the higher-levels of the B-Tree, because they are more likely to be accessed again soon. The higher-levels have exponentially lesser nodes than the lower-levels, so it is feasible to cache them. In the case of splits or merges, the changes might propagate to the higher-levels, so having the higher-levels in cache is useful and help reduce the number of disk accesses.

Another good aspect here is that multiple writes to the same page can be batched together, and the page can be flushed to disk only once. For eaxmple, if 10 splits or merges propagate to the root node, then all of them can be batched together and the page can be flushed to disk only once.

**Prefetching and Immediate Eviction**

Prefetching: In some cases, when a page is read from disk, we can already tell which pages would be likely to be accessed soon. Example, we can load the next leaf nodes during a range scan.

Immediate Eviction: In some cases, we can evict a page from the cache as soon as it has been read from disk. Example, if a maintenance process loads the page, it's unlikely that the page would be useful for any future queries, so it can be evicted immediately.

## Page Replacement

Page replacement: When the page cache is full, and a new page needs to be loaded, then some existing page needs to be evicted to make space for the new page.

Page replacement policy / Eviction policy: The algorithm that decides which page to evict.

In an ideal world, we would want to evict the pages that would not be touched for the longest time in the future. But, this is not possible to predict.

Belady's Anomaly: Using larger cache might not always reduce the number of evictions. Belady's anomaly shows that increasing the number of pages might still increase the number of evictions in some cases, if the eviction policy is not good.

We discuss some eviction policies below.

### FIFO and LRU

FIFO == First In First Out
LRU == Least Recently Used

FIFO eviction policy: Evict the page that was loaded first.
LRU eviction policy: Evict the page that was accessed least recently.

LRU-K eviction policy: Evict the page that was accessed least recently, but taking only the last K accesses into account.

### Clock

Clock sweep algorithm holds references to pages and an "access bit" in a circular buffer.

The clock hand moves around the clock, and the page that it points to is a candidate for eviction.

1. If access bit is 1, and the page is unreferenced, then the access bit is set to 0 and the clock hand moves to the next page.
2. If access bit is 0, then the page becomes a candidate and is scheduled for eviction.
3. If the page is currently referenced, don't change the access bit.

| Access Bit | Is Referenced | Action                                                |
|------------|---------------|-------------------------------------------------------|
| 1          | Unreferenced  | Set access bit to 0 and move the clock hand to the next page  |
| 0          | -             | Page becomes a candidate for eviction and is scheduled for eviction   |
| -          | Referenced    | Do not change the access bit                           |

### LFU

LFU == Least Frequently Used

TinyLFU is a frequency based page eviction policy that uses this.

## Recovery

Write-Ahead-Log (WAL) (also known as commit log) == Append-only auxiliary disk-resident structure used for crash and recovery.

WAL functionalities:
1. Allows page cache to buffer updates to disk-resident pages.
2. Persist all operations on disk until the caches copied of pages are flushed to disk.
3. Allow lost in-memory changes to be reconstructed from the log in the case of failures.

With the above functionalities, it plays an important role in transaction processing. In the case of crash, uncommited data is replayed from the log and the pre-crash state is restored.

## Log Semantics

WAL consists of a sequence of log records. Each log record contains a unique, monotonically increasing LSN (log-sequence-number) that identifies the log record.

Contents of log records are cached in the log buffer, and are flushed to disk in a `force` operation. Flushing happens in the sequence of LSNs.

WAL also holds records that indicate the start and end of a transaction. A transaction cannot be considered committed until the log is forced to the LSN of the commit record.

Compensation Log Records (CLRs): To be able to recover from crashes **during rollback or recovery**, we use CLR. Good link explaining CLRs: https://dba.stackexchange.com/a/299446


**Checkpoints**

Checkpoints are used to reduce the amount of work that needs to be done during recovery. Sync checkpoint is a process to force all dirty pages to be flushed to disk.

But, flushing the entire contents on disk is impractical. So, we use fuzzy checkpoints. A fuzzy checkpoint begins with a `begin_checkpoint` record, and ends with an `end_checkpoint` record, and contains information about the dirty pages and transaction. Once pages are flushed (asynchronously), the `last_checkpoint` pointer is updated to point to the `begin_checkpoint` record.

## Operation Versus Data Log

Shadow paging: It's a copy-on-write technique to ensure transaction atomicity by placing new contents in a new unpublished "shadow page", which are made visible by toggling a pointer (to point it to the new page).

We can use 2 types of logs:
1. Physical log: Stores complete page state.
2. Logical log: Stores only the operations.

Practically, databases use a combination of both:
1. Physical logging to perform a redo (since that improves recovery time)
2. Logical logging to perform an undo (since that improves concurrency and performance)

## Steal and Force Policies

Databases define steal/no-steal and force/no-force policies to determine "when" the in-memory changes are flushed to disk.

**Steal Policy:**

- Steal policy: Allows dirty pages to be flushed to disk before the transaction commits.
- No-steal policy: Dirty pages are not allowed to be flushed to disk before the transaction commits.
- "Stealing" a dirty page means flushing it to disk and replacing it with a different page.

**Force Policy:**

- Force policy: Requires dirty pages to be flushed to disk "IMMEDIATELY BEFORE" the transaction commits.
- No-force policy: Allows transaction to commit even if the dirty pages have not been flushed to disk.
- "Forcing" a dirty page means flushing it to disk before the commit.

I found the explanation in the book quite confusing. I found a much better explanation here that cleared things for me: https://stackoverflow.com/a/37861999

**Why are we discussing these policies in the context of recovery?**

- Using no-steal policy allows implementing recovery using only redo entries.
- Using no-force policy allows us to buffer several updates to pages in the page cache, and flush them to disk in a single batch. This improves performance.

General rule of thumb:
- If any pages touched by a transaction are flushed, we need to keep "undo" information in the log until it commits, so that we can use that to roll the transaction back.
- If any pages touched by a transaction are not flushed, we need to keep "redo" information in the log until it commits, so that we can use that to reach the pre-crash state.

## ARIES

ARIES == Algorithm for Recovery and Isolation Exploiting Semantics

It is a steal/no-force recovery algorithm.

It uses:
1. physical redo to improve performance during recovery.
2. logical undo to improve performance during normal operation (since they can be applied to pages independently).
3. WAL records to implement repeating history during recovery.

When database restarts after the crash, recovery happens in 3 phases:
1. Analysis Phase: Identify the dirty pages and the transactions that were active at the time of the crash.
2. Redo Phase: Repeats the history and restores the databse to the state it was in at the time of the crash. This phase is done for both committed and uncommitted transactions, whose contents were not flushed to disk.
3. Undo Phase: Rolls back the uncommitted transactions in reverse chronological order.

## Concurrency Control

3 types of techniques to handle concurrent transactions:
1. Optimistic concurrency control (OCC): Transactions do not block each other. If execution results in conflict, one of the transactions is aborted.
2. Pessimistic concurrency control (PCC): Prevents conflicts by locking data.
3. Multi-version concurrency control (MVCC): Allows multiple versions of the same data to exist at the same time.

✨ **Side learning:**

I found the above explanation from the book quite confusing. I found a much better explanation here that cleared things for me: https://stackoverflow.com/a/39269085

TLDR of the above link:
1. Optimistic Concurrency Control: This approach assumes that conflicting operations do not happen frequently (that is, it's optimistic!). This is why it allows transactions to proceed without locking the data. If a conflict happens, then one of the transactions is aborted.
2. Pessimistic Concurrency Control: This approach assumes that conflicting operations happen frequently (that's why it's called pessimistic). This is why it locks the data before allowing transactions to proceed.
3. Multi-version concurrency control (MVCC): This is a concurrency control method that typically belong in the category of Optimistic Concurrency Control. It is theoretically possible to combine it with pessimistic methods (like locking), but its multi-version nature is best combined with optimistic methods.

## Serializability

Schedule == List of operations required to execute a set of transactions from database's perspective.

"Complete" schedule: A schedule that contains all the operations of all the transactions.

"Correct" schedule: If the schedule is logically equivalent to the original lists of operations, but their parts can be executed in parallel or reordered.

"Serial" schedule: A schedule in which the operations of each transaction are executed consecutively without interleaving. This significantly reduces the concurrency, hence reducing throughput.

Serializability: A schedule is serializable if it is equivalent to "some" serial schedule.

For example, if we have 3 transactions T1, T2, and T3, then the following are the possible serial schedules:
1. T1, T2, T3
2. T1, T3, T2
3. T2, T1, T3
4. T2, T3, T1
5. T3, T1, T2
6. T3, T2, T1

## Transaction Isolation

"Isolation level" defines the degree to which the operations of a transaction are visible to other transactions.

## Read and Write Anomalies

3 types of read anomalies:
1. Dirty read: A transaction reads a value that has not yet been committed.
2. Non-repeatable read: A transaction reads a value, and then reads the same value again, but gets a different value the second time.
3. Phantom read: A transaction reads a set of rows that satisfy a search condition, and then repeats the search, but the set of rows that satisfy the search condition has changed in the meantime. It is similar to non-repeatable read, but it's for range queries.

3 types of write anomalies:
1. Lost update: A transaction overwrites the changes made by another transaction. Hence, the updates made by the other transaction are lost.
2. Dirty write: A transaction reads an uncommitted value (that is, dirty read), modifies it, and then commits it. Hence, transaction results are based on uncommitted data.
3. Write skew: When each individual transaction respects the invariant, but the invariant is violated when the transactions are considered together.

## Isolation Levels

| Isolation Level | Dirty | Non-Repeatable | Phantom |
|-----------------|------------|---------------------|--------------|
| Read Uncommitted | ✅          | ✅                   | ✅            |
| Read Committed   | ❌          | ✅                   | ✅            |
| Repeatable Read  | ❌          | ❌                   | ✅            |
| Serializable     | ❌          | ❌                   | ❌            |

✅ means that the anomaly is allowed for that isolation level.

❌ means that the anomaly is not allowed for that isolation level.

#### Snapshot Isolation

Snapshot isolation allows transactions to read data as it existed at the start of the transaction. Each transaction takes a snapshot of the database at the start of the transaction, and reads from that snapshot for the rest of the transaction. The transaction commits only if the values it has modified have not been modified by any other transaction. If the values have been modified, then the transaction is aborted and rolled back.

This makes "lost update" anomaly impossible, because the transaction is aborted if the values it has modified have been modified by another transaction.

However, a "write skew" anomaly is still possible.

## Optimistic Concurrency Control

Optimistic concurrency control (OCC) assumes that conflicting operations do not happen frequently.

3 phases of transaction execution:
1. Read Phase: Transaction reads the data it needs. After this step, all transaction dependencies (read set) and the output (write set) are known.
2. Validation Phase: Determines whether committing the transaction preserves ACID properties.
3. Write Phase: If validation phase has been successful, transaction commits the "write set" to the database.

"Validation" is done by checking for conflicts with other transactions.
1. Backward oriented: Checks conflicts with transactions that have already been committed.
2. Forward oriented: Checks conflicts with transactions that are in the validation phase.

## Multiversion Concurrency Control

MVCC distinguishes between committed and uncommitted versions of the data. It allows multiple versions of the same data to exist at the same time.

Major use case for MVCC is snapshot isolation.

## Pessimistic Concurrency Control

This assumes that conflicting operations happen frequently, hence is more conservative than optimistic concurrency control.

**Timestamp ordering**

This is one of the simplest pessimistic lock-free concurrency control schemes.

The transaction manager maintains `max_read_timestamp` and `max_write_timestamp` for each value.

When a transaction reads a value, `max_read_timestamp` is updated. When a transaction writes a value, it sets `max_write_timestamp` is updated.

- Read operations that try to read a value with a timestamp that is less than `max_write_timestamp` are aborted, since the value has been modified by a transaction that started after the current transaction.
- Write operations that try to write a value with a timestamp that is less than `max_read_timestamp` are aborted, since the value has been read by a transaction that started after the current transaction.

## Lock-Based Concurrency Control

Lock-based concurrency control is a pessimistic concurrency control scheme that uses locks on database objects (instead of trying to resolve schedules).

**Two-phase locking**: This separates the locking phase from the unlocking phase.
1. Locking / Growing / Expanding phase: Acquire all the locks that are needed by the transaction. **No locks are released.**
2. Unlocking / Shrinking phase: Release all the locks that were acquired by the transaction during the growing phase. **No locks are acquired.**

**Deadlocks**: A situation when two or more transactions are waiting for each other to release the locks. Example, T1 has locked A and is waiting for B, and T2 has locked B and is waiting for A. So both are forever waiting for each other to release the locks, hence are said to be "in deadlock".

**How to handle deadlocks?** --- Introduce timeouts and abort long-running transactions.

**How to detect deadlocks?** --- Using wait-for graphs.
1. This is a directed graph where each transaction is a node, and there is an edge from T1 to T2 if T1 is waiting for T2 to release the lock. If there is a cycle in the graph, then there is a deadlock.
2. Deadlock detection can be done periodically, or everytime the wait-for graph is updated.

**How to avoid deadlocks?** --- By restricting lock acquisition. Transaction manager can use timestamps to determine priority order of transactions. A lower timestamp means higher priority.

If T1 attempts to acquire a lock that is held by T2, and T1 started before T2 (that is, T1 has higher priority), then we can use one of the following:
1. Wait-die: T1 waits for T2 to release the lock.
2. Wound-wait: T1 aborts T2 and acquires the lock.

## Locks

- Locks are used to isolate and schedule transactions and manage database contents, NOT the internal storage structure itself.
- Locks are acquired on keys.
- Locks are generally stored and managed outside of the tree implementation.
- Locks are more heavyweight than latches.
- Locks are held for the duration of the transaction.

## Latches

- Latches guard the internal storage structure, that is, the physical representation.
- Latches are acquired on pages.
- Even the lockless concurrency control techniques have to use latches.
- Any page has to be latched for safe concurrent access.

**Example use-case:** Modifications on leaf-level can propagate to higher levels of B-Tree. In such cases, latches are obtained on multiple levels. This is to ensure that executing queries on the higher levels does not result in inconsistent data (such as incomplete writes or partial node splits).

## Readers-Writer Lock

Readers-Writer Lock (RW Lock) is a lock that allows concurrent (shared) read access, but exclusive write access.

## Latch Crabbing

Latch crabbing (also called latch coupling) is an optimisation technique to minimise the time for which a latch is held. This is especially useful when we have a simple implementation for latches, where we obtain latches on all the pages from the root to the target leaf node.

Latch crabbing allows:
1. Holding latches for less time
2. Releasing them when the executing operation doesn't need them anymore

**When can the latch be released?**
1. On a read path, the latch on parent node can be released once the child node has been located.
2. On an insert path, the latch on parent node can be released if the operation is not going to modify the parent node (via splits).
3. On a delete path, the latch on parent node can be released if the operation is not going to modify the parent node (via merges).

## Latch Upgrading and Pointer Chasing

Latch upgrading: Instead of acquiring exclusive locks during traversal, we can acquire shared locks and then upgrade them to exclusive locks when needed.

For example: Write operations on leaf nodes require exclusive locks. But, we can acquire shared locks on the parent nodes, and then upgrade them to exclusive locks "if needed" (due to splits or merges).

If multiple threads are trying to upgrade the same latch, then only one of them can succeed. The other threads have to wait or restart.

Pointer chasing: Every request goes through the root node, so we can always latch the root node optimistically. Price of "retry" (pointer chasing) is very rarely paid for root nodes since splits/merges rarely propagate to the root node.

## Blink-Trees

Blink-Trees allow a state called "half-split" where a node is referenced by sibling pointer, but not by the parent's child pointer.

Advantage: We don't need to hold parent lock when descending to child level, even if it's going to lead to a split. The new node can be made visible through the sibling link, and we can let the parent pointer update lazily without sacrificing correctness. This prevents the search path from being blocked by the parent node, causing unnecessary restarts, as all elements in the tree are accessible.

## Pending Doubts

1. In the "Log Semantics" section (Page 91), it's mentioned that "Pages are flushed asynchronously and, once this is done, the last_checkpoint record is updated with the LSN of the begin_checkpoint record and, in case of a crash, the recovery process will start from there"

Shouldn't the "last_checkpoint" record be updated with the LSN of the "end_checkpoint" record?

2. In "Latch Crabbing" section, it was mentioned that "If the child page is still not loaded in the page cache, we can either latch a future loading page, or release a parent latch and restart the root-to-leaf pass after the page is loaded to reduce contention."

What does it mean?

3. In "Latch Upgrading and Pointer Chasing" section, it was mentioned that "The root is always the last to be split, since all of its children have to fill up first. This means that the root node can always be latched optimistically, and the price of a retry (pointer chasing) is seldom paid."

Does "the root node can always be latched optimistically" mean that all threads can be allowed to acquire a "shared latch" on the root node? If not, what does it mean?

## Things to Read

1. Franck Pachot posted a blog: [Isolation Levels in Modern SQL Databases](https://dev.to/franckpachot/isolation-levels-part-i-introduction-bd5)
2. Can't wait to read WAL and Lock related content in [PostgreSQL 14 Internals](https://postgrespro.com/community/books/internals) book.

## Reading group discussion

This section contains anything worth mentioning that came up as part of the weekly reading group discussion. I will update it when I am adding notes for the next chapter.
