# DOCUMENT 1 — TRAINING CONTENT MATERIAL

---

## ENTERPRISE SOLUTION ARCHITECTURE TRAINING
**Phase:** Architectural Foundations & Design Thinking
**Day:** 3 of 14 | **Date:** Jun 15, 2026 | **Session:** 09:00–13:30 (4.5 Hours) | **Break:** 11:10–11:20
**Topics:** Distributed Systems & Event-Driven Architecture
**Subtopics:** Microservices vs. SOA vs. Space-Based Architecture | Event-Driven Patterns: Kafka/RabbitMQ, Event Sourcing, CQRS | Non-Blocking I/O, Reactive Patterns & Workflow Engines (Camunda/Temporal) | Hands-on: Building resilient event flows with failure simulation
**Geography Focus:** 🇮🇳 India (Primary) | 🇸🇬 Singapore (Secondary) | 🌏 Cross-Border
**Leadership Level:** Technology Manager / Enterprise Solution Architect
**Difficulty:** Senior/Enterprise SA Level
**Copilot Integration:** Active — Prompts embedded throughout

---

## ADAPTED SESSION TIMETABLE: 09:00–13:30

| Time        | Block                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------ |
| 09:00–09:05 | Session Open: Trainer Energy Setter + Agenda Flash                                               |
| 09:05–09:15 | Recap Ritual: Day 2 → Day 3 Bridge                                                               |
| 09:15–10:15 | Block 1: Core Concept Teaching — Distributed Systems Fundamentals + Architecture Styles (60 min) |
| 10:15–10:25 | Checkpoint 1: Q&A + Whiteboard Challenge                                                         |
| 10:25–11:10 | Block 2: Event-Driven Patterns — Kafka, Event Sourcing, CQRS (45 min)                            |
| 11:10–11:20 | Break — 10 Minutes (Strict)                                                                      |
| 11:20–12:00 | Block 3: Reactive Patterns + Workflow Engines + Chaos Scenarios (40 min)                         |
| 12:00–12:10 | Checkpoint 2: Scenario Debate                                                                    |
| 12:10–12:50 | Battle Drill: Hands-On Exercise (40 min — End of Class)                                          |
| 12:50–13:05 | Exercise Review + Feedback (15 min)                                                              |
| 13:05–13:15 | Daily Assignment Brief (10 min)                                                                  |
| 13:15–13:30 | Food for Thought + Next Day Preview (15 min)                                                     |

---

# SECTION 0: SESSION OPEN (09:00–09:05)

### TRAINER ENERGY SETTER

---

🎙️ OPEN WITH:

"On October 14, 2023, UPI processed 11.4 billion transactions in a single month — an average of 4,380 transactions every second, around the clock. Not 4,380 per minute. Per second. Now here is what nobody tells you: UPI does not run on one system. It runs on a choreography of dozens of independent services — bank switches, NPCI routing, fraud detection, notification systems, settlement engines — all talking to each other asynchronously, all potentially failing independently, all expected to produce a consistent financial result at the end. When someone in Chennai pays a street vendor ₹35 for chai using UPI, that ₹35 touches at minimum eight distributed systems before the vendor's phone beeps. And every single one of those eight systems can fail independently. Today we learn how to build systems that survive that reality — not by preventing failure, but by designing for it so thoroughly that failure becomes invisible to the user."

📰 NEWS HOOK (Search this):

"NPCI UPI 11 billion transactions October 2023 architecture resilience"

❓ TODAY'S BIG QUESTION:

"When distributed systems fail — and they will — how do you design event flows that are resilient enough that the end user never knows the failure happened?"

📋 AGENDA FLASH:
- Why microservices, SOA, and space-based architecture are not interchangeable buzzwords — and which one you actually need
- How Kafka, Event Sourcing, and CQRS work together to build systems that remember everything and lie to no one
- How workflow engines save you from writing distributed transaction logic that will eventually kill your team

---

# SECTION 1: RECAP RITUAL (09:05–09:15)

### 🔁 RECAP RITUAL: Day 2 → Day 3 Bridge

---

**Part A: Concept Flashback (2–3 minutes)**

---

Trainer reads or paraphrases:

"On Thursday we drew boundaries — around domains, around data, around the language teams use when they talk about their systems. We said a Bounded Context is a promise: inside this boundary, words mean exactly one thing. We implemented that promise in code using Hexagonal Architecture — where the domain is a fortress with defined gates called ports, and every infrastructure dependency enters only through those gates as adapters. We built a consent service for a Singapore-style financial data exchange where three different bank adapters — DBS, OCBC, CPF — all translated their native bank models into the same clean domain language before the core ever touched the data. The Anti-Corruption Layer was not a pattern on a whiteboard. It was a 20-line method called `translateToAccountSummary` that stood between your clean domain and 15 years of bank legacy. Today we take those clean domain events — the `ConsentGranted`, the `RetrievalCompleted` — and we ask the harder question: what happens when those events need to travel across a network to fifty other services, survive a server restart, replay on demand, and drive an audit trail that a regulator can query three years from now? That is distributed event-driven architecture. And that is today."

---

**Part B: Rapid-Fire Quiz (5–7 minutes)**

---

RF-1: What is a Bounded Context and what is the single most dangerous thing that happens when two teams share a domain model without explicit context boundaries?
→ Expected: A Bounded Context is an explicit boundary within which a domain model is internally consistent. The danger: model drift — the same term silently acquires different meanings in each team's code, causing integration failures that are invisible until production.

RF-2: In Hexagonal Architecture, if the PostgreSQL adapter and the in-memory adapter both implement the same repository port — what is the ONE thing that must be true of both implementations for the architecture to hold?
→ Expected: Both must satisfy the same interface contract — same method signatures, same return types, same exception contracts. The core must be unable to distinguish between them at runtime. Substitutability — Liskov Substitution Principle.

RF-3: You receive a POST request with the same X-Idempotency-Key twice. What must your API do on the second call and why?
→ Expected: Return the same response as the first call without processing the request again. Idempotency prevents duplicate operations on network retry — critical for financial APIs where a duplicate consent or payment is a regulatory and operational failure.

RF-4: What is the Anti-Corruption Layer pattern and in one sentence — what does it protect?
→ Expected: A translation layer at a Bounded Context boundary that converts an upstream system's model into your domain model. It protects your domain's ubiquitous language from being polluted by a legacy or external system's terminology and structure.

RF-5: Name the three context mapping patterns discussed yesterday and give one sentence on when you would use each.
→ Expected: Published Language (use when you are the platform and many consumers must conform to your contract — e.g., SGFinDex). Anti-Corruption Layer (use when you consume a legacy or external system you cannot change — protect your model from theirs). Shared Kernel (use when two closely aligned teams share a small, stable subset of the model and can release jointly — high coordination cost).

---

💡 TRAINER NOTE: RF-5 will separate the candidates who processed the concepts from those who sat through them. If fewer than half can name all three patterns — do a 2-minute concept recap before Block 1. The bridge to today's content requires understanding that domain events (what Block 1 introduces) are the mechanism by which Bounded Contexts communicate without sharing models.

---

**Part C: Bridge Statement**

Yesterday we built clean boundaries between systems. Today we wire those systems together with events — and we learn what happens when the wire breaks.

---

# SECTION 2: BLOCK 1 — CORE CONCEPT TEACHING (09:15–10:15)

---

## CONCEPT 1: Architecture Styles — Microservices vs. SOA vs. Space-Based

---

### A1. THE PROBLEM STATEMENT

---

🚨 REAL WORLD PROBLEM: India's IRCTC — The Architecture That Made a Nation Swear

GEOGRAPHY: 🇮🇳 India
DOMAIN: Government / Enterprise / Crisis

THE SITUATION:

IRCTC's Tatkal booking window opens at 10:00 AM for AC classes and 11:00 AM for non-AC classes. For exactly five minutes on high-demand routes — Mumbai–Delhi, Chennai–Bangalore — somewhere between 600,000 and 800,000 concurrent users attempt to book simultaneously. This is not a gradual traffic ramp. It is a vertical wall. At 09:59:59, the system has normal load. At 10:00:01, it is under assault by two-thirds of a million people who all clicked "Book Ticket" within the same second.

The original IRCTC architecture — built in the early 2000s and progressively patched — was a classic three-tier monolith: a web layer, an application server cluster, and an Oracle database. The Oracle database was the single arbiter of seat inventory. Every booking request required a SELECT FOR UPDATE on the seat availability table to prevent double-booking. Under Tatkal load, this table became a serialisation bottleneck — thousands of transactions queuing for row-level locks on a handful of rows. The result was a thundering herd: the database ground to near-halt, the application servers timed out waiting for DB responses, users saw "service unavailable," and then — in the worst architectural irony — retried. Making it worse.

Between 2012 and 2016, IRCTC processed approximately 7 million transactions per day at peak — but Tatkal windows regularly produced 30-minute brownouts. Social media during Tatkal windows became a live performance review of a government architecture decision made in 2002.

THE PAIN:
- Estimated ₹150 crore per year in lost booking revenue during brownout windows
- Support call volume increased 800% during Tatkal windows — each call costing ₹45 in agent time
- Tatkal touts — agents using bots to book and resell tickets — exploited brownout periods because their automated retries succeeded when human users gave up, creating a systemic fairness failure
- TRAI received 14,000 complaints in one month about IRCTC availability in 2014

THE QUESTION ON THE TABLE:

"If you had been handed the IRCTC re-architecture mandate in 2016 — ₹200 crore budget, 18-month timeline, cannot take the system offline — what architecture style do you choose, and what is your first design decision?"

