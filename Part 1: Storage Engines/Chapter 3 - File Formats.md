# Chapter 3: File Formats

Data layout is much more important in disk than in memory. For a disk-based data structure to be efficient, we need to:
1. lay it out in a way that allows quick/efficient access to the data we need.
2. consider the specific characteristics of the underlying storage medium.
3. come up with binary data formats, and also ways to serialize and deserialize data.

## Binary Encoding

To store data on disk, we need a format that is compact + easy to serialize and deserialize. We can use binary encoding for this purpose.

### Primitive Types

Types like integer, data, and string can be represented in their raw binary forms. However, for multi-byte numeric values, we need to consider the byte order (endianness) of the system.

✨ **Side learning:**
Why do we still have endianness issues in modern systems? There's no good answer to this, we've just failed to standardize on a single byte order: https://softwareengineering.stackexchange.com/a/301184

2 types of endiannes:
1. Big-endian: Most significant byte first.
2. Little-endian: Least significant byte first.

Floating point numbers are represented in IEEE 754 standard. A 32 bit number is represented as:
1. 1 bit for sign
2. 8 bits for exponent
3. 23 bits for mantissa

### Strings and Variable-Size Data

Variable-sized data types such as Strings and Arrays of fixed-size data can be serialized as a number (size) followed by the actual data.

For example, a string "Hello" can be serialized as:
```
5Hello
```

An alternative approach is to use a special character to mark the end of the string. For example, a string "Hello" can be serialized as:
```
Hello\0
```

### Bit-Packed Data

Some data types are not byte-aligned, and it's a waste to use the whole byte to store them. For example, a boolean value can be represented in 1 bit. We can use bit-packing to store such data types.

1. Boolean: We can say that 1 is true and 0 is false.
2. Enums can be represented as a number, and we can use bit-packing to store them.
3. Flags: Flag is a combination of boolean values and enums. If we have 3 flags, we can use 3 bits, each representing the value of a flag.

## General Principles

The file usually starts with a fixed-size header and may end with a fixed-size trailer. The rest of the file is split into pages. A very raw diagram of a file is as follows:

```
+-------------------+
| Header            |
+-------------------+
| Page 1            |
+-------------------+
| Page 2            |
+-------------------+
| ...               |
+-------------------+
| Page N            |
+-------------------+
| Trailer           |
+-------------------+
```

If we have a structure having a mix of fixed-size and variable-size fields, we can store the fixed-size fields in the structure itself,

## Page Structure

The data and index files (that we studied in chapter 1) are partitioned into fixed-size units called pages (ranging from 4KB to 16KB).

**Example of an on-disk B-Tree node**
Each B-Tree node occupies one or more pages linked together. So we interchangably use the terms page and node. A B-Tree node can be represented as follows:

```
-------------------------------------------------------------------
| p0 | k1 | v1 | p1 | k2 | v2 | ... | kn | vn | pn | Unused space |
-------------------------------------------------------------------
```
where:
- p = pointers to child pages
- k = keys
- v = values

Downsides of this approach:
1. Inserting a key-value pair in the middle of the node requires shifting all the subsequent key-value pairs.
2. Doesn't allow managing variable-size data efficiently.

## Slotted Pages

Main problem with variable-size data is free space management, that is, reclaiming space from deleted records and reusing it for new records.

We can simply re-write the whole page whenever we need to update a record, but that would be inefficient and we would have to preserve record offsets as any out-of-page pointers might be referencing them. Hence, instead of this, we use a technique called "slotted page structure". This approach is used by many databases, like PostgreSQL.

We organise the page into a collection of slots or cells, with the pointers and cells being present on the opposite sides of the page.

Slotted page structure is as follows:
```
----------------------------------------------------------------------------------------------------
| Header | Pointer to cell 1 | Pointer to cell 2 |----->>> Free Space  <<<-------| Cell 2 | Cell 1 |
----------------------------------------------------------------------------------------------------
```

We will look at an example of the above structure in a little while.

For now, let's understand some aspects of this page format that justifies the design decisions:
1. Problem: We need to store variable-size records with a minimal overhead.
Solution: The only overhead in this approach is the "pointer to cell 1" part, that is, a pointer array.

2. Problem: We need to be able to reclaim space occupied by the removed records.
Solution: Space can be reclaimed by defragmenting and rewriting the page.

3. Problem: We need to be able to reference records in the page without regard to their exact locations.
Solution: Out-of-page pointers can reference the page and the slot IDs. They don't have to care about the exact location of the slot/cell. The exact location of the slot/cell is relative to the page.

## Cell Layout

2 types of cells:
1. Key cells: They contain a separator key and a pointer to the page between two neighboring pointers.
2. Key-Value cells: They contain keys and data records associated with them.

Assumption: All cells in a page are of the same type. This allows us to store metadata of the cell only once in the page header, instead of storing it for each cell.

**Key Cell**

We need to know:
1. Cell type: This is inferred from the page header.
2. Key size
3. Page ID: This is ID of the page this cell is pointing to.
4. Key bytes

