# Pre-Read Documentation

## Senior Engineer → Solution Architect Accelerated Program

### Bridging the Gap: From Implementation Thinking to Architectural Thinking

---

> **Document Purpose:** This pre-read is mandatory preparation material for all participants enrolling in the Senior Engineer → Solution Architect program. It is designed to elevate a mid-level software engineer's conceptual foundation to the entry threshold required for Day 1 of this course. Read this document in sequence. Do not skip sections.

---

# Table of Contents

1. [How to Use This Document](#)
2. [Abbreviations & Glossary Master Reference](#)
3. [Part 1 — Architectural Foundations & Design Thinking](#)
4. [Part 2 — Microservices, AI & Modernization](#)
5. [Part 3 — DevSecOps, Deployment & Excellence](#)
6. [Terms to Google — Research Guide by Reader Profile](#)
7. [Self-Assessment Checklist Before Day 1](#)

---

> **Note on Length:** This document is structured across multiple parts. Each part aligns to a course phase. You are currently reading **Part 0 (Orientation) + Part 1 (Architectural Foundations)**. Subsequent parts will follow upon prompt.

---

# How to Use This Document

```
┌─────────────────────────────────────────────────────────────────┐
│                    READING FLOW GUIDE                           │
│                                                                 │
│  Step 1 ──► Read Section 2 (Glossary) first                    │
│             Bookmark it. Return to it as you read.             │
│                                                                 │
│  Step 2 ──► Follow Parts 1 → 2 → 3 in sequence                │
│             Each Part maps to a Course Phase                   │
│                                                                 │
│  Step 3 ──► After each Part, review the                        │
│             "Terms to Google" section for that Part            │
│                                                                 │
│  Step 4 ──► Complete Self-Assessment Checklist                 │
│             before attending Day 1 (Jun 11)                    │
│                                                                 │
│  Step 5 ──► Attempt the LMS Pre-Program Assessment             │
│             (Due: Before Jun 11)                               │
└─────────────────────────────────────────────────────────────────┘
```

**Who should read this?**

| Reader Profile                                                    | Reading Priority                                   |
| ----------------------------------------------------------------- | -------------------------------------------------- |
| Mid-level engineer (3–5 years) with limited architecture exposure | Read every section carefully                       |
| Senior engineer (5–8 years) comfortable with microservices        | Focus on Parts 2 and 3, skim Part 1                |
| Engineer transitioning from monolith/legacy systems               | Focus heavily on Part 1 and Part 2 Legacy sections |
| Engineer from infrastructure/DevOps background                    | Focus on Part 1 (design thinking) and Part 3       |

---

# Section 2 — Abbreviations & Glossary Master Reference

> **Reading Instruction:** You do not need to memorize all of these before starting. Read them once to get familiarity. When you encounter a term in later sections, return here for the precise definition.

## 2.1 Abbreviations

| Abbreviation | Full Form                                                        | Context                                                                         |
| ------------ | ---------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| ADR          | Architecture Decision Record                                     | Documentation of an architectural choice and its rationale                      |
| API          | Application Programming Interface                                | Contract between software components                                            |
| BFF          | Backend For Frontend                                             | API layer tailored to a specific client type                                    |
| CAP          | Consistency, Availability, Partition Tolerance                   | Fundamental theorem of distributed systems                                      |
| CDC          | Change Data Capture                                              | Technique to track database changes in real-time                                |
| CI/CD        | Continuous Integration / Continuous Delivery                     | Automated build, test, and deploy pipeline                                      |
| CQRS         | Command Query Responsibility Segregation                         | Pattern separating read and write data models                                   |
| DAST         | Dynamic Application Security Testing                             | Security testing against running application                                    |
| DDD          | Domain-Driven Design                                             | Software design approach aligned with business domain                           |
| DLQ          | Dead Letter Queue                                                | Queue holding messages that could not be processed                              |
| EDA          | Event-Driven Architecture                                        | System architecture based on event production and consumption                   |
| ELK          | Elasticsearch, Logstash, Kibana                                  | Logging and observability stack                                                 |
| HPA          | Horizontal Pod Autoscaler                                        | Kubernetes mechanism to auto-scale pods                                         |
| IaC          | Infrastructure as Code                                           | Managing infrastructure through code (e.g., Terraform)                          |
| IdP          | Identity Provider                                                | Service that authenticates identities (e.g., Keycloak, Azure AD)                |
| IoT          | Internet of Things                                               | Network of physical devices connected to the internet                           |
| mTLS         | Mutual Transport Layer Security                                  | Two-way TLS certificate verification between services                           |
| NFR          | Non-Functional Requirement                                       | Requirements about how a system behaves (performance, security, etc.)           |
| OWASP        | Open Web Application Security Project                            | Security standard and vulnerability reference                                   |
| PACELC       | Partition, Availability, Consistency, Else, Latency, Consistency | Extended CAP theorem                                                            |
| RBAC         | Role-Based Access Control                                        | Access control model based on user roles                                        |
| RPO          | Recovery Point Objective                                         | Maximum acceptable data loss in time during failure                             |
| RTO          | Recovery Time Objective                                          | Maximum acceptable downtime during failure                                      |
| SAGA         | —                                                                | Pattern for managing distributed transactions through compensating transactions |
| SAST         | Static Application Security Testing                              | Security scanning of source code                                                |
| SCA          | Software Composition Analysis                                    | Scanning third-party libraries for vulnerabilities                              |
| SLA          | Service Level Agreement                                          | Contractual commitment on service availability                                  |
| SLI          | Service Level Indicator                                          | Actual measured metric (e.g., p99 latency)                                      |
| SLO          | Service Level Objective                                          | Target value for an SLI (e.g., 99.9% uptime)                                    |
| SOA          | Service-Oriented Architecture                                    | Architecture using coarse-grained services over shared bus                      |
| TCO          | Total Cost of Ownership                                          | Full lifetime cost of a technology choice                                       |
| TLS          | Transport Layer Security                                         | Encryption protocol for data in transit                                         |
| WAF          | Web Application Firewall                                         | Security layer filtering malicious HTTP traffic                                 |
| ZTA          | Zero Trust Architecture                                          | Security model where no entity is inherently trusted                            |

---

## 2.2 Glossary of Core Concepts

**Aggregate (DDD)**
> A cluster of domain objects that are treated as a single unit for data changes. Every aggregate has a root entity (Aggregate Root) through which all external interactions happen. Example: In a procurement system, a `PurchaseOrder` is an aggregate root; its `LineItems` and `ApprovalHistory` are part of the same aggregate.

**Bounded Context (DDD)**
> A clearly defined boundary within which a specific domain model applies and has a consistent meaning. The same word "Account" means something different in the banking context versus the user authentication context. These are two separate bounded contexts.

**Bulkhead Pattern**
> Borrowed from ship design — compartments that prevent water from flooding the entire ship if one section is breached. In software, it isolates failures in one part of the system from cascading to others. Implemented using thread pool isolation or process separation.

**Circuit Breaker**
> A pattern that prevents an application from repeatedly calling a failing service. The circuit "opens" after a threshold of failures, rejecting calls immediately. After a timeout, it enters a half-open state to test recovery. Made popular by Netflix Hystrix (now Resilience4j).

**Eventual Consistency**
> A consistency model where, after a period with no new updates, all replicas of a data item will eventually converge to the same value. Contrast with strong consistency, where all reads return the most recent write immediately.

**Event Sourcing**
> Storing state as a sequence of events rather than the current state snapshot. Instead of updating a row, you append an event like `OrderPlaced`, `OrderApproved`, `OrderShipped`. The current state is derived by replaying events.

**Hexagonal Architecture**
> Also called "Ports and Adapters." The core business logic sits in the center. Communication happens through defined ports (interfaces). Adapters connect the ports to the outside world (REST, database, message queues). This keeps business logic completely independent of infrastructure.

**Idempotency**
> An operation is idempotent if performing it multiple times produces the same result as performing it once. Critical for distributed systems where retries are common. A payment deduction that can be retried without double-charging is idempotent.

**Non-Functional Requirement (NFR)**
> Requirements that define how a system performs its functions, not what it does. Examples: "The system must respond to 95% of API calls within 200ms under 10,000 concurrent users" or "The system must be available 99.95% of the time."

**Observability**
> The ability to understand the internal state of a system from its external outputs. Built on three pillars: Metrics (numerical measurements), Logs (event records), and Traces (request journeys across services).

**Polyglot Persistence**
> Using different database technologies for different parts of the same system, each chosen for its suitability to that specific use case. Example: PostgreSQL for transactional data, Redis for caching, MongoDB for document storage, Neo4j for graph relationships.

**Service Mesh**
> An infrastructure layer that handles service-to-service communication, providing traffic management, security (mTLS), and observability without requiring changes to application code. Istio and Linkerd are common implementations.

**Strangler Fig Pattern**
> A migration strategy where a new system gradually replaces an old one by routing traffic progressively. Like a strangler fig tree that slowly grows around and replaces a host tree. The old system is "strangled" over time rather than replaced in a big-bang migration.

**Ubiquitous Language (DDD)**
> A shared vocabulary used consistently by both developers and domain experts in code, documentation, and conversation. If business calls it a "Tender," the code should use `Tender`, not `Bid`, `Request`, or `Contract`.

**Zero Trust Architecture**
> A security model based on the principle "never trust, always verify." Every access request is authenticated, authorized, and validated regardless of whether it originates inside or outside the network perimeter.

---

# Part 1 — Architectural Foundations & Design Thinking

## Covers: Days 1 through 6 (Jun 11–18)

---

## Chapter 1.1 — From Coder to Architect: The Mindset Shift

### 1.1.1 What Changes When You Become an Architect

A software engineer solves a defined problem using code. A solution architect decides *which problem to solve*, *why that approach*, *at what cost*, and *what gets sacrificed in the process*.

This is not just a seniority promotion. It is a fundamental shift in the unit of thinking.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ENGINEER vs ARCHITECT: UNIT OF THINKING                    │
│                                                                         │
│  ENGINEER                          ARCHITECT                            │
│  ─────────────────────             ─────────────────────────────        │
│  Unit: Function / Class            Unit: System / Subsystem             │
│  Time horizon: Sprint              Time horizon: 3–5 years              │
│  Question: "How to build this?"    Question: "Should we build this?"    │
│  Constraint: Story points          Constraint: Budget, Team, Ops risk   │
│  Success: Tests pass               Success: Business goal achieved      │
│  Trade-off: Performance vs Effort  Trade-off: Cost vs Resilience        │
│                                               vs Time-to-market         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.1.2 Non-Functional Requirements — The Architect's Primary Language

When a business stakeholder says "build a citizen grievance portal for 50 lakh residents of Tamil Nadu," an engineer hears features. An architect hears NFRs.

**NFR Categories Every Architect Must Assess:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NFR TAXONOMY FOR GOV-SCALE SYSTEMS                   │
│                                                                         │
│  PERFORMANCE          RELIABILITY          SECURITY                     │
│  ─────────────        ────────────         ────────                     │
│  Response time        Availability         Authentication               │
│  Throughput           Fault tolerance      Authorization                │
│  Latency (p95/p99)    RTO / RPO            Data privacy                 │
│  Concurrent users     Disaster recovery    Audit trail                  │
│                                                                         │
│  SCALABILITY          MAINTAINABILITY      COMPLIANCE                   │
│  ───────────          ───────────────      ──────────                   │
│  Horizontal scale     Code modularity      PDPA (Singapore)             │
│  Vertical scale       Deployability        DPDP Act (India)             │
│  Elastic scaling      Testability          GovTech standards            │
│  Data growth          Observability        MeitY guidelines             │
│                                                                         │
│  INTEROPERABILITY     COST                                              │
│  ────────────────     ────                                              │
│  API standards        TCO                                               │
│  Data formats         OpEx vs CapEx                                     │
│  Protocol support     Licensing model                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real-World Production Scenario — GSTN (Goods and Services Tax Network, India)**

> The GSTN system was designed to handle 65 crore (650 million) invoices per month at launch in 2017. The NFRs were brutally demanding:
> - **Throughput:** 3 million invoices per hour at peak
> - **Availability:** 99.5% SLA (critical national infrastructure)
> - **Response time:** API response under 3 seconds for invoice filing
> - **Concurrent users:** 300,000 simultaneous users during filing season
>
> The system faced catastrophic failures in the first month of GST rollout. July 2017 saw the portal crash repeatedly during peak filing periods. The root cause was inadequate load testing against actual NFRs and an architecture that was not designed for the actual burst traffic patterns of Indian businesses filing at the last moment before deadlines. This is a textbook case of what happens when NFRs are treated as aspirational rather than contractual design constraints.

**Key Takeaway:** NFRs must be quantified, tested, and designed into the system from the start — not added later.

---

### 1.1.3 Trade-off Thinking — The Core Architect Skill

Every architectural decision involves giving something up. There is no perfect architecture. There are only architectures that are appropriate or inappropriate for a given context.

**The Three Primary Trade-off Axes:**

```
                        CONSISTENCY
                             │
                             │
                    ┌────────┴────────┐
                    │                 │
                    │   Strong        │
                    │   Consistency   │
                    │   (PostgreSQL   │
                    │   ACID txns)    │
                    │                 │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
   AVAILABILITY         TRADE-OFF            PARTITION
   (System stays           ZONE             TOLERANCE
   online even                         (Works despite
   during failures)                    network splits)
         │                                       │
         └──────────────────►────────────────────┘
                  (CAP Theorem forces a choice)

         EXAMPLES:
         ─────────
         CA choice: PostgreSQL — consistent & available, 
                    fails under network partition
         
         AP choice: DynamoDB — available & partition tolerant, 
                    eventual consistency
         
         CP choice: HBase — consistent & partition tolerant, 
                    may refuse requests during partition
```

**Government-Scale Trade-off Example — India Stack / DigiLocker**

> DigiLocker serves 250 million+ registered users storing government documents (Aadhaar, PAN, driving licence). When designing the document retrieval system, architects faced a critical trade-off:
>
> - **Option A:** Strong consistency — Every document read goes to the primary database. Guaranteed latest version. Cost: Higher latency, single point of contention, limited throughput.
> - **Option B:** Eventual consistency — Documents cached at edge nodes. Users may see a slightly old version for a few seconds. Cost: Tiny risk of serving stale data.
>
> The choice was Option B with a carefully controlled staleness window (typically under 5 seconds). The rationale: documents like birth certificates don't change in seconds. The availability gain (serving millions of requests without hitting central database each time) far outweighed the negligible staleness risk. This is a deliberate, documented trade-off — not an accident.

---

### 1.1.4 Architecture Decision Records (ADRs)

An ADR is a short document that captures a single significant architectural decision. It is not a design document. It is a decision log.

**Why ADRs Matter in Government Projects**

Government systems have audit requirements, change management processes, and sometimes span across ministries with different stakeholders. When a team member leaves or a ministry auditor asks "why did you choose MongoDB over Oracle," the ADR is the answer. Without it, institutional knowledge disappears.

**ADR Structure (Michael Nygard Format, adapted):**

```
┌─────────────────────────────────────────────────────────────────┐
│  ADR-007: Use Kafka as the Event Backbone                       │
│  Date: 2024-03-15                                               │
│  Status: ACCEPTED                                               │
├─────────────────────────────────────────────────────────────────┤
│  CONTEXT                                                        │
│  ───────                                                        │
│  The National Health Stack needs to process 2 million          │
│  vaccination events per day from 800,000 health workers        │
│  across 28 states. Systems consuming this data include:        │
│  Co-WIN dashboard, AEFI reporting, state health portals,       │
│  and MoHFW analytics. Direct API calls between systems         │
│  create tight coupling and cascading failure risk.             │
├─────────────────────────────────────────────────────────────────┤
│  DECISION                                                       │
│  ────────                                                       │
│  We will use Apache Kafka as the central event streaming       │
│  platform. Producers publish vaccination events to Kafka.      │
│  Each consuming system subscribes independently.               │
├─────────────────────────────────────────────────────────────────┤
│  RATIONALE                                                      │
│  ─────────                                                      │
│  • Decouples producers from consumers                          │
│  • Replay capability for state systems that go offline         │
│  • 7-day retention allows late-joining consumers               │
│  • Proven at 10M+ events/day in production (LinkedIn, CRED)   │
├─────────────────────────────────────────────────────────────────┤
│  CONSEQUENCES                                                   │
│  ────────────                                                   │
│  POSITIVE:                                                      │
│  • Any new consumer can subscribe without producer changes     │
│  • Operational replay capability during state portal outages   │
│                                                                 │
│  NEGATIVE:                                                      │
│  • Operations team needs Kafka expertise (training required)   │
│  • Additional infrastructure cost: ~₹8–12L/year for managed   │
│    Kafka on cloud for this volume                              │
│  • Message ordering guaranteed only per partition              │
├─────────────────────────────────────────────────────────────────┤
│  ALTERNATIVES CONSIDERED                                        │
│  ───────────────────────                                        │
│  RabbitMQ — Rejected: No log retention/replay, lower          │
│             throughput ceiling                                 │
│  REST APIs — Rejected: Tight coupling, no fan-out without      │
│              duplicated calls                                  │
│  AWS SNS/SQS — Rejected: Vendor lock-in, data residency       │
│                concerns for MoHFW                              │
└─────────────────────────────────────────────────────────────────┘
```

**Traceability: Business Requirement → Design → Code**

```
Business Requirement
        │
        │  "All vaccination events must be available to all
        │   state health portals within 60 seconds of recording"
        │
        ▼
Architectural Decision (ADR-007)
        │
        │  Use Kafka with 60-second SLO on consumer lag
        │  Monitoring: Consumer group lag alert at 50 seconds
        │
        ▼
Design Artifact
        │
        │  VaccinationEventProducer → kafka topic: vacc.events.v1
        │  Consumer Group: state-portal-consumer
        │  Partitioning: by state_code (28 partitions)
        │
        ▼
Code Implementation
        │
        │  @KafkaProducer(topic = "vacc.events.v1")
        │  VaccinationEventProducerService.java
        │
        ▼
Test Coverage
        │
        │  VaccinationEventIntegrationTest.java
        │  Verifies: event reaches consumer within 60 seconds
        │  under 10,000 events/minute load
        ▼
```

---

### 1.1.5 Tech Stack Selection Framework & TCO Analysis

Choosing a technology is not a preference exercise. It is an engineering decision with financial, operational, and long-term consequences.

**The SCORE Framework for Tech Stack Selection:**

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SCORE FRAMEWORK                                   │
│                                                                      │
│  S — Suitability       Does it solve the actual problem?            │
│  C — Community         Is it mature? Active maintenance?            │
│  O — Operability       Can our ops team manage it in production?    │
│  R — Regulatory Fit    Does it meet data residency/compliance?      │
│  E — Economics (TCO)   What is the full 5-year cost?               │
└──────────────────────────────────────────────────────────────────────┘
```

**TCO Analysis Example — Choosing between Self-Hosted Elasticsearch vs OpenSearch on AWS**

*Context: Singapore Government Agency building a legal case search system with 50TB of documents*

```
┌──────────────────────────────────────────────────────────────────────┐
│  TCO COMPARISON (5-Year Estimate, SGD)                               │
│                                                                      │
│  COST COMPONENT          Self-Hosted         AWS OpenSearch          │
│  ─────────────────────   ─────────────────   ──────────────────      │
│  Infrastructure          $180,000            $420,000                │
│  (Servers/VMs)                                                       │
│                                                                      │
│  Licensing               $0 (OSS)            $0 (OSS + AWS markup)  │
│                                                                      │
│  Operations Staff        $650,000            $120,000                │
│  (1.5 FTE × 5 yrs)      (dedicated SRE)     (shared SRE time)       │
│                                                                      │
│  Patching & Upgrades     $80,000             Included in service     │
│                                                                      │
│  Downtime Risk           HIGH                LOW                     │
│  (no managed failover)   (est. $40,000       (SLA-backed)           │
│                           cost impact)                               │
│                                                                      │
│  Data Egress             $0                  $15,000                 │
│                                                                      │
│  TOTAL 5-YEAR TCO        ~$950,000           ~$555,000               │
│                                                                      │
│  DECISION: AWS OpenSearch wins on TCO despite higher                 │
│  infrastructure cost, due to ops savings.                            │
│                                                                      │
│  CAVEAT: Data residency review required. AWS Singapore region        │
│  used. Data classification assessment mandatory for legal docs.      │
└──────────────────────────────────────────────────────────────────────┘
```

> **Important Note on Figures:** TCO figures above are illustrative order-of-magnitude estimates based on publicly available AWS pricing and typical Singapore government procurement patterns. Actual costs depend on reserved instance commitments, data volumes, and agency-specific negotiated rates. Always conduct your own TCO analysis using the GovTech-approved cloud calculator or your organisation's FinOps tooling.

---

## Chapter 1.2 — API-First, Domain-Driven Design & System Design

### 1.2.1 Domain-Driven Design — Why It Matters at Government Scale

Government systems are complex not primarily because of technical reasons, but because of domain complexity. A land registration system in India involves the Revenue Department, Sub-Registrar's Office, Banks for mortgage, Municipal Corporation for tax linkage, and the ULPIN (Unique Land Parcel Identification Number) system. Each of these has its own language, its own rules, and its own understanding of what "property" means.

DDD gives us tools to manage this complexity without letting it collapse into a single enormous ball of tangled code.

**The Three Strategic DDD Patterns:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DDD STRATEGIC PATTERNS                               │
│                                                                         │
│  1. BOUNDED CONTEXT                                                     │
│  ──────────────────                                                     │
│  A conceptual boundary within which a domain model is consistent.       │
│  Outside this boundary, the same term may mean something different.     │
│                                                                         │
│  EXAMPLE: National Land Records System                                  │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
│  │  Registration   │  │    Revenue &    │  │   Mortgage &    │        │
│  │    Context      │  │   Taxation      │  │   Banking       │        │
│  │                 │  │    Context      │  │    Context      │        │
│  │  "Property" =   │  │  "Property" =   │  │  "Property" =   │        │
│  │  Legal parcel   │  │  Tax unit with  │  │  Collateral     │        │
│  │  with owner &   │  │  assessed       │  │  asset with     │        │
│  │  survey number  │  │  market value   │  │  encumbrance    │        │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘        │
│           │                    │                    │                  │
│           └────────────────────┴────────────────────┘                  │
│                    Integration via Context Map                          │
│                    (API contracts, not shared DB)                       │
│                                                                         │
│  2. UBIQUITOUS LANGUAGE                                                 │
│  ──────────────────────                                                 │
│  Use the same word in conversation, documentation, and code.           │
│                                                                         │
│  WRONG:  Code says `PropertyRegistrationRequest`                       │
│          DM calls it `DeedSubmission`                                  │
│          UI shows `Application Form`                                    │
│          Business says `Deed Registration`                             │
│                                                                         │
│  RIGHT:  All layers use `DeedRegistration` consistently.               │
│                                                                         │
│  3. CONTEXT MAPPING                                                     │
│  ─────────────────                                                      │
│  Defines the relationship between bounded contexts.                    │
│  Patterns: Shared Kernel, Customer-Supplier,                           │
│            Anticorruption Layer, Open Host Service                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Aggregate Design — The Most Misunderstood DDD Concept**

```
┌─────────────────────────────────────────────────────────────────────────┐
│              AGGREGATE DESIGN: RATION CARD MANAGEMENT                  │
│              (Like Tamil Nadu's TNPDS system)                          │
│                                                                         │
│  WRONG DESIGN (Anemic Model):                                          │
│  ─────────────────────────────                                         │
│  RationCardService.addMember(cardId, member)   ← Business logic in     │
│  RationCardService.updateAllotment(cardId)         service layer,      │
│  RationCardService.suspend(cardId)                 not in entities     │
│                                                                         │
│  CORRECT DESIGN (Rich Aggregate):                                      │
│  ──────────────────────────────                                        │
│                                                                         │
│         ┌──────────────────────────────┐                               │
│         │  RationCard (Aggregate Root) │                               │
│         │  ─────────────────────────── │                               │
│         │  cardNumber: CardNumber      │ ← Value Object               │
│         │  status: CardStatus          │                               │
│         │  headOfFamily: Member        │                               │
│         │                              │                               │
│         │  + addMember(member)         │ ← Business rules enforced    │
│         │    [max 8 members allowed]   │   inside the aggregate       │
│         │  + updateAllotment(month)    │                               │
│         │  + suspend(reason)           │                               │
│         │  + reinstate(order)          │                               │
│         └───────────┬──────────────────┘                               │
│                     │ contains                                          │
│          ┌──────────┴───────────┐                                      │
│          │                      │                                       │
│  ┌───────┴──────┐    ┌──────────┴───────┐                              │
│  │   Member     │    │ MonthlyAllotment  │                              │
│  │  (Entity)    │    │    (Entity)       │                              │
│  │              │    │                   │                              │
│  │ aadhaarHash  │    │ month: YearMonth  │                              │
│  │ name         │    │ riceKg: Weight    │                              │
│  │ relationship │    │ wheatKg: Weight   │                              │
│  └──────────────┘    └───────────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 1.2.2 Hexagonal Architecture & Clean Architecture

**The Problem This Solves**

In a typical layered architecture, your business logic often depends directly on frameworks, databases, and HTTP. When the government mandates moving from Oracle to PostgreSQL, or from on-premise to cloud, your entire codebase is affected because the business logic is tangled with infrastructure.

Hexagonal architecture prevents this.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HEXAGONAL ARCHITECTURE                               │
│                    (Ports and Adapters)                                 │
│                                                                         │
│                                                                         │
│   ┌──────────┐          ┌──────────────────────────────┐               │
│   │  REST    │──────────►                              │               │
│   │  Client  │  (Port)  │                              │               │
│   └──────────┘          │    DOMAIN CORE               │               │
│                          │    (Pure Business Logic)     │               │
│   ┌──────────┐          │                              │               │
│   │  Kafka   │──────────►    GrievanceService           │               │
│   │ Consumer │  (Port)  │    GrievancePolicy            │               │
│   └──────────┘          │    GrievanceRepository        │               │
│                          │    (interface, not impl)     │               │
│   ┌──────────┐          │                              │               │
│   │  Batch   │──────────►                              │               │
│   │  Job     │  (Port)  │                              │               │
│   └──────────┘          └──────────────┬───────────────┘               │
│                                        │ (Port)                         │
│                         ┌──────────────┴───────────────┐               │
│                         │                              │               │
│               ┌─────────┴──────┐          ┌───────────┴──────┐        │
│               │  PostgreSQL    │          │   Redis Cache    │        │
│               │  Adapter       │          │   Adapter        │        │
│               │                │          │                  │        │
│               │  Implements    │          │  Implements      │        │
│               │  Grievance     │          │  Grievance       │        │
│               │  Repository    │          │  Repository      │        │
│               └────────────────┘          └──────────────────┘        │
│                                                                         │
│  KEY PRINCIPLE:                                                         │
│  The Domain Core has ZERO imports from frameworks, databases,          │
│  or HTTP libraries. It depends only on interfaces (ports).             │
│  Adapters implement those interfaces.                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Scenario Where This Saved a Project**

> The Integrated Grievance Redressal System (IGRS) for a State Government was initially built with direct JPA/Hibernate calls embedded in service classes. When the government mandated migration from Oracle to a NIC-hosted PostgreSQL cluster due to licensing cost reduction, the team discovered that SQL hints, Oracle-specific date functions, and CONNECT BY queries were scattered across 200+ service methods. A migration that should have taken 2 months took 9 months and required touching 340 files. Had hexagonal architecture been used, only the repository adapter layer (approximately 15 classes) would have required changes.

---

### 1.2.3 API-First Design — OpenAPI & AsyncAPI

**API-First means the API contract is defined and agreed upon before implementation begins.**

This is critical in government systems where:
- Multiple agencies consume your API
- Different teams build different services simultaneously
- The API must be versioned and backward-compatible
- Procurement and integration contracts reference the API specification

**OpenAPI (REST APIs) — Minimal Example for a Government Context**

```yaml
# National Health Stack — Patient Registration API
# OpenAPI 3.1 Specification

openapi: 3.1.0
info:
  title: Patient Registration Service
  version: v2.1.0
  description: |
    Registers new patients in the Ayushman Bharat Digital Mission (ABDM) 
    Health ID system. Compliant with ABDM API specifications v2.0.

paths:
  /patients:
    post:
      operationId: registerPatient
      summary: Register a new patient and generate ABHA ID
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PatientRegistrationRequest'
      responses:
        '201':
          description: Patient registered successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PatientRegistrationResponse'
        '409':
          description: Patient already registered (Aadhaar linked)
        '422':
          description: Validation failure (invalid Aadhaar or mobile)

components:
  schemas:
    PatientRegistrationRequest:
      type: object
      required: [aadhaarHash, mobile, name, dateOfBirth, gender]
      properties:
        aadhaarHash:
          type: string
          description: SHA-256 hash of Aadhaar number. Never send raw Aadhaar.
          example: "a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3"
        mobile:
          type: string
          pattern: '^[6-9]\d{9}$'
        name:
          type: string
          maxLength: 100
        dateOfBirth:
          type: string
          format: date
        gender:
          type: string
          enum: [M, F, O]
```

**API Versioning Strategies:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API VERSIONING STRATEGIES                            │
│                                                                         │
│  STRATEGY 1: URL Path Versioning (Most Common in Gov)                  │
│  ─────────────────────────────────────────────────────                 │
│  GET /api/v1/citizens/{id}                                              │
│  GET /api/v2/citizens/{id}   ← New version with additional fields      │
│                                                                         │
│  PRO: Explicit, easy to understand, easy to route via gateway          │
│  CON: URL changes break bookmarks, violates REST purity                │
│                                                                         │
│  STRATEGY 2: Header Versioning                                          │
│  ─────────────────────────────                                         │
│  GET /api/citizens/{id}                                                 │
│  Accept-Version: v2                                                     │
│                                                                         │
│  PRO: Clean URL, same resource path                                    │
│  CON: Less visible, harder to test in browser                          │
│                                                                         │
│  STRATEGY 3: Content Negotiation                                        │
│  ─────────────────────────────                                         │
│  Accept: application/vnd.gov.health.v2+json                            │
│                                                                         │
│  PRO: RESTfully correct                                                 │
│  CON: Complex for government integrators unfamiliar with HTTP headers  │
│                                                                         │
│  GOVERNMENT RECOMMENDATION:                                             │
│  Use URL Path Versioning for public/inter-agency APIs.                 │
│  Maintain v-1 for minimum 12 months after v-new release.              │
│  Deprecation notice minimum 6 months prior via API Gateway.           │
└─────────────────────────────────────────────────────────────────────────┘
```

**AsyncAPI — For Event-Driven Systems**

AsyncAPI is to message-driven systems what OpenAPI is to REST. It documents Kafka topics, message schemas, and consumer/producer relationships.

```yaml
# AsyncAPI 2.6 — Vaccination Event Stream
asyncapi: 2.6.0
info:
  title: Co-WIN Vaccination Event Stream
  version: 1.3.0

channels:
  vaccination/events/beneficiary-vaccinated:
    description: Published when a beneficiary receives a vaccine dose
    publish:
      operationId: onBeneficiaryVaccinated
      message:
        $ref: '#/components/messages/VaccinationEvent'

components:
  messages:
    VaccinationEvent:
      payload:
        type: object
        required: [beneficiaryId, vaccineCode, doseNumber, vaccinatedAt, facilityId]
        properties:
          beneficiaryId:
            type: string
            description: Beneficiary Reference ID from Co-WIN
          vaccineCode:
            type: string
            enum: [COVISHIELD, COVAXIN, SPUTNIK_V, CORBEVAX, COVOVAX]
          doseNumber:
            type: integer
            minimum: 1
            maximum: 4
          vaccinatedAt:
            type: string
            format: date-time
          facilityId:
            type: string
            description: Health facility code from NHA facility registry
```

---

## Chapter 1.3 — Distributed Systems & Event-Driven Architecture

### 1.3.1 Microservices vs SOA vs Space-Based Architecture

Before choosing an architecture style, you must understand what problem each one solves.

```
┌─────────────────────────────────────────────────────────────────────────┐
│           ARCHITECTURE STYLE COMPARISON                                 │
│                                                                         │
│  MONOLITH                                                               │
│  ────────                                                               │
│  Everything in one deployable unit.                                    │
│  ┌─────────────────────────────────────┐                               │
│  │  UI + Business Logic + Data Access  │                               │
│  │  All in one WAR/JAR/DLL             │                               │
│  └─────────────────────────────────────┘                               │
│                                                                         │
│  WHEN IT WORKS: Small team, well-defined domain, startup phase         │
│  BREAKS AT: Different release cadences, team size > 12, scale needs   │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  SOA (Service-Oriented Architecture)                                    │
│  ───────────────────────────────────                                   │
│  Coarse-grained services connected via Enterprise Service Bus (ESB)   │
│                                                                         │
│  ┌───────────┐     ┌─────────────┐     ┌───────────────┐              │
│  │  Service  │────►│     ESB     │◄────│   Service     │              │
│  │  (Large)  │     │ (MuleSoft / │     │   (Large)     │              │
│  └───────────┘     │  WSO2)      │     └───────────────┘              │
│                     └─────────────┘                                    │
│                                                                         │
│  WHEN IT WORKS: Large enterprises with heterogeneous legacy systems    │
│  BREAKS AT: ESB becomes bottleneck, high coupling through shared bus  │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  MICROSERVICES                                                          │
│  ─────────────                                                          │
│  Fine-grained, independently deployable services, each owning its DB  │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │              │
│  │  + DB    │  │  + DB    │  │  + DB    │  │  + DB    │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │              │                    │
│       └──────────────┴──────────────┴──────────────┘                  │
│              Communication via APIs or Events                           │
│                                                                         │
│  WHEN IT WORKS: Large teams, independent scaling needs, cloud-native  │
│  BREAKS AT: Network overhead, distributed transactions, ops complexity │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  SPACE-BASED ARCHITECTURE                                               │
│  ─────────────────────────                                             │
│  Data and processing co-located in an in-memory grid                  │
│  Used for extreme low-latency requirements (trading, real-time fraud)  │
│                                                                         │
│  WHEN IT WORKS: Real-time processing, millisecond latency required    │
│  USED IN: Stock exchanges, payment fraud detection, telemetry          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 1.3.2 Event-Driven Patterns — Kafka, Event Sourcing, CQRS

**Understanding Kafka's Core Concepts**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KAFKA ARCHITECTURE CONCEPTS                          │
│                                                                         │
│  TOPIC: A named stream of records (like a table in a database)         │
│  PARTITION: A topic is split into partitions for parallelism           │
│  OFFSET: A sequential ID for each message within a partition           │
│  PRODUCER: Writes messages to a topic                                   │
│  CONSUMER: Reads messages from a topic                                  │
│  CONSUMER GROUP: A group of consumers sharing the work of a topic      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────┐           │
│  │                  TOPIC: tax.returns.filed               │           │
│  │                                                         │           │
│  │  Partition 0:  [msg0]─[msg1]─[msg2]─[msg3] ───────►   │           │
│  │  Partition 1:  [msg0]─[msg1]─[msg2] ────────────────►  │           │
│  │  Partition 2:  [msg0]─[msg1]─[msg2]─[msg3]─[msg4] ─►  │           │
│  │                                                         │           │
│  └─────────────────────────────────────────────────────────┘           │
│       ▲                                                                 │
│       │ writes                        reads                            │
│  ┌────┴────┐                    ┌─────────────────────┐                │
│  │ ITR     │                    │  Consumer Group:    │                │
│  │ Filing  │                    │  tax-processing     │                │
│  │ Portal  │                    │  ┌───────────────┐  │                │
│  │         │                    │  │ Consumer-1    ├──► Partition 0   │
│  │         │                    │  │ Consumer-2    ├──► Partition 1   │
│  └─────────┘                    │  │ Consumer-3    ├──► Partition 2   │
│                                  │  └───────────────┘  │                │
│                                  └─────────────────────┘                │
│                                                                         │
│  KEY PROPERTY: Messages are retained even after consumption.           │
│  Another consumer group (e.g., fraud-detection) can independently     │
│  read the same messages from offset 0.                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

**Event Sourcing — Explained Through GST Audit Trail**

In a traditional system, when a GST return is amended, you update the existing record. You lose the history of what it was before.

In Event Sourcing, you never update. You append.

```
TRADITIONAL (Mutable State):
─────────────────────────────
DB Row: { returnId: R001, taxableAmount: 500000, status: AMENDED }
                                    ↑
                              History lost. Who changed it? When? Why?

EVENT SOURCED (Immutable Log):
──────────────────────────────
Event 1: ReturnFiled      { returnId: R001, taxableAmount: 450000, filedAt: 2024-07-31 }
Event 2: ReturnAmended    { returnId: R001, newAmount: 500000, reason: "Missed invoice", amendedAt: 2024-08-15 }
Event 3: RefundClaimed    { returnId: R001, refundAmount: 5000, claimedAt: 2024-09-01 }

Current State = Replay(Event1 + Event2 + Event3)
Audit State at any point = Replay(Events up to that timestamp)

WHY THIS MATTERS FOR GOVERNMENT:
• Complete audit trail is mandatory (CAG audit requirements)
• Any point-in-time state reconstruction possible
• Dispute resolution: "What was the state of this return on Aug 10?"
  → Replay events 1 through 1 → exact state before amendment
```

**CQRS — Command Query Responsibility Segregation**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CQRS PATTERN                                    │
│                                                                         │
│  PROBLEM: A single data model optimised for writes is often poorly     │
│  suited for reads (complex joins, aggregations, dashboard queries)      │
│                                                                         │
│  SOLUTION: Separate the write model from the read model                │
│                                                                         │
│                        ┌─────────────┐                                 │
│                        │   Client    │                                  │
│                        └──────┬──────┘                                 │
│                               │                                         │
│              ┌────────────────┴─────────────────┐                      │
│              │                                   │                      │
│              ▼                                   ▼                      │
│   ┌──────────────────┐             ┌───────────────────────┐           │
│   │ COMMAND SIDE      │            │    QUERY SIDE          │           │
│   │ (Write Model)     │            │    (Read Model)        │           │
│   │                   │            │                        │           │
│   │ Commands:         │            │ Queries:               │           │
│   │ FileReturn        │            │ GetReturnSummary       │           │
│   │ AmendReturn       │            │ GetTaxpayerDashboard   │           │
│   │ ClaimRefund       │            │ GetAuditReport         │           │
│   │                   │            │                        │           │
│   │ DB: PostgreSQL    │            │ DB: Elasticsearch /    │           │
│   │ (ACID, normalised)│            │     MongoDB            │           │
│   │ Optimised for     │            │ Denormalised, fast     │           │
│   │ transactional     │            │ read-optimised         │           │
│   │ consistency       │            │ views                  │           │
│   └─────────┬─────────┘            └───────────────────────┘           │
│             │                                 ▲                         │
│             │   Events published              │                         │
│             └──────────────────────────────────┘                        │
│                  (Sync via Kafka / Change Data Capture)                 │
│                                                                         │
│  REAL USE CASE:                                                         │
│  Income Tax e-Filing system: Filing (write) uses ACID PostgreSQL.      │
│  ITR dashboard showing aggregate data for 8 crore+ taxpayers uses      │
│  Elasticsearch read model updated asynchronously.                      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 1.3.3 Reactive Patterns & Workflow Engines

**Non-Blocking I/O — Why It Matters**

In a traditional blocking model, a thread waits for a database query to complete before doing anything else. Under high load, you run out of threads.

In a non-blocking model, the thread registers a callback and is freed to serve other requests while the database responds. This allows serving significantly more concurrent requests with the same hardware.

```
BLOCKING (Traditional):
────────────────────────
Thread-1: Request ──► Database query ──────────────────────► Response
          (Thread blocked waiting for DB, doing nothing useful)

Thread-2: Request ──────────────────────────────────────────► (Waiting for Thread-1 to free)

NON-BLOCKING (Reactive):
──────────────────────────
Thread-1: Request ──► Database query registered ──► free to handle next request
                              │
                     DB responds (callback)
                              │
Thread-1 (or any free thread): ──► Process result ──► Response

RESULT: Same thread count, 5–10x more concurrent requests
FRAMEWORKS: Spring WebFlux (Java), FastAPI async (Python), Node.js
```

**Workflow Engines — Camunda & Temporal**

Government processes involve long-running, multi-step, multi-actor workflows. A building permit approval can take 3–6 weeks, involving 5+ departments. You cannot keep this in memory or in a simple state machine in your database.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  WITHOUT A WORKFLOW ENGINE:                                             │
│  ──────────────────────────                                             │
│  Status stored in DB: { permitId: P001, status: "PENDING_FIRE_NOC" }  │
│                                                                         │
│  Problems:                                                              │
│  • Who triggered the next step? Manual follow-up required.             │
│  • If a step times out, who is notified?                               │
│  • How do you show the permit applicant where their application is?    │
│  • Compensation if step-3 fails after step-1 and step-2 succeeded?    │
│                                                                         │
│  WITH A WORKFLOW ENGINE (Camunda/Temporal):                            │
│  ──────────────────────────────────────────                            │
│                                                                         │
│  Building Permit Process (BPMN):                                        │
│                                                                         │
│  [Submit Application] ──► [Structural Check] ──► [Fire NOC Check]     │
│         │                        │                       │             │
│         │               [Reject if failed]      [Auto-escalate         │
│         │                                         after 7 days]        │
│         │                                                │             │
│         └────────────────────────────────────► [Issue Permit]          │
│                                                                         │
│  BENEFITS:                                                              │
│  • Visual process definition (BPMN diagrams)                           │
│  • Automatic retries, timeouts, escalations                            │
│  • Full audit trail of every state transition                          │
│  • Human task management built-in                                      │
│  • Works across multiple services (durable execution)                  │
│                                                                         │
│  REAL CASE: Tamil Nadu's TNSMART (single-window clearance system)      │
│  uses workflow orchestration for permit approvals across PWD,          │
│  Fire, Electricity, and Revenue departments.                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 1.4 — Data Architecture, NoSQL & Search

### 1.4.1 Polyglot Persistence — Using the Right Database for the Right Job

```
┌─────────────────────────────────────────────────────────────────────────┐
│              POLYGLOT PERSISTENCE REFERENCE MAP                         │
│                                                                         │
│  DATABASE TYPE     EXAMPLES           BEST FOR              AVOID FOR   │
│  ─────────────     ────────           ────────              ─────────   │
│  Relational        PostgreSQL         Transactions,         Write-heavy  │
│  (RDBMS)           MySQL, Oracle      complex queries,      time-series  │
│                                       reporting             data         │
│                                                                         │
│  Document          MongoDB,           Semi-structured       Complex      │
│                    CouchDB            data, flexible        joins        │
│                                       schema, catalogs                  │
│                                                                         │
│  Key-Value         Redis, DynamoDB    Caching, sessions,    Complex      │
│                                       rate limiting,        queries      │
│                                       leaderboards                      │
│                                                                         │
│  Graph             Neo4j,             Relationship          High-volume  │
│                    Amazon Neptune     traversal:            write OLTP   │
│                                       fraud rings,                      │
│                                       org hierarchies                   │
│                                                                         │
│  Time-Series       InfluxDB,          Metrics, sensor       General      │
│                    TimescaleDB        data, IoT streams     purpose data │
│                                                                         │
│  Search            Elasticsearch,     Full-text search,     Primary      │
│                    OpenSearch,        faceted filtering,    transactional│
│                    Apache Solr        log analytics         store        │
│                                                                         │
│  Wide-Column       Apache Cassandra,  High write           Complex       │
│                    HBase, DynamoDB    throughput, time-    queries,      │
│                                       ordered data          joins        │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Government System Architecture — Singapore's CODEX (Government Data Exchange)**

```
                    CODEX-Style Data Architecture
                    ─────────────────────────────

  Citizen-Facing Portal
          │
          ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  API Gateway (Kong / AWS API GW)            │
  └────────────────────────────┬────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
  │  Citizen      │    │  Transaction  │    │  Search &     │
  │  Profile      │    │  Processing   │    │  Discovery    │
  │  Service      │    │  Service      │    │  Service      │
  │               │    │               │    │               │
  │  DB: MongoDB  │    │  DB:          │    │  DB:          │
  │  (flexible    │    │  PostgreSQL   │    │  Elasticsearch│
  │  citizen      │    │  (ACID for    │    │  (fast search │
  │  attributes   │    │  financial    │    │  across all   │
  │  vary by      │    │  transactions)│    │  services)    │
  │  service type)│    │               │    │               │
  └───────────────┘    └───────────────┘    └───────────────┘
          │                    │                    ▲
          │                    │                    │
          ▼                    ▼                    │
  ┌───────────────┐    ┌───────────────┐           │
  │  Session &    │    │  Event Log    │ ──────────►│
  │  Cache        │    │  (Kafka)      │  (CDC sync  │
  │               │    │               │   to Search)│
  │  DB: Redis    │    │               │            │
  │  (fast read   │    │               │            │
  │  for active   │    │               │            │
  │  sessions)    │    │               │            │
  └───────────────┘    └───────────────┘            │
```

---

### 1.4.2 Sharding, Partitioning & Geo-Unit Design

**Sharding** is horizontal partitioning — distributing data across multiple database nodes so no single node holds all data.

**When You Need Sharding**

- A single database node cannot hold all the data or handle all the writes
- Write throughput exceeds single-node capacity
- You need geographic data isolation for regulatory compliance

**Government Sharding Example — Election Commission of India Voter Registry**

> India's voter registry has approximately 97 crore (970 million) registered voters. Sharding by **state code + constituency code** is natural:
>
> - Shard 1: All voters in Tamil Nadu (approx. 6.2 crore)
> - Shard 2: All voters in Maharashtra (approx. 9.5 crore)
> - Shard N: Each state/UT gets its own shard
>
> This design ensures:
> - Writes during voter addition/deletion are localised (Tamil Nadu ECI doesn't write to Maharashtra's shard)
> - Queries are localised (booth officer queries only their constituency shard)
> - Regulatory isolation — state-level data stays within the state's infrastructure zone
>
> **Challenge:** Cross-shard queries (national aggregate reports) require a separate analytics layer (typically read replicas feeding into a data warehouse, not the OLTP shards themselves).

**Consistency Models in Distributed Systems:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  CONSISTENCY MODELS EXPLAINED                           │
│                                                                         │
│  STRONG CONSISTENCY                                                      │
│  ───────────────────                                                    │
│  After a write completes, every subsequent read sees that write.       │
│  No stale reads possible.                                               │
│                                                                         │
│  Example: Bank debit. After ₹10,000 is debited, every read of          │
│  the balance must show the reduced amount. No exceptions.              │
│                                                                         │
│  Cost: Higher latency, limited throughput, reduced availability         │
│  Implementation: Single-master DB, 2-phase commit, Paxos/Raft          │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  EVENTUAL CONSISTENCY                                                    │
│  ─────────────────────                                                  │
│  After a write, replicas will eventually converge to the same value.   │
│  Reads may temporarily return stale data.                              │
│                                                                         │
│  Example: Citizen address update on Aadhaar. After updating address,  │
│  different government portals that cache citizen data may show         │
│  the old address for up to a few minutes. Acceptable because          │
│  the eventual state will converge and the window is bounded.          │
│                                                                         │
│  Cost: Developers must handle stale reads, conflict resolution         │
│  Benefit: High availability, low latency, partition tolerant           │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  CAUSAL CONSISTENCY (Middle Ground)                                     │
│  ──────────────────────────────────                                    │
│  Operations that are causally related are seen in order.               │
│  Unrelated operations may be seen out of order.                        │
│                                                                         │
│  Example: Comment posted, then reply to that comment. Any user         │
│  who sees the reply must also have seen the original comment.          │
│  Used in: MongoDB (causally consistent sessions)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 1.5 — Security, Zero Trust & Emerging Technologies

### 1.5.1 Zero Trust Architecture

Zero Trust is not a product you buy. It is a security philosophy and a set of design principles.

**The Old Model (Perimeter Security):**
> "Everything inside the corporate network is trusted. Everything outside is untrusted."
> This model broke when employees started working from home, services moved to cloud, and insiders became a threat vector.

**Zero Trust Principles:**
> "Never trust, always verify. Verify explicitly. Use least privilege. Assume breach."

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST ARCHITECTURE LAYERS                       │
│                                                                         │
│  REQUEST FLOW through Zero Trust:                                       │
│                                                                         │
│  User / Device                                                          │
│      │                                                                  │
│      ▼                                                                  │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  LAYER 1: IDENTITY VERIFICATION                     │               │
│  │  • Multi-Factor Authentication (MFA)               │               │
│  │  • Government employee certificate (PKI)           │               │
│  │  • Keycloak / Azure AD B2C token issuance          │               │
│  └────────────────────────┬────────────────────────────┘               │
│                           │ (Token issued)                              │
│                           ▼                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  LAYER 2: DEVICE VERIFICATION                       │               │
│  │  • Is the device registered? (MDM check)           │               │
│  │  • Is the device OS patched?                       │               │
│  │  • Is the device certificate valid?                │               │
│  └────────────────────────┬────────────────────────────┘               │
│                           │                                             │
│                           ▼                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  LAYER 3: NETWORK LAYER                             │               │
│  │  • Micro-segmentation (VPC, security groups)       │               │
│  │  • No implicit internal trust                      │               │
│  │  • Service-to-service: mTLS mandatory              │               │
│  └────────────────────────┬────────────────────────────┘               │
│                           │                                             │
│                           ▼                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  LAYER 4: APPLICATION LAYER                         │               │
│  │  • API Gateway: JWT validation on every request    │               │
│  │  • RBAC: Role checked against requested resource   │               │
│  │  • Attribute-Based Access Control (ABAC) for       │               │
│  │    fine-grained decisions                          │               │
│  └────────────────────────┬────────────────────────────┘               │
│                           │                                             │
│                           ▼                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  LAYER 5: DATA LAYER                               │               │
│  │  • Encryption at rest (AES-256)                    │               │
│  │  • Field-level encryption for PII                  │               │
│  │  • Database activity monitoring and alerting       │               │
│  └─────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Incident — SingHealth Data Breach (Singapore, 2018)**

> The SingHealth data breach is one of the most significant cybersecurity incidents in Singapore's public sector history. 1.5 million patient records (including Prime Minister Lee Hsien Loong's records) were exfiltrated between June and July 2018.
>
> **Root Cause Factors (as per the COI Report):**
> - Weak passwords on a Citrix server used as an entry point
> - Excessive network connectivity — once inside, the attacker moved laterally across the network with minimal controls
> - The connected SCM (Structured Query Machine) network was not adequately segmented
> - No MFA on critical systems
> - Insufficient monitoring and alerting
>
> **What Zero Trust Would Have Prevented:**
> - mTLS between services: Lateral movement would require valid certificates at each hop
> - Micro-segmentation: Breach of one network zone would not grant access to the patient records zone
> - Continuous authentication: Re-authentication required for accessing sensitive data, not just at login
>
> **Outcome:** SGD 1 million fine on IHiS and SGD 250,000 on SingHealth. Sweeping policy changes across Singapore government IT including mandatory MFA and network segmentation requirements now embedded in IM8 (Instruction Manual 8 — Singapore Government IT security standard).

---

### 1.5.2 API Gateway Security, Keycloak, mTLS & Secret Management

**How Authentication Works in a Multi-Service Government System:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│         OAUTH 2.0 + OIDC FLOW IN GOVERNMENT CONTEXT                    │
│         (Using Keycloak as the Identity Provider)                      │
│                                                                         │
│  1. Citizen opens eBiz portal (Singapore) / Udyam portal (India)       │
│                                                                         │
│  2. Portal redirects to Keycloak login page                            │
│     GET https://keycloak.gov.in/auth/realms/ndh/protocol/openid-connect│
│                                                                         │
│  3. Citizen enters credentials. Keycloak validates. Issues tokens:     │
│     • Access Token (JWT, short-lived, 15 min) — for API calls         │
│     • Refresh Token (longer-lived, 8 hours) — for getting new access  │
│     • ID Token — contains citizen identity claims                      │
│                                                                         │
│  4. Portal calls Service A with the Access Token:                      │
│     Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...                     │
│                                                                         │
│  5. API Gateway validates the JWT:                                      │
│     • Signature verification using Keycloak's public key               │
│     • Expiry check                                                      │
│     • Scope/role check against the requested endpoint                  │
│                                                                         │
│  6. If valid, gateway forwards to Service A                            │
│     Service A trusts the token — it does NOT call Keycloak again      │
│     (This is the efficiency of JWT over session tokens)                │
│                                                                         │
│  JWT ANATOMY:                                                           │
│  ┌─────────────┬──────────────────────────────┬────────────────┐       │
│  │   HEADER    │          PAYLOAD              │   SIGNATURE    │       │
│  │             │                              │                │       │
│  │ alg: RS256  │ sub: citizen-id-12345        │ Verifiable     │       │
│  │ typ: JWT    │ roles: [taxpayer, employer]  │ using         │       │
│  │             │ exp: 1717987200              │ Keycloak's    │       │
│  │             │ iss: keycloak.gov.in         │ public key    │       │
│  └─────────────┴──────────────────────────────┴────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

**mTLS — Mutual TLS for Service-to-Service Security**

In regular TLS, only the server presents a certificate (you trust the website). In mTLS, both the server AND the client present certificates. This means a rogue service cannot connect even if it's inside the network.

```
REGULAR TLS:
  Client ──────────────────────────────────► Server
         "Are you the real Tax Service?"
         Server presents certificate → trusted
         Client sends request

mTLS:
  Client ──────────────────────────────────► Server
         "Are you the real Tax Service?"
         Server presents certificate
         "Are YOU the real Portal Service?"
         Client presents certificate
         Both verified → connection established

  Implication: Even if an attacker injects a service inside your 
  Kubernetes cluster, it cannot call other services without a 
  valid certificate issued by your Certificate Authority.
```

**Secret Management — What NOT to Do**

```
# NEVER DO THIS (seen in real government project code)
database.password = Admin@123
api.key = sk-prod-xKj9mN2pL8...

# In application.properties committed to GitLab
spring.datasource.password=GovProd@2023

# The above is not hypothetical. Government project codebases have 
# been found with production credentials in public repositories.
# This has led to real data exposures.

# CORRECT APPROACH: Use a Secret Manager
# Vault (HashiCorp), AWS Secrets Manager, Azure Key Vault

# Application retrieves secrets at runtime:
SECRET = vault_client.read("secret/prod/database")["password"]
```

---

### 1.5.3 IoT, Blockchain & Edge Computing Integration Patterns

**IoT in Government Context — Smart City Infrastructure**

```
┌─────────────────────────────────────────────────────────────────────────┐
│          IOT ARCHITECTURE: TRAFFIC MANAGEMENT SYSTEM                   │
│          (Modelled on Bengaluru/Singapore Smart Traffic)               │
│                                                                         │
│  EDGE LAYER (at traffic junction)                                       │
│  ─────────────────────────────────                                     │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │
│  │ Traffic Camera │  │ Loop Detector  │  │  Air Quality   │           │
│  │ (ANPR + Count) │  │ (Vehicle Count)│  │  Sensor        │           │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘           │
│          └────────────────────┴────────────────────┘                   │
│                               │                                         │
│                    ┌──────────┴──────────┐                             │
│                    │  EDGE GATEWAY       │                             │
│                    │  (Raspberry Pi /    │                             │
│                    │   Industrial IoT GW)│                             │
│                    │                     │                             │
│                    │  Local Processing:  │                             │
│                    │  • Filter noise     │                             │
│                    │  • Aggregate 1min   │                             │
│                    │  • Alert on anomaly │                             │
│                    └──────────┬──────────┘                             │
│                               │ (MQTT / AMQP)                          │
│                    ┌──────────▼──────────┐                             │
│                    │  CLOUD LAYER        │                             │
│                    │  Kafka Topic:       │                             │
│                    │  traffic.events     │                             │
│                    └──────────┬──────────┘                             │
│                               │                                         │
│          ┌────────────────────┼───────────────────┐                   │
│          ▼                    ▼                   ▼                   │
│  ┌───────────────┐  ┌────────────────┐  ┌───────────────┐            │
│  │  Signal       │  │  Traffic       │  │  City-wide    │            │
│  │  Control      │  │  Analytics     │  │  Dashboard    │            │
│  │  Service      │  │  (ML models)   │  │  (Kibana)     │            │
│  │  (real-time)  │  │  (batch)       │  │  (monitoring) │            │
│  └───────────────┘  └────────────────┘  └───────────────┘            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Blockchain in Government — Where It Genuinely Adds Value**

Blockchain is frequently misapplied. Before using it, ask: "Does this problem need an immutable, distributed, multi-party shared ledger?"

```
┌─────────────────────────────────────────────────────────────────────────┐
│         BLOCKCHAIN: GENUINE USE CASES vs MISAPPLICATION                 │
│                                                                         │
│  GENUINE USE CASES IN GOVERNMENT:                                       │
│  ─────────────────────────────────                                     │
│                                                                         │
│  1. Land Registry (Andhra Pradesh blockchain pilot)                    │
│     Multiple parties: Sub-Registrar, Banks, Revenue Dept               │
│     Problem: Fraudulent double-registration, record tampering           │
│     Value: Immutable record, no single point of tampering              │
│                                                                         │
│  2. Supply Chain for Pharmaceutical Verification                       │
│     Multiple parties: Manufacturer, Distributor, Hospital, CDSCO      │
│     Problem: Counterfeit drugs, parallel grey market                   │
│     Value: Provenance tracking, tamper-evident                         │
│                                                                         │
│  3. Cross-Border Trade Documents (Singapore Tradenet evolution)        │
│     Multiple parties: Exporter, Customs, Bank, Insurer, Port           │
│     Problem: Document reconciliation across sovereign entities          │
│     Value: Single shared truth, no need to trust counterparty          │
│                                                                         │
│  MISAPPLICATION (Use a regular database instead):                      │
│  ─────────────────────────────────────────────────                    │
│  • Single ministry tracking internal approvals                         │
│  • Application that only one agency writes to                          │
│  • Use case where existing trusted authority can maintain the record   │
│  • Use case needing speed (blockchain is slow — 15-45 TPS for         │
│    Ethereum, versus 50,000+ TPS for PostgreSQL)                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 1.6 — Project Milestone 1 Preview: Architecture Blueprint

By Day 6, you will produce:

1. **Architecture Blueprint** — A diagram showing the system components, their interactions, and the technology choices
2. **ADRs** — Minimum 3 Architecture Decision Records for your system's key choices
3. **NFR Matrix** — Quantified non-functional requirements with justification
4. **Traceability Matrix** — Mapping from business requirements to design decisions to technology choices

**What a Good Architecture Blueprint Looks Like**

```
┌─────────────────────────────────────────────────────────────────────────┐
│           ARCHITECTURE BLUEPRINT COMPONENTS                             │
│                                                                         │
│  CONTEXT DIAGRAM (C4 Level 1)                                          │
│  Shows the system in relation to its users and external systems        │
│                                                                         │
│  CONTAINER DIAGRAM (C4 Level 2)                                        │
│  Shows the deployable units (services, databases, message queues)      │
│                                                                         │
│  COMPONENT DIAGRAM (C4 Level 3) — for critical services only          │
│  Shows the internal structure of key services                          │
│                                                                         │
│  INFRASTRUCTURE DIAGRAM                                                 │
│  Shows cloud/on-prem deployment topology, availability zones           │
│                                                                         │
│  DATA FLOW DIAGRAM                                                      │
│  Shows how data moves through the system and where it rests            │
│                                                                         │
│  SECURITY ARCHITECTURE DIAGRAM                                          │
│  Shows trust boundaries, authentication points, encryption layers      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Part 1 — Terms to Google: Research Guide

> Use these search terms to deepen your understanding of Part 1 topics. Categorised by reader type.

## For All Readers

| Topic       | Search Term                                                    | Why                                      |
| ----------- | -------------------------------------------------------------- | ---------------------------------------- |
| ADR format  | `"Michael Nygard ADR format"`                                  | The standard format used in the course   |
| CAP Theorem | `"CAP theorem explained with examples"`                        | Foundational distributed systems concept |
| DDD         | `"Domain-Driven Design Eric Evans blue book summary"`          | The original source                      |
| API First   | `"API First design benefits government APIs"`                  | Philosophy behind Day 2                  |
| Zero Trust  | `"NIST Zero Trust Architecture SP 800-207"`                    | The US NIST standard, widely referenced  |
| NFRs        | `"non-functional requirements template software architecture"` | Practical templates                      |

## For Engineers from Monolith Backgrounds

| Topic                  | Search Term                                                     | Why                               |
| ---------------------- | --------------------------------------------------------------- | --------------------------------- |
| Microservices basics   | `"Martin Fowler microservices article"`                         | Best introductory reference       |
| Event Sourcing         | `"event sourcing explained Greg Young GOTO conference"`         | Original author's talk            |
| CQRS                   | `"CQRS pattern Martin Fowler bliki"`                            | Concise authoritative explanation |
| Hexagonal Architecture | `"Alistair Cockburn hexagonal architecture ports and adapters"` | Original paper                    |

## For Engineers from Infrastructure/DevOps Backgrounds

| Topic            | Search Term                                           | Why                            |
| ---------------- | ----------------------------------------------------- | ------------------------------ |
| DDD              | `"DDD for non-programmers bounded context explained"` | Accessible intro               |
| Aggregate design | `"DDD aggregate design rules Vaughn Vernon"`          | Practical implementation guide |
| OpenAPI          | `"OpenAPI specification tutorial swagger"`            | Learn the format               |
| CQRS             | `"CQRS without event sourcing simple example"`        | Simpler form first             |

## India/Singapore Government Context

| Topic                 | Search Term                                                     | Why                         |
| --------------------- | --------------------------------------------------------------- | --------------------------- |
| India API Policy      | `"India API policy MeitY open API framework"`                   | Regulatory context          |
| Singapore IM8         | `"Singapore IM8 cybersecurity requirements GovTech"`            | Security compliance context |
| ABDM architecture     | `"Ayushman Bharat Digital Mission ABDM technical architecture"` | Real health stack example   |
| GSTN architecture     | `"GSTN architecture technical design GST network"`              | Real fintech-scale example  |
| NDH                   | `"National Data and Analytics Platform NDAP India"`             | Data platform context       |
| SingPass architecture | `"SingPass MyInfo architecture GovTech Singapore"`              | Identity context            |

---

*Document Version: 1.0 | Course Batch: June 2026 | Prepared for: Senior Engineer → Solution Architect Program*

*This document is proprietary training material. Do not distribute outside the enrolled cohort.*