# 如何在 Mac OS X 编译 ClickHouse

可以在 Mac OS X 10.12 环境下编译。如果您使用的是早期版本，您可以尝试在本说明中使用 Gentoo Prefix 和 clang sl 构建 ClickHouse。
经过适当的修改，它也可以在任何其他 Linux 发行版上运行。


## 安装 Homebrew

```bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## 安装所需的编译器，工具，类库

```bash
brew install cmake gcc icu4c mysql openssl unixodbc libtool gettext homebrew/dupes/zlib readline boost --cc=gcc-7
```

## 拉取 ClickHouse 源码

为了获取到 ClickHouse 源码的最新稳定版本：

```bash
git clone -b stable --recursive --depth=10 git@github.com:yandex/ClickHouse.git
# or: git clone -b stable --recursive --depth=10 https://github.com/yandex/ClickHouse.git

cd ClickHouse
```

若为开发需要, 切换到 `master` 分支。
若需要最新的候选版本，切换到 `testing` 分支。

## 编译 ClickHouse

```bash
mkdir build
cd build
cmake .. -DCMAKE_CXX_COMPILER=`which g++-7` -DCMAKE_C_COMPILER=`which gcc-7`
make -j `sysctl -n hw.ncpu`
cd ..
```

## 警告

如果您打算运行 clickhouse-server，请确保增加系统的 `maxfiles` 变量。更多相关详细信息，请参见[MacOS.md](https://github.com/yandex/ClickHouse/blob/master/MacOS.md)。

