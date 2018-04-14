# RowBinary

以二进制格式逐行格式化和解析数据。行和值连续列出，没有分隔符。
这种格式比 Native 格式效率低，因为它是基于行的。

整数使用固定长度的小端表示法。 例如，UInt64 使用8个字节。
DateTime 被表示为 UInt32 类型的Unix 时间戳值。
Date 被表示为 UInt16 对象，它的值为 1970-01-01以来的天数。
字符串表示为 varint 长度（无符号[LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟字符串的字节数。
FixedString 被简单地表示为一个字节序列。

数组表示为 varint 长度（无符号[LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟有序的数组元素。

