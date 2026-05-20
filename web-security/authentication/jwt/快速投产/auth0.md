# Auth0：零后端代码接入 JWT 认证

## 场景（Scenario）

你有一个 SPA 前端 + API 后端的应用。你不想自己管密钥、不想管 token 签发刷新、不想管密码存储和重置。你想把认证这件事完全交给专业服务，自己的代码只关心业务逻辑。

## 目标（Target）

完成后你将拥有：
- 前端登录弹窗（Auth0 Universal Login），支持邮箱/密码和社交登录
- 后端通过中间件自动验证 JWT，无需写任何认证代码
- Auth0 控制台可管理用户、查看登录日志、配置登录方式
- 整个过程后端不接触密钥，不存储密码

## 技术选型（Tooling）

| 工具                     | 理由                           |
| ---------------------- | ---------------------------- |
| **Auth0**              | 认证领域头部 SaaS，免费额度 25000 MAU   |
| **@auth0/auth0-react** | 官方 React SDK，处理前端所有认证流程      |
| **express-jwt**        | Node.js 中间件，验证 Auth0 签发的 JWT |

为什么选 Auth0 而不是 Firebase Auth / Cognito：Auth0 的 Universal Login 是开箱即用的托管登录页，其他服务需要自己搭 UI。当你不想碰任何认证 UI 时，Auth0 是最快的路径。

## 分步引导

### Step 1：注册 Auth0 并创建应用（5 分钟）

**做什么**：在 Auth0 控制台创建 Tenant 和 Application。

