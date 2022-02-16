# HTTP

请求-响应协议。

使用TCP作为支撑传输层的协议。

默认工作在TCP协议的80端口。

## HTTP头部信息

- 通用头部

- 请求头部

  告诉服务器的信息，自己允许哪些媒体类型。

- 响应头部

  告知客户端的服务器信息等

- 实体头部

## Keep-Alive

是否每次完成请求处理后立即断开TCP链接

HTTP/1.0 非Keep-Alive

HTTP/1.1 默认Keep-Alive

## 状态保持

- Session

  保存在服务端

- Cookie

  保存在客户端

## get和post

### 区别

- get请求的数据会放在url之后，请求的参数会被完整的保留在浏览器记录中。

  post参数放在请求主题中，参数不会被保留，相对更加安全。

- get请求只支持url编码，post请求支持多种编码格式。

- get只支持ASCII字符格式的参数，而post没有区别。

- get提交的数据大小有限制（浏览器的url限制），而post的数据没限制

- get方法产生一个TCP数据包，post方法产生两个

## 状态码

200 请求成功
204 请求成功但无内容返回
206 范围请求成功

301 永久重定向； 30(2|3|7)临时重定向，语义和实现有略微区别；
304 带if-modified-since 请求首部的条件请求，条件没有满足

400 语法错误
401 需要认证信息
403 拒绝访问
404 找不到资源
412 除if-modified-since 以外的条件请求，条件未满足

500 服务器错误
503 服务器宕机了

## HTTP/1.1和HTTP/1.0的区别

- 缓存处理

  1.1有更多关于缓存的字段

- 节约带宽

  1.1允许只请求部分资源，range头域控制

- 错误的通知管理

  1.1新增了24个错误状态码

- Host请求头

  新增了host请求头，辨识虚拟主机（同一个ip不同主机）

- keep-Alive

  1.1默认Keep-Alive

## HTTP/1.x和HTTP/2.0的区别

- 1.x的文本采用字符串传送。2.0采用二进制传输，封装成帧，帧组成了数据流
- 2.0支持多路复用，通过一个请求实现多个HTTP请求传输
- 2.0有信息压缩
- 2.0支持在客户端未经请求许可的情况下进行服务器推送

## QUIC（quick UDP Internet Connect）

HTTP/3的前身。

主要提升有：

- 采用UDP，更快。

- 具有加密机制。

- 运行在用户域，而不是系统内核

- 具有纠错机制

  包含了本身以外其他数据包的内容

## HTTP/3

主要特点：

- 采用udp通信
- QUIC保证安全性，建立连接就完成了TLS加密握手
- 头部数据压缩

# HTTPS

在HTTP的基础上通过传输加密和身份认证的方式保证了数据的安全性。

工作流程如下：

- 客户端发起一个HTTPS请求，并连接到服务器的443端口，发送信息主要包括自身所支持的算法列表和密钥长度等。
- 服务端将自身所支持的所有加密算法与客户端的算法列表对比并选择一种支持的加密算法，然后将它和其他密钥组件一同发送给客户端
- 服务端向客户端发送一个包含数字证书的报文，该数字证书中包含证书的颁发机构，过期事件，服务端的公钥等信息
- 最后服务端发送一个完成报文通知客户端SSL的第一阶段已经协商完成。
- SSL第一次协商完成后，客户端发送一个回应报文，报文中包含一个客户端生成的随机密码穿，并且该报文时经过证书中的公钥加密过的
- 紧接着这客户端会发送一个提报文提示密钥
- 客户端向服务端发送一个finish报文，包含至今为止所有报文的整体校验值，最终协商是否完成取决于服务端能否成功解密
- 客户端再发送一个同样功能的报文，验证客户端是否能正确解密。
- SSL完成，之后进行和HTTP相同的通信过程

## 加密方式

对称与非对称相结合。

首先使用SSL/TLS协议弥补非对称加密的缺点，进一步采用证书加强安全性，

之后协商对称密钥。

### 客户端为什么信任第三方证书

没有CA机构的私钥，无法得到加密后的签名。

客户端解密后会发现原文和其那名解密后不一样。
