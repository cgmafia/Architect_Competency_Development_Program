# DOCUMENT 2 — LAB PRACTICAL CODE PROJECT

---

## LAB MANUAL: Day 3 — Distributed Systems & Event-Driven Architecture
**Phase:** Architectural Foundations & Design Thinking
**Topics:** Kafka Event Streaming, Event Sourcing, CQRS, Saga Pattern, Reactive Patterns, Workflow Engines
**Tech Stack:** Java 17 | Python 3.10+ | Apache Kafka | Linux Ubuntu 22.04 | Azure Free Tier
**IaC:** Terraform | Orchestration: Docker Compose | CI: GitHub Actions
**Geography Context:** 🇮🇳 India (CoWIN/UPI-inspired) / 🇸🇬 Singapore (PayNow/CPF-inspired)
**Copilot Usage:** Embedded prompts for code understanding throughout
**Pre-requisites:** Day 2 lab completed, Docker running, Java 17, Python 3.10+, 8GB RAM minimum for Kafka stack
**Estimated Lab Time:** 120 minutes | In-Class Demo: 45 minutes

---

# L0. LAB CONTEXT & ARCHITECTURE NARRATIVE

---

LAB NARRATIVE

SCENARIO:
You are building **RemitFlow** — a simplified cross-border remittance event platform inspired by the PayNow-UPI corridor. A Singapore resident initiates a remittance to an India UPI VPA. The system processes this through an event-driven pipeline: consent verification, FX rate lock, source debit, cross-border routing, beneficiary credit, and settlement notification. Every state transition emits an immutable domain event to Kafka. The system implements CQRS — the write side uses Event Sourcing to record every remittance state transition; the read side maintains three independent projections: a remittance status view, a real-time dashboard, and an audit trail projection.

The lab also implements a simplified Saga orchestrator using Python that coordinates the remittance steps with explicit compensating transactions — demonstrating how a Temporal-style workflow operates without the full Temporal infrastructure.

```
ARCHITECTURE OVERVIEW:

[Singapore Resident: "Send S$500 to UPI VPA anand@upi"]
        │
        │ POST /remittances (REST — driving adapter)
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  remittance-command-service  (Java 17, Port 8080)               │
│                                                                  │
│  WRITE SIDE — Event Sourced                                     │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  RemittanceAggregate (Aggregate Root)                   │   │
│  │  State derived from event stream                        │   │
│  │                                                         │   │
│  │  Events emitted:                                        │   │
│  │  RemittanceInitiated → FXRateLocked → DebitCompleted   │   │
│  │  → CrossBorderRoutingInitiated → CreditCompleted       │   │
│  │  → RemittanceCompleted                                 │   │
│  │  (OR: CompensatingEvents on failure)                   │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │ Emit to Kafka                         │
└─────────────────────────┼────────────────────────────────────── ┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  Apache Kafka (Docker)                                          │
│                                                                 │
│  Topic: remittance-events (partitioned by remittanceId)        │
│  Topic: remittance-commands (Saga orchestrator input)          │
│  Topic: remittance-dlq (Dead Letter Queue — failed events)     │
│  Topic: audit-events (immutable audit trail — 5yr retention)   │
└──────┬──────────────────┬──────────────────┬────────────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐  ┌─────────────────┐  ┌──────────────────┐
│  saga-      │  │  projection-    │  │  audit-          │
│  orchestrator│  │  service        │  │  service         │
│  (Python)   │  │  (Python)       │  │  (Python)        │
│  Port 8081  │  │  Port 8082      │  │  Port 8083       │
│             │  │                 │  │                  │
│  Coordinates│  │  READ SIDE:     │  │  Immutable       │
│  Saga steps │  │  3 Projections: │  │  audit trail     │
│  with       │  │  - Status View  │  │  per MAS TRM     │
│  compensating│  │  - Dashboard   │  │  5-year          │
│  transactions│  │  - Audit Feed  │  │  retention       │
└─────────────┘  └─────────────────┘  └──────────────────┘

CONCEPTS FROM TRAINING THIS LAB DEMONSTRATES:
□ Event Sourcing — RemittanceAggregate derives state from event history (Block 2)
□ CQRS — separate command service and projection service (Block 2, Use Case 1)
□ Kafka partitioning — remittanceId as partition key guarantees ordering (Block 2)
□ Saga Pattern — orchestration with compensating transactions (Block 3, Pattern 1)
□ Dead Letter Queue — failed events routed for manual inspection (Block 3)
□ CAP theorem in practice — command side CP, read side AP (Block 1, Concept 2)
□ At-least-once vs exactly-once — per-topic semantic choice demonstrated (Block 2)

PRODUCTION DELTA:
- Lab uses Kafka in Docker (single broker). Production: Kafka cluster with 3+ brokers,
  replication factor 3, rack-aware partition assignment across Azure availability zones.
- Lab Saga orchestrator is a simplified Python state machine. Production: Temporal.io
  workflow engine with durable execution, activity retries, and workflow versioning.
- Lab has no FX rate service — uses static mock rate. Production: real-time FX feed
  with rate lock expiry enforced as a Saga timeout.
- Lab event store is Kafka itself. Production: EventStoreDB or Kafka with
  compacted topics for aggregate snapshots.
- Azure Free Tier: Kafka runs locally in Docker. Azure Event Hubs (Kafka-compatible
  API) used for cloud deployment — free tier: 10 consumer groups, 1GB/day.
```

---

# L1. ENVIRONMENT SETUP

---

**STEP 1 of 9: Verify available memory**

WHY: Kafka + Zookeeper + 4 Python services + 1 Java service requires minimum 6GB RAM. Running on a machine with less will cause Kafka broker to OOM-kill during demo.

```bash
$ free -h
```

Expected Output:
```
              total        used        free
Mem:           15Gi        4.2Gi       9.8Gi
Swap:           2Gi        0.0Gi       2.0Gi
```

⚠️ IF total memory is below 8GB:
```bash
# Reduce Kafka heap size in docker-compose.yml:
# KAFKA_HEAP_OPTS: "-Xmx512m -Xms256m"
# This allows Kafka to run on 6GB total system RAM
```

✅ VERIFY:
```bash
$ docker info | grep "Total Memory"
# Expected: Total Memory: 15.xxxGiB (or your system total)
```

---

**STEP 2 of 9: Clone the lab repository**

```bash
$ git clone https://github.com/remitflow-lab/day3-event-driven.git
$ cd day3-event-driven
$ ls -la
```

Expected Output:
```
drwxr-xr-x  remittance-command-service/
drwxr-xr-x  saga-orchestrator/
drwxr-xr-x  projection-service/
drwxr-xr-x  audit-service/
drwxr-xr-x  infra/
drwxr-xr-x  scripts/
-rw-r--r--  docker-compose.yml
-rw-r--r--  Makefile
```

---

**STEP 3 of 9: Start Kafka infrastructure**

WHY: Kafka must be fully started before any application service attempts to connect. Kafka startup takes 20–30 seconds. Starting services before Kafka is ready causes connection refused errors that candidates incorrectly diagnose as application bugs.

```bash
$ docker compose up -d zookeeper kafka
$ sleep 30
$ docker compose logs kafka | tail -5
```

Expected Output:
```
kafka  | [2026-06-15 09:20:01,234] INFO [KafkaServer id=1] started
kafka  | [2026-06-15 09:20:01,891] INFO [Partition remittance-events-0 broker=1]
kafka  | [2026-06-15 09:20:01,892] INFO Kafka version: 3.6.0
```

⚠️ IF you see "Connection to node -1 could not be established":
```bash
$ docker compose restart kafka
$ sleep 20
$ docker compose logs kafka | grep "started"
```

✅ VERIFY:
```bash
$ docker compose exec kafka kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --list
# Expected: (empty — no topics yet, but command succeeds)
```

---

**STEP 4 of 9: Create Kafka topics**

WHY: Topics must be pre-created with correct partition counts and retention policies. Auto-topic creation in production is disabled — it bypasses partition key planning and retention governance.

```bash
$ chmod +x scripts/create-topics.sh
$ ./scripts/create-topics.sh
```

Expected Output:
```
Creating topic: remittance-events (partitions: 8, replication: 1)
Created topic remittance-events.
Creating topic: remittance-commands (partitions: 4, replication: 1)
Created topic remittance-commands.
Creating topic: remittance-dlq (partitions: 2, replication: 1)
Created topic remittance-dlq.
Creating topic: audit-events (partitions: 4, replication: 1, retention: -1)
Created topic audit-events.
All topics created successfully.
```

⚠️ IF you see "Topic already exists":
```bash
$ docker compose exec kafka kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --delete --topic remittance-events
# Wait 5 seconds, then re-run create-topics.sh
```

✅ VERIFY:
```bash
$ docker compose exec kafka kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --describe --topic remittance-events
```

Expected Output:
```
Topic: remittance-events  PartitionCount: 8  ReplicationFactor: 1
  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
  Partition: 1  Leader: 1  Replicas: 1  Isr: 1
  ...
```

---

**STEP 5 of 9: Build the Java command service**

```bash
$ cd remittance-command-service
$ ./mvnw clean package -DskipTests -q
```

Expected Output:
```
[INFO] BUILD SUCCESS
[INFO] Total time: 52.341 s
```

✅ VERIFY:
```bash
$ ls target/remittance-command-service-*.jar
# Expected: target/remittance-command-service-0.1.0.jar
```

---

**STEP 6 of 9: Install Python dependencies for all Python services**

```bash
$ cd ../saga-orchestrator
$ python3 -m venv venv && source venv/bin/activate
$ pip install -r requirements.txt -q
$ deactivate

$ cd ../projection-service
$ python3 -m venv venv && source venv/bin/activate
$ pip install -r requirements.txt -q
$ deactivate

$ cd ../audit-service
$ python3 -m venv venv && source venv/bin/activate
$ pip install -r requirements.txt -q
$ deactivate

$ cd ..
```

Expected Output (per service):
```
Successfully installed confluent-kafka-1.9.2 fastapi-0.110.0 
pydantic-2.6.0 uvicorn-0.27.0
```

✅ VERIFY:
```bash
$ cd saga-orchestrator && source venv/bin/activate
$ python3 -c "from confluent_kafka import Consumer; print('Kafka client OK')"
$ deactivate && cd ..
# Expected: Kafka client OK
```

---

**STEP 7 of 9: Start all application services**

```bash
$ cd ..
$ docker compose up --build -d
$ sleep 25
```

Expected Output:
```
remittance-command  | INFO  c.r.RemittanceCommandApp - Started in 5.234 seconds
saga-orchestrator   | INFO:     Application startup complete.
projection-service  | INFO:     Application startup complete.
audit-service       | INFO:     Application startup complete.
```

✅ VERIFY all services healthy:
```bash
$ curl -s http://localhost:8080/actuator/health | python3 -m json.tool
# Expected: {"status":"UP"}

$ curl -s http://localhost:8081/health | python3 -m json.tool
# Expected: {"status":"healthy","service":"saga-orchestrator"}

$ curl -s http://localhost:8082/health | python3 -m json.tool
# Expected: {"status":"healthy","service":"projection-service"}

$ curl -s http://localhost:8083/health | python3 -m json.tool
# Expected: {"status":"healthy","service":"audit-service"}
```

---

**STEP 8 of 9: Verify Kafka connectivity from all services**

```bash
$ docker compose exec kafka kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --list
```

Expected Output:
```
saga-orchestrator-group
projection-service-group
audit-service-group
```

---

**STEP 9 of 9: Run smoke test**

```bash
$ chmod +x scripts/smoke-test.sh
$ ./scripts/smoke-test.sh
```

Expected Output:
```
[SMOKE TEST] Creating test remittance...
[SMOKE TEST] Remittance ID: rem-a1b2c3d4
[SMOKE TEST] Checking event propagation...
[SMOKE TEST] ✅ remittance-events topic received 1 event
[SMOKE TEST] ✅ audit-events topic received 1 event
[SMOKE TEST] ✅ projection-service status endpoint updated
[SMOKE TEST] All smoke tests passed. Environment ready.
```

⚠️ IF smoke test fails with "No events received":
```bash
$ docker compose logs remittance-command | grep "ERROR"
# Check for Kafka connection refused — restart if needed
$ docker compose restart remittance-command
$ sleep 10 && ./scripts/smoke-test.sh
```

---

# L2. PROJECT STRUCTURE

---

