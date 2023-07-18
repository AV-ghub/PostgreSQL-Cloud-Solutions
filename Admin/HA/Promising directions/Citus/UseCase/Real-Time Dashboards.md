> https://docs.citusdata.com/en/v11.3/use_cases/realtime_analytics.html

Citus provides real-time queries over large datasets. One workload we commonly see at Citus involves powering real-time dashboards of event data.

### Data Model
The data we’re dealing with is an immutable stream of log data. We’ll insert directly into Citus but it’s also common for this data to first be routed through something like **Kafka**. Doing so has the usual advantages, and makes it easier to pre-aggregate the data once data volumes become unmanageably high.

Once you’re ingesting data, you can run dashboard queries such as:
```
SELECT
  site_id,
  date_trunc('minute', ingest_time) as minute,
  COUNT(1) AS request_count,
  SUM(CASE WHEN (status_code between 200 and 299) THEN 1 ELSE 0 END) as success_count,
  SUM(CASE WHEN (status_code between 200 and 299) THEN 0 ELSE 1 END) as error_count,
  SUM(response_time_msec) / COUNT(1) AS average_response_time_msec
FROM http_request
WHERE date_trunc('minute', ingest_time) > now() - '5 minutes'::interval
GROUP BY site_id, minute
ORDER BY minute ASC;
```
The setup described above works, but has two drawbacks:
- Your HTTP analytics **dashboard must go over each row every time it needs** to generate a graph. For example, if your clients are interested in trends over the past year, your queries will aggregate every row for the past year from scratch.
- Your storage **costs will grow proportionally** with the ingest rate and the length of the queryable history. In practice, you may want to keep raw events for a shorter period of time (one month) and look at historical graphs over a longer time period (years).

### Rollups
You can overcome both drawbacks by rolling up the raw data into a **pre-aggregated** form. 

You could do this by adding a **crontab** entry on the coordinator node:
```
* * * * * psql -c 'SELECT rollup_http_request();'
```
Alternatively, an extension such as **pg_cron** allows you to schedule recurring queries directly from the database.

### Expiring Old Data
use standard queries to delete expired data

In production you could wrap these queries in a function and call it every minute in a cron job.

### Approximate Distinct Counts
A datatype called **hyperloglog**, or HLL, can answer the query approximately; it takes a surprisingly small amount of space to tell you approximately how many unique elements are in a set. 

> https://github.com/citusdata/postgresql-hll

First you must install the HLL extension; the github repo has instructions. Next, you have to enable it:
```
CREATE EXTENSION hll;
```
First add a column to the rollup table.
```
ALTER TABLE http_request_1min ADD COLUMN distinct_ip_addresses hll;
```

Next use our custom aggregation to populate the column. Just add it to the query in our rollup function:
```
@@ -1,10 +1,12 @@
  INSERT INTO http_request_1min (
    ...
+   , distinct_ip_addresses
  ) SELECT
    ...
+   , hll_add_agg(hll_hash_text(ip_address)) AS distinct_ip_addresses
  FROM http_request
```
Dashboard queries are a little more complicated, you have to read out the distinct number of IP addresses by calling the hll_cardinality function:
```
SELECT ...,
       hll_cardinality(distinct_ip_addresses) AS distinct_ip_address_count
  FROM http_request_1min
 WHERE ingest_time > date_trunc('minute', now()) - interval '5 minutes';
```
### Unstructured Data with JSONB
First, add the new column to our rollup table:
```
ALTER TABLE http_request_1min ADD COLUMN country_counters JSONB;
```
Next, include it in the rollups by modifying the rollup function:
```
  INSERT INTO http_request_1min (
    ...
+   , country_counters
  ) SELECT
    ...
    SUM(response_time_msec) / COUNT(1) AS average_response_time_msec
- FROM http_request
+   , jsonb_object_agg(request_country, country_count) AS country_counters
+ FROM (
+   SELECT *,
+     count(1) OVER (
+       PARTITION BY site_id, date_trunc('minute', ingest_time), request_country
+     ) AS country_count
+   FROM http_request
+ ) h
```
Now, if you want to get the number of requests which came from America in your dashboard, you can modify the dashboard query to look like this:
```
SELECT  ...
  COALESCE(country_counters->>'USA', '0')::int AS american_visitors
FROM http_request_1min
WHERE ingest_time > date_trunc('minute', now()) - '5 minutes'::interval;
```