A variable size key cell can look like:
```
----------------------------------
| Key size | Page ID | Key bytes |
----------------------------------
```

**Key-Value Cell**

We need to know:
1. Cell type: This is inferred from the page header.
2. Key size
3. Value size
4. Key bytes
5. Value (data record) bytes

A variable size key-value cell can look like:
```
-----------------------------------------------------
| Key size | Value size | Key bytes | Value bytes |
-----------------------------------------------------
```

## Combining Cells into Slotted Pages

We follow the structure of a slotted page that we learned above:
```
----------------------------------------------------------------------------------------------------
| Header | Pointer to cell 1 | Pointer to cell 2 |----->>> Free Space  <<<-------| Cell 2 | Cell 1 |
----------------------------------------------------------------------------------------------------
```

Cell Offsets/Pointers: Pointer to cell 1, Pointer to cell 2, etc. are called cell offsets or cell pointers.

Keys can be inserted out of order, but the cell offsets are always kept in the order of keys. Why?
Remember: Node of B-Tree == Page. The separator keys of the node are stored in sorted order to allow binary search. The cells essentially map to the separator keys.

**Step by step example of a page that holds names**

1. Insert "NameB"
```
------------------------------------------
| Header | PointerB | Free Space | NameB |
------------------------------------------ 
```

2. Insert "NameA"
```
-------------------------------------------------------------
| Header | PointerA | PointerB | Free Space | NameA | NameB |
-------------------------------------------------------------
```
Note that PointerA is before PointerB because NameA comes before NameB.

3. Insert "NameC"
```
--------------------------------------------------------------------------------
| Header | PointerA | PointerB | PointerC | Free Space | NameC | NameA | NameB |
--------------------------------------------------------------------------------
```

## Managing Variable-Size Data

Removing a cell doesn't have to mean that we have to move all the subsequent cells. We can simply mark the cell as deleted and reuse the space when we need to insert a new cell.

When we mark a cell as deleted, an in-memory availability list is updated. This list contains the offsets of freed segments and their sizes. When inserting a new cell, we can use the space from the availability list if it's big enough.

If we cannot find a big enough space in the availability list, we can defragment the page by moving the cells to the beginning of the page and updating the cell offsets. This is an expensive operation, so we should do it only when we have to.

If there's still not enough free space after defragmentation, we have to create an Overflow page. We will learn about overflow pages in chapter 4.

## Versioning

The binary data format might change over time, as the developers add new features or fix bugs. This can cause problems when we have to read old data using the new version of the software. Hence, most of the times, any storage engine has to support more than one serialization formats.

To figure out which serialization format to use, we make use of a version number, which helps determine the format of the data.

Example:
1. Apache Cassandra uses a version number prefix in the filenames.
2. PostgreSQL stores the version number in the PG_VERSION file.

After finding out the version of the file, we can create a version-specific reader to read the file contents.

## Checksumming

1. Checksums provide weakest guarantee of data integrity, and cannot detect corruption in multiple bits.
2. CRCs can help detect corruption in multiple bits.

We update the checksum whenever we write a page to disk. When reading a page, we can verify the checksum to detect corruption.

Computing checksum over file contents is expensive, so we can compute it over the page contents and store it in the page header. This also means that we don't have to discard the whole file if the corruption is detected in one page.

## Pending Doubts

1. On page 51 (in the General Principles section), we have an example showcasing how we can store a structure having a mix of fixed-size and variable-size fields. I have some confusion in that.

Let's say we have a structure like this:
```
struct Person {
    int age; // fixed size of say, 2 bytes
    char gender; // fixed size of say, 1 byte
    string name; // variable size where say, the number of characters is the number of bytes required
}
```

And we have to store 2 persons:
- Person 1: (age, gender, name) = (10, M, Akshat)
- Person 2: (age, gender, name) = (15, M, Phil)

How would we store this? Is it like this?
```
Header: 2Age1Gender
Page 1: 10M6Akshat
Page 2: 15M4Phil
Trailer
```

The book seems to indicate that the variable-sized fields would also be part of the structure, but I am not sure how that would work. I think I am missing something here.

[Update] Answer by Alex Petrov (author): What I tried to say was that the structure holding the employee data can have first a block with fixed data followed by the variable size data. 

2. On a similar note as question 1, what is the difference between "offset" and "length"? Context: The book says "To avoid calculations involving multiple fields, we can encode both offset and length to the fixed-size area. In this case, we can locate any variable-size field separately."

[Update]
Offset = address in memory where the item starts
Length = Length of the item in bytes.

3. Why is "[byte] flags" present in the representation of a Key-Value cell on page 55? What is the purpose of this? Why was it not present in the representation of a Key cell on page 54?

4. In Figure 3-9, it should be "Occupied cells are shown in gray." instead of "Occupied pages are shown in gray."?

[Update] Yes. It's a typo.

## Things to Read

NA at the moment.

## Reading group discussion

Nothing "extra" was discussed that is worth mentioning here.
