# Mongo代理

* MongoDB[架构概述](TODO:)
* [v1 API参考](TODO:)
* [v2 API参考](TODO:)

## 故障注入

Mongo代理过滤器支持故障注入。有关如何配置，请参阅v1和v2 API参考。

## 统计

每个配置的MongoDB代理筛选器都具有以*mongo.<stat_prefix\>.*为根的统计信息。以下为统计数据：

|名称|类型|描述|  
|----|---|---|
|decoding_error|计数器|MongoDB协议解码错误数|
|delay_injected|计数器|注入延迟数|
|op_get_more|计数器|OP_GET_MORE消息数|
|op_insert|计数器|OP_INSERT消息数|
|op_kill_cursors|计数器|OP_KILL_CURSORS消息数|
|op_query|计数器|OP_QUERY消息数|
|op_query_tailable_cursor|计数器|设置了tailable游标标志的OP_QUERY消息数|
|op_query_no_cursor_timeout|计数器|未设置游标超时标志的OP_QUERY消息数|
|op_query_await_data|计数器|设置了等待数据标志的OP_QUERY消息数|
|op_query_exhaust|计数器|设置耗尽标志的OP_QUERY消息数|
|op_query_no_max_time|计数器|未设置maxTimeMS的查询数|
|op_query_scatter_get|计数器|分散获取的查询数|
|op_query_multi_get|计数器|多次获取的查询数|
|op_query_active|计量器|当前活动查询数|
|op_reply|计数器|OP_REPLY消息数|
|op_reply_cursor_not_found|计数器|设置了游标未找到标志的OP_REPLY消息数|
|op_reply_query_failure|计数器|设置了请求失败标志的OP_REPLY消息数|
|op_reply_valid_cursor|计数器|设置了有效游标的OP_REPLY消息数|
|cx_destroy_local_with_active_rq|计数器|活动的查询销毁的本地连接数|
|cx_destroy_remote_with_active_rq|计数器|活动的查询销毁的远程连接数|
|cx_drain_close|计数器|在服务器耗尽期间，连接在响应后正常关闭总数|

### 分散获取

Envoy将 *分散获取* 定义为不使用 *_id* 字段作为查询参数的查询 。Envoy同时查询顶级文档以及 *_id* 的 *$query* 字段。

### 批量获取

Envoy将 *批量获取* 定义为使用 *_id* 字段作为查询参数的查询，但其中 *_id*不是标量值（即文档或数组）。 Envoy同时查询顶级文档以及 *_id* 的 *$query* 字段。

### $comment解析

如果查询具有顶级 *$comment* 字段（通常在 *$query* 字段之外），Envoy会将其解析为JSON并查找以下结构：

```json
{
  "callingFunction": "..."
}
```

> callingFunction  <br/>
    *(必填，字符串）* 进行查询的函数。如果可用，该功能将用于[调用点](#按集合和调用点查询统计)查询统计信息。

### 按命令统计

MongoDB过滤器将收集 *mongo.<stat_prefix\>.cmd.<cmd\>.*命名空间下的命令的统计信息。

|名称|类型|描述|  
|----|---|---|
|total|计数器|命令总数|
|reply_num_docs|直方图|回复的文件数量|
|reply_size|直方图|回复的大小（以字节为单位）|
|reply_time_ms|直方图|命令时间（以毫秒为单位）|

### 按集合查询统计

MongoDB过滤器将收集 *mongo.<stat_prefix\>.collection.<collection\>.query.* 命名空间下的命令的统计信息。

|名称|类型|描述|  
|----|---|---|
|total|计数器|命令总数|
|scatter_get|计数器|分散获取总数|
|multi_get|计数器|批量获取总数|
|reply_num_docs|直方图|回复的文件数量|
|reply_size|直方图|回复的大小（以字节为单位）|
|reply_time_ms|直方图|命令时间（以毫秒为单位）|

### 按集合和调用点查询统计

如果应用程序在 *[$comment](#$comment解析)* 字段中提供调用函数，Envoy将生成每个调用点统计信息。这些统计信息与[按集合查询统计](#按集合查询统计)的一致，但是收集在 *mongo.<stat_prefix\>.collection.<collection\>.callsite.<callsite\>.query.* 命名空间下。

## 运行时

Mongo代理过滤器支持以下运行时配置：

**mongo.connection_logging_enabled**  <br/>
将启用日志记录的连接百分比，默认为100。这样只允许百分之几的连接具有日志记录，但是这些连接上的所有消息都会被记录。

**mongo.proxy_enabled**  <br/>
启用代理的连接百分比。默认为100。

**mongo.logging_enabled**  <br/>
将被记录日志的消息的百分比。默认为100.如果小于100，则查询可能不会回复日志记录信息。

**mongo.mongo.drain_close_enabled**  <br/>
如果服务器正在耗尽状态则将将会耗尽的链接关闭，否则将尝试关闭耗尽状态。默认为100。

**mongo.fault.fixed_delay.percent**  <br/>
当没有活动故障时，符合条件的MongoDB操作受注入故障影响的概率。默认为配置中指定的百分比。

**mongo.fault.fixed_delay.duration_ms**  <br/>
延迟持续时间（以毫秒为单位）。默认为config中指定的*duration_ms*。

## 访问日志格式

访问日志格式不可自定义，并具有以下格式：

```json
{"time": "...", "message": "...", "upstream_host": "..."}
```

**time**  <br/>
解析完整消息的系统时间(毫秒)。

**message**  <br/>
消息的文本扩展。消息是否完全展开取决于上下文。有时会显示摘要数据以避免极大的日志大小。

**upstream_host**  <br/>
连接所代理的上游主机（如果可用）。如果过滤器与[TCP代理过滤器](tcp_proxy_filter.md)一起使用，则会填充此值。