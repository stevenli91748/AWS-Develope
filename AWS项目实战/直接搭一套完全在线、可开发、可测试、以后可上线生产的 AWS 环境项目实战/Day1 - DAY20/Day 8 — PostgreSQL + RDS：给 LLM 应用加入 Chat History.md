# Day 8 — PostgreSQL + RDS：给 LLM 应用加入 Chat History

今天把 Day 7 的：

```text
Cognito
   ↓
JWT
   ↓
FastAPI
   ↓
Bedrock
```

升级成真正的**多用户 LLM SaaS 后端**：

```text
User
 ↓
Cognito
 ↓
JWT
 ↓
API Gateway
 ↓
ECS / FastAPI
 ├── PostgreSQL / RDS
 │     ├── users
 │     ├── conversations
 │     └── messages
 │
 └── Bedrock
```

今天重点不是把数据库做得很复杂，而是建立一个**正确的生产级数据模型**。

---

# 1. Day 8 完成目标

```text
□ PostgreSQL
□ Amazon RDS
□ VPC / Security Group
□ SQLAlchemy
□ Alembic
□ users
□ conversations
□ messages
□ Cognito user_id
□ 保存 Chat History
□ 查询历史消息
□ 用户数据隔离
□ ECS → RDS
```

---

# 2. 为什么现在加入数据库？

Day 7：

```text
User
 ↓
JWT
 ↓
FastAPI
 ↓
Bedrock
 ↓
Answer
```

问题：

用户刷新网页：

```text
Chat History ❌
```

重新登录：

```text
Conversation ❌
```

现在：

```text
User
 ↓
JWT
 ↓
user_id
 ↓
PostgreSQL
 ↓
Conversation
 ↓
Messages
```

所以：

```text
User A → 只能看到 A 的 Chat
User B → 只能看到 B 的 Chat
```

---

# 3. 今天选择 PostgreSQL

本地：

```text
PostgreSQL
```

AWS：

```text
Amazon RDS for PostgreSQL
```

AWS RDS 是托管数据库服务，可以减少数据库安装、备份、维护等基础设施工作。

我们的技术栈：

```text
FastAPI
   ↓
SQLAlchemy
   ↓
PostgreSQL
```

---

# 4. 数据库设计

今天先建立三个核心表：

```text
users
   │
   └── conversations
            │
            └── messages
```

关系：

```text
User
 1
 │
 │
 N
Conversation
 1
 │
 │
 N
Message
```

---

# 5. users

```text
users
--------------------------------
id
cognito_user_id
email
created_at
```

例如：

```text
id | cognito_user_id | email
---|-----------------|-------------------
1  | abc-123         | user@gmail.com
2  | xyz-456         | test@gmail.com
```

这里：

```text
cognito_user_id
```

对应 Cognito JWT 的：

```text
sub
```

---

# 6. conversations

```text
conversations
--------------------------------
id
user_id
title
created_at
updated_at
```

例如：

```text
id | user_id | title
---|---------|----------------
1  | 1       | AWS RAG Project
2  | 1       | FastAPI Help
3  | 2       | Python Question
```

---

# 7. messages

```text
messages
--------------------------------
id
conversation_id
role
content
created_at
```

例如：

```text
id | conversation | role      | content
---|--------------|-----------|----------------
1  | 1            | user      | What is RAG?
2  | 1            | assistant | RAG is...
3  | 1            | user      | Explain more
4  | 1            | assistant | ...
```

---

# 8. 最终关系

```text
users
  │
  │ 1:N
  ▼
conversations
  │
  │ 1:N
  ▼
messages
```

这是今天最重要的数据库设计。

---

# 9. 安装数据库依赖

进入：

```powershell
cd backend
```

安装：

```powershell
pip install sqlalchemy
pip install psycopg[binary]
pip install alembic
```

然后：

```powershell
pip freeze > requirements.txt
```

---

# 10. 项目目录升级

现在：

```text
backend/
│
├── app/
│   ├── api/
│   │   ├── routes.py
│   │   └── dependencies.py
│   │
│   ├── models/
│   │   ├── user.py
│   │   ├── conversation.py
│   │   └── message.py
│   │
│   ├── schemas/
│   │   ├── chat.py
│   │   └── conversation.py
│   │
│   ├── services/
│   │   ├── llm_service.py
│   │   └── chat_service.py
│   │
│   ├── db/
│   │   ├── database.py
│   │   └── base.py
│   │
│   └── main.py
│
├── alembic/
├── tests/
├── Dockerfile
└── requirements.txt
```

