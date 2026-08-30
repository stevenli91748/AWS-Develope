# Day 12 — S3 + PDF 上传 + 异步 RAG Ingestion

今天把 Day 11 的“文本 RAG Demo”升级成真正的 **AWS Document RAG Pipeline**。

目标是实现：

```text
React
  │
  │ Upload PDF
  ▼
FastAPI
  │
  ▼
Amazon S3
  │
  │ event
  ▼
Document Ingestion Worker
  │
  ├── Download PDF
  ├── Extract Text
  ├── Chunk
  ├── Embedding
  └── PostgreSQL + pgvector
```

最终用户可以：

> 上传一个 PDF → 系统自动处理 → 然后直接问 PDF 里的内容。

---

# 1. Day 12 今天完成什么

```text
□ Amazon S3 Bucket
□ IAM 权限
□ PDF Upload API
□ React PDF Upload
□ S3 Object
□ Document Status
□ PDF Text Extraction
□ Chunking
□ Embedding
□ pgvector
□ Async Ingestion
□ Error Handling
□ RAG Query
□ Source Citation
```

---

# 2. 今天的最终架构

```text
                    React
                      │
                 Upload PDF
                      │
                      ▼
                  FastAPI
                      │
                Presigned URL
                      │
                      ▼
                  Amazon S3
                      │
                 PDF Object
                      │
                      ▼
              Ingestion Worker
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Extract      Chunk      Metadata
        Text
          │
          ▼
      Embedding
          │
          ▼
 PostgreSQL + pgvector
          │
          ▼
    Vector Search
          │
          ▼
      RAGService
          │
          ▼
       Bedrock
          │
          ▼
    Answer + Sources
```

---

# 3. 为什么使用 S3？

不要让用户：

```text
React
 ↓
FastAPI
 ↓
PDF
 ↓
FastAPI Memory
```

因为 PDF 可能：

```text
10 MB
100 MB
500 MB
```

而且 Backend 不应该承担大量文件传输。

AWS 更合理的架构：

```text
React
   ↓
S3
```

Backend 只负责：

```text
Authentication
Authorization
Metadata
Presigned URL
Processing
```

---

# 4. 创建 S3 Bucket

AWS Console：

```text
AWS Console
 ↓
S3
 ↓
Create bucket
```

例如：

```text
llm-rag-documents-<unique-id>
```

Bucket 必须使用**全局唯一名称**。

建议：

```text
Block all public access = ON
```

不要让 PDF 公开访问。

---

# 5. Bucket 目录设计

不要把所有文件：

```text
bucket/
  file1.pdf
  file2.pdf
```

建议：

```text
bucket/
│
├── users/
│   ├── user-123/
│   │   ├── documents/
│   │   │   ├── doc-001/
│   │   │   │   └── original.pdf
│   │   │   └── doc-002/
│   │
│   └── user-456/
```

也就是：

```text
S3
 ↓
user_id
 ↓
document_id
```

这样以后做权限控制非常容易。

---

# 6. S3 Object Key

Python：

```python
s3_key = (
    f"users/{user_id}/"
    f"documents/{document_id}/"
    f"original.pdf"
)
```

例如：

```text
users/123/documents/987/original.pdf
```

---

# 7. IAM

Backend 不应该拥有：

```text
AmazonS3FullAccess
```

生产环境应该最小权限。

