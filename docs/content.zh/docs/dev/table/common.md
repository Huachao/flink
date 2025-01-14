---
title: "概念与通用 API"
weight: 1
type: docs
aliases:
  - /zh/dev/table/common.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# 概念与通用 API

Table API 和 SQL 集成在同一套 API 中。这套 API 的核心概念是`Table`，用作查询的输入和输出。本文介绍了 Table API 和 SQL 查询程序的通用结构、如何注册 `Table` 、如何查询 `Table` 以及如何输出 `Table` 。

<a name="main-differences-between-the-two-planners"></a>

两种计划器（Planner）的主要区别
-----------------------------------------

1. Blink 将批处理作业视作流处理的一种特例。严格来说，`Table` 和 `DataSet` 之间不支持相互转换，并且批处理作业也不会转换成 `DataSet` 程序而是转换成 `DataStream` 程序，流处理作业也一样。
2. Blink 计划器不支持  `BatchTableSource`，而是使用有界的  `StreamTableSource` 来替代。
3. 旧计划器和 Blink 计划器中 `FilterableTableSource` 的实现是不兼容的。旧计划器会将 `PlannerExpression` 下推至 `FilterableTableSource`，而 Blink 计划器则是将 `Expression` 下推。
4. 基于字符串的键值配置选项仅在 Blink 计划器中使用。（详情参见 [配置]({{< ref "docs/dev/table/config" >}}) ）
5. `PlannerConfig` 在两种计划器中的实现（`CalciteConfig`）是不同的。
6. Blink 计划器会将多sink（multiple-sinks）优化成一张有向无环图（DAG），`TableEnvironment` 和 `StreamTableEnvironment` 都支持该特性。旧计划器总是将每个sink都优化成一个新的有向无环图，且所有图相互独立。
7. 旧计划器目前不支持 catalog 统计数据，而 Blink 支持。

<a name="structure-of-table-api-and-sql-programs"></a>

Table API 和 SQL 程序的结构
---------------------------------------

所有用于批处理和流处理的 Table API 和 SQL 程序都遵循相同的模式。下面的代码示例展示了 Table API 和 SQL 程序的通用结构。

{{< tabs "62be8916-7ab4-4831-bd19-05c591181835" >}}
{{< tab "Java" >}}
```java

// create a TableEnvironment for specific planner batch or streaming
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// create an input Table
tableEnv.executeSql("CREATE TEMPORARY TABLE table1 ... WITH ( 'connector' = ... )");
// register an output Table
tableEnv.executeSql("CREATE TEMPORARY TABLE outputTable ... WITH ( 'connector' = ... )");

// create a Table object from a Table API query
Table table2 = tableEnv.from("table1").select(...);
// create a Table object from a SQL query
Table table3 = tableEnv.sqlQuery("SELECT ... FROM table1 ... ");

// emit a Table API result Table to a TableSink, same for SQL result
TableResult tableResult = table2.executeInsert("outputTable");
tableResult...

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala

// create a TableEnvironment for specific planner batch or streaming
val tableEnv = ... // see "Create a TableEnvironment" section

// create an input Table
tableEnv.executeSql("CREATE TEMPORARY TABLE table1 ... WITH ( 'connector' = ... )")
// register an output Table
tableEnv.executeSql("CREATE TEMPORARY TABLE outputTable ... WITH ( 'connector' = ... )")

// create a Table from a Table API query
val table2 = tableEnv.from("table1").select(...)
// create a Table from a SQL query
val table3 = tableEnv.sqlQuery("SELECT ... FROM table1 ...")

// emit a Table API result Table to a TableSink, same for SQL result
val tableResult = table2.executeInsert("outputTable")
tableResult...

```
{{< /tab >}}
{{< tab "Python" >}}
```python

# create a TableEnvironment for specific planner batch or streaming
table_env = ... # see "Create a TableEnvironment" section

# register an input Table
table_env.executeSql("CREATE TEMPORARY TABLE table1 ... WITH ( 'connector' = ... )")
# register an output Table
table_env.executeSql("CREATE TEMPORARY TABLE outputTable ... WITH ( 'connector' = ... )")

# create a Table from a Table API query
table2 = table_env.from_path("table1").select(...)
# create a Table from a SQL query
table3 = table_env.sql_query("SELECT ... FROM table1 ...")

# emit a Table API result Table to a TableSink, same for SQL result
table_result = table3.execute_insert("outputTable")
table_result...

```
{{< /tab >}}
{{< /tabs >}}

