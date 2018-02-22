# 运算符

根据它们的优先级和相关性，所有运算符都将在查询解析阶段转换为相应的函数。
按照它们的优先级来排列运算符组合（在列表中的位置越高，运算符越早连接到它的参数）。

## 访问运算符

`a[N]`	： 访问数组中的一个元素； `arrayElement(a, N)` 函数

`a.N`	： 访问元组中的一个元素； `tupleElement(a, N)` 函数

## 数值否定运算符

`-a` 	：  `negate (a)` 函数

## 乘法除法运算符

`a * b` 	：  `multiply (a, b) ` 函数

`a / b` 	：  `divide(a, b) ` 函数

`a % b`	：  `modulo(a, b) ` 函数

## 加法减法运算符

`a + b`	：  `plus(a, b) ` 函数

`a - b` 	：  `minus(a, b) ` 函数

## 比较运算符

`a = b`	：  `equals(a, b) ` 函数

`a == b`	：  ` equals(a, b) ` 函数

`a != b`	：  `notEquals(a, b) ` 函数

`a <> b`	：  `notEquals(a, b) ` 函数

`a <= b`	：  `lessOrEquals(a, b) ` 函数

`a >= b`	：  `greaterOrEquals(a, b) ` 函数

`a < b`	：  `less(a, b) ` 函数

`a > b`	：  `greater(a, b) ` 函数

`a LIKE s`	：  `like(a, b) ` 函数

`a NOT LIKE s`	：  `notLike(a, b) ` 函数

`a BETWEEN b AND c`	：  与 `a >= b AND a <= c` 一样

## 数据集中的运算符

*参考 "IN operators"*

`a IN ...`	：  `in(a, b)` 函数

`a NOT IN ...`	：  `notIn(a, b) ` 函数

`a GLOBAL IN ...`	：  `globalIn(a, b) ` 函数

`a GLOBAL NOT IN ...`	：  `globalNotIn(a, b) ` 函数

## 逻辑否定运算符

`NOT a`  `not(a) ` 函数

## 逻辑 "AND" 运算符

`a AND b`	： `and(a, b) ` 函数

## 逻辑 "OR" 运算符

`a OR b`	：  `or(a, b) ` 函数

## 条件运算符

`a ? b : c`	：  `if(a, b, c) ` 函数

注意:

条件运算符计算 b 和 c 的值，n 满足条件 a，n 返回相应的值。但如果 b 或 c 是一个 `arrayJoin()` 函数，则无论 a 条件如何，每行都将被复制。

## 条件表达式

```sql
CASE [x]
    WHEN a THEN b
    [WHEN ... THEN ...]
    ELSE c
END
```

如果 `x` 满足条件， 执行 (x, \[a, ...\], \[b, ...\], c)，否则执行`multiIf(a, b, ..., c)`。

## 连接运算符

`s1 || s2`	：  `concat(s1, s2) ` 函数

## Lambda 表达式

`x -> expr`	：  `lambda(x, expr) ` 函数

以下运算符没有优先级，因为 y 是括号：

## 创建数组运算符

`[x1, ...]`	：  `array(x1, ...) ` 函数

## 创建元组运算符

`(x1, x2, ...)`	：  `tuple(x2, x2, ...) ` 函数

## 相关性

所有的二元运算符都具有相关性。 例如，`1 + 2 + 3`被转换为`plus(plus(1, 2), 3)`。
有时候这不按期望执行。例如，` SELECT 4 > 2 > 3`将返回结果0。

为了提高效率，`and` 和 `or` 函数接受任意数量的参数。`AND` 和 `OR` 运算符的相应变换为对应函数的调用。
