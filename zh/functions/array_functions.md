# 与数组相关的函数

## empty

为空数组返回1，或者为非空数组返回0。
结果类型是UInt8。
该函数也适用于字符串。

## notEmpty

为空数组返回0，或者为非空数组返回1。
结果类型是UInt8。
该函数也适用于字符串。


## length

返回数组中的条目数。
结果类型是UInt64。
该函数也适用于字符串。



## emptyArrayUInt8, emptyArrayUInt16, emptyArrayUInt32, emptyArrayUInt64

## emptyArrayInt8, emptyArrayInt16, emptyArrayInt32, emptyArrayInt64

## emptyArrayFloat32, emptyArrayFloat64

## emptyArrayDate, emptyArrayDateTime

## emptyArrayString

接受零参数，并返回相应类型的空数组。



## emptyArrayToSingle

接受一个空数组，同时返回一个等于默认值的单元素数组。



## range(N)

返回从0到N-1的数组数。
以防万一，如果在一个数据块中创建总长度超过100,000,000个元素的数组，则会引发异常。



## array(x1, ...), operator \[x1, ...\]

从函数参数中创建一个数组。
参数必须是常量，并且具有最小通用类型的数据类型。至少传递一个参数，否则不清楚创建哪种类型的数组。也就是说，你不能使用这个函数来创建一个空数组（为此，使用上面描述的 'emptyArray *' 函数）。
返回一个 'Array(T)' 类型的结果，其中 'T' 是传递参数中最小的通用类型。



## arrayConcat

绑定作为参数传递的数组。

```
arrayConcat(arrays)
```

**参数**

- `arrays` – 逗号分隔的[values]的数组。

**示例**

```sql
SELECT arrayConcat([1, 2], [3, 4], [5, 6]) AS res
```

```
┌─res───────────┐
│ [1,2,3,4,5,6] │
└───────────────┘
```

## arrayElement(arr, n), operator arr[n]

从数组 'arr' 中获取索引为 'n' 的元素。'n' 必须是任意的整数类型。
数组中的索引从1开始。
支持负数索引。在这种情况下，它从末尾选择相应的元素。例如，'arr [-1]' 是数组中的最后一项。


如果索引落在数组边界之外，它将返回一些默认值（数值为0，字符串为空字符串等）。



## has(arr, elem)

检查 'arr' 数组是否有 'elem' 元素。
如果元素不在数组中，则返回0；如果在数组中，则返回1。



## indexOf(arr, x)

如果它在数组中，则返回 'x' 元素的索引（从1开始）；如果不在数组中，则返回0。



## countEqual(arr, x)

返回数组中的等于x的元素数量，等同与 arrayCount (elem->elem = x， arr)。


## arrayEnumerate(arr)

返回数组 [1, 2, 3, ..., length (arr) ]

此函数通常被用于ARRAY JOIN。在使用ARRAY JOIN后，它为每个数组执行计数。例如：




```sql
SELECT
    count() AS Reaches,
    countIf(num = 1) AS Hits
FROM test.hits
ARRAY JOIN
    GoalsReached,
    arrayEnumerate(GoalsReached) AS num
WHERE CounterID = 160656
LIMIT 10
```

```text
┌─Reaches─┬──Hits─┐
│   95606 │ 31406 │
└─────────┴───────┘
```

在这个例子中，Reaches 是转换次数（在使用 ARRAY JOIN 后收到的字符串），Hits是页面浏览量（在使用 ARRAY JOIN之前的字符串）。在这种情况下，您可以通过更简单的方式获得相同的结果：



```sql
SELECT
    sum(length(GoalsReached)) AS Reaches,
    count() AS Hits
FROM test.hits
WHERE (CounterID = 160656) AND notEmpty(GoalsReached)
```

```text
┌─Reaches─┬──Hits─┐
│   95606 │ 31406 │
└─────────┴───────┘
```
该功能也可用于高阶功能。例如，您可以使用它为匹配条件的元素获取数组索引。


## arrayEnumerateUniq(arr, ...)

返回与源数组大小相同的数组，为每个元素提示相同值的元素的位置在哪。
例如: arrayEnumerateUniq(\[10, 20, 10, 30\]) = \[1,  1,  2,  1\]。

使用 ARRAY JOIN 和数组元素聚合时，此函数比较有用。例如：


示例：

```sql
SELECT
    Goals.ID AS GoalID,
    sum(Sign) AS Reaches,
    sumIf(Sign, num = 1) AS Visits
FROM test.visits
ARRAY JOIN
    Goals,
    arrayEnumerateUniq(Goals.ID) AS num
WHERE CounterID = 160656
GROUP BY GoalID
ORDER BY Reaches DESC
LIMIT 10
```