**注意：** Table API 和 SQL 查询可以很容易地集成并嵌入到 DataStream 或 DataSet 程序中。 请参阅[与 DataStream 和 DataSet API 结合](#integration-with-datastream-and-dataset-api) 章节了解如何将 DataSet 和 DataStream 与表之间的相互转化。

{{< top >}}

<a name="create-a-tableenvironment"></a>

创建 TableEnvironment
-------------------------

`TableEnvironment` 是 Table API 和 SQL 的核心概念。它负责:

* 在内部的 catalog 中注册 `Table`
* 注册外部的 catalog
* 加载可插拔模块
* 执行 SQL 查询
* 注册自定义函数 （scalar、table 或 aggregation）
* 将 `DataStream` 或 `DataSet` 转换成 `Table`
* 持有对  `ExecutionEnvironment` 或 `StreamExecutionEnvironment` 的引用

`Table` 总是与特定的 `TableEnvironment` 绑定。不能在同一条查询中使用不同 TableEnvironment 中的表，例如，对它们进行 join 或 union 操作。

`TableEnvironment` 可以通过静态方法 `BatchTableEnvironment.create()` 或者 `StreamTableEnvironment.create()` 在 `StreamExecutionEnvironment` 或者 `ExecutionEnvironment` 中创建，`TableConfig` 是可选项。`TableConfig`可用于配置`TableEnvironment`或定制的查询优化和转换过程(参见 [查询优化](#query-optimization))。

请确保选择与你的编程语言匹配的特定的计划器`BatchTableEnvironment`/`StreamTableEnvironment`。

如果两种计划器的 jar 包都在 classpath 中（默认行为），你应该明确地设置要在当前程序中使用的计划器。

{{< tabs "74c10eb1-1267-4b4e-89cd-2d85f761d139" >}}
{{< tab "Java" >}}
```java

// **********************
// FLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

EnvironmentSettings fsSettings = EnvironmentSettings.newInstance().useOldPlanner().inStreamingMode().build();
StreamExecutionEnvironment fsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment fsTableEnv = StreamTableEnvironment.create(fsEnv, fsSettings);
// or TableEnvironment fsTableEnv = TableEnvironment.create(fsSettings);

// ******************
// FLINK BATCH QUERY
// ******************
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.table.api.bridge.java.BatchTableEnvironment;

ExecutionEnvironment fbEnv = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment fbTableEnv = BatchTableEnvironment.create(fbEnv);

// **********************
// BLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings);
// or TableEnvironment bsTableEnv = TableEnvironment.create(bsSettings);

// ******************
// BLINK BATCH QUERY
// ******************
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

EnvironmentSettings bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build();
TableEnvironment bbTableEnv = TableEnvironment.create(bbSettings);

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala

// **********************
// FLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.table.api.EnvironmentSettings
import org.apache.flink.table.api.bridge.scala.StreamTableEnvironment

val fsSettings = EnvironmentSettings.newInstance().useOldPlanner().inStreamingMode().build()
val fsEnv = StreamExecutionEnvironment.getExecutionEnvironment
val fsTableEnv = StreamTableEnvironment.create(fsEnv, fsSettings)
// or val fsTableEnv = TableEnvironment.create(fsSettings)

// ******************
// FLINK BATCH QUERY
// ******************
import org.apache.flink.api.scala.ExecutionEnvironment
import org.apache.flink.table.api.bridge.scala.BatchTableEnvironment

val fbEnv = ExecutionEnvironment.getExecutionEnvironment
val fbTableEnv = BatchTableEnvironment.create(fbEnv)

// **********************
// BLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.table.api.EnvironmentSettings
import org.apache.flink.table.api.bridge.scala.StreamTableEnvironment

val bsEnv = StreamExecutionEnvironment.getExecutionEnvironment
val bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build()
val bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings)
// or val bsTableEnv = TableEnvironment.create(bsSettings)

// ******************
// BLINK BATCH QUERY
// ******************
import org.apache.flink.table.api.{EnvironmentSettings, TableEnvironment}

val bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build()
val bbTableEnv = TableEnvironment.create(bbSettings)

```
{{< /tab >}}
{{< tab "Python" >}}
```python

# **********************
# FLINK STREAMING QUERY
# **********************
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings

f_s_env = StreamExecutionEnvironment.get_execution_environment()
f_s_settings = EnvironmentSettings.new_instance().use_old_planner().in_streaming_mode().build()
f_s_t_env = StreamTableEnvironment.create(f_s_env, environment_settings=f_s_settings)

# ******************
# FLINK BATCH QUERY
# ******************
from pyflink.dataset import ExecutionEnvironment
from pyflink.table import BatchTableEnvironment

f_b_env = ExecutionEnvironment.get_execution_environment()
f_b_t_env = BatchTableEnvironment.create(f_b_env, table_config)

# **********************
# BLINK STREAMING QUERY
# **********************
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings

b_s_env = StreamExecutionEnvironment.get_execution_environment()
b_s_settings = EnvironmentSettings.new_instance().use_blink_planner().in_streaming_mode().build()
b_s_t_env = StreamTableEnvironment.create(b_s_env, environment_settings=b_s_settings)

# ******************
# BLINK BATCH QUERY
# ******************
from pyflink.table import EnvironmentSettings, BatchTableEnvironment

b_b_settings = EnvironmentSettings.new_instance().use_blink_planner().in_batch_mode().build()
b_b_t_env = BatchTableEnvironment.create(environment_settings=b_b_settings)

```
{{< /tab >}}
{{< /tabs >}}

**注意：** 如果`/lib`目录中只有一种计划器的 jar 包，则可以使用`useAnyPlanner`（python 使用 `use any_u_planner`）创建 `EnvironmentSettings`。

{{< top >}}

<a name="create-tables-in-the-catalog"></a>

在 Catalog 中创建表
-------------------------------

`TableEnvironment` 维护着一个由标识符（identifier）创建的表 catalog 的映射。标识符由三个部分组成：catalog 名称、数据库名称以及对象名称。如果 catalog 或者数据库没有指明，就会使用当前默认值（参见[表标识符扩展](#expanding-table-identifiers)章节中的例子）。

`Table` 可以是虚拟的（视图 `VIEWS`）也可以是常规的（表 `TABLES`）。视图 `VIEWS`可以从已经存在的`Table`中创建，一般是 Table API 或者 SQL 的查询结果。 表`TABLES`描述的是外部数据，例如文件、数据库表或者消息队列。

<a name="temporary-vs-permanent-tables"></a>

### 临时表（Temporary Table）和永久表（Permanent Table）

表可以是临时的，并与单个 Flink 会话（session）的生命周期相关，也可以是永久的，并且在多个 Flink 会话和群集（cluster）中可见。

永久表需要 [catalog]({{< ref "docs/dev/table/catalogs" >}})（例如 Hive Metastore）以维护表的元数据。一旦永久表被创建，它将对任何连接到 catalog 的 Flink 会话可见且持续存在，直至被明确删除。

另一方面，临时表通常保存于内存中并且仅在创建它们的 Flink 会话持续期间存在。这些表对于其它会话是不可见的。它们不与任何 catalog 或者数据库绑定但可以在一个命名空间（namespace）中创建。即使它们对应的数据库被删除，临时表也不会被删除。

<a name="shadowing"></a>

#### 屏蔽（Shadowing）

可以使用与已存在的永久表相同的标识符去注册临时表。临时表会屏蔽永久表，并且只要临时表存在，永久表就无法访问。所有使用该标识符的查询都将作用于临时表。

这可能对实验（experimentation）有用。它允许先对一个临时表进行完全相同的查询，例如只有一个子集的数据，或者数据是不确定的。一旦验证了查询的正确性，就可以对实际的生产表进行查询。

<a name="create-a-table"></a>

### 创建表

<a name="virtual-tables"></a>

#### 虚拟表

在 SQL 的术语中，Table API 的对象对应于`视图`（虚拟表）。它封装了一个逻辑查询计划。它可以通过以下方法在 catalog 中创建：

{{< tabs "775d4e7e-3b10-409b-a0e2-3abb2e025586" >}}
{{< tab "Java" >}}
```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// table is the result of a simple projection query
Table projTable = tableEnv.from("X").select(...);

// register the Table projTable as table "projectedTable"
tableEnv.createTemporaryView("projectedTable", projTable);
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv = ... // see "Create a TableEnvironment" section

// table is the result of a simple projection query
val projTable: Table = tableEnv.from("X").select(...)

// register the Table projTable as table "projectedTable"
tableEnv.createTemporaryView("projectedTable", projTable)
```
{{< /tab >}}
{{< tab "Python" >}}
```python
# get a TableEnvironment
table_env = ... # see "Create a TableEnvironment" section

# table is the result of a simple projection query
proj_table = table_env.from_path("X").select(...)

# register the Table projTable as table "projectedTable"
table_env.register_table("projectedTable", proj_table)
```
{{< /tab >}}
{{< /tabs >}}

**注意：** 从传统数据库系统的角度来看，`Table` 对象与 `VIEW` 视图非常像。也就是，定义了 `Table` 的查询是没有被优化的，
而且会被内嵌到另一个引用了这个注册了的 `Table`的查询中。如果多个查询都引用了同一个注册了的`Table`，那么它会被内嵌每个查询中并被执行多次，
也就是说注册了的`Table`的结果**不会**被共享（注：Blink 计划器的`TableEnvironment`会优化成只执行一次）。

{{< top >}}

<a name="connector-tables"></a>

#### Connector Tables

另外一个方式去创建 `TABLE` 是通过 [connector]({{< ref "docs/connectors/table/overview" >}}) 声明。Connector 描述了存储表数据的外部系统。存储系统例如 Apache Kafka 或者常规的文件系统都可以通过这种方式来声明。

{{< tabs "0b2e02bb-4185-4036-b948-85961ad05764" >}}
{{< tab "Java" >}}
```java
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .createTemporaryTable("MyTable")
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .createTemporaryTable("MyTable")
```
{{< /tab >}}
{{< tab "Python" >}}
```python
table_environment \
    .connect(...) \
    .with_format(...) \
    .with_schema(...) \
    .in_append_mode() \
    .create_temporary_table("MyTable")
```
{{< /tab >}}
{{< tab "DDL" >}}
```sql
tableEnvironment.executeSql("CREATE [TEMPORARY] TABLE MyTable (...) WITH (...)")
```
{{< /tab >}}
{{< /tabs >}}

<a name="expanding-table-identifiers"></a>

### 扩展表标识符

表总是通过三元标识符注册，包括 catalog 名、数据库名和表名。

用户可以指定一个 catalog 和数据库作为 "当前catalog" 和"当前数据库"。有了这些，那么刚刚提到的三元标识符的前两个部分就可以被省略了。如果前两部分的标识符没有指定，
那么会使用当前的 catalog 和当前数据库。用户也可以通过 Table API 或 SQL 切换当前的 catalog 和当前的数据库。

标识符遵循 SQL 标准，因此使用时需要用反引号（`` ` ``）进行转义。

{{< tabs "d673e532-0525-480c-9976-da0db11284bc" >}}
{{< tab "Java" >}}
```java
TableEnvironment tEnv = ...;
tEnv.useCatalog("custom_catalog");
tEnv.useDatabase("custom_database");

Table table = ...;

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'custom_database'
tableEnv.createTemporaryView("exampleView", table);

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'other_database'
tableEnv.createTemporaryView("other_database.exampleView", table);

// register the view named 'example.View' in the catalog named 'custom_catalog'
// in the database named 'custom_database'
tableEnv.createTemporaryView("`example.View`", table);

// register the view named 'exampleView' in the catalog named 'other_catalog'
// in the database named 'other_database'
tableEnv.createTemporaryView("other_catalog.other_database.exampleView", table);

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tEnv: TableEnvironment = ...;
tEnv.useCatalog("custom_catalog")
tEnv.useDatabase("custom_database")

val table: Table = ...;

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'custom_database'
tableEnv.createTemporaryView("exampleView", table)

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'other_database'
tableEnv.createTemporaryView("other_database.exampleView", table)

// register the view named 'example.View' in the catalog named 'custom_catalog'
// in the database named 'custom_database'
tableEnv.createTemporaryView("`example.View`", table)

// register the view named 'exampleView' in the catalog named 'other_catalog'
// in the database named 'other_database'
tableEnv.createTemporaryView("other_catalog.other_database.exampleView", table)
```
{{< /tab >}}
{{< /tabs >}}

<a name="query-a-table"></a>

查询表
-------------

<a name="table-api"></a>

### Table API

Table API 是关于 Scala 和 Java 的集成语言式查询 API。与 SQL 相反，Table API 的查询不是由字符串指定，而是在宿主语言中逐步构建。

Table API 是基于 `Table` 类的，该类表示一个表（流或批处理），并提供使用关系操作的方法。这些方法返回一个新的 Table 对象，该对象表示对输入 Table 进行关系操作的结果。 一些关系操作由多个方法调用组成，例如 `table.groupBy(...).select()`，其中 `groupBy(...)` 指定 `table` 的分组，而 `select(...)` 在  `table` 分组上的投影。

文档 [Table API]({{< ref "docs/dev/table/tableApi" >}}) 说明了所有流处理和批处理表支持的 Table API 算子。

以下示例展示了一个简单的 Table API 聚合查询：

{{< tabs "6c900a03-63e1-44cc-88ee-1f05093fdfc2" >}}
{{< tab "Java" >}}
```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register Orders table

// scan registered Orders table
Table orders = tableEnv.from("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter($("cCountry").isEqual("FRANCE"))
  .groupBy($("cID"), $("cName"))
  .select($("cID"), $("cName"), $("revenue").sum().as("revSum"));

// emit or convert Table
// execute query
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv = ... // see "Create a TableEnvironment" section

// register Orders table

// scan registered Orders table
val orders = tableEnv.from("Orders")
// compute revenue for all customers from France
val revenue = orders
  .filter($"cCountry" === "FRANCE")
  .groupBy($"cID", $"cName")
  .select($"cID", $"cName", $"revenue".sum AS "revSum")

// emit or convert Table
// execute query
```

**Note:** The Scala Table API uses Scala String interpolation that starts with a dollar sign (`$`) to reference the attributes of a `Table`. The Table API uses Scala implicits. Make sure to import
* `org.apache.flink.table.api._` - for implicit expression conversions
* `org.apache.flink.api.scala._` and `org.apache.flink.table.api.bridge.scala._` if you want to convert from/to DataStream.
{{< /tab >}}
{{< tab "Python" >}}
```python
# get a TableEnvironment
table_env = # see "Create a TableEnvironment" section

# register Orders table

# scan registered Orders table
orders = table_env.from_path("Orders")
# compute revenue for all customers from France
revenue = orders \
    .filter(orders.cCountry == 'FRANCE') \
    .group_by(orders.cID, orders.cName) \
    .select(orders.cID, orders.cName, orders.revenue.sum.alias('revSum'))

# emit or convert Table
# execute query
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="sql"></a>

### SQL

Flink SQL 是基于实现了SQL标准的 [Apache Calcite](https://calcite.apache.org) 的。SQL 查询由常规字符串指定。

文档 [SQL]({{< ref "docs/dev/table/sql/overview" >}}) 描述了Flink对流处理和批处理表的SQL支持。

下面的示例演示了如何指定查询并将结果作为 `Table` 对象返回。

{{< tabs "6f2ce0fa-748f-479e-ade1-20bb30b1b84e" >}}
{{< tab "Java" >}}
```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// emit or convert Table
// execute query
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv = ... // see "Create a TableEnvironment" section

// register Orders table

// compute revenue for all customers from France
val revenue = tableEnv.sqlQuery("""
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

// emit or convert Table
// execute query
```

{{< /tab >}}
{{< tab "Python" >}}
```python
# get a TableEnvironment
table_env = ... # see "Create a TableEnvironment" section

# register Orders table

# compute revenue for all customers from France
revenue = table_env.sql_query(
    "SELECT cID, cName, SUM(revenue) AS revSum "
    "FROM Orders "
    "WHERE cCountry = 'FRANCE' "
    "GROUP BY cID, cName"
)

# emit or convert Table
# execute query
```
{{< /tab >}}
{{< /tabs >}}

如下的示例展示了如何指定一个更新查询，将查询的结果插入到已注册的表中。

{{< tabs "9c184c39-3741-48b6-9dda-67c1f3cf5b76" >}}
{{< tab "Java" >}}
```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.executeSql(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv = ... // see "Create a TableEnvironment" section

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.executeSql("""
  |INSERT INTO RevenueFrance
  |SELECT cID, cName, SUM(revenue) AS revSum
  |FROM Orders
  |WHERE cCountry = 'FRANCE'
  |GROUP BY cID, cName
  """.stripMargin)

```

{{< /tab >}}
{{< tab "Python" >}}
```python
# get a TableEnvironment
table_env = ... # see "Create a TableEnvironment" section

# register "Orders" table
# register "RevenueFrance" output table

# compute revenue for all customers from France and emit to "RevenueFrance"
table_env.execute_sql(
    "INSERT INTO RevenueFrance "
    "SELECT cID, cName, SUM(revenue) AS revSum "
    "FROM Orders "
    "WHERE cCountry = 'FRANCE' "
    "GROUP BY cID, cName"
)

```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="mixing-table-api-and-sql"></a>

### 混用 Table API 和 SQL

Table API 和 SQL 查询的混用非常简单因为它们都返回 `Table` 对象：

* 可以在 SQL 查询返回的 `Table` 对象上定义 Table API 查询。
* 在 `TableEnvironment` 中注册的[结果表](#register-a-table)可以在 SQL 查询的 `FROM` 子句中引用，通过这种方法就可以在 Table API 查询的结果上定义 SQL 查询。

{{< top >}}

<a name="emit-a-table"></a>

输出表
------------

`Table` 通过写入 `TableSink` 输出。`TableSink` 是一个通用接口，用于支持多种文件格式（如 CSV、Apache Parquet、Apache Avro）、存储系统（如 JDBC、Apache HBase、Apache Cassandra、Elasticsearch）或消息队列系统（如 Apache Kafka、RabbitMQ）。

批处理 `Table` 只能写入 `BatchTableSink`，而流处理 `Table` 需要指定写入 `AppendStreamTableSink`，`RetractStreamTableSink` 或者 `UpsertStreamTableSink`。

请参考文档 [Table Sources & Sinks]({{< ref "docs/dev/table/sourcesSinks" >}}) 以获取更多关于可用 Sink 的信息以及如何自定义 `TableSink`。

方法 `Table.executeInsert(String tableName)` 将 `Table` 发送至已注册的 `TableSink`。该方法通过名称在 catalog 中查找 `TableSink` 并确认`Table` schema 和 `TableSink` schema 一致。

下面的示例演示如何输出 `Table`：

{{< tabs "b7f0d5ac-fb30-4ce7-b101-8fc7af2a3b98" >}}
{{< tab "Java" >}}
```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// create an output Table
final Schema schema = new Schema()
    .field("a", DataTypes.INT())
    .field("b", DataTypes.STRING())
    .field("c", DataTypes.BIGINT());

tableEnv.connect(new FileSystem().path("/path/to/file"))
    .withFormat(new Csv().fieldDelimiter('|').deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("CsvSinkTable");

// compute a result Table using Table API operators and/or SQL queries
Table result = ...
// emit the result Table to the registered TableSink
result.executeInsert("CsvSinkTable");

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv = ... // see "Create a TableEnvironment" section

// create an output Table
val schema = new Schema()
    .field("a", DataTypes.INT())
    .field("b", DataTypes.STRING())
    .field("c", DataTypes.BIGINT())

tableEnv.connect(new FileSystem().path("/path/to/file"))
    .withFormat(new Csv().fieldDelimiter('|').deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("CsvSinkTable")

// compute a result Table using Table API operators and/or SQL queries
val result: Table = ...

// emit the result Table to the registered TableSink
result.executeInsert("CsvSinkTable")

```
{{< /tab >}}
{{< tab "Python" >}}
```python
# get a TableEnvironment
table_env = ... # see "Create a TableEnvironment" section

# create a TableSink
table_env.connect(FileSystem().path("/path/to/file")))
    .with_format(Csv()
                 .field_delimiter(',')
                 .deriveSchema())
    .with_schema(Schema()
                 .field("a", DataTypes.INT())
                 .field("b", DataTypes.STRING())
                 .field("c", DataTypes.BIGINT()))
    .create_temporary_table("CsvSinkTable")

# compute a result Table using Table API operators and/or SQL queries
result = ...

# emit the result Table to the registered TableSink
result.execute_insert("CsvSinkTable")

```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="translate-and-execute-a-query"></a>

翻译与执行查询
-----------------------------

两种计划器翻译和执行查询的方式是不同的。

{{< tabs "58b3e383-ade2-482d-a86f-b4f243ffce48" >}}
{{< tab "Blink planner" >}}
不论输入数据源是流式的还是批式的，Table API 和 SQL 查询都会被转换成 [DataStream]({{< ref "docs/dev/datastream/overview" >}}) 程序。查询在内部表示为逻辑查询计划，并被翻译成两个阶段：

1. 优化逻辑执行计划
2. 翻译成 DataStream 程序

Table API 或者 SQL 查询在下列情况下会被翻译：

* 当 `TableEnvironment.executeSql()` 被调用时。该方法是用来执行一个 SQL 语句，一旦该方法被调用， SQL 语句立即被翻译。
* 当 `Table.executeInsert()` 被调用时。该方法是用来将一个表的内容插入到目标表中，一旦该方法被调用， TABLE API 程序立即被翻译。
* 当 `Table.execute()` 被调用时。该方法是用来将一个表的内容收集到本地，一旦该方法被调用， TABLE API 程序立即被翻译。
* 当 `StatementSet.execute()` 被调用时。`Table` （通过 `StatementSet.addInsert()` 输出给某个 `Sink`）和 INSERT 语句 （通过调用 `StatementSet.addInsertSql()`）会先被缓存到 `StatementSet` 中，`StatementSet.execute()` 方法被调用时，所有的 sink 会被优化成一张有向无环图。
* 当 `Table` 被转换成 `DataStream` 时（参阅[与 DataStream 和 DataSet API 结合](#integration-with-datastream-and-dataset-api)）。转换完成后，它就成为一个普通的 DataStream 程序，并会在调用 `StreamExecutionEnvironment.execute()` 时被执行。

<span class="label label-danger">注意</span> **从 1.11 版本开始，`sqlUpdate` 方法 和 `insertInto` 方法被废弃，从这两个方法构建的 Table 程序必须通过 `StreamTableEnvironment.execute()` 方法执行，而不能通过 `StreamExecutionEnvironment.execute()` 方法来执行。**
{{< /tab >}}
{{< tab "Old planner" >}}
Table API 和 SQL 查询会被翻译成 [DataStream]({{< ref "docs/dev/datastream/overview" >}}) 或者 [DataSet]({{< ref "docs/dev/dataset/overview" >}}) 程序， 这取决于它们的输入数据源是流式的还是批式的。查询在内部表示为逻辑查询计划，并被翻译成两个阶段：

1. 优化逻辑执行计划
2. 翻译成 DataStream 或 DataSet 程序

Table API 或者 SQL 查询在下列情况下会被翻译：

* 当 `TableEnvironment.executeSql()` 被调用时。该方法是用来执行一个 SQL 语句，一旦该方法被调用， SQL 语句立即被翻译。
* 当 `Table.executeInsert()` 被调用时。该方法是用来将一个表的内容插入到目标表中，一旦该方法被调用， TABLE API 程序立即被翻译。
* 当 `Table.execute()` 被调用时。该方法是用来将一个表的内容收集到本地，一旦该方法被调用， TABLE API 程序立即被翻译。
* 当 `StatementSet.execute()` 被调用时。`Table` （通过 `StatementSet.addInsert()` 输出给某个 `Sink`）和 INSERT 语句 （通过调用 `StatementSet.addInsertSql()`）会先被缓存到 `StatementSet` 中，`StatementSet.execute()` 方法被调用时，所有的 sink 会被优化成一张有向无环图。
* 对于 Streaming 而言，当`Table` 被转换成 `DataStream` 时（参阅[与 DataStream 和 DataSet API 结合](#integration-with-datastream-and-dataset-api)）触发翻译。转换完成后，它就成为一个普通的 DataStream 程序，并会在调用 `StreamExecutionEnvironment.execute()` 时被执行。对于 Batch 而言，`Table` 被转换成 `DataSet` 时（参阅[与 DataStream 和 DataSet API 结合](#integration-with-datastream-and-dataset-api)）触发翻译。转换完成后，它就成为一个普通的 DataSet 程序，并会在调用 `ExecutionEnvironment.execute()` 时被执行。

<span class="label label-danger">注意</span> **从 1.11 版本开始，`sqlUpdate` 方法 和 `insertInto` 方法被废弃。对于 Streaming 而言，如果一个 Table 程序是从这两个方法构建出来的，必须通过 `StreamTableEnvironment.execute()` 方法执行，而不能通过 `StreamExecutionEnvironment.execute()` 方法执行；对于 Batch 而言，如果一个 Table 程序是从这两个方法构建出来的，必须通过 `BatchTableEnvironment.execute()` 方法执行，而不能通过 `ExecutionEnvironment.execute()` 方法执行。**

{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="integration-with-datastream-and-dataset-api"></a>

与 DataStream 和 DataSet API 结合
-------------------------------------------

在流处理方面两种计划器都可以与 `DataStream` API 结合。只有旧计划器可以与 `DataSet API` 结合。在批处理方面，Blink 计划器不能同两种计划器中的任何一个结合。

**注意：** 下文讨论的 `DataSet` API 只与旧计划起有关。

Table API 和 SQL 可以被很容易地集成并嵌入到 [DataStream]({{< ref "docs/dev/datastream/overview" >}}) 和 [DataSet]({{< ref "docs/dev/dataset/overview" >}}) 程序中。例如，可以查询外部表（例如从 RDBMS），进行一些预处理，例如过滤，投影，聚合或与元数据 join，然后使用 DataStream 或 DataSet API（以及在这些 API 之上构建的任何库，例如 CEP 或 Gelly）。相反，也可以将 Table API 或 SQL 查询应用于 DataStream 或 DataSet 程序的结果。

这种交互可以通过 `DataStream` 或 `DataSet` 与 `Table` 的相互转化实现。本节我们会介绍这些转化是如何实现的。

<a name="implicit-conversion-for-scala"></a>

### Scala 隐式转换

Scala Table API 含有对 `DataSet`、`DataStream` 和 `Table` 类的隐式转换。 通过为 Scala DataStream API 导入 `org.apache.flink.table.api.bridge.scala._` 包以及 `org.apache.flink.api.scala._` 包，可以启用这些转换。

<a name="create-a-view-from-a-datastream-or-dataset"></a>

### 通过 DataSet 或 DataStream 创建`视图`

在 `TableEnvironment` 中可以将 `DataStream` 或 `DataSet` 注册成视图。结果视图的 schema 取决于注册的 `DataStream` 或 `DataSet` 的数据类型。请参阅文档 [数据类型到 table schema 的映射](#mapping-of-data-types-to-table-schema)获取详细信息。

**注意：** 通过 `DataStream` 或 `DataSet` 创建的视图只能注册成临时视图。

{{< tabs "f293c77d-7c44-4d1f-9ddf-776d856b52b2" >}}
{{< tab "Java" >}}
```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as View "myTable" with fields "f0", "f1"
tableEnv.createTemporaryView("myTable", stream);

// register the DataStream as View "myTable2" with fields "myLong", "myString"
tableEnv.createTemporaryView("myTable2", stream, $("myLong"), $("myString"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get TableEnvironment
// registration of a DataSet is equivalent
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

val stream: DataStream[(Long, String)] = ...

// register the DataStream as View "myTable" with fields "f0", "f1"
tableEnv.createTemporaryView("myTable", stream)

// register the DataStream as View "myTable2" with fields "myLong", "myString"
tableEnv.createTemporaryView("myTable2", stream, 'myLong, 'myString)
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="convert-a-datastream-or-dataset-into-a-table"></a>

### 将 DataStream 或 DataSet 转换成表

与在 `TableEnvironment` 注册 `DataStream` 或 `DataSet` 不同，DataStream 和 DataSet 还可以直接转换成 `Table`。如果你想在 Table API 的查询中使用表，这将非常便捷。

{{< tabs "53265853-e08d-4f70-93f8-c0f6d1b75e48" >}}
{{< tab "Java" >}}
```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, $("myLong"), $("myString"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get TableEnvironment
// registration of a DataSet is equivalent
val tableEnv = ... // see "Create a TableEnvironment" section

val stream: DataStream[(Long, String)] = ...

// convert the DataStream into a Table with default fields "_1", "_2"
val table1: Table = tableEnv.fromDataStream(stream)

// convert the DataStream into a Table with fields "myLong", "myString"
val table2: Table = tableEnv.fromDataStream(stream, $"myLong", $"myString")
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="convert-a-table-into-a-datastream-or-dataset"></a>

### 将表转换成 DataStream 或 DataSet

`Table` 可以被转换成 `DataStream` 或 `DataSet`。通过这种方式，定制的 DataSet 或 DataStream 程序就可以在 Table API 或者 SQL 的查询结果上运行了。

将 `Table` 转换为 `DataStream` 或者 `DataSet` 时，你需要指定生成的 `DataStream` 或者 `DataSet` 的数据类型，即，`Table` 的每行数据要转换成的数据类型。通常最方便的选择是转换成 `Row` 。以下列表概述了不同选项的功能：

- **Row**: 字段按位置映射，字段数量任意，支持 `null` 值，无类型安全（type-safe）检查。
- **POJO**: 字段按名称映射（POJO 必须按`Table` 中字段名称命名），字段数量任意，支持 `null` 值，无类型安全检查。
- **Case Class**: 字段按位置映射，不支持 `null` 值，有类型安全检查。
- **Tuple**: 字段按位置映射，字段数量少于 22（Scala）或者 25（Java），不支持 `null` 值，无类型安全检查。
- **Atomic Type**: `Table` 必须有一个字段，不支持 `null` 值，有类型安全检查。

<a name="convert-a-table-into-a-datastream"></a>

#### 将表转换成 DataStream

流式查询（streaming query）的结果表会动态更新，即，当新纪录到达查询的输入流时，查询结果会改变。因此，像这样将动态查询结果转换成 DataStream 需要对表的更新方式进行编码。

将 `Table` 转换为 `DataStream` 有两种模式：

1. **Append Mode**: 仅当动态 `Table` 仅通过`INSERT`更改进行修改时，才可以使用此模式，即，它仅是追加操作，并且之前输出的结果永远不会更新。
2. **Retract Mode**: 任何情形都可以使用此模式。它使用 boolean 值对 `INSERT` 和 `DELETE` 操作的数据进行标记。

{{< tabs "9533a9f4-e6e8-44d7-a29c-33713724eacc" >}}
{{< tab "Java" >}}
```java
// get StreamTableEnvironment.
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into an append DataStream of Row by specifying the class
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer>
//   via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataStream<Tuple2<String, Integer>> dsTuple =
  tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream<Tuple2<Boolean, X>>.
//   The boolean field indicates the type of the change.
//   True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream =
  tableEnv.toRetractStream(table, Row.class);

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get TableEnvironment.
// registration of a DataSet is equivalent
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

// Table with two fields (String name, Integer age)
val table: Table = ...

// convert the Table into an append DataStream of Row
val dsRow: DataStream[Row] = tableEnv.toAppendStream[Row](table)

// convert the Table into an append DataStream of Tuple2[String, Int]
val dsTuple: DataStream[(String, Int)] dsTuple =
  tableEnv.toAppendStream[(String, Int)](table)

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream[(Boolean, X)].
//   The boolean field indicates the type of the change.
//   True is INSERT, false is DELETE.
val retractStream: DataStream[(Boolean, Row)] = tableEnv.toRetractStream[Row](table)
```
{{< /tab >}}
{{< /tabs >}}

**注意：** 文档[动态表](streaming/dynamic_tables.html)给出了有关动态表及其属性的详细讨论。

<span class="label label-danger">注意</span> **一旦 Table 被转化为 DataStream，必须使用 StreamExecutionEnvironment 的 execute 方法执行该 DataStream 作业。**

<a name="convert-a-table-into-a-dataset"></a>

#### 将表转换成 DataSet

将 `Table` 转换成 `DataSet` 的过程如下：

{{< tabs "65c0fb90-3108-4dc4-9130-626712549183" >}}
{{< tab "Java" >}}
```java
// get BatchTableEnvironment
BatchTableEnvironment tableEnv = BatchTableEnvironment.create(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into a DataSet of Row by specifying a class
DataSet<Row> dsRow = tableEnv.toDataSet(table, Row.class);

// convert the Table into a DataSet of Tuple2<String, Integer> via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataSet<Tuple2<String, Integer>> dsTuple =
  tableEnv.toDataSet(table, tupleType);
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get TableEnvironment
// registration of a DataSet is equivalent
val tableEnv = BatchTableEnvironment.create(env)

// Table with two fields (String name, Integer age)
val table: Table = ...

// convert the Table into a DataSet of Row
val dsRow: DataSet[Row] = tableEnv.toDataSet[Row](table)

// convert the Table into a DataSet of Tuple2[String, Int]
val dsTuple: DataSet[(String, Int)] = tableEnv.toDataSet[(String, Int)](table)
```
{{< /tab >}}
{{< /tabs >}}

<span class="label label-danger">注意</span> **一旦 Table 被转化为 DataSet，必须使用 ExecutionEnvironment 的 execute 方法执行该 DataSet 作业。**

{{< top >}}

<a name="mapping-of-data-types-to-table-schema"></a>

### 数据类型到 Table Schema 的映射

Flink 的 DataStream 和 DataSet APIs 支持多样的数据类型。例如 Tuple（Scala 内置以及Flink Java tuple）、POJO 类型、Scala case class 类型以及 Flink 的 Row 类型等允许嵌套且有多个可在表的表达式中访问的字段的复合数据类型。其他类型被视为原子类型。下面，我们讨论 Table API 如何将这些数据类型类型转换为内部 row 表示形式，并提供将 `DataStream` 转换成 `Table` 的样例。

数据类型到 table schema 的映射有两种方式：**基于字段位置**或**基于字段名称**。

**基于位置映射**

基于位置的映射可在保持字段顺序的同时为字段提供更有意义的名称。这种映射方式可用于*具有特定的字段顺序*的复合数据类型以及原子类型。如 tuple、row 以及 case class 这些复合数据类型都有这样的字段顺序。然而，POJO 类型的字段则必须通过名称映射（参见下一章）。可以将字段投影出来，但不能使用`as`重命名。

定义基于位置的映射时，输入数据类型中一定不能存在指定的名称，否则 API 会假定应该基于字段名称进行映射。如果未指定任何字段名称，则使用默认的字段名称和复合数据类型的字段顺序，或者使用 `f0` 表示原子类型。

{{< tabs "1abe538a-9dc4-4fb4-900a-f27cb888d20d" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section;

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "myLong" only
Table table = tableEnv.fromDataStream(stream, $("myLong"));

// convert DataStream into Table with field names "myLong" and "myInt"
Table table = tableEnv.fromDataStream(stream, $("myLong"), $("myInt"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

val stream: DataStream[(Long, Int)] = ...

// convert DataStream into Table with default field names "_1" and "_2"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field "myLong" only
val table: Table = tableEnv.fromDataStream(stream, $"myLong")

// convert DataStream into Table with field names "myLong" and "myInt"
val table: Table = tableEnv.fromDataStream(stream, $"myLong", $"myInt")
```
{{< /tab >}}
{{< /tabs >}}

**基于名称的映射**

基于名称的映射适用于任何数据类型包括 POJO 类型。这是定义 table schema 映射最灵活的方式。映射中的所有字段均按名称引用，并且可以通过 `as` 重命名。字段可以被重新排序和映射。

若果没有指定任何字段名称，则使用默认的字段名称和复合数据类型的字段顺序，或者使用 `f0` 表示原子类型。

{{< tabs "e6952073-a5a0-45ff-800e-bd4126c09b26" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "f1" only
Table table = tableEnv.fromDataStream(stream, $("f1"));

// convert DataStream into Table with swapped fields
Table table = tableEnv.fromDataStream(stream, $("f1"), $("f0"));

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
Table table = tableEnv.fromDataStream(stream, $("f1").as("myInt"), $("f0").as("myLong"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

val stream: DataStream[(Long, Int)] = ...

// convert DataStream into Table with default field names "_1" and "_2"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field "_2" only
val table: Table = tableEnv.fromDataStream(stream, $"_2")

// convert DataStream into Table with swapped fields
val table: Table = tableEnv.fromDataStream(stream, $"_2", $"_1")

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
val table: Table = tableEnv.fromDataStream(stream, $"_2" as "myInt", $"_1" as "myLong")
```
{{< /tab >}}
{{< /tabs >}}

<a name="atomic-types"></a>

#### 原子类型

Flink 将基础数据类型（`Integer`、`Double`、`String`）或者通用数据类型（不可再拆分的数据类型）视为原子类型。原子类型的 `DataStream` 或者 `DataSet` 会被转换成只有一条属性的 `Table`。属性的数据类型可以由原子类型推断出，还可以重新命名属性。

{{< tabs "03abca94-5825-4ba7-8ef0-213362c3aaff" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Long> stream = ...

// convert DataStream into Table with default field name "f0"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field name "myLong"
Table table = tableEnv.fromDataStream(stream, $("myLong"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

val stream: DataStream[Long] = ...

// convert DataStream into Table with default field name "f0"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field name "myLong"
val table: Table = tableEnv.fromDataStream(stream, $"myLong")
```
{{< /tab >}}
{{< /tabs >}}

<a name="tuples-scala-and-java-and-case-classes-scala-only"></a>

#### Tuple类型（Scala 和 Java）和 Case Class类型（仅 Scala）

Flink 支持 Scala 的内置 tuple 类型并给 Java 提供自己的 tuple 类型。两种 tuple 的 DataStream 和 DataSet 都能被转换成表。可以通过提供所有字段名称来重命名字段（基于位置映射）。如果没有指明任何字段名称，则会使用默认的字段名称。如果引用了原始字段名称（对于 Flink tuple 为`f0`、`f1` ... ...，对于 Scala tuple 为`_1`、`_2` ... ...），则 API 会假定映射是基于名称的而不是基于位置的。基于名称的映射可以通过 `as` 对字段和投影进行重新排序。

{{< tabs "130f44c6-7432-465a-ae8a-b4c436888361" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// convert DataStream into Table with default field names "f0", "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myLong", "myString" (position-based)
Table table = tableEnv.fromDataStream(stream, $("myLong"), $("myString"));

// convert DataStream into Table with reordered fields "f1", "f0" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1"), $("f0"));

// convert DataStream into Table with projected field "f1" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1"));

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1").as("myString"), $("f0").as("myLong"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

val stream: DataStream[(Long, String)] = ...

// convert DataStream into Table with renamed default field names '_1, '_2
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with field names "myLong", "myString" (position-based)
val table: Table = tableEnv.fromDataStream(stream, $"myLong", $"myString")

// convert DataStream into Table with reordered fields "_2", "_1" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"_2", $"_1")

// convert DataStream into Table with projected field "_2" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"_2")

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"_2" as "myString", $"_1" as "myLong")

// define case class
case class Person(name: String, age: Int)
val streamCC: DataStream[Person] = ...

// convert DataStream into Table with default field names 'name, 'age
val table = tableEnv.fromDataStream(streamCC)

// convert DataStream into Table with field names 'myName, 'myAge (position-based)
val table = tableEnv.fromDataStream(streamCC, $"myName", $"myAge")

// convert DataStream into Table with reordered and aliased fields "myAge", "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"age" as "myAge", $"name" as "myName")

```
{{< /tab >}}
{{< /tabs >}}

<a name="pojo-java-and-scala"></a>

#### POJO 类型 （Java 和 Scala）

Flink 支持 POJO 类型作为复合类型。确定 POJO 类型的规则记录在[这里]({{< ref "docs/dev/serialization/types_serialization" >}}#pojos).

在不指定字段名称的情况下将 POJO 类型的 `DataStream` 或 `DataSet` 转换成 `Table` 时，将使用原始 POJO 类型字段的名称。名称映射需要原始名称，并且不能按位置进行。字段可以使用别名（带有 `as` 关键字）来重命名，重新排序和投影。

{{< tabs "c4bd0a25-c14c-44f8-8353-add5c453c4fd" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// Person is a POJO with fields "name" and "age"
DataStream<Person> stream = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("age").as("myAge"), $("name").as("myName"));

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name"));

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

// Person is a POJO with field names "name" and "age"
val stream: DataStream[Person] = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"age" as "myAge", $"name" as "myName")

// convert DataStream into Table with projected field "name" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"name")

// convert DataStream into Table with projected and renamed field "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"name" as "myName")
```
{{< /tab >}}
{{< /tabs >}}

<a name="row"></a>

#### Row类型

`Row` 类型支持任意数量的字段以及具有 `null` 值的字段。字段名称可以通过 `RowTypeInfo` 指定，也可以在将 `Row` 的 `DataStream` 或 `DataSet` 转换为 `Table` 时指定。Row 类型的字段映射支持基于名称和基于位置两种方式。字段可以通过提供所有字段的名称的方式重命名（基于位置映射）或者分别选择进行投影/排序/重命名（基于名称映射）。

{{< tabs "3434160b-c826-4064-897c-18d7d7cf7103" >}}
{{< tab "Java" >}}
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
DataStream<Row> stream = ...

// convert DataStream into Table with default field names "name", "age"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
Table table = tableEnv.fromDataStream(stream, $("myName"), $("myAge"));

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"), $("age").as("myAge"));

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name"));

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
// get a TableEnvironment
val tableEnv: StreamTableEnvironment = ... // see "Create a TableEnvironment" section

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
val stream: DataStream[Row] = ...

// convert DataStream into Table with default field names "name", "age"
val table: Table = tableEnv.fromDataStream(stream)

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
val table: Table = tableEnv.fromDataStream(stream, $"myName", $"myAge")

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"name" as "myName", $"age" as "myAge")

// convert DataStream into Table with projected field "name" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"name")

// convert DataStream into Table with projected and renamed field "myName" (name-based)
val table: Table = tableEnv.fromDataStream(stream, $"name" as "myName")
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}

<a name="query-optimization"></a>

查询优化
------------------

{{< tabs "76bc65c8-6242-448a-a9d8-1b2909ad2198" >}}
{{< tab "Blink planner" >}}
Apache Flink 使用并扩展了 Apache Calcite 来执行复杂的查询优化。
这包括一系列基于规则和成本的优化，例如：

* 基于 Apache Calcite 的子查询解相关
* 投影剪裁
* 分区剪裁
* 过滤器下推
* 子计划消除重复数据以避免重复计算
* 特殊子查询重写，包括两部分：
    * 将 IN 和 EXISTS 转换为 left semi-joins
    * 将 NOT IN 和 NOT EXISTS 转换为 left anti-join
* 可选 join 重新排序
    * 通过 `table.optimizer.join-reorder-enabled` 启用

**注意：** 当前仅在子查询重写的结合条件下支持 IN / EXISTS / NOT IN / NOT EXISTS。

优化器不仅基于计划，而且还基于可从数据源获得的丰富统计信息以及每个算子（例如 io，cpu，网络和内存）的细粒度成本来做出明智的决策。

高级用户可以通过 `CalciteConfig` 对象提供自定义优化，可以通过调用  `TableEnvironment＃getConfig＃setPlannerConfig` 将其提供给 TableEnvironment。
{{< /tab >}}
{{< tab "Old planner" >}}
Apache Flink 利用 Apache Calcite 来优化和翻译查询。当前执行的优化包括投影和过滤器下推，子查询消除以及其他类型的查询重写。原版计划程序尚未优化 join 的顺序，而是按照查询中定义的顺序执行它们（FROM 子句中的表顺序和/或 WHERE 子句中的 join 谓词顺序）。

通过提供一个 `CalciteConfig` 对象，可以调整在不同阶段应用的优化规则集合。这个对象可以通过调用构造器 `CalciteConfig.createBuilder()` 创建，并通过调用 `tableEnv.getConfig.setPlannerConfig(calciteConfig)` 提供给 TableEnvironment。
{{< /tab >}}
{{< /tabs >}}

<a name="explaining-a-table"></a>

解释表
------------------

Table API 提供了一种机制来解释计算 `Table` 的逻辑和优化查询计划。
这是通过 `Table.explain()` 方法或者 `StatementSet.explain()` 方法来完成的。`Table.explain()` 返回一个 Table 的计划。`StatementSet.explain()` 返回多 sink 计划的结果。它返回一个描述三种计划的字符串：

1. 关系查询的抽象语法树（the Abstract Syntax Tree），即未优化的逻辑查询计划，
2. 优化的逻辑查询计划，以及
3. 物理执行计划。

可以用 `TableEnvironment.explainSql()` 方法和 `TableEnvironment.executeSql()` 方法支持执行一个 `EXPLAIN` 语句获取逻辑和优化查询计划，请参阅 [EXPLAIN]({{< ref "docs/dev/table/sql/explain" >}}) 页面.

以下代码展示了一个示例以及对给定 `Table` 使用 `Table.explain()` 方法的相应输出：

{{< tabs "9cd6973f-8b4e-4e80-8197-ba95713b489f" >}}
{{< tab "Java" >}}
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);

DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));

// explain Table API
Table table1 = tEnv.fromDataStream(stream1, $("count"), $("word"));
Table table2 = tEnv.fromDataStream(stream2, $("count"), $("word"));
Table table = table1
  .where($("word").like("F%"))
  .unionAll(table2);
System.out.println(table.explain());

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tEnv = StreamTableEnvironment.create(env)

val table1 = env.fromElements((1, "hello")).toTable(tEnv, $"count", $"word")
val table2 = env.fromElements((1, "hello")).toTable(tEnv, $"count", $"word")
val table = table1
  .where($"word".like("F%"))
  .unionAll(table2)
println(table.explain())

```
{{< /tab >}}
{{< tab "Python" >}}
```python
env = StreamExecutionEnvironment.get_execution_environment()
t_env = StreamTableEnvironment.create(env)

table1 = t_env.from_elements([(1, "hello")], ["count", "word"])
table2 = t_env.from_elements([(1, "hello")], ["count", "word"])
table = table1 \
    .where(table1.word.like('F%')) \
    .union_all(table2)
print(table.explain())

```
{{< /tab >}}
{{< /tabs >}}

上述例子的结果是：
<div>
```text
== Abstract Syntax Tree ==
LogicalUnion(all=[true])
  LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
    FlinkLogicalDataStreamScan(id=[1], fields=[count, word])
  FlinkLogicalDataStreamScan(id=[2], fields=[count, word])

== Optimized Logical Plan ==
DataStreamUnion(all=[true], union all=[count, word])
  DataStreamCalc(select=[count, word], where=[LIKE(word, _UTF-16LE'F%')])
    DataStreamScan(id=[1], fields=[count, word])
  DataStreamScan(id=[2], fields=[count, word])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

Stage 2 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 3 : Operator
		content : from: (count, word)
		ship_strategy : REBALANCE

		Stage 4 : Operator
			content : where: (LIKE(word, _UTF-16LE'F%')), select: (count, word)
			ship_strategy : FORWARD

			Stage 5 : Operator
				content : from: (count, word)
				ship_strategy : REBALANCE
```
</div>

以下代码展示了一个示例以及使用 `StatementSet.explain()` 的多 sink 计划的相应输出：

{{< tabs "77dd0499-7cec-4283-8353-0aadf5a3ab44" >}}
{{< tab "Java" >}}
```java

EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
TableEnvironment tEnv = TableEnvironment.create(settings);

final Schema schema = new Schema()
    .field("count", DataTypes.INT())
    .field("word", DataTypes.STRING());

tEnv.connect(new FileSystem().path("/source/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource1");
tEnv.connect(new FileSystem().path("/source/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource2");
tEnv.connect(new FileSystem().path("/sink/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink1");
tEnv.connect(new FileSystem().path("/sink/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink2");

StatementSet stmtSet = tEnv.createStatementSet();

Table table1 = tEnv.from("MySource1").where($("word").like("F%"));
stmtSet.addInsert("MySink1", table1);

Table table2 = table1.unionAll(tEnv.from("MySource2"));
stmtSet.addInsert("MySink2", table2);

String explanation = stmtSet.explain();
System.out.println(explanation);

```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val settings = EnvironmentSettings.newInstance.useBlinkPlanner.inStreamingMode.build
val tEnv = TableEnvironment.create(settings)

val schema = new Schema()
    .field("count", DataTypes.INT())
    .field("word", DataTypes.STRING())

tEnv.connect(new FileSystem().path("/source/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource1")
tEnv.connect(new FileSystem().path("/source/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource2")
tEnv.connect(new FileSystem().path("/sink/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink1")
tEnv.connect(new FileSystem().path("/sink/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink2")

val stmtSet = tEnv.createStatementSet()

val table1 = tEnv.from("MySource1").where($"word".like("F%"))
stmtSet.addInsert("MySink1", table1)

val table2 = table1.unionAll(tEnv.from("MySource2"))
stmtSet.addInsert("MySink2", table2)

val explanation = stmtSet.explain()
println(explanation)

```
{{< /tab >}}
{{< tab "Python" >}}
```python
settings = EnvironmentSettings.new_instance().use_blink_planner().in_streaming_mode().build()
t_env = TableEnvironment.create(environment_settings=settings)

schema = Schema()
    .field("count", DataTypes.INT())
    .field("word", DataTypes.STRING())

t_env.connect(FileSystem().path("/source/path1")))
    .with_format(Csv().deriveSchema())
    .with_schema(schema)
    .create_temporary_table("MySource1")
t_env.connect(FileSystem().path("/source/path2")))
    .with_format(Csv().deriveSchema())
    .with_schema(schema)
    .create_temporary_table("MySource2")
t_env.connect(FileSystem().path("/sink/path1")))
    .with_format(Csv().deriveSchema())
    .with_schema(schema)
    .create_temporary_table("MySink1")
t_env.connect(FileSystem().path("/sink/path2")))
    .with_format(Csv().deriveSchema())
    .with_schema(schema)
    .create_temporary_table("MySink2")

stmt_set = t_env.create_statement_set()

table1 = t_env.from_path("MySource1").where(col('word').like('F%'))
stmt_set.add_insert("MySink1", table1)

table2 = table1.union_all(t_env.from_path("MySource2"))
stmt_set.add_insert("MySink2", table2)

explanation = stmt_set.explain()
print(explanation)

```
{{< /tab >}}
{{< /tabs >}}

多 sink 计划的结果是：
<div>
```text

== Abstract Syntax Tree ==
LogicalLegacySink(name=[MySink1], fields=[count, word])
+- LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
   +- LogicalTableScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]])

LogicalLegacySink(name=[MySink2], fields=[count, word])
+- LogicalUnion(all=[true])
   :- LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]])
   +- LogicalTableScan(table=[[default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]]])

== Optimized Logical Plan ==
Calc(select=[count, word], where=[LIKE(word, _UTF-16LE'F%')], reuse_id=[1])
+- TableSourceScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]], fields=[count, word])

LegacySink(name=[MySink1], fields=[count, word])
+- Reused(reference_id=[1])

LegacySink(name=[MySink2], fields=[count, word])
+- Union(all=[true], union=[count, word])
   :- Reused(reference_id=[1])
   +- TableSourceScan(table=[[default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]]], fields=[count, word])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 2 : Operator
		content : CsvTableSource(read fields: count, word)
		ship_strategy : REBALANCE

		Stage 3 : Operator
			content : SourceConversion(table:Buffer(default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]), fields:(count, word))
			ship_strategy : FORWARD

			Stage 4 : Operator
				content : Calc(where: (word LIKE _UTF-16LE'F%'), select: (count, word))
				ship_strategy : FORWARD

				Stage 5 : Operator
					content : SinkConversionToRow
					ship_strategy : FORWARD

					Stage 6 : Operator
						content : Map
						ship_strategy : FORWARD

Stage 8 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 9 : Operator
		content : CsvTableSource(read fields: count, word)
		ship_strategy : REBALANCE

		Stage 10 : Operator
			content : SourceConversion(table:Buffer(default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]), fields:(count, word))
			ship_strategy : FORWARD

			Stage 12 : Operator
				content : SinkConversionToRow
				ship_strategy : FORWARD

				Stage 13 : Operator
					content : Map
					ship_strategy : FORWARD

					Stage 7 : Data Sink
						content : Sink: CsvTableSink(count, word)
						ship_strategy : FORWARD

						Stage 14 : Data Sink
							content : Sink: CsvTableSink(count, word)
							ship_strategy : FORWARD

```
</div>

{{< top >}}


