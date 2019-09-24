#查询

## 创建数据库

创建 db_name 数据库

```sql
CREATE DATABASE [IF NOT EXISTS] db_name
```

`一个数据库` 是表的一个目录。
如果包含 `IF NOT EXISTS`， 如果数据库已经存在， 查询不返回错误。

## 创建表

`CREATE TABLE` 查询有多种形式。

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1]，
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2]，
    。。。
) ENGINE = engine
```

在 'db' 数据库中或者当前数据库中创建一个表名为 'name' 的表，如果 'db' 没有被设置， 表的结构指定在 brackets 和 'engine' 引擎中。
该表的结构是多个列的描述。 如果引擎支持索引，索引可以在表引擎的参数中指定。

在最简单的情况下， 一个列的描述符是 `name type`。 例如: `RegionID UInt32`。
表达式也可以定义为默认值。

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name AS [db2.]name2 [ENGINE = engine]
```

用另外一个表的结构创建表。 你能够为表指定不同的引擎。 如果引擎没有被指定， 相同的引擎将被应用于 `db2。name2` 表。

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name ENGINE = engine AS SELECT ...
```

创建一个带有`SELECT`查询结果类似结构的表， 带有 'engine' 引擎， 同时从SELECT中添加数据。

在所有的情况下， 如果 `IF NOT EXISTS` 被指定， 查询不返回错误， 如果表已经存在，查询不返回错误。这种情况下， 此查询不做任何事情。

### 默认值

列描述中可以指定一个表达式来表示默认值，用下列其中之一方式：`DEFAULT expr`， `MATERIALIZED expr`， `ALIAS expr`。
示例: `URLDomain String DEFAULT domain(URL)`。

如果没有指定表达式来表示默认值， 默认值将被设置为0(数字)， 空字符串(字符)， 空数组(数组)， 和 `0000-00-00`(日期) 或者 `0000-00-00 00:00:00` (日期)。 NULLs 值不被支持。

如果默认表达式被定义， 列类型是可选的。 如果没有一个明确的定义类型， 默认表达式类型被使用。 示例: `EventDate DEFAULT toDate(EventTime)` –  'Date' 类型将被用在'EventDate' 列上。

如果数据类型和默认的表达式被明确定义出来， 此表达式将被转换到指定的类型(使用类型转换函数)。 示例: `Hits UInt32 DEFAULT 0` 与 `Hits UInt32 DEFAULT toUInt32(0)`意思相同。

默认表达式可以定义为表常量和列的任意表达式。 当创建和更改表结构时， 它将检查表达式是否包含循环。 对于INSERT操作， 它检查了表达式是否可解析 – 能够计算所有传递进去的列。

`DEFAULT expr`

正常默认值。 如果 INSERT 查询并不指定对应的列， 它将通过计算对应的表达式来填充。

`MATERIALIZED expr`

物化表达式。 这样一个列不能被指定到INSERT操作中， 因为它经常被计算。
对于一个INSERT操作来说，没有一个列的列表， 这些列将不被考虑。
另外， 此列不能被替换，当在SELECT语句中使用*号时。 这个将保证不可变性， 获得的 dump 使用 `SELECT *` 插回到表中， 使用 INSERT不能指定列的列表。

`ALIAS expr`

近义词。 这样的列根本不在表中保存。
它的值不插入到表中， 同时它没有被替换， 当在SELECT查询中使用*号时。
如果昵称在查询解析时被扩展， 它能够被用在 SELECTs 中。

当使用 ALTER 查询时添加新的列， 这些列中旧的数据不会被写。 相反， 当读取旧的数据时， 新列没有值， 表达式默认情况下是被计算。 然而， 如果运行表达式需要不同的列， 列没有在查询中提示， 这些列将被额外读取， 但是仅对于需要它的数据块。

如果你添加新列到一个表中， 但是又改变了它的默认表达式， 此值所使用的旧值将被改变 (此值并不保存在磁盘上)。 注意:当运行 Merge 北京线程时， 对于丢失正在合并部分的列的数据 被写到已经合并的部分。

在嵌套的数据结构中， 不可能设置默认值。

### 临时表

在所有情况下， 如果 `TEMPORARY` 被指定， 一个临时表将被创建。 临时表有如下的特性:

- 当 session 结束时， 临时表消失， 包括连接丢失。
- 一个临时表用Memory引擎被创建。 其他表引擎不被支持。
- 对于一个临时表， DB 不能被指定。 它被创建在数据库之外。
- 如果一个临时表与另外的表名称相同， 同时一个查询指定表名， 不指定DB， 临时表将被使用。
- 对于分布式查询处理， 查询中使用的临时表被传递给远程服务器。

在大多数情况下， 临时表不是手工创建， 而是使用外表查询， 或者是分布式 `(GLOBAL) IN` 时。 对于更多信息， 查看章节

分布式 DDL 查询 (ON CLUSTER 语句)
----------------------------------------------

`CREATE`， `DROP`， `ALTER`， 和 `RENAME` 查询支持在集群中分布式执行。
例如， 如下的查询创建了 `all_hits` `Distributed` 表， 在每个主机中的 `cluster`:

```sql
CREATE TABLE IF NOT EXISTS all_hits ON CLUSTER cluster (p Date, i Int32) ENGINE = Distributed(cluster, default, hits)
```

为了正确地运行这些查询， 每个主机必须有相同的集群定义 (为了简化同步配置， 你能够从ZooKeeper进行订阅)。 他们也必须连接到ZooKeeper 服务器上。
查询的本地版本将最终在集群的每台机器上被实现， 即使一些主机当前是不可用的。 在单个主机内的执行查询顺序是有保障的。
` ALTER` 查询对于复制表来说还不支持。

## CREATE VIEW

```sql
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...
```

创建一个视图。 有2中类型的视图: Normal 和 MATERIALIZED。

当创建一个物化视图时， 你必须指定 ENGINE – 对于存储数据的表引擎。

一个物化视图工作如下: 当使用SELECT插入数据到表中时， 插入数据的部分通过SELECT查询被转化， 同时结果被插入到视图中。

Normal视图不存储任何数据， 但是从另外一个表中执行一个读操作。 换句话说， 一个normal视图不比一个保存的查询多。 当从一个视图中读取时， 这个保存的查询被用于作为FROM语句中的一个子查询。

作为一个示例， 假设你已经创建了一个视图:

```sql
CREATE VIEW view AS SELECT ...
```

同时写到一个查询中:

```sql
SELECT a, b, c FROM view
```

此查询最终与使用子查询等价:

```sql
SELECT a, b, c FROM (SELECT ...)
```

物化视图保存数据通过对应的SELECT查询来转换。

当创建一个物化视图时， 你必须指定 ENGINE – 保存数据的表引擎。

一个物化视图被安排如下: 当插入数据到指定SELECT的表中时， 插入数据的部分通过SELECT查询来转化， 同时结果被插入到视图中。

如果你指定了POPULATE， 当它创建时， 现有的表数据被插入到视图中， 好像进行了一个 `CREATE TABLE ... AS SELECT ...` 。 否则， 在创建视图之后， 此查询仅包含插入在表中的数据。 我们不推荐使用 POPULATE， 在创建视图过程中插入到表中的数据不插入到视图中。

一个 `SELECT` 查询能够包含 `DISTINCT`, `GROUP BY`, `ORDER BY`, `LIMIT`... 注意: 对应的转换独立地在每个插入数据的数据块中执行。 例如， 如果 `GROUP BY` 被设置， 数据在插入时就进行聚合， 但是仅在单个插入的数据包中。 数据并不进一步聚合。 此异常是当使用一个ENGINE引擎时， 独立运行数据聚合， 例如 `SummingMergeTree`。

在物化视图的 `ALTER` 查询执行上正在开发中， 因此他们使用上不是特别方便。 如果物化视图使用结构 ``TO [db.]name``， 你能够 ``DETACH`` 此视图， 在目标表中运行 ``ALTER``， 然后 ``ATTACH`` 之前卸载的 (``DETACH``) 视图。

视图看起来与正常表一样。 例如， 他们以`SHOW TABLES`查询的结果被列出。

在删除视图方面，没有另外的语句。 如果要删除视图， 则使用 `DROP TABLE`。

## 挂载

此查询与`CREATE`相同， 但是

- 并没有使用`CREATE` 关键字， 而是使用了`ATTACH`。
- 此查询并不在磁盘上创建数据， 而是假设数据已经在指定的位置， 仅是添加有关表的信息到服务器中。
当执行 ATTACH 查询之后， 服务器将知道表的存在。

如果表在之前执行了`DETACH`， 这意味着此结构是已知的， 你能够使用没有定义结构的简写。

```sql
ATTACH TABLE [IF NOT EXISTS] [db.]name
```

此查询在启动服务器时被执行。 服务器保存了表的元数据作为带有`ATTACH`查询的文件， 它直接运行在启动时(在服务器上创建)。

## DROP

此查询有2种类型: `DROP DATABASE` 和 `DROP TABLE`。

```sql
DROP DATABASE [IF EXISTS] db [ON CLUSTER cluster]
```

删除'db'数据库内所有的表， 然后删除'db'自己。
如果 `IF EXISTS` 被指定， 如果此数据库不存在， 它不返回错误。

```sql
DROP TABLE [IF EXISTS] [db.]name [ON CLUSTER cluster]
```

删除此表。
如果 `IF EXISTS` 被指定， 它不返回一个错误， 如果此表不存在或者数据库不存在。

## DETACH

删除服务器中有关'name' 表中的信息。 然后服务器不知道有此表的存在。

```sql
DETACH TABLE [IF EXISTS] [db.]name
```

它不删除表的数据或元数据。 在下一次服务器启动时， 服务器将读取元数据， 再次找到此表。
与之类似， 使用`ATTACH`查询， 一个 "detached" 表能够被重新挂载 (除了系统表， 它没有元数据存储)。

注意， 没有 `DETACH DATABASE` 语句。

## 重命名

重命名一个或多个表。

```sql
RENAME TABLE [db11.]name11 TO [db12.]name12, [db21.]name21 TO [db22.]name22, ... [ON CLUSTER cluster]
```

所有的表都在全局锁下进行重命名。 重命名表是一个轻操作。 如果你在TO之后提示另外一个数据库， 此表将移动到此数据库中。 然而， 带有数据库的目录必须在相同的文件系统内 (否则， 将发生错误)。

<a name="query_language_queries_alter"></a>

## 更新

目前 `ALTER` 语句仅支持 `*MergeTree` 表， `Merge` 和 `Distributed`。 此查询有一些变种版本。

### 列操作

更新表结构。

```sql
ALTER TABLE [db].name [ON CLUSTER cluster] ADD|DROP|MODIFY COLUMN ...
```

在查询中， 指定一个或多个逗号分隔动作的列表。
每一个动作都是在一个列上的操作。

如下的动作支持:

```sql
ADD COLUMN name [type] [default_expr] [AFTER name_after]
```

添加一个新的列到特定名称、类型和 `default_expr`的表中 (查询章节 "默认表达式")。 如果你指定了 `AFTER name_after` (另一个列的名称)， 列被添加到指定字段之后。 否则， 列被添加到表的末尾。 注意目前没有方法可以添加列到表的开头。 一系列动作时候， 'name_after' 是一个列的名称， 此列被添加到之前的动作上。

添加一个新列即可改变列的结构， 在执行动作之前。 在ALTER之后， 此数据并没有出现在磁盘上。 当从表中读取数据时， 如果数据丢失， 它使用默认值填充 (如果有1，执行默认表达式， 或者使用0或空字符串)。 对于一个字段， 当从表中读取数据时， 如果数据为空， 它使用默认值填充 (如果有1，执行默认表达式， 或者使用0或空字符串)。 在合并数据块后， 字段出现在磁盘上，(请查看 MergeTree)。

此方法允许我们完成 ALTER 查询， 不需要增加旧数据的数据量。

```sql
DROP COLUMN name
```

删除带有名称'name'的列。
删除文件系统中的数据。 此语句删除整个文件， 查询立即完成。

```sql
MODIFY COLUMN name [type] [default_expr]
```

更新 'name' 字段的类型为 'type' 同时默认的表达式为 'default_expr'。 当更新类型时值被转换，如果'toType'函数被使用时。

如果仅有默认的表达式被改变， 此查询并不做任何复杂的操作， 并且查询立即完成。

更新字段类型是一个稍微复杂的动作 – 它更新带有数据的文件内容。 对于大表， 它可能需要比较长的时间。

有下面几个处理阶段:

- 准备带有更新数据的临时(新)文件。
- 重命名旧文件。
- 重命名临时(新)文件到旧名称。
- 删除旧文件。

Only the first stage takes time。 If there is a failure at this stage， the data is not changed。
If there is a failure during one of the successive stages， data can be restored manually。 The exception is if the old files were deleted from the file system but the data for the new files did not get written to the disk and was lost。

There is no support for changing the column type in arrays and nested data structures。

The `ALTER` query lets you create and delete separate elements (columns) in nested data structures， but not whole nested data structures。 To add a nested data structure， you can add columns with a name like `name。nested_name` and the type `Array(T)`。 A nested data structure is equivalent to multiple array columns with a name that has the same prefix before the dot。

There is no support for deleting columns in the primary key or the sampling key (columns that are in the `ENGINE` expression)。 Changing the type for columns that are included in the primary key is only possible if this change does not cause the data to be modified (for example， it is allowed to add values to an Enum or change a type with `DateTime`  to `UInt32`)。

If the `ALTER` query is not sufficient for making the table changes you need， you can create a new table， copy the data to it using the `INSERT SELECT` query， then switch the tables using the `RENAME` query and delete the old table。

The `ALTER` query blocks all reads and writes for the table。 In other words， if a long `SELECT` is running at the time of the `ALTER` query， the `ALTER` query will wait for it to complete。 At the same time， all new queries to the same table will wait while this `ALTER` is running。

For tables that don't store data themselves (such as `Merge` and `Distributed`)， `ALTER` just changes the table structure， and does not change the structure of subordinate tables。 For example， when running ALTER for a `Distributed` table， you will also need to run `ALTER` for the tables on all remote servers。

The `ALTER` query for changing columns is replicated。 The instructions are saved in ZooKeeper， then each replica applies them。 All `ALTER` queries are run in the same order。 The query waits for the appropriate actions to be completed on the other replicas。 However， a query to change columns in a replicated table can be interrupted， and all actions will be performed asynchronously。

### Manipulations with partitions and parts

It only works for tables in the `MergeTree` family。 The following operations are available:

- `DETACH PARTITION` – Move a partition to the 'detached' directory and forget it。
- `DROP PARTITION` – Delete a partition。
- `ATTACH PART|PARTITION` – Add a new part or partition from the `detached` directory to the table。
- `FREEZE PARTITION` – Create a backup of a partition。
- `FETCH PARTITION` – Download a partition from another server。

Each type of query is covered separately below。

A partition in a table is data for a single calendar month。 This is determined by the values of the date key specified in the table engine parameters。 Each month's data is stored separately in order to simplify manipulations with this data。

A "part" in the table is part of the data from a single partition， sorted by the primary key。

You can use the `system。parts` table to view the set of table parts and partitions:

```sql
SELECT * FROM system.parts WHERE active
```

`active` – Only count active parts。 Inactive parts are， for example， source parts remaining after merging to a larger part – these parts are deleted approximately 10 minutes after merging。

Another way to view a set of parts and partitions is to go into the directory with table data。
Data directory: `/var/lib/clickhouse/data/database/table/`，
where `/var/lib/clickhouse/` is the path to the ClickHouse data， 'database' is the database name， and 'table' is the table name。 Example:

```bash
$ ls -l /var/lib/clickhouse/data/test/visits/
total 48
drwxrwxrwx 2 clickhouse clickhouse 20480 may   13 02:58 20140317_20140323_2_2_0
drwxrwxrwx 2 clickhouse clickhouse 20480 may   13 02:58 20140317_20140323_4_4_0
drwxrwxrwx 2 clickhouse clickhouse  4096 may   13 02:55 detached
-rw-rw-rw- 1 clickhouse clickhouse     2 may   13 02:58 increment。txt
```

Here， `20140317_20140323_2_2_0` and ` 20140317_20140323_4_4_0` are the directories of data parts。

Let's break down the name of the first part: `20140317_20140323_2_2_0`。

- `20140317` is the minimum date of the data in the chunk。
- `20140323` is the maximum date of the data in the chunk。
- `2` is the minimum number of the data block。
- `2` is the maximum number of the data block。
- `0` is the chunk level (the depth of the merge tree it is formed from)。

Each piece relates to a single partition and contains data for just one month。
`201403` is the name of the partition。 A partition is a set of parts for a single month。

On an operating server， you can't manually change the set of parts or their data on the file system， since the server won't know about it。
For non-replicated tables， you can do this when the server is stopped， but we don't recommended it。
For replicated tables， the set of parts can't be changed in any case。

The `detached` directory contains parts that are not used by the server - detached from the table using the `ALTER 。。。 DETACH` query。 Parts that are damaged are also moved to this directory， instead of deleting them。 You can add， delete， or modify the data in the 'detached' directory at any time – the server won't know about this until you make the `ALTER TABLE 。。。 ATTACH` query。

```sql
ALTER TABLE [db.]table DETACH PARTITION 'name'
```

Move all data for partitions named 'name' to the 'detached' directory and forget about them。
The partition name is specified in YYYYMM format。 It can be indicated in single quotes or without them。

After the query is executed， you can do whatever you want with the data in the 'detached' directory — delete it from the file system， or just leave it。

The query is replicated – data will be moved to the 'detached' directory and forgotten on all replicas。 The query can only be sent to a leader replica。 To find out if a replica is a leader， perform SELECT to the 'system。replicas' system table。 Alternatively， it is easier to make a query on all replicas， and all except one will throw an exception。

```sql
ALTER TABLE [db.]table DROP PARTITION 'name'
```

The same as the `DETACH` operation。 Deletes data from the table。 Data parts will be tagged as inactive and will be completely deleted in approximately 10 minutes。 The query is replicated – data will be deleted on all replicas。

```sql
ALTER TABLE [db.]table ATTACH PARTITION|PART 'name'
```

Adds data to the table from the 'detached' directory。

It is possible to add data for an entire partition or a separate part。 For a part， specify the full name of the part in single quotes。

The query is replicated。 Each replica checks whether there is data in the 'detached' directory。 If there is data， it checks the integrity， verifies that it matches the data on the server that initiated the query， and then adds it if everything is correct。 If not， it downloads data from the query requestor replica， or from another replica where the data has already been added。

So you can put data in the 'detached' directory on one replica， and use the ALTER 。。。 ATTACH query to add it to the table on all replicas。

```sql
ALTER TABLE [db.]table FREEZE PARTITION 'name'
```

Creates a local backup of one or multiple partitions。 The name can be the full name of the partition (for example， 201403)， or its prefix (for example， 2014): then the backup will be created for all the corresponding partitions。

The query does the following: for a data snapshot at the time of execution， it creates hardlinks to table data in the directory `/var/lib/clickhouse/shadow/N/。。。`

`/var/lib/clickhouse/` is the working ClickHouse directory from the config。
`N` is the incremental number of the backup。

The same structure of directories is created inside the backup as inside `/var/lib/clickhouse/`。
It also performs 'chmod' for all files， forbidding writes to them。

The backup is created almost instantly (but first it waits for current queries to the corresponding table to finish running)。 At first， the backup doesn't take any space on the disk。 As the system works， the backup can take disk space， as data is modified。 If the backup is made for old enough data， it won't take space on the disk。

After creating the backup， data from `/var/lib/clickhouse/shadow/` can be copied to the remote server and then deleted on the local server。
The entire backup process is performed without stopping the server。

The `ALTER ... FREEZE PARTITION` query is not replicated。 A local backup is only created on the local server。

As an alternative， you can manually copy data from the `/var/lib/clickhouse/data/database/table` directory。
But if you do this while the server is running， race conditions are possible when copying directories with files being added or changed， and the backup may be inconsistent。 You can do this if the server isn't running – then the resulting data will be the same as after the `ALTER TABLE t FREEZE PARTITION` query。

`ALTER TABLE ... FREEZE PARTITION` only copies data， not table metadata。 To make a backup of table metadata， copy the file  `/var/lib/clickhouse/metadata/database/table。sql`

To restore from a backup:

> - Use the CREATE query to create the table if it doesn't exist。 The query can be taken from an 。sql file (replace `ATTACH` in it with `CREATE`)。
- Copy the data from the data/database/table/ directory inside the backup to the `/var/lib/clickhouse/data/database/table/detached/ directory。`
- Run `ALTER TABLE 。。。 ATTACH PARTITION YYYYMM` queries， where `YYYYMM` is the month， for every month。

In this way， data from the backup will be added to the table。
Restoring from a backup doesn't require stopping the server。

### Backups and replication

Replication provides protection from device failures。 If all data disappeared on one of your replicas， follow the instructions in the "Restoration after failure" section to restore it。

For protection from device failures， you must use replication。 For more information about replication， see the section "Data replication"。

Backups protect against human error (accidentally deleting data， deleting the wrong data or in the wrong cluster， or corrupting data)。
For high-volume databases， it can be difficult to copy backups to remote servers。 In such cases， to protect from human error， you can keep a backup on the same server (it will reside in `/var/lib/clickhouse/shadow/`)。

```sql
ALTER TABLE [db.]table FETCH PARTITION 'name' FROM 'path-in-zookeeper'
```

This query only works for replicatable tables。

It downloads the specified partition from the shard that has its `ZooKeeper path` specified in the `FROM` clause， then puts it in the `detached` directory for the specified table。

Although the query is called `ALTER TABLE`， it does not change the table structure， and does not immediately change the data available in the table。

Data is placed in the `detached` directory。 You can use the `ALTER TABLE 。。。 ATTACH` query to attach the data。

The ` FROM`  clause specifies the path in ` ZooKeeper`。 For example， `/clickhouse/tables/01-01/visits`。
Before downloading， the system checks that the partition exists and the table structure matches。 The most appropriate replica is selected automatically from the healthy replicas。

The `ALTER ... FETCH PARTITION` query is not replicated。 The partition will be downloaded to the 'detached' directory only on the local server。 Note that if after this you use the `ALTER TABLE ... ATTACH` query to add data to the table， the data will be added on all replicas (on one of the replicas it will be added from the 'detached' directory， and on the rest it will be loaded from neighboring replicas)。

### Synchronicity of ALTER queries

For non-replicatable tables， all `ALTER` queries are performed synchronously。 For replicatable tables， the query just adds instructions for the appropriate actions to `ZooKeeper`， and the actions themselves are performed as soon as possible。 However， the query can wait for these actions to be completed on all the replicas。

For `ALTER ... ATTACH|DETACH|DROP` queries， you can use the `replication_alter_partitions_sync` setting to set up waiting。
Possible values: `0` – do not wait; `1` – only wait for own execution (default); `2` – wait for all。

<a name="query_language_queries_show_databases"></a>

## SHOW DATABASES

```sql
SHOW DATABASES [INTO OUTFILE filename] [FORMAT format]
```

Prints a list of all databases。
This query is identical to `SELECT name FROM system。databases [INTO OUTFILE filename] [FORMAT format]`。

See also the section "Formats"。

## SHOW TABLES

```sql
SHOW TABLES [FROM db] [LIKE 'pattern'] [INTO OUTFILE filename] [FORMAT format]
```

Displays a list of tables

- tables from the current database， or from the 'db' database if "FROM db" is specified。
- all tables， or tables whose name matches the pattern， if "LIKE 'pattern'" is specified。

This query is identical to: `SELECT name FROM system。tables WHERE database = 'db' [AND name LIKE 'pattern'] [INTO OUTFILE filename] [FORMAT format]`。

See the section "LIKE operator" also。

## SHOW PROCESSLIST

```sql
SHOW PROCESSLIST [INTO OUTFILE filename] [FORMAT format]
```

Outputs a list of queries currently being processed， other than `SHOW PROCESSLIST` queries。

Prints a table containing the columns:

**user** – The user who made the query。 Keep in mind that for distributed processing， queries are sent to remote servers under the 'default' user。 SHOW PROCESSLIST shows the username for a specific query， not for a query that this query initiated。

**address** – The name of the host that the query was sent from。 For distributed processing， on remote servers， this is the name of the query requestor host。 To track where a distributed query was originally made from， look at SHOW PROCESSLIST on the query requestor server。

**elapsed** – The execution time， in seconds。 Queries are output in order of decreasing execution time。

**rows_read**， **bytes_read** – How many rows and bytes of uncompressed data were read when processing the query。 For distributed processing， data is totaled from all the remote servers。 This is the data used for restrictions and quotas。

**memory_usage** – Current RAM usage in bytes。 See the setting 'max_memory_usage'。

**query** – The query itself。 In INSERT queries， the data for insertion is not output。

**query_id** - The query identifier。 Non-empty only if it was explicitly defined by the user。 For distributed processing， the query ID is not passed to remote servers。

This query is identical to: `SELECT * FROM system。processes [INTO OUTFILE filename] [FORMAT format]`。

Tip (execute in the console):

```bash
watch -n1 "clickhouse-client --query='SHOW PROCESSLIST'"
```

## SHOW CREATE TABLE

```sql
SHOW CREATE TABLE [db.]table [INTO OUTFILE filename] [FORMAT format]
```

Returns a single `String`-type 'statement' column， which contains a single value – the `CREATE` query used for creating the specified table。

## DESCRIBE TABLE

```sql
DESC|DESCRIBE TABLE [db.]table [INTO OUTFILE filename] [FORMAT format]
```

Returns two `String`-type columns: `name` and `type`， which indicate the names and types of columns in the specified table。

Nested data structures are output in "expanded" format。 Each column is shown separately， with the name after a dot。

## EXISTS

```sql
EXISTS TABLE [db.]name [INTO OUTFILE filename] [FORMAT format]
```

Returns a single `UInt8`-type column， which contains the single value `0` if the table or database doesn't exist， or `1` if the table exists in the specified database。

## USE

```sql
USE db
```

Lets you set the current database for the session。
The current database is used for searching for tables if the database is not explicitly defined in the query with a dot before the table name。
This query can't be made when using the HTTP protocol， since there is no concept of a session。

## SET

```sql
SET param = value
```

Allows you to set `param` to `value`。 You can also make all the settings from the specified settings profile in a single query。 To do this， specify 'profile' as the setting name。 For more information， see the section "Settings"。
The setting is made for the session， or for the server (globally) if `GLOBAL` is specified。
When making a global setting， the setting is not applied to sessions already running， including the current session。 It will only be used for new sessions。

When the server is restarted， global settings made using `SET` are lost。
To make settings that persist after a server restart， you can only use the server's config file。

## OPTIMIZE

```sql
OPTIMIZE TABLE [db.]name [PARTITION partition] [FINAL]
```

Asks the table engine to do something for optimization。
Supported only by `*MergeTree` engines， in which this query initializes a non-scheduled merge of data parts。
If you specify a `PARTITION`， only the specified partition will be optimized。
If you specify `FINAL`， optimization will be performed even when all the data is already in one part。

<a name="queries-insert"></a>

## INSERT

Adding data。

Basic query format:

```sql
INSERT INTO [db.]table [(c1, c2, c3)] VALUES (v11, v12, v13), (v21, v22, v23), ...
```

The query can specify a list of columns to insert `[(c1， c2， c3)]`。 In this case， the rest of the columns are filled with:

- The values calculated from the `DEFAULT`  expressions specified in the table definition。
- Zeros and empty strings， if `DEFAULT` expressions are not defined。

If [strict_insert_defaults=1](../operations/settings/settings.md#settings-strict_insert_defaults), columns that do not have ` DEFAULT` defined must be listed in the query。

The INSERT can pass data in any [format](../formats/index.md#formats) supported by ClickHouse。 The format must be specified explicitly in the query:

```sql
INSERT INTO [db.]table [(c1, c2, c3)] FORMAT format_name data_set
```

For example， the following query format is identical to the basic version of INSERT ... VALUES:

```sql
INSERT INTO [db.]table [(c1, c2, c3)] FORMAT Values (v11, v12, v13), (v21, v22, v23), ...
```

ClickHouse removes all spaces and one line feed (if there is one) before the data。 When forming a query， we recommend putting the data on a new line after the query operators (this is important if the data begins with spaces)。

Example:

```sql
INSERT INTO t FORMAT TabSeparated
11  Hello, world!
22  Qwerty
```

You can insert data separately from the query by using the command-line client or the HTTP interface。 For more information， see the section "[Interfaces](../interfaces/index.md#interfaces)"。

### Inserting the results of `SELECT`

```sql
INSERT INTO [db.]table [(c1, c2, c3)] SELECT ...
```

Columns are mapped according to their position in the SELECT clause。 However， their names in the SELECT expression and the table for INSERT may differ。 If necessary， type casting is performed。

None of the data formats except Values allow setting values to expressions such as `now()`， `1 + 2`，  and so on。 The Values format allows limited use of expressions， but this is not recommended， because in this case inefficient code is used for their execution。

Other queries for modifying data parts are not supported: `UPDATE`, `DELETE`, `REPLACE`, `MERGE`, `UPSERT`, `INSERT UPDATE`。
However， you can delete old data using `ALTER TABLE ... DROP PARTITION`。

### Performance considerations

`INSERT` sorts the input data by primary key and splits them into partitions by month。 If you insert data for mixed months， it can significantly reduce the performance of the `INSERT` query。 To avoid this:

- Add data in fairly large batches， such as 100，000 rows at a time。
- Group data by month before uploading it to ClickHouse。

Performance will not decrease if:

- Data is added in real time。
- You upload data that is usually sorted by time。

## SELECT

Data sampling。

```sql
SELECT [DISTINCT] expr_list
    [FROM [db.]table | (subquery) | table_function] [FINAL]
    [SAMPLE sample_coeff]
    [ARRAY JOIN ...]
    [GLOBAL] ANY|ALL INNER|LEFT JOIN (subquery)|table USING columns_list
    [PREWHERE expr]
    [WHERE expr]
    [GROUP BY expr_list] [WITH TOTALS]
    [HAVING expr]
    [ORDER BY expr_list]
    [LIMIT [n, ]m]
    [UNION ALL ...]
    [INTO OUTFILE filename]
    [FORMAT format]
    [LIMIT n BY columns]
