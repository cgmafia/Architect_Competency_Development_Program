# Pre-Assessment Test
## Senior Engineer → Solution Architect Accelerated Program
### June 2026 Cohort | Batch: 20–25 Participants

---

> **Assessment Administration Guide**
>
> **Duration:** 3 Hours (No extensions)
> **Format:** Online via LMS + Code Submission via Git repository
> **Sections:** 7 Sections | 200 Total Points
> **Passing Threshold for Program Entry:** 120/200 (60%)
> **Recommended Score for Full Readiness:** 160/200 (80%)
>
> **Instructions to Candidate:**
> - Read every question fully before answering
> - For code questions, write real, compilable code — not pseudocode unless explicitly asked
> - For paragraph/essay questions, quality of reasoning matters more than length
> - You may use official documentation (MDN, Spring docs, Kubernetes docs) but not AI tools
> - Time is a real constraint. Move on if stuck. Return if time permits.
> - Submit all answers via LMS before the timer expires. Late submissions are not accepted.

---

# Assessment Structure Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ASSESSMENT STRUCTURE                                 │
│                                                                         │
│  SECTION   TYPE                          POINTS   TIME GUIDE           │
│  ───────   ────                          ──────   ──────────           │
│  A         Multiple Choice (Core)        30       20 minutes           │
│  B         Conceptual Short Answer       30       25 minutes           │
│  C         System Design Scenario        40       45 minutes           │
│  D         Code Reading & Review         25       25 minutes           │
│  E         Code Writing                  30       30 minutes           │
│  F         Trade-off & Research Essay    25       20 minutes           │
│  G         Situational Judgement         20       15 minutes           │
│            ──────────────────────────────────────────────────          │
│  TOTAL                                   200      180 minutes          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# SECTION A — Multiple Choice Questions
## Core Technical Knowledge
### 30 Points | 15 Questions | 2 Points Each | Time Guide: 20 Minutes

**Instructions:** Select the single best answer for each question.

---

**A1.** A government payment portal must process 800,000 transactions per day. During the last 3 days of the financial year, traffic spikes to 5× the daily average within a 4-hour window. Which database consistency model is most appropriate for the transaction ledger that records debit/credit entries?

- A) Eventual consistency — high availability is more important than ledger accuracy
- B) Strong consistency — every debit must be immediately reflected in all reads
- C) Causal consistency — transaction order matters but not cross-user reads
- D) Read-your-writes consistency — sufficient because users only see their own transactions

---

**A2.** You are reviewing an Architecture Decision Record (ADR) submitted by a junior architect who recommends using blockchain for a state government's internal employee leave management system. The primary stated reason is "tamper-evident audit trail." Which response best reflects sound architectural judgment?

- A) Approve — blockchain provides an excellent tamper-evident audit trail for any use case
- B) Reject — blockchain has no legitimate use in government systems
- C) Reject — a traditional database with proper access controls, triggers, and write-once audit tables achieves the same goal without blockchain's operational overhead and throughput limitations
- D) Approve with conditions — as long as they use a private permissioned blockchain

---

**A3.** In a microservices system serving a national health portal, the `AppointmentService` calls `DoctorProfileService` synchronously. During a load test, `DoctorProfileService` begins responding in 8 seconds. Which sequence of events MOST accurately describes what will happen to `AppointmentService` without resilience patterns?

- A) AppointmentService will queue requests and process them when DoctorProfileService recovers
- B) AppointmentService threads will block waiting for DoctorProfileService responses, exhausting its thread pool, causing AppointmentService itself to become unresponsive to all callers
- C) AppointmentService will automatically retry failed requests with exponential backoff
- D) The API gateway will detect the slowdown and route traffic to a backup AppointmentService instance

---

**A4.** In Kubernetes, what is the functional difference between a liveness probe and a readiness probe?

- A) Liveness checks if the pod can receive traffic; readiness checks if the pod process is alive
- B) Liveness failure causes the pod to be restarted; readiness failure removes the pod from the service endpoint list without restarting it
- C) Both perform the same function but liveness is for HTTP services and readiness is for TCP services
- D) Readiness failure causes the pod to be restarted; liveness failure removes the pod from rotation

