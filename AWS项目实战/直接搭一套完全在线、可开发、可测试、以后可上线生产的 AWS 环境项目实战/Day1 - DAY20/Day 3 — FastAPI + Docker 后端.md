# Day 3 — FastAPI + Docker 后端

今天开始进入真正的 **LLM Full-Stack Backend**。

目标：

```text
React
  │
  │ HTTP
  ▼
FastAPI
  │
  ▼
/api/health
/api/chat
```

**今天先不接 Bedrock。**
先把后端 API + Docker 跑通。Day 4 再把 Docker 放到 AWS ECR，Day 5 上 ECS Fargate。

---

# 1. 今天完成什么

```text
□ Python 环境
□ FastAPI
□ Pydantic
□ API
□ CORS
□ Swagger
□ pytest
□ Docker
□ GitHub
```

最终架构：

```text
Browser
   │
   ▼
React
   │
   │ POST /api/chat
   ▼
FastAPI
   │
   ├── Health Check
   ├── Chat API
   └── Future: Bedrock
```

---

# 2. 创建 Backend

你的项目建议现在变成：

```text
aws-llm-platform/
│
├── frontend/
│
└── backend/
```

如果目前前端是独立 repository，我建议从现在开始建立一个总项目：

```text
aws-llm-platform
```

进入：

```powershell
mkdir aws-llm-platform
cd aws-llm-platform

mkdir backend
cd backend
```

---

# 3. 创建 Python Virtual Environment

检查 Python：

```powershell
python --version
```

建议：

```text
Python 3.12+
```

创建虚拟环境：

```powershell
python -m venv .venv
```

激活：

```powershell
.venv\Scripts\Activate.ps1
```

如果 PowerShell 报 execution policy：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

然后再次：

```powershell
.venv\Scripts\Activate.ps1
```

看到：

```text
(.venv)
```

说明成功。

---

# 4. 安装 FastAPI

```powershell
pip install fastapi uvicorn[standard] pydantic-settings
```

测试：

```powershell
pip list
```

应该看到：

```text
fastapi
uvicorn
pydantic
pydantic-settings
```

---

# 5. 建立 Backend 结构

最终：

```text
backend/
│
├── app/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── chat.py
│   │
│   └── services/
│       ├── __init__.py
│       └── llm_service.py
│
├── tests/
│   └── test_api.py
│
├── requirements.txt
├── Dockerfile
├── .dockerignore
└── .gitignore
```

创建目录：

```powershell
mkdir app
mkdir app\api
mkdir app\schemas
mkdir app\services
mkdir tests

New-Item app\__init__.py
New-Item app\api\__init__.py
New-Item app\schemas\__init__.py
New-Item app\services\__init__.py
```

---

# 6. 创建第一个 FastAPI

`app/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title="AWS LLM Platform API",
    version="0.1.0",
    description="Backend API for an AWS-based LLM application",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.get("/api/health")
async def health_check():
    return {
        "status": "ok",
        "service": "aws-llm-platform-backend",
        "version": "0.1.0",
    }


@app.get("/")
async def root():
    return {
        "message": "AWS LLM Platform API"
    }
```

---

# 7. 启动服务器

```powershell
uvicorn app.main:app --reload
```

看到：

```text
Uvicorn running on
http://127.0.0.1:8000
```

打开：

```text
http://localhost:8000
```

应该得到：

```json
{
  "message": "AWS LLM Platform API"
}
```

---

# 8. 测试 Health API

打开：

```text
http://localhost:8000/api/health
```

应该得到：

```json
{
  "status": "ok",
  "service": "aws-llm-platform-backend",
  "version": "0.1.0"
}
```

这就是以后 Kubernetes/ECS/Load Balancer 经常使用的：

```text
Health Check
```

---

# 9. FastAPI Swagger

打开：

```text
http://localhost:8000/docs
```

你会看到：

```text
AWS LLM Platform API

GET /api/health
GET /
```

这就是 FastAPI 自动生成的 OpenAPI/Swagger UI。

这对你以后做美国 AI Engineer 面试项目非常有用。

---