```

All the clauses are optional， except for the required list of expressions immediately after SELECT。
The clauses below are described in almost the same order as in the query execution conveyor。

If the query omits the `DISTINCT`， `GROUP BY` and `ORDER BY` clauses and the `IN` and `JOIN` subqueries， the query will be completely stream processed， using O(1) amount of RAM。
Otherwise， the query might consume a lot of RAM if the appropriate restrictions are not specified: `max_memory_usage`， `max_rows_to_group_by`， `max_rows_to_sort`， `max_rows_in_distinct`， `max_bytes_in_distinct`， `max_rows_in_set`， `max_bytes_in_set`， `max_rows_in_join`， `max_bytes_in_join`， `max_bytes_before_external_sort`， `max_bytes_before_external_group_by`。 For more information， see the section "Settings"。 It is possible to use external sorting (saving temporary tables to a disk) and external aggregation。 `The system does not have "merge join"`。

### FROM clause

If the FROM clause is omitted， data will be read from the `system。one` table。
The 'system。one' table contains exactly one row (this table fulfills the same purpose as the DUAL table found in other DBMSs)。

The FROM clause specifies the table to read data from， or a subquery， or a table function; ARRAY JOIN and the regular JOIN may also be included (see below)。

Instead of a table， the SELECT subquery may be specified in brackets。
In this case， the subquery processing pipeline will be built into the processing pipeline of an external query。
In contrast to standard SQL， a synonym does not need to be specified after a subquery。 For compatibility， it is possible to write 'AS name' after a subquery， but the specified name isn't used anywhere。

A table function may be specified instead of a table。 For more information， see the section "Table functions"。

To execute a query， all the columns listed in the query are extracted from the appropriate table。 Any columns not needed for the external query are thrown out of the subqueries。
If a query does not list any columns (for example， SELECT count() FROM t)， some column is extracted from the table anyway (the smallest one is preferred)， in order to calculate the number of rows。

The FINAL modifier can be used only for a SELECT from a CollapsingMergeTree table。 When you specify FINAL， data is selected fully "collapsed"。 Keep in mind that using FINAL leads to a selection that includes columns related to the primary key， in addition to the columns specified in the SELECT。 Additionally， the query will be executed in a single stream， and data will be merged during query execution。 This means that when using FINAL， the query is processed more slowly。 In most cases， you should avoid using FINAL。 For more information， see the section "CollapsingMergeTree engine"。

### SAMPLE clause

The SAMPLE clause allows for approximated query processing。 Approximated query processing is only supported by MergeTree\* type tables， and only if the sampling expression was specified during table creation (see the section "MergeTree engine")。

`SAMPLE` has the `format SAMPLE k`， where `k` is a decimal number from 0 to 1， or `SAMPLE n`， where 'n' is a sufficiently large integer。

In the first case， the query will be executed on 'k' percent of data。 For example， `SAMPLE 0。1` runs the query on 10% of data。
In the second case， the query will be executed on a sample of no more than 'n' rows。 For example， `SAMPLE 10000000` runs the query on a maximum of 10，000，000 rows。

Example:

```sql
SELECT
    Title,
    count() * 10 AS PageViews