---

**A5.** A team is migrating a 15-year-old Oracle-based state treasury system to PostgreSQL microservices. The legacy system is live and processing transactions 24×7. Which migration approach MINIMISES risk while ensuring zero downtime?

- A) Big bang migration — export all data, run new system in parallel for 1 week, then cut over on a Sunday night
- B) Strangler Fig with CDC — route new functionality to new services gradually while Debezium keeps new PostgreSQL in sync with legacy Oracle
- C) Direct cut-over — run new system in staging for 6 months, then replace legacy in one deployment
- D) Fork the codebase — run both systems indefinitely and merge data monthly

---

**A6.** Which statement about the SAGA pattern is CORRECT?

- A) SAGA uses a two-phase commit coordinator to ensure atomic distributed transactions
- B) SAGA breaks a distributed transaction into local transactions, each publishing events; failures are handled by compensating transactions
- C) SAGA is only applicable to choreography-based architectures and cannot be used with orchestrators
- D) SAGA guarantees strong consistency across all participating services

---

**A7.** You have a Terraform configuration that manages 200 resources across a government cloud environment. A new engineer accidentally runs `terraform apply` with a misconfigured variable pointing to the wrong environment, destroying 15 production resources before someone stops the run. Which Terraform feature, if properly implemented, would have MOST directly prevented this?

- A) Terraform modules
- B) Remote state with S3 backend
- C) Workspace separation with environment-specific variable files and mandatory `plan` review gates in CI/CD before apply
- D) Terraform Cloud

---

**A8.** In the context of Zero Trust Architecture, a service inside the Kubernetes cluster makes a call to another internal service. Which statement BEST reflects Zero Trust principles?

- A) The call should be trusted automatically because both services are inside the cluster network perimeter
- B) The call must be authenticated and authorised at the network level using mTLS certificates, and at the application level using service account tokens, regardless of network location
- C) Zero Trust only applies to external traffic entering the cluster; internal service-to-service calls are trusted by definition
- D) Zero Trust requires re-authenticating the end user for every internal service call

---

**A9.** A team is designing a citizen notification system that sends SMS and email for 12 different government services. Each service currently calls the notification logic directly. What architectural pattern BEST solves this duplication while enabling independent scaling of notification delivery?

- A) Extract notification code into a shared library imported by all 12 services
- B) Create a dedicated NotificationService that the 12 services publish events to; NotificationService subscribes and handles delivery independently
- C) Use a database table as a notification queue that all services write to and a cron job reads from
- D) Standardise the notification code and copy it into each service to avoid network calls

---

**A10.** What is the PRIMARY purpose of an idempotency key in a payment API?

- A) To encrypt the payment request payload
- B) To authenticate the calling service
- C) To ensure that retrying the same request due to network failure does not result in duplicate processing
- D) To rate-limit payment requests per user

---

**A11.** You are reviewing a Dockerfile for a government web service. The final image is 1.8 GB and the base image is `ubuntu:22.04`. The application is a Spring Boot JAR. What is the MOST impactful change to reduce image size and attack surface?

- A) Use `COPY --chown` to reduce layer count
- B) Switch to a multi-stage build using `eclipse-temurin:17-jre-alpine` as the final stage, including only the JAR and its runtime dependencies
- C) Use `RUN apt-get clean` at the end of the Dockerfile
- D) Compress the JAR file before copying it into the image

---

**A12.** Which of the following CORRECTLY describes the relationship between SLA, SLO, and SLI?

- A) SLA is internal target, SLO is contractual commitment, SLI is the measurement
- B) SLI is the measurement, SLO is the internal target (stricter than SLA), SLA is the contractual commitment with consequences for breach
- C) SLO and SLA are the same; SLI is the tool used to measure them
- D) SLI is the contractual commitment, SLO is the team's aspirational target, SLA is what Prometheus measures

---

**A13.** In Domain-Driven Design, an Aggregate Root serves which PRIMARY function?

