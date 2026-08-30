# Day 11 — RAG 基础：PDF → Chunk → Embedding → Vector Search

今天正式进入 **LLM Application 最核心的能力之一：RAG（Retrieval-Augmented Generation）**。

Day 10 是：

```text
User
 ↓
ChatService
 ↓
ContextManager
 ↓
Bedrock
 ↓
LLM
```

Day 11 升级为：

```text
                    User Question
                         │
                         ▼
                    ChatService
                         │
                         ▼
                    RAGService
                         │
                ┌────────┴────────┐
                ▼                 ▼
           Embedding          PostgreSQL
                │              pgvector
                ▼                 │
          Query Vector ◄── Similarity Search
                                  │
                                  ▼
                             Top K Chunks
                                  │
                                  ▼
                              Context
                                  │
                                  ▼
                               Bedrock
                                  │
                                  ▼
                              Answer
```

---

# 1. 今天完成什么

```text
□ RAG 原理
□ Document
□ Chunk
□ Embedding
□ Vector
□ pgvector
□ Vector Search
□ Top-K Retrieval
□ RAGService
□ PostgreSQL Vector DB
□ PDF 文档准备
□ RAG API
□ 测试
```

今天**先不做复杂 Agent、LangChain、LlamaIndex**。

先把底层 RAG 原理自己实现一次。

---

# 2. RAG 是什么？

普通 LLM：

```text
User
 ↓
LLM
 ↓
Answer
```

问题：

```text
LLM不知道你的私人资料
LLM可能产生幻觉
LLM知识可能过时
```

RAG：

```text
User Question
      │
      ▼
Search Knowledge Base
      │
      ▼
Relevant Documents
      │
      ▼
LLM
      │
      ▼
Answer
```

所以：

> **RAG = Retrieval + Generation**

---

# 3. 今天我们的项目

假设你上传：

```text
AWS_LLM_Architecture.pdf
```

里面：

```text
AWS
Bedrock
FastAPI
PostgreSQL
RAG
```

用户问：

```text
What database does this application use?
```

系统不是直接问 LLM。

而是：

```text
Question
 ↓
Embedding
 ↓
Vector Search
 ↓
找到相关 PDF 内容
 ↓
把内容放进 Prompt
 ↓
Bedrock
 ↓
Answer
```

---

# 4. 第一步：安装 pgvector

你的 PostgreSQL 需要支持：

```text
vector
```

检查：

```sql id="h3c0r9"
CREATE EXTENSION IF NOT EXISTS vector;
```

然后：

```sql id="t2i3pp"
SELECT extname
FROM pg_extension;
```

应该看到：

```text id="pazw4d"
vector
```

---

# 5. AWS RDS 注意事项

如果你使用：

```text id="o1r0xe"
Amazon RDS for PostgreSQL
```

先确认当前 RDS PostgreSQL 版本和 AWS 支持的 `pgvector` 扩展版本。

AWS 官方文档：

[Amazon RDS PostgreSQL extensions](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html?utm_source=chatgpt.com)

如果：

```sql
CREATE EXTENSION vector;
```

成功，就可以继续。

---

# 6. 什么是 Embedding？

假设：

```text id="83p6n7"
"RAG uses vector search"
```

Embedding Model：

```text
Text
 ↓
Embedding Model
 ↓
[0.12, -0.83, 0.44, ...]
```

这个数组就是：

# Vector

例如：

```text id="ygtfpu"
[
  0.12,
  -0.83,
  0.44,
  0.71,
  ...
]
```

---

# 7. 为什么需要 Vector？

因为：

```text id="hcm6oi"
"How does RAG work?"
```

和：

```text id="5y8u0h"
"Explain retrieval augmented generation."
```

虽然文字完全不同，但语义相近。

Embedding 的目的就是把：

```text id="q5kvk2"
Text
```

变成：

```text id="4g3k1k"
Semantic Vector
```

然后：

```text id="6sydko"
Question Vector
       ↓
Similarity Search
       ↓
最相似文本
```

---

# 8. RAG 的第一阶段：Document Ingestion

完整流程：

```text id="h9zqjg"
PDF
 ↓
Extract Text
 ↓
Clean Text
 ↓
Chunk
 ↓
Embedding
 ↓
Vector DB
```

今天重点理解这个 Pipeline。

---

# 9. 什么是 Chunk？

不要把整个 PDF：

```text id="h3p5td"
100 pages
```

一次性 embedding。

应该：

```text id="rv5gdz"
Document
 ↓
Chunk 1
Chunk 2
Chunk 3
...
Chunk N
```

例如：

```text id="6e0zj5"
Chunk 1:
AWS Bedrock provides...

Chunk 2:
Amazon RDS provides...

Chunk 3:
FastAPI is...
```