FROM hits_distributed
SAMPLE 0.1
WHERE
    CounterID = 34
    AND toDate(EventDate) >= toDate('2013-01-29')
    AND toDate(EventDate) <= toDate('2013-02-04')
    AND NOT DontCountHits
    AND NOT Refresh
    AND Title != ''
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```

In this example， the query is executed on a sample from 0。1 (10%) of data。 Values of aggregate functions are not corrected automatically， so to get an approximate result， the value 'count()' is manually multiplied by 10。

When using something like `SAMPLE 10000000`， there isn't any information about which relative percent of data was processed or what the aggregate functions should be multiplied by， so this method of writing is not always appropriate to the situation。

A sample with a relative coefficient is "consistent": if we look at all possible data that could be in the table， a sample (when using a single sampling expression specified during table creation) with the same coefficient always selects the same subset of possible data。 In other words， a sample from different tables on different servers at different times is made the same way。

For example， a sample of user IDs takes rows with the same subset of all the possible user IDs from different tables。 This allows using the sample in subqueries in the IN clause， as well as for manually correlating results of different queries with samples。

### ARRAY JOIN clause

Allows executing JOIN with an array or nested data structure。 The intent is similar to the 'arrayJoin' function， but its functionality is broader。

`ARRAY JOIN` is essentially `INNER JOIN` with an array。 Example:

```text
:) CREATE TABLE arrays_test (s String, arr Array(UInt8)) ENGINE = Memory

