# RAG System Architecture

## High-Level System Architecture

```mermaid
graph TB
    Users[Users<br/>Desktop/Mobile/Tablet] -->|HTTPS| ALB[Load Balancer<br/>AWS ALB]
    ALB --> NextJS[Next.js App<br/>ECS/EKS 2-10 instances]

    NextJS --> Frontend[Frontend Layer<br/>React Components]
    NextJS --> Backend[Backend Layer<br/>API Routes]
    NextJS --> Workers[Background Jobs<br/>BullMQ Workers]

    Backend --> PG[(PostgreSQL<br/>RDS)]
    Backend --> Chroma[(ChromaDB<br/>Vector DB)]
    Backend --> Redis[(Redis<br/>ElastiCache)]
    Backend --> Drive[Google Drive API]
    Backend --> Claude[Anthropic Claude]

    Workers --> PG
    Workers --> Chroma
    Workers --> Redis
    Workers --> S3[AWS S3<br/>Backups]
```

---

## RAG Pipeline Flow

```mermaid
flowchart LR
    A[User Query] --> B[Preprocessing]
    B --> C[Generate<br/>Embedding]
    C --> D[Vector Search<br/>ChromaDB]
    D --> E[Retrieve<br/>Top-K Chunks]
    E --> F[Rerank<br/>Results]
    F --> G[Assemble<br/>Context]
    G --> H[Build Prompt]
    H --> I[Claude API<br/>Generate Answer]
    I --> J[Post-process<br/>Citations]
    J --> K[Return to User]
```

---

## Search Flow (Detailed)

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Next.js UI
    participant API as /api/search
    participant Cache as Redis
    participant Auth as Auth Service
    participant Embed as Embedder
    participant DB as ChromaDB
    participant LLM as Claude API

    U->>UI: Enter query
    UI->>API: POST /api/search
    API->>Cache: Check cache

    alt Cache Hit
        Cache-->>API: Return result
        API-->>UI: Return answer
    else Cache Miss
        API->>Auth: Validate & get permissions
        Auth-->>API: Accessible doc IDs
        API->>Embed: Generate embedding
        Embed-->>API: Query vector
        API->>DB: Vector search + filter
        DB-->>API: Top chunks
        API->>LLM: Generate answer
        LLM-->>API: Answer + citations
        API->>Cache: Store result
        API-->>UI: Return answer
    end

    UI-->>U: Display with sources
```

---

## Document Ingestion Flow

```mermaid
flowchart TD
    A[Admin Triggers Sync] --> B[BullMQ Job Created]
    B --> C[Worker: Fetch Files from Drive]
    C --> D[Download File Content]
    D --> E[Extract Text<br/>PDF/DOCX/etc]
    E --> F[Chunk Text<br/>512-1024 tokens]
    F --> G[Generate Embeddings<br/>OpenAI/Cohere]
    G --> H{Store Data}
    H --> I[ChromaDB<br/>Vectors]
    H --> J[PostgreSQL<br/>Metadata]
    H --> K[S3<br/>Backup]
```

---

## Component Interaction

```mermaid
graph TB
    subgraph Next.js
        Search[Search Page]
        Admin[Admin Dashboard]
        SearchSvc[Search Service]
        SyncSvc[Sync Service]
    end

    Search --> SearchSvc
    Admin --> SyncSvc

    SearchSvc --> Prisma[Prisma Client]
    SearchSvc --> ChromaClient[ChromaDB Client]
    SearchSvc --> ClaudeClient[Claude SDK]

    SyncSvc --> Prisma
    SyncSvc --> ChromaClient
    SyncSvc --> DriveAPI[Google Drive API]

    Prisma --> PG[(PostgreSQL)]
    ChromaClient --> Chroma[(ChromaDB)]
    ClaudeClient --> Claude[Anthropic API]
