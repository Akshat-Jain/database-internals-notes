# Part 1: Storage Engines

Storage engine == Component of DBMS responsible for storing + retrieving data in memory + disk.

Storage engines can be considered a completely separate part of DBMS, to the extent that you can plug in a different storage engine into your database system depending on your use-case. Example: MySQL allows switching between InnoDB, MyISAM, and RocksDB.

✨ **Side learning**
1. Postgres apparently has only one storage engine, and is developing a new engine called zheap.
2. https://wiki.postgresql.org/wiki/Future_of_storage has an interesting table at the end of the page comparing different storage engines and if they would be useful in Postgres context.
3. MySQL allows using storage engines on a per-table level: `CREATE TABLE t (i INT) ENGINE = InnoDB;`


## Comparing databases
Large number of factors affect performance:
1. Schema
2. Number of clients
3. Rate of read and write queries
4. Types of queries
5. etc etc

## TPC-C Benchmark

TPC == Transaction Performance Council

TPC-C is an OLTP benchmark and tests a mix of read-only and update transactions aiming to simulate common workloads.

✨ **Side learning:** https://docs.yugabyte.com/preview/benchmark/tpcc-ysql/ - We can see the kind of metrics evaluated and the kind of factors coming into play.

🤯 Other than being used for comparing databases, benchmarks are also used to define SLAs!!!

## Understanding Trade-Offs

No storage engine is perfect. There are tradeoffs.

Example:
1. Saving records in the order of insertion == faster writes + slower reads (well, technically depends on the query, but you get the idea)
2. Saving records in a sorted manner == slower writes + faster reads (again, technically depends on the query)
