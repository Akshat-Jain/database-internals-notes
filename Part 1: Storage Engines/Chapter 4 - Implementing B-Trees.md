# Chapter 4: Implementing B-Trees

## Page Header

Page header contains useful metadata like number of cells in the page, page contents and layout, etc.

- Postgres stores page size and layout version in the header.
- MySQL InnoDB stores number of heap records etc in the header.

## Magic Numbers

Magic number is a multi-byte block placed in the header of the file or page. These are used to identify the file type and version.

✨ **Side learning:**
For anyone interested in checking the different use-cases for which Postgres leverages magic numbers, I think this is a good place to start: [PostgreSQL Code search for Magic Number](https://github.com/search?q=repo%3Apostgres%2Fpostgres%20magic%20number&type=code)

## Sibling Links

For going from child1 to child2, we need to go to the parent and then to child2. This is a lot of work. So, we can add a link from child1 to child2. This is called a sibling link.

A sample tree showing sibling link:
```
        parent
      /       \
    child1 -> child2
```

Downside of sibling links: They need to be updated during splits and merges, so that's extra overhead.

## Rightmost Pointers

A node having 1 separator key looks like:
```
[left child pointer] [separator key] [right child pointer]
```

Similarly, a node having multiple separator keys looks like:
```
p1 k1 p2 k2 p3 k3 p4
```

To be able to pair each separator key with its corresponding child pointer, we add a rightmost pointer to the node. This is a pointer to the rightmost child of the node.

So instead of the above representation, we get:
```
    [other stuff] p4 (p4 is stored separately in the page header)
    p1 k1 p2 k2 p3 k3
```

## Node High Keys

This is an alternative to rightmost pointers. Instead of storing the rightmost pointer separately (to be able to pair the remaining keys with the pointers in a 1:1 fashion), we can add a high key to the node.

So instead of the above representation, we get:
```
p1 k1 p2 k2 p3 k3 p4 k4 (where k4 is the high key)
```

This simplifies handling edge cases.

## Overflow pages

If a payload is too large to fit in a page, we can store it in an overflow page. This is a linked list of pages.

Most B-Tree implementations allow storing upto a fixed number of payload bytes in the page itself. If the payload is larger than that, it is stored in an overflow page.

Example:
```
Primary page:
    [payload1 with pointer to "payload 1 continued"]
    [payload2 with pointer to "payload 2 continued"]
    [free space in the page]

Overflow page:
    [payload1 continued]
    [payload2 continued]
    [free space in the page]
```

Overflow pages require some bookkeeping. Page ID of first overflow page is stored in the primary page. Each overflow page stores the page ID of the next overflow page (if the next page exists).

## Binary Search

Insertion point: It is the index of the first element that is greater than the given key.

The binary search algorithm returns the insertion point if the key is not found in the array (since majority of searches are not going to result in exact matches).

## Binary Search with Indirection Pointers

Since cell offsets are stored in a sorted way (as we learned in previous chapters), we pick the middle cell offset and compare it with the key we are searching for.

- If the key is greater than the middle cell offset, we pick the middle cell offset of the right half of the array and repeat the process.
- If the key is smaller than the middle cell offset, we pick the middle cell offset of the left half of the array and repeat the process.

## Propagating Splits and Merges

B-Tree splits and merges can propagate up the tree. This is because the parent node needs to be updated with the new separator key and the new child pointer.

If we include parent pointers in the nodes, we can easily propagate splits and merges up the tree, but this adds extra overhead for changing parent pointers during splits and merges.

**Breadcrumbs**

Other option is to use a stack to keep track of the path from the root to the leaf node. This is a common approach used in B-Tree implementations, and is called breadcrumbing.

✨ **Side learning** (this one is unrelated to databases)

"Breadcrumb" is apparently a standard terminology not just specific to databases. For example: Breadcrumb navigation is a term used in web development. It is a navigation scheme that reveals the user's location in a website or web application. Source:
[Breadcrum Navigation](https://www.smashingmagazine.com/2009/03/breadcrumbs-in-web-design-examples-and-best-practices/)

Breadcrumbs contain references to all the nodes traversed from the root to the leaf node. This is a stack of node pointers + insertion point in each node.

## Rebalancing

Some implementations try to postpone splits and merges as much as possible, in order to amortize the cost.

This is done by:
1. rebalancing elements with the level
2. moving elements from higher occupancy nodes to lower occupancy nodes

This helps improving node occupancy, which reduces number of levels in a tree and helps prevent potentially higher cost of splits and merges.

B* trees are an example of such an implementation.
1. They keep distributing data between neighbouring nodes until all sibling nodes are full.
2. Instead of splitting a single node into 2 half nodes, they split 2 nodes into 3 nodes (each of them being 2/3 full).

## Right-Only Appends

It's frequent to see database systems use auto-incremented IDs for primary keys. This means that all insertions will happen in the rightmost leaf, which also means that most of the splits would occur on the rightmost node on each level. So, the idea here is to use that fact and think about optimizations.

Postgres calls this "fastpath".

SQLite has a similar concept and calls it "quickbalance".

## Bulk Loading

If we have presorted data, and we want to bulk load it, or if we have to rebuilt the tree (for example, for defragmentation), we can leverage the idea of right-only appends and use it to optimise the process for bulk loading.

High-level steps:
1. We can compose the tree from bottom up, starting by writing presorted data on the leaf-node-level page-wise (we don't need to insert the elements one by one since we already know their order!).
2. After leaf-level is done, we can move up to the parent level and repeat the process.

## Compression

Compression is a technique to reduce the size of the data. It is used to reduce the size of the data on disk, and also to reduce the amount of data that needs to be read from disk.

Tradeoff:
1. Larger compression ratios == less size on disk == less data to read from disk. But this also means that we need to spend more RAM and CPU cycles to compress and decompress the data.
2. Smaller compression ratios == more size on disk == more data to read from disk. But we need to spend less RAM and CPU cycles to compress and decompress the data.

Compression can be done at different levels:
1. files: This is impractical since we need to decompress the entire file to read a single page.
2. pages: This is a good approach, and allows coupling the compression/decopression with page loading/flushing. However, the compressed page now occupies only a fraction of a disk block, so any transfers would page in extra bytes (since transfers are usually done in units of disk blocks).
3. data (row-wise or column-wise): In this case, page loading/flushing is not coupled with compression/decompression.

## Vacuum and Maintenance

- Addressable data records: Data records that can be reached by following pointers down from the root node. On a page level, cells that have cell pointers to them are addressable.
- Non-addressable data records: Data records that are not referenced anywhere and cannot be read or interpreted.

We have some processes that happen in the background to help us avoid the overhead during inserts/updates/deletes.

**Fragmentation Caused by Updates and Deletes**

Different cases:
1. Deletion of an element: On the leaf level, deletes only remove cell offsets from the page header. We don't need to zero-fill the cell itself, as eventually that area would be overwritten by new data anyway. And we don't need to worry about reading garbage data since we made it non-addressable by removing the cell offset from the page header.
2. Page split: During page splits, only the cell offsets are split, and we don't have to explicitly remove the cells from the page (that have been moved to the new page).

## Page Defragmentation

Compaction / Vacuum: The maintenance process that takes care of space reclamation and page rewrites.

Page re-writes can be done synchronously or asynchronously.

Whenever pages are rewritten, they may also get relocated to new positions in the file.

IDs of the newly available on-disk pages are added to a "free page list" (also called freelist). This information has to be persisted to survive node crashes and restarts.

## Pending Doubts

1. Are "magic numbers" similar to how we use prefixes to identify file types such as png, jpg, etc? Is that a good analogy for intuition?

2. In "Overflow Pages" section, it was written that: "Using this approach, we cannot end up in a situation where the page has no free space, as it will always have at least max_payload_size bytes." --- Why? How?

3. In "Bulk Loading" section, it was mentioned that: "Immutable B-Trees can be created in the same manner but, unlike mutable B-Trees, they require no space overhead for subsequent modifications, since all operations on a tree are final. All pages can be completely filled up, improving occupancy and resulting into better performance." --- What does the following things mean?
    - "they require no space overhead for subsequent modifications" - what space overhead are we talking about here?
    - "All pages can be completely filled up, improving occupancy and resulting into better performance" - what does mutability have to do with occupancy?

4. In "Page Defragmentation" section, it was mentioned that: "Whenever pages are rewritten, they may also get relocated to new positions in the file." --- Why can we not always do them in-place?

5. In "Page Defragmentation" section, it was mentioned that: "Page rewrites can be done synchronously on write if the page does not have enough free physical space (to avoid creating unnecessary overflow pages)" --- Why do we need to avoid creating unnecessary overflow pages?

## Things to Read

1. Use-cases of immutable B-Trees

## Reading group discussion

This section contains anything worth mentioning that came up as part of the weekly reading group discussion. I will update it when I am adding notes for the next chapter.
