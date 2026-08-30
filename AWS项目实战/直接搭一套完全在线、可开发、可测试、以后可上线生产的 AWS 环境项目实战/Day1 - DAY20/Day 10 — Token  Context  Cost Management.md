# Day 10 — Token / Context / Cost Management

今天开始进入真正的 **LLM Application Engineering**。

Day 9 你的系统是：

```text
React
 ↓
Cognito / JWT
 ↓
API Gateway
 ↓
FastAPI
 ↓
ChatService
 ├── PostgreSQL
 └── Bedrock
```

今天升级成：

```text
ChatService
     ↓
ContextManager
     ↓
Token Budget
     ↓
Prompt
     ↓
Bedrock
     ↓
Token Usage
     ↓
Cost Tracking
     ↓
PostgreSQL
```

---

# 1. Day 10 完成目标

```text
□ Token 是什么
□ Context Window
□ Input Tokens
□ Output Tokens
□ Token Budget
□ History 截断
□ Context Manager
□ Cost Calculation
□ Usage Logging
□ PostgreSQL usage 表
□ API 返回 usage
□ 单元测试
```

---

# 2. 为什么必须管理 Token？

假设用户连续聊天：

```text
Message 1
Message 2
...
Message 500
```

如果每次都把全部历史发送给 LLM：

```text
Input Tokens ↑
     ↓
Cost ↑
     ↓
Latency ↑
     ↓
Context Limit
```

所以不能简单：

```python
messages = all_messages
```

生产级 LLM 应用必须控制：

```text
Context
Token
Cost
Latency
```

---

# 3. Token 到底是什么？

不要把：

```text
1 word = 1 token
```

当成绝对规则。

LLM 通常把文本拆成 token。

例如：

```text
Hello, how are you?
```

可能被拆成若干 token。

对于英文：

```text
token ≈ word/subword/punctuation
```

对于中文：

```text
中文字符
```

token 化规则可能明显不同。

所以程序里应该：

```text
Text
 ↓
Tokenizer
 ↓
Token Count
```

而不是自己猜。

---

# 4. 今天建立 Token Budget

假设我们给一次请求：

```text
Total Context Budget = 8,000 tokens
```

分配：

```text
System Prompt       500
Conversation        6,000
Current Message    1,000
Reserved Output      500
-------------------------
Total               8,000
```

也就是说：

```text
Input + Output
```

都必须考虑。

---

# 5. Context Manager

创建：

```text
backend/app/services/context_manager.py
```

核心接口：

```python
class ContextManager:

    def build_context(
        self,
        system_prompt,
        history,
        current_message,
        max_input_tokens,
    ):
        pass
```

以后：

```text
ChatService
    ↓
ContextManager
    ↓
LLMService
```

---

# 6. 不要简单使用最后 20 条

Day 9：

```text
Last 20 messages
```

今天升级成：

```text
Token-based history
```

例如：

```text
Maximum input tokens = 6000
```

从最新消息往前：

```text
Message 20 ✓
Message 19 ✓
Message 18 ✓
...
Message 8 ✓
Message 7 ✗
```

最终：

```text
Recent History
```

刚好控制在 budget 内。

---

# 7. Token Counter

如果你使用的模型有对应 tokenizer，优先使用模型/SDK提供的 tokenizer。

对于无法直接获得精确 tokenizer 的情况，可以先做一个**估算器**，但要明确：

> 估算值只用于预算控制，不作为账单的精确值。

例如：

```python
class TokenCounter:

    def estimate(self, text: str) -> int:
        return max(1, len(text) // 4)
```

这是粗略估算。

生产环境应该替换成与你实际模型匹配的 tokenizer。

---

# 8. Context Manager 实现

先建立简单版本：

```python
class ContextManager:

    def __init__(
        self,
        token_counter,
        max_input_tokens=6000,
    ):
        self.token_counter = token_counter
        self.max_input_tokens = max_input_tokens

    def build_context(
        self,
        system_prompt,
        history,
        current_message,
    ):

        messages = []

        system_tokens = (
            self.token_counter
            .estimate(system_prompt)
        )

        current_tokens = (
            self.token_counter
            .estimate(current_message)
        )

        remaining = (
            self.max_input_tokens
            - system_tokens
            - current_tokens
        )

        selected = []

        for message in reversed(history):

            tokens = (
                self.token_counter
                .estimate(message.content)
            )

            if tokens > remaining:
                break

            selected.append(message)
            remaining -= tokens

        selected.reverse()

        for message in selected:

            messages.append(
                {
                    "role": message.role,
                    "content": [
                        {
                            "text": message.content
                        }
                    ],
                }
            )

        messages.append(
            {
                "role": "user",
                "content": [
                    {
                        "text": current_message
                    }
                ],
            }
        )

        return messages
```

