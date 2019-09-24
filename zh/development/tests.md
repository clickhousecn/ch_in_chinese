# 如何运行 ClickHouse 测试

`clickhouse-test` 程序是用来做功能测试的，它是用 Python 2.x 实现的。运行它之前也需要安装三方包。

```bash
$ pip install lxml termcolor
```

简而言之：

- 将 `clickhouse` 程序放在 `/usr/bin` (或者 `PATH`) 目录下。
- 在 `/usr/bin` 目录下创建一个指向 `clickhouse` 的软链 `clickhouse-client`。
- 启动 `clickhouse` 服务。
- 运行 `cd dbms/tests/`。
- 执行 `./clickhouse-test`。

## 使用示例

执行 `./clickhouse-test --help` 来查看更多的运行选项。

在不需要创建软链或修改 `PATH` 的情况下运行测试：

```bash
./clickhouse-test -c "../../build/dbms/src/Server/clickhouse --client"
```

运行单个测试，比如 `00395_nullable`:

```bash
./clickhouse-test 00395
```