```
day3-event-driven/
│
├── scripts/
│   ├── create-topics.sh              # Kafka topic creation with config
│   ├── smoke-test.sh                 # Environment verification
│   ├── produce-load.sh               # Simulate high-volume event load
│   ├── inject-failure.sh             # Chaos injection for demo
│   └── teardown.sh                   # Clean environment
│
├── remittance-command-service/        # WRITE SIDE — Java 17, Spring Boot 3
│   ├── src/main/java/com/remitflow/
│   │   ├── domain/
│   │   │   ├── RemittanceAggregate.java     # Event-sourced aggregate root
│   │   │   ├── RemittanceId.java            # Value object
│   │   │   ├── Money.java                   # Value object (amount + currency)
│   │   │   ├── RemittanceStatus.java        # Domain enum
│   │   │   └── events/
│   │   │       ├── RemittanceEvent.java     # Sealed interface — all events
│   │   │       ├── RemittanceInitiated.java # Record
│   │   │       ├── FXRateLocked.java        # Record
│   │   │       ├── DebitCompleted.java      # Record
│   │   │       ├── CreditCompleted.java     # Record
│   │   │       ├── RemittanceCompleted.java # Record
│   │   │       └── RemittanceFailed.java    # Record (compensation)
│   │   ├── application/
│   │   │   ├── port/
│   │   │   │   ├── in/
│   │   │   │   │   └── InitiateRemittanceUseCase.java
│   │   │   │   └── out/
│   │   │   │       ├── EventStorePort.java          # Append events
│   │   │   │       └── EventPublisherPort.java       # Publish to Kafka
│   │   │   └── RemittanceCommandService.java
│   │   ├── adapter/
│   │   │   ├── in/web/
│   │   │   │   ├── RemittanceController.java
│   │   │   │   └── RemittanceRequestDTO.java
│   │   │   └── out/
│   │   │       ├── kafka/KafkaEventPublisher.java    # Driven adapter
│   │   │       └── store/InMemoryEventStore.java     # Driven adapter
│   │   └── RemittanceCommandApp.java
│   ├── src/test/java/com/remitflow/
│   │   ├── domain/RemittanceAggregateTest.java
│   │   └── application/RemittanceCommandServiceTest.java
│   └── pom.xml
│
├── saga-orchestrator/                  # SAGA — Python 3.10
│   ├── src/
│   │   ├── domain/
│   │   │   ├── saga_state.py           # SagaState dataclass
│   │   │   └── compensation.py         # Compensating transaction definitions
│   │   ├── orchestrator/
│   │   │   ├── remittance_saga.py      # Saga state machine
│   │   │   └── saga_repository.py      # In-memory saga state store
│   │   ├── adapters/
│   │   │   ├── kafka_consumer.py       # Consumes remittance-events
│   │   │   └── kafka_producer.py       # Produces compensation commands
│   │   └── api/
│   │       └── router.py               # Saga status endpoint
│   └── main.py
│
├── projection-service/                 # READ SIDE — Python 3.10
│   ├── src/
│   │   ├── projections/
│   │   │   ├── status_projection.py    # Current remittance status
│   │   │   ├── dashboard_projection.py # Real-time aggregate stats
│   │   │   └── feed_projection.py      # Recent activity feed
│   │   ├── adapters/
│   │   │   └── kafka_consumer.py
│   │   └── api/
│   │       └── router.py
│   └── main.py
│
├── audit-service/                      # AUDIT TRAIL — Python 3.10
│   ├── src/
│   │   ├── audit/
│   │   │   └── audit_log.py            # Immutable append-only audit log
│   │   ├── adapters/
│   │   │   └── kafka_consumer.py
│   │   └── api/
│   │       └── router.py
│   └── main.py
│
├── infra/
│   └── terraform/
│       ├── main.tf                     # Azure Event Hubs (Kafka-compatible)
│       ├── variables.tf
│       └── outputs.tf
│
└── docker-compose.yml
```

---

# L3. CODE BLOCKS

---

## KAFKA TOPIC CREATION SCRIPT

```
FILE: scripts/create-topics.sh
LANGUAGE: Bash
PURPOSE: Creates all Kafka topics with production-appropriate configuration.
         Partition count, retention, and replication factor are architecture decisions.
CONCEPTS DEMONSTRATED: Kafka topic design (Block 2), retention policy for audit (Use Case 2)
COPY-PASTE READY: YES
PRODUCTION DELTA: Replication factor = 3 in production. Add --config min.insync.replicas=2
                  to ensure durability guarantees. Add rack-aware partition assignment.
```

```bash
#!/usr/bin/env bash
# scripts/create-topics.sh
#
# ARCHITECTURAL NOTE ON PARTITION COUNT:
# remittance-events: 8 partitions
# WHY 8: At demo scale, 8 consumers can process in parallel.
# Production sizing: (peak TPS × average processing time ms) / 1000
# Example: 500 TPS × 200ms / 1000 = 100 partitions minimum.
# Kafka partitions cannot easily be reduced after creation.
# Always provision more than current need. Over-provision by 2×.
#
# PARTITION KEY DECISION (documented here — not just in ADR):
# remittanceId as partition key → all events for one remittance
# are guaranteed to arrive at the same partition in order.
# If we used customerId as key → all of one customer's remittances
# go to same partition. Good for customer-level ordering.
# BAD for our case: a customer with 100 remittances creates
# ordering dependency across all of them unnecessarily.
# remittanceId is the correct granularity.

set -e

KAFKA_BOOTSTRAP="localhost:9092"
KAFKA_EXEC="docker compose exec kafka"

echo "Creating Kafka topics for RemitFlow..."

# Topic 1: remittance-events
# All state-changing events for remittances
# Partition key: remittanceId (guarantees per-remittance ordering)
# Retention: 7 days (operational window for replay/debugging)
# Exactly-once: YES — financial events must not be processed twice
$KAFKA_EXEC kafka-topics.sh \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --create \
  --topic remittance-events \
  --partitions 8 \
  --replication-factor 1 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=1 \
  --if-not-exists
echo "Created topic: remittance-events"

# Topic 2: remittance-commands
# Saga orchestrator command channel
# Partition key: sagaId
# Retention: 24 hours (commands processed and discarded)
$KAFKA_EXEC kafka-topics.sh \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --create \
  --topic remittance-commands \
  --partitions 4 \
  --replication-factor 1 \
  --config retention.ms=86400000 \
  --if-not-exists
echo "Created topic: remittance-commands"

# Topic 3: remittance-dlq (Dead Letter Queue)
# Events that failed processing after max retries
# Partition key: original-topic + partition + offset
# Retention: 30 days (manual investigation window)
# OPERATIONAL NOTE: Monitor DLQ size as a health metric.
# A growing DLQ is a silent production incident.
$KAFKA_EXEC kafka-topics.sh \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --create \
  --topic remittance-dlq \
  --partitions 2 \
  --replication-factor 1 \
  --config retention.ms=2592000000 \
  --if-not-exists
echo "Created topic: remittance-dlq"

# Topic 4: audit-events
# Immutable audit trail — MAS TRM requirement
# Retention: -1 = FOREVER (never delete)
# Partition key: remittanceId
# COMPLIANCE NOTE: MAS TRM 2021 Section 9.4 requires audit logs
# to be retained for minimum 5 years. retention.ms=-1 satisfies this.
# In production: also configure log.cleanup.policy=compact
# so that the latest state per key is always retained even
# after the time-based log is compacted.
$KAFKA_EXEC kafka-topics.sh \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --create \
  --topic audit-events \
  --partitions 4 \
  --replication-factor 1 \
  --config retention.ms=-1 \
  --config cleanup.policy=compact \
  --if-not-exists
echo "Created topic: audit-events (retention: PERMANENT)"

echo ""
echo "All topics created successfully."
echo ""
echo "Topic summary:"
$KAFKA_EXEC kafka-topics.sh \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --list
```

---

## REMITTANCE DOMAIN — Event-Sourced Aggregate (Java 17)

```
FILE: remittance-command-service/src/main/java/com/remitflow/domain/events/RemittanceEvent.java
LANGUAGE: Java 17
PURPOSE: Sealed interface defining all possible events for RemittanceAggregate.
         Sealed + records = compile-time exhaustive event handling.
CONCEPTS DEMONSTRATED: Event Sourcing event hierarchy, Java 17 sealed classes + records
COPY-PASTE READY: YES
PRODUCTION DELTA: Add event schema version field for schema evolution.
                  Add cryptographic hash of previous event for tamper detection.
```

```java
// FILE: com/remitflow/domain/events/RemittanceEvent.java
//
// ARCHITECTURAL NOTE:
// Sealed interface + records is the ideal Java 17 pattern for domain events.
//
// WHY SEALED: The compiler knows ALL possible event types at compile time.
// When you add a new event type, every switch expression in the codebase
// that handles RemittanceEvent must be updated — or it fails to compile.
// This is a FEATURE: it forces you to handle every event type in every
// event handler. No silent "default" case that ignores new events.
//
// WHY RECORDS: Events are immutable value objects. Records give you
// immutability, equals/hashCode, toString, and canonical constructor
// for free. A domain event should never be mutated after creation.
// If you need to "update" an event — you emit a new event.

package com.remitflow.domain.events;

import java.math.BigDecimal;
import java.time.Instant;

/**
 * SEALED INTERFACE: RemittanceEvent
 *
 * Every state transition in the RemittanceAggregate lifecycle
 * is represented by exactly one of these record types.
 * The event log IS the source of truth.
 * The aggregate state is always derived by replaying these events.
 *
 * SCHEMA VERSIONING NOTE (for production):
 * When you need to add a field to an existing event type:
 * 1. Add the field with a default value (backward compatible — MINOR version)
 * 2. If removing or changing a field — create a new event type (MAJOR version)
 * 3. The event store must support reading both old and new schema versions
 * This is the hardest part of Event Sourcing in production.
 */
public sealed interface RemittanceEvent
        permits RemittanceEvent.RemittanceInitiated,
                RemittanceEvent.FXRateLocked,
                RemittanceEvent.DebitCompleted,
                RemittanceEvent.CrossBorderRoutingInitiated,
                RemittanceEvent.CreditCompleted,
                RemittanceEvent.RemittanceCompleted,
                RemittanceEvent.RemittanceFailed,
                RemittanceEvent.DebitReversed,
                RemittanceEvent.FXRateReleased {

    // Every event carries these fields — the envelope
    String remittanceId();
    Instant occurredAt();
    int schemaVersion(); // For future schema evolution — always 1 for new events

    // ─────────────────────────────────────────────────────────────────
    // HAPPY PATH EVENTS
    // ─────────────────────────────────────────────────────────────────

    /**
     * A new remittance has been initiated by a Singapore resident.
     * This is the FIRST event — replaying from this event initialises
     * the aggregate's state.
     */
    record RemittanceInitiated(
            String remittanceId,
            String customerId,          // Singapore SingPass pseudonymous ID
            String sourceCurrency,      // SGD
            BigDecimal sourceAmount,    // e.g., 500.00
            String destinationCurrency, // INR
            String beneficiaryVpa,      // e.g., anand@upi
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    /**
     * FX rate has been locked for this remittance.
     * The rate is immutably recorded here.
     * If the rate changes before processing — this event is the proof
     * of what rate was agreed at initiation.
     * SAGA NOTE: FX rate lock has a 60-second TTL.
     * If DebitCompleted does not follow within 60s —
     * FXRateReleased compensating event must be emitted.
     */
    record FXRateLocked(
            String remittanceId,
            BigDecimal fxRate,          // e.g., 62.45 (1 SGD = 62.45 INR)
            BigDecimal destinationAmount, // sourceAmount × fxRate
            Instant lockExpiresAt,      // occurredAt + 60 seconds
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    /**
     * Source account (Singapore) has been debited.
     * After this event: money has left the sender's account.
     * A compensating event (DebitReversed) MUST be emitted
     * if any subsequent step fails.
     * The bank transaction reference is recorded for reconciliation.
     */
    record DebitCompleted(
            String remittanceId,
            String bankTransactionRef,  // Singapore bank's reference
            BigDecimal amountDebited,
            String currency,
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    record CrossBorderRoutingInitiated(
            String remittanceId,
            String routingReference,    // MAS-approved FX channel reference
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    record CreditCompleted(
            String remittanceId,
            String upiTransactionId,    // India UPI transaction reference
            BigDecimal amountCredited,
            String currency,            // INR
            String beneficiaryVpa,
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    record RemittanceCompleted(
            String remittanceId,
            BigDecimal sourceAmount,
            String sourceCurrency,
            BigDecimal destinationAmount,
            String destinationCurrency,
            BigDecimal fxRateUsed,
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    // ─────────────────────────────────────────────────────────────────
    // COMPENSATING EVENTS (Saga failure handling)
    // ─────────────────────────────────────────────────────────────────

    /**
     * The remittance has failed and cannot be completed.
     * Contains the step at which failure occurred and the reason.
     * The failedAtStep field tells the Saga orchestrator
     * which compensating transactions to execute.
     */
    record RemittanceFailed(
            String remittanceId,
            String failedAtStep,        // e.g., "CROSS_BORDER_ROUTING"
            String reason,
            boolean debitOccurred,      // true = DebitReversed must follow
            boolean fxLockOccurred,     // true = FXRateReleased must follow
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    /**
     * COMPENSATING EVENT: Reverses the source account debit.
     * Emitted when the remittance fails AFTER DebitCompleted.
     * This is money returning to the sender's account.
     * CRITICAL: This event must be exactly-once processed.
     */
    record DebitReversed(
            String remittanceId,
            String originalBankTransactionRef,
            String reversalTransactionRef,
            BigDecimal amountReversed,
            String currency,
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}

    /**
     * COMPENSATING EVENT: Releases the FX rate lock.
     * Emitted when the remittance fails before debit,
     * or when the FX lock expires (TTL exceeded).
     */
    record FXRateReleased(
            String remittanceId,
            String reason,              // "FAILURE_BEFORE_DEBIT" | "LOCK_EXPIRED"
            Instant occurredAt,
            int schemaVersion
    ) implements RemittanceEvent {}
}
```

---

