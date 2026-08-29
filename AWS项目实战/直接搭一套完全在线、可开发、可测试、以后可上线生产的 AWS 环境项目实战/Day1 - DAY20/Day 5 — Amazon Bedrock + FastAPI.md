# Day 5 — Amazon Bedrock + FastAPI：第一次真正调用 LLM

今天是整个项目的**第一个核心 AI Day**。

目标：

```text
React
   ↓
FastAPI
   ↓
Amazon Bedrock
   ↓
Foundation Model
   ↓
LLM Response
   ↓
React
```

完成以后，你的应用不再是 Day 3 的 Demo Response，而是真正调用 AWS 大模型。

---

# 1. 今天完成什么

```text
□ 开通/确认 Bedrock Model Access
□ IAM 权限
□ boto3
□ Bedrock Runtime
□ LLM Service
□ /api/chat
□ Prompt
□ Token 参数
□ 本地测试
□ Docker 测试
□ ECS 测试
```

今天先做：

> **单模型 + 非流式 Chat**

Streaming 放到 Day 6。

---

# 2. 今天的架构

```text
                    React
                      │
                      │ POST /api/chat
                      ▼
                 FastAPI
                      │
                      ▼
                LLMService
                      │
                    boto3
                      │
                      ▼
            Amazon Bedrock Runtime
                      │
                      ▼
                Foundation Model
                      │
                      ▼
                   Response
```

---

# 3. 为什么选择 Bedrock？

以前可能是：

```text
FastAPI
   ↓
OpenAI API
```

现在：

```text
FastAPI
   ↓
AWS SDK / boto3
   ↓
Amazon Bedrock
```

优势：

* AWS IAM 管理权限
* 不需要自己部署 GPU
* 可以使用多个 Foundation Models
* 与 AWS S3 / RDS / ECS 等服务集成方便
* 更适合你的 AWS LLM Engineer 项目

Amazon Bedrock Runtime 是实际发送模型推理请求的 API 层。

---

# 4. 第一步：确认 Bedrock Model Access

进入 AWS Console：