- A) It is the database primary key for a group of related tables
- B) It is the single entry point through which all external interactions with the aggregate occur, enforcing business invariants and consistency boundaries
- C) It is the parent class from which all domain entities inherit common fields
- D) It represents the microservice boundary in a system using DDD

---

**A14.** A government portal's Elasticsearch cluster shows degraded performance. Investigation reveals the cluster has 200 indices with 5 shards each (1,000 total shards) but only 3 data nodes. What is MOST LIKELY causing the performance issue?

- A) Elasticsearch does not support more than 100 indices per cluster
- B) Too many shards per node — each shard consumes JVM heap; over-sharding causes heap pressure and garbage collection pauses, degrading performance
- C) The indices are using the wrong mapping type
- D) The cluster needs more master nodes to manage 200 indices

---

**A15.** Which OWASP Top 10 vulnerability is demonstrated by this code?

```java
String query = "SELECT * FROM citizens WHERE name = '" + request.getParam("name") + "'";
ResultSet rs = statement.executeQuery(query);
```

- A) Broken Access Control
- B) Security Misconfiguration
- C) Injection (SQL Injection)
- D) Insecure Deserialization

---

# SECTION B — Conceptual Short Answer
## Clarity of Concepts
### 30 Points | 6 Questions | 5 Points Each | Time Guide: 25 Minutes

**Instructions:** Answer each question in 3–6 sentences. Be precise. Bullet points are acceptable. Vague or circular answers score 0.

---

**B1.** Explain the difference between horizontal scaling and vertical scaling. For a stateless microservice handling citizen login requests, which would you prefer and why? What characteristic of the service makes this the right choice?

*(5 points)*

---

**B2.** A product manager tells you: "We need 99.999% uptime for our pension disbursement portal." As the architect, what is your first response — and what specific questions do you ask before agreeing to design for that SLA?

*(5 points)*

---

**B3.** In your own words, explain what eventual consistency means. Give one example of a government system where eventual consistency is acceptable, and one where it is not. Justify both choices.

*(5 points)*

---

**B4.** What is the difference between authentication and authorisation? In a multi-ministry government portal where Ministry A staff should not access Ministry B's citizen records, which mechanism (authentication or authorisation) primarily enforces this, and how?

*(5 points)*

---

**B5.** Explain what a Dead Letter Queue (DLQ) is, why it exists in event-driven systems, and what operational procedure should be in place for messages that land in a DLQ. Give a concrete government system example.

*(5 points)*

---

**B6.** A new team member proposes adding Redis caching to every database query to improve performance. As the architect reviewing this proposal, what are the two most important questions you would ask before approving, and what failure scenario would you highlight as a risk?

*(5 points)*

---

# SECTION C — System Design Scenario
## Design Thinking & Architectural Judgment
### 40 Points | Time Guide: 45 Minutes

**Instructions:** Read the scenario completely before drawing or writing anything. You will be evaluated on the quality of your thinking, the appropriateness of your decisions, and your ability to identify trade-offs — not on the elegance of your diagrams. Diagrams may be hand-drawn and photographed or drawn using text-based ASCII notation.

---

## The Scenario

The Government of Tamil Nadu has commissioned a **Unified District Grievance Management System (UDGMS)** to replace 38 separate departmental complaint portals. The system must:

**Functional Scope:**
- Allow any resident (citizen) to file a grievance against any of 38 departments (Water, Roads, Electricity, Revenue, Health, Education, etc.)
- Allow departmental officers to view, respond to, and resolve grievances assigned to their department
- Automatically escalate unresolved grievances after defined SLA periods (e.g., 7 days unresolved → escalate to District Collector)
- Generate weekly reports for the Chief Minister's Office (CMO) aggregating resolution statistics by district and department
- Send SMS and email notifications to citizens at each status change

**Scale:**
- Tamil Nadu has 38 districts, 232 taluks, and approximately 7.8 crore (78 million) residents
- Expected: 50,000 grievances filed per day during normal operation; 200,000 per day during post-disaster scenarios (cyclone, flood)
- 8,000 departmental officers across 38 departments accessing the system
- CMO dashboard must refresh every 15 minutes with near-real-time data

