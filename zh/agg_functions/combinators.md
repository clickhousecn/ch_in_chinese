<a name="aggregate_functions_combinators"></a>

# 聚合函数后缀

我们可以通过添加以下后缀来改变聚合函数的行为。

## -If

后缀 -If 可以加在任意聚合函数的后面。在这种情况下，聚合函数额外需要一个类型为 UInt8 的条件参数。聚合函数仅仅处理符合该条件的行。如果没有一行符合要求，聚合函数返回默认值（通常是 0 或者空字符串）。

例子：`sumIf(column, cond)`, `countIf(cond)`, `avgIf(x, cond)`, `quantilesTimingIf(level1, level2)(x, cond)`, `argMinIf(arg, val, cond)`，等等。

有了这种带条件的聚合函数，你可以同时计算多个条件的聚合而无需使用子查询或者 `JOIN`。例如，在 Yandex.Metrica 当中，带条件的聚合函数被用于网段比较。

## -Array

后缀 -Array 可以加在任意聚合函数的后面。在这种情况下，聚合函数需要的参数类型由原来的 'T' 变成了数组参数 'Array(T)'。如果该聚合函数接受多个数组参数，这些数组参数的长度必须相同。在处理数组的时候，其行为等价于将所有行的数组合并成为一个数组，然后用原聚合函数处理这个数组里面的所有元素。

示例1： `sumArray(arr)` - 将所有数组的所有元素求和。在这个例子里面，也可以简化为 `sum(arraySum(arr))`。

> 译者注：
>
> 考虑表：
>
> | DataArray |
> | --------- |
> | [1, 2, 3] |
> | [2, 3, 4, 5] |
>
> 下列SQL
> ```
> SELECT sumArray(DataArray) FROM
> (
>   SELECT arrayJoin([[1, 2, 3], [2, 3, 4, 5]]) As DataArray FROM system.one
> )
> ```
> 的结果为：`20`。


示例2： `uniqArray(arr)` – 求所有数组的所有元素有多少相异的值。这个例子可以简化为 `uniq(arrayJoin(arr))`，但arrayJoin并不是任何时候都可以使用。

> 译者注：
>
> 考虑表：
> 
> | DataArray |
> | --------- |
> | [1, 2, 3] |
> | [2, 3, 4, 5] |
> 
> 下列SQL
> ```
> SELECT uniqArray(DataArray) FROM
> (
>   SELECT arrayJoin([[1, 2, 3], [2, 3, 4, 5]]) As DataArray FROM system.one
> )
> ```
> 的结果为：`5`。

-If 和 -Array 后缀可以合并使用，'Array' 必须在 'If' 之前。比如：`uniqArrayIf(arr, cond)`，`quantilesTimingArrayIf(level1, level2)(arr, cond)`。条件的类型依然是 UInt8。

> 译者注：
>
> 考虑表：
> 
> | DataArray |
> | --------- |
> | [1, 2, 3] |
> | [2, 3, 4, 5] |
> 
> 下列SQL
> ```
> SELECT sumArrayIf(DataArray, length(DataArray) = 3) FROM
> (
>   SELECT arrayJoin([[1, 2, 3], [2, 3, 4, 5]]) As DataArray FROM system.one
> )
> ```
> 的结果为：`6`。

## -State

使用该后缀将返回一个中间聚合状态（比如说，对于 `uniq`，这个中间状态是一个包含一些不同元素的哈希表），而不是最终的结果（比如说，对于 `uniq`，那就是多少个不同的元素。这样我们可以之后再处理（使用-Merge），或者存储在一个表里以供以后再聚合。详情见 "AggregatingMergeTree" 和 "Functions for working with intermediate aggregation states"。

## -Merge

使用该后缀可以接受一个中间聚合状态参数，然后返回最终的结果。

## -MergeState

使用该后缀与-Merge类似，都可以合并多个中间聚合状态。与之不同的是，它返回的是也是一个中间聚合状态。

## -ForEach

-ForEach垂直地用原来的聚合函数处理所有数组同一列的数据，然后将聚合的结果重新组装成一个数组返回。例如，`sumForEach` 用于数组 `[1, 2]`, `[3, 4, 5]` 和 `[6, 7]` 会得到结果 `[10, 13, 5]`。
