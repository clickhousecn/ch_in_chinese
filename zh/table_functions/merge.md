# merge

`merge(db_name, 'tables_regexp')` - 创建一个临时的 Merge 表。更多信息，参考 "[表引擎 - Merge](../table_engines/merge.md)" 部分。

表的结构必须与第一个满足正则表达式的表结构一致。

下面是译者补充的示例：
```sql
CREATE TABLE sample1 (x UInt64, d Date DEFAULT today()) ENGINE = MergeTree(d, intHash64(x), intHash64(x), 10);
CREATE TABLE sample2 (x UInt64, d Date DEFAULT today()) ENGINE = MergeTree(d, intHash64(x), intHash64(x), 10);
INSERT INTO sample1 (x) SELECT number AS x FROM system.numbers LIMIT 100000;
INSERT INTO sample2 (x) SELECT number AS x FROM system.numbers LIMIT 200000;

SELECT sum(x) / 3000000  FROM merge(currentDatabase(), '^sample\\d$');
```
