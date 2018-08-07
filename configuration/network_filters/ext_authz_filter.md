# 外部授权

* 外部授权[架构概述](TODO:)
* [网络过滤器v2 API参考](TODO:)

外部授权网络过滤器调用外部授权服务以检查传入请求是否已获得授权。如果网络过滤器认为该请求未经授权，则将关闭该连接。

```
注：
建议首先在过滤器链中配置此过滤器，以便在处理请求的其余过滤器之前授权请求。
```

传递给授权服务的请求内容由[CheckRequest](TODO:)指定。

网络过滤器gRPC服务可以配置如下。您可以在[网络过滤器](TODO:)中查看所有配置选项。

## 示例

样例过滤器配置如下：

```yaml
filters:
  - name: envoy.ext_authz
    stat_prefix: ext_authz
    grpc_service:
      envoy_grpc:
        cluster_name: ext-authz

clusters:
  - name: ext-authz
    type: static
    http2_protocol_options: {}
    hosts:
      - socket_address: { address: 127.0.0.1, port_value: 10003 }
```

## 统计

网络过滤器的输出统计信息在*config.ext_authz.*的命名空间中。

|名称|类型|描述|  
|----|---|---|
|total|计数器|过滤器的总响应数|
|error|计数器|联系外部服务的错误总数|
|denied|计数器|来自授权服务的拒绝流量的总响应|
|failure_mode_allow|计数器|由于failure_mode_allow设置为true，因错误而被允许通过的请求总数|
|ok|计数器|授权服务允许流量的总响应|
|cx_closed|计数器|已关闭的总连接数|
|active|计量器|传输到授权服务的当前活动请求总数|