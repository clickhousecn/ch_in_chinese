# 入门

## 系统要求

ClickHouse 不是一个跨平台的系统。它需要工作在 Linux Ubuntu Precise（12.04）或更新的版本，操作系统需要是 x86_64 架构且支持 SSE 4.2 指令集。
查看是否支持  SSE 4.2 ：

```bash
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

推荐使用 Ubuntu Trusty，Ubuntu Xenial 或 Ubuntu Precise。终端必须使用 UTF-8 编码（在 Ubuntu 中默认是此编码）。

## 安装

若是为了测试或开发来使用它，ClickHouse 可以安装在单台服务器上或台式计算机上。

### 从包中安装

在 `/etc/apt/sources.list` (或在单独的 `/etc/apt/sources.list.d/clickhouse.list` 文件中), 添加以下库:

```text
deb http://repo.yandex.ru/clickhouse/trusty stable main
```

在其他版本的Ubuntu上，将 `trusty` 替换成  `xenial` 或者  `precise`。如果想使用最新的测试版本，请将 `stable` 替换为 `testing`。

然后运行：

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4    # optional
sudo apt-get update
sudo apt-get install clickhouse-client clickhouse-server-common
```

您也可以从这里手动下载和安装软件包：
<http://repo.yandex.ru/clickhouse/trusty/pool/main/c/clickhouse/>
<http://repo.yandex.ru/clickhouse/xenial/pool/main/c/clickhouse/>
<http://repo.yandex.ru/clickhouse/precise/pool/main/c/clickhouse/>

ClickHouse 可以设置访问限制。它们位于 `users.xml` 文件（`config.xml` 旁边）。
默认情况下，`default` 用户可以在任何地方访问，且不需要密码。请参阅 `user/default/networks`。
有关更多信息，请参阅 [配置文件](../operations/configuration_files.md) 一节

### 从源码安装

要编译，请先参考：build.md 文件。

可以编译对应的包来安装它们。
也可以不安装对应的包来使用程序。

```text
Client: dbms/src/Client/
Server: dbms/src/Server/
```

对于服务端，可以创建一个数据目录，例如：

```text
/opt/clickhouse/data/default/
/opt/clickhouse/metadata/default/
```

(服务端配置文件是可以配置的。)
可以运行 `chown` 命令来将该目录归属给指定用户。

服务端配置中的日志路径为 `src/dbms/src/Server/config.xml`。

### 其他安装方法

Docker 镜像: <https://hub.docker.com/r/yandex/clickhouse-server/>

RPM 包，支持 CentOS 和 RHEL: <https://github.com/Altinity/clickhouse-rpm-install>

Gentoo overlay: <https://github.com/kmeaw/clickhouse-overlay>

## 启动

要启动服务 (或者作为后台进程启动), 运行:

```bash
sudo service clickhouse-server start
```

在 `/var/log/clickhouse-server/` 目录看到对应的日志。

如果服务没有启动，请检查文件 `/etc/clickhouse-server/config.xml` 中的配置。

您也可以从控制台启动服务：

```bash
clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

在这种情况下，日志将被打印到控制台，这在开发过程中很方便。如果配置文件位于当前目录中，则不需要指定 `--config-file` 参数。默认情况下，它使用 `./config.xml`。

您可以使用命令行客户端来连接服务：

```bash
clickhouse-client
```

默认参数表示用 `default` 用户以及空密码来连接 `localhost:9000` 服务。
客户端也可以用来连接远程服务。例如：

```bash
clickhouse-client --host=example.com
```

更多信息，查看 "[命令行客户端](../interfaces/cli.md)" 部分。

检查系统运行情况:

```bash
milovidov@hostname:~/work/metrica/src/dbms/src/Client$ ./clickhouse-client
ClickHouse client version 0.0.18749.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.18749.

:) SELECT 1

SELECT 1

┌─1─┐
│ 1 │
└───┘

1 rows in set. Elapsed: 0.003 sec.

:)
```

**恭喜您, 系统正常工作!**

您可以下载测试数据集来继续尝试 ClickHouse：

```eval_rst
.. toctree::
    :glob:

    example_datasets/*
```

