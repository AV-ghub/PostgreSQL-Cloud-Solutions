## Overview
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


























