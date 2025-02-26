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
    GROUP BY region ORDER BY region;
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
    ORDER BY s.region;
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
CREATE ROLE view_reader WITH PASSWORD NULL
```

```sql
> SHOW ROLES;                                                              
   username   | options | member_of
--------------+---------+------------
  admin       |         | {}
  root        |         | {admin}
  view_reader |         | {}
(3 rows)
```

```sql
> SHOW GRANTS ON station_count_by_region;
  database_name | schema_name |       table_name        | grantee | privilege_type | is_grantable
----------------+-------------+-------------------------+---------+----------------+---------------
  oltaptest     | public      | station_count_by_region | admin   | ALL            |      t
  oltaptest     | public      | station_count_by_region | root    | ALL            |      t
(2 rows)
```

```sql
GRANT SELECT ON station_count_by_region TO view_reader
```

```sql
>  SHOW GRANTS ON station_count_by_region;                                                                   
  database_name | schema_name |       table_name        |   grantee   | privilege_type | is_grantable
----------------+-------------+-------------------------+-------------+----------------+---------------
  oltaptest     | public      | station_count_by_region | admin       | ALL            |      t
  oltaptest     | public      | station_count_by_region | root        | ALL            |      t
  oltaptest     | public      | station_count_by_region | view_reader | SELECT         |      f
(3 rows)
```

Logged in now as the `view_reader` user, I can `SELECT` from the view while the underlying tables remain unaccessible to me.

```sql
> SELECT * FROM station_count_by_region;
           region           | s_count
----------------------------+----------
  AP East (Hong Kong)       |      43
  AP Northeast (Seoul)      |      49
  AP Northeast (Tokyo)      |      48
  AP South (Mumbai)         |      42
  AP Southeast (Singapore)  |      47
  AP Southeast (Sydney)     |      47
  Africa (Cape Town)        |      50
  CA Central (Montreal)     |      39
  EU Central (Frankfurt)    |      51
  EU North (Stockholm)      |      45
  EU South (Milan)          |      38
  EU West (Ireland)         |      40
  EU West (London)          |      41
  EU West (Paris)           |      42
  Middle East (Bahrain)     |      42
  Middle East (UAE)         |      40
  South America (São Paulo) |      40
  US Central (Iowa)         |      51
  US East (N. Virginia)     |      40
  US East (Ohio)            |      41
  US Mountain (Utah)        |      40
  US West (N. California)   |      39
  US West (Oregon)          |      45
(23 rows)
```

```sql
> SELECT * FROM stations;                                                                             
ERROR: user view_reader does not have SELECT privilege on relation stations
SQLSTATE: 42501
```

```sql
> SELECT * FROM datapoints;                                                                           
ERROR: user view_reader does not have SELECT privilege on relation datapoints
SQLSTATE: 42501
```


>[!WARNING]
> Don't forget to log out and log back in as `root` user to continue.


## Abstracting Underlaying Table Complexity

As the applications supported by the database(s) evolve, the tables may need alterations, like adding new columns or changing column type. Let's consider an application that queries the `datapoints` table for ten random row using a wildcard for the columns:

```sql
> SELECT * FROM datapoints ORDER BY random() LIMIT 10;                                                             
              at             |               station                | param0 | param1 |  param2  | param3  |              param4
