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






























