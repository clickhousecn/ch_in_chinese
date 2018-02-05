<a name="table_engines-mergetree"></a>

# MergeTree

MergeTree是ClickHouse特有的一种数据表引擎。

MergeTree 允许您依据主键和日期创建索引，并进行实时的数据更新操作。

MergeTree 是 ClickHouse 里最为先进的表引擎。

请注意不要将 MergeTree 跟 Merge 引擎混淆。

MergeTree 引擎在创建时接收以下4个参数，

一个日期字段的名称 （索引字段）
一个统一化表达式 【可选的】
一个含有主键相关字段的元组
以及，稀疏索引之粒度。

就像这样：

不使用统一化表达式的例子

```sql
MergeTree(EventDate, (CounterID, EventDate), 8192)
```

使用统一化表达式的例子

```sql
MergeTree(EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID)), 8192)
```

任意一个MergeTree引擎的数据表必须含有一个独立的日期型字段（而非日期时间型字段）。

主键可以是任意表达式构成的元组（通常是列名称的元组），或者是单独一个字段。

MergeTree 允许使用任意的表达式作为 【可选的】 统一化表达式 。 这个表达式必须在主键（元组）中；上面的例子使用了用户ID的哈希作为统一化表达式，旨在近乎随机地在 CounterID 和指定日期范围内打乱数据条目。换而言之，当我们在查询中使用 SAMPLE 子句时，我们就可以得到一个近乎随机分布的用户列表。

在实际工作时， MergeTree 将数据分割为小的切片作为单位进行处理。 每个切片之间依照主键排序。每个切片记录了指定的开始日期和结束日期。在您插入数据时，MergeTree 就会对数据进行排序处理，以保证存储在切片内的数据有序。 

切片之间的合并过程会在系统后台定期自动执行。MergeTree 引擎会选择几个相邻的切片进行合并（通常是较小的切片）， 然后对二者合并、排序。

具体而言, 向 MergeTree 表中插入数据时，引擎会首先对新数据执行递增排序而保存切片；其后，数据切片之间又会进一步合并，以减少总体切片数量。 因此，合并过程本身并无过多排序工作。

向 MergeTree 插入数据时，不同月份的数据会被自动分散在不同切片中。不同月份的切片不会被合并。这样一来，修改小部分数据将会更加轻松。

切片合并时设有体积上限，以避免切片合并产生庞大的新切片。

除了保存切片中的数据外, 引擎会额外保存一个索引文件。The index file contains the primary key value for every 'index_granularity' row in the table. In other words, this is an abbreviated index of sorted data.

For columns, "marks" are also written to each 'index_granularity' row so that data can be read in a specific range.

When reading from a table, the SELECT query is analyzed for whether indexes can be used.
An index can be used if the WHERE or PREWHERE clause has an expression (as one of the conjunction elements, or entirely) that represents an equality or inequality comparison operation, or if it has IN above columns that are in the primary key or date, or Boolean operators over them.

Thus, it is possible to quickly run queries on one or many ranges of the primary key. In the example given, queries will work quickly for a specific counter, for a specific counter and range of dates, for a specific counter and date, for multiple counters and a range of dates, and so on.

```sql
SELECT count() FROM table WHERE EventDate = toDate(now()) AND CounterID = 34
SELECT count() FROM table WHERE EventDate = toDate(now()) AND (CounterID = 34 OR CounterID = 42)
SELECT count() FROM table WHERE ((EventDate >= toDate('2014-01-01') AND EventDate <= toDate('2014-01-31')) OR EventDate = toDate('2014-05-01')) AND CounterID IN (101500, 731962, 160656) AND (CounterID = 101500 OR EventDate != toDate('2014-05-01'))
```

All of these cases will use the index by date and by primary key. The index is used even for complex expressions. Reading from the table is organized so that using the index can't be slower than a full scan.

In this example, the index can't be used:

```sql
SELECT count() FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'
```

