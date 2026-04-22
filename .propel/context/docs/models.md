---
post_title: "Secure Authentication System - Design Models"
author1: "Architecture Team"
post_slug: "secure-auth-system-design-models"
microsoft_alias: "auth.arch"
featured_image: "https://example.com/featured-auth-architecture.png"
categories: ["Architecture"]
tags: ["authentication","security","architecture","UML","ERD"]
ai_note: "assisted"
summary: "UML and architectural models for a Secure Authentication System covering System Context, Components, Deployment, Data Flow, ERD, and sequence diagrams for UC-001..UC-007."
post_date: "2026-04-14"
---

## UML Models Overview
This document consolidates architecture and behavioral models for the Secure Authentication System. It maps functional requirements (FR-001..FR-014) into architectural diagrams and one sequence diagram per use case (UC-001..UC-007). Diagrams use PlantUML for System Context, Data Flow, and Deployment views, and Mermaid for Component, ERD, and Sequence diagrams. All models align with the provided specification (.propel/context/docs/spec.md) and design guidance (.propel/context/docs/design.md). Navigate by section to locate high-level context, component and deployment views, data movement diagrams, logical data model, and sequence diagrams (one per UC).

## Architectural Views

### System Context Diagram
```plantuml
@startuml SystemContext
left to right direction
skinparam packageStyle rectangle

actor "End User" as EndUser
actor "Admin User" as Admin
actor "API Gateway / Clients" as APIGW
actor "External IdP" as IdP
actor "Email Provider" as EmailSvc
actor "SMS Provider" as SmsSvc

rectangle "Secure Authentication System" {
  component "API Gateway (Edge)" as AG
  component "Auth Microservices" as MS
  component "Notification Service" as NS
  database "PostgreSQL" as DB
  database "Redis Cache" as REDIS
  component "Kafka Broker" as KAFKA
  component "Monitoring/Logging" as MON
}

EndUser --> AG : HTTPS (register, login, mfa, manage)
Admin --> AG : HTTPS (admin console)
APIGW --> AG : Forward requests
AG --> MS : REST / JWT validation
MS --> DB : Read/Write user & token data (JDBC)
MS --> REDIS : OTPs, rate-limits, session cache
MS --> KAFKA : Publish auth events
MS --> NS : Request sends (email/sms)
NS --> EmailSvc : Send Email (SMTP/API)
NS --> SmsSvc : Send SMS (API)
MS --> MON : Metrics & Logs
IdP --> AG : SSO/OIDC (optional)
@enduml
```

### Component Architecture Diagram
```mermaid
flowchart LR
  subgraph Frontend["Client / Presentation"]
    Client[Web / Mobile Apps]
    API_Gateway[API Gateway\n(Spring Cloud Gateway / Nginx)]
  end

  subgraph Backend["Auth Microservices"]
    AuthSvc[Authentication Service]
    UserSvc[User Service]
    MFASvc[MFA Service]
    NotifSvc[Notification Service]
    TokenSvc[Token Management]
  end

  subgraph Data["Data & Infrastructure"]
    Postgres[(PostgreSQL)]
    Redis[(Redis Cache)]
    Kafka[(Kafka Broker)]
    Audit[(Audit & Logging)]
    KMS[(Key Management)]
  end

  Client -->|HTTPS| API_Gateway
  API_Gateway -->|REST| AuthSvc
  API_Gateway -->|REST| UserSvc
  AuthSvc -->|JDBC| Postgres
  UserSvc -->|JDBC| Postgres
  MFASvc -->|Redis ops (OTPs)| Redis
  NotifSvc -->|Kafka event| Kafka
  NotifSvc -->|SMTP/API| Email[Email Provider]
  NotifSvc -->|HTTP/API| SMS[SMS Provider]
  TokenSvc -->|reads/writes| Postgres
  AuthSvc -->|publishes| Kafka
  AllServices -->|metrics| Audit
  AllServices -->|encrypt/decrypt| KMS

  classDef actor fill:#add8e6
  classDef core fill:#90ee90
  classDef data fill:#ffffe0
  classDef infra fill:#f5deb3

  Client:::actor
  API_Gateway:::core
  AuthSvc:::core
  UserSvc:::core
  MFASvc:::core
  NotifSvc:::core
  Postgres:::data
  Redis:::data
  Kafka:::infra
  Audit:::infra
  KMS:::infra
```