CREATE TABLE arrays_test
(
    s String,
    arr Array(UInt8)
) ENGINE = Memory

Ok。

0 rows in set。 Elapsed: 0.001 sec。

:) INSERT INTO arrays_test VALUES ('Hello', [1,2]), ('World', [3,4,5]), ('Goodbye', [])

INSERT INTO arrays_test VALUES

Ok.

3 rows in set. Elapsed: 0.001 sec。

:) SELECT * FROM arrays_test

SELECT *
FROM arrays_test

┌─s───────┬─arr─────┐
│ Hello   │ [1，2]   │
│ World   │ [3，4，5] │
│ Goodbye │ []      │
└─────────┴─────────┘

3 rows in set. Elapsed: 0.001 sec.

:) SELECT s, arr FROM arrays_test ARRAY JOIN arr

SELECT s, arr
FROM arrays_test
ARRAY JOIN arr

┌─s─────┬─arr─┐
│ Hello │   1 │
│ Hello │   2 │
│ World │   3 │
│ World │   4 │
│ World │   5 │
└───────┴─────┘

5 rows in set. Elapsed: 0.001 sec.
```

An alias can be specified for an array in the ARRAY JOIN clause。 In this case， an array item can be accessed by this alias， but the array itself by the original name。 Example:

```text
:) SELECT s, arr, a FROM arrays_test ARRAY JOIN arr AS a