**怎么做**：
1. 注册 [auth0.com](https://auth0.com)，创建 Tenant
2. Applications → Create Application → Single Page Web Application
3. Settings 中配置：
   - **Allowed Callback URLs**: `http://localhost:5173/callback`
   - **Allowed Logout URLs**: `http://localhost:5173`
   - **Allowed Web Origins**: `http://localhost:5173`
4. 记录 **Domain**（如 `dev-xxx.us.auth0.com`）和 **Client ID**

**为什么**：
- Callback URL 是 Auth0 登录完成后跳回你应用的地址，不配置会被拒绝
- SPA 类型选择 "Single Page Web Application"，Auth0 会自动启用 PKCE 流程（公钥客户端不需要 client_secret）

---

### Step 2：前端接入 Auth0 SDK（10 分钟）

**做什么**：在 React 应用中集成 Auth0，实现登录/登出/获取 token。

**怎么做**：
```bash
npm install @auth0/auth0-react
```

```tsx
// src/main.tsx
import { Auth0Provider } from "@auth0/auth0-react";

root.render(
  <Auth0Provider
    domain="dev-xxx.us.auth0.com"
    clientId="your-client-id"
    authorizationParams={{
      redirect_uri: window.location.origin + "/callback",
    }}
  >
    <App />
  </Auth0Provider>
);
```

```tsx
// src/components/LoginButton.tsx
import { useAuth0 } from "@auth0/auth0-react";

export function LoginButton() {
  const { loginWithRedirect, logout, isAuthenticated, user } = useAuth0();

  if (isAuthenticated) {
    return (
      <div>
        <span>{user?.name}</span>
        <button onClick={() => logout({ logoutParams: { returnTo: window.location.origin } })}>
          登出
        </button>
      </div>
    );
  }

  return <button onClick={() => loginWithRedirect()}>登录</button>;
}
```

```tsx
// src/utils/api.ts
import { useAuth0 } from "@auth0/auth0-react";

// 在组件外获取 token 的方式（Axios 拦截器等场景）
let getAccessTokenSilently: () => Promise<string>;

export function setTokenGetter(fn: () => Promise<string>) {
  getAccessTokenSilently = fn;
}

// 在 App 组件中调用一次
// const { getAccessTokenSilently } = useAuth0();
// setTokenGetter(getAccessTokenSilently);

export async function getAuthHeaders() {
  const token = await getAccessTokenSilently();
  return { Authorization: `Bearer ${token}` };
}
```

**为什么**：
- `loginWithRedirect()` 跳转到 Auth0 托管的登录页（Universal Login），你不需要写任何登录 UI
- `getAccessTokenSilently()` 自动处理 token 缓存和刷新——SDK 在内存中缓存 access_token，过期前自动用 iframe 静默刷新，你完全不用管
- Auth0 签发的 JWT 使用 RS256（非对称签名），公钥通过 JWKS 端点分发，后端不需要存储密钥

---

### Step 3：API 端配置（3 分钟）

**做什么**：在 Auth0 中创建 API 资源，定义 JWT 的 audience。

**怎么做**：
1. Auth0 Dashboard → APIs → Create API
2. **Name**: `My API`
3. **Identifier**: `https://api.myapp.com`（这就是 audience，可以不是真实 URL）
4. **Signing Algorithm**: RS256（默认）

**为什么**：
- Auth0 只有在请求中指定了 `audience` 时才会返回 JWT（否则返回不透明的 access_token）
- `audience` 声明告诉后端「这个 token 是给哪个 API 用的」，防止跨 API 重放
- RS256 意味着用私钥签名、公钥验证，你的后端只需要公钥

在前端 Provider 中补上 audience：
```tsx
<Auth0Provider
  domain="dev-xxx.us.auth0.com"
  clientId="your-client-id"
  authorizationParams={{
    redirect_uri: window.location.origin + "/callback",
    audience: "https://api.myapp.com",  // 加这一行
  }}
>
```

---

### Step 4：后端验证 JWT（10 分钟）

**做什么**：在 Express 中间件中验证 Auth0 签发的 JWT。

**怎么做**：
```bash
npm install express-jwt jwks-rsa
```

```js
// src/middleware/auth.js
const { expressjwt: expressJwt } = require("express-jwt");
const jwksRsa = require("jwks-rsa");

const checkJwt = expressJwt({
  secret: jwksRsa.expressJwtSecret({
    cache: true,
    rateLimit: true,
    jwksRequestsPerMinute: 5,
    jwksUri: "https://dev-xxx.us.auth0.com/.well-known/jwks.json",
  }),
  audience: "https://api.myapp.com",
  issuer: "https://dev-xxx.us.auth0.com/",
  algorithms: ["RS256"],
});

module.exports = { checkJwt };
```

在路由上使用：
```js
const express = require("express");
const { checkJwt } = require("./middleware/auth");

const app = express();

// 受保护的路由
app.get("/api/tasks", checkJwt, (req, res) => {
  // req.auth 包含 JWT 的 payload
  const userId = req.auth.sub; // Auth0 用户 ID，格式是 "auth0|xxx"
  res.json({ tasks: [], userId });
});
```

**为什么**：
- `jwks-rsa` 自动从 Auth0 的 JWKS 端点获取公钥，并缓存起来。密钥轮换对你完全透明
- `express-jwt` 验证签名、过期时间、audience、issuer——你不需要写任何验证逻辑
- `req.auth` 就是 JWT payload，`sub` 字段是 Auth0 的用户唯一标识

**验证**：
```bash
# 先从前端获取 access_token（Auth0 SDK 的 getAccessTokenSilently 返回的值）
curl http://localhost:3000/api/tasks \
  -H "Authorization: Bearer eyJ..."

# 不带 token → 401 Unauthorized
curl http://localhost:3000/api/tasks
```

---

### Step 5：用 Auth0 Action 自定义 token 内容（可选，5 分钟）

**做什么**：在 JWT payload 中加入你自己的业务字段（如角色、组织 ID）。

**怎么做**：
1. Auth0 Dashboard → Actions → Flows → Login
2. Create Action → "Add custom claims"
3. 编辑代码：
```js
exports.onExecutePostLogin = async (event, api) => {
  // 从你的数据库或 Auth0 user metadata 读取角色
  const role = event.user.app_metadata?.role || "user";
  api.accessToken.setCustomClaim("https://myapp.com/role", role);
};
```
4. Deploy → 拖到 Login Flow 中 → Apply

**为什么**：
- Auth0 不允许你在自定义命名空间之外加声明（防止和标准声明冲突），所以用 `https://myapp.com/role` 这种 URI 格式
- 角色信息进 token 后，后端可以直接从 `req.auth["https://myapp.com/role"]` 读取，不用再查数据库

---

## 与 PyJWT 方案的关键区别

| 维度        | PyJWT 自管（pyjwt-fastapi.md） | Auth0 托管（本文）      |
| --------- | -------------------------- | ----------------- |
| 谁签发 token | 你的代码                       | Auth0             |
| 密钥管理      | 你自己保管 SECRET_KEY           | Auth0 管理，自动轮换     |
| 密码存储      | 你用 bcrypt 哈希存数据库           | Auth0 存，你碰不到      |
| Token 刷新  | 你写刷新接口和逻辑                  | SDK 自动处理（静默刷新）    |
| 社交登录      | 需要自己对接各平台 OAuth            | 控制台点几下开关          |
| 用户管理      | 你自己的用户表                    | Auth0 Dashboard   |
| 撤销 token  | 需要自己实现黑名单                  | Auth0 支持 token 吊销 |
| 离线可用      | 是                          | 否（需要网络访问 Auth0）   |
| 成本        | 免费                         | 免费 25000 MAU，之后付费 |

**选择原则**：如果你不想碰任何认证逻辑、需要快速上线、支持社交登录 → Auth0。如果你需要完全掌控、有合规要求不能把用户数据给第三方、或需要离线运行 → PyJWT 自管。
