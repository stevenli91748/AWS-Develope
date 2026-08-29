# Day 4 — ECR + ECS Fargate：把 FastAPI 后端放进 AWS

今天是一个关键节点：

> **把 Day 3 本地 Docker FastAPI，真正部署到 AWS。**

今天先不接 Bedrock。完成后，你会拥有一个可以通过公网 HTTPS/API 访问的 AWS 后端。

---

## 1. 今天的目标架构

```text
GitHub
   │
   ▼
Docker Image
   │
   ▼
Amazon ECR
   │
   ▼
ECS Fargate
   │
   ▼
FastAPI
   │
   ▼
/api/health
/api/chat
```

完整环境：

```text
                 Internet
                    │
                    ▼
              AWS ECS Fargate
                    │
              ┌─────┴─────┐
              │ FastAPI   │
              │ Container │
              └─────┬─────┘
                    │
             Day 5 → Bedrock
```

---

# 2. 今天学习的 AWS 服务

| 服务             | 用途              |
| -------------- | --------------- |
| ECR            | 保存 Docker Image |
| ECS            | 管理 Container    |
| Fargate        | 无服务器 Container  |
| IAM            | AWS 权限          |
| VPC            | 网络              |
| Security Group | 防火墙             |
| CloudWatch     | Container Logs  |

AWS 官方对 ECS Fargate 的定位就是运行容器而无需管理底层服务器。

---

# 3. 先确认 Day 3 Docker

进入：

```powershell
cd backend
```

检查：

```powershell
docker images
```

应该看到：

```text
aws-llm-platform-backend
```

运行：

```powershell
docker run -p 8000:8000 aws-llm-platform-backend
```

测试：

```text
http://localhost:8000/api/health
```

成功以后 `Ctrl+C`。

---

# 4. 创建 ECR Repository

AWS Console：

