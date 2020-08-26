# nginx 499
nginx 499 表示客户端已经关闭连接
出现这个问题只有两种情况：
- 服务端超时未响应，客户端超时了  （排查服务端为什么一直不响应了，检查是否有慢查询或者网络是否有问题）
- nginx 代理端主动关闭了客户端连接

nginx 什么时候会主动关闭客户端连接：客户端连接 keepalive 超时了
客户端 keepalive 参数
keepalive_timeout：客户端连接最多被保持多长时间，如果超过该时间则会断开连接
keepalive_request: 每个连接处理多少个请求，就会被强制关闭连接

服务端 keepalive 参数
keepalive：与服务端保持的空闲连接最大为 keepalive 数目，空闲超过该数目就会关闭