---

# 9. 注意一个关键问题

不能只考虑：

```text
history tokens
```

必须考虑：

```text
system prompt
+
history
+
current message
+
output reservation
```

所以更合理：

```text
Model Context Budget
        │
        ├── System
        ├── History
        ├── Current User
        └── Reserved Output
```

---

# 10. 推荐预算

开发环境可以先：

```text
Context Budget = 8,000
Output Budget  = 1,000
```

例如：

```python
MAX_CONTEXT_TOKENS = 8000
MAX_OUTPUT_TOKENS = 1000
```

实际可用 input：

```text
8000 - 1000
=
7000
```

---

# 11. LLMService 修改

Day 9：

```python
response = self.client.converse(...)
```

今天需要读取：

```text
usage
```

Bedrock Converse API 的响应包含模型输出以及使用量等信息，具体字段取决于 API/模型响应。

结构上我们希望得到：

```python
usage = response.get(
    "usage",
    {}
)
```

然后：

```python
input_tokens = usage.get(
    "inputTokens",
    0
)

output_tokens = usage.get(
    "outputTokens",
    0
)

total_tokens = usage.get(
    "totalTokens",
    input_tokens + output_tokens,
)
```

---

# 12. LLMResponse

不要只返回：

```python
return answer
```

升级：

```python
class LLMResponse:

    def __init__(
        self,
        text,
        input_tokens,
        output_tokens,
        total_tokens,
    ):
        self.text = text
        self.input_tokens = input_tokens
        self.output_tokens = output_tokens
        self.total_tokens = total_tokens
```

这样：

```text
LLM
 ↓
Answer
+
Usage
```

---

# 13. ChatService 升级

最终：

```text
ChatService
```

变成：

```text
1. Authenticate User
2. Get Conversation
3. Load History
4. Context Manager
5. Call Bedrock
6. Save User Message
7. Save Assistant Message
8. Save Usage
9. Return Response
```

代码结构：

```python
history = (
    self.message_service
    .get_recent_messages(
        db,
        conversation.id,
        limit=100,
    )
)

messages = (
    self.context_manager
    .build_context(
        system_prompt=system_prompt,
        history=history,
        current_message=message,
    )
)

llm_response = (
    self.llm_service
    .generate_messages(messages)
)
```

---

# 14. 新建 Usage 表

今天非常重要。

创建：

```text
app/models/llm_usage.py
```

```python
class LLMUsage(Base):

    __tablename__ = "llm_usage"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"),
        index=True,
        nullable=False,
    )

    conversation_id: Mapped[int] = mapped_column(
        ForeignKey("conversations.id"),
        index=True,
        nullable=False,
    )

    model_id: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
    )

    input_tokens: Mapped[int] = mapped_column(
        nullable=False
    )

    output_tokens: Mapped[int] = mapped_column(
        nullable=False
    )

    total_tokens: Mapped[int] = mapped_column(
        nullable=False
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )
```

---

# 15. Database Migration

运行：

```powershell
alembic revision --autogenerate -m "add llm usage"
```

然后：

```powershell
alembic upgrade head
```

检查：

```sql
\dt
```

应该：

```text
users
conversations
messages
llm_usage
```

---

# 16. 为什么 Usage 要关联 User？

因为以后你可以查询：

```text
User A
 ↓
10,000 tokens
```

User B：

```text
 ↓
200,000 tokens
```

于是你可以做：

```text
Usage Dashboard
```

---

# 17. Cost Calculation

今天先建立统一接口：

```python
class CostService:

    def calculate(
        self,
        model_id,
        input_tokens,
        output_tokens,
    ):
        pass
```

例如：

```text
input_tokens
output_tokens
model_id
       ↓
CostService
       ↓
estimated_cost
```

