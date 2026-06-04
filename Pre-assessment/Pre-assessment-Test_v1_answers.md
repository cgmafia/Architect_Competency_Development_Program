# Pre-Assessment Test
## Senior Engineer → Solution Architect Accelerated Program
### June 2026 Cohort | Batch: 20–25 Participants
# ANSWER SHEET & EVALUATOR GUIDE

## ⚠️ Evaluator Access Only — Do Not Distribute to Candidates

---

## Section A — Answer Key

| Q   | Answer | Explanation                                                                                                                                                                                                                                                                     |
| --- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | **B**  | Financial transaction ledgers require strong consistency. A debit must be immediately visible to all reads — eventual consistency in a payment ledger risks double-spending or incorrect balance reads. Options A, C, D are insufficient for financial integrity.               |
| A2  | **C**  | Blockchain adds significant operational complexity, throughput limitations (15–45 TPS vs millions for RDBMS), and cost for no additional benefit over a well-designed RDBMS audit trail with write-once tables. The question tests ability to challenge over-engineering.       |
| A3  | **B**  | Thread pool exhaustion through cascading failure is the accurate description. This is the core problem that bulkhead and circuit breaker patterns solve. Option A is incorrect — there is no automatic queuing. Options C and D describe patterns that don't exist by default.  |
| A4  | **B**  | This is precise Kubernetes behaviour. Liveness failure = kubelet kills and restarts the container. Readiness failure = pod removed from service endpoints (traffic stops flowing to it) but the pod is not restarted.                                                           |
| A5  | **B**  | Strangler Fig with CDC is the correct risk-minimal, zero-downtime approach for live government systems. Big bang (A) is extremely high risk. Direct cut-over (C) still requires downtime. Option D is not a migration strategy.                                                 |
| A6  | **B**  | This is the precise definition of SAGA. Option A describes 2PC which SAGA explicitly avoids. Option C is wrong — SAGA works with orchestrators (Temporal, Camunda). Option D is wrong — SAGA provides eventual consistency, not strong consistency.                             |
| A7  | **C**  | Workspace separation with mandatory plan review is the most direct prevention. Modules (A) organise code but don't prevent wrong-environment apply. Remote state (B) enables team use but doesn't prevent wrong-variable apply. Terraform Cloud (D) alone doesn't prevent this. |
| A8  | **B**  | Zero Trust requires verification at every hop regardless of network location. Internal does not mean trusted. Both mTLS (network layer) and token validation (application layer) are required.                                                                                  |
| A9  | **B**  | A dedicated event-driven NotificationService is the correct pattern. Shared library (A) couples services and requires redeployment of all 12 on change. DB polling (C) is fragile. Copy-paste (D) is anti-pattern.                                                              |
| A10 | **C**  | The precise purpose of idempotency keys is safe retry behaviour. Not encryption (A), not authentication (B), not rate limiting (D).                                                                                                                                             |
| A11 | **B**  | Multi-stage build with Alpine JRE is the most impactful change — reduces image from ~1.8GB to ~150–200MB, eliminates build tools, reduces CVE surface. Option C helps slightly but not significantly.                                                                           |
| A12 | **B**  | This is the correct hierarchy. SLI = measured value, SLO = internal target (stricter), SLA = contractual commitment. Option A reverses SLA and SLO.                                                                                                                             |
| A13 | **B**  | The Aggregate Root as single entry point enforcing invariants is the precise DDD definition. It is not a database key (A), not a base class (C), not a microservice boundary (D) — though bounded context relates to service boundaries.                                        |
| A14 | **B**  | Over-sharding is a well-documented Elasticsearch performance problem. Each shard consumes JVM heap (~30MB minimum). 1,000 shards on 3 nodes = 333 shards/node × 30MB = ~10GB just for shard overhead, leaving little heap for indexing/search.                                  |
| A15 | **C**  | SQL Injection via string concatenation. OWASP A03:2021 Injection.                                                                                                                                                                                                               |

**Section A Scoring:** 2 points per correct answer. No partial credit.

---

## Section B — Answer Rubric

### B1 — Horizontal vs Vertical Scaling (5 points)

**Full Credit Answer Elements:**
- **Vertical scaling:** Increasing resources (CPU, RAM) of the existing machine. Has an upper limit. Single point of failure. No downtime-free in traditional sense.
- **Horizontal scaling:** Adding more instances of the service. No single upper limit. Stateless services are trivially horizontally scalable.
- **Preference:** Horizontal scaling for a stateless login service
- **Justification:** Stateless means any instance can handle any request — no session affinity needed. Each instance handles a share of traffic. Can scale to near-infinite capacity. Kubernetes HPA automates this. Vertical scaling has hard limits and creates single points of failure.

| Score | Criteria                                                                                                          |
| ----- | ----------------------------------------------------------------------------------------------------------------- |
| 5     | Both concepts correctly defined, correct choice made, statelessness correctly identified as the enabling property |
| 4     | Both concepts defined, correct choice, partial justification                                                      |
| 3     | Correct choice with reasonable justification but incomplete definitions                                           |
| 2     | Correct choice but weak or incorrect justification                                                                |
| 1     | Partial understanding shown                                                                                       |
| 0     | Wrong choice or no understanding demonstrated                                                                     |

---

### B2 — Challenging the 99.999% SLA Requirement (5 points)

**Full Credit Answer Elements:**
- **First response:** Do not agree immediately. Push back and ask clarifying questions.
- **Questions to ask:**
  1. What is the actual business impact per minute of downtime? (Quantify the need)
  2. What is the budget for this SLA? (99.999% = ~5 minutes/year downtime requires significant redundancy investment)
  3. Is this a contractual SLA or an aspirational goal?
  4. What are the specific failure modes being guarded against? (Network? DB? Application?)
  5. Does the dependency stack (NIC data centre, BSNL SMS gateway) support 99.999%?
- **Key insight:** If any dependency has a lower SLA, the system SLA cannot exceed the weakest dependency.
- **Alternative:** Often the real need is "high availability during business hours" which is significantly cheaper to design for than 99.999% calendar uptime.

| Score | Criteria                                                                                          |
| ----- | ------------------------------------------------------------------------------------------------- |
| 5     | Correctly pushes back, asks at least 3 substantive questions, identifies dependency chain problem |
| 4     | Pushes back, asks 2–3 good questions                                                              |
| 3     | Pushes back but questions are shallow or generic                                                  |
| 2     | Accepts SLA at face value and starts designing for it                                             |
| 0     | Immediately agrees and starts talking about active-active multi-region setup                      |

---

### B3 — Eventual Consistency (5 points)

**Full Credit Answer Elements:**
- **Definition:** After a write, replicas may temporarily return stale data. Given sufficient time with no new writes, all replicas will converge to the same value. The period of inconsistency is bounded and typically short (milliseconds to seconds).
- **Acceptable example:** DigiLocker document metadata cache, Aadhaar address updates visible across ministry portals, citizen benefit eligibility cache. Justification: Changes to these don't happen in seconds; the staleness window is acceptable given the access pattern.
- **Not acceptable example:** Bank debit/credit, pension payment status, GST return filing confirmation. Justification: The same amount cannot be debited twice due to stale read; a pensioner needs to know their payment status immediately.