---

# 11. 创建数据库连接

创建：

```text
app/db/database.py
```

```python
import os

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,
)

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)
```

---

# 12. FastAPI Database Dependency

继续：

```python
from collections.abc import Generator


def get_db() -> Generator:
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()
```

以后：

```python
@router.get("/conversations")
def get_conversations(
    db: Session = Depends(get_db),
):
    ...
```

FastAPI 会自动管理：

```text
Request
 ↓
DB Session
 ↓
Query
 ↓
Response
 ↓
Close Session
```

---

# 13. Base Model

创建：

```text
app/db/base.py
```

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

---

# 14. User Model

创建：

```text
app/models/user.py
```

```python
from datetime import datetime

from sqlalchemy import String, DateTime
from sqlalchemy.orm import Mapped, mapped_column

from app.db.base import Base


class User(Base):

    __tablename__ = "users"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    cognito_user_id: Mapped[str] = mapped_column(
        String(255),
        unique=True,
        index=True,
        nullable=False,
    )

    email: Mapped[str] = mapped_column(
        String(320),
        nullable=False,
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )
```

---

# 15. Conversation Model

```python
from datetime import datetime

from sqlalchemy import (
    String,
    DateTime,
    ForeignKey,
)

from sqlalchemy.orm import (
    Mapped,
    mapped_column,
)

from app.db.base import Base


class Conversation(Base):

    __tablename__ = "conversations"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"),
        index=True,
        nullable=False,
    )

    title: Mapped[str] = mapped_column(
        String(255),
        default="New Chat",
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )

    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )
```

---

# 16. Message Model

```python
from datetime import datetime

from sqlalchemy import (
    String,
    Text,
    DateTime,
    ForeignKey,
)

from sqlalchemy.orm import (
    Mapped,
    mapped_column,
)

from app.db.base import Base


class Message(Base):

    __tablename__ = "messages"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    conversation_id: Mapped[int] = mapped_column(
        ForeignKey("conversations.id"),
        index=True,
        nullable=False,
    )

    role: Mapped[str] = mapped_column(
        String(20),
        nullable=False,
    )

    content: Mapped[str] = mapped_column(
        Text,
        nullable=False,
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )
```

---

# 17. 初始化 Alembic

运行：

```powershell
alembic init alembic
```

目录：

```text
alembic/
├── versions/
├── env.py
└── script.py.mako
```

修改：

```text
alembic.ini
```

不要把生产密码直接写进去。

使用：

```text
DATABASE_URL
```

环境变量。

---

# 18. DATABASE_URL

本地开发：

```env
DATABASE_URL=postgresql+psycopg://postgres:password@localhost:5432/llm_platform
```

AWS：

```env
DATABASE_URL=postgresql+psycopg://USER:PASSWORD@RDS_ENDPOINT:5432/llm_platform
```

**密码不要提交 Git。**

---

# 19. 创建数据库

本地 PostgreSQL：

```sql
CREATE DATABASE llm_platform;
```

然后：

```powershell
alembic revision --autogenerate -m "create users conversations messages"
```

检查 migration。

然后：

```powershell
alembic upgrade head
```

---

# 20. 进入 PostgreSQL 检查

```sql
\dt
```

应该看到：

```text
users
conversations
messages
```

检查：

```sql
\d users
```

以及：

```sql
\d conversations
\d messages
```

---

# 21. 现在连接 Cognito

这是 Day 8 的核心。

Day 7：

```text
JWT
 ↓
Cognito sub
```

Day 8：

```text
Cognito sub
 ↓
users.cognito_user_id
```

例如：

```text
JWT
{
    "sub": "abc-123"
}
```

数据库：

```text
users
--------------------------
id = 1
cognito_user_id = abc-123
email = user@gmail.com
```

---

# 22. 第一次用户登录

流程：

```text
User Login
    ↓
Cognito
    ↓
JWT
    ↓
API
    ↓
sub = abc-123
    ↓
SELECT users
WHERE cognito_user_id = abc-123
```