-----------------------------+--------------------------------------+--------+--------+----------+---------+-----------------------------------
  2024-04-22 13:37:24.650472 | 43e02023-318a-456b-afcc-f942b9a2535c |    232 |    431 | -236.574 | -267.83 | J6L4EZ00OVJ2FYOH7O3DKAG9N2FM1YCX
  2024-01-04 09:27:44.055629 | d043a889-2543-49e0-b14c-bf162d870138 |    412 |   -439 | -995.707 |  722.17 | 7YYR5MJLB130V1L
  2024-01-01 19:47:07.569224 | 59d9a32f-bb9b-4e68-ae29-3b07fd63b8fa |    822 |    306 | -540.151 |  661.79 | ZJHI584OM7EA2HGKTNMED36YA6A
  2024-09-29 15:41:22.547995 | 6c4d1c8d-ae07-4c41-867b-a94c39fc0dce |    207 |    199 |   966.73 |  852.67 | ADSFZ814BDT0WJA9D5C6BAQ
  2024-11-21 12:54:58.784143 | eb45253f-40f7-49e0-928e-5e992a2f6f29 |    963 |    909 | -592.247 |  338.93 | Z2T0IO50GUBKNK3OLXUQWZ
  2024-09-16 13:49:08.088792 | 15e6388c-d6cb-4ab7-a1c9-74279ed85b65 |     42 |   -661 |  558.163 | -512.06 | 6HTU762POZC26RPAJGIQ0OY5IVD
  2024-11-17 15:13:10.101037 | 442f7037-e1d4-41f4-96a2-3267985db530 |    546 |    612 | -526.802 |  328.56 | 7VOK9W8B8XI2DS4F78MT7
  2024-01-11 20:23:38.951431 | 65007721-e6c6-42b9-83bc-794b3ec49fb5 |    561 |   -198 | -352.847 |  -393.2 | MRM1EUYL6X3LKJEAGO76D
  2024-10-07 12:26:31.833473 | aeba590d-23b1-483f-8160-d6484e57e319 |    896 |   -199 |  829.801 |  285.29 | 5YG7GFIJYVBXWKWM8
  2024-01-07 07:23:58.807308 | b340926f-1e93-4b13-8ab6-e909bcf61251 |    492 |   -922 |  879.538 |   168.8 | JZBV2FW58FS6RNOV4WRX4P22SZ4ZQH5
(10 rows)

Time: 4.435s total (execution 4.434s / network 0.001s)
```

The application logic expects the result set to have seven columns. Now let's imagine that a new column is added to the `datapoints` tables:

```sql
> ALTER TABLE datapoints ADD COLUMN param5 JSONB DEFAULT '{}';
```

If the application logic is not prepared to handle an eight-column results set from the same `SELECT` query, the change in the table structure we just introduced has broke the application.

```sql
> SELECT * FROM datapoints ORDER BY random() LIMIT 10;                                                             
              at             |               station                | param0 | param1 |  param2  | param3  |             param4              | param5
-----------------------------+--------------------------------------+--------+--------+----------+---------+---------------------------------+---------
  2024-07-31 13:48:54.931012 | 4355b1e5-8ea6-4c0a-af60-3753dab17216 |    185 |   -871 |   635.05 | -710.54 | BL96AYU7UOAB8P                  | {}
  2024-03-07 08:08:09.851961 | 616ca193-6b44-4f3e-ab13-e97a62a7ef6a |    800 |     11 |   419.02 |  263.11 | W8YHWQAAFMVAU                   | {}
  2024-04-30 12:07:55.096663 | e0b0f651-22fa-41bd-9655-188ce7084118 |    987 |   -359 | -398.192 |  152.57 | AUIM0N46W5BEXEC                 | {}
  2024-02-03 05:39:25.867489 | 86f8abc5-ae00-4de6-b028-f3a6e3372488 |    796 |   -261 | -396.695 | -805.73 | K3FMYHDIVL53FWM2NMO1G           | {}
  2024-02-04 21:43:47.266435 | e8b67169-7227-46f6-9773-138f4186ad96 |     99 |    804 |  477.407 |  106.78 | V6QLTK3CAD7DLY                  | {}
  2024-10-05 19:03:10.919085 | fd867da2-5cb5-4d22-b67c-f3fe7595808e |    684 |    138 | -360.136 | -522.33 | QD5VJ5V065HXYLRHO082            | {}
  2024-02-26 15:00:40.049237 | c863dcfe-7f0a-4c69-82b0-676e5ba540e9 |    744 |    253 |   99.463 | -310.16 | 1LJMYAHT                        | {}
  2024-12-15 16:25:40.518032 | 578adbd8-139f-435d-9fc7-d331fe5a43ef |    169 |    809 |   879.19 | -912.75 | TG1FTKU451E9Z056YUI93O5MFI61UAN | {}
  2024-08-16 04:54:21.498491 | 733225a2-ea0b-4d43-861b-c861e04263f7 |    978 |    -71 | -466.731 |  613.45 | D4ZE8XR2SY6H1VBAR49OQWS         | {}
  2024-03-04 20:43:42.718847 | e168f135-2ea9-4b7e-b94a-f792888f560f |    744 |    690 |   39.526 |  392.86 | WWRY6B4TE                       | {}