| Score | Criteria                                                           |
| ----- | ------------------------------------------------------------------ |
| 5     | Correct definition, both examples correct with clear justification |
| 4     | Correct definition, one example strong, one weak                   |
| 3     | Correct definition, examples given but justification lacking       |
| 2     | Vague definition, examples given without justification             |
| 1     | Confuses eventual consistency with eventual failure                |
| 0     | Cannot define it                                                   |

---

### B4 — Authentication vs Authorisation (5 points)

**Full Credit Answer Elements:**
- **Authentication:** Verifying identity — "Who are you?" (SSO token, PKI certificate, password)
- **Authorisation:** Verifying permission — "Are you allowed to do this?" (RBAC, ABAC, policy)
- **Which mechanism:** Authorisation (not authentication). A Ministry A officer is successfully authenticated as a real person. The system knows who they are. But authorisation policy must enforce that their role (`MINISTRY_A_OFFICER`) does not grant access to `MINISTRY_B` resources.
- **How:** RBAC with ministry-scoped roles. Every API checks the JWT role claim against the resource's ministry scope. An officer with `role: MINISTRY_A_OFFICER` calling `GET /api/ministry-b/citizens` receives HTTP 403 Forbidden — not 401 (which would be an authentication failure).

| Score | Criteria                                                                                                                 |
| ----- | ------------------------------------------------------------------------------------------------------------------------ |
| 5     | Both correctly defined, correctly identifies authorisation as the enforcement mechanism, explains how (RBAC, 403 vs 401) |
| 4     | Both defined, correct mechanism identified, partial how                                                                  |
| 3     | Both defined, correct mechanism but incorrect or missing implementation detail                                           |
| 2     | Only one correctly defined                                                                                               |
| 1     | Confuses the two concepts                                                                                                |

---

### B5 — Dead Letter Queue (5 points)

**Full Credit Answer Elements:**
- **Definition:** A queue (or topic) where messages are sent after failing to be processed a configured number of times. Prevents failed messages from blocking the main queue or being silently lost.
- **Why it exists:** In event-driven systems, a message may fail repeatedly due to bugs, corrupt data, or downstream unavailability. Without a DLQ, the consumer retries forever (blocking progress) or drops the message (data loss). DLQ provides a safe holding area.
- **Operational procedure:** Monitoring on DLQ depth (alert if > 0 for critical topics), regular review of DLQ messages by operations team, root cause analysis, replay after fix, alerting on DLQ growth rate.
- **Government example:** Vaccination event consumer fails because a hospital code in the event doesn't exist in the master registry. Event goes to `vaccination.events.dlq`. Operations team investigates, discovers data entry error in hospital registry, corrects the registry, replays the event.

| Score | Criteria                                                                                                        |
| ----- | --------------------------------------------------------------------------------------------------------------- |
| 5     | Correct definition, correct reason for existence, meaningful operational procedure, relevant government example |
| 4     | Definition correct, reason correct, procedure mentioned but thin                                                |
| 3     | Definition correct, some operational awareness                                                                  |
| 2     | Partially correct definition                                                                                    |
| 1     | Knows the term but cannot explain purpose                                                                       |

---

### B6 — Redis Caching Review (5 points)

**Full Credit Answer Elements:**
- **Question 1:** What is the cache invalidation strategy? (When data changes in the DB, how does the cache reflect this? TTL only? Event-driven invalidation? Write-through?) If no strategy exists, the cache will serve stale data.
- **Question 2:** What is the read/write ratio for these queries? Caching is only beneficial for data that is read far more than it is written. Caching write-heavy data adds complexity with little benefit.
- **Failure scenario:** Cache inconsistency — a citizen's eligibility status is updated in the database (they become ineligible for a scheme due to income change), but the cache still serves the old eligibility data for 1 hour. The citizen continues to receive benefits they are no longer entitled to. This is both a financial and legal problem.

| Score | Criteria                                                                                                                         |
| ----- | -------------------------------------------------------------------------------------------------------------------------------- |
| 5     | Both questions are specific and technically meaningful, failure scenario is concrete and demonstrates government-domain thinking |
| 4     | Two good questions, failure scenario present but vague                                                                           |
| 3     | One strong question, reasonable failure scenario                                                                                 |
| 2     | Questions are generic ("what if Redis goes down?")                                                                               |
| 1     | Cannot formulate meaningful questions                                                                                            |

---

## Section C — System Design Rubric

### C1 — Architecture Overview (15 points)

**Expected Services (candidates may name differently — evaluate by function):**

```
Expected Architecture Components:

Core Services:
• GrievanceService — CRUD for grievances, status management
• CitizenAuthService / Integration with TN One ID (SSO)
• OfficerPortalService / OfficerBFF — dashboard, assignment
• NotificationService — SMS (BSNL gateway), email
• EscalationService — timer-based escalation logic
• ReportingService / Analytics — CMO dashboard
• SearchService — grievance search for officers

Data Stores:
• GrievanceService: PostgreSQL (transactional data, 7-year retention)
• Search: Elasticsearch (full-text search for officers)
• Cache: Redis (session, frequently accessed reference data)
• Reporting: MongoDB or a data warehouse pattern (pre-aggregated)

Communication:
• Citizen-facing APIs: REST (simple, works with 2G, cacheable)
• Officer dashboard: REST with WebSocket for real-time updates
• Escalation triggers: Kafka events or scheduled jobs
• Notification: Async via Kafka (NotificationService subscribes)
• CMO reporting: Async, Kafka-driven denormalisation

2G constraint should trigger:
• BFF for mobile with minimal response payloads
• Pagination mandatory
• Image/attachment compression
• Offline-capable filing (local draft → sync on connectivity)
• HTTPS compression (gzip)
```

| Score | Criteria                                                                                                                                                                                                                   |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 13–15 | All major services present, communication choices justified with async/sync rationale, database choices appropriate for each service's access pattern, 2G constraint meaningfully addressed with specific design decisions |
| 10–12 | Most services present, reasonable communication choices, some database justification, 2G partially addressed                                                                                                               |
| 7–9   | Core services present, limited justification for choices, 2G mentioned but not designed for                                                                                                                                |
| 4–6   | Partial service list, no justification for technology choices                                                                                                                                                              |
| 1–3   | Monolith proposed or severely incomplete                                                                                                                                                                                   |
| 0     | No architecture attempted                                                                                                                                                                                                  |

---

### C2 — Escalation Workflow (10 points)

**Expected Answer:**

- **Technology:** Workflow engine (Camunda, Temporal) or scheduled job with persistent timers stored in database. Key insight: In-memory timers are lost on restart. Timer state MUST be persisted (database-backed scheduler, not `Thread.sleep(7 days)`). Camunda's timer events or Temporal's durable timers are the correct answers.

