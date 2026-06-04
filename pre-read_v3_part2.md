# Pre-Read Documentation — Part 2

## Microservices, AI & Modernization

### Aligned to Course Phase 2 | Days 7–11 | Jun 19–25, 2026

---

> **Reading Note:** This is Part 2 of the Pre-Read Documentation. If you have not read Part 1, return to it first. The concepts in Part 2 build directly on Part 1 foundations. Glossary and abbreviations are in Section 2 of Part 1.

---

# Part 2 Table of Contents

- [Chapter 2.1 — Advanced Microservices & Mobile-First Design](#)
- [Chapter 2.2 — Legacy Modernization & Migration Playbook](#)
- [Chapter 2.3 — AI-Assisted Development & Validation](#)
- [Chapter 2.4 — Performance Engineering & Observability](#)
- [Chapter 2.5 — Project Milestone 2 Preview](#)
- [Part 2 — Terms to Google](#)

---

## Chapter 2.1 — Advanced Microservices & Mobile-First Design

### 2.1.1 What "Advanced" Means in Microservices

By now you understand what microservices are — independently deployable services, each owning its own data. What most engineers do not fully grasp until they operate microservices in production is the operational and design complexity that comes with this style.

The problems you almost never encounter in a monolith become daily realities in a microservices system:

```
┌─────────────────────────────────────────────────────────────────────────┐
│          PROBLEMS THAT APPEAR ONLY IN MICROSERVICES PRODUCTION         │
│                                                                         │
│  IN A MONOLITH              IN MICROSERVICES                           │
│  ──────────────             ────────────────                           │
│  Function call fails?       Network call fails? Timeout?               │
│  → Stack trace is clear     → Which service? Which hop?                │
│                                                                         │
│  Transaction rolls back?    Distributed transaction spans 4 services?  │
│  → ACID handles it          → Partial commits. Who compensates?        │
│                                                                         │
│  Method called twice?       API called twice due to retry?             │
│  → Idempotent by default    → Double processing. Double payment?       │
│                                                                         │
│  One slow database query?   One slow downstream service?               │
│  → Visible in profiler      → Thread pool exhausted. Cascades          │
│                               to all callers. System-wide slowdown.   │
│                                                                         │
│  Deploy update?             Deploy microservice update?                │
│  → One WAR file             → Which version is compatible with which? │
│                               Contract breaking change? API version?  │
│                                                                         │
│  These are the problems Days 7–11 are designed to solve.              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.1.2 Mobile-First Architecture

**What Mobile-First Architecture Means for Backend Engineers**

Mobile-first does not mean designing a UI that looks good on small screens. For a backend architect, it means designing the system under the assumption that a significant portion of users are on:

- Devices with intermittent or slow connectivity (2G/3G in rural India)
- Devices with limited battery and compute
- Networks with high latency and packet loss
- Connections that drop unexpectedly

This changes almost every backend decision.

```
┌─────────────────────────────────────────────────────────────────────────┐
│         MOBILE-FIRST BACKEND ARCHITECTURE PRINCIPLES                    │
│                                                                         │
│  PRINCIPLE 1: OFFLINE-FIRST WITH SYNC                                  │
│  ─────────────────────────────────────                                 │
│  The app must work without connectivity. Changes sync when online.     │
│                                                                         │
│  Example: PM-KISAN mobile app for farmers                              │
│  • Farmer fills land details offline                                    │
│  • App stores in local SQLite (on device)                              │
│  • When connectivity resumes, syncs to central server                  │
│  • Conflict resolution: "Server wins" or "Last write wins" defined     │
│    explicitly in architecture (not left to chance)                     │
│                                                                         │
│  SYNC ARCHITECTURE:                                                     │
│                                                                         │
│  ┌──────────────┐          ┌───────────────────────────────────┐       │
│  │Mobile Device │          │       Backend                     │       │
│  │              │          │                                   │       │
│  │ Local DB     │◄────────►│  Sync Service                    │       │
│  │ (SQLite /    │  Sync    │  • Differential sync             │       │
│  │  Realm)      │  API     │    (only changed records)        │       │
│  │              │          │  • Conflict detection            │       │
│  │ Change log   │──────────►  • Idempotent apply              │       │
│  │ (delta)      │  Push    │  • Retry with exponential        │       │
│  │              │  changes │    backoff                       │       │
│  └──────────────┘          └───────────────────────────────────┘       │
│                                                                         │
│  PRINCIPLE 2: EDGE CACHING                                              │
│  ─────────────────────────                                             │
│  Serve static and semi-static content from edge nodes close to user.  │
│                                                                         │
│  Without edge caching:                                                  │
│  Mobile in Leh, Ladakh → Request hits server in Mumbai data centre    │
│  Round trip: ~120ms base latency + processing                          │
│                                                                         │
│  With edge caching (CDN):                                               │
│  Mobile in Leh → Hits nearest CDN PoP → Served in ~8ms               │
│                                                                         │
│  What to cache at edge:                                                 │
│  • Government forms (PDF templates)                                    │
│  • Scheme eligibility criteria (changes weekly, not per request)       │
│  • Geographic reference data (state, district, block codes)            │
│  • Public holiday calendars                                             │
│                                                                         │
│  What NOT to cache at edge:                                             │
│  • Personalised citizen data                                            │
│  • Payment transaction status                                           │
│  • Real-time inventory (ration stock levels)                           │
│                                                                         │
│  PRINCIPLE 3: API PAYLOAD OPTIMISATION                                  │
│  ─────────────────────────────────────                                 │
│  Mobile clients on 2G cannot afford 200KB JSON responses.             │
│                                                                         │
│  Techniques:                                                            │
│  • GraphQL: Client requests only fields it needs                       │
│  • Sparse fieldsets in REST: GET /citizens/{id}?fields=name,mobile    │
│  • Protocol Buffers (Protobuf): Binary serialisation, 3–10x smaller   │
│  • Pagination: Never return unbounded lists                            │
│  • Compression: gzip/brotli on all responses                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Production Case — India Post Payments Bank (IPPB)**

> IPPB serves customers at 1.36 lakh post offices, many in rural areas with poor connectivity. The mobile app needed to function at the last mile.
>
> **Architecture Choices Made:**
> - Offline balance caching with 15-minute staleness window (acceptable for micro-credit users)
> - Aadhaar-based biometric authentication integrated with offline fallback to PIN when biometric service unreachable
> - Minimum API response payload: <8KB per screen (enforced at API gateway level via payload size monitoring)
> - USSD fallback channel for feature phones with no smartphone capability
>
> **What This Required in Backend Architecture:**
> - BFF (Backend For Frontend) pattern: A dedicated API layer for the mobile app that aggregates data from multiple microservices and returns only what the mobile screen needs
> - Separate BFF for web/desktop (different data requirements, no need for payload optimisation)
> - API response time SLO: 95th percentile under 800ms (generous for mobile network conditions)

---

### 2.1.3 The SAGA Pattern — Distributed Transactions Without 2-Phase Commit

**The Core Problem**

In a monolith with a single database, transactions are managed by ACID. You call `BEGIN`, do your operations, and call `COMMIT` or `ROLLBACK`. All or nothing.

In microservices, a business transaction might span:
- Service A (Citizen profile) → writes to Database A
- Service B (Payment) → writes to Database B
- Service C (Document issuance) → writes to Database C

You cannot call `BEGIN` across three databases owned by three services. This is the distributed transaction problem.

**2-Phase Commit (2PC) — Why We Avoid It**

2PC is a protocol where a coordinator asks all participants to prepare (phase 1), then tells them all to commit (phase 2). The problem:
- The coordinator becomes a single point of failure
- All participants are locked during the protocol
- In a network partition, participants can be left in a prepared state forever (blocking)
- Does not scale beyond a handful of services

**SAGA Pattern — The Microservices Solution**

A SAGA breaks a distributed business transaction into a sequence of local transactions, each of which publishes an event or message that triggers the next step. If any step fails, compensating transactions undo the previous steps.

```
┌─────────────────────────────────────────────────────────────────────────┐
│         SAGA PATTERN: PASSPORT APPLICATION PROCESS                      │
│         (Modelled on Passport Seva Portal, India)                       │
│                                                                         │
│  HAPPY PATH (Choreography-based SAGA):                                  │
│                                                                         │
│  Step 1: ApplicationService                                             │
│  ─────────────────────────                                              │
│  Action: Create application record, reserve appointment slot           │
│  Event Published: ApplicationCreated                                    │
│          │                                                              │
│          ▼                                                              │
│  Step 2: PaymentService (listens for ApplicationCreated)               │
│  ─────────────────────────────────────────────────────                 │
│  Action: Charge fee to citizen's linked payment method                 │
│  Event Published: PaymentSucceeded                                      │
│          │                                                              │
│          ▼                                                              │
│  Step 3: PoliceVerificationService (listens for PaymentSucceeded)      │
│  ─────────────────────────────────────────────────────                 │
│  Action: Queue police verification request                              │
│  Event Published: VerificationQueued                                    │
│          │                                                              │
│          ▼                                                              │
│  Step 4: PassportPrintingService (listens for VerificationCleared)     │
│  ─────────────────────────────────────────────────────                 │
│  Action: Schedule passport printing and dispatch                        │
│  Event Published: PassportDispatched                                    │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  FAILURE PATH (with Compensating Transactions):                         │
│                                                                         │
│  Step 1: ApplicationService → ApplicationCreated ✅                    │
│          │                                                              │
│          ▼                                                              │
│  Step 2: PaymentService → Payment FAILS (insufficient balance)  ❌     │
│          │                                                              │
│          │ Publishes: PaymentFailed                                     │
│          ▼                                                              │
│  COMPENSATION: ApplicationService (listens for PaymentFailed)          │
│  ────────────────────────────────────────────────────────────          │
│  Action: Release reserved appointment slot                              │
│  Action: Mark application as PAYMENT_FAILED                            │
│  Action: Notify citizen via SMS/email                                   │
│                                                                         │
│  RESULT: No data inconsistency. Appointment slot is freed.             │
│  Citizen is informed. No dangling records.                             │
│                                                                         │
│  KEY POINT: No step waits for another service to confirm.             │
│  All communication is asynchronous via events.                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Choreography vs Orchestration in SAGAs**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CHOREOGRAPHY                    ORCHESTRATION                          │
│  (Event-driven, decentralised)   (Central coordinator)                 │
│                                                                         │
│  Services react to events        A Saga Orchestrator                   │
│  from other services.            tells services what to do.            │
│                                                                         │
│  Service A publishes event →     Orchestrator → calls Service A        │
│  Service B reacts →              Orchestrator ← A responds             │
│  Service C reacts                Orchestrator → calls Service B        │
│                                  Orchestrator ← B responds             │
│                                                                         │
│  PRO: No central component       PRO: Easy to visualise and debug      │
│  PRO: Loose coupling             PRO: Explicit saga state visible      │
│  CON: Hard to trace the flow     CON: Orchestrator is a central piece  │
│  CON: Risk of cyclic events      CON: Additional service to manage     │
│                                                                         │
│  CHOOSE CHOREOGRAPHY WHEN:       CHOOSE ORCHESTRATION WHEN:            │
│  • Few steps (≤ 4)              • Many steps (5+)                     │
│  • Team is distributed          • Complex compensation needed          │
│  • Loose coupling is priority   • Auditability is priority            │
│                                  • Temporal/Camunda available          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.1.4 Idempotency — Making Retries Safe

**Why Retries Happen**

In any distributed system, network calls can fail due to timeouts, transient errors, or network blips. The correct response is to retry. But retrying a non-idempotent operation is dangerous.

```
SCENARIO: Farmer subsidy disbursement (PM-KISAN)
─────────────────────────────────────────────────

1. PaymentService calls BankTransferService:
   POST /transfers { beneficiaryId: FARMER-001, amount: 2000 }

2. BankTransferService processes the payment. ₹2,000 debited.

3. Network timeout occurs BEFORE the response reaches PaymentService.

4. PaymentService does not know: Did the payment succeed or fail?
   It retries:
   POST /transfers { beneficiaryId: FARMER-001, amount: 2000 }

5. WITHOUT IDEMPOTENCY: ₹4,000 debited. Farmer overpaid.
   This is a real problem in production payment systems.

6. WITH IDEMPOTENCY KEY:
   POST /transfers
   Idempotency-Key: PM-KISAN-FARMER-001-Q2-2024
   { beneficiaryId: FARMER-001, amount: 2000 }
   
   BankTransferService stores the Idempotency-Key with the result.
   On retry, it sees the same key, returns the stored result.
   No duplicate processing. ₹2,000 disbursed exactly once.
```

**Implementing Idempotency:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 IDEMPOTENCY IMPLEMENTATION PATTERN                      │
│                                                                         │
│  STEP 1: Client generates unique Idempotency-Key                       │
│  ─────────────────────────────────────────────                         │
│  Key format: {operation}-{entityId}-{period}                           │
│  Example: "transfer-FARMER001-2024Q2"                                  │
│  Sent in header: Idempotency-Key: transfer-FARMER001-2024Q2            │
│                                                                         │
│  STEP 2: Server receives request                                        │
│  ────────────────────────────                                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  Receive Request                                    │               │
│  │         │                                           │               │
│  │         ▼                                           │               │
│  │  Check idempotency store (Redis/DB)                │               │
│  │  for this key                                       │               │
│  │         │                                           │               │
│  │    ┌────┴────┐                                      │               │
│  │    │         │                                      │               │
│  │  FOUND     NOT FOUND                                │               │
│  │    │         │                                      │               │
│  │    ▼         ▼                                      │               │
│  │  Return    Process request                          │               │
│  │  cached    Store result in idempotency              │               │
│  │  result    store with TTL (e.g., 24 hours)         │               │
│  │            Return result                            │               │
│  └─────────────────────────────────────────────────────┘               │
│                                                                         │
│  STEP 3: Idempotency Store (Redis with TTL)                            │
│  ─────────────────────────────────────────                             │
│  Key: "transfer-FARMER001-2024Q2"                                      │
│  Value: { status: SUCCESS, transferId: TXN-98765, amount: 2000 }      │
│  TTL: 24 hours (after which, same key = new operation)                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.1.5 High Concurrency Patterns — Rate Limiting, Bulkheads & Circuit Breakers

**The Last-Mile Rush Problem**

Government portals face extreme burst traffic patterns that commercial systems rarely see. Last day of GST filing, last day of income tax submission, last day of scholarship applications — citizens behave as a herd.

```
TYPICAL TRAFFIC PATTERN - INCOME TAX PORTAL (India)
─────────────────────────────────────────────────────

Normal Days (Jan-Jun):
Requests/min:  ████ (50,000)

Filing Season (Jul):
Requests/min:  ████████████████████████████████████ (400,000)

Last 3 Days Before Deadline:
Requests/min:  ██████████████████████████████████████████████████ (900,000)

Last 6 Hours:
               System cannot cope → Citizens face errors → 
               Public outrage → Emergency deadline extension

This pattern has repeated every year until infrastructure 
was redesigned for elastic scaling with pre-provisioned capacity.
```

**Rate Limiting — Protecting Your Service**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RATE LIMITING STRATEGIES                             │
│                                                                         │
│  FIXED WINDOW:                                                          │
│  Allow 1000 requests per minute per user.                              │
│  Window resets every minute at the clock boundary.                     │
│  Problem: User can make 1000 requests at 11:59 and 1000 at 12:00      │
│           = 2000 in 2 seconds. Window boundary exploitation.           │
│                                                                         │
│  SLIDING WINDOW:                                                        │
│  Allow 1000 requests in any rolling 60-second window.                 │
│  Prevents boundary exploitation.                                        │
│  Implementation: Redis sorted set with timestamp as score.             │
│                                                                         │
│  TOKEN BUCKET:                                                          │
│  Bucket holds N tokens. Each request consumes 1 token.                │
│  Bucket refills at rate R tokens/second.                               │
│  Allows burst up to bucket capacity.                                   │
│  Good for: APIs that want to allow short bursts but control average.  │
│                                                                         │
│  RATE LIMIT TIERS (Government Portal Example):                         │
│  ──────────────────────────────────────────────                        │
│  ┌─────────────────────────────────────────────────────┐               │
│  │  Tier        │  Limit          │  Who                │              │
│  │  ──────────  │  ─────────────  │  ────────────────   │              │
│  │  Anonymous   │  10 req/min     │  Unauthenticated    │              │
│  │  Citizen     │  60 req/min     │  Authenticated user │              │
│  │  CA/Agent    │  300 req/min    │  Tax practitioner   │              │
│  │  System      │  5,000 req/min  │  B2B API consumers  │              │
│  │  Internal    │  Unlimited      │  Internal services  │              │
│  └─────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Bulkhead Pattern — Fault Isolation**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BULKHEAD PATTERN ILLUSTRATED                         │
│                                                                         │
│  WITHOUT BULKHEAD:                                                      │
│  ──────────────────                                                     │
│  One shared thread pool for all downstream calls.                      │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  THREAD POOL (20 threads)                                │          │
│  │  ████████████████████ ← All 20 occupied calling         │          │
│  │                         slow Police Verification API    │          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  RESULT: Even fast calls (Payment Status, Citizen Lookup) are blocked. │
│  One slow downstream service brings down all functionality.            │
│                                                                         │
│  WITH BULKHEAD:                                                         │
│  ──────────────                                                         │
│  Separate thread pools per downstream dependency.                      │
│                                                                         │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐        │
│  │ Police Verify  │  │  Payment        │  │  Citizen Lookup  │        │
│  │ Thread Pool    │  │  Thread Pool    │  │  Thread Pool     │        │
│  │  (5 threads)  │  │  (10 threads)   │  │  (5 threads)     │        │
│  │  ██████ ← 5   │  │  ░░░░░░░░░░     │  │  ░░░░░░          │        │
│  │  blocked      │  │  8 threads free │  │  3 threads free  │        │
│  └────────────────┘  └─────────────────┘  └──────────────────┘        │
│                                                                         │
│  RESULT: Police Verification slowdown affects only its 5 threads.     │
│  Payment and Citizen Lookup continue to serve requests normally.       │
│  Failure is contained. The ship does not sink.                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Circuit Breaker — State Machine**

```
                    CIRCUIT BREAKER STATE MACHINE
                    ──────────────────────────────

                    Requests flowing normally
                           │
                           ▼
                    ┌─────────────┐
                    │   CLOSED    │ ← Normal operation
                    │             │   All requests pass through
                    └──────┬──────┘
                           │
                    Failure threshold exceeded
                    (e.g., 5 failures in 10 seconds)
                           │
                           ▼
                    ┌─────────────┐
                    │    OPEN     │ ← Fail fast
                    │             │   No requests sent to downstream
                    │             │   Returns error immediately
                    └──────┬──────┘   (saves threads and time)
                           │
                    After timeout period
                    (e.g., 30 seconds)
                           │
                           ▼
                    ┌─────────────┐
                    │  HALF-OPEN  │ ← Allow 1 test request
                    │             │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         SUCCESS                    FAILURE
              │                         │
              ▼                         ▼
         CLOSED again              OPEN again
         (reset)                   (wait more)


REAL INCIDENT WHERE THIS MATTERED:

COWIN Vaccination System, May 2021:
During the 18–44 age group registration opening, the OTP 
verification service (calling telecom operators' SMS gateway) 
became overwhelmed. Without circuit breakers, every API 
request waited 30 seconds for OTP timeout before failing.
Server thread pools filled with waiting threads.
System appeared to hang for all users.

With circuit breakers: After first batch of OTP failures,
circuit opens. Subsequent calls fail fast (in milliseconds).
Users get an immediate "OTP service temporarily unavailable, 
try again shortly" message. System continues to function 
for other operations (slot checking, certificate download).
```

---

## Chapter 2.2 — Legacy Modernization & Migration Playbook

### 2.2.1 Why Legacy Systems Persist in Government

Before discussing migration strategies, understand why government systems remain legacy longer than commercial systems.

```
┌─────────────────────────────────────────────────────────────────────────┐
│          WHY GOVERNMENT LEGACY SYSTEMS LIVE SO LONG                    │
│                                                                         │
│  FACTOR               GOVERNMENT REALITY                               │
│  ──────               ───────────────────                              │
│  Risk tolerance       Zero tolerance for downtime in citizen services  │
│                       A failed migration = public & media outcry       │
│                                                                         │
│  Procurement cycles   New system needs tendering: 12–24 months         │
│                       parallel running adds more months               │
│                                                                         │
│  Staff knowledge      Deep domain experts retire. System knowledge     │
│                       is tribal. Nobody fully understands the legacy  │
│                                                                         │
│  Undocumented rules   30-year-old COBOL code may encode pension       │
│                       calculation rules that exist nowhere else       │
│                                                                         │
│  Integration web      Legacy system integrated with 40+ other agencies │
│                       Each integration is a migration dependency      │
│                                                                         │
│  Budget cycles        Modernisation is multi-year, multi-crore.       │
│                       Annual budget cycles disrupt continuity         │
│                                                                         │
│  IMPLICATION FOR ARCHITECTS:                                            │
│  You will almost never replace a government system in one shot.       │
│  You will always be migrating incrementally, alongside a live         │
│  system that is serving real citizens, every day.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2.2 Migration Strategies: Strangler Fig, CDC & Parallel Run

**Strategy 1: The Strangler Fig Pattern**

Named after the strangler fig tree that grows around a host tree over decades and eventually replaces it.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 STRANGLER FIG MIGRATION PATTERN                         │
│                                                                         │
│  PHASE 0: STATUS QUO                                                   │
│  ─────────────────                                                     │
│                                                                         │
│  All Traffic ────────────────────────────────► LEGACY SYSTEM           │
│                                                                         │
│                                                                         │
│  PHASE 1: INTRODUCE FACADE (API Gateway / Routing Layer)               │
│  ────────────────────────────────────────────────────────              │
│                                                                         │
│  All Traffic ──────────► FACADE ────────────────────────► LEGACY      │
│                          (routes all to legacy, new system shadow)    │
│                                                                         │
│                                                                         │
│  PHASE 2: INCREMENTALLY MIGRATE ENDPOINTS                              │
│  ─────────────────────────────────────────                             │
│                                                                         │
│  Traffic ──► FACADE ──► /citizen-profile ──────────► NEW SERVICE      │
│                    │                                                    │
│                    └──► /payments ─────────────────► LEGACY           │
│                    │                                                    │
│                    └──► /documents ────────────────► LEGACY           │
│                                                                         │
│                                                                         │
│  PHASE 3: COMPLETE MIGRATION                                            │
│  ─────────────────────────                                             │
│                                                                         │
│  Traffic ──► FACADE ──► /citizen-profile ──────────► NEW SERVICE      │
│                    │                                                    │
│                    └──► /payments ─────────────────► NEW SERVICE      │
│                    │                                                    │
│                    └──► /documents ────────────────► NEW SERVICE      │
│                                                                         │
│  LEGACY SYSTEM DECOMMISSIONED                                           │
│                                                                         │
│  REAL CASE: NIC migrating a state treasury system from COBOL/CICS     │
│  to Java microservices used exactly this pattern over 3 years.        │
│  The façade was an API Gateway with routing rules managed in a        │
│  configuration file — toggled per module without downtime.            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Strategy 2: Change Data Capture (CDC)**

CDC is a technique to capture every change (INSERT, UPDATE, DELETE) happening in a database and stream it to another system in real-time or near-real-time.

This is essential when migrating because:
- The legacy system is still being used during migration
- The new system needs a copy of all data, continuously updated
- You cannot shut down the legacy system to do a bulk copy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  CDC ARCHITECTURE WITH DEBEZIUM                         │
│                                                                         │
│  HOW IT WORKS:                                                          │
│                                                                         │
│  Legacy Oracle DB                                                        │
│  ┌──────────────────────────────────────────────┐                      │
│  │  Table: CITIZEN_MASTER                       │                      │
│  │  INSERT: citizen_id=12345, name="Ravi Kumar" │                      │
│  │  UPDATE: citizen_id=12345, address="Chennai" │                      │
│  └──────────────┬───────────────────────────────┘                      │
│                 │                                                        │
│                 │ Oracle reads transaction log (Redo Log)               │
│                 ▼                                                        │
│  ┌─────────────────────────────┐                                        │
│  │  Debezium Connector         │ ← Open source CDC tool                │
│  │  (reads Oracle redo log)    │   deployed as Kafka Connect           │
│  └──────────────┬──────────────┘                                        │
│                 │                                                        │
│                 │ Streams change events                                 │
│                 ▼                                                        │
│  ┌─────────────────────────────┐                                        │
│  │  Kafka Topic:               │                                        │
│  │  db.citizen_master.changes  │                                        │
│  │                             │                                        │
│  │  { op: "c", // c=create     │                                        │
│  │    after: {                 │                                        │
│  │      citizen_id: 12345,     │                                        │
│  │      name: "Ravi Kumar"     │                                        │
│  │    }                        │                                        │
│  │  }                          │                                        │
│  └──────────────┬──────────────┘                                        │
│                 │                                                        │
│          ┌──────┴──────────────────┐                                   │
│          ▼                         ▼                                   │
│  ┌──────────────┐          ┌──────────────────┐                        │
│  │  New System  │          │  Elasticsearch   │                        │
│  │  PostgreSQL  │          │  (search index   │                        │
│  │  (consumes   │          │   updated live)  │                        │
│  │   events,    │          │                  │                        │
│  │   builds new │          │                  │                        │
│  │   data model)│          │                  │                        │
│  └──────────────┘          └──────────────────┘                        │
│                                                                         │
│  RESULT: New system stays in sync with legacy in near-real-time.      │
│  Legacy continues operating. No downtime. No bulk copy.               │
└─────────────────────────────────────────────────────────────────────────┘
```

**Strategy 3: Parallel Run**

Both old and new systems run simultaneously. The same input goes to both. Outputs are compared. The new system is validated before traffic is switched.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PARALLEL RUN PATTERN                                 │
│                                                                         │
│  Traffic ──► SHADOW PROXY                                               │
│                   │                                                     │
│          ┌────────┴────────────┐                                       │
│          ▼                     ▼                                       │
│   LEGACY SYSTEM          NEW SYSTEM                                    │
│   (Authoritative)        (Candidate)                                   │
│         │                     │                                         │
│         ▼                     ▼                                         │
│   Response sent          Response logged                               │
│   to user                (not sent to user)                            │
│         │                     │                                         │
│         └─────────┬───────────┘                                        │
│                   ▼                                                     │
│           COMPARISON ENGINE                                             │
│           Checks: Are responses identical?                             │
│           Logs: Any discrepancies                                       │
│           Alerts: On divergence rate > threshold                       │
│                                                                         │
│  WHEN TO SWITCH:                                                        │
│  Parallel run for N weeks until divergence rate < 0.01%               │
│  Then switch new system to authoritative                               │
│  Keep legacy in shadow for another N weeks as fallback                │
│                                                                         │
│  PROS:                                                                  │
│  • No risk to users during validation                                  │
│  • Catches edge cases that tests miss                                  │
│  • Builds confidence with stakeholders                                 │
│                                                                         │
│  CONS:                                                                  │
│  • Double infrastructure cost during parallel period                  │
│  • Write operations need careful handling (don't double-write to DB)  │
│  • Stateful operations are complex to shadow                          │
│                                                                         │
│  REAL CASE: NPCI (National Payments Corporation of India) used        │
│  parallel run for UPI 2.0 features, shadowing against UPI 1.x         │
│  for 6 weeks before full cutover. Divergence in mandate              │
│  processing logic was caught and fixed before it reached users.       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2.3 Database Migration — Schema Evolution & Downtime Minimisation

**The Core Problem**

Changing a database schema in a live system is one of the most dangerous operations in software engineering. An `ALTER TABLE ADD COLUMN NOT NULL` on a 500-million-row table can lock the entire table for hours, causing complete service outage.

**Schema Evolution Techniques:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│          ZERO-DOWNTIME SCHEMA MIGRATION TECHNIQUES                      │
│                                                                         │
│  TECHNIQUE 1: EXPAND-CONTRACT (BEST PRACTICE)                          │
│  ───────────────────────────────────────────                           │
│                                                                         │
│  Goal: Rename column `citizen_name` to `full_name`                    │
│                                                                         │
│  WRONG WAY (causes downtime):                                           │
│  ALTER TABLE citizens RENAME COLUMN citizen_name TO full_name;         │
│  → Old code reading `citizen_name` breaks immediately                  │
│                                                                         │
│  RIGHT WAY (Expand-Contract):                                           │
│                                                                         │
│  PHASE 1 - EXPAND:                                                      │
│  ALTER TABLE citizens ADD COLUMN full_name VARCHAR(200);               │
│  (Deploy code that writes to BOTH citizen_name AND full_name)          │
│  (Deploy code that reads from citizen_name, falls back to full_name)  │
│                                                                         │
│  PHASE 2 - BACKFILL:                                                    │
│  UPDATE citizens SET full_name = citizen_name WHERE full_name IS NULL; │
│  (Done in batches of 10,000 rows to avoid lock contention)            │
│                                                                         │
│  PHASE 3 - SWITCH READS:                                               │
│  Deploy code that reads from full_name (primary) only                  │
│                                                                         │
│  PHASE 4 - CONTRACT:                                                    │
│  ALTER TABLE citizens DROP COLUMN citizen_name;                        │
│  (Old column gone. No code reads it anymore. Safe to drop.)           │
│                                                                         │
│  Duration: Days to weeks depending on table size and validation        │
│  Downtime: ZERO                                                         │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  TECHNIQUE 2: ONLINE DDL (Database Native)                             │
│  ─────────────────────────────────────────                             │
│  PostgreSQL: Most DDL is non-blocking since version 11+               │
│  MySQL/Aurora: pt-online-schema-change or gh-ost tool                 │
│  Oracle: DBMS_REDEFINITION for online table reorganisation             │
│                                                                         │
│  TECHNIQUE 3: BLUE-GREEN DATABASE DEPLOYMENT                           │
│  ───────────────────────────────────────────                           │
│  Maintain two database environments.                                   │
│  Migrate schema on inactive (blue).                                    │
│  Switch traffic via connection pool configuration.                     │
│  Rollback: Switch back within seconds.                                 │
│  Cost: Double database infrastructure during migration window.         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Flyway and Liquibase — Migration Tooling**

These are version control systems for database schemas. Every schema change is a versioned migration script that runs in order, tracked in a migration history table.

```
Flyway Migration Script Example:
─────────────────────────────────

V001__create_citizen_table.sql
V002__add_aadhaar_hash_column.sql
V003__create_grievance_table.sql
V004__add_index_on_mobile_number.sql

Flyway tracks which scripts have run.
On deployment, it runs only new scripts.
Never re-runs executed scripts.
Migrations are stored in version control alongside code.
Schema changes go through the same code review process as code.
```

---

### 2.2.4 Infrastructure & Storage Sizing — Cost/Performance Trade-offs

**How to Size Infrastructure — The Architect's Responsibility**

Many engineers believe sizing is a DevOps or infrastructure team concern. It is not. The architect must provide the sizing model because only the architect understands the expected load, data growth, and NFRs.

```
┌─────────────────────────────────────────────────────────────────────────┐
│          INFRASTRUCTURE SIZING METHODOLOGY                              │
│                                                                         │
│  STEP 1: Define the Load Profile                                       │
│  ────────────────────────────                                           │
│  • Peak concurrent users: 50,000                                       │
│  • Average session duration: 8 minutes                                 │
│  • Requests per session: 15                                            │
│  • Peak requests/second: 50,000 × (15/480) = ~1,562 RPS              │
│  • Design for: 2× peak = 3,124 RPS (safety margin)                   │
│                                                                         │
│  STEP 2: Define Compute Requirements per Service                       │
│  ──────────────────────────────────────────────                        │
│  Profile each service:                                                  │
│  • CitizenProfileService: 8ms avg response, CPU-light                 │
│  • DocumentGenerationService: 2 seconds, CPU-heavy (PDF generation)   │
│  • SearchService: 50ms avg, memory-heavy (Elasticsearch)              │
│                                                                         │
│  STEP 3: Calculate per Service                                         │
│  ──────────────────────────                                             │
│                                                                         │
│  CitizenProfileService:                                                 │
│  • 1,562 RPS × 0.008s = 12.5 concurrent threads needed               │
│  • Round up to 16 threads (one pod = 8 threads)                      │
│  • Minimum: 2 pods. With redundancy: 4 pods                          │
│  • Per pod: 0.5 CPU, 512MB RAM                                        │
│  • Total: 4 × 0.5 CPU = 2 vCPU, 4 × 512MB = 2GB RAM                │
│                                                                         │
│  STEP 4: Database Sizing                                               │
│  ───────────────────────                                               │
│  • Records: 15 crore citizen records × 2KB avg = 300GB                │
│  • Growth: 2% per year = 6GB/year                                     │
│  • Indexes: ~40% of data size = 120GB                                 │
│  • Replication lag buffer: 50GB                                        │
│  • Total storage with 2× safety: ~950GB → Provision 1TB SSD          │
│                                                                         │
│  STEP 5: Cost Estimate                                                 │
│  ─────────────────────                                                 │
│  (Use actual cloud provider pricing for your target environment)      │
│  Do not guess. Use the AWS/Azure/GCP pricing calculator or            │
│  NIC/NICSI pricing schedule for government cloud.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 2.3 — AI-Assisted Development & Validation

### 2.3.1 The Architect's Relationship with AI Tools

AI-assisted development tools (GitHub Copilot, GPT-4/5, Claude) are now a reality in software development. For an architect, the risk is different from that of a junior developer.

A junior developer using AI incorrectly might produce buggy code. An architect using AI incorrectly might make flawed architectural decisions based on AI-generated analysis, or deploy AI-generated code with security vulnerabilities into production government systems.

**The Mental Model for AI-Assisted Architecture Work:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│            AI AS A JUNIOR ARCHITECT, NOT AN ORACLE                     │
│                                                                         │
│  USE AI FOR:                    DO NOT USE AI FOR:                     │
│  ─────────────────────────      ──────────────────────────────         │
│  First draft of ADRs            Final security decisions               │
│  Boilerplate code generation    Compliance interpretations             │
│  Test case brainstorming        Data classification decisions          │
│  Documentation scaffolding      Architecture sign-off                  │
│  Trade-off enumeration          Production secret handling             │
│  Refactoring suggestions        Choosing regulatory frameworks         │
│                                                                         │
│  TREAT AI OUTPUT LIKE CODE FROM A VERY FAST BUT INEXPERIENCED          │
│  COLLEAGUE:                                                             │
│  • Review every line before accepting                                   │
│  • Check for security issues the AI missed                             │
│  • Validate that the approach suits your specific context              │
│  • Never paste production credentials into AI prompts                  │
│  • Never paste PII or sensitive citizen data into AI prompts           │
│                                                                         │
│  NOTE ON GOVERNMENT CONTEXT:                                            │
│  Many government agencies have policies prohibiting the use of         │
│  commercial AI tools with organisational data. Check your agency's     │
│  acceptable use policy. Singapore GovTech has published guidance       │
│  on approved AI tools for government engineers. India MeitY has       │
│  issued advisories on AI tool use in government projects.             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2.3.2 Prompt Engineering for Architecture Work

Prompt engineering is the skill of constructing inputs to AI models that produce useful, accurate, and appropriately scoped outputs.

**Prompt Pattern 1: Architecture Trade-off Analysis**

```
WEAK PROMPT:
────────────
"Should I use Kafka or RabbitMQ?"

AI response: Generic comparison that doesn't apply to your context.

STRONG PROMPT:
──────────────
"I am designing an event backbone for a government vaccination 
 tracking system in India. Requirements:
 
 - 2 million vaccination events per day, with peaks of 500,000
   events in a 2-hour window during drive days
 - Events must be retained for minimum 7 years (audit requirement)
 - 12 downstream consumers including state portals, AEFI reporting,
   and MoHFW analytics
 - Operations team has 3 engineers with Linux/networking background
   but no messaging system experience
 - Infrastructure: On-premise NIC cloud (OpenStack based)
 
 Compare Apache Kafka and RabbitMQ for this use case.
 For each, provide:
 1. Fit for the retention requirement
 2. Fit for the fan-out to 12 consumers
 3. Operational complexity for the described team
 4. Limitations in this specific context
 
 Conclude with a recommendation and the key risks of that choice."

AI response: Specific, contextual, actionable.
```

**Prompt Pattern 2: Code Generation with Context**

```
WEAK PROMPT:
────────────
"Write a Kafka consumer in Java"

STRONG PROMPT:
──────────────
"Write a Spring Boot Kafka consumer in Java 17 for the following:

 Topic: vaccination.events.v1
 Message format: JSON matching this schema:
 { vaccineCode: string, beneficiaryId: string, facilityId: string,
   doseNumber: int, vaccinatedAt: ISO-8601 datetime }
 
 Requirements:
 - Consumer group: state-portal-consumer
 - Handle deserialization errors without stopping the consumer
   (log error, publish to DLQ topic: vaccination.events.dlq)
 - Implement idempotency: check Redis before processing
   (idempotency key: beneficiaryId + doseNumber + date)
 - Ensure at-least-once processing with manual offset commit
   only after successful processing and idempotency store update
 - Include unit tests using EmbeddedKafka
 
 Follow Spring Boot 3.x idioms. Use Resilience4j for retry."
```

---

### 2.3.3 AI Code Validation — Security Scanning of AI Output

AI models can generate code with subtle security vulnerabilities. In government systems, these can be critical.

**Categories of Security Issues in AI-Generated Code:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│          COMMON SECURITY ISSUES IN AI-GENERATED CODE                   │
│                                                                         │
│  1. SQL INJECTION                                                       │
│  ──────────────                                                         │
│  AI generates:                                                          │
│  String query = "SELECT * FROM citizens WHERE id = " + citizenId;     │
│  jdbcTemplate.query(query, ...);                                        │
│                                                                         │
│  Correct:                                                               │
│  jdbcTemplate.query(                                                    │
│    "SELECT * FROM citizens WHERE id = ?",                              │
│    new Object[]{citizenId}, ...);                                       │
│                                                                         │
│  2. INSECURE RANDOM FOR TOKENS                                         │
│  ─────────────────────────────                                         │
│  AI generates:                                                          │
│  String token = String.valueOf(new Random().nextInt(999999));          │
│                                                                         │
│  Correct:                                                               │
│  String token = SecureRandom.getInstance("SHA1PRNG")                   │
│                   .ints(6, 0, 10)                                       │
│                   .mapToObj(Integer::toString)                          │
│                   .collect(Collectors.joining());                       │
│                                                                         │
│  3. MISSING AUTHENTICATION CHECK                                        │
│  ──────────────────────────────                                        │
│  AI generates endpoint without @PreAuthorize or role check.            │
│  Looks functional in unit tests. Exposes admin API publicly.           │
│                                                                         │
│  4. LOGGING SENSITIVE DATA                                              │
│  ──────────────────────────                                            │
│  AI generates:                                                          │
│  log.info("Processing payment for citizen: {}, Aadhaar: {}",          │
│            citizen.getName(), citizen.getAadhaar());                   │
│                                                                         │
│  Aadhaar appearing in log files is a UIDAI Act violation.             │
│                                                                         │
│  5. MISSING INPUT VALIDATION                                            │
│  ───────────────────────────                                           │
│  AI generates DTOs without Bean Validation annotations.               │
│  Missing @NotNull, @Size, @Pattern on critical fields.                │
└─────────────────────────────────────────────────────────────────────────┘
```

**AI Output Validation Checklist (Use Before Every Merge):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│          AI CODE VALIDATION CHECKLIST                                   │
│                                                                         │
│  SECURITY                                          CHECKED              │
│  ────────                                          ───────              │
│  □ No raw SQL string concatenation                                      │
│  □ No hardcoded credentials or API keys                                │
│  □ No sensitive data in log statements                                 │
│  □ Authentication/authorisation present on all endpoints               │
│  □ Input validation on all user-supplied data                          │
│  □ SecureRandom used (not java.util.Random) for tokens                │
│  □ Dependencies are not known vulnerable versions (check NVD)         │
│                                                                         │
│  CORRECTNESS                                                            │
│  ───────────                                                            │
│  □ Business logic matches requirement (AI may hallucinate rules)      │
│  □ Edge cases handled (null, empty, boundary values)                  │
│  □ Error handling is appropriate (not swallowed silently)             │
│  □ Transactional boundaries are correct                                │
│                                                                         │
│  QUALITY                                                                │
│  ───────                                                                │
│  □ No unused imports, variables, or dead code                         │
│  □ Methods are single-responsibility                                   │
│  □ Tests exist and cover the generated code path                      │
│  □ Code matches team's style guide and patterns                       │
│                                                                         │
│  TOOLS TO AUTOMATE THIS:                                               │
│  SAST: SonarQube, Checkmarx, Semgrep                                  │
│  Dependency scanning: OWASP Dependency-Check, Snyk                    │
│  Secret scanning: GitLeaks, TruffleHog, GitHub Secret Scanning        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 2.4 — Performance Engineering & Observability

### 2.4.1 Load and Stress Testing — JMeter, Gatling, k6

**Types of Performance Tests — Know the Difference**

```
┌─────────────────────────────────────────────────────────────────────────┐
│             PERFORMANCE TEST TYPES DEFINED                              │
│                                                                         │
│  LOAD TEST                                                              │
│  ─────────                                                              │
│  Simulates expected normal load.                                        │
│  Goal: Verify system meets NFRs under normal conditions.               │
│  Example: "Simulate 10,000 concurrent users filing ITR"               │
│  Pass criteria: p95 response time < 3s, zero errors                   │
│                                                                         │
│  STRESS TEST                                                            │
│  ───────────                                                            │
│  Pushes beyond expected load to find the breaking point.               │
│  Goal: Find where the system fails and how it fails.                  │
│  Example: Ramp to 50,000 concurrent users, observe failure mode       │
│  Key question: Does it fail gracefully or catastrophically?           │
│                                                                         │
│  SOAK TEST (ENDURANCE)                                                  │
│  ──────────────────────                                                 │
│  Run at normal load for extended period (8–72 hours).                 │
│  Goal: Find memory leaks, connection pool exhaustion, disk fill.      │
│  Example: Run at 5,000 users for 24 hours, watch memory growth.       │
│                                                                         │
│  SPIKE TEST                                                             │
│  ──────────                                                             │
│  Sudden large increase in load, then return to normal.                │
│  Simulates: Flash sale, viral notification, news event.               │
│  Government case: Sudden portal announcement via PM's tweet.          │
│                                                                         │
│  VOLUME TEST                                                            │
│  ───────────                                                            │
│  Test with large data volumes rather than high concurrency.           │
│  Example: Generate 5 crore citizen records, then run queries.         │
│  Goal: Find data-volume-related performance degradation.              │
└─────────────────────────────────────────────────────────────────────────┘
```

**Practical k6 Script Example — Citizen Portal Load Test**

```javascript
// k6 load test for Citizen Grievance Portal
// Run: k6 run --vus 1000 --duration 5m citizen_portal_test.js

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const loginDuration = new Trend('login_duration');
const grievanceSubmitDuration = new Trend('grievance_submit_duration');

// Test configuration: Ramp up, sustain, ramp down
export const options = {
  stages: [
    { duration: '2m', target: 1000 },   // Ramp to 1000 users in 2 min
    { duration: '5m', target: 1000 },   // Sustain 1000 users for 5 min
    { duration: '2m', target: 5000 },   // Spike to 5000 users
    { duration: '3m', target: 5000 },   // Sustain spike
    { duration: '2m', target: 0 },      // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<3000'], // 95% of requests under 3 seconds
    'errors': ['rate<0.01'],             // Less than 1% error rate
    'login_duration': ['p(99)<5000'],   // 99th percentile login under 5s
  },
};

// Simulate citizen filing a grievance
export default function () {
  
  // Step 1: Login
  const loginStart = Date.now();
  const loginRes = http.post('https://grievance.gov.in/api/auth/login', {
    mobile: '9876543210',
    otp: '123456',  // Use test OTP in staging environment
  });
  loginDuration.add(Date.now() - loginStart);
  
  check(loginRes, {
    'login status 200': (r) => r.status === 200,
    'login has token': (r) => r.json('accessToken') !== undefined,
  }) || errorRate.add(1);
  
  const token = loginRes.json('accessToken');
  const headers = { Authorization: `Bearer ${token}` };
  
  sleep(1); // Simulate user reading the page
  
  // Step 2: Submit Grievance
  const submitStart = Date.now();
  const grievanceRes = http.post(
    'https://grievance.gov.in/api/grievances',
    JSON.stringify({
      categoryCode: 'WATER_SUPPLY',
      description: 'No water supply for 3 days in area',
      locationCode: 'TN-CB-001',
    }),
    { headers: { ...headers, 'Content-Type': 'application/json' } }
  );
  grievanceSubmitDuration.add(Date.now() - submitStart);
  
  check(grievanceRes, {
    'grievance submitted 201': (r) => r.status === 201,
    'grievance has ID': (r) => r.json('grievanceId') !== undefined,
  }) || errorRate.add(1);
  
  sleep(2);
}
```

---

### 2.4.2 Observability — The Three Pillars

Observability is not monitoring. Monitoring tells you when something is wrong. Observability tells you *why* it is wrong, even for problems you did not anticipate.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   THE THREE PILLARS OF OBSERVABILITY                    │
│                                                                         │
│  PILLAR 1: METRICS                                                      │
│  ─────────────────                                                     │
│  Numerical measurements aggregated over time.                          │
│                                                                         │
│  What to measure:                                                       │
│  • RED metrics (for services): Rate, Errors, Duration                  │
│  • USE metrics (for resources): Utilisation, Saturation, Errors        │
│                                                                         │
│  Rate:    Requests per second to GrievanceService                      │
│  Errors:  HTTP 5xx errors per second                                   │
│  Duration: p50, p95, p99 response time                                 │
│                                                                         │
│  Tools: Prometheus (collection) + Grafana (visualisation)              │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  PILLAR 2: LOGS                                                         │
│  ─────────────                                                          │
│  Timestamped records of discrete events.                               │
│                                                                         │
│  LOG LEVELS (use correctly):                                            │
│  ERROR — Something failed. Needs immediate attention.                  │
│  WARN  — Something unexpected. May need attention.                      │
│  INFO  — Normal business events (payment processed, user logged in)    │
│  DEBUG — Detailed diagnostic (disabled in production by default)       │
│                                                                         │
│  STRUCTURED LOGGING (mandatory for ELK/observability):                 │
│  Wrong:  log.info("User 12345 filed grievance WATER-001")             │
│  Right:  log.info("Grievance filed",                                   │
│            kv("userId", 12345),                                        │
│            kv("grievanceId", "WATER-001"),                             │
│            kv("category", "WATER_SUPPLY"),                             │
│            kv("traceId", MDC.get("traceId")));                        │
│                                                                         │
│  Output: {"level":"INFO","userId":12345,                               │
│           "grievanceId":"WATER-001","traceId":"abc-123"}               │
│  Searchable, parseable, queryable in Kibana.                           │
│                                                                         │
│  ─────────────────────────────────────────────────────────────         │
│                                                                         │
│  PILLAR 3: TRACES                                                       │
│  ─────────────────                                                      │
│  Records of a single request's journey across multiple services.       │
│                                                                         │
│  Without tracing: "The API is slow" — which of 8 service hops is it? │
│  With tracing: You see the full journey with timing for each hop.     │
│                                                                         │
│  Example trace for "Submit Grievance" request:                         │
│                                                                         │
│  [API Gateway]         0ms ──────────────────────────────► 450ms      │
│    [GrievanceService]  5ms ────────────────────────────►  445ms       │
│      [CitizenService]  8ms ──────────────────────────►    320ms       │
│        [PostgreSQL]   10ms ──────────────────────────►    305ms ← 😱  │
│      [NotifService]  335ms ──────────────────────────►    440ms       │
│        [SMSGateway]  337ms ──────────────────────────►    438ms       │
│                                                                         │
│  FINDING: PostgreSQL call takes 295ms. Index missing on filter column │
│                                                                         │
│  Tools: Jaeger, Zipkin, OpenTelemetry                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

**The Prometheus + Grafana Stack — How It Works**

```
┌─────────────────────────────────────────────────────────────────────────┐
│               PROMETHEUS + GRAFANA ARCHITECTURE                         │
│                                                                         │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐      │
│  │ GrievanceService│   │  PaymentService  │   │  CitizenService │      │
│  │                 │   │                 │   │                 │      │
│  │ /actuator/      │   │ /actuator/      │   │ /actuator/      │      │
│  │  metrics        │   │  metrics        │   │  metrics        │      │
│  │ (Prometheus     │   │ (Prometheus     │   │ (Prometheus     │      │
│  │  format)        │   │  format)        │   │  format)        │      │
│  └────────┬────────┘   └────────┬────────┘   └────────┬────────┘      │
│           │                     │                     │               │
│           └─────────────────────┼─────────────────────┘               │
│                                 │ Scrapes every 15 seconds            │
│                                 ▼                                      │
│                    ┌─────────────────────────┐                        │
│                    │      PROMETHEUS          │                        │
│                    │  (Time-series database)  │                        │
│                    │                         │                        │
│                    │  Stores metrics with    │                        │
│                    │  labels and timestamps  │                        │
│                    │                         │                        │
│                    │  Alert rules:           │                        │
│                    │  IF error_rate > 1%     │                        │
│                    │  FOR 2 minutes          │                        │
│                    │  THEN fire alert        │                        │
│                    └────────────┬────────────┘                        │
│                                 │                                      │
│                    ┌────────────┴────────────┐                        │
│                    │       GRAFANA           │                        │
│                    │   (Visualisation)       │                        │
│                    │                         │                        │
│                    │  Dashboards:            │                        │
│                    │  • Service health       │                        │
│                    │  • Request rates        │                        │
│                    │  • Error rates          │                        │
│                    │  • Infrastructure CPU   │                        │
│                    └─────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Observability Incident — Understanding the Value**

> A state government's social welfare portal saw its response time degrade every day between 10:00 AM and 11:00 AM by approximately 40%. The infrastructure team increased server capacity twice without improvement. The problem persisted for 3 months.
>
> When distributed tracing (Jaeger) was finally implemented, the trace data revealed:
> - The slow path was a specific API: `GET /eligibility-check`
> - One hop in the trace took 600–800ms every morning
> - That hop was a database query: `SELECT * FROM scheme_eligibility WHERE...`
> - The query was not slow in the evening
>
> **Root cause found:** A batch job ran every morning at 9:55 AM performing VACUUM ANALYZE on the `scheme_eligibility` table (500 million rows), which locked statistics updates and caused query planner to use a suboptimal plan for 45 minutes after.
>
> **Fix:** Schedule the batch job to run at 2:00 AM. Response time problem disappeared.
>
> **Time to find without tracing:** 3 months of guessing.  
> **Time to find with tracing:** 2 hours after tracing was enabled.

---

### 2.4.3 SLO, SLI & Error Budgets

**The SLA vs SLO vs SLI Hierarchy:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   SLA, SLO, SLI HIERARCHY                              │
│                                                                         │
│  SLA (Service Level Agreement)                                          │
│  ──────────────────────────────                                        │
│  The CONTRACT with the customer/ministry.                              │
│  "The Grievance Portal will be available 99.5% of the time monthly."  │
│  Breach = financial penalty / contract consequence                     │
│                                                                         │
│  SLO (Service Level Objective)                                          │
│  ──────────────────────────────                                        │
│  The INTERNAL TARGET. Should be stricter than SLA.                    │
│  "We target 99.7% availability internally."                            │
│  Buffer between SLO and SLA = your error budget                       │
│                                                                         │
│  SLI (Service Level Indicator)                                          │
│  ──────────────────────────────                                        │
│  The MEASUREMENT. What you actually measure.                           │
│  "Uptime = (total_minutes - error_minutes) / total_minutes"           │
│  This is what Prometheus actually records.                             │
│                                                                         │
│  ERROR BUDGET:                                                          │
│  ─────────────                                                          │
│  If SLO = 99.7%, monthly error budget = 0.3% × 30 days × 24 hrs      │
│                                        = 2.16 hours of downtime/month │
│                                                                         │
│  Once error budget is consumed: STOP new feature deployments.         │
│  All energy goes to reliability until next month.                     │
│  This is the Google SRE model.                                         │
│                                                                         │
│  EXAMPLE SLOs FOR GOVERNMENT PORTAL:                                   │
│  ──────────────────────────────────                                    │
│  Availability SLO: 99.7% monthly                                       │
│  Latency SLO: 95% of requests complete within 3 seconds               │
│  Error rate SLO: Less than 0.5% of requests return 5xx               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 2.5 — Project Milestone 2 Preview

By Day 11, you will produce:

**1. Working Microservices**
A set of microservices implementing the core domain from your Phase 1 blueprint, demonstrating:
- Service boundaries aligned with bounded contexts
- Event-driven communication via a message broker
- SAGA implementation for at least one distributed transaction
- Idempotency implementation on critical endpoints

**2. AI-Assisted Code**
Evidence of AI-assisted development with the validation checklist applied, including:
- Prompt templates used
- Security scan output (SonarQube or equivalent)
- Test coverage report

**3. Performance Report**
Load test results from k6 or Gatling showing:
- Baseline performance metrics
- Identified bottleneck(s)
- Optimisation applied and result
- Comparison of before/after

```
┌─────────────────────────────────────────────────────────────────────────┐
│          MILESTONE 2 ARCHITECTURE DEFENCE PREPARATION GUIDE            │
│                                                                         │
│  QUESTIONS YOUR INSTRUCTOR PANEL WILL LIKELY ASK:                      │
│                                                                         │
│  1. "Why did you choose choreography over orchestration for your SAGA? │
│      What would break if consumer X goes offline for 2 hours?"         │
│                                                                         │
│  2. "Your load test shows p99 latency of 8 seconds. Walk me through   │
│      how you would identify and fix this in production."               │
│                                                                         │
│  3. "Show me how your system handles a duplicate payment request."     │
│      → Expect to walk through your idempotency implementation          │
│                                                                         │
│  4. "Your AI-generated code has this pattern: [shows snippet].        │
│      What is the security risk here and how would you fix it?"        │
│                                                                         │
│  5. "If the payment service goes down, what happens to the citizen's  │
│      application? Is data consistent? Is the user informed?"           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2 — Terms to Google: Research Guide

## Core Microservices Concepts

| Topic            | Search Term                                               | Why                          |
| ---------------- | --------------------------------------------------------- | ---------------------------- |
| SAGA Pattern     | `"SAGA pattern microservices choreography orchestration"` | Core Day 7 concept           |
| Idempotency      | `"idempotency distributed systems implementation Redis"`  | Implementation detail needed |
| Bulkhead Pattern | `"bulkhead pattern Resilience4j Java example"`            | Code-level implementation    |
| Circuit Breaker  | `"circuit breaker Resilience4j Spring Boot tutorial"`     | Practical implementation     |
| Rate Limiting    | `"rate limiting token bucket implementation Redis"`       | API gateway concept          |
| BFF Pattern      | `"backend for frontend pattern microservices"`            | Mobile-first architecture    |

## Legacy Migration

| Topic                   | Search Term                                          | Why                                |
| ----------------------- | ---------------------------------------------------- | ---------------------------------- |
| Strangler Fig           | `"strangler fig pattern Martin Fowler"`              | Original description by the author |
| CDC                     | `"Debezium change data capture tutorial Kafka"`      | Primary CDC tool used in course    |
| Flyway                  | `"Flyway database migration Spring Boot tutorial"`   | Migration tooling                  |
| Zero downtime migration | `"zero downtime database migration expand contract"` | Core technique                     |
| Parallel run            | `"parallel run pattern legacy system migration"`     | Migration strategy                 |

## AI-Assisted Development

| Topic                   | Search Term                                                | Why                               |
| ----------------------- | ---------------------------------------------------------- | --------------------------------- |
| Prompt engineering      | `"prompt engineering software architecture use cases"`     | Day 9 foundation                  |
| GitHub Copilot security | `"GitHub Copilot security vulnerabilities generated code"` | Risk awareness                    |
| SAST tools              | `"SonarQube Java security rules OWASP"`                    | Validation tooling                |
| AI coding governance    | `"GovTech Singapore AI tools policy government engineers"` | Regulatory context                |
| Semgrep                 | `"Semgrep security rules Java tutorial"`                   | Code scanning tool used in course |

## Performance Engineering

| Topic              | Search Term                                              | Why                       |
| ------------------ | -------------------------------------------------------- | ------------------------- |
| k6 tutorial        | `"k6 load testing tutorial getting started"`             | Primary tool for Day 10   |
| Prometheus basics  | `"Prometheus metrics scraping tutorial Spring Boot"`     | Observability stack       |
| Grafana dashboards | `"Grafana dashboard tutorial Prometheus datasource"`     | Visualisation             |
| Jaeger tracing     | `"Jaeger distributed tracing Spring Boot OpenTelemetry"` | Distributed tracing setup |
| SLO error budget   | `"Google SRE SLO error budget explained"`                | SRE concepts for Day 10   |
| JVM heap analysis  | `"Java heap dump analysis production memory leak"`       | Common production problem |

## India/Singapore Government Context

| Topic                      | Search Term                                               | Why                              |
| -------------------------- | --------------------------------------------------------- | -------------------------------- |
| IPPB architecture          | `"India Post Payments Bank IPPB technology architecture"` | Mobile-first government case     |
| NPCI UPI architecture      | `"UPI technical architecture NPCI microservices"`         | High-scale payment system        |
| GovTech SRE practices      | `"GovTech Singapore SRE practices government systems"`    | Singapore public sector SRE      |
| NIC cloud                  | `"NIC cloud MeghRaj government cloud India"`              | Infrastructure context for India |
| Passport Seva architecture | `"Passport Seva Portal technology architecture TCS"`      | Government workflow example      |

---

*Document Version: 1.0 | Pre-Read Part 2 of 3 | Senior Engineer → Solution Architect Program | June 2026*