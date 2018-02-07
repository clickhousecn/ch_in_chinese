<a name="aggregate_functions"></a>

# 聚合函数

聚合函数的行为和其他[主流](http://www.sql-tutorial.com/sql-aggregate-functions-sql-tutorial)的 SQL 基本相同。

除此之外，Clickhouse 还支持：

- [带形参的聚合函数](parametric_functions.md#aggregate_functions_parametric)，除了接受列为参数之外，还另外需要一些静态的参数去定义该聚合函数的行为。
- [聚合函数后缀](combinators.md#aggregate_functions_combinators)，用于改变和增强聚合函数的行为。

**目录**

```eval_rst
.. toctree::

   reference
   parametric_functions
   combinators
```