- **Escalation Flow:**
```
Level 0: Filed → Assigned to Department Officer (Day 0)
Level 1: Unresolved after 7 days → Escalate to Department Head
Level 2: Unresolved after 14 days → Escalate to District Collector
Level 3: Unresolved after 21 days → Escalate to Divisional Commissioner
Level 4: Unresolved after 30 days → Flag to CMO dashboard as "Critical Pending"
```

- **CMO Reporting Data:**
  - Grievance ID, district, department, filed date, current status
  - Escalation level, escalation timestamps at each level
  - Resolution time (if resolved)
  - Whether escalation was required to resolve (escalation effectiveness metric)

| Score | Criteria                                                                                                              |
| ----- | --------------------------------------------------------------------------------------------------------------------- |
| 9–10  | Persistent timer correctly identified (not in-memory), 3+ escalation levels with actors, reporting data comprehensive |
| 7–8   | Persistent timer identified, escalation flow present, some reporting data                                             |
| 5–6   | Scheduled job mentioned without addressing restart problem, escalation flow present                                   |
| 3–4   | In-memory timer proposed (does not survive restart), escalation flow partial                                          |
| 1–2   | Vague approach, no escalation flow                                                                                    |

---

### C3 — Peak Load & Hybrid Cloud (10 points)

**Expected Answer:**

- **Auto-scaling:** HPA on citizen-facing services, Kafka consumer group scaling (add partitions + consumers), Redis cache hit rate under burst reduces DB pressure
- **Hybrid strategy:**
  - **Stays on-premise (NIC):** Core database (data sovereignty), officer authentication (PKI), audit logs, core grievance data store
  - **Bursts to AWS:** Stateless API services (GrievanceService pods), notification processing, read replicas for CMO dashboard, Elasticsearch for search
  - Data flows: Kafka connects NIC event backbone to AWS consumers; new grievances written to NIC PostgreSQL primary, replicated to AWS read replica
- **Risk:** Data synchronisation lag between NIC primary and AWS burst services could cause stale reads. Mitigation: Read your own writes via write-through to primary, accept bounded eventual consistency for CMO dashboard (15-minute refresh anyway).

| Score | Criteria                                                                                                                                   |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 9–10  | Auto-scaling mechanisms named, clear hybrid split with justification for what stays vs what bursts, specific risk identified and mitigated |
| 7–8   | Auto-scaling mentioned, hybrid split reasonable, risk acknowledged                                                                         |
| 5–6   | "Scale on cloud" mentioned without specifics, partial hybrid strategy                                                                      |
| 3–4   | No hybrid strategy, only auto-scaling                                                                                                      |
| 1–2   | No meaningful content                                                                                                                      |

---

### C4 — Identify Your Own Gaps (5 points)

**Award marks for intellectual honesty and quality of identified gaps. Accept any well-reasoned gap. Common correct gaps include:**

- Security: Citizen data encryption at rest not specified
- Disaster recovery: RPO/RTO not defined, backup strategy for 7-year retained data
- Offline sync conflict resolution not detailed
- BSNL SMS gateway failure — no fallback notification channel
- Elasticsearch index sizing for 50M+ grievances over 7 years not addressed
- Officer load balancing between departments not defined
- API versioning strategy for the 15+ department integrations not specified

| Score | Criteria                                                                                     |
| ----- | -------------------------------------------------------------------------------------------- |
| 5     | Two genuine, non-trivial gaps identified with clear risk explanation and credible mitigation |
| 3–4   | Two gaps but one is trivial or risk explanation weak                                         |
| 2     | One meaningful gap                                                                           |
| 1     | Gaps identified but reasoning vague                                                          |
| 0     | Claims no gaps OR fails to attempt                                                           |

---

## Section D — Code Review Answer Key

### D1a — Security Vulnerabilities (6 points)

**Expected vulnerabilities (1.5 points each, minimum 4):**

| Vulnerability                                        | Location                                                           | Impact                                                                                                                                          |
| ---------------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **SQL Injection** (OWASP A03)                        | `searchCitizens` method, string concatenation in SQL               | Attacker can extract all citizen data, drop tables, bypass authentication. In a government system with 7.8 crore records, catastrophic.         |
| **PII in Logs** (OWASP A09 — Security Logging)       | `getCitizen` — `auditLogger.log(... + citizen.get().getAadhaar())` | Aadhaar number appears in log files. ELK stores these indefinitely. Every person with log access can read Aadhaar numbers. UIDAI Act violation. |
| **Missing Authentication/Authorisation** (OWASP A01) | All 4 endpoints — no `@PreAuthorize`, no role check                | Any unauthenticated user can read, search, update, or delete any citizen record. In a public deployment, complete data breach.                  |
| **Unrestricted Destructive Operation** (OWASP A01)   | `deleteCitizen` — no role check, no soft-delete                    | Any caller can delete any citizen record permanently. In government context, identity destruction. No audit trail on deletion.                  |
| **Missing Input Validation** (OWASP A03)             | `updateAddress` — no `@Valid`, no `@NotNull`                       | Null or malformed address accepted, potential for data corruption or null pointer in processing.                                                |
| **Mass Assignment / Over-exposure** (OWASP A08)      | `getCitizen` — returns full `Citizen` object including Aadhaar     | Should return a DTO with only required fields. Raw entity with sensitive fields returned to any caller.                                         |

*Award 1.5 per clearly identified vulnerability with impact. Minimum 4 to achieve full 6 points.*

---

### D1b — Aadhaar in ELK (2 points)

**Full credit answer:**
- ELK stores logs, often with long retention (90 days to 7 years in government)
- Elasticsearch indices are searchable by anyone with Kibana access (developers, ops team, security team, auditors)
- A simple Kibana query `aadhaar: 123456789012` allows any ELK user to look up any citizen's activities by Aadhaar number
- This creates a non-IAM-governed side channel for PII access that bypasses all application-layer authorisation controls
- Also: logs may be exported to CSV for audit reporting, ending up in unprotected spreadsheets

| Score | Criteria                                                                        |
| ----- | ------------------------------------------------------------------------------- |
| 2     | Identifies that ELK is a non-IAM access path, explains the side channel risk    |
| 1     | Mentions "logs shouldn't have Aadhaar" without explaining the specific ELK risk |
| 0     | Only mentions UIDAI Act violation (already given in question)                   |

---

### D1c — Endpoint 3 Checklist (2 points)

**Expected checklist (any 3 of these):**

```
□ No authentication check — who is authorised to update a citizen's address?
□ No authorisation check — can this citizen update another citizen's address?
  (IDOR — Insecure Direct Object Reference)
□ No audit trail — who changed the address and when?
□ No input validation on the AddressRequest body
□ No optimistic locking — concurrent updates may overwrite each other
□ Returns generic string "Address updated" — should return updated resource
  or at least acknowledge what was changed for audit purposes
```