如果不存在：

```text
INSERT users
```

如果已经存在：

```text
SELECT existing user
```

---

# 23. User Service

创建：

```text
app/services/user_service.py
```

逻辑：

```python
def get_or_create_user(
    db,
    cognito_user_id,
    email,
):
    user = (
        db.query(User)
        .filter(
            User.cognito_user_id
            == cognito_user_id
        )
        .first()
    )

    if user:
        return user

    user = User(
        cognito_user_id=cognito_user_id,
        email=email,
    )

    db.add(user)
    db.commit()
    db.refresh(user)

    return user
```

---

# 24. Chat Service

创建：

```text
app/services/chat_service.py
```

逻辑：

```text
User
 ↓
Conversation
 ↓
User Message
 ↓
Bedrock
 ↓
AI Message
```

代码结构：

```python
class ChatService:

    def chat(
        self,
        db,
        user,
        message,
    ):
        # 1. create conversation

        # 2. save user message

        # 3. call Bedrock

        # 4. save assistant message

        # 5. return answer
        pass
```

今天先把结构建立起来。

Day 9 再把完整 Chat Service 做完。

---

# 25. 最重要的安全规则

**永远不要：**

```python
conversation = db.query(
    Conversation
).filter(
    Conversation.id == conversation_id
).first()
```

然后直接返回。

为什么？

攻击者可以：

```text
User A
 ↓
GET /conversations/999
```

而 `999` 可能属于 User B。

---

# 26. 正确查询

必须：

```python
conversation = (
    db.query(Conversation)
    .filter(
        Conversation.id == conversation_id,
        Conversation.user_id == user.id,
    )
    .first()
)
```

也就是：

```text
conversation_id
+
current_user_id
```

两个条件。

这是：

> **Authorization at the data-access layer**

非常重要。

---

# 27. Conversation API

建立：

```text
POST /api/conversations
GET  /api/conversations
GET  /api/conversations/{id}
DELETE /api/conversations/{id}
```

最终：

```text
React
 ↓
JWT
 ↓
API Gateway
 ↓
FastAPI
 ↓
current_user
 ↓
PostgreSQL
```

---

# 28. Chat API

Day 5：

```text
POST /api/chat
```

Day 8：

```json
{
  "conversation_id": 1,
  "message": "Explain RAG."
}
```

流程：

```text
JWT
 ↓
User
 ↓
Conversation
 ↓
Messages
 ↓
Bedrock
 ↓
Assistant Message
 ↓
PostgreSQL
```

---

# 29. Chat History API

例如：

```text
GET /api/conversations/1/messages
```

返回：

```json
{
  "conversation_id": 1,
  "messages": [
    {
      "role": "user",
      "content": "What is RAG?"
    },
    {
      "role": "assistant",
      "content": "RAG is..."
    }
  ]
}
```

React 就可以：

```text
打开 Chat
 ↓
GET messages
 ↓
显示历史
```

---

# 30. Amazon RDS

进入：