```

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | Next.js 14 + React | UI & Server Components |
| **UI** | Tailwind + shadcn/ui | Styling |
| **Backend** | Next.js API Routes | REST API |
| **ORM** | Prisma | Database layer |
| **Database** | PostgreSQL (RDS) | Metadata & users |
| **Vector DB** | ChromaDB | Embeddings |
| **Cache** | Redis (ElastiCache) | Cache + Queue |
| **LLM** | Claude 3.5 Sonnet | Answer generation |
| **Embeddings** | OpenAI/Cohere | Text vectors |
| **Storage** | AWS S3 | Backups |
| **Auth** | NextAuth.js | OAuth 2.0 |
| **Deploy** | AWS ECS/EKS | Containers |

---

## Security Layers

1. **Network**: VPC, Security Groups, HTTPS/TLS 1.3
2. **Application**: NextAuth OAuth, JWT, CSRF protection, Rate limiting
3. **Data**: Encryption at rest/transit, Permission inheritance
4. **API**: Secrets Manager, IAM roles, Input validation
5. **Audit**: CloudWatch logs, Access logs, Query logging

---

## Scalability

| Component | Current | Max | Strategy |
|-----------|---------|-----|----------|
| Next.js | 2-10 containers | 50+ | Horizontal (ECS) |
| PostgreSQL | Single | Multi-AZ | Vertical + Replicas |
| ChromaDB | Single | Clustered | Horizontal |
| Redis | Single node | Cluster | 3+ nodes |
| Workers | 2-5 | 20+ | Horizontal (BullMQ) |

---

## PostgreSQL Database Schema

### Entity Relationship Diagram

```mermaid
erDiagram
    User ||--o{ SearchHistory : creates
    User ||--o{ Bookmark : creates
    User ||--o{ SearchFeedback : provides
    User {
        string id PK
        string email UK
        string name
        string googleId UK
        string image
        timestamp createdAt
        timestamp updatedAt
    }

    Document ||--o{ Bookmark : referenced_by
    Document ||--o{ DocumentChunk : contains
    Document {
        string id PK
        string driveId UK
        string title
        string path
        string fileType
        int sizeBytes
        string author
        timestamp createdAt
        timestamp modifiedAt
        timestamp lastSynced
        int chunkCount
        json permissions
    }

    DocumentChunk {
        string id PK
        string documentId FK
        int chunkIndex
        string chromaId UK
        text content
        int tokenCount
        timestamp createdAt
    }

    SearchHistory ||--o{ SearchFeedback : receives
    SearchHistory {
        string id PK
        string userId FK
        text query
        text answer
        decimal confidence
        int sourceCount
        json sources
        int responseTimeMs
        timestamp createdAt
    }

    Bookmark {
        string id PK
        string userId FK
        string documentId FK
        string chunkId
        text note
        string[] tags
        timestamp createdAt
    }

    SearchFeedback {
        string id PK
        string searchHistoryId FK
        string userId FK
        int rating
        text comment
        string issueType
        timestamp createdAt
    }

    SyncJob {
        string id PK
        string status
        timestamp startedAt
        timestamp completedAt
        int totalDocuments
        int processedDocuments
        int failedDocuments
        json errors
        int retryCount
        json metadata
    }

    Session {
        string id PK
        string sessionToken UK
        string userId FK
        timestamp expires
        timestamp createdAt
        timestamp updatedAt
    }

    Account {
        string id PK
        string userId FK
        string provider
        string providerAccountId
        string type
        string accessToken
        string refreshToken
        int expiresAt
        string scope
    }
```

### Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// User Management (NextAuth.js compatible)
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  googleId      String?   @unique
  image         String?
  emailVerified DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  // Relations
  accounts       Account[]
  sessions       Session[]
  searchHistory  SearchHistory[]
  bookmarks      Bookmark[]
  searchFeedback SearchFeedback[]

  @@index([email])
  @@index([googleId])
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@index([userId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expires])
}

// Document Management
model Document {
  id          String   @id @default(cuid())
  driveId     String   @unique
  title       String
  path        String
  fileType    String
  sizeBytes   Int
  author      String?
  createdAt   DateTime
  modifiedAt  DateTime
  lastSynced  DateTime @default(now())
  chunkCount  Int      @default(0)
  permissions Json     // Store Google Drive permissions as JSON

  // Relations
  chunks    DocumentChunk[]
  bookmarks Bookmark[]

  @@index([driveId])
  @@index([lastSynced])
  @@index([fileType])
  @@index([author])
}

model DocumentChunk {
  id         String   @id @default(cuid())
  documentId String
  chunkIndex Int
  chromaId   String   @unique // ChromaDB vector ID
  content    String   @db.Text
  tokenCount Int
  createdAt  DateTime @default(now())

  document Document @relation(fields: [documentId], references: [id], onDelete: Cascade)

  @@unique([documentId, chunkIndex])
  @@index([chromaId])
  @@index([documentId])
}

// Search & Analytics
model SearchHistory {
  id             String   @id @default(cuid())
  userId         String
  query          String   @db.Text
  answer         String?  @db.Text
  confidence     Decimal? @db.Decimal(3, 2)
  sourceCount    Int      @default(0)
  sources        Json?    // Array of source documents
  responseTimeMs Int?
  createdAt      DateTime @default(now())

  user     User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  feedback SearchFeedback[]

  @@index([userId, createdAt(sort: Desc)])
  @@index([createdAt(sort: Desc)])
}

model Bookmark {
  id         String   @id @default(cuid())
  userId     String
  documentId String
  chunkId    String?
  note       String?  @db.Text
  tags       String[] // PostgreSQL array type
  createdAt  DateTime @default(now())

  user     User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  document Document @relation(fields: [documentId], references: [id], onDelete: Cascade)

  @@unique([userId, documentId, chunkId])
  @@index([userId, createdAt(sort: Desc)])
  @@index([documentId])
}

model SearchFeedback {
  id              String   @id @default(cuid())
  searchHistoryId String
  userId          String
  rating          Int // 1-5 stars
  comment         String?  @db.Text
  issueType       String? // irrelevant, incorrect, incomplete, other
  createdAt       DateTime @default(now())

  searchHistory SearchHistory @relation(fields: [searchHistoryId], references: [id], onDelete: Cascade)
  user          User          @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([searchHistoryId])
  @@index([userId])
  @@index([createdAt(sort: Desc)])
}

// Background Jobs
model SyncJob {
  id                  String    @id @default(cuid())
  status              String // pending, running, completed, failed
  startedAt           DateTime?
  completedAt         DateTime?
  totalDocuments      Int       @default(0)
  processedDocuments  Int       @default(0)
  failedDocuments     Int       @default(0)
  errors              Json? // Array of error objects
  retryCount          Int       @default(0)
  metadata            Json? // Additional job metadata
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt

  @@index([status])
  @@index([createdAt(sort: Desc)])
}
```

### Table Descriptions

| Table | Purpose | Key Features |
|-------|---------|--------------|
| **User** | User accounts | NextAuth.js compatible, Google OAuth |
| **Account** | OAuth provider data | Stores access/refresh tokens |
| **Session** | User sessions | JWT session management |
| **Document** | Document metadata | Maps to Google Drive files |
| **DocumentChunk** | Text chunks | Links to ChromaDB vectors |
| **SearchHistory** | Search logs | Analytics & user history |
| **Bookmark** | User bookmarks | Saved documents/chunks |
| **SearchFeedback** | User ratings | Quality improvement data |
| **SyncJob** | Background jobs | Sync status tracking |

### Indexes Strategy

**High Performance Indexes:**
```sql
-- User lookups
CREATE INDEX idx_user_email ON "User"(email);
CREATE INDEX idx_user_googleId ON "User"("googleId");

-- Document queries
CREATE INDEX idx_document_driveId ON "Document"("driveId");
CREATE INDEX idx_document_lastSynced ON "Document"("lastSynced");
CREATE INDEX idx_document_fileType ON "Document"("fileType");

-- Search history (most recent first)
CREATE INDEX idx_search_history_user_date ON "SearchHistory"("userId", "createdAt" DESC);
CREATE INDEX idx_search_history_date ON "SearchHistory"("createdAt" DESC);

-- Chunk lookups
CREATE INDEX idx_chunk_chromaId ON "DocumentChunk"("chromaId");
CREATE INDEX idx_chunk_document ON "DocumentChunk"("documentId");

-- Bookmarks
CREATE INDEX idx_bookmark_user_date ON "Bookmark"("userId", "createdAt" DESC);

-- Sync jobs
CREATE INDEX idx_sync_job_status ON "SyncJob"(status);
```

### Sample Queries

**Get user's recent searches:**
```sql
SELECT
  sh.id,
  sh.query,
  sh.answer,
  sh.confidence,
  sh.sourceCount,
  sh.createdAt,
  COUNT(sf.id) as feedback_count,
  AVG(sf.rating) as avg_rating
FROM "SearchHistory" sh
LEFT JOIN "SearchFeedback" sf ON sf."searchHistoryId" = sh.id
WHERE sh."userId" = $1
GROUP BY sh.id
ORDER BY sh."createdAt" DESC
LIMIT 20;
```

**Get document with chunks:**
```sql
SELECT
  d.*,
  json_agg(
    json_build_object(
      'id', dc.id,
      'chunkIndex', dc."chunkIndex",
      'chromaId', dc."chromaId",
      'tokenCount', dc."tokenCount"
    ) ORDER BY dc."chunkIndex"
  ) as chunks
FROM "Document" d
LEFT JOIN "DocumentChunk" dc ON dc."documentId" = d.id
WHERE d."driveId" = $1
GROUP BY d.id;
```

**Sync job statistics:**
```sql
SELECT
  status,
  COUNT(*) as job_count,
  AVG("processedDocuments") as avg_processed,
  AVG(EXTRACT(EPOCH FROM ("completedAt" - "startedAt"))) as avg_duration_seconds
FROM "SyncJob"
WHERE "startedAt" IS NOT NULL
GROUP BY status;
```

---

## Folder Structure

```
app/
├── api/
│   ├── search/route.ts
│   ├── documents/route.ts
│   ├── sync/trigger/route.ts
│   └── auth/[...nextauth]/route.ts
├── (dashboard)/
│   ├── search/page.tsx
│   └── history/page.tsx
└── (admin)/
    └── dashboard/page.tsx

lib/
├── prisma.ts
├── chromadb.ts
├── claude.ts
└── queue.ts

services/
├── ingestion/
│   ├── fetcher.ts
│   ├── chunker.ts
│   └── embedder.ts
└── search/
    ├── retriever.ts
    └── answer-generator.ts

workers/
└── sync.worker.ts

prisma/
└── schema.prisma
```
