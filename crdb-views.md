# CockroachDB Views

CockroachDB supports both regular SQL views and materialized views. In this article, I will review the key similarities and differences between them. I will then discuss how they can be leveraged in modern software applications.

A view is a read-only virtual table created based on a SELECT statement that pulls data from one or multiple tables. The result set from this query defines the view. Once created, a view can be queried using SELECT, just like a regular table.

In this article, I'll be using a simple database with two tables to demonstrate the various features and capabilities of CockroachDB views. Before we begin delving into the details, let's look at the database schema.

```sql
CREATE TABLE stations (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    region STRING NOT NULL,
    CONSTRAINT stations_pkey PRIMARY KEY (id ASC),
    INDEX index_region (region ASC)
)
```

```sql
CREATE TABLE datapoints (
    at TIMESTAMP NOT NULL,
    station UUID NOT NULL,
    param0 INT8 NULL,
    param1 INT8 NULL,
    param2 FLOAT8 NULL,
    param3 FLOAT8 NULL,
    param4 STRING NULL,
    CONSTRAINT "primary" PRIMARY KEY (at ASC, station ASC),
    CONSTRAINT datapoints_station_fkey
    FOREIGN KEY (station) REFERENCES public.stations(id)
    ON DELETE CASCADE NOT VALID,
    INDEX datapoints_at (at ASC),
    INDEX datapoints_param0_rec_idx (param0 ASC),
    INDEX datapoints_station_storing_rec_idx (station ASC)
        STORING (param0, param1, param2, param3, param4)
)
```

The `datapoints` table has a composite primary key and a foreign key that references the primary key from the `stations` table. Simply put, a `station` with a unique ID and located in some geographic region logs a timestamped datapoint consisting of two integers, two floats, and a string.

Let's now create our first view.

```sql
CREATE VIEW station_count_by_region AS
    SELECT region, COUNT(*) AS s_count FROM stations
    GROUP BY region ORDER BY region
```

The newly create view will show up in the list of tables:

```sql
> SHOW TABLES;
  schema_name |       table_name        |       type        | owner | estimated_row_count |              locality
--------------+-------------------------+-------------------+-------+---------------------+--------------------------------------
  public      | datapoints              | table             | root  |             8965278 | REGIONAL BY TABLE IN PRIMARY REGION
  public      | station_count_by_region | view              | root  |                   0 | REGIONAL BY TABLE IN PRIMARY REGION
  public      | stations                | table             | root  |                1000 | REGIONAL BY TABLE IN PRIMARY REGION
(3 rows)

Time: 86ms total (execution 85ms / network 1ms
```

And just like with regular tables, we can inspect the view structure:

```sql
> SHOW CREATE TABLE station_count_by_region;
        table_name        |               create_statement
--------------------------+-----------------------------------------------
  station_count_by_region | CREATE VIEW public.station_count_by_region (
                          |     region,
                          |     s_count
                          | ) AS SELECT
                          |         region, count(*) AS s_count
                          |     FROM
                          |         oltaptest.public.stations
                          |     GROUP BY
                          |         region
                          |     ORDER BY
                          |         region
(1 row)

Time: 101ms total (execution 100ms / network 1ms)
```

We can now query our new view:

```sql
> SELECT * FROM station_count_by_region ORDER BY s_count;
           region           | s_count
----------------------------+----------
  EU South (Milan)          |      38
  CA Central (Montreal)     |      39
  US West (N. California)   |      39
  Middle East (UAE)         |      40
  EU West (Ireland)         |      40
  US Mountain (Utah)        |      40
  US East (N. Virginia)     |      40
  South America (São Paulo) |      40
  EU West (London)          |      41
  US East (Ohio)            |      41
  AP South (Mumbai)         |      42
  EU West (Paris)           |      42
  Middle East (Bahrain)     |      42
  AP East (Hong Kong)       |      43
  US West (Oregon)          |      45
  EU North (Stockholm)      |      45
  AP Southeast (Singapore)  |      47
  AP Southeast (Sydney)     |      47
  AP Northeast (Tokyo)      |      48
  AP Northeast (Seoul)      |      49
  Africa (Cape Town)        |      50
  US Central (Iowa)         |      51
  EU Central (Frankfurt)    |      51
(23 rows)

Time: 6ms total (execution 6ms / network 1ms)
```

```sql
> SELECT region FROM station_count_by_region WHERE s_count >= 50;                                            
          region
--------------------------
  Africa (Cape Town)
  EU Central (Frankfurt)
  US Central (Iowa)
(3 rows)

Time: 5ms total (execution 4ms / network 0ms)
```


We can clearly see that, from the application programmer's perspective, querying a view is simpler than querying the underlying table. This becomes especially evident with complex queries that selectively pull columns across multiple tables and perform various aggregations, calculations, ordering, and grouping. Here is a slightly more complex example.


```sql
CREATE VIEW datapoint_aggregations_by_region AS
    SELECT s.region, count(dp.at),
        min(dp.at), max(dp.at), sum(dp.param0), round(avg(dp.param2),5)
    FROM datapoints AS dp JOIN stations AS s
    ON s.id=dp.station GROUP BY s.region
    ORDER BY s.region
```


