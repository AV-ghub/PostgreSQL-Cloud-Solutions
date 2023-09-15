## How to Deploy PostgreSQL for High Availability
> https://severalnines.com/blog/how-deploy-postgresql-high-availability/

You can implement **two types of Standby databases**, based on the nature of the replication:
- Logical Standbys – The replication between Primary and Standby is made via SQL statements.
- Physical Standbys – The replication between Primary and Standby is made via the internal data structure modifications.

### Primary-Standby Architectures
In the case of PostgreSQL, a **stream of write-ahead log (WAL)** records is used to keep the Standby databases synchronized. This can be **synchronous or asynchronous**, and the entire database server is replicated.

Primary-Standby setup is not enough to effectively ensure high availability, as you also need to handle failures.
PostgreSQL itself does not include an automatic failover mechanism, so this will **require some custom script or third-party tools** for this automation.
Your application needs to be notified accordingly to start using the new Primary.

From ***version 10*** on, PostgreSQL includes a built-in option to set up ***logical replication***

### Primary-Primary Architectures
PostgreSQL does not yet support this architecture “natively”
#### Tools
> https://severalnines.com/blog/top-pg-clustering-high-availability-solutions-postgresql/

### Load Balancing
Not only are these tools helpful in **balancing the load** of your databases, but they also help applications **get redirected to the available/healthy nodes** and even **specify ports with different roles**.

***HAProxy*** is a load balancer that distributes traffic from one origin to one or more destinations and can define specific rules and/or protocols for this task. If any of the destinations stop responding, they are marked as offline, and the traffic is sent to the rest of the available destinations.

***Keepalived*** is a service that allows you to configure a virtual IP address within an active/passive group of servers. This virtual IP address is assigned to an active server. If this server fails, the IP address is automatically migrated to the “Secondary” passive server, allowing it to continue working with the same IP address in a transparent way for the systems.

## How to Achieve PostgreSQL High Availability with pgBouncer
> https://severalnines.com/blog/how-achieve-postgresql-high-availability-pgbouncer/

### Scaling Connections in PostgreSQL Using Connection Pooling

#### Reasons for Connection Pooling
1. To avoid crashing your server. PostgreSQL servers are **limited to a number of clients** they handle at a time depending on the memory parameter. If this number is surpassed, then you will end up crashing the server. With connection pooling, clients use a set number of connections.
2. Normally, database requests are **executed in a serial manner** with a criterion of first-in first-out. Therefore the approach should be to make a single connection with pipelined requests which can be executed simultaneously rather than each at a time.
3. Improve on security. Often a connection involves a **handshake that may take 25-35 ms** average over which an SSL is established, passwords are checked and sharing of the configuration information. All this work for each connected user will result in **extensive usage of memory**. However, with connection pooling, the number of connections is reduced hence saving on memory.

### How PgBouncer Works
When PgBouncer receives a connection, it **performs the authentication**, which depends on the method specified in the configuration file. PgBouncer **supports all the authentication mechanisms that the PostgreSQL server supports**. After this, PgBouncer **checks for a cached connection**, with the same username+database combination. If a cached connection is found, it returns the connection to the client, if not, it **creates a new connection**. 

#### The PgBouncer behavior depends on the pooling mode configured:
* **session** pooling (default)
* **transaction** pooling
* **statement** pooling

To balance queries between several servers, on the PgBouncer side, it may be a good idea to make [**server_lifetime**](https://dba.stackexchange.com/questions/172206/what-is-the-sense-to-use-server-lifetime-with-pgbouncer) smaller and also turn [**server_round_robin**](https://postgrespro.ru/docs/postgrespro/10/pgbouncer#PGBOUNCER-SERVER-ROUND-ROBIN) on. By default, idle connections are reused by the LIFO algorithm, which may work not so well when a load-balancer is used.
> The server_lifetime enforces returning allocated sources to operation system. One hour is conservative value - and can be shorter - 10 minutes. pgbouncer close only not active connection - so **if the connection is used by any client**: opened transaction, long active command, .. **server_lifetime cannot be applied**.

> [PgBouncer **pgbouncer.ini** official documentation](https://www.pgbouncer.org/config.html).