SELECT s, arr, a
FROM arrays_test
ARRAY JOIN arr AS a

┌─s─────┬─arr─────┬─a─┐
│ Hello │ [1,2]   │ 1 │
│ Hello │ [1,2]   │ 2 │
│ World │ [3,4,5] │ 3 │
│ World │ [3,4,5] │ 4 │
│ World │ [3,4,5] │ 5 │
└───────┴─────────┴───┘

5 rows in set. Elapsed: 0.001 sec。
```

Multiple arrays of the same size can be comma-separated in the ARRAY JOIN clause。 In this case， JOIN is performed with them simultaneously (the direct sum， not the direct product)。 Example:

```text
:) SELECT s, arr, a, num. mapped FROM arrays_test ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num, arrayMap(x -> x + 1, arr) AS mapped

SELECT s, arr, a, num, mapped
FROM arrays_test
ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num, arrayMap(lambda(tuple(x), plus(x, 1)), arr) AS mapped

┌─s─────┬─arr─────┬─a─┬─num─┬─mapped─┐
│ Hello │ [1,2]   │ 1 │   1 │      2 │
│ Hello │ [1,2]   │ 2 │   2 │      3 │
│ World │ [3,4,5] │ 3 │   1 │      4 │
│ World │ [3,4,5] │ 4 │   2 │      5 │
│ World │ [3,4,5] │ 5 │   3 │      6 │
└───────┴─────────┴───┴─────┴────────┘

5 rows in set. Elapsed: 0.002 sec.

:) SELECT s,arr, a, num, arrayEnumerate(arr) FROM arrays_test ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num

SELECT s, arr, a, num, arrayEnumerate(arr)
FROM arrays_test
ARRAY JOIN arr AS a, arrayEnumerate(arr) AS num

┌─s─────┬─arr─────┬─a─┬─num─┬─arrayEnumerate(arr)─┐
│ Hello │ [1,2]   │ 1 │   1 │ [1,2]               │
│ Hello │ [1,2]   │ 2 │   2 │ [1,2]               │
│ World │ [3,4,5] │ 3 │   1 │ [1,2,3]             │
│ World │ [3,4,5] │ 4 │   2 │ [1,2,3]             │
│ World │ [3,4,5] │ 5 │   3 │ [1,2,3]             │
└───────┴─────────┴───┴─────┴─────────────────────┘

5 rows in set. Elapsed: 0.002 sec.
```

ARRAY JOIN also works with nested data structures. Example:

```text
:) CREATE TABLE nested_test (s String, nest Nested(x UInt8, y UInt32)) ENGINE = Memory

CREATE TABLE nested_test
(
    s String,
    nest Nested(
    x UInt8,
    y UInt32)
) ENGINE = Memory

Ok.

0 rows in set. Elapsed: 0.006 sec.

:) INSERT INTO nested_test VALUES ('Hello', [1,2], [10,20]), ('World', [3,4,5], [30,40,50]), ('Goodbye', [], [])

INSERT INTO nested_test VALUES

Ok.

3 rows in set. Elapsed: 0.001 sec.

:) SELECT * FROM nested_test

SELECT *
FROM nested_test

┌─s───────┬─nest.x──┬─nest.y─────┐
│ Hello   │ [1,2]   │ [10,20]    │
│ World   │ [3,4,5] │ [30,40,50] │
│ Goodbye │ []      │ []         │
└─────────┴─────────┴────────────┘

3 rows in set. Elapsed: 0.001 sec.

:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN nest

┌─s─────┬─nest.x─┬─nest.y─┐
│ Hello │      1 │     10 │
│ Hello │      2 │     20 │
│ World │      3 │     30 │
│ World │      4 │     40 │
│ World │      5 │     50 │
└───────┴────────┴────────┘

5 rows in set. Elapsed: 0.001 sec.
```

When specifying names of nested data structures in ARRAY JOIN, the meaning is the same as ARRAY JOIN with all the array elements that it consists of. Example:

```text
:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest.x, nest.y

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN `nest.x`, `nest.y`

┌─s─────┬─nest.x─┬─nest.y─┐
│ Hello │      1 │     10 │
│ Hello │      2 │     20 │
│ World │      3 │     30 │
│ World │      4 │     40 │
│ World │      5 │     50 │
└───────┴────────┴────────┘

5 rows in set. Elapsed: 0.001 sec.
```

This variation also makes sense:

```text
:) SELECT s, nest.x, nest.y FROM nested_test ARRAY JOIN nest.x

SELECT s, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN `nest.x`

┌─s─────┬─nest.x─┬─nest.y─────┐
│ Hello │      1 │ [10,20]    │
│ Hello │      2 │ [10,20]    │
│ World │      3 │ [30,40,50] │
│ World │      4 │ [30,40,50] │
│ World │      5 │ [30,40,50] │
└───────┴────────┴────────────┘

5 rows in set. Elapsed: 0.001 sec.
```

An alias may be used for a nested data structure, in order to select either the JOIN result or the source array. Example:

```text
:) SELECT s, n.x, n.y, nest.x, nest.y FROM nested_test ARRAY JOIN nest AS n

SELECT s, `n.x`, `n.y`, `nest.x`, `nest.y`
FROM nested_test
ARRAY JOIN nest AS n

┌─s─────┬─n.x─┬─n.y─┬─nest.x──┬─nest.y─────┐
│ Hello │   1 │  10 │ [1,2]   │ [10,20]    │
│ Hello │   2 │  20 │ [1,2]   │ [10,20]    │
│ World │   3 │  30 │ [3,4,5] │ [30,40,50] │
│ World │   4 │  40 │ [3,4,5] │ [30,40,50] │
│ World │   5 │  50 │ [3,4,5] │ [30,40,50] │
└───────┴─────┴─────┴─────────┴────────────┘

5 rows in set. Elapsed: 0.001 sec.
```

Example of using the arrayEnumerate function:

```text
:) SELECT s, n.x, n.y, nest.x, nest.y, num FROM nested_test ARRAY JOIN nest AS n, arrayEnumerate(nest.x) AS num

SELECT s, `n.x`, `n.y`, `nest.x`, `nest.y`, num
FROM nested_test
ARRAY JOIN nest AS n, arrayEnumerate(`nest.x`) AS num

┌─s─────┬─n.x─┬─n.y─┬─nest.x──┬─nest.y─────┬─num─┐
│ Hello │   1 │  10 │ [1,2]   │ [10,20]    │   1 │
│ Hello │   2 │  20 │ [1,2]   │ [10,20]    │   2 │
│ World │   3 │  30 │ [3,4,5] │ [30,40,50] │   1 │
│ World │   4 │  40 │ [3,4,5] │ [30,40,50] │   2 │
│ World │   5 │  50 │ [3,4,5] │ [30,40,50] │   3 │
└───────┴─────┴─────┴─────────┴────────────┴─────┘

