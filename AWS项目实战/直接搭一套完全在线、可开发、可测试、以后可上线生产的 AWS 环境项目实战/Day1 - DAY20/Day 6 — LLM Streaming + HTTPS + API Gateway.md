# Day 6 — LLM Streaming + HTTPS + API Gateway

今天开始把你的项目从“能调用 LLM”升级成**真正像 ChatGPT 一样的 LLM Web Application**。

今天目标：

```text
React
   │
   │ HTTPS
   ▼
API Gateway
   │
   ▼
ECS Fargate
   │
   ▼
FastAPI
   │
   ▼
Amazon Bedrock
   │
   │ Streaming
   ▼
React Chat UI
```

最终效果：

```text
You:
Explain RAG

AI:
RAG stands for...
        ↓
Retrieval-Augmented...
        ↓
Generation...
```

而不是等模型全部生成以后一次性返回。

---

# 1. 今天完成 7 件事

```text
□ Bedrock Streaming
□ FastAPI StreamingResponse
□ React Streaming UI
□ API Gateway
□ HTTPS
□ CORS
□ ECS → API Gateway → React
```

今天的重点是 **Streaming + Cloud API 架构**。

---

# 2. 为什么需要 Streaming？

Day 5：

```text
React
 ↓
FastAPI
 ↓
Bedrock
 ↓
等待 5 秒
 ↓
完整答案
 ↓
React
```

用户体验：

```text
[Loading...............]
```

Streaming：

```text
React
 ↑
 ↑ chunk 1
 ↑ chunk 2
 ↑ chunk 3
 ↑ chunk 4
 ↑
FastAPI
 ↑
Bedrock
```

用户体验：

```text
AI:
RAG
RAG is
RAG is a technique
RAG is a technique that...
```

---

# 3. Bedrock Streaming API

Day 5 使用：

```text
converse()
```

今天使用：

```text
converse_stream()
```

核心：

```python
response = client.converse_stream(...)
```

然后：

```python
for event in response["stream"]:
    ...
```

AWS 官方 Converse Streaming API：

[Amazon Bedrock ConverseStream API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ConverseStream.html?utm_source=chatgpt.com)

---

# 4. 修改 LLMService

打开：

```text
backend/app/services/llm_service.py
```

改成：

```python
import os
import boto3


class LLMService:

    def __init__(self):
        self.region = os.getenv(
            "AWS_REGION",
            "us-west-2",
        )

        self.model_id = os.getenv(
            "BEDROCK_MODEL_ID",
            "amazon.nova-lite-v1:0",
        )

        self.client = boto3.client(
            "bedrock-runtime",
            region_name=self.region,
        )

    def stream_generate(self, message: str):

        response = self.client.converse_stream(
            modelId=self.model_id,

            system=[
                {
                    "text": (
                        "You are a helpful AI assistant "
                        "for an AWS LLM engineering platform."
                    )
                }
            ],

            messages=[
                {
                    "role": "user",
                    "content": [
                        {
                            "text": message
                        }
                    ],
                }
            ],

            inferenceConfig={
                "maxTokens": 1000,
                "temperature": 0.2,
                "topP": 0.9,
            },
        )

        for event in response["stream"]:

            if "contentBlockDelta" in event:

                delta = event["contentBlockDelta"]

                text = delta.get("delta", {}).get("text")

                if text:
                    yield text


llm_service = LLMService()
```

---

# 5. FastAPI Streaming Endpoint

修改：

```text
app/api/routes.py
```

增加：

```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse

from app.schemas.chat import ChatRequest
from app.services.llm_service import llm_service


router = APIRouter(prefix="/api")


@router.post("/chat/stream")
async def chat_stream(request: ChatRequest):

    return StreamingResponse(
        llm_service.stream_generate(
            request.message
        ),
        media_type="text/plain",
    )
```

现在：

```text
POST /api/chat/stream
```

会逐段返回：

```text
RAG
is
a
technique
...
```

---

# 6. 测试 FastAPI Streaming

启动：

```powershell
uvicorn app.main:app --reload
```

Swagger：

```text
http://localhost:8000/docs
```

注意：

Swagger 对 Streaming 的体验不是特别好。

所以可以用 curl：

```powershell
curl.exe -N `
-X POST `
http://localhost:8000/api/chat/stream `
-H "Content-Type: application/json" `
-d "{\"message\":\"Explain RAG in simple terms.\"}"
```

如果看到内容逐步出现：

```text
RAG
is
a
retrieval...
```

说明：

# ✅ Bedrock Streaming → FastAPI 成功

---

# 7. 为什么使用 `StreamingResponse`？

普通：

```text
JSON Response
```

是：

```text
FastAPI
 ↓
等待全部结果
 ↓
JSON
 ↓
Browser
```

Streaming：

```text
FastAPI
 ↓
chunk
 ↓
Browser

chunk
 ↓
Browser

chunk
 ↓
Browser
```

这对于：

* LLM Chat
* Agent
* Coding Assistant
* RAG
* Long generation

非常重要。

---

# 8. React Streaming

Day 5 的：

