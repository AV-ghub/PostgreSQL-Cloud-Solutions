> https://docs.citusdata.com/en/v11.3/use_cases/multi_tenant.html#let-s-make-an-app-ad-analytics

### Common
For SaaS applications, each tenant’s data can be stored together in a single database instance and kept isolated from and invisible to other tenants. 

This is efficient in three ways. 

***First***, application improvements apply to all clients. 
***Second***, sharing a database between tenants uses hardware efficiently. 
***Last***, it is much simpler to manage a single database for all tenants than a different database server for each tenant.

Client code requires minimal modifications and can continue to use full SQL capabilities.

Citus allows users to write multi-tenant applications as if they are connecting to a single PostgreSQL database, when in fact the database is a horizontally scalable cluster of machines. Client code requires minimal modifications and can continue to use full SQL capabilities.

### Scaling the Relational Data Model
Multi-tenant applications have a nice property that we can take advantage of: ***queries usually always request information for one tenant at a time, not a mix of tenants***.

We do this in Citus by making sure every table in our schema has a column to clearly mark which tenant owns which rows. In the ad analytics application the tenants are companies, so we must ensure all tables have a **company_id** column.

In Citus’ terminology company_id will be the **distribution column**

### Preparing Tables and Ingesting Data
The schema we have created so far uses **a separate id column as primary key for each table**. Citus **requires that primary and foreign key constraints include the distribution column**. 

Once the schema is ready, we can tell Citus to **create shards** on the workers. From the coordinator node, run:
```
SELECT create_distributed_table('companies',   'id');
SELECT create_distributed_table('campaigns',   'company_id');
SELECT create_distributed_table('ads',         'company_id');
SELECT create_distributed_table('clicks',      'company_id');
SELECT create_distributed_table('impressions', 'company_id');
```
he create_distributed_table function informs Citus that a table should be distributed among nodes and that future incoming queries to those tables should be planned for distributed execution. The function also creates shards for the table on worker nodes, which are low-level units of data storage Citus uses to assign data to nodes.

 ### Sharing Data Between Tenants 

Sometimes there is data that can be shared by all tenants, and doesn’t “belong” to any tenant in particular. 

We do this in Citus by designating geo_ips as a **reference table**.
```
-- Make synchronized copies of geo_ips on all workers
SELECT create_reference_table('geo_ips');
```
Reference tables are replicated across all worker nodes, and Citus automatically keeps them in sync during modifications. 

### Online Changes to the Schema
Another challenge with multi-tenant systems is keeping the schemas for all the tenants in sync. Any schema change needs to be consistently reflected across all the tenants. In Citus, you can simply use standard PostgreSQL DDL commands to change the schema of your tables, and Citus will propagate them from the coordinator node to the workers using a **two-phase commit** protocol.

### When Data Differs Across Tenants
PostgreSQL provides a much easier way with its unstructured column types, notably JSONB.
```
SELECT
  user_data->>'is_mobile' AS is_mobile,
  count(*) AS count
FROM clicks
WHERE company_id = 5
GROUP BY user_data->>'is_mobile'
ORDER BY count DESC;
```
The database administrator can even create a **partial index** to improve speed for an individual tenant’s query patterns. Here is one to improve company 5’s filters for clicks from users on mobile devices:
```
CREATE INDEX click_user_data_is_mobile
ON clicks ((user_data->>'is_mobile'))
WHERE company_id = 5;
```
Additionally, PostgreSQL supports **GIN indices on JSONB**. Creating a GIN index on a JSONB column will create an index on every key and value within that JSON document. This speeds up a number of JSONB operators such as ?, ?|, and ?&.
```
CREATE INDEX click_user_data
ON clicks USING gin (user_data);

-- this speeds up queries like, "which clicks have
-- the is_mobile key present in user_data?"

SELECT id
  FROM clicks
 WHERE user_data ? 'is_mobile'
   AND company_id = 5;
```
### Scaling Hardware Resources
Being able to **rebalance data** in the Citus cluster allows you to grow your data size or number of customers and improve performance on demand. **Adding new machines** allows you to keep data in memory even when it is much larger than what a single machine can store.

Also, if data increases for only a few large tenants, then **you can isolate those particular tenants** to separate nodes for better performance.

To scale out your Citus cluster, first **add a new worker node** to it. To move your existing data, you can ask Citus to rebalance the data. 
```
SELECT citus_rebalance_start();
```
### Dealing with Big Tenants
To improve resource allocation and make guarantees of tenant QoS it is worthwhile to **move large tenants to dedicated nodes**. Citus provides the tools to do this.

First isolate the tenant’s data to a dedicated shard suitable to move. The CASCADE option also applies this change to the rest of our tables distributed by company_id.
```
SELECT isolate_tenant_to_new_shard(
  'companies', 5, 'CASCADE'
);
```
Next we move the data across the network to a new dedicated node.



































