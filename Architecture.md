Tôi sẽ thiết kế theo hướng **Atlas – Project Knowledge Intelligence Platform** mà chúng ta vừa thống nhất.

Mục tiêu kiến trúc:

* **Markdown/CSV là nguồn dữ liệu gốc (Source of Truth)**
* **Concept/Knowledge Graph là lớp tri thức**
* **AI chỉ hỗ trợ, không tự quyết định toàn bộ**
* Hệ thống phải:

  * dễ mở rộng
  * không lock vào một AI provider
  * có thể scale ingestion
  * có audit/evidence
  * hỗ trợ query dạng impact analysis

---

# 1. Tổng quan kiến trúc

```text
                         User
                          |
                          |
                    Web Application
                          |
                          |
                  API Gateway / BFF
                          |
        +-----------------+----------------+
        |                                  |
        v                                  v

 Knowledge Service                 File Service
        |                                  |
        |                                  |
        v                                  v

 PostgreSQL                       Object Storage
 (Knowledge Model)                (.md/.csv files)

        |
        |
        v

 Knowledge Engine

        |
        +----------------+
        |                |
        v                v

 Graph Engine       AI Engine

        |                |
        |                |
        v                v

 Impact Analysis    AI Suggestion

```

---

# 2. Monorepo Structure

Tôi khuyên dùng monorepo.

Tool:

* pnpm workspace
* Turborepo

Cấu trúc:

```
atlas/

├── apps/
│
│   ├── web/
│   │
│   ├── api/
│   │
│   └── worker/
│
├── packages/
│
│   ├── database/
│   │
│   ├── knowledge-core/
│   │
│   ├── markdown-parser/
│   │
│   ├── ai-engine/
│   │
│   ├── graph-engine/
│   │
│   ├── shared/
│
├── infra/
│
│   ├── docker-compose.yml
│   ├── postgres/
│   └── minio/
│
└── package.json

```

---

# 3. Frontend Architecture

## Stack

```
Next.js
TypeScript
React
Tailwind
shadcn/ui
React Query
Zustand
React Flow
```

---

## Nhiệm vụ FE

Không xử lý business logic.

Chỉ:

* hiển thị
* interaction
* gọi API

---

# FE Modules

## 3.1 Project Workspace

```
/projects/:id
```

Layout:

```
+--------------------------------+

Project: Payment System


Documents | Graph | Search | AI

+--------------------------------+

```

---

## 3.2 Document Explorer

Hiển thị:

```
documents

payment.md

architecture.md

database.md

```

Click:

```
payment.md

line 20-30

PaymentService uses Stripe

```

---

## 3.3 Knowledge Graph UI

Library:

## React Flow

Model:

```
Node

{
 id,
 type,
 label
}


Edge

{
 source,
 target,
 relation
}

```

Ví dụ:

```
        Payment

          |
        uses

          |

       Stripe


```

---

## 3.4 Concept Management

UI:

```
Concepts

+ Add Concept


Payment

User

Order

Database

```

---

## 3.5 Impact Analysis UI

Input:

```
Change Payment
```

Output:

```
Affected:

Payment API

Order Service

Database

Test Case

```

---

# 4. Backend Architecture

## Stack

```
Fastify
TypeScript
Zod
Drizzle ORM
PostgreSQL
```

---

Backend chia module.

```
api/

src/

 modules/

   project/

   document/

   concept/

   graph/

   search/

   analysis/

   ai/

```

---

# 5. API Layer

Ví dụ:

## Project API

```
POST

/projects


GET

/projects/:id

```

---

## Document API

Upload:

```
POST

/projects/:id/documents

```

Response:

```json
{
 id:"doc_123",
 status:"processing"
}

```

---

## Concept API

Create:

```
POST

/concepts

```

Body:

```json
{
"name":"Payment",
"type":"DOMAIN"
}

```

---

## Graph API

```
GET

/projects/:id/graph

```

Response:

```json
{
nodes:[
 {
 id:"payment",
 label:"Payment"
 }
],

edges:[
 {
 source:"payment",
 target:"stripe",
 relation:"uses"
 }
]

}

```

---

# 6. Database Design

Dùng:

## PostgreSQL

Extension:

```
pgvector
```

---

## Tables

# Project

```sql
projects

id

name

description

created_at

```

---

# Document

```sql
documents

id

project_id

filename

storage_path

hash

version

```

---

# Document Section

Không lưu toàn file.

Chia logical section.

```sql
document_sections


id

document_id

title

content

start_line

end_line

```

Ví dụ:

```
payment.md

Section:

Payment Flow

line 20-50

```

