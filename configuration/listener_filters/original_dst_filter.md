# 原始目标过滤器

原始目标监听器过滤器在iptables REDIRECT目标重定向连接时，或者通过iptables TPROXY目标结合设置监听器的[透明](TODO:)选项来读取SO_ORIGINAL_DST socket选项。随后在Envoy的处理中将还原的目标地址视为连接的本地地址，而不是监听器正在监听的地址。此外，[原始目标集群](TODO:)可用于将HTTP请求或TCP连接转发到还原的目标地址。

* [v2 API参考](TODO:)