```typescript
const response = await fetch(...)
const data = await response.json()
```

不能继续用了。

因为现在服务器不是一次返回 JSON。

我们使用：

```text
response.body
 ↓
ReadableStream
 ↓
Reader
 ↓
TextDecoder
```

---

# 9. React Streaming Code

在 React 中：

```typescript
const streamChat = async () => {
  if (!input.trim()) return;

  const message = input;

  setMessages((prev) => [
    ...prev,
    {
      role: "user",
      content: message,
    },
    {
      role: "assistant",
      content: "",
    },
  ]);

  setInput("");

  const response = await fetch(
    `${API_BASE_URL}/api/chat/stream`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        message,
      }),
    }
  );

  if (!response.body) {
    throw new Error("Streaming is not supported");
  }

  const reader = response.body.getReader();

  const decoder = new TextDecoder();

  while (true) {
    const { value, done } =
      await reader.read();

    if (done) break;

    const chunk =
      decoder.decode(value, {
        stream: true,
      });

    setMessages((prev) => {
      const updated = [...prev];

      const last = updated.length - 1;

      updated[last] = {
        ...updated[last],
        content:
          updated[last].content + chunk,
      };

      return updated;
    });
  }
};
```

---

# 10. API URL 不要写死

不要：

```typescript
const API_BASE_URL =
  "http://54.xxx.xxx.xxx:8000";
```

应该：

```text
.env.development
```

```env
VITE_API_BASE_URL=http://localhost:8000
```

以后：

```text
.env.production
```

```env
VITE_API_BASE_URL=https://api.yourdomain.com
```

代码：

```typescript
const API_BASE_URL =
  import.meta.env.VITE_API_BASE_URL;
```

这是非常重要的工程习惯。

---

# 11. 现在解决 HTTPS

Day 4：

```text
Amplify
   ↓
HTTP
   ↓
ECS Public IP:8000
```

不适合生产。

我们改成：

```text
React
   ↓
HTTPS
   ↓
API Gateway
   ↓
ECS
   ↓
FastAPI
```

最终：

```text
https://api.yourdomain.com
```

---

# 12. API Gateway

进入：