## REMITTANCE AGGREGATE — Event Sourced (Java 17)

```
FILE: remittance-command-service/src/main/java/com/remitflow/domain/RemittanceAggregate.java
LANGUAGE: Java 17
PURPOSE: Event-sourced aggregate root. State is ALWAYS derived from event history.
         Never mutate state directly — always emit an event, then apply it.
CONCEPTS DEMONSTRATED: Event Sourcing apply() pattern, aggregate reconstitution,
                       domain invariant enforcement, Java 17 pattern matching
COPY-PASTE READY: YES
PRODUCTION DELTA: Add optimistic concurrency control (expectedVersion parameter
                  on save — prevents lost update in concurrent scenarios).
                  Add snapshot support for aggregates with 1000+ events.
```

```java
// FILE: com/remitflow/domain/RemittanceAggregate.java
//
// ARCHITECTURAL NOTE ON EVENT SOURCING APPLY PATTERN:
//
// Traditional (state-based) aggregate:
//   void completeDebit(Amount amount) {
//     this.status = DEBIT_COMPLETED;    // ← Direct state mutation
//     this.debitedAmount = amount;
//   }
//
// Event-sourced aggregate:
//   void completeDebit(Amount amount) {
//     var event = new DebitCompleted(...);  // ← Create event
//     apply(event);                          // ← Apply to local state
//     domainEvents.add(event);              // ← Record for publishing
//   }
//
//   private void apply(DebitCompleted e) {
//     this.status = DEBIT_COMPLETED;    // ← State mutation ONLY here
//     this.debitedAmount = e.amountDebited();
//   }
//
// The apply() method is also called during RECONSTITUTION:
//   static RemittanceAggregate reconstitute(List<RemittanceEvent> history) {
//     var aggregate = new RemittanceAggregate();
//     history.forEach(aggregate::applyEvent);  // ← Replay all events
//     return aggregate;                         // ← State is identical to live
//   }
//
// This is the KEY INSIGHT: the same apply() method that handles
// new events during runtime handles replayed events during reconstitution.
// One code path. No divergence. Consistent state always.

package com.remitflow.domain;

import com.remitflow.domain.events.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

public final class RemittanceAggregate {

    // Current state — always derived from event history
    private String remittanceId;
    private String customerId;
    private BigDecimal sourceAmount;
    private String sourceCurrency;
    private BigDecimal destinationAmount;
    private String destinationCurrency;
    private String beneficiaryVpa;
    private BigDecimal fxRate;
    private Instant fxLockExpiresAt;
    private RemittanceStatus status;
    private String failureReason;
    private boolean debitOccurred;

    // Event version — for optimistic concurrency control
    // In production: check expectedVersion before saving
    private long version = 0;

    // Pending events — applied to state, not yet published to Kafka
    private final List<RemittanceEvent> pendingEvents = new ArrayList<>();

    // ─── FACTORY: Create new remittance ───────────────────────────────

    public static RemittanceAggregate initiate(
            String remittanceId,
            String customerId,
            String sourceCurrency,
            BigDecimal sourceAmount,
            String destinationCurrency,
            String beneficiaryVpa
    ) {
        if (sourceAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new RemittanceInvariantException(
                "Source amount must be positive, got: " + sourceAmount
            );
        }
        if (beneficiaryVpa == null || !beneficiaryVpa.contains("@")) {
            throw new RemittanceInvariantException(
                "Invalid UPI VPA format: " + beneficiaryVpa
            );
        }

        var aggregate = new RemittanceAggregate();
        var event = new RemittanceEvent.RemittanceInitiated(
                remittanceId, customerId, sourceCurrency, sourceAmount,
                destinationCurrency, beneficiaryVpa, Instant.now(), 1
        );
        aggregate.applyAndRecord(event);
        return aggregate;
    }

    // ─── FACTORY: Reconstitute from event history ─────────────────────

    /**
     * Rebuilds aggregate state by replaying all historical events.
     * Used when loading from the event store.
     * The result is IDENTICAL to the live aggregate that processed
     * those events originally — this is the Event Sourcing guarantee.
     */
    public static RemittanceAggregate reconstitute(
            List<RemittanceEvent> history
    ) {
        if (history.isEmpty()) {
            throw new RemittanceInvariantException(
                "Cannot reconstitute aggregate from empty event history"
            );
        }
        var aggregate = new RemittanceAggregate();
        history.forEach(aggregate::applyEvent); // Replay — no recording
        return aggregate;
    }

    // ─── COMMANDS ─────────────────────────────────────────────────────

    public void lockFXRate(BigDecimal fxRate, BigDecimal destinationAmount) {
        requireStatus(RemittanceStatus.INITIATED);

        var event = new RemittanceEvent.FXRateLocked(
                remittanceId, fxRate, destinationAmount,
                Instant.now().plusSeconds(60), // 60-second FX lock TTL
                Instant.now(), 1
        );
        applyAndRecord(event);
    }

    public void completeDebit(String bankTransactionRef) {
        requireStatus(RemittanceStatus.FX_LOCKED);
        checkFXLockNotExpired();

        var event = new RemittanceEvent.DebitCompleted(
                remittanceId, bankTransactionRef,
                sourceAmount, sourceCurrency, Instant.now(), 1
        );
        applyAndRecord(event);
    }

    public void initiateRouting(String routingReference) {
        requireStatus(RemittanceStatus.DEBITED);

        var event = new RemittanceEvent.CrossBorderRoutingInitiated(
                remittanceId, routingReference, Instant.now(), 1
        );
        applyAndRecord(event);
    }

    public void completeCredit(
            String upiTransactionId,
            BigDecimal amountCredited
    ) {
        requireStatus(RemittanceStatus.ROUTING_INITIATED);

        var event = new RemittanceEvent.CreditCompleted(
                remittanceId, upiTransactionId, amountCredited,
                destinationCurrency, beneficiaryVpa, Instant.now(), 1
        );
        applyAndRecord(event);
    }

    public void complete() {
        requireStatus(RemittanceStatus.CREDITED);

        var event = new RemittanceEvent.RemittanceCompleted(
                remittanceId, sourceAmount, sourceCurrency,
                destinationAmount, destinationCurrency,
                fxRate, Instant.now(), 1
        );
        applyAndRecord(event);
    }

    /**
     * COMMAND: Fail this remittance.
     * Automatically determines which compensating events are needed
     * based on how far the saga has progressed.
     * The Saga orchestrator calls this when any step fails.
     */
    public void fail(String failedAtStep, String reason) {
        if (status == RemittanceStatus.COMPLETED ||
            status == RemittanceStatus.FAILED) {
            throw new RemittanceInvariantException(
                "Cannot fail remittance in terminal status: " + status
            );
        }

        var event = new RemittanceEvent.RemittanceFailed(
                remittanceId, failedAtStep, reason,
                debitOccurred,          // Tells Saga: reverse debit needed?
                fxLockExpiresAt != null, // Tells Saga: release FX lock needed?
                Instant.now(), 1
        );
        applyAndRecord(event);
    }

    public void reverseDebit(String reversalRef) {
        requireStatus(RemittanceStatus.FAILED);

        var event = new RemittanceEvent.DebitReversed(
                remittanceId, "original-ref", reversalRef,
                sourceAmount, sourceCurrency, Instant.now(), 1
        );
        applyAndRecord(event);
    }

    // ─── EVENT APPLICATION (apply pattern) ───────────────────────────

    /**
     * Applies an event to aggregate state AND records it as pending.
     * Called for NEW events during command processing.
     */
    private void applyAndRecord(RemittanceEvent event) {
        applyEvent(event);
        pendingEvents.add(event);
    }

    /**
     * Applies an event to aggregate state WITHOUT recording.
     * Called during RECONSTITUTION (replaying historical events).
     * Java 17 pattern matching switch — exhaustive by sealed interface.
     * Adding a new event type to the sealed interface forces
     * this switch to be updated — compile error prevents omission.
     */
    private void applyEvent(RemittanceEvent event) {
        version++;
        switch (event) {
            case RemittanceEvent.RemittanceInitiated e -> {
                this.remittanceId = e.remittanceId();
                this.customerId = e.customerId();
                this.sourceAmount = e.sourceAmount();
                this.sourceCurrency = e.sourceCurrency();
                this.destinationCurrency = e.destinationCurrency();
                this.beneficiaryVpa = e.beneficiaryVpa();
                this.status = RemittanceStatus.INITIATED;
                this.debitOccurred = false;
            }
            case RemittanceEvent.FXRateLocked e -> {
                this.fxRate = e.fxRate();
                this.destinationAmount = e.destinationAmount();
                this.fxLockExpiresAt = e.lockExpiresAt();
                this.status = RemittanceStatus.FX_LOCKED;
            }
            case RemittanceEvent.DebitCompleted e -> {
                this.status = RemittanceStatus.DEBITED;
                this.debitOccurred = true;
            }
            case RemittanceEvent.CrossBorderRoutingInitiated e -> {
                this.status = RemittanceStatus.ROUTING_INITIATED;
            }
            case RemittanceEvent.CreditCompleted e -> {
                this.status = RemittanceStatus.CREDITED;
            }
            case RemittanceEvent.RemittanceCompleted e -> {
                this.status = RemittanceStatus.COMPLETED;
            }
            case RemittanceEvent.RemittanceFailed e -> {
                this.status = RemittanceStatus.FAILED;
                this.failureReason = e.reason();
            }
            case RemittanceEvent.DebitReversed e -> {
                this.status = RemittanceStatus.DEBIT_REVERSED;
            }
            case RemittanceEvent.FXRateReleased e -> {
                this.fxLockExpiresAt = null;
            }
        }
    }

    // ─── INVARIANT HELPERS ────────────────────────────────────────────

    private void requireStatus(RemittanceStatus required) {
        if (this.status != required) {
            throw new RemittanceInvariantException(
                "Expected status " + required + " but was " + this.status +
                " for remittance " + remittanceId
            );
        }
    }

    private void checkFXLockNotExpired() {
        if (fxLockExpiresAt != null &&
            Instant.now().isAfter(fxLockExpiresAt)) {
            throw new RemittanceInvariantException(
                "FX rate lock has expired for remittance " + remittanceId +
                ". Lock expired at: " + fxLockExpiresAt
            );
        }
    }

    // ─── READ ACCESSORS ───────────────────────────────────────────────

    public List<RemittanceEvent> popPendingEvents() {
        var events = List.copyOf(pendingEvents);
        pendingEvents.clear();
        return events;
    }

    public String remittanceId() { return remittanceId; }
    public RemittanceStatus status() { return status; }
    public long version() { return version; }
    public boolean debitOccurred() { return debitOccurred; }
    public BigDecimal sourceAmount() { return sourceAmount; }
    public String sourceCurrency() { return sourceCurrency; }
    public BigDecimal destinationAmount() { return destinationAmount; }
    public String destinationCurrency() { return destinationCurrency; }
}
```

---

## KAFKA EVENT PUBLISHER — Driven Adapter (Java 17)

```
FILE: remittance-command-service/src/main/java/com/remitflow/adapter/out/kafka/KafkaEventPublisher.java
LANGUAGE: Java 17
PURPOSE: Driven adapter — publishes domain events to Kafka.
         Implements exactly-once semantics for financial events.
CONCEPTS DEMONSTRATED: Kafka producer with idempotent writes, exactly-once semantics,
                       partition key assignment, structured event envelope
COPY-PASTE READY: YES
PRODUCTION DELTA: Add Kafka Streams Schema Registry for Avro schema validation.
                  Add OpenTelemetry trace context propagation in Kafka headers.
```

