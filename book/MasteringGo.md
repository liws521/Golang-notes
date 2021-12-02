# Meta
# Text
- [Mastering Go](https://books.studygolang.com/Mastering_Go_ZH_CN/)



## 网络编程基础
### 关于net/http, net和http.RoundTripper
- 本章的核心是net/http包，它提供了用于开发强大的Web客户端和服务端的函数功能。在这个包中，http.Get()和https.Get()方法作为客户端，可以用来发送HTTP和HTTPS请求，而http.ListenAndServe()函数可用于创建Web服务器，并且指定服务器监听的IP地址和TCP端口号，然后在该函数中处理传入的请求
- 接口http.RoundTripper可以使Go的元素能够很方便的拥有执行HTTP事务的能力。简单地说，这意味着Go元素可以为http.Request返回http.Response结构
- http.Response类型, 响应
- http.Request, 请求
- http.Transport
    - http.Transport是一个包含大量字段的复杂结构。好消息是在编写HTTP相关程序时，并不需要经常使用http.Transport结构，并且在使用时不需要处理它的所有字段
    - http.Transport结构实现了http.RoundTripper接口，并且支持HTTP、HTTPS和HTTP代理的模式。不过http.Transport是一个低级别的结构，本章中使用的http.Client结构则是一个高级别的HTTP客户端实现