<a name="table_functions-remote"></a>

# remote

使用`remote` 函数可以不用创建 `Distributed` 表而达到访问远程服务的目的。

特点:

```sql
remote('addresses_expr', db, table[, 'user'[, 'password']])
remote('addresses_expr', db.table[, 'user'[, 'password']])
```

`addresses_expr` 是一个生成远程服务端地址的表达式。可以只是一个服务端地址。服务端地址的形式是 `host:port`，或者仅仅是 `host`。`host` 可以用服务器名称来指定，或者是 IPv4/IPV6 地址。其中 IPv6 地址是以方括号格式来指定的。`port` 代表服务端监听的端口。如果省略了 `port` 参数，会使用配置文件中的 `tcp_port` 配置的端口（默认是9000）。

<div class="admonition important">

对于 IPv6 地址，`port` 是必须指定的。

</div>

示例:

```text
example01-01-1
example01-01-1:9000
localhost
127.0.0.1
[::]:9000
[2a02:6b8:0:1111::11]:9000
```

可以用逗号来分割多个地址。在多个地址的情况下，ClickHouse 会使用分布式执行，因此他会并行地发送请求给指定的服务端（恰如多个服务端含有不同的数据分片）

示例:

```text
example01-01-1,example01-02-1
```

表达式的一些部分可以花括号来指定。上面的示例也可以用下面的形式来表示：

```text
example01-0{1,2}-1
```

花括号中可以用两个点号分割两个值来表示一个数值范围（非负整数）。这种表达式可以生产一系列的分片地址。如果第一个字母是以零开头的，那么这些值都是以相同的零对齐方式形成的。上面的示例也可以用下面的形式来表示：

```text
example01-{01..02}-1
```

如果你有多对花括号，则生成相应集的笛卡儿积。

地址和部分花括号的地址可以用管道符号 `|` 来分割。这种情况下，相对应的地址集合被当做是副本地址，查询会被发送到第一个健康的副本地址去。副本的顺序选择方式见 [load_balancing](../operations/settings/settings.md#settings-load_balancing) 部分。

示例:

```text
example01-{01..02}-{1|2}
```

上述示例指定了两个分片地址，每个地址对应两个副本地址。

地址数目上限由一个常数指定，目前这个值是1000。

使用 `remote` 表函数比创建一个 `Distributed` 表更不合适，因为使用 `remote` 表函数，每次请求都会重新建立新的连接。此外，如果设置了服务端域名，域名会被解析，但解析不同副本域名的错误并不会被计入。所以当执行大规模分布式查询时，总是要提前创建 `Distributed` 表，而不要使用 `remote` 表函数。

`remote` 表函数在以下情况下会有用：

- 查询一个指定的服务端数据，当需要对数据进行对比，调试和测试的时候。
- 以研究目的对不同 ClickHouse 集群进行查询。
- 手动方式生成且不常见的分布式查询。
- 服务器地址经常被重定义的分布式查询。

如果 `user` 未被指定，默认是 `default`。
如果 `password` 未被指定，会使用空字符串密码。