```java
// FILE: com/remitflow/adapter/out/kafka/KafkaEventPublisher.java
//
// ARCHITECTURAL NOTE ON EXACTLY-ONCE SEMANTICS:
//
// The default Kafka producer behaviour is AT-LEAST-ONCE:
// Message sent → Ack received → Commit
// If crash happens between send and ack → retry → duplicate
//
// For financial events (DebitCompleted, CreditCompleted):
// A duplicate event means: "this account was debited twice"
// That is a catastrophic failure.
//
// EXACTLY-ONCE requires:
// 1. enable.idempotence=true (prevents duplicates within one session)
// 2. transactional.id set (enables transactions across multiple sends)
// 3. Producer wraps sends in beginTransaction/commitTransaction
//
// COST: ~20% throughput reduction vs. at-least-once
// WORTH IT: For DebitCompleted, CreditCompleted — absolutely yes.
// For informational events (DashboardUpdated) — use at-least-once.
//
// This is why we have multiple topics with different semantic guarantees.

package com.remitflow.adapter.out.kafka;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.remitflow.application.port.out.EventPublisherPort;
import com.remitflow.domain.events.RemittanceEvent;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.util.Properties;
import java.util.concurrent.ExecutionException;

@Component
public class KafkaEventPublisher implements EventPublisherPort {

    private static final String REMITTANCE_EVENTS_TOPIC = "remittance-events";
    private static final String AUDIT_EVENTS_TOPIC = "audit-events";

    private KafkaProducer<String, String> producer;
    private final ObjectMapper objectMapper;

    public KafkaEventPublisher() {
        this.objectMapper = new ObjectMapper()
                .registerModule(new JavaTimeModule());
    }

    @PostConstruct
    void initialise() {
        var props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                StringSerializer.class.getName());

        // EXACTLY-ONCE CONFIGURATION
        // enable.idempotence: Kafka broker deduplicates messages
        // from this producer using sequence numbers.
        // Prevents duplicates caused by producer retry after network failure.
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");

        // transactional.id: Unique per producer instance.
        // Enables atomic writes across multiple topics/partitions.
        // If we publish to remittance-events AND audit-events in one
        // transaction — either both commit or neither does.
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
                "remittance-command-service-producer-1");

        // acks=all: Producer waits for ALL in-sync replicas to acknowledge.
        // Prevents message loss if leader broker fails immediately after write.
        // Required for exactly-once and for financial event durability.
        props.put(ProducerConfig.ACKS_CONFIG, "all");

        // Throughput tuning — balance latency vs. throughput
        props.put(ProducerConfig.LINGER_MS_CONFIG, "5");
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, "16384");

        this.producer = new KafkaProducer<>(props);
        this.producer.initTransactions();
    }

    @Override
    public void publish(RemittanceEvent event) {
        // Wrap in Kafka transaction: remittance-events AND audit-events
        // both receive the event atomically.
        // Either both are written or neither is — no partial state.
        producer.beginTransaction();
        try {
            var envelope = buildEventEnvelope(event);
            var json = objectMapper.writeValueAsString(envelope);

            // Partition key = remittanceId
            // All events for one remittance go to the same partition.
            // This guarantees ordering of events for a single remittance.
            var remittanceKey = event.remittanceId();

            // Send to remittance-events topic
            var remittanceRecord = new ProducerRecord<>(
                    REMITTANCE_EVENTS_TOPIC,
                    remittanceKey,
                    json
            );
            producer.send(remittanceRecord).get(); // Synchronous for exactly-once

            // Send same event to audit-events topic
            // audit-events has permanent retention — MAS TRM compliance
            var auditRecord = new ProducerRecord<>(
                    AUDIT_EVENTS_TOPIC,
                    remittanceKey,
                    json
            );
            producer.send(auditRecord).get();

            producer.commitTransaction();

        } catch (Exception e) {
            // Abort transaction — neither topic receives the event
            producer.abortTransaction();
            throw new EventPublishingException(
                "Failed to publish event for remittance: " +
                event.remittanceId(), e
            );
        }
    }

    private EventEnvelope buildEventEnvelope(RemittanceEvent event) {
        // Java 17 pattern matching to extract event type name cleanly
        // switch expression returns the type name string
        var eventType = switch (event) {
            case RemittanceEvent.RemittanceInitiated e  -> "RemittanceInitiated";
            case RemittanceEvent.FXRateLocked e         -> "FXRateLocked";
            case RemittanceEvent.DebitCompleted e       -> "DebitCompleted";
            case RemittanceEvent.CrossBorderRoutingInitiated e
                                                        -> "CrossBorderRoutingInitiated";
            case RemittanceEvent.CreditCompleted e      -> "CreditCompleted";
            case RemittanceEvent.RemittanceCompleted e  -> "RemittanceCompleted";
            case RemittanceEvent.RemittanceFailed e     -> "RemittanceFailed";
            case RemittanceEvent.DebitReversed e        -> "DebitReversed";
            case RemittanceEvent.FXRateReleased e       -> "FXRateReleased";
        };

        return new EventEnvelope(
                event.remittanceId(),
                eventType,
                event.schemaVersion(),
                event.occurredAt().toString(),
                "remittance-command-service",
                event
        );
    }

    @PreDestroy
    void shutdown() {
        if (producer != null) {
            producer.close();
        }
    }

    // Envelope wraps every domain event with routing metadata
    // Consumers use eventType to deserialise the payload correctly
    record EventEnvelope(
            String aggregateId,
            String eventType,
            int schemaVersion,
            String occurredAt,
            String source,
            Object payload
    ) {}
}
```

---

## SAGA ORCHESTRATOR (Python 3.10)

```
FILE: saga-orchestrator/src/orchestrator/remittance_saga.py
LANGUAGE: Python 3.10
PURPOSE: Saga state machine that coordinates remittance steps.
         Consumes events from Kafka, determines next step or compensation.
         Simplified Temporal-style workflow without the full Temporal infrastructure.
CONCEPTS DEMONSTRATED: Saga orchestration pattern, compensating transactions,
                       state machine with explicit transition rules
COPY-PASTE READY: YES
PRODUCTION DELTA: Replace this state machine with Temporal.io workflow.
                  Temporal provides: durable execution (survives process restarts),
                  activity retries with exponential backoff, workflow versioning,
                  timeout enforcement as first-class citizens.
```

```python
# saga-orchestrator/src/orchestrator/remittance_saga.py
#
# ARCHITECTURAL NOTE:
# This file implements a simplified Saga orchestrator.
# It is a state machine that:
# 1. Receives domain events from Kafka (remittance-events topic)
# 2. Determines the next step in the saga based on current state
# 3. Sends commands to participant services
# 4. On failure: determines and executes compensating transactions
#
# In production: this logic lives inside a Temporal Workflow definition.
# Temporal makes this state machine DURABLE — if the orchestrator process
# crashes mid-saga, Temporal replays the workflow history and resumes
# exactly where it left off. This lab's in-memory state machine loses
# all in-flight sagas on process restart — the primary production delta.
#
# WHY ORCHESTRATION over CHOREOGRAPHY for this use case:
# The remittance saga has 6 steps with strict ordering requirements.
# Failure at step 4 (cross-border routing) requires reversing step 3
# (debit) and releasing step 2 (FX lock). In choreography, this
# compensation logic is distributed across 3 different services.
# Debugging a stuck compensation chain requires correlating events
# across 3 services' logs. In orchestration: the orchestrator knows
# the full saga state — debugging is one service's state machine.

from __future__ import annotations

import json
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum, auto
from typing import Callable


class SagaStep(Enum):
    """
    Ordered steps in the remittance saga.
    The saga advances through these steps in sequence.
    Failure at any step triggers compensation from that step backwards.
    """
    INITIATED = auto()
    FX_RATE_LOCKING = auto()
    FX_RATE_LOCKED = auto()
    DEBITING = auto()
    DEBITED = auto()
    ROUTING = auto()
    CREDITED = auto()
    COMPLETED = auto()
    # Compensation states
    COMPENSATING = auto()
    COMPENSATION_COMPLETED = auto()
    FAILED = auto()


class SagaFailureReason(Enum):
    FX_RATE_UNAVAILABLE = "FX_RATE_UNAVAILABLE"
    DEBIT_FAILED = "DEBIT_FAILED"
    ROUTING_TIMEOUT = "ROUTING_TIMEOUT"
    CREDIT_FAILED = "CREDIT_FAILED"
    FX_LOCK_EXPIRED = "FX_LOCK_EXPIRED"


@dataclass
class CompensationAction:
    """
    A compensating transaction to execute when saga fails.
    Ordered: compensation executes in reverse order of saga steps.
    """
    step: str
    command_type: str
    payload: dict


@dataclass
class RemittanceSagaState:
    """
    The complete state of one remittance saga execution.
    In production (Temporal): this state is persisted durably
    by the Temporal server and survives process restarts.
    In lab: stored in InMemorySagaRepository — lost on restart.
    """
    saga_id: str
    remittance_id: str
    current_step: SagaStep
    created_at: datetime
    updated_at: datetime
    compensation_actions: list[CompensationAction] = field(default_factory=list)
    failure_reason: str | None = None
    retry_count: int = 0
    max_retries: int = 3

    def advance_to(self, step: SagaStep) -> None:
        """Advance saga to next step. Records transition timestamp."""
        self.current_step = step
        self.updated_at = datetime.now(timezone.utc)

    def record_compensation(self, action: CompensationAction) -> None:
        """
        Record a compensation action as saga progresses.
        Compensations are added in forward order;
        executed in REVERSE order on failure.
        """
        self.compensation_actions.append(action)

    def get_compensations_in_reverse(self) -> list[CompensationAction]:
        """Return compensating transactions in reverse saga order."""
        return list(reversed(self.compensation_actions))


class RemittanceSagaOrchestrator:
    """
    Saga Orchestrator: Coordinates the remittance lifecycle.

    Receives events from Kafka, advances saga state machine,
    sends commands to participant services, handles failures
    with compensating transactions.

    DEPENDENCY INJECTION:
    command_publisher: callable that sends commands to Kafka
    saga_repository: stores and retrieves saga state
    """

    def __init__(
        self,
        command_publisher: Callable[[str, dict], None],
        saga_repository,
    ) -> None:
        self._publish_command = command_publisher
        self._repository = saga_repository

    def handle_event(self, event_type: str, payload: dict) -> None:
        """
        Main event handler — called by Kafka consumer for each event.
        Routes to appropriate handler based on event type.
        """
        remittance_id = payload.get("remittanceId")
        if not remittance_id:
            return

        # Pattern matching on event type string
        # In production: use Avro schema registry type discrimination
        match event_type:
            case "RemittanceInitiated":
                self._handle_initiated(remittance_id, payload)
            case "FXRateLocked":
                self._handle_fx_locked(remittance_id, payload)
            case "DebitCompleted":
                self._handle_debited(remittance_id, payload)
            case "CrossBorderRoutingInitiated":
                self._handle_routing_initiated(remittance_id, payload)
            case "CreditCompleted":
                self._handle_credited(remittance_id, payload)
            case "RemittanceFailed":
                self._handle_failed(remittance_id, payload)
            case _:
                # Unknown event type — log and continue
                # Do NOT raise — consuming service must never crash
                # on unknown event types (open-closed principle for consumers)
                print(json.dumps({
                    "level": "WARN",
                    "message": "Unknown event type — ignoring",
                    "eventType": event_type,
                    "remittanceId": remittance_id,
                }))

    def _handle_initiated(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Saga Step 1: RemittanceInitiated received.
        Create saga state. Send FX rate lock command.
        """
        saga = RemittanceSagaState(
            saga_id=str(uuid.uuid4()),
            remittance_id=remittance_id,
            current_step=SagaStep.INITIATED,
            created_at=datetime.now(timezone.utc),
            updated_at=datetime.now(timezone.utc),
        )
        self._repository.save(saga)
        saga.advance_to(SagaStep.FX_RATE_LOCKING)
        self._repository.save(saga)

        # Send command to FX service
        # In production: this is an HTTP call to the FX rate service
        # wrapped in a Temporal Activity with retry policy
        self._publish_command("LockFXRate", {
            "remittanceId": remittance_id,
            "sourceCurrency": payload.get("sourceCurrency", "SGD"),
            "destinationCurrency": payload.get("destinationCurrency", "INR"),
            "amount": payload.get("sourceAmount"),
            "sagaId": saga.saga_id,
        })

        self._log_step(remittance_id, "FX_RATE_LOCKING", "Sent LockFXRate command")

    def _handle_fx_locked(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Saga Step 2: FXRateLocked received.
        Record FX release as compensation (in case we need to unwind).
        Send debit command.
        """
        saga = self._repository.find(remittance_id)
        if saga is None:
            return

        # Record compensation: if we fail after this point,
        # the FX lock must be released
        saga.record_compensation(CompensationAction(
            step="FX_RATE_LOCKED",
            command_type="ReleaseFXRate",
            payload={
                "remittanceId": remittance_id,
                "reason": "SAGA_COMPENSATION",
            }
        ))

        saga.advance_to(SagaStep.FX_RATE_LOCKED)
        saga.advance_to(SagaStep.DEBITING)
        self._repository.save(saga)

        # Send debit command to Singapore bank adapter
        self._publish_command("DebitSourceAccount", {
            "remittanceId": remittance_id,
            "amount": payload.get("sourceAmount"),
            "currency": payload.get("sourceCurrency", "SGD"),
            "fxRate": payload.get("fxRate"),
            "sagaId": saga.saga_id,
        })

        self._log_step(remittance_id, "DEBITING", "Sent DebitSourceAccount command")

    def _handle_debited(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Saga Step 3: DebitCompleted received.
        CRITICAL POINT: Money has left sender's account.
        Record debit reversal as compensation — MUST execute if anything fails.
        Send cross-border routing command.
        """
        saga = self._repository.find(remittance_id)
        if saga is None:
            return

        # Record compensation: debit reversal
        # This is the most critical compensation in the saga.
        # If cross-border routing fails, this MUST execute.
        # The amount must match exactly.
        saga.record_compensation(CompensationAction(
            step="DEBITED",
            command_type="ReverseDebit",
            payload={
                "remittanceId": remittance_id,
                "bankTransactionRef": payload.get("bankTransactionRef"),
                "amount": payload.get("amountDebited"),
                "currency": payload.get("currency", "SGD"),
                "reason": "SAGA_COMPENSATION",
            }
        ))

        saga.advance_to(SagaStep.DEBITED)
        saga.advance_to(SagaStep.ROUTING)
        self._repository.save(saga)

        self._publish_command("InitiateCrossBorderRouting", {
            "remittanceId": remittance_id,
            "beneficiaryVpa": payload.get("beneficiaryVpa"),
            "destinationAmount": payload.get("destinationAmount"),
            "destinationCurrency": payload.get("destinationCurrency", "INR"),
            "sagaId": saga.saga_id,
        })

        self._log_step(remittance_id, "ROUTING", "Sent InitiateCrossBorderRouting command")

    def _handle_credited(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Saga Step 4: CreditCompleted received.
        Money has arrived at beneficiary.
        Send completion command — saga approaching terminal state.
        """
        saga = self._repository.find(remittance_id)
        if saga is None:
            return

        saga.advance_to(SagaStep.CREDITED)
        self._repository.save(saga)

        self._publish_command("CompleteRemittance", {
            "remittanceId": remittance_id,
            "sagaId": saga.saga_id,
        })

        self._log_step(remittance_id, "COMPLETING", "Sent CompleteRemittance command")

    def _handle_routing_initiated(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Routing initiated. No state change needed —
        waiting for CreditCompleted or timeout.
        In production: Temporal sets a timer here.
        If CreditCompleted not received within 30s:
        Temporal fires a timeout signal → execute compensation.
        """
        self._log_step(remittance_id, "ROUTING_IN_PROGRESS",
                       "Cross-border routing initiated — awaiting credit confirmation")

    def _handle_failed(
        self,
        remittance_id: str,
        payload: dict
    ) -> None:
        """
        Saga failure handler.
        Executes all recorded compensating transactions in REVERSE order.
        Each compensation is sent as a command to the appropriate service.

        CRITICAL DESIGN RULE:
        Compensating transactions must be IDEMPOTENT.
        If the compensation command is processed twice (at-least-once delivery),
        the second execution must not cause additional side effects.
        ReverseDebit processed twice = two reversals = money sent back TWICE.
        Use idempotency key (remittanceId + compensationStep) to prevent this.
        """
        saga = self._repository.find(remittance_id)
        if saga is None:
            return

        saga.advance_to(SagaStep.COMPENSATING)
        saga.failure_reason = payload.get("reason", "UNKNOWN")
        self._repository.save(saga)

        compensations = saga.get_compensations_in_reverse()

        self._log_step(
            remittance_id,
            "COMPENSATING",
            f"Executing {len(compensations)} compensating transactions"
        )

        for compensation in compensations:
            # Add idempotency key to each compensation command
            compensation.payload["idempotencyKey"] = (
                f"{remittance_id}-{compensation.step}-compensation"
            )
            self._publish_command(
                compensation.command_type,
                compensation.payload
            )
            self._log_step(
                remittance_id,
                f"COMPENSATION_{compensation.step}",
                f"Sent {compensation.command_type}"
            )

        saga.advance_to(SagaStep.COMPENSATION_COMPLETED)
        self._repository.save(saga)

    def _log_step(
        self,
        remittance_id: str,
        step: str,
        message: str
    ) -> None:
        print(json.dumps({
            "level": "INFO",
            "message": message,
            "remittanceId": remittance_id,
            "sagaStep": step,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }))
```

