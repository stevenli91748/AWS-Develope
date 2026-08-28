# Day 2 — React + TypeScript + AWS Amplify 在线前端

今天目标：

> **把 React 前端部署到 AWS Amplify，让你的浏览器可以直接访问一个真正运行在 AWS 上的 LLM 应用前端。**

最终：

```text
Windows
   │
   ▼
GitHub
   │
   ▼
AWS Amplify
   │
   ▼
React + TypeScript
   │
   ▼
HTTPS Web App
```

---

## 1. 今天完成什么

今天完成 6 件事：

```text
□ React + TypeScript 项目
□ GitHub Repository
□ AWS Amplify
□ 自动部署
□ HTTPS
□ 第一个 Chat UI
```

最终页面：

```text
┌────────────────────────────────────────────┐
│             AWS LLM Platform                │
├────────────────────────────────────────────┤
│                                            │
│  AI Assistant                              │
│                                            │
│  User: Explain RAG                         │
│                                            │
│  AI: RAG combines retrieval with...        │
│                                            │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ Ask something...                     │  │
│  └───────────────────────────────┐      │  │
│                                  │ Send │  │
│                                  └──────┘  │
└────────────────────────────────────────────┘
```

---

# 2. 创建 React 项目

打开 PowerShell。

如果你还没有 Node.js，先安装 LTS：

