# Star scheme

编译 dbgen: <https://github.com/vadimtk/ssb-dbgen>

```bash
git clone git@github.com:vadimtk/ssb-dbgen.git
cd ssb-dbgen
make
```

这个过程中会有一些警告信息，但这是正常的。

将 ` dbgen`  and ` dists.dss`  放在 800GB 空闲磁盘的任意目录下。

生成数据：

```bash
./dbgen -s 1000 -T c
./dbgen -s 1000 -T l
```

在 ClickHouse 中创建对应表：

```sql
CREATE TABLE lineorder (
        LO_ORDERKEY             UInt32,
        LO_LINENUMBER           UInt8,
        LO_CUSTKEY              UInt32,
        LO_PARTKEY              UInt32,
        LO_SUPPKEY              UInt32,
        LO_ORDERDATE            Date,
        LO_ORDERPRIORITY        String,
        LO_SHIPPRIORITY         UInt8,
        LO_QUANTITY             UInt8,
        LO_EXTENDEDPRICE        UInt32,
        LO_ORDTOTALPRICE        UInt32,
        LO_DISCOUNT             UInt8,
        LO_REVENUE              UInt32,
        LO_SUPPLYCOST           UInt32,
        LO_TAX                  UInt8,
        LO_COMMITDATE           Date,
        LO_SHIPMODE             String
)Engine=MergeTree(LO_ORDERDATE,(LO_ORDERKEY,LO_LINENUMBER,LO_ORDERDATE),8192);

CREATE TABLE customer (
        C_CUSTKEY       UInt32,
        C_NAME          String,
        C_ADDRESS       String,
        C_CITY          String,
        C_NATION        String,
        C_REGION        String,
        C_PHONE         String,
        C_MKTSEGMENT    String,
        C_FAKEDATE      Date
)Engine=MergeTree(C_FAKEDATE,(C_CUSTKEY,C_FAKEDATE),8192);

CREATE TABLE part (
        P_PARTKEY       UInt32,
        P_NAME          String,
        P_MFGR          String,
        P_CATEGORY      String,
        P_BRAND         String,
        P_COLOR         String,
        P_TYPE          String,
        P_SIZE          UInt8,
        P_CONTAINER     String,
        P_FAKEDATE      Date
)Engine=MergeTree(P_FAKEDATE,(P_PARTKEY,P_FAKEDATE),8192);

CREATE TABLE lineorderd AS lineorder ENGINE = Distributed(perftest_3shards_1replicas, default, lineorder, rand());
CREATE TABLE customerd AS customer ENGINE = Distributed(perftest_3shards_1replicas, default, customer, rand());
CREATE TABLE partd AS part ENGINE = Distributed(perftest_3shards_1replicas, default, part, rand());
```

要在单个服务器上进行测试，只需使用 MergeTree 表。
对于分布式测试，需要在配置文件中配置 `perftest_3shards_1replicas` 集群。
接下来，在每个服务器上创建 MergeTree 表，并在其上面创建一个分布式表。

下载数据（在分布式版本中将 'customer' 更改为 'customerd' ）：

```bash
cat customer.tbl | sed 's/$/2000-01-01/' | clickhouse-client --query "INSERT INTO customer FORMAT CSV"
cat lineorder.tbl | clickhouse-client --query "INSERT INTO lineorder FORMAT CSV"
```
