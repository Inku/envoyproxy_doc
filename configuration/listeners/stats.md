# 统计

## 监听器

每个监听器都有一个以*listener.<address\>.* 为根的统计树，以下是统计数据:

|名称|类型|描述|  
|----|---|---|
|downstream_cx_total|计数器|总连接数|
|downstream_cx_destroy|计数器|总销毁链接输|
|downstream_cx_active|计量器|当前活跃连接数|
|downstream_cx_length_ms|直方图|链接时长（毫秒）|
|no_filter_chain_match|计数器|不匹配任何过滤器链的链接总数|
|ssl.connection_error|计数器|总TLS连接错误数，不包括证书验证失败|
|ssl.handshake|计数器|成功完成TLS链接握手总数|
|ssl.session_reused|计数器|恢复TLS会话成功总数|
|ssl.no_certificate|计数器|没有客户端证书的成功TLS连接总数|
|ssl.fail_verify_no_cert|计数器|由于缺少客户端证书而失败的TLS连接总数|
|ssl.fail_verify_error|计数器|CA验证失败的TLS连接总数|
|ssl.fail_verify_san|计数器|SAN验证失败的TLS连接总数|
|ssl.fail_verify_cert_hash|计数器|证书固定验证失败的TLS连接总数|
|ssl.cipher.<cipher\>|计数器|使用<cipher\>的TLS连接总数|

## 监听管理器

监听管理器有一个以*listener_manager.* 为根的统计树，以下是统计数据。统计名称中的所有`:`替换为`_`。

|名称|类型|描述|  
|----|---|---|
|listener_added|计数器|添加的监听器总数（通过静态配置或LDS）|
|listener_modified|计数器|修改的监听器总数（通过LDS）|
|listener_removed|计数器|移除的监听器总数（通过LDS）|
|listener_create_success|计数器|成功添加至工作进程的监听器总数|
|listener_create_failure|计数器|添加至工作进程失败的监听器总数|
|total_listeners_warming|计量器|当前警告状态监听器数量|
|total_listeners_active|计量器|当前活跃状态监听器数量|
|total_listeners_draining|计量器|当前耗尽的监听器数量|