# 10. 创建 Chat Schema

`app/schemas/chat.py`

```python
from pydantic import BaseModel, Field


class ChatRequest(BaseModel):
    message: str = Field(
        ...,
        min_length=1,
        max_length=4000,
        description="User message",
    )


class ChatResponse(BaseModel):
    answer: str
    model: str
```

这里使用 Pydantic 做：

```text
Request Validation
```

例如：

```json
{
  "message": "What is RAG?"
}
```

---

# 11. 创建 LLM Service

现在虽然还没有接 Bedrock，但我们先建立正确的架构。

`app/services/llm_service.py`

```python
class LLMService:

    async def generate(self, message: str) -> str:
        return (
            f"Demo response: I received your message: {message}"
        )


llm_service = LLMService()
```

为什么要这样设计？

不要把：

```text
Bedrock API
```

直接写进：

```text
FastAPI Route
```

以后：

```text
API
 ↓
LLMService
 ↓
Bedrock
```

这样更容易测试，也方便以后换：

```text
Bedrock
OpenAI
Anthropic
Gemini
Ollama
```

---

# 12. 创建 Chat API

`app/api/routes.py`

```python
from fastapi import APIRouter

from app.schemas.chat import ChatRequest, ChatResponse
from app.services.llm_service import llm_service


router = APIRouter(prefix="/api")


@router.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):

    answer = await llm_service.generate(
        request.message
    )

    return ChatResponse(
        answer=answer,
        model="demo-model",
    )
```

然后修改：

`app/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.api.routes import router


app = FastAPI(
    title="AWS LLM Platform API",
    version="0.1.0",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(router)


@app.get("/api/health")
async def health_check():
    return {
        "status": "ok",
        "service": "aws-llm-platform-backend",
        "version": "0.1.0",
    }
```

---

# 13. 测试 Chat API

重新运行：

```powershell
uvicorn app.main:app --reload
```

打开：

```text
http://localhost:8000/docs
```

找到：

```text
POST /api/chat
```

点击：

```text
Try it out
```

输入：

```json
{
  "message": "What is RAG?"
}
```

执行。

应该得到：

```json
{
  "answer": "Demo response: I received your message: What is RAG?",
  "model": "demo-model"
}
```

🎉

现在你已经有：

```text
POST /api/chat
```

---

# 14. React → FastAPI

这一步非常重要。

把 Day 2 React 的：

```text
Backend API will be connected on Day 3.
```

换成真正 API 调用。

例如：

```typescript
const sendMessage = async () => {
  if (!input.trim()) return;

  const userMessage = {
    role: "user" as const,
    content: input,
  };

  setMessages((prev) => [...prev, userMessage]);

  const response = await fetch(
    "http://localhost:8000/api/chat",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        message: input,
      }),
    }
  );

  const data = await response.json();

  setMessages((prev) => [
    ...prev,
    {
      role: "assistant",
      content: data.answer,
    },
  ]);

  setInput("");
};
```

现在：

```text
React
  │
  │ POST
  ▼
FastAPI
  │
  ▼
LLMService
  │
  ▼
Demo Response
```

---

# 15. 测试完整链路

启动 Backend：

```powershell
uvicorn app.main:app --reload
```

启动 Frontend：

```powershell
npm run dev
```

浏览器：

```text
http://localhost:5173
```

输入：

```text
Explain RAG
```

点击 Send。

如果显示：

```text
You:
Explain RAG

AI:
Demo response: I received your message: Explain RAG
```

那么：

# ✅ React → FastAPI 已经打通

---

# 16. 添加 pytest

安装：

```powershell
pip install pytest httpx
```

`tests/test_api.py`

```python
from fastapi.testclient import TestClient

from app.main import app


client = TestClient(app)


def test_health_check():
    response = client.get("/api/health")

    assert response.status_code == 200
    assert response.json()["status"] == "ok"


def test_chat():
    response = client.post(
        "/api/chat",
        json={
            "message": "What is RAG?"
        },
    )

    assert response.status_code == 200

    data = response.json()

    assert "answer" in data
    assert data["model"] == "demo-model"
```