| Score | Criteria                                                  |
| ----- | --------------------------------------------------------- |
| 2     | At least 3 valid points, with IDOR or audit trail present |
| 1     | 1–2 valid points                                          |
| 0     | Generic or irrelevant checklist                           |

---

### D2a — Kafka Consumer Bottleneck (2 points)

**Full credit:** The `certificateService.generate()` call takes 3–5 seconds synchronously inside the consumer. Kafka consumers have a `max.poll.interval.ms` (default 5 minutes, but processing latency accumulates). With 3–5 seconds per message and high event volume, the consumer cannot keep pace with the producer. Consumer lag grows linearly. At peak (2 million events/day = ~23 events/second), a 5-second-per-event consumer processes only 12 events/second — half the incoming rate. Lag grows without bound.

---

### D2b — Silent Exception Swallowing (2 points)

**Expected consequences:**
1. **Failed vaccinations are silently lost** — if `repository.save()` throws (DB unavailable), the event is never processed. The consumer commits the offset. The vaccination record is permanently lost. The citizen receives no certificate. There is no alert. The system appears healthy.
2. **No dead letter queue** — failed events should go to a DLQ for reprocessing. Without DLQ + proper error handling, no recovery mechanism exists. Data loss is permanent.

---

### D2c — Missing Idempotency (2 points)

**Expected scenario:**
- Network partition between Kafka broker and consumer after `repository.save()` succeeds but before offset commit
- Kafka redelivers the message on consumer restart
- `repository.save()` runs again — citizen gets two vaccination records, or duplicate certificate generation, or double SMS/email notification
- In a government vaccination system, a duplicate record could incorrectly show a citizen as having received 3 doses when they had 2, affecting their vaccination certificate and international travel document validity

---

### D2d — Additional Issue (2 points)

**Accept any well-explained issue not already covered:**
- PII in log statement (`event.getAadhaar()` in log)
- No transaction boundary — save succeeds but email fails, leaving data inconsistent
- No message filtering — consumer processes all events, even if they belong to a different state/region
- `emailService` not declared/injected — compilation error (if noticed)
- No retry with backoff — all errors treated the same regardless of type

---

### D3a — Pattern Identification (2 points)

**Event Sourcing** (1 point) + **CQRS** (1 point)

Both must be named correctly. Accept "Event Store pattern" for Event Sourcing.

---

### D3b — Point-in-Time Investigation (3 points)

**Full credit:**
- Event Sourcing strongly supports this investigation
- The append-only event log contains every state change with timestamps
- To reconstruct the permit state on a specific date 8 months ago: replay all events with `timestamp <= target_date`
- This gives the exact state of the permit at that moment — which documents were verified, who approved what step, and on what date
- This is superior to traditional databases where UPDATE overwrites history — you cannot reconstruct what the record looked like before the update without a separate audit table
- The immutable log is the audit trail itself

| Score | Criteria                                                                                |
| ----- | --------------------------------------------------------------------------------------- |
| 3     | Correctly explains event replay for point-in-time, contrasts with mutable DB limitation |
| 2     | Understands event replay but doesn't contrast with alternative                          |
| 1     | Knows it helps but cannot explain how                                                   |

---

### D3c — Eventual Consistency in Read Model (2 points)

**Term:** Eventual consistency (between write model/event log and read-optimised view)

**Acceptable in this context:** Yes, with justification:
- The read model is for the citizen portal to display permit status
- A permit approval event being visible on the portal within seconds (not milliseconds) is acceptable
- The event log (ground truth) has the correct state immediately; the read model catches up within seconds
- For an administrative/regulatory investigation (as in D3b), investigators use the event log directly, not the read model
- Not acceptable if the requirement were real-time financial settlement — but permit status display tolerates seconds of lag

---

## Section E — Code Writing Answer Key

### E1 — Idempotent REST Endpoint (12 points)

**Reference Implementation (Java/Spring Boot):**

```java
@RestController
@RequestMapping("/api/grievances")
@Validated
public class GrievanceController {

    private static final Set<String> VALID_DEPARTMENTS = 
        Set.of("WATER", "ROADS", "HEALTH", "REVENUE", "ELECTRICITY");
    
    // In production: Replace with Redis with TTL
    // RedisTemplate<String, IdempotencyRecord> redisTemplate
    private final Map<String, GrievanceResponse> idempotencyStore = 
        new ConcurrentHashMap<>();
    
    @Autowired
    private GrievanceService grievanceService;

    @PostMapping
    public ResponseEntity<?> submitGrievance(
            @RequestHeader(value = "Idempotency-Key", required = false) 
                String idempotencyKey,
            @RequestBody @Valid GrievanceRequest request) {
        
        // Validate idempotency key presence
        if (idempotencyKey == null || idempotencyKey.isBlank()) {
            return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(Map.of("error", "Idempotency-Key header is required"));
        }
        
        // Validate department code
        if (!VALID_DEPARTMENTS.contains(request.getDepartmentCode())) {
            return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(Map.of(
                    "error", "Invalid department code",
                    "validValues", VALID_DEPARTMENTS
                ));
        }
        
        // Check idempotency store
        // In production: idempotencyStore.get() calls Redis with TTL check
        GrievanceResponse existingResponse = 
            idempotencyStore.get(idempotencyKey);
        
        if (existingResponse != null) {
            // Request already processed — return cached result
            return ResponseEntity
                .status(HttpStatus.OK)  // 200 for replay, not 201
                .header("Idempotency-Replay", "true")
                .body(existingResponse);
        }
        
        // Process new request
        GrievanceResponse response = grievanceService.create(request);
        
        // Store result in idempotency store
        // In production: redisTemplate.opsForValue().set(
        //     idempotencyKey, response, 24, TimeUnit.HOURS)
        idempotencyStore.put(idempotencyKey, response);
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(response);
    }
}

// Request DTO with validation
@Data
public class GrievanceRequest {
    
    @NotBlank(message = "citizenId is required")
    private String citizenId;
    
    @NotBlank(message = "departmentCode is required")
    private String departmentCode;
    
    @NotBlank(message = "description is required")
    @Size(max = 1000, message = "description must not exceed 1000 characters")
    private String description;
    
    @NotBlank(message = "locationCode is required")
    private String locationCode;
}
```

**Scoring Breakdown:**

| Criteria                                          | Points | Evaluator Check |
| ------------------------------------------------- | ------ | --------------- |
| Idempotency key check (missing = 400)             | 1      | ✓/✗             |
| Idempotency store lookup before processing        | 1.5    | ✓/✗             |
| Store result after processing                     | 1      | ✓/✗             |
| Return cached result on replay (not reprocess)    | 1      | ✓/✗             |
| TTL concept mentioned (even if not implemented)   | 0.5    | ✓/✗             |
| Department code validation (correct valid values) | 1.5    | ✓/✗             |
| Description not blank + max 1000 chars            | 1.5    | ✓/✗             |
| 201 for new, 200 for replay                       | 1.5    | ✓/✗             |
| 400 for missing/invalid inputs                    | 1.5    | ✓/✗             |
| Error handling (try-catch or framework-level)     | 1      | ✓/✗             |
| **Total**                                         | **12** |                 |

