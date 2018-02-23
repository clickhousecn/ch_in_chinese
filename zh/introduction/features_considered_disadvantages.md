# ClickHouse 的缺点

1. 不支持事务。
2. 对于聚合，查询结果必须可以在单个服务器上的 RAM 中存下。但查询的源数据量是没有上限的。
3. 缺乏完整的 UPDATE / DELETE 实现。


译注：
1.ClickHouse 目前（2018-02-23）已经支持即使在单机 RAM 存不下中间结果的情况下， `GROUP BY` 和 `ORDER BY` 子句也可以很好地工作，参考 [group-by-in-external-memory](https://clickhouse.yandex/docs/en/query_language/queries/#group-by-in-external-memory) 和 [order-by-clause](https://clickhouse.yandex/docs/en/query_language/queries/#order-by-clause)。

2.但对于（IN，JOIN，DISTINCT），还依赖于中间数据要存入单机内存限制。