---

### A2. MENTAL MODEL — Architecture Style Comparison

---

🏠 LAYER 1 — ANALOGY:

Think of three ways to run a large hospital. A monolith is one giant building where every department — emergency, surgery, pharmacy, billing — shares the same corridors, the same elevators, and the same central database of patient records. When the elevator breaks, everyone is affected. SOA is the same hospital broken into wings connected by a standardised internal phone system — each wing has its own staff and processes, but they call each other using agreed protocols. Slow, but organised. Microservices is when each department moves into its own building across the city, each with its own staff, its own records, its own entrance. Faster and more independent — but now you need a city-wide coordination system, and getting a patient from emergency to surgery requires an ambulance and real-time communication. Space-Based Architecture is different entirely — it is like having a hundred satellite clinics distributed across the city, each carrying a complete copy of the data it needs for its local catchment. No central database. Each clinic serves its neighbourhood. Synchronisation happens in the background. This is what you use when the elevator problem is so severe that no building layout helps — you have to eliminate the centralised building entirely.

🏗️ LAYER 2 — TECHNICAL DEFINITION (Architect Grade):

**Monolith:** Single deployable unit. All components share a process, memory, and database. Low operational complexity. High coupling. Scaling requires scaling the entire unit.

**SOA (Service-Oriented Architecture):** Coarse-grained services communicating via an Enterprise Service Bus (ESB) using standardised protocols (SOAP/WSDL, WS-*). Services share a canonical data model. Governance is centralised at the ESB. The ESB becomes a smart pipe — containing routing logic, transformation, orchestration. High governance overhead. Common in enterprise and government systems pre-2012.

**Microservices:** Fine-grained, independently deployable services communicating via lightweight protocols (REST/gRPC/events). Each service owns its data store. No shared database. Smart endpoints, dumb pipes (Kafka/RabbitMQ carry messages without transformation logic). Decentralised governance. Teams own services end-to-end. Requires mature DevOps culture.

**Space-Based Architecture (SBA):** Eliminates the database as a centralised bottleneck by distributing both processing and data across a grid of processing units. Each processing unit holds an in-memory data grid — a full or partial replica of the data it needs. Writes are processed in-memory and asynchronously persisted. Designed specifically for extreme peak loads where the database is the bottleneck. IRCTC's Tatkal problem is the textbook SBA use case.

⚙️ LAYER 3 — UNDER THE HOOD:

In a Space-Based Architecture implementation for IRCTC Tatkal:

A processing unit is a self-contained cluster node that holds seat inventory for a specific set of trains in an in-memory data grid (Hazelcast, Apache Ignite, or GemFire). When a booking request arrives, it is routed to the processing unit that owns that train's inventory — no database query required. The booking decision is made entirely in-memory in microseconds. The result is written to a messaging grid and asynchronously persisted to the database. The database is no longer in the critical path for the booking decision. A seat that shows as available in the in-memory grid is guaranteed to be available because that processing unit is the single writer for that partition of inventory. No row-level locking. No thundering herd. The database catches up asynchronously.

The trade-off: if a processing unit fails before the async write completes, you have lost those in-flight booking decisions. This requires careful design of the data synchronisation layer and the failure recovery protocol. SBA trades ACID guarantees for extreme scalability — and requires explicit engineering effort to manage the resulting eventual consistency.

🔗 CERTIFICATION LINK:

Maps to Azure Well-Architected Framework — Performance Efficiency Pillar. AWS SAP: Design for high availability and fault tolerance. Also directly relevant to TOGAF Architecture Patterns — distinguishing between integration styles and selecting based on quality attribute trade-offs.

---

### A3. VISUAL ARCHITECTURE DIAGRAMS

---

**Diagram 1: The Three Styles — Side by Side Comparison**

```
    SOA                    MICROSERVICES              SPACE-BASED
    
┌─────────────┐          ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐
│   ESB       │          │Svc A │ │Svc B │          │ PU-1 │ │ PU-2 │
│  (Smart     │◄────────▶│      │ │      │          │ Data │ │ Data │
│   Pipe)     │          └──┬───┘ └──┬───┘          │ Grid │ │ Grid │
│  Routing    │             │        │               │ +CPU │ │ +CPU │
│  Transform  │          ┌──┴────────┴──┐           └──┬───┘ └──┬───┘
│  Orchestrate│          │ Message Bus  │              │        │
└──────┬──────┘          │ (Dumb Pipe) │           ┌──┴────────┴──┐
       │                 └──────┬───────┘           │ Messaging   │
┌──────▼──────┐                 │                   │ Grid        │
│  CANONICAL  │          ┌──────┴──────┐           └──────┬───────┘
│  DATA MODEL │          │  Svc C      │                  │
│  (shared)   │          │  (owns DB)  │           ┌──────▼───────┐
└─────────────┘          └─────────────┘           │  DB Writer   │
       │                                            │  (async,     │
┌──────▼──────┐          Each svc: own DB           │  NOT in      │
│  SHARED DB  │          Dumb pipes                 │  critical    │
│  (central)  │          Decentralised              │  path)       │
└─────────────┘          governance                 └──────────────┘

Smart pipe +             Dumb pipe +                No central DB in
Shared model =           Each svc owns data =       critical path =
Governance overhead      DevOps maturity needed     Extreme scale
Good for enterprise      Good for independent       Good for thundering
integration where        team scaling with          herd elimination
teams share model        bounded contexts
```

**Diagram 2: IRCTC Tatkal — Before and After**

```
BEFORE (Monolith + Shared Oracle DB):

10:00:00 AM — 700,000 users hit "Book Ticket"
                          │
               ┌──────────▼──────────┐
               │  App Server Cluster │
               │  (8 nodes, 64 cores)│
               └──────────┬──────────┘
                          │ 700,000 concurrent
                          │ SELECT FOR UPDATE
                          ▼
               ┌─────────────────────┐
               │   Oracle DB         │ ◄── 💥 BOTTLENECK
               │   seat_inventory    │     Row locks = serialisation
               │   table             │     700K threads queuing
               │   (single node)     │     for 2 rows per train
               └─────────────────────┘
               
RESULT: DB lock wait timeout → App timeout → User retry → Worse

─────────────────────────────────────────────────────────────────

AFTER (Space-Based Architecture):

10:00:00 AM — 700,000 users hit "Book Ticket"
                          │
               ┌──────────▼──────────┐
               │  Load Balancer +    │
               │  Request Router     │
               │  (routes by train   │
               │   number hash)      │
               └──────────┬──────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PU: Trains   │ │ PU: Trains   │ │ PU: Trains   │
│ 1–1000       │ │ 1001–2000    │ │ 2001–3000    │
│              │ │              │ │              │
│ In-Memory    │ │ In-Memory    │ │ In-Memory    │
│ Seat Grid    │ │ Seat Grid    │ │ Seat Grid    │
│              │ │              │ │              │
│ Booking      │ │ Booking      │ │ Booking      │
│ Decision     │ │ Decision     │ │ Decision     │
│ IN MEMORY    │ │ IN MEMORY    │ │ IN MEMORY    │
│ < 5ms        │ │ < 5ms        │ │ < 5ms        │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        │ Async write
                        ▼
               ┌─────────────────────┐
               │   DB Writer         │
               │   (NOT in booking   │
               │    critical path)   │
               │   Persists booking  │
               │   confirmation      │
               │   eventually        │
               └─────────────────────┘

RESULT: Each train's inventory managed by exactly ONE processing unit.
        No shared lock. No thundering herd.
        700K concurrent requests → handled in parallel.
        DB catches up asynchronously. Booking confirmed in < 100ms.
```

**Diagram 3: SOA vs. Microservices — The ESB Trap**

```
SOA — The ESB becomes the bottleneck it was meant to prevent:

┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ Svc A  │  │ Svc B  │  │ Svc C  │  │ Svc D  │
└───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
    └───────────┴───────────┴───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │        ESB            │ ◄── Every message goes through here
            │  (Oracle Service Bus  │     Business logic leaks into ESB
            │   / IBM MQ / TIBCO)   │     ESB team becomes a bottleneck
            │                       │     Version upgrades require ESB team
            │  Routing              │     ESB down = everything down
            │  Transformation       │     SPOF disguised as integration layer
            │  Orchestration        │
            │  Error handling       │
            └───────────────────────┘
            
MICROSERVICES — Smart endpoints, dumb pipes:

┌────────┐         ┌───────────────┐         ┌────────┐
│ Svc A  │────────▶│ Kafka Topic   │────────▶│ Svc B  │
│(Producer│        │ (Dumb Pipe —  │         │(Consumer│
│)        │        │  just carries │         │)        │
└────────┘         │  messages)    │         └────────┘
                   └───────────────┘
                   
No transformation in the pipe.
No routing logic in the pipe.
Business logic stays in the services.
Kafka fails → services degrade gracefully with retry.
Kafka recovers → messages replay from offset.
No messages lost.
```

---

### A4. ARCHITECTURE DECISION RECORD

---

ADR-003: Select Microservices over SOA for NPCI UPI Next-Generation Platform

DATE: Simulated — 2016, NPCI UPI architecture design phase
STATUS: Accepted

CONTEXT:
NPCI is redesigning UPI to handle projected 10x growth from 100M to 1B transactions per month. The existing architecture is SOA-based with an ESB handling routing between bank switches. The ESB is a commercial product (TIBCO BusinessWorks) that requires specialist expertise, has 6-month upgrade cycles, and has become the single point of failure for all bank-to-bank routing.