---

### E2 — Circuit Breaker (8 points)

**Reference Implementation (Java):**

```java
public class CircuitBreaker {

    public enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private final int failureThreshold = 5;
    private final long openTimeoutMs = 30_000; // 30 seconds
    private long openedAt = 0;
    private boolean halfOpenRequestInFlight = false;

    public State getState() {
        // Check if OPEN timeout has elapsed → transition to HALF_OPEN
        if (state == State.OPEN && 
            System.currentTimeMillis() - openedAt >= openTimeoutMs) {
            state = State.HALF_OPEN;
            halfOpenRequestInFlight = false;
        }
        return state;
    }

    public synchronized <T> T execute(Supplier<T> operation) 
            throws CircuitOpenException {
        
        State currentState = getState();
        
        if (currentState == State.OPEN) {
            throw new CircuitOpenException(
                "Circuit is OPEN. Failing fast. Retry after " + 
                (openTimeoutMs - (System.currentTimeMillis() - openedAt)) 
                + "ms");
        }
        
        if (currentState == State.HALF_OPEN && halfOpenRequestInFlight) {
            throw new CircuitOpenException(
                "Circuit is HALF_OPEN. One test request already in flight.");
        }
        
        if (currentState == State.HALF_OPEN) {
            halfOpenRequestInFlight = true;
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }

    private synchronized void onSuccess() {
        failureCount = 0;
        state = State.CLOSED;
        halfOpenRequestInFlight = false;
    }

    private synchronized void onFailure() {
        failureCount++;
        halfOpenRequestInFlight = false;
        if (state == State.HALF_OPEN || failureCount >= failureThreshold) {
            state = State.OPEN;
            openedAt = System.currentTimeMillis();
            failureCount = 0;
        }
    }
}

// Custom exception
public class CircuitOpenException extends RuntimeException {
    public CircuitOpenException(String message) { super(message); }
}

// Test demonstrating state transitions
public class CircuitBreakerTest {
    public static void main(String[] args) throws InterruptedException {
        CircuitBreaker cb = new CircuitBreaker();
        
        // Trigger 5 failures → OPEN
        for (int i = 0; i < 5; i++) {
            try {
                cb.execute(() -> { throw new RuntimeException("Service down"); });
            } catch (Exception ignored) {}
        }
        assert cb.getState() == CircuitBreaker.State.OPEN : "Should be OPEN";
        System.out.println("After 5 failures: " + cb.getState()); // OPEN
        
        // Try in OPEN state → CircuitOpenException
        try {
            cb.execute(() -> "test");
        } catch (CircuitOpenException e) {
            System.out.println("Correctly blocked: " + e.getMessage());
        }
        
        // Simulate 30 seconds passing (for test, manipulate time or use small timeout)
        Thread.sleep(30_100);
        assert cb.getState() == CircuitBreaker.State.HALF_OPEN : "Should be HALF_OPEN";
        System.out.println("After 30s timeout: " + cb.getState()); // HALF_OPEN
        
        // Success in HALF_OPEN → CLOSED
        cb.execute(() -> "success");
        assert cb.getState() == CircuitBreaker.State.CLOSED : "Should be CLOSED";
        System.out.println("After success: " + cb.getState()); // CLOSED
    }
}
```

| Criteria                                         | Points |
| ------------------------------------------------ | ------ |
| Three states defined correctly                   | 1      |
| CLOSED → OPEN after 5 failures                   | 1      |
| OPEN → HALF_OPEN after timeout                   | 1      |
| HALF_OPEN allows exactly 1 request               | 0.5    |
| HALF_OPEN success → CLOSED                       | 0.5    |
| HALF_OPEN failure → OPEN again                   | 0.5    |
| CircuitOpenException when OPEN                   | 0.5    |
| `getState()` method present                      | 0.5    |
| Test demonstrates at least 2 state transitions   | 1.5    |
| Thread safety considered (synchronized / atomic) | 1      |

---

### E3 — SQL Query Optimisation (10 points)

**E3a — Root Cause (2 points):**

- **Full Sequential Scan:** The `WHERE` clause filters on `created_at` and `district_code`. Without indexes on these columns, the database performs a full sequential scan of all 50 million rows before filtering, joining, and aggregating.
- Secondary cause: The `JOIN` to departments without an index on `department_id` causes nested loop scan.

**E3b — Index Statement (4 points):**

```sql
-- Primary index: composite covering the filter and grouping columns
-- Column order matters: most selective filter first
CREATE INDEX idx_grievances_district_created 
ON grievances (district_code, created_at, department_id, status, resolved_at);

-- Reasoning:
-- district_code: First filter applied — highly selective for one district query
-- created_at: Date range filter — narrows to 1 year
-- department_id: Used in JOIN — covered by index, avoids table lookup
-- status: Used in CASE WHEN — covering index avoids row fetch
-- resolved_at: Used in AVG calculation — covering index avoids row fetch

-- For the CMO all-Tamil Nadu query (no district filter), 
-- a separate index is optimal:
CREATE INDEX idx_grievances_created_dept 
ON grievances (created_at, department_id, status, resolved_at);
```

**E3c — CMO Dashboard Architecture (4 points):**

**Full credit answer — must include a pre-aggregation strategy:**

- **Problem:** Running this query over 50 million rows across all 38 districts every 15 minutes is not feasible regardless of indexing. The CMO query is a full-table aggregation.

- **Solution: Materialised/Pre-aggregated View approach:**
  - A background job (scheduled every 15 minutes) computes the aggregation and writes results to a small `grievance_stats_cache` table (38 rows — one per department, or 38×38 = 1,444 rows for district+department)
  - CMO dashboard queries the `grievance_stats_cache` table instead of the raw `grievances` table
  - Query on cache returns in <100ms regardless of grievance table size

- **Alternative (more sophisticated):** CQRS with a dedicated reporting data store. Grievance events are streamed via Kafka. A consumer maintains pre-aggregated counters in Redis or MongoDB. CMO dashboard reads Redis — sub-millisecond response.

- **Why index alone is insufficient:** Full aggregation over 50M rows × 38 departments × date filter is a CPU and I/O intensive operation. Index helps filter but the GROUP BY + SUM + AVG still processes millions of rows. Pre-aggregation eliminates this at query time.

---

## Section F — Essay Rubric

### F1 — Build vs Buy API Gateway (12 points)

**Evaluator Framework:**

**Criteria Quality (3 points):**

Strong criteria include:
- Total Cost of Ownership (build cost + ongoing maintenance vs licensing)
- Team expertise: Does the team have API gateway domain knowledge?
- Time to value: How long to build vs configure?
- Feature completeness: Does the existing need match what you'd build?
- Vendor/community support: Is the open-source solution actively maintained?
- Compliance: Does it support government security requirements (OAuth, mTLS, rate limiting)?
- Operational complexity: Who maintains it in production?

