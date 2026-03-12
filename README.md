# QuickQuid: System Architecture & Technical Specification

## 1. High-Level System Architecture
The system uses a modular monolith approach designed for event-driven scaling via NATS JetStream and state-managed performance via Redis.

```mermaid
graph TD
    subgraph Client_Layer
        Web[Next.js 15 App]
        PersonalSites[user.quickquid.in Subdomains]
    end

    subgraph Infrastructure_PaaS
        Coolify[Coolify Orchestrator - Ubuntu VPS]
        Nix[Nix Flake Build System]
    end

    subgraph Backend_Services_Go
        Gateway[API Gateway]
        Auth[EDU Verification & Identity]
        Escrow[Escrow State Machine]
        Chat[Chat & Object Messaging]
        Market[Marketplace & Search]
    end

    subgraph Storage_Event_Layer
        NATS((NATS JetStream - Event Bus))
        Redis[(Redis - Escrow Hot State & Sessions)]
        Postgres[(PostgreSQL - Primary DB)]
        R2[Cloudflare R2 - Portfolios & Files]
    end

    Web <--> Gateway
    Gateway <--> Auth & Escrow & Chat & Market
    Escrow <--> Redis
    Chat <--> NATS
    Escrow <--> NATS
    Auth & Market & Escrow --> Postgres
    PersonalSites <--> R2

2. User Lifecycle & Page Workflow
Detailed mapping of the student journey from EDU verification to professional networking and personal site management.
flowchart TD
    subgraph Onboarding
        Start[Sign-In via EDU Email] --> Verify{Verification}
        Verify -->|Success| Profile[Build Profile]
        Profile --> Subdomain[Provision user.quickquid.in]
    end

    subgraph Platforms
        Subdomain --> Site[Personal Portfolio Site]
        Subdomain --> Dash[User Dashboard]
    end

    subgraph Dashboard_Modules
        Dash --> Inbox[Unified Inbox - Chat Objects]
        Dash --> Hunt[Job Hunt - Search]
        Dash --> Chain[Freelance Chain - Network]
    end

    subgraph Professional_Output
        Site --> Rent[Rentable Ad/Page Space]
        Chain --> Team[Form Virtual Agency]
        Hunt --> Work[Apply & Set Escrow]
    end

3. Buyer & Company Pipeline
Flow for clients to post requirements, interview talent, and manage milestone-based payments.
flowchart LR
    subgraph Hiring
        Post[Post Job] --> Match[Redis Filtering]
        Match --> Interview[Interview Suite]
    end

    subgraph Contract_Management
        Interview --> Terms[Set Milestones]
        Terms --> Lock[Lock Escrow Funds]
        Lock --> Release[Automated Payout]
    end

4. Escrow State Machine Logic
Deterministic fund management using Redis for high-speed state transitions and Postgres for the permanent ledger.
stateDiagram-v2
    [*] --> Proposed: Freelancer/Buyer Drafts Terms
    Proposed --> Locked: Buyer Deposits to Escrow
    Locked --> InProgress: Work Commences
    InProgress --> Delivered: Work Uploaded to R2
    Delivered --> Released: Buyer Approves
    Delivered --> Dispute: Arbitration Triggered
    Dispute --> Released: Manual/Legal Resolution
    Released --> [*]: Funds Dispersed

5. Unified Inbox: Chat Object Sequence
Interaction logic for rendering "Special Objects" (contracts, payments, portfolios) within a real-time stream.
sequenceDiagram
    participant U1 as User A
    participant B as Go Backend
    participant R as Redis (Activity)
    participant N as NATS (Event Bus)
    participant U2 as User B

    U1->>B: Send Object (type: PAYMENT_REQUEST)
    B->>R: Cache Active Transaction
    B->>N: Publish Event to Subject
    N->>U2: Deliver WebSocket Payload
    Note over U2: Client decodes JSON and renders Actionable Card
    U2->>B: Interaction (Accept/Pay)
    B->>R: Update Hot State

6. Freelance Chain: Peer Connection Logic
The "Chain" allows students to link profiles to form collaborative units for complex projects.
graph BT
    Dev[Developer] --> Chain[Freelance Chain]
    Designer[Designer] --> Chain
    Writer[Content Writer] --> Chain
    Chain --> Agency[Virtual Agency Object]
    Agency --> MultiPay[Split-Payment Escrow]
    Client[Company/Buyer] --> MultiPay

7. Technical Stack Specification
| Layer | Technology | Implementation Detail |
|---|---|---|
| Language | Go (Golang) | All backend logic, microservices, and concurrency. |
| Frontend | Next.js 15 | SSR for subdomains; React Server Components for chat objects. |
| Build System | Nix / Flakes | Hermetic builds; identical local/prod environments. |
| Deployment | Coolify | Automated orchestration on Ubuntu VPS infrastructure. |
| Event Bus | NATS JetStream | Real-time messaging and persistent event logs. |
| Hot Layer | Redis | Escrow activity tracking, locks, and session caching. |
| Cold Layer | PostgreSQL | Source of truth for users, history, and financial records. |
| File Store | Cloudflare R2 | S3-compatible, zero-egress storage for portfolios. |
8. Data Model: Special Objects
Go struct definition for interactive chat messages.
type SpecialObject struct {
    ObjectID    string                 `json:"object_id"`
    SenderID    string                 `json:"sender_id"`
    Type        string                 `json:"type"` // e.g., MILESTONE, PAYMENT, PORTFOLIO
    Payload     map[string]interface{} `json:"payload"` // Data for React component
    Status      string                 `json:"status"` // PENDING, ACCEPTED, PAID
    CreatedAt   int64                  `json:"created_at"`
}

9. Infrastructure Optimization
 * Performance: Redis handles all "hot" escrow activity, bypassing Postgres disk I/O for 90% of state updates.
 * Storage: Large files (Portfolios/Assets) are never stored in the database; only Cloudflare R2 URLs are persisted.
 * Builds: Nix-built containers contain no shell or extra binaries (distroless), minimizing security attack vectors.
 * Scaling: NATS allows services to be moved to different servers without changing a single line of code.
<!-- end list -->

**Would you like me to generate the specific Nix Flake file to initialize this environment locally?**
