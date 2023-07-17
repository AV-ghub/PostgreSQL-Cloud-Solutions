## Overview
> https://docs.citusdata.com/en/v11.3/get_started/what_is_citus.html

With Citus you get distributed Postgres features like **sharding, distributed tables, reference tables, a distributed query engine, columnar storage**—and as of Citus 11.0, the **ability to query from any node**. The Citus combination of **parallelism, keeping more data in memory, and higher I/O bandwidth** can lead to significant performance improvements for multi-tenant SaaS applications, customer-facing real-time analytics dashboards, and time series workloads.

Here are a few **examples of large-scale** customers
- Algolia 5-10B rows ingested per day
- Heap 700+ billion events 1.4PB of data on a 100-node Citus database cluster
- Pex 80B rows updated/day 20-node Citus database cluster 2.4TB memory, 1280 cores, and 80TB of data …with plans to grow to 45 nodes
- MixRank 10 PB of time series data

Citus provides full SQL coverage for this workload, and **enables scaling** out your relational database **to 100K+ tenants**. 
Citus **supports tenant isolation** to provide performance guarantees **for large tenants**, and has the concept of **reference tables to reduce data duplication across tenants**.

Some **advantages of Citus** for multi-tenant applications:
- Fast queries for all tenants
- Sharding logic in the database, not the application
- Hold more data than possible in single-node PostgreSQL
- Scale out without giving up SQL
- Maintain performance under high concurrency
- Fast metrics analysis across customer base
- Easily scale to handle new customer signups
- Isolate resource usage of large and small customers

#### Real-Time Analytics
Citus supports real-time queries over large datasets. Commonly these queries occur in rapidly growing event systems or systems with time series data.

**Citus’ benefits** here are its ability to parallelize query execution and scale linearly with the number of worker databases in a cluster. Some advantages of Citus for real-time applications:
- Maintain sub-second responses as the dataset grows
- Analyze new events and new data as it happens, in real-time
- Parallelize SQL queries
- Scale out without giving up SQL
- Maintain performance under high concurrency
- Fast responses to dashboard queries
- Use one database, not a patchwork
- Rich PostgreSQL data types and extensions

#### When Citus is Inappropriate
Some workloads **don’t need a powerful distributed database**, while others **require a large flow of information between worker nodes**. In the first case Citus is unnecessary, and in the second not generally performant. Here are some examples:

- When you do not expect your workload to ever grow beyond **a single Postgres node**
- Offline analytics, **without the need for real-time** ingest nor real-time queries
- Analytics apps that **do not need to support a large number of concurrent users**
- Queries that **return data-heavy ETL results rather than summaries**

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





















