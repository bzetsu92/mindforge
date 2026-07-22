Tôi sẽ thiết kế DB theo hướng **Knowledge Graph + Evidence + Human Semantic Layer**, không theo kiểu RAG thuần.

Mục tiêu DB phải đáp ứng:

1. Lưu project
2. Lưu file Markdown/CSV metadata
3. Lưu concept (node)
4. Lưu quan hệ giữa concept
5. Lưu bằng chứng (evidence)
6. Lưu lịch sử thay đổi
7. Hỗ trợ search + vector
8. Hỗ trợ impact analysis

Dùng:

* PostgreSQL
* pgvector extension

---

# ERD tổng quan

```text
                         projects
                            |
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v

    documents          concepts             users
        |                   |
        |                   |
        v                   |
 document_sections          |
        |                   |
        |                   |
        +---------+---------+
                  |
                  v

          document_concepts
                  |
                  |
                  v

              relations
                  |
                  |
        +---------+---------+
        |                   |
        v                   v

 source_concept        target_concept


                  |
                  v

             evidences


                  |
                  v

             embeddings

```

---

# 1. projects

Project là container lớn nhất.

```sql
projects
```

```sql
id                  UUID PK
name                varchar
description         text
status              varchar
created_by          UUID
created_at          timestamp
updated_at          timestamp
```

Ví dụ:

```
E-Commerce Platform
```

---

# 2. users

Người dùng hệ thống.

```sql
users
```

```sql
id              UUID PK
email           varchar
name            varchar
role            varchar
created_at      timestamp
```

Role:

```
OWNER
MEMBER
VIEWER
```

---

# 3. project_members

Một user có nhiều project.

```sql
project_members
```

```sql
id

project_id FK

user_id FK

permission

created_at
```

---

# 4. documents

File vật lý.

Quan trọng:

**Không lưu content lớn ở đây.**

File nằm:

```
filesystem
MinIO
S3
```

DB chỉ index.

```sql
documents
```

```sql
id                  UUID PK

project_id          FK

filename            varchar

storage_path        varchar

file_type           varchar

hash                varchar

version             int

status              varchar

created_by          UUID

created_at

updated_at
```

Ví dụ:

```
payment-flow.md

storage_path:

projects/a/docs/payment-flow.md
```

---

# 5. document_sections

Chia document thành vùng semantic.

Ví dụ:

```md
# Payment

## Retry Logic

...
```

Thành:

```
Document

 |
 +-- Payment Section

 |
 +-- Retry Logic Section

```

Table:

```sql
document_sections
```

```sql
id

document_id FK

parent_id FK NULL

title

content

section_type

start_line

end_line

created_at
```

Ví dụ:

```
payment.md

section:
Retry Logic

line:
50-80
```

---

# 6. concepts (CORE)

Đây là node.

Thay vì keyword.

```sql
concepts
```

```sql
id UUID PK

project_id FK

name

type

description

status

created_by

created_at

updated_at
```

Ví dụ:

```
Payment

type:
DOMAIN_CONCEPT
```

---

## type

Ví dụ:

```
DOMAIN
SERVICE
API
DATABASE
DECISION
PERSON
SYSTEM
TECHNOLOGY
```

---

# 7. concept_aliases

Giải quyết:

```
PaymentService
payment-service
Payment Service
```

cùng một concept.

```sql
concept_aliases
```

```sql
id

concept_id FK

alias

created_at
```

Ví dụ:

```
Payment

aliases:

payment-service
pay-service

```

---

# 8. document_concepts

Mapping:

Document liên quan concept nào.

```sql
document_concepts
```

```sql
id

document_id FK

concept_id FK

confidence

source

created_by

created_at
```

Ví dụ:

```
payment.md

     |
     |
Payment
```

source:

```
USER
AI
IMPORT
```

---

# 9. relations (GRAPH EDGE)

Đây là cạnh graph.

```sql
relations
```

```sql
id UUID PK

project_id FK

source_concept_id FK

target_concept_id FK

relation_type

confidence

status

created_by

created_at
```

Ví dụ:

```
Payment

depends_on

Stripe
```

---

## relation_type

Ví dụ:

```
DEPENDS_ON

USES

CONTAINS

IMPLEMENTS

CALLS

CREATES

UPDATES

RELATED_TO

REPLACES
```

---

# 10. evidences (rất quan trọng)

Mọi relation phải truy ra nguồn.

```sql
evidences
```

```sql
id

relation_id FK NULL

concept_id FK NULL

document_id FK

section_id FK

quote

line_start

line_end

confidence

created_at
```

Ví dụ:

```
Relation:

Payment uses Stripe


Evidence:

payment.md

line 30

"PaymentService calls Stripe API"
```

---

# 11. embeddings

Cho semantic search.

Dùng pgvector.

```sql
embeddings
```

```sql
id

entity_type

entity_id

content

embedding VECTOR

created_at
```

entity_type:

```
DOCUMENT

SECTION

CONCEPT
```

Ví dụ:

```
Payment concept embedding
```

---

# 12. decisions

Tôi nghĩ nên tách riêng.

Vì quyết định dự án rất quan trọng.

Ví dụ:

"Why use Redis?"

```sql
decisions
```

```sql
id

project_id

title

context

decision

status

created_at
```

---

Relation:

```
Decision

supports

Concept
```

---

# 13. change_events

Sau này làm impact analysis.

Lưu lịch sử thay đổi.

```sql
change_events
```

```sql
id

project_id

entity_type

entity_id

change_type

old_value

new_value

created_at
```

Ví dụ:

```
PaymentStatus

changed

v1 -> v2
```

---

# 14. ingestion_jobs

Theo dõi import.

```sql
ingestion_jobs
```

```sql
id

project_id

document_id

status

progress

error

created_at
```

Status:

```
PENDING
PROCESSING
DONE
FAILED
```

---

# Index quan trọng

## Concept search

```sql
CREATE INDEX idx_concept_name
ON concepts(name);
```

---

## Graph traversal

Rất quan trọng:

```sql
CREATE INDEX idx_relation_source
ON relations(source_concept_id);


CREATE INDEX idx_relation_target
ON relations(target_concept_id);
```

---

## Document lookup

```sql
CREATE INDEX idx_document_project
ON documents(project_id);
```

---

## Vector

pgvector:

```sql
CREATE INDEX embedding_idx
ON embeddings
USING ivfflat (embedding vector_cosine_ops);
```

---

# Query ví dụ

## "Payment ảnh hưởng gì?"

Tìm node:

```sql
SELECT *
FROM concepts
WHERE name='Payment';
```

Traversal:

```
Payment

 |
 relations

 |
 affected nodes

```

Lấy evidence:

```
relations
    |
evidences
    |
documents
```

Cuối cùng đưa context cho LLM.

---

# Điểm tôi muốn nhấn mạnh

DB này **không phải RAG database**.

Nó là:

```
Document Store
+
Semantic Layer
+
Knowledge Graph
+
Evidence System
```

Trong đó:

* `concepts` = bộ não
* `relations` = sự hiểu biết
* `documents` = trí nhớ gốc
* `evidence` = sự tin cậy

Đây là cấu trúc đủ tốt để sau này mở rộng:

* AI agent
* impact analysis
* architecture analysis
* requirement traceability
* project history.
