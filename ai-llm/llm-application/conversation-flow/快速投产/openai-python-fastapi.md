# Mastery: 用 OpenAI Python SDK + FastAPI 实现对话流

## 场景

你有一个 Python 后端（FastAPI），需要提供一个 AI 对话 API：前端发消息，LLM 流式回复，支持多轮对话。

## 目标

完成后：
- 一个 SSE 端点 `POST /chat`，流式返回 LLM 响应
- 客户端传入 `session_id`，自动维护多轮对话历史
- 可用 curl 或任意前端对接

## 技术选型

**OpenAI Python SDK** (`openai`) + **FastAPI** + **SSE** (`sse-starlette`)
- 理由：OpenAI SDK 原生支持 streaming，FastAPI 是 Python 生态最主流的异步框架
- SSE 比 WebSocket 更简单，单向流式推送足够覆盖对话场景
- 对话历史用内存字典管理（生产环境可替换为 Redis）

## 分步引导

### Step 1：安装依赖

**做什么**
安装 OpenAI SDK、FastAPI 和 SSE 支持。

**怎么做**
```bash
pip install openai fastapi uvicorn sse-starlette
```

环境变量：
```bash
export OPENAI_API_KEY=sk-xxx
```

**为什么**
`openai` 是官方 SDK，streaming 支持最稳定。`sse-starlette` 让 FastAPI 能输出 SSE 格式的流式响应。

---

### Step 2：实现对话管理 + 流式端点

**做什么**
创建一个 FastAPI 应用，包含对话历史管理和流式聊天端点。

**怎么做**
`main.py`：
```python
import json
from collections import defaultdict
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from openai import OpenAI
from pydantic import BaseModel
from sse_starlette.sse import EventSourceResponse

app = FastAPI()
client = OpenAI()

# 内存存储对话历史（生产环境替换为 Redis/DB）
sessions: dict[str, list[dict]] = defaultdict(list)


class ChatRequest(BaseModel):
    message: str
    session_id: str = "default"


@app.post("/chat")
async def chat(req: ChatRequest):
    history = sessions[req.session_id]
    history.append({"role": "user", "content": req.message})

    def generate():
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=history,
            stream=True,
        )
        assistant_content = ""
        for chunk in response:
            delta = chunk.choices[0].delta
            if delta.content:
                assistant_content += delta.content
                yield {"data": json.dumps({"content": delta.content})}
        # 流结束后，把完整回复存入历史
        history.append({"role": "assistant", "content": assistant_content})
        yield {"data": "[DONE]"}

    return EventSourceResponse(generate())
```

**为什么**
核心模式是「先流式输出，结束后存历史」。`stream=True` 让 OpenAI SDK 返回迭代器，每个 chunk 包含一小段文本。`EventSourceResponse` 将生成器转为 SSE 格式。`session_id` 隔离不同用户的对话历史。

---

### Step 3：运行并测试

**做什么**
启动服务，用 curl 验证流式对话。

**怎么做**
```bash
uvicorn main:app --reload
```

测试第一轮：
```bash
curl -N -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "什么是 JWT？", "session_id": "test"}'
```
观察输出逐行流式出现。

测试追问（同一 session_id）：
```bash
curl -N -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "它有什么缺点？", "session_id": "test"}'
```
LLM 会理解「它」指的是 JWT——对话历史生效。

**为什么**
`-N` 禁用 curl 缓冲，让你实时看到流式输出。同一 session_id 验证多轮上下文是否正常传递。

---

### Step 4：添加历史清理端点（可选）

**做什么**
加一个端点清除对话历史，防止内存泄漏。

**怎么做**
```python
@app.delete("/chat/{session_id}")
async def clear_session(session_id: str):
    sessions.pop(session_id, None)
    return {"status": "cleared"}
```

**为什么**
内存存储会无限增长。生产环境应该用 TTL 自动过期（Redis EXPIRE）或持久化到数据库。这里提供手动清理作为最小可用方案。

---

## 参考资料

- [OpenAI Python SDK Streaming](https://github.com/openai/openai-python#streaming)
- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [sse-starlette](https://github.com/sysid/sse-starlette)
- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
