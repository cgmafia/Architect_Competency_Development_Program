# Pre-Assessment Test
## Senior Engineer → Solution Architect Accelerated Program
### June 2026 Cohort | Batch: 20–25 Participants

---

**Document Type:** Candidate Assessment Paper
**Duration:** 90 Minutes (Strictly enforced — no extensions)
**Total Marks:** 150 Points
**Passing Threshold:** 90/150 (60%) — Minimum for program entry
**Recommended Readiness Score:** 120/150 (80%)
**Submission:** Via LMS portal before timer expiry. Late submissions are not evaluated.

---

## Instructions to Candidate

Read every question completely before writing your answer. For code questions, write real executable code in your chosen language — not pseudocode unless the question explicitly permits it. State your programming language at the top of every code answer.

For paragraph and essay questions, the quality of your reasoning matters significantly more than the length of your response. Vague, generic, or circular answers score zero regardless of length.

You may refer to official documentation — Spring Framework docs, Kubernetes documentation, MDN Web Docs, PostgreSQL documentation. You may not use AI tools, ask colleagues, or access community forums.

Time is a genuine constraint in this assessment. If you are stuck on a question, move forward and return to it only if time permits. A partially answered question scores more than a blank one.

There are seven sections. Read the time guide for each section and manage your 90 minutes accordingly.

---

## Assessment Structure

| Section   | Type                                       | Points  | Time Guide     |
| --------- | ------------------------------------------ | ------- | -------------- |
| A         | Multiple Choice — Core Technical Knowledge | 20      | 12 minutes     |
| B         | Conceptual Short Answer                    | 20      | 18 minutes     |
| C         | System Design Scenario                     | 35      | 25 minutes     |
| D         | Code Reading and Review                    | 20      | 15 minutes     |
| E         | Code Writing                               | 25      | 10 minutes     |
| F         | Trade-off and Research Essay               | 20      | 6 minutes      |
| G         | Situational Judgement                      | 10      | 4 minutes      |
| **Total** |                                            | **150** | **90 minutes** |

---

# Section A — Multiple Choice Questions

**Core Technical Knowledge | 20 Points | 10 Questions | 2 Points Each**

Select the single best answer for each question. No partial credit.

---

**A1.** A government payment portal processes 800,000 transactions per day. During the last three days of the financial year, traffic spikes to five times the daily average within a four-hour window. Which database consistency model is most appropriate for the transaction ledger that records debit and credit entries?

A. Eventual consistency — high availability is more important than ledger accuracy

B. Strong consistency — every debit must be immediately reflected in all reads

C. Causal consistency — transaction order matters but not cross-user reads

D. Read-your-writes consistency — sufficient because users only see their own transactions

---

**A2.** In a microservices system serving a national health portal, the AppointmentService calls DoctorProfileService synchronously. During a load test, DoctorProfileService begins responding in eight seconds. Which sequence of events most accurately describes what will happen to AppointmentService without resilience patterns in place?

A. AppointmentService will queue requests and process them when DoctorProfileService recovers

B. AppointmentService threads will block waiting for responses, exhausting its thread pool, causing AppointmentService itself to become unresponsive to all callers

C. AppointmentService will automatically retry failed requests with exponential backoff

D. The API gateway will detect the slowdown and route traffic to a backup instance

---

**A3.** In Kubernetes, what is the functional difference between a liveness probe and a readiness probe?

A. Liveness checks if the pod can receive traffic; readiness checks if the pod process is alive

B. Liveness failure causes the pod to be restarted; readiness failure removes the pod from the service endpoint list without restarting it

C. Both perform the same function but liveness is for HTTP services and readiness is for TCP services

D. Readiness failure causes the pod to be restarted; liveness failure removes the pod from rotation

---

**A4.** Which statement about the SAGA pattern is correct?

A. SAGA uses a two-phase commit coordinator to ensure atomic distributed transactions

B. SAGA breaks a distributed transaction into local transactions, each publishing events; failures are handled by compensating transactions