*Award 3 for 4+ substantive criteria, 2 for 3 criteria, 1 for generic criteria*

**Recommendation (4 points):**

Expected recommendation: **Adopt an open-source solution (Kong or Tyk)**

Justification:
- 8 engineers cannot build, maintain, and secure an API gateway that also manages 120 APIs and 400 consumers while delivering features
- Kong/Tyk provide: rate limiting, authentication plugins, analytics, developer portal, circuit breaking — out of the box
- Government reference customers exist for both (NIC has deployed Kong)
- Open-source avoids vendor lock-in while community maintains security patches
- TCO of custom build includes: development time, ongoing maintenance, security patching, documentation

*Award 4 for clear recommendation with multi-factor justification. Award 2–3 if recommendation is present but justification is thin. Award 1 if recommendation is present but wrong (recommending custom build without strong justification).*

**Risk Analysis (3 points):**

Top risks and mitigations:
- Kong/Tyk becomes unsupported → Mitigation: Choose widely-adopted projects, contribute to community, maintain architecture so gateway is replaceable
- Open-source configuration complexity → Mitigation: Invest in IaC for gateway config (Terraform provider for Kong), operator training
- Feature gap for specific government requirement → Mitigation: Evaluate requirements against Kong/Tyk feature matrix before committing

**Additional Unknowns (2 points):**

- Current API traffic volumes and rate limiting needs
- Authentication requirements (which IdPs, OAuth flows?)
- Whether GovTech/NIC has an approved gateway solution
- Data classification of APIs (some may require on-premise processing)

---

### F2 — Explaining SAGA to Non-Technical Stakeholder (13 points)

**Evaluator Framework:**

**Tone and Acknowledgement (2 points):**
- Does not dismiss the concern
- Validates that the question reflects sound intuition
- Uses respectful, non-condescending language
- Example: "That is exactly the right question to ask, and your instinct about transactions is correct for a single database. Let me explain why our situation is different."

**Technical Explanation Clarity (4 points):**
- Correctly explains that traditional database transactions require all operations to be in the same database
- Explains that pension disbursement spans Eligibility (one system) → Payment (another system) → Bank Transfer (third party) → Notification — four separate databases/systems
- A single database transaction cannot span across systems owned by different teams with different databases
- If Bank Transfer succeeds but Notification fails, we cannot roll back the bank transfer after the money has moved

**Analogy Quality (3 points):**

*Strong analogies:*
- "Imagine you are sending a money order. The post office deducts from your account, the recipient picks it up at the destination post office. If the recipient post office is closed, the money doesn't magically go back to you — the post office has a process: they hold the money order, notify you, and you can reclaim it. SAGA works the same way — if a step fails, there is a defined reversal process for each completed step."
- "Think of a relay race. If a runner drops the baton, the race doesn't restart from the beginning — the team has a specific rule for what to do. SAGA defines those rules for each step in advance."

*Weak analogies (award partial credit):* Generic business process comparisons without capturing the compensation/reversal concept

**Honesty About Complexity (2 points):**
- Acknowledges SAGA is more complex to build and test
- Explains what the team does: write compensating transactions for each step, write integration tests for failure paths, use a workflow engine (Temporal/Camunda) to manage state, build monitoring for saga state visibility

**Recommendation Clarity (2 points):**
- Clear, definitive recommendation to proceed with SAGA
- States what the alternative (2PC across systems) would require and why it is worse
- Ends with confidence, not uncertainty

---

## Section G — Situational Judgement Rubric

### G1 — Production Incident at 11:45 PM

**Best Answer: B (Rollback immediately)**

| Choice | Score | Reasoning                                                                                                                                                                                                                                  |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| B      | 5     | Correct. 4,000 active citizens. 12% error rate. Rollback is the fastest path to service restoration. Root cause investigation is a next-day activity with full team. This is standard incident response for a deployment-correlated issue. |
| D      | 3     | Pod restart is reasonable but less targeted than rollback. Restarts may not fix connection pool config issue and waste 30 minutes of citizen impact.                                                                                       |
| C      | 1     | Hotfix deployment introduces another change into an already broken production environment. High risk of making things worse.                                                                                                               |
| A      | 0     | Waiting until morning with 4,000 citizens experiencing errors is unacceptable for a government service. No credit.                                                                                                                         |

*Award points for justification quality even if choice is suboptimal.*

---

### G2 — Production Credentials in Code

**Best Answer: B (Escalate formally)**

| Choice | Score | Reasoning                                                                                                                                                                                                                                                                    |
| ------ | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| B      | 5     | Correct. Credentials in code — even private repos — must be treated as compromised. Private repos can be breached. Git history persists even after deletion. Formal escalation protects the organisation and the candidate. Seniority is irrelevant to security obligations. |
| C      | 2     | At least the problem is fixed, but the credentials remain compromised in git history. The security team needs to know to rotate them. Doing it quietly is insufficient and potentially creates liability.                                                                    |
| D      | 1     | Raising it again is correct but letting it go is not.                                                                                                                                                                                                                        |
| A      | 0     | Private repos being "safe" is demonstrably false. No credit.                                                                                                                                                                                                                 |

---

### G3 — Go-Live Failing NFRs

**Best Answer: B (Controlled go-live with throttling)**

| Choice | Score | Reasoning                                                                                                                                                                                                                                                                                                                                         |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| B      | 5     | Best answer. Balances delivery pressure with responsible engineering. Rate limiting at 40% of target load is honest — the system is tested to that load. Formal documentation protects the architect. Committed remediation dates maintain accountability.                                                                                        |
| C      | 3–4   | Technically correct but politically naive for government projects. Stopping go-live entirely requires strong justification. If the risk is formally documented and accepted by the CIO, a controlled go-live may be preferable to further delay. Award 3 if justification is only "NFRs matter." Award 4 if includes specific threshold analysis. |
| D      | 2     | Well-intentioned but engineering under pressure is a leading cause of new defects. 4-week work in 3 weeks via overtime is a risk pattern, not a solution.                                                                                                                                                                                         |
| A      | 0     | Going live knowingly at 40% capacity without mitigation or disclosure is engineering malpractice for a citizen-facing government service. No credit.                                                                                                                                                                                              |

---

### G4 — New Cloud Service Appears Mid-Design

**Best Answer: B (Document, defer, review in 12 months)**

| Choice | Score | Reasoning                                                                                                                                                                                                                                           |
| ------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| B      | 5     | Correct. A 6-month-old service with no government references is not production-ready for a government system. Documenting it in the ADR demonstrates due diligence. The 12-month review keeps optionality open without disrupting current delivery. |
| D      | 3–4   | Prototyping is reasonable but using project budget without approval and potentially disrupting the main design has project management risk. Award 4 if justification includes executive approval and ring-fenced prototype budget.                  |
| C      | 1     | Ignoring it is intellectually dishonest. The architect should always document evaluated alternatives.                                                                                                                                               |
| A      | 0     | Pivoting 40% of a government architecture design for a 6-month-old unproven service with no government references is irresponsible. No credit.                                                                                                      |

