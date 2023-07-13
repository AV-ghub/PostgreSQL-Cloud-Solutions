## Universe
> https://docs.yugabyte.com/preview/architecture/concepts/universe/
A YugabyteDB universe is ***a group of nodes (virtual machines, physical machines, or containers)*** that collectively function as a resilient and scalable distributed database.

### Organization of data
A YugabyteDB universe can consist of ***one or more namespaces***. Each of these ***namespaces can contain one or more user tables***.

YugabyteDB ***automatically shards, replicates, and load-balances*** these tables across the nodes in the universe

YugabyteDB **_automatically handles failures_**, such as node, disk, AZ, or region failures, as well as redistributes and rereplicates data back to desired levels across the remaining available nodes

A namespace in **_YSQL_** is referred to as a database and is logically identical to a namespace in other RDBMS (such as PostgreSQL).

A namespace in **_YCQL_** is referred to as a keyspace and is logically identical to a keyspace in Apache **_Cassandra's CQL_**.

## Component services
A universe comprises of two sets of servers: YugabyteDB Tablet Server (**_YB-TServer_**) and YugabyteDB Master Server (**_YB-Master_**). The sets of YB-TServer and YB-Master servers form two respective distributed services **_using Raft as a building block_**. High availability (HA) of these services is achieved by the failure detection, leader election, and data replication mechanisms in the Raft implementation.

### YB-TServer service
The YugabyteDB Tablet Server (YB-TServer) service is _**responsible for the input-output (I/O)**_ of the end-user requests in a YugabyteDB cluster. Data for a table is split (sharded) into tablets. Each tablet is composed of **_one or more tablet peers_**, depending on the _**replication factor**_. Each YB-TServer hosts one or more tablet peers.

#### Server-global block cache
The _**block cache is shared across different tablets in a given YB-TServer**_, leading to highly-efficient memory utilization in cases when one tablet is read more often than others.

### YB-Master service
The YB-Master service keeps the _**system metadata**_ and records

The YB-Master service is also responsible for _**coordinating background operations**_, such as load-balancing or initiating rereplication of under-replicated data, as well as performing a variety of administrative operations such as creating, altering, and dropping tables.

Examples of such operations include user-issued CREATE TABLE, ALTER TABLE, and DROP TABLE requests, as well as creating a backup of a table. The YB-Master performs these operations with a _**guarantee that the operation is propagated to all tablets irrespective of the state**_ of the YB-TServers hosting these tablets.

Each YB-Master _**stores system metadata**_, including information about namespaces, tables, roles, permissions, and assignments of tablets to YB-TServers. These system records are _**replicated across the YB-Masters**_ for redundancy using Raft as well. 

The smart clients query the _**YB-Master**_ for the tablet _**to YB-TServer map and cache it**_. By doing so, the smart clients _**can communicate directly with the correct YB-TServer**_ to serve various queries _**without incurring additional network hops**_.

Some operations are performed throughout the lifetime of the universe, _**in the background**_, without impacting foreground read and write performance.

The _**YB-Master**_ leader _**does the initial placement**_ (at CREATE TABLE time) of tablets across YB-TServers to enforce any user-defined data placement constraints and _**ensure uniform load**_. 

YB-Masters also ensures that each node has a symmetric number of tablet-peer leaders across eligible nodes.

The YB-Master receives _**heartbeats**_ from all the YB-TServers, and tracks their liveness. It detects if any YB-TServers has failed and keeps track of the time interval for which the YB-TServer remains in a failed state. If the time duration of the failure extends beyond a threshold, it _**finds replacement YB-TServers to which**_ the tablet data of the failed YB-TServer is _**rereplicated**_. Rereplication is initiated in a throttled fashion by the YB-Master leader so as to not impact the foreground operations of the universe.