This view keeps track of the number of datapoints logged by all the stations in each region, along with some statistical data about the datapoints' values. It effectively abstracts the underlying tables and makes consuming these stats much easier and more flexible for the user or application.

```sql
> SELECT * FROM datapoint_aggregations_by_region ORDER BY count;                                             
           region           | count  |            min             |            max             |    sum    |  round
----------------------------+--------+----------------------------+----------------------------+-----------+-----------
  EU South (Milan)          | 340351 | 2024-01-01 00:01:32.609678 | 2025-12-31 21:12:33.453168 | 170458536 | -0.48192
  CA Central (Montreal)     | 349670 | 2024-01-01 00:02:57.980154 | 2025-12-31 01:09:15.627789 | 175302382 | -1.53916
  US West (N. California)   | 350124 | 2024-01-01 00:00:12.516719 | 2025-12-31 16:49:36.597159 | 175319370 |  0.62825
  US East (N. Virginia)     | 358040 | 2024-01-01 00:04:27.843226 | 2025-12-31 17:54:48.949153 | 179662525 | -0.48187
  South America (São Paulo) | 358608 | 2024-01-01 00:00:58.795327 | 2025-12-31 07:00:47.371553 | 179525128 |  0.68684
  EU West (Ireland)         | 358780 | 2024-01-01 00:02:13.307588 | 2025-12-31 11:07:44.458828 | 180006357 | -0.83027
  Middle East (UAE)         | 359131 | 2024-01-01 00:04:19.00066  | 2025-12-31 12:27:50.06722  | 180010196 | -1.19439
  US Mountain (Utah)        | 359201 | 2024-01-01 00:01:45.94707  | 2025-12-31 20:58:29.325941 | 179906621 |  0.26272
  EU West (London)          | 367823 | 2024-01-01 00:00:15.368232 | 2025-12-31 23:36:58.871405 | 184170156 |  0.70646
  US East (Ohio)            | 368165 | 2024-01-01 00:00:40.68811  | 2025-12-31 13:00:36.747867 | 184330097 |  -0.1577
  EU West (Paris)           | 376763 | 2024-01-01 00:01:36.959771 | 2025-12-31 21:33:26.791649 | 188771663 |  0.23122
  Middle East (Bahrain)     | 377314 | 2024-01-01 00:02:50.66522  | 2025-12-31 17:16:53.973532 | 188989857 | -1.57602
  AP South (Mumbai)         | 378091 | 2024-01-01 00:01:10.243655 | 2025-12-31 22:11:12.574419 | 189645615 | -0.01053
  AP East (Hong Kong)       | 385864 | 2024-01-01 00:00:35.968385 | 2025-12-31 20:47:58.457602 | 193312606 |  2.44166
  US West (Oregon)          | 403201 | 2024-01-01 00:00:07.869981 | 2025-12-31 19:06:13.687629 | 202159966 |   0.0564
  EU North (Stockholm)      | 403811 | 2024-01-01 00:01:03.080359 | 2025-12-31 16:59:28.252544 | 202480668 |  0.51228
  AP Southeast (Sydney)     | 420176 | 2024-01-01 00:02:51.279948 | 2025-12-31 22:11:52.821738 | 210640573 | -0.36275
  AP Southeast (Singapore)  | 420736 | 2024-01-01 00:00:28.748414 | 2025-12-31 18:20:55.740461 | 210901653 | -0.82661
  AP Northeast (Tokyo)      | 429476 | 2024-01-01 00:00:51.013368 | 2026-01-01 00:15:51.983925 | 215381118 |  1.39954
  AP Northeast (Seoul)      | 438708 | 2024-01-01 00:03:53.638725 | 2025-12-31 22:45:57.040125 | 219870328 | -0.24195
  Africa (Cape Town)        | 448461 | 2024-01-01 00:00:53.847385 | 2025-12-31 15:29:16.767401 | 224538752 |  0.15731
  EU Central (Frankfurt)    | 457041 | 2024-01-01 00:02:19.140562 | 2025-12-31 20:00:30.427561 | 229046388 |  0.49903
  US Central (Iowa)         | 457199 | 2024-01-01 00:00:28.579195 | 2025-12-31 21:53:40.025324 | 229038926 | -0.58941
(23 rows)

Time: 16.185s total (execution 16.184s / network 0.001s)
```


Since views act as a buffer between the underlying tables and the queries run against them, they can serve as an effective mechanism for controlling data exposure. For example, the station IDs in the `stations` and `datapoints` tables are not exposed via the view. Therefore, a user who has access to the view but not the underlying tables will not be able to see them. Let's demonsterate this on an example.

Create a new user:

```sql

```




In real world applicaiton, this is essential in protecting Personal Private Information (PPI) such as Social Security Numbers (SSN), dates of birth, medical information, etc.


3. **Both Can Be Used for Security and Access Control**  
   - Views and materialized views can restrict access to sensitive data by **hiding specific columns or rows** from users.  