---

## PROJECTION SERVICE (Python 3.10)

```
FILE: projection-service/src/projections/status_projection.py
LANGUAGE: Python 3.10
PURPOSE: READ SIDE — maintains a current-state view of each remittance.
         This is the CQRS query model. AP behaviour — slightly eventual consistency
         with the write side is acceptable for status display.
CONCEPTS DEMONSTRATED: CQRS projection, eventual consistency, AP read model
COPY-PASTE READY: YES
PRODUCTION DELTA: Replace in-memory dict with Redis sorted set for
                  fast lookups + TTL-based expiry of completed remittances.
```

```python
# projection-service/src/projections/status_projection.py
#
# ARCHITECTURAL NOTE ON CQRS PROJECTIONS:
#
# The write side (RemittanceAggregate) enforces all business rules.
# The read side (this projection) serves queries efficiently.
# They are DIFFERENT models optimised for different purposes.
#
# Write side: normalised, consistent, enforces invariants
# Read side: denormalised, optimised for the specific query shape
#
# This projection builds a "remittance status" view that answers:
# "What is the current state of remittance X?" in < 1ms.
# Without CQRS: this query would require joining multiple tables
# or replaying the event history on every read.
#
# EVENTUAL CONSISTENCY NOTE:
# This projection may be 1-2 seconds behind the write side.
# A user who initiates a remittance and immediately queries status
# might briefly see "INITIATED" instead of "FX_LOCKED".
# This is ACCEPTABLE for a status display.
# It is NOT acceptable for the debit decision itself.
# That distinction — acceptable vs. unacceptable staleness — is
# a business decision, not a technical one.

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional


@dataclass
class RemittanceStatusView:
    """
    Read model for remittance status queries.
    Denormalised — all information needed for the status page
    in one object. No joins. No aggregate replay. O(1) lookup.
    """
    remittance_id: str
    customer_id: str
    status: str
    source_amount: float
    source_currency: str
    destination_amount: float | None
    destination_currency: str
    beneficiary_vpa: str
    fx_rate: float | None
    failure_reason: str | None
    initiated_at: str
    last_updated_at: str
    events_received: int = 0


class RemittanceStatusProjection:
    """
    Projection: Remittance Status View

    Consumes remittance-events from Kafka.
    Updates an in-memory read model on each event.
    Serves status queries from the read model — never from event store.

    IDEMPOTENCY:
    This projection must be idempotent — processing the same event
    twice must produce the same result as processing it once.
    Kafka at-least-once delivery means duplicates can occur.
    Each event application is a pure state update (set, not append)
    so duplicate application is naturally idempotent here.
    """

    def __init__(self) -> None:
        # In-memory read store — key: remittance_id → status view
        # Production: Redis with TTL for completed remittances
        self._views: dict[str, RemittanceStatusView] = {}

    def apply(self, event_type: str, payload: dict) -> None:
        """
        Apply a domain event to update the read model.
        Called by Kafka consumer for each event.
        """
        remittance_id = payload.get("remittanceId")
        if not remittance_id:
            return

        match event_type:
            case "RemittanceInitiated":
                self._apply_initiated(remittance_id, payload)
            case "FXRateLocked":
                self._apply_fx_locked(remittance_id, payload)
            case "DebitCompleted":
                self._apply_debited(remittance_id, payload)
            case "CrossBorderRoutingInitiated":
                self._apply_routing(remittance_id, payload)
            case "CreditCompleted":
                self._apply_credited(remittance_id, payload)
            case "RemittanceCompleted":
                self._apply_completed(remittance_id, payload)
            case "RemittanceFailed":
                self._apply_failed(remittance_id, payload)
            case "DebitReversed":
                self._apply_debit_reversed(remittance_id, payload)
            case _:
                pass  # Unknown events are ignored — forward compatibility

    def get_status(
        self,
        remittance_id: str
    ) -> Optional[RemittanceStatusView]:
        """
        Query: Get current status of a remittance.
        O(1) lookup. No event replay. No database query.
        This is the CQRS query model in action.
        """
        return self._views.get(remittance_id)

    def get_all_active(self) -> list[RemittanceStatusView]:
        """Query: Get all non-terminal remittances for dashboard."""
        terminal_states = {"COMPLETED", "FAILED", "DEBIT_REVERSED"}
        return [
            view for view in self._views.values()
            if view.status not in terminal_states
        ]

    def _apply_initiated(self, remittance_id: str, payload: dict) -> None:
        self._views[remittance_id] = RemittanceStatusView(
            remittance_id=remittance_id,
            customer_id=payload.get("customerId", ""),
            status="INITIATED",
            source_amount=float(payload.get("sourceAmount", 0)),
            source_currency=payload.get("sourceCurrency", "SGD"),
            destination_amount=None,
            destination_currency=payload.get("destinationCurrency", "INR"),
            beneficiary_vpa=payload.get("beneficiaryVpa", ""),
            fx_rate=None,
            failure_reason=None,
            initiated_at=payload.get("occurredAt", ""),
            last_updated_at=datetime.now(timezone.utc).isoformat(),
            events_received=1,
        )

    def _apply_fx_locked(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "FX_LOCKED"
        view.fx_rate = float(payload.get("fxRate", 0))
        view.destination_amount = float(payload.get("destinationAmount", 0))
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_debited(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "DEBITED"
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_routing(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "ROUTING_IN_PROGRESS"
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_credited(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "CREDITED"
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_completed(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "COMPLETED"
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_failed(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "FAILED"
        view.failure_reason = payload.get("reason", "UNKNOWN")
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1

    def _apply_debit_reversed(self, remittance_id: str, payload: dict) -> None:
        view = self._views.get(remittance_id)
        if view is None:
            return
        view.status = "DEBIT_REVERSED"
        view.last_updated_at = datetime.now(timezone.utc).isoformat()
        view.events_received += 1
```

---

## KAFKA CONSUMER — Shared Infrastructure (Python 3.10)

```
FILE: projection-service/src/adapters/kafka_consumer.py
LANGUAGE: Python 3.10
PURPOSE: Driving adapter — consumes events from Kafka and routes to projections.
         Implements at-least-once semantics with idempotent projection handlers.
         Includes dead letter queue routing for poison pill messages.
CONCEPTS DEMONSTRATED: Kafka consumer group, offset management, DLQ pattern
COPY-PASTE READY: YES
PRODUCTION DELTA: Add OpenTelemetry span extraction from Kafka headers.
                  Add consumer lag metrics emission to Prometheus.
                  Add graceful shutdown with offset commit before SIGTERM.
```

```python
# projection-service/src/adapters/kafka_consumer.py
#
# ARCHITECTURAL NOTE ON CONSUMER GROUP SEMANTICS:
#
# Consumer Group: "projection-service-group"
# All instances of the projection-service share this group ID.
# Kafka assigns each partition to exactly ONE instance in the group.
#
# If we run 2 projection-service instances and have 8 partitions:
# Instance 1: handles partitions 0,1,2,3
# Instance 2: handles partitions 4,5,6,7
#
# CRITICAL: Both instances must maintain IDENTICAL projection state.
# In this lab: each instance has its own in-memory projection.
# An API request hitting Instance 1 might return different data
# than the same request hitting Instance 2.
# This is the distributed state problem with in-memory projections.
#
# PRODUCTION SOLUTION: Replace in-memory dict with shared Redis cluster.
# Both instances read/write to the same Redis.
# Partition rebalancing no longer causes state loss.
#
# OFFSET MANAGEMENT:
# enable.auto.commit=False: We commit offsets MANUALLY after processing.
# This gives us at-least-once semantics:
# Process → Commit offset (success path)
# Process → Crash before commit → Replay on restart (duplicate, but idempotent)
# If we used auto.commit: crash after commit but before processing = message lost

import json
import threading
from typing import Callable

from confluent_kafka import Consumer, KafkaError, KafkaException, Producer


class RemittanceEventConsumer:
    """
    Kafka consumer for remittance-events topic.
    Routes each event to the registered handler.
    Handles poison pill messages by routing to DLQ.
    """

    MAX_RETRIES = 3
    DLQ_TOPIC = "remittance-dlq"

    def __init__(
        self,
        bootstrap_servers: str,
        group_id: str,
        topics: list[str],
        event_handler: Callable[[str, dict], None],
    ) -> None:
        self._handler = event_handler
        self._running = False

        # Consumer configuration
        self._consumer = Consumer({
            "bootstrap.servers": bootstrap_servers,
            "group.id": group_id,
            "auto.offset.reset": "earliest",
            # Manual offset commit — at-least-once semantics
            # We commit AFTER successful processing
            "enable.auto.commit": False,
            # Session timeout — if consumer does not poll within this window,
            # Kafka triggers partition rebalancing
            # Tune based on your processing time per message
            "session.timeout.ms": 30000,
            "max.poll.interval.ms": 300000,
        })
        self._consumer.subscribe(topics)

        # DLQ producer — for messages that fail after MAX_RETRIES
        self._dlq_producer = Producer({
            "bootstrap.servers": bootstrap_servers,
        })

        self._thread: threading.Thread | None = None

    def start(self) -> None:
        """Start consuming in a background thread."""
        self._running = True
        self._thread = threading.Thread(
            target=self._consume_loop,
            name=f"kafka-consumer-{id(self)}",
            daemon=True,
        )
        self._thread.start()

    def stop(self) -> None:
        """Graceful shutdown — commit current offsets before stopping."""
        self._running = False
        if self._thread:
            self._thread.join(timeout=10)
        self._consumer.close()

    def _consume_loop(self) -> None:
        """
        Main consume loop.
        Polls Kafka for messages. Routes to handler. Commits offset.
        Retries on transient failures. Routes to DLQ on persistent failure.
        """
        while self._running:
            try:
                # Poll with 1-second timeout
                # Timeout allows the running check to be evaluated
                msg = self._consumer.poll(timeout=1.0)

                if msg is None:
                    continue  # No message — poll again

                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        # End of partition — not an error, just caught up
                        continue
                    raise KafkaException(msg.error())

                self._process_with_retry(msg)

            except Exception as e:
                print(json.dumps({
                    "level": "ERROR",
                    "message": "Fatal consumer error",
                    "error": str(e),
                }))
                # In production: emit metric, alert on-call, attempt recovery
                # Here: log and continue to avoid consumer crash
                # A crashed consumer causes partition rebalancing —
                # impacting all consumers in the group

    def _process_with_retry(self, msg) -> None:
        """
        Process one message with retry logic.
        On MAX_RETRIES exhaustion: route to DLQ and commit offset.
        The DLQ message preserves original message for manual investigation.
        """
        retry_count = 0
        while retry_count <= self.MAX_RETRIES:
            try:
                value = json.loads(msg.value().decode("utf-8"))
                event_type = value.get("eventType")
                payload = value.get("payload", value)

                self._handler(event_type, payload)

                # SUCCESS: commit offset manually
                # This tells Kafka: "I have processed up to this offset"
                # On consumer restart: resume from NEXT offset
                self._consumer.commit(msg)
                return

            except json.JSONDecodeError as e:
                # Malformed JSON — no point retrying
                # Route directly to DLQ
                self._route_to_dlq(msg, f"JSON decode error: {e}")
                self._consumer.commit(msg)
                return

            except Exception as e:
                retry_count += 1
                if retry_count > self.MAX_RETRIES:
                    # Exhausted retries — route to DLQ, commit, move on
                    # DO NOT leave this message blocking the partition
                    # A stuck message blocks all subsequent messages
                    # on the same partition — the "poison pill" problem
                    self._route_to_dlq(msg, f"Max retries exceeded: {e}")
                    self._consumer.commit(msg)
                    return

                # Exponential backoff before retry
                import time
                time.sleep(0.1 * (2 ** retry_count))

    def _route_to_dlq(self, original_msg, error_reason: str) -> None:
        """
        Route a failed message to the Dead Letter Queue.
        DLQ message includes original message + failure metadata.
        Operations team monitors DLQ for manual investigation.
        A growing DLQ = silent production incident.
        """
        dlq_payload = json.dumps({
            "originalTopic": original_msg.topic(),
            "originalPartition": original_msg.partition(),
            "originalOffset": original_msg.offset(),
            "originalKey": original_msg.key().decode("utf-8")
                          if original_msg.key() else None,
            "originalValue": original_msg.value().decode("utf-8"),
            "errorReason": error_reason,
            "routedAt": __import__("datetime").datetime.utcnow().isoformat(),
        })

        self._dlq_producer.produce(
            topic=self.DLQ_TOPIC,
            key=original_msg.key(),
            value=dlq_payload.encode("utf-8"),
        )
        self._dlq_producer.flush()

        print(json.dumps({
            "level": "ERROR",
            "message": "Message routed to DLQ",
            "topic": original_msg.topic(),
            "partition": original_msg.partition(),
            "offset": original_msg.offset(),
            "reason": error_reason,
        }))
```