---

---

# Evaluator Scorecard & Report Generation Matrix

## Master Scorecard Template

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRE-ASSESSMENT SCORECARD                             │
│                    Senior Engineer → Solution Architect                 │
│                    June 2026 Cohort                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  Candidate Name: ________________________                               │
│  Employee ID: ___________________________                               │
│  Current Role: __________________________                               │
│  Organisation: __________________________                               │
│  Assessment Date: _______________________                               │
│  Evaluator: _____________________________                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SECTION      MAX    RAW     WEIGHTED   COMPETENCY                     │
│               PTS    SCORE   SCORE      MEASURED                       │
│  ──────────   ───    ─────   ────────   ──────────────────────         │
│  A - MCQ      30     ___     ___        Core Technical Knowledge       │
│  B - Short    30     ___     ___        Conceptual Clarity             │
│  C - Design   40     ___     ___        Architectural Thinking         │
│  D - Review   25     ___     ___        Code Comprehension & Analysis  │
│  E - Writing  30     ___     ___        Implementation Ability         │
│  F - Essay    25     ___     ___        Communication & Research       │
│  G - SJT      20     ___     ___        Judgement & Decision-Making    │
│  ──────────   ───    ─────   ────────                                  │
│  TOTAL        200    ___     ___                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Competency Dimension Scoring Matrix

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    COMPETENCY DIMENSION MATRIX                                      │
│                                                                                     │
│  COMPETENCY         SECTIONS     MAX    SCORE   %      LEVEL                       │
│  ──────────         ────────     ───    ─────   ─      ─────                       │
│                                                                                     │
│  Technical          A + D        55     ___     ___    □Novice □Developing         │
│  Knowledge          (MCQ +              ___            □Competent □Expert          │
│                     Code Review)                                                   │
│                                                                                     │
│  Design &           C            40     ___     ___    □Novice □Developing         │
│  Architecture                                          □Competent □Expert          │
│                                                                                     │
│  Implementation     E            30     ___     ___    □Novice □Developing         │
│  Skill                                                 □Competent □Expert          │
│                                                                                     │
│  Conceptual         B            30     ___     ___    □Novice □Developing         │
│  Clarity                                               □Competent □Expert          │
│                                                                                     │
│  Communication      F            25     ___     ___    □Novice □Developing         │
│  & Research                                            □Competent □Expert          │
│                                                                                     │
│  Judgement          G            20     ___     ___    □Novice □Developing         │
│                                                        □Competent □Expert          │
│                                                                                     │
│  COMPETENCY LEVEL SCALE:                                                            │
│  Novice (<50%): Foundational gaps. Pre-reading mandatory.                          │
│  Developing (50–69%): Some gaps. Focused pre-reading required.                     │
│  Competent (70–84%): Ready for course with standard preparation.                   │
│  Expert (85–100%): Can contribute as a peer in course discussions.                 │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Evaluator Annotation Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EVALUATOR ANNOTATION CODES                           │
│                                                                         │
│  Use these codes in margins when reviewing answers:                    │
│                                                                         │
│  ✓+   Exceeds expectation — shows depth beyond what was asked         │
│  ✓    Correct and complete                                              │
│  ✓-   Correct but incomplete or imprecise                              │
│  ~    Partially correct — award partial marks                          │
│  ✗    Incorrect                                                         │
│  NT   Not attempted                                                     │
│  GK   Good reasoning but incorrect conclusion                          │
│  IH   Intellectual honesty shown (positive signal)                     │
│  RED  Flag — concerning answer (security blindspot, poor judgement)    │
│  DOM  Strong domain knowledge demonstrated (positive signal)           │
│  COM  Communication clarity — unusually clear or unusually unclear     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Candidate Report Template

