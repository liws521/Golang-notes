
[Golang从入门到微服务](https://www.bilibili.com/video/BV1Sg411T7TV?p=1)


- 工程管理
    - src
    - bin
    - pkg, 依赖包
- GOOS, GOARCH, 编译时指定操作系统和架构
- 在Goland里把Terminal配置成bash, file-setting-tools-terminal, 把shell path改为git的bin/bash.exe
- GOBIN, 指定go install时的目录
- go env


## 网络编程
- [网络编程看这个](https://www.topgoer.com/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/)
### 网络分层
- OSI七层网络模型, 物理层, 数据链路层, 网络层, 传输层, 会话层, 表示层, 应用层
- TCP/IP四层, 数据链路层, 网络层, 传输层, 应用层
- 传输层TCP/UDP, 应用层HTTP
### socket编程
- Client
    - dial, 指定IP:Port
    - 写数据
    - 读取Server响应
    - 关闭连接
- Server
    - listen, 设置监听的IP:Port
    - Accept, 接收连接
    - 读取Client发来的数据
    - 处理数据
    - 把处理后的结果写回客户端
    - 关闭连接
### http
- Web中用的, java, php, go(beego/gin), python
- C++不写Web, 所以对http协议不熟
- http是应用层协议, 底层还是依赖传输层(短连接tcp), 网络层(ip)
- 无状态的, 每一次请求都是独立的, 下次请求需要重新建立连接
- http是标准协议, https不是标准协议, https = http + ssl(非对称加密, 数字证书)
- http相当于明文传输, https加密了, 安全一些
---
- http请求报文格式
- 请求行, 方法 + URL + 协议版本号
    - 请求方法, GET获取数据, POST上传数据(比如表单/json格式), PUT修改数据, DELETE删除数据
    - url中也可以放数据, url?name=liws&age=18, ?分割url和参数, 参数之间用&分割
- 请求头
    - key: value格式
    - 可以写很多, 协议自带的或者自定义的都可
- 空行
- 请求包体(可选的)
- Postman软件, 模拟客户端向服务端发送各种请求
---
- http响应体
- 状态行, 协议格式 + 状态码(404) + 状态描述
    - 1xx 客户端可以继续发送请求
    - 2xx 正常访问
    - 3xx 重定向
    - 4xx
        - 401 未授权
        - 404 Not found
    - 5xx
        - 501 Internal Error (服务器内部错误)
- 响应头
- 空行
- 响应包体