---

## DOMAIN TESTS — Event Sourcing Verification (Java 17)

```
FILE: remittance-command-service/src/test/java/com/remitflow/domain/RemittanceAggregateTest.java
LANGUAGE: Java 17
PURPOSE: Proves that event sourcing reconstitution produces identical state
         to live aggregate processing. This is the EVENT SOURCING GUARANTEE test.
CONCEPTS DEMONSTRATED: Aggregate reconstitution, domain invariant testing,
                       compensating event verification
COPY-PASTE READY: YES
```

```java
// FILE: com/remitflow/domain/RemittanceAggregateTest.java
//
// THE MOST IMPORTANT TEST IN EVENT SOURCING:
// Reconstitute an aggregate from its event history and
// verify that the reconstituted state is identical to
// the state produced by live command processing.
// If this test passes: your event sourcing is correct.
// If this test fails: your apply() method has a bug.

package com.remitflow.domain;

import com.remitflow.domain.events.RemittanceEvent;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Nested;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

@DisplayName("RemittanceAggregate — Event Sourcing Tests")
class RemittanceAggregateTest {

    private RemittanceAggregate createTestAggregate() {
        return RemittanceAggregate.initiate(
                "rem-test-001",
                "cust-sg-001",
                "SGD",
                new BigDecimal("500.00"),
                "INR",
                "anand@upi"
        );
    }

    @Test
    @DisplayName("New aggregate starts in INITIATED status")
    void newAggregateIsInitiated() {
        var aggregate = createTestAggregate();
        assertThat(aggregate.status()).isEqualTo(RemittanceStatus.INITIATED);
        assertThat(aggregate.version()).isEqualTo(1);
    }

    @Test
    @DisplayName("Invalid UPI VPA format is rejected by domain invariant")
    void invalidVpaRejected() {
        assertThatThrownBy(() -> RemittanceAggregate.initiate(
                "rem-test-002",
                "cust-sg-001",
                "SGD",
                new BigDecimal("500.00"),
                "INR",
                "invalid-vpa-no-at-sign"  // Missing @ — invalid UPI VPA
        ))
        .isInstanceOf(RemittanceInvariantException.class)
        .hasMessageContaining("Invalid UPI VPA format");
    }

    @Nested
    @DisplayName("Event Sourcing Reconstitution — THE CORE GUARANTEE")
    class ReconstitutionTests {

        @Test
        @DisplayName("Reconstituted aggregate matches live aggregate state")
        void reconstitutionProducesIdenticalState() {
            // STEP 1: Process commands on a live aggregate
            var liveAggregate = createTestAggregate();

            // Collect events as they are produced
            var allEvents = new java.util.ArrayList<>(
                liveAggregate.popPendingEvents()
            );

            liveAggregate.lockFXRate(
                new BigDecimal("62.45"),
                new BigDecimal("31225.00")
            );
            allEvents.addAll(liveAggregate.popPendingEvents());

            liveAggregate.completeDebit("DBSSG-TXN-001");
            allEvents.addAll(liveAggregate.popPendingEvents());

            // STEP 2: Reconstitute a NEW aggregate from the event history
            var reconstitutedAggregate = RemittanceAggregate.reconstitute(allEvents);

            // STEP 3: Verify identical state
            // This is the EVENT SOURCING GUARANTEE:
            // Replay the history → get the same state.
            // If this assertion fails, your apply() method is wrong.
            assertThat(reconstitutedAggregate.status())
                    .isEqualTo(liveAggregate.status())
                    .isEqualTo(RemittanceStatus.DEBITED);

            assertThat(reconstitutedAggregate.version())
                    .isEqualTo(liveAggregate.version())
                    .isEqualTo(3); // 3 events applied

            assertThat(reconstitutedAggregate.sourceAmount())
                    .isEqualByComparingTo(liveAggregate.sourceAmount());

            assertThat(reconstitutedAggregate.debitOccurred())
                    .isEqualTo(liveAggregate.debitOccurred())
                    .isTrue();
        }

        @Test
        @DisplayName("Failure after debit sets debitOccurred=true — Saga uses this")
        void failureAfterDebitRecordsDebitOccurred() {
            var aggregate = createTestAggregate();
            aggregate.popPendingEvents(); // Clear initiation events

            aggregate.lockFXRate(
                new BigDecimal("62.45"),
                new BigDecimal("31225.00")
            );
            aggregate.popPendingEvents();

            aggregate.completeDebit("DBSSG-TXN-002");
            aggregate.popPendingEvents();

            // Now fail — simulating cross-border routing failure
            aggregate.fail("CROSS_BORDER_ROUTING", "UPI VPA not found");
            var failureEvents = aggregate.popPendingEvents();

            // The RemittanceFailed event must record that debit occurred
            // The Saga orchestrator reads this flag to determine
            // whether it needs to execute DebitReversed compensation
            assertThat(failureEvents).hasSize(1);
            var failedEvent = (RemittanceEvent.RemittanceFailed) failureEvents.get(0);
            assertThat(failedEvent.debitOccurred()).isTrue();
            assertThat(failedEvent.failedAtStep()).isEqualTo("CROSS_BORDER_ROUTING");
        }

        @Test
        @DisplayName("Cannot complete remittance in wrong status — invariant enforced")
        void statusInvariantPreventsInvalidTransition() {
            var aggregate = createTestAggregate();

            // Try to complete without debiting first
            assertThatThrownBy(aggregate::complete)
                    .isInstanceOf(RemittanceInvariantException.class)
                    .hasMessageContaining("Expected status");
        }

        @Test
        @DisplayName("Pending events are cleared after pop — no double publishing")
        void pendingEventsClearedAfterPop() {
            var aggregate = createTestAggregate();
            var events = aggregate.popPendingEvents();

            assertThat(events).hasSize(1);
            // Second pop — must be empty
            // If not: events would be published to Kafka twice
            assertThat(aggregate.popPendingEvents()).isEmpty();
        }
    }
}
```

---

## INFRASTRUCTURE — Terraform for Azure Event Hubs

```
FILE: infra/terraform/main.tf
LANGUAGE: HCL
PURPOSE: Provisions Azure Event Hubs with Kafka-compatible API.
         Allows the same Java/Python Kafka clients to connect to Azure Event Hubs
         without code changes — only bootstrap server configuration changes.
CONCEPTS DEMONSTRATED: Cloud-native Kafka alternative, IaC for event streaming
COPY-PASTE READY: YES
PRODUCTION DELTA: Add Azure Private Endpoints to prevent public internet exposure.
                  Add Azure Monitor diagnostic settings for consumer lag alerting.
                  Add geo-redundant namespace for Singapore + India regional deployment.
```

```hcl
# infra/terraform/main.tf
#
# ARCHITECTURAL NOTE: AZURE EVENT HUBS vs. SELF-MANAGED KAFKA
#
# Azure Event Hubs provides a Kafka-compatible API.
# Your existing Kafka producers/consumers connect unchanged.
# Only the bootstrap.servers config value changes.
#
# WHEN TO USE AZURE EVENT HUBS:
# ✅ You want managed infrastructure (no Kafka broker management)
# ✅ MAS TRM compliance documentation needed (Azure is MAS-recognised)
# ✅ Auto-scaling throughput units without partition rebalancing
# ✅ Built-in Azure Monitor integration
#
# WHEN TO STICK WITH SELF-MANAGED KAFKA:
# ❌ You need Kafka Streams (not supported in Event Hubs)
# ❌ You need exactly-once semantics with transactions (limited support)
# ❌ You need log compaction (not supported in Event Hubs Standard tier)
# ❌ You need retention > 7 days on Standard tier (Premium: 90 days)
#
# For our audit-events topic (permanent retention): self-managed Kafka
# or Event Hubs Premium tier is required.

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "remitflow" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment  = "lab"
    project      = "remitflow-day3"
    cost-centre  = "training"
    geography    = "singapore"
  }
}

# Event Hubs Namespace — the Kafka cluster equivalent
# Standard tier: Kafka-compatible API, up to 10 consumer groups,
# 1GB/day ingress (free tier equivalent for lab purposes)
resource "azurerm_eventhub_namespace" "remitflow" {
  name                = "evhns-remitflow-${var.environment}"
  location            = azurerm_resource_group.remitflow.location
  resource_group_name = azurerm_resource_group.remitflow.name
  sku                 = "Standard"
  capacity            = 1  # 1 throughput unit = 1MB/s ingress

  # Kafka endpoint — your existing Kafka clients connect here
  # bootstrap.servers = evhns-remitflow-lab.servicebus.windows.net:9093
  kafka_enabled = true

  tags = azurerm_resource_group.remitflow.tags
}

# Event Hub (= Kafka Topic) for remittance-events
resource "azurerm_eventhub" "remittance_events" {
  name                = "remittance-events"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  resource_group_name = azurerm_resource_group.remitflow.name
  partition_count     = 8      # Matches our lab Docker Kafka config
  message_retention   = 7      # Days — Standard tier max: 7 days
}

# Event Hub for audit-events
# NOTE: Standard tier max retention is 7 days.
# For MAS TRM 5-year retention: use Event Hubs Premium (90 days)
# and archive to Azure Blob Storage for long-term retention.
resource "azurerm_eventhub" "audit_events" {
  name                = "audit-events"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  resource_group_name = azurerm_resource_group.remitflow.name
  partition_count     = 4
  message_retention   = 7  # Lab: 7 days. Production: Premium + Blob archive
}

# Event Hub for DLQ
resource "azurerm_eventhub" "remittance_dlq" {
  name                = "remittance-dlq"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  resource_group_name = azurerm_resource_group.remitflow.name
  partition_count     = 2
  message_retention   = 7
}

# Consumer Groups — one per consumer service
# Event Hubs Standard: max 10 consumer groups per Event Hub

resource "azurerm_eventhub_consumer_group" "saga_orchestrator" {
  name                = "saga-orchestrator-group"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  eventhub_name       = azurerm_eventhub.remittance_events.name
  resource_group_name = azurerm_resource_group.remitflow.name
}

resource "azurerm_eventhub_consumer_group" "projection_service" {
  name                = "projection-service-group"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  eventhub_name       = azurerm_eventhub.remittance_events.name
  resource_group_name = azurerm_resource_group.remitflow.name
}

resource "azurerm_eventhub_consumer_group" "audit_service" {
  name                = "audit-service-group"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  eventhub_name       = azurerm_eventhub.remittance_events.name
  resource_group_name = azurerm_resource_group.remitflow.name
}

# Shared access policy — Kafka clients use this for authentication
# In production: use Managed Identity instead of connection strings
resource "azurerm_eventhub_namespace_authorization_rule" "remitflow_app" {
  name                = "remitflow-app-policy"
  namespace_name      = azurerm_eventhub_namespace.remitflow.name
  resource_group_name = azurerm_resource_group.remitflow.name
  listen              = true
  send                = true
  manage              = false  # App does not need manage — principle of least privilege
}
```

