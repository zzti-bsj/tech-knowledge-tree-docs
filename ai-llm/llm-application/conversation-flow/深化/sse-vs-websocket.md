# SSE vs WebSocket：LLM 对话流该选哪个？

结论：**绝大多数 LLM 对话流场景，SSE 就是正确答案。** 原因不是 SSE「更好」，而是 LLM 对话的数据流模式天然是单向的——用户发请求，服务端流式返回 token。WebSocket 的双向能力在这个模式下几乎用不到，却要承担更复杂的状态管理和运维成本。只有当你需要「在流式输出过程中从客户端主动干预」（取消生成、中途注入指令）时，WebSocket 才有真正的价值。

## 为什么 LLM 对话是单向流

回顾 README 第三阶段的描述：Streaming 的本质是前端从「等待完整响应」变成「逐块渲染」。核心数据流长这样：

```
客户端 --请求--> 服务端 --stream--> 客户端
        POST           data: {"delta": "JWT"}
                       data: {"delta": " 是一种"}
                       data: [DONE]
```

这是一个经典的 **请求-流式响应** 模式。客户端发一次请求，服务端持续推送数据直到完成。整个过程客户端不需要再发送任何东西。

SSE 的设计恰好匹配这个模式：
- 基于 HTTP，一次请求建立长连接
- 服务端单向推送 `data:` 行
- 自带 `Last-Event-ID` 重连机制
- 浏览器原生 `EventSource` API 支持

对比 WebSocket：它是全双工的，两端可以随时互发消息。但 LLM 对话流中，**流式输出期间客户端没有什么可说的**。用户要做的只是等待——看 token 一个个冒出来。双向通道的一半在这段时间里是闲置的。

## SSE 的隐藏优势

Mastery 文档中 OpenAI Python + FastAPI 的实现已经展示了 SSE 的简洁性。一个 `EventSourceResponse` 包装一个生成器，就完成了流式输出。这种简洁不是表面上的代码量少，而是**架构复杂度的差异**：

**协议层面**：SSE 是标准 HTTP。这意味着它天然继承 HTTP 生态的一切——CDN 缓存、反向代理透传、负载均衡器的 health check、API Gateway 的限流策略。WebSocket 是独立协议（`Upgrade: websocket`），每经过一层基础设施都要确认「是否支持 WebSocket」，在生产环境中这是一个被低估的摩擦源。

**连接管理**：SSE 连接断了，浏览器 `EventSource` 自动重连，带上 `Last-Event-ID`，服务端可以从断点续传。WebSocket 断了，应用层需要自己实现重连逻辑和状态恢复。对于 LLM 流式输出这种「一旦开始就不应中断」的场景，SSE 的自动重连是实实在在的生产力。

**基础设施兼容性**：很多企业的网络环境（企业代理、防火墙）对 WebSocket 长连接不友好，但 HTTP 长连接通常没问题。SSE 基于 HTTP/1.1 的 chunked transfer，穿墙能力更强。

## WebSocket 什么时候才值得

说 SSE 够用，不等于 WebSocket 没有场景。有三个具体的场景需要认真考虑 WebSocket：

### 场景一：取消生成（Cancellation）

用户发现 LLM 的回答偏了，想立刻停止。在 SSE 模式下，客户端可以关闭连接来「取消」，但服务端感知到连接关闭可能有延迟——LLM API 的调用可能还在继续，白白消耗 token 和算力。

WebSocket 模式下，客户端可以发一个明确的 `cancel` 消息，服务端立刻停止调用 LLM API。这个差别在高并发场景下是有意义的：早一秒取消，早一秒释放资源。

### 场景二：中途干预（Mid-stream Intervention）

更高级的交互模式：LLM 正在生成回答，用户中途插入补充信息或修改指令。比如：

```
LLM: JWT 是一种令牌标准，它...
用户: 等等，只说和 OAuth 的区别
LLM: 好的，JWT 和 OAuth 的核心区别是...
```

这要求在流式输出期间，客户端能向服务端发消息。SSE 做不到（它是单向的），WebSocket 天然支持。但这种交互模式目前还很少见，属于第四阶段（Agent / Tool Use）的高级场景。

### 场景三：高频双向交互

当对话流演化为 Agent 模式（README 第四阶段），LLM 可能需要多次调用工具、前端需要实时显示工具调用的进度、用户可能在工具执行期间给出反馈。这种多轮交互模式才真正需要 WebSocket 的全双工能力。

## 生产环境的真实考量

选型的最终裁判不是理论分析，而是运维事实：

**负载均衡**：SSE 是 HTTP 请求，负载均衡器可以正常路由。WebSocket 需要配置 sticky session 或者使用 Redis adapter 来跨节点共享状态，否则重连可能被路由到不同节点。对于 SSE，即使路由到新节点，`Last-Event-ID` 也让续传成为可能（前提是服务端做了事件持久化）。

**连接数限制**：浏览器对同一域名的 HTTP 连接有上限（通常 6 个），SSE 会占用一个。如果同一页面需要多个流式输出（比如多个 Agent 并行），WebSocket 的复用能力更合适。但单对话窗口场景下，一个 SSE 连接足够。

**认证与安全**：SSE 走 HTTP，标准的 `Authorization` header 或 cookie 认证直接适用。WebSocket 的认证需要在握手阶段处理（通常通过 query parameter 或 subprotocol），或者在建立后发第一条认证消息。前者有 token 泄露到日志的风险，后者增加了复杂度。

## 决策框架

```
你的 LLM 对话流需要客户端在生成过程中主动发消息吗？
├── 不需要 → SSE（覆盖 90% 的场景）
└── 需要
    ├── 只是取消 → SSE + 关闭连接就够了
    └── 中途干预 / Agent 多轮 → WebSocket
```

一个务实的路径：**先用 SSE 跑起来**（正如 FastAPI mastery 文档中 30 分钟就能实现的），遇到真正的双向需求再迁移到 WebSocket。过早引入 WebSocket 的复杂度，是对工程资源的浪费——因为它解决的问题在大多数 LLM 对话流中根本不存在。
