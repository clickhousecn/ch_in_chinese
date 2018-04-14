<a name="format_capnproto"></a>

# CapnProto

Cap'n Proto 是一种二进制消息格式，类似 Protocol Buffers 和 Thriftis，但与 JSON 或 MessagePack 格式不一样。

Cap'n Proto 消息格式是严格类型的，而不是自我描述，这意味着它们不需要外部的描述。这种格式可以实时地应用，并针对每个查询进行缓存。


```sql
SELECT SearchPhrase, count() AS c FROM test.hits
       GROUP BY SearchPhrase FORMAT CapnProto SETTINGS schema = 'schema:Message'
```

其中 `schema.capnp` 描述如下：

```
struct Message {
  SearchPhrase @0 :Text;
  c @1 :Uint64;
}
```


格式文件存储的目录可以在服务配置中的[ format_schema_path ](../operations/server_settings/settings.md#server_settings-format_schema_path) 指定。

Cap'n Proto 反序列化是很高效的，通常不会增加系统的负载。

