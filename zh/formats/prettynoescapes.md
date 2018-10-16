# PrettyNoEscapes

与 `Pretty` 格式不一样的是，它不使用 ANSI 字符转义， 这在浏览器显示数据以及在使用 `watch` 命令行工具是有必要的。

示例：

```bash
watch -n1 "clickhouse-client --query='SELECT * FROM system.events FORMAT PrettyCompactNoEscapes'"
```

您可以使用 HTTP 接口来获取数据，显示在浏览器中。

## PrettyCompactNoEscapes

用法类比上述。

## PrettySpaceNoEscapes

用法类比上述。