To check whether ClickHouse can use the index when executing the query, use the settings [ force_index_by_date](../operations/settings/settings.md#settings-settings-force_index_by_date)  and [ force_primary_key](../operations/settings/settings.md#settings-settings-force_primary_key).

The index by date only allows reading those parts that contain dates from the desired range. However, a data part may contain data for many dates (up to an entire month), while within a single part the data is ordered by the primary key, which might not contain the date as the first column. Because of this, using a query with only a date condition that does not specify the primary key prefix will cause more data to be read than for a single date.

For concurrent table access, we use multi-versioning. In other words, when a table is simultaneously read and updated, data is read from a set of parts that is current at the time of the query. There are no lengthy locks. Inserts do not get in the way of read operations.

Reading from a table is automatically parallelized.

The `OPTIMIZE` query is supported, which calls an extra merge step.

You can use a single large table and continually add data to it in small chunks – this is what MergeTree is intended for.

Data replication is possible for all types of tables in the MergeTree family (see the section "Data replication").


# MergeTree

举例

不包含示例的mergeTree：

```text
    MergeTree(EventDate,  (CounterID,EventDate),  8192)  
```
    其中， EventDate 是一个日期字段， CounterID是一个UInt64类型的字段

包含示例的mergeTree：

```text
MergeTree(  EventDate,  intHash32  (UserId),  CounterID,   EventDate,  intHash32(UserID)),  8192)  
```

  一个MergeTree类型的表必须有一个包含Date类型的列，在上面的例子里，该列是EventDate，这个日期列的类型必须是'Date'(而非‘DateTime’)

  其中，主键为一个元组，元组中可以包含字段的组合  或  一条表达式。

  这个可选的参数 示例 可以是任何的表达式，但是这个表达式必须出现在主键里。

  上面的示例里使用的是一个哈希类型的userID，来伪随机的对主键里的CounterID和EventDate进行打散。换句话说，当使用了这个示例列的时候，可以伪随机的将用户打散成为均匀的子集。



合并过程
 一张MergeTree表由很多个Part (单列的数据切片) 构成。每一个part按照内部对数据按照主键进行了排序。除此之外，每一个Part含有一个最小日期和最大日期。当插入数据的时候，会将插入的数据创建在一个新的Part 之中。

    同时会在后台周期性的进行merge的过程，当merge的时候，很多个part会被选中，通常是最小的一些part，然后merge成为一个大的排好序的part。

  换句话说，整个这个合并排序的过程是在数据插入表的时候进行的。这个merge会导致这个表总是由少量的排序好的part构成，而且这个merge本身并没有做特别多的工作。

  在插入数据的过程中，属于不同的month的数据会被分割成不同的part，这些归属于不同的month的part是永远不会merge到一起的。这么做的目的是provide local data modification(比较容易做备份)。

  这些part在进行合并的时候会有一个大小的阈值，所以不会有太长的merge过程。

  对于每一个part，会生成一个索引文件。这个索引文件存储了表里面每一个索引块里数据的主键的value值，换句话说，这是个对有序数据的小型索引。

  对列来说，在每一个索引块里的数据也写入了标记，从而让数据可以在明确的数值范围内被查找到。

  当读表里的数据时，SELECT查询会被转化为要使用哪些索引。这些索引会被用在判断where条件或者prewhere条件中，来判断是否打中了这些索引区间。

  因此，能够快速查询一个或多个主键范围的值。在下面的示例中，能够快速的查询一个明确的counter，指定范围的日期区间里的一个明确的counter，各种counter的集合等。

示例贴：

  所有的这些示例都是用了日期索引和主键。索引会被用到复杂的表达式计算中，所以读一个组织结构化的表不会比全表扫描慢。

下面的例子中，索引不会被用到。

select count() from table where counterId = 34 or url like '%upyachka%'

  日期索引只能读出包含日期查询条件的语句。然而，一个数据part可能包含很多日期的数据，在一个单一的part里，数据是按照主键进行排序的，可能不会将日期作为第一列。因此，如果查询中只是加了日期范围限定没有加主键的限定会导致遍历更多的数据行。

  对于同时读和更新的表，插入操作不会阻塞读的操作。

  读表的行为自动就是并行化的。

  有额外的merge步骤来支持最优化的查询。

  可以用一个很大的单表来不断的往里添加数据。

  数据负值在mergetree这种表引擎中也是支持的，详细看下面的部分 ““data replication””