---

# 18. 为什么不能把价格硬编码到 ChatService？

不要：

```python
cost = (
    input_tokens * 0.000003
)
```

直接写在：

```text
ChatService
```

因为以后：

```text
Model A
Model B
Model C
```

价格不同。

应该：

```text
ChatService
     ↓
LLMService
     ↓
Usage
     ↓
CostService
```

---

# 19. Model Pricing Config

开发阶段：

```python
MODEL_PRICING = {

    "model-a": {
        "input_per_1k": 0.001,
        "output_per_1k": 0.002,
    },

}
```

实际项目中一定要使用你所选 Bedrock 模型当前官方价格，不要把示例价格当成真实账单价格。

---

# 20. 计算公式

如果：

```text
Input = 10,000 tokens
Output = 2,000 tokens
```

价格：

```text
Input Cost
=
10,000 / 1,000
×
input price
```

Output：

```text
2,000 / 1,000
×
output price
```

总成本：

```text
Input Cost
+
Output Cost
```

---

# 21. API Response

Day 9：

```json
{
  "conversation_id": 1,
  "answer": "..."
}
```

Day 10：

```json
{
  "conversation_id": 1,
  "answer": "...",
  "usage": {
    "input_tokens": 1200,
    "output_tokens": 300,
    "total_tokens": 1500
  }
}
```

开发阶段非常有用。

生产环境可以根据用户权限决定是否返回详细 usage。

---

# 22. React 显示 Token

例如：

```text
AI Answer

RAG is a technique...

────────────────────

Tokens: 1,500
```

开发模式可以：

```text
Input: 1,200
Output: 300
Total: 1,500
```

生产用户通常不需要看到这些。

---

# 23. Context 管理真正的架构

现在：

```text
                  Chat Request
                       │
                       ▼
                 ChatService
                       │
                       ▼
                Load Messages
                       │
                       ▼
                ContextManager
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
         System      History    Current
         Prompt                  Message
            │          │          │
            └──────────┼──────────┘
                       ▼
                  Token Budget
                       │
                       ▼
                    Bedrock
                       │
                       ▼
                 LLM Response
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          Answer               Usage
             │                   │
             ▼                   ▼
         messages             llm_usage
```

这已经是比较标准的 LLM Application Architecture。

---

# 24. 一个非常重要的问题：Context Overflow

假设：

```text
System = 500
History = 20,000
Current = 1,000
Output = 2,000
```

总计：

```text
23,500 tokens
```

如果模型/应用预算只有：

```text
8,000
```

就必须：

```text
History
 ↓
truncate
```

不能：

```text
直接发送
```

---

# 25. 下一阶段的高级方案

今天：

```text
Recent Messages
```

以后：

```text
Long Conversation
       │
       ▼
Conversation Summary
       │
       +
       ▼
Recent Messages
```

例如：

```text
Summary:
User is building an AWS LLM application.
The backend uses FastAPI, PostgreSQL and Bedrock.
The user wants to implement RAG next.

Recent:
User: How should embeddings work?
AI: ...
```

这样：

```text
Context
↓
小很多
```

---

# 26. Cost Tracking

以后数据库可以查询：

```sql
SELECT
    user_id,
    SUM(input_tokens),
    SUM(output_tokens),
    SUM(total_tokens)
FROM llm_usage
GROUP BY user_id;
```

得到：

```text
User    Input    Output    Total
---------------------------------
A       50K      10K       60K
B       120K     20K       140K
C       10K      2K        12K
```

这就是：

# AI SaaS Usage Metering

---

# 27. 可以进一步做每日统计

以后：

```text
usage_daily
```

例如：

```text
date
user_id
model_id
input_tokens
output_tokens
estimated_cost
```

然后 React Dashboard：

```text
┌──────────────────────────────┐
│ AI Usage                     │
│                              │
│ Today:       12,450 tokens   │
│ This Month:  320K tokens     │
│ Estimated:   $4.28           │
│                              │
│ Requests: 128                │
└──────────────────────────────┘
```

这会让你的 GitHub 项目明显更像一个真实 SaaS。

---

# 28. Day 10 单元测试

创建：

```text
tests/test_context_manager.py
```

测试：

### Test 1

```text
History 很短
```

结果：

```text
全部保留
```