DECISION:
Adopt microservices architecture with event-driven communication via a distributed message streaming platform. Decompose by domain: Payment Initiation Service, VPA Resolution Service, Bank Switch Routing Service, Settlement Service, Notification Service, Fraud Detection Service — each independently deployable.

RATIONALE:
SOA's ESB centralises intelligence in the pipe — creating an organisational and technical bottleneck as NPCI adds partner banks. Each new bank integration requires ESB configuration changes managed by a single team. Microservices with event-driven communication allows each bank's switch adapter to be developed, deployed, and scaled independently. The message broker (Kafka) carries messages without transformation logic — bank-specific adaptation happens in each service's adapter layer.

CONSEQUENCES:
- ✅ Bank switch adapters developed by independent teams without ESB team coordination
- ✅ Individual services scale independently — fraud detection can scale 10x during suspicious activity surges without scaling payment initiation
- ✅ ESB single point of failure eliminated — Kafka cluster with replication provides fault tolerance
- ⚠️ Distributed tracing becomes mandatory — a payment touching 8 services requires correlation IDs at every hop
- ⚠️ Data consistency across services requires explicit design — no database-level ACID across service boundaries
- 🔄 Operations team must mature significantly — container orchestration, service mesh, distributed observability all required

COMPLIANCE NOTE:
RBI Payment System Guidelines require audit trails for all payment transactions for 5 years. In a microservices architecture, this audit trail must be constructed from distributed logs — requiring a centralised log aggregation system (ELK/Splunk) with immutable append-only storage. Each service must emit structured audit events to a dedicated audit topic in Kafka.

---

## CONCEPT 2: CAP Theorem — The Architect's Non-Negotiable Reality

Before event-driven patterns, architects must internalise the CAP constraint. Every distributed system design decision downstream traces back to this.

---

### A1. THE PROBLEM STATEMENT (abbreviated — connect to earlier scenario)

---

During the IRCTC brownout, a specific question arose in a war room: "Can we show the user that seat 34B on Train 12952 is available, let them initiate booking, and then tell them it was taken during processing?" The answer determines your consistency model. The answer shapes your entire event-driven architecture. This is CAP in practice.

---

### A2. MENTAL MODEL — CAP Theorem

---

🏠 LAYER 1 — ANALOGY:

You run a chain of 500 grocery stores across India. Every store sells rice. You have three guarantees you want to offer customers: (1) Every store always shows the correct rice stock — **Consistency**. (2) Every store is always open and will always answer a customer's query — **Availability**. (3) Even if the phone lines between stores and headquarters go down — each store can still operate independently — **Partition Tolerance**. In a distributed system — phone lines WILL go down. That is a guarantee, not a risk. So you can only truly offer two of the three. If you insist on Consistency and Partition Tolerance (CP): during a network partition, stores that cannot confirm current stock will close rather than show potentially wrong data. If you insist on Availability and Partition Tolerance (AP): during a partition, stores stay open and serve customers using their last known stock — which might be wrong. There is no store that is always open, always correct, and always independent of headquarters.

🏗️ LAYER 2 — TECHNICAL DEFINITION (Architect Grade):

CAP Theorem (Eric Brewer, 2000; formally proved Gilbert & Lynch, 2002): In the presence of a network partition, a distributed system can guarantee either Consistency (every read receives the most recent write or an error) OR Availability (every request receives a response — not necessarily the most recent write). Partition Tolerance is non-negotiable in any real network — networks partition. Therefore the real choice is: CP or AP? For financial systems: CP (strong consistency, accept unavailability during partition). For social feeds, notifications, recommendations: AP (accept stale data, stay available). PACELC extends CAP: even without a partition, there is a trade-off between latency and consistency — Elasticsearch chooses low latency over strong consistency even in non-partitioned states.

⚙️ LAYER 3 — UNDER THE HOOD:

In the UPI context: the payment debit decision is CP. When a network partition occurs between NPCI and a bank's CBS, NPCI refuses to confirm the debit rather than risk a phantom debit or a double debit. The user sees "transaction pending" — an honest, consistent response. The notification that a payment succeeded is AP — it can be delivered eventually, and a slight delay is acceptable. The fraud score for a merchant is AP — slightly stale fraud scores are acceptable; unavailability of the fraud service during a partition is not acceptable (it would block all payments). Every component in a distributed system should have an explicit CP vs. AP decision in its ADR.

---

**CAP Decision Matrix — Reference for Battle Drill and Assignment:**

```
SYSTEM COMPONENT          CP or AP?   REASONING
─────────────────────────────────────────────────────────────────────
UPI Payment Debit          CP          Wrong debit = regulatory violation
UPI Payment Credit         CP          Wrong credit = financial loss
Aadhaar Authentication     CP          Wrong identity = security failure
UPI Transaction Status     AP          Slightly stale status acceptable
Push Notification          AP          Delayed notification acceptable
Fraud Score Lookup         AP          Stale score acceptable; unavailability not
GSTN Return Filing         CP          Filing date is a hard regulatory deadline
CoWIN Slot Availability    AP          Show stale slots; confirm at booking
SGFinDex Data Retrieval    CP          Stale financial data = regulatory risk
SingPass Authentication    CP          Wrong identity = critical failure
SGX Trade Execution        CP          Wrong execution = market integrity failure
MRT Real-time Tracking     AP          Slightly stale location acceptable
```

---

# SECTION 3: CHECKPOINT 1 (10:15–10:25)

### 🎯 CHECKPOINT 1: Depth Probe — Distributed Systems Fundamentals

---

**Q1 — TYPE: RECALL**

QUESTION:
"CAP Theorem says you choose between CP and AP during a network partition. A junior engineer on your team says 'we will use MongoDB because it supports both strong and eventual consistency — so we get both CP and AP.' What is wrong with this statement and how do you correct it?"

STRONG ANSWER CONTAINS:
- MongoDB offers configurable consistency — you choose per-operation. But you cannot be CP AND AP simultaneously during a partition
- Choosing strong consistency (CP) in MongoDB means reads may block or fail during a partition
- Choosing eventual consistency (AP) means reads may return stale data
- The junior engineer is conflating "configurable" with "both simultaneously" — these are different things
- The correct statement: MongoDB lets you choose your CAP position per use case, which is powerful — but the theorem still applies

RED FLAGS:
- Agrees with the junior engineer
- Does not understand that the choice is at partition time, not at design time

FOLLOW-UP PROBE:
"In your IRCTC seat booking service — you are using MongoDB for seat inventory. A network partition occurs between your Mumbai processing node and the central MongoDB primary in Delhi. What happens to a booking request that arrives at Mumbai during the partition — if you have configured readPreference: primary?"

🤖 COPILOT PROMPT (Show live):
"Explain the CAP theorem trade-off specifically for a train seat booking system in India. Which components should be CP and which should be AP? Justify each choice."

---

**Q2 — TYPE: SCENARIO**

QUESTION:
"You are the Lead Architect at NPCI. The Head of Product says: 'I want UPI to show real-time merchant balance so merchants can see incoming payments the moment they happen.' The Head of Engineering says: 'Real-time balance requires strong consistency and will add 800ms to every payment.' The CFO says: 'We cannot add 800ms — our SLA with banks is 1 second end-to-end.' Walk me through your architecture decision and the trade-off you explicitly accept."

STRONG ANSWER CONTAINS:
- CQRS separation: the write model (actual balance) is CP and slow. The read model (display balance) is AP and fast
- Merchant display balance uses an eventually consistent read replica updated within 2–5 seconds of payment
- The merchant sees "balance updated a few seconds ago" — explicitly communicated, not hidden
- The payment SLA is preserved because the display is decoupled from the payment critical path
- The trade-off explicitly accepted: merchant might see a 5-second lag in balance display — acceptable for display, never acceptable for the actual debit/credit decision

RED FLAGS:
- Suggests adding more database replicas without addressing the consistency model
- Proposes a solution that adds latency to the payment path

FOLLOW-UP PROBE:
"The merchant's display shows ₹5,000 balance. In the next 3 seconds, two payments of ₹200 arrive. The display hasn't updated yet. The merchant makes a withdrawal decision based on the ₹5,000 display. What is the risk and how does your architecture mitigate it?"

---

**Q3 — TYPE: TRADE-OFF**

QUESTION:
"Compare SOA with an ESB versus Microservices with Kafka for integrating 28 state government systems with the Central GSTN platform. Specifically: which performs better when the GSTN team needs to add a new integration for one state, and which performs better when the entire integration layer needs to be audited by C&AG?"

STRONG ANSWER CONTAINS:
- SOA/ESB: adding one state requires ESB configuration change — centralised, governed, auditable. C&AG audit is straightforward — one system to audit. But ESB team becomes bottleneck for every new state
- Microservices/Kafka: each state has its own adapter service — independent deployment, state team can develop their adapter. C&AG audit requires inspecting 28 adapter services — distributed audit complexity
- For a government context with strong audit requirements and slower integration cadence — SOA's centralised governance may actually be preferable
- The "right" answer: hybrid — lightweight API gateway (not ESB) for centralised governance and audit, microservices adapters per state for independent development
- Key insight: architecture style selection must consider the operating model, not just the technical trade-offs

RED FLAGS:
- Automatically recommends microservices without addressing the audit complexity
- Does not acknowledge that SOA's governance model has genuine advantages in regulated government contexts

---

**Q4 — TYPE: FLAW FINDER**

QUESTION:
"Review this architecture description and find at least 3 distributed systems flaws:"