(10 rows)

Time: 9.920s total (execution 9.919s / network 0.001s)
```

To ensure that schema change don't negatively affect the applications using our database, we can decouple the two using a view.

```sql
> CREATE VIEW datapoints_view AS SELECT at, station, param0, param1, param2, param3, param4 FROM datapoints;
```

Using this new view instead of the underlying table directly protect the application logic from breaking if schema changes are introduced over time.

```sql
> SELECT * FROM datapoints_view ORDER BY random() LIMIT 10;                                                        
              at             |               station                | param0 | param1 |  param2  | param3  |           param4
-----------------------------+--------------------------------------+--------+--------+----------+---------+------------------------------
  2024-08-25 14:04:17.994741 | 2eafe51e-48d5-4fd5-a48b-24011c93761d |    451 |    871 |   639.16 |   -5.22 | 0HDGG0XFQS94DH1
  2024-08-14 14:58:03.715754 | 44abb1a9-4b58-4176-95e6-f4768d3c1ccc |    222 |   -601 | -633.222 |  -480.4 | JX88IT3MP6L3Q0L0GPIT
  2024-01-07 15:03:52.48959  | 00700ae4-9b0f-4254-b374-07f7da901b7e |    625 |    539 | -588.502 | -303.34 | NXM0DV46Y3CRUQQG
  2024-06-16 14:07:12.820695 | c8a97e4b-b967-4faa-a256-9f9d026a148a |    507 |    633 |  563.489 |  208.52 | EVGYJAVCICY9KFDD999QT8WQ1S
  2024-01-09 05:33:46.011819 | 2c276cdd-a5fa-4472-816f-875f4a34c4c1 |    585 |    718 |   82.759 | -208.72 | EID9P2RQHFAUA
  2024-08-27 16:51:51.984799 | becdf27a-99a2-4fbf-90ed-8970b3dc0eeb |     33 |    719 |     26.6 | -524.14 | FJRQMLPYM5UI5W9M6G7YU
  2024-01-12 18:00:33.384313 | fdd6c283-b0b3-4c8e-9119-cbc60811f5ac |    146 |   -726 |  527.435 | -668.44 | 2OKMRASRT91IPWXNSLMLER
  2024-01-11 14:49:10.817618 | 1fe82a22-d7ae-4c79-80a4-d4e2f604a76f |    566 |    672 |  983.465 | -536.14 | T24VYSZWGTE
  2024-05-04 15:47:56.163271 | 876fff3f-f596-4d35-8761-3bcabcb6814b |    776 |   -976 |    534.9 |  503.45 | IDCCUUEGTYH98VI0MI0X15M7ZY6
  2024-08-02 14:30:04.514187 | 155765ec-1911-4905-834d-ec303c61b1e1 |     89 |   -163 |   -18.21 |  730.63 | 2HHY98YIN
(10 rows)

Time: 17.584s total (execution 17.583s / network 0.001s)
```



# Materialized Views


## Materialized Views and CDC

>[!NOTE]
> Here I'm going to talk about creating a CHANGEFEED based on a materialized view since an MV is like a table...





5. **Both Can Improve Query Performance (in Different Ways)**  
   - While materialized views store results physically for faster access, **even standard views** can enhance performance by encapsulating optimized queries.  



### **Key Difference**  
- **Views** always pull the latest data dynamically from the base tables.  
- **Materialized Views** store data physically and need to be **refreshed** to reflect updates.  

Would you like an example to illustrate their similarities and differences? 😊


## Views

## What Are Views Used For?


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
