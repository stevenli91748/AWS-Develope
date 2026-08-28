# Day 1 — AWS LLM 在线开发环境

今天目标只有一个：

> **建立你的 AWS 云端开发基础，并让 GitHub → AWS 的开发路线跑通。**

今天**先不碰 Bedrock、RAG、LangChain**。先把基础环境搭稳。

---

## 1. Day 1 最终架构

今天完成：

```text
Windows PC
    │
    │ Browser
    ▼
GitHub
    │
    ▼
AWS
 ├── IAM
 ├── IAM Identity Center
 ├── S3
 ├── CloudShell
 └── CloudWatch
```

后面逐步增加：

```text
Day 2     React + Amplify
Day 3     FastAPI
Day 4     Docker + ECR
Day 5     ECS Fargate
Day 6     Bedrock
...
```

---

# 2. 今天需要准备

### 必须

* AWS Account
* GitHub Account
* Windows PC
* Chrome / Edge
* 手机（AWS MFA）

### 建议

安装：

* Git
* VS Code

---

# 3. AWS Account

进入 AWS 官方网站：

[AWS Console](https://console.aws.amazon.com/?utm_source=chatgpt.com)

登录以后进入：

```text
AWS Console
     ↓
IAM
```

---

# 4. 不要使用 Root Account 开发

这是今天非常重要的一步。

Root：

```text
AWS Root Account
        │
        └── 只负责账户级操作
```

开发：

```text
IAM Identity Center
        │
        └── Developer
```

以后：

```text
Root
  ↓
紧急/账户管理

Developer
  ↓
日常开发
```

**不要把 Root Access Key 放到电脑里。**

---

# 5. 设置 MFA

进入：

```text
IAM
 ↓
Root user
 ↓
Security credentials
 ↓
Multi-factor authentication
```

使用：

* Google Authenticator
* Microsoft Authenticator
* Authy

任选一个即可。

---

# 6. 建立开发者身份

AWS Console：

```text
IAM Identity Center
        ↓
Enable
        ↓
Users
        ↓
Add user
```

例如：

```text
Username:
ai-developer

Email:
你的邮箱

First name:
AI

Last name:
Developer
```

然后建立：

```text
Permission Set
```

学习阶段可以先使用：

```text
AdministratorAccess
```

但**只建议用于个人学习账号**。

正式项目以后改成：

```text
PowerUserAccess
+
必要的额外权限
```

---

# 7. AWS Region

你在美国西海岸，我建议 Day 1 统一：

```text
us-west-2
Oregon
```

后面所有实验尽量使用：

```text
AWS Region
us-west-2
```

这样可以避免：

```text
S3       us-east-1
RDS      us-west-2
Bedrock  us-east-1
ECS      us-west-2
```

导致环境越来越乱。

---

# 8. 开启 Billing Alert

进入：

```text
Billing
 ↓
Budgets
 ↓
Create budget
```

建立：

```text
Monthly Cost Budget
```

例如：

```text
Budget:
$50 / month
```

然后设置：

```text
80%
90%
100%
```

发送 Email Alert。

### 为什么？

因为我们后面会使用：

```text
ECS
RDS
S3
Bedrock
CloudWatch
```

其中部分服务会产生费用。

---

# 9. 建立第一个 S3 Bucket

进入：

```text
S3
 ↓
Create bucket
```

名称必须全球唯一。

例如：

```text
peng-ai-llm-dev-2026
```

如果已经被使用：

```text
peng-ai-llm-dev-2026-xxxx
```

Region：

```text
us-west-2
```

暂时：

```text
Block all public access = ON
```

然后：

```text
Create bucket
```

---

# 10. 建立项目目录

以后我们的 GitHub 项目统一采用：

```text
aws-llm-platform/
│
├── frontend/
│
├── backend/
│
├── ai/
│
├── tests/
│
├── infrastructure/
│
├── docs/
│
├── .github/
│
├── docker-compose.yml
├── README.md
└── .gitignore
```

最终会变成：

```text
aws-llm-platform
│
├── frontend
│   └── React / Next.js
│
├── backend
│   └── FastAPI
│
├── ai
│   ├── llm
│   ├── rag
│   └── agents
│
├── tests
│   ├── unit
│   ├── integration
│   └── evaluation
│
├── infrastructure
│   └── AWS CDK
│
└── docs
    ├── architecture
    ├── api
    └── deployment
```

这会成为你后面**美国 AI Engineer 求职 GitHub 项目**的基础。

---

# 11. GitHub 建仓库

进入：

[GitHub](https://github.com/?utm_source=chatgpt.com)

创建：

```text
Repository name:

aws-llm-platform
```

选择：

```text
Private
```

先不要公开。

勾选：

```text
Add README
```

然后：

```text
Create repository
```

---

# 12. Windows 安装 Git

打开 PowerShell：

```powershell
git --version
```

如果已经安装，会看到：

```text
git version 2.x.x
```

如果没有，安装：

[Git 官方下载](https://git-scm.com/downloads?utm_source=chatgpt.com)

然后：

```powershell
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
```

检查：

```powershell
git config --list
```

---

# 13. 安装 VS Code

[Visual Studio Code](https://code.visualstudio.com/?utm_source=chatgpt.com)

打开 VS Code。

安装这些 Extensions：

```text
Python
Pylance
Docker
GitHub Copilot
AWS Toolkit
YAML
REST Client
```

今天最重要的是：

```text
Python
Docker
AWS Toolkit
GitHub Copilot
```

---

# 14. AWS CloudShell

现在做一个非常重要的测试。

AWS Console 右上角打开：

```text
CloudShell
```

执行：

```bash
aws --version
```

然后：

```bash
aws sts get-caller-identity
```

如果返回类似：

```json
{
    "UserId": "...",
    "Account": "...",
    "Arn": "..."
}
```

说明：

> **AWS CLI → AWS Account 已经打通。**

---

# 15. 测试 S3

CloudShell：

```bash
aws s3 ls
```

应该可以看到刚才建立的 bucket。

然后：

```bash
aws s3 ls s3://你的-bucket-name
```

例如：

```bash
aws s3 ls s3://peng-ai-llm-dev-2026
```

如果没有错误：

**Day 1 的 AWS 基础环境已经成功。**

---

# 16. 建立第一个 README

在 GitHub 的 `README.md` 中先写：

```text
# AWS LLM Platform

A production-oriented LLM application platform built on AWS.

## Architecture

Frontend:
- React
- TypeScript

Backend:
- Python
- FastAPI

AI:
- Amazon Bedrock
- LangChain
- LangGraph
- RAG

Infrastructure:
- AWS
- ECS Fargate
- S3
- RDS PostgreSQL
- API Gateway
- Cognito

DevOps:
- Docker
- GitHub Actions
- CloudWatch
```

以后我们每天都会继续修改这个 README。

---

# 17. Day 1 作业

今天你只需要完成这 **8 件事**：

```text
□ 1. AWS Account
□ 2. Root MFA
□ 3. IAM Identity Center
□ 4. Developer User
□ 5. us-west-2
□ 6. Billing Budget
□ 7. S3 Bucket
□ 8. GitHub aws-llm-platform
```

然后测试：

```bash
aws sts get-caller-identity
```

以及：

```bash
aws s3 ls
```

---

# 18. Day 1 面试题

今天顺便记住这几个：

### Q1：为什么不能使用 Root Account 开发？

**Answer:**

> The AWS root user has unrestricted access to the account. For daily development, we should use IAM identities with least-privilege permissions.

---

### Q2：IAM 是什么？

> AWS Identity and Access Management controls authentication and authorization for AWS resources.

---

### Q3：S3 用来干什么？

> Amazon S3 is an object storage service commonly used for documents, images, logs, datasets, and files.

在我们的 LLM 项目里：

```text
PDF
DOCX
TXT
Images
Training data
       ↓
      S3
```

---

### Q4：为什么选择 us-west-2？

因为我们统一开发环境的 Region，减少跨 Region 的复杂度。**但后面 Bedrock 的模型可用性要单独检查，不能假设所有模型在 us-west-2 都可用。**

---

# Day 1 完成标准

最后你应该能得到：

```text
AWS Account
     │
     ├── IAM Identity Center ✅
     ├── MFA ✅
     ├── Billing Alert ✅
     ├── S3 ✅
     └── CloudShell AWS CLI ✅

GitHub
     │
     └── aws-llm-platform ✅
```

**Day 2 我们直接开始搭 React + TypeScript + AWS Amplify，把第一个在线前端跑起来，然后连接 GitHub。**
