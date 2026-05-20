# 知识索引

## Web安全

### 身份认证

- [为什么Token前面要加Bearer](web-security/authentication/why-bearer.md)
- [JWT认证——从会话到无状态令牌的演进](web-security/authentication/jwt/README.md)
  - [双Token机制——安全刷新与撤销的完整方案【RLSys】](web-security/authentication/jwt/dual-token【RLSys】.md)
  - 深化
    - [JWT与OAuth——协议层的正交关系](web-security/authentication/jwt/深化/jwt-vs-oauth.md)
    - [JWT"无状态"在生产中的真实边界](web-security/authentication/jwt/深化/stateless-myth.md)
  - 快速投产
    - [Auth0 托管认证接入](web-security/authentication/jwt/快速投产/auth0.md)
    - [PyJWT + FastAPI 自管理认证实战](web-security/authentication/jwt/快速投产/pyjwt-fastapi.md)

## AI与大语言模型

### LLM应用开发

- [对话流管理——从上下文拼接到结构化Agent的演进](ai-llm/llm-application/conversation-flow/README.md)
  - 深化
    - [Agent框架与原始对话循环的选择边界](ai-llm/llm-application/conversation-flow/深化/agent-frameworks-vs-raw-conversation.md)
    - [上下文截断策略——从失败模式出发的选择](ai-llm/llm-application/conversation-flow/深化/context-truncation-strategies.md)
    - [生产级对话历史存储——从内存到三层架构](ai-llm/llm-application/conversation-flow/深化/production-history-storage.md)
    - [SSE vs WebSocket——LLM场景下的传输选择](ai-llm/llm-application/conversation-flow/深化/sse-vs-websocket.md)
    - [流式输出错误处理——不可逆传输下的工程策略](ai-llm/llm-application/conversation-flow/深化/streaming-error-handling.md)
  - 快速投产
    - [LangChain Python 流式对话](ai-llm/llm-application/conversation-flow/快速投产/langchain-python.md)
    - [OpenAI SDK + FastAPI SSE 流式接口](ai-llm/llm-application/conversation-flow/快速投产/openai-python-fastapi.md)
