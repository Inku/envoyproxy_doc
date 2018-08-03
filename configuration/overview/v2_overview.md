# 概述（v2 API）

Envoy v2 APIs在[数据层API](https://github.com/envoyproxy/data-plane-api/blob/master/README.md)存储定义为[proto3](https://developers.google.com/protocol-buffers/docs/proto3) [Protocol Buffers](https://developers.google.com/protocol-buffers/)。他在发展了现有的[v1 APIs及思想](v1_overview.md)来支持了：

* 通过gRPC流式传输[xDS](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)API更新。 这减少了资源需求并可以降低更新延迟。
* 一种新的REST-JSON API，其中JSON/YAML格式通过[proto3规范JSON mapping](https://developers.google.com/protocol-buffers/docs/proto3#json)派生。
* 通过文件系统，REST-JSON或gRPC终端进行更新。
* 通过扩展终端分配API以及负载和资源利用率报告到管理服务器来实现高级负载均衡。
* 当需要更强的[一致性与排队状态](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)。v2 API也维护极限数据的最终一致性模型。

有关Envoy与管理服务器之间v2消息交换方面的更多详细信息，请参阅xDS协议说明。

## 引导配置

要使用v2 API，必须提供引导配置文件。这提供了静态服务器配置，并配置Envoy以在[需要时访问动态配置](//TODO)。与v1 JSON/YAML配置一样，这是通过`-c`参数在命令行上提供，即：

```
./envoy -c <path to config>.{json,yaml,pb,pb_text}
```

扩展名反映了基础v2配置表达。

[Bootstrap](//TODO)消息是配置的根。[Bootstrap](//TODO)消息中的一个关键概念是静态和动态资源之间的区别。诸如[Listener](//TODO)或[Cluster](//TODO)之类的资源可以静态地配置在[static_resources](//TODO)中，其他具有xDS服务，例如[LDS](//TODO)或[CDS](//TODO)需要配置在[dynamic_resources](//TODO)中。

## 示例

下面我们将使用YAML配置来运行一个将HTTP从127.0.0.1:10000代理到127.0.0.2:1234的服务示例。

### 静态配置

下面提供了一个最小化的完整静态引导程序配置：

```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1234
```

### 静态配置配合动态EDS

基于上述配置，通过在127.0.0.3:5678上监听的[EDS](//TODO) gRPC管理服务器实现[动态终端发现](//TODO)的配置如下:

```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    lb_policy: ROUND_ROBIN
    type: EDS
    eds_cluster_config:
      eds_config:
        api_config_source:
          api_type: GRPC
          grpc_services:
            envoy_grpc:
              cluster_name: xds_cluster
  - name: xds_cluster
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5678
```

请注意，上述*xds_cluster*被定义为将Envoy指向管理服务器。 即使在完全动态的配置中，也需要定义一些静态资源以将Envoy指向其xDS管理服务器。

在上面的示例中，EDS管理服务器可以返回[DiscoveryResponse](//TODO)的原型编码：

```
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
  cluster_name: some_service
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 127.0.0.2
            port_value: 1234
```

上述版本控制和type URL scheme在[流式gRPC订阅协议](//TODO)文档中有更详细的说明。

### 动态配置

完全动态的引导程序配置，其中通过xDS动态发现属于管理服务器的资源以外的所有资源，如下所示：

```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

dynamic_resources:
  lds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        envoy_grpc:
          cluster_name: xds_cluster
  cds_config:
    api_config_source:
      api_type: GRPC
      grpc_services:
        envoy_grpc:
          cluster_name: xds_cluster

static_resources:
  clusters:
  - name: xds_cluster
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5678
```

管理服务器可以使用以下配置响应LDS请求：

```
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.api.v2.Listener
  name: listener_0
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 10000
  filter_chains:
  - filters:
    - name: envoy.http_connection_manager
      config:
        stat_prefix: ingress_http
        codec_type: AUTO
        rds:
          route_config_name: local_route
          config_source:
            api_config_source:
              api_type: GRPC
              grpc_services:
                envoy_grpc:
                  cluster_name: xds_cluster
        http_filters:
        - name: envoy.router
```

管理服务器可以使用以下配置响应RDS请求：

```
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.api.v2.RouteConfiguration
  name: local_route
  virtual_hosts:
  - name: local_service
    domains: ["*"]
    routes:
    - match: { prefix: "/" }
      route: { cluster: some_service }
```

管理服务器可以使用以下配置响应CDS请求：

```
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.api.v2.Cluster
  name: some_service
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    eds_config:
      api_config_source:
        api_type: GRPC
        grpc_services:
          envoy_grpc:
            cluster_name: xds_cluster
```

管理服务器可以使用以下配置响应EDS请求：

```
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
  cluster_name: some_service
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 127.0.0.2
            port_value: 1234
```

## 从v1配置升级

虽然可以编写新的v2引导程序JSON/YAML，但将现有的[v1 JSON/YAML配置](//TODO)升级到v2是更方便的。要执行此操作（在Envoy源代码树中），您可以运行：

```
bazel run //tools:v1_to_bootstrap <path to v1 JSON/YAML configuration file>
```

## 管理服务器

v2 xDS管理服务器将根据gRPC和/或REST服务的要求实现以下终端。在流式gRPC和REST-JSON情况下，可以根据[xDS协议](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)发送[DiscoveryRequest]()并接收[DiscoveryResponse](//TODO)。

### gRPC流式传输终端

`POST /envoy.api.v2.ClusterDiscoveryService/StreamClusters`

有关服务定义，请参阅[cds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/cds.proto)。当Envoy作为客户端使用时将

```
cds_config:
  api_config_source:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: some_xds_cluster
```

配置到[Bootstrap]()的[dynamic_resources]()中。

`POST /envoy.api.v2.EndpointDiscoveryService/StreamEndpoints`

有关服务定义，请参阅[eds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/eds.proto)。当Envoy作为客户端使用时将

```
cds_config:
  api_config_source:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: some_xds_cluster
```

配置到[Cluster]()的[eds_cluster_config]()字段。

`POST /envoy.api.v2.ListenerDiscoveryService/StreamListeners`

有关服务定义，请参阅[lds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/lds.proto)。当Envoy作为客户端使用时将

```
lds_config:
  api_config_source:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: some_xds_cluster
```

配置到[Bootstrap]()的[dynamic_resources]()中。

`POST /envoy.api.v2.RouteDiscoveryService/StreamRoutes`

有关服务定义，请参阅[rds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/rds.proto)。当Envoy作为客户端使用时将

```
route_config_name: some_route_name
config_source:
  api_config_source:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: some_xds_cluster
```

配置到[HttpConnectionManager]()的[rds]()字段。

### REST终端

`POST /v2/discovery:clusters`

有关服务定义，请参阅[cds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/cds.proto)。当Envoy作为客户端使用时将

```
cds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

配置到[Bootstrap]()的[dynamic_resources]()中。

`POST /v2/discovery:endpoints`

有关服务定义，请参阅[eds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/eds.proto)。当Envoy作为客户端使用时将

```
eds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

配置到[Cluster]()的[eds_cluster_config]()字段。

`POST /v2/discovery:listeners`

有关服务定义，请参阅[lds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/lds.proto)。当Envoy作为客户端使用时将

```
lds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

配置到[Bootstrap]()的[dynamic_resources]()中。

`POST /v2/discovery:routes`

有关服务定义，请参阅[rds.proto](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/rds.proto)。当Envoy作为客户端使用时将

```
route_config_name: some_route_name
config_source:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

配置到[HttpConnectionManager]()的[rds]()字段。

## 聚合发现服务

虽然Envoy从根本上采用了最终的一致性模型，但ADS提供了对API更新推送进行排序的机会，并确保单个管理服务器对Envoy节点的API更新具有亲和力。ADS允许管理服务器在单个双向gRPC流上传递一个或多个API及其资源。如果没有这个，一些API（如RDS和EDS）可能需要管理多个流并连接到不同的管理服务器。

ADS将允许通过适当的顺序进行无中断的配置更新。例如，假设foo.com已映射到集群X.我们希望将路由表中的foo.com映射改为在集群Y。为此，一个CDS/EDS更新必须首先同时包含两个集群X和Y。

如果没有ADS，CDS/EDS/RDS流可能指向不同的管理服务器，或者位于需要协调的不同gRPC流/连接的同一管理服务器上。EDS资源请求可能会拆分为两个不同的流，一个用于X，一个用于Y。ADS可以将这些流合并到单个流并连接到单个管理服务器，避免了更新时需要进行分布式同步保证正确序列。使用ADS，管理服务器将在单个流上提供CDS，EDS和RDS更新。

ADS仅适用于gRPC流（不支持REST），本文档对此进行了更全面的描述。gRPC终端：

`POST /envoy.api.v2.AggregatedDiscoveryService/StreamAggregatedResources`

有关服务定义，请参阅[discovery.proto ](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/discovery.proto)。当Envoy作为客户端使用时将

```
ads_config:
  api_type: GRPC
  grpc_services:
    envoy_grpc:
      cluster_name: some_ads_cluster
```

配置到[Bootstrap]()的[dynamic_resources]()中。

设置此项后，可以将上述任何配置源设置为使用ADS通道。 例如，可以更改LDS配置，从

```
lds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

修改至

```
lds_config: {ads: {}}
```

这意味着LDS流将通过共享ADS通道定向到some_ads_cluster。

## 管理服务不可达

当Envoy实例失去与管理服务器的连接时，Envoy将锁定先前的配置，同时在后台主动重试以重新建立与管理服务器的连接。

Envoy debug日志会记录每次尝试连接时都无法与管理服务器建立连接详情。

[upstream_cx_connect_fail](//TODO)是当前集群指向管理服务器的集群及统计服务器。它提供了监控此行为的信号。

## 状态

除非另有说明，否则[v2 API参考](//TODO)中描述的所有功能都可以被实现。在v2 API参考和[v2 API仓库](//TODO)中，除非它们被标记为*草稿* 或*实验* ，否则所有原型状态都是*冻结* 。在这里，*冻结* 意味着我们不可以破坏线性格式兼容性。

*冻结* 的原型可以通过[向下兼容](https://developers.google.com/protocol-buffers/docs/overview#how-do-they-work)的方式来进行扩展，例如添加新字段。上述protos中的字段可能会在不再需要其相关功能时进行废弃或[改变变更策略](https://github.com/envoyproxy/envoy/blob/master//CONTRIBUTING.md#breaking-change-policy)。虽然冻结的API保留了其线性格式兼容性，但我们保留更改原型名称空间，文件位置和嵌套关系的权利，这可能会导致代码更改。我们的目标是尽量减少客户流失。

原型被标记位*草稿* ，意味着它们接近最终确定，可能少部分在Envoy中被应用，但可能有在标记位冻结 前破坏线性格式。

原型被标记位*实验* ，与草稿原型有着同样的警告，并且在Envoy应用和标记为冻结之前有大概率修改。

[此处](https://github.com/envoyproxy/envoy/issues?q=is%3Aopen+is%3Aissue+label%3A%22v2+API%22)跟踪当前开放的v2 API issues。
