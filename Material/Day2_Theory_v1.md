# DOCUMENT 1 — TRAINING CONTENT MATERIAL

---

## ENTERPRISE SOLUTION ARCHITECTURE TRAINING
**Phase:** Architectural Foundations & Design Thinking
**Day:** 2 of 14 | **Date:** Jun 12, 2026 | **Session:** 09:00–13:30 (4.5 Hours) | **Break:** 11:10–11:20
**Topics:** API-First Design, Domain-Driven Design & System Design
**Subtopics:** Bounded Contexts, Aggregates, Ubiquitous Language | Hexagonal/Clean Architecture | OpenAPI/AsyncAPI | Interoperability Patterns & Versioning | Government Service Mesh Design
**Geography Focus:** 🇸🇬 Singapore (Primary) | 🇮🇳 India (Secondary) | 🌏 Cross-Border
**Leadership Level:** Technology Manager / Enterprise Solution Architect
**Difficulty:** Senior/Enterprise SA Level
**Copilot Integration:** Active — Prompts embedded throughout

---

## ADAPTED SESSION TIMETABLE: 09:00–13:30

| Time        | Block                                                                             |
| ----------- | --------------------------------------------------------------------------------- |
| 09:00–09:05 | Session Open: Trainer Energy Setter + Agenda Flash                                |
| 09:05–09:15 | Recap Ritual: Day 1 → Day 2 Bridge                                                |
| 09:15–10:15 | Block 1: Core Concept Teaching — DDD + Hexagonal Architecture (60 min)            |
| 10:15–10:25 | Checkpoint 1: Q&A + Whiteboard Challenge                                          |
| 10:25–11:10 | Block 2: API-First Design + OpenAPI/AsyncAPI + Interoperability Patterns (45 min) |
| 11:10–11:20 | Break — 10 Minutes (Strict)                                                       |
| 11:20–12:00 | Block 3: Architecture Patterns + Chaos Scenarios (40 min)                         |
| 12:00–12:10 | Checkpoint 2: Scenario Debate                                                     |
| 12:10–12:50 | Battle Drill: Hands-On Exercise (40 min — End of Class)                           |
| 12:50–13:05 | Exercise Review + Feedback (15 min)                                               |
| 13:05–13:15 | Daily Assignment Brief (10 min)                                                   |
| 13:15–13:30 | Food for Thought + Next Day Preview (15 min)                                      |

---

# SECTION 0: SESSION OPEN (09:00–09:05)

### TRAINER ENERGY SETTER

---

🎙️ OPEN WITH:

"Singapore's MyInfo API is consumed by over 700 private sector organisations — banks, insurers, telcos, property platforms — all reading the same citizen data in real time. That API contract is so sacred that GovTech maintains backward compatibility across major versions without a single breaking change affecting downstream consumers. Meanwhile, in India, a major state government's interoperability initiative failed not because of bad code — but because three departments used three different definitions of the word 'resident.' One team meant 'person currently living in the state.' Another meant 'person born in the state.' A third meant 'person registered in the state voter roll.' Same word. Three bounded contexts. Zero ubiquitous language. The entire integration collapsed under the weight of that one semantic disagreement. Today we make sure that never happens on your watch."

📰 NEWS HOOK (Search this):

"Singapore GovTech MyInfo API versioning strategy 2024 breaking changes"

❓ TODAY'S BIG QUESTION:

"How do you design a system where 50 different teams, across 3 organisations, in 2 countries, can integrate with your platform — without ever needing to call you for clarification?"

📋 AGENDA FLASH:
- How DDD gives you a shared language that survives organisational change
- How API-First design turns your architecture into a contract, not a conversation
- How interoperability patterns separate you from the teams that ship, then apologise

---

# SECTION 1: RECAP RITUAL (09:05–09:15)

### 🔁 RECAP RITUAL: Day 1 → Day 2 Bridge

---

**Part A: Concept Flashback (2–3 minutes)**

---

Trainer reads or paraphrases:

"Yesterday we stepped into the architect's chair for the first time — and the first thing we discovered is that the chair is uncomfortable on purpose. We learned that every architectural decision has a non-functional shadow: the NFR that nobody writes down but everyone screams about at 2am during an incident. We used ADRs to make those invisible decisions visible — not because process demands it, but because the next architect who inherits this system deserves to know why the decision was made, not just what it was. We evaluated technology stacks against TCO, not marketing brochures. And we left with one uncomfortable truth: a coder optimises for today's requirement, an architect optimises for tomorrow's constraint. Today, we go one level deeper — into the language of the system itself. Because you cannot design what you cannot name."

---

**Part B: Rapid-Fire Quiz (5–7 minutes)**

---

No hints. No discussion. One-liner answers only. Trainer picks candidates randomly.

RF-1: What is an NFR, and name two that are always in tension with each other.
→ Expected: Non-Functional Requirement. Consistency vs. Availability (CAP theorem). Security vs. Performance also acceptable.

RF-2: An ADR has a STATUS field. What does "Superseded" mean and why does it matter?
→ Expected: The decision was replaced by a newer ADR. Matters because it preserves the evolution trail — you know what was decided before and why it changed.

RF-3: You are selecting between two message brokers. One has lower latency, one has stronger compliance documentation for RBI. Which do you pick and what framework guides that decision?
→ Expected: Depends on context. If regulatory workload — compliance wins. Framework: Weighted technology evaluation matrix with criteria weighted by business context.

RF-4: What is TCO and why is it dangerous to exclude it from a build vs. buy decision?
→ Expected: Total Cost of Ownership — includes licensing, operations, training, migration, support over time. Dangerous because initial cost looks cheap but 3-year TCO often reverses the decision.

RF-5: A business stakeholder says "we need 99.999% uptime." What is your first question back to them?
→ Expected: "What does downtime cost you per minute?" — to understand whether the NFR is actually justified by business impact, and whether they are willing to pay for the 5-nines architecture it requires.

---

💡 TRAINER NOTE: Award whiteboard points. If fewer than 3/5 correct on RF-1 and RF-2, spend two additional minutes recapping ADR structure before moving forward. RF-5 is the tell — a candidate who answers with "we need multiple availability zones" instead of the business question has not yet made the mindset shift.

---

**Part C: Bridge Statement**

Yesterday we learned to make decisions. Today we learn to speak the language that makes those decisions implementable by 50 teams who will never sit in the same room as you.

---

# SECTION 2: BLOCK 1 — CORE CONCEPT TEACHING (09:15–10:15)

---

## CONCEPT 1: Domain-Driven Design — Bounded Contexts, Aggregates, Ubiquitous Language

---

### A1. THE PROBLEM STATEMENT

---

🚨 REAL WORLD PROBLEM: The Three Definitions of "Customer" — OCBC Singapore Integration Crisis

GEOGRAPHY: 🇸🇬 Singapore
DOMAIN: Banking / Enterprise Integration

THE SITUATION:

OCBC Bank Singapore operates across retail banking, wealth management, NISP (Indonesia subsidiary), and corporate banking divisions. In 2019, a major internal platform consolidation initiative attempted to build a unified "Customer 360" view — a single API that any internal system could call to retrieve everything about a customer. The initiative lasted 14 months and was quietly shelved.

The reason was not technical. The reason was semantic. The Retail Banking team defined a "Customer" as any person with an active CASA account. The Wealth Management team defined a "Customer" as any individual with a minimum AUM of S$200,000, including joint account holders counted separately. Corporate Banking defined a "Customer" as a legal entity — which might have zero humans directly associated with it in their system. The Know-Your-Customer (KYC) team had a fourth definition: any entity that had completed MAS-mandated due diligence, which included persons who had been rejected for accounts.

These four definitions lived in four separate databases, four separate codebases, maintained by four separate teams who had never agreed on a schema. The integration team built an API that tried to merge all four. Every field became nullable. Every response required downstream interpretation. The API became a data dump that callers could not trust.

THE PAIN:
- 14 months of engineering effort written off — estimated S$4.2M in lost productivity
- Downstream teams built compensating logic in their own systems, creating 11 different "Customer" representations across the bank's microservices landscape
- MAS Technology Risk Management audit flagged inconsistent customer data as a regulatory risk — the same customer appeared with different risk ratings in different systems
- Two major product launches were delayed because the product teams could not get a consistent answer to "does this customer already have product X?"

THE QUESTION ON THE TABLE:

"If you were the Chief Architect at OCBC in 2019, what would you have done differently in the first 30 days of this initiative?"

---

### A2. MENTAL MODEL — DDD FOUNDATIONS

---

🏠 LAYER 1 — ANALOGY:

Think of a large hospital. The word "patient" means something different to the Emergency Department, the Billing team, the Pharmacy, and the Insurance Claims department. The ED cares about vitals and triage category. Billing cares about insurance coverage and payment history. Pharmacy cares about allergies and current medications. Insurance cares about policy numbers and pre-authorisation codes. If you tried to build one single "Patient" object that satisfied all four — you would build a monster object that satisfies nobody. The solution is not to merge the definitions. The solution is to give each department its own precise definition of "patient" for their context, and then build explicit translation layers at the boundaries where they must talk to each other. That is Bounded Context.

