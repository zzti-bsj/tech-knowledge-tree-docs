# Agent Frameworks vs Raw Conversation Flow

Agent 框架（LangGraph、CrewAI、AutoGen）不是对话流的替代品，而是对话流的超集——它们在原始对话循环之上叠加了工具调用、规划循环和多 Agent 协调。如果你的场景只是"用户问、LLM 答"，原始对话流就够了；只有当 LLM 需要自主决定调什么工具、执行多少步、甚至与其他 Agent 协作时，才值得承受框架带来的复杂度。

## 从原始对话流到 Agent：一层层加上去的东西

README 第四阶段定义了关键转变：对话流从"纯文本往返"变成"带副作用的编排循环"。这个转变不是一步跳到的，而是逐层叠加的：

```
原始对话流（while True: user → LLM → response）
  └─ + 工具调用（LLM 返回 tool_call，你执行后把结果塞回 messages）
      └─ + 规划循环（LLM 自主决定是否继续调工具，而非只调一次）
          └─ + 多 Agent（不同角色的 LLM 实例协作）
              └─ + 持久化 & 人机协作（checkpoints、human-in-the-loop）
```

每一层都有明确的代价。理解这些层次，才能判断该停在哪一层。

## 第一层：原始对话流 + 手动工具调用

这是最轻量的方式。核心就是一个 while 循环：

```python
messages = []
while True:
    response = client.chat.completions.create(model="gpt-4", messages=messages)
    if response.choices[0].message.tool_calls:
        for tool_call in response.choices[0].message.tool_calls:
            result = execute_tool(tool_call)
            messages.append({"role": "tool", "content": result})
        continue  # 把工具结果送回 LLM，让它决定下一步
    else:
        print(response.choices[0].message.content)
        break
```

**适用场景**：工具数量少（< 10），调用逻辑线性（调一次出结果），不需要跨步骤的状态管理。

**复杂度成本**：几乎为零。你完全控制 messages 数组的结构，调试就是打印 messages。

**局限**：当工具变多、LLM 需要多步规划（先搜 A 再用 A 的结果查 B）、或者需要中途暂停恢复时，手写循环会迅速失控。

## 第二层：引入 Agent 框架——为什么它们存在

Agent 框架解决的核心问题是**编排复杂性**。当 LLM 不再只调一次工具就结束，而是需要自主规划执行路径时，你需要：

1. **状态管理**：当前走到哪一步，中间结果是什么
2. **路由逻辑**：根据 LLM 的输出决定下一步去哪个节点
3. **循环控制**：什么时候停止，什么时候重试
4. **可观测性**：一个多步 Agent 调用跑了 30 秒，中间发生了什么

这些东西手写都能写，但每个项目都会重新发明一遍。框架把它们标准化了。

## 三个框架，三种哲学

### LangGraph：图即代码

LangGraph 把对话流建模为有向图——节点是计算单元（LLM 调用、工具执行），边是条件路由。

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("llm", call_llm)
graph.add_node("tools", execute_tools)
graph.add_conditional_edges("llm", should_use_tools, {
    True: "tools",
    False: END
})
graph.add_edge("tools", "llm")  # 工具执行完回到 LLM
```

**本质**：它就是一个带状态的 while 循环，但用图的结构显式表达了"可能走哪些路径"。对比原始写法，区别在于**路由逻辑从隐式的 if-else 变成了显式的图的拓扑**。

**优势**：路径可见、可序列化（checkpoints）、支持 human-in-the-loop、可视化调试。

**代价**：图的定义本身是额外的抽象层。简单场景下，你为了"显式"付出的代码量可能比原始循环还多。

### CrewAI：角色驱动的多 Agent

CrewAI 的核心抽象是 Agent（角色 + 目标 + 工具集）和 Task（具体任务 + 期望输出）。多个 Agent 组成 Crew 协作完成任务。

```python
researcher = Agent(role="研究员", goal="收集信息", tools=[search_tool])
writer = Agent(role="写作者", goal="撰写报告", tools=[])

task1 = Task(description="研究 X 主题", agent=researcher)
task2 = Task(description="基于研究结果写报告", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[task1, task2])
crew.kickoff()
```

**本质**：每个 Agent 内部就是一个对话流 + 工具调用循环。CrewAI 额外提供了 Agent 间传递结果的机制和执行顺序管理。

**适用场景**：任务天然可以拆分为不同角色的协作（研究员 + 写手 + 审核员）。

**代价**：多 Agent 意味着多倍的 LLM 调用和 token 消耗。Agent 间通信的不确定性也会放大——一个 Agent 的输出偏差会级联影响下游。

### AutoGen：对话即协议

AutoGen 把多 Agent 协作建模为 Agent 之间的对话。核心概念是"两个 Agent 互相发消息，直到问题解决"。

**本质**：它把 README 第四阶段的"带副作用的编排循环"推到了极致——每个 Agent 都是一个对话流的参与者，工具调用发生在对话内部。

**适用场景**：需要 Agent 之间协商、辩论、互审的场景。

**代价**：对话轮次不可预测，成本控制困难。适合实验原型，生产环境需要严格限制轮次上限。

## 决策框架：什么时候用什么

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 纯对话，无工具 | 原始对话流 | 框架是多余抽象 |
| 有工具，但逻辑简单（调一次出结果） | 原始对话流 + tool calling | 一个循环 + switch-case 足够 |
| 多步工具调用，需要规划（先 A 后 B） | LangGraph | 图结构显式表达路径，支持 checkpoint |
| 需要人工审批中间步骤 | LangGraph | human-in-the-loop 是一等公民 |
| 多角色协作完成复杂任务 | CrewAI | 角色抽象天然匹配 |
| Agent 间需要协商/辩论 | AutoGen | 对话协议灵活 |
| 生产环境，高可靠性要求 | LangGraph 或原始 | 可控性和可观测性优先 |

## 核心判断原则

**每加一层抽象，调试难度指数增长。** 原始对话流的问题，打印 messages 就能定位。LangGraph 的问题，你需要理解图的状态快照。多 Agent 框架的问题，你需要追踪跨 Agent 的消息传递链。

因此：

1. **从原始对话流开始**。只有当 while 循环里的 if-else 多到你自己都理不清时，才考虑 LangGraph。
2. **从单 Agent 开始**。只有当一个 Agent + 工具集无法覆盖任务时，才考虑多 Agent。
3. **把框架当加速器，不是拐杖**。理解原始对话流是理解所有框架的前提。框架的每个概念都能映射回 messages 数组的操作——如果你无法建立这个映射，说明你可能在不必要的地方引入了框架。

## 一句话总结

Agent 框架是对话流的"脚手架"——在构建复杂多步编排时帮你省去重复工作，但建筑本身（对话流 + 工具调用）的地基你仍然需要理解。用原始对话流验证完核心逻辑，再决定是否需要框架来管理复杂性。