```
DESIGN DESCRIPTION:

Our payment service calls the fraud detection service synchronously 
via REST before processing every payment. The fraud service calls 
the customer profile service synchronously to get customer history. 
The customer profile service calls the transaction history database. 
All three calls have a 5-second timeout. If any call fails, the 
payment fails with a 500 error. We have configured all three services 
to retry 3 times with no delay on failure. The services share the 
same PostgreSQL database for their operational data.
```

STRONG ANSWER CONTAINS:
- Flaw 1: Synchronous chain creates cascading failure — if transaction history DB is slow, fraud is slow, payment is slow. 5-second timeout × 3 services = 15-second potential failure path
- Flaw 2: Retry without backoff or jitter causes retry storms — when fraud service fails, all payment services retry simultaneously at the same moment, amplifying the overload
- Flaw 3: Shared database — if one service's query causes DB contention, all three services degrade. This is the distributed monolith anti-pattern
- Flaw 4 (bonus): Synchronous fraud check in payment critical path — fraud scoring should be asynchronous. Accept payment, flag for fraud review in parallel, reverse if fraud confirmed. Blocking payment on fraud score adds latency and creates availability dependency

RED FLAGS:
- Only identifies the timeout issue without explaining the cascade
- Does not identify shared database as a flaw

---

**Q5 — TYPE: WHITEBOARD**

QUESTION:
"2 minutes. Draw the architecture for a distributed UPI payment flow. Show at minimum: the initiating bank, NPCI routing, the beneficiary bank, and how a failure at the beneficiary bank CBS is handled without corrupting the payer's account."

STRONG ANSWER CONTAINS:
- Saga pattern (even if not named): compensating transaction at payer bank if beneficiary CBS fails
- Idempotency key propagated across all hops
- NPCI as the state machine for the transaction lifecycle
- Timeout and compensation logic shown explicitly

🤖 COPILOT PROMPT (Show live):
"Draw the architecture for a UPI payment flow across two banks via NPCI. Show how the system handles a failure at the beneficiary bank's core banking system without creating an inconsistent state at the payer bank."

---

# SECTION 4: BLOCK 2 — ADVANCED CONCEPTS + USE CASES (10:25–11:10)

---

## Event-Driven Architecture: Kafka, Event Sourcing, CQRS

---

### THE PROBLEM STATEMENT (Block 2 opener)

Every senior engineer in this room has debugged a production issue where the database showed one thing, the application showed another, and the logs showed a third. You spent four hours on a call trying to reconstruct what actually happened in the system — which sequence of events led to this inconsistent state. You had no answer because the system recorded the current state — not the history of how it got there. Event Sourcing fixes this structurally. Not by adding better logging. By making the event history the primary data — and deriving current state from it.

---

### USE CASE 1: CoWIN — Real-Time Vaccine Slot Booking at 850 Million Scale

GEOGRAPHY: 🇮🇳 India
DOMAIN: Government / Crisis / Healthcare

PROBLEM NARRATIVE:

When India opened COVID-19 vaccine registration for the 45+ age group on April 1, 2021, CoWIN received 8.5 million registrations in the first 60 minutes. When slots opened for 18+ on May 1, 2021, the platform received 3,000 API requests per second at peak — a 20x spike from normal operating load. The challenge was specific: vaccine slot availability is a contested resource. A slot can go from available to booked in milliseconds. Showing a user a slot that is "available" and then telling them it is gone when they book is an acceptable user experience. Showing a user a "confirmed" booking that is later cancelled because two users booked the same slot is a catastrophic failure — both medically (people travel to vaccination centres unnecessarily) and politically.

The CoWIN architecture team — operating under MeitY oversight — had to solve a specific consistency problem: make slot availability reads highly available and fast (AP behaviour — show approximate availability), while making slot reservation decisions strongly consistent (CP behaviour — never double-book). The solution required splitting reads from writes — which is the CQRS pattern — and making the write path use event sourcing so that every slot state transition was immutable, replayable, and auditable.

SCALE PARAMETERS:
- Peak Concurrent Users: 8.5 million (April 1, 2021 — 18+ opening)
- Peak API Requests: 3,000/second
- Total Registrations: 1 billion+ by December 2021
- Slot Decision Latency SLA: < 200ms for confirmation
- Audit Requirement: Every booking decision must be replayable for 5 years (MeitY data retention)
- Zero tolerance: Double-booking of a vaccine slot

SOLUTION ARCHITECTURE:

```
                    CoWIN CQRS + EVENT SOURCING ARCHITECTURE

[User: "Show me available slots near Delhi"]
        │
        │ GET /slots?pin=110001 (HIGH TRAFFIC — 3000 req/sec)
        ▼
┌──────────────────────────────┐
│  READ SIDE (Query Model)     │  AP — Availability over Consistency
│                              │  
│  Redis Cache Cluster         │  Updated every 30 seconds from
│  Slot Availability Index     │  write-side event stream
│  (Pre-computed per PIN code) │
│                              │
│  Response: < 10ms            │
│  May show slots taken 30s    │
│  ago — ACCEPTABLE            │
└──────────────────────────────┘

[User: "Book slot at Centre X on June 15, 10:00 AM"]
        │
        │ POST /bookings (LOWER TRAFFIC — actual reservation)
        ▼
┌──────────────────────────────────────────────────────┐
│  WRITE SIDE (Command Model)                          │  CP
│                                                      │
│  Booking Command Handler                             │
│  │                                                   │
│  ▼                                                   │
│  ┌─────────────────────────────────────────────┐    │
│  │  Slot Aggregate (Event Sourced)             │    │
│  │                                             │    │
│  │  Current state DERIVED from event history:  │    │
│  │  1. SlotCreated {centreId, date, time,     │    │
│  │                  capacity: 10}              │    │
│  │  2. SlotBooked {userId: U001, timestamp}   │    │
│  │  3. SlotBooked {userId: U002, timestamp}   │    │
│  │  ...                                        │    │
│  │  9. SlotBooked {userId: U009, timestamp}   │    │
│  │  → availableCount = 10 - 9 = 1            │    │
│  │                                             │    │
│  │  INVARIANT: availableCount >= 0            │    │
│  │  If booking request arrives and count = 0: │    │
│  │  → Emit SlotFullEvent (no state change)    │    │
│  │  → Return "slot full" to user              │    │
│  │  NEVER: two SlotBooked for same slot + seq │    │
│  └──────────────────┬──────────────────────────┘    │
│                     │                                │
│  ┌──────────────────▼──────────────────────────┐    │
│  │  Event Store (append-only)                  │    │
│  │  Kafka Topic: slot-events (partitioned      │    │
│  │  by centreId — guarantees ordering per      │    │
│  │  centre)                                    │    │
│  └──────────────────┬──────────────────────────┘    │
└────────────────────┬─────────────────────────────────┘
                     │
                     │ Event stream
                     ▼
        ┌────────────────────────────┐
        │  Projection Workers        │
        │                            │
        │  Worker 1: Update Redis    │
        │  Slot Availability Index   │
        │  (drives the READ SIDE)    │
        │                            │
        │  Worker 2: Update          │
        │  Notification Queue        │
        │  (SMS/Push to user)        │
        │                            │
        │  Worker 3: Update          │
        │  Analytics DB              │
        │  (MeitY reporting)         │
        └────────────────────────────┘
```

PATTERN APPLIED: CQRS (Command Query Responsibility Segregation) + Event Sourcing + Kafka partitioned by aggregate ID

WHY NOT traditional RDBMS with SELECT FOR UPDATE: At 3,000 requests/second targeting the same slot rows, database row-level locking creates the exact thundering herd problem IRCTC experienced. The contention at the DB layer cannot be resolved by adding replicas — replicas only help reads, not write contention.

WHY NOT distributed lock (Redis SETNX): A Redis distributed lock for each slot booking would work for moderate concurrency but becomes a bottleneck at CoWIN scale. Additionally, Redis lock expiry during a network partition creates a split-brain scenario where two nodes both believe they hold the lock. Event sourcing with Kafka partition ordering is structurally safer.

CRIME/LOOPHOLE ANGLE:
CoWIN faced large-scale bot attacks — automated scripts booking slots immediately on availability, selling the booking confirmation codes on black markets. The architecture's defence: rate limiting at API gateway per Aadhaar ID (one booking per ID per slot window). However, Aadhaar-linked rate limiting requires Aadhaar authentication on every booking — adding latency in the critical path. The architectural trade-off: add 150ms for Aadhaar validation in the booking flow to prevent bot exploitation, or skip it and accept bot abuse. CoWIN chose validation — a security-over-performance trade-off documented in their architecture review.

RESULT:
- Zero double-bookings reported during the platform's operational period
- 99.8% availability maintained during peak days through read/write separation
- Full audit trail of every slot state transition available for government reporting
- Event replay used to reconstruct booking history during technical investigations

---

### USE CASE 2: Singapore CPF Board — Event Sourcing for Retirement Account Audit

GEOGRAPHY: 🇸🇬 Singapore
DOMAIN: Government / Banking / Compliance

PROBLEM NARRATIVE:

Singapore's Central Provident Fund manages retirement savings for 4 million members. Every employer contribution, member withdrawal, investment transfer, and MediShield premium deduction must be traceable — not just in terms of current balance, but in terms of the exact sequence of events that produced that balance. When a member retires and queries their CPF statement, they expect to see every transaction for the past 40 years. When the CPF Board is audited by the Auditor-General's Office, the auditors can request reconstruction of any account's state at any point in time — not just the current balance.

A traditional database approach — storing current balance and a transaction log as a secondary table — has a subtle flaw: the transaction log is documentation, not the source of truth. If a bug causes the balance to be updated without a corresponding log entry, the balance is wrong and the discrepancy may not be discovered for months. In an Event Sourced system, the event log IS the source of truth. The current balance is a projection derived from the event log. A bug that updates the projection without emitting an event cannot exist by design — the projection is always derived, never mutated directly.