```hcl
# infra/terraform/outputs.tf

output "eventhubs_namespace_fqdn" {
  description = "Bootstrap server for Kafka clients"
  value       = "${azurerm_eventhub_namespace.remitflow.name}.servicebus.windows.net:9093"
}

output "kafka_connection_string" {
  description = "Connection string for Kafka SASL/SSL authentication"
  value       = azurerm_eventhub_namespace_authorization_rule.remitflow_app.primary_connection_string
  sensitive   = true  # Never log this value
}

output "resource_group_name" {
  value = azurerm_resource_group.remitflow.name
}
```

---

# L4. STEP-BY-STEP EXECUTION GUIDE

---

**STEP 1 of 7: Initiate a remittance and observe event propagation**

🎯 OBJECTIVE: Confirm that a single command triggers the full event chain across all services.

CONCEPT LINK: Event-Driven Architecture — command triggers events which trigger projections (Block 2).

```bash
$ curl -X POST http://localhost:8080/remittances \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "customerId": "CUST-SG-001",
    "sourceCurrency": "SGD",
    "sourceAmount": 500.00,
    "destinationCurrency": "INR",
    "beneficiaryVpa": "anand@upi"
  }' | python3 -m json.tool
```

Expected Output:
```json
{
  "remittanceId": "rem-a1b2c3d4-e5f6-7890",
  "status": "INITIATED",
  "message": "Remittance initiated. Processing via event-driven pipeline."
}
```

```bash
# Store the remittance ID
$ REMITTANCE_ID="rem-a1b2c3d4-e5f6-7890"
```

🔍 WHAT TO OBSERVE: Copy the remittanceId. This ID is the Kafka partition key — all events for this remittance go to the same partition.

---

**STEP 2 of 7: Watch events flow through Kafka in real time**

🎯 OBJECTIVE: Observe the event stream as it propagates from command service through Kafka to consumers.

CONCEPT LINK: Kafka partitioning and consumer groups (Block 2).

```bash
# Open a second terminal window
# Watch remittance-events topic in real time
$ docker compose exec kafka kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic remittance-events \
    --from-beginning \
    --property print.key=true \
    --property key.separator=" | "
```

Expected Output (you will see events appearing as the Saga processes):
```
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"RemittanceInitiated",...}
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"FXRateLocked",...}
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"DebitCompleted",...}
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"CrossBorderRoutingInitiated",...}
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"CreditCompleted",...}
rem-a1b2c3d4 | {"aggregateId":"rem-a1b2c3d4","eventType":"RemittanceCompleted",...}
```

🔍 WHAT TO OBSERVE: All events share the SAME partition key (`rem-a1b2c3d4`). This guarantees they arrive at the same Kafka partition in the order they were produced. Without this — a consumer might process DebitCompleted before FXRateLocked — a financial disaster.

⚠️ COMMON MISTAKE: Candidates use the Kafka console consumer with `--from-beginning` in production demos. This replays ALL messages from the beginning of time — including millions of historical messages. In a lab with a fresh topic, this is fine. In production, use `--offset latest`.

---

**STEP 3 of 7: Query the projection service**

🎯 OBJECTIVE: Confirm CQRS read/write separation — query the read model without touching the event store.

CONCEPT LINK: CQRS — separate read and write models (Block 2, Use Case 1).

```bash
$ curl -s http://localhost:8082/remittances/$REMITTANCE_ID/status \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "remittanceId": "rem-a1b2c3d4-e5f6-7890",
  "customerId": "CUST-SG-001",
  "status": "COMPLETED",
  "sourceAmount": 500.0,
  "sourceCurrency": "SGD",
  "destinationAmount": 31225.0,
  "destinationCurrency": "INR",
  "beneficiaryVpa": "anand@upi",
  "fxRate": 62.45,
  "failureReason": null,
  "initiatedAt": "2026-06-15T09:35:00Z",
  "lastUpdatedAt": "2026-06-15T09:35:03Z",
  "eventsReceived": 6
}
```

🔍 WHAT TO OBSERVE: `eventsReceived: 6` — the projection processed all 6 events. `lastUpdatedAt` is 3 seconds after `initiatedAt` — this is the eventual consistency lag. The write side committed the `RemittanceCompleted` event 3 seconds ago; the projection processed it almost immediately. This demonstrates AP behaviour — the read model is eventually consistent with the write model, not immediately consistent.

---

**STEP 4 of 7: Inject a failure and observe Saga compensation**

🎯 OBJECTIVE: Demonstrate the Saga compensating transactions — the most important failure-handling pattern in distributed systems.

CONCEPT LINK: Saga Pattern with compensating transactions (Block 3, Pattern 1).

```bash
# Inject a failure at the cross-border routing step
$ curl -X POST http://localhost:8080/remittances \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "customerId": "CUST-SG-002",
    "sourceCurrency": "SGD",
    "sourceAmount": 750.00,
    "destinationCurrency": "INR",
    "beneficiaryVpa": "FAIL_AT_ROUTING@upi"
  }' | python3 -m json.tool

$ FAILED_REMITTANCE_ID="<copy from response>"
```

Now watch the Kafka console consumer in the second terminal. You will see:

```
rem-b2c3d4e5 | {"eventType":"RemittanceInitiated",...}
rem-b2c3d4e5 | {"eventType":"FXRateLocked",...}
rem-b2c3d4e5 | {"eventType":"DebitCompleted",...}
rem-b2c3d4e5 | {"eventType":"CrossBorderRoutingInitiated",...}
rem-b2c3d4e5 | {"eventType":"RemittanceFailed","failedAtStep":"CROSS_BORDER_ROUTING",...}
rem-b2c3d4e5 | {"eventType":"DebitReversed",...}     ← COMPENSATION
rem-b2c3d4e5 | {"eventType":"FXRateReleased",...}    ← COMPENSATION
```

```bash
# Verify the projection shows DEBIT_REVERSED
$ curl -s http://localhost:8082/remittances/$FAILED_REMITTANCE_ID/status \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "status": "DEBIT_REVERSED",
  "failureReason": "UPI VPA not found: FAIL_AT_ROUTING@upi",
  "eventsReceived": 7
}
```

🔍 WHAT TO OBSERVE: The saga processed 7 events — 5 forward steps + 2 compensating events. The `DebitReversed` event was emitted AFTER `RemittanceFailed` — in reverse order of how the debit was taken. The FX lock was also released. The event log now contains the full story: what happened, what failed, and what was corrected. This is the audit trail that satisfies MAS TRM requirements.

⚠️ COMMON MISTAKE: Candidates forget that compensating transactions must be idempotent. If `DebitReversed` is processed twice, the account gets reversed twice — catastrophic. The idempotency key (`remittanceId-DEBITED-compensation`) prevents this. Point to the Saga orchestrator code where the idempotency key is set.

🤖 COPILOT PROMPT:
"Explain what just happened in this Kafka event sequence: RemittanceInitiated → FXRateLocked → DebitCompleted → RemittanceFailed → DebitReversed → FXRateReleased. What architectural pattern is this? What would have happened if DebitReversed was NOT idempotent and it was processed twice by the Kafka consumer?"

---

**STEP 5 of 7: Query the audit service**

🎯 OBJECTIVE: Verify the immutable audit trail maintained by the audit service.

CONCEPT LINK: MAS TRM audit requirements, Event Sourcing for compliance (Block 2, Use Case 2).

```bash
$ curl -s http://localhost:8083/audit/$FAILED_REMITTANCE_ID \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "remittanceId": "rem-b2c3d4e5",
  "auditEntries": [
    {"sequence": 1, "eventType": "RemittanceInitiated", "occurredAt": "..."},
    {"sequence": 2, "eventType": "FXRateLocked", "occurredAt": "..."},
    {"sequence": 3, "eventType": "DebitCompleted", "occurredAt": "..."},
    {"sequence": 4, "eventType": "CrossBorderRoutingInitiated", "occurredAt": "..."},
    {"sequence": 5, "eventType": "RemittanceFailed", "occurredAt": "..."},
    {"sequence": 6, "eventType": "DebitReversed", "occurredAt": "..."},
    {"sequence": 7, "eventType": "FXRateReleased", "occurredAt": "..."}
  ],
  "totalEvents": 7,
  "retentionPolicy": "PERMANENT",
  "complianceNote": "MAS TRM 2021 Section 9.4 — 5-year minimum retention"
}
```

🔍 WHAT TO OBSERVE: Sequence numbers are assigned by the audit service in the order events were received from Kafka. This sequence is the audit trail. Notice that `RetentionPolicy: PERMANENT` — the `audit-events` Kafka topic was created with `retention.ms=-1` (never delete). This is the MAS TRM compliance mechanism coded into infrastructure.

---

**STEP 6 of 7: Simulate consumer lag and observe impact**

🎯 OBJECTIVE: Demonstrate the Kafka consumer lag concept from the chaos scenario — the financial year-end incident.

CONCEPT LINK: Chaos Scenario — UPI Settlement Consumer Lag (Block 3).

```bash
# Pause the projection service to simulate lag
$ docker compose pause projection-service

# Now produce 20 remittances rapidly
$ ./scripts/produce-load.sh 20

# Check consumer lag
$ docker compose exec kafka kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --group projection-service-group
```

Expected Output (while projection-service is paused):
```
GROUP                    TOPIC               PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
projection-service-group remittance-events   0          100             120             20
projection-service-group remittance-events   1          95              113             18
...
TOTAL LAG: ~160 events
```

🔍 WHAT TO OBSERVE: LAG column shows how many messages the projection service has not yet processed. This is a BUSINESS RISK metric. In the chaos scenario — 2.3 million lag means 2.3 million payment events not yet fraud-checked. In our lab: 160 events means the status dashboard is showing stale data.

```bash
# Resume the projection service — watch lag drain to zero
$ docker compose unpause projection-service
$ sleep 5

# Check lag again
$ docker compose exec kafka kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe \
    --group projection-service-group
```

Expected Output:
```
TOTAL LAG: 0
```

🔍 WHAT TO OBSERVE: Once resumed, the consumer processes the backlog rapidly and lag returns to zero. In the chaos scenario — the fix was to ADD MORE CONSUMER INSTANCES to drain the lag faster. Kafka's partition-based parallelism allows this without code changes.

---

**STEP 7 of 7: Run domain tests — verify event sourcing guarantee**

```bash
$ cd remittance-command-service
$ ./mvnw test -Dtest="RemittanceAggregateTest" -q
```

Expected Output:
```
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time: 4.123 s
```

🔍 WHAT TO OBSERVE: The reconstitution test (`reconstitutionProducesIdenticalState`) is the proof that event sourcing works. It creates a live aggregate, processes commands, collects events, creates a NEW aggregate from those events, and verifies identical state. Zero infrastructure. Zero Kafka. Pure domain logic. This test must always pass — if it fails, your event sourcing implementation has a bug.

---

# L5. TRAINER DEMO SCRIPT

---

🎬 TRAINER DEMO SCRIPT: The Event That Cannot Be Unseen

SETUP CHECK (Before showing screen):
□ All 4 services running and healthy
□ Kafka console consumer open in second terminal window
□ Browser tab open to projection service: `http://localhost:8082/dashboard`
□ `RemittanceAggregate.java` open in code editor at the `applyEvent()` method

---

TALKING POINT 1 (While showing applyEvent() switch statement):

"Look at this switch statement. It is sealed — the Java compiler knows every possible case. If I add a new event type to the RemittanceEvent sealed interface and forget to handle it here — this switch statement does not compile. The build breaks. In a 10-person team with 50 event types, this compile-time safety prevents the most common Event Sourcing bug: a new event type that silently does nothing when replayed because someone forgot to add a case. This is Java 17 working FOR your architecture."

👉 POINT AT: The sealed interface permits clause and the switch expression cases in applyEvent()

ASK AUDIENCE: "If I removed the `sealed` keyword from RemittanceEvent — what architectural guarantee do I lose, and where exactly would a bug become possible?"

EXPECTED RESPONSES: Lose compile-time exhaustiveness. A new event type added by one developer would not trigger a compiler error in applyEvent() — it would hit the implicit default/fall-through and do nothing. The aggregate's state would be wrong after reconstitution.

---

BREAK IT (Intentional failure for learning):

Add a new event type to the sealed interface without adding a case to applyEvent():

```java
// Add to RemittanceEvent sealed interface permits clause:
permits RemittanceEvent.RemittanceInitiated,
        // ... existing events ...
        RemittanceEvent.FraudFlagRaised  // ← NEW — add to permits

// Add the record:
record FraudFlagRaised(
    String remittanceId,
    String reason,
    Instant occurredAt,
    int schemaVersion
) implements RemittanceEvent {}
```

Try to compile:

```bash
$ ./mvnw compile -q
```

🔴 SHOW:
```
[ERROR] RemittanceAggregate.java: the switch expression does not cover
all possible input values — FraudFlagRaised is not handled
```

EXPLAIN: "The compiler caught a production bug before it reached runtime. Without sealed classes, this would compile fine, deploy to production, and silently produce wrong aggregate state whenever a FraudFlagRaised event was replayed. You would discover this during an audit query — six months from now — when a regulatory report shows wrong account states."

FIX IT:
Add the case to applyEvent():

```java
case RemittanceEvent.FraudFlagRaised e -> {
    this.status = RemittanceStatus.FRAUD_FLAGGED;
}
```

Recompile — clean build.

---