5 rows in set. Elapsed: 0.002 sec.
```

The query can only specify a single ARRAY JOIN clause.

The corresponding conversion can be performed before the WHERE/PREWHERE clause (if its result is needed in this clause), or after completing WHERE/PREWHERE (to reduce the volume of calculations).

### JOIN 语句

JOIN标准语句， 与ARRAY JOIN 关系不大。



[GLOBAL] ANY|ALL INNER|LEFT [OUTER] JOIN (subquery)|table USING columns_list
在子查询中执行JOIN查询。 在查询处理开始时， 子查询在 JOIN 以后指定， 他的结果保存在内存中。 然后从指定在FROM语句中的"左关联" 表中读取， 在读取过程中， 对于从"左关联" 表中读取的每行记录， 和从子查询结果("右关联")表中查询的数据满足USING中的匹配条件。



表名能够被指定来替代子查询。 这等于SELECT * FROM table subquery， 在特定情况下，当表有Join 引擎时 – 一个数组准备 join。



在JOIN中所有的列如果不需要，则从子查询中删除。


有如下几种类型的JOIN:


INNER 或 LEFT 类型: 如果 INNER 被指定，结果仅包含这些行，在右表中匹配的行。 如果 LEFT 被指定，任何在左表中的行没有匹配右表中的行都将被分配默认值 - 0或空行。 LEFT OUTER 可以被写来替代LEFT; OUTER 不影响任何事情。



ANY 或 ALL 字符串: 如果ANY 被指定和右表有一些匹配的行，只有第一行发现被Join，。 如果 ALL 被指定，右表有一些匹配的行，数据将按照行数相乘。



使用 ALL 对应的 JOIN 语义。 使用ANY 是最优的。 如果右表仅有一个匹配行， ANY 和 ALL 的结果是相同的。 你必须指定 ANY 或 ALL (默认情况下，2个都不选择)。



GLOBAL 分布:



当使用 JOIN语句， 查询将被发送至远程服务器。 子查询将被运行在每个节点上，为了让右表和JOIN查询运行在此表上。 换句话说， 右表单独运行在每个服务器上。



当使用 GLOBAL ... JOIN， 首先 请求服务器运行子查询来计算右表。 此临时表被传递到每个远程服务器， 查询运行在临时表上。



当使用 GLOBAL JOINs时需要小心。 更多信息， 请查看章节 "分布式子查询"。



任意的JOINs都是有可能的。 例如， GLOBAL ANY LEFT OUTER JOIN。



当运行一个 JOIN时， 执行顺序没有优化，和其他的查询阶段相比。 Join在WHERE过滤之前和聚合之前被执行。 为了设置处理顺序， 我们推荐在一个子查询里运行一个JOIN子查询。



示例:

SELECT
    CounterID,
    hits,
    visits
FROM
(
    SELECT
        CounterID,
        count() AS hits
    FROM test.hits
    GROUP BY CounterID
) ANY LEFT JOIN
(
    SELECT
        CounterID,
        sum(Sign) AS visits
    FROM test.visits
    GROUP BY CounterID
) USING CounterID
ORDER BY hits DESC
LIMIT 10
┌─CounterID─┬───hits─┬─visits─┐
│   1143050 │ 523264 │  13665 │
│    731962 │ 475698 │ 102716 │
│    722545 │ 337212 │ 108187 │
│    722889 │ 252197 │  10547 │
│   2237260 │ 196036 │   9522 │
│  23057320 │ 147211 │   7689 │
│    722818 │  90109 │  17847 │
│     48221 │  85379 │   4652 │
│  19762435 │  77807 │   7026 │
│    722884 │  77492 │  11056 │
└───────────┴────────┴────────┘

子查询不允许你设置名称或使用他们，对于从一个特定的查询中引用一个列。 指定在 USING 中的列必须在两个子查询中都有相同的名字， 其他的列必须单独命名。 在子查询中你可以使用别名来更改列的名称，和在子查询中的列 (例如别名使用 'hits' 和 'visits')。



USING 语句指定一个或多个列进行Join， 建立这些列的等值列。 列的列表被设置不需要brackets。 因此不支持复杂JOIN条件。



右表 (子查询结果) 驻留在内存中。 If 如果没有足够的内存，也不能运行 JOIN。



仅有一个JOIN 能够被指定在一个查询中。 为了运行多个 JOINs， 你能够放它们在子查询中。



每次一个查询运行相同的 JOIN， 子查询再次运行 – 结果不被缓存。 为了避免这个， 使用特定的'Join' 表引擎， 它是一个预处理数组，在内存中进行join操作。 对于更多信息， 请查看章节 "表引擎， Join"。



在一些情况下，使用 IN 语句比 JOIN 效率更高。在不同类型的JOIN查询下，最高效的是ANY LEFT JOIN，其次是ANY INNER JOIN，最后是ALL LEFT JOIN 和 ALL INNER JOIN。

如果你想要一个JOIN 来关联维度表 (这些是一些小表，包含维度属性， 如营销活动的名称)， 一个 JOIN 可能并不是特别合适，由于 bulky 的语句，右表对于每个查询重新访问。 在这些场景下， 有一个 "外部字典" 特性，你应该使用它来替换 JOIN。 更多信息， 请查看章节 "外部字典"。



### WHERE 语句

如果有一个 WHERE 语句， 它必须包含一个带有UInt8类型的表达式。 通常情况下是带有比较和逻辑操作符的表达式。
此表达式将被用于在所有其他转换之前过滤数据。

如果数据库表引擎支持索引， 表达式使用索引来评估。

### PREWHERE 语句

此语句与WHERE语句效果相同。 主要区别是数据从表中读取。
当使用 PREWHERE 时， 首先， 只有执行PREWHERE语句的列被读取。 然后， 其他列在运行query查询时被读取， 只读取当PREWHERE表达式为 ture时这些数据块。

如果有过滤条件， 不适合索引， 使用PREWHERE是有意义的， 在查询中被用于少数列， 但是提供了一个很强有力的数据过滤。 这减少了数据量的读取。

例如， 使用PREWHERE进行抽取大量的列是有非常有用的， 但是仅为一些列进行过滤。

PREWHERE 仅被`*MergeTree`表引擎家族支持。

一个查询可能同时指定 PREWHERE 和 WHERE。 在这种情况下， PREWHERE 优先于 WHERE。

切记， 对于PREWHERE来说， 仅指定有索引的列并没有太大意义， 因为当你使用一个索引， 仅有匹配索引的数据块被读取。

如果 'optimize_move_to_prewhere' 设置被设为1， 同时 PREWHERE 被忽略， 此系统使用启发式自动从WHERE到PREWHERE 移动表达式的部分。

### GROUP BY 语句

这是列式DBMS最重要的部分。

如果有一个 GROUP BY 语句， 它必包含一个表达式列表。 每个表达式将被引用作为一个"key"。
在SELECT， HAVING， 和 ORDER BY 语句中的所有的表达式必须从 keys 中或aggregate函数中被计算。 换句话说， 从表中查询出来的列必须被用在 keys 或在 aggregate 函数中。

如果一个查询仅包含表的列在aggregate函数中， GROUP BY 语句能够忽略， 同时假设通过一个keys的空集合进行聚合。

示例:


```sql
SELECT
    count(),
    median(FetchTiming > 60 ? 60 : FetchTiming),
    count() - sum(Refresh)
FROM hits
```

However， in contrast to standard SQL， if the table doesn't have any rows (either there aren't any at all， or there aren't any after using WHERE to filter)， an empty result is returned， and not the result from one of the rows containing the initial values of aggregate functions。

As opposed to MySQL (and conforming to standard SQL)， you can't get some value of some column that is not in a key or aggregate function (except constant expressions)。 To work around this， you can use the 'any' aggregate function (get the first encountered value) or 'min/max'。

Example:

```sql
SELECT
    domainWithoutWWW(URL) AS domain,
    count(),
    any(Title) AS title -- getting the first occurred page header for each domain.
FROM hits
GROUP BY domain
```

For every different key value encountered， GROUP BY calculates a set of aggregate function values。

GROUP BY is not supported for array columns。

A constant can't be specified as arguments for aggregate functions。 Example: sum(1)。 Instead of this， you can get rid of the constant。 Example: `count()`。

#### WITH TOTALS modifier

If the WITH TOTALS modifier is specified， another row will be calculated。 This row will have key columns containing default values (zeros or empty lines)， and columns of aggregate functions with the values calculated across all the rows (the "total" values)。

This extra row is output in JSON\*， TabSeparated\*， and Pretty\* formats， separately from the other rows。 In the other formats， this row is not output。

In JSON\* formats， this row is output as a separate 'totals' field。 In TabSeparated\* formats， the row comes after the main result， preceded by an empty row (after the other data)。 In Pretty\* formats， the row is output as a separate table after the main result。

`WITH TOTALS` can be run in different ways when HAVING is present。 The behavior depends on the 'totals_mode' setting。
By default， `totals_mode = 'before_having'`。 In this case， 'totals' is calculated across all rows， including the ones that don't pass through HAVING and 'max_rows_to_group_by'。

The other alternatives include only the rows that pass through HAVING in 'totals'， and behave differently with the setting `max_rows_to_group_by` and `group_by_overflow_mode = 'any'`。

`after_having_exclusive` – Don't include rows that didn't pass through `max_rows_to_group_by`。 In other words， 'totals' will have less than or the same number of rows as it would if `max_rows_to_group_by` were omitted。

`after_having_inclusive` – Include all the rows that didn't pass through 'max_rows_to_group_by' in 'totals'。 In other words， 'totals' will have more than or the same number of rows as it would if `max_rows_to_group_by` were omitted。

`after_having_auto` – Count the number of rows that passed through HAVING。 If it is more than a certain amount (by default， 50%)， include all the rows that didn't pass through 'max_rows_to_group_by' in 'totals'。 Otherwise， do not include them。

`totals_auto_threshold` – By default， 0。5。 The coefficient for `after_having_auto`。

If `max_rows_to_group_by` and `group_by_overflow_mode = 'any'` are not used， all variations of `after_having` are the same， and you can use any of them (for example， `after_having_auto`)。

You can use WITH TOTALS in subqueries， including subqueries in the JOIN clause (in this case， the respective total values are combined)。

#### GROUP BY in external memory

You can enable dumping temporary data to the disk to restrict memory usage during GROUP BY。
The `max_bytes_before_external_group_by` setting determines the threshold RAM consumption for dumping GROUP BY temporary data to the file system。 If set to 0 (the default)， it is disabled。

When using `max_bytes_before_external_group_by`， we recommend that you set max_memory_usage about twice as high。 This is necessary because there are two stages to aggregation: reading the date and forming intermediate data (1) and merging the intermediate data (2)。 Dumping data to the file system can only occur during stage 1。 If the temporary data wasn't dumped， then stage 2 might require up to the same amount of memory as in stage 1。

For example， if `max_memory_usage` was set to 10000000000 and you want to use external aggregation， it makes sense to set `max_bytes_before_external_group_by` to 10000000000， and max_memory_usage to 20000000000。 When external aggregation is triggered (if there was at least one dump of temporary data)， maximum consumption of RAM is only slightly more than ` max_bytes_before_external_group_by`。

With distributed query processing， external aggregation is performed on remote servers。 In order for the requestor server to use only a small amount of RAM， set ` distributed_aggregation_memory_efficient`  to 1。

When merging data flushed to the disk， as well as when merging results from remote servers when the ` distributed_aggregation_memory_efficient` setting is enabled， consumes up to 1/256 \* the number of threads from the total amount of RAM。

When external aggregation is enabled， if there was less than ` max_bytes_before_external_group_by`  of data (i。e。 data was not flushed)， the query runs just as fast as without external aggregation。 If any temporary data was flushed， the run time will be several times longer (approximately three times)。

If you have an ORDER BY with a small LIMIT after GROUP BY， then the ORDER BY CLAUSE will not use significant amounts of RAM。
But if the ORDER BY doesn't have LIMIT， don't forget to enable external sorting (`max_bytes_before_external_sort`)。

### LIMIT N BY clause

LIMIT N BY COLUMNS selects the top N rows for each group of COLUMNS。 LIMIT N BY is not related to LIMIT; they can both be used in the same query。 The key for LIMIT N BY can contain any number of columns or expressions。

Example:

```sql
SELECT
    domainWithoutWWW(URL) AS domain,
    domainWithoutWWW(REFERRER_URL) AS referrer,
    device_type,
    count() cnt