### Test 2

```text
History 很长
```

结果：

```text
旧消息被删除
```

### Test 3

```text
Current message
```

必须始终保留。

### Test 4

```text
System prompt
```

必须始终保留。

### Test 5

```text
Total tokens > budget
```

必须：

```text
truncate
```

---

# 29. Day 10 最重要的测试

制造：

```text
100 条消息
```

然后：

```text
MAX_CONTEXT_TOKENS = 2000
```

确认：

```text
最终 Context
≤ 2000
```

这是今天的核心测试。

---

# 30. Day 10 面试题

### Q1：什么是 Context Window？

> The maximum amount of tokenized context that a model can process in a request, including the prompt and generated output constraints.

---

### Q2：为什么不能无限发送 Chat History？

因为：

```text
Cost
Latency
Context Limit
```

都会增加。

---

### Q3：如何解决长对话？

```text
Truncation
+
Summarization
+
Retrieval
+
Token Budget
```

---

### Q4：为什么需要 Token Tracking？

因为：

```text
Usage
 ↓
Cost
 ↓
Quota
 ↓
Billing
 ↓
Monitoring
```

---

### Q5：为什么 Usage 要单独存数据库？

因为以后需要：

```text
User Usage
Conversation Usage
Model Usage
Daily Usage
Cost Analysis
```

---

# 31. 今天项目结构

```text
backend/
│
├── app/
│   │
│   ├── api/
│   │
│   ├── models/
│   │   ├── user.py
│   │   ├── conversation.py
│   │   ├── message.py
│   │   └── llm_usage.py
│   │
│   ├── services/
│   │   ├── chat_service.py
│   │   ├── llm_service.py
│   │   ├── conversation_service.py
│   │   ├── message_service.py
│   │   ├── prompt_service.py
│   │   ├── context_manager.py
│   │   └── cost_service.py
│   │
│   └── db/
│
├── alembic/
│
└── tests/
    ├── test_chat_service.py
    ├── test_context_manager.py
    └── test_cost_service.py
```

---

# 32. Day 10 最终架构

```text
                           React
                             │
                             ▼
                        API Gateway
                             │
                           JWT
                             │
                             ▼
                          FastAPI
                             │
                             ▼
                       ChatService
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        PostgreSQL     ContextManager    LLMService
              │              │              │
              │         Token Budget        │
              │              │              ▼
              │              │           Bedrock
              │              │              │
              │              │              ▼
              │              │          LLM Response
              │              │              │
              │              └──────────────┤
              │                             │
              ▼                             ▼
        Chat History                   Token Usage
                                             │
                                             ▼
                                        llm_usage
                                             │
                                             ▼
                                       Cost Tracking
```

---

# 33. Day 10 Git Commit

```powershell
git add .
```

```powershell
git commit -m "Day 10: add token context and LLM usage tracking"
```

```powershell
git push
```

README：

```markdown
## LLM Context Management

The platform implements:

- Token budget management
- Conversation truncation
- Context window management
- LLM usage tracking
- Input/output token tracking
- Estimated cost tracking
```

---

# 🎯 Day 10 完成标准

```text
□ ContextManager
□ Token Budget
□ History Truncation
□ Current Message Preservation
□ System Prompt Preservation
□ Bedrock Usage Extraction
□ Input Token Tracking
□ Output Token Tracking
□ Total Token Tracking
□ llm_usage table
□ CostService
□ Context Unit Tests
□ GitHub Commit
```

现在你的项目已经有了：

```text
AWS
+
React
+
FastAPI
+
Cognito
+
JWT
+
PostgreSQL
+
RDS
+
Bedrock
+
Multi-turn Chat
+
Context Management
+
Token Tracking
+
Cost Tracking
```

**Day 11 下一步进入真正的 AI 核心：RAG。**

架构会升级成：

```text
                    User Question
                         │
                         ▼
                    Embedding
                         │
                         ▼
                  Vector Database
                         │
                  Similarity Search
                         │
                         ▼
                  Relevant Chunks
                         │
                         ▼
                    Context
                         │
                         ▼
                      LLM
                         │
                         ▼
                 Grounded Answer
```

也就是开始做一个**真正可以放到 GitHub、用于 2026 美国 AI Engineer 求职的 RAG 项目**。
