## Overview
### Goals
- enabling distributed transactions, as well as removing the pain of eventual consistency issues and stale reads
- Create an always-on database that accepts reads and writes on all nodes without generating conflicts
- deployment in any environment, without tying you to any platform or vendor

CockroachDB uses "consistency" in both the sense of *ACID semantics* and the *CAP theorem*

CockroachDB uses the Raft consensus protocol.
> Raft (algorithm): https://en.wikipedia.org/wiki/Raft_(algorithm)

A *consensus-based notion* of high availability that lets each node in the cluster handle reads and writes for a subset of the stored data (on a per-range basis).

### Cluster
A group of interconnected CockroachDB nodes that function as a single distributed SQL database server. Nodes collaboratively organize transactions, and rebalance workload and data storage to optimize performance and fault-tolerance.

### Node
An individual instance of CockroachDB. One or more nodes form a cluster.

### Range
CockroachDB stores all user data (tables, indexes, etc.) and almost all system data in a sorted map of key-value pairs. This ***keyspace is divided into contiguous chunks called ranges***, such that ***every key is found in one range***.

### Overview
> https://www.cockroachlabs.com/docs/v23.1/architecture/overview#overview

You can *send SQL requests to any node*; this makes CockroachDB easy to ***integrate with load balancers***

Ultimately, data is written to and read from disk using an efficient storage engine, which is able to keep track of the data's timestamp. This has the benefit of letting us support the SQL standard ***AS OF SYSTEM TIME*** clause, letting you find historical data for a period of time.

At the highest level, CockroachDB converts clients' SQL statements into ***key-value (KV) data***, which is distributed among nodes and written to disk.

## SQL Layer
### Physical planning
More concretely, the physical planning phase transforms the optimized logical plan generated during logical planning into a directed acyclic graph (DAG) of physical SQL operators. These operators can be viewed by running the EXPLAIN(DISTSQL) statement.

The decision about whether to distribute a query across multiple nodes is made by a heuristic that estimates the quantity of data that would need to be sent over the network. Queries that only need ***a small number of rows are executed on the gateway node***. Other queries are distributed ***across multiple nodes***.

For example, when a query is distributed, the ***physical planning phase splits the scan operations from the logical plan into multiple physical TableReader operators***, one for each node containing a range read by the scan. The remaining ***logical operations*** (which may perform filters, joins, and aggregations) are then ***scheduled on the same nodes*** as the TableReaders. This results in computations being ***performed as close to the physical data as possible***.

### Query execution
Components of the physical plan are sent to one or more nodes for execution. ***On each node, CockroachDB spawns a logical processor to compute a part of the query***. Logical processors inside or across nodes communicate with each other over a logical flow of data. The ***combined results*** of the query are ***sent back to the first node*** where the query was received, to be sent further to the SQL client.

### Vectorized query execution
If vectorized execution is enabled, the physical plan is sent to nodes to be processed by the vectorized execution engine.
> https://www.cockroachlabs.com/docs/v23.1/vectorized-execution
#### hash and merge joins
40x faster hash joiner with vectorized execution
> https://www.cockroachlabs.com/blog/vectorized-hash-joiner/

### Schema changes
CockroachDB performs schema changes, such as the addition of columns or secondary indexes, using a protocol that ***allows tables to remain online*** (i.e., able to serve reads and writes) during the schema change.

## Transaction Layer
> https://www.cockroachlabs.com/docs/v23.1/architecture/transaction-layer

CockroachDB ***executes all transactions at*** the strongest ANSI transaction isolation level: ***SERIALIZABLE***. All other ANSI transaction isolation levels (e.g., SNAPSHOT, READ UNCOMMITTED, READ COMMITTED, and REPEATABLE READ) are automatically upgraded to SERIALIZABLE. Weaker isolation levels have historically been used to maximize transaction throughput. However, ***recent research***
> http://www.bailis.org/papers/acidrain-sigmod2017.pdf

has demonstrated that the use of weak isolation levels results in substantial vulnerability to concurrency-based attacks.

## Distribution Layer
### Overview
CockroachDB stores data in a monolithic sorted map of key-value pairs. This key-space describes all of the data in your cluster, as well as its location, and is ***divided into what we call "ranges"***, contiguous chunks of the key-space, so that ***every key can always be found in a single range***.

CockroachDB implements a sorted map to enable:

- Simple lookups: Because we identify which nodes are responsible for certain portions of the data, queries are able to quickly locate where to find the data they want.
- Efficient scans: By defining the order of data, it's easy to find data within a particular range during a scan.




















