## Citus Columnar
> https://github.com/citusdata/citus/blob/main/src/backend/columnar/README.md

### Limitations
- Support for PostgreSQL server ***versions 12+*** only
- ***Append-only*** (no UPDATE/DELETE support)
- No support for ***serializable*** isolation level
- No support for ***foreign keys***
> https://www.citusdata.com/blog/2021/03/06/citus-10-columnar-compression-for-postgres/

>[UPDATE in Sep 2021]: Good news, as of Citus 10.2, Citus columnar now supports _**btree and hash indexes and the constraints requiring them (e.g.: primary key)**_. So as of 10.2 or later, Citus columnar allows you to create such indexes and do plain index-scan on those indexes when it’s beneficial, but does not yet support index-only and bitmap scans.

### Common
Note that VACUUM (without FULL) is much faster on a columnar table, because it scans only the metadata, and not the actual data.

### Install PostgreSQL 15 and the Citus extension
> https://docs.citusdata.com/en/stable/installation/single_node.html
> https://docs.citusdata.com/en/stable/installation/single_node_debian.html