FROM hits
GROUP BY domain, referrer, device_type
ORDER BY cnt DESC
LIMIT 5 BY domain, device_type
LIMIT 100
```

The query will select the top 5 referrers for each `domain， device_type` pair， but not more than 100 rows (`LIMIT n BY + LIMIT`)。

### HAVING clause

Allows filtering the result received after GROUP BY， similar to the WHERE clause。
WHERE and HAVING differ in that WHERE is performed before aggregation (GROUP BY)， while HAVING is performed after it。
If aggregation is not performed， HAVING can't be used。

<a name="query_language-queries-order_by"></a>

### ORDER BY clause

The ORDER BY clause contains a list of expressions， which can each be assigned DESC or ASC (the sorting direction)。 If the direction is not specified， ASC is assumed。 ASC is sorted in ascending order， and DESC in descending order。 The sorting direction applies to a single expression， not to the entire list。 Example: `ORDER BY Visits DESC， SearchPhrase`

For sorting by String values， you can specify collation (comparison)。 Example: `ORDER BY SearchPhrase COLLATE 'tr'` - for sorting by keyword in ascending order， using the Turkish alphabet， case insensitive， assuming that strings are UTF-8 encoded。 COLLATE can be specified or not for each expression in ORDER BY independently。 If ASC or DESC is specified， COLLATE is specified after it。 When using COLLATE， sorting is always case-insensitive。

We only recommend using COLLATE for final sorting of a small number of rows， since sorting with COLLATE is less efficient than normal sorting by bytes。

Rows that have identical values for the list of sorting expressions are output in an arbitrary order， which can also be nondeterministic (different each time)。
If the ORDER BY clause is omitted， the order of the rows is also undefined， and may be nondeterministic as well。

When floating point numbers are sorted， NaNs are separate from the other values。 Regardless of the sorting order， NaNs come at the end。 In other words， for ascending sorting they are placed as if they are larger than all the other numbers， while for descending sorting they are placed as if they are smaller than the rest。

Less RAM is used if a small enough LIMIT is specified in addition to ORDER BY。 Otherwise， the amount of memory spent is proportional to the volume of data for sorting。 For distributed query processing， if GROUP BY is omitted， sorting is partially done on remote servers， and the results are merged on the requestor server。 This means that for distributed sorting， the volume of data to sort can be greater than the amount of memory on a single server。

If there is not enough RAM， it is possible to perform sorting in external memory (creating temporary files on a disk)。 Use the setting `max_bytes_before_external_sort` for this purpose。 If it is set to 0 (the default)， external sorting is disabled。 If it is enabled， when the volume of data to sort reaches the specified number of bytes， the collected data is sorted and dumped into a temporary file。 After all data is read， all the sorted files are merged and the results are output。 Files are written to the /var/lib/clickhouse/tmp/ directory in the config (by default， but you can use the 'tmp_path' parameter to change this setting)。

Running a query may use more memory than 'max_bytes_before_external_sort'。 For this reason， this setting must have a value significantly smaller than 'max_memory_usage'。 As an example， if your server has 128 GB of RAM and you need to run a single query， set 'max_memory_usage' to 100 GB， and 'max_bytes_before_external_sort' to 80 GB。

External sorting works much less effectively than sorting in RAM。

### SELECT clause

The expressions specified in the SELECT clause are analyzed after the calculations for all the clauses listed above are completed。
More specifically， expressions are analyzed that are above the aggregate functions， if there are any aggregate functions。
The aggregate functions and everything below them are calculated during aggregation (GROUP BY)。
These expressions work as if they are applied to separate rows in the result。

### DISTINCT clause

If DISTINCT is specified， only a single row will remain out of all the sets of fully matching rows in the result。
The result will be the same as if GROUP BY were specified across all the fields specified in SELECT without aggregate functions。 But there are several differences from GROUP BY:

- DISTINCT can be applied together with GROUP BY。
- When ORDER BY is omitted and LIMIT is defined， the query stops running immediately after the required number of different rows has been read。
- Data blocks are output as they are processed， without waiting for the entire query to finish running。

DISTINCT is not supported if SELECT has at least one array column。

### LIMIT clause

LIMIT m allows you to select the first 'm' rows from the result。
LIMIT n， m allows you to select the first 'm' rows from the result after skipping the first 'n' rows。

'n' and 'm' must be non-negative integers。

If there isn't an ORDER BY clause that explicitly sorts results， the result may be arbitrary and nondeterministic。

### UNION ALL clause

You can use UNION ALL to combine any number of queries。 Example:

```sql
SELECT CounterID, 1 AS table, toInt64(count()) AS c
    FROM test.hits
    GROUP BY CounterID

UNION ALL

SELECT CounterID, 2 AS table, sum(Sign) AS c
    FROM test.visits
    GROUP BY CounterID
    HAVING c > 0
```

Only UNION ALL is supported。 The regular UNION (UNION DISTINCT) is not supported。 If you need UNION DISTINCT， you can write SELECT DISTINCT from a subquery containing UNION ALL。

Queries that are parts of UNION ALL can be run simultaneously， and their results can be mixed together。

The structure of results (the number and type of columns) must match for the queries。 But the column names can differ。 In this case， the column names for the final result will be taken from the first query。

Queries that are parts of UNION ALL can't be enclosed in brackets。 ORDER BY and LIMIT are applied to separate queries， not to the final result。 If you need to apply a conversion to the final result， you can put all the queries with UNION ALL in a subquery in the FROM clause。

### INTO OUTFILE clause

Add the `INTO OUTFILE filename` clause (where filename is a string literal) to redirect query output to the specified file。
In contrast to MySQL， the file is created on the client side。 The query will fail if a file with the same filename already exists。
This functionality is available in the command-line client and clickhouse-local (a query sent via HTTP interface will fail)。

The default output format is TabSeparated (the same as in the command-line client batch mode)。

### FORMAT clause

Specify 'FORMAT format' to get data in any specified format。
You can use this for convenience， or for creating dumps。
For more information， see the section "Formats"。
If the FORMAT clause is omitted， the default format is used， which depends on both the settings and the interface used for accessing the DB。 For the HTTP interface and the command-line client in batch mode， the default format is TabSeparated。 For the command-line client in interactive mode， the default format is PrettyCompact (it has attractive and compact tables)。

When using the command-line client， data is passed to the client in an internal efficient format。 The client independently interprets the FORMAT clause of the query and formats the data itself (thus relieving the network and the server from the load)。

### IN operators

The `IN`， `NOT IN`， `GLOBAL IN`， and `GLOBAL NOT IN` operators are covered separately， since their functionality is quite rich。

The left side of the operator is either a single column or a tuple。

Examples:

```sql
SELECT UserID IN (123, 456) FROM ...
SELECT (CounterID, UserID) IN ((34, 123), (101500, 456)) FROM ...
```

If the left side is a single column that is in the index， and the right side is a set of constants， the system uses the index for processing the query。

Don't list too many values explicitly (i。e。 millions)。 If a data set is large， put it in a temporary table (for example， see the section "External data for query processing")， then use a subquery。

The right side of the operator can be a set of constant expressions， a set of tuples with constant expressions (shown in the examples above)， or the name of a database table or SELECT subquery in brackets。

If the right side of the operator is the name of a table (for example， `UserID IN users`)， this is equivalent to the subquery `UserID IN (SELECT * FROM users)`。 Use this when working with external data that is sent along with the query。 For example， the query can be sent together with a set of user IDs loaded to the 'users' temporary table， which should be filtered。

If the right side of the operator is a table name that has the Set engine (a prepared data set that is always in RAM)， the data set will not be created over again for each query。

The subquery may specify more than one column for filtering tuples。
Example:

```sql
SELECT (CounterID, UserID) IN (SELECT CounterID, UserID FROM ...) FROM ...
```

The columns to the left and right of the IN operator should have the same type。

The IN operator and subquery may occur in any part of the query， including in aggregate functions and lambda functions。
Example:

```sql
SELECT
    EventDate,
    avg(UserID IN
    (
        SELECT UserID
        FROM test.hits
        WHERE EventDate = toDate('2014-03-17')
    )) AS ratio