SCALE PARAMETERS:
- Active Members: 4 million
- Monthly Contribution Events: ~15 million (employer contributions for all employed members)
- Event Retention: Permanent — CPF accounts persist for a member's lifetime
- Audit Query: "What was member X's OA balance on March 15, 2019?" — must be answerable within 30 seconds
- Regulatory: CPF Act requires full transaction auditability; Auditor-General's Office compliance

SOLUTION ARCHITECTURE:

```
CPF EVENT SOURCING ARCHITECTURE

[Employer submits monthly CPF contribution]
        │
        │ ContributionSubmitted Command
        ▼
┌─────────────────────────────────────────────────────┐
│  CPF Account Aggregate (Event Sourced)              │
│                                                     │
│  State reconstructed from event stream:             │
│  Event 1: AccountOpened {memberId, date: 1985}      │
│  Event 2: ContributionReceived {amount: 800, OA}    │
│  Event 3: ContributionReceived {amount: 800, OA}    │
│  ...                                                │
│  Event 47,832: ContributionReceived {amount:1200}   │
│  Event 47,833: InvestmentTransfer {amount:50000}    │
│  → Current OA Balance: $248,500                     │
│                                                     │
│  BUSINESS RULE ENFORCED IN AGGREGATE:               │
│  OA withdrawal only permitted if:                   │
│  - Member age >= 55 (CPF Act, Section 15)          │
│  - Balance after withdrawal >= $20,000 (BRS)        │
│  - No existing lien on account                      │
│                                                     │
│  If rule violated → Emit WithdrawalRejected event  │
│  NEVER modify event history                         │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  EVENT STORE                                        │
│  Append-only. Immutable. Partitioned by memberId.  │
│                                                     │
│  Stream: cpf-account-{memberId}                    │
│  Retention: PERMANENT (CPF Act requirement)         │
│  Replication: 3× across Singapore AZs              │
│                                                     │
│  POINT-IN-TIME QUERY SUPPORT:                      │
│  "State on 2019-03-15" =                           │
│  Read events up to that timestamp → Replay         │
│  → Exact balance at that moment                    │
└──────────────────────┬──────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
┌─────────────────┐       ┌─────────────────────┐
│  BALANCE        │       │  AUDIT PROJECTION   │
│  PROJECTION     │       │                     │
│                 │       │  Pre-built view for │
│  Current state  │       │  Auditor-General    │
│  cached in      │       │  Office queries     │
│  PostgreSQL     │       │                     │
│  (fast reads    │       │  Includes: who      │
│  for member     │       │  initiated each     │
│  portal)        │       │  event, from which  │
│                 │       │  system, with which │
└─────────────────┘       │  authorisation      │
                          └─────────────────────┘
```

PATTERN APPLIED: Event Sourcing + CQRS with multiple projections (balance projection, audit projection, analytics projection)

WHY NOT traditional audit log table: Audit log as a secondary table is documentation, not source of truth. Event sourcing makes the log primary — projections are derived, never authoritative. This eliminates the class of bugs where state and log diverge.

WHY NOT blockchain for immutability: Blockchain provides distributed immutability — useful when multiple untrusting parties share a ledger. CPF Board is the sole fiduciary. Internal event store with cryptographic hash chaining (each event includes hash of previous event) provides immutability without blockchain's performance overhead. MAS has not mandated blockchain for government ledgers.

CRIME/LOOPHOLE ANGLE:
Event sourcing creates an audit trail that cannot be retroactively modified — but it does not prevent fraudulent events from being inserted in the first place. A privileged insider with write access to the event store could insert a fraudulent ContributionReceived event. Defence: event store write access is restricted to the CPF Account Aggregate service only, using managed identity. All events are digitally signed at emission. The Auditor-General's Office runs independent event hash verification monthly.

RESULT:
- Any historical balance query answerable by replaying events to a timestamp — auditor requirement satisfied
- Zero discrepancies between transaction log and account balance (structurally impossible in event sourced system)
- New analytics projection added in 3 weeks — required only building a new projection worker, no schema migration

---

### Kafka Deep Dive — What Actually Happens Inside

Architecture knowledge that separates senior engineers from solution architects:

```
KAFKA INTERNAL ARCHITECTURE — What you must understand:

Topic: upi-payment-events
Partitions: 16 (partitioned by payment_initiating_bank_id)
Replication Factor: 3

PARTITION 0 (Bank: HDFC):
Leader: Broker 1
Replicas: Broker 2, Broker 3

Offset: 0    [PaymentInitiated: txn-001, ₹500, HDFC→SBI]
Offset: 1    [PaymentInitiated: txn-002, ₹200, HDFC→ICICI]
Offset: 2    [PaymentCompleted: txn-001, confirmed]
Offset: 3    [PaymentFailed: txn-002, CBS timeout]
             ← Consumer reads from here (current offset)

CRITICAL CONCEPTS FOR ARCHITECTS:

1. ORDERING GUARANTEE:
   Kafka guarantees order WITHIN a partition, NOT across partitions.
   If txn-001 and txn-002 must be processed in order:
   → They must go to the SAME partition (same partition key).
   For UPI: all events for one payment use txnId as partition key.

2. CONSUMER GROUP SEMANTICS:
   Consumer Group: fraud-detection-service
   Each partition assigned to exactly ONE consumer in the group.
   16 partitions → 16 fraud detection instances max useful.
   Adding a 17th instance = one idle instance.
   
3. AT-LEAST-ONCE vs. EXACTLY-ONCE:
   Default: AT-LEAST-ONCE (consumer commits offset after processing)
   If consumer crashes after processing but before committing:
   → Message replayed on restart → processed twice.
   For financial events: EXACTLY-ONCE processing required.
   → Use Kafka transactions + idempotent producers.
   → Cost: ~20% throughput reduction. Worth it for money.

4. LOG COMPACTION vs. RETENTION:
   Retention (time-based): keep events for 7 days, then delete.
   Log compaction (key-based): keep only latest event per key.
   For event sourcing: use RETENTION (you need history).
   For configuration topics (current state only): use COMPACTION.

5. THE QUESTION NOBODY ASKS BUT SHOULD:
   "What is your consumer lag SLA?"
   If fraud detection consumer is 50,000 messages behind:
   Payments processed 50,000 events ago have not been fraud-checked.
   Consumer lag is a BUSINESS RISK metric, not just a technical metric.
   Present it in the weekly architecture health report.
```

---

🤖 COPILOT LIVE DEMO

WHEN: After explaining Kafka partitioning and consumer groups

TRAINER ACTION: Open Microsoft Copilot

PROMPT TO RUN:
"I am designing a Kafka-based event streaming architecture for UPI payment processing in India at 5,000 transactions per second. Help me determine: (1) how many partitions I need for the payment-events topic, (2) what partition key I should use and why, (3) whether to use at-least-once or exactly-once delivery semantics and the trade-offs, (4) what my consumer lag SLA should be for fraud detection."

CRITIQUE EXERCISE: Ask the class — "Did Copilot address the ordering guarantee requirement for a single payment's events? Did it explain the throughput cost of exactly-once semantics? What India-specific context did it miss — such as NPCI's settlement windows and the batch reconciliation that happens at end of day?"

TEACHING POINT: Kafka configuration is deeply context-dependent. Copilot gives you the framework — you supply the business context that makes the numbers real.

---

# SECTION 5: BREAK (11:10–11:20)

---

While you rest — A thought to carry:

"Workflow engines like Camunda and Temporal exist because someone built a distributed transaction system using raw Kafka consumers, got paged at 3am when it failed halfway through a 12-step process, and spent 6 hours figuring out which step it had reached before the crash. We answer how to never be that person in the next block."

🤖 OPTIONAL COPILOT PROMPT (try on your phone):
"What is the difference between orchestration and choreography in microservices event-driven architecture? When does each approach break down and why do workflow engines exist?"

Session resumes: 11:20 sharp

---

# SECTION 6: BLOCK 3 — ARCHITECTURE PATTERNS + CHAOS SCENARIOS (11:20–12:00)

---

## A. ARCHITECTURE PATTERNS DEEP DIVE

---

### Pattern 1: Saga Pattern — Distributed Transactions Without Two-Phase Commit

**Formal Definition:** A sequence of local transactions where each transaction publishes an event or message that triggers the next local transaction. If a step fails, compensating transactions are executed to undo the preceding steps. Two variants: Choreography (services react to events independently) and Orchestration (a central coordinator directs the saga).

**When to Use:**
- Multi-service business transactions that must span multiple Bounded Contexts
- When two-phase commit (2PC) is not viable due to service autonomy or performance requirements
- Long-running transactions (order fulfilment, loan processing, insurance claim) where steps may take hours

**When NOT to Use:**
- When you have fewer than 3 services involved — simpler patterns suffice
- When the compensating transaction semantics are unclear — if you cannot define "undo," you cannot Saga
- When the business cannot tolerate intermediate inconsistent states visible to users

**Choreography vs. Orchestration:**