5. **Both Can Improve Query Performance (in Different Ways)**  
   - While materialized views store results physically for faster access, **even standard views** can enhance performance by encapsulating optimized queries.  

6. **Both Can Be Based on Multiple Tables**  
   - Views and materialized views can **combine multiple tables** using joins, aggregations, and transformations to present data in a structured format.  

7. **Both Can Be Used in the Same Queries**  
   - You can query a view just like a table, and materialized views function similarly, making them easy to integrate into applications.  

### **Key Difference**  
- **Views** always pull the latest data dynamically from the base tables.  
- **Materialized Views** store data physically and need to be **refreshed** to reflect updates.  

Would you like an example to illustrate their similarities and differences? 😊


## Views

An **SQL view** is a **virtual table** that is based on the result of a SQL query. Unlike physical tables, views do not store data themselves. Instead, they dynamically present data from one or more underlying tables whenever they are queried.  

## What Are Views Used For?

1. Simplifying Complex Queries
Views allow you to encapsulate complex SQL logic, so users can access the results without writing long queries repeatedly.  

2. Enhancing Security and Access Control
Views can restrict access to specific columns or rows, ensuring that users only see the data they are authorized to view.  

3. Providing Data Abstraction
Views offer a layer of abstraction by presenting a consistent structure to applications, even if the underlying tables change.  

4. Improving Maintainability
Instead of modifying queries across multiple applications, you can update a view in one place to apply changes system-wide.  

5. Facilitating Data Aggregation and Reporting
Views make it easy to create summary tables for reporting, combining multiple sources into a single, readable format.  


## Materialized views


### **What Are Materialized Views?**  
A **materialized view** is a **physical, stored result** of a query, unlike a regular view, which is just a virtual representation of data. The materialized view is computed and stored in the database, meaning it does not dynamically update with every query like a standard view. Instead, it must be **refreshed** explicitly or on a schedule.  

## What Are Materialized Views Used For?

1. Performance Optimization
Since materialized views store precomputed results, queries run much faster, reducing the load on the database.  

2. Reducing Computation Costs
Queries that involve heavy joins, aggregations, or calculations can be precomputed and stored, avoiding repeated expensive operations.  

3. Supporting Analytics & Reporting
Materialized views are useful for real-time or near-real-time analytics, as they provide a snapshot of data without needing to reprocess large datasets.  

4. Decoupling Transactional and Analytical Workloads
Instead of running analytics directly on live transactional tables, materialized views can provide a separate, optimized dataset for analytics while keeping transactional performance intact.  

5. Indexing for Faster Query Performance
CockroachDB allows materialized views to have indexes, making queries even more efficient.

## Key Differences: View vs. Materialized View  
| Feature | Standard View | Materialized View |
|---------|--------------|------------------|
| Data Storage | No (virtual) | Yes (physical) |
| Query Performance | Depends on the underlying query | Faster (precomputed results) |
| Updates | Always up-to-date | Must be refreshed manually or on a schedule |
| Use Case | Simplifying queries, security | Speeding up analytics, reducing expensive computations |






## Code Examples


```sql
CREATE MATERIALIZED VIEW datapoints_json AS
SELECT CAST(
    format(
    '{"at": "%s", "station": "%s", "param0": %s, "param1": %s, "param2": %s, "param3": %s, "param4": "%s"}',
    at, station, param0, param1, param2, param3, param4
    ) AS jsonb
) AS datapoint
FROM datapoints
```


```sql
SELECT jsonb_pretty(datapoint) AS datapoing FROM datapoints_json LIMIT 10
```

```sql
SELECT jsonb_pretty(datapoint)  AS param4 FROM datapoints_json WHERE datapoint->'param0' = '20' ORDER BY datapoint->'at' LIMIT 5
```



```sql
SELECT COUNT(*) AS s_count, region
FROM stations                                                           
GROUP BY region ORDER BY s_count
```


```sql
SELECT s.region, count(dp.at), min(dp.at), max(dp.at), sum(dp.param0),
round(avg(dp.param2),5)
FROM datapoints AS dp JOIN stations AS s ON s.id=dp.station
GROUP BY s.region ORDER BY s.region
```


```sql
WITH t AS (
    SELECT generate_series  (
        (SELECT date(now()))::timestamp,
        (SELECT date(now()))::timestamp + '1 day' - '1 hour',
        '1 hour'::interval
    ) :: timestamp AS period
)
SELECT t.period, count(dp.at)
FROM t AS t LEFT JOIN datapoints AS dp
ON t.period <= dp.at AND dp.at < t.period +'1 hour'
GROUP BY t.period ORDER BY t.period
```


```sql
WITH t AS (
    SELECT generate_series  (
        (SELECT date(now()))::timestamp,
        (SELECT date(now()))::timestamp + '1 day' - '1 hour',
        '1 hour'::interval
    ) :: timestamp AS period
)
SELECT t.period, array_to_json(array_agg(row_to_json(dp)))
FROM t AS t LEFT JOIN datapoints AS dp                    
ON t.period <= dp.at AND dp.at < t.period +'1 hour'
GROUP BY t.period ORDER BY t.period
```