C. SAGA is only applicable to choreography-based architectures and cannot be used with orchestrators

D. SAGA guarantees strong consistency across all participating services

---

**A5.** You have a Terraform configuration managing 200 resources across a government cloud environment. A new engineer accidentally runs terraform apply with a misconfigured variable pointing to the wrong environment, destroying 15 production resources. Which Terraform feature, if properly implemented, would most directly have prevented this?

A. Terraform modules

B. Remote state with S3 backend

C. Workspace separation with environment-specific variable files and mandatory plan review gates in CI/CD before apply is permitted

D. Terraform Cloud subscription

---

**A6.** What is the primary purpose of an idempotency key in a payment API?

A. To encrypt the payment request payload end-to-end

B. To authenticate the identity of the calling service

C. To ensure that retrying the same request due to network failure does not result in duplicate processing

D. To rate-limit payment requests per registered user

---

**A7.** Which statement correctly describes the relationship between SLA, SLO, and SLI?

A. SLA is the internal target, SLO is the contractual commitment, SLI is the measurement

B. SLI is the measured value, SLO is the internal target which is stricter than the SLA, SLA is the contractual commitment with consequences for breach

C. SLO and SLA are the same thing; SLI is the tool used to measure them

D. SLI is the contractual commitment, SLO is the team's aspirational target, SLA is what Prometheus measures

---

**A8.** In Domain-Driven Design, an Aggregate Root serves which primary function?

A. It is the database primary key for a group of related tables

B. It is the single entry point through which all external interactions with the aggregate occur, enforcing business invariants and consistency boundaries

C. It is the parent class from which all domain entities inherit common fields

D. It represents the microservice boundary in a system using DDD

---

**A9.** A team is migrating a 15-year-old Oracle-based state treasury system to PostgreSQL microservices. The legacy system is live and processing transactions 24 hours a day, 7 days a week. Which migration approach minimises risk while ensuring zero downtime?

A. Big bang migration — export all data, run new system in parallel for one week, then cut over on a Sunday night

B. Strangler Fig with CDC — route new functionality to new services gradually while Debezium keeps new PostgreSQL in sync with legacy Oracle

C. Direct cut-over — run the new system in staging for six months, then replace legacy in one deployment

D. Fork the codebase — run both systems indefinitely and merge data monthly

---

**A10.** Which OWASP Top 10 vulnerability is demonstrated by this code?

```java
String query = "SELECT * FROM citizens WHERE name = '"
             + request.getParam("name") + "'";
ResultSet rs = statement.executeQuery(query);
```

A. Broken Access Control

B. Security Misconfiguration

C. Injection — specifically SQL Injection

D. Insecure Deserialization

---

# Section B — Conceptual Short Answer

**Clarity of Concepts | 20 Points | 4 Questions | 5 Points Each**

Answer each question in four to six sentences. Be precise. Bullet points are acceptable where they aid clarity. Vague or circular answers score zero.

---

**B1.** Explain the difference between horizontal scaling and vertical scaling. For a stateless microservice handling citizen login requests, which would you prefer and why? Identify the specific characteristic of a stateless service that makes one approach clearly superior.

---

**B2.** A product manager tells you the pension disbursement portal needs 99.999% uptime. As the architect, what is your immediate response, and what specific questions do you ask before agreeing to design for that target? Explain what makes this SLA particularly difficult to achieve in a government data centre context.

---

**B3.** Explain what eventual consistency means. Provide one government system example where eventual consistency is acceptable and one where it is not. Justify both choices with reference to the specific business consequence of a stale read in each case.

---

**B4.** What is the difference between authentication and authorisation? In a multi-ministry government portal where Ministry A staff must not access Ministry B citizen records, which mechanism primarily enforces this boundary, how does it do so technically, and what HTTP status code indicates a failure at each stage?

---

# Section C — System Design Scenario

**Design Thinking and Architectural Judgement | 35 Points | 25 Minutes**

