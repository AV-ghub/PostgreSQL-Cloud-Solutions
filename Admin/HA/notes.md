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