---

# 10. Chunk Size

开发阶段可以先：

```text id="g1n5wa"
chunk_size = 800 characters
overlap = 100
```

例如：

```text id="ly1c9f"
Chunk 1
0 ─────── 800

Chunk 2
700 ─────── 1500

Chunk 3
1400 ─────── 2200
```

这就是：

# Chunk Overlap

---

# 11. 为什么需要 Overlap？

没有 overlap：

```text id="z6p5yk"
Chunk 1:
AWS Bedrock is a...

Chunk 2:
large-scale foundation model...
```

一句话可能刚好被切断。

Overlap：

```text id="z1o0z3"
Chunk 1:
AWS Bedrock is a large-scale foundation model...

Chunk 2:
foundation model available through AWS...
```

上下文更完整。

---

# 12. 创建 Document Model

```text id="0y99cu"
backend/app/models/document.py
```

```python id="2e9g1u"
class Document(Base):

    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"),
        index=True,
        nullable=False,
    )

    filename: Mapped[str] = mapped_column(
        String(255),
        nullable=False,
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.utcnow,
    )
```

---

# 13. 创建 DocumentChunk

```text id="p9ed78"
backend/app/models/document_chunk.py
```

```python id="7qff8u"
from pgvector.sqlalchemy import Vector


class DocumentChunk(Base):

    __tablename__ = "document_chunks"

    id: Mapped[int] = mapped_column(
        primary_key=True
    )

    document_id: Mapped[int] = mapped_column(
        ForeignKey("documents.id"),
        index=True,
        nullable=False,
    )

    content: Mapped[str] = mapped_column(
        Text,
        nullable=False,
    )

    embedding = mapped_column(
        Vector(1024)
    )

    chunk_index: Mapped[int] = mapped_column(
        nullable=False
    )
```

**注意：`1024` 只是示例。**

实际维度必须和你选择的 embedding model 输出维度一致。

---

# 14. 安装 pgvector Python

```powershell id="9e54y8"
pip install pgvector
```

如果你使用：

```text id="djof2f"
requirements.txt
```

加入：

```text id="f3kqf5"
pgvector
```

---

# 15. 创建 Embedding Service

```text id="b3x1d6"
app/services/embedding_service.py
```

```python id="b1z7a3"
class EmbeddingService:

    def __init__(self, bedrock_client, model_id):
        self.client = bedrock_client
        self.model_id = model_id

    def embed(self, text: str):

        response = self.client.invoke_model(
            modelId=self.model_id,
            body=...
        )

        return ...
```

今天先把接口设计好。

重点不是把代码写死。

---

# 16. 为什么 Embedding Service 单独存在？

因为以后可能：

```text id="0xq4qy"
Amazon Titan Embeddings
```

换成：

```text id="w5xk4e"
Cohere Embeddings
```

甚至：

```text id="3a3p3x"
OpenAI Embeddings
```

你的：

```text id="k19e5d"
RAGService
```

不应该改变。

应该：

```text id="p8kmjv"
RAGService
     ↓
EmbeddingService
     ↓
Embedding Provider
```

这就是：

# Loose Coupling

---

# 17. AWS Bedrock Embedding

今天可以使用 Amazon Bedrock 中支持的 embedding model。

AWS 官方 Bedrock 文档：

[Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html?utm_source=chatgpt.com)

具体模型 ID、embedding 维度和请求格式，**以你 AWS Region 当前可用的模型为准**。

先检查：

```text id="1p1u8x"
AWS Console
 ↓
Amazon Bedrock
 ↓
Models
 ↓
Embedding Models
```

---

# 18. Chunk Service

创建：

```text id="kvmy8m"
app/services/chunk_service.py
```

```python id="f6w0y7"
class ChunkService:

    def split(
        self,
        text,
        chunk_size=800,
        overlap=100,
    ):

        chunks = []

        start = 0

        while start < len(text):

            end = start + chunk_size

            chunk = text[start:end]

            chunks.append(chunk)

            start = end - overlap

        return chunks
```

这是最简单的版本。

以后升级：

```text id="6b5g0s"
Recursive Character Splitter
Semantic Chunking
Markdown Chunking
Document-aware Chunking
```

---

# 19. Document Ingestion Pipeline

现在建立：

```text id="v0t3ce"
RAGIngestionService
```

逻辑：

```text
PDF
 ↓
Text Extraction
 ↓
ChunkService
 ↓
EmbeddingService
 ↓
PostgreSQL
```

伪代码：

```python id="r36w1s"
def ingest(document):

    text = extract_text(document)

    chunks = chunk_service.split(text)

    for index, chunk in enumerate(chunks):

        vector = embedding_service.embed(
            chunk
        )

        save_chunk(
            content=chunk,
            embedding=vector,
            chunk_index=index,
        )
```