运行：

```powershell
pytest
```

应该：

```text
2 passed
```

这一步非常重要。

你的项目开始从：

```text
Demo
```

变成：

```text
Software Engineering Project
```

---

# 17. requirements.txt

执行：

```powershell
pip freeze > requirements.txt
```

以后 Docker 会使用它。

---

# 18. Dockerfile

创建：

`Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 19. .dockerignore

创建：

`.dockerignore`

```text
.venv
__pycache__
*.pyc
.pytest_cache
.git
.env
tests
```

---

# 20. Build Docker Image

确认 Docker Desktop 已启动。

检查：

```powershell
docker --version
```

然后：

```powershell
docker build -t aws-llm-platform-backend .
```

查看：

```powershell
docker images
```

应该看到：

```text
aws-llm-platform-backend
```

---

# 21. 运行 Container

```powershell
docker run -p 8000:8000 aws-llm-platform-backend
```

打开：

```text
http://localhost:8000/api/health
```

如果返回：

```json
{
  "status": "ok",
  "service": "aws-llm-platform-backend",
  "version": "0.1.0"
}
```

说明：

# ✅ FastAPI → Docker 成功

---

# 22. 今天的最终架构

Day 1：

```text
AWS
├── IAM
├── S3
└── CloudShell
```

Day 2：

```text
GitHub
   ↓
Amplify
   ↓
React
```

Day 3：

```text
GitHub
   │
   ├──────────────┐
   ↓              ↓
Amplify          Docker
   ↓              ↓
React           FastAPI
   │              │
   └──────HTTP────┘
```

完整：

```text
                    AWS
                     │
              ┌──────┴──────┐
              │             │
          Amplify          S3
              │
            React
              │
              │ HTTP
              ▼
        FastAPI Backend
              │
              ▼
         LLM Service
              │
              │
       ┌──────┴──────┐
       │             │
    Day 5          Future
       │
    Bedrock
```

---

# 23. Day 3 项目结构

现在 GitHub 最好整理成：

```text
aws-llm-platform/
│
├── frontend/
│   ├── src/
│   ├── package.json
│   └── ...
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── schemas/
│   │   ├── services/
│   │   └── main.py
│   │
│   ├── tests/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── .dockerignore
│
├── docs/
│
└── README.md
```

---

# 24. Day 3 必须理解的 5 个概念

### ① FastAPI

```text
Python Web Framework
```

### ② API

```text
React
 ↓
HTTP
 ↓
FastAPI
```

### ③ Pydantic

```text
JSON
 ↓
Validation
 ↓
Python Object
```

### ④ Docker

```text
Application
+
Dependencies
+
Runtime
       ↓
   Container
```

### ⑤ Service Layer

```text
API Route
    ↓
Service
    ↓
LLM
```

这是后面接 Bedrock 的关键。

---

# 25. Day 3 面试题

### Q1：为什么 FastAPI？

> FastAPI is a modern Python framework for building high-performance APIs. It provides automatic OpenAPI documentation, type validation, and asynchronous support.

### Q2：为什么 Docker？

> Docker packages the application and its dependencies into a consistent container, making development, testing, and deployment more reproducible.

### Q3：为什么不能把 Bedrock 调用直接写在 API Route？

因为应该分层：

```text
Controller / Route
       ↓
Service
       ↓
LLM Provider
```

这样：

```text
Testability ↑
Maintainability ↑
Extensibility ↑
```

---

# Day 3 完成标准

你今天最终应该能证明：

```text
✅ React running
✅ FastAPI running
✅ GET /api/health
✅ POST /api/chat
✅ Swagger /docs
✅ pytest
✅ Docker build
✅ Docker run
✅ React → FastAPI
```

最关键的一条：

```text
React
  ↓
POST /api/chat
  ↓
FastAPI
  ↓
LLMService
  ↓
Response
```

**Day 4 就进入 AWS：把这个 Docker Backend 推到 ECR，并准备部署到 ECS Fargate。Day 5 再正式接 Amazon Bedrock，让 `POST /api/chat` 第一次返回真正的 LLM 回答。**