Read the entire scenario before drawing or writing anything. You are evaluated on the quality of your thinking, the appropriateness of decisions, and your ability to articulate trade-offs — not on diagram aesthetics. Use Mermaid syntax for diagrams if working digitally. Hand-drawn diagrams are accepted if working on paper.

---

## The Scenario

The Government of Tamil Nadu has commissioned a Unified District Grievance Management System to replace 38 separate departmental complaint portals. The following requirements apply.

**Functional Scope**

Any resident may file a grievance against any of 38 departments including Water, Roads, Electricity, Revenue, Health, and Education. Departmental officers view, respond to, and resolve grievances assigned to their department. The system automatically escalates unresolved grievances after defined SLA periods — seven days unresolved escalates to the District Collector. The Chief Minister's Office receives a weekly report aggregating resolution statistics by district and department. Citizens receive SMS and email notifications at each status change.

**Scale**

Tamil Nadu has 38 districts and approximately 7.8 crore residents. Expected volume is 50,000 grievances per day during normal operation and up to 200,000 per day during post-disaster scenarios such as cyclone or flood. Eight thousand departmental officers across 38 departments access the system. The CMO dashboard must refresh every 15 minutes.

**Non-Functional Requirements**

Availability is 99.5% monthly. Citizen-facing APIs must respond at p95 under two seconds. Officer dashboards must respond at p95 under four seconds. Grievances must be retained for seven years to satisfy government audit requirements. Citizens authenticate via Tamil Nadu One ID SSO. Officers authenticate via Government PKI certificate. The system must function on 2G mobile connections for rural citizens.

**Constraints**

The system must be deployable on the NIC Tamil Nadu State Data Centre with the option to burst to AWS ap-south-1 Mumbai during peak load. It must integrate with the existing Tamil Nadu One ID for citizen authentication and send notifications via the BSNL SMS gateway under an existing government contract. The budget for third-party managed services is limited and open-source is preferred where operationally feasible.

---

**C1. Architecture Overview (15 points)**

Draw a high-level architecture for the Unified District Grievance Management System using a Mermaid diagram. Following the diagram, provide a written description that covers:

- The primary services and their single-sentence responsibilities
- Which service communications are synchronous and which are asynchronous, with a one-sentence justification for each choice
- Your database choice for each service with a one-sentence justification
- How the 2G mobile constraint specifically changes your API design decisions

---

**C2. Escalation Workflow Design (10 points)**

Design the automatic escalation mechanism by answering the following questions.

First, identify what technology or pattern you would use to implement the seven-day escalation timer and explain specifically why an in-memory timer is insufficient for this requirement.

Second, draw the escalation flow using a Mermaid sequence or state diagram showing at least three escalation levels and the actors involved at each level.

Third, state what data fields must be captured to support CMO reporting on escalation rates by district and department.

---

**C3. Identify Your Own Gaps (10 points)**

Review the architecture you have just designed. Identify three specific weaknesses or risks that you have not fully addressed. For each gap, state what it is, explain why it represents a genuine risk in this government context, and propose how you would address it if given more time.

Candidates who claim their design has no gaps will receive zero marks for this question. Every architecture has gaps.

---

# Section D — Code Reading and Review

**Code Comprehension and Critical Analysis | 20 Points | 15 Minutes**

Read each code snippet carefully before answering the questions below it.

---

## D1. Security Code Review (12 points)

Review the following Spring Boot REST controller for a government citizen data API.

```java
@RestController
@RequestMapping("/api/citizens")
public class CitizenController {

    @Autowired
    private CitizenRepository citizenRepository;

    @Autowired
    private AuditLogger auditLogger;

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

    @GetMapping("/search")
    public List<Citizen> searchCitizens(@RequestParam String query) {
        String sql = "SELECT * FROM citizens WHERE name LIKE '%"
                   + query + "%' OR mobile LIKE '%" + query + "%'";
        return citizenRepository.executeRawQuery(sql);
    }

    @PutMapping("/{id}/address")
    public ResponseEntity<String> updateAddress(
            @PathVariable Long id,
            @RequestBody AddressRequest address) {
        citizenRepository.updateAddress(id, address);
        return ResponseEntity.ok("Address updated successfully");
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<String> deleteCitizen(@PathVariable Long id) {
        citizenRepository.deleteById(id);
        return ResponseEntity.ok("Deleted");
    }
}
```