[Node.js 官方网站](https://nodejs.org/?utm_source=chatgpt.com)

检查：

```powershell
node --version
npm --version
```

建议：

```text
Node.js 22+
npm 10+
```

然后进入你的项目目录：

```powershell
cd C:\Users\你的用户名
```

创建 React + TypeScript：

```powershell
npm create vite@latest aws-llm-platform-frontend -- --template react-ts
```

进入：

```powershell
cd aws-llm-platform-frontend
```

安装：

```powershell
npm install
```

运行：

```powershell
npm run dev
```

看到：

```text
Local:
http://localhost:5173/
```

打开浏览器：

```text
http://localhost:5173
```

如果看到 Vite + React 页面：

**成功。**

---

# 3. 在 VS Code 打开

```powershell
code .
```

项目结构：

```text
aws-llm-platform-frontend/
│
├── public/
├── src/
│   ├── assets/
│   ├── App.tsx
│   ├── App.css
│   ├── index.css
│   └── main.tsx
│
├── package.json
├── tsconfig.json
├── vite.config.ts
└── README.md
```

---

# 4. 第一个 Chat UI

打开：

```text
src/App.tsx
```

替换为：

```tsx
import { useState } from "react";
import "./App.css";

type Message = {
  role: "user" | "assistant";
  content: string;
};

function App() {
  const [input, setInput] = useState("");
  const [messages, setMessages] = useState<Message[]>([]);

  const sendMessage = () => {
    if (!input.trim()) return;

    const userMessage: Message = {
      role: "user",
      content: input,
    };

    const assistantMessage: Message = {
      role: "assistant",
      content: "Backend API will be connected on Day 3.",
    };

    setMessages((prev) => [
      ...prev,
      userMessage,
      assistantMessage,
    ]);

    setInput("");
  };

  return (
    <div className="app">
      <header>
        <h1>AWS LLM Platform</h1>
        <p>AI Engineering Development Environment</p>
      </header>

      <main className="chat-container">
        <div className="messages">
          {messages.length === 0 && (
            <div className="welcome">
              <h2>AI Assistant</h2>
              <p>Ask me anything.</p>
            </div>
          )}

          {messages.map((message, index) => (
            <div
              key={index}
              className={`message ${message.role}`}
            >
              <strong>
                {message.role === "user" ? "You" : "AI"}
              </strong>

              <p>{message.content}</p>
            </div>
          ))}
        </div>

        <div className="input-area">
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === "Enter") {
                sendMessage();
              }
            }}
            placeholder="Ask something..."
          />

          <button onClick={sendMessage}>
            Send
          </button>
        </div>
      </main>
    </div>
  );
}

export default App;
```

---

# 5. 修改 CSS

打开：

```text
src/App.css
```

替换：

```css
.app {
  min-height: 100vh;
  background: #f5f7fa;
}

header {
  padding: 24px 40px;
  background: white;
  border-bottom: 1px solid #ddd;
}

header h1 {
  margin: 0;
}

header p {
  margin-top: 8px;
  color: #666;
}

.chat-container {
  max-width: 900px;
  margin: 40px auto;
  background: white;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.messages {
  min-height: 500px;
}

.welcome {
  text-align: center;
  padding-top: 180px;
  color: #666;
}

.message {
  margin-bottom: 20px;
  padding: 16px;
  border-radius: 10px;
}

.message.user {
  background: #eef4ff;
}

.message.assistant {
  background: #f3f3f3;
}

.message p {
  margin-bottom: 0;
}

.input-area {
  display: flex;
  gap: 12px;
  border-top: 1px solid #ddd;
  padding-top: 20px;
}

.input-area input {
  flex: 1;
  padding: 14px;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 16px;
}

.input-area button {
  padding: 14px 24px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
}
```

---

# 6. 测试

运行：

```powershell
npm run dev
```

打开：

```text
http://localhost:5173
```

输入：

```text
What is RAG?
```

点击：

```text
Send
```

现在应该看到：

```text
You
What is RAG?

AI
Backend API will be connected on Day 3.
```

注意：

**今天 AI 还没有真正连接 LLM。**

这是故意的。

Day 3 我们开始：

```text
React
  ↓
FastAPI
```

Day 5：

```text
FastAPI
  ↓
Amazon Bedrock
```

---

# 7. Git 初始化

在项目目录：

```powershell
git init
```

检查：

```powershell
git status
```

添加：

```powershell
git add .
```

提交：

```powershell
git commit -m "Day 2: create React TypeScript frontend"
```

---

# 8. 连接 GitHub

在 GitHub 创建：

```text
aws-llm-platform-frontend
```

然后：

```powershell
git branch -M main
```

设置远程仓库：

```powershell
git remote add origin YOUR_GITHUB_REPOSITORY_URL
```

例如：

```powershell
git remote -v
```

然后：

```powershell
git push -u origin main
```

进入 GitHub，确认代码已经出现。

---

# 9. AWS Amplify

进入：

[AWS Amplify Console](https://console.aws.amazon.com/amplify/?utm_source=chatgpt.com)

选择：

```text
Create new app
```

然后：

```text
GitHub
```

授权 GitHub。

选择：

```text
aws-llm-platform-frontend
```

Branch：

```text
main
```

---

# 10. Amplify Build Settings

Vite 项目通常可以使用：

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - "**/*"
  cache:
    paths:
      - node_modules/**/*
```

Amplify 会执行：

```text
GitHub
   ↓
npm ci
   ↓
npm run build
   ↓
dist/
   ↓
AWS Amplify Hosting
```

---

# 11. Deploy

点击：

```text
Save and deploy
```

等待：

```text
Provision
   ↓
Build
   ↓
Deploy
   ↓
Verify
```

最后会得到一个 AWS Amplify URL。

类似：

```text
https://main.xxxxx.amplifyapp.com
```

打开它。

如果看到：

```text
AWS LLM Platform

AI Assistant

Ask something...
```

🎉 **Day 2 完成。**

---

# 12. 现在你的架构

Day 1：

```text
AWS
 │
 ├── IAM
 ├── S3
 └── CloudShell
```

Day 2：

```text
                    Internet
                       │
                       ▼
                    GitHub
                       │
                       ▼
                 AWS Amplify
                       │
                       ▼
               React + TypeScript
                       │
                       ▼
                  Chat UI
```

---

# 13. 为什么不用 EC2 部署前端？

这是今天非常重要的工程知识。

传统方式：

```text
React
 ↓
EC2
 ↓
Nginx
 ↓
Internet
```

我们现在：

```text
React
 ↓
Amplify
 ↓
CloudFront
 ↓
Internet
```

前端是静态资源，没必要为了它维护一台 EC2。

这样：

```text
成本 ↓
维护 ↓
部署速度 ↑
HTTPS ↑
CDN ↑
CI/CD ↑
```

---

# 14. Day 2 GitHub 项目结构

现在你的 GitHub：

```text
aws-llm-platform-frontend
│
├── public/
├── src/
│   ├── App.tsx
│   ├── App.css
│   ├── index.css
│   └── main.tsx
│
├── package.json
├── tsconfig.json
├── vite.config.ts
├── README.md
└── .gitignore
```

这就是你的第一个 **AWS LLM Full-Stack 项目的 frontend repository**。

---

# 15. Day 2 面试题

### Q1. Why use React + TypeScript?

> React provides a component-based UI architecture, while TypeScript provides static typing and improves maintainability and reliability.

### Q2. Why use AWS Amplify?

> Amplify provides managed frontend hosting and Git-based CI/CD, making it easy to deploy and update web applications.

### Q3. React 和 FastAPI 怎么通信？

下一阶段：

```text
React
  │
  │ HTTP / HTTPS
  ▼
FastAPI
  │
  ▼
LLM
```

例如：

```http
POST /api/chat
Content-Type: application/json

{
  "message": "What is RAG?"
}
```

返回：

```json
{
  "answer": "RAG stands for Retrieval-Augmented Generation..."
}
```

---

# 16. 今天最重要的知识

你今天一定要理解这条：

```text
GitHub
   ↓
Amplify
   ↓
Build
   ↓
Deploy
   ↓
HTTPS
   ↓
React Application
```

这已经是一个真正的 **Cloud-based frontend development environment**。

---

## Day 2 完成标准

完成下面 7 项：

```text
□ Node.js
□ React + TypeScript
□ Vite
□ Chat UI
□ Git
□ GitHub
□ AWS Amplify deployment
```

**Day 3 的核心会非常关键：我们开始搭 `Python + FastAPI + Docker` 后端，并让你今天的 React 前端第一次通过 API 调用后端。**