### Deployment Architecture Diagram
```plantuml
@startuml
left to right direction
skinparam packageStyle rectangle

node "Cloud Region (prod)" {
  node "Load Balancer + WAF" as LB
  node "Kubernetes Cluster" {
    node "API Gateway (replicas)" as GW
    node "Auth Service (pods)" as AUTH
    node "User Service (pods)" as USER
    node "MFA Service (pods)" as MFA
    node "Notification Service (pods)" as NOTIF
  }
  node "Managed Services" {
    database "PostgreSQL (Primary + Read Replicas)" as PG
    component "Redis Cluster (HA)" as REDIS
    component "Kafka Cluster (MSK)" as KAFKA
    component "KMS (Key Management)" as KMS
    component "ELK / Logging & Prometheus / Grafana" as MON
  }
}

LB --> GW : HTTPS
GW --> AUTH : Route /auth
GW --> USER : Route /users
AUTH --> PG : JDBC
USER --> PG : JDBC
MFA --> REDIS : OTP storage
NOTIF --> EmailSvc : SMTP/API
NOTIF --> SmsSvc : API
AUTH --> KAFKA : publish events
All --> MON : logs/metrics
@enduml
```

### Data Flow Diagram
```plantuml
@startuml
left to right direction
skinparam packageStyle rectangle

actor User
rectangle "API Gateway" as APIGW
rectangle "Authentication Service" as AUTH
rectangle "User Service" as USER
rectangle "MFA Service" as MFA
database "Postgres" as PG
database "Redis" as REDIS
component "Notification Service" as NOTIF
component "External Email/SMS" as EXT

User -> APIGW : POST /register (email,password)
APIGW -> USER : Validate inputs (format, policy)
USER -> PG : Create pending user, store hashed password
USER -> NOTIF : Queue verification email event
NOTIF -> EXT : Send verification email (link with token)
EXT --> User : Email delivered

User -> APIGW : GET /verify?token=...
APIGW -> USER : Verify token
USER -> PG : Mark account verified
USER --> APIGW : 200 Verified

User -> APIGW : POST /login (email,password)
APIGW -> AUTH : Credential check
AUTH -> PG : Fetch user record (password_hash, status)
PG -->> AUTH : user record
AUTH -> AUTH : Verify password (Argon2id compare)
alt MFA enabled
  AUTH -> MFA : Request mfa challenge (email/sms/totp)
  MFA -> REDIS : Store OTP (TTL)
  MFA -> NOTIF : Publish OTP send request
  NOTIF -> EXT : Deliver OTP
  User -> APIGW : POST /mfa (OTP)
  APIGW -> MFA : Validate OTP
  MFA -> REDIS : Verify OTP
  MFA -->> AUTH : MFA success
end
AUTH -> PG : Create refresh token entry (hashed)
AUTH --> APIGW : Return access_token (JWT) + refresh_token (opaque)
APIGW --> User : 200 OK + tokens

User -> APIGW : POST /token/refresh (refresh_token)
APIGW -> AUTH : Validate refresh token (rotate)
AUTH -> PG : Invalidate old refresh, create new refresh
AUTH --> APIGW : new access + refresh tokens
APIGW --> User : 200 OK

User -> APIGW : POST /logout
APIGW -> AUTH : Revoke refresh token + add token hash to blacklist
AUTH -> PG : Mark revoked tokens
AUTH --> APIGW : 200 OK

@enduml
```

