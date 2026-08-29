# Day 7 — Cognito + JWT：给 LLM 应用加入用户登录

今天把你的项目从：

```text
React
  ↓
API
  ↓
LLM
```

升级成：

```text
User
 ↓
Cognito Login
 ↓
JWT
 ↓
API Gateway
 ↓
ECS/FastAPI
 ↓
Bedrock
```

这是从 **Demo → 多用户 AI SaaS** 的关键一步。

---

# 1. 今天完成什么

```text
□ Amazon Cognito User Pool
□ 注册/登录
□ JWT Access Token
□ React 登录
□ API Gateway JWT Authorizer
□ FastAPI 获取用户身份
□ /api/me
□ /api/chat 用户隔离
□ 测试认证失败/成功
```

今天**先不做数据库**。Day 8 再加入 PostgreSQL。

---

# 2. 今天最终架构

```text
                    User
                     │
                     ▼
               React / Amplify
                     │
                     ▼
              Amazon Cognito
                     │
                  Login
                     │
                     ▼
                  JWT
                     │
                     ▼
              API Gateway
                     │
              JWT Authorizer
                     │
                     ▼
              ECS Fargate
                     │
                   FastAPI
                     │
                     ▼
              Amazon Bedrock
```

Cognito User Pools 可以作为用户目录和身份认证服务，并支持 OAuth 2.0/OIDC 等标准身份能力。

---

# 3. 创建 Cognito User Pool

进入：

