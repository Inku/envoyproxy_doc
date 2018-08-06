# 开始使用

本节将为您提供一个非常简单的配置，并提供一些示例配置。

Envoy目前不提供单独的预构建二进制文件，但是提供了Docker镜像，这是开始使用Envoy的最快方法。如果您希望在Docker容器之外使用Envoy，您需要[构建它](TODO:)。

这些示例使用v2 Envoy API，但仅使用了最有效切需求最简单的静态配置功能。对于更复杂的要求你需要[动态配置](TODO:)的支持。

## 快速入门运行简单示例

这些指令从Envoy仓库中的文件运行。以下部分提供了有关相同配置的配置文件和执行步骤的更详细说明。

[configs/google_com_proxy.v2.yaml](https://github.com/envoyproxy/envoy/blob/master/configs/google_com_proxy.v2.yaml)中提供了一个最小化Envoy配置，可用于验证基本的纯HTTP代理。但这并不代表一个现实的Enovy部署实例：

```
$ docker pull envoyproxy/envoy:v1.7.0
$ docker run --rm -d -p 10000:10000 envoyproxy/envoy:v1.7.0
$ curl -v localhost:10000
```

使用的Docker镜像包含1.7.0版本的Envoy和基本的Envoy配置。此基本配置让Envoy将传入的请求路由到*.google.com。

```
注：
而由于大陆无法访问google可能会返回`upstream connect error or disconnect/reset before headers`
并不是配置问题，可以暂时忽略继续向下阅读。
```

## 简易配置

可以使用在命令行中作为参数传入的单个YAML文件来配置Envoy。

配置管理服务器需要[admin message](TODO:)。使用*address*字段指定侦听地址，在本例中是0.0.0.0:9901。

```
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```

[static_resources](TODO:)包含了Envoy启动时所需要的所有静态配置，而不是在Envoy运行时进行动态资源配置。[v2 API概述](TODO:)描述了这一点。

```
static_resources:
```

以下是[listeners](TODO:)示例:

```
listeners:
- name: listener_0
  address:
    socket_address: { address: 0.0.0.0, port_value: 10000 }
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
              route: { host_rewrite: www.google.com, cluster: service_google }
        http_filters:
        - name: envoy.router
```

以下是[clusters](TODO:)示例:

```
clusters:
- name: service_google
  connect_timeout: 0.25s
  type: LOGICAL_DNS
  # Comment out the following line to test on v6 networks
  dns_lookup_family: V4_ONLY
  lb_policy: ROUND_ROBIN
  hosts: [{ socket_address: { address: google.com, port_value: 443 }}]
  tls_context: { sni: www.google.com }
```

## 使用Envoy Docker镜像
使用您本地定义的envoy.yaml(如上所述)创建一个简易的Dockerfile来启动Envoy.您可以参考[命令行选项](TODO:)

```
FROM envoyproxy/envoy:latest
COPY envoy.yaml /etc/envoy/envoy.yaml
```

使用以下命令构建运行您的配置的Docker镜像:

```
$ docker build -t envoy:v1 .
```

之后你可以使用以下命令来启动:

```
$ docker run -d --name envoy -p 9901:9901 -p 10000:10000 envoy:v1
```

最后启动测试:

```
$ curl -v localhost:10000
```

如果你想通过docker-compose来使用envoy，你可以使用数据卷来重写提供配置文件。

## 沙箱

我们使用Docker Compose创建了许多沙箱，它们配置了不同的环境来测试Envoy的功能并显示样本配置。 我们会随着大家的兴趣来增加更多展示不同功能的沙箱。 目前以下沙箱可供使用：

* [Front Proxy](TODO:)
* [Zipkin Tracing](TODO:)
* [Jaeger Tracing](TODO:)
* [Jaeger Native Tracing](TODO:)
* [gRPC Bridge](TODO:)

## 其他用例

除了代理本身，Envoy还捆绑为几个针对特定用例的开源发行版的一部分。

* [Envoy作为Kubernetes的API网关](TODO:)
