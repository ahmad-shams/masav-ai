# masav-ai Architecture

## Overview

**masav-ai** is an AI-powered payment processing system that integrates with MASAV (מסב — Israeli Automated Bank Services), enabling intelligent direct debit and credit transfer operations with fraud detection, reconciliation, and natural language querying capabilities.

---

## System Architecture Diagram

```mermaid
graph TB
    subgraph Clients["Client Layer"]
        WebApp["Web Application\n(Admin Dashboard)"]
        API_Client["API Clients\n(Internal Services)"]
        CLI["CLI Tool"]
    end

    subgraph Gateway["API Gateway Layer"]
        APIGW["API Gateway\n(Rate Limiting / Auth)"]
        Auth["Auth Service\n(JWT / OAuth2)"]
    end

    subgraph Core["Core Application Layer"]
        PaymentSvc["Payment Service\n(Orchestrator)"]
        AISvc["AI Service\n(Claude / LLM)"]
        FraudSvc["Fraud Detection\n(ML Model)"]
        ReconcileSvc["Reconciliation\nService"]
        NotificationSvc["Notification\nService"]
    end

    subgraph MASAV["MASAV Integration Layer"]
        MasavGateway["MASAV Gateway\n(File Processor)"]
        FileGen["File Generator\n(Y2/Y3 Format)"]
        FileParser["Response Parser\n(ACK / Return Files)"]
        SFTP["SFTP Client\n(Bank Transfer)"]
    end

    subgraph Storage["Storage Layer"]
        PostgreSQL[("PostgreSQL\n(Transactions / Accounts)")]
        Redis[("Redis\n(Cache / Queues)")]
        S3[("S3 / Object Store\n(MASAV Files / Audit Logs)")]
    end

    subgraph External["External Services"]
        BankNetwork["Israeli Bank Network\n(MASAV Clearinghouse)"]
        LLM["Claude API\n(Anthropic)"]
        EmailSMS["Email / SMS\n(Notifications)"]
        Monitoring["Monitoring\n(Datadog / Sentry)"]
    end

    %% Client → Gateway
    WebApp --> APIGW
    API_Client --> APIGW
    CLI --> APIGW

    %% Gateway → Auth
    APIGW --> Auth
    Auth --> PaymentSvc

    %% Core Service interactions
    PaymentSvc --> AISvc
    PaymentSvc --> FraudSvc
    PaymentSvc --> ReconcileSvc
    PaymentSvc --> MasavGateway
    PaymentSvc --> NotificationSvc

    %% AI Service → LLM
    AISvc --> LLM

    %% MASAV Layer
    MasavGateway --> FileGen
    MasavGateway --> FileParser
    FileGen --> SFTP
    FileParser --> SFTP
    SFTP --> BankNetwork

    %% Storage
    PaymentSvc --> PostgreSQL
    PaymentSvc --> Redis
    MasavGateway --> S3
    ReconcileSvc --> PostgreSQL

    %% Notifications
    NotificationSvc --> EmailSMS

    %% Monitoring
    PaymentSvc --> Monitoring
    MasavGateway --> Monitoring
```

---

## Component Descriptions

### Client Layer
| Component | Description |
|-----------|-------------|
| **Web Application** | Admin dashboard for managing payments, viewing transaction history, and AI-assisted queries |
| **API Clients** | Internal microservices consuming the payment API (e.g., billing, subscriptions) |
| **CLI Tool** | Developer/ops tool for triggering batch jobs and manual reconciliation |

### API Gateway Layer
| Component | Description |
|-----------|-------------|
| **API Gateway** | Single entry point; handles rate limiting, request routing, and TLS termination |
| **Auth Service** | Issues and validates JWT tokens; integrates with OAuth2/SSO providers |

### Core Application Layer
| Component | Description |
|-----------|-------------|
| **Payment Service** | Central orchestrator; manages the full lifecycle of a payment instruction |
| **AI Service** | Wraps the Claude API; handles natural language queries, anomaly explanations, and smart categorization |
| **Fraud Detection** | ML-based model scoring transactions before submission to MASAV |
| **Reconciliation Service** | Matches MASAV return files against internal transaction records |
| **Notification Service** | Sends payment status updates to customers and operators |

### MASAV Integration Layer
| Component | Description |
|-----------|-------------|
| **MASAV Gateway** | Core integration; manages the MASAV file lifecycle (create → send → receive → parse) |
| **File Generator** | Produces Y2 (debit) / Y3 (credit) formatted files per MASAV specification |
| **Response Parser** | Parses ACK and return files from the bank clearinghouse |
| **SFTP Client** | Securely transfers files to/from the bank's SFTP server |

### Storage Layer
| Component | Description |
|-----------|-------------|
| **PostgreSQL** | Primary datastore for transactions, accounts, mandates, and audit trails |
| **Redis** | Caching layer and message queue (Bull/BullMQ) for async processing |
| **S3 / Object Store** | Long-term storage for raw MASAV files and compliance audit logs |

### External Services
| Service | Description |
|---------|-------------|
| **Israeli Bank Network** | MASAV clearinghouse operated by the Bank of Israel |
| **Claude API (Anthropic)** | LLM backend for the AI service features |
| **Email / SMS** | Customer-facing notifications on payment status |
| **Monitoring** | Observability stack for metrics, errors, and alerting |

---

## Data Flow — Payment Submission

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant PS as Payment Service
    participant AI as AI / Fraud Service
    participant MG as MASAV Gateway
    participant Bank as Bank Network
    participant DB as PostgreSQL

    Client->>API: POST /payments (batch)
    API->>PS: Authenticated request
    PS->>AI: Score transactions for fraud
    AI-->>PS: Risk scores
    PS->>DB: Persist transactions (PENDING)
    PS->>MG: Submit batch to MASAV
    MG->>MG: Generate Y2/Y3 file
    MG->>Bank: SFTP upload
    Bank-->>MG: ACK file received
    MG-->>PS: Submission confirmed
    PS->>DB: Update status (SUBMITTED)
    PS-->>Client: 202 Accepted

    Note over Bank,MG: Next clearing cycle (T+1)
    Bank->>MG: Return file (settled / rejected)
    MG->>PS: Parse results
    PS->>DB: Update status (SETTLED / RETURNED)
    PS->>Client: Webhook notification
```

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js / TypeScript |
| Framework | NestJS or Express |
| Database | PostgreSQL + Prisma ORM |
| Cache / Queue | Redis + BullMQ |
| File Storage | AWS S3 / MinIO |
| AI | Anthropic Claude API |
| Auth | JWT + OAuth2 |
| MASAV Protocol | Custom SFTP + Y2/Y3 file format |
| Containerization | Docker + Docker Compose |
| Orchestration | Kubernetes (production) |
| CI/CD | GitHub Actions |
| Monitoring | Datadog / Sentry |
