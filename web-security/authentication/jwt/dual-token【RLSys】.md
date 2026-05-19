# 双 Token 机制

## 核心思路

JWT 无法撤销，有效期越长风险越大。双 Token 机制把 JWT 的有效期压到极短，用一个独立的长效令牌来续期：

- **access_token**（JWT，30分钟）— 用于 API 认证
- **refresh_token**（UUID，7天）— 仅用于换新的双 Token，存在数据库可撤销

## 工作流程

```
登录 → 后端返回 access_token + refresh_token

正常请求：携带 access_token → 通过

access_token 过期：
  1. API 返回 401
  2. 前端拦截器自动用 refresh_token 调 /auth/refresh
  3. 后端验证 refresh_token，签发新的双 Token（旧的 refresh_token 失效）
  4. 前端用新 token 重试原请求，用户无感知

refresh_token 也过期：跳转登录页
```

## 关键设计

### 轮换机制（Rotation）
每次刷新时，旧的 refresh_token 立即失效，签发新的。这样即使 refresh_token 被盗用，合法用户下次刷新时会让旧的失效，盗用者无法继续使用。

如果后端检测到一个已被撤销的 refresh_token 被使用，说明可能被盗用，会撤销该用户所有 refresh_token。

### 互斥锁
多个 API 请求同时 401 时，只发一次 refresh 请求，其他请求排队等待结果后重试。避免重复刷新导致 token 竞争。

### 数据库存储
refresh_token 存在 `refresh_tokens` 表中，支持：
- 过期清理
- 主动撤销（登出）
- 异常检测（已撤销的 token 被使用 → 撤销该用户所有 token）

### 启动时主动刷新
App.vue 启动时检查 token 状态，如果 access_token 过期但有有效的 refresh_token，静默刷新。用户长时间未访问后打开页面，只要在 7 天内就免登录。

## API 端点

| 端点                    | 说明                                                  |
| --------------------- | --------------------------------------------------- |
| `POST /auth/login`    | 登录，返回 `{ access_token, refresh_token, expires_in }` |
| `POST /auth/register` | 注册，返回同上                                             |
| `POST /auth/refresh`  | 用 refresh_token 换新双 Token（轮换）                       |
| `POST /auth/logout`   | 撤销 refresh_token                                    |

## 前端实现要点

- `utils/tokenRefresh.ts` — 刷新管理器（互斥锁 + 请求排队）
- `utils/logout.ts` — 统一登出（调后端撤销 + 清本地）
- Axios 响应拦截器 — 401 时自动刷新重试，用原始实例重试避免丢失 baseURL
- 路由守卫 — 检查 JWT 的 exp 字段判断是否过期，过期且无 refresh_token 则直接跳登录

## 后端实现要点

- `app/core/config.py` — access_token 30分钟，refresh_token 7天
- `app/api/v1/auth.py` — refresh 端点用 `SELECT ... FOR UPDATE` 行级锁防并发
- `db/migrate/versions/1766500000_create_refresh_tokens_table.py` — refresh_tokens 表迁移
