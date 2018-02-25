# 如何在 Linux 中编译 ClickHouse

编译可以在 Ubuntu 12.04， 14.04 或者更新的 Linux 系统环境执行。
经过适当的修改，它也可以在任何其他 Linux 发行版上运行。
构建过程并不打算在支持在  Mac OS X 环境下工作。
仅支持具有 SSE 4.2的 x86_64 系统。对 AArch64 的支持是实验性的功能。

测试系统是否支持 SSE 4.2，运行：

```bash
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

## 安装 Git 和 CMake

```bash
sudo apt-get install git cmake3
```

或者在较新的系统中运行编译。

## 检测 CPU 核数

```bash
export THREADS=$(grep -c ^processor /proc/cpuinfo)
```

## 安装 GCC 7

它可以通过很多方法来安装。

### 从 PPA 包安装

```bash
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-7 g++-7
```

### 源码安装

参考 [https://github.com/yandex/ClickHouse/blob/master/utils/prepare-environment/install-gcc.sh]

## 使用 GCC 7 来编译

```bash
export CC=gcc-7
export CXX=g++-7
```

## 从包管理中安装三方的库

```bash
sudo apt-get install libicu-dev libreadline-dev libmysqlclient-dev libssl-dev unixodbc-dev
```

## 拉取 ClickHouse 源码

为了获取到 ClickHouse 源码的最新稳定版本：

```bash
git clone -b stable --recursive git@github.com:yandex/ClickHouse.git
# or: git clone -b stable --recursive https://github.com/yandex/ClickHouse.git

cd ClickHouse
```

若为开发需要, 切换到 `master` 分支。
若需要最新的候选版本，切换到 `testing` 分支。

## 编译 ClickHouse

有两种不同的构建方式。

### 构建发布的版本

安装构建 Debian 包需要的环境

```bash
sudo apt-get install devscripts dupload fakeroot debhelper
```

安装最新版本的 Clang。

Clang 会嵌入到 ClickHouse 包中并在运行时使用。最低版本是5.0。 这是可选的。

安装 Clang 的方法, 参考 `utils/prepare-environment/install-clang.sh`

若为了开发目的，您也可以用 Clang 来编译 ClickHouse。
若要发布生成环境的版本， GCC是必须的。

执行发布脚本：

```bash
rm -f ../clickhouse*.deb
./release
```

在上一级目录可以找到编译后的包：

```bash
ls -l ../clickhouse*.deb
```

请注意，debian 包并不是必须的。
ClickHouse 除了 libc 外没有任何运行时依赖关系，所以它其实可以工作在任何 Linux 系统下。

在开发服务器上安装新构建的软件包：

```bash
sudo dpkg -i ../clickhouse*.deb
sudo service clickhouse-server start
```

### 源码编译

```bash
mkdir build
cd build
cmake ..
make -j $THREADS
cd ..
```

若要生成一个可执行文件，运行 `make clickhouse`.
它将会生成可执行文件 `dbms/src/Server/clickhouse`， 可执行文件可以搭配 `client` 及 `server` 参数来使用。