[Amazon Bedrock Console](https://console.aws.amazon.com/bedrock/?utm_source=chatgpt.com)

Region：

```text
us-west-2
```

找到：

```text
Model access
```

或者当前控制台中的模型访问/模型目录入口。

选择一个你账户所在 Region 可用的模型。

**不要死记某个 Model ID。**

因为 Bedrock 的模型可用性和模型 ID 会随 Region、账户状态和 AWS 更新而变化。

---

# 5. 推荐 Day 5 模型

今天建议优先选择：

```text
Amazon Nova
```

或者：

```text
Anthropic Claude
```

如果你的 Bedrock Console 中可用。

例如模型 ID 可能类似：

```text
amazon.nova-lite-v1:0
```

或者 Claude 对应的当前模型 ID。

### 非常重要

不要直接复制网上旧教程里的 Model ID。

先在：

```text
Bedrock
 ↓
Model catalog
```

确认你自己的：

```text
Region
+
Model ID
+
Access
```

---

# 6. IAM 权限

进入：

[AWS IAM Console](https://console.aws.amazon.com/iam/?utm_source=chatgpt.com)

Day 1 建立的 Developer 权限可以用于学习环境。

但是我们的**生产设计**以后应该是：

```text
ECS Task
   ↓
Task Role
   ↓
bedrock:InvokeModel
```

而不是：

```text
AWS Access Key
Secret Key
放进 Docker
```

### 这一点必须记住：

**绝对不要把 AWS Secret Access Key 写进：**

```text
.env
GitHub
Dockerfile
React
README
```

---

# 7. ECS Task Role

Day 5 本地测试可以使用：

```text
AWS CLI credentials
```

但是 ECS 必须使用：

```text
ECS Task Role
```

架构：

```text
ECS Task
   │
   ▼
Task Role
   │
   ▼
Bedrock
```

这叫：

> IAM Role-based authentication

而不是：

> Hard-coded credentials

---

# 8. 创建 Bedrock Policy

IAM：

```text
Policies
 ↓
Create policy
```

JSON 可以先使用：

```json id="3ck6zn"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": "*"
    }
  ]
}
```

Policy Name：

```text id="q9d1q8"
AWSLLMPlatformBedrockPolicy
```

### 学习环境

`Resource: "*"` 可以先使用。

### 生产环境

以后应该尽量限制：

```text
specific model ARN
```

这属于 **Least Privilege**。

---

# 9. 安装 boto3

进入 backend：

```powershell id="09d31f"
cd backend
```

激活：

```powershell id="ax1f7v"
.venv\Scripts\Activate.ps1
```

安装：

```powershell id="3ymz8j"
pip install boto3
```

然后：

```powershell id="z4sn6r"
pip freeze > requirements.txt
```

---

# 10. 创建 Bedrock Client

修改：

```text id="8k0a7n"
app/services/llm_service.py
```

先写：

```python id="j8d2qx"
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

    async def generate(self, message: str) -> str:

        response = self.client.converse(
            modelId=self.model_id,
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
        )

        return response["output"]["message"]["content"][0]["text"]


llm_service = LLMService()
```

这里我们使用 Bedrock 的 **Converse API**。

它非常适合我们的应用，因为你的应用层可以保持：

```text
message
 ↓
LLM
 ↓
response
```

而不需要为每个模型写完全不同的请求结构。

AWS 对 Converse API 的说明也强调了它用于以统一接口调用支持 Converse 的模型。

---

# 11. 环境变量

创建：

```text id="h0xg3p"
.env
```

内容：

```env id="m7c6xg"
AWS_REGION=us-west-2
BEDROCK_MODEL_ID=amazon.nova-lite-v1:0
```

注意：

如果你的 Bedrock Console 显示的是其他 Model ID：

```text
BEDROCK_MODEL_ID=你的实际Model ID
```

---

# 12. `.gitignore`

确保：

```text id="f5r4j0"
.env
```

已经存在。

例如：

```text id="f8t2c9"
.venv/
.env
__pycache__/
.pytest_cache/
```

然后：

```powershell id="d8g2q6"
git status
```

**确保 `.env` 没有出现在 Git 要提交的文件里。**

---

# 13. 安装 python-dotenv

```powershell id="x0o1j9"
pip install python-dotenv
```

然后：

```powershell id="7g6j2d"
pip freeze > requirements.txt
```

修改 `app/main.py`：

```python id="c5z4d8"
from dotenv import load_dotenv

load_dotenv()
```

放在 import 区域即可。

这样本地运行：

```text id="t9g3r6"
.env
 ↓
Environment Variables
 ↓
boto3
 ↓
Bedrock
```

---

# 14. 测试 Bedrock

启动：

```powershell id="0k8x6h"
uvicorn app.main:app --reload
```

打开：

```text id="8t8h7v"
http://localhost:8000/docs
```

找到：

```text id="v2v6i3"
POST /api/chat
```

输入：

```json id="n9d6yw"
{
  "message": "Explain Retrieval-Augmented Generation in simple terms."
}
```

执行。

如果成功：

```json id="8z9e7x"
{
  "answer": "Retrieval-Augmented Generation...",
  "model": "amazon.nova-lite-v1:0"
}
```

🎉

这意味着：

# 你的第一个 AWS LLM API 已经成功。

---

# 15. 你的系统现在发生了什么？

以前：

```text id="8k1o2g"
React
 ↓
FastAPI
 ↓
Demo Response
```

现在：

```text id="gkzjhc"
React
 ↓
FastAPI
 ↓
LLMService
 ↓
boto3
 ↓
Bedrock Runtime
 ↓
Foundation Model
 ↓
LLM
 ↓
FastAPI
 ↓
React
```

这是整个项目的第一个真正 AI Pipeline。

---

# 16. 给 LLM 增加 System Prompt

不要直接：

```python
messages=[
    {
        "role": "user",
        ...
    }
]
```

我们开始建立真正的 LLM Application。

修改：

```python id="q4b7f2"
response = self.client.converse(
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
)
```

现在：

```text id="v5z7s6"
System Prompt
       ↓
User Prompt
       ↓
LLM
```

---

# 17. 增加 Generation Parameters

可以进一步控制：

```python id="2y6l11"
inferenceConfig={
    "maxTokens": 1000,
    "temperature": 0.2,
    "topP": 0.9,
}
```

完整：

```python id="0i7g2m"
response = self.client.converse(
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
```

---

# 18. 为什么 temperature = 0.2？

我们的应用是：

```text
AI Engineer Platform
```

不是：

```text
Creative Writing App
```

所以希望：

```text
temperature ↓
predictability ↑
```

例如：

```text
0.0–0.3
```

适合：

* 技术问答
* RAG
* Coding
* Documentation
* Agent Tool Calling

以后做创作类 AI 可以提高。

---

# 19. 测试几个问题

依次测试：

### Test 1

```text
What is AWS Bedrock?
```

### Test 2

```text
Explain RAG.
```

### Test 3

```text
Write a Python FastAPI endpoint.
```

### Test 4

```text
What is the difference between ECS and EKS?
```

### Test 5

```text
Explain vector embeddings.
```

确认每次都得到真正的 LLM response。

---

# 20. 添加异常处理

生产环境不能直接：

```python
return response["output"]["message"]["content"][0]["text"]
```

应该考虑：

```text id="1k1j7g"
Bedrock
 ↓
Timeout
 ↓
Throttling
 ↓
AccessDenied
 ↓
Invalid Model
 ↓
Network Error
```

先简单增加：

```python id="4f0p3x"
try:
    response = self.client.converse(
        modelId=self.model_id,
        system=[
            {
                "text": (
                    "You are a helpful AI assistant."
                )
            }
        ],
        messages=[
            {
                "role": "user",
                "content": [
                    {"text": message}
                ],
            }
        ],
        inferenceConfig={
            "maxTokens": 1000,
            "temperature": 0.2,
        },
    )

    return response["output"]["message"]["content"][0]["text"]

except Exception as exc:
    raise RuntimeError(
        f"LLM request failed: {exc}"
    ) from exc
```

Day 6 再做更正规的 exception handling。

---

# 21. 测试 pytest

你 Day 3 已经有：

```text id="qx1wzv"
tests/test_api.py
```

现在不要直接测试真实 Bedrock。

原因：

```text
pytest
 ↓
Bedrock
 ↓
$$$
```

而且测试会受到：

```text
Network
AWS
Model Availability
Rate Limits
```

影响。

所以真正工程项目应该：

```text id="t0qz1n"
Unit Test
    ↓
Mock Bedrock
```

而：

```text id="m72x4e"
Integration Test
    ↓
Real Bedrock
```

这是今天非常重要的工程概念。

---

# 22. Docker 更新

因为我们增加了：

```text
boto3
python-dotenv
```

所以：

```powershell id="p3t4cq"
docker build -t aws-llm-platform-backend .
```

运行：

```powershell id="0cq1is"
docker run --rm `
-p 8000:8000 `
-e AWS_REGION=us-west-2 `
-e BEDROCK_MODEL_ID=你的MODEL_ID `
aws-llm-platform-backend
```

但这里有一个问题：

Docker Container 必须拥有 AWS 权限。

**不要把 Access Key 写进 Dockerfile。**

本地开发可以通过 AWS CLI credential chain。

---

# 23. ECS 的正确方式

Day 4 的 ECS：

```text id="7q0y4w"
ECS
 ↓
Task
 ↓
Task Role
 ↓
Bedrock
```

Task Role 需要：

```text id="g9y5z0"
bedrock:InvokeModel
```

然后 ECS Container：

```text id="e3e8r0"
AWS_REGION=us-west-2
BEDROCK_MODEL_ID=...
```

不需要：

```text id="9otz8u"
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

---

# 24. ECS Task Role

进入：

[AWS IAM Console](https://console.aws.amazon.com/iam/?utm_source=chatgpt.com)

找到 ECS Task Role。

附加：

```text id="6qj5yz"
AWSLLMPlatformBedrockPolicy
```

最终：

```text id="x6x8z8"
ECS Task
   │
   ▼
IAM Task Role
   │
   ▼
bedrock:InvokeModel
   │
   ▼
Amazon Bedrock
```

这就是 AWS 正确的身份认证方式。

---

# 25. Docker Push

重新 Build：

```powershell id="3ik4yb"
docker build -t aws-llm-platform-backend .
```

Tag：

```powershell id="8n5s1w"
docker tag aws-llm-platform-backend:latest `
YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend:latest
```

Push：

```powershell id="lq8q9a"
docker push `
YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend:latest
```

---

# 26. ECS 更新

进入：

```text id="c8w0p4"
ECS
 ↓
Task Definitions
 ↓
aws-llm-platform-backend
 ↓
Create new revision
```

Container：

```text id="m4g8h4"
Image:
新的 ECR image

Environment:
AWS_REGION=us-west-2
BEDROCK_MODEL_ID=你的MODEL_ID
```

Task Role：

```text id="8r3b5x"
Bedrock Task Role
```

Deploy。

---

# 27. 查看 CloudWatch

进入：

[Amazon CloudWatch Console](https://console.aws.amazon.com/cloudwatch/?utm_source=chatgpt.com)

查看：

```text id="0w3wdo"
ECS
 ↓
Container Logs
```

正常应该看到：

```text id="bb7n9d"
Application startup complete
```

如果出现：

```text id="7k8f5q"
AccessDeniedException
```

重点检查：

```text id="w0n7uk"
ECS Task Role
       ↓
bedrock:InvokeModel
```

如果：

```text id="n8v1y4"
ResourceNotFoundException
```

检查：

```text id="6z8b4h"
Region
Model ID
Model Access
```

---

# 28. Day 5 完整架构

现在你的项目已经从普通 Web App 进入真正的 AI Application：

```text id="8w9e5j"
                     User
                       │
                       ▼
                    React
                       │
                       ▼
                 AWS Amplify
                       │
                       │ API
                       ▼
                ECS Fargate
                       │
                       ▼
                    FastAPI
                       │
                       ▼
                 LLMService
                       │
                       ▼
                    boto3
                       │
                       ▼
              Bedrock Runtime
                       │
                       ▼
                 Foundation
                    Model
                       │
                       ▼
                  LLM Answer
```

---

# 29. Day 5 面试题

### Q1：What is Amazon Bedrock?

> Amazon Bedrock is a fully managed AWS service that provides API access to foundation models from multiple model providers.

### Q2：Bedrock Runtime 是什么？

> Bedrock Runtime provides APIs for sending inference requests to foundation models.

### Q3：为什么 ECS 不应该保存 AWS Access Key？

因为：

```text
Access Key
   ↓
长期凭证
   ↓
泄露风险
```

正确：

```text
ECS Task
   ↓
IAM Task Role
   ↓
Temporary Credentials
```

### Q4：为什么使用 Converse API？

> The Converse API provides a consistent interface for conversational requests across supported foundation models.

### Q5：Unit Test 为什么不能每次调用 Bedrock？

因为：

```text
Cost
Latency
Network dependency
Rate limits
Non-deterministic output
```

所以：

```text
Unit Test
   ↓
Mock

Integration Test
   ↓
Real Bedrock
```

---

# 30. Day 5 完成标准

完成这些：

```text
□ Bedrock Model Access
□ IAM Bedrock Policy
□ boto3
□ Bedrock Runtime
□ Converse API
□ System Prompt
□ Temperature / maxTokens
□ /api/chat → Real LLM
□ Docker
□ ECS Task Role
□ CloudWatch
```

最终你应该可以做到：

```text
打开浏览器
     ↓
React Chat
     ↓
输入：

"Explain RAG"
     ↓
FastAPI
     ↓
Amazon Bedrock
     ↓
真实 LLM
     ↓
返回答案
```

# 🎯 Day 5 的里程碑

到这里，你已经完成：

```text
Day 1  AWS 基础
Day 2  React + Amplify
Day 3  FastAPI + Docker
Day 4  ECR + ECS Fargate
Day 5  Bedrock LLM
```

也就是：

> **AWS + React + FastAPI + Docker + ECS + Bedrock 的完整 LLM Application 基础架构已经建立。**

**Day 6 我们把 Chat 改成真正的 Streaming Response（类似 ChatGPT 一样一个字/一段一段返回），同时解决 Amplify → ECS 的 HTTPS、CORS 和 API Gateway/ALB 问题。**
