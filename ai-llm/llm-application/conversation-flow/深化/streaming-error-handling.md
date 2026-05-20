# 流式中断的优雅降级

流式输出的错误处理比同步请求困难一个量级——同步请求失败可以整体重试，但流式输出一旦开始，你已经把部分内容发给了客户端，无法撤回。核心策略是：**已发送的内容绝不丢弃，未完成的部分尝试恢复，恢复不了就体面收尾**。

## 问题：为什么流式让错误处理变难

README 第三阶段指出，流式输出的「关键转变」是前端从「等待完整响应」变成「逐块渲染 + 处理中断/错误」。但两个 mastery 实现（`openai-python-fastapi` 和 `langchain-python`）都没有处理任何错误——`generate()` 里的 `for chunk in response` 和 `chain_with_history.stream()` 都假设流会正常完成。

同步请求的错误处理很简单：请求失败，返回 500，前端显示「出错了请重试」。但流式场景下：

1. **状态不可逆**——已经 SSE 推送的 chunk 无法收回，客户端已经开始渲染 markdown
2. **边界模糊**——流在中间断开时，客户端怎么知道这是「正常结束但内容被截断」还是「异常中断」？
3. **历史污染**——mastery 文档里 `history.append({"role": "assistant", ...})` 在流结束后才执行；如果流中断，这一步不执行，下一轮对话就缺少这条 assistant 消息，上下文断裂

## 三种主要故障模式

### 1. 网络断开

**表现**：TCP 连接断开，SSE 流突然终止，客户端收不到 `[DONE]` 信号。

**根因**：客户端网络切换（WiFi → 4G）、代理超时、服务端进程重启。

**难点**：服务端可能还在正常生成 token，只是发不出去了；或者服务端已经挂了，客户端还在等待。

### 2. Token 超限

**表现**：LLM API 返回 `finish_reason: "length"`，输出被截断但流正常结束。

**根因**：对话历史过长（mastery 文档里的 `sessions` 字典或 `InMemoryChatMessageHistory` 不断增长），加上用户当前问题，总 token 超过模型的 `max_tokens`。

**难点**：这不算「异常」——API 正常返回了，但内容不完整。如果不检查 `finish_reason`，客户端会认为回复完整，把截断内容存入历史。

### 3. 内容审核拦截

**表现**：LLM API 返回内容安全错误（如 OpenAI 的 `content_policy_violation`），流可能在第一个 chunk 之前或中间被中止。

**根因**：用户输入或 LLM 生成的部分内容触发了安全过滤。

**难点**：审核可能在生成中途触发——前几个 chunk 正常推送，突然中断。这比「直接拒绝」更难处理，因为你已经发了一部分内容。

## 工程策略

### 策略一：带状态追踪的流式生成

核心改造 mastery 文档中的 `generate()` 函数，加入状态追踪：

```python
def generate():
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=history,
        stream=True,
    )
    assistant_content = ""
    finish_reason = None
    try:
        for chunk in response:
            choice = chunk.choices[0]
            delta = choice.delta
            if delta.content:
                assistant_content += delta.content
                yield {"data": json.dumps({"content": delta.content})}
            if choice.finish_reason:
                finish_reason = choice.finish_reason
    except Exception as e:
        # 流中断：发送错误信号 + 已收集的内容
        yield {"data": json.dumps({
            "error": "stream_interrupted",
            "partial_content": assistant_content,
            "message": "回复中断，已保留已生成的内容",
        })}
        # 仍然保存部分内容到历史（标记为不完整）
        if assistant_content:
            history.append({"role": "assistant", "content": assistant_content})
            history.append({"role": "system", "content": "[上条回复因网络中断被截断]"})
        return

    # 正常结束，但检查 finish_reason
    if finish_reason == "length":
        yield {"data": json.dumps({
            "warning": "truncated",
            "message": "回复因长度限制被截断，可发送「继续」获取后续内容",
        })}
    elif finish_reason == "content_filter":
        yield {"data": json.dumps({
            "error": "content_filtered",
            "partial_content": assistant_content,
            "message": "部分内容因安全策略被过滤",
        })}

    history.append({"role": "assistant", "content": assistant_content})
    yield {"data": "[DONE]"}
```

关键点：
- **`try/except` 包裹整个流循环**，不是只包 API 调用
- **总是检查 `finish_reason`**，不能假设 `null` 就是一切正常
- **部分内容也写入历史**，否则下一轮对话上下文断裂

### 策略二：客户端侧的超时与重试

服务端挂了的时候，客户端必须自己兜底：

```javascript
const evtSource = new EventSource("/chat");
let accumulated = "";
let lastChunkTime = Date.now();

evtSource.onmessage = (event) => {
    lastChunkTime = Date.now();
    const data = JSON.parse(event.data);
    if (data === "[DONE]") { /* 正常结束 */ return; }
    if (data.error) { showPartialWithWarning(accumulated, data.message); return; }
    accumulated += data.content;
    render(data.content);
};

// 心跳超时检测
setInterval(() => {
    if (Date.now() - lastChunkTime > 15000) { // 15 秒无新 chunk
        evtSource.close();
        showPartialWithWarning(accumulated, "连接超时，已保留已接收的内容");
    }
}, 3000);
```

为什么是 15 秒而不是更短：LLM 生成本身有延迟，特别是复杂推理时两个 chunk 之间可能间隔数秒。太短会误判正常延迟为超时。

### 策略三：Token 超限的「继续」机制

当 `finish_reason === "length"` 时，不丢弃已生成的内容，而是允许用户触发续写：

```python
# 用户发送"继续"时
if user_message == "继续" and last_response_was_truncated:
    history.append({"role": "user", "content": "请继续上面的回答"})
    # 新的流式请求，LLM 会基于历史继续生成
```

这比「重新生成完整回答」更省 token，也更符合用户预期。前提是截断的内容已经写入了历史。

## 告诉用户什么

部分失败时的用户提示原则：**诚实但不恐慌**。

| 场景 | 不要说 | 应该说 |
|------|--------|--------|
| 网络中断 | 「系统错误」 | 「回复中断，已保留上方内容。点击重试继续」 |
| Token 截断 | 无提示（用户以为回复完了） | 「回复因长度限制被截断，发送「继续」获取剩余内容」 |
| 内容审核 | 「你的输入违规」 | 「部分内容无法显示，已过滤」 |

共性原则：
- **显示已生成的内容**，不要因为后几行触发了审核就把前面正常的内容也删掉
- **提供明确的下一步操作**（重试、继续、重新提问），而不是只报错
- **区分「你的问题有问题」和「系统出了问题」**——审核拦截是前者，网络断开是后者

## 总结

流式错误处理的本质是在「已经无法原子回滚」的约束下做最优恢复。三个动作：保留已发送内容、尝试补全未完成部分、给用户可操作的反馈。Mastery 文档里的 `generate()` 循环是起点，生产环境必须加 `try/except`、检查 `finish_reason`、处理部分内容写入历史。
