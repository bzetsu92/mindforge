Tôi sẽ xem đây như bước **phân tích nghiệp vụ + thiết kế core domain**, chưa đi vào code. Mục tiêu là tạo một nền tảng đủ rõ để sau này bạn vẽ:

* Mindmap hệ thống
* Use Case Diagram
* Domain Model
* Architecture Diagram
* Database Design

Tôi sẽ không bắt đầu từ công nghệ, mà bắt đầu từ **bản chất sản phẩm**.

---

# 1. Định nghĩa lại bài toán

## Tên tạm

**Project Knowledge Intelligence System**

Mục tiêu:

> Biến tài liệu dự án (Markdown/CSV) thành một hệ thống tri thức có cấu trúc, có khả năng truy vấn ảnh hưởng, phụ thuộc, lịch sử quyết định và quan hệ giữa các thành phần.

---

# 2. Vấn đề thực tế cần giải quyết

Trong một project lớn:

```text
Project
 |
 +-- Requirement
 |
 +-- Design
 |
 +-- Decision
 |
 +-- API
 |
 +-- Database
 |
 +-- Deployment
 |
 +-- Issue
```

Tri thức nằm rải rác:

```
payment.md
architecture.md
api.md
decision.md
database.md
```

Con người hiểu:

> "Nếu đổi payment_status thì cần sửa những đâu?"

Nhưng máy không biết.

---

# 3. Core concept của hệ thống

Không lấy Document làm trung tâm.

Sai:

```
Document
   |
   |
AI đọc
```

Đúng:

```
Knowledge Model
        |
        |
Document là nguồn chứng cứ
```

---

# 4. Các thực thể cốt lõi (Core Domain)

Tôi đề xuất 7 entity chính.

---

# Entity 1: Project

Đại diện một dự án.

Ví dụ:

```
E-Commerce System
```

Thuộc tính:

```
Project
 |
 + name
 + description
 + members
 + created_at
```

---

# Entity 2: Source Document

Tài liệu gốc.

Ví dụ:

```
payment-flow.md
database-design.md
```

Vai trò:

**Source of Truth**

Không phải graph.

---

Quan hệ:

```
Project

   contains

Document
```

---

# Entity 3: Knowledge Node (quan trọng nhất)

Đây là ý tưởng "keyword" của bạn nhưng nâng cấp.

Không gọi keyword.

Gọi:

## Concept Node

Ví dụ:

```
Payment
User
Order
Authentication
Database
Redis
Stripe
```

Mỗi node có:

```
Concept

id

name

type

description

aliases

owner
```

---

Node type:

Ví dụ:

```
Business Concept

Technical Component

Decision

System

API

Database Entity
```

---

# Entity 4: Relation

Quan hệ giữa node.

Ví dụ:

```
Payment

depends_on

Stripe API
```

hoặc:

```
Order

creates

Transaction
```

Model:

```
Relation

source

target

type

confidence

created_by
```

---

# Entity 5: Evidence

Đây là phần rất quan trọng.

Mọi tri thức phải có nguồn.

Ví dụ:

AI nói:

> Payment sử dụng Stripe

Evidence:

```
payment.md

line 25-30

"PaymentService calls Stripe API"
```

---

Model:

```
Evidence

document_id

position

content

linked_node
```

---

# Entity 6: Annotation

Đây là lớp con người tương tác.

Ví dụ:

User upload:

```
payment.md
```

User gắn:

```
Payment
Stripe
Transaction
```

Đây chính là lớp semantic.

---

# Entity 7: Query / Investigation

User không chỉ search.

User điều tra.

Ví dụ:

```
Impact Analysis

"Đổi Payment ảnh hưởng gì?"
```

---

# 5. User Roles

Tôi nghĩ cần 4 role.

---

## Role 1: Knowledge Owner

Người quản lý tri thức.

Use case:

* tạo concept
* sửa concept
* merge concept
* quản lý taxonomy

---

## Role 2: Developer

Sử dụng để hiểu hệ thống.

Use case:

* tìm dependency
* xem architecture
* xem decision

---

## Role 3: AI Agent

Không phải user nhưng là actor.

Nhiệm vụ:

* suggest concept
* detect relation
* summarize

---

## Role 4: Admin

Quản lý:

* project
* permission
* storage

---

# 6. Core User Flow

## Flow 1: Khởi tạo Project

Actor:

Knowledge Owner

```
Create Project

      |
      v

Upload documents

      |
      v

System parse files

      |
      v

Create document index

```

---

# Flow 2: Xây Semantic Layer

Đây là core.

```
User creates Concept

      |
      v

Payment

      |
      v

Attach documents

      |
      v

Build graph
```

---

# Flow 3: AI Suggestion

```
New document

      |
      v

System analyzes

      |
      v

Suggest:

Payment 90%
API 80%
Database 60%

      |
      v

User approve
```

---

# Flow 4: Knowledge Exploration

User:

```
Show Payment
```

System:

```
Payment

 |
 + Stripe

 |
 + Transaction

 |
 + Payment DB
```

---

# Flow 5: Impact Analysis

Đây là killer feature.

User:

```
Change PaymentStatus
```

System:

Traversal:

```
PaymentStatus

 |
Database

 |
API

 |
Frontend

 |
Test Case

```

Output:

```
Affected:

1. Payment API
2. Order Service
3. Dashboard
4. Test scenario #22

Evidence:

xxx.md line 30
```

---

# Flow 6: Decision Tracking

Ví dụ:

```
Why use Redis Queue?
```

System:

```
Decision Node

       |
       |
Evidence

architecture.md

line 50
```

---

# 7. Mindmap tổng quan

Bạn có thể vẽ như sau:

```
                 Project Knowledge System


                         |
        -----------------------------------
        |                 |               |

   Source Layer     Knowledge Layer   Intelligence Layer


        |                 |               |

 Documents          Concepts          AI Assistant

 Markdown           Relations         Suggestion

 CSV                Evidence          Reasoning


                         |

                 Investigation


                         |

              Impact Analysis
              Dependency
              Decision Search

```

---

# 8. Core Architecture Principle

Tôi nghĩ cần khóa 5 nguyên tắc:

## 1. Document không phải Knowledge

Document chỉ là evidence.

---

## 2. Human defines vocabulary

Con người định nghĩa:

"Project này có những khái niệm gì?"

---

## 3. AI assists, not owns truth

AI đề xuất.

Không tự quyết.

---

## 4. Everything traceable

Mọi node/relation phải có evidence.

---

## 5. Incremental update

Không rebuild toàn bộ.

Chỉ update phần thay đổi.

---

# 9. Thứ tự xây MVP hợp lý

Không làm AI trước.

## Phase 1

Document + Concept

```
Upload md

Create concept

Attach document
```

---

## Phase 2

Graph

```
Concept relation

Graph visualization
```

---

## Phase 3

AI assistant

```
Suggest

Summarize

Query
```

---

## Phase 4

Impact Analysis

```
Change X

Find affected Y
```

---

Kết luận:

Core của hệ thống này không phải là "RAG".
Không phải "AI đọc file".
Không phải "Graph database".

Core là:

> **Một Knowledge Model có kiểm soát, nơi Document là evidence, Concept là node, Relation là tri thức, AI là lớp hỗ trợ xây dựng và khai thác.**

Nếu đi tiếp bước sau, tôi nghĩ nên thiết kế tiếp **Domain Model + Use Case Diagram chuẩn UML** trước khi chọn DB/framework.
