
## 物化视图


### 物化视图源表\-\-基础数据源


创建源表，因为我们的目标涉及报告聚合数据而不是单条记录，所以我们可以解析它，将信息传递给物化视图，并丢弃实际传入的数据。这符合我们的目标并节省了存储空间，因此我们将使用`Null`表引擎。



```
CREATE DATABASE IF NOT EXISTS analytics;

CREATE TABLE analytics.hourly_data
(
    `domain_name` String,
    `event_time` DateTime,
    `count_views` UInt64
)
ENGINE = Null;

```

注意：可以在Null表上创建物化视图。因此，写入表的数据最终会影响视图，但原始原始数据仍将被丢弃


### 月度汇总表和物化视图


对于第一个物化视图，需要创建 `Target` 表(本例子中为`analytics.monthly_aggregated_data`），例中将按月份和域名存储视图的总和。



```
CREATE TABLE analytics.monthly_aggregated_data
(
    `domain_name` String,
    `month` Date,
    `sumCountViews` AggregateFunction(sum, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY (domain_name, month);

```

将转发`Target`表上数据的物化视图如下：



```
CREATE MATERIALIZED VIEW analytics.monthly_aggregated_data_mv
TO analytics.monthly_aggregated_data
AS
SELECT
    toDate(toStartOfMonth(event_time)) AS month,
    domain_name,
    sumState(count_views) AS sumCountViews
FROM analytics.hourly_data
GROUP BY domain_name, month;

```

### 年度汇总表和物化视图


现在，创建第二个物化视图，该视图将链接到之前的目标表`monthly_aggregated_data`。
首先，创建一个新的目标表，该表将存储每个域名每年汇总的视图总和。



```
CREATE TABLE analytics.year_aggregated_data
(
    `domain_name` String,
    `year` UInt16,
    `sumCountViews` UInt64
)
ENGINE = SummingMergeTree()
ORDER BY (domain_name, year);

```

然后创建物化视图，此步骤定义级联。`FROM` 语句将使用`monthly_aggregated_data`表，这意味着数据流将是：
1\.数据到达`hourly_data`表。
2\.ClickHouse会将收到的数据转发到第一个物化视图`monthly_aggregated_data` 表
3\.最后，步骤2中接收到的数据将被转发到 `year_aggregated_data`。



```
CREATE MATERIALIZED VIEW analytics.year_aggregated_data_mv
TO analytics.year_aggregated_data
AS
SELECT
    toYear(toStartOfYear(month)) AS year,
    domain_name,
    sumMerge(sumCountViews) as sumCountViews
FROM analytics.monthly_aggregated_data
GROUP BY domain_name, year;

```

注意：


在使用物化视图时，一个常见的误解是数据是从表中读取的，这不是`Materialized views`的工作方式；转发的数据是插入的数据块，而不是表中的最终结果。


想象一下，在这个例子中，`monthly_aggregated_data`中使用的引擎是一个折叠合并树(`CollapsingMergeTree`)，转发到第二个物化视图`year_aggregated_data_mv` 的数据将不是折叠表的最终结果，它将转发具有正如`SELECT… GROUP BY`中定义的字段的数据块。


如果末正在使用`CollapsingMergeTree`、`ReplacingMergeTree`，甚至`SummingMergeTree`，并且计划创建级联物化视图，则需要了解此处描述的限制。


### 采集数据


现在是时候通过插入一些数据来测试我们的级联物化视图了:



```
INSERT INTO analytics.hourly_data (domain_name, event_time, count_views)
VALUES ('clickhouse.com', '2019-01-01 10:00:00', 1),
       ('clickhouse.com', '2019-02-02 00:00:00', 2),
       ('clickhouse.com', '2019-02-01 00:00:00', 3),
       ('clickhouse.com', '2020-01-01 00:00:00', 6);

```

查询`analytics.hourly_data`的内容，将查不到任何记录，因为表引擎为`Null`，但数据已被处理



```
 SELECT * FROM analytics.hourly_data

```

输出：



```
domain_name|event_time|count_views|
-----------+----------+-----------+

```

### 结果


如果尝试查询目标表的`sumCountViews`字段值，将看到字段值以二进制表示（在某些终端中），因为该值不是以数字的形式存储，而是以`AggregateFunction`类型存储的。要获得聚合的最终结果，应该使用`-Merge`后缀。


通过以下查询，`sumCountViews`字段值无法正常显示：



```
SELECT sumCountViews FROM analytics.monthly_aggregated_data

```

输出：



```
sumCountViews|
-------------+
             |
             |
             |

```

使用 `Merge`后缀获取 `sumCountViews` 值:



```
SELECT sumMerge(sumCountViews) as sumCountViews
FROM analytics.monthly_aggregated_data;

```

输出：



```
sumCountViews|
-------------+
           12|

```

在`AggregatingMergeTree` 中将`AggregateFunction` 定义为`sum`，因此可以使用`sumMerge`。当在`AggregateFunction`上使用函数`avg`时，则将使用`avgMerge`，以此类推。



```
SELECT month, domain_name, sumMerge(sumCountViews) as sumCountViews
FROM analytics.monthly_aggregated_data
GROUP BY domain_name, month

```

输出：



```
month     |domain_name   |sumCountViews|
----------+--------------+-------------+
2020-01-01|clickhouse.com|            6|
2019-01-01|clickhouse.com|            1|
2019-02-01|clickhouse.com|            5|

```

现在我们可以查看物化视图是否符合我们定义的目标。


现在已经将数据存储在目标表`monthly_aggregated_data`中，可以按月聚合每个域名的数据：



```
SELECT month, domain_name, sumMerge(sumCountViews) as sumCountViews
FROM analytics.monthly_aggregated_data
GROUP BY domain_name, month;

```

输出：



```
month     |domain_name   |sumCountViews|
----------+--------------+-------------+
2020-01-01|clickhouse.com|            6|
2019-01-01|clickhouse.com|            1|
2019-02-01|clickhouse.com|            5|

```

按年聚合每个域名的数据:



```
SELECT year, domain_name, sum(sumCountViews)
FROM analytics.year_aggregated_data
GROUP BY domain_name, year;

```

输出：



```
year|domain_name   |sum(sumCountViews)|
----+--------------+------------------+
2019|clickhouse.com|                 6|
2020|clickhouse.com|                 6|

```

### 组合多个源表来创建单个目标表


物化视图还可以用于将多个源表组合以到一个目标表中。这对于创建类似于 `UNION ALL`逻辑的物化视图非常有用。


首先，创建两个代表不同指标集的源表:



```
CREATE TABLE analytics.impressions
(
    `event_time` DateTime,
    `domain_name` String
) ENGINE = MergeTree ORDER BY (domain_name, event_time);

CREATE TABLE analytics.clicks
(
    `event_time` DateTime,
    `domain_name` String
) ENGINE = MergeTree ORDER BY (domain_name, event_time);

```

然后使用组合的指标集创建 `Target`表：



```
CREATE TABLE analytics.daily_overview
(
    `on_date` Date,
    `domain_name` String,
    `impressions` SimpleAggregateFunction(sum, UInt64),
    `clicks` SimpleAggregateFunction(sum, UInt64)
) ENGINE = AggregatingMergeTree ORDER BY (on_date, domain_name);

```

创建两个指向同一`Target`表的物化视图。不需要显式地包含缺少的列：



```
CREATE MATERIALIZED VIEW analytics.daily_impressions_mv
TO analytics.daily_overview
AS                                                
SELECT
    toDate(event_time) AS on_date,
    domain_name,
    count() AS impressions,
    0 clicks   --<<<--- 如果去掉该列，则默认为 clicks为0
FROM                                              
    analytics.impressions
GROUP BY toDate(event_time) AS on_date, domain_name;

CREATE MATERIALIZED VIEW analytics.daily_clicks_mv
TO analytics.daily_overview
AS
SELECT
    toDate(event_time) AS on_date,
    domain_name,
    count() AS clicks,
    0 impressions    --<<<---如果去掉该列，则默认为 impressions 为0
FROM
    analytics.clicks
GROUP BY toDate(event_time) AS on_date, domain_name;

```

现在，当插入值时，这些值将被聚合到`Target`表中的相应列中：



```
INSERT INTO analytics.impressions (domain_name, event_time)
VALUES ('clickhouse.com', '2019-01-01 00:00:00'),
       ('clickhouse.com', '2019-01-01 12:00:00'),
       ('clickhouse.com', '2019-02-01 00:00:00'),
       ('clickhouse.com', '2019-03-01 00:00:00')
;

INSERT INTO analytics.clicks (domain_name, event_time)
VALUES ('clickhouse.com', '2019-01-01 00:00:00'),
       ('clickhouse.com', '2019-01-01 12:00:00'),
       ('clickhouse.com', '2019-03-01 00:00:00')
;

```

查询目标表 the `Target` table:



```
SELECT
    on_date,
    domain_name,
    sum(impressions) AS impressions,
    sum(clicks) AS clicks
FROM
    analytics.daily_overview
GROUP BY
    on_date,
    domain_name
;

```

输出：



```
on_date   |domain_name   |impressions|clicks|
----------+--------------+-----------+------+
2019-01-01|clickhouse.com|          2|     2|
2019-03-01|clickhouse.com|          1|     1|
2019-02-01|clickhouse.com|          1|     0|

```

### 参考链接


[https://clickhouse.com/docs/en/guides/developer/cascading\-materialized\-views](https://github.com)


## AggregateFunction


聚合函数有一个实现定义的中间状态，可以序列化为`AggregateFunction(...)`数据类型，并通常通过[物化视图](https://github.com)存储在表中。生成聚合函数状态的常见方法是使用`State`后缀调用聚合函数。为了以后能获得聚合的最终结果，必须使用带有`-Merge`后缀的相同聚合函数。


`AggregateFunction(name, types_of_arguments...)` — 参数数据类型。


参数说明：


* 聚合函数名称。如果名称对应的聚合函数鞋带参数，则还需要为其它指定参数。
* 聚合函数参数类型。


示例



```
CREATE TABLE testdb.aggregated_test_tb
(   
    `__name__` String, 
    `count` AggregateFunction(count),
    `avg_val` AggregateFunction(avg, Float64),
    `max_val` AggregateFunction(max, Float64),
    `time_max` AggregateFunction(argMax, DateTime, Float64),
    `mid_val` AggregateFunction(quantiles(0.5, 0.9), Float64) 
) ENGINE = AggregatingMergeTree() 
ORDER BY (__name__);

```

备注：如果上述SQL未添加`ORDER BY (__name__, create_time)`，执行会报类似如下错误：



```
SQL 错误 [42]: ClickHouse exception, code: 42, host: 192.168.88.131, port: 8123; Code: 42, e.displayText() = DB::Exception: Storage AggregatingMergeTree requires 3 to 4 parameters: 
name of column with date,
[sampling element of primary key],
primary key expression,
index granularity

```

**创建数据源表并插入测试数据**



```
CREATE TABLE testdb.test_tb 
(
    `__name__` String, 
    `create_time` DateTime, 
    `val` Float64
) ENGINE = MergeTree() 
PARTITION BY toStartOfWeek(create_time) 
ORDER BY (__name__, create_time);

INSERT INTO testdb.test_tb(`__name__`, `create_time`, `val`) VALUES
('xiaoxiao', now(), 80.5),
('xiaolin', addSeconds(now(), 10), 89.5),
('xiaohong', addSeconds(now(), 20), 90.5),
('lisi', addSeconds(now(), 30), 79.5),
('zhangshang', addSeconds(now(), 40), 60),
('wangwu', addSeconds(now(), 50), 65);

```

**插入数据**


使用以`State`后缀的聚合函数的`INSERT SELECT` 以插入数据\-\-比如希望获取目标列数据均值，即`avg(target_column)`，那么插入数据时使用的聚合函数为`avgState`，`*State`聚合函数返回状态(`state`)，而不是最终值。换句话说，返回一个 `AggregateFunction` 类型的值。



```
INSERT INTO testdb.aggregated_test_tb (`__name__`, `count`, `avg_val`, `max_val`, `time_max`, `mid_val`)
SELECT `__name__`,
countState() AS count,
avgState(val) AS avg_val, 
maxState(val) AS max_val,
argMaxState(create_time, val) AS time_max,
quantilesState(0.5, 0.9)(val) AS `mid_val`
FROM testdb.test_tb
GROUP BY `__name__`, toStartOfMinute(create_time);

```

注意：`SELECT`语句中的字段，要么使用聚合函数调用(比如上述`val`字段)，要么保持原字段不变(比如上述`__name__`字段)，保持原字段不变时，该字段必须包含于`GROUP BY`子句中，否则会报类似如下错误：



```
SQL 错误 [215]: ClickHouse exception, code: 215, host: 192.168.88.131, port: 8123; Code: 215, e.displayText() = DB::Exception: Column `__name__` is not under aggregate function and not in GROUP BY (version 20.3.5.21 (official build))

```

**查询数据**


从`AggregatingMergeTree`表中查询数据时，使用`GROUP BY`子句和与插入数据时相同的聚合函数，但使用`Merge`后缀，比如插入数据时使用的聚合函数为`avgState`，那么查询时使用的聚合函数为`avgMerge`。


后缀为`Merge`的聚合函数接受一组状态，将它们组合在一起，并返回完整数据聚合的结果。


例如，以下两个查询返回相同的结果



```
SELECT `__name__`, 
create_time,
avgMerge(avg_val) AS avg_val, 
maxMerge(max_val) AS max_val
FROM ( 
SELECT `__name__`, 
toStartOfMinute(create_time) AS create_time,
avgState(val) AS avg_val, 
maxState(val) AS max_val
FROM testdb.test_tb
GROUP BY `__name__`, create_time
)
GROUP BY `__name__`, create_time;

SELECT `__name__`, 
toStartOfMinute(create_time) AS create_time,
avg(val) AS avg_val, 
max(val) AS max_val
FROM testdb.test_tb
GROUP BY `__name__`, create_time;

```

例子：



```
SELECT `__name__`, 
countMerge(`count`), 
avgMerge(`avg_val`), 
maxMerge(`max_val`),
argMaxMerge(`time_max`),
quantilesMerge(0.5, 0.9)(`mid_val`)
FROM testdb.aggregated_test_tb
GROUP BY `__name__`;

```

### 参考链接


[https://clickhouse.com/docs/en/sql\-reference/data\-types/aggregatefunction](https://github.com)


## AggregatingMergeTree


引擎继承自[MergeTree](https://github.com)，更改了数据块合并的逻辑。ClickHouse使用一条存储了聚合函数状态组合的单条记录(在一个数据块中)替换带有相同主键(或更准确地说，用相同的[排序键](https://github.com):[蓝猫机场加速器](https://dahelaoshi.com))的所有行


说明：数据块是指ClickHouse存储数据的基本单位


可以使用 `AggregatingMergeTree` 表进行增量数据聚合，包括聚合物化视图。


引擎处理以下类型的所有列：


* `AggregateFunction`
* `SimpleAggregateFunction`


如果能减少有序行数，则使用`AggregatingMergeTree`是合适的


### 建表



```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = AggregatingMergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[TTL expr]
[SETTINGS name=value, ...]

```

有关请求参数的描述，参阅[请求描述](https://github.com)


**查询语句**


创建`AggregatingMergeTree`表与创建`MergeTree`表的子句相同。


查询和插入


要插入数据，使用[INSERT SELECT](https://github.com)使用`aggregateState`函数进行查询。从`AggregatingMergeTree`表中查询数据时，使用`GROUP BY`子句和与插入数据时相同的聚合函数，但使用`Merge`后缀。


在`SELECT`查询的结果中，`AggregateFunction`类型的值对所有ClickHouse输出格式都有特定于实现的二进制表示。例如，如果你可以使用`SELECT`查询将数据转储为`TabSeparated`格式，则可以使用`INSERT`查询将此转储重新加载。


### 一个物化视图示例



```
CREATE DATABASE testdb;

```

创建存放原始数据的`testdb.visits`表:



```
CREATE TABLE testdb.visits
(
    StartDate DateTime64, 
    CounterID UInt64,
    Sign Nullable(Int32),
    UserID Nullable(Int32)
) ENGINE = MergeTree 
ORDER BY (StartDate, CounterID);

```

说明：上述`StartDate DateTime64,`  如果写成`StartDate DateTime64 NOT NULL,`  运行会报错，如下：



```
Expected one of: CODEC, ALIAS, TTL, ClosingRoundBracket, Comma, DEFAULT, MATERIALIZED, COMMENT, token (version 20.3.5.21 (official build))

```

接下来，创建一个`AggregatingMergeTree`表，该表将存储`AggregationFunction`，用于跟踪访问总数和唯一用户数。


创建一个`AggregatingMergeTree` 物化视图，用于监视`testdb.revisits`表，并使用`AggregateFunction` 类型：



```
CREATE TABLE testdb.agg_visits (
    StartDate DateTime64,
    CounterID UInt64,
    Visits AggregateFunction(sum, Nullable(Int32)),
    Users AggregateFunction(uniq, Nullable(Int32))
)
ENGINE = AggregatingMergeTree() ORDER BY (StartDate, CounterID);

```


```
SQL 错误 [70]: ClickHouse exception, code: 70, host: 192.168.88.131, port: 8123; Code: 70, e.displayText() = DB::Exception: Conversion from AggregateFunction(sum, Int32) to AggregateFunction(sum, Nullable(Int32)) is not supported: while converting source column Visits to destination column Visits: while pushing to view testdb.visits_mv (version 20.3.5.21 (official build))


```


```
CREATE TABLE testdb.agg_visits (
    StartDate DateTime64,
    CounterID UInt64,
    Visits AggregateFunction(sum, Int32),
    Users AggregateFunction(uniq, Int32)
)
ENGINE = AggregatingMergeTree() ORDER BY (StartDate, CounterID);

```

创建一个物化视图，从`testdb.revisits`填充`testdb.agg_visits`：



```
CREATE MATERIALIZED VIEW testdb.visits_mv TO testdb.agg_visits
AS SELECT
    StartDate,
    CounterID,
    sumState(Sign) AS Visits,
    uniqState(UserID) AS Users
FROM testdb.visits
GROUP BY StartDate, CounterID;

```

插入数据到 `testdb.visits` 表:



```
INSERT INTO testdb.visits (StartDate, CounterID, Sign, UserID)
 VALUES (1667446031000, 1, 3, 4), (1667446031000, 1, 6, 3);

```

数据被同时插入到`testdb.revisits`和`testdb.agg_visits`中。


执行诸如 `SELECT ... GROUP BY ...`的语句查询物化视图`test.mv_visits`以获取聚合数据



```
SELECT
    StartDate,
    sumMerge(Visits) AS Visits,
    uniqMerge(Users) AS Users
FROM testdb.agg_visits
GROUP BY StartDate
ORDER BY StartDate;

```

输出：



```
StartDate          |Visits|Users|
-------------------+------+-----+
2022-11-03 11:27:11|     9|    2|

```

在`testdb.revisits`中添加另外2条记录，但这次尝试对其中一条记录使用不同的时间戳:



```
INSERT INTO testdb.visits (StartDate, CounterID, Sign, UserID)
 VALUES (1669446031000, 2, 5, 10), (1667446031000, 3, 7, 5);

```

再次查询，输出如下：



```
StartDate          |Visits|Users|
-------------------+------+-----+
2022-11-03 11:27:11|    16|    3|
2022-11-26 15:00:31|     5|    1|

```

### 参考链接


[https://clickhouse.com/docs/en/engines/table\-engines/mergetree\-family/aggregatingmergetree](https://github.com)