**D1a.** Identify all security vulnerabilities in this code. For each vulnerability, name it using OWASP terminology, identify the specific method where it occurs, and explain the potential impact in a government system handling citizen data. A minimum of four distinct vulnerabilities are present. (8 points)

**D1b.** The audit logger on the getCitizen endpoint writes the Aadhaar number to the log. Beyond the UIDAI Act violation, what specific operational security risk does this create in a system where logs are shipped to an ELK stack accessible to the operations and development teams? (2 points)

**D1c.** The updateAddress endpoint accepts a citizen ID in the path and an address in the request body. Without writing any code, list three specific things this endpoint is missing from a correctness and security standpoint. (2 points)

---

## D2. Distributed Systems Review (8 points)

Review this Kafka consumer implementation for a vaccination event processor.

```java
@Component
public class VaccinationEventConsumer {

    @Autowired
    private VaccinationRepository repository;

    @Autowired
    private CertificateService certificateService;

    @KafkaListener(topics = "vaccination.events",
                   groupId = "vacc-processor")
    public void processEvent(VaccinationEvent event) {
        try {
            repository.save(event);

            byte[] certificate = certificateService.generate(event);

            emailService.send(event.getEmail(), certificate);

            log.info("Processed vaccination for: "
                   + event.getBeneficiaryId()
                   + ", Aadhaar: " + event.getAadhaar());

        } catch (Exception e) {
            log.error("Failed to process event: " + e.getMessage());
        }
    }
}
```

**D2a.** This consumer will experience severe performance problems under the vaccination event volumes described in the course context (two million events per day). Identify the specific bottleneck and explain mechanically why it causes consumer group lag to grow. (2 points)

**D2b.** The catch block swallows all exceptions silently. Describe the two most serious consequences of this behaviour in a production vaccination tracking system serving citizens. (2 points)

**D2c.** There is no idempotency check. Describe a specific, realistic scenario in this system where the absence of idempotency causes a real-world problem for a citizen, including what the citizen experiences and what the data consequence is. (2 points)

**D2d.** Identify one additional problem in this code that has not been addressed by the questions above and explain its operational impact. (2 points)

---

# Section E — Code Writing

**Implementation Ability | 25 Points | 10 Minutes**

State your programming language at the top of each answer. Write real, runnable code. You are evaluated on correctness, production readiness, and handling of edge cases.

---

**E1. Idempotent REST Endpoint (15 points)**

Write a REST endpoint for processing a grievance submission that is idempotent. The language and framework are your choice.

The endpoint must accept a POST request with a JSON body containing citizenId, departmentCode, description, and locationCode. It must require an Idempotency-Key header and return HTTP 400 if this header is absent or blank. It must check whether the idempotency key has been seen before — use an in-memory ConcurrentHashMap if Redis is not available and clearly comment what Redis would replace. If the key exists, return the previously stored response without reprocessing, with a clear indicator in the response or header that this is a replay. If the key is new, save the grievance, store the result against the idempotency key, and return HTTP 201 with the new grievanceId.

Input validation must enforce that description is not blank and does not exceed 1000 characters, and that departmentCode is one of WATER, ROADS, HEALTH, REVENUE, or ELECTRICITY. Return appropriate HTTP status codes for all validation failures. Handle exceptions — do not let them propagate to the client without a structured error response.

---

**E2. SQL Query and Index Analysis (10 points)**

You are given the following query running against a grievance database. It takes 45 seconds on a table with 50 million rows.