**Non-Functional Requirements:**
- Availability: 99.5% monthly (approximately 3.6 hours downtime per month allowed)
- Response time: Citizen-facing APIs p95 < 2 seconds; officer dashboard p95 < 4 seconds
- Data retention: Grievances retained for 7 years (government audit requirement)
- Security: Citizen login via Tamil Nadu One ID (SSO); officer login via Government PKI certificate
- The system must work on 2G mobile connections for rural citizens

**Constraints:**
- Must be deployable on NIC Tamil Nadu State Data Centre (on-premise) with option to burst to AWS ap-south-1 (Mumbai) during peak load
- Must integrate with existing Tamil Nadu One ID for citizen authentication
- Must send notifications via BSNL SMS gateway (existing government contract)
- Budget for third-party managed services is limited — prefer open-source where operationally feasible

---

## Questions

**C1. Architecture Overview (15 points)**

Draw and describe a high-level architecture for UDGMS. Your answer must include:

a) The primary services/components you would create and their responsibilities (name them and describe each in 1 sentence)

b) How these services communicate (synchronous vs asynchronous — justify each choice)

c) Your database choices for each service (name the database type and justify in 1 sentence per choice)

d) How the 2G mobile constraint influences your API design decisions

---

**C2. Escalation Workflow Design (10 points)**

Design the automatic escalation mechanism. Answer the following:

a) What technology/pattern would you use to implement the 7-day escalation timer? (Hint: Consider what happens if the server restarts after 3 days — is the timer lost?)

b) Draw the escalation flow showing at least 3 escalation levels and the actors involved

c) What data must be stored to support CMO reporting on escalation rates?

---

**C3. Peak Load & Hybrid Cloud Strategy (10 points)**

Post-cyclone Vardah (2016), Chennai received 400,000 public complaints in 72 hours. Design for this burst scenario:

a) How does your architecture handle 4× normal load without manual intervention?

b) Describe specifically how the hybrid NIC + AWS burst strategy works — which components scale to cloud and which stay on-premise, and why?

c) What is the risk of your chosen approach and how do you mitigate it?

---

**C4. Identify the Gaps (5 points)**

Looking at your own design, identify TWO architectural weaknesses or risks you have not fully addressed. For each:
- State what the gap is
- Explain why it is a risk
- Propose how you would address it given more time

*(Note: Candidates who claim no gaps exist will lose all 5 points. Every architecture has gaps.)*

---

# SECTION D — Code Reading & Review
## Code Comprehension and Critical Analysis
### 25 Points | Time Guide: 25 Minutes

**Instructions:** Read each code snippet carefully. Answer the questions that follow. You do not need to rewrite the code unless specifically asked.

---

## D1. Security Review (10 points)

Review the following Spring Boot REST controller for a government citizen data API:

```java
@RestController
@RequestMapping("/api/citizens")
public class CitizenController {

    @Autowired
    private CitizenRepository citizenRepository;
    
    @Autowired
    private AuditLogger auditLogger;

    // Endpoint 1: Get citizen by ID
    @GetMapping("/{id}")
    public ResponseEntity<Citizen> getCitizen(@PathVariable Long id) {
        Optional<Citizen> citizen = citizenRepository.findById(id);
        if (citizen.isPresent()) {
            auditLogger.log("Citizen record accessed: " + id + 
                          ", Aadhaar: " + citizen.get().getAadhaar());
            return ResponseEntity.ok(citizen.get());
        }
        return ResponseEntity.notFound().build();
    }

    // Endpoint 2: Search citizens
    @GetMapping("/search")
    public List<Citizen> searchCitizens(@RequestParam String query) {
        String sql = "SELECT * FROM citizens WHERE name LIKE '%" + query + "%' " +
                     "OR mobile LIKE '%" + query + "%'";
        return citizenRepository.executeRawQuery(sql);
    }

    // Endpoint 3: Update citizen address
    @PutMapping("/{id}/address")
    public ResponseEntity<String> updateAddress(
            @PathVariable Long id,
            @RequestBody AddressRequest address) {
        citizenRepository.updateAddress(id, address);
        return ResponseEntity.ok("Address updated successfully");
    }

    // Endpoint 4: Delete citizen record
    @DeleteMapping("/{id}")
    public ResponseEntity<String> deleteCitizen(@PathVariable Long id) {
        citizenRepository.deleteById(id);
        return ResponseEntity.ok("Deleted");
    }
}
```

