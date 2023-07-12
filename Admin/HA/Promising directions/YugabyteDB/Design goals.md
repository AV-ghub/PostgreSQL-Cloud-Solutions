## Design goals
> https://docs.yugabyte.com/preview/architecture/design-goals/
### Consistency
Supports distributed transactions while offering strong consistency

### CAP theorem and split-brain
The architectural design of YugabyteDB is similar to ***Google Cloud Spanner***.
> Inside Cloud Spanner and the CAP Theorem
> https://cloud.google.com/blog/products/databases/inside-cloud-spanner-and-the-cap-theorem

### Multi-row ACID transactions
YugabyteDB supports multi-row transactions with three isolation levels: ***Serializable, Snapshot*** (also known as repeatable read), ***and Read Committed*** isolation.

By default Read Committed of YugabyteDB's transactional layer maps to Snapshot isolation, thus making ***Snapshot isolation default*** in YSQL.

***Read Committed*** support in YugabyteDB is currently ***in Beta***.

### YSQL
YSQL is a fully-relational SQL API that is wire-compatible with the SQL language in PostgreSQL. It is ***best fit for RDBMS workloads*** that need horizontal write scalability and global data distribution, while also using relational modeling features such as Joins, distributed transactions, and referential integrity (such as foreign keys). Note that YSQL ***reuses the native query layer of the PostgreSQL*** open source project.

#### In addition:
YSQL supports wide SQL functionality, such as the following:

- All data types
- Built-in functions and expressions
- Joins (inner join, outer join, full outer join, cross join, natural join)
- Constraints (primary key, foreign key, unique, not null, check)
- Secondary indexes (including multi-column and covering columns)
- Distributed transactions (Serializable, Snapshot, and Read Committed Isolation)
- Views
- Stored procedures
- Triggers

### YCQL
YCQL is a semi-relational SQL API that is best suited for internet-scale OLTP and HTAP applications needing massive write scalability and fast queries. YCQL supports distributed transactions, strongly-consistent secondary indexes, and a native JSON column type. YCQL has its roots in the Cassandra Query Language.

### Performance
Written in C++ to ensure high performance and the ability to use large memory heaps (RAM) as an internal database cache, YugabyteDB is optimized primarily to run on SSDs and Non-Volatile Memory Express (NVMe) drives.

### Cloud-native architecture
Ability to run on any public cloud or on-premises data center. This ***includes commodity hardware on bare metal machines, virtual machines, and containers***.
Not having hard external dependencies. For example, YugabyteDB does not rely on atomic clocks, but can use an atomic clock if available.

#### Kubernetes-ready
YugabyteDB works natively in Kubernetes and other containerized environments as a stateful application.

#### Open source
YugabyteDB is open source under the very permissive Apache 2.0 license.