```text
┌──GoalID─┬─Reaches─┬─Visits─┐
│   53225 │    3214 │   1097 │
│ 2825062 │    3188 │   1097 │
│   56600 │    2803 │    488 │
│ 1989037 │    2401 │    365 │
│ 2830064 │    2396 │    910 │
│ 1113562 │    2372 │    373 │
│ 3270895 │    2262 │    812 │
│ 1084657 │    2262 │    345 │
│   56599 │    2260 │    799 │
│ 3271094 │    2256 │    812 │
└─────────┴─────────┴────────┘
```

在这个例子中，每个目标ID都有一个转换次数的计算（目标嵌套数据结构中的每个元素都是达到的目标，我们称之为转换）以及会话数量。如果没有 ARRAY JOIN，我们可以将会话的数量计算为总和（Sign）。但是在这种特殊情况下，行与嵌套目标结构相乘，所以为了在这之后计算每个会话，我们将条件应用到 arrayEnumerateUniq（Goals.ID）函数的值中。



arrayEnumerateUniq 函数可以将多个相同大小的数组作为参数。
在这种情况下，唯一性被认为是所有数组中相同位置元素的元组。



```sql
SELECT arrayEnumerateUniq([1, 1, 1, 2, 2, 2], [1, 1, 2, 1, 1, 2]) AS res
```

```text
┌─res───────────┐
│ [1,2,1,1,2,1] │
└───────────────┘
```

当使用带有嵌套数据结构的 ARRAY JOIN，并在这个结构中的多个元素之间进一步聚合时，这是必要的。



## arrayPopBack

删除数组中的最后一项。



```
arrayPopBack(array)
```

**参数**

- `array` – 数组.

**示例**

```sql
SELECT arrayPopBack([1, 2, 3]) AS res
```

```
┌─res───┐
│ [1,2] │
└───────┘
```

## arrayPopFront

删除数组中的第一项。

```
arrayPopFront(array)
```

**Arguments**

- `array` – 数组.

**Example**

```sql
SELECT arrayPopFront([1, 2, 3]) AS res
```

```
┌─res───┐
│ [2,3] │
└───────┘
```

## arrayPushBack

删除数组中的最后一项。
```
arrayPushBack(array, single_value)
```

**Arguments**

- `array` – Array.
- `single_value` – 单个值。只有数字可以添加到带有数值的数组中，只有将字符串能够添加到字符串数组中。在添加数值时，ClickHouse会自动为数组的数据类型设置 'single_value' 类型。有关ClickHouse中数据类型的更多信息，请参阅“[数据类型](../data_types/index.md#data_types)”。



**示例**

```sql
SELECT arrayPushBack(['a'], 'b') AS res
```

```
┌─res───────┐
│ ['a','b'] │
└───────────┘
```

## arrayPushFront

添加一个元素到数组的开头.

```
arrayPushFront(array, single_value)
```

**参数**

- `array` – Array.
- `single_value` – 单个值。只有数字可以添加到带有数值的数组中，只有将字符串能够添加到字符串数组中。在添加数值时，ClickHouse会自动为数组的数据类型设置 'single_value' 类型。有关ClickHouse中数据类型的更多信息，请参阅“[数据类型](../data_types/index.md#data_types)”。

**示例**

```sql
SELECT arrayPushBack(['b'], 'a') AS res
```

```
┌─res───────┐
│ ['a','b'] │
└───────────┘
```

## arraySlice

返回一个数组的切片.

```
arraySlice(array, offset[, length])
```

**示例**

- `array` – 数组数据.
- `offset` – 数组边界的偏移量。正数表示左侧的偏移量，负数表示右侧的缩进量。数组项的编号从1开始。
- `length` - 所需切片的长度。如果你指定一个负值，函数返回一个打开的切片 [offset，array_length - length）。如果省略该值，则函数返回切片[offset，the_end_of_array]。


**示例**

```sql
SELECT arraySlice([1, 2, 3, 4, 5], 2, 3) AS res
```

```
┌─res─────┐
│ [2,3,4] │
└─────────┘
```

## arrayUniq(arr, ...)

如果传递一个参数，它会计算数组中不同元素的数量。如果传递多个参数，它将计算多个数组中相应位置的不同元组的数量。

如果你想得到一个数组中唯一元素的列表，你可以使用 arrayReduce('groupUniqArray', arr)。


## arrayJoin(arr)

一个特殊的函数. 请查看章节 ["ArrayJoin function"](array_join.md#functions_arrayjoin)。