[Amazon Cognito Console](https://console.aws.amazon.com/cognito/?utm_source=chatgpt.com)

选择：

```text
User pools
   ↓
Create user pool
```

名字：

```text
aws-llm-platform-users
```

---

# 4. Sign-in options

选择：

```text
Email
```

也可以：

```text
Username
```

今天推荐：

```text
Email
```

因为你的应用最终是：

```text
AI SaaS
```

用户一般使用：

```text
email + password
```

---

# 5. Password Policy

开发环境可以：

```text
Minimum length:
8
```

生产环境建议：

```text
12+
```

并启用：

```text
Uppercase
Lowercase
Number
Special character
```

---

# 6. MFA

Day 7：

```text
MFA:
Optional
```

先不要把 MFA 搞复杂。

后面生产阶段再加入：

```text
TOTP
SMS
Passkeys
```

---

# 7. 创建 User Pool

名称：

```text
aws-llm-platform-users
```

完成后你会得到：

```text
User Pool ID

例如：
us-west-2_xxxxx
```

记录：

```text
AWS_COGNITO_USER_POOL_ID
```

---

# 8. 创建 App Client

进入：

```text
User Pool
 ↓
App clients
 ↓
Create app client
```

名字：

```text
aws-llm-platform-web
```

应用类型：

```text
Single-page application
```

不要生成 Client Secret。

因为：

> React 是 public client。

Browser 中不能安全保存：

```text
client_secret
```

---

# 9. Cognito Domain

进入：

```text
Branding
 / Domain
```

创建一个 Cognito Hosted UI domain。

例如：

```text
aws-llm-platform-xxxxx.auth.us-west-2.amazoncognito.com
```

具体域名以你的 AWS Console 分配为准。

---

# 10. OAuth 配置

设置：

```text
OAuth 2.0 grant types

Authorization code grant
```

Scope：

```text
openid
email
profile
```

最终：

```text
React
 ↓
Cognito Hosted UI
 ↓
Login
 ↓
Authorization Code
 ↓
JWT
```

---

# 11. Callback URL

开发环境：

```text
http://localhost:5173/
```

生产环境：

```text
https://你的-amplify-domain/
```

例如：

```text
https://main.xxxxx.amplifyapp.com/
```

注意：

**Callback URL 必须和 Cognito 配置完全匹配。**

---

# 12. React 安装 Amplify

进入：

```powershell
cd frontend
```

安装：

```powershell
npm install aws-amplify
```

---

# 13. 创建 Cognito 配置

例如：

```text
src/config/amplify.ts
```

```typescript
import { Amplify } from "aws-amplify";

Amplify.configure({
  Auth: {
    Cognito: {
      userPoolId:
        import.meta.env.VITE_COGNITO_USER_POOL_ID,

      userPoolClientId:
        import.meta.env.VITE_COGNITO_CLIENT_ID,

      loginWith: {
        oauth: {
          domain:
            import.meta.env.VITE_COGNITO_DOMAIN,

          scopes: [
            "openid",
            "email",
            "profile",
          ],

          redirectSignIn: [
            window.location.origin + "/",
          ],

          redirectSignOut: [
            window.location.origin + "/",
          ],

          responseType: "code",
        },
      },
    },
  },
});
```

---

# 14. 环境变量

`.env.development`

```env
VITE_COGNITO_USER_POOL_ID=us-west-2_xxxxx
VITE_COGNITO_CLIENT_ID=xxxxxxxx
VITE_COGNITO_DOMAIN=xxxxx.auth.us-west-2.amazoncognito.com
```

注意：

React 中的：

```text
VITE_*
```

**不是秘密。**

但是：

```text
AWS Secret Access Key
Cognito Client Secret
Database Password
```

绝对不能放进去。

---

# 15. 初始化 Amplify

在：

```text
src/main.tsx
```

加入：

```typescript
import "./config/amplify";
```

例如：

```typescript
import React from "react";
import ReactDOM from "react-dom/client";

import "./config/amplify";
import App from "./App";

ReactDOM.createRoot(
  document.getElementById("root")!
).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

---

# 16. Login Button

创建：

```text
src/components/LoginButton.tsx
```

```typescript
import { signInWithRedirect } from "aws-amplify/auth";

export default function LoginButton() {

  const login = async () => {
    await signInWithRedirect();
  };

  return (
    <button onClick={login}>
      Login
    </button>
  );
}
```

点击：

```text
Login
 ↓
Cognito
 ↓
Email
Password
 ↓
Redirect
 ↓
React
```

---

# 17. Logout

```typescript
import { signOut } from "aws-amplify/auth";

export default function LogoutButton() {

  const logout = async () => {
    await signOut();
  };

  return (
    <button onClick={logout}>
      Logout
    </button>
  );
}
```

---

# 18. 获取当前用户

```typescript
import { getCurrentUser } from "aws-amplify/auth";

const user = await getCurrentUser();

console.log(user);
```

可能看到：

```text
username
userId
signInDetails
```

---

# 19. 获取 JWT

最重要的代码：

```typescript
import { fetchAuthSession } from "aws-amplify/auth";

const session = await fetchAuthSession();

const accessToken =
  session.tokens?.accessToken?.toString();
```

得到：

```text
eyJraWQiOi...
```

这就是 JWT。

---

# 20. React 调用后端

Day 6：

```typescript
fetch(
  `${API_BASE_URL}/api/chat`
)
```

现在：

```typescript
const session =
  await fetchAuthSession();

const token =
  session.tokens?.accessToken?.toString();

const response = await fetch(
  `${API_BASE_URL}/api/chat`,
  {
    method: "POST",

    headers: {
      "Content-Type": "application/json",

      Authorization:
        `Bearer ${token}`,
    },

    body: JSON.stringify({
      message: input,
    }),
  }
);
```

于是：

```text
React
  │
  │ Authorization: Bearer JWT
  ▼
API Gateway
```

---

# 21. API Gateway JWT Authorizer

进入：

[Amazon API Gateway Console](https://console.aws.amazon.com/apigateway/?utm_source=chatgpt.com)

选择：

```text
aws-llm-platform-api
```

创建：

```text
Authorizers
 ↓
Create
```

类型：

```text
JWT
```

Issuer：

```text
https://cognito-idp.us-west-2.amazonaws.com/YOUR_USER_POOL_ID
```

Audience：

```text
YOUR_COGNITO_APP_CLIENT_ID
```

---

# 22. JWT 验证过程

用户请求：

```text
Authorization:
Bearer eyJ...
```

API Gateway：

```text
JWT
 ↓
Signature
 ↓
Issuer
 ↓
Audience
 ↓
Expiration
```

验证成功：

```text
→ ECS
```

失败：

```text
→ 401 Unauthorized
```

所以：

```text
没有 JWT
     ↓
     ❌
```

```text
JWT 无效
     ↓
     ❌
```

```text
JWT 正确
     ↓
     ✅
ECS
```

---

# 23. 给 API Route 加 Authorizer

例如：

```text
POST /api/chat
```

设置：

```text
Authorization:
JWT
```

同样：

```text
POST /api/chat/stream
```

设置：

```text
JWT
```

但是：

```text
GET /api/health
```

可以保持：

```text
No authorization
```

因为 Health Check 不需要登录。

最终：

```text
/api/health
   ↓
Public

/api/chat
   ↓
JWT

/api/chat/stream
   ↓
JWT
```

---

# 24. FastAPI 读取用户信息

虽然 API Gateway 已经验证 JWT，但后端仍然需要知道：

> **到底是哪一个用户？**

API Gateway 会把 JWT claims 传递给后端集成。

FastAPI 可以从 headers/context 中获取相关身份信息，具体取决于你配置的 API Gateway 集成。

学习阶段先建立一个简单依赖：

```python
from fastapi import Header, HTTPException


async def get_user_id(
    x_user_id: str | None = Header(
        default=None
    )
):

    if not x_user_id:
        raise HTTPException(
            status_code=401,
            detail="Unauthorized",
        )

    return x_user_id
```

不过这里要特别注意：

**不要在生产环境直接信任客户端自己发送的 `X-User-ID`。**

生产环境应该使用：

```text
API Gateway verified JWT claims
            ↓
trusted identity
            ↓
FastAPI
```

而不是：

```text
Browser
 ↓
X-User-ID: hacker
```

---

# 25. `/api/me`

创建：

```python
@router.get("/me")
async def get_me():
    return {
        "message": "Authenticated user"
    }
```

最终我们希望：

```text
GET /api/me
Authorization: Bearer JWT
```

返回：

```json
{
  "user_id": "xxxxxxxx",
  "email": "user@example.com"
}
```

Day 7 可以先完成 API Gateway 认证。

Day 8 我们把真实 claims → FastAPI → PostgreSQL 完整打通。

---

# 26. 一个非常重要的概念：JWT 不等于 Session

传统 Web：

```text
Login
 ↓
Server Session
 ↓
Session ID
```

现在：

```text
Login
 ↓
Cognito
 ↓
JWT
 ↓
API
```

JWT 中包含 claims，例如：

```text
sub
email
iss
aud
exp
```

其中：

```text
sub
```

通常是用户的稳定唯一标识。

以后数据库：

```text
users
```

可以：

```text
user_id = Cognito sub
```

---

# 27. 用户隔离

这是今天最重要的 SaaS 思维。

以后：

```text
User A
 ↓
Chat A

User B
 ↓
Chat B
```

数据库：

```text
conversations

id
user_id
title
created_at
```

例如：

```text
conversation_id | user_id
----------------|---------
1               | AAA
2               | AAA
3               | BBB
```

User A：

```sql
WHERE user_id = 'AAA'
```

只能看到：

```text
1
2
```

看不到：

```text
3
```

这叫：

> **Tenant/User Data Isolation**

---

# 28. `/api/chat` 最终应该变成

```text
JWT
 ↓
user_id
 ↓
ChatService
 ↓
Conversation
 ↓
LLM
```

而不是：

```text
message
 ↓
LLM
```

最终：

```python
await chat_service.chat(
    user_id=user_id,
    message=request.message,
)
```

---

# 29. 测试认证

### Test 1 — 没有 Token

```http
POST /api/chat
```

结果：

```text
401 Unauthorized
```

### Test 2 — 错误 Token

```http
Authorization: Bearer abc
```

结果：

```text
401 Unauthorized
```

### Test 3 — 正确 Token

```http
Authorization: Bearer eyJ...
```

结果：

```text
200 OK
```

---

# 30. 浏览器测试

现在整个流程：

```text
打开：
http://localhost:5173
```

点击：

```text
Login
```

跳转：

```text
Cognito
```

输入：

```text
Email
Password
```

登录成功：

```text
Cognito
 ↓
React
```

React：

```typescript
await fetchAuthSession()
```

拿到：

```text
JWT
```

然后：

```text
POST /api/chat
Authorization: Bearer JWT
```

---

# 31. 今天的安全原则

记住这 5 条：

```text
1. 不在 React 保存 AWS Secret Key

2. 不在 Docker 保存 AWS Secret Key

3. ECS 使用 IAM Task Role

4. API 使用 JWT Authentication

5. 后端根据 JWT identity 隔离用户数据
```

这五条在美国 AI Engineer / Cloud Engineer 面试中都很重要。

---

# 32. Day 7 项目目录

现在建议：

```text
aws-llm-platform/
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── LoginButton.tsx
│   │   │   ├── LogoutButton.tsx
│   │   │   └── Chat.tsx
│   │   │
│   │   ├── config/
│   │   │   └── amplify.ts
│   │   │
│   │   └── App.tsx
│   │
│   └── .env.development
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   └── routes.py
│   │   │
│   │   ├── services/
│   │   │   └── llm_service.py
│   │   │
│   │   ├── schemas/
│   │   │   └── chat.py
│   │   │
│   │   └── main.py
│   │
│   ├── tests/
│   │
│   ├── Dockerfile
│   └── requirements.txt
│
└── README.md
```

---

# 33. Git Commit

完成后：

```powershell
git add .
```

```powershell
git commit -m "Day 7: add Cognito authentication and JWT authorization"
```

```powershell
git push
```

README 增加：

```markdown
## Authentication

Authentication:
- Amazon Cognito
- OAuth 2.0
- OpenID Connect
- JWT
- API Gateway JWT Authorizer
```

---

# 34. Day 7 面试题

### Q1：Cognito User Pool 是什么？

> A managed user directory and authentication service for web and mobile applications.

### Q2：为什么 React 不能保存 Client Secret？

> A browser-based SPA is a public client, so secrets cannot be safely kept in browser code.

### Q3：JWT 有什么作用？

> JWT carries authenticated identity and claims that APIs can validate and use for authorization.

### Q4：为什么 API Gateway 验证 JWT？

> It prevents unauthenticated requests from reaching backend services and centralizes API authorization.

### Q5：为什么后端还需要 user ID？

因为：

```text
Authentication
    ↓
Who are you?

Authorization
    ↓
What can you access?
```

例如：

```text
User A
 ↓
Conversation A

User B
 ↓
Conversation B
```

---

# 35. Day 7 最终架构

```text
                         USER
                           │
                           ▼
                    ┌────────────┐
                    │   React    │
                    │  Amplify   │
                    └─────┬──────┘
                          │
                          ▼
                    ┌────────────┐
                    │  Cognito   │
                    │ User Pool  │
                    └─────┬──────┘
                          │
                         JWT
                          │
                          ▼
                    ┌────────────┐
                    │API Gateway │
                    │JWT Auth    │
                    └─────┬──────┘
                          │
                          ▼
                    ┌────────────┐
                    │    ECS     │
                    │  FastAPI   │
                    └─────┬──────┘
                          │
                          ▼
                    ┌────────────┐
                    │  Bedrock   │
                    │    LLM     │
                    └────────────┘
```

---

# Day 7 完成标准

```text
□ Cognito User Pool
□ Cognito App Client
□ Hosted UI
□ OAuth Authorization Code
□ React Login
□ React Logout
□ fetchAuthSession()
□ JWT
□ API Gateway JWT Authorizer
□ /api/chat authentication
□ 401 test
□ Authenticated request test
□ GitHub commit
```

### 下一步 Day 8

我们把真正的数据库加入：

```text
Cognito
   ↓
user_id
   ↓
PostgreSQL / Amazon RDS
   ↓
Users
Conversations
Messages
   ↓
FastAPI
   ↓
Bedrock
```

然后你的 LLM 应用就开始具备真正的 **ChatGPT-style conversation history + 多用户数据隔离**。