---

# 20. Vector Search

这是 RAG 最核心的一步。

用户：

```text id="n6kh5w"
What is Amazon Bedrock?
```

首先：

```text id="y91d7x"
Question
 ↓
Embedding
 ↓
Query Vector
```

然后 PostgreSQL：

```text id="j7y87d"
Query Vector
      ↓
Vector Similarity
      ↓
Top K
```

---

# 21. pgvector Similarity Search

例如：

```sql id="eb7v2e"
SELECT
    id,
    content
FROM document_chunks
ORDER BY embedding <=> :query_vector
LIMIT 5;
```

这里：

```text id="9b08z7"
<=>
```

用于 cosine distance。

你要理解：

```text id="xw0b7f"
Query Vector
      │
      ▼
Similarity
      │
      ▼
Most Relevant Chunks
```

---

# 22. 为什么 Top K？

不要：

```text id="3ghx94"
找到 1,000 chunks
全部发给 LLM
```

而是：

```text id="4i6e8v"
Top 3
Top 5
Top 10
```

例如：

```text id="v2uyk3"
Question
 ↓
Vector Search
 ↓
Top 5 chunks
 ↓
LLM
```

开发阶段先：

```python id="jcf4ma"
TOP_K = 5
```

---

# 23. RAGService

现在终于把它连接起来。

```text id="p8tj7c"
app/services/rag_service.py
```

```python id="ce6l4j"
class RAGService:

    def __init__(
        self,
        embedding_service,
        vector_repository,
        llm_service,
    ):
        self.embedding_service = (
            embedding_service
        )

        self.vector_repository = (
            vector_repository
        )

        self.llm_service = (
            llm_service
        )

    def answer(self, question):

        query_vector = (
            self.embedding_service
            .embed(question)
        )

        chunks = (
            self.vector_repository
            .similarity_search(
                query_vector,
                limit=5,
            )
        )

        context = "\n\n".join(
            chunk.content
            for chunk in chunks
        )

        prompt = f"""
Answer the question using
the provided context.

Context:
{context}

Question:
{question}
"""

        return self.llm_service.generate(
            prompt
        )
```

---

# 24. 这就是 RAG

```text id="4r9nqp"
Question
   │
   ▼
Embedding
   │
   ▼
Vector Search
   │
   ▼
Top 5 Chunks
   │
   ▼
Context
   │
   ▼
Prompt
   │
   ▼
Bedrock
   │
   ▼
Answer
```

你现在应该把这张图彻底理解。

---

# 25. RAG Prompt

不要：

```text id="5m1t6f"
Answer this question:
{question}
```

而应该：

```text id="8e9v3j"
You are an AI assistant.

Answer the question using
only the provided context.

If the answer is not present
in the context, say that you
do not have enough information.

Context:
{context}

Question:
{question}
```

这个设计会明显降低：

```text id="qwrq9n"
Hallucination
```

---

# 26. 用户数据隔离

这是你的项目非常重要的一点。

不能：

```sql id="h2h5sh"
SELECT *
FROM document_chunks
ORDER BY ...
LIMIT 5;
```

因为 User A：

```text id="ujy2s6"
private.pdf
```

User B：

```text id="ny3i1b"
```

可能搜到 A 的资料。

应该：

```text id="zskm20"
User ID
 ↓
Documents
 ↓
Chunks
 ↓
Vector Search
```

例如：

```sql id="2ozr2e"
SELECT
    dc.id,
    dc.content
FROM document_chunks dc
JOIN documents d
    ON dc.document_id = d.id
WHERE d.user_id = :user_id
ORDER BY dc.embedding <=> :query_vector
LIMIT 5;
```

这是生产级 RAG 必须考虑的问题。

---

# 27. RAG API

创建：

```text id="x7hmh2"
POST /api/rag/query
```

Request：

```json id="zrlcny"
{
  "question": "What is Amazon Bedrock?"
}
```

Response：

```json id="xg6g7n"
{
  "answer": "Amazon Bedrock is...",
  "sources": [
    {
      "document": "AWS_LLM_Architecture.pdf",
      "chunk": 3
    }
  ]
}
```

---

# 28. 为什么要返回 Sources？

这是 RAG 与普通 Chat 最大的产品区别之一。

用户看到：

```text id="4f1pvn"
Answer:

Amazon Bedrock is...

Sources:

AWS_LLM_Architecture.pdf
Chunk 3
```

这样用户可以：

```text id="c5cmu1"
验证答案
```

这叫：

# Grounded Generation

---

# 29. 今天先不要做的东西

Day 11 **不要一次加入**：

