# Mastery: 用 LangChain Python 实现对话流

## 场景

你想要一个更灵活的对话系统——不仅仅是一问一答，还需要自动摘要历史、支持多模型切换、后续可能加工具调用。用 LangChain 的抽象层来管理这些复杂性。

## 目标

完成后：
- 一个命令行对话程序，流式输出
- 自动管理对话历史（带窗口截断）
- 模型可一键切换（OpenAI / Anthropic / 本地）

## 技术选型

**LangChain** (`langchain` + `langchain-openai` + `langchain-community`)
- 理由：内置 Conversation Memory 抽象，支持多种截断策略
- Runnable 统一接口，换模型只改一行
- 后续加 tool calling / RAG / Agent 扩展方便

## 分步引导

### Step 1：安装依赖

**做什么**
安装 LangChain 核心和 OpenAI 集成。

**怎么做**
```bash
pip install langchain langchain-openai
```

环境变量：
```bash
export OPENAI_API_KEY=sk-xxx
```

**为什么**
LangChain 是模型无关的框架。`langchain-openai` 是具体的模型 provider，安装独立意味着换模型时不需要重装核心。

---

### Step 2：创建对话链（带历史管理）

**做什么**
用 LangChain 的 `ChatPromptTemplate` + `RunnableWithMessageHistory` 构建对话链。

**怎么做**
`chat.py`：
```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables import RunnableWithMessageHistory
from langchain_core.output_parsers import StrOutputParser

# 模型（换模型只改这一行）
llm = ChatOpenAI(model="gpt-4o-mini", streaming=True)

# Prompt 模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的 AI 助手。"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])

# 构建链
chain = prompt | llm | StrOutputParser()

# 内存历史存储
store: dict[str, InMemoryChatMessageHistory] = {}

def get_session_history(session_id: str) -> InMemoryChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# 包装成带历史的链
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)
```

**为什么**
`RunnableWithMessageHistory` 自动在每次调用前后管理消息历史——你不需要手动 append user/assistant 消息。`MessagesPlaceholder` 是历史消息的插入点。`|` 管道语法让 prompt → model → parser 形成清晰的数据流。

---

### Step 3：流式对话循环

**做什么**
创建交互式命令行对话，流式输出。

**怎么做**
追加到 `chat.py`：
```python
print("对话已开始（输入 quit 退出）\n")

while True:
    user_input = input("你: ")
    if user_input.strip().lower() == "quit":
        break

    print("AI: ", end="", flush=True)
    for chunk in chain_with_history.stream(
        {"input": user_input},
        config={"configurable": {"session_id": "default"}},
    ):
        print(chunk, end="", flush=True)
    print("\n")
```

运行：
```bash
python chat.py
```

测试追问：
```
你: 什么是 JWT？
AI: JWT 是 JSON Web Token，一种开放标准...
你: 它有什么缺点？
AI: 它的缺点包括...          ← 「它」自动指向上文的 JWT
```

**为什么**
`.stream()` 返回生成器，逐 token 输出。`flush=True` 确保终端实时显示。LangChain 内部自动把每次交互存入 `InMemoryChatMessageHistory`，下次调用时自动注入。

---

### Step 4：切换截断策略（可选扩展）

**做什么**
当对话变长时，控制历史窗口大小。

**怎么做**
```python
from langchain_core.messages import trim_messages

# 在链中插入截断步骤
def trim_history(messages):
    return trim_messages(
        messages,
        max_tokens=4000,
        strategy="last",         # 保留最近的消息
        token_counter=len,       # 简化计数，生产环境用 tiktoken
        include_system=True,
    )
```

**为什么**
LangChain 的 `trim_messages` 提供多种策略（`last` 保留最近的，`first` 保留开头的）。这是 README 中提到的 Memory 抽象的具体实现——截断策略可插拔，不影响对话链的其他部分。

---

## 参考资料

- [LangChain 官方文档](https://python.langchain.com/)
- [RunnableWithMessageHistory API](https://python.langchain.com/docs/how_to/message_history/)
- [LangChain Streaming](https://python.langchain.com/docs/how_to/streaming/)
- [LangChain GitHub](https://github.com/langchain-ai/langchain)
- [trim_messages](https://python.langchain.com/docs/how_to/trim_messages/)
