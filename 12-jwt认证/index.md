# 

>#### 问：什么是 JWT ？

JWT（JSON Web Token）是目前最流行的**跨域认证**解决方案，是一种基于 Token 的认证授权机制。从 JWT 的全称可以看出，JWT 本身也是 Token，一种规范化之后的 JSON 结构的 Token。

**JWT 自身包含了身份验证所需要的所有信息**，因此，我们的**服务器不需要存储 Session 信息**。这显然增加了系统的可用性和伸缩性，大大减轻了服务端的压力。

---

JWT 本质上就是一组字符串，通过（`.`）切分成三个为 Base64 编码的部分：

- **Header**：描述 JWT 的元数据，定义了生成签名的算法以及 `Token` 的类型；
- **Payload**：用来存放实际需要传递的数据；
- **Signature（签名）**：服务器通过 Payload、Header 和私钥（Secret）使用 Header 里面指定的签名算法（默认是 HMAC SHA256）生成。

JWT 通常是这样的：`xxxxx.yyyyy.zzzzz`。

示例：

```token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

你可以在 [jwt.ioopen in new window](https://jwt.io/) 这个网站上对其 JWT 进行解码，解码之后得到的就是 Header、Payload、Signature 这三部分。

![image-20230308170148320](https://chuyu-typora.oss-cn-hangzhou.aliyuncs.com/image/image-20230308170148320.png)

> #### 问：如何基于 JWT 进行身份验证？

在基于 JWT 进行身份验证的的应用程序中，服务器通过 Payload、Header 和 Secret（密钥） 创建 JWT 并将 JWT 发送给客户端。**客户端接收到 JWT 之后，会将其保存在 Cookie 或者 localStorage 里面**，以后客户端发出的所有请求**都会携带**这个令牌。

**两点建议：**

1. 建议将 JWT 存放在 localStorage 中，放在 Cookie 中会有 CSRF 风险。
2. 请求服务端并携带 JWT 的常见做法是将其放在 HTTP Header 的 `Authorization` 字段中（`Authorization: Bearer Token`）。

> #### 问：如何提高 JWT 的安全性？

1. JWT 存放在 localStorage 中而不是 Cookie 中，避免 CSRF 风险；
2. 一定不要将隐私信息存放在 Payload 当中，因为它不加密；
3. JWT 的过期时间不易过长。

> #### 问：JWT 为啥不会有 CSRF 攻击漏洞？

一般情况下我们使用 JWT 的话，在我们登录成功获得 JWT 之后，一般会选择存放在 localStorage 中。前端的每一个请求后续都会附带上这个 JWT，整个过程压根不会涉及到 Cookie。因此，即使你点击了非法链接发送了请求到服务端，这个非法请求也是不会携带 JWT 的，所以这个请求将是非法的。

总结来说就一句话：**使用 JWT 进行身份验证不需要依赖 Cookie ，因此可以避免 CSRF 攻击。**

但是 XSS 依然会有。

> #### 问：JWT 重放攻击怎么防御？

- JWT **过期时间设置的短点**；
- 增加**时间戳**机制，但时间同步是个问题；
- 客户端每次需要携带服务端分发的**随机数**，随机数的维护是个问题；
- 客户端与服务端商量**流水号**，落后的序号将被认定为重放攻击，分布式场景下序号同步是个问题。


