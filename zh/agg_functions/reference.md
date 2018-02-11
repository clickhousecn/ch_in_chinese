<a name="aggregate_functions_reference"></a>

# 函数文档

## count()

返回类型为 UInt64 的行数。可以不带任何参数。
语法 `COUNT(DISTINCT x)` 不被支持，但可以用另外的聚合函数 `uniq` 达到类似的目的。

`SELECT count() FROM table` 查询并没有被优化，因为Clickhouse没有单独地保存这个信息。它会用某个比较小的列来统计行数。

## any(x)

返回第一个值。
查询可以用任何顺序执行，或者在不同的时间用不同的顺序执行，所以这个函数的结果是不确定的。如果需要确定的结果，可以使用 'min' 或者 'max’ 。

某些情况下可以保证顺序，比如当 SELECT 来自一个有 ORDER BY 的子查询。

当一个 `SELECT` 查询有 `GROUP BY` 语句，或者至少一个聚合函数， 与 MySQL 不同的是，ClickHouse 要求所有在 `SELECT`, `HAVING`, 和 `ORDER BY` 的表达式必须来自 `GROUP BY` 的键，或者来自聚合函数。也就是说，每一个被 SELECT 的列必须要不用在 `GROUP BY` 键上或者在聚合函数里面。要实现 MySQL 的行为，可以使用 `any`。