```
CHOREOGRAPHY SAGA (Services react to events):
Each service listens for events and decides its own action.
No central coordinator.

PaymentService ──[PaymentInitiated]──▶ FraudService
FraudService ──[FraudCleared]──▶ BankSwitchService
BankSwitchService ──[DebitCompleted]──▶ NotificationService

ADVANTAGE: Loose coupling — services don't know about each other
PROBLEM: Saga logic distributed across services.
         Hard to see the full flow in one place.
         Debugging a stuck saga requires correlating events across
         multiple services' logs.

─────────────────────────────────────────────────────────────────

ORCHESTRATION SAGA (Central coordinator — use Temporal/Camunda):
One coordinator service manages the entire saga.
Each step is a call to a participant service.

         ┌─────────────────────────────┐
         │  UPI Payment Orchestrator   │
         │  (Temporal Workflow)        │
         │                             │
         │  Step 1: → FraudService     │
         │          ← FraudCleared     │
         │                             │
         │  Step 2: → DebitService     │
         │          ← DebitConfirmed   │
         │                             │
         │  Step 3: → BankSwitch       │
         │          ← CreditConfirmed  │
         │                             │
         │  Step 4: → Notification     │
         │          ← Notified         │
         │                             │
         │  IF Step 3 fails:           │
         │  → DebitService.compensate()│
         │  → Emit PaymentFailed       │
         └─────────────────────────────┘

ADVANTAGE: Saga logic visible in one place.
           Temporal persists saga state — crash-safe.
           Retry and timeout logic in one location.
COST: Orchestrator is a coordination service that must be managed.
```

---

### Pattern 2: Non-Blocking I/O + Reactive Patterns

**The Problem with Blocking:**

```
BLOCKING I/O (Thread-per-Request — Traditional Spring MVC):

Thread 1: PaymentRequest ──▶ [WAITING for Fraud API 800ms] ──▶ Response
Thread 2: PaymentRequest ──▶ [WAITING for Bank CBS 1200ms] ──▶ Response  
Thread 3: PaymentRequest ──▶ [WAITING for DB 50ms] ──▶ Response
...
Thread 200: PaymentRequest ──▶ [WAITING] ──▶ Response
Thread 201: PaymentRequest ──▶ REJECTED — Thread pool exhausted
            
At 5,000 TPS with 200-thread pool: 
200 threads × 1-second average wait = 200 concurrent requests max.
At thread 201: "Connection refused" errors.
Scaling: add more threads (memory cost) or add more servers (cost).

─────────────────────────────────────────────────────────────────

NON-BLOCKING I/O (Reactive — Spring WebFlux / Python asyncio):

Single thread: 
  PaymentRequest-1 → [async call to Fraud API, do NOT wait]
  PaymentRequest-2 → [async call to Bank CBS, do NOT wait]
  PaymentRequest-3 → [async call to DB, do NOT wait]
  ...
  [Fraud API responds] → continue PaymentRequest-1 processing
  [DB responds] → continue PaymentRequest-3 processing
  [Bank CBS responds] → continue PaymentRequest-2 processing

One thread handles thousands of concurrent requests.
Thread is never IDLE-WAITING — always processing a different request.

WHEN TO USE REACTIVE:
✅ High I/O, low CPU — payment routing, API gateway, proxy services
✅ High concurrency with many slow external calls
✅ Streaming data — real-time dashboards, event feeds

WHEN NOT TO USE REACTIVE:
❌ CPU-intensive work — machine learning inference, image processing
❌ Teams unfamiliar with async programming — debugging async stack 
   traces is significantly harder than blocking code
❌ Simple CRUD services — complexity overhead not worth it
```

---

## B. CHAOS & PRESSURE SIMULATION

---

🔥 CHAOS SCENARIO: The Kafka Lag Tsunami — UPI Settlement Window

SEVERITY: P0 | TIME: 23:45 IST, March 31 (Financial Year End) | GEOGRAPHY: 🇮🇳 India

THE INCIDENT (Trainer reads aloud):

"It is 23:45 IST on March 31. This is the last 15 minutes of the financial year. Every payment initiated before midnight must settle tonight or it counts in the wrong financial year for millions of businesses filing their GST returns. Your UPI settlement pipeline processes payment-completed events from Kafka, updates settlement accounts, and triggers end-of-day reconciliation with all partner banks.

At 23:47, your Kafka consumer lag monitoring fires a CRITICAL alert. The fraud-detection consumer group — which must process and clear every payment event before it reaches the settlement topic — is 2.3 million events behind. It has been falling behind since 22:30 when a code deployment added an expensive database lookup to the fraud scoring logic. Your settlement pipeline cannot process what fraud detection has not cleared. You have 13 minutes before midnight. At midnight, the settlement window closes. Any unprocessed payment is a failed settlement — affecting approximately 340,000 business transactions.

Your phone rings. It is the NPCI settlement desk. They want to know your ETA. Your director is in the room. The RBI operations team has sent an email asking for status. What do you do in the next 3 minutes?"

⏱️ TRAINER: 3 minutes silent thinking. Then open the floor.

BLAST RADIUS ANALYSIS:

Directly Affected: UPI settlement pipeline — 340,000 pending business transactions

Cascade Risk: If settlement misses midnight deadline — NPCI must file an exception report with RBI. Merchant bank accounts show incorrect end-of-day balance. GST reconciliation for 340,000 businesses affected. Next-day liquidity calculations wrong.

Data Risk: No data loss — Kafka retains messages. Risk is temporal: settlement window, not data integrity.

Financial Impact: Estimated ₹2,400 crore in transaction value pending settlement. ₹85 lakh in direct penalty exposure under NPCI settlement rules for missed windows.

WRONG APPROACHES:

❌ WRONG APPROACH 1: Rollback the deployment immediately.
WHY WRONG: Rolling back takes 8–12 minutes for a Java service. You have 13 minutes total. By the time rollback completes, you have missed the window. And rollback does not process the 2.3 million backed-up messages.

❌ WRONG APPROACH 2: Skip fraud detection for the pending messages — push directly to settlement.
WHY WRONG: Skipping fraud detection means potentially settling fraudulent transactions permanently. RBI guidelines require fraud screening before settlement. This creates a regulatory violation far worse than a missed settlement window.

CORRECT RESPONSE ARCHITECTURE:

IMMEDIATE (0–5 min):
- Scale the fraud detection consumer group horizontally — add 8 additional consumer instances immediately. Kafka's partition assignment rebalances within 30–60 seconds. With 16 partitions previously served by 4 consumers (4 partitions each), adding 8 consumers means each consumer handles 1 partition — 4× processing throughput
- Verify that the new deployment's expensive DB lookup can be short-circuited with a feature flag — disable the new lookup for existing consumers without rollback

SHORT-TERM (5–13 min):
- Monitor consumer lag metric in real time — target clearing 2.3M messages at 4× throughput = approximately 10 minutes if fraud scoring averages 50ms per event with 16 parallel consumers
- Communicate to NPCI: "Consumer lag reduction in progress, ETA 23:58 — requesting 15-minute settlement window extension"
- Prepare settlement pipeline to run in burst mode once fraud backlog clears

POST-INCIDENT:
- Architecture change: consumer lag SLA alert must fire at 50,000 messages behind — not 2.3 million. The alert threshold was set for normal operations; financial year end is not normal operations
- Consumer scaling must be automated — KEDA (Kubernetes Event-Driven Autoscaler) scales consumer replicas based on Kafka consumer lag metric automatically
- Deployment freeze window: no deployments between 20:00 and 02:00 on financial year-end dates

POST-MORTEM TEMPLATE:

TIMELINE:
- 22:30 — Fraud detection consumer lag begins growing (deployment introduced 200ms DB lookup per event)
- 23:47 — Alert fires at 2.3M message lag threshold (threshold too high — root cause of late detection)
- 23:49 — Incident commander assigned; consumer scaling initiated
- 23:52 — 8 additional consumer instances online; lag processing rate increases 4×
- 00:03 — Fraud backlog cleared; settlement pipeline running
- 00:18 — NPCI granted 20-minute extension; settlement completed
- 00:34 — All 340,000 transactions settled; NPCI notified

ROOT CAUSE (5-Why):
- Why 1: Settlement missed intended midnight deadline
- Why 2: Fraud detection consumer lag was 2.3M messages at 23:47
- Why 3: Deployment at 22:30 added 200ms synchronous DB lookup per event — reducing throughput from 50K events/min to 12K events/min
- Why 4: Deployment was not load-tested against financial year-end volume profile
- Why 5: Load test profiles used average daily volume, not peak seasonal volume — no financial year-end scenario in test suite

ACTION ITEMS:
- Implement KEDA-based auto-scaling for all Kafka consumer groups with lag-based scaling policy | Owner: Platform Engineering | Due: 14 days
- Add financial year-end, GST deadline, and budget day to load test scenario library | Owner: Performance Engineering | Due: 21 days
- Reduce consumer lag alert threshold to 50,000 messages for settlement-critical topics | Owner: Observability Team | Due: 3 days

---

# SECTION 7: CHECKPOINT 2 (12:00–12:10)

### 🎯 CHECKPOINT 2: Scenario Debate — Event-Driven Patterns

---

**Q1 — TYPE: PATTERN APPLICATION DEBATE**

"Two architects are designing the CPF monthly contribution processing pipeline. Architect A says: 'Use Event Sourcing for contributions — every debit and credit is an event, the balance is a projection, and we have full auditability.' Architect B says: 'Event Sourcing is over-engineering for what is essentially an accounting ledger. Use a standard double-entry accounting database with an immutable transaction table.' Pair up. Argue both positions for 4 minutes."

Expected resolution: Both positions are defensible. Event Sourcing wins on auditability and temporal queries ("what was the balance on date X?"). Double-entry accounting wins on operational simplicity and established tooling. The real answer for CPF: use double-entry accounting as the financial ledger (it is a proven pattern for money), augmented with an event log for audit — not full Event Sourcing. Event Sourcing is appropriate when you need to derive multiple different projections from the same history, or when temporal queries at arbitrary points are a core requirement. For a simple balance-plus-history model, it is often over-engineering.