[Amazon API Gateway Console](https://console.aws.amazon.com/apigateway/?utm_source=chatgpt.com)

创建：

```text
Create API
```

建议：

```text
HTTP API
```

名字：

```text
aws-llm-platform-api
```

---

# 13. API Gateway 路由

建立：

```text
POST /api/chat
POST /api/chat/stream
GET  /api/health
```

架构：

```text
API Gateway
│
├── GET /api/health
│
├── POST /api/chat
│
└── POST /api/chat/stream
```

---

# 14. 这里有一个重要工程问题

**API Gateway 并不是所有类型的 Streaming 都适合直接代理。**

对于真正的 token-by-token LLM streaming，需要认真选择 AWS 的 API/compute 组合以及响应流能力。

因此今天学习阶段可以先保持：

```text
Local:
React → FastAPI → Bedrock Streaming
```

而 AWS 公网 API 第一版：

```text
React
 ↓
API Gateway
 ↓
ECS
 ↓
FastAPI
```

先验证普通 `/api/chat`。

Streaming 公网链路我们在后面的生产化阶段再专门处理，避免把一个学习 Day 搞成架构坑。

---

# 15. CORS

你的前端：

```text
https://xxxxx.amplifyapp.com
```

后端：

```text
https://api.yourdomain.com
```

这是不同 Origin。

所以 FastAPI：

```python
app.add_middleware(
    CORSMiddleware,

    allow_origins=[
        "https://xxxxx.amplifyapp.com"
    ],

    allow_credentials=True,

    allow_methods=["*"],

    allow_headers=["*"],
)
```

开发：

```text
http://localhost:5173
```

生产：

```text
https://你的Amplify域名
```

---

# 16. 不要这样写生产 CORS

不要：

```python
allow_origins=["*"]
```

生产应该：

```text
React Domain
       ↓
CORS Allowlist
```

例如：

```python
allow_origins=[
    "https://main.xxxxx.amplifyapp.com"
]
```

---

# 17. HTTPS 域名

最终建议：

```text
Frontend:

https://app.yourdomain.com


Backend:

https://api.yourdomain.com
```

架构：

```text
                    Internet
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
       app.yourdomain.com   api.yourdomain.com
              │                 │
              ▼                 ▼
          Amplify          API Gateway
                                │
                                ▼
                          ECS Fargate
                                │
                                ▼
                             FastAPI
                                │
                                ▼
                            Bedrock
```

如果你目前没有自己的域名：

**今天可以先使用 Amplify 提供的域名和 API Gateway 默认域名。**

---

# 18. API Gateway 的作用

不要把 API Gateway 理解成：

> “另一个 FastAPI。”

它的作用更像：

```text
Internet
   │
   ▼
API Gateway
   │
   ├── Authentication
   ├── Rate limiting
   ├── Routing
   ├── Monitoring
   └── Backend integration
```

然后：

```text
API Gateway
      ↓
ECS
```

---

# 19. 今天加入一个重要的 Header

以后我们会加入：

```http
Authorization: Bearer <JWT>
```

今天暂时：

```http
Content-Type: application/json
```

Day 7 开始 Cognito。

最终：

```text
React
 ↓
Cognito Login
 ↓
JWT
 ↓
API Gateway
 ↓
FastAPI
```

---

# 20. 测试完整系统

### Frontend

```text
https://xxxxx.amplifyapp.com
```

输入：

```text
Explain RAG.
```

### API

```text
POST /api/chat
```

### Backend

```text
ECS
 ↓
FastAPI
```

### AI

```text
Bedrock
 ↓
Foundation Model
```

### Response

```text
AI:
RAG is...
```

---

# 21. CloudWatch

今天继续看：

```text
CloudWatch
 ↓
Logs
 ↓
ECS
```

重点观察：

```text
Request
Latency
Exception
Bedrock Error
```

以后我们会加入：

```text
request_id
user_id
model
latency
token_usage
```

---

# 22. 给每个请求加 Request ID

这是今天值得提前做的工程实践。

FastAPI：

```python
import uuid

from fastapi import Request


@app.middleware("http")
async def add_request_id(
    request: Request,
    call_next,
):
    request_id = str(uuid.uuid4())

    response = await call_next(request)

    response.headers[
        "X-Request-ID"
    ] = request_id

    return response
```

现在：

```text
React
 ↓
Request
 ↓
request_id
 ↓
FastAPI
 ↓
Bedrock
```

以后出现问题：

```text
User:
"My AI request failed."

Engineer:
"Give me your request ID."

User:
"abc-123"

Engineer:
CloudWatch → abc-123
```

这就是生产系统的思路。

---

# 23. Day 6 测试

### Test 1

```text
Explain RAG.
```

### Test 2

```text
Explain AWS ECS Fargate.
```

### Test 3

```text
Write a FastAPI health check endpoint.
```

### Test 4

```text
Explain vector embeddings.
```

确认：

```text
□ Streaming
□ No API errors
□ Correct CORS
□ CloudWatch logs
```

---

# 24. Day 6 面试题

### Q1：为什么 LLM Application 使用 Streaming？

> Streaming improves user experience by displaying generated content incrementally instead of waiting for the entire model response.

### Q2：什么是 `ReadableStream`？

> It allows the browser to consume response data incrementally as chunks become available.

### Q3：为什么 API Gateway？

> API Gateway provides a managed entry point for backend APIs and can handle routing, authorization, throttling, monitoring, and integration with backend services.

### Q4：什么是 CORS？

> CORS is a browser security mechanism that controls which origins are allowed to access resources from another origin.

### Q5：为什么生产环境不能直接 ECS Public IP？

因为：

```text
Public IP
 ↓
绕过统一 API Layer
 ↓
安全/认证/路由/监控难管理
```

生产：

```text
Internet
 ↓
HTTPS
 ↓
API Gateway / ALB
 ↓
Private ECS
```

---

# 25. Day 6 最终架构

```text
                         USER
                          │
                          ▼
                  ┌──────────────┐
                  │    React     │
                  │ TypeScript   │
                  └──────┬───────┘
                         │
                       HTTPS
                         │
                         ▼
                  ┌──────────────┐
                  │ API Gateway  │
                  └──────┬───────┘
                         │
                         ▼
                  ┌──────────────┐
                  │ ECS Fargate  │
                  │              │
                  │   FastAPI    │
                  └──────┬───────┘
                         │
                    boto3 / AWS SDK
                         │
                         ▼
                  ┌──────────────┐
                  │   Bedrock    │
                  │ Foundation   │
                  │    Model     │
                  └──────┬───────┘
                         │
                      Streaming
                         │
                         ▼
                       USER
```

---

# 26. Day 1 → Day 6

你现在已经走了一个很完整的路线：

```text
Day 1
AWS / IAM / S3
       ↓
Day 2
React / TypeScript / Amplify
       ↓
Day 3
FastAPI / Pydantic / Docker
       ↓
Day 4
ECR / ECS / Fargate
       ↓
Day 5
Bedrock / LLM / Converse API
       ↓
Day 6
Streaming / API Gateway / HTTPS
```

这已经不是单纯的“学 AWS”，而是在搭一个真正的：

> **Cloud-native LLM Application Platform**

---

## Day 6 完成标准

```text
□ Bedrock converse_stream()
□ FastAPI StreamingResponse
□ React ReadableStream
□ /api/chat/stream
□ API Gateway
□ HTTPS
□ CORS
□ CloudWatch
□ Request ID
□ GitHub commit
```

### Day 7

下一步开始 **Cognito + JWT Authentication + 用户系统**：

```text
React
   ↓
Cognito Login
   ↓
JWT
   ↓
API Gateway
   ↓
FastAPI
   ↓
Bedrock
```

然后每个用户可以拥有自己的：

```text
Conversation
Chat History
Documents
RAG Knowledge Base
Agent Sessions
```

这一步之后，你的项目就开始进入真正的**多用户 LLM SaaS 架构**。
