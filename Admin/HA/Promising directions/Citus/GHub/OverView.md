## Citus Columnar
> https://github.com/citusdata/citus/blob/main/src/backend/columnar/README.md

### Limitations
- Support for PostgreSQL server ***versions 12+*** only
- ***Append-only*** (no UPDATE/DELETE support)
- No support for ***serializable*** isolation level
- No support for ***foreign keys***

### Common
Note that VACUUM (without FULL) is much faster on a columnar table, because it scans only the metadata, and not the actual data.

### Install PostgreSQL 15 and the Citus extension
> https://docs.citusdata.com/en/stable/installation/single_node.html
> https://docs.citusdata.com/en/stable/installation/single_node_debian.html