FROM test.hits
GROUP BY EventDate
ORDER BY EventDate ASC
```

```text
┌──EventDate─┬────ratio─┐
│ 2014-03-17 │        1 │
│ 2014-03-18 │ 0.807696 │
│ 2014-03-19 │ 0.755406 │
│ 2014-03-20 │ 0.723218 │
│ 2014-03-21 │ 0.697021 │
│ 2014-03-22 │ 0.647851 │
│ 2014-03-23 │ 0.648416 │
└────────────┴──────────┘
```

For each day after March 17th， count the percentage of pageviews made by users who visited the site on March 17th。
A subquery in the IN clause is always run just one time on a single server。 There are no dependent subqueries。

<a name="queries-distributed-subrequests"></a>

#### Distributed subqueries

There are two options for IN-s with subqueries (similar to JOINs): normal `IN`  / ` OIN`  and `IN GLOBAL`  / `GLOBAL JOIN`。 They differ in how they are run for distributed query processing。

<div class="admonition attention">

Remember that the algorithms described below may work differently depending on the [].../operations/settings/settings.md#settings-distributed_product_mode) `distributed_product_mode` setting。

</div>

When using the regular IN， the query is sent to remote servers， and each of them runs the subqueries in the `IN` or `JOIN` clause。

When using `GLOBAL IN`  / `GLOBAL JOINs`， first all the subqueries are run for `GLOBAL IN`  / `GLOBAL JOINs`， and the results are collected in temporary tables。 Then the temporary tables are sent to each remote server， where the queries are run using this temporary data。

For a non-distributed query， use the regular `IN` / `JOIN`。

Be careful when using subqueries in the  `IN` / `JOIN` clauses for distributed query processing。

Let's look at some examples。 Assume that each server in the cluster has a normal **local_table**。 Each server also has a **distributed_table** table with the **Distributed** type， which looks at all the servers in the cluster。

For a query to the **distributed_table**， the query will be sent to all the remote servers and run on them using the **local_table**。

For example， the query

```sql
SELECT uniq(UserID) FROM distributed_table
```

will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table
```

and run on each of them in parallel， until it reaches the stage where intermediate results can be combined。 Then the intermediate results will be returned to the requestor server and merged on it， and the final result will be sent to the client。

Now let's examine a query with IN:

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

- Calculation of the intersection of audiences of two sites。

This query will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM local_table WHERE CounterID = 34)
```

In other words， the data set in the IN clause will be collected on each server independently， only across the data that is stored locally on each of the servers。

This will work correctly and optimally if you are prepared for this case and have spread data across the cluster servers such that the data for a single UserID resides entirely on a single server。 In this case， all the necessary data will be available locally on each server。 Otherwise， the result will be inaccurate。 We refer to this variation of the query as "local IN"。

To correct how the query works when data is spread randomly across the cluster servers， you could specify **distributed_table** inside a subquery。 The query would look like this:

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

This query will be sent to all remote servers as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

The subquery will begin running on each remote server。 Since the subquery uses a distributed table， the subquery that is on each remote server will be resent to every remote server as

```sql
SELECT UserID FROM local_table WHERE CounterID = 34
```

For example， if you have a cluster of 100 servers， executing the entire query will require 10，000 elementary requests， which is generally considered unacceptable。

In such cases， you should always use GLOBAL IN instead of IN。 Let's look at how it works for the query

```sql
SELECT uniq(UserID) FROM distributed_table WHERE CounterID = 101500 AND UserID GLOBAL IN (SELECT UserID FROM distributed_table WHERE CounterID = 34)
```

The requestor server will run the subquery

```sql
SELECT UserID FROM distributed_table WHERE CounterID = 34
```

and the result will be put in a temporary table in RAM。 Then the request will be sent to each remote server as

```sql
SELECT uniq(UserID) FROM local_table WHERE CounterID = 101500 AND UserID GLOBAL IN _data1
```

and the temporary table '_data1' will be sent to every remote server together with the query (the name of the temporary table is implementation-defined)。

This is more optimal than using the normal IN。 However， keep the following points in mind:

1。 When creating a temporary table， data is not made unique。 To reduce the volume of data transmitted over the network， specify DISTINCT in the subquery。 (You don't need to do this for a normal IN。)
2。 The temporary table will be sent to all the remote servers。 Transmission does not account for network topology。 For example， if 10 remote servers reside in a datacenter that is very remote in relation to the requestor server， the data will be sent 10 times over the channel to the remote datacenter。 Try to avoid large data sets when using GLOBAL IN。
3。 When transmitting data to remote servers， restrictions on network bandwidth are not configurable。 You might overload the network。
4。 Try to distribute data across servers so that you don't need to use GLOBAL IN on a regular basis。
5。 If you need to use GLOBAL IN often， plan the location of the ClickHouse cluster so that a single group of replicas resides in no more than one data center with a fast network between them， so that a query can be processed entirely within a single data center。

It also makes sense to specify a local table in the `GLOBAL IN` clause， in case this local table is only available on the requestor server and you want to use data from it on remote servers。

### Extreme values

In addition to results， you can also get minimum and maximum values for the results columns。 To do this， set the **extremes** setting to 1。 Minimums and maximums are calculated for numeric types， dates， and dates with times。 For other columns， the default values are output。

An extra two rows are calculated – the minimums and maximums， respectively。 These extra two rows are output in JSON\*， TabSeparated\*， and Pretty\* formats， separate from the other rows。 They are not output for other formats。

In JSON\* formats， the extreme values are output in a separate 'extremes' field。 In TabSeparated\* formats， the row comes after the main result， and after 'totals' if present。 It is preceded by an empty row (after the other data)。 In Pretty\* formats， the row is output as a separate table after the main result， and after 'totals' if present。

Extreme values are calculated for rows that have passed through LIMIT。 However， when using 'LIMIT offset， size'， the rows before 'offset' are included in 'extremes'。 In stream requests， the result may also include a small number of rows that passed through LIMIT。

### Notes

The `GROUP BY` and `ORDER BY` clauses do not support positional arguments。 This contradicts MySQL， but conforms to standard SQL。
For example， `GROUP BY 1， 2` will be interpreted as grouping by constants (i。e。 aggregation of all rows into one)。

You can use synonyms (`AS` aliases) in any part of a query。

You can put an asterisk in any part of a query instead of an expression。 When the query is analyzed， the asterisk is expanded to a list of all table columns (excluding the `MATERIALIZED` and `ALIAS` columns)。 There are only a few cases when using an asterisk is justified:

- When creating a table dump。
- For tables containing just a few columns， such as system tables。
- For getting information about what columns are in a table。 In this case， set `LIMIT 1`。 But it is better to use the `DESC TABLE` query。
- When there is strong filtration on a small number of columns using `PREWHERE`。
- In subqueries (since columns that aren't needed for the external query are excluded from subqueries)。

In all other cases， we don't recommend using the asterisk， since it only gives you the drawbacks of a columnar DBMS instead of the advantages。 In other words using the asterisk is not recommended。

## KILL QUERY

```sql
KILL QUERY WHERE <where expression to SELECT FROM system。processes query> [SYNC|ASYNC|TEST] [FORMAT format]
```

Attempts to terminate queries currently running。
The queries to terminate are selected from the system。processes table for which expression_for_system。processes is true。

Examples:

```sql
KILL QUERY WHERE query_id='2-857d-4a57-9ee0-327da5d60a90'
```

Terminates all queries with the specified query_id。

```sql
KILL QUERY WHERE user='username' SYNC
```

Synchronously terminates all queries run by `username`。

Readonly-users can only terminate their own requests。
By default， the asynchronous version of queries is used (`ASYNC`)， which terminates without waiting for queries to complete。
The synchronous version (`SYNC`) waits for all queries to be completed and displays information about each process as it terminates。
The response contains the `kill_status` column， which can take the following values:

1。 'finished' – The query completed successfully。
2。 'waiting' – Waiting for the query to finish after sending it a signal to terminate。
3。 The other values ​​explain why the query can't be terminated。

A test query (`TEST`) only checks the user's rights and displays a list of queries to terminate。