🏗️ LAYER 2 — TECHNICAL DEFINITION (Architect Grade):

A Bounded Context is an explicit boundary within which a domain model applies. Inside a Bounded Context, every term in the Ubiquitous Language has one precise, agreed meaning shared between domain experts and engineers. The model inside is internally consistent. At the boundary, Context Mapping patterns (Anti-Corruption Layer, Published Language, Open Host Service, Shared Kernel) define how models translate across boundaries without polluting each other. An Aggregate is a cluster of domain objects treated as a single unit for data changes — with one designated Aggregate Root that controls all access and enforces invariants. All writes go through the root. External references to the aggregate use only the root's identity.

⚙️ LAYER 3 — UNDER THE HOOD:

An Aggregate Root enforces consistency boundaries in a distributed system. When you model a bank Account as an Aggregate Root with Transactions as child entities, you are making an explicit architectural statement: no Transaction can exist without an Account, no Transaction can be modified except through the Account, and the Account enforces the invariant that the balance can never go below zero. In an event-sourced system, the Aggregate Root emits Domain Events when its state changes — AccountDebited, AccountCredited — which other Bounded Contexts subscribe to through an event bus without needing direct coupling. The Ubiquitous Language is not documentation — it is the shared vocabulary that appears identically in conversations, user stories, API contracts, class names, database table names, and event names. When the word in the code matches the word the business uses, the translation tax goes to zero.

🔗 CERTIFICATION LINK:

Azure Solutions Architect Expert — Design patterns for enterprise integration. Maps directly to the "Design for reliability" and "Design application architecture" domains. Also aligns with TOGAF 10 Architecture Content Framework — specifically the distinction between Business Architecture (the language of the domain) and Application Architecture (the technical realisation).

---

### A3. VISUAL ARCHITECTURE DIAGRAMS

---

**Diagram 1: The Problem — One Model to Rule Them All (Anti-Pattern)**

```
                        ❌ ANTI-PATTERN: GOD OBJECT
                        
┌─────────────────────────────────────────────────────────────┐
│                   "Customer" API v1.0                       │
│                                                             │
│  customerId, name, email, phone,                            │
│  casaAccountNumber?, wealthAUM?,                            │
│  corporateEntityType?, kycStatus?,                          │
│  kycRejectionReason?, jointHolderIds?,                      │
│  riskRating?, insurancePolicyNumber?,                       │
│  lastTransactionDate?, relationshipManagerId?...            │
│                                                             │
│  (47 fields. 31 are nullable. Nobody trusts any of them.)   │
└─────────────────────────────────────────────────────────────┘
         ▲              ▲              ▲              ▲
         │              │              │              │
   Retail API     Wealth API    Corp API       KYC API
   (uses 8        (uses 11      (uses 6         (uses 5
    fields)        fields)       fields)         fields)
    
RESULT: Every consumer writes defensive null-checks.
        No single team owns the model.
        Schema changes break 4 consumers simultaneously.
```

**Diagram 2: The Solution — Bounded Contexts with Context Mapping**

```
        ✅ BOUNDED CONTEXT MODEL
        
┌──────────────────────┐      ┌──────────────────────────┐
│  RETAIL BANKING BC   │      │  WEALTH MANAGEMENT BC    │
│                      │      │                          │
│  Customer {          │      │  Client {                │
│    casaId            │      │    wealthClientId        │
│    name              │      │    displayName           │
│    accountStatus     │      │    aum                   │
│    kycVerified       │      │    riskProfile           │
│    branchCode        │      │    relationshipMgr       │
│  }                   │      │    portfolioIds[]        │
│                      │      │  }                       │
│  Language:           │      │                          │
│  "Customer" =        │      │  Language:               │
│  CASA account holder │      │  "Client" = AUM ≥ S$200k │
└──────────┬───────────┘      └──────────────┬───────────┘
           │                                 │
           │    Anti-Corruption Layer        │
           │◄────────────────────────────────┤
           │                                 │
           ▼                                 ▼
┌─────────────────────────────────────────────────────────────┐
│              INTEGRATION / TRANSLATION LAYER                │
│                                                             │
│  CustomerToClientTranslator {                               │
│    translate(Customer c) → Client                           │
│    // Only called when Wealth needs to onboard              │
│    // a Retail customer. Explicit mapping. Logged.          │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
           │                                 │
           ▼                                 ▼
┌──────────────────────┐      ┌──────────────────────────┐
│  CORPORATE BC        │      │  KYC/COMPLIANCE BC       │
│                      │      │                          │
│  Entity {            │      │  Subject {               │
│    legalEntityId     │      │    subjectId             │
│    registeredName    │      │    dueDiligenceStatus    │
│    acraNumber        │      │    riskClassification    │
│    directors[]       │      │    screeningResults[]    │
│    ubo[]             │      │    (includes REJECTED)   │
│  }                   │      │  }                       │
└──────────────────────┘      └──────────────────────────┘
```

**Diagram 3: Aggregate Root Pattern — Bank Account**

```
        AGGREGATE: BankAccount
        
        ┌─────────────────────────────────────────┐
        │  BankAccount (AGGREGATE ROOT)           │
        │                                         │
        │  accountId: AccountId                   │◄── External systems 
        │  customerId: CustomerId                 │    reference ONLY
        │  balance: Money                         │    via accountId
        │  status: AccountStatus                  │
        │  overdraftLimit: Money                  │
        │                                         │
        │  debit(amount, idempotencyKey) →        │◄── ALL writes
        │    throws InsufficientFundsException     │    go through
        │    emits: AccountDebited event          │    the ROOT
        │                                         │
        │  credit(amount, idempotencyKey) →       │
        │    emits: AccountCredited event         │
        │                                         │
        │  [INVARIANT ENFORCED HERE]:             │
        │    balance >= -overdraftLimit            │
        └────────────────┬────────────────────────┘
                         │ owns
              ┌──────────┴──────────┐
              ▼                     ▼
   ┌──────────────────┐   ┌──────────────────────┐
   │  Transaction     │   │  StatementEntry       │
   │  (Entity)        │   │  (Value Object)       │
   │                  │   │                       │
   │  txnId           │   │  date                 │
   │  amount          │   │  description          │
   │  timestamp       │   │  runningBalance       │
   │  type            │   │                       │
   │  idempotencyKey  │   │  [Immutable —         │
   │                  │   │   never modified,     │
   │  [Cannot exist   │   │   only appended]      │
   │   without        │   └──────────────────────┘
   │   AccountId]     │
   └──────────────────┘

   ⚠️ RULE: No external system gets a reference to Transaction directly.
            They request through BankAccount.getTransaction(txnId).
```

---

### A4. ARCHITECTURE DECISION RECORD

---

ADR-002: Adopt Bounded Context Decomposition Before API Design

DATE: Jun 12, 2026 (simulated; mirrors OCBC-class decision circa Q1 2019)
STATUS: Accepted

CONTEXT:
Platform consolidation initiative requires a unified customer data API. Four internal teams have divergent domain models for "Customer." Initial proposal is to build a canonical Customer object and force all teams to migrate to it.

DECISION:
Reject the canonical model approach. Instead, formally identify and document Bounded Contexts through Event Storming workshops with each domain team. Each Bounded Context owns its model. Integration happens through Context Mapping patterns — specifically Anti-Corruption Layers at every cross-boundary call.

RATIONALE:
A canonical model is an organisational decision disguised as a technical one. It requires all teams to agree on a single truth before any code is written. In a bank with 20+ years of legacy, that agreement will never come. ACL-based Context Mapping allows each team to work in their own ubiquitous language while providing translation at boundaries — which is where translation belongs.

CONSEQUENCES:
- ✅ Teams retain autonomy over their domain models
- ✅ Schema changes in one BC do not cascade to others
- ✅ Regulatory audit trail is cleaner — each BC owns its own data classification
- ⚠️ Requires investment in Event Storming facilitation skills (3–4 sessions per BC boundary)
- ⚠️ Translation layers must be tested explicitly — they become a new failure point
- 🔄 API gateway must route by BC, not by entity type — routing strategy must be redesigned

COMPLIANCE NOTE:
MAS TRM 2021, Section 9.2.1 (Data Management): Requires financial institutions to maintain data lineage and ownership clarity. Bounded Contexts directly support this by making data ownership explicit and bounded. DPDP Act 2023, Section 8: Data fiduciaries must know exactly what personal data they hold and where. A canonical model with 47 nullable fields makes this impossible to audit.

---

## CONCEPT 2: Hexagonal Architecture (Ports & Adapters)

---

### A1. THE PROBLEM STATEMENT

---

🚨 REAL WORLD PROBLEM: India's GSTN — The Database That Ate The Application

GEOGRAPHY: 🇮🇳 India
DOMAIN: Government / Enterprise

THE SITUATION:

When India's Goods and Services Tax Network went live on July 1, 2017, it was processing returns for 8 million registered taxpayers. By 2022, that number crossed 13 million. The original GSTN architecture, built in the 2014–2016 period, had a structural problem that compounded with every new integration request: the application logic was directly coupled to Oracle database stored procedures. Business rules — tax calculations, return validation, invoice matching — lived inside the database as PL/SQL. When MeitY mandated that GSTN must expose its data to new consumers (GST Suvidha Providers, the e-invoice system, the e-way bill system, and eventually Account Aggregators), the team discovered they could not expose the functionality without exposing the database. Every new integration required schema access, stored procedure documentation, and in some cases — direct DB user grants to external systems.

