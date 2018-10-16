# Clickhouse 架构概述

ClickHouse 是一个列式数据库管理系统。这不仅仅体现在其数据是按列存储的，还体现在查询时数据也是以数组（向量，或者同一列中的一些值）的方式执行。只要有可能，操作都是以数组为单位而不是单一的值执行。这被称作“向量化查询执行”，用于降低处理数据时所需要的调度开销。

> 这个想法并不新鲜。其最早可追溯到编程语言 `APL`，以及其衍生语言 `A+`， `J`， `K`和 `Q`。面向数组编程在学界数据处理中应用非常广泛。同样地，在关系数据库中也有运用了这个想法的例子，比如 `Vectorwise`。
>
> 为了加快执行查询的速度，我们有两种不同的方法：向量化查询执行和运行时生成代码。对于后者，所有查询的二进制代码都是即时生成，这样就可以避免间接引用和动态调度。不过，这两种方法各有千秋，并没有哪个在任何情况下都比对方好。运行时生成代码的好处时它可以把很多操作捆绑在一起，从而更好地利用 CPU 的算力和流水线。向量化查询执行在某些情况下并不实用，因为使用它时需要往 CPU 缓存读取和写入临时的向量数据，而如果这些临时的数据太大以至于无法完整放入 L2 缓存，这种算法就会出现问题。当然，向量化查询执行可以很好地利用 CPU 的 SIMD 功能。我们的几位朋友在[一篇研究](http://15721.courses.cs.cmu.edu/spring2016/papers/p5-sompolski.pdf)中指出如果这两种方法混用的话，可能得到更好的效果。ClickHouse 主要使用向量化查询执行，同时少量地尝试使用运行时生成代码（仅用在 GROUP BY 第一阶段的内部循环）。

## 列

`IColumn` 接口用于在内存中表示一列，以及列里的数据。这个接口提供了一些辅助函数，用于编写关系运算符。几乎所有的操作都是 immutable 的：不能修改原来的列对象，而只会生成新的列对象。举个例子，`IColumn::filter` 方法的参数是一个过滤器掩码字符，生成一个新的只包含过滤后数据的列。这个方法用于 `WHERE` 和 `HAVING` 这种关系运算符。另外的例子包括用于 `ORDER BY` 的 `IColumn::permute`，用于 `LIMIT` 的 `IColumn::cut`，等等。

几种基于 `IColumn` 接口的实现（包括 `ColumnUInt8`，`ColumnString` 等等）分别定义了列的值如何储存在内存中，通常是一个连续的数组。对于整形的列，仅仅需要一个连续的数组，例如 `std::vector`。对于字符串或者数组的列，需要两个 vectors：一个储存所有元素的数据，另外一个储存每个元素在数据数组中的偏移位置。还有一个特殊的列实现 `ColumnConst`，用于在内存里储存一个常量作为一个列。

## 字段

然而，我们也可以使用单个的值。我们有一个专用的类 `Field`；它实际上是 `UInt64`， `Int64`， `Float64`， `String` 和 `Array` 的一个可区分的联合（Discriminated Union）。`IColumn` 有一个返回类型为 `Field` 的 `operator[]` 方法，用于拿到第 n 个值；有一个 `insert` 方法将一个 `Field` 添加到列的末端。这些方法都不是很有效率，因为它们都需要用临时的 `Field` 对象来存储单个的值。更高效的方法包括 `insertFrom` 和 `insertRangeFrom`，等等。

`Field` 本身并没有带有详细的表中数据类型信息。举个例子，`UInt8`，`UInt16`，`UInt32` 和 `UInt64` 在一个 `Field` 中都以 `UInt64` 来表示。

## 不严格的封装

> 译者注：英文文档原文是 `Leaky Abstraction`，直译为“泄露的抽象”，下文主要在描述尽管有 `IColumn` 这一层抽象，为了执行效率，真正的执行还是会把该对象重新类型转换成真正的具体类型以对其结构里的数据直接的操作，所以意译为 `不严格的封装`

`IColumn` 定义了一些常见的关系类数据转化的方法，但对于特定的类型所需要的方法，仍然捉襟见肘。举个栗子， `ColumnUInt64` 没有将两列求和的方法，而 `ColumnString` 没有寻找子字符串的方法。不少这样的函数都在 `IColumn` 之外实现。

很多以列为对象的函数都可以有两种写法，一种是通用而低效的方式通过 `IColumn` 获取 `Field` 的值，另一种是假设知道数据在内存中如何存储而用专门的方式获得其值。在第二种方法中，函数在访问数据时将 `IColumn` 类型转换成具体的某种实现，从而直接与其内部结构打交道。举个例子， `ColumnUInt64` 有一个返回内部数组引用的方法 `getData`，还有另一个方法直接读写那个数组。实际上，我们会故意泄露（本来被抽象成 `IColumn` 的）列对象的内部的数据结构来达到依据不同的类型来高效执行函数的目的。

## 数据类型

`IDataType` 用于序列化和反序列化，也就是对以二进制或者文本表示的一串列的值或者单一值进行读写。
`IDataType` 的具体实现与表格中的数据类型一一对应。比如，实现的类型包括 `DataTypeUInt32`，`DataTypeDateTime` 和 `DataTypeString` 等。

`IDataType` 和 `IColumn` 没有很强烈的相关。不同的数据类型在内存中可以用同一种 `IColumn` 实现。例如， `DataTypeUInt32` 和 `DataTypeDateTime` 都可以用 `ColumnUInt32` 或者 `ColumnConstUInt32` 表示。另外，同一种数据类型也可以用多于一种的 `IColumn` 实现。例如，`DataTypeUInt8` 可以用 `ColumnUInt8` 或者 `ColumnConstUInt8` 表示。

`IDataType` 仅存储元数据。 比如，`DataTypeUInt8` 实际上除了虚表指针之外并不存储任何东西；`DataTypeFixedString` 也仅仅储存了 `N`（定长字符串的长度）。

`IDataType` 对于不同的数据格式都有辅助方法。比如说序列化可能包含引号的值，序列化成 JSON，序列化成 XML 格式的一部分等。但这些辅助方法与数据格式没有一一对应的关系。例如，数据格式 `Pretty` 和 `TabSeparated` 都可以使用 `IDataType` 接口的 `serializeTextEscaped` 辅助方法。

## 区块（Block）

区块用于在内存中表示表的一部分；它其实是个三元组：`(IColumn, IDataType, column name)`。查询在执行时，数据将以区块为单位处理。一个区块里面含有数据（在 `IColumn` 对象里面），用于告知我们如何处理这个列的类型信息（在 `IDataType`），还有列名（有可能来源于表的原始的列名，又或者人工命名的存储临时计算的结果的列名。

当我们需要基于某个区块里面的列去计算某些函数的值时，我们会加在这个区块中加入一个新列来存储计算结果；我们不需要用到原始列，因为所有这些操作都不会改变原始列的值（immutable）。然后，不需要的列会直接从区块中删除（而不是改变其值）。这会为消除常见的字表达式提供方便。

每部分的被处理的数据都会产生区块。请注意对于相同类型的计算，列名和列类型在不同的区块中用的是相同的对象，只是数据不同。 分开区块头和数据会给我们带来好处，因为不这样做的话，小的区块需要额外付出存储列名和共享指针（shared_ptrs）的代价。

## 区块流（Block Streams）

区块流用于处理数据。我们用区块流来从一个地方读数据，做数据变换，或者把数据写在另一个地方。其中 `IBlockInputStream` 有用于拿到下一个可读区块的 `read` 方法， `IBlockOutputStream` 有用于将区块写到某个地方的 `write` 方法。

区块流用于：

1. 读写一个表。该表返回一个用于读或者写区块的流。
2. 实现数据格式。比如，如果你想在终端以 `Pretty` 数据格式显示数据，你需要把区块推送到一个一个区块输出流，然后让该区块流格式化其数据。
3. 进行数据变换。假设你有一个 `IBlockInputStream`，并且你想筛选部分数据。你可以新建一个 `FilterBlockInputStream`，并用你的区块流来初始化这个区块流。这样当你从 `FilterBlockInputStream` 获取区块时，它会从你的区块流里面拿到区块，根据条件过滤一下，然后返回过滤后的区块给你。查询执行的流水线都以这样的方式表示。

我们有更复杂的数据变换。比如，当你从 `AggregatingBlockInputStream` 拿数据时，它会先从它的数据来源读取所有的数据，进行聚合，然后返回一个聚合过的流给你。另一个例子是 `UnionBlockInputStream`，它可以在构造函数中接受多个输入源和线程数，然后发起相应的线程并行地从输入源中读取数据。

> 区块流是用“拉”的方法来实现流程控制：当你从第一个流拉一个区块的时候，它会从它的上游的流将所需要的区块也拉下来，带动整个执行流水线。“拉”和“推”其实都不是最好的解决方案，因为这样的话控制流程就不能显式地表现出来，从而限制了很多特性的实现，其中包括多查询同步执行（合并多重流水线）。我们可以使用协程或者额外的互相等待的线程来克服这个限制，但如果我们能把控制流程显式表示，我们会有更多的可能性：比如说如果我们知道什么时候程序把数据从一个计算单位传给另一个计算单位的时候。更多的想法请参阅这篇 [很好的文章](http://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/)。
>
> 要注意的是，在每一步里，查询执行流水线都会创建临时的数据。为了能让临时的数据完整地放进 CPU 缓存，我们尽量会控制每个区块的大小足够小。如果这个前提成立的话，读写临时数据相对于其他计算操作所需的时间几乎可以忽略不计。另一种可能性是将同一流水线里的很多操作合并在一起，这样可以让流水线很短，因而不需要太多的临时数据。这有其好处，但也有一些不足。比如，一个分段的流水线可以很容易实现缓存运算中间的结果，同时“偷取”其它类似的查询的运算中间的结果，以及合并类似查询的流水线。

## 格式

数据格式是在区块流中实现的。其中有专门为展示用的给客户端的输出格式，比如 `Pretty` 格式, 只由 `IBlockOutputStream` 提供（译者注：`PrettyBlockOutputStream` 继承 `IBlockOutputStream`）. 也有一些同时支持输入格式和输出格式，比如 `TabSeparated` 和 `JSONEachRow`。

另外还有基于行的流 `IRowInputStream` 和 `IRowOutputStream`。它们允你许以每行为单位（而不是区块）来推/拉数据。事实上它们只是用来简化基于行的格式的实现：包装类 `BlockInputStreamFromRowInputStream` 和and `BlockOutputStreamFromRowOutputStream` 可以将基于行的流转化成正常的基于区块的流。

## I/O

抽象类 `ReadBuffer` 和 `WriteBuffer` 用于基于字节的 input/output，而不是原生的 C++ `iostream` 里面的相应的类。别担心，任何一个足够成熟的 C++ 项目都会有其充分的理由不去用 `iostream`。

`ReadBuffer` 和 `WriteBuffer` 本质上就是一段连续的缓冲区和一个指向缓冲区某个位置的指针。其实现有可能（也有可能不）管理缓冲区所占的内存的周期。`ReadBuffer` 有一个虚函数方法用于填充数据，`WriteBuffer` 有一个虚函数方法用于将缓冲区的内容清空并输出到另一个地方，不过这些虚函数方法很少被用到。

 `ReadBuffer` 和 `WriteBuffer` 的具体实现通常用于1）文件，文件描述符和网络套接字；2）压缩（`CompressedWriteBuffer` 初始化时需要用另外一个 WriteBuffer，进行压缩，然后才输出到其他地方；3）其他目的，比如 `ConcatReadBuffer`（连接多个缓冲区的读缓冲区）， `LimitReadBuffer`（有上限的读缓冲区）和 `HashingWriteBuffer`（哈希化写缓冲区）————类名应该很能说明其用途了。

`ReadBuffer` 和 `WriteBuffer` 假设输入输出都是字节。如果需要格式化的输入输出（比如，用十进制输出数字），可以使用 `ReadHelpers` 和 `WriteHelpers` 头文件里面的辅助函数。

让我们看一下当我们把结果写成 `JSON` 格式并输出到 stdout ，其中发生了什么。假设结果集存在某个 `IBlockInputStream`里。我们倒推一下整个写的过程：在最后一步我们需要一个 `WriteBufferFromFileDescriptor(STDOUT_FILENO)` 来将字节写到 stdout。为了生成 `JSONRow` 格式的字节，我们需要一个 `JSONRowOutputStream`，而这个流需要用上面这个 `WriteBuffer` 来初始化。由于 `JSONRowOutputStream` 是一个基于行的流，我们还需要继承了 `IBlockOutputStream` 的 `BlockOutputStreamFromRowOutputStream` 将本来基于区块的流转化为基于行的流。现在我们拥有一个 `IBlockInputStream` 和一个 `IBlockOutputStream`，我们可以调用 `copyData` 方法将 `IBlockInputStream` 里面的数据转移到 `IBlockOutputStream` 里面去。若要细究内部实现的话，`JSONRowOutputStream` 负责写不同的 JSON 分隔符，然后以 `IColumn` 引用和行数作为参数来调用方法 `IDataType::serializeTextJSON`。 接着，`IDataType::serializeTextJSON` 会根据不同的类型来调用 `WriteHelpers.h`里面的某个方法：比如遇到数字时调用 `writeText`，遇到字符串时调用 `writeJSONString`。

## 表

表在代码里用 `IStorage` 接口表示。不同的表引擎会使用不同的接口实现，比如 `StorageMergeTree`，`StorageMemory`，等等。这些类相应的对象其实都是表。

`IStorage` 最重要的方法是 `read` 和 `write`，其次还有其它方法，包括 `alter`，`rename`，`drop`，等等。`read` 方法包括一下参数：需要从表中读取的所有列，以 `AST`（抽象语法树）表示的查询语句，以及期望返回的流的数目。函数返回1）一个或多个 `IBlockInputStream` 对象；2）关于在查询执行途中在这个表引擎里完成了的数据处理的信息。

大多数情况下，`read` 只需要负责从一个表里面读取制定的列。查询解释器会负责剩下的数据处理。.

但也有几种值得注意的例外：
- 表引擎可以利用传进 `read` 方法里的 AST 查询推导出是否或者如何使用索引，进而减少从表里读出来的数据量；
- 有时在某一步中表引擎自己也可以自行处理数据。例如， `StorageDistributed` 可以发一个查询到别的服务器，让它们去处理数据，然后将返回的数据进行汇总，再将汇总后的数据返回，最后让查询解析器做收尾的工作。

前文已述表的 `read` 方法可以通过返回多个 `IBlockInputStream` 对象来并行处理数据。这多个区块输入流可以从一个表中并行地读入数据。然后你可以将每个流用多个变换包装起来（比如表达式计算，或者筛选结果）。这些流将会被独立地计算，然后可以被放在一个 `UnionBlockInputStream` 里面，以完成并行用多个流读数据的工作。

`TableFunction` 是指用于返回一个临时的 `IStorage` 对象的函数，用于 `FROM` 子语句。

如果你想知道一个表引擎的轮廓如何编写，可以看看简单的例子，比如 `StorageMemory` 和 `StorageTinyLog`。

> `IStorage` 的 `read` 方法返回 `QueryProcessingStage`，里面包含在这个存储器里查询里的哪些 parts 已经进行的计算。目前我们只提供粒度很低的信息，而没办法提供类似于“目前进度是这个 part 里面的某个范围里的数据已经处理好了”这种级别的信息。

## Parsers（解析器）

在 Clickhouse 里，每个查询都使用手写的递归下降解析器（recursive descent parser）来解析。比如，`ParserSelectQuery` 会递归地调用旗下的解析器去分别解析查询的每个部分。解析器返回的是 `AST`。`AST` 用节点表示，每个节点都是一个 `IAST`。

> 由于历史原因，我们没有使用解析器生成器。

## Interpreters（解释器）

解释器用于基于一个 `AST` 生成查询执行流水线。简单的解释器包括 `InterpreterExistsQuery` 和 `InterpreterDropQuery`，等等。复杂的如 `InterpreterSelectQuery`，等等。一个查询执行流水线包含一些区块输入流和区块输出流。例如，透过解释器来解释 `SELECT` 查询会得到用于读取结果的 `IBlockInputStream`，透过解释器解释 `INSERT` 查询会得到用于写结果的 `IBlockOutputStream`；透过解释器解释 `INSERT SELECT` 是一个返回空结果的 `IBlockInputStream`，但它同同时把数据从 `SELECT` 复制到 `INSERT` 里。

`InterpreterSelectQuery `使用了 `ExpressionAnalyzer` 和 `ExpressionActions` 来做查询分析和变换。大多数基于规则的查询优化都在这里进行。`ExpressionAnalyzer` 目前写得比较混乱，亟待重写：很多查询变换和优化都应在重构并提取到另外的类以达到模块化。

## 函数

函数分成两类：普通函数和聚合函数。关于聚合函数的介绍详见下一章节。

普通函数不会改变行数————它们看起来像是在独立地处理每一行，尽管事实上为了实现向量化查询执行，它们是以区块为单位被调用。

部分函数，包括 `blockSize`，`rowNumberInBlock`和`runningAccumulate`，使用时就会暴露了并不是按每一行来调用的这一事实。

ClickHouse 使用强类型，所以隐式类型转换不会发生。如果函数被传入不支持的的类型，会扔出一个异常。但同一个函数可以用重载来支持多种类型。例如，加法函数（相对应 `+` 这个运算符）对于所有的数值类型的搭配都适用：`UInt8` + `Float32`，`UInt16` + `Int8`，等等。有一些函数接受的参数数量是任意的，比如 `concat` 函数。

实现一个函数不算太方便，因为一个函数需要显式地分派支持的数据类型和 `IColumns`。举个例子，对于每种数值类型，常量和非常量，左值和右值，`plus` 函数都需要通过 C++ template 去生成相应的代码。

> 如果不想使用 template 的话，在这里使用运行时代码生成也是一个好的选择。这样做我们就可以使用上Fused Multiply-Add（积和熔加运算，见[FMA指令集](https://zh.wikipedia.org/wiki/FMA%E6%8C%87%E4%BB%A4%E9%9B%86)和[乘积累加运算](https://zh.wikipedia.org/wiki/%E4%B9%98%E7%A9%8D%E7%B4%AF%E5%8A%A0%E9%81%8B%E7%AE%97)），或者在同一个循环里面做多次比较。
>
> 由于我们使用了向量化查询执行，所有的函数并不会执行短路逻辑。举个例子，如果你写 `WHERE f(x) AND g(y)`，`f(x)` 和 `g(x)` 都会被计算，哪怕 `f(x)` 值为零的行（当 `f(x)` 是常数零食除外）。但如果 `f(x)` 可以去掉很多行，而且算 `f(x)` 需要的资源远少于 `g(y)`，实现一个多程计算的效果会更好：先算 `f(x)`，然后过滤不符合的列，然后才对剩下的少量数据计算 `g(y)`。

## 聚合函数Aggregate Functions

聚合函数都使用状态。它们将传过来的值汇总成一些状态，然后让你从那些状态里拿到你要的结果。它们可以用 `IAggregateFunction` 接口管理。状态可以很简单（`AggregateFunctionCount` 只包含一个 `UInt64` 值）或者很复杂（`AggregateFunctionUniqCombined` 包括一个一维数组，一个哈希表和一个 `HyperLogLog` 概率数据结构）。

当执行多层 `GROUP BY` 查询碰到需要处理多个状态的时候，状态储存在 `Arena` 里（一个内存池），或者在内存的其它合适的地方。状态可能有一个复杂的构造函数和析构函数————比如说，复杂的聚合状态可以自行安排占用更多的内存。这需要更仔细地创建和销毁状态，以及合理地传递状态的拥有权，并记录好谁什么时候要销毁这些状态。

当使用分布式查询需要在网络上传输，或者由于内存不足需要写到硬盘上时，聚合状态可以被序列化或者反序列化。聚合状态甚至可以用 `DataTypeAggregateFunction` 存在表中以便增量存储聚合数据。

> 聚合状态的序列化数据格式目前并没有保存版本号。如果这些数据只是暂时地保存的话，这种做法没有问题。但是我们有 `AggregatingMergeTree` 表引擎用于增量聚合，而且已经有人在生产环境使用了。所以以后我们如果要更改任何聚合函数的序列化格式，我们需要为其兼容以前的功能。

## Server

The server implements several different interfaces:
- An HTTP interface for any foreign clients.
- A TCP interface for the native ClickHouse client and for cross-server communication during distributed query execution.
- An interface for transferring data for replication.

Internally, it is just a basic multithreaded server without coroutines, fibers, etc. Since the server is not designed to process a high rate of simple queries but is intended to process a relatively low rate of complex queries, each of them can process a vast amount of data for analytics.

The server initializes the `Context` class with the necessary environment for query execution: the list of available databases, users and access rights, settings, clusters, the process list, the query log, and so on. This environment is used by interpreters.

We maintain full backward and forward compatibility for the server TCP protocol: old clients can talk to new servers and new clients can talk to old servers. But we don't want to maintain it eternally, and we are removing support for old versions after about one year.

> For all external applications, we recommend using the HTTP interface because it is simple and easy to use. The TCP protocol is more tightly linked to internal data structures: it uses an internal format for passing blocks of data and it uses custom framing for compressed data. We haven't released a C library for that protocol because it requires linking most of the ClickHouse codebase, which is not practical.

## Distributed query execution

Servers in a cluster setup are mostly independent. You can create a `Distributed` table on one or all servers in a cluster. The `Distributed` table does not store data itself – it only provides a "view" to all local tables on multiple nodes of a cluster. When you SELECT from a `Distributed` table, it rewrites that query, chooses remote nodes according to load balancing settings, and sends the query to them. The `Distributed` table requests remote servers to process a query just up to a stage where intermediate results from different servers can be merged. Then it receives the intermediate results and merges them. The distributed table tries to distribute as much work as possible to remote servers, and does not send much intermediate data over the network.

> Things become more complicated when you have subqueries in IN or JOIN clauses and each of them uses a `Distributed` table. We have different strategies for execution of these queries.
>
> There is no global query plan for distributed query execution. Each node has its own local query plan for its part of the job. We only have simple one-pass distributed query execution: we send queries for remote nodes and then merge the results. But this is not feasible for difficult queries with high cardinality GROUP BYs or with a large amount of temporary data for JOIN: in such cases, we need to "reshuffle" data between servers, which requires additional coordination. ClickHouse does not support that kind of query execution, and we need to work on it.

## Merge Tree

`MergeTree` is a family of storage engines that supports indexing by primary key. The primary key can be an arbitary tuple of columns or expressions. Data in a `MergeTree` table is stored in "parts". Each part stores data in the primary key order (data is ordered lexicographically by the primary key tuple). All the table columns are stored in separate `column.bin` files in these parts. The files consist of compressed blocks. Each block is usually from 64 KB to 1 MB of uncompressed data, depending on the average value size. The blocks consist of column values placed contiguously one after the other. Column values are in the same order for each column (the order is defined by the primary key), so when you iterate by many columns, you get values for the corresponding rows.

The primary key itself is "sparse". It doesn't address each single row, but only some ranges of data. A separate `primary.idx` file has the value of the primary key for each N-th row, where N is called `index_granularity` (usually, N = 8192). Also, for each column, we have `column.mrk` files with "marks," which are offsets to each N-th row in the data file. Each mark is a pair: the offset in the file to the beginning of the compressed block, and the offset in the decompressed block to the beginning of data. Usually compressed blocks are aligned by marks, and the offset in the decompressed block is zero. Data for `primary.idx` always resides in memory and data for `column.mrk` files is cached.

When we are going to read something from a part in `MergeTree`, we look at `primary.idx` data and locate ranges that could possibly contain requested data, then look at `column.mrk` data and calculate offsets for where to start reading those ranges. Because of sparseness, excess data may be read. ClickHouse is not suitable for a high load of simple point queries, because the entire range with `index_granularity` rows must be read for each key, and the entire compressed block must be decompressed for each column. We made the index sparse because we must be able to maintain trillions of rows per single server without noticeable memory consumption for the index. Also, because the primary key is sparse, it is not unique: it cannot check the existence of the key in the table at INSERT time. You could have many rows with the same key in a table.

When you `INSERT` a bunch of data into `MergeTree`, that bunch is sorted by primary key order and forms a new part. To keep the number of parts relatively low, there are background threads that periodically select some parts and merge them to a single sorted part. That's why it is called `MergeTree`. Of course, merging leads to "write amplification". All parts are immutable: they are only created and deleted, but not modified. When SELECT is run, it holds a snapshot of the table (a set of parts). After merging, we also keep old parts for some time to make recovery after failure easier, so if we see that some merged part is probably broken, we can replace it with its source parts.

`MergeTree` is not an LSM tree because it doesn't contain "memtable" and "log": inserted data is written directly to the filesystem. This makes it suitable only to INSERT data in batches, not by individual row and not very frequently – about once per second is ok, but a thousand times a second is not. We did it this way for simplicity's sake, and because we are already inserting data in batches in our applications.

> MergeTree tables can only have one (primary) index: there aren't any secondary indices. It would be nice to allow multiple physical representations under one logical table, for example, to store data in more than one physical order or even to allow representations with pre-aggregated data along with original data.
>
> There are MergeTree engines that are doing additional work during background merges. Examples are `CollapsingMergeTree` and `AggregatingMergeTree`. This could be treated as special support for updates. Keep in mind that these are not real updates because users usually have no control over the time when background merges will be executed, and data in a `MergeTree` table is almost always stored in more than one part, not in completely merged form.

## Replication

Replication in ClickHouse is implemented on a per-table basis. You could have some replicated and some non-replicated tables on the same server. You could also have tables replicated in different ways, such as one table with two-factor replication and another with three-factor.

Replication is implemented in the `ReplicatedMergeTree` storage engine. The path in `ZooKeeper` is specified as a parameter for the storage engine. All tables with the same path in `ZooKeeper` become replicas of each other: they synchronise their data and maintain consistency. Replicas can be added and removed dynamically simply by creating or dropping a table.

Replication uses an asynchronous multi-master scheme. You can insert data into any replica that has a session with `ZooKeeper`, and data is replicated to all other replicas asynchronously. Because ClickHouse doesn't support UPDATEs, replication is conflict-free. As there is no quorum acknowledgment of inserts, just-inserted data might be lost if one node fails.

Metadata for replication is stored in ZooKeeper. There is a replication log that lists what actions to do. Actions are: get part; merge parts; drop partition, etc. Each replica copies the replication log to its queue and then executes the actions from the queue. For example, on insertion, the "get part" action is created in the log, and every replica downloads that part. Merges are coordinated between replicas to get byte-identical results. All parts are merged in the same way on all replicas. To achieve this, one replica is elected as the leader, and that replica initiates merges and writes "merge parts" actions to the log.

Replication is physical: only compressed parts are transferred between nodes, not queries. To lower the network cost (to avoid network amplification), merges are processed on each replica independently in most cases. Large merged parts are sent over the network only in cases of significant replication lag.

In addition, each replica stores its state in ZooKeeper as the set of parts and its checksums. When the state on the local filesystem diverges from the reference state in ZooKeeper, the replica restores its consistency by downloading missing and broken parts from other replicas. When there is some unexpected or broken data in the local filesystem, ClickHouse does not remove it, but moves it to a separate directory and forgets it.

> The ClickHouse cluster consists of independent shards, and each shard consists of replicas. The cluster is not elastic, so after adding a new shard, data is not rebalanced between shards automatically. Instead, the cluster load will be uneven. This implementation gives you more control, and it is fine for relatively small clusters such as tens of nodes. But for clusters with hundreds of nodes that we are using in production, this approach becomes a significant drawback. We should implement a table engine that will span its data across the cluster with dynamically replicated regions that could be split and balanced between clusters automatically.