```sql
SELECT
    d.department_name,
    COUNT(g.id)                                              AS total_grievances,
    SUM(CASE WHEN g.status = 'RESOLVED' THEN 1 ELSE 0 END) AS resolved_count,
    AVG(JULIANDAY(g.resolved_at)
        - JULIANDAY(g.created_at))                          AS avg_resolution_days
FROM   grievances g
JOIN   departments d ON g.department_id = d.id
WHERE  g.created_at  >= '2024-01-01'
  AND  g.created_at  <  '2025-01-01'
  AND  g.district_code = 'TN-CB'
GROUP  BY d.department_name
ORDER  BY total_grievances DESC;
```

**E2a.** Identify the most likely reason this query is slow on 50 million rows. (2 points)

**E2b.** Write the exact SQL CREATE INDEX statement or statements that would most improve this query's performance. Justify why you chose those specific columns in that specific order. (4 points)

**E2c.** The CMO dashboard runs this same query aggregated across all of Tamil Nadu — no district filter — every 15 minutes, and it must return in under two seconds. An index alone is insufficient. Propose and briefly explain an architectural solution that solves this requirement at the system design level, not just the database level. (4 points)

---

# Section F — Trade-off and Research Essay

**Communication, Research Depth, and Reasoning | 20 Points | 6 Minutes**

Write structured, professional responses as you would present to a technical panel or ministry stakeholder. Clarity and the quality of your reasoning are evaluated.

---

**F1. Explaining a Technical Decision to a Non-Technical Stakeholder (20 points)**

You have decided to implement the SAGA pattern for the pension payment disbursement workflow. The workflow spans EligibilityService, PaymentService, BankTransferService, and NotificationService.

A senior IAS officer who is Joint Secretary of the Finance Department and oversees this project asks you directly: "Why can't we just use a normal database transaction like we always have? This SAGA thing sounds complicated and risky. What if something goes wrong in the middle?"

Write your response exactly as you would deliver it in a meeting with this officer. Your response must acknowledge the validity of the concern genuinely, explain why a traditional database transaction does not work across four separate services using a non-technical analogy that a senior administrator would follow, explain what SAGA does and the protection it provides in plain language, be honest about the added complexity and what your team does to manage it, and end with a clear and confident recommendation.

Do not use jargon without immediately explaining it. Do not be condescending. Do not be vague to avoid the hard parts of the explanation.

---

# Section G — Situational Judgement

**Decision-Making Under Pressure | 10 Points | 2 Scenarios | 5 Points Each**

For each scenario, choose one action and write three to four sentences justifying your choice. You are evaluated on the quality of your reasoning, not just your selection.

---

**G1.** It is 11:45 PM. You are the on-call architect. The state pension portal is experiencing a 12% error rate. Preliminary investigation suggests a deployment six hours ago changed the database connection pool configuration. Four thousand pensioners are actively trying to check their payment status. The deployment was approved and tested in staging.

Choose one action:

A. Wait until morning when the full team is available to investigate the root cause properly before taking any action

B. Immediately roll back the deployment to the previous stable version, restore service, and investigate root cause during business hours with the full team

C. Increase the connection pool size further via a hotfix deployment to resolve what appears to be a connection issue

D. Restart all pods to clear connection state and monitor for 30 minutes before deciding

Your choice and justification:

---

**G2.** During a code review, you discover that a colleague who is senior to you in organisational tenure has committed production database credentials directly into the application code in a private GitLab repository. When you raise it informally, they respond that it is a private repository accessible only to your team and that it is safe enough. What do you do?

Choose one action:

A. Accept the explanation — private repositories are adequately safe for a government internal system

B. Escalate formally to the project manager and security team, document the finding, and request that the credentials be rotated and moved to a secret manager immediately — the seniority of the colleague is irrelevant to the security obligation

C. Fix it quietly yourself without involving anyone to avoid conflict with a senior colleague

D. Raise the concern with the colleague one more time and, if they still refuse, let the matter rest

Your choice and justification:

---

*Assessment Document Version 1.0*
*Senior Engineer to Solution Architect Accelerated Program — June 2026*