# Chapter 2: B-Tree Basics

B-Trees are one of the most popular storage structures used in databases.



## Binary Search Trees (BST)

A binary search tree is a sorted, in-memory data structure that is used for efficient key-value lookups.

BST invariant: For any node, all the keys in its left subtree are less than the node's key, and all the keys in its right subtree are greater than or equal to the node's key.

Tree balancing: If the tree is not balanced, the height of the tree can be as high as the number of nodes in the tree. This can lead to O(n) lookup time.

Balanced tree == height of the tree is O(log n), where n is the number of nodes in the tree + difference in height of left and right subtrees is at most 1.

We balance the tree by rotating the nodes. The rotation is done in such a way that the BST invariant is maintained. For example:

```
Before balancing:
        3
      /
     2 (pivot!!)
    /
   1

After balancing:
        2
      /   \
     1     3
```

## Trees for Disk-Based Storage

(Fanout == branching factor == number of children a node can have)

BSTs are not suitable for disk-based storage because of the following reasons:
1. Locality: New element might not be written to the same page as its parent, leading to more disk seeks
2. Low fanout == more height == more disk seeks required to find the element


## Storage types

Before we discuss better structures suitable for disk-based storage, let's discuss the different types of storage available.

**Hard disk drives (HDDs):** 
HDDs are mechanical devices that have a spinning disk with a read/write head. The read/write head moves to the location of the data to be read/written. HDDs are slow because of the mechanical movement involved to position the read/write head to the correct location. The reading/writing itself is fast once the head is positioned correctly.

Smallest unit of data that can be read/written to an HDD is a sector (typically ranges from 512 bytes to 4 KB).

**Solid state drives (SSDs):**

SSDs are faster than HDDs because they don't have any moving parts.

SSD structure:
1. SSDs have one or more dies.
2. Each die has multiple planes.
3. Each plane has multiple blocks.
4. Each block has multiple pages.
5. Each page has multiple arrays.
6. Each array has multiple strings.
7. Each string has multiple memory cells.

🤯 In SSDs, the data is read/written at the page level, but is erased at the block level.

✨ **Side learning:**
But why?
This is because of the physical properties of quantum tunneling. Long story short, a significant amount of voltage is required to erase data, and it is difficult to target that voltage at a more granular level without negatively affecting the surrounding cells. Hence, the smallest unit of data that can be erased is a block.

Source:
1. https://qr.ae/pKHTdl
2. https://www.techtarget.com/searchstorage/definition/solid-state-storage-SSS-garbage-collection

## On-disk data structures: B-Trees

B-Trees are based on binary search trees, but have higher fanout, and hence lesser height.

Binary Tree == 1 key + 2 child pointers per node.
2-3 Tree == 2 keys + 3 child pointers per node.

Property: All keys inside a B-Tree node are sorted. To locate a key within a node, we can use binary search.

Occupancy of a node == number of keys in a node in comparison to the maximum number of keys that can be stored in a node.

B-Tree characteristics:
1. Higher fanout leads to amortized cost of keeping the tree balanced.
2. Reduced number of seeks by storing keys and child pointers in a single block or consecutive blocks.
3. Balancing operations happen when the nodes are either full or nearly empty.

## Separator Keys

Separator Keys == Index entries == Divider cells

Say "p" denotes a child pointer, and "k" denotes a key. Then, a separator key is of the form:

```
p1 k1 p2 k2 p3
```

Here:
1. p1 points to the child node with keys < k1
2. p2 points to the child node with keys >= k1 and <= k2
3. p3 points to the child node with keys > k2

## Sibling pointers

Sibling pointers == Next and previous pointers == Link to the next and previous node at the same level