🤖 COPILOT LIVE PROMPT (Run while demo code is visible):

"I am using Java 17 sealed interfaces for domain events in an Event Sourced system. What are the specific benefits of sealed interfaces for event sourcing compared to using a regular interface or abstract class? What happens at runtime if a new event type is added to a non-sealed hierarchy and a switch expression does not handle it?"

Expected output: Should cover exhaustiveness checking, compile-time safety, pattern matching optimization. Class should critique: did Copilot explain the specific link between sealed + switch expressions and event sourcing correctness? Did it address the runtime NPE or ClassCastException that occurs without sealed?

---

# L6. VERIFICATION CHECKLIST

---

Lab Completion Verification:

□ Kafka running with 4 topics — verify:
  ```bash
  docker compose exec kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
  # Expected: audit-events, remittance-commands, remittance-dlq, remittance-events
  ```

□ Successful remittance shows COMPLETED status in projection — verify:
  ```bash
  curl http://localhost:8082/remittances/{id}/status | python3 -m json.tool
  # Expected: "status": "COMPLETED"
  ```

□ Failed remittance shows DEBIT_REVERSED with 7 audit events — verify:
  ```bash
  curl http://localhost:8083/audit/{failed-id} | python3 -m json.tool
  # Expected: "totalEvents": 7
  ```

□ Consumer lag drains to zero after projection service resume — verify:
  ```bash
  docker compose exec kafka kafka-consumer-groups.sh \
    --bootstrap-server localhost:9092 \
    --describe --group projection-service-group
  # Expected: all LAG values = 0
  ```

□ Domain tests pass in under 5 seconds — verify:
  ```bash
  cd remittance-command-service && ./mvnw test -Dtest="RemittanceAggregateTest" -q
  # Expected: BUILD SUCCESS
  ```

□ audit-events topic has retention.ms=-1 — verify:
  ```bash
  docker compose exec kafka kafka-topics.sh \
    --bootstrap-server localhost:9092 \
    --describe --topic audit-events | grep "retention.ms"
  # Expected: retention.ms=-1
  ```

□ Teardown complete — verify:
  ```bash
  docker compose down -v
  # -v flag removes volumes — clears Kafka data for next lab
  ```

---

# L7. EXTENSIVE DOCUMENTATION

---

## L7.1 Architecture Decision Documentation

**Why Kafka over Azure Service Bus for this lab:**

The lab requires event replay for aggregate reconstitution — a core Event Sourcing requirement. Azure Service Bus Standard tier does not support message replay after consumption. Kafka's log-based architecture retains all messages for the configured retention period (or permanently for audit-events), enabling the reconstitute() method to replay the full event history. This is a capability difference, not a preference.

**What the code demonstrates at depth:**

The `applyEvent()` method using Java 17 sealed interface pattern matching is not syntactic sugar — it is an architectural guarantee. Every time a new event type is added to the domain, the compiler enforces that every event handler (applyEvent, switch in KafkaEventPublisher, Saga orchestrator match statement) handles the new type. This is the distributed team coordination mechanism for Event Sourcing systems: the sealed interface is the contract, and the compiler is the enforcer.

**Production scale mapping:**

At UPI scale (5,000 TPS), the remittance-events topic would need approximately 100 partitions (500 TPS per partition × 10 = 5,000 TPS headroom). Consumer groups scale to 100 instances per service. The DLQ monitoring becomes critical — at 5,000 TPS, a 0.01% DLQ rate means 50 messages per second requiring manual investigation. DLQ size alert at 1,000 messages would fire within 20 seconds of any systematic failure.

---

## L7.2 Pattern Deep-Dive

**Pattern: Event Sourcing + CQRS Combined**

Formal Definition: Event Sourcing stores state changes as an immutable sequence of events rather than as a current snapshot. CQRS separates the command model (write, consistent) from the query model (read, optimised). Combined: writes go through the event-sourced aggregate; reads are served from projections built from the event stream.

Variants:
- Event Sourcing without CQRS: single read/write model, both serve from event replay — simpler but does not scale reads independently
- CQRS without Event Sourcing: separate read/write models with traditional state storage — more common, lower complexity, loses temporal query capability
- Full ES+CQRS: as demonstrated in this lab — maximum auditability and scalability, highest complexity

When this pattern fails:
- When the team lacks operational maturity for consumer lag monitoring — a silent DLQ or growing lag creates data corruption invisible at the API layer
- When event schema evolution is not planned — changing an event record structure requires migrating all historical events or maintaining multiple schema versions forever
- When the business cannot tolerate eventual consistency on reads — some workflows require read-your-own-writes consistency that CQRS complicates

Famous implementations:
- Microsoft's Azure DevOps (formerly VSTS) rebuilt on Event Sourcing + CQRS for work item tracking at scale
- LinkedIn's Activity Feeds use Event Sourcing for the social graph
- NPCI UPI settlement engine uses event-driven settlement with audit log as primary record

---

## L7.3 Production Readiness Gap Analysis

| Lab Has                    | Production Needs                                   | Effort |
| -------------------------- | -------------------------------------------------- | ------ |
| Single Kafka broker        | 3-broker cluster with rack-aware replication       | High   |
| In-memory Saga state       | Temporal.io durable workflow engine                | High   |
| In-memory projection store | Redis Cluster with persistence                     | Medium |
| No schema registry         | Confluent Schema Registry with Avro                | Medium |
| Static FX rate mock        | Real-time FX feed with lock expiry enforcement     | High   |
| Single partition key       | Composite key strategy for hot partition avoidance | Medium |
| Manual consumer scaling    | KEDA auto-scaling on consumer lag metric           | Low    |
| No consumer lag alerting   | Prometheus + Grafana consumer lag dashboard        | Low    |
| No event encryption        | Field-level encryption for PII in events           | High   |

---

## L7.4 Security Review

The lab stores `customerId` and `beneficiaryVpa` in plaintext in Kafka events. Under PDPA Singapore, these are personal data requiring encryption in transit (TLS — provided by Kafka SSL configuration) and at rest (field-level encryption using application-layer encryption before producing to Kafka).

OWASP Top 10 mapping:
- A02 Cryptographic Failures: PII in Kafka events unencrypted — production requires field-level AES-256 before producing
- A04 Insecure Design: The DLQ contains original message payloads including PII — DLQ access must be restricted to authorised operations personnel only
- A09 Security Logging Failures: The audit-events topic IS the security log — its permanent retention and immutability are correctly implemented

MAS TRM 2021, Section 11.2: Requires encryption of sensitive data in transit and at rest. Kafka TLS configuration (SSL listener) handles transit. Application-layer field encryption handles rest.

DPDP Act 2023: `customerId` mapped to a real person constitutes personal data. Cross-border transfer of this data (Singapore to India via the UPI corridor) requires MeitY approval under Section 16. The architecture must document which fields constitute personal data and the legal basis for cross-border transfer.

---

## L7.5 Cost Architecture (Azure Free Tier)

| Resource                          | Free/Standard Tier    | Lab Usage     | Cost     |
| --------------------------------- | --------------------- | ------------- | -------- |
| Kafka (Docker local)              | Free                  | Full lab      | ₹0 / S$0 |
| Azure Event Hubs Standard         | 10M events/month free | ~5,000 events | ₹0 / S$0 |
| Azure Container Apps (4 services) | 180K vCPU-sec/month   | ~8,000 sec    | ₹0 / S$0 |

Production Equivalent (Singapore, cross-border remittance at 10,000 remittances/day):

| Resource                                           | Spec                   | Monthly Cost  |
| -------------------------------------------------- | ---------------------- | ------------- |
| Azure Event Hubs Premium (for permanent retention) | 1 PU                   | S$1,100/month |
| Temporal Cloud (Saga orchestration)                | 5M workflow executions | S$450/month   |
| Redis Enterprise (projection store)                | 6GB cluster            | S$380/month   |
| Azure Container Apps (4 services, auto-scale)      | avg 2 vCPU             | S$290/month   |

Cost Optimisation:
- Self-managed Kafka on Azure VMs (3× D4s_v3): S$680/month — cheaper than Event Hubs Premium at scale above 500M events/month
- Temporal self-hosted on AKS: eliminates S$450/month Temporal Cloud cost but adds Kubernetes operational overhead

---

## L7.6 Regulatory Compliance Notes

| Regulation                | Relevance                           | Gap in Lab                          | Production Fix                                                |
| ------------------------- | ----------------------------------- | ----------------------------------- | ------------------------------------------------------------- |
| MAS TRM 2021, S11.2       | PII encryption in Kafka             | No field encryption                 | AES-256 field encryption before produce                       |
| MAS MPI Licence Condition | FX rate audit trail                 | FX rate recorded in event — correct | Add MAS-approved FX data source reference                     |
| DPDP Act 2023, S16        | Cross-border personal data transfer | No transfer mechanism               | MeitY-approved cross-border data flow documentation           |
| RBI PA/PG Guidelines      | Payment audit trail                 | Audit events topic correct          | Add RBI-mandated fields (device fingerprint, IP, geolocation) |
| PDPA S24                  | Data breach notification            | No breach detection                 | Azure Sentinel integration with DLQ anomaly detection         |

---

## L7.7 Alternative Approaches

THIS LAB USES: Custom Python Saga state machine + Kafka

ALTERNATIVE 1: Temporal.io Workflow Engine
WOULD WORK WHEN: Production remittance system requiring crash-safe saga execution, automatic retry, timeout enforcement, and workflow versioning. Temporal persists workflow state durably — process restarts resume from last committed step, not from scratch.
TRADE-OFF vs. Custom: Temporal eliminates the entire custom state machine and saga repository. Adds operational dependency on Temporal server (self-hosted or Temporal Cloud). Debugging uses Temporal's Web UI instead of Kafka console. Strongly recommended for production.

ALTERNATIVE 2: AWS Step Functions (or Azure Durable Functions)
WOULD WORK WHEN: Team is already on AWS/Azure and wants managed saga orchestration without self-hosting Temporal. Durable Functions provide similar crash-safe execution model as Temporal.
TRADE-OFF vs. Custom: Vendor lock-in. Azure Durable Functions use Azure Storage as the workflow state backend — adds Azure dependency. Not Kafka-native. Better for teams not already committed to Kafka.

ALTERNATIVE 3: Choreography-based Saga (no orchestrator)
WOULD WORK WHEN: Simple 2–3 step sagas where compensation logic is straightforward. Each service reacts to events independently — no central coordinator.
TRADE-OFF vs. Orchestration: Lower operational complexity (no orchestrator service to manage). Higher debugging complexity — saga state is distributed across all participants' logs. As demonstrated in the chaos scenario: debugging a stuck saga in choreography requires correlating events across 6 different services. Orchestration concentrates that visibility.

---

## L7.8 Copilot Lab Prompts

UNDERSTANDING PROMPT:
"Explain the `reconstitute()` method in the `RemittanceAggregate` class. What is the Event Sourcing guarantee it tests? What would happen if the `applyEvent()` method had a bug where it set the wrong status for `CreditCompleted`? How would this bug manifest in production and how would you detect it?"

EXTENSION PROMPT:
"How would I add snapshot support to the RemittanceAggregate to avoid replaying 10,000+ events for long-lived remittances? What changes are needed in the aggregate, the event store, and the reconstitution factory method?"

CRITIQUE PROMPT:
"Review this Kafka consumer implementation for production readiness issues. Focus on: offset commit timing relative to message processing, DLQ routing for poison pill messages, consumer group rebalancing handling, and graceful shutdown. Code: [paste KafkaEventConsumer class]"

INDIA/SINGAPORE CONTEXT PROMPT:
"How would the RemittanceAggregate's event sourcing design need to change to comply with both MAS Technology Risk Management 2021 guidelines for audit trail immutability and India's RBI Payment Aggregator guidelines for cross-border payment record keeping? What specific event fields are required by each regulation?"

---

✅ DAY 3 COMPLETE

📘 Document 1: Training Content — Distributed Systems & Event-Driven Architecture ✅ Generated

🔬 Document 2: Lab Manual — RemitFlow: Event Sourcing + CQRS + Saga in Practice ✅ Generated

📊 Quality Gates: All 5 passed ✅

Geography: 🇮🇳 India (IRCTC/CoWIN/UPI/GSTN primary) + 🇸🇬 Singapore (CPF/PayNow secondary) + 🌏 Cross-border (UPI-PayNow Saga) ✅

Copilot Integration: 6 embedded prompts across both documents ✅

Documentation Templates: Technology Evaluation Matrix (Kafka vs. ASB vs. RabbitMQ), Incident Post-Mortem (Financial Year End) ✅

Java 17 Features: Sealed interfaces, records, pattern matching switch expressions, text blocks, var ✅

Python 3.10+ Features: match-case, X|Y union types, dataclasses, type hints, structural pattern matching ✅

Kafka Concepts Demonstrated: Partitioning, consumer groups, exactly-once semantics, DLQ, consumer lag, retention policies ✅

👀 Day 4 Preview: Data Architecture, NoSQL & Search

"Tomorrow we answer the question you have been building toward for three days: where does all this event data actually live at scale? We go deep into polyglot persistence — why MongoDB, Redis, Neo4J, and DynamoDB each solve a problem the others cannot — and we design a citizen data platform that answers complex government queries in under 2 seconds across 1.3 billion records."
---