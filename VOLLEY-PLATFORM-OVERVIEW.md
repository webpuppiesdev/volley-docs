# Volley — Full Application Overview

> What Volley is, how it works end-to-end, and how it turns email campaigns into delivered, tracked, and analyzed communications — for stakeholders who want substance without the infrastructure details.

**See also:** [Setup Guide](./setup-guide.md) (deployment) · [API Reference](./api-reference.md) (endpoints) · [User Manual](./user-manual.md) (usage)

---

## Table of Contents

- [What Is Volley?](#what-is-volley)
- [The Big Picture](#the-big-picture)
- [How It Works — The Simple Version](#how-it-works--the-simple-version)
  - [Campaign Creation Flow](#campaign-creation-flow)
  - [Email Delivery Flow](#email-delivery-flow)
  - [Event Tracking Flow](#event-tracking-flow)
- [The Tech Stack](#the-tech-stack)
- [Architecture Deep Dive](#architecture-deep-dive)
  - [Frontend Architecture](#frontend-architecture)
  - [Backend Architecture — Lambda Functions](#backend-architecture--lambda-functions)
  - [Request Lifecycle](#request-lifecycle)
- [Authentication & Authorization](#authentication--authorization)
  - [Token Architecture](#token-architecture)
  - [Role-Based Access Control](#role-based-access-control)
  - [Auth Flow — Step by Step](#auth-flow--step-by-step)
- [Database Schema (DynamoDB)](#database-schema-dynamodb)
  - [Users Table](#users-table)
  - [Data Table](#data-table)
  - [Access Patterns](#access-patterns)
- [The Email Engine](#the-email-engine)
  - [Campaign State Machine](#campaign-state-machine)
  - [Sending Pipeline](#sending-pipeline)
  - [Template Rendering](#template-rendering)
  - [Event Tracking & Analytics](#event-tracking--analytics)
  - [Suppression & Compliance](#suppression--compliance)
- [Contact Management](#contact-management)
  - [Contact Lifecycle](#contact-lifecycle)
  - [Segmentation Engine](#segmentation-engine)
  - [CSV Import Pipeline](#csv-import-pipeline)
- [Template System](#template-system)
  - [Template Types](#template-types)
  - [Version Control](#version-control)
  - [Variable System](#variable-system)
- [CRM Integrations](#crm-integrations)
- [AI Features (AWS Bedrock)](#ai-features-aws-bedrock)
- [AWS Infrastructure](#aws-infrastructure)
  - [Service Map](#service-map)
  - [Deployment Pipeline](#deployment-pipeline)
  - [Naming Conventions](#naming-conventions)
- [Security Architecture](#security-architecture)
- [Data Flow Examples](#data-flow-examples)
  - [Campaign Execution — End to End](#campaign-execution--end-to-end)
  - [Contact Import — End to End](#contact-import--end-to-end)
  - [One-Click Unsubscribe](#one-click-unsubscribe)
- [Quick Stats](#quick-stats)

---

## What Is Volley?

The Volley (Electronic Direct Mail) Platform is a self-hosted, serverless email marketing system built entirely on AWS. It handles the complete lifecycle of email campaigns — from contact management and template creation through audience segmentation, email delivery, and real-time analytics tracking. The platform is a monorepo containing a React frontend, Node.js Lambda backend, and shared TypeScript types, deployed across 12+ AWS services with zero standing servers. Users create campaigns with visual or code-based email templates, target segments of their contact database, schedule or immediately send campaigns, and track opens, clicks, bounces, and complaints in real time. The system supports multi-user access with role-based permissions, CRM integrations (HubSpot, Zoho), and AI-powered email content generation via AWS Bedrock.

---

## The Big Picture

```mermaid
graph LR
    subgraph "Users"
        ADMIN["Admin<br/><i>Full access,<br/>user management</i>"]
        EDITOR["Editor<br/><i>Create campaigns,<br/>templates, contacts</i>"]
        VIEWER["Viewer<br/><i>Read-only<br/>dashboards</i>"]
    end

    subgraph "Frontend (React + Vite)"
        UI["Dashboard"]
        CAMP["Campaign Builder"]
        TMPL["Template Editor"]
        CONT["Contact Manager"]
        SEG["Segment Builder"]
    end

    subgraph "API Gateway + Lambda"
        AUTH["Auth Lambda"]
        AUTHZ["Authorizer Lambda"]
        API_CAMP["Campaigns Lambda"]
        API_TMPL["Templates Lambda"]
        API_CONT["Contacts Lambda"]
        API_AI["AI Lambda"]
        API_ADMIN["Admin Lambda"]
    end

    subgraph "Email Pipeline"
        SQS["SQS Queue"]
        SENDER["Sender Lambda"]
        SES["Amazon SES"]
        SNS["SNS Events"]
        WEBHOOK["Webhook Lambda"]
    end

    subgraph "Storage"
        DYNAMO["DynamoDB<br/><i>2 tables</i>"]
        S3_TMPL["S3<br/><i>Templates</i>"]
        S3_FRONT["S3 + CloudFront<br/><i>Frontend</i>"]
    end

    ADMIN & EDITOR & VIEWER --> UI & CAMP & TMPL & CONT & SEG
    UI & CAMP & TMPL & CONT & SEG --> AUTHZ
    AUTHZ --> AUTH & API_CAMP & API_TMPL & API_CONT & API_AI & API_ADMIN
    API_CAMP --> SQS
    SQS --> SENDER
    SENDER --> SES
    SES --> SNS --> WEBHOOK
    AUTH & API_CAMP & API_TMPL & API_CONT & API_ADMIN --> DYNAMO
    API_TMPL & SENDER --> S3_TMPL
    WEBHOOK --> DYNAMO

    style ADMIN fill:#E74C3C,color:#fff
    style EDITOR fill:#F5A623,color:#fff
    style VIEWER fill:#4A90D9,color:#fff
    style SQS fill:#FF9900,color:#fff
    style SENDER fill:#FF9900,color:#fff
    style SES fill:#FF9900,color:#fff
    style SNS fill:#FF9900,color:#fff
    style WEBHOOK fill:#FF9900,color:#fff
    style DYNAMO fill:#4053D6,color:#fff
    style S3_TMPL fill:#3F8624,color:#fff
    style S3_FRONT fill:#3F8624,color:#fff
    style AUTHZ fill:#7B68EE,color:#fff
```

**Everything is serverless.** There are no EC2 instances, no containers, no standing servers. Every compute action runs as a Lambda function invoked by API Gateway (for API requests), SQS (for email sending), or SNS (for delivery event processing). The frontend is a static React SPA served from S3 via CloudFront.

---

## How It Works — The Simple Version

### Campaign Creation Flow

```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant Frontend as React Frontend
    participant API as API Gateway
    participant Lambda as Campaigns Lambda
    participant DB as DynamoDB

    User->>Frontend: Create new campaign
    Frontend->>API: POST /campaigns
    API->>Lambda: Route request
    Lambda->>DB: Store CAMPAIGN#{id}#META (status: draft)
    DB-->>Lambda: Success
    Lambda-->>Frontend: Campaign created

    User->>Frontend: Select template + segment
    Frontend->>API: PUT /campaigns/{id}
    API->>Lambda: Update campaign
    Lambda->>DB: Update templateId, segmentId
    DB-->>Lambda: Success
    Lambda-->>Frontend: Campaign updated

    User->>Frontend: Click "Send Now" or "Schedule"
    Frontend->>API: POST /campaigns/{id}/execute (or /schedule)
    API->>Lambda: Trigger execution
    Note over Lambda: Evaluates segment → queues emails to SQS
```

A campaign starts as a **draft** — just a name, subject line, template, and target segment. The user builds it iteratively: pick a template, choose a segment, preview the audience, then execute or schedule. No email leaves the system until the user explicitly triggers it.

### Email Delivery Flow

```mermaid
sequenceDiagram
    participant Lambda as Campaigns Lambda
    participant SQS as SQS Queue
    participant Sender as Sender Lambda
    participant S3 as S3 (Templates)
    participant SES as Amazon SES
    participant DB as DynamoDB

    Lambda->>Lambda: Evaluate segment rules against contacts
    Lambda->>SQS: Queue email jobs (1 per contact)
    SQS->>Sender: Consume batch of messages
    Sender->>DB: Check suppression list
    Sender->>S3: Load HTML template
    Sender->>Sender: Render with Handlebars (contact variables)
    Sender->>SES: Send email (with tracking headers)
    SES-->>Sender: MessageId
    Sender->>DB: Write EMAIL#{contactId} record (idempotent)
    Sender->>DB: Increment sentCount (atomic ADD)
    Sender->>Sender: Check if campaign complete
    Note over Sender: sentCount + failedCount >= totalEmails?<br/>→ Transition to completed/failed
```

Each contact in the segment becomes a separate SQS message. The Sender Lambda processes these in batches, rendering personalized HTML for each recipient. Sends are idempotent — the same email job processed twice will not send twice.

### Event Tracking Flow

```mermaid
sequenceDiagram
    participant SES as Amazon SES
    participant SNS as SNS Topic
    participant Webhook as Webhook Lambda
    participant DB as DynamoDB
    participant Frontend as Dashboard

    SES->>SNS: Delivery/Open/Click/Bounce/Complaint event
    SNS->>Webhook: Forward event notification
    Webhook->>Webhook: Parse event type + extract campaignId
    Webhook->>DB: Increment analytics counter (atomic ADD)
    Note over Webhook: Bounce/Complaint → Add to suppression list
    Frontend->>DB: Poll campaign analytics
    DB-->>Frontend: Real-time counts
```

SES publishes delivery events to an SNS topic. The Webhook Lambda parses each event and atomically increments the relevant counter on the campaign record. Bounces and complaints automatically add the recipient to the suppression list, preventing future sends.

---

## The Tech Stack

```mermaid
graph TD
    subgraph "Frontend"
        REACT["React 19"]
        VITE["Vite 8"]
        TANSTACK["TanStack Router +<br/>React Query + Table"]
        ZUSTAND["Zustand<br/><i>Auth state</i>"]
        TAILWIND["Tailwind CSS 4 +<br/>shadcn/ui"]
        MONACO["Monaco Editor<br/><i>Code templates</i>"]
        EMAILEDIT["React Email Editor<br/><i>Visual builder</i>"]
        RECHARTS["Recharts<br/><i>Analytics charts</i>"]
    end

    subgraph "Backend"
        NODE["Node.js 22"]
        ESBUILD["esbuild<br/><i>Bundler</i>"]
        JWT["jsonwebtoken<br/><i>Auth tokens</i>"]
        BCRYPT["bcryptjs<br/><i>Password hashing</i>"]
        HBS["Handlebars<br/><i>Template rendering</i>"]
        MJML["MJML<br/><i>Responsive email markup</i>"]
        ZOD["Zod<br/><i>Schema validation</i>"]
    end

    subgraph "AWS Services"
        LAMBDA["Lambda<br/><i>12 functions</i>"]
        APIGW["API Gateway<br/><i>REST API</i>"]
        DYNDB["DynamoDB<br/><i>2 tables</i>"]
        SQS_S["SQS<br/><i>Email queue</i>"]
        SES_S["SES v2<br/><i>Email delivery</i>"]
        S3_S["S3<br/><i>Templates + frontend</i>"]
        CF["CloudFront<br/><i>CDN</i>"]
        EB["EventBridge Scheduler<br/><i>Scheduled campaigns</i>"]
        SNS_S["SNS<br/><i>Delivery events</i>"]
        BEDROCK["Bedrock<br/><i>AI content generation</i>"]
    end

    subgraph "Shared"
        TYPES["TypeScript interfaces"]
        VALID["Zod schemas"]
        CONST["Role constants"]
    end

    style REACT fill:#61DAFB,color:#000
    style VITE fill:#646CFF,color:#fff
    style NODE fill:#339933,color:#fff
    style LAMBDA fill:#FF9900,color:#fff
    style DYNDB fill:#4053D6,color:#fff
    style SES_S fill:#DD344C,color:#fff
    style BEDROCK fill:#00A4EF,color:#fff
```

### Monorepo Structure

```
edm-aws/
├── packages/
│   ├── frontend/          React 19 + Vite SPA
│   │   ├── src/
│   │   │   ├── routes/    File-based routing (TanStack Router)
│   │   │   ├── components/ UI components (shadcn/ui)
│   │   │   ├── hooks/     React Query data hooks
│   │   │   ├── stores/    Zustand state (auth, theme)
│   │   │   └── lib/       API client, utilities
│   │   └── vite.config.ts
│   │
│   ├── backend/           Lambda functions
│   │   ├── src/
│   │   │   ├── functions/  10 API + 1 sender + 1 webhook
│   │   │   ├── lib/       Shared utilities (db, jwt, middleware)
│   │   │   └── types/     TypeScript interfaces
│   │   └── scripts/       Build configuration (esbuild)
│   │
│   └── shared/            Cross-package types + validation
│       ├── types/
│       ├── validation/
│       └── constants/
│
├── scripts/               AWS deployment scripts (18+ bash)
│   ├── deploy.sh          Master deployment orchestrator
│   ├── setup-*.sh         Per-service provisioning
│   ├── seed-*.sh          Initial data seeding
│   └── teardown.sh        Full resource cleanup
│
└── docs/                  Documentation
```

---

## Architecture Deep Dive

### Frontend Architecture

```mermaid
graph TD
    subgraph "Routing Layer (TanStack Router — File-Based)"
        ROOT["__root.tsx<br/><i>Query + Theme providers</i>"]
        PUBLIC["Public Routes"]
        PROTECTED["_authenticated.tsx<br/><i>Auth guard layout</i>"]
    end

    subgraph "Public Pages"
        LOGIN["login.tsx"]
        SIGNUP["signup.tsx"]
        FORGOT["forgot-password.tsx"]
        RESET["reset-password.tsx"]
        VERIFY["verify-email.tsx"]
        INVITE["accept-invite.tsx"]
    end

    subgraph "Protected Pages"
        DASH["dashboard.tsx<br/><i>Analytics overview</i>"]
        ANALYTICS["analytics.tsx<br/><i>Global stats</i>"]
        CAMPAIGNS["campaigns/<br/><i>index · new · $campaignId</i>"]
        CONTACTS["contacts/<br/><i>index · import · $contactId<br/>segments · segments/$segmentId</i>"]
        TEMPLATES["templates/<br/><i>index · new · $templateId/<br/>builder · edit · preview · history</i>"]
        SETTINGS["settings/<br/><i>index · users · domains · integrations</i>"]
        SUPPRESS["suppression.tsx"]
    end

    ROOT --> PUBLIC --> LOGIN & SIGNUP & FORGOT & RESET & VERIFY & INVITE
    ROOT --> PROTECTED --> DASH & ANALYTICS & CAMPAIGNS & CONTACTS & TEMPLATES & SETTINGS & SUPPRESS

    style ROOT fill:#7B68EE,color:#fff
    style PROTECTED fill:#E74C3C,color:#fff
    style DASH fill:#50C878,color:#fff
```

**State management is intentionally layered:**

| Layer | Tool | What It Manages |
|-------|------|----------------|
| **Server state** | React Query (5min staleTime, 1 retry) | All API data — campaigns, contacts, templates, analytics |
| **Auth state** | Zustand (localStorage persistence) | User profile, access token, `isAuthenticated` flag |
| **Theme state** | next-themes | Light/dark mode preference |
| **Component state** | React hooks | Forms, modals, local UI state |

**The API client** is a single generic `apiClient<T>()` function that handles JSON serialization, Bearer token injection, and automatic 401 → refresh → retry with a mutex to prevent duplicate refresh calls across concurrent requests.

### Backend Architecture — Lambda Functions

The backend is 12 Lambda functions. 10 serve API endpoints via API Gateway. 1 processes email sends from SQS. 1 processes delivery events from SNS.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  API GATEWAY (REST)                                                      │
│  ┌──────────────┐                                                        │
│  │  Authorizer   │ ← Validates JWT on every request                      │
│  │  Lambda       │   Returns: userId, email, role as context             │
│  └──────┬───────┘                                                        │
│         │                                                                │
│  ┌──────▼───────────────────────────────────────────────────────────┐    │
│  │                        API Routes                                │    │
│  │                                                                  │    │
│  │  /auth/*          → Auth Lambda          (login, register, etc.) │    │
│  │  /admin/*         → Admin Lambda         (user management)       │    │
│  │  /contacts/*      → Contacts Lambda      (CRUD + import/export)  │    │
│  │  /campaigns/*     → Campaigns Lambda     (CRUD + execute)        │    │
│  │  /templates/*     → Templates Lambda     (CRUD + render)         │    │
│  │  /domain/*        → Domain Lambda        (SES domain status)     │    │
│  │  /integrations/*  → Integrations Lambda  (HubSpot, Zoho)        │    │
│  │  /ai/*            → AI Lambda            (generate, improve)     │    │
│  │                                                                  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  EVENT-DRIVEN FUNCTIONS                                                  │
│                                                                          │
│  SQS (email-queue)  → Sender Lambda    (render + send via SES)          │
│  SNS (ses-events)   → Webhook Lambda   (track delivery/open/click)      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Full API Route Map

| Function | Method | Route | Auth | Description |
|----------|--------|-------|------|-------------|
| **auth** | POST | `/auth/login` | None | Username/password login |
| | POST | `/auth/register` | None | New user registration |
| | POST | `/auth/refresh` | Cookie | Refresh access token |
| | POST | `/auth/logout` | Cookie | Clear session |
| | POST | `/auth/verify-email` | None | Verify email token |
| | POST | `/auth/forgot-password` | None | Request password reset |
| | POST | `/auth/reset-password` | None | Reset with token |
| | POST | `/auth/accept-invite` | None | Accept admin invite |
| **admin** | GET | `/admin/users` | Admin | List all users |
| | POST | `/admin/users` | Admin | Create user |
| | POST | `/admin/users/invite` | Admin | Send invite email |
| | PUT | `/admin/users/{userId}` | Admin | Update user |
| | POST | `/admin/users/{userId}/disable` | Admin | Disable account |
| **contacts** | GET | `/contacts` | Any | List with pagination |
| | POST | `/contacts` | Editor+ | Create contact |
| | GET | `/contacts/{contactId}` | Any | Get contact details |
| | PUT | `/contacts/{contactId}` | Editor+ | Update contact |
| | DELETE | `/contacts/{contactId}` | Editor+ | Delete contact |
| | POST | `/contacts/import` | Editor+ | Bulk CSV import |
| | GET | `/contacts/export` | Any | CSV export |
| | GET | `/contacts/segments` | Any | List segments |
| | POST | `/contacts/segments` | Editor+ | Create segment |
| | GET/PUT/DELETE | `/contacts/segments/{segmentId}` | Editor+ | Segment CRUD |
| **campaigns** | GET | `/campaigns` | Any | List campaigns |
| | POST | `/campaigns` | Editor+ | Create campaign |
| | GET/PUT/DELETE | `/campaigns/{campaignId}` | Editor+ | Campaign CRUD |
| | GET | `/campaigns/{campaignId}/analytics` | Any | Per-campaign stats |
| | GET | `/campaigns/analytics` | Any | Global analytics |
| | GET | `/campaigns/{campaignId}/audience` | Any | Preview segment members |
| | POST | `/campaigns/{campaignId}/schedule` | Editor+ | Schedule send |
| | POST | `/campaigns/{campaignId}/cancel` | Editor+ | Cancel scheduled send |
| | POST | `/campaigns/{campaignId}/execute` | Editor+ | Send immediately |
| | POST/GET | `/campaigns/unsubscribe` | None | One-click unsubscribe |
| | GET/POST/DELETE | `/campaigns/suppressions` | Editor+ | Suppression list |
| **templates** | GET | `/templates` | Any | List templates |
| | POST | `/templates` | Editor+ | Create template |
| | GET/PUT/DELETE | `/templates/{templateId}` | Editor+ | Template CRUD |
| | POST | `/templates/{templateId}/render` | Any | Render with sample data |
| | GET | `/templates/{templateId}/versions` | Any | Version history |
| | POST | `/templates/{templateId}/revert` | Editor+ | Revert to version |
| | POST | `/templates/{templateId}/duplicate` | Editor+ | Clone template |
| **domain** | GET | `/domain/status` | Admin | SES domain verification status |
| | GET | `/domain/reputation` | Admin | Sender reputation metrics |
| | POST | `/domain/verify` | Admin | Trigger domain verification |
| **integrations** | GET | `/integrations` | Admin | List integrations |
| | GET/POST/DELETE | `/integrations/hubspot` | Admin | HubSpot connection |
| | GET/POST/DELETE | `/integrations/zoho` | Admin | Zoho connection |
| | POST | `/integrations/hubspot/sync` | Admin | Sync HubSpot contacts |
| | POST | `/integrations/zoho/sync` | Admin | Sync Zoho contacts |
| **ai** | POST | `/ai/generate` | Editor+ | Generate email content |
| | POST | `/ai/improve` | Editor+ | Suggest improvements |
| | POST | `/ai/agent` | Editor+ | Multi-turn AI chat |

### Request Lifecycle

Every API request follows the same path:

```
┌───────────────────────────────────────────────────────────────────────┐
│  1. BROWSER → API GATEWAY                                             │
│     Headers: Authorization: Bearer {accessToken}                      │
│     Body: JSON payload                                                │
│                                                                       │
│  2. API GATEWAY → AUTHORIZER LAMBDA                                   │
│     Extracts JWT from header                                          │
│     Verifies signature + expiration (HS256)                           │
│     Returns IAM policy + context: { userId, email, role }             │
│                                                                       │
│  3. API GATEWAY → TARGET LAMBDA                                       │
│     event.requestContext.authorizer = { userId, email, role }         │
│     event.pathParameters = { campaignId, contactId, etc. }            │
│     event.body = JSON string                                          │
│                                                                       │
│  4. LAMBDA HANDLER                                                    │
│     ├── Check OPTIONS → Return CORS preflight                        │
│     ├── Parse path + HTTP method                                     │
│     ├── Check role authorization (if write operation)                 │
│     ├── Route to business logic function                             │
│     └── Return { statusCode, headers (CORS), body: JSON }            │
│                                                                       │
│  5. RESPONSE → BROWSER                                                │
│     Success: { success: true, data: {...} }                           │
│     Error:   { success: false, error: "message" }                     │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Authentication & Authorization

### Token Architecture

```mermaid
graph LR
    subgraph "Access Token (JWT)"
        AT_TYPE["Type: JWT (HS256)"]
        AT_TTL["TTL: 15 minutes"]
        AT_STORE["Storage: Memory only"]
        AT_PAY["Payload: sub, email,<br/>role, iat, exp"]
    end

    subgraph "Refresh Token"
        RT_TYPE["Type: UUID"]
        RT_TTL["TTL: 7 days"]
        RT_STORE["Storage: HttpOnly cookie<br/>(Secure, SameSite=None)"]
        RT_DB["DynamoDB: REFRESH#{token}"]
    end

    subgraph "Authorizer"
        VERIFY["Verify JWT signature"]
        CHECK["Check expiration"]
        POLICY["Return IAM policy<br/>+ user context"]
    end

    AT_TYPE --> VERIFY
    VERIFY --> CHECK --> POLICY

    style AT_TYPE fill:#50C878,color:#fff
    style RT_TYPE fill:#4A90D9,color:#fff
    style VERIFY fill:#7B68EE,color:#fff
```

### Role-Based Access Control

```
┌─────────────────────────────────────────────────────────────────┐
│  ROLE HIERARCHY                                                  │
│                                                                  │
│  admin ──────────────────────────────────────────────────────── │
│  │  Full platform access                                        │
│  │  User management (create, invite, disable)                   │
│  │  Domain configuration (SES verification)                     │
│  │  CRM integrations (HubSpot, Zoho)                           │
│  │  All editor + viewer permissions                             │
│  │                                                              │
│  editor ─────────────────────────────────────────────────────── │
│  │  Create/edit campaigns, templates, contacts                  │
│  │  Execute and schedule campaigns                              │
│  │  Import/export contacts                                      │
│  │  Manage segments and suppression lists                       │
│  │  Use AI features                                             │
│  │  All viewer permissions                                      │
│  │                                                              │
│  viewer ─────────────────────────────────────────────────────── │
│     Read-only access to all data                                │
│     View dashboards and analytics                               │
│     Preview templates                                           │
│     Export contacts                                              │
└─────────────────────────────────────────────────────────────────┘
```

### Auth Flow — Step by Step

```mermaid
sequenceDiagram
    participant Browser as Browser
    participant Store as Zustand Auth Store
    participant API as API Client
    participant GW as API Gateway
    participant Auth as Auth Lambda
    participant DB as DynamoDB

    Note over Browser,DB: LOGIN FLOW
    Browser->>API: POST /auth/login {email, password}
    API->>GW: Forward (no auth header)
    GW->>Auth: Route to auth handler
    Auth->>DB: Lookup USER by email (GSI1)
    DB-->>Auth: User record
    Auth->>Auth: bcrypt.compare(password, hash)
    Auth->>Auth: Generate JWT (15min) + UUID refresh token
    Auth->>DB: Store REFRESH#{token}#META (TTL: 7 days)
    Auth-->>Browser: {accessToken, user} + Set-Cookie: refreshToken

    Browser->>Store: Save user + token in memory

    Note over Browser,DB: AUTHENTICATED REQUEST
    Browser->>API: GET /campaigns (Bearer token)
    API->>GW: Authorization: Bearer {jwt}
    GW->>GW: Authorizer Lambda validates JWT
    GW->>Auth: Forward with user context
    Auth-->>Browser: Campaign data

    Note over Browser,DB: TOKEN REFRESH (on 401)
    Browser->>API: Any request → 401 Unauthorized
    API->>API: Mutex lock (prevent duplicate refreshes)
    API->>GW: POST /auth/refresh (cookie sent automatically)
    GW->>Auth: Extract refreshToken from cookie
    Auth->>DB: Lookup REFRESH#{token} → validate
    Auth->>Auth: Generate new JWT
    Auth-->>API: {accessToken}
    API->>Store: Update token
    API->>GW: Retry original request with new token
    GW-->>Browser: Success

    Note over Browser,DB: LOGOUT
    Browser->>API: POST /auth/logout
    API->>Auth: Clear refresh token
    Auth->>DB: Delete REFRESH#{token}
    Browser->>Store: Clear auth state
    Browser->>Browser: Broadcast logout to other tabs
```

---

## Database Schema (DynamoDB)

The platform uses two DynamoDB tables with single-table design patterns. All records use composite keys (PK + SK) for efficient access.

### Users Table

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE: {TENANT_PREFIX}-users                                            │
│  Billing: On-Demand (pay-per-request)                                    │
│  GSI1: email (for login lookup)                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  RECORD TYPE           PK                        SK                      │
│  ─────────────────────────────────────────────────────────────────────── │
│  User Profile          USER#{userId}             PROFILE                 │
│  Refresh Token Lookup  REFRESH#{tokenValue}      META                   │
│  User Session          USER#{userId}             SESSION#{sessionId}     │
│  Email Verify Token    VERIFY#{token}            META                   │
│  Password Reset Token  RESET#{token}             META                   │
│                                                                          │
│  USER ATTRIBUTES                                                         │
│  ─────────────────────────────────────────────────────────────────────── │
│  userId         String     UUID                                          │
│  email          String     Unique (GSI1)                                 │
│  passwordHash   String     bcrypt (10 rounds)                            │
│  name           String     Display name                                  │
│  role           String     admin | editor | viewer                       │
│  status         String     active | disabled | pending                   │
│  emailVerified  Boolean    Email verification status                     │
│  lastLoginAt    String     ISO timestamp                                 │
│  createdAt      String     ISO timestamp                                 │
│  updatedAt      String     ISO timestamp                                 │
│                                                                          │
│  TOKEN ATTRIBUTES                                                        │
│  ─────────────────────────────────────────────────────────────────────── │
│  refreshToken   String     UUID value                                    │
│  expiresAt      Number     TTL (epoch seconds) — auto-deleted by DDB     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Table

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE: {TENANT_PREFIX}-data                                             │
│  Billing: On-Demand (pay-per-request)                                    │
│  GSI1: GSI1PK + GSI1SK (for alternative access patterns)                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  RECORD TYPE           PK                        SK                      │
│  ─────────────────────────────────────────────────────────────────────── │
│  Contact               CONTACT#{contactId}       PROFILE                 │
│  Segment               SEGMENT#{segmentId}       META                   │
│  Template              TEMPLATE#{templateId}     META                   │
│  Template Version      TEMPLATE#{templateId}     VERSION#{versionNum}   │
│  Campaign              CAMPAIGN#{campaignId}     META                   │
│  Email Send Record     CAMPAIGN#{campaignId}     EMAIL#{contactId}      │
│  Campaign Audit Log    CAMPAIGN#{campaignId}     AUDIT#{ts}#{uuid}      │
│  Suppression Entry     SUPPRESSION#{email}       META                   │
│                                                                          │
│  CONTACT ATTRIBUTES                                                      │
│  ─────────────────────────────────────────────────────────────────────── │
│  contactId       String     UUID                                         │
│  email           String     Recipient email                              │
│  firstName       String     First name                                   │
│  lastName        String     Last name                                    │
│  tags            List       String tags for segmentation                 │
│  customFields    Map        Arbitrary key-value metadata                 │
│  status          String     active | unsubscribed | bounced              │
│  syncSource      String     local | hubspot | zoho                       │
│  externalIds     Map        { hubspotId, zohoId }                        │
│                                                                          │
│  CAMPAIGN ATTRIBUTES                                                     │
│  ─────────────────────────────────────────────────────────────────────── │
│  campaignId      String     UUID                                         │
│  name            String     Campaign name                                │
│  subject         String     Email subject line                           │
│  templateId      String     Linked template                              │
│  segmentId       String     Target segment                               │
│  status          String     draft|scheduled|executing|completed|failed   │
│  totalEmails     Number     Total recipients                             │
│  sentCount       Number     Successfully sent                            │
│  failedCount     Number     Failed to send                               │
│  deliveredCount  Number     Confirmed delivered                          │
│  openedCount     Number     Unique opens                                 │
│  clickedCount    Number     Unique clicks                                │
│  bouncedCount    Number     Hard/soft bounces                            │
│  complainedCount Number     Spam complaints                              │
│                                                                          │
│  TEMPLATE ATTRIBUTES                                                     │
│  ─────────────────────────────────────────────────────────────────────── │
│  templateId      String     UUID                                         │
│  name            String     Template name                                │
│  type            String     code | visual                                │
│  currentVersion  Number     Latest version number                        │
│  s3Key           String     Rendered HTML in S3                          │
│  designS3Key     String     Visual builder JSON in S3                    │
│  mjmlS3Key       String     MJML source in S3                            │
│  variables       List       Handlebars variable names                    │
│  isStarter       Boolean    Pre-built starter template flag              │
│                                                                          │
│  SEGMENT ATTRIBUTES                                                      │
│  ─────────────────────────────────────────────────────────────────────── │
│  segmentId       String     UUID                                         │
│  name            String     Segment name                                 │
│  description     String     Optional description                         │
│  rules           List       Array of {field, operator, value} rules      │
│  logic           String     AND | OR (combine rules)                     │
│  memberCount     Number     Cached member count                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Access Patterns

| Pattern | Table | Key Condition | Use Case |
|---------|-------|---------------|----------|
| Get user by ID | users | PK=`USER#{id}`, SK=`PROFILE` | Profile lookup |
| Get user by email | users | GSI1 on `email` | Login |
| Validate refresh token | users | PK=`REFRESH#{token}`, SK=`META` | Token refresh |
| List all contacts | data | PK begins_with `CONTACT#` | Contact list |
| Get campaign analytics | data | PK=`CAMPAIGN#{id}`, SK=`META` | Dashboard |
| List email send records | data | PK=`CAMPAIGN#{id}`, SK begins_with `EMAIL#` | Delivery tracking |
| Check suppression | data | PK=`SUPPRESSION#{email}`, SK=`META` | Pre-send check |
| Get template versions | data | PK=`TEMPLATE#{id}`, SK begins_with `VERSION#` | Version history |

---

## The Email Engine

### Campaign State Machine

```mermaid
stateDiagram-v2
    [*] --> draft: Campaign created

    draft --> scheduled: Schedule (EventBridge cron)
    draft --> executing: Execute now
    draft --> [*]: Delete

    scheduled --> executing: EventBridge triggers at scheduled time
    scheduled --> draft: Cancel schedule

    executing --> completed: All emails sent successfully
    executing --> failed: Some/all emails failed

    completed --> [*]
    failed --> [*]
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  STATE TRANSITIONS — WHAT TRIGGERS EACH                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  draft → scheduled                                                   │
│    Trigger: POST /campaigns/{id}/schedule                            │
│    Action:  Create EventBridge one-time schedule                     │
│    Target:  Campaigns Lambda with { campaignId, action: "execute" }  │
│                                                                      │
│  draft → executing                                                   │
│    Trigger: POST /campaigns/{id}/execute                             │
│    Action:  Evaluate segment, queue emails to SQS                    │
│                                                                      │
│  scheduled → draft                                                   │
│    Trigger: POST /campaigns/{id}/cancel                              │
│    Action:  Delete EventBridge schedule, reset status                │
│                                                                      │
│  scheduled → executing                                               │
│    Trigger: EventBridge fires at scheduled time                      │
│    Action:  Invokes Campaigns Lambda → evaluate + queue              │
│                                                                      │
│  executing → completed                                               │
│    Trigger: Sender Lambda detects sentCount + failedCount >= total   │
│    Action:  Conditional update to "completed"                        │
│                                                                      │
│  executing → failed                                                  │
│    Trigger: Same check, but failedCount > 0                          │
│    Action:  Conditional update to "failed"                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Sending Pipeline

```mermaid
graph TD
    EXEC["Campaign Execute"]
    EVAL["Evaluate Segment Rules"]
    FILTER["Filter: active contacts only<br/>+ not in suppression list"]
    QUEUE["Queue to SQS<br/>(1 message per contact)"]
    SENDER["Sender Lambda<br/>(SQS consumer)"]
    SUPP_CHECK["Check suppression list"]
    LOAD["Load template from S3"]
    RENDER["Render with Handlebars<br/>(contact variables)"]
    MIME["Build MIME message<br/>+ List-Unsubscribe headers"]
    SES["Send via SES v2"]
    RECORD["Write idempotent<br/>EMAIL#{contactId} record"]
    COUNT["Atomic increment<br/>sentCount or failedCount"]
    COMPLETE{"sentCount + failedCount<br/>>= totalEmails?"}
    DONE["Transition campaign<br/>to completed/failed"]

    EXEC --> EVAL --> FILTER --> QUEUE
    QUEUE --> SENDER --> SUPP_CHECK
    SUPP_CHECK -->|"Suppressed"| COUNT
    SUPP_CHECK -->|"Not suppressed"| LOAD --> RENDER --> MIME --> SES
    SES --> RECORD --> COUNT
    COUNT --> COMPLETE
    COMPLETE -->|"Yes"| DONE
    COMPLETE -->|"No"| SENDER

    style EXEC fill:#4A90D9,color:#fff
    style SES fill:#DD344C,color:#fff
    style DONE fill:#50C878,color:#fff
    style SUPP_CHECK fill:#F5A623,color:#fff
```

### SQS Email Job Structure

Each contact in the segment becomes one SQS message:

```json
{
  "campaignId": "uuid",
  "contactId": "uuid",
  "email": "recipient@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "subject": "Your exclusive offer inside",
  "templateS3Key": "templates/tpl-123/v3.html",
  "customFields": {
    "company": "Acme Corp",
    "plan": "enterprise"
  }
}
```

### Template Rendering

```
┌─────────────────────────────────────────────────────────────────┐
│  RENDERING PIPELINE                                              │
│                                                                  │
│  1. Load HTML from S3 (templateS3Key)                           │
│     └── Pre-compiled from MJML or visual builder JSON           │
│                                                                  │
│  2. Handlebars compilation                                       │
│     └── Template variables: {{firstName}}, {{lastName}},         │
│         {{email}}, {{company}}, any customField key              │
│                                                                  │
│  3. Variable injection                                           │
│     └── Each contact's data fills their template copy            │
│                                                                  │
│  4. Unsubscribe link injection                                   │
│     └── {{unsubscribeUrl}} = /campaigns/unsubscribe              │
│         ?cid={campaignId}&email={email}&token={HMAC}             │
│                                                                  │
│  5. MIME message construction                                    │
│     └── Headers:                                                 │
│         List-Unsubscribe: <{url}>                               │
│         List-Unsubscribe-Post: List-Unsubscribe=One-Click       │
│         X-Campaign-Id: {campaignId}                              │
│         X-Contact-Id: {contactId}                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Event Tracking & Analytics

SES publishes delivery events via a Configuration Set → SNS → Webhook Lambda pipeline.

| Event | Source | Action | Counter Updated |
|-------|--------|--------|----------------|
| **Send** | SES queues email | Log send record | — |
| **Delivery** | MTA confirms receipt | Increment | `deliveredCount` |
| **Open** | Tracking pixel loaded | Increment | `openedCount` |
| **Click** | Tracked link clicked | Increment | `clickedCount` |
| **Bounce** | Hard or soft bounce | Increment + suppress (hard only) | `bouncedCount` |
| **Complaint** | Recipient marks as spam | Increment + suppress | `complainedCount` |

All counter updates use DynamoDB's atomic `ADD` operation — no read-modify-write races.

### Suppression & Compliance

```
┌─────────────────────────────────────────────────────────────────┐
│  SUPPRESSION LIST                                                │
│                                                                  │
│  Records: SUPPRESSION#{email}#META in data table                 │
│                                                                  │
│  Auto-added on:                                                  │
│  ├── Hard bounce (permanent delivery failure)                    │
│  ├── Spam complaint (recipient reported as spam)                 │
│  └── One-click unsubscribe (RFC 8058 compliant)                 │
│                                                                  │
│  Checked:                                                        │
│  ├── Before every individual email send (Sender Lambda)          │
│  └── Suppressed contacts are skipped, never re-contacted         │
│                                                                  │
│  One-Click Unsubscribe:                                          │
│  ├── URL: GET /campaigns/unsubscribe?cid=X&email=Y&token=Z      │
│  ├── Token: HMAC-SHA256(email + campaignId, UNSUBSCRIBE_SECRET)  │
│  ├── No authentication required (per RFC 8058)                   │
│  └── Adds to suppression list immediately                        │
│                                                                  │
│  Manual management:                                              │
│  ├── GET  /campaigns/suppressions — List all                     │
│  ├── POST /campaigns/suppressions — Add manually                 │
│  └── DELETE /campaigns/suppressions — Remove entry               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Contact Management

### Contact Lifecycle

```mermaid
stateDiagram-v2
    [*] --> active: Created (manual or import)

    active --> unsubscribed: One-click unsubscribe
    active --> bounced: Hard bounce detected
    active --> active: Updated / synced

    unsubscribed --> active: Manual reactivation
    bounced --> [*]: Suppressed permanently

    note right of active: Eligible for campaigns
    note right of unsubscribed: In suppression list
    note right of bounced: In suppression list
```

### Segmentation Engine

Segments are dynamic filters defined by rules. Each rule has a **field**, **operator**, and **value**. Rules combine with **AND** or **OR** logic.

```
┌─────────────────────────────────────────────────────────────────┐
│  SEGMENT RULE STRUCTURE                                          │
│                                                                  │
│  {                                                               │
│    "rules": [                                                    │
│      { "field": "status",    "operator": "equals",  "value": "active" },
│      { "field": "tags",      "operator": "contains","value": "premium" },
│      { "field": "firstName", "operator": "exists",  "value": null }
│    ],                                                            │
│    "logic": "AND"                                                │
│  }                                                               │
│                                                                  │
│  SUPPORTED OPERATORS                                             │
│  ─────────────────────────────────────────────────────────────── │
│  equals         Exact match (case-sensitive)                     │
│  not_equals     Inverse of equals                                │
│  contains       Substring or array element match                 │
│  not_contains   Inverse of contains                              │
│  starts_with    String prefix match                              │
│  exists         Field is present and non-null                    │
│  not_exists     Field is absent or null                          │
│  greater_than   Numeric comparison                               │
│  less_than      Numeric comparison                               │
│                                                                  │
│  EVALUATION                                                      │
│  ─────────────────────────────────────────────────────────────── │
│  Segments are evaluated at send time — not pre-computed.         │
│  All contacts are scanned and filtered by rules.                 │
│  memberCount is cached after evaluation for display purposes.    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### CSV Import Pipeline

```mermaid
graph TD
    UPLOAD["User uploads CSV file"]
    PARSE["PapaParse: parse CSV<br/>(frontend)"]
    MAP["Column mapping UI<br/>(email, firstName, etc.)"]
    VALIDATE["Validate rows:<br/>email format, required fields"]
    PREVIEW["Preview: show first 10 rows"]
    SEND["POST /contacts/import<br/>(batch payload)"]
    DEDUP["Backend: deduplicate by email"]
    WRITE["Write CONTACT#{id} records<br/>(batch PutItem)"]
    RESULT["Return: created, updated,<br/>skipped counts"]

    UPLOAD --> PARSE --> MAP --> VALIDATE --> PREVIEW --> SEND
    SEND --> DEDUP --> WRITE --> RESULT

    style UPLOAD fill:#4A90D9,color:#fff
    style RESULT fill:#50C878,color:#fff
```

---

## Template System

### Template Types

| Type | Editor | Storage | Best For |
|------|--------|---------|----------|
| **Visual** | React Email Editor (drag & drop) | Design JSON in S3 (`designS3Key`) + compiled HTML (`s3Key`) | Non-technical users, quick campaigns |
| **Code** | Monaco Editor (HTML/MJML) | MJML source in S3 (`mjmlS3Key`) + compiled HTML (`s3Key`) | Developers, complex layouts |

Both types compile to final HTML stored in S3. The Sender Lambda always loads the compiled HTML — it doesn't know or care which editor created it.

### Version Control

```
┌─────────────────────────────────────────────────────────────────┐
│  TEMPLATE VERSIONING                                             │
│                                                                  │
│  Every save creates a new version:                               │
│  ├── TEMPLATE#{id}#VERSION#1  (original)                         │
│  ├── TEMPLATE#{id}#VERSION#2  (first edit)                       │
│  ├── TEMPLATE#{id}#VERSION#3  (current)                          │
│  └── TEMPLATE#{id}#META.currentVersion = 3                       │
│                                                                  │
│  S3 keys include version:                                        │
│  ├── templates/{templateId}/v1.html                              │
│  ├── templates/{templateId}/v2.html                              │
│  └── templates/{templateId}/v3.html                              │
│                                                                  │
│  Operations:                                                     │
│  ├── GET  /templates/{id}/versions → List all versions           │
│  ├── POST /templates/{id}/revert   → Set currentVersion = N      │
│  └── POST /templates/{id}/duplicate → Clone as new template      │
│                                                                  │
│  Reverting does NOT delete newer versions — it repoints the      │
│  currentVersion counter, preserving full history.                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Variable System

Templates use Handlebars syntax for personalization:

| Variable | Source | Example |
|----------|--------|---------|
| `{{firstName}}` | Contact record | "Jane" |
| `{{lastName}}` | Contact record | "Doe" |
| `{{email}}` | Contact record | "jane@example.com" |
| `{{unsubscribeUrl}}` | Auto-generated | HMAC-signed unsubscribe link |
| `{{customFields.company}}` | Contact custom fields | "Acme Corp" |
| `{{customFields.*}}` | Contact custom fields | Any user-defined field |

---

## CRM Integrations

```mermaid
graph LR
    subgraph "HubSpot"
        HS_AUTH["Private App Token"]
        HS_API["HubSpot API"]
        HS_SYNC["Sync contacts"]
    end

    subgraph "Zoho"
        ZO_AUTH["OAuth 2.0<br/>Refresh Token"]
        ZO_API["Zoho CRM API"]
        ZO_SYNC["Sync contacts"]
    end

    subgraph "Volley"
        INT_LAMBDA["Integrations Lambda"]
        DEDUP["Deduplication engine"]
        DB["DynamoDB contacts"]
    end

    HS_AUTH --> HS_API --> HS_SYNC --> INT_LAMBDA
    ZO_AUTH --> ZO_API --> ZO_SYNC --> INT_LAMBDA
    INT_LAMBDA --> DEDUP --> DB

    style HS_AUTH fill:#FF7A59,color:#fff
    style ZO_AUTH fill:#DC2626,color:#fff
    style DB fill:#4053D6,color:#fff
```

| Integration | Auth Method | Sync Direction | Dedup Key | Stored In |
|-------------|-------------|----------------|-----------|-----------|
| **HubSpot** | Private App token | HubSpot → Volley | `externalIds.hubspotId` | DynamoDB (encrypted token) |
| **Zoho** | OAuth 2.0 refresh token | Zoho → Volley | `externalIds.zohoId` | DynamoDB (OAuth credentials) |

**Sync behavior:**
- Imports contacts with email deduplication
- Maps external IDs for future syncs
- Sets `syncSource` to `hubspot` or `zoho`
- Preserves local customizations (tags, custom fields)
- Token refresh handled automatically (Zoho OAuth)

---

## AI Features (AWS Bedrock)

```mermaid
graph TD
    subgraph "AI Lambda (/ai/*)"
        GEN["POST /ai/generate<br/><i>Create email from prompt</i>"]
        IMP["POST /ai/improve<br/><i>Suggest improvements</i>"]
        AGENT["POST /ai/agent<br/><i>Multi-turn chat</i>"]
    end

    subgraph "AWS Bedrock"
        MODEL["Claude Haiku 4.5<br/><i>Temperature: 0.3</i><br/><i>Max tokens: 4096</i>"]
    end

    GEN --> MODEL
    IMP --> MODEL
    AGENT --> MODEL

    style MODEL fill:#00A4EF,color:#fff
    style GEN fill:#50C878,color:#fff
    style IMP fill:#F5A623,color:#fff
    style AGENT fill:#7B68EE,color:#fff
```

### Three AI Capabilities

| Feature | Input | Output | Use Case |
|---------|-------|--------|----------|
| **Generate** | Prompt + tone (professional, casual, urgent, friendly) | Subject line + preview text + body HTML | Create email copy from scratch |
| **Improve** | Current subject + body + optional goal | Array of suggestions: `{type, original, improved, reason}` + effectiveness score | Polish existing content |
| **Agent** | Conversation history + current message | Final text + list of tools used | Interactive AI assistant with tool access |

**Agent mode** supports multi-turn conversation with tool execution (content analysis, email generation, suggestion generation). It runs an agentic loop of up to 5 iterations: model → tools → results → model, preventing infinite loops.

**Configuration:**
- Model: `us.anthropic.claude-haiku-4-5-20251001-v1:0` (configurable via `AI_MODEL_ID`)
- Region: `us-east-1` (separate from main infrastructure region)
- Temperature: 0.3 (deterministic, consistent output)
- IAM: `bedrock:InvokeModel` permission on `arn:aws:bedrock:*::foundation-model/*`

---

## AWS Infrastructure

### Service Map

```mermaid
graph TD
    subgraph "Compute"
        LAMBDA["Lambda<br/>12 functions<br/>Node.js 22"]
        APIGW["API Gateway<br/>REST API + Authorizer"]
    end

    subgraph "Storage"
        DDB["DynamoDB<br/>2 tables (on-demand)"]
        S3T["S3 — Templates<br/>HTML, MJML, design JSON"]
        S3F["S3 — Frontend<br/>Static SPA assets"]
    end

    subgraph "Email"
        SES["SES v2<br/>Domain-verified sender"]
        SQS["SQS<br/>Email job queue"]
        SNS["SNS<br/>Delivery events"]
    end

    subgraph "Scheduling"
        EB["EventBridge Scheduler<br/>Campaign schedules"]
    end

    subgraph "CDN"
        CF["CloudFront<br/>Frontend distribution"]
    end

    subgraph "AI"
        BR["Bedrock<br/>Claude model inference"]
    end

    subgraph "Observability"
        CW["CloudWatch<br/>Lambda logs + metrics"]
    end

    APIGW --> LAMBDA
    LAMBDA --> DDB & S3T & SQS & SES & BR
    SQS --> LAMBDA
    SNS --> LAMBDA
    SES --> SNS
    EB --> LAMBDA
    CF --> S3F

    style LAMBDA fill:#FF9900,color:#fff
    style APIGW fill:#FF9900,color:#fff
    style DDB fill:#4053D6,color:#fff
    style SES fill:#DD344C,color:#fff
    style SQS fill:#FF9900,color:#fff
    style CF fill:#8C4FFF,color:#fff
    style BR fill:#00A4EF,color:#fff
```

### Deployment Pipeline

The master `deploy.sh` script orchestrates full infrastructure provisioning in dependency order:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT ORDER (scripts/deploy.sh)                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Step  Script                    What It Creates                        │
│  ────────────────────────────────────────────────────────────────────── │
│    1    validate-env.sh           Check .env for required variables       │
│    2    build.ts                  esbuild all Lambda functions            │
│    3    setup-dynamodb.sh         Create users + data tables             │
│    4    setup-s3-buckets.sh       Create templates + frontend buckets    │
│    5    setup-ses.sh              Domain identity + verification         │
│    6    setup-sqs.sh              Email queue                            │
│    7    setup-lambdas.sh          Deploy all 12 Lambda functions         │
│    8    setup-sender-lambda.sh    Configure sender with SQS trigger      │
│    9    setup-api-gateway.sh      REST API + routes + authorizer         │
│   10    setup-eventbridge.sh      Campaign scheduler group               │
│   11    setup-ses-tracking.sh     Config set + SNS + webhook trigger     │
│   12    setup-cloudfront.sh       Frontend CDN distribution              │
│   13    seed-data.sh              Admin user + starter templates         │
│                                                                          │
│  TEARDOWN: scripts/teardown.sh removes ALL resources in reverse order   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Naming Conventions

All AWS resources use a tenant prefix for multi-tenant isolation:

| Resource | Naming Pattern | Example |
|----------|---------------|---------|
| DynamoDB tables | `{TENANT_PREFIX}-users`, `-data` | `edm-users`, `edm-data` |
| S3 buckets | `{TENANT_PREFIX}-templates-{ACCOUNT_ID}` | `edm-templates-123456789` |
| Lambda functions | `{TENANT_PREFIX}-{function}` | `edm-auth`, `edm-campaigns` |
| API Gateway | `{TENANT_PREFIX}-api` | `edm-api` |
| SQS queue | `{TENANT_PREFIX}-email-queue` | `edm-email-queue` |
| SNS topic | `{TENANT_PREFIX}-ses-events` | `edm-ses-events` |
| EventBridge group | `{TENANT_PREFIX}-campaigns` | `edm-campaigns` |
| IAM roles | `{TENANT_PREFIX}-api-role`, `-scheduler-role` | `edm-api-role` |

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  SECURITY LAYERS                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TRANSPORT                                                       │
│  ├── HTTPS throughout (CloudFront + API Gateway)                │
│  └── TLS termination at edge (CloudFront)                       │
│                                                                  │
│  AUTHENTICATION                                                  │
│  ├── JWT: 15-min TTL, HS256, verified on every request          │
│  ├── Refresh token: 7-day TTL, HttpOnly cookie, rotation        │
│  ├── Password: bcryptjs hashing (10 rounds)                     │
│  └── Email verification required for new accounts               │
│                                                                  │
│  AUTHORIZATION                                                   │
│  ├── Role-based: admin > editor > viewer                        │
│  ├── Every write operation checks role in handler               │
│  └── Authorizer Lambda returns role as request context          │
│                                                                  │
│  API SECURITY                                                    │
│  ├── CORS: Locked to FRONTEND_URL origin (not wildcard)         │
│  ├── OPTIONS preflight: Handled in every Lambda handler         │
│  └── Input validation: Zod schemas on request bodies            │
│                                                                  │
│  EMAIL SECURITY                                                  │
│  ├── SES domain verification: SPF + DKIM + DMARC                │
│  ├── Suppression list: Bounces + complaints auto-suppressed     │
│  ├── Unsubscribe HMAC: SHA256(email + campaignId, secret)       │
│  └── RFC 8058: List-Unsubscribe + List-Unsubscribe-Post         │
│                                                                  │
│  DATA PROTECTION                                                 │
│  ├── DynamoDB: Encryption at rest (AWS managed)                 │
│  ├── S3: Encryption at rest (SSE-S3)                            │
│  ├── Token TTL: Auto-deleted by DynamoDB TTL                    │
│  └── No PII in logs (CloudWatch)                                │
│                                                                  │
│  INFRASTRUCTURE                                                  │
│  ├── IAM: Least-privilege roles per function                    │
│  ├── Lambda: Execution role scoped to specific resources        │
│  └── No standing servers — zero attack surface at rest          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Examples

### Campaign Execution — End to End

```
STAGE 1: PREPARATION (User → Frontend → API)
┌────────────────────────────────────────────────────────────────┐
│  User creates campaign: name, subject line                      │
│  User selects template: picks from template gallery             │
│  User selects segment: "Active Premium Subscribers"             │
│  User previews audience: sees matching contacts + count         │
│  User clicks "Send Now"                                         │
│  → POST /campaigns/{id}/execute                                 │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼

STAGE 2: QUEUING (Campaigns Lambda)
┌────────────────────────────────────────────────────────────────┐
│  Evaluate segment rules against all contacts in DynamoDB        │
│  Filter: status=active AND not in suppression list              │
│  Result: 2,500 matching contacts                                │
│  Set campaign: totalEmails=2500, status=executing               │
│  Queue 2,500 SQS messages (1 per contact)                      │
│  Each message: { campaignId, contactId, email, templateS3Key }  │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼

STAGE 3: SENDING (Sender Lambda — SQS consumer)
┌────────────────────────────────────────────────────────────────┐
│  For each SQS message:                                          │
│  1. Check SUPPRESSION#{email} — skip if exists                  │
│  2. Load HTML from S3: templates/tpl-123/v3.html                │
│  3. Compile with Handlebars: inject firstName, lastName, etc.   │
│  4. Build MIME: add List-Unsubscribe, campaign tracking headers │
│  5. SES SendEmailCommand → get MessageId                        │
│  6. Write CAMPAIGN#{id}#EMAIL#{contactId} (idempotent PutItem)  │
│  7. Atomic ADD sentCount (or failedCount on error)              │
│  8. Check completion: 2500 == sentCount + failedCount?          │
│     → Yes: update status to completed/failed                    │
└──────────────────────────────┬─────────────────────────────────┘
                               │
                               ▼

STAGE 4: TRACKING (SES → SNS → Webhook Lambda)
┌────────────────────────────────────────────────────────────────┐
│  SES Configuration Set fires events:                            │
│  → Delivery: atomic ADD deliveredCount                          │
│  → Open: atomic ADD openedCount                                 │
│  → Click: atomic ADD clickedCount                               │
│  → Bounce: atomic ADD bouncedCount + SUPPRESSION#{email}        │
│  → Complaint: atomic ADD complainedCount + SUPPRESSION#{email}  │
│                                                                  │
│  Frontend polls /campaigns/{id} for real-time dashboard updates │
└────────────────────────────────────────────────────────────────┘
```

### Contact Import — End to End

```
┌────────────────────────────────────────────────────────────────┐
│  1. User uploads contacts.csv (frontend)                        │
│  2. PapaParse parses CSV → array of rows                        │
│  3. Column mapping UI: user maps CSV headers to fields          │
│     "Email Address" → email, "First" → firstName, etc.          │
│  4. Validation: check email format, required fields             │
│  5. Preview: show first 10 rows for confirmation                │
│  6. POST /contacts/import → batch payload to Contacts Lambda    │
│  7. Backend deduplicates by email                               │
│     → Existing: update fields (merge)                           │
│     → New: create CONTACT#{uuid}#PROFILE                        │
│  8. Return: { created: 1847, updated: 612, skipped: 41 }       │
└────────────────────────────────────────────────────────────────┘
```

### One-Click Unsubscribe

```
┌────────────────────────────────────────────────────────────────┐
│  1. Recipient clicks unsubscribe link in email                  │
│     GET /campaigns/unsubscribe?cid=abc&email=x@y.com&token=... │
│                                                                  │
│  2. Campaigns Lambda (no auth required):                        │
│     a. Verify HMAC: SHA256(email + campaignId, secret) == token │
│     b. Create SUPPRESSION#{email}#META in DynamoDB              │
│     c. Update contact status to "unsubscribed"                  │
│     d. Return confirmation page                                 │
│                                                                  │
│  3. Future sends:                                                │
│     Sender Lambda checks suppression list → skips this email    │
│     This contact will never receive another campaign             │
│                                                                  │
│  Per RFC 8058: No login, no confirmation step, immediate effect │
└────────────────────────────────────────────────────────────────┘
```

---

## Environment Variables

### Frontend (Vite)

| Variable | Description | Example |
|----------|-------------|---------|
| `VITE_API_URL` | API Gateway endpoint | `https://abc123.execute-api.ap-southeast-2.amazonaws.com/prod` |
| `VITE_APP_NAME` | Branding name | `Volley` |
| `VITE_LOGO_URL` | Logo image URL | `https://cdn.example.com/logo.png` |
| `VITE_PRIMARY_COLOR` | Theme accent color | `#2563EB` |

### Backend (Lambda)

| Variable | Description | Example |
|----------|-------------|---------|
| `USERS_TABLE_NAME` | Users DynamoDB table | `edm-users` |
| `DATA_TABLE_NAME` | Data DynamoDB table | `edm-data` |
| `TEMPLATES_BUCKET_NAME` | S3 bucket for templates | `edm-templates-123456789` |
| `JWT_SECRET` | JWT signing secret (32+ bytes) | `(random string)` |
| `UNSUBSCRIBE_SECRET` | HMAC secret for unsubscribe tokens | `(random string)` |
| `SENDER_DOMAIN` | Verified SES domain | `mail.example.com` |
| `APP_NAME` | Application name for emails | `Volley` |
| `FRONTEND_URL` | CloudFront URL (CORS origin) | `https://d1234.cloudfront.net` |
| `EMAIL_QUEUE_URL` | SQS queue URL | `https://sqs.ap-southeast-2.amazonaws.com/123/edm-email-queue` |
| `AWS_REGION` | Primary AWS region | `ap-southeast-2` |
| `BEDROCK_REGION` | Bedrock model region | `us-east-1` |
| `AI_MODEL_ID` | Bedrock model ID | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |
| `SCHEDULER_GROUP_NAME` | EventBridge group | `edm-campaigns` |
| `SENDER_LAMBDA_ARN` | Sender Lambda ARN | `arn:aws:lambda:...` |
| `SCHEDULER_ROLE_ARN` | EventBridge execution role | `arn:aws:iam::...:role/...` |
| `HUBSPOT_ACCESS_TOKEN` | HubSpot private app token | `(optional)` |
| `ZOHO_CLIENT_ID` | Zoho OAuth client ID | `(optional)` |
| `ZOHO_CLIENT_SECRET` | Zoho OAuth secret | `(optional)` |
| `ZOHO_REFRESH_TOKEN` | Zoho OAuth refresh token | `(optional)` |

---

## Quick Stats

| Metric | Value |
|--------|-------|
| **Architecture** | Serverless monorepo (React + Lambda + DynamoDB) |
| **Packages** | 3 (frontend, backend, shared) |
| **Lambda functions** | 12 (10 API + 1 sender + 1 webhook) |
| **API routes** | 45+ endpoints across 8 function groups |
| **DynamoDB tables** | 2 (users, data) — on-demand billing |
| **S3 buckets** | 2 (templates, frontend) |
| **AWS services used** | 12 (Lambda, API GW, DynamoDB, SQS, SES, S3, CloudFront, EventBridge, SNS, CloudWatch, IAM, Bedrock) |
| **Frontend framework** | React 19 + Vite 8 + TanStack Router |
| **UI library** | shadcn/ui + Tailwind CSS 4 |
| **Template editors** | 2 (Visual drag-and-drop, Monaco code editor) |
| **Auth method** | JWT (15min) + Refresh token (7-day HttpOnly cookie) |
| **User roles** | 3 (admin, editor, viewer) |
| **Email rendering** | Handlebars + MJML → HTML |
| **AI model** | Claude Haiku 4.5 via AWS Bedrock |
| **AI features** | 3 (generate, improve, agent chat) |
| **CRM integrations** | 2 (HubSpot, Zoho) |
| **Campaign states** | 5 (draft, scheduled, executing, completed, failed) |
| **Contact statuses** | 3 (active, unsubscribed, bounced) |
| **Tracked events** | 6 (send, delivery, open, click, bounce, complaint) |
| **Deployment scripts** | 18+ bash scripts |
| **Primary region** | ap-southeast-2 (Sydney) |
| **Bedrock region** | us-east-1 (N. Virginia) |
| **Standing servers** | 0 — fully serverless |

---

*For deployment instructions, see [Setup Guide](./setup-guide.md).*
*For endpoint details, see [API Reference](./api-reference.md).*
*For user-facing documentation, see [User Manual](./user-manual.md).*
*Last updated: 2026-04-02*