```text id="2mjtvf"
LangChain
LlamaIndex
LangGraph
Agent
Hybrid Search
Reranker
GraphRAG
Multimodal RAG
```

先理解：

```text id="h9ekp7"
Document
 ↓
Chunk
 ↓
Embedding
 ↓
Vector DB
 ↓
Similarity Search
 ↓
Context
 ↓
LLM
```

这是地基。

---

# 30. Day 11 测试

准备一个小文本：

```text id="pfk2u6"
AWS Bedrock is a fully managed
service that provides access to
foundation models through APIs.

Amazon RDS is a managed
relational database service.

FastAPI is a Python web framework
for building APIs.
```

切成：

```text id="e0i5pb"
Chunk 1
AWS Bedrock...

Chunk 2
Amazon RDS...

Chunk 3
FastAPI...
```

---

# 31. Test 1

Question：

```text id="w9y8tm"
What is AWS Bedrock?
```

应该返回：

```text id="h2xx4j"
Bedrock...
```

而不是：

```text id="g7r1y9"
FastAPI...
```

---

# 32. Test 2

```text id="ojqj95"
What database does the
application use?
```

应该找到：

```text id="r3dbx3"
Amazon RDS
```

---

# 33. Test 3

```text id="l0n2p4"
What is FastAPI?
```

应该：

```text id="h5j8r7"
FastAPI...
```

---

# 34. Test 4 — 不存在的信息

问题：

```text id="1e4xw9"
What operating system does
this company use?
```

如果文档没有：

```text id="zy5jcg"
I don't have enough information
from the provided context.
```

不要让 LLM 自己编。

---

# 35. Day 11 面试题

### Q1：什么是 RAG？

> RAG retrieves relevant information from an external knowledge base and provides that information to an LLM as context before generation.

---

### Q2：为什么需要 Embedding？

> Embeddings represent text as vectors so that semantically similar content can be retrieved using vector similarity search.

---

### Q3：为什么要 Chunk？

> Large documents need to be split into smaller pieces so retrieval can return focused and relevant context.

---

### Q4：为什么要 Chunk Overlap？

> Overlap helps preserve context across chunk boundaries.

---

### Q5：为什么 Vector Search 要 Top-K？

> To retrieve a small set of highly relevant chunks while controlling context size, latency, and cost.

---

### Q6：RAG 如何减少 Hallucination？

核心：

```text id="h2c2mk"
Retrieve
 ↓
Relevant Context
 ↓
Grounded Prompt
 ↓
LLM
```

不是说 RAG 可以完全消除 hallucination。

---

# 36. Day 11 Git Commit

```powershell id="3c8ef4"
git add .
```

```powershell id="uhbq2j"
git commit -m "Day 11: implement basic RAG pipeline"
```

```powershell id="ef3w71"
git push
```

README 加：

```markdown id="q6xyc7"
## RAG Architecture

The application implements a basic
Retrieval-Augmented Generation pipeline:

Document
→ Chunking
→ Embedding
→ PostgreSQL + pgvector
→ Similarity Search
→ Context Construction
→ Amazon Bedrock
→ Grounded Answer
```

---

# 37. Day 11 最终架构

```text
                         React
                           │
                           ▼
                      FastAPI API
                           │
                           ▼
                       RAGService
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        Embedding      PostgreSQL     Bedrock
          Model          pgvector        LLM
             │             │             │
             │             ▼             │
             │        Vector Search      │
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    Relevant Chunks
                           │
                           ▼
                      Context
                           │
                           ▼
                         Prompt
                           │
                           ▼
                         Answer
```

---

# 🎯 Day 11 完成标准

```text
□ 理解 RAG
□ PDF/Text → Document
□ Document → Chunks
□ Chunk → Embedding
□ PostgreSQL + pgvector
□ Vector Similarity Search
□ Top-K Retrieval
□ RAGService
□ Context Construction
□ Grounded Prompt
□ Sources
□ User Data Isolation
□ RAG API
□ RAG Tests
□ GitHub Commit
```

### Day 12

明天把今天的基础 RAG 升级成真正的**生产级 Document RAG**：

```text
PDF
 │
 ▼
S3
 │
 ▼
Document Processing
 │
 ├── PDF Text Extraction
 ├── Metadata
 ├── Chunking
 ├── Embedding
 └── pgvector
          │
          ▼
       Retrieval
          │
     ┌────┴────┐
     ▼         ▼
 Metadata    Vector
 Filtering   Search
     │         │
     └────┬────┘
          ▼
       Top-K
          │
          ▼
        Rerank
          │
          ▼
         LLM
          │
          ▼
      Answer + Citations
```

**Day 12 的重点会是：AWS S3 + PDF 上传 + 文档解析 + 异步 Ingestion Pipeline。**
