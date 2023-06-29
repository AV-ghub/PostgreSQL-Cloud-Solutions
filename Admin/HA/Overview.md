> https://severalnines.com/blog/how-deploy-postgresql-high-availability/

### You can implement two types of Standby databases, based on the nature of the replication:
- Logical Standbys – The replication between Primary and Standby is made via SQL statements.
- Physical Standbys – The replication between Primary and Standby is made via the internal data structure modifications.

### Primary-Standby Architectures
From ***version 10*** on, PostgreSQL includes a built-in option to set up ***logical replication***

Primary-Standby setup is not enough to effectively ensure high availability, as you also need to handle failures.
PostgreSQL itself does not include an automatic failover mechanism
your application needs to be notified accordingly to start using the new Primary

### Primary-Primary Architectures
PostgreSQL does not yet support this architecture “natively”
#### Tools
> https://severalnines.com/blog/top-pg-clustering-high-availability-solutions-postgresql/

### How to Deploy PostgreSQL for High Availability
> https://severalnines.com/blog/how-deploy-postgresql-high-availability/
#### Load Balancing
Not only are these tools helpful in balancing the load of your databases, but they also help applications get redirected to the available/healthy nodes and even specify ports with different roles.

***HAProxy*** is a load balancer that distributes traffic from one origin to one or more destinations and can define specific rules and/or protocols for this task. If any of the destinations stop responding, they are marked as offline, and the traffic is sent to the rest of the available destinations.

***Keepalived*** is a service that allows you to configure a virtual IP address within an active/passive group of servers. This virtual IP address is assigned to an active server. If this server fails, the IP address is automatically migrated to the “Secondary” passive server, allowing it to continue working with the same IP address in a transparent way for the systems.

### How to Achieve PostgreSQL High Availability with pgBouncer
> https://severalnines.com/blog/how-achieve-postgresql-high-availability-pgbouncer/