---

**Q2 — TYPE: MANAGEMENT ESCALATION**

"Your CTO asks you in a 10-minute slot: 'Our competitors are all saying they use event-driven microservices. Should we migrate our monolith to microservices and Kafka?' You have 3 minutes to give a structured response. Go."

Strong answer structure:
- Question 1 back: "What problem are we solving?" — microservices solve specific problems (independent scaling, team autonomy, technology diversity). If those problems don't exist, microservices add complexity without benefit.
- Data point: The monolith is a competitive disadvantage only if it cannot be deployed independently by teams, cannot scale components independently, or cannot evolve without coordination delays.
- Risk: Migrating to microservices without the supporting DevOps maturity (CI/CD, observability, service mesh) creates a distributed monolith — all the complexity of microservices, none of the benefits.
- Recommendation framing: "Let me assess our top 3 scaling bottlenecks and team coordination friction points. If those trace to the monolith's structure, we have a business case for targeted decomposition. If they trace to process or DevOps maturity — we fix those first."

---

**Q3 — TYPE: CROSS-BORDER CONSIDERATION**

"The UPI-PayNow cross-border payment corridor processes a payment. The payer's bank in India debits the account (UPI side). Before the credit arrives at the Singapore PayNow side, the SGD exchange rate changes by 0.3%. The beneficiary receives slightly less SGD than the payer expected. This has happened 15,000 times this month. Describe the architecture of the event flow and identify where in the Saga the exchange rate lock must be acquired."

Strong answer:
- Exchange rate must be locked at payment initiation — not at credit time. This is a Saga step: AcquireExchangeRateLock must be Step 1, before DebitPayer
- The rate lock has an expiry (typically 30–60 seconds for FX) — the Saga must complete within that window or compensate
- If the Saga takes longer than the FX lock expiry: two options — re-quote (requires user re-confirmation) or absorb the difference within a tolerance band (bank policy decision)
- Architecture: the FX rate lock service is a participant in the cross-border payment orchestration Saga, with a compensating transaction (ReleaseFXLock) if any subsequent step fails

🤖 COPILOT VALIDATION — Run Live:

"Design the Saga pattern for a cross-border UPI to PayNow payment that must lock an exchange rate at initiation, debit in INR, convert, and credit in SGD — with compensating transactions if any step fails. Identify the orchestrator, participants, and compensating transactions for each step."

Class critique: Did Copilot identify the FX lock expiry as a time constraint on the Saga? Did it address what happens if the Saga is paused (user confirmation required) and the FX lock expires? Did it mention MAS or RBI regulatory requirements for cross-border payments?

---

# SECTION 8: BATTLE DRILL — HANDS-ON EXERCISE (12:10–12:50)

### ⚡ BATTLE DRILL 3: Design the Resilient Event Flow

TIME: 12:10–12:50 (40 minutes)
FORMAT: Teams of 3

---

SCENARIO BRIEF:

You are the architecture team at a Singapore-based fintech licensed under MAS as a Major Payment Institution. You are building a cross-border remittance platform — "RemitSG" — that allows Singapore residents to send money to India via the PayNow-UPI corridor. A remittance involves: customer authentication (SingPass), FX rate quotation, source account debit (Singapore bank), cross-border routing (MAS-approved FX channel), beneficiary credit (India UPI VPA), and confirmation notification (both sender and receiver). The entire flow must complete within 60 seconds or be automatically reversed. MAS requires a complete audit event trail for every remittance.

Your mission — 40 minutes to produce:

□ Event Flow Diagram: The complete event-driven architecture for RemitSG. Show every event emitted, every service that consumes it, and every compensating event for failure scenarios. Include the Saga orchestrator.

□ Kafka Topic Design: List every Kafka topic needed, partition key for each, retention policy, and whether at-least-once or exactly-once semantics are required per topic.

□ CAP Decision Table: For each service in your architecture (minimum 5 services), document explicitly — CP or AP, and one sentence justifying the choice.

□ Failure Mode Analysis: Identify the top 3 failure scenarios in your design and the exact recovery mechanism for each (not "retry" — the specific compensating event or rollback step).

TOOLS AVAILABLE:
□ Whiteboard / Paper
□ Microsoft Copilot (validate after drafting — do not generate first)
□ Personal notes from today's session

🤖 COPILOT RULES FOR THIS DRILL:
- Allowed: After drafting your Kafka topic design, ask Copilot to review it for partition key mistakes
- Allowed: Use Copilot to check if your Saga compensating transactions are complete
- Not Allowed: Ask Copilot to design the event flow before you have drawn it yourself

EVALUATION RUBRIC:

| Criterion                                                            | Score |
| -------------------------------------------------------------------- | ----- |
| Event Flow Completeness (all 6 remittance steps covered with events) | /10   |
| Failure Handling (3 explicit failure modes with compensating events) | /10   |
| Kafka Design (partition keys justified, retention appropriate)       | /10   |
| CAP Decisions (5+ services with explicit CP/AP justification)        | /10   |
| MAS Compliance (audit trail mechanism explicitly shown)              | /10   |
| Copilot Usage Quality (prompts precise, output critically evaluated) | /5    |
| Communication Clarity (can explain to MAS auditor in 5 minutes)      | /5    |
| TOTAL                                                                | /60   |

FEEDBACK TEMPLATE:

WHAT WORKED: _______________________________________________________________

WHAT'S MISSING: ___________________________________________________________

WHAT A PRINCIPAL ARCHITECT WOULD ADD: ______________________________________

CERTIFICATION ALIGNMENT: Maps to Azure Solutions Architect Expert — Design application architecture (event-driven architecture, message queues, reliable messaging). Also maps to AWS SAP — Design resilient architectures with event-driven patterns.

---

# SECTION 9: DOCUMENTATION TEMPLATES + PROFESSIONAL COMMUNICATION

### ⬆️ LEVELLING UP: INCIDENT POST-MORTEM + TECHNOLOGY EVALUATION

---

Today's template: Technology Evaluation Matrix for Message Broker Selection — a decision an architect must make defensibly, not by gut feel.

---

TECHNOLOGY EVALUATION: Message Broker Selection for Government-Scale Event Platform

| CRITERIA                | Weight | Apache Kafka | Azure Service Bus | RabbitMQ | Notes                                               |
| ----------------------- | ------ | ------------ | ----------------- | -------- | --------------------------------------------------- |
| Throughput (msgs/sec)   | 25%    | 9/10         | 7/10              | 6/10     | Kafka: millions/sec; ASB: 10K/sec; RMQ: 50K/sec     |
| Message Ordering        | 15%    | 8/10         | 8/10              | 6/10     | Kafka: per-partition; ASB: sessions; RMQ: per-queue |
| Event Replay            | 20%    | 10/10        | 4/10              | 2/10     | Kafka: full log retention; ASB/RMQ: limited         |
| Operational Complexity  | 10%    | 5/10         | 9/10              | 7/10     | Kafka: high ops burden; ASB: managed service        |
| MAS/RBI Compliance Docs | 15%    | 7/10         | 9/10              | 6/10     | Azure: MAS-recognised cloud; Kafka: self-managed    |
| Vendor Risk             | 10%    | 9/10         | 6/10              | 8/10     | Kafka: open source; ASB: Azure lock-in              |
| Team Capability         | 5%     | 7/10         | 8/10              | 8/10     | Assumes existing Java/cloud team                    |
| **WEIGHTED SCORE**      | 100%   | **8.05**     | **7.30**          | **5.85** |                                                     |
| **RECOMMENDATION**      |        | ✅ Kafka      |                   |          | For event sourcing + replay requirement             |

RECOMMENDATION RATIONALE:
Kafka's event replay capability (scored 10/10, weighted 20%) is non-negotiable for the Event Sourcing pattern required by CPF-style audit requirements. Azure Service Bus, while operationally simpler and MAS-documented, does not support log replay — messages are consumed and deleted. For a remittance platform requiring 5-year audit trail replay, Kafka is the only viable option. Azure Service Bus is recommended for notification and alerting topics where replay is not required — reducing operational overhead for non-critical paths.

---

INCIDENT POST-MORTEM REPORT: UPI Settlement Consumer Lag — Financial Year End

Severity: P0 | Status: CLOSED | Date: April 1, 2026

EXECUTIVE SUMMARY (For CTO/Board — 5 sentences):
On March 31 at 23:47 IST, the UPI settlement pipeline experienced a critical consumer lag event in the fraud detection consumer group, reaching 2.3 million unprocessed events. The root cause was a deployment at 22:30 that inadvertently increased per-event processing time by 4×. Immediate scaling of consumer instances recovered processing throughput; NPCI granted a 20-minute settlement window extension. All 340,000 pending transactions settled successfully by 00:34 IST. Three architecture-level remediation actions have been initiated — consumer auto-scaling, alert threshold adjustment, and financial calendar load testing — with completion deadlines within 21 days.

TIMELINE:
- 22:30 — Deployment of fraud-detection-service v2.4.1 (introduced 200ms DB lookup)
- 23:47 — Kafka consumer lag alert fires at 2.3M message threshold
- 23:48 — Incident commander assigned; war room opened
- 23:49 — Root cause identified (deployment correlation)
- 23:51 — NPCI notified; extension requested
- 23:52 — 8 additional consumer instances deployed; rebalancing initiated
- 00:03 — Consumer lag cleared; settlement pipeline activated
- 00:18 — NPCI extension granted (20 minutes)
- 00:34 — Settlement complete; incident closed