[Amazon ECR Console](https://console.aws.amazon.com/ecr/?utm_source=chatgpt.com)

选择：

```text
Repositories
    ↓
Create repository
```

名称：

```text
aws-llm-platform-backend
```

设置：

```text
Visibility:
Private
```

然后：

```text
Create
```

最终：

```text
ECR
└── aws-llm-platform-backend
```

---

# 5. 登录 ECR

AWS Console 打开你的 ECR Repository。

点击：

```text
View push commands
```

AWS 会给你当前 Region 对应的命令。

PowerShell 执行类似：

```powershell
aws ecr get-login-password --region us-west-2 |
docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com
```

成功：

```text
Login Succeeded
```

---

# 6. 给 Docker Image 打 Tag

假设你的 AWS Account ID 是：

```text
123456789012
```

执行：

```powershell
docker tag `
aws-llm-platform-backend:latest `
123456789012.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend:latest
```

检查：

```powershell
docker images
```

应该有：

```text
aws-llm-platform-backend
123456789012.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend
```

---

# 7. Push 到 ECR

```powershell
docker push `
123456789012.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend:latest
```

完成后：

```text
ECR
└── aws-llm-platform-backend
       └── latest
```

🎉

现在你的 Docker Image 已经进入 AWS。

---

# 8. 为什么需要 ECR？

记住：

```text
Docker
    ↓
Image
    ↓
ECR
    ↓
ECS
```

ECR：

> Container Registry

ECS：

> Container Orchestration

Fargate：

> Container Compute

所以：

```text
ECR = 放镜像

ECS = 管理容器

Fargate = 跑容器
```

---

# 9. 创建 ECS Cluster

进入：

[Amazon ECS Console](https://console.aws.amazon.com/ecs/?utm_source=chatgpt.com)

选择：

```text
Clusters
 ↓
Create cluster
```

名字：

```text
aws-llm-platform-dev
```

Infrastructure：

```text
AWS Fargate
```

创建。

最终：

```text
ECS
└── aws-llm-platform-dev
```

---

# 10. 创建 Task Definition

进入：

```text
Task definitions
    ↓
Create new task definition
```

名称：

```text
aws-llm-platform-backend
```

选择：

```text
Launch type:
AWS Fargate
```

---

# 11. CPU / Memory

Day 4 开发测试可以使用：

```text
CPU:
0.25 vCPU

Memory:
0.5 GB
```

如果你的应用启动或后面加入 AI SDK 后内存不足，再提高到：

```text
0.5 vCPU
1 GB
```

**不要一开始开很大的 Fargate。**

---

# 12. Container 设置

Container name：

```text
backend
```

Image URI：

```text
YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/aws-llm-platform-backend:latest
```

Port：

```text
8000
```

Protocol：

```text
TCP
```

---

# 13. CloudWatch Logs

Container logging：

```text
Use log collection
```

选择：

```text
Amazon CloudWatch Logs
```

例如：

```text
/aws/ecs/aws-llm-platform
```

这样以后你可以看到：

```text
FastAPI startup
HTTP requests
Exceptions
Stack traces
```

这是生产环境必须掌握的能力。

---

# 14. 创建 Task Definition

点击：

```text
Create
```

最终：

```text
ECS
│
├── Cluster
│
└── Task Definition
      │
      └── backend container
             │
             └── port 8000
```

---

# 15. 创建 ECS Service

进入：

```text
Cluster
 ↓
aws-llm-platform-dev
 ↓
Create service
```

Compute：

```text
Fargate
```

Task Definition：

```text
aws-llm-platform-backend
```

Service name：

```text
backend
```

Desired tasks：

```text
1
```

---

# 16. VPC

如果 AWS 自动提供：

```text
Default VPC
```

Day 4 可以先使用。

选择至少两个 Subnets。

例如：

```text
us-west-2a
us-west-2b
```

以后生产环境我们再自己设计：

```text
VPC
├── Public Subnet
├── Private Subnet
├── NAT Gateway
└── Security Groups
```

---

# 17. Security Group

创建：

```text
aws-llm-platform-backend-sg
```

开发阶段暂时允许：

```text
TCP
8000
```

但注意：

**不要把生产环境设计成 `0.0.0.0/0 → 8000`。**

今天只是为了验证服务。

真正生产架构：

```text
Internet
   ↓
ALB :443
   ↓
ECS
   ↓
FastAPI :8000
```

ECS 的 8000 端口应该只允许 ALB 访问。

---

# 18. Public IP

Day 4 为了简单测试：

```text
Auto-assign public IP:
ENABLED
```

以后正式生产：

```text
Internet
    ↓
ALB
    ↓
Private ECS
```

**不要让 ECS Task 直接暴露公网。**

---

# 19. Deploy

点击：

```text
Create
```

等待：

```text
Provisioning
       ↓
Pending
       ↓
Running
```

最终：

```text
Desired:
1

Running:
1
```

如果：

```text
Running = 1
```

说明 Container 成功运行。

---

# 20. 找到 Public IP

进入：

```text
ECS
 ↓
Cluster
 ↓
Service
 ↓
Tasks
 ↓
backend
```

点击 Task。

找到：

```text
Networking
 ↓
Public IP
```

例如：

```text
54.xxx.xxx.xxx
```

---

# 21. 测试 FastAPI

浏览器：

```text
http://YOUR_PUBLIC_IP:8000/api/health
```

应该：

```json
{
  "status": "ok",
  "service": "aws-llm-platform-backend",
  "version": "0.1.0"
}
```

如果成功：

# 🎉 FastAPI 已经真正运行在 AWS

---

# 22. Swagger

打开：

```text
http://YOUR_PUBLIC_IP:8000/docs
```

应该看到：

```text
AWS LLM Platform API

GET  /api/health

POST /api/chat
```

这意味着：

```text
Windows
   │
Internet
   │
   ▼
AWS
   │
ECS Fargate
   │
Docker
   │
FastAPI
```

已经完全打通。

---

# 23. 查看 CloudWatch

进入：

[Amazon CloudWatch Console](https://console.aws.amazon.com/cloudwatch/?utm_source=chatgpt.com)

找到：

```text
Logs
 ↓
Log groups
 ↓
/aws/ecs/aws-llm-platform
```

你应该看到类似：

```text
INFO: Started server process
INFO: Waiting for application startup
INFO: Application startup complete
INFO: Uvicorn running on
```

以后出现：

```text
Bedrock error
RAG error
Database error
Agent error
```

都会通过这里排查。

---

# 24. 现在把 React 接到 AWS Backend

Day 2 React 原来：

```text
React
 ↓
localhost:8000
```

现在改成：

```text
React
 ↓
AWS ECS
 ↓
FastAPI
```

例如：

```typescript
const API_BASE_URL =
  "http://YOUR_ECS_PUBLIC_IP:8000";
```

然后：

```typescript
const response = await fetch(
  `${API_BASE_URL}/api/chat`,
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
```

现在：

```text
AWS Amplify
      │
      │ HTTPS
      ▼
React
      │
      │ HTTP
      ▼
ECS Fargate
      │
      ▼
FastAPI
```

不过这里会遇到一个非常重要的问题：

# ⚠️ CORS + HTTPS

你的 Amplify 是：

```text
https://xxxxx.amplifyapp.com
```

而 ECS 现在：

```text
http://PUBLIC_IP:8000
```

这不是最终生产方案。

所以 Day 5/6 我们会改成：

```text
Amplify
   │
 HTTPS
   ▼
API Gateway / ALB
   │
 HTTPS
   ▼
ECS Fargate
   │
   ▼
FastAPI
```

这样才是真正的生产架构。

---

# 25. GitHub 提交

把今天的 Docker / backend 更新提交：

```powershell
git add .
```

```powershell
git commit -m "Day 4: containerize and deploy backend to AWS ECS"
```

```powershell
git push
```

建议 README 加：

```text
## AWS Deployment

Backend:
- Docker
- Amazon ECR
- Amazon ECS
- AWS Fargate
- CloudWatch
```

---

# 26. Day 4 你必须理解的 AWS 架构

这是今天最重要的知识：

```text
Dockerfile
    ↓
Docker Image
    ↓
ECR
    ↓
ECS Task Definition
    ↓
ECS Service
    ↓
Fargate
    ↓
Container
    ↓
FastAPI
```

---

# 27. ECR vs ECS vs Fargate

面试一定会问。

### ECR

```text
Where is my Docker image?
```

答案：

> Amazon ECR stores container images.

### ECS

```text
How do I manage containers?
```

答案：

> Amazon ECS orchestrates and manages containerized applications.

### Fargate

```text
Where does the container run?
```

答案：

> AWS Fargate provides serverless compute for containers without requiring you to manage EC2 servers.

---

# 28. Day 4 面试题

### Q1

**Why use ECS Fargate instead of EC2?**

> Fargate allows us to run containers without managing the underlying EC2 instances.

### Q2

**What is an ECS Task Definition?**

> A task definition describes how an ECS task should run, including the container image, CPU, memory, ports, environment variables, and logging configuration.

### Q3

**What is an ECS Service?**

> An ECS service maintains the desired number of running tasks and replaces unhealthy tasks when necessary.

### Q4

**Why use ECR?**

> ECR provides a managed private registry for storing Docker container images used by ECS.

---

# 29. 今天最终架构

```text
                    GitHub
                       │
                       ▼
                  Dockerfile
                       │
                       ▼
                 Docker Image
                       │
                       ▼
                      ECR
                       │
                       ▼
                 ECS Fargate
                       │
                       ▼
                  FastAPI
                       │
             ┌─────────┴─────────┐
             │                   │
        /api/health          /api/chat
```

而 Day 1–4 已经形成：

```text
                    AWS
                     │
       ┌─────────────┼──────────────┐
       │             │              │
    Amplify         S3          CloudWatch
       │                            ▲
     React                          │
       │                            │
       ▼                            │
    ECS Fargate ─────────────── Logs
       │
    FastAPI
       │
    Docker
       │
      ECR
```

---

# Day 4 完成标准

你完成下面这些，就可以进入 Day 5：

```text
□ ECR Repository
□ Docker image pushed to ECR
□ ECS Cluster
□ Task Definition
□ Fargate Service
□ Running Task = 1
□ /api/health 成功
□ /docs 成功
□ CloudWatch Logs 成功
□ GitHub commit
```

### Day 5

下一步我们正式进入这个项目最核心的一步：

```text
React
   ↓
FastAPI
   ↓
Amazon Bedrock
   ↓
Claude / Amazon Nova
   ↓
真正的 LLM Response
```

然后我们会把 **AWS IAM + Bedrock + boto3 + Streaming + Prompt Management** 一起搭起来，让你的 `/api/chat` 第一次真正成为 LLM API。