[Amazon RDS Console](https://console.aws.amazon.com/rds/?utm_source=chatgpt.com)

选择：

```text
Create database
```

Engine：

```text
PostgreSQL
```

学习环境可以选择：

```text
Dev/Test
```

不要为了学习一开始就选择高规格实例。

---

# 31. RDS 网络架构

不要：

```text
Internet
   ↓
RDS Public
```

正确：

```text
Internet
   ↓
API Gateway
   ↓
ECS
   ↓
Private Network
   ↓
RDS PostgreSQL
```

也就是：

```text
RDS
 ↓
Private
```

---

# 32. Security Group

建立：

```text
RDS Security Group
```

Inbound：

```text
PostgreSQL
TCP 5432
Source:
ECS Security Group
```

而不是：

```text
0.0.0.0/0
```

千万不要：

```text
Internet
 ↓
5432
 ↓
RDS
```

---

# 33. ECS → RDS

最终：

```text
ECS Security Group
       │
       │ TCP 5432
       ▼
RDS Security Group
```

RDS Security Group：

```text
Inbound
PostgreSQL
5432
Source = ECS SG
```

这样只有 ECS 可以访问数据库。

---

# 34. ECS Environment Variables

ECS Task Definition：

```text
AWS_REGION=us-west-2

BEDROCK_MODEL_ID=...

DATABASE_URL=...
```

但是：

**生产环境不要把数据库密码直接作为普通环境变量长期管理。**

下一阶段我们会升级成：

```text
AWS Secrets Manager
        ↓
ECS Task
        ↓
DATABASE_URL / DB credentials
```

AWS Secrets Manager 用于安全地管理数据库密码、API keys 等应用 secrets。

---

# 35. 今天先理解 Secrets Manager

最终：

```text
ECS
 │
 │ IAM Role
 ▼
Secrets Manager
 │
 ▼
DB Password
```

而不是：

```text
GitHub
 ↓
.env
 ↓
DB Password
```

---

# 36. Day 8 测试

### Test 1

用户 A 登录：

```text
Cognito
 ↓
JWT
```

创建：

```text
Conversation A
```

---

### Test 2

发送：

```text
What is RAG?
```

数据库：

```text
conversations
+
messages
```

应该有：

```text
user
assistant
```

---

### Test 3

刷新页面。

React：

```text
GET /api/conversations
```

应该还能看到：

```text
Conversation A
```

---

### Test 4

打开：

```text
GET /api/conversations/1/messages
```

应该得到历史。

---

### Test 5

换 User B。

User B：

```text
GET /api/conversations
```

应该：

```text
[]
```

而不能看到 User A。

---

# 37. Day 8 最重要的安全测试

模拟：

```text
User A
```

拿到：

```text
conversation_id = 100
```

然后 User B：

```text
GET /api/conversations/100
```

结果必须：

```text
404
```

或者：

```text
403
```

**绝对不能返回 User A 的数据。**

---

# 38. 今天的完整架构

```text
                         USER
                           │
                           ▼
                     React / Amplify
                           │
                           ▼
                       Cognito
                           │
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
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
                FastAPI        Bedrock
                    │             │
                    │             ▼
                    │             LLM
                    │
                    ▼
               PostgreSQL
                  RDS
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
      users   conversations  messages
```

---

# 39. Day 8 面试题

### Q1：为什么 Cognito `sub` 可以作为用户身份？

因为它是 Cognito 为用户提供的稳定唯一标识，可以作为应用数据库中的外部身份 ID。

---

### Q2：为什么不直接把 email 当 user ID？

Email 可能发生变化。

更合理：

```text
Cognito sub
    ↓
stable identity
```

Email：

```text
user attribute
```

---

### Q3：为什么 ECS 可以访问 RDS，而 Internet 不应该直接访问？

因为：

```text
ECS Security Group
       ↓
RDS Security Group
```

可以限制数据库访问来源。

---

### Q4：为什么需要 Alembic？

因为数据库 Schema 需要：

```text
Version Control
Migration
Rollback
Deployment
```

而不是每次手工修改数据库。

---

### Q5：为什么查询 Conversation 必须带 user_id？

防止：

```text
IDOR
```

即用户通过修改资源 ID 访问其他用户的数据。

这是 Web/API 安全中非常重要的一类授权问题。

---

# 40. Day 8 Git Commit

完成后：

```powershell
git add .
```

```powershell
git commit -m "Day 8: add PostgreSQL chat persistence"
```

```powershell
git push
```

README 增加：

```markdown
## Database

- PostgreSQL
- Amazon RDS
- SQLAlchemy
- Alembic
- User isolation
- Conversation persistence
- Message history
```

---

# 🎯 Day 8 完成标准

```text
□ PostgreSQL 本地运行
□ SQLAlchemy
□ Alembic
□ users
□ conversations
□ messages
□ Cognito sub → user
□ Conversation CRUD
□ Message persistence
□ User data isolation
□ RDS PostgreSQL
□ ECS → RDS
□ Security Group
□ GitHub
```

到这里，你的项目已经具备：

```text
Authentication
       +
Authorization
       +
Database
       +
LLM
       +
Cloud Infrastructure
```

这已经开始接近**美国 2026 AI Engineer 求职可以放到 GitHub 的完整项目**。

**Day 9 的重点：把 `ChatService` 真正做完整——历史消息 → Prompt → Bedrock → 保存 Assistant Response，并实现 ChatGPT 风格的连续多轮对话。**