```
╔═════════════════════════════════════════════════════════════════════════╗
║            PRE-ASSESSMENT EVALUATION REPORT                             ║
║            Senior Engineer → Solution Architect Program                 ║
║            June 2026 Cohort                                             ║
╠═════════════════════════════════════════════════════════════════════════╣
║  CANDIDATE INFORMATION                                                  ║
║  ─────────────────────                                                  ║
║  Name:              ________________________________                    ║
║  Assessment Date:   ________________________________                    ║
║  Evaluator:         ________________________________                    ║
║  Report Generated:  ________________________________                    ║
╠═════════════════════════════════════════════════════════════════════════╣
║  OVERALL RESULT                                                         ║
║  ──────────────                                                         ║
║                                                                         ║
║  Total Score:    ___ / 200   (____ %)                                  ║
║                                                                         ║
║  ┌──────────────────────────────────────────────────────────────┐      ║
║  │  Score Band        Range        Recommendation               │      ║
║  │  ──────────────    ─────────    ──────────────────────────── │      ║
║  │  Distinguished     ≥ 170        □ Enrol — Peer Contributor   │      ║
║  │  Proficient        140–169      □ Enrol — Standard Track     │      ║
║  │  Adequate          120–139      □ Enrol — Enhanced Support   │      ║
║  │  Developing        100–119      □ Conditional — Pre-work     │      ║
║  │  Not Ready         < 100        □ Defer — Foundational Work  │      ║
║  └──────────────────────────────────────────────────────────────┘      ║
║                                                                         ║
║  THIS CANDIDATE FALLS IN BAND: _____________________________           ║
║  RECOMMENDATION: _____________________________________________          ║
╠═════════════════════════════════════════════════════════════════════════╣
║  SECTION-BY-SECTION RESULTS                                             ║
║  ─────────────────────────                                              ║
║                                                                         ║
║  A — Core Technical Knowledge     ___ / 30   (____%)                   ║
║  B — Conceptual Clarity           ___ / 30   (____%)                   ║
║  C — System Design                ___ / 40   (____%)                   ║
║  D — Code Comprehension           ___ / 25   (____%)                   ║
║  E — Code Writing                 ___ / 30   (____%)                   ║
║  F — Communication & Research     ___ / 25   (____%)                   ║
║  G — Situational Judgement        ___ / 20   (____%)                   ║
╠═════════════════════════════════════════════════════════════════════════╣
║  COMPETENCY RADAR PROFILE                                               ║
║  ────────────────────────                                               ║
║                                                                         ║
║  Technical Knowledge    [████████████░░░░] ___% □N □D □C □E           ║
║  Design & Architecture  [████████░░░░░░░░] ___% □N □D □C □E           ║
║  Implementation Skill   [██████████░░░░░░] ___% □N □D □C □E           ║
║  Conceptual Clarity     [████████████░░░░] ___% □N □D □C □E           ║
║  Communication          [██████████████░░] ___% □N □D □C □E           ║
║  Judgement              [████████████████] ___% □N □D □C □E           ║
║                                                                         ║
║  N=Novice D=Developing C=Competent E=Expert                            ║
╠═════════════════════════════════════════════════════════════════════════╣
║  IDENTIFIED STRENGTHS                                                   ║
║  ────────────────────                                                   ║
║                                                                         ║
║  1. _______________________________________________________________      ║
║     _______________________________________________________________      ║
║                                                                         ║
║  2. _______________________________________________________________      ║
║     _______________________________________________________________      ║
║                                                                         ║
║  3. _______________________________________________________________      ║
║     _______________________________________________________________      ║
╠═════════════════════════════════════════════════════════════════════════╣
║  IDENTIFIED GAPS & DEVELOPMENT AREAS                                    ║
║  ───────────────────────────────────                                    ║
║                                                                         ║
║  CRITICAL GAPS (Must address before Day 1):                            ║
║                                                                         ║
║  1. _______________________________________________________________      ║
║     Recommended resource: ______________________________________        ║
║                                                                         ║
║  2. _______________________________________________________________      ║
║     Recommended resource: ______________________________________        ║
║                                                                         ║
║  SIGNIFICANT GAPS (Address during program):                            ║
║                                                                         ║
║  1. _______________________________________________________________      ║
║                                                                         ║
║  2. _______________________________________________________________      ║
╠═════════════════════════════════════════════════════════════════════════╣
║  SECURITY MINDSET ASSESSMENT                                            ║
║  ────────────────────────────                                           ║
║  (Evaluator: Specifically assess based on D1, G2 responses)           ║
║                                                                         ║
║  □ Strong — Proactively identifies security issues without prompting   ║
║  □ Adequate — Identifies obvious security issues when asked            ║
║  □ Developing — Misses significant security issues in code review      ║
║  □ Concerning — Demonstrated poor security judgement (flag to PM)      ║
║                                                                         ║
║  Notes: _______________________________________________________         ║
╠═════════════════════════════════════════════════════════════════════════╣
║  COMMUNICATION & STAKEHOLDER READINESS                                  ║
║  ─────────────────────────────────────                                  ║
║  (Evaluator: Assess based on Section F responses)                      ║
║                                                                         ║
║  □ Ready to present to technical + non-technical stakeholders          ║
║  □ Ready for technical stakeholders; needs coaching for non-technical  ║
║  □ Needs significant coaching before stakeholder-facing role           ║
║                                                                         ║
║  Notes: _______________________________________________________         ║
╠═════════════════════════════════════════════════════════════════════════╣
║  FACILITATOR GUIDANCE                                                   ║
║  ────────────────────                                                   ║
║  (Private — for facilitator use only)                                  ║
║                                                                         ║
║  Suggested seating/grouping consideration:                             ║
║  □ Pair with stronger architect background participant                 ║
║  □ Suitable to lead peer review sessions                               ║
║  □ May need additional check-ins on Days 1–3                          ║
║                                                                         ║
║  Special observations:                                                  ║
║  ______________________________________________________________         ║
║  ______________________________________________________________         ║
║  ______________________________________________________________         ║
╠═════════════════════════════════════════════════════════════════════════╣
║  CONDITIONAL ENROLMENT CONDITIONS                                       ║
║  (Complete only if recommendation is Conditional or Defer)             ║
║                                                                         ║
║  Required actions before enrolment confirmed:                          ║
║                                                                         ║
║  1. □ Complete LMS module: _________________________________           ║
║  2. □ Submit written reflection on: ________________________           ║
║  3. □ Re-assessment on section(s): _________________________           ║
║                                                                         ║
║  Deadline for conditional requirements: ____________________           ║
╠═════════════════════════════════════════════════════════════════════════╣
║  EVALUATOR SIGN-OFF                                                     ║
║                                                                         ║
║  Evaluator Name:    ____________________________                        ║
║  Designation:       ____________________________                        ║
║  Signature:         ____________________________                        ║
║  Date:              ____________________________                        ║
║                                                                         ║
║  Second Evaluator   ____________________________  (if score 100–120)   ║
║  (Borderline cases require two evaluators)                             ║
╚═════════════════════════════════════════════════════════════════════════╝
```

---

## Batch Summary Dashboard

```
┌─────────────────────────────────────────────────────────────────────────┐
│              BATCH SUMMARY — PROGRAMME COORDINATOR VIEW                 │
│              Cohort: June 2026 | Assessment Date: ___________           │
│              Total Assessed: _____ / Target: 20–25                     │
├─────────────────────────────────────────────────────────────────────────┤
│  DISTRIBUTION BY BAND                                                   │
│  ────────────────────                                                   │
│  Distinguished (≥170):  _____ candidates  (____%)                      │
│  Proficient  (140–169): _____ candidates  (____%)                      │
│  Adequate    (120–139): _____ candidates  (____%)                      │
│  Developing  (100–119): _____ candidates  (____%)                      │
│  Not Ready   (<100):    _____ candidates  (____%)                      │
├─────────────────────────────────────────────────────────────────────────┤
│  BATCH COMPETENCY GAPS (Sections where avg < 60%)                      │
│  ─────────────────────────────────────────────────                      │
│  □ Section A — Technical Knowledge   Avg: ___/30                       │
│  □ Section B — Conceptual Clarity    Avg: ___/30                       │
│  □ Section C — System Design         Avg: ___/40                       │
│  □ Section D — Code Review           Avg: ___/25                       │
│  □ Section E — Code Writing          Avg: ___/30                       │
│  □ Section F — Communication         Avg: ___/25                       │
│  □ Section G — Judgement             Avg: ___/20                       │
├─────────────────────────────────────────────────────────────────────────┤
│  FACILITATOR ACTIONS BASED ON BATCH GAPS                               │
│  ───────────────────────────────────────                                │
│  If Section C avg < 60%: Extend Day 1 system design fundamentals      │
│  If Section D avg < 60%: Add code review lab in Day 1 afternoon       │
│  If Section E avg < 60%: Add pair programming warm-up exercise        │
│  If Section G avg < 60%: Add ethical/judgement discussion in Day 1    │
├─────────────────────────────────────────────────────────────────────────┤
│  SECURITY MINDSET FLAGS                                                 │
│  ──────────────────────                                                 │
│  Candidates flagged "Concerning" security mindset: _____               │
│  Names: _________________________________________________               │
│  Action: Notify Programme Manager before Day 1                         │
├─────────────────────────────────────────────────────────────────────────┤
│  DEFERRED CANDIDATES                                                    │
│  ───────────────────                                                    │
│  Count: _____                                                           │
│  Names + scores: _______________________________________________        │
│  Recommended deferral period: 3 months with LMS completion            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*Assessment Document Version: 1.0*
*Senior Engineer → Solution Architect Program | June 2026*
*This document contains answer keys — Evaluator Access Only*
*Candidate-facing version excludes Sections: Answer Key, Rubrics, Scorecard, Report Template*