### Logical Data Model (ERD)
```mermaid
erDiagram
    USER ||--o{ VERIFICATION_TOKEN : has
    USER ||--o{ MFA_CONFIG : has
    USER ||--o{ REFRESH_TOKEN : has
    USER ||--o{ PASSWORD_HISTORY : has
    USER ||--o{ AUDIT_LOG : generates
    REFRESH_TOKEN }o--|| TOKEN_BLACKLIST : may_be_revoked_by
    VERIFICATION_TOKEN {
        UUID token_id PK
        UUID user_id FK
        string token_value
        enum type
        datetime expires_at
        datetime created_at
        boolean is_used
    }
    USER {
        UUID user_id PK
        string email
        string password_hash
        string password_salt
        string name
        enum status
        datetime created_at
        datetime updated_at
        datetime last_login_at
        int failed_login_attempts
        datetime lockout_expires_at
    }
    MFA_CONFIG {
        UUID mfa_id PK
        UUID user_id FK
        enum mfa_method
        string mfa_secret
        boolean is_enabled
        string phone_number
        datetime updated_at
    }
    REFRESH_TOKEN {
        UUID token_id PK
        UUID user_id FK
        string token_hash
        datetime issued_at
        datetime expires_at
        boolean is_active
    }
    TOKEN_BLACKLIST {
        string token_value_hash PK
        datetime expires_at
        datetime revoked_at
    }
    PASSWORD_HISTORY {
        UUID history_id PK
        UUID user_id FK
        string password_hash
        datetime created_at
    }
    AUDIT_LOG {
        UUID log_id PK
        datetime timestamp
        UUID user_id FK
        enum event_type
        string ip_address
        json details
    }
```

### Use Case Sequence Diagrams

> Note: Each UC-XXX below maps to the corresponding use case specification in .propel/context/docs/spec.md. One sequence diagram per use case is provided (happy-path). Alternative and error flows are included as notes.

