# LLM Conversation Flow

## 问题起源

LLM API 本质是**无状态的请求-响应**——你发一段 prompt，它回一段 completion。但真实的产品交互是对话：用户会追问、纠正、补充上下文。怎么把无状态的 API 变成有状态的对话？

## 演化路径

### 第一阶段：手动拼接上下文

最朴素的做法——每次请求时，把历史消息手动拼进 prompt：

```
[用户] 什么是 JWT？
[助手] JWT 是一种令牌标准...
[用户] 它和 Session 有什么区别？  ← 新请求，但需要带上前面的对话
```

**问题**：上下文窗口有限（token budget），对话越来越长怎么办？删哪些、留哪些？手动管理极易出错。

### 第二阶段：框架抽象（Memory / Chat History）

LangChain、LlamaIndex 等框架引入了 **Conversation Memory** 概念——自动管理消息历史，提供不同截断策略：

- **ConversationBufferMemory**：全量保留（简单但浪费 token）
- **ConversationBufferWindowMemory**：滑动窗口，只保留最近 N 轮
- **ConversationSummaryMemory**：用 LLM 自动摘要旧对话（省 token 但多一次 LLM 调用）

本质：**把「怎么管理上下文」这个工程问题抽象成了可插拔的策略**。

### 第三阶段：Streaming（流式输出）

用户不想等 10 秒看一大段文字突然出现。Streaming 让 token 逐个返回，用户体验接近实时打字。

技术上从 `POST /completions` 的同步响应变成 **Server-Sent Events (SSE)** 或 **WebSocket** 流：

```
data: {"choices":[{"delta":{"content":"JWT"}}]}
data: {"choices":[{"delta":{"content":" 是一种"}}]}
data: {"choices":[{"delta":{"content":"开放标准"}}]}
data: [DONE]
```

**关键转变**：前端从「等待完整响应」变成「逐块渲染 markdown + 处理中断/错误」。

### 第四阶段：结构化对话流（Agent / Tool Use）

对话不再只是「用户问、LLM 答」。LLM 可以主动调用工具（搜索、计算、API），多轮编排：

```
用户：帮我查一下北京明天的天气
  → LLM 调用 weather_api 工具
  → 拿到结果，组织自然语言回复
```

**本质变化**：对话流从「纯文本往返」变成「带副作用的编排循环」——LLM 决定何时调工具、如何整合结果、是否继续追问。

### 当前状态

现代 LLM 对话流需要同时处理：
1. **上下文管理**（token budget 内保留最相关信息）
2. **流式输出**（SSE/WebSocket）
3. **工具调用**（function calling / tool use）
4. **错误与中断**（网络断开、token 超限、内容审核）

这些组合在一起，就是「LLM 对话流」这个技术主题的核心。