ROOT CAUSE ANALYSIS (5-Why):
- Why 1: Settlement missed midnight deadline
- Why 2: Fraud detection consumer processing rate dropped 75% at 22:30
- Why 3: v2.4.1 deployment added synchronous PostgreSQL lookup for each event (200ms per event)
- Why 4: Deployment was not performance-tested against financial year-end volume
- Why 5: Load test suite contained only average daily volume profiles — no seasonal peak scenarios

ACTION ITEMS:

| Item                                                             | Owner         | Priority | Due Date | Status      |
| ---------------------------------------------------------------- | ------------- | -------- | -------- | ----------- |
| Implement KEDA auto-scaling for fraud-detection consumers        | Platform Eng  | P0       | Apr 14   | In Progress |
| Add financial year-end to load test scenarios                    | Perf Eng      | P1       | Apr 21   | Not Started |
| Reduce consumer lag alert threshold to 50K messages              | Observability | P0       | Apr 3    | Completed   |
| Mandatory perf gate in deployment pipeline for consumer services | DevOps        | P1       | Apr 14   | In Progress |

---

# SECTION 10: MICROSOFT COPILOT INTEGRATION

### 🤖 COPILOT PROMPT BANK — DAY 3

**TYPE 1: CONCEPT VALIDATION (During CAP Theorem teaching)**

WHEN: After presenting the CAP decision matrix

PROMPT TO RUN:
"For each of the following UPI system components, determine whether CP (Consistency + Partition Tolerance) or AP (Availability + Partition Tolerance) is the correct CAP choice and explain why: (1) Payment debit decision, (2) Transaction status display, (3) Fraud score lookup, (4) Push notification delivery, (5) End-of-day settlement processing. Include RBI regulatory implications for each choice."

CRITIQUE EXERCISE: Ask the class — "Did Copilot correctly identify that fraud score lookup should be AP — and that the RBI implication is that fraud screening can use slightly stale scores but the final settlement audit must be CP? Did it address the settlement window regulatory constraint?"

---

**TYPE 2: DOCUMENT GENERATION (During Saga pattern teaching)**

COPILOT PROMPT:
"Act as a senior solution architect designing a Saga pattern for cross-border payment processing between India (UPI) and Singapore (PayNow). Generate: (1) the orchestration Saga step sequence with compensation steps, (2) the Kafka topic design for the Saga events, (3) the timeout handling strategy for each step. Ensure compliance with both RBI cross-border payment guidelines and MAS Major Payment Institution requirements."

TRAINER INSTRUCTIONS: Run live. Show the class how adding "Include the specific MAS MPI licence condition for cross-border FX" to the prompt improves the regulatory specificity of the output. Prompt iteration is a professional skill.

---

**TYPE 3: ARCHITECTURE REVIEW (After Battle Drill)**

COPILOT REVIEW PROMPT:
"Review this event-driven architecture for a Singapore cross-border remittance platform. Identify: (1) missing compensating transactions in the Saga, (2) Kafka partition key mistakes that could cause ordering violations, (3) services that have the wrong CAP designation, (4) MAS Technology Risk Management 2021 compliance gaps. Architecture description: [candidate's architecture pasted]"

---

**TYPE 4: RESEARCH ACCELERATION (For daily assignment)**

COPILOT RESEARCH PROMPT:
"Summarise the architectural decisions behind India's CoWIN vaccine booking platform's CQRS implementation and Kafka-based event streaming. What specific challenges did the team face at 3,000 requests per second and how were they resolved? Focus on lessons relevant to a solution architect preparing for Azure Solutions Architect Expert certification."

---

# SECTION 11: DAILY ASSIGNMENT (13:05–13:15)

### 📋 ARCHITECTURE KATA: Day 3 — Design the GSTN Real-Time E-Invoice Event Platform

---

DAILY ASSIGNMENT: Day 3
Estimated Effort: 90–120 minutes | Due: Before Day 4 starts (Jun 16, 09:00)

SCENARIO:
You are the Solution Architect at GSTN (Goods and Services Tax Network). MeitY has mandated that e-invoicing must become real-time by April 2027 — currently, invoice registration takes up to 5 minutes under the IRP (Invoice Registration Portal) architecture. The new architecture must: process 50 million invoices per day (peak: 8 million between 09:00–11:00 IST on the last 3 days of every month), provide real-time fraud detection (invoices used for fake GST credit claims — a ₹2,000 crore annual problem), give the tax authority a real-time view of all invoices in their jurisdiction, and maintain a 5-year immutable audit trail of every invoice state transition. The system must use CQRS and Event Sourcing. Kafka is the mandated event streaming platform.

DELIVERABLES:

□ 1. Event-Driven Architecture Diagram: Complete CQRS + Event Sourcing design. Show: command side, event store, Kafka topics, projection workers, query side (minimum 3 projections: real-time tax authority dashboard, fraud detection feed, merchant invoice history)
□ 2. Kafka Topic Design Table: Topic name, partition key, retention policy, consumer groups, at-least-once vs. exactly-once semantics — for every topic in your design
□ 3. CAP Decision Table: Every service with explicit CP vs. AP decision and regulatory justification (DPDP Act 2023, CERT-In, RBI where applicable)
□ 4. Saga Design: The invoice registration Saga — steps, participants, compensating transactions for: (a) fraud detection rejects the invoice, (b) IRP hash generation fails, (c) network timeout after debit but before credit
□ 5. Consumer Lag SLA Document: Define consumer lag alert thresholds for each consumer group, the auto-scaling trigger, and the escalation path if lag cannot be cleared before the GST filing deadline

ASPECT FOCUS FOR TODAY: Event-Driven Architecture, Kafka Design, Resilience at Scale

SELF-EVALUATION CHECKLIST:

Before submitting, verify:
□ Does my event store design support temporal queries ("show me this invoice's state on date X")?
□ Does my Kafka partition key guarantee ordering for all events on the same invoice?
□ Are all my CAP decisions explicitly justified with regulatory context?
□ Does my Saga have a compensating transaction for EVERY step — not just the obvious ones?
□ Is my consumer lag SLA calibrated for GST filing deadline peak load, not average daily load?
□ Did I use Copilot to review and document what it added vs. what I rejected?

COPILOT CHALLENGE:
"Review this Kafka topic design for a GST e-invoice platform processing 50 million invoices per day. Check for: partition key ordering violations, retention policy gaps for 5-year audit requirement, missing dead-letter queue configuration, and consumer group isolation issues. Topic design: [paste your design]"

---

# SECTION 12: FOOD FOR THOUGHT + NEXT DAY PREVIEW (13:15–13:30)

### 🧠 ARCHITECT'S MEDITATION: Day 3

---

💭 PROVOCATIONS (No right answer — discuss with a peer tonight):

1. Event Sourcing gives you an immutable audit trail. But who audits the audit trail? If someone with privileged access to the Kafka broker inserts a fraudulent event — say, a ₹50 lakh bank transfer event that never actually happened — your event-sourced system will faithfully compute a balance that includes the fraud. The immutability that protects you from accidental corruption also protects a sophisticated attacker from detection. How do you architect against this?

2. India processes 11 billion UPI transactions per month. Every transaction is an event. Stored for 5 years per RBI guidelines — that is potentially 660 billion events. At 500 bytes per event average: 330 terabytes of event data. This is not a storage problem — modern storage handles this. But replaying 330 terabytes of events to reconstruct a single account's state takes weeks. Event Sourcing's promise of "reconstruct any state from history" starts to break down at this scale. What architectural pattern solves this? (Hint: think about snapshots. We cover this in Day 4's data architecture session.)

3. The Saga pattern says: if step 5 fails, run compensating transactions for steps 1–4. But compensation is not reversal. A sent notification cannot be unsent. An SMS saying "your payment was successful" cannot be recalled after the payment fails in step 6. In financial systems, what is the user experience obligation when a Saga's compensation makes the system consistent again, but the user has already acted on incorrect information?

READ TONIGHT:

🔍 Search: "NPCI UPI architecture microservices event driven 2023"
🔍 Search: "Temporal workflow engine distributed saga pattern"
🔍 Search: "Kafka exactly once semantics production trade-offs"
📖 Book Reference: "Designing Data-Intensive Applications" by Martin Kleppmann — Chapter 11: "Stream Processing" (the definitive reference for everything covered today)

ANTI-PATTERN TO RESEARCH:

🚫 "The Dual Write Problem" — Research what happens when a service writes to a database AND publishes an event to Kafka in the same operation, without a transaction spanning both. What is the failure mode? What are the two standard solutions (Outbox Pattern and Event Sourcing)?

TOMORROW'S TEASER:

👀 Day 4 Preview: Data Architecture, NoSQL & Search

"Tomorrow we solve a problem that will make today's event architecture richer: where do you actually store 660 billion events? And when you need to answer 'find all suspicious invoices from merchants in Karnataka filed between September and December 2025' in under 2 seconds — what database do you use and why is the answer almost certainly not PostgreSQL? We also cover the one data architecture decision that has killed more government digital transformation projects than any other: polyglot persistence without a data ownership strategy."

We will explore: MongoDB vs. Redis vs. Neo4J vs. DynamoDB use cases, Sharding and partitioning for global scale, Search architecture, Consistency models

🎯 Come prepared to answer:
"You have a citizen data platform. One query: 'Find all citizens in Singapore who have a CPF balance above S$500,000, live in the Central region, and have not logged into SingPass in the last 18 months' — for a government outreach programme. What database, what index strategy, and what is your response time SLA?"

---