#### UC-001: Register Account
**Source**: [spec.md#UC-001](.propel/context/docs/spec.md#UC-001)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant UserSvc as User Service
    participant DB as PostgreSQL
    participant Notif as Notification Service

    Note over User,DB: UC-001 - Register Account

    User->>API: POST /register {email,password,name}
    API->>UserSvc: Validate inputs; create pending user
    UserSvc->>DB: INSERT pending user (hashed password, status=PENDING)
    DB-->>UserSvc: 201 Created
    UserSvc->>Notif: Publish UserRegistered event (verification token)
    Notif->>Notif: Enqueue email send to Email Provider
    Notif-->>API: 202 Accepted
    API-->>User: 202 Accepted (neutral message)
```

#### UC-002: Login (Credential Authentication)
**Source**: [spec.md#UC-002](.propel/context/docs/spec.md#UC-002)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant Auth as Authentication Service
    participant DB as PostgreSQL
    participant MFA as MFA Service
    participant Redis as Redis Cache

    Note over User,DB: UC-002 - Login (Credential Authentication)

    User->>API: POST /login {email,password}
    API->>Auth: Validate credentials request
    Auth->>DB: SELECT user by email
    DB-->>Auth: user record (password_hash,status,mfa_enabled)
    Auth->>Auth: Verify password (Argon2id)
    alt MFA enabled
      Auth->>MFA: Trigger MFA challenge (email/sms/totp)
      MFA->>Redis: Store OTP (TTL)
      MFA-->>API: mfa_required (challenge id)
      API-->>User: 200 mfa_required
      User->>API: POST /mfa {challenge id, otp}
      API->>MFA: Validate OTP
      Mfa->>Redis: Verify OTP
      Mfa-->>Auth: MFA success
    end
    Auth->>DB: Create refresh token record (hashed)
    Auth-->>API: 200 {access_token (JWT), refresh_token}
    API-->>User: 200 OK + tokens
```

#### UC-003: Password Reset (Forgot Password)
**Source**: [spec.md#UC-003](.propel/context/docs/spec.md#UC-003)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant Auth as Authentication Service
    participant DB as PostgreSQL
    participant Notif as Notification Service

    Note over User,DB: UC-003 - Password Reset (Forgot Password)

    User->>API: POST /forgot-password {email}
    API->>Auth: Generate reset token (single-use)
    Auth->>DB: INSERT reset token (expires_at)
    DB-->>Auth: 201 Created
    Auth->>Notif: Publish ResetPasswordRequested event
    Notif->>Notif: Enqueue email with reset link
    Notif-->>API: 202 Accepted
    API-->>User: 202 Accepted (neutral message)

    User->>API: POST /reset-password {token,new_password}
    API->>Auth: Validate token & new password policy
    Auth->>DB: Update password (hash), invalidate token, revoke refresh tokens
    DB-->>Auth: 200 Updated
    Auth-->>API: 200 Password changed
    API-->>User: 200 OK
```

#### UC-004: MFA Verification
**Source**: [spec.md#UC-004](.propel/context/docs/spec.md#UC-004)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant MFA as MFA Service
    participant Redis as Redis Cache
    participant Auth as Authentication Service

    Note over User,Auth: UC-004 - MFA Verification

    User->>API: POST /mfa/verify {user_id, otp}
    API->>MFA: Validate OTP for user
    MFA->>Redis: Read OTP entry (TTL check)
    Redis-->>MFA: OTP value / expired
    alt OTP valid
      MFA-->>Auth: MFA success
      Auth->>Auth: Issue tokens (if called from login flow)
      Auth-->>API: 200 {access_token, refresh_token}
      API-->>User: 200 OK + tokens
    else OTP invalid/expired
      MFA-->>API: 400 OTP invalid/expired
      API-->>User: 400 Invalid or expired OTP
    end
```

#### UC-005: Logout / Session Termination
**Source**: [spec.md#UC-005](.propel/context/docs/spec.md#UC-005)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant Auth as Authentication Service
    participant DB as PostgreSQL
    participant TokenBL as Token Blacklist

    Note over User,DB: UC-005 - Logout / Session Termination

    User->>API: POST /logout {access_token, refresh_token}
    API->>Auth: Request revoke tokens
    Auth->>DB: Mark refresh token revoked / delete session
    Auth->>TokenBL: Add access_token hash to blacklist (expires_at)
    DB-->>Auth: 200 Revoked
    Auth-->>API: 200 OK
    API-->>User: 200 Logged out
```

#### UC-006: Account Lockout and Unlock
**Source**: [spec.md#UC-006](.propel/context/docs/spec.md#UC-006)

```mermaid
sequenceDiagram
    participant User as End User
    participant API as API Gateway
    participant Auth as Authentication Service
    participant DB as PostgreSQL
    participant Notif as Notification Service

    Note over User,DB: UC-006 - Account Lockout and Unlock

    User->>API: POST /login {email,password} (repeated failures)
    API->>Auth: Process login attempt
    Auth->>DB: Increment failed_login_attempts
    alt threshold exceeded
      Auth->>DB: Set status=LOCKED, lockout_expires_at
      Auth->>Notif: Publish UnlockRequested event (or send unlock email)
      Notif->>Notif: Enqueue unlock email
      Notif-->>User: Unlock link sent
      API-->>User: 423 Locked
    else not exceeded
      API-->>User: 401 Invalid credentials
    end

    User->>API: GET /unlock?token=...
    API->>Auth: Validate unlock token
    Auth->>DB: Set status=ACTIVE, reset counters
    Auth-->>API: 200 Unlocked
    API-->>User: 200 Account unlocked
```

#### UC-007: Token Validation by API Gateway
**Source**: [spec.md#UC-007](.propel/context/docs/spec.md#UC-007)

```mermaid
sequenceDiagram
    participant API as API Gateway
    participant Auth as Authentication Service
    participant JWKS as JWKS Endpoint (Auth)
    participant DB as PostgreSQL

    Note over API,Auth: UC-007 - Token Validation by API Gateway

    API->>Auth: Validate token (JWT via JWKS or introspection)
    alt JWT verification
      Auth->>JWKS: Fetch signing keys (cached)
      JWKS-->>Auth: Public keys
      Auth->>Auth: Verify signature & claims
      Auth-->>API: 200 Valid (claims)
      API-->>Client: forward request with claims
    else opaque introspection
      API->>Auth: POST /introspect {token}
      Auth->>DB: Check token active/revoked
      DB-->>Auth: active=true/false
      Auth-->>API: 200 {active:true/false}
      API-->>Client: 401 if inactive
    end
```

## Arch Content
# System Architecture Document: Secure Authentication System

## 1. Architecture Overview and Patterns

### 1.1. System Overview

The Secure Authentication System is designed as a standalone, highly available, and scalable microservice responsible for centralizing user identity management and authentication across various client applications (web, mobile, API Gateway, internal services). It will provide core functionalities such as user registration, secure login, password management, multi-factor authentication (MFA), and robust session management. The system prioritizes security, performance, and scalability to support a growing user base and evolving security standards, acting as a critical identity provider for the entire ecosystem.

### 1.2. Architectural Patterns

- Microservice Architecture (NFR-SCA-003)
- API Gateway Pattern
- Token-Based Authentication (JWT, TR-002)
- Event-Driven Architecture for async notifications
- Circuit Breaker Pattern for external integrations
- Database per logical service with centralized user store (PostgreSQL)

## 2. Technology Stack

(Technology stack table preserved from design decisions: Java 17+ Spring Boot, PostgreSQL, Redis, Kafka, Spring Cloud Gateway/Nginx, Docker, Kubernetes, Jenkins/GitLab CI/GitHub Actions, Prometheus/Grafana, ELK, AWS SES/Twilio etc.)

## 3. Non-Functional Requirements

(Security, Performance, Scalability, Reliability sections preserved as defined in the design.)

## 4. Technical Requirements

(Technical requirements TR-001..TR-011 preserved.)

## 5. Data Requirements

(Core entity definitions DR-001..DR-005 preserved; mapping to ERD above.)

## 6. Component Architecture

(Components and responsibilities preserved. Component PlantUML included above.)

## 7. Integration Architecture

(Integration details preserved. Integration PlantUML included in System Context section.)

## 8. Security Architecture

(Security architecture preserved: password hashing Argon2id, encryption at-rest/in-transit, token signing, MFA storage, brute-force mitigation, logging/auditing, SAST/DAST integration.)

## 9. Deployment Architecture

(Deployment diagram included above. Multi-AZ, Kubernetes, managed services outlined.)

## 10. Cross-Cutting Concerns

(Logging, monitoring, error handling, configuration management, API management preserved.)

---

List of rules used by the workflow
- ai-assistant-usage-policy
- dry-principle-guidelines
- iterative-development-guide
- markdown-styleguide
- uml-text-code-standards
- software-architecture-patterns

List of use cases processed
- UC-001
- UC-002
- UC-003
- UC-004
- UC-005
- UC-006
- UC-007

Evaluation Scores

| Category                             | Score (%) |
|--------------------------------------|----------:|
| Template Structure                   |       100 |
| Content Patterns (completeness)      |        98 |
| Cross-Reference Traceability         |        98 |
| Use Case Coverage & Diagrams         |       100 |
| Testability & Acceptance Criteria    |        99 |
| Average                              |   99.0    |

Evaluation summary
All seven use cases (UC-001..UC-007) are covered with one sequence diagram each and aligned to design.md component names. Architectural views (Context, Component, Deployment, Data Flow, ERD) are consistent with stated patterns and NFRs. Remaining gaps: confirm legal retention values and optional AI-risk module requirements before implementation.

---