**D1a.** Identify ALL security vulnerabilities in this code. For each vulnerability:
- Name the vulnerability type (use OWASP terminology where applicable)
- Identify the specific line(s) where it occurs
- Explain the potential impact in a government context
*(6 points — 1.5 points per correct vulnerability found, minimum 4 expected)*

**D1b.** The `auditLogger` on endpoint 1 is logging the Aadhaar number. Beyond the privacy law violation (UIDAI Act), what operational security risk does this create in a system using an ELK stack? *(2 points)*

**D1c.** Endpoint 3 updates a citizen's address. Write a 3-line checklist of what this endpoint is missing from a correctness and security standpoint (no code required — checklist only). *(2 points)*

---

## D2. Distributed Systems Code Review (8 points)

Review this Kafka consumer implementation for a vaccination event processor:

```java
@Component
public class VaccinationEventConsumer {

    @Autowired
    private VaccinationRepository repository;
    
    @Autowired
    private CertificateService certificateService;

    @KafkaListener(topics = "vaccination.events", groupId = "vacc-processor")
    public void processEvent(VaccinationEvent event) {
        try {
            // Save to database
            repository.save(event);
            
            // Generate certificate (calls external PDF service - takes 3-5 seconds)
            byte[] certificate = certificateService.generate(event);
            
            // Send certificate via email
            emailService.send(event.getEmail(), certificate);
            
            // Log success
            log.info("Processed vaccination for: " + event.getBeneficiaryId() 
                   + ", Aadhaar: " + event.getAadhaar());
                   
        } catch (Exception e) {
            log.error("Failed to process event: " + e.getMessage());
        }
    }
}
```

**D2a.** This consumer will experience performance problems at scale. Identify the specific bottleneck and explain why it will cause consumer group lag to grow. *(2 points)*

**D2b.** The exception handler silently swallows all errors. What are the TWO most serious consequences of this in a production vaccination tracking system? *(2 points)*

**D2c.** There is no idempotency check. Describe a specific scenario in this system where the absence of idempotency causes a real-world problem for a citizen. *(2 points)*

**D2d.** Identify one additional issue beyond those mentioned above and explain its impact. *(2 points)*

---

## D3. Architecture Pattern Identification (7 points)

Read the following system description and answer the questions:

> *"When a government officer approves a building permit, the system records the approval event to an append-only log. The current permit status is never stored directly — instead, it is computed by replaying all events from the log: `PermitApplicationReceived`, `DocumentsVerified`, `SiteInspectionScheduled`, `SiteInspectionCompleted`, `FireNOCApproved`, `PermitApproved`. A separate read-optimised view is maintained by a background process that consumes these events and writes a pre-computed status to a fast-read database for the citizen portal."*

**D3a.** Name the TWO architectural patterns being used in this system. *(2 points)*

**D3b.** A citizen claims their permit was wrongly rejected. The Vigilance Department needs to know the exact state of the permit as it stood on a specific date 8 months ago. How does this architecture support or hinder that investigation? *(3 points)*

**D3c.** What is the technical term for the risk that the read-optimised view shows a different status than the event log for a brief period? Is this acceptable in this context? Justify your answer. *(2 points)*

---

# SECTION E — Code Writing
## Hands-On Implementation Ability
### 30 Points | Time Guide: 30 Minutes

**Instructions:** Write real, runnable code. Language choice: Java (Spring Boot), Python (FastAPI/Flask), or Node.js (Express). State your language at the top of each answer. You will be evaluated on correctness, production readiness, and handling of edge cases — not just on whether the happy path works.

---

**E1. Idempotent REST Endpoint (12 points)**

Write a REST endpoint for processing a grievance submission that is idempotent. The endpoint must:

