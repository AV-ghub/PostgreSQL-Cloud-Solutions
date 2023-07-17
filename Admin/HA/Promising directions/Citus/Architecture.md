## Overview
> https://docs.citusdata.com/en/v11.3/get_started/what_is_citus.html
With Citus you get distributed Postgres features like **sharding, distributed tables, reference tables, a distributed query engine, columnar storage**—and as of Citus 11.0, the **ability to query from any node**. The Citus combination of **parallelism, keeping more data in memory, and higher I/O bandwidth** can lead to significant performance improvements for multi-tenant SaaS applications, customer-facing real-time analytics dashboards, and time series workloads.

Here are a few **examples of large-scale** customers
- Algolia 5-10B rows ingested per day
- Heap 700+ billion events 1.4PB of data on a 100-node Citus database cluster
- Pex 80B rows updated/day 20-node Citus database cluster 2.4TB memory, 1280 cores, and 80TB of data …with plans to grow to 45 nodes
- MixRank 10 PB of time series data

> https://docs.citusdata.com/en/v11.3/get_started/concepts.html
### Nodes
Allows commodity database servers (called nodes) to coordinate with one another in a ***“shared nothing”*** architecture. The nodes form a *cluster* that allows PostgreSQL to hold more data and use more CPU cores than would be possible on a single computer. This architecture also allows the database ***to scale by simply adding more nodes*** to the cluster.

### Coordinator and Workers
Every cluster has one special node called the ***coordinator*** (the others are known as ***workers***). Applications send their queries to the coordinator node which relays it to the relevant workers and ***accumulates the results***.

For each query, the coordinator ***either routes*** it to a single worker node, ***or parallelizes*** it across several depending on whether the required data lives on a single node or multiple. The coordinator knows how to do this by ***consulting its metadata tables***. These Citus-specific tables track the DNS names and health of worker nodes, and the distribution of data across nodes.

## Distributed Data
### Table Types
#### Type 1: Distributed Tables
Horizontally partitioned across worker nodes.
The component worker tables are called ***shards***.

#### Distribution Column
Citus uses algorithmic sharding to assign rows to shards, based on the value of a particular table column called the ***distribution column***.

#### Type 2: Reference Tables
A type of distributed table whose entire ***contents are concentrated into a single shard which is replicated on every worker***. Thus queries on any worker can access the reference information locally, without the network overhead. Reference tables ***have no distribution column***.

Reference tables are ***typically small***, and are used to store data that is relevant to queries running on any worker node.

When interacting with a reference table we ***automatically perform two-phase commits*** (2PC) on transactions.

#### Type 3: Local Tables
When you use Citus, the coordinator node you connect to and interact with is a regular PostgreSQL database with the Citus extension installed. Thus you can create ordinary tables and choose not to shard them. This is useful for small administrative tables.

Citus allows ***shards to be replicated*** for protection against data loss using PostgreSQL ***streaming replication***. 

### Parallelism
Queries reading or affecting shards spread evenly across many nodes are able to run at “real-time” speed. Note that the results of the query still need to pass back through the coordinator node, so ***the speedup is most apparent when the final results are compact***.

### Query Execution
The coordinator node has a ***connection pool*** for each session. Each query (such as SELECT * FROM foo in the diagram) is limited to opening at most citus.max_adaptive_executor_pool_size (integer) simultaneous connections for its tasks per worker. That setting is configurable at the session level, for priority management.

When a task finishes using a connection, the session pool will hold the connection open for later.





















