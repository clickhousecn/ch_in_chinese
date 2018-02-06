# Tuple(T1, T2, ...)

Tuples can't be written to tables (other than Memory tables). They are used for temporary column grouping. Columns can be grouped when an IN expression is used in a query, and for specifying certain formal parameters of lambda functions. For more information, see "IN operators" and "Higher order functions".
元组不能写入表中（除 Memory 表外）。它们会用于临时列分组。在某个查询中，IN 表达式和带特定参数的 lambda 函数可以来对临时列进行分组。更多信息，参见 "IN 操作符" and "高阶函数"。

元组可以当做查询的结果输出。这种情况下，除了 JSON 外的其他文本格式，括号中的值都是以逗号分隔的。在 JSON 格式中，元组会以数组方式输出（在中括号内）。