例如：

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
```

限制在：

```text
arn:aws:s3:::YOUR_BUCKET/users/*
```

AWS IAM 官方文档：

[AWS IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html?utm_source=chatgpt.com)

---

# 8. Python 安装 boto3

```powershell
pip install boto3
```

加入：

```text
boto3
```

如果使用：

```text
requirements.txt
```

更新：

```powershell
pip freeze > requirements.txt
```

---

# 9. S3 Service

创建：

```text
backend/app/services/s3_service.py
```

```python
import boto3


class S3Service:

    def __init__(self, bucket_name):

        self.bucket_name = bucket_name

        self.client = boto3.client(
            "s3"
        )

    def upload_file(
        self,
        file_path,
        key,
    ):

        self.client.upload_file(
            file_path,
            self.bucket_name,
            key,
        )

    def download_file(
        self,
        key,
        file_path,
    ):

        self.client.download_file(
            self.bucket_name,
            key,
            file_path,
        )
```

---

# 10. AWS Credentials

本地开发**不要**：

```python
boto3.client(
    "s3",
    aws_access_key_id="xxx",
    aws_secret_access_key="xxx"
)
```

更好的方式：

```text
AWS CLI
 ↓
Credential Provider Chain
 ↓
boto3
```

本地：

```powershell
aws configure
```

然后：

```powershell
aws sts get-caller-identity
```

如果成功：

```text
Account
Arn
UserId
```

说明 AWS credentials 正常。

---

# 11. EC2 / ECS 部署以后

生产环境不要把 Access Key 写进：

```text
.env
```

而应该使用：

```text
EC2 IAM Role
```

或者：

```text
ECS Task Role
```

或者：

```text
EKS IAM / Pod Identity
```

这就是 AWS production security 的基本思路。

---

# 12. Document Model 升级

Day 11：

```text
Document
```

今天加入：

```text
s3_key
status
file_size
mime_type
```

例如：

```python
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

    s3_key: Mapped[str] = mapped_column(
        String(1024),
        nullable=False,
    )

    status: Mapped[str] = mapped_column(
        String(30),
        default="uploaded",
        nullable=False,
    )

    file_size: Mapped[int | None] = (
        mapped_column()
    )

    mime_type: Mapped[str | None] = (
        mapped_column(String(100))
    )

    created_at: Mapped[datetime] = (
        mapped_column(
            DateTime,
            default=datetime.utcnow,
        )
    )
```

---

# 13. Document Status

非常重要。

定义：

```text
uploaded
processing
completed
failed
```

状态：

```text
uploaded
   ↓
processing
   ↓
completed
```

如果失败：

```text
processing
   ↓
failed
```

React 可以显示：

```text
AWS.pdf
Uploading...
```

然后：

```text
AWS.pdf
Processing...
```

最后：

```text
AWS.pdf
Ready
```

---

# 14. 为什么需要 Status？

因为 PDF ingestion 是：

```text
异步任务
```

不是：

```text
POST /upload
 ↓
等待 60 秒
 ↓
返回
```

用户体验很差。

正确：

```text
Upload
 ↓
202 Accepted
 ↓
Processing
 ↓
Ready
```

---

# 15. Upload API

创建：

```text
POST /api/documents
```

Request：

```text
multipart/form-data
```

例如：

```python
@router.post("/documents")
async def upload_document(
    file: UploadFile,
    db: Session = Depends(get_db),
    user = Depends(get_current_user),
):
    ...
```

---

# 16. 不推荐直接让 FastAPI 上传大文件到 S3

开发 Demo 可以：

```text
React
 ↓
FastAPI
 ↓
S3
```

但是生产级架构更推荐：

```text
React
 ↓
FastAPI
 ↓
Presigned URL
 ↓
S3
```

原因：

```text
Browser
   │
   └──────→ S3
```

Backend 不经过文件内容。

---

# 17. Presigned URL

FastAPI：

```python
def create_upload_url(
    self,
    key,
    content_type,
):

    return self.client.generate_presigned_url(
        "put_object",

        Params={
            "Bucket": self.bucket_name,
            "Key": key,
            "ContentType": content_type,
        },

        ExpiresIn=900,
    )
```

15 分钟有效。

---

# 18. Upload 流程

React：

```text
POST /api/documents/upload-url
```

Backend：

```text
1. Authenticate
2. Create Document
3. Generate S3 key
4. Generate Presigned URL
5. Return URL
```

Response：

```json
{
  "document_id": 123,
  "upload_url": "..."
}
```

React：

```text
PUT upload_url
```

直接上传 S3。

---

# 19. React 上传

概念代码：

```typescript
const response = await fetch(
  "/api/documents/upload-url",
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      filename: file.name,
      content_type: file.type,
    }),
  }
);

const data = await response.json();
```

然后：

```typescript
await fetch(
  data.upload_url,
  {
    method: "PUT",
    headers: {
      "Content-Type": file.type,
    },
    body: file,
  }
);
```

现在：

```text
Browser
   │
   └───────────────→ S3
```

---

# 20. PDF Processing

上传以后：

```text
S3
 ↓
Ingestion Worker
 ↓
PDF
 ↓
Text Extraction
```

Python 可以使用：

```text
pypdf
```

安装：

```powershell
pip install pypdf
```

---

# 21. PDF Extractor

创建：

```text
app/services/pdf_service.py
```

```python
from pypdf import PdfReader


class PDFService:

    def extract_text(
        self,
        file_path,
    ):

        reader = PdfReader(
            file_path
        )

        pages = []

        for page in reader.pages:

            text = page.extract_text()

            if text:
                pages.append(text)

        return "\n".join(pages)
```

---

# 22. 注意扫描 PDF

`pypdf` 对：

```text
Text PDF
```

比较合适。

但是：

```text
Scanned PDF
```

可能没有文本层。

例如：

```text
PDF
 ↓
Image
```

这时候：

```text
pypdf
 ↓
空文本
```

以后可以升级：

```text
Amazon Textract
```

用于 OCR / document extraction。

AWS Textract 官方文档：

[Amazon Textract](https://docs.aws.amazon.com/textract/latest/dg/what-is.html?utm_source=chatgpt.com)

**今天先不加入 Textract。**

---

# 23. Ingestion Pipeline

现在创建：

```text
app/services/ingestion_service.py
```

逻辑：

```python
def ingest(document):

    document.status = "processing"

    download_from_s3(
        document.s3_key
    )

    text = pdf_service.extract_text(
        local_file
    )

    chunks = chunk_service.split(
        text
    )

    for index, chunk in enumerate(chunks):

        embedding = (
            embedding_service
            .embed(chunk)
        )

        save_chunk(
            document_id=document.id,
            chunk_index=index,
            content=chunk,
            embedding=embedding,
        )

    document.status = "completed"
```

---

# 24. 失败处理

必须：

```python
try:

    ingest(document)

except Exception:

    document.status = "failed"

    db.commit()

    raise
```

这样：

```text
PDF
 ↓
processing
 ↓
ERROR
 ↓
failed
```

React：

```text
AWS.pdf
❌ Processing failed
```

---

# 25. 为什么不能让 Upload API 等待？

错误：

```text
POST /upload
 ↓
Download
 ↓
PDF parsing
 ↓
Chunk
 ↓
Embedding 1000 chunks
 ↓
Vector DB
 ↓
Response
```

可能几十秒甚至更久。

正确：

```text
POST /upload
 ↓
S3
 ↓
202
```

然后：

```text
Worker
 ↓
Process
```

---

# 26. 最简单的异步方案

开发阶段可以使用：

```text
FastAPI BackgroundTasks
```

例如：

```python
background_tasks.add_task(
    ingestion_service.ingest,
    document.id,
)
```

但是要知道：

> `BackgroundTasks` 更适合轻量后台任务，不是可靠的生产级任务队列。

如果 EC2/ECS 进程崩溃：

```text
Task
 ↓
可能丢失
```

---

# 27. 生产级方案

以后升级：

```text
S3
 ↓
S3 Event
 ↓
SQS
 ↓
Worker
 ↓
RAG Ingestion
```

完整：

```text
                   S3
                    │
                ObjectCreated
                    │
                    ▼
                   SQS
                    │
                    ▼
              Ingestion Worker
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        PDF       Chunk    Embedding
                              │
                              ▼
                         pgvector
```

这才是比较好的 AWS 架构。

---

# 28. 今天先用 BackgroundTasks

因为我们现在的目标是：

```text
学习
+
跑通
```

不是今天就搭一个巨型生产系统。

Day 12：

```text
S3
+
FastAPI BackgroundTasks
```

Day 18/19 再升级：

```text
SQS
+
Worker
```

---

# 29. Document Chunk Metadata

Day 11：

```text
content
embedding
chunk_index
```

今天增加：

```text
page_number
```

最终：

```python
class DocumentChunk(Base):

    id

    document_id

    chunk_index

    page_number

    content

    embedding
```

这样回答：

```text
Where did you find this?
```

可以：

```text
AWS.pdf
Page 12
```

---

# 30. PDF Page-aware Chunking

不要：

```text
PDF
 ↓
全部文字
 ↓
Chunk
```

更好的：

```text
Page 1
 ↓
Chunks

Page 2
 ↓
Chunks

Page 3
 ↓
Chunks
```

保存：

```text
page_number
```

例如：

```text
chunk 0 → page 1
chunk 1 → page 1
chunk 2 → page 2
chunk 3 → page 2
```

这对 Citation 很重要。

---

# 31. RAG Query

现在用户：

```text
What does the document say about AWS Bedrock?
```

流程：

```text
Question
 ↓
Embedding
 ↓
pgvector
 ↓
Top 5
 ↓
Chunks
 ↓
Context
 ↓
Bedrock
```

但是现在增加：

```text
document_id
```

例如：

```text
Question
+
document_id
```

可以实现：

> 只在指定 PDF 中搜索。

---

# 32. Metadata Filtering

SQL：

```sql
SELECT
    dc.id,
    dc.content,
    dc.page_number
FROM document_chunks dc
JOIN documents d
    ON dc.document_id = d.id
WHERE
    d.user_id = :user_id
    AND d.id = :document_id
ORDER BY
    dc.embedding <=> :query_vector
LIMIT 5;
```

这是非常重要的：

```text
User isolation
+
Document isolation
+
Vector Search
```

---

# 33. RAG Response

返回：

```json
{
  "answer": "Amazon Bedrock provides...",
  "sources": [
    {
      "document_id": 123,
      "filename": "AWS.pdf",
      "page": 12,
      "chunk_id": 45
    }
  ]
}
```

React：

```text
Answer
────────────────────
Amazon Bedrock provides...

Sources
────────────────────
📄 AWS.pdf — Page 12
```

---

# 34. 今天完成后的用户体验

用户打开：

```text
AI Knowledge Assistant
```

点击：

```text
+ Upload Document
```

选择：

```text
AWS_Architecture.pdf
```

看到：

```text
AWS_Architecture.pdf
Uploading...
```

然后：

```text
AWS_Architecture.pdf
Processing...
```

最后：

```text
AWS_Architecture.pdf
✓ Ready
```

然后：

```text
User:
What is the architecture of this application?
```

AI：

```text
The application uses React,
FastAPI, PostgreSQL and Amazon Bedrock...

Sources:
AWS_Architecture.pdf — Page 4
```

这已经非常接近一个真正的：

# AI Knowledge Base SaaS

---

# 35. 今天的测试

### Test 1 — 上传

```text
AWS.pdf
```

检查：

```text
S3
 └── users/xxx/documents/xxx/original.pdf
```

---

### Test 2 — Document

数据库：

```text
documents

id
user_id
filename
s3_key
status
```

---

### Test 3 — Processing

确认：

```text
uploaded
 ↓
processing
 ↓
completed
```

---

### Test 4 — Chunks

```sql
SELECT
    document_id,
    chunk_index,
    page_number
FROM document_chunks
ORDER BY chunk_index;
```

---

### Test 5 — Vector

检查：

```text
embedding
```

不是：

```text
NULL
```

---

### Test 6 — RAG

问：

```text
What is Amazon Bedrock?
```

应该返回：

```text
Answer
+
AWS.pdf
+
Page
```

---

# 36. Security Test

User A：

```text
AWS.pdf
```

User B：

```text
POST /api/rag/query
document_id=123
```

必须：

```text
403 / 404
```

不能返回：

```text
AWS.pdf content
```

这个测试一定做。

---

# 37. Day 12 面试题

### Q1：为什么使用 S3？

> S3 provides durable, scalable object storage and allows the backend to avoid handling large file payloads directly.

### Q2：为什么使用 Presigned URL？

> It allows clients to upload directly to S3 without exposing AWS credentials or routing large files through the application server.

### Q3：为什么 ingestion 要异步？

> Document processing can be expensive and time-consuming, so asynchronous processing improves API responsiveness and scalability.

### Q4：生产环境为什么考虑 SQS？

```text
Reliable Queue
+
Retry
+
Decoupling
+
Scalability
```

### Q5：为什么保存 page_number？

为了：

```text
Citation
Debugging
User Verification
```

---

# 38. 今天的 Git Commit

```powershell
git add .
```

```powershell
git commit -m "Day 12: add S3 document ingestion pipeline"
```

```powershell
git push
```

README：

```markdown
## Document RAG

The application supports:

- Secure PDF upload
- Amazon S3 storage
- Presigned URLs
- Asynchronous document processing
- PDF text extraction
- Page-aware chunking
- Embeddings
- PostgreSQL pgvector
- Metadata filtering
- Source citations
```

---

# 39. Day 12 项目结构

```text
backend/
│
├── app/
│   │
│   ├── api/
│   │   └── routes/
│   │       ├── chat.py
│   │       ├── documents.py
│   │       └── rag.py
│   │
│   ├── models/
│   │   ├── user.py
│   │   ├── conversation.py
│   │   ├── message.py
│   │   ├── document.py
│   │   ├── document_chunk.py
│   │   └── llm_usage.py
│   │
│   ├── services/
│   │   ├── chat_service.py
│   │   ├── llm_service.py
│   │   ├── context_manager.py
│   │   ├── embedding_service.py
│   │   ├── chunk_service.py
│   │   ├── rag_service.py
│   │   ├── s3_service.py
│   │   ├── pdf_service.py
│   │   └── ingestion_service.py
│   │
│   └── db/
│
├── alembic/
│
└── tests/
```

---

# 40. Day 12 最终 AWS 架构

```text
                         Internet
                            │
                            ▼
                         CloudFront
                            │
                    ┌───────┴───────┐
                    ▼               ▼
                  React          API Gateway
                                    │
                                    ▼
                                  FastAPI
                                    │
                 ┌──────────────────┼─────────────────┐
                 │                  │                 │
                 ▼                  ▼                 ▼
                S3               RDS PostgreSQL     Bedrock
                 │                  │                 │
                 │                  │                 │
                 ▼                  ▼                 │
          Document Storage      pgvector             │
                 │                  ▲                 │
                 │                  │                 │
                 └──────→ Ingestion ┘                 │
                            │                          │
                            ├── PDF Extraction         │
                            ├── Chunking               │
                            └── Embedding ─────────────┘
```

---

# 🎯 Day 12 完成标准

```text
□ S3 Bucket
□ Private S3
□ IAM Least Privilege
□ Presigned Upload
□ PDF Upload UI
□ Document Model
□ Document Status
□ PDF Text Extraction
□ Page-aware Chunking
□ Embedding
□ pgvector
□ User Isolation
□ Document Isolation
□ Async Ingestion
□ RAG Query
□ Source Citation
□ Error Handling
□ Tests
□ GitHub Commit
```

## Day 12 你真正学到的核心

不是简单的：

```text
"怎么上传 PDF"
```

而是这一套 **AI Engineering Pipeline**：

```text
             DOCUMENT INGESTION
                     │
                     ▼
                  Amazon S3
                     │
                     ▼
               Async Processing
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Parse      Chunk     Metadata
          │          │          │
          └──────────┼──────────┘
                     ▼
                 Embedding
                     │
                     ▼
              PostgreSQL
                pgvector
                     │
                     ▼
               Vector Search
                     │
                     ▼
                Top-K Chunks
                     │
                     ▼
                  Bedrock
                     │
                     ▼
             Answer + Citation
```

**Day 13** 下一步建议进入 **RAG Retrieval Optimization**：不只是“向量搜索 Top-5”，而是加入 **Metadata Filtering + Hybrid Search + Reranking + Retrieval Evaluation**。这一步会开始接近美国 AI Engineer 面试中真正会问的 RAG 系统设计。