- Accept a POST request with a JSON body containing: `citizenId`, `departmentCode`, `description`, `locationCode`
- Require an `Idempotency-Key` header (return 400 if missing)
- Check if the idempotency key has been seen before (use an in-memory Map or Redis pseudocode if Redis not available — clearly comment your approach)
- If the key exists: return the previously stored response (do not process again)
- If the key is new: save the grievance, store the result against the idempotency key with a 24-hour TTL concept, return 201 with the new grievanceId
- Return appropriate HTTP status codes for all cases
- Include input validation: `description` must not be blank and must be under 1000 characters; `departmentCode` must be one of: `WATER`, `ROADS`, `HEALTH`, `REVENUE`, `ELECTRICITY`

*(12 points: 4 for idempotency logic, 3 for validation, 3 for correct HTTP semantics, 2 for error handling)*

---

**E2. Circuit Breaker Simulation (8 points)**

Without using any external library (no Resilience4j, no Hystrix), implement a simple CircuitBreaker class in your chosen language that:

- Has three states: CLOSED, OPEN, HALF_OPEN
- Opens after 5 consecutive failures
- Transitions to HALF_OPEN after 30 seconds in OPEN state
- In HALF_OPEN state: allows exactly 1 request through; if it succeeds → CLOSED; if it fails → OPEN again
- Throws a `CircuitOpenException` (or returns an error) when the circuit is OPEN
- Include a `getState()` method

Write a brief test demonstrating the state transitions.

*(8 points: 3 for state machine correctness, 3 for transition logic, 2 for test)*

---

**E3. SQL Query Optimisation (10 points)**

You are given this query that runs against a grievance database. It is taking 45 seconds on a table with 50 million rows:

```sql
SELECT 
    d.department_name,
    COUNT(g.id) AS total_grievances,
    SUM(CASE WHEN g.status = 'RESOLVED' THEN 1 ELSE 0 END) AS resolved_count,
    AVG(JULIANDAY(g.resolved_at) - JULIANDAY(g.created_at)) AS avg_resolution_days
FROM grievances g
JOIN departments d ON g.department_id = d.id
WHERE 
    g.created_at >= '2024-01-01' 
    AND g.created_at < '2025-01-01'
    AND g.district_code = 'TN-CB'
GROUP BY d.department_name
ORDER BY total_grievances DESC;
```

**E3a.** Identify the likely reason this query is slow on 50 million rows. *(2 points)*

**E3b.** Write the exact SQL `CREATE INDEX` statement(s) that would most improve this query's performance. Justify why you chose those specific columns and in that order. *(4 points)*

**E3c.** The CMO dashboard runs this query (aggregated for all of Tamil Nadu, not one district) every 15 minutes and it must return in under 2 seconds. Index alone will not be sufficient. Propose and briefly explain an architectural solution beyond just adding an index. *(4 points)*

---

# SECTION F — Trade-off & Research Essay
## Communication, Research Depth & Reasoning Quality
### 25 Points | 2 Questions | Time Guide: 20 Minutes

**Instructions:** Write structured, professional responses. These answers demonstrate how you would communicate architectural decisions to a technical panel or ministry stakeholder. Clarity, structure, and the quality of your reasoning are evaluated — not word count.

---

**F1. The Build vs Buy Decision (12 points)**

A State Government CIO asks you to evaluate whether to build a custom API Gateway in-house or adopt an open-source solution (Kong, Tyk, or NGINX Plus). The government has 120 APIs across 15 departments, 400 API consumers (internal and external), and a team of 8 engineers.

Write a structured recommendation (as you would present it to the CIO) that includes:

a) The criteria you would use to make this decision (at least 4 criteria)

b) Your recommendation with clear justification

c) The top risk of your recommendation and how you would mitigate it

d) What you would need to know additionally before finalising the recommendation

*(12 points: 3 for criteria quality, 4 for recommendation strength and justification, 3 for risk analysis, 2 for intellectual honesty about unknowns)*

---

**F2. Explaining a Technical Decision to a Non-Technical Stakeholder (13 points)**

You have decided to implement the SAGA pattern for the pension payment disbursement workflow. The workflow spans: `EligibilityService` → `PaymentService` → `BankTransferService` → `NotificationService`.

