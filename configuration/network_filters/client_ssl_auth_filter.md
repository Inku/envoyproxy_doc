# 客户端TLS身份认证

* 客户端TLS验证过滤器[架构概述](TODO:)
* [v1 API参考](TODO:)
* [v2 API参考](TODO:)

## 统计

每个客户端TLS身份认证过滤器都有一个以*auth.clientssl.<stat_prefix\>.* 为根的统计树，以下是统计数据:

|名称|类型|描述|  
|----|---|---|
|update_success|计数器|主要更新成功总数|
|update_failure|计数器|主要更新失败总数|
|auth_no_ssl|计数器|由于没有TLS而忽略的总连接数|
|auth_ip_white_list|计数器|由于IP白名单而允许的总连接数|
|auth_digest_match|计数器|由于证书匹配而允许的总连接数|
|auth_digest_no_match|计数器|由于证书不匹配而拒绝的总连接数|
|total_principals|计量器|总负载主体|

## REST API

`GET /v1/certs/list/approved`

身份验证过滤器将在每个刷新间隔调用此API以获取已认证的证书/主体的当前列表。预期的JSON响应如下所示：

```json
{
  "certificates": []
}
```

> certificates  <br/>
    *(必填，数组）* 认证的证书/主体列表。

每个证书对象定义为：

```json
{
  "fingerprint_sha256": "...",
}
```

> fingerprint_sha256  <br/>
     *(必填，字符串）* 已认证的客户端证书的SHA256哈希值。Envoy会将此哈希值与提供的客户端证书相匹配，以确定是否存在摘要匹配。