---

# Concept Node

Core.

```sql
concepts


id

project_id

name

type

description

created_by

```

---

# Document Concept

Mapping:

```sql
document_concepts


document_id

concept_id

confidence

source

```

Ví dụ:

```
payment.md

belongs_to

Payment

```

---

# Relation

Graph.

```sql
relations


id

source_concept_id

target_concept_id

relation_type

confidence

```

Ví dụ:

```
Payment

depends_on

Stripe

```

---

# Evidence

Rất quan trọng.

```sql
evidences


id

relation_id

document_id

section_id

quote

```

---

# Embedding

```sql
embeddings


id

entity_id

vector

```

Dùng pgvector.

---

# 7. File Storage

Không lưu Markdown trong DB.

MVP:

```
local filesystem
```

Production:

```
MinIO
/
S3

```

Ví dụ:

```
storage/

project-001/

 documents/

   payment.md

   architecture.md

```

DB:

```
documents.storage_path

project-001/documents/payment.md

```

---

# 8. Worker Architecture

Không xử lý ingestion trong API.

Dùng:

```
BullMQ
+
Redis

```

Flow:

```
Upload file


    |
    v


Create Job


    |
    v


Worker


    |
    +---- Parse Markdown

    |
    +---- Extract sections

    |
    +---- Create embeddings

    |
    +---- Suggest concepts

    |
    +---- Update graph

```

---

# 9. Markdown Processing

Library:

```
remark
unified

```

Flow:

Input:

```md
# Payment

PaymentService uses Stripe

```

AST:

```
Heading

Paragraph

Link

Code block

```

Convert:

```
Document Section

```

---

# 10. AI Engine

Không để AI tự tạo graph.

Architecture:

```
AI Engine


providers/

 openai.ts

 ollama.ts

 anthropic.ts


prompts/

 classify.ts

 summarize.ts

 relation.ts

```

---

AI functions:

## Concept Suggestion

Input:

```
Document section

Existing concepts

```

Output:

```json
[
 {
 concept:"Payment",
 confidence:0.92
 }
]

```

---

## Relation Suggestion

Input:

```
PaymentService calls Stripe

```

Output:

```json
{
source:"PaymentService",

target:"Stripe",

relation:"uses"

}

```

---

# 11. Graph Engine

Không cần Neo4j.

Service:

```
graph-engine

```

Nhiệm vụ:

## Traversal

Ví dụ:

```
Payment

depth 3


```

Query:

```
Payment

 |
 |
API

 |
 |
Database

```

---

# 12. Search System

Hybrid:

## Keyword

Postgres FTS

## Semantic

pgvector

Flow:

```
Question


 |
 |
Embedding


 |
 |
Vector search


 +
Graph search


 |
 |
Answer

```

---

# 13. Luồng nghiệp vụ hoàn chỉnh

## Upload Project

```
User

 |
Upload md

 |
API

 |
Storage

 |
Create Document

 |
Queue Job

 |
Worker

 |
Parser

 |
Concept Mapping

 |
Graph Update

```

---

## Ask Impact

```
User:

"Đổi Payment ảnh hưởng gì?"


        |

API


        |

Find Concept Payment


        |

Graph Traversal


        |

Collect Evidence


        |

LLM summarize


        |

Answer

```

---

# 14. Công nghệ chốt

| Layer    | Technology     |
| -------- | -------------- |
| FE       | Next.js        |
| UI       | shadcn/ui      |
| Graph UI | React Flow     |
| State    | Zustand        |
| API      | Fastify        |
| ORM      | Drizzle        |
| DB       | PostgreSQL     |
| Vector   | pgvector       |
| Queue    | BullMQ         |
| Cache    | Redis          |
| Storage  | MinIO/S3       |
| Parser   | unified/remark |
| AI       | OpenAI SDK     |
| Runtime  | Node.js        |

---

# 15. Thứ tự build thực tế

Không build tất cả.

## Phase 1 – Core

```
Project
Document
Concept
Document-Concept mapping
```

---

## Phase 2 – Graph

```
Relation
Graph UI
Evidence
```

---

## Phase 3 – AI

```
Suggestion
Search
Summary
```

---

## Phase 4 – Intelligence

```
Impact analysis
Change detection
Project memory
```

---

Điểm quan trọng nhất của kiến trúc này:

**Database không phải trung tâm. AI cũng không phải trung tâm.**

Trung tâm là:

```
Concept Model
+
Evidence Model
+
Relation Model
```

Ba thứ này quyết định Atlas có trở thành "project brain" hay chỉ là một chatbot đọc Markdown.
