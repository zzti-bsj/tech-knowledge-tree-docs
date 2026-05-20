# Deepen: 生产环境的对话历史存储与过期

生产环境存储对话历史，首选 **Redis（带 TTL 过期）** 作为主存储，满足低延迟读写、自动过期、水平扩展三个核心需求。长期归档需求配合数据库或对象存储做冷热分层。mastery 文档中的 `defaultdict` 内存字典和 `InMemoryChatMessageHistory` 都是单进程玩具方案——进程重启丢数据、无过期机制导致内存泄漏、多实例部署时 session 无法共享。

## 真实需求：为什么内存方案不够用

mastery 文档展示了两种内存存储：

- **LangChain 方案**（langchain-python）：`store: dict[str, InMemoryChatMessageHistory] = {}`——纯 Python 字典，只增不删
- **FastAPI 方案**（openai-python-fastapi）：`sessions: dict[str, list[dict]] = defaultdict(list)`——同样只增不删，虽有手动清理端点但无自动过期

这两者共享三个生产环境致命问题：

1. **无持久化**——进程重启（部署、崩溃、OOM Kill）所有对话丢失
2. **无过期机制**——session 只增不减，长期运行必然内存泄漏。FastAPI 文档的 Step 4 注释也承认：「生产环境应该用 TTL 自动过期（Redis EXPIRE）」
3. **不支持水平扩展**——多实例部署时，请求路由到不同实例拿不到历史，对话上下文断裂

## 存储方案对比

### Redis + TTL：主力方案

对话历史本质是「短生命周期、高频读写、需要自动过期」的热数据，Redis 天然匹配：

```
# 写入一条消息
HSET session:abc123 messages '[{"role":"user","content":"什么是JWT"}]'

# 设置过期（对话结束后 30 分钟自动清理）
EXPIRE session:abc123 1800
```

关键设计点：

- **TTL 续期策略**：每次交互刷新 TTL（类似 HTTP Session），活跃对话不会被误清。用户离开后 TTL 自然到期
- **数据结构选择**：简单场景用 `STRING` 存整个消息列表 JSON；消息量大时用 `LIST` + `LPUSH`/`LRANGE` 分页读取
- **截断在写入时做**：不要存全量历史再读出来截断。写入时按 token budget 裁剪，保持 Redis 中只存有效数据

LangChain 生态有 `RedisChatMessageHistory` 可直接替换 `InMemoryChatMessageHistory`，接口一致，改动极小。

### 关系型数据库：需要持久化的场景

Redis 适合「热对话」，但有些场景需要跨会话查询：

- 用户想查看历史对话记录
- 合规要求保留对话审计日志
- 对话数据分析（用户行为、LLM 回复质量）

典型做法：对话结束后，异步从 Redis 写入数据库（PostgreSQL / MySQL）。表设计沿 session_id + created_at 组织，不做实时查询的瓶颈。

### 对象存储（S3 / OSS）：长期冷归档

对话日志量大、查询频率低时，直接写对象存储：

- 按日期分区：`s3://chat-logs/2026/05/20/session-abc123.json`
- 成本极低，适合合规保留和离线分析
- 不适合实时对话读取，只做归档

### 冷热分层架构

实际生产通常是三层组合：

```
请求 → Redis（热，TTL 30min）→ 对话结束 → DB（温，按 session 查询）→ 定期归档 → S3（冷）
```

## Session 生命周期管理

### Session ID 生成

- 前端生成 UUID v4，后端不维护 session 映射表
- 如果需要用户级别关联，用 `user_id + device_id + timestamp` 组合，但对话读取仍走 session_id

### 过期触发时机

| 触发方式 | 实现 | 适用场景 |
|----------|------|----------|
| Redis TTL | `EXPIRE session:id 1800` | 用户离开后自动清理 |
| 显式结束 | 用户点击「新对话」时 DEL key | 即时释放 |
| 定时扫描 | 定期扫描 DB 中超时 session | 补充 Redis 未覆盖的清理 |
| 容量保护 | Redis maxmemory + allkeys-lru | 兜底，防止 Redis OOM |

### 对话结束的判定

TTL 续期模式下不需要精确判定「对话结束」——用户停止交互，TTL 自然到期。如果需要更及时的清理（节省内存），可以监听前端事件（页面关闭、切换话题）主动触发 DEL。

## 选型决策要点

**选 Redis 的信号**：
- 对话是短生命周期（分钟到小时级）
- 需要自动过期
- 多实例部署，需要共享 session
- 读写延迟要求低（每次 LLM 调用前都要读历史）

**选数据库的信号**：
- 对话需要长期保留和检索
- 有合规审计要求
- 用户需要回看历史对话
- 单实例部署且并发量不大（此时用数据库也能扛）

**不选内存的信号**：
- 只要是生产环境，就不该用进程内字典存 session
- 唯一例外：单实例、对话不需要跨重启保留、且有过期清理机制——即便如此，Redis 也是更低成本的选择

## 参考来源

- README.md：对话流四阶段演化中「上下文管理」的工程问题定义
- mastery/langchain-python.md：`InMemoryChatMessageHistory` + `store` 字典的内存实现
- mastery/openai-python-fastapi.md：`defaultdict` 内存方案及 Step 4 中对 TTL/Redis 的前瞻性注释
