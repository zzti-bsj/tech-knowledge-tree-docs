# 为什么要在 token 前面加 Bearer？

因为 HTTP 的 `Authorization` 头被设计成一个**多用途字段**，需要一种机制区分不同类型的凭证。`Bearer` 是凭证类型的标识符，告诉服务器「后面这段字符串是一个持有者令牌（bearer token）」。它不是 JWT 的专属要求，而是 HTTP 认证框架的通用约定。

## Authorization 头的设计

HTTP 规范（RFC 7235）定义 `Authorization` 头的格式是：

```
Authorization: <scheme> <credentials>
```

`scheme`（方案）标识凭证的类型，`credentials` 是具体的凭证内容。这是一个可扩展的框架——任何新的认证方式都可以定义自己的 scheme，而不用发明新的 header。

实际在用的 scheme：

| Scheme      | 凭证格式                        | 典型场景              |
| ----------- | --------------------------- | ----------------- |
| `Basic`     | `base64(username:password)` | 简单的内网认证           |
| `Digest`    | MD5 哈希摘要                    | 比 Basic 安全一点的密码认证 |
| `Bearer`    | 任意令牌字符串                     | OAuth 2.0 / JWT   |
| `Negotiate` | SPNEGO/Kerberos token       | Windows 域认证       |

如果没有 scheme 前缀，服务器收到一段字符串，不知道该怎么解析它——这是密码的 base64？还是 token？还是摘要？scheme 消除了这个歧义。

## Bearer 这个词的含义

Bearer 的字面意思就是「持有者」。`Bearer token` 的语义是：**谁持有这个 token，谁就拥有它代表的权限**。

这跟现实生活中的「持票人」概念一样——电影票上不印你的名字，谁拿着票谁就能进场。Bearer token 也是如此：它不绑定特定的客户端或设备，任何能出示它的人都能使用。

这个语义暗示了 bearer token 的安全要求：**必须防止 token 被截获**。因为一旦别人拿到了你的 token，他就拥有了你的一切权限，服务器无法区分「真正的你」和「偷了 token 的人」。这也是为什么 Bearer token 必须通过 HTTPS 传输——在明文 HTTP 下，token 等于明文密码。

## 为什么不是 "JWT"

一个自然的疑问：既然发的是 JWT，为什么不叫 `Authorization: JWT eyJ...`？

因为 Bearer scheme 的定义（RFC 6750）**不关心 token 的具体格式**。Bearer 后面可以是 JWT，可以是 opaque token（一串 UUID），可以是任何字符串。验证方决定怎么解析它——JWT 用签名验证，opaque token 用数据库查询。

这种设计让客户端不需要知道 token 的内部格式。你的前端代码只需要知道「拿到 token，放到 Bearer 后面发出去」，不需要关心后端用的是 JWT 还是其他什么方案。如果后端从 JWT 换成了别的东西，前端代码一行都不用改。

这也是 Auth0（mastery/auth0.md）和 PyJWT（mastery/pyjwt-fastapi.md）虽然内部实现完全不同，但前端发送方式一模一样的原因——都遵循 Bearer scheme，差异被封装在后端。

## 三种传递 Bearer Token 的方式

RFC 6750 定义了三种方式：

1. **Authorization Header**（推荐）：`Authorization: Bearer <token>`
2. **Form Body**：`access_token=<token>`（仅限 POST 请求）
3. **Query Parameter**：`?access_token=<token>`（仅限没有其他选择时）

方式 1 是标准做法，两个 mastery 文档都用了这种方式。方式 3 会把 token 暴露在 URL 里（浏览器历史、服务器日志、Referrer header 都可能泄露），只在无法使用 header 的场景（如文件下载）中才考虑。