## anyHeavy
利用 [heavy hitters](http://www.cs.umd.edu/~samir/498/karp.pdf) 算法寻找一个频繁出现的值。如果有一个值在每个查询线程都有超过一半的出现概率，将会返回该值。通常结果是不确定的。

```
anyHeavy(column)
```

**参数**

- `column` – 列名称。

**示例**

使用 [OnTime](../getting_started/example_datasets/ontime.md#example_datasets-ontime) 数据集，用 `AirlineID` 查询频繁出现的航班。

```sql
SELECT anyHeavy(AirlineID) AS res
FROM ontime
```

```
┌───res─┐
│ 19690 │
└───────┘
```

## anyLast(x)

返回最后一个值。其结果不确定性如同 `any`。

## min(x)

返回最小值。

## max(x)

返回最大值。

## argMin(arg, val)

返回 `val` 最小那一行 `arg` 的值。如果有多行数据 `val` 值均为最小值，返回第一个遇到的值。

## argMax(arg, val)

返回 `val` 最大那一行 `arg` 的值。如果有多行数据 `val` 值均为最大值，返回第一个遇到的值。

## sum(x)

求和，只适用于数值。

## sumWithOverflow(x)

用参数所用的类型存储和；如溢出，返回错误。只适用于数值。

## sumMap(key, value)

把 'value' 数组和 'key' 数组看作一组map，然后对多组map求和。对每一行，value 数组和 key 数组的元素个数必须相等。
返回一个数组对：键会被排序，值是原始 map 里面其键相对的值的总和。

示例：

```sql
CREATE TABLE sum_map(
    date Date,
    timeslot DateTime,
    statusMap Nested(
        status UInt16,
        requests UInt64
    )
) ENGINE = Log;
INSERT INTO sum_map VALUES
    ('2000-01-01', '2000-01-01 00:00:00', [1, 2, 3], [10, 10, 10]),
    ('2000-01-01', '2000-01-01 00:00:00', [3, 4, 5], [10, 10, 10]),
    ('2000-01-01', '2000-01-01 00:01:00', [4, 5, 6], [10, 10, 10]),
    ('2000-01-01', '2000-01-01 00:01:00', [6, 7, 8], [10, 10, 10]);
SELECT
    timeslot,
    sumMap(statusMap.status, statusMap.requests)
FROM sum_map
GROUP BY timeslot
```

```text
┌────────────timeslot─┬─sumMap(statusMap.status, statusMap.requests)─┐
│ 2000-01-01 00:00:00 │ ([1,2,3,4,5],[10,10,20,10,10])               │
│ 2000-01-01 00:01:00 │ ([4,5,6,7,8],[10,10,20,10,10])               │
└─────────────────────┴──────────────────────────────────────────────┘
```

## avg(x)

求平均值。只适用于数值。结果类型为 Float64。

## uniq(x)

近似求不同值个数。适用类型包括数值，字符串，日期，日期与时间，多参数和 tuple 参数。

该函数使用一个自适应采样算法，计算状态所用的 hash 大小最大为 65536。
当数据量比较小的时候（小于 65536）这个算法能提供非常准确的结果和高效的 CPU 使用率（使用不太多这种函数进行计算时，`uniq` 的速度几乎与其它聚合函数相当）。

其结果是确定的（不受查询执行顺序影响）。

## uniqCombined(x)

近似求不同值个数。适用类型包括数值，字符串，日期，日期与时间，多参数和 tuple 参数。

这个函数用了三种不同的算法：数组，哈希表和带有误差修正表的[HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog)。其内存使用是 `uniq` 的几分之一，准确性是其几倍有余，但速度会稍微慢一点，尽管有时它甚至更快，比如传输大量聚合状态的分布式查询中。最大聚合状态是 96 KiB (HyperLogLog of 217 6-bit cells).

其结果是确定的（不受查询执行顺序影响）。

`uniqCombined` 是个很好的统计不同值个数的默认选择。

## uniqHLL12(x)

使用[HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog)算法来近似求不同值个数。
用了212 5-bit cells。每个状态的大小略微大于2.5 KB.

其结果是确定的（不受查询执行顺序影响）。

大多数情况下，使用`uniq`和`uniqCombined`足矣。

## uniqExact(x)

返回不同值个数的精确值。

在大数据的环境中，使用近似值一般没有什么问题，所以建议还是使用 `uniq` 函数。
如果你真的需要准确值，可以使用 `uniqExact` 函数。

`uniqExact` 函数相比于 `uniq` 需要更多的内存，因为其聚合状态（哈希表）的大小随着不同的值的个数增长而增长，并没有上限。

## groupArray(x), groupArray(max_size)(x)

将结果转化为数组，但不保证数组内元素的顺序。

第二种（含有 `max_size` 语法）限制了数组的大小。例如 `groupArray (1) (x)` 与 `[any (x)]` 是等价的。

某些情况下可以保证顺序，比如当 SELECT 来自一个有 ORDER BY的子查询。

<a name="agg_functions_groupArrayInsertAt"></a>

## groupArrayInsertAt(default, len)(x, pos)

> 译者注：
> 这个函数由于英文文档并没有标明参数名称，所以具体做什么的有点难以理解。其实大多数函数的用法的最好文档是clickhouse本身的测试用例。比如这个函数可以参考[这个测试用例](https://github.com/yandex/ClickHouse/blob/cfc4c987c5e1b4290ceb15247846ca39225496eb/dbms/tests/queries/0_stateless/00459_group_array_insert_at.sql)，运行
> ```
> SELECT groupArrayInsertAt(toString(number), number * 2)
> FROM (
>   SELECT * FROM system.numbers LIMIT 10
> );
> ```
> 得到
> ```
> ['0','','1','','2','','3','','4','','5','','6','','7','','8','','9']
> ```
> 大概就可以理解这个函数究竟是什么黑魔法了。
> 上述例子省略了可选形参 (default, len)，其作用下文叙述。

将一个值插入数组的指定位置。

该函数有两个参数：值和位置。如果多于一行 pos 是相同的话（比如说 pos 是个常数，`groupArrayInsertAt(value, 3) FROM (...)`），那么结果数组中位置在 pos 的值有可能是这些行的值的任意一个（在单线程环境下，值会是第一个）。如果某个位置并没有被赋值，将使用空字符串或者零之类的默认值。

可选的形参：

- 空位置的默认值；
- 生成的数组的长度。这让你能够保证返回的数组长度与其他聚合键一致（如有）。使用这个形参时，默认值必须指明。

## groupUniqArray(x)

生成仅有相异的元素组成的数组。其内存消耗与 `uniqExact` 相同。

## quantile(level)(x)

近似得到指定的level的分位值。'level'是一个 0 - 1 之间的浮点数常量。
我们建议'level'取值范围为0.01..0.99
不要使用0, 1，因为你可以使用 `min` 和 `max` 函数代替。

包括这个函数的所有分位值函数在内，'level'都是一个可以忽略的形参，其默认值是 0.5（这时相当于中位数）。

该函数适用于数值，日期，日期与时间。
对于数值实参，将返回Float64。与时间有关的实参会返回实参的类型。

该函数[水塘抽样法](https://zh.wikipedia.org/wiki/%E6%B0%B4%E5%A1%98%E6%8A%BD%E6%A8%A3)，水塘大小不大于 8192。
根据分位值的定义，如果需要的话，结果可能是两个相邻的值得线性模拟。
该函数的准确性很低。另见： `quantileTiming`, `quantileTDigest`, `quantileExact`.

函数结果取决于查询运行顺序，因此是不确定的。

当在一个查询中使用多个不同'level'的 `quantile` （或类似的）函数时，每个函数内部的状态并不会被合并或共享，所以效率会比 `quantiles` （或类似的）函数低。在这种情况下，建议用后者。

## quantileDeterministic(level)(x, determinator)

原理与 `quantile` 相同，只是结果是确定的，并不依赖查询执行的顺序。

为了实现这种效果，函数需要第二个参数determinator；该参数的哈希值用于水塘抽样法的随机数生成器的种子。为了让该函数正确地执行，同一个determinator不能被频繁地使用。作为例子，你可以使用事件ID或者用户ID。

如果想用于时间长度相关的分位值，建议使用专用函数  `quantileTiming`.

## quantileTiming(level)(x)

用确定的精度来计算指定的level的分位值。适用于数值。设计用于计算以毫秒为单位的页面加载时间。

如果结果数值大于 30,000（页面加载时间超过30秒），那么结果将会是 30,000。

如果数据总量少于 5670 的话，计算结果将会是精确的。

否则：
- 如果所有的数值都小于 1024（时间少于 1024 毫秒）的话，那么计算结果将会是精确的；
- 否则，计算结果将会取整到 16 毫秒的倍数。

> 译者注：
> `quantileTiming` 内部一共有 3 种不同的数据结构来处理不同总数量的元素的分位值。其中 Tiny 和 Medium在当且仅当总数量小于 [8*(1+1024+(30000-1024)/16)/2/2](https://github.com/yandex/ClickHouse/blob/da1233fe3f1cd7c5e21deecfb4304cf8fd6e42c5/dbms/src/AggregateFunctions/QuantileTiming.h#L545) = 5672 时才会使用，所有元素的值都会被单独地储存在数组里，因而结果是准确的。而 Large 采用更加复杂的动态结构，具体算法可以阅读源代码。

函数不支持负数数值。

返回的结果的类型为 Float32。如果该函数并没有接收到任何数值（比如，当使用 `quantileTimingIf` 而所有行的条件都不符合的时候），函数将返回 'nan'。这样做的目的是区分数值为0与没有数值的情况。另见 "ORDER BY clause" 关于 NaN 排序的注解。

其结果是确定的（不受查询执行顺序影响）。

当使用这个函数来求页面加载速度的分位值时，其速度与精确度都好于 `quantile` 函数。

## quantileTimingWeighted(level)(x, weight)

不同于 'quantileTiming' （译者注：原文为 'medianTiming'，但根据上下文，推断为笔误），这个函数有第二个参数 "weight"。Weight 是一个非负整数。

其结果相当于求每一行的数值 x 出现了 weight 这么多次的分位值（译者注：适合原始数据是类似直方图那样的加权数据，见 `quantileExactWeighted` 的解释）。

## quantileExact(level)(x)

求精确的分位值。所有传入的值将会储存在一个数组中，并被部分地排序。因此该函数需要 O(n) 大小的内存，其中 n 是传入的数值的个数。当 n 并不大时，这个函数是相当高效的。

## quantileExactWeighted(level)(x, weight)

求精确的分位值，并假设每一个值 x 都出现了 weight 那么多次。这个函数的参数可以被视为一个直方图，其中 x 是直方图的样本值，weight 是直方图里的样本出现次数；和直方图不同的是，x 值允许重复，所以某种意义上这个函数会对进来的参数进行相加整合。

与 `quantileExact` 函数不同的是，该函数使用了哈希表作为其算法。因此，如果 x 的值有大量重复的话，使用这个函数（将 weight 设置为 1 )反而会比 `quantileExact` 对内存的需求更少。

## quantileTDigest(level)(x)

近似地使用  [t-digest](https://github.com/tdunning/t-digest/blob/master/docs/t-digest-paper/histo.pdf) 算法来求分位值。最大误差为 1%。状态的内存使用与数值个数对数相关。

该函数的性能逊于 `quantile` 和 `quantileTiming`。但其内存消耗与精度比比 `quantile`好很多。

函数结果取决于查询运行顺序，因此是不确定的。

## median

所有求分位值的函数都有其对应的中位数函数：`median`，`medianDeterministic`，`medianTiming`，`medianTimingWeighted`，`medianExact`，`medianExactWeighted` 和 `medianTDigest`。当使用 `quantile` 函数的默认值时，其行为和对应的 `median` 函数相同。

## quantiles(level1, level2, ...)(x)

所有求分位值的函数都有其返回多个分位值的函数：`quantiles`，`quantilesDeterministic`，`quantilesTiming`，`quantilesTimingWeighted`，`quantilesExact`，`quantilesExactWeighted`，`quantilesTDigest`。这些函数一次性计算所有 level 的分位值，然后以数组的形式返回结果。

## varSamp(x)

求样本方差 `Σ((x - x̅)^2) / (n - 1)`，其中 `n` 是样本大小， `x̅` 是 `x` 的平均值。

如果 `x` 是一个随机变量的样本，函数返回该随机变量的无偏的估计值。

函数返回 `Float64`。当 `n <= 1` 时，函数返回 `+∞`。

## varPop(x)

求总体方差 `Σ((x - x̅)^2) / n`（译者注：原文为 `n - 1`，但根据函数名称和程序代码，应为 `n`），其中 `n` 是样本大小， `x̅` 是 `x` 的平均值。

函数返回 `Float64`。

## stddevSamp(x)

求样本标准差，相当于 `varSamp(x)` 的算术平方根。

## stddevPop(x)

求总体标准差，相当于 `varPop(x)` 的算术平方根。

## topK

返回包含出现次数最多的K个值的数组。结果依频率降序排列（与值本身无关）。

分析 TopK 的算法为 [ Filtered Space-Saving](http://www.l2f.inesc-id.pt/~fmmb/wiki/uploads/Work/misnis.ref0a.pdf)，基于 reduce-and-combine 算法 [Parallel Space Saving](https://arxiv.org/pdf/1401.0702.pdf)。

```
topK(N)(column)
```

这个函数并不保证结果的准确性。在某些情况下，其有可能返返回出现次数并不是最多的那些结果。

我们建议使用时保证 `N < 10 `，因为大的 `N` 值会会导致速度变慢。最大支持的 N 值为 65536。

**参数**

- 'N' – 最频繁出现的多少个值。
- ' x ' – 列名称。

**示例**

我们用 [OnTime](../getting_started/example_datasets/ontime.md#example_datasets-ontime) 数据库去查找最频繁出现的航班 ID，其列名为 `AirlineID`。

```sql
SELECT topK(3)(AirlineID) AS res
FROM ontime
```

```
┌─res─────────────────┐
│ [19393,19790,19805] │
└─────────────────────┘
```

## covarSamp(x, y)

求样本协方差 `Σ((x - x̅)(y - y̅)) / (n - 1)`。

函数返回 `Float64`。当 `n <= 1` 时，函数返回 `+∞`。

## covarPop(x, y)

求总体协方差 `Σ((x - x̅)(y - y̅)) / n`。

## corr(x, y)

求皮尔逊相关系数 `Σ((x - x̅)(y - y̅)) / sqrt(Σ((x - x̅)^2) * Σ((y - y̅)^2))`。