B-Trees have sibling pointers on the leaf nodes. This allows us to traverse the leaf nodes in a linked list fashion, and helping range queries (as we don't have to go back to the parent to find the next node).

## B-Tree Lookup Complexity

log M, where M is the total number of nodes in the tree.

## B-Tree Node Splitting

When a node is full, we split it into two nodes. The middle key is moved to the parent node, and the remaining keys are distributed between the two nodes.

Say we have a maximum occupancy of 3 keys per node.

Parent node: 18 _ _
Left pointer of 18 points to a node having: 10 13 15

If we insert 11, we will have to split the child node since we cannot have "10 11 13 15" in a single node.

So, we split "10 11 13 15" into "10 11" and "13 15". The middle key "13" is moved to the parent node (while keeping the parent node's keys sorted), and the remaining keys are distributed between the two nodes.

So, final state of the tree will be:
Parent node: 13 18 _
Left pointer of 13 points to a node having: 10 11 _
Pointer between 13 and 18 points to a node having: 13 15 _

You can refer to diagram on page 40 of the book for a visual representation.

## B-Tree Node Merging

Deletions are done by removing the key and value from the leaf node. If the neighbouring nodes have too few values, the sibling nodes are merged. This is called underflow.

Example:
```
Parent node: 11 20 _
Pointer between 11 and 20 points to a node having: 15 16 _
Pointer between 20 and _ points to a node having: 25 30 _

If we delete 16, we will have to merge the two nodes since the occupancy of the two nodes will be 15 _ _ and 25 30 _ respectively.

For merging, we move elements from the right sibling node to the left sibling node, and update the parent node accordingly.

So, final state of the tree after merging will be:
Parent node: 11 _ _
Pointer between 11 and _ points to a node having: 15 25 30.
```

## Pending Doubts

1. Page 35 says "B-Trees are a page organization technique (i.e., they are used to organize and navigate fixed-size pages), we often use terms node and page interchangeably". What does it mean? How are nodes equivalent to pages?

[Update] Answer: We understand this better in chapter 3.

Other answers I got from other members of the reading group:
As far as implementations go, databases talk in terms of pages instead of blocks. For instance, in SQLite's BTree implementation, they exclusively talk in terms of pages. The size of the page can be between 512 bytes and 65536 bytes, and each page contains pointers to other pages.

In the case of Postgres, things become a bit more complicated. For instance, they say they split a heap or index file in blocks. But a block can be unformatted, whereas a page is a well formed slotted page. All the access method implementations (including BTree) work with pages.

2. SSD has page inside blocks. But in chapter 1, we saw that database had 1 page = multiple blocks? Any historic context on why we have this discrepancy in naming? Any intuitive explanation for this?

[Update] Answer: It's just confusing, but it's what it is.

Other answers I got from other members of the reading group:
It seems that everyone uses the terms differently depending on context. For example, there is the context of the physical disk, of the interface the disk uses to communicate (e.g. NVMe, NTFS, etc.), the file system, and databases. It doesn't help that every vendor at every layer there often uses a different or varied term too.

Databases mostly seem to use the term "page" to refer to the minimum number of bytes they read or write. Whereas "block" seems to more refer to either the file system, disk protocol (NVMe, etc.), or physical disk minimums. And "page" is always a multiple of the block size so that you aren't reading/writing partial blocks which incurs additional IO.

Various sites to increase the confusion even further:
- https://superuser.com/questions/1110355/what-is-the-difference-between-a-disk-block-and-a-sector
- https://www.reddit.com/r/sysadmin/comments/37a8xi/eli5_the_differences_between_a_chunk_page_block/
- https://dba.stackexchange.com/a/241042
- https://wiki.archlinux.org/title/Advanced_Format

Drop me an email if you would like to answer any of the doubts, or discuss: [Let's talk!](mailto:me@akjn.dev?subject=[Chapter%202]%20Doubts)

## Things to Read

NA at the moment.

## Reading group discussion

This section contains anything worth mentioning that came up as part of the weekly reading group discussion.

1. There is a good explanation of B-Trees in the Introduction to Algorithms book by Cormen that you may have lying around from school or work. It includes a nice explanation of how (spinning) disks work which is rare for an algorithms book.

2. Good explanation of Red-Black and 2-3 trees: https://www.cs.umd.edu/class/fall2019/cmsc420-0201/Lects/lect06-aa.pdf

3. Fly.io has a great blog, here's one from Ben Johnson about SQLite btrees: https://fly.io/blog/sqlite-internals-btree/