# PyJWT + FastAPI：30 分钟接入 JWT 认证

## 场景（Scenario）

你有一个 FastAPI 后端 API，需要保护某些接口只允许已登录用户访问。前端是 SPA（React/Vue），登录后拿到 token，后续请求通过 Authorization header 携带。

## 目标（Target）

完成后你将拥有：
- 登录接口返回 JWT access_token 和 refresh_token
- 受保护接口通过依赖注入自动校验 token
- token 过期后前端无感刷新
- 可用 curl 或 Postman 验证完整流程

## 技术选型（Tooling）

| 工具                  | 版本     | 理由                             |
| ------------------- | ------ | ------------------------------ |
| **PyJWT**           | ^2.8   | Python 生态事实标准，纯 JWT 编解码，专注做一件事 |
| **FastAPI**         | ^0.100 | 项目已有，不用额外引入                    |
| **passlib[bcrypt]** | ^1.7   | 密码哈希，生产必须                      |

为什么不选 `python-jose`：维护停滞，PyJWT 2.x 已经支持所有必要功能。

## 分步引导

### Step 1：安装依赖（2 分钟）

**做什么**：安装 JWT 和密码处理库。

**怎么做**：
```bash
pip install PyJWT passlib[bcrypt]
```

**为什么**：PyJWT 负责 token 签发和验证，passlib 负责密码安全存储。两个职责，两个库，不混在一起。

---

### Step 2：配置 JWT 参数（3 分钟）

**做什么**：创建 JWT 配置和工具函数。

**怎么做**：
```python
# app/core/jwt.py
from datetime import datetime, timedelta, timezone
import jwt

SECRET_KEY = "your-secret-key"  # 生产环境从环境变量读取
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30


def create_access_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(data: dict) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(days=7)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_token(token: str) -> dict:
    return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
```

**为什么**：
- `exp` 声明让 token 自动过期，不需要你手动检查时间
- `type` 字段区分 access 和 refresh，防止用 refresh token 调 API
- `jwt.encode` 返回的就是字符串，不需要额外编码

---

### Step 3：登录接口签发 token（10 分钟）

**做什么**：实现 `/auth/login` 接口，验证密码后返回双 token。

**怎么做**：
```python
# app/api/v1/auth.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from passlib.context import CryptContext
from app.core.jwt import create_access_token, create_refresh_token

router = APIRouter()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 假设你有一个获取用户的方法
# from app.services.user import get_user_by_username


class LoginRequest(BaseModel):
    username: str
    password: str


@router.post("/login")
async def login(req: LoginRequest):
    user = get_user_by_username(req.username)  # 你的用户查询逻辑
    if not user or not pwd_context.verify(req.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="用户名或密码错误")

    token_data = {"sub": str(user.id), "role": user.role}
    return {
        "access_token": create_access_token(token_data),
        "refresh_token": create_refresh_token(token_data),
        "token_type": "bearer",
    }
```

**为什么**：
- `passlib.verify` 自动处理 bcrypt 的 salt，你不需要手动管理
- `sub`（subject）是 JWT 标准声明，放用户 ID 是惯例
- 返回 `token_type: bearer` 是 OAuth2 规范，前端库通常期望这个字段

**验证**：
```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "123456"}'
# 应该返回 { "access_token": "eyJ...", "refresh_token": "eyJ...", "token_type": "bearer" }
```

---

### Step 4：FastAPI 依赖注入保护接口（10 分钟）

**做什么**：创建一个依赖，从请求头提取并验证 token。

**怎么做**：
```python
# app/core/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from app.core.jwt import decode_token

security = HTTPBearer()


async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    token = credentials.credentials
    try:
        payload = decode_token(token)
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token 已过期")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="无效 Token")

    if payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="需要 access token")

    return {"user_id": payload["sub"], "role": payload.get("role")}
```

在需要保护的接口上使用：
```python
# app/api/v1/tasks.py
from fastapi import APIRouter, Depends
from app.core.deps import get_current_user

router = APIRouter()


@router.get("/tasks")
async def list_tasks(user=Depends(get_current_user)):
    # user["user_id"] 就是当前登录用户
    return {"tasks": [], "user_id": user["user_id"]}
```

**为什么**：
- `HTTPBearer()` 自动从 `Authorization: Bearer <token>` 提取 token，不用手动解析 header
- `Depends(get_current_user)` 让任何接口一行代码获得认证，FastAPI 的依赖注入是干这事最优雅的方式
- 检查 `type != "access"` 防止 refresh token 被用来调 API

**验证**：
```bash
# 不带 token → 403
curl http://localhost:8000/api/v1/tasks

# 带上一步拿到的 access_token → 200
curl http://localhost:8000/api/v1/tasks \
  -H "Authorization: Bearer eyJ..."
```

---

### Step 5：Token 刷新接口（5 分钟）

**做什么**：实现 `/auth/refresh`，用 refresh_token 换新的 access_token。

**怎么做**：
```python
# 在 app/api/v1/auth.py 中添加
@router.post("/refresh")
async def refresh(req: RefreshRequest):
    try:
        payload = decode_token(req.refresh_token)
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Refresh token 已过期，请重新登录")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="无效的 refresh token")

    if payload.get("type") != "refresh":
        raise HTTPException(status_code=401, detail="需要 refresh token")

    token_data = {"sub": payload["sub"], "role": payload.get("role")}
    return {
        "access_token": create_access_token(token_data),
        "token_type": "bearer",
    }
```

**为什么**：
- 只返回新的 access_token，不返回新的 refresh_token（简单方案）
- 如果需要 refresh token 也可轮换（更安全），参考 `dual-token【RLSys】.md` 中的 rotation 机制
- 前端在 access_token 过期（收到 401）时自动调用此接口，对用户无感

---

## 下一步

到这里你已经有一个可工作的 JWT 认证系统。但还有几个生产必须做的事：

| 事项                     | 优先级 | 说明                          |
| ---------------------- | --- | --------------------------- |
| SECRET_KEY 从环境变量读取     | P0  | 硬编码 = 泄露                    |
| HTTPS                  | P0  | HTTP 下 token 可被中间人截获        |
| Token 黑名单（可选）          | P1  | JWT 本身不可撤销，如需即时踢人需加黑名单      |
| Refresh Token Rotation | P1  | 每次刷新时也换 refresh_token，防止重放  |
| 密钥轮换策略                 | P2  | 定期更换 SECRET_KEY，见 deepen 文档 |

更安全的 refresh token 方案（DB 存储 + Rotation），参见同级 `dual-token【RLSys】.md`。
