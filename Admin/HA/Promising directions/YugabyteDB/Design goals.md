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