A senior IAS officer (Joint Secretary, Finance Department) who oversees the project asks you: *"Why can't we just use a normal database transaction like we always have? This SAGA thing sounds complicated and risky."*

Write your response as you would actually speak or write it to this officer. Your response must:

a) Acknowledge the validity of the concern

b) Explain why a traditional database transaction does not work here, using an analogy that a non-technical person can understand

c) Explain what SAGA does and the protection it provides, again using non-technical language

d) Be honest about the added complexity and what the team does to manage it

e) End with a clear recommendation

*(13 points: 2 for tone/acknowledgement, 4 for clarity of technical explanation, 3 for analogy quality, 2 for honesty about complexity, 2 for recommendation clarity)*

---

# SECTION G — Situational Judgement
## Decision-Making Under Pressure
### 20 Points | 4 Scenarios | 5 Points Each | Time Guide: 15 Minutes

**Instructions:** For each scenario, choose ONE action and write 2–4 sentences justifying your choice. There are no trick answers, but some choices are clearly better than others. You are evaluated on your reasoning, not just your selection.

---

**G1.** It is 11:45 PM. You are the on-call architect. The state pension portal is experiencing a 12% error rate. Preliminary investigation suggests a recent deployment (6 hours ago) changed the database connection pool configuration. 4,000 pensioners are actively trying to check their payment status. The deployment was approved and tested in staging. What do you do?

- **Option A:** Wait until morning when the full team is available to investigate the root cause properly before taking any action
- **Option B:** Immediately roll back the deployment to the previous stable version, restore service, then investigate root cause during business hours with full team
- **Option C:** Increase the connection pool size further via a hotfix deployment to resolve what appears to be a connection issue
- **Option D:** Restart all pods to clear any connection state and monitor for 30 minutes before deciding

Your choice: _______ | Justification:

---

**G2.** During a code review, you discover that a colleague (senior to you in tenure, not in role) has committed production database credentials directly into the application code in a private GitLab repository. The colleague is defensive when you raise it informally, saying "it's a private repo, only our team can see it." What do you do?

- **Option A:** Accept the explanation — private repos are safe enough for a government internal system
- **Option B:** Escalate immediately to the project manager and security team, document the finding, and request the credentials be rotated and moved to a secret manager — the seniority of the colleague is irrelevant to the security obligation
- **Option C:** Fix it yourself quietly without involving anyone to avoid conflict
- **Option D:** Raise it again with the colleague one more time, and if they still refuse, let it go

Your choice: _______ | Justification:

---

**G3.** Your team is 3 weeks from a mandated go-live date for a citizen portal. A load test reveals the system handles only 40% of the target peak load before response times exceed SLOs. Fixing this properly would require refactoring the database access layer — estimated 4 weeks of work. The project manager suggests going live anyway and fixing it post-launch. What is your recommendation?

- **Option A:** Agree to go live — the system works for 40% of peak load and most days won't hit peak
- **Option B:** Formally document the risk in writing, recommend a controlled go-live with artificial traffic throttling (rate limiting at the gateway) to cap users at the tested safe load, with a clear post-launch remediation plan with committed dates
- **Option C:** Escalate to stop the go-live entirely — no system should go live failing its NFRs
- **Option D:** Rush the refactoring in 3 weeks by the whole team working overtime

Your choice: _______ | Justification:

---

**G4.** You are three months into designing a complex government data exchange platform. A new cloud provider announces a fully managed service that would replace three of your custom-built components. Adopting it would require restructuring 40% of your current design. The service is 6 months old, has no government references, and is not yet on the approved vendor list. What do you do?

- **Option A:** Immediately pivot to adopt the new service — it eliminates custom code and reduces maintenance burden
- **Option B:** Document the new service as a future consideration, note it in your ADR as an alternative evaluated and deferred pending maturity, proceed with current design, and schedule a review in 12 months once the service has government-scale production references
- **Option C:** Ignore it — you are too far into the current design to consider changes
- **Option D:** Prototype the new service in parallel using project budget to evaluate before committing

Your choice: _______ | Justification:

---

---

*Assessment Document Version: 1.0*
*Senior Engineer → Solution Architect Program | June 2026*