This is the textbook definition of a system that has no ports. The business logic has no interface. The application is the database.

THE PAIN:
- Every new GSP (GST Suvidha Provider) onboarding required 4–6 weeks of custom integration work involving DBA access
- A schema migration to support e-invoicing broke 3 existing GSP integrations simultaneously
- Performance testing for peak load (GST filing deadlines — last 3 days of every month) was impossible without production-like database access, creating security risk
- Estimated ₹340 crore in rework across GSTN and GSP ecosystem between 2017–2020 attributable to tight coupling

THE QUESTION ON THE TABLE:

"If you had been the Solution Architect for GSTN in 2014, what architectural principle would you have insisted on before a single line of application code was written?"

---

### A2. MENTAL MODEL — HEXAGONAL ARCHITECTURE

---

🏠 LAYER 1 — ANALOGY:

Your home has a power socket on the wall. The socket has a standard interface — two or three pins at specific voltages. You do not know or care whether your electricity comes from coal, solar, hydro, or a nuclear plant. You do not rewire your laptop when the power source changes. Your laptop has a plug — a port — and the wall has an adapter that connects the plug to whatever power source is behind it. Hexagonal Architecture works identically. Your application core — the business logic — has ports: defined interfaces for what it needs (input) and what it provides (output). Adapters implement those interfaces. You can swap a REST adapter for a gRPC adapter. You can swap an Oracle adapter for a PostgreSQL adapter. The core never changes. The core never knows.

🏗️ LAYER 2 — TECHNICAL DEFINITION (Architect Grade):

Hexagonal Architecture (Ports and Adapters, Alistair Cockburn, 2005) organises an application into three concentric zones. The Core contains pure domain logic and domain model — no framework dependencies, no database imports, no HTTP knowledge. The Ports are interfaces (Java: interfaces, Python: abstract base classes or protocols) that define what the core needs from the outside world (driven/secondary ports) and what the outside world can do to the core (driving/primary ports). Adapters are implementations of ports — a REST Controller is a driving adapter, a JPA Repository is a driven adapter, a Kafka consumer is a driving adapter, a third-party payment gateway client is a driven adapter. Dependency direction always points inward — from adapter to port to core. The core depends on nothing outside itself.

⚙️ LAYER 3 — UNDER THE HOOD:

The practical consequence of Hexagonal Architecture in a government system context: your test suite tests the core with zero infrastructure. You inject a fake/in-memory adapter for the database port. You inject a fake adapter for the external government API port. Your unit tests run in milliseconds. Your integration tests — which use real adapters — are separated and run in CI pipelines with real infrastructure. When the government mandates a switch from Oracle to PostgreSQL (as MeitY has pushed for open-source adoption), you implement a new PostgreSQL adapter, run the existing test suite against it, and you are done. Nothing in the core changes. This is not theoretical — it is the architecture that separates systems that survive government mandate changes from systems that require full rewrites.

🔗 CERTIFICATION LINK:

Maps to Azure Well-Architected Framework — Reliability Pillar (isolating components for independent deployment and testing). Also relevant to the Azure Solutions Architect Expert exam domain: "Design application architecture" — specifically designing for loose coupling and testability.

---

### A3. VISUAL ARCHITECTURE DIAGRAMS

---

**Diagram 1: Hexagonal Architecture — Full Structure**

```
          ╔═══════════════════════════════════════════════════════╗
          ║           HEXAGONAL ARCHITECTURE                     ║
          ╚═══════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────┐
│                         DRIVING ADAPTERS                                │
│                    (What calls your application)                        │
│                                                                         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐    │
│   │ REST API     │  │ gRPC Service │  │ Kafka Consumer           │    │
│   │ Controller   │  │ Handler      │  │ (Event-driven trigger)   │    │
│   └──────┬───────┘  └──────┬───────┘  └───────────┬──────────────┘    │
│          │                 │                       │                   │
└──────────┼─────────────────┼───────────────────────┼───────────────────┘
           │                 │                       │
           ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    DRIVING PORTS (Input Interfaces)                     │
│                                                                         │
│   TaxReturnSubmissionPort  │  InvoiceValidationPort  │  RefundPort     │
│   (interface/protocol)     │  (interface/protocol)   │                 │
└────────────────────────────┼────────────────────────────────────────────┘
                             │
                             ▼
           ┌─────────────────────────────────────────┐
           │                                         │
           │         APPLICATION CORE                │
           │                                         │
           │  ┌─────────────────────────────────┐   │
           │  │   DOMAIN MODEL                  │   │
           │  │                                 │   │
           │  │   TaxReturn (Aggregate Root)    │   │
           │  │   Invoice (Entity)              │   │
           │  │   TaxPayer (Entity)             │   │
           │  │   GSTCalculationService         │   │
           │  │   ReturnValidationService       │   │
           │  │                                 │   │
           │  │   [Zero external imports]       │   │
           │  │   [Pure business logic]         │   │
           │  │   [Testable in milliseconds]    │   │
           │  └─────────────────────────────────┘   │
           │                                         │
           └───────────────┬─────────────────────────┘
                           │
┌──────────────────────────┼─────────────────────────────────────────────┐
│                    DRIVEN PORTS (Output Interfaces)                     │
│                                                                         │
│   TaxReturnRepository  │  NotificationPort  │  AuditLogPort            │
│   (interface)          │  (interface)       │  (interface)             │
└────────────────────────┼───────────────────────────────────────────────┘
           │             │                   │
           ▼             ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        DRIVEN ADAPTERS                                  │
│               (What your application calls out to)                      │
│                                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ PostgreSQL  │  │ Oracle DB    │  │ SMS Gateway  │  │ Kafka      │  │
│  │ Adapter     │  │ Adapter      │  │ Adapter      │  │ Producer   │  │
│  │ (implements │  │ (implements  │  │              │  │ Adapter    │  │
│  │  Repo port) │  │  same port)  │  │              │  │            │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘

KEY INSIGHT: Swapping Oracle → PostgreSQL = implement new adapter.
             Core tests pass unchanged. That is the entire value.
```

**Diagram 2: Before vs. After — GSTN Architecture**

```
BEFORE (GSTN 2017 — Coupled):

┌──────────────────────────────────────────────────────────┐
│  GST Suvidha Provider (External)                         │
└────────────────────────────┬─────────────────────────────┘
                             │  Direct DB Connection
                             │  (Schema access required)
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Oracle Database                                         │
│                                                          │
│  Business Logic IN STORED PROCEDURES:                    │
│  - calculate_igst(invoice_id)                            │
│  - validate_gstr1(taxpayer_id, period)                   │
│  - match_invoices(supplier_gstin, buyer_gstin)           │
│                                                          │
│  Application = Database. No separation.                  │
└──────────────────────────────────────────────────────────┘
PROBLEM: Every new consumer needs DB access. Schema change = all consumers break.

─────────────────────────────────────────────────────────────────────

AFTER (Hexagonal Target):

┌───────────┐  ┌───────────┐  ┌────────────┐  ┌───────────────────┐
│ GSP REST  │  │ e-Invoice │  │ e-Way Bill │  │ Account           │
│ Adapter   │  │ Adapter   │  │ Adapter    │  │ Aggregator Adapter│
└─────┬─────┘  └─────┬─────┘  └──────┬─────┘  └─────────┬─────────┘
      │               │               │                   │
      └───────────────┴───────────────┴───────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  Driving Ports   │
                    │  (API Contracts) │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────────────────────────────────┐
                    │  APPLICATION CORE (Java/Python)              │
                    │  GSTCalculationService (Pure Logic)          │
                    │  ReturnValidationService (Pure Logic)        │
                    │  InvoiceMatchingService (Pure Logic)         │
                    │  [No Oracle imports. No HTTP imports.]       │
                    └────────────────────────────┬─────────────────┘
                                                 │
                                        ┌────────▼─────────┐
                                        │  Driven Ports    │
                                        │  (Repo interfaces│
                                        │   Notif interfaces│
                                        └────────┬─────────┘
                                                 │
                              ┌──────────────────┴───────────┐
                              ▼                              ▼
                    ┌──────────────────┐         ┌──────────────────┐
                    │ PostgreSQL       │         │ Notification     │
                    │ Adapter          │         │ Adapter (SMS/    │
                    │ (implements      │         │ Email/UMANG)     │
                    │  repository port)│         └──────────────────┘
                    └──────────────────┘
```

---

# SECTION 3: CHECKPOINT 1 (10:15–10:25)

### 🎯 CHECKPOINT 1: Depth Probe — DDD & Hexagonal Architecture

---

**Q1 — TYPE: RECALL**

QUESTION:
"A colleague says 'we should create one Aggregate for the entire Order domain — OrderHeader, OrderLines, Payments, Shipments, Returns, all in one.' As the lead architect, what is your response, and what is the specific risk they are creating?"

STRONG ANSWER CONTAINS:
- Aggregates should be small. Large aggregates serialise all concurrent access through the root, creating contention
- One Aggregate = one transaction boundary. Putting Payment and Shipment in the same Aggregate means they cannot be updated concurrently by independent services
- The right decomposition: Order, Payment, Shipment, and Return are likely separate Aggregates, coordinated via Domain Events

RED FLAGS:
- Candidate agrees with the colleague because "it keeps everything in one place"
- Candidate talks about database tables instead of domain boundaries

FOLLOW-UP PROBE:
"If Payment and Order are separate Aggregates, how do you ensure that a paid order is never shipped without a corresponding payment record? Walk me through the mechanism."

🤖 COPILOT PROMPT (Show live):
"Explain the trade-offs of large vs. small aggregates in Domain-Driven Design. Give a concrete example from a payment processing system."

---

**Q2 — TYPE: SCENARIO**

QUESTION:
"You are the Lead Architect at GovTech Singapore. You are designing the next version of MyInfo — the national identity data platform used by 700+ organisations. The product team wants a single canonical 'Citizen' object that everyone gets when they call the MyInfo API. The legal team says different organisations can only see different subsets of citizen data. The security team says data minimisation is mandatory under PDPA. How do you architect the response model, and which DDD pattern is central to your solution?"

STRONG ANSWER CONTAINS:
- Bounded Context per consumer type: banks get financial profile context, telcos get identity verification context, etc.
- Published Language pattern — MyInfo exposes a well-defined, versioned language that consumers use, not a canonical internal model
- Data projection by scope: the API response is shaped by the consent scope granted, not by a single canonical schema
- PDPA compliance through data minimisation is structurally enforced by the Bounded Context model, not just a filter

RED FLAGS:
- Suggests a single canonical JSON schema with all fields and let consumers ignore what they don't need — this is the anti-pattern and violates data minimisation
- Does not mention PDPA or consent scopes

FOLLOW-UP PROBE:
"How does your design handle the scenario where a new organisation type — say, a licensed moneylender — needs a data profile that doesn't map cleanly to any existing context?"

🤖 COPILOT PROMPT (Show live):
"How does Singapore's MyInfo platform use API scoping and consent to implement data minimisation under PDPA? What architectural patterns support this?"

---

**Q3 — TYPE: TRADE-OFF**

QUESTION:
"Compare a Shared Kernel context mapping pattern against an Anti-Corruption Layer, specifically for integrating a 15-year-old legacy IRCTC reservation system with a new microservices-based travel platform. Which do you choose and why?"

STRONG ANSWER CONTAINS:
- Shared Kernel: both teams share a subset of the domain model — requires high coordination, shared ownership, joint releases. Works only when both teams can evolve together
- ACL: the new system has a translation layer that converts the legacy model to its own model. Legacy never changes. New system is protected from legacy model pollution
- For a 15-year-old legacy: ACL is the clear choice. The legacy cannot be changed quickly, the teams are separate, and protecting the new domain model from legacy concepts (like IRCTC's archaic PNR structure) is critical
- ACL adds latency and complexity at the boundary — this is the trade-off to acknowledge

RED FLAGS:
- Chooses Shared Kernel without acknowledging the organisational coordination cost
- Does not mention that Shared Kernel requires both teams to agree on every schema change

FOLLOW-UP PROBE:
"The legacy team says the ACL translation will lose data. Specifically, IRCTC's seat preference model has 14 fields that your new platform doesn't have equivalents for. How do you handle that?"

---

**Q4 — TYPE: FLAW FINDER**

QUESTION:
"Here is an architecture description from a team's design document. Find at least 3 structural flaws:"

```
DESIGN DOCUMENT EXCERPT:

Our UserService is the central service. All other services call UserService 
to get user data before processing any request. UserService connects directly 
to the Users table in the shared database. OrderService, PaymentService, 
NotificationService, and ReportingService all import the UserService client 
library. The UserService Aggregate contains: userId, name, email, phone, 
shippingAddresses[], paymentMethods[], orderHistory[], loyaltyPoints, 
preferredLanguage, notificationPreferences[], deviceTokens[].

When a user updates their email, UserService updates the DB and calls 
PaymentService, NotificationService, and ReportingService synchronously 
to propagate the change.
```

STRONG ANSWER CONTAINS:
- Flaw 1: Synchronous fan-out on email update — if any downstream service is down, the email update fails. This is a distributed monolith masquerading as microservices
- Flaw 2: God Aggregate — orderHistory[] and paymentMethods[] do not belong in a User aggregate. They belong to Order and Payment bounded contexts respectively
- Flaw 3: Shared database with direct table access — UserService "owns" a table that others can see. This is structural coupling at the database layer
- Flaw 4 (bonus): All services importing a UserService client library means a client library update forces all consumers to re-deploy — this is deployment coupling

RED FLAGS:
- Only finds one flaw
- Does not identify the synchronous fan-out as a reliability risk

---

**Q5 — TYPE: WHITEBOARD**

QUESTION:
"You have 2 minutes. On paper or whiteboard, draw the Hexagonal Architecture for a simplified UPI payment flow. The system must accept a payment initiation from a mobile app, validate the VPA (Virtual Payment Address), debit the payer account, credit the payee account, and emit a payment completed event. Mark your ports and adapters clearly."

STRONG ANSWER CONTAINS:
- Core: PaymentApplicationService with business logic
- Driving port: PaymentInitiationPort (called by mobile adapter)
- Driven ports: AccountRepository port, VPALookupPort, EventPublisherPort
- Adapters: REST/gRPC driving adapter, NPCI switch adapter, Bank CBS adapter, Kafka producer adapter
- Domain events: PaymentCompleted, PaymentFailed

🤖 COPILOT PROMPT (Show live):
"Draw the ports and adapters for a simplified UPI payment processing service. List the driving ports, driven ports, and example adapters for each."

---

# SECTION 4: BLOCK 2 — ADVANCED CONCEPTS + USE CASES (10:25–11:10)

---

## API-First Design: OpenAPI, AsyncAPI & Interoperability Patterns

---

### THE PROBLEM STATEMENT (Block 2 opener — keep brief)

Before API-First, most teams built the implementation first and generated the API documentation as an afterthought. The result: the API documentation describes what was built, not what was designed. It reflects implementation decisions — not consumer needs. API-First inverts this. The API contract is the first artefact. It is reviewed, iterated, and approved before any implementation begins. The contract is the source of truth. The code is a downstream artefact.

---

### USE CASE 1: Singapore SGFinDex — The Financial Data Exchange API Ecosystem

GEOGRAPHY: 🇸🇬 Singapore
DOMAIN: Banking / Government / Financial Services

PROBLEM NARRATIVE:

SGFinDex (Singapore Financial Data Exchange) launched in December 2020 as the world's first public digital infrastructure that allows individuals to retrieve their financial data from across multiple banks and government agencies, with their explicit consent. A Singapore citizen can log in with SingPass, consent to share their CPF data, DBS account data, OCBC mortgage data, and IRAS tax assessment — all in a single authorised API flow — and have a financial planning application consolidate this into a complete personal financial picture.

The architectural challenge is profound. SGFinDex is not a central database. It is a consent-brokered, federated API network where GovTech acts as the orchestrator but holds no financial data itself. Every participating bank must expose a standardised API endpoint — the SGFinDex Published Language — that responds identically regardless of the bank's internal data model. DBS uses one core banking system. OCBC uses another. UOB uses a third. All three must produce identical API responses for the same data categories. The API contract is fixed. The implementation is each bank's problem.

SCALE PARAMETERS:
- Participating Institutions: 9 banks + CPF Board + IRAS + HDB (as of 2024)
- Concurrent Consent Sessions: Up to 50,000 during peak financial planning periods
- API Response SLA: 95th percentile under 3 seconds per institution
- Data Freshness: Near-real-time (within 24 hours for account balances)
- Regulatory Deadline: MAS mandated onboarding timeline — phased, 6-month windows
- Versioning Constraint: No breaking changes permitted after production launch without 12-month deprecation notice

SOLUTION ARCHITECTURE:

```
┌──────────────────────────────────────────────────────────────────────┐
│  USER JOURNEY: "Show me all my money in one place"                   │
└──────────────────────────────────────────────────────────────────────┘

[Citizen] → [SingPass Login + Consent Grant]
              │
              ▼
┌─────────────────────────┐
│  GovTech Consent Layer  │◄── Published API Contract (OpenAPI spec)
│  (SGFinDex Orchestrator)│    Version: v2.1 (immutable after launch)
└────────────┬────────────┘
             │ OAuth 2.0 + PKCE consent tokens
             │ (one per institution, scoped to data category)
             │
   ┌─────────┴──────────────────────────┐
   │         │                          │
   ▼         ▼                          ▼
┌──────┐  ┌──────┐               ┌──────────────┐
│ DBS  │  │ OCBC │   ...         │  CPF Board   │
│ API  │  │ API  │               │  API         │
│      │  │      │               │              │
│ ACL  │  │ ACL  │               │ ACL          │
│ (DBS │  │(OCBC │               │(CPF internal │
│  core│  │ core │               │ model →      │
│ model│  │ model│               │ SGFinDex     │
│  →   │  │  →   │               │ Published    │
│SGFin │  │SGFin │               │ Language)    │
│ spec)│  │ spec)│               │              │
└──────┘  └──────┘               └──────────────┘
   │         │                          │
   └─────────┴──────────────────────────┘
             │ Standardised JSON responses
             │ (per SGFinDex OpenAPI Published Language)
             ▼
┌─────────────────────────┐
│  Financial Planning App │
│  (Third-party consumer) │
│  Receives unified view  │
│  with zero knowledge of │
│  underlying bank systems│
└─────────────────────────┘
```

PATTERN APPLIED: Published Language + Anti-Corruption Layer at each institution

WHY NOT a Central Data Warehouse: Would require banks to push citizen financial data to a government-operated database — politically and legally untenable under PDPA. Consent would need to be for perpetual storage, not point-in-time retrieval.

WHY NOT Point-to-Point Direct APIs: 9 banks × N applications = N×9 bespoke integrations. Every bank API change breaks all applications. Not scalable and not auditable for consent.

CRIME/LOOPHOLE ANGLE:
A threat actor who compromises the consent token issuing infrastructure can impersonate a citizen and retrieve their complete financial picture. The SGFinDex model mitigates this through SingPass MFA as the consent gate — but the token itself, once issued, is the key. Token theft via man-in-the-browser attacks targeting the financial planning app is the primary threat vector. Architecture response: short-lived tokens (15-minute expiry), binding token to device fingerprint, and PKCE to prevent authorisation code interception.

RESULT:
- 650,000+ consent authorisations processed in first 18 months
- Zero breaking API changes required across participating institutions
- MAS audit confirmed data sovereignty maintained — GovTech holds no financial data at rest
- Time to onboard a new financial institution: reduced from 18 months (estimated point-to-point) to 4–6 months using Published Language specification

---

### USE CASE 2: India ONDC — Open Network for Digital Commerce Interoperability

GEOGRAPHY: 🇮🇳 India
DOMAIN: Government / E-Commerce / Enterprise

PROBLEM NARRATIVE:

ONDC (Open Network for Digital Commerce) is India's answer to platform monopoly in e-commerce. Instead of a buyer being locked into Amazon's app to buy from Amazon sellers and a Flipkart app to buy from Flipkart sellers — ONDC creates a protocol layer where any buyer app can transact with any seller app. A buyer using Paytm's ONDC interface can purchase from a seller listed on Meesho's ONDC interface. The transaction flows through the ONDC network using a standardised AsyncAPI-based protocol (Beckn Protocol) — an event-driven message schema that defines search, select, init, confirm, status, cancel, and update flows as asynchronous messages.

The architectural challenge: ONDC does not own any buyer data or seller data. It is a pure protocol network. Every message is signed with the sender's private key and verified by the receiver. There is no central broker. A buyer app sends a broadcast search message to the network; all seller apps that can fulfil respond directly to the buyer app's callback URL. This is a fully decentralised, event-driven interoperability architecture at national scale.

SCALE PARAMETERS:
- Registered Buyers (Jan 2025): 8.5 million
- Registered Sellers: 750,000+
- Daily Transactions: 250,000+ (growing 40% quarter-on-quarter as of Q1 2025)
- Message Types: 14 standard Beckn API calls per transaction lifecycle
- Latency SLA: Search response aggregation within 2 seconds across all responding seller nodes
- Regulatory: DPIIT mandate, MeitY oversight

SOLUTION ARCHITECTURE:

```
  ┌────────────────────────────────────────────────────────────────┐
  │                    ONDC PROTOCOL NETWORK                       │
  │              (No central data store — pure protocol)           │
  └────────────────────────────────────────────────────────────────┘

[Buyer on Paytm App]
        │
        │ /search (AsyncAPI — Beckn Protocol)
        │ {intent: "tomatoes 1kg", location: "Bengaluru 560001"}
        │ [Signed with Paytm Buyer App private key]
        ▼
┌───────────────────┐
│  ONDC Registry    │◄── Lookup: which seller nodes serve 560001?
│  (Lookup only —   │
│   holds no data)  │
└────────┬──────────┘
         │ Routes to: [BigBasket Node] [Dunzo Node] [Local Kirana Node]
         │
   ┌─────┴─────┬──────────────┐
   ▼           ▼              ▼
┌────────┐ ┌────────┐  ┌──────────────┐
│BigBask.│ │ Dunzo  │  │Local Kirana  │
│ Seller │ │ Seller │  │ Aggregator   │
│  Node  │ │  Node  │  │ Node         │
└────┬───┘ └────┬───┘  └──────┬───────┘
     │          │              │
     └──────────┴──────────────┘
                │
                │ /on_search responses (async callbacks)
                │ [Each signed with seller node private key]
                ▼
[Paytm Buyer App callback URL]
        │
        │ [Aggregates responses, displays to buyer]
        │ [Buyer selects BigBasket item]
        │
        │ /select → /init → /confirm (per Beckn lifecycle)
        ▼
[BigBasket Seller Node]
        │
        │ Order confirmed. Payment via Paytm UPI.
        ▼
[Delivery fulfilment — outside ONDC protocol]
```

PATTERN APPLIED: AsyncAPI Published Language (Beckn Protocol) + Decentralised Event-Driven Architecture

WHY NOT REST Synchronous: Search across 750,000 sellers cannot be synchronous — the request would need to wait for the slowest seller node. Async callback pattern allows parallel responses with timeout-based aggregation.

WHY NOT Central Broker (like Kafka): Defeats the decentralisation purpose. A central broker creates a chokepoint, a surveillance point, and a regulatory complexity (who owns the broker owns the data).

CRIME/LOOPHOLE ANGLE:
Fake seller nodes — an actor registers as a seller node, receives search requests (learning buyer intent and location data at scale), but never fulfils orders. ONDC's mitigation: cryptographic signing means every message is attributable; registry requires GSTIN verification for seller onboarding; reputation scoring is built into the protocol for long-term fraud deterrence.

RESULT:
- India's e-commerce ecosystem opened to any technology provider without platform lock-in
- 1,300+ cities covered by ONDC network as of 2024
- Interoperability achieved without a single central database holding buyer or seller data

---

## Versioning Strategies — The Architect's Responsibility

API versioning is one of the most consequential decisions an architect makes. It is permanent. Once consumers are on v1, you cannot force them off. Here is the decision framework:

**URL Path Versioning** `/api/v1/customers`, `/api/v2/customers`
Use when: Major breaking changes. Clear separation. Easy routing. Simple to understand.
Avoid when: You want consumers to automatically get improvements without code changes.

**Header Versioning** `Accept: application/vnd.myapi.v2+json`
Use when: Semantic versioning matters, you want the URL to remain stable.
Avoid when: Consumers are non-technical (header manipulation is non-obvious).

**Query Parameter Versioning** `/api/customers?version=2`
Use when: Rapid prototyping, easy browser testing.
Avoid when: Production APIs. Caching behaviour becomes unpredictable.

**Semantic Versioning for APIs — The Contract Rule:**

```
MAJOR version: Breaking change. Consumers MUST update.
               Old version supported for [deprecation period — minimum 12 months for Gov].
               
MINOR version: New fields, new endpoints. Backward compatible.
               Consumers MAY update for new features.
               
PATCH version: Bug fixes. No contract change.
               Consumers need not change anything.
               
THE CARDINAL SIN: Removing a field in a MINOR or PATCH update.
                  This is a breaking change disguised as non-breaking.
                  It will cause production failures at 2am.
```

---

🤖 COPILOT LIVE DEMO

WHEN: After explaining API versioning strategies

TRAINER ACTION: Open Microsoft Copilot

PROMPT TO RUN:
"I am designing an API for a government citizen services platform in Singapore. The API will be consumed by 700+ organisations. What API versioning strategy should I use, and what are the risks of each approach? Include considerations for MAS Technology Risk Management guidelines."

EXPECTED COPILOT OUTPUT: Should cover URL versioning as most common, header versioning for semantic clarity, deprecation timelines, and ideally mention backward compatibility as a non-negotiable.

CRITIQUE EXERCISE: Ask class — "What did Copilot miss? Did it mention AsyncAPI? Did it address the regulatory deprecation notice period? Did it consider the impact on smaller consumer organisations that cannot respond quickly to version changes?"

TEACHING POINT: Copilot gives you the 80% answer quickly. The 20% that is missing is always the contextual, regulatory, and stakeholder-specific nuance. That 20% is what you are paid for.

---

# SECTION 5: BREAK (11:10–11:20)

---

While you rest — A thought to carry:

"If your API is the product — and 700 organisations depend on its contract — who owns the API contract? The team that builds it? The team that consumes it? Or someone else entirely? We answer this in the next block."

🤖 OPTIONAL COPILOT PROMPT (try on your phone):
"What is an API governance model? How do large organisations like GovTech Singapore or NPCI India manage API contracts across hundreds of consumers?"

Session resumes: 11:20 sharp

---

# SECTION 6: BLOCK 3 — ARCHITECTURE PATTERNS + CHAOS SCENARIOS (11:20–12:00)

---

## A. ARCHITECTURE PATTERNS DEEP DIVE

---

### Pattern: Strangler Fig for API-First Migration

This pattern is the bridge between Block 2 (API-First) and what comes in Day 8 (Legacy Modernisation). Introduce it here because it is the most common context in which API versioning decisions become irreversible.

**Named Pattern:** Strangler Fig (Martin Fowler, 2004)

**Formal Definition:** Incrementally replace a legacy system by placing a new system alongside it, routing specific functions to the new system while the legacy continues to handle the remainder. Over time, the new system "strangles" the legacy until the legacy can be decommissioned.

**When to Use:**
- Legacy system cannot be taken offline for a rewrite
- Business continuity is non-negotiable during migration
- The legacy system has value that cannot be replicated quickly

**When NOT to Use:**
- The legacy system's data model is so corrupt that any facade over it inherits the corruption
- The migration team does not have access to change routing logic at the edge (no API gateway control)
- The legacy system has no automated tests — you will not know when the strangler breaks existing behaviour

**Implementation on Azure Free Tier:**
Azure API Management (Developer tier — free for 30 days, then minimal cost) acts as the routing facade. New endpoints route to new microservices (Azure Container Apps — free tier available). Legacy endpoints continue routing to the legacy system.

```
STRANGLER FIG PATTERN — GSTN MIGRATION EXAMPLE:

Phase 1 (Month 1-3): Facade in place, 100% to legacy
┌────────────┐   All traffic   ┌────────────────────┐
│ API Gateway│────────────────▶│ Legacy GSTN System │
│ (new facade│                 │ (Oracle + PL/SQL)  │
│  in place) │                 └────────────────────┘
└────────────┘

Phase 2 (Month 4-8): New service handles read-only queries
┌────────────┐  /returns (GET) ┌────────────────────┐
│ API Gateway│────────────────▶│ New Returns Service │
│            │                 │ (Java 17 + Postgres)│
│            │  all writes     └────────────────────┘
│            │────────────────▶┌────────────────────┐
└────────────┘                 │ Legacy GSTN System │
                               └────────────────────┘

Phase 3 (Month 9-14): New service handles all traffic
┌────────────┐   All traffic   ┌────────────────────┐
│ API Gateway│────────────────▶│ New GSTN Platform  │
│            │                 │ (Full replacement) │
└────────────┘                 └────────────────────┘
                               Legacy decommissioned.

KEY ARCHITECTURE REQUIREMENT:
- API contract at the gateway NEVER changes across all phases
- Consumers see the same endpoints, same response schema throughout
- Switchover is invisible to consumers
```

**Java 17 concept-level — Port Interface for Strangler:**

```java
// FILE: src/main/java/com/gstn/returns/application/port/TaxReturnRepository.java
// PURPOSE: Driven port — isolates core from storage implementation
// PATTERN: Hexagonal Architecture + Strangler Fig migration support
// PRODUCTION DELTA: Would include pagination, cursor-based for large datasets

package com.gstn.returns.application.port;

import com.gstn.returns.domain.TaxReturn;
import com.gstn.returns.domain.TaxPeriod;
import com.gstn.returns.domain.GstIn;

import java.util.Optional;
import java.util.List;

// This interface is what the core depends on.
// TODAY: OracleAdapter implements this (calls legacy PL/SQL)
// TOMORROW: PostgresAdapter implements this (calls new schema)
// The core never changes. The adapter swaps.
public interface TaxReturnRepository {
    
    Optional<TaxReturn> findByGstinAndPeriod(GstIn gstin, TaxPeriod period);
    
    List<TaxReturn> findPendingReturns(GstIn gstin);
    
    // Idempotency key prevents duplicate submissions — critical for 
    // government filing systems where network retries are common
    TaxReturn save(TaxReturn taxReturn, String idempotencyKey);
    
    boolean existsByIdempotencyKey(String idempotencyKey);
}
```

---

## B. CHAOS & PRESSURE SIMULATION

---

🔥 CHAOS SCENARIO: The API Contract Breaks on Budget Day

SEVERITY: P0 | TIME: 08:47 SGT, February 14 | GEOGRAPHY: 🇸🇬 Singapore

THE INCIDENT (Trainer reads aloud):

"It is 08:47 on the morning of Singapore's Budget Day — the most watched financial announcement of the year. The CPF Board has integrated with three major banks via the SGFinDex API to allow citizens to view their CPF contributions alongside bank savings for financial planning. At 08:47, three minutes before trading opens on SGX and forty minutes before the Budget speech is expected to reference digital financial inclusion milestones, your monitoring dashboard goes critical red. The SGFinDex data retrieval API — specifically the CPF contribution endpoint — is returning HTTP 200 responses with an empty `contributions` array for 340,000 citizens who tried to check their CPF balance this morning through their banking apps.

Your Slack is flooded. DBS is on the phone. OCBC's integration team sent an email at 08:51 saying their CPF widget shows 'no contribution history' for all users. The CPF Board's operations team is calling your director. The Minister's office has sent a message to the GovTech CEO: 'Is the system ready for today?' Your director has just walked into the room and asked you one question: 'What happened and what is the ETA to fix it?'

You have 30 seconds to decide your first action."

⏱️ TRAINER: Give candidates 3 minutes silent thinking time. No discussion. Then open the floor.

BLAST RADIUS ANALYSIS:

Directly Affected: CPF contribution display across DBS digibank, OCBC app, UOB TMRW — 3 consumer applications

Cascade Risk: If the fix requires an API schema change, all 9 SGFinDex participating institutions need to redeploy their ACL adapters within the same window — a 4–6 hour operation that cannot happen during market hours

Data Risk: Citizen financial data is NOT exposed (the array is empty, not incorrect). PDPA exposure is low. But MAS TRM requires notification of any service degradation affecting >10,000 customers within 1 hour of detection.

Financial Impact: Estimated S$1.2M per hour in reputational cost (Budget Day brand association). Zero direct financial loss since no transactions are affected.

WRONG APPROACHES:

❌ WRONG APPROACH 1: The junior engineer who immediately rolls back the CPF Board API deployment.
WHY WRONG: You do not yet know if CPF's deployment caused this. Rolling back could destroy evidence and may not fix anything if the issue is in the ACL translation layer on the consumer side.

❌ WRONG APPROACH 2: The siloed DBA who checks the CPF database for missing data.
WHY WRONG: The HTTP 200 with empty array is not a database error — a database error returns 500 or 503. HTTP 200 with empty data means the data was queried and returned nothing, or the serialisation lost the data. Database investigation wastes 20 minutes on the wrong layer.

CORRECT RESPONSE ARCHITECTURE:

IMMEDIATE (0–5 min):
- Check API gateway access logs: is the CPF endpoint receiving requests and returning 200?
- Check CPF Board ACL adapter logs: what is the raw response from CPF's internal system?
- If internal system returns data but API returns empty — the ACL translation is the bug
- Assign P0 incident commander. All communication goes through one person.

SHORT-TERM (5–30 min):
- If the ACL adapter has a serialisation bug (most likely — empty array not null check): hotfix the adapter, deploy to staging, run smoke test against CPF test endpoint
- Enable feature flag to route CPF contribution calls to cached yesterday-data as fallback while fix is deployed
- Send MAS notification at T+45 minutes (before the 1-hour MAS TRM deadline)

RECOVERY (30 min–2 hrs):
- Deploy fixed ACL adapter. Validate against all 9 institution test environments.
- Gradually re-enable live CPF calls: 10% traffic, monitor, 50%, monitor, 100%
- Coordinate with CPF Board comms team on citizen-facing message

POST-INCIDENT:
- Root cause was a null-vs-empty-list handling bug introduced in a minor version release of the Published Language deserialiser
- Architecture change: contract tests (consumer-driven contract testing using Pact) must pass in CI before any Published Language schema release — this prevents serialisation incompatibilities from reaching production

POST-MORTEM TEMPLATE:

TIMELINE:
- 08:47 — Alert triggered: CPF contribution endpoint returning empty arrays
- 08:52 — Incident commander assigned, bridge call opened with CPF Board and DBS
- 09:04 — Root cause identified: null coalescing bug in ACL deserialiser introduced in v2.1.3 release (deployed 08:15)
- 09:31 — Fix deployed to staging, validated
- 09:58 — Fix deployed to production, traffic gradually restored
- 10:15 — Full service restored. MAS notified at 09:41 (within 1-hour window).

ROOT CAUSE (5-Why):
- Why 1: CPF contribution data showing empty for all users
- Why 2: ACL deserialiser converted null JSON field to empty list instead of preserving null for absent data
- Why 3: v2.1.3 introduced a utility method that defaulted null collections to empty lists globally
- Why 4: The change was categorised as a patch (no API contract change) and skipped contract test validation
- Why 5: Contract testing was not enforced as a CI gate for patch releases — only for major and minor

ACTION ITEMS:
- Enforce Pact contract tests as mandatory CI gate for ALL release types, including patches | Owner: Platform Engineering | Priority: P0 | Due: 5 business days
- Add integration test case: "null field in upstream response → preserved as null, not converted to empty collection" | Owner: CPF ACL Team | Due: 3 business days
- MAS TRM incident report submission | Owner: Technology Risk team | Due: 14 days (per MAS guidelines)

---

# SECTION 7: CHECKPOINT 2 (12:00–12:10)

### 🎯 CHECKPOINT 2: Scenario Debate — API Contracts & Interoperability

---

**Q1 — TYPE: PATTERN APPLICATION DEBATE**

"Two architects are arguing. Architect A says: 'We should version our government service APIs by URL path — `/api/v1/`, `/api/v2/` — because it is simple for consumers to understand.' Architect B says: 'We should never change the URL — use Accept headers for versioning, because the URL IS the resource identity and should not change.' Pair up. One of you argues Architect A's position. The other argues Architect B's position. 3 minutes. Go."

Expected outcome: Both positions have merit. The real answer depends on consumer technical sophistication, caching strategy, and operational complexity. For government-facing APIs where consumers include small organisations with limited technical teams — URL versioning wins on simplicity. For internal platform APIs — header versioning gives more flexibility. The resolution is: it depends on who your consumers are, and that is an architectural decision you make consciously.

---

**Q2 — TYPE: MANAGEMENT ESCALATION**

"Your CTO has just received a complaint from HDFC Bank's CTO. HDFC Bank integrated with your government platform 18 months ago. You just released v2.0 of your API with a breaking change — a field they depend on was removed. They are threatening to escalate to SEBI. You have 5 minutes to draft the talking points your CTO should use in the call. What do you say and, more importantly, what is the architectural failure that led to this situation?"

Strong answer:
- Immediate: acknowledge, take responsibility, offer 6-month parallel operation of v1 and v2
- Architectural failure: no deprecation policy was in place, or it was not enforced. Breaking change should have triggered a 12-month notice period with v1 kept alive during migration
- Long-term: institute API governance board, mandatory consumer impact assessment before any breaking change

---

**Q3 — TYPE: CROSS-BORDER CONSIDERATION**

"India's UPI and Singapore's PayNow are now linked. A payment initiated in India via UPI reaches a beneficiary in Singapore via PayNow. The API contract for this cross-border flow involves NPCI on the India side and MAS-regulated entities on the Singapore side. The Singapore side requires that every API response includes a `regulatoryTraceId` field for MAS audit purposes. The India side never had this field. How do you handle this in your API design, and which context mapping pattern applies?"

Strong answer:
- Anti-Corruption Layer on the Singapore-side connector translates the India UPI response (which has no `regulatoryTraceId`) by generating and appending a Singapore-side trace ID at the boundary
- The trace ID is generated by the Singapore ACL based on the UPI transaction reference — creating a bidirectional audit link
- The India side never changes. The Singapore side adds the required regulatory context in its own ACL
- This is an example of Conformist + ACL hybrid: India NPCI is upstream and too large to influence (conformist), so Singapore builds a protective ACL

---

🤖 COPILOT VALIDATION — Run Live:

"I am designing a cross-border payment API between India's UPI network and Singapore's PayNow. Singapore's MAS requires a regulatory trace ID in every transaction response. India's UPI specification does not include this field. How should I design the API translation layer? Which Domain-Driven Design context mapping pattern applies?"

Class critique: What did Copilot recommend? Did it suggest ACL? Did it understand the regulatory asymmetry? What did it miss about the operational complexity of maintaining an ACL at a cross-border payment boundary?

---

# SECTION 8: BATTLE DRILL — HANDS-ON EXERCISE (12:10–12:50)

### ⚡ BATTLE DRILL 2: Design the Government Service Mesh

TIME: 12:10–12:50 (40 minutes — 4.5hr session adjustment)
FORMAT: Teams of 3

---

SCENARIO BRIEF:

You are the Solution Architecture team at GovTech Singapore. The Smart Nation Programme Office has mandated that by December 2026, all government digital services must be accessible through a unified Service Mesh API layer — a single integration point where any authorised application (internal or private sector) can access government data services through a standardised, versioned, consent-brokered API catalogue.

You have been given 40 minutes to produce the initial architecture proposal for this Government Service Mesh. The current state: 47 different government agencies have 47 different API styles, authentication mechanisms, and data formats. Some are REST, some are SOAP (yes, 2026 and there is still SOAP), some expose database views directly. Your solution must not require any agency to change their internal systems immediately — migration is phased over 24 months.

YOUR MISSION:

You have 40 minutes to produce:

□ Architecture Diagram: The Government Service Mesh — Context Level (C4 Level 1) and Container Level (C4 Level 2). Show at minimum: consumer layer, API gateway/mesh layer, agency adapter layer, and one example agency backend.

□ Top 3 ADRs: The three most consequential architectural decisions in this design. Use the ADR format from Day 1. Include compliance context (MAS TRM / PDPA).

□ API Governance Policy (1 page): Who owns the API contract? How is versioning decided? What is the deprecation process? What happens when an agency refuses to update their backend?

TOOLS AVAILABLE:

□ Whiteboard / Paper
□ Microsoft Copilot (allowed — with conditions below)
□ Personal notes from today's session

🤖 COPILOT RULES FOR THIS DRILL:

- Allowed: Use Copilot to validate your approach after you draft it
- Allowed: Use Copilot to check if there are real-world examples you should reference
- Not Allowed: Ask Copilot to generate the architecture directly
- Not Allowed: Use Copilot output as your first draft — you must think first
- Show your Copilot prompts — they will be reviewed as part of feedback

EVALUATION RUBRIC:

| Criterion                                                                         | Score |
| --------------------------------------------------------------------------------- | ----- |
| Architecture Completeness (C4 L1 + L2 present, all layers identified)             | /10   |
| Non-Functional Requirements (at least 4 NFRs called out with justification)       | /10   |
| Trade-off Articulation (ADRs show WHY, not just WHAT)                             | /10   |
| Scalability & Resilience Thinking (what happens when agency X goes down)          | /10   |
| Security Posture & Compliance (PDPA, MAS TRM explicitly referenced)               | /10   |
| Copilot Usage Quality (prompts are precise, output was critically evaluated)      | /5    |
| Communication Clarity (could you present this to the SNPO Director in 5 minutes?) | /5    |
| TOTAL                                                                             | /60   |

FEEDBACK TEMPLATE:

WHAT WORKED: _______________________________________________________________

WHAT'S MISSING: ___________________________________________________________

WHAT A PRINCIPAL ARCHITECT WOULD ADD: ______________________________________

CERTIFICATION ALIGNMENT: This exercise maps directly to the Azure Solutions Architect Expert exam domain "Design application architecture" — specifically designing API management solutions, integration patterns, and governance for multi-tenant platforms.

---

# SECTION 9: DOCUMENTATION TEMPLATES + PROFESSIONAL COMMUNICATION

### ⬆️ LEVELLING UP: API GOVERNANCE DOCUMENT

Today's template: the RFC (Request for Comments) — the document an architect writes when proposing a significant API design decision that affects multiple teams.

---

RFC-0047: Adopt AsyncAPI Specification for All Event-Driven Government APIs

Status: DRAFT
Author: [Solution Architect, GovTech Platform Team] | Date: Jun 12, 2026 | Expires: Jul 12, 2026

Abstract:
This RFC proposes mandating AsyncAPI 3.0 specification for all new event-driven APIs published within the Government Service Mesh. Currently, event-driven integrations between agencies use informal message schemas documented in Word documents or not at all. This creates integration risk and prevents automated consumer contract testing.

Motivation:
Three integration incidents in the past 6 months (Incident #INC-2024-0341, #INC-2024-0587, #INC-2025-0012) were caused by undocumented changes to event message schemas. In each case, a producing agency changed a field name or type without notifying consumers. AsyncAPI specification, enforced as a CI gate, would have prevented all three incidents through schema validation.

Proposed Solution:
All new event-driven APIs within the Government Service Mesh must: (1) have an AsyncAPI 3.0 specification file committed to the central API Registry before production deployment; (2) pass automated schema validation in CI pipeline; (3) follow semantic versioning with a 12-month deprecation notice for breaking changes; (4) be registered in the API Catalogue accessible to all authorised agencies.

Alternatives Considered:

| Alternative                             | Reason Not Chosen                                                                                             |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Continue with informal Word doc schemas | Does not prevent undocumented changes; not machine-readable                                                   |
| OpenAPI for event-driven APIs           | OpenAPI is designed for request/response, not event streams; forces synchronous mental model onto async flows |
| CloudEvents specification only          | CloudEvents defines envelope, not payload schema; insufficient for contract enforcement                       |

Security Considerations:
AsyncAPI specs will be published in the internal API Catalogue — access restricted to authorised agencies via SingPass-federated authentication. Specs must not include sample data containing real citizen identifiers.

Compliance Considerations:
MAS TRM 2021, Section 9.3 (API Management): Requires documented API contracts and version management. This RFC directly fulfils that requirement for event-driven APIs. PDPA: AsyncAPI specs define data fields but not data retention — separate Data Classification must be attached to each spec.

Open Questions:
1. Should legacy event-driven APIs (existing systems) be required to retroactively produce AsyncAPI specs? If so, what is the timeline and who funds the effort?
2. Is the API Registry a new system to procure, or can it be built on existing GovTech API Management infrastructure?

References: AsyncAPI Specification 3.0 (asyncapi.com), MAS TRM 2021, ADR-002 (Bounded Context decomposition)

---

**Stakeholder Communication Template — Architecture Recommendation Email**

Subject: Architecture Recommendation — API Versioning Policy for Government Service Mesh — For Approval

Dear [Director, Smart Nation Programme Office],

RECOMMENDATION IN ONE LINE:
Adopt URL path versioning with a mandatory 12-month parallel operation window for all Government Service Mesh APIs — enforced by API Management gateway policy, not by individual agency agreement.

WHY NOW:
The Government Service Mesh is in design phase. Versioning policy decisions made now become permanent constraints on all 47 agencies. Getting this wrong will cost significantly more to remediate after go-live than to specify correctly today.

OPTIONS EVALUATED:

Option A: URL path versioning with 12-month deprecation window — Score: 9/10 ← RECOMMENDED
Option B: Header-based versioning — Score: 6/10 (technically superior but operationally complex for smaller agencies)
Option C: No enforced versioning policy — Score: 2/10 (this is the current state that caused three incidents)

RISKS IF WE DELAY:
- Each agency will independently adopt a versioning approach, creating 47 incompatible patterns
- First breaking change after go-live (estimated Month 8) will affect all consumers without a managed migration path
- MAS TRM audit readiness for API documentation requires a documented versioning policy

I REQUEST:
□ Approval to proceed with Option A as the Government Service Mesh standard
□ A 30-minute discussion call by June 19, 2026

Detailed RFC-0047 and Technology Evaluation Matrix accessible at [link].

Regards,
[Name] | Senior Solution Architect, GovTech Platform Engineering

---

# SECTION 10: MICROSOFT COPILOT INTEGRATION

### 🤖 COPILOT PROMPT BANK — DAY 2

**TYPE 1: CONCEPT VALIDATION (During DDD teaching)**

WHEN: After explaining Bounded Contexts

PROMPT TO RUN:
"Explain Domain-Driven Design Bounded Contexts to a senior software engineer transitioning to solution architect. Use a real example from a government digital services platform. Include: how to identify context boundaries, what goes wrong when boundaries are wrong, and the relationship between Bounded Contexts and microservices."

CRITIQUE EXERCISE: Ask class — Did Copilot confuse microservices boundaries with Bounded Context boundaries? They are related but not the same. A Bounded Context can be implemented by multiple microservices, or by a monolith.

---

**TYPE 2: DOCUMENT GENERATION (During RFC template teaching)**

COPILOT PROMPT:
"Act as a senior solution architect at a Singapore government technology agency. Generate an RFC proposing the adoption of consumer-driven contract testing (using Pact) for all APIs in a government service mesh. Include: motivation, proposed solution, alternatives considered, security and PDPA compliance considerations."

TRAINER INSTRUCTIONS: Run live. Show how the output quality improves when you add "Include specific reference to MAS TRM 2021 Section 9" to the prompt. This teaches prompt refinement as a professional skill.

---

**TYPE 3: ARCHITECTURE REVIEW (After Battle Drill)**

COPILOT REVIEW PROMPT:
"Review this government service mesh architecture for a platform serving 47 agencies in Singapore. Identify: (1) security vulnerabilities at the API gateway layer, (2) scalability bottlenecks during peak events like Singapore Budget Day, (3) compliance gaps under MAS TRM 2021 and PDPA, (4) versioning risks if agencies adopt different update cadences. Architecture: [candidate's architecture pasted]"

---

**TYPE 4: RESEARCH ACCELERATION (For daily assignment)**

COPILOT RESEARCH PROMPT:
"Summarise the key architectural lessons from Singapore's SGFinDex implementation and India's ONDC Beckn Protocol for a solution architect preparing for Azure Solutions Architect Expert certification. Focus on: API-First design, interoperability patterns, and consumer-driven contract management."

---

# SECTION 11: DAILY ASSIGNMENT (13:05–13:15)

### 📋 ARCHITECTURE KATA: Day 2 — Design the ONDC Seller Onboarding API

---

DAILY ASSIGNMENT: Day 2
Estimated Effort: 90–120 minutes | Due: Before Day 3 starts (Jun 15, 09:00)

SCENARIO:
You are the API Architect at DPIIT (Department for Promotion of Industry and Internal Trade), responsible for the ONDC network's seller onboarding API. Currently, sellers are onboarded through a manual process that takes 14 days. The product team wants a self-service API that allows any Seller Node Application (SNA) to programmatically onboard a new seller in under 24 hours. The API will be consumed by 500+ SNAs across India, ranging from large platforms like Meesho with 50-person engineering teams to small regional aggregators with 2-person tech teams. The API must comply with DPDP Act 2023 (seller business data is personal data under the Act), be versionable without breaking existing SNA integrations, and support both synchronous onboarding (for premium SNAs) and asynchronous onboarding (for standard SNAs where verification takes time).

DELIVERABLES (Submit as PDF or shared document):

□ 1. OpenAPI 3.1 specification (YAML) for the Seller Onboarding API — minimum: /sellers POST, /sellers/{sellerId} GET, /sellers/{sellerId}/status GET
□ 2. AsyncAPI 3.0 specification for the asynchronous onboarding events — minimum: SellerOnboardingInitiated, SellerVerificationCompleted, SellerOnboardingFailed
□ 3. Top 3 ADRs — must include: versioning decision, sync vs. async decision, authentication mechanism decision
□ 4. Bounded Context diagram — identify at minimum: Seller Identity BC, Verification BC, Catalogue BC, and their integration pattern
□ 5. API Governance Policy (one page) — how will breaking changes be managed given 500+ consumers of varying technical sophistication?
□ 6. DPDP Act 2023 compliance note — which data fields in your API constitute personal data, and what controls are required?

ASPECT FOCUS FOR TODAY: Interoperability, API-First Design, Context Boundaries

SELF-EVALUATION CHECKLIST:

Before submitting, verify:
□ Does my OpenAPI spec use semantic versioning and include a deprecation header?
□ Does my AsyncAPI spec define message schemas, not just event names?
□ Does each ADR include DPDP/MAS regulatory context where relevant?
□ Is my Bounded Context diagram based on domain language — not just technical layers?
□ Is my API Governance Policy realistic for a team of 500 SNAs with varying capability?
□ Did I use Copilot to review my work and document what it added and what I rejected?

COPILOT CHALLENGE:

Use this Copilot prompt to get a second opinion on your API design:
"Review this OpenAPI specification for a seller onboarding API that will be consumed by 500+ organisations in India. Check for: (1) missing error response codes, (2) fields that may constitute personal data under DPDP Act 2023, (3) versioning gaps, (4) security vulnerabilities at the API contract level. Specification: [paste your spec]"

Document: What did Copilot flag that you missed? What did you disagree with and why?

---

# SECTION 12: FOOD FOR THOUGHT + NEXT DAY PREVIEW (13:15–13:30)

### 🧠 ARCHITECT'S MEDITATION: Day 2

---

💭 PROVOCATIONS (No right answer — discuss with a peer tonight):

1. If the Ubiquitous Language is so important — who owns it? The domain expert who cannot write code, or the engineer who cannot understand the business? And when they disagree, who wins?

2. Singapore's MyInfo publishes a fixed API contract that 700 organisations depend on. That contract is a promise. When the government's own data model changes — say, the definition of "permanent resident" is legally amended — does the API contract change? And if it does, who bears the cost of 700 organisations updating their integrations?

3. ONDC's decentralised architecture means no single entity can surveil the entire network's transaction patterns. But law enforcement in India has asked for a mechanism to flag suspicious transactions. If you add that mechanism, you centralise something that was deliberately decentralised. If you don't, you are told you are enabling crime. What do you design, and how do you sleep at night?

READ TONIGHT:

🔍 Search: "Beckn Protocol ONDC architecture deep dive"
🔍 Search: "SGFinDex architecture GovTech Singapore API"
🔍 Search: "consumer-driven contract testing Pact microservices postmortem"
📖 Book Reference: "Domain-Driven Design" by Eric Evans — Chapter 14: "Maintaining Model Integrity" (specifically the Context Map section)

ANTI-PATTERN TO RESEARCH:

🚫 "The Distributed Monolith" — Research what it is, how it happens to teams that decompose by technical layer rather than by domain, and where you have seen it in your own career.

TOMORROW'S TEASER:

👀 Day 3 Preview: Distributed Systems & Event-Driven Architecture

"Kafka is not a database. If you are using it like one, we need to talk. Tomorrow we go into the engine room of every system that claims to be 'real-time' — and we find out which ones actually are, which ones are lying, and which ones will fail in ways that are architecturally fascinating and professionally catastrophic."

We will explore: Event Sourcing vs. CQRS vs. plain messaging, Reactive Patterns and back-pressure, Workflow Engines (Camunda/Temporal) for long-running processes

🎯 Come prepared to answer:
"If Kafka goes down for 4 minutes during UPI peak hours, and your system uses Kafka as the only channel between your payment initiator and your bank CBS connector — what exactly happens to the 2.3 million transactions that were in flight? Walk me through the failure, second by second."

---
