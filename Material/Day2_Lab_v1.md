# DOCUMENT 2 — LAB PRACTICAL CODE PROJECT

---

## LAB MANUAL: Day 2 — API-First Design, DDD & Hexagonal Architecture
**Phase:** Architectural Foundations & Design Thinking
**Topics:** Bounded Contexts, Aggregates, Hexagonal Architecture, OpenAPI/AsyncAPI, Interoperability Patterns
**Tech Stack:** Java 17 | Python 3.10+ | Linux Ubuntu 22.04 | Azure Free Tier
**IaC:** Terraform | Orchestration: Docker Compose | CI: GitHub Actions
**Geography Context:** 🇸🇬 Singapore (SGFinDex-inspired) / 🇮🇳 India (ONDC-inspired)
**Copilot Usage:** Embedded prompts for code understanding throughout
**Pre-requisites:** Java 17 installed, Docker Desktop, Azure CLI, Python 3.10+, Git — assumes 7–12+ years engineering experience
**Estimated Lab Time:** 120 minutes | In-Class Demo: 45 minutes

---

# L0. LAB CONTEXT & ARCHITECTURE NARRATIVE

---

LAB NARRATIVE

SCENARIO:
You are building the core of a simplified Government Financial Data Exchange — inspired by Singapore's SGFinDex — called **GovFinEx**. A citizen authenticates via a consent service, selects which financial institutions to retrieve data from, and receives a unified financial summary. The system is composed of three Bounded Contexts implemented as separate services: the **Consent BC** (manages citizen consent grants), the **DataRetrieval BC** (fetches data from institution adapters), and the **Aggregation BC** (aggregates multi-institution responses into a unified view).

This lab demonstrates Hexagonal Architecture in practice — the DataRetrieval BC's core logic has zero knowledge of whether it is talking to a PostgreSQL adapter, a mock institution adapter, or a real bank API. You swap adapters without touching core logic. You also implement an OpenAPI-first workflow: the API contract is written first, code is generated from it, and the contract becomes the CI gate.

WHAT THIS LAB BUILDS:

A three-service system where:
- `consent-service` (Python 3.10) manages consent grants with an OpenAPI 3.1 spec as the source of truth
- `data-retrieval-service` (Java 17) implements Hexagonal Architecture with swappable institution adapters
- `aggregation-service` (Python 3.10) consumes events published by data-retrieval-service via a lightweight message channel

```
ARCHITECTURE OVERVIEW:

[Citizen Client]
      │
      │ POST /consents (OpenAPI contract)
      ▼
┌─────────────────────┐
│  consent-service    │  Python 3.10
│  (Port 8081)        │  FastAPI + OpenAPI 3.1
│                     │
│  Consent BC         │
│  ConsentAggregate   │
└──────────┬──────────┘
           │ ConsentGranted event
           │ (in-process queue for lab —
           │  Kafka in production)
           ▼
┌─────────────────────────────────────────────────────────────┐
│  data-retrieval-service    Java 17                          │
│  (Port 8080)               Spring Boot 3.x                  │
│                                                             │
│  DataRetrieval BC                                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  APPLICATION CORE (no framework imports)             │  │
│  │  DataRetrievalService                                │  │
│  │  FinancialDataAggregate                              │  │
│  └──────────────┬───────────────────────────────────────┘  │
│                 │                                           │
│  ┌──────────────┴──────────────────────────────────────┐   │
│  │  DRIVEN PORTS (interfaces)                          │   │
│  │  InstitutionDataPort                                │   │
│  │  DataRetrievalRepository                            │   │
│  │  EventPublisherPort                                 │   │
│  └──────────────┬───────────────────────────────────────┘  │
│                 │                                           │
│  ┌──────────────┴──────────────────────────────────────┐   │
│  │  DRIVEN ADAPTERS (implementations)                  │   │
│  │  MockDBSAdapter    (implements InstitutionDataPort) │   │
│  │  MockOCBCAdapter   (implements InstitutionDataPort) │   │
│  │  MockCPFAdapter    (implements InstitutionDataPort) │   │
│  │  InMemoryRepository (implements DataRetrievalRepo)  │   │
│  │  InMemoryEventPublisher (implements EventPublisher) │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │ DataRetrievalCompleted event
                              ▼
┌─────────────────────┐
│  aggregation-service│  Python 3.10
│  (Port 8082)        │  FastAPI
│                     │
│  Aggregation BC     │
│  Consumes events,   │
│  builds unified     │
│  financial summary  │
└─────────────────────┘

CONCEPTS FROM TRAINING THIS LAB DEMONSTRATES:
□ Hexagonal Architecture — Ports and Adapters (Block 1, Section A2)
□ Bounded Context separation — three independent services, three domain models (Block 1, Concept 1)
□ API-First Design — OpenAPI spec written before consent-service code (Block 2)
□ Anti-Corruption Layer — institution adapters translate bank model to domain model (Block 2, Use Case 1)
□ Aggregate Root pattern — ConsentAggregate, FinancialDataAggregate (Block 1, Diagram 3)
□ Published Language — shared event schema between services (Block 2, Use Case 2)

PRODUCTION DELTA:
- Lab uses in-memory event publishing. Production uses Apache Kafka with schema registry.
- Lab uses mock institution adapters. Production uses HTTPS clients with mTLS to real institution endpoints.
- Lab has no authentication. Production uses OAuth 2.0 + PKCE with SingPass as the identity provider.
- Lab uses SQLite via in-memory store. Production uses PostgreSQL with read replicas.
- Azure Free Tier: We use Azure Container Registry (free tier) and Azure Container Apps (free tier allows 180,000 vCPU-seconds/month).
```

---

# L1. ENVIRONMENT SETUP

---

**STEP 1 of 8: Verify Java 17**

WHY: The data-retrieval-service uses Java 17 features — records, sealed classes, text blocks, pattern matching. Java 11 or below will not compile this code.

```bash
$ java -version
```

Expected Output:
```
openjdk version "17.0.10" 2024-01-16
OpenJDK Runtime Environment (build 17.0.10+7)
OpenJDK 64-Bit Server VM (build 17.0.10+7, mixed mode, sharing)
```

⚠️ IF YOU SEE version 11 or below:
```bash
$ sudo apt-get install -y openjdk-17-jdk
$ sudo update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java
```

⚠️ IF YOU SEE "command not found":
```bash
$ sudo apt-get update && sudo apt-get install -y openjdk-17-jdk
```

✅ VERIFY:
```bash
$ javac -version
# Expected: javac 17.0.x
```

---

**STEP 2 of 8: Verify Python 3.10+**

WHY: consent-service and aggregation-service use structural pattern matching (match-case), which requires Python 3.10+. Type hints use the `X | Y` union syntax introduced in 3.10.

```bash
$ python3 --version
```

Expected Output:
```
Python 3.10.12
```

⚠️ IF YOU SEE Python 3.9 or below:
```bash
$ sudo apt-get install -y python3.10 python3.10-venv python3.10-pip
$ sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1
```

✅ VERIFY:
```bash
$ python3 -c "import sys; print(sys.version_info >= (3, 10))"
# Expected: True
```

---

**STEP 3 of 8: Verify Docker and Docker Compose**

WHY: All three services are containerised. Docker Compose orchestrates them as a local development stack — simulating what Kubernetes does in production.

```bash
$ docker --version && docker compose version
```

Expected Output:
```
Docker version 24.0.7, build afdd53b
Docker Compose version v2.21.0
```

⚠️ IF Docker is not installed:
```bash
$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
$ sudo usermod -aG docker $USER && newgrp docker
```

✅ VERIFY:
```bash
$ docker run --rm hello-world | grep "Hello from Docker"
# Expected: Hello from Docker!
```

---

**STEP 4 of 8: Clone the Lab Repository**

WHY: All code is pre-structured to match the architecture diagram. You will fill in the critical sections — the ports, adapters, and domain model.

```bash
$ git clone https://github.com/govfinex-lab/day2-hexagonal-api-first.git
$ cd day2-hexagonal-api-first
$ ls -la
```

Expected Output:
```
drwxr-xr-x  consent-service/
drwxr-xr-x  data-retrieval-service/
drwxr-xr-x  aggregation-service/
drwxr-xr-x  infra/
drwxr-xr-x  contracts/
-rw-r--r--  docker-compose.yml
-rw-r--r--  Makefile
-rw-r--r--  README.md
```

⚠️ IF git clone fails (network restriction): The trainer will provide a USB copy or Azure Storage SAS URL.

✅ VERIFY:
```bash
$ cat Makefile | grep "build-all"
# Expected: build-all: consent data-retrieval aggregation
```

---

**STEP 5 of 8: Install Python dependencies for consent-service**

WHY: FastAPI generates interactive OpenAPI documentation automatically — allowing you to see the API contract live in the browser. Pydantic enforces schema validation at the boundary — this IS the port enforcement in Python.

```bash
$ cd consent-service
$ python3 -m venv venv
$ source venv/bin/activate
$ pip install -r requirements.txt
```

Expected Output:
```
Successfully installed fastapi-0.110.0 pydantic-2.6.0 uvicorn-0.27.0 
httpx-0.27.0 pytest-7.4.0 pytest-asyncio-0.23.0
```

⚠️ IF pip install fails with SSL error:
```bash
$ pip install --trusted-host pypi.org --trusted-host files.pythonhosted.org -r requirements.txt
```

✅ VERIFY:
```bash
$ python3 -c "import fastapi; import pydantic; print('OK')"
# Expected: OK
```

---

**STEP 6 of 8: Build the Java data-retrieval-service**

WHY: Maven downloads all dependencies and compiles. If this step fails, every subsequent step fails. Run this first, fix any issues, then continue.

```bash
$ cd ../data-retrieval-service
$ ./mvnw clean package -DskipTests
```

Expected Output (last 5 lines):
```
[INFO] BUILD SUCCESS
[INFO] Total time:  47.832 s
[INFO] Finished at: 2026-06-12T09:25:11+08:00
[INFO] Final Memory: 42M/512M
```

⚠️ IF you see "Could not resolve dependencies":
```bash
$ ./mvnw clean package -DskipTests -Dmaven.repo.local=.m2
# Uses local repo cache — for offline/restricted network environments
```

⚠️ IF you see "Java version mismatch":
```bash
$ export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
$ ./mvnw clean package -DskipTests
```

✅ VERIFY:
```bash
$ ls target/data-retrieval-service-*.jar
# Expected: target/data-retrieval-service-0.1.0.jar
```

---

**STEP 7 of 8: Install Python dependencies for aggregation-service**

```bash
$ cd ../aggregation-service
$ python3 -m venv venv
$ source venv/bin/activate
$ pip install -r requirements.txt
```

Expected Output:
```
Successfully installed fastapi-0.110.0 pydantic-2.6.0 uvicorn-0.27.0
```

✅ VERIFY:
```bash
$ python3 -c "import fastapi; print('aggregation-service deps OK')"
# Expected: aggregation-service deps OK
```

---

**STEP 8 of 8: Start the full stack**

WHY: Docker Compose starts all three services with their environment variables, port mappings, and inter-service networking. This simulates a local multi-service deployment.

```bash
$ cd ..
$ docker compose up --build
```

Expected Output (last 10 lines):
```
consent-service      | INFO:     Application startup complete.
consent-service      | INFO:     Uvicorn running on http://0.0.0.0:8081
data-retrieval       | INFO  o.s.b.w.e.t.TomcatWebServer - Tomcat started on port(s): 8080
data-retrieval       | INFO  c.g.DataRetrievalApp - Started DataRetrievalApp in 4.821 seconds
aggregation-service  | INFO:     Application startup complete.
aggregation-service  | INFO:     Uvicorn running on http://0.0.0.0:8082
```

⚠️ IF port 8080 already in use:
```bash
$ sudo lsof -ti:8080 | xargs kill -9
$ docker compose up --build
```

✅ VERIFY:
```bash
$ curl -s http://localhost:8081/health | python3 -m json.tool
# Expected: {"status": "healthy", "service": "consent-service", "version": "0.1.0"}

$ curl -s http://localhost:8080/actuator/health | python3 -m json.tool
# Expected: {"status": "UP"}

$ curl -s http://localhost:8082/health | python3 -m json.tool
# Expected: {"status": "healthy", "service": "aggregation-service"}
```

---

# L2. PROJECT STRUCTURE

---

```
day2-hexagonal-api-first/
│
├── contracts/                              # API-First: contracts live here FIRST
│   ├── openapi/
│   │   └── consent-api-v1.yaml            # OpenAPI 3.1 spec — written before code
│   └── asyncapi/
│       └── financial-events-v1.yaml       # AsyncAPI 3.0 — event Published Language
│
├── consent-service/                        # Bounded Context: Consent
│   ├── src/
│   │   ├── domain/
│   │   │   ├── __init__.py
│   │   │   ├── consent.py                 # ConsentAggregate — pure domain logic
│   │   │   └── events.py                  # Domain events
│   │   ├── application/
│   │   │   ├── __init__.py
│   │   │   └── consent_service.py         # Application service (use cases)
│   │   ├── ports/
│   │   │   ├── __init__.py
│   │   │   ├── consent_repository.py      # Driven port: persistence interface
│   │   │   └── event_publisher.py         # Driven port: event publishing interface
│   │   ├── adapters/
│   │   │   ├── __init__.py
│   │   │   ├── in_memory_repository.py    # Driven adapter: in-memory store
│   │   │   └── in_memory_publisher.py     # Driven adapter: in-memory event bus
│   │   └── api/
│   │       ├── __init__.py
│   │       ├── router.py                  # Driving adapter: FastAPI REST
│   │       └── schemas.py                 # API DTOs (not domain objects)
│   ├── tests/
│   │   ├── test_domain.py                 # Tests core with zero infrastructure
│   │   └── test_api.py                    # Integration tests with test adapters
│   ├── main.py                            # Application entry point
│   └── requirements.txt
│
├── data-retrieval-service/                 # Bounded Context: DataRetrieval
│   ├── src/
│   │   └── main/
│   │       └── java/
│   │           └── com/govfinex/retrieval/
│   │               ├── domain/
│   │               │   ├── FinancialData.java          # Aggregate Root
│   │               │   ├── AccountSummary.java         # Entity
│   │               │   ├── InstitutionId.java          # Value Object
│   │               │   └── Money.java                  # Value Object
│   │               ├── application/
│   │               │   ├── port/
│   │               │   │   ├── in/
│   │               │   │   │   └── RetrieveFinancialDataUseCase.java  # Driving port
│   │               │   │   └── out/
│   │               │   │       ├── InstitutionDataPort.java           # Driven port
│   │               │   │       ├── DataRetrievalRepository.java       # Driven port
│   │               │   │       └── EventPublisherPort.java            # Driven port
│   │               │   └── DataRetrievalService.java   # Application service
│   │               ├── adapter/
│   │               │   ├── in/
│   │               │   │   └── web/
│   │               │   │       └── DataRetrievalController.java       # Driving adapter
│   │               │   └── out/
│   │               │       ├── institution/
│   │               │       │   ├── MockDBSAdapter.java                # Driven adapter
│   │               │       │   ├── MockOCBCAdapter.java               # Driven adapter
│   │               │       │   └── MockCPFAdapter.java                # Driven adapter
│   │               │       ├── persistence/
│   │               │       │   └── InMemoryDataRetrievalRepository.java
│   │               │       └── messaging/
│   │               │           └── InMemoryEventPublisher.java
│   │               └── DataRetrievalApp.java
│   ├── src/test/java/com/govfinex/retrieval/
│   │   ├── domain/
│   │   │   └── FinancialDataTest.java     # Pure domain tests — zero Spring context
│   │   └── application/
│   │       └── DataRetrievalServiceTest.java  # Tests with fake adapters
│   ├── pom.xml
│   └── Dockerfile
│
├── aggregation-service/                    # Bounded Context: Aggregation
│   ├── src/
│   │   ├── domain/
│   │   │   └── financial_summary.py        # FinancialSummary — aggregation model
│   │   ├── application/
│   │   │   └── aggregation_service.py      # Aggregates multi-institution data
│   │   ├── ports/
│   │   │   └── event_consumer.py           # Driven port: event consumption
│   │   ├── adapters/
│   │   │   └── in_memory_consumer.py       # Driven adapter: reads from shared bus
│   │   └── api/
│   │       ├── router.py
│   │       └── schemas.py
│   ├── main.py
│   └── requirements.txt
│
├── infra/
│   └── terraform/
│       ├── main.tf                         # Azure Container Apps (free tier)
│       ├── variables.tf
│       ├── outputs.tf
│       └── providers.tf
│
├── docker-compose.yml                      # Local development orchestration
├── Makefile                                # make build, make test, make demo
└── README.md
```

---

# L3. CODE BLOCKS

---

## CONTRACT FIRST — OpenAPI Specification

```
FILE: contracts/openapi/consent-api-v1.yaml
LANGUAGE: YAML (OpenAPI 3.1)
PURPOSE: The API contract written BEFORE any consent-service code.
         This is the source of truth. The FastAPI code is generated FROM this.
CONCEPTS DEMONSTRATED: API-First Design (Block 2), Published Language (Block 2, Use Case 1)
COPY-PASTE READY: YES
PRODUCTION DELTA: Would include OAuth2/PKCE security schemes, rate limiting headers,
                  Singapore NRIC/FIN validation patterns, MAS TRM audit headers
```

```yaml
openapi: "3.1.0"

info:
  title: GovFinEx Consent API
  version: "1.0.0"
  description: |
    Manages citizen consent grants for financial data retrieval.
    Inspired by Singapore SGFinDex consent model.
    
    VERSIONING POLICY:
    - Major version: Breaking change. 12-month parallel operation guaranteed.
    - Minor version: Backward compatible additions only.
    - No field removal without major version increment.
    
    COMPLIANCE: Personal Data Protection Act (Singapore) — consent is
    explicit, purpose-specific, and revocable at any time.
  contact:
    name: GovFinEx Platform Team
    email: platform@govfinex.gov.sg

servers:
  - url: http://localhost:8081
    description: Local development
  - url: https://api.govfinex.gov.sg/consent/v1
    description: Production (Singapore region)

paths:
  /consents:
    post:
      operationId: createConsent
      summary: Create a new consent grant
      description: |
        Citizen explicitly consents to data retrieval from specified
        financial institutions for a defined purpose and duration.
        Each consent is scoped to specific data categories.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateConsentRequest"
            example:
              citizenId: "S1234567A"
              institutions: ["DBS", "OCBC", "CPF"]
              dataCategories: ["ACCOUNT_BALANCE", "CPF_CONTRIBUTIONS"]
              purposeCode: "FINANCIAL_PLANNING"
              durationDays: 30
      responses:
        "201":
          description: Consent created successfully
          headers:
            # Idempotency key returned for deduplication
            X-Idempotency-Key:
              schema:
                type: string
              description: Echo of request idempotency key
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ConsentResponse"
        "400":
          description: Invalid consent request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "409":
          description: Active consent already exists for this citizen and purpose
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
      parameters:
        # Idempotency header — critical for financial/government APIs
        # Prevents duplicate consent grants on network retry
        - name: X-Idempotency-Key
          in: header
          required: true
          schema:
            type: string
            format: uuid
          description: Client-generated UUID for idempotent submission

  /consents/{consentId}:
    get:
      operationId: getConsent
      summary: Retrieve consent details
      parameters:
        - name: consentId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ConsentResponse"
          description: Consent found
        "404":
          description: Consent not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"

  /consents/{consentId}/revoke:
    post:
      operationId: revokeConsent
      summary: Revoke an active consent
      description: |
        PDPA requirement: citizen can revoke consent at any time.
        Revoked consents are retained for audit purposes but
        no further data retrieval is permitted.
      parameters:
        - name: consentId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          description: Consent revoked
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ConsentResponse"
        "404":
          description: Consent not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "409":
          description: Consent already revoked or expired
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"

  /health:
    get:
      operationId: healthCheck
      summary: Service health check
      responses:
        "200":
          description: Service healthy
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/HealthResponse"

components:
  schemas:
    CreateConsentRequest:
      type: object
      required:
        - citizenId
        - institutions
        - dataCategories
        - purposeCode
        - durationDays
      properties:
        citizenId:
          type: string
          description: |
            Citizen identifier. In production: SingPass-issued pseudonymous ID.
            PDPA: This is personal data. Stored encrypted at rest.
          example: "S1234567A"
        institutions:
          type: array
          items:
            type: string
            enum: ["DBS", "OCBC", "UOB", "CPF", "IRAS", "HDB"]
          minItems: 1
          description: Financial institutions consented to
        dataCategories:
          type: array
          items:
            type: string
            enum:
              - ACCOUNT_BALANCE
              - TRANSACTION_HISTORY
              - CPF_CONTRIBUTIONS
              - TAX_ASSESSMENT
              - LOAN_DETAILS
          minItems: 1
          description: |
            Data minimisation enforcement: citizen consents only to
            specific categories, not all data. PDPA compliance.
        purposeCode:
          type: string
          enum:
            - FINANCIAL_PLANNING
            - LOAN_APPLICATION
            - INSURANCE_UNDERWRITING
          description: Purpose limitation — consent is bound to declared purpose
        durationDays:
          type: integer
          minimum: 1
          maximum: 90
          description: Consent validity period. Max 90 days — MAS guidance.

    ConsentResponse:
      type: object
      properties:
        consentId:
          type: string
          format: uuid
        citizenId:
          type: string
        status:
          type: string
          enum: ["ACTIVE", "REVOKED", "EXPIRED"]
        institutions:
          type: array
          items:
            type: string
        dataCategories:
          type: array
          items:
            type: string
        purposeCode:
          type: string
        createdAt:
          type: string
          format: date-time
        expiresAt:
          type: string
          format: date-time
        revokedAt:
          type: string
          format: date-time
          nullable: true

    ErrorResponse:
      type: object
      required:
        - code
        - message
        - traceId
      properties:
        code:
          type: string
          example: "CONSENT_ALREADY_EXISTS"
        message:
          type: string
          example: "An active consent already exists for this citizen and purpose"
        traceId:
          type: string
          format: uuid
          description: Correlates to distributed trace for debugging

    HealthResponse:
      type: object
      properties:
        status:
          type: string
          enum: ["healthy", "degraded", "unhealthy"]
        service:
          type: string
        version:
          type: string
```

---

## CONSENT SERVICE — Domain Layer (Python 3.10)

```
FILE: consent-service/src/domain/consent.py
LANGUAGE: Python 3.10
PURPOSE: ConsentAggregate — pure domain logic with zero framework dependencies.
         Tests run in milliseconds. No database. No HTTP. No FastAPI.
CONCEPTS DEMONSTRATED: Aggregate Root (Block 1, Diagram 3), Bounded Context domain model
COPY-PASTE READY: YES
PRODUCTION DELTA: Would add encryption for citizenId field at domain layer,
                  more sophisticated invariant checks per MAS TRM data classification
```

```python
# consent-service/src/domain/consent.py
#
# ARCHITECTURAL NOTE:
# This file has ZERO imports from FastAPI, SQLAlchemy, Redis, or any
# infrastructure library. This is intentional and non-negotiable.
# The domain model must be testable with: python3 -m pytest tests/test_domain.py
# That test runs in < 50ms with zero external dependencies.
# If you add 'import fastapi' here, you have violated Hexagonal Architecture.

from __future__ import annotations

import uuid
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import final


class ConsentStatus(Enum):
    ACTIVE = "ACTIVE"
    REVOKED = "REVOKED"
    EXPIRED = "EXPIRED"


class DataCategory(Enum):
    ACCOUNT_BALANCE = "ACCOUNT_BALANCE"
    TRANSACTION_HISTORY = "TRANSACTION_HISTORY"
    CPF_CONTRIBUTIONS = "CPF_CONTRIBUTIONS"
    TAX_ASSESSMENT = "TAX_ASSESSMENT"
    LOAN_DETAILS = "LOAN_DETAILS"


class PurposeCode(Enum):
    FINANCIAL_PLANNING = "FINANCIAL_PLANNING"
    LOAN_APPLICATION = "LOAN_APPLICATION"
    INSURANCE_UNDERWRITING = "INSURANCE_UNDERWRITING"


# Domain exception — not an HTTP exception, not a framework exception.
# The adapter layer translates this to HTTP 409.
class ConsentInvariantViolation(Exception):
    """Raised when a domain invariant is violated."""
    pass


class ConsentAlreadyRevokedException(ConsentInvariantViolation):
    """Consent cannot be revoked if already revoked or expired."""
    pass


class ConsentExpiredException(ConsentInvariantViolation):
    """Consent cannot be used after expiry."""
    pass


@dataclass
class ConsentId:
    """
    Value Object — identity of a consent grant.
    Immutable. Equality based on value, not reference.
    In DDD: Value Objects have no identity of their own.
    """
    value: str = field(default_factory=lambda: str(uuid.uuid4()))

    def __post_init__(self) -> None:
        # Validate UUID format — fail fast at construction
        try:
            uuid.UUID(self.value)
        except ValueError:
            raise ConsentInvariantViolation(
                f"ConsentId must be a valid UUID, got: {self.value}"
            )

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, ConsentId):
            return False
        return self.value == other.value

    def __hash__(self) -> int:
        return hash(self.value)


@dataclass
class ConsentDuration:
    """
    Value Object — duration of consent validity.
    MAS guidance: maximum 90 days for financial data access consent.
    Business rule lives here, not in the API layer.
    """
    days: int

    # MAS-aligned maximum — if business rules change, change here only
    MAX_DURATION_DAYS: int = field(default=90, init=False, repr=False)

    def __post_init__(self) -> None:
        if self.days < 1:
            raise ConsentInvariantViolation(
                "Consent duration must be at least 1 day"
            )
        if self.days > self.MAX_DURATION_DAYS:
            raise ConsentInvariantViolation(
                f"Consent duration cannot exceed {self.MAX_DURATION_DAYS} days "
                f"per MAS financial data access guidelines"
            )

    def expiry_from(self, start: datetime) -> datetime:
        return start + timedelta(days=self.days)


@final
class ConsentAggregate:
    """
    AGGREGATE ROOT: ConsentAggregate

    Enforces ALL invariants for the Consent Bounded Context.
    External code interacts ONLY through the public methods below.
    Direct attribute mutation is prohibited — use the methods.

    Domain invariants enforced:
    1. An active consent cannot be re-activated once revoked
    2. Consent duration cannot exceed 90 days (MAS guidance)
    3. Data categories cannot be expanded after consent is granted
       (PDPA: scope cannot be widened without new consent)
    4. A revoked consent retains its history for audit purposes
    """

    def __init__(
        self,
        consent_id: ConsentId,
        citizen_id: str,
        institutions: frozenset[str],
        data_categories: frozenset[DataCategory],
        purpose_code: PurposeCode,
        duration: ConsentDuration,
        created_at: datetime | None = None,
    ) -> None:
        # Use frozenset — categories cannot be mutated after construction
        # This enforces the PDPA invariant: scope cannot expand without new consent
        self._consent_id = consent_id
        self._citizen_id = citizen_id
        self._institutions = institutions
        self._data_categories = data_categories
        self._purpose_code = purpose_code
        self._duration = duration
        self._created_at = created_at or datetime.now(timezone.utc)
        self._expires_at = duration.expiry_from(self._created_at)
        self._status = ConsentStatus.ACTIVE
        self._revoked_at: datetime | None = None

        # Domain events — collected here, published by the application service
        # The domain does not know HOW events are published (Kafka/HTTP/memory)
        # That is the EventPublisherPort's concern
        self._domain_events: list[dict] = []
        self._domain_events.append(self._build_granted_event())

    def revoke(self, revoked_at: datetime | None = None) -> None:
        """
        COMMAND: Revoke this consent grant.

        PDPA requirement: Citizens must be able to revoke at any time.
        Domain invariant: Cannot revoke what is already revoked or expired.
        """
        match self._status:
            case ConsentStatus.REVOKED:
                raise ConsentAlreadyRevokedException(
                    f"Consent {self._consent_id.value} is already revoked"
                )
            case ConsentStatus.EXPIRED:
                raise ConsentAlreadyRevokedException(
                    f"Consent {self._consent_id.value} has expired — "
                    f"nothing to revoke"
                )
            case ConsentStatus.ACTIVE:
                self._revoked_at = revoked_at or datetime.now(timezone.utc)
                self._status = ConsentStatus.REVOKED
                self._domain_events.append(self._build_revoked_event())

    def check_active(self) -> None:
        """
        QUERY: Validates consent is usable for data retrieval.
        Raises if expired or revoked — called by DataRetrieval BC before fetch.
        """
        now = datetime.now(timezone.utc)

        if self._status == ConsentStatus.REVOKED:
            raise ConsentAlreadyRevokedException(
                f"Consent {self._consent_id.value} has been revoked"
            )

        if now > self._expires_at:
            # Auto-transition to EXPIRED on check — lazy state update
            self._status = ConsentStatus.EXPIRED
            raise ConsentExpiredException(
                f"Consent {self._consent_id.value} expired at {self._expires_at}"
            )

    def allows_institution(self, institution: str) -> bool:
        """Query: Does this consent cover the given institution?"""
        return institution in self._institutions

    def allows_category(self, category: DataCategory) -> bool:
        """Query: Does this consent cover the given data category?"""
        return category in self._data_categories

    def pop_domain_events(self) -> list[dict]:
        """
        Called by Application Service after persisting the aggregate.
        Events are consumed once — prevents double-publishing.
        This pattern: Outbox Pattern (simplified for lab).
        """
        events = list(self._domain_events)
        self._domain_events.clear()
        return events

    # Read-only properties — external code cannot mutate state
    @property
    def consent_id(self) -> ConsentId:
        return self._consent_id

    @property
    def citizen_id(self) -> str:
        return self._citizen_id

    @property
    def status(self) -> ConsentStatus:
        return self._status

    @property
    def institutions(self) -> frozenset[str]:
        return self._institutions

    @property
    def data_categories(self) -> frozenset[DataCategory]:
        return self._data_categories

    @property
    def purpose_code(self) -> PurposeCode:
        return self._purpose_code

    @property
    def created_at(self) -> datetime:
        return self._created_at

    @property
    def expires_at(self) -> datetime:
        return self._expires_at

    @property
    def revoked_at(self) -> datetime | None:
        return self._revoked_at

    def _build_granted_event(self) -> dict:
        return {
            "eventType": "ConsentGranted",
            "consentId": self._consent_id.value,
            "citizenId": self._citizen_id,
            "institutions": list(self._institutions),
            "dataCategories": [c.value for c in self._data_categories],
            "purposeCode": self._purpose_code.value,
            "expiresAt": self._expires_at.isoformat(),
            "occurredAt": self._created_at.isoformat(),
        }

    def _build_revoked_event(self) -> dict:
        return {
            "eventType": "ConsentRevoked",
            "consentId": self._consent_id.value,
            "citizenId": self._citizen_id,
            "occurredAt": self._revoked_at.isoformat(),
        }
```

---

## CONSENT SERVICE — Ports (Python 3.10)

```
FILE: consent-service/src/ports/consent_repository.py
LANGUAGE: Python 3.10
PURPOSE: Driven port — abstract interface for persistence.
         The domain and application layer depend on THIS, not on SQLAlchemy or PostgreSQL.
CONCEPTS DEMONSTRATED: Hexagonal Architecture — Driven Ports (Block 1, Diagram 1)
COPY-PASTE READY: YES
PRODUCTION DELTA: Would add async methods, pagination, optimistic locking version field
```

```python
# consent-service/src/ports/consent_repository.py
#
# ARCHITECTURAL NOTE:
# This is a PORT — an interface that the application core depends on.
# Notice: zero infrastructure imports. This file compiles and runs
# with no database installed on the machine.
# The InMemoryConsentRepository and (in production) PostgresConsentRepository
# both implement this interface. The application service never knows which.

from abc import ABC, abstractmethod
from typing import Optional

from src.domain.consent import ConsentAggregate, ConsentId


class ConsentRepository(ABC):
    """
    Driven Port: Persistence abstraction for ConsentAggregate.

    Implementations:
    - InMemoryConsentRepository (lab/test)
    - PostgresConsentRepository (production)
    - RedisConsentRepository (if consent lookup speed is critical)

    The application service imports THIS class, not any implementation.
    Dependency injection wires the correct implementation at startup.
    """

    @abstractmethod
    def save(self, consent: ConsentAggregate) -> None:
        """
        Persist a consent aggregate.
        Upsert semantics: creates or updates.
        Must be idempotent — safe to call multiple times with same consent.
        """
        ...

    @abstractmethod
    def find_by_id(self, consent_id: ConsentId) -> Optional[ConsentAggregate]:
        """
        Retrieve a consent by its identity.
        Returns None if not found — caller handles the not-found case.
        Never raises for not-found — that is the caller's domain decision.
        """
        ...

    @abstractmethod
    def find_active_by_citizen_and_purpose(
        self,
        citizen_id: str,
        purpose_code: str
    ) -> Optional[ConsentAggregate]:
        """
        Business query: does this citizen already have an active consent
        for this purpose? Used to enforce the 409 Conflict response.
        """
        ...
```

```
FILE: consent-service/src/ports/event_publisher.py
LANGUAGE: Python 3.10
PURPOSE: Driven port — abstract interface for event publishing.
CONCEPTS DEMONSTRATED: Hexagonal Architecture — Driven Ports, Published Language pattern
COPY-PASTE READY: YES
PRODUCTION DELTA: Would use Kafka with Avro schema registry in production
```

```python
# consent-service/src/ports/event_publisher.py
#
# ARCHITECTURAL NOTE:
# This port decouples the domain from the messaging infrastructure.
# In the lab: InMemoryEventPublisher (events stored in a list).
# In production: KafkaEventPublisher (events sent to Kafka topic).
# The application service calls publish_event() — it never knows
# whether events go to Kafka, RabbitMQ, or a test list.

from abc import ABC, abstractmethod


class EventPublisher(ABC):
    """
    Driven Port: Event publishing abstraction.

    The domain emits events. This port publishes them.
    Implementations: InMemoryEventPublisher, KafkaEventPublisher,
                     SNSEventPublisher (AWS), ServiceBusPublisher (Azure)
    """

    @abstractmethod
    def publish(self, event: dict) -> None:
        """
        Publish a domain event to the event channel.
        The event dict follows the Published Language schema
        defined in contracts/asyncapi/financial-events-v1.yaml.
        """
        ...

    @abstractmethod
    def get_published_events(self) -> list[dict]:
        """
        TEST SUPPORT: Retrieve all published events.
        Implemented only in InMemoryEventPublisher.
        Production implementation raises NotImplementedError.
        This makes test-support intent explicit.
        """
        ...
```

---

## CONSENT SERVICE — Adapters (Python 3.10)

```
FILE: consent-service/src/adapters/in_memory_repository.py
LANGUAGE: Python 3.10
PURPOSE: Driven adapter — in-memory implementation of ConsentRepository port.
         Used in lab and tests. Swap for PostgresAdapter in production.
CONCEPTS DEMONSTRATED: Hexagonal Architecture — Driven Adapters
COPY-PASTE READY: YES
PRODUCTION DELTA: Replace with SQLAlchemy async PostgreSQL adapter.
                  Add connection pooling, retry logic, circuit breaker.
```

```python
# consent-service/src/adapters/in_memory_repository.py

from typing import Optional

from src.domain.consent import ConsentAggregate, ConsentId, ConsentStatus
from src.ports.consent_repository import ConsentRepository


class InMemoryConsentRepository(ConsentRepository):
    """
    Driven Adapter: In-memory storage for ConsentAggregate.

    ARCHITECTURAL NOTE:
    This class knows about Python dicts and lists.
    The APPLICATION CORE knows nothing about this class.
    The APPLICATION CORE depends only on ConsentRepository (the port).

    If we swap this for PostgresConsentRepository:
    - Zero changes to domain layer
    - Zero changes to application service
    - Zero changes to API layer
    - Only change: dependency injection wiring in main.py

    That is the entire value proposition of Hexagonal Architecture.
    """

    def __init__(self) -> None:
        # Key: consent_id string → ConsentAggregate
        self._store: dict[str, ConsentAggregate] = {}

    def save(self, consent: ConsentAggregate) -> None:
        self._store[consent.consent_id.value] = consent

    def find_by_id(self, consent_id: ConsentId) -> Optional[ConsentAggregate]:
        return self._store.get(consent_id.value)

    def find_active_by_citizen_and_purpose(
        self,
        citizen_id: str,
        purpose_code: str
    ) -> Optional[ConsentAggregate]:
        # Linear scan — acceptable for lab (hundreds of records)
        # Production: indexed query on (citizen_id, purpose_code, status)
        for consent in self._store.values():
            if (
                consent.citizen_id == citizen_id
                and consent.purpose_code.value == purpose_code
                and consent.status == ConsentStatus.ACTIVE
            ):
                return consent
        return None
```

```
FILE: consent-service/src/adapters/in_memory_publisher.py
LANGUAGE: Python 3.10
PURPOSE: Driven adapter — in-memory event publisher. Stores events in a list.
         Aggregation service polls this shared list (simulating message bus).
CONCEPTS DEMONSTRATED: Hexagonal Architecture — Driven Adapters, Published Language
COPY-PASTE READY: YES
PRODUCTION DELTA: Replace with KafkaEventPublisher using confluent-kafka-python
```

```python
# consent-service/src/adapters/in_memory_publisher.py

import json
from datetime import datetime, timezone

from src.ports.event_publisher import EventPublisher


# Shared in-memory event bus — simulates Kafka topic for lab purposes
# In production: this would be a Kafka topic or Azure Service Bus topic
# IMPORTANT: This is NOT thread-safe for high concurrency. Lab only.
_EVENT_BUS: list[dict] = []


class InMemoryEventPublisher(EventPublisher):
    """
    Driven Adapter: Publishes events to an in-memory list.

    ARCHITECTURAL NOTE:
    The Published Language for events is defined in:
    contracts/asyncapi/financial-events-v1.yaml

    Every event published here MUST conform to that schema.
    In production, schema validation would be enforced by
    the Kafka Schema Registry at publish time.
    """

    def publish(self, event: dict) -> None:
        # Add publish metadata — in production, Kafka adds this automatically
        enriched_event = {
            **event,
            "publishedAt": datetime.now(timezone.utc).isoformat(),
            "source": "consent-service",
            "specVersion": "1.0",
        }
        _EVENT_BUS.append(enriched_event)

        # Structured logging — in production: correlate with trace ID
        print(
            json.dumps({
                "level": "INFO",
                "message": "Domain event published",
                "eventType": event.get("eventType"),
                "consentId": event.get("consentId"),
            })
        )

    def get_published_events(self) -> list[dict]:
        """Returns a snapshot of all published events. For tests and lab demo."""
        return list(_EVENT_BUS)


def get_event_bus() -> list[dict]:
    """
    Module-level access to the shared event bus.
    Aggregation service imports this to poll for new events.
    Production equivalent: Kafka consumer group polling a topic.
    """
    return _EVENT_BUS
```

---

## CONSENT SERVICE — Application Service (Python 3.10)

```
FILE: consent-service/src/application/consent_service.py
LANGUAGE: Python 3.10
PURPOSE: Application service — orchestrates domain objects and ports.
         Contains use case logic. No HTTP, no DB — only domain + ports.
CONCEPTS DEMONSTRATED: Application Service layer, Use Case orchestration
COPY-PASTE READY: YES
PRODUCTION DELTA: Add distributed tracing (OpenTelemetry), async/await,
                  idempotency key deduplication at application layer
```

```python
# consent-service/src/application/consent_service.py
#
# ARCHITECTURAL NOTE:
# The Application Service sits BETWEEN the driving adapter (FastAPI router)
# and the domain. It:
# 1. Calls the domain to enforce business rules
# 2. Uses ports to persist and publish
# 3. Has NO knowledge of HTTP, JSON, or databases
#
# The Application Service is where you ADD orchestration logic
# (e.g., "after creating consent, also notify the citizen via SMS")
# WITHOUT polluting the domain with that concern.

from dataclasses import dataclass
from datetime import datetime, timezone

from src.domain.consent import (
    ConsentAggregate,
    ConsentDuration,
    ConsentId,
    ConsentInvariantViolation,
    DataCategory,
    PurposeCode,
)
from src.ports.consent_repository import ConsentRepository
from src.ports.event_publisher import EventPublisher


# Command object — what the driving adapter sends to the application service
# Note: NOT a Pydantic model. Not an API DTO. Pure Python dataclass.
# The API layer translates its DTO into this command.
@dataclass(frozen=True)
class CreateConsentCommand:
    citizen_id: str
    institutions: frozenset[str]
    data_categories: frozenset[DataCategory]
    purpose_code: PurposeCode
    duration_days: int
    idempotency_key: str


@dataclass(frozen=True)
class RevokeConsentCommand:
    consent_id: str


class ConsentAlreadyExistsError(Exception):
    """Raised when an active consent already exists for citizen + purpose."""
    pass


class ConsentNotFoundError(Exception):
    """Raised when a consent cannot be found by ID."""
    pass


class ConsentApplicationService:
    """
    Application Service: Consent Use Cases

    Dependencies injected at construction — never imported directly.
    This is what makes the application service testable with fake adapters.
    """

    def __init__(
        self,
        consent_repository: ConsentRepository,
        event_publisher: EventPublisher,
    ) -> None:
        # Port interfaces — not concrete implementations
        # The service never knows if this is Postgres or in-memory
        self._repository = consent_repository
        self._publisher = event_publisher

    def create_consent(self, command: CreateConsentCommand) -> ConsentAggregate:
        """
        USE CASE: Create Consent Grant

        Flow:
        1. Check for duplicate active consent (idempotency)
        2. Construct domain aggregate (validates invariants)
        3. Persist via repository port
        4. Publish domain events via publisher port
        5. Return aggregate to driving adapter
        """

        # Step 1: Idempotency — check for existing active consent
        existing = self._repository.find_active_by_citizen_and_purpose(
            citizen_id=command.citizen_id,
            purpose_code=command.purpose_code.value,
        )
        if existing is not None:
            raise ConsentAlreadyExistsError(
                f"Active consent already exists for citizen "
                f"'{command.citizen_id}' with purpose "
                f"'{command.purpose_code.value}'"
            )

        # Step 2: Construct aggregate — domain enforces all invariants here
        # If duration > 90 days, ConsentDuration raises ConsentInvariantViolation
        consent = ConsentAggregate(
            consent_id=ConsentId(),
            citizen_id=command.citizen_id,
            institutions=command.institutions,
            data_categories=command.data_categories,
            purpose_code=command.purpose_code,
            duration=ConsentDuration(days=command.duration_days),
            created_at=datetime.now(timezone.utc),
        )

        # Step 3: Persist
        self._repository.save(consent)

        # Step 4: Publish domain events (e.g., ConsentGranted)
        # Events were collected in the aggregate during construction
        for event in consent.pop_domain_events():
            self._publisher.publish(event)

        return consent

    def revoke_consent(self, command: RevokeConsentCommand) -> ConsentAggregate:
        """USE CASE: Revoke Consent"""

        consent = self._repository.find_by_id(ConsentId(command.consent_id))
        if consent is None:
            raise ConsentNotFoundError(
                f"Consent '{command.consent_id}' not found"
            )

        # Domain enforces revocation invariants (cannot revoke expired/revoked)
        consent.revoke()
        self._repository.save(consent)

        for event in consent.pop_domain_events():
            self._publisher.publish(event)

        return consent

    def get_consent(self, consent_id: str) -> ConsentAggregate:
        """USE CASE: Retrieve Consent Details"""

        consent = self._repository.find_by_id(ConsentId(consent_id))
        if consent is None:
            raise ConsentNotFoundError(f"Consent '{consent_id}' not found")
        return consent
```

---

## CONSENT SERVICE — API Adapter (Python 3.10)

```
FILE: consent-service/src/api/router.py
LANGUAGE: Python 3.10
PURPOSE: Driving adapter — FastAPI REST controller.
         Translates HTTP requests into application commands.
         Translates domain exceptions into HTTP responses.
CONCEPTS DEMONSTRATED: Hexagonal Architecture — Driving Adapter
COPY-PASTE READY: YES
PRODUCTION DELTA: Add OAuth2 dependency, request logging middleware,
                  rate limiting, OpenTelemetry trace injection
```

```python
# consent-service/src/api/router.py
#
# ARCHITECTURAL NOTE:
# This is the DRIVING ADAPTER. It knows about HTTP. It knows about FastAPI.
# But it does NOT contain business logic.
# Business logic lives in ConsentApplicationService and ConsentAggregate.
#
# The translation pattern here:
# HTTP Request → API DTO (schemas.py) → Domain Command → Application Service
# Application Service → Domain Aggregate → API DTO (schemas.py) → HTTP Response
#
# Domain exceptions are caught here and translated to HTTP status codes.
# The domain never throws HTTP exceptions.

import uuid
from fastapi import APIRouter, HTTPException, Header, status

from src.api.schemas import (
    CreateConsentRequestDTO,
    ConsentResponseDTO,
    HealthResponseDTO,
)
from src.application.consent_service import (
    ConsentApplicationService,
    ConsentAlreadyExistsError,
    ConsentNotFoundError,
    CreateConsentCommand,
    RevokeConsentCommand,
)
from src.domain.consent import (
    ConsentInvariantViolation,
    DataCategory,
    PurposeCode,
)

router = APIRouter()


def _build_consent_response(consent) -> ConsentResponseDTO:
    """
    Private translator: ConsentAggregate → ConsentResponseDTO
    Exists here, not in the domain, because DTOs are an API concern.
    """
    return ConsentResponseDTO(
        consentId=consent.consent_id.value,
        citizenId=consent.citizen_id,
        status=consent.status.value,
        institutions=list(consent.institutions),
        dataCategories=[c.value for c in consent.data_categories],
        purposeCode=consent.purpose_code.value,
        createdAt=consent.created_at.isoformat(),
        expiresAt=consent.expires_at.isoformat(),
        revokedAt=consent.revoked_at.isoformat() if consent.revoked_at else None,
    )


def create_router(service: ConsentApplicationService) -> APIRouter:
    """
    Factory function: creates router with injected application service.
    Enables testing with a fake service implementation.
    """

    @router.post(
        "/consents",
        status_code=status.HTTP_201_CREATED,
        response_model=ConsentResponseDTO,
        summary="Create a new consent grant",
    )
    def create_consent(
        request: CreateConsentRequestDTO,
        x_idempotency_key: str = Header(..., alias="X-Idempotency-Key"),
    ) -> ConsentResponseDTO:

        # Validate idempotency key format
        try:
            uuid.UUID(x_idempotency_key)
        except ValueError:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail={
                    "code": "INVALID_IDEMPOTENCY_KEY",
                    "message": "X-Idempotency-Key must be a valid UUID",
                    "traceId": str(uuid.uuid4()),
                },
            )

        # Translate API DTO to domain command
        # This translation is the driving adapter's responsibility
        try:
            command = CreateConsentCommand(
                citizen_id=request.citizenId,
                institutions=frozenset(request.institutions),
                data_categories=frozenset(
                    DataCategory(cat) for cat in request.dataCategories
                ),
                purpose_code=PurposeCode(request.purposeCode),
                duration_days=request.durationDays,
                idempotency_key=x_idempotency_key,
            )
        except ValueError as e:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail={
                    "code": "INVALID_ENUM_VALUE",
                    "message": str(e),
                    "traceId": str(uuid.uuid4()),
                },
            )

        # Call application service — domain exceptions translated here
        try:
            consent = service.create_consent(command)
            return _build_consent_response(consent)

        except ConsentAlreadyExistsError as e:
            # Domain rule violation → HTTP 409 Conflict
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail={
                    "code": "CONSENT_ALREADY_EXISTS",
                    "message": str(e),
                    "traceId": str(uuid.uuid4()),
                },
            )
        except ConsentInvariantViolation as e:
            # Domain invariant violation → HTTP 400 Bad Request
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail={
                    "code": "INVARIANT_VIOLATION",
                    "message": str(e),
                    "traceId": str(uuid.uuid4()),
                },
            )

    @router.get(
        "/consents/{consent_id}",
        response_model=ConsentResponseDTO,
        summary="Retrieve consent details",
    )
    def get_consent(consent_id: str) -> ConsentResponseDTO:
        try:
            consent = service.get_consent(consent_id)
            return _build_consent_response(consent)
        except ConsentNotFoundError:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail={
                    "code": "CONSENT_NOT_FOUND",
                    "message": f"Consent '{consent_id}' not found",
                    "traceId": str(uuid.uuid4()),
                },
            )

    @router.post(
        "/consents/{consent_id}/revoke",
        response_model=ConsentResponseDTO,
        summary="Revoke an active consent",
    )
    def revoke_consent(consent_id: str) -> ConsentResponseDTO:
        try:
            consent = service.revoke_consent(RevokeConsentCommand(consent_id))
            return _build_consent_response(consent)
        except ConsentNotFoundError:
            raise HTTPException(status_code=404, detail="Consent not found")
        except ConsentInvariantViolation as e:
            raise HTTPException(status_code=409, detail=str(e))

    @router.get("/health", response_model=HealthResponseDTO)
    def health() -> HealthResponseDTO:
        return HealthResponseDTO(
            status="healthy",
            service="consent-service",
            version="0.1.0"
        )

    return router
```

---

## DATA RETRIEVAL SERVICE — Hexagonal Architecture (Java 17)

```
FILE: data-retrieval-service/src/main/java/com/govfinex/retrieval/domain/FinancialData.java
LANGUAGE: Java 17
PURPOSE: Aggregate Root — FinancialData. Pure domain logic, zero Spring imports.
CONCEPTS DEMONSTRATED: Aggregate Root, Java 17 records for Value Objects,
                       sealed classes for domain hierarchy
COPY-PASTE READY: YES
PRODUCTION DELTA: Add optimistic locking version field, domain event outbox pattern
```

```java
// FILE: com/govfinex/retrieval/domain/FinancialData.java
//
// ARCHITECTURAL NOTE:
// Zero Spring imports in this file. Zero JPA annotations.
// This class compiles and tests with plain javac.
// If you add @Entity or @Autowired here, you have broken Hexagonal Architecture.
// The persistence adapter (InMemoryDataRetrievalRepository) handles
// the mapping between this domain object and whatever storage is used.

package com.govfinex.retrieval.domain;

import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * AGGREGATE ROOT: FinancialData
 *
 * Represents the result of a citizen's consented data retrieval
 * from one financial institution. One aggregate per institution per consent.
 *
 * Domain invariants enforced:
 * 1. A retrieval cannot be completed without at least one account summary
 * 2. A failed retrieval cannot be transitioned to completed
 * 3. Account summaries cannot be added to a failed or completed retrieval
 */
public final class FinancialData {

    private final RetrievalId retrievalId;
    private final String consentId;
    private final InstitutionId institutionId;
    private final String citizenId;
    private final Instant initiatedAt;

    private RetrievalStatus status;
    private final List<AccountSummary> accountSummaries;
    private Instant completedAt;
    private String failureReason;

    // Domain events collected during lifecycle
    // Application service publishes these after persisting
    private final List<RetrievalDomainEvent> domainEvents;

    public FinancialData(
            RetrievalId retrievalId,
            String consentId,
            InstitutionId institutionId,
            String citizenId
    ) {
        this.retrievalId = retrievalId;
        this.consentId = consentId;
        this.institutionId = institutionId;
        this.citizenId = citizenId;
        this.initiatedAt = Instant.now();
        this.status = RetrievalStatus.INITIATED;
        this.accountSummaries = new ArrayList<>();
        this.domainEvents = new ArrayList<>();

        // Domain event: retrieval started
        this.domainEvents.add(new RetrievalDomainEvent.RetrievalInitiated(
                retrievalId.value(), consentId, institutionId.value(), citizenId
        ));
    }

    /**
     * COMMAND: Record account summaries from institution adapter.
     *
     * Anti-Corruption Layer responsibility: the adapter has already
     * translated the bank's native model into AccountSummary domain objects.
     * This method receives clean domain objects — not bank API response DTOs.
     */
    public void completeWithData(List<AccountSummary> summaries) {
        if (this.status != RetrievalStatus.INITIATED) {
            throw new RetrievalInvariantException(
                "Cannot complete retrieval in status: " + this.status
            );
        }
        if (summaries == null || summaries.isEmpty()) {
            throw new RetrievalInvariantException(
                "Cannot complete retrieval with empty account summaries"
            );
        }

        this.accountSummaries.addAll(summaries);
        this.status = RetrievalStatus.COMPLETED;
        this.completedAt = Instant.now();

        this.domainEvents.add(new RetrievalDomainEvent.RetrievalCompleted(
                retrievalId.value(), consentId, institutionId.value(),
                summaries.size(), completedAt
        ));
    }

    /**
     * COMMAND: Mark retrieval as failed.
     * Preserves reason for audit log — MAS TRM requires failure audit trail.
     */
    public void fail(String reason) {
        if (this.status == RetrievalStatus.COMPLETED) {
            throw new RetrievalInvariantException(
                "Cannot fail a completed retrieval"
            );
        }
        this.status = RetrievalStatus.FAILED;
        this.failureReason = reason;

        this.domainEvents.add(new RetrievalDomainEvent.RetrievalFailed(
                retrievalId.value(), consentId, institutionId.value(), reason
        ));
    }

    public List<RetrievalDomainEvent> popDomainEvents() {
        var events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }

    // Immutable read accessors
    public RetrievalId retrievalId() { return retrievalId; }
    public String consentId() { return consentId; }
    public InstitutionId institutionId() { return institutionId; }
    public String citizenId() { return citizenId; }
    public RetrievalStatus status() { return status; }
    public List<AccountSummary> accountSummaries() {
        return Collections.unmodifiableList(accountSummaries);
    }
    public Instant initiatedAt() { return initiatedAt; }
    public Instant completedAt() { return completedAt; }
    public String failureReason() { return failureReason; }
}
```

```java
// FILE: com/govfinex/retrieval/domain/AccountSummary.java
//
// Java 17 RECORD — Value Object
// Records are immutable by design. Perfect for Value Objects in DDD.
// Equality is structural (based on values), not referential.
// No boilerplate: equals, hashCode, toString generated automatically.

package com.govfinex.retrieval.domain;

import java.math.BigDecimal;
import java.util.Currency;

/**
 * VALUE OBJECT: AccountSummary
 *
 * Represents one account's summary within a financial data retrieval.
 * Immutable — once created, values cannot change.
 * This is the DOMAIN representation — not the bank's native format.
 * The Anti-Corruption Layer in each institution adapter translates
 * the bank's native account model into this domain record.
 */
public record AccountSummary(
        String accountNumber,    // Masked in production: show last 4 only
        AccountType accountType,
        Money availableBalance,
        Money currentBalance,
        String currency,
        String institutionCode   // Which institution this came from
) {
    // Compact constructor for validation
    public AccountSummary {
        if (accountNumber == null || accountNumber.isBlank()) {
            throw new RetrievalInvariantException("Account number cannot be blank");
        }
        if (availableBalance == null || currentBalance == null) {
            throw new RetrievalInvariantException("Balance cannot be null");
        }
    }
}
```

```java
// FILE: com/govfinex/retrieval/domain/RetrievalDomainEvent.java
//
// Java 17 SEALED CLASS — Domain Event hierarchy
// Sealed classes enumerate all valid subtypes at compile time.
// The compiler enforces exhaustive handling in switch expressions.
// Perfect for domain event hierarchies where you know all event types.

package com.govfinex.retrieval.domain;

import java.time.Instant;

/**
 * SEALED CLASS: RetrievalDomainEvent
 *
 * All domain events emitted by FinancialData aggregate.
 * Sealed: only these three event types can exist.
 * Pattern matching switch in the event publisher
 * handles each case with compile-time exhaustiveness check.
 */
public sealed interface RetrievalDomainEvent
        permits RetrievalDomainEvent.RetrievalInitiated,
                RetrievalDomainEvent.RetrievalCompleted,
                RetrievalDomainEvent.RetrievalFailed {

    String retrievalId();
    String consentId();

    // Java 17 records implementing sealed interface
    // Each record is an immutable, self-describing event

    record RetrievalInitiated(
            String retrievalId,
            String consentId,
            String institutionId,
            String citizenId
    ) implements RetrievalDomainEvent {}

    record RetrievalCompleted(
            String retrievalId,
            String consentId,
            String institutionId,
            int accountCount,
            Instant completedAt
    ) implements RetrievalDomainEvent {}

    record RetrievalFailed(
            String retrievalId,
            String consentId,
            String institutionId,
            String reason
    ) implements RetrievalDomainEvent {}
}
```

---

## DATA RETRIEVAL SERVICE — Ports (Java 17)

```java
// FILE: com/govfinex/retrieval/application/port/out/InstitutionDataPort.java
//
// ARCHITECTURAL NOTE:
// This is the most architecturally significant interface in the lab.
// It is the boundary between the APPLICATION CORE and the INSTITUTION ADAPTERS.
// DBS, OCBC, CPF Board — each gets one adapter that implements this interface.
// The DataRetrievalService calls this port — it never knows which bank is behind it.

package com.govfinex.retrieval.application.port.out;

import com.govfinex.retrieval.domain.AccountSummary;
import com.govfinex.retrieval.domain.InstitutionId;

import java.util.List;

/**
 * Driven Port: Institution Data Access
 *
 * Anti-Corruption Layer contract:
 * Implementations MUST translate the institution's native data model
 * into AccountSummary domain objects BEFORE returning.
 * The core never sees raw bank API responses.
 *
 * Implementations:
 * - MockDBSAdapter (lab)
 * - MockOCBCAdapter (lab)
 * - MockCPFAdapter (lab)
 * - HttpDBSAdapter (production — calls DBS SGFinDex endpoint with mTLS)
 */
public interface InstitutionDataPort {

    /**
     * Fetch account summaries for a citizen from this institution.
     *
     * @param citizenId      The citizen's pseudonymous identifier
     * @param consentId      The consent authorising this retrieval (for audit)
     * @param institutionId  Which institution this adapter serves
     * @return List of domain-model AccountSummary objects (never null, may be empty)
     * @throws InstitutionUnavailableException if the institution endpoint is down
     * @throws UnauthorisedDataAccessException if consent does not cover this institution
     */
    List<AccountSummary> fetchAccountSummaries(
            String citizenId,
            String consentId,
            InstitutionId institutionId
    );

    /**
     * Returns the InstitutionId this adapter serves.
     * Used by DataRetrievalService to route to the correct adapter.
     */
    InstitutionId supports();
}
```

---

## DATA RETRIEVAL SERVICE — Mock Institution Adapters (Java 17)

```java
// FILE: com/govfinex/retrieval/adapter/out/institution/MockDBSAdapter.java
//
// ANTI-CORRUPTION LAYER demonstration:
// This adapter translates DBS's "native" response format
// (simulated here as inner classes) into domain AccountSummary objects.
// The core never sees DbsAccountRecord or DbsBalanceInfo.

package com.govfinex.retrieval.adapter.out.institution;

import com.govfinex.retrieval.application.port.out.InstitutionDataPort;
import com.govfinex.retrieval.domain.*;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.List;

@Component
public class MockDBSAdapter implements InstitutionDataPort {

    // Simulated DBS "native" response model
    // In production: this would be the actual DBS SGFinDex API response DTO
    // deserialized from their HTTPS endpoint
    private record DbsAccountRecord(
            String acctNo,
            String acctType,
            double avlBal,    // DBS uses double — domain uses BigDecimal (precision)
            double curBal,
            String ccy
    ) {}

    @Override
    public List<AccountSummary> fetchAccountSummaries(
            String citizenId,
            String consentId,
            InstitutionId institutionId
    ) {
        // PRODUCTION NOTE: This would be an HTTPS call to DBS's SGFinDex endpoint
        // with the citizen's consent token. Response would be parsed from JSON.
        // Here we return realistic mock data.

        List<DbsAccountRecord> dbsResponse = simulateDbsApiCall(citizenId);

        // ANTI-CORRUPTION LAYER TRANSLATION:
        // DBS's model → our domain model
        // Notice: double → BigDecimal (precision fix)
        //         DBS account type codes → our AccountType enum
        //         acctNo masked to last 4 digits (PDPA)
        return dbsResponse.stream()
                .map(this::translateToAccountSummary)
                .toList();
    }

    private AccountSummary translateToAccountSummary(DbsAccountRecord dbs) {
        return new AccountSummary(
                maskAccountNumber(dbs.acctNo()),      // PDPA: mask account number
                mapAccountType(dbs.acctType()),        // Translate DBS codes
                new Money(BigDecimal.valueOf(dbs.avlBal()), dbs.ccy()),
                new Money(BigDecimal.valueOf(dbs.curBal()), dbs.ccy()),
                dbs.ccy(),
                "DBS"
        );
    }

    private String maskAccountNumber(String fullNumber) {
        // Show only last 4 digits — PDPA data minimisation
        if (fullNumber == null || fullNumber.length() <= 4) return fullNumber;
        return "****" + fullNumber.substring(fullNumber.length() - 4);
    }

    private AccountType mapAccountType(String dbsCode) {
        // DBS uses their own type codes — we map to our domain enum
        return switch (dbsCode) {
            case "SVGS", "SAV" -> AccountType.SAVINGS;
            case "CURR", "CUR" -> AccountType.CURRENT;
            case "FD"          -> AccountType.FIXED_DEPOSIT;
            default            -> AccountType.UNKNOWN;
        };
    }

    private List<DbsAccountRecord> simulateDbsApiCall(String citizenId) {
        // Realistic mock response — simulates what DBS SGFinDex endpoint returns
        return List.of(
                new DbsAccountRecord("0039123456789", "SVGS", 12450.75, 12450.75, "SGD"),
                new DbsAccountRecord("0039987654321", "CURR", 3200.00, 3200.00, "SGD"),
                new DbsAccountRecord("0039111222333", "FD",   50000.00, 50000.00, "SGD")
        );
    }

    @Override
    public InstitutionId supports() {
        return InstitutionId.of("DBS");
    }
}
```

---

## DATA RETRIEVAL SERVICE — Application Service (Java 17)

```java
// FILE: com/govfinex/retrieval/application/DataRetrievalService.java
//
// The Application Service that orchestrates:
// 1. Receives a retrieval request
// 2. Creates a FinancialData aggregate for each institution
// 3. Calls the correct InstitutionDataPort adapter
// 4. Persists and publishes events
// Note the use of Java 17 pattern matching and text blocks.

package com.govfinex.retrieval.application;

import com.govfinex.retrieval.application.port.in.RetrieveFinancialDataUseCase;
import com.govfinex.retrieval.application.port.out.*;
import com.govfinex.retrieval.domain.*;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
public class DataRetrievalService implements RetrieveFinancialDataUseCase {

    private final Map<InstitutionId, InstitutionDataPort> institutionAdapters;
    private final DataRetrievalRepository repository;
    private final EventPublisherPort eventPublisher;

    // Constructor injection — all ports, never concrete classes
    public DataRetrievalService(
            List<InstitutionDataPort> adapters,
            DataRetrievalRepository repository,
            EventPublisherPort eventPublisher
    ) {
        // Build routing map: InstitutionId → correct adapter
        // Adding a new bank = add one new adapter class + Spring @Component
        // Zero changes to this service
        this.institutionAdapters = adapters.stream()
                .collect(Collectors.toMap(
                        InstitutionDataPort::supports,
                        Function.identity()
                ));
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public List<FinancialData> retrieveForConsent(
            String consentId,
            String citizenId,
            List<String> institutionCodes
    ) {
        var results = new ArrayList<FinancialData>();

        for (String code : institutionCodes) {
            var institutionId = InstitutionId.of(code);
            var financialData = new FinancialData(
                    new RetrievalId(),
                    consentId,
                    institutionId,
                    citizenId
            );

            try {
                // Route to correct adapter — pattern from routing map
                var adapter = institutionAdapters.get(institutionId);
                if (adapter == null) {
                    financialData.fail(
                        "No adapter registered for institution: " + code
                    );
                } else {
                    var summaries = adapter.fetchAccountSummaries(
                            citizenId, consentId, institutionId
                    );
                    financialData.completeWithData(summaries);
                }
            } catch (Exception e) {
                // Isolate failure: one institution failing does not fail others
                // Blast radius containment — core pattern for multi-institution fetch
                financialData.fail(e.getMessage());
            }

            repository.save(financialData);

            // Java 17 pattern matching on sealed class — exhaustive
            // Compiler enforces all event types are handled
            for (var event : financialData.popDomainEvents()) {
                String eventJson = switch (event) {
                    case RetrievalDomainEvent.RetrievalInitiated e ->
                        buildInitiatedJson(e);
                    case RetrievalDomainEvent.RetrievalCompleted e ->
                        buildCompletedJson(e);
                    case RetrievalDomainEvent.RetrievalFailed e ->
                        buildFailedJson(e);
                };
                eventPublisher.publish(eventJson);
            }

            results.add(financialData);
        }

        return results;
    }

    // Java 17 text blocks for JSON construction
    // In production: use Jackson ObjectMapper — never manual JSON string building
    private String buildCompletedJson(RetrievalDomainEvent.RetrievalCompleted e) {
        return """
                {
                  "eventType": "RetrievalCompleted",
                  "retrievalId": "%s",
                  "consentId": "%s",
                  "institutionId": "%s",
                  "accountCount": %d,
                  "completedAt": "%s"
                }
                """.formatted(
                e.retrievalId(), e.consentId(), e.institutionId(),
                e.accountCount(), e.completedAt()
        );
    }

    private String buildInitiatedJson(RetrievalDomainEvent.RetrievalInitiated e) {
        return """
                {"eventType":"RetrievalInitiated","retrievalId":"%s","consentId":"%s"}
                """.formatted(e.retrievalId(), e.consentId());
    }

    private String buildFailedJson(RetrievalDomainEvent.RetrievalFailed e) {
        return """
                {"eventType":"RetrievalFailed","retrievalId":"%s","reason":"%s"}
                """.formatted(e.retrievalId(), e.reason());
    }
}
```

---

## DOMAIN TESTS — Zero Infrastructure (Java 17)

```java
// FILE: src/test/java/com/govfinex/retrieval/domain/FinancialDataTest.java
//
// ARCHITECTURAL PROOF: These tests have ZERO Spring context, ZERO database,
// ZERO network calls. They run in < 100ms total.
// If your domain tests require a Spring context, your architecture is wrong.

package com.govfinex.retrieval.domain;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

@DisplayName("FinancialData Aggregate — Domain Invariant Tests")
class FinancialDataTest {

    private FinancialData createTestAggregate() {
        return new FinancialData(
                new RetrievalId(),
                "consent-uuid-123",
                InstitutionId.of("DBS"),
                "S1234567A"
        );
    }

    private List<AccountSummary> sampleSummaries() {
        return List.of(new AccountSummary(
                "****6789",
                AccountType.SAVINGS,
                new Money(BigDecimal.valueOf(12450.75), "SGD"),
                new Money(BigDecimal.valueOf(12450.75), "SGD"),
                "SGD",
                "DBS"
        ));
    }

    @Test
    @DisplayName("New aggregate starts in INITIATED status")
    void newAggregateIsInitiated() {
        var data = createTestAggregate();
        assertThat(data.status()).isEqualTo(RetrievalStatus.INITIATED);
    }

    @Test
    @DisplayName("Completing with data transitions to COMPLETED")
    void completeWithDataTransitionsStatus() {
        var data = createTestAggregate();
        data.completeWithData(sampleSummaries());
        assertThat(data.status()).isEqualTo(RetrievalStatus.COMPLETED);
        assertThat(data.accountSummaries()).hasSize(1);
    }

    @Test
    @DisplayName("Cannot complete with empty summaries — invariant enforced")
    void cannotCompleteWithEmptySummaries() {
        var data = createTestAggregate();
        assertThatThrownBy(() -> data.completeWithData(List.of()))
                .isInstanceOf(RetrievalInvariantException.class)
                .hasMessageContaining("empty account summaries");
    }

    @Test
    @DisplayName("Cannot fail a completed retrieval — invariant enforced")
    void cannotFailCompletedRetrieval() {
        var data = createTestAggregate();
        data.completeWithData(sampleSummaries());

        assertThatThrownBy(() -> data.fail("network error"))
                .isInstanceOf(RetrievalInvariantException.class)
                .hasMessageContaining("Cannot fail a completed retrieval");
    }

    @Test
    @DisplayName("Domain events emitted correctly on completion")
    void domainEventsEmittedOnCompletion() {
        var data = createTestAggregate();

        // Pop initiation events first
        data.popDomainEvents();

        data.completeWithData(sampleSummaries());
        var events = data.popDomainEvents();

        assertThat(events).hasSize(1);
        assertThat(events.get(0))
                .isInstanceOf(RetrievalDomainEvent.RetrievalCompleted.class);

        // Events consumed — should be empty now
        assertThat(data.popDomainEvents()).isEmpty();
    }
}
```

---

## INFRASTRUCTURE — Docker Compose

```
FILE: docker-compose.yml
LANGUAGE: YAML
PURPOSE: Orchestrates all three services for local development.
         Simulates a multi-service deployment without Kubernetes complexity.
CONCEPTS DEMONSTRATED: Service isolation per Bounded Context
COPY-PASTE READY: YES
PRODUCTION DELTA: Would use Kubernetes manifests (Helm charts),
                  add resource limits, liveness/readiness probes, secrets management
```

```yaml
# docker-compose.yml
#
# ARCHITECTURAL NOTE:
# Three services = three Bounded Contexts = three independent deployable units.
# They communicate via the shared event bus (in-memory list shared via volume).
# In production: replace the shared volume with Apache Kafka.
# The service code does NOT change when you make this swap —
# only the EventPublisher adapter implementation changes.

version: "3.9"

services:

  consent-service:
    build:
      context: ./consent-service
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
    environment:
      - SERVICE_NAME=consent-service
      - LOG_LEVEL=INFO
      # In production: inject secrets via Azure Key Vault references,
      # never as plain environment variables
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - govfinex-network

  data-retrieval-service:
    build:
      context: ./data-retrieval-service
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=local
      - SERVER_PORT=8080
    depends_on:
      consent-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    networks:
      - govfinex-network

  aggregation-service:
    build:
      context: ./aggregation-service
      dockerfile: Dockerfile
    ports:
      - "8082:8082"
    environment:
      - SERVICE_NAME=aggregation-service
      - CONSENT_SERVICE_URL=http://consent-service:8081
      - DATA_RETRIEVAL_URL=http://data-retrieval-service:8080
    depends_on:
      data-retrieval-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - govfinex-network

networks:
  govfinex-network:
    driver: bridge
    # All services on isolated network —
    # they cannot be reached from outside except via mapped ports
    # In production: network policies enforced at Kubernetes layer
```

---

## INFRASTRUCTURE — Terraform (Azure Free Tier)

```
FILE: infra/terraform/main.tf
LANGUAGE: HCL (Terraform)
PURPOSE: Provision Azure Container Apps (free tier) for all three services.
CONCEPTS DEMONSTRATED: IaC for multi-service deployment, Azure free tier optimisation
COPY-PASTE READY: YES — targets Azure free tier resources
PRODUCTION DELTA: Add Azure Application Gateway, Azure Key Vault references,
                  managed identity, private endpoints, Azure Monitor workspace
```

```hcl
# infra/terraform/main.tf
#
# ARCHITECTURAL NOTE:
# Azure Container Apps free tier allows:
# - 180,000 vCPU-seconds/month
# - 360,000 GB-seconds/month
# - 2 million requests/month
# This is sufficient for lab and demo workloads.
# All three services will fit within free tier allocation.

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
  required_version = ">= 1.5.0"
}

provider "azurerm" {
  features {}
  # Authenticate via: az login (Azure CLI)
  # In production: use managed identity or service principal
}

# Resource Group — logical container for all lab resources
resource "azurerm_resource_group" "govfinex" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment  = "lab"
    project      = "govfinex-day2"
    cost-centre  = "training"
    # Tag every resource — FinOps fundamental practice
  }
}

# Log Analytics Workspace — required for Container Apps environment
resource "azurerm_log_analytics_workspace" "govfinex" {
  name                = "law-govfinex-${var.environment}"
  location            = azurerm_resource_group.govfinex.location
  resource_group_name = azurerm_resource_group.govfinex.name
  sku                 = "PerGB2018"
  retention_in_days   = 30  # Minimum — reduce cost

  tags = azurerm_resource_group.govfinex.tags
}

# Container Apps Environment — the hosting environment for all services
resource "azurerm_container_app_environment" "govfinex" {
  name                       = "cae-govfinex-${var.environment}"
  location                   = azurerm_resource_group.govfinex.location
  resource_group_name        = azurerm_resource_group.govfinex.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.govfinex.id

  tags = azurerm_resource_group.govfinex.tags
}

# Consent Service — Azure Container App
resource "azurerm_container_app" "consent_service" {
  name                         = "ca-consent-service"
  container_app_environment_id = azurerm_container_app_environment.govfinex.id
  resource_group_name          = azurerm_resource_group.govfinex.name
  revision_mode                = "Single"

  template {
    container {
      name   = "consent-service"
      image  = "govfinexlab/consent-service:latest"
      cpu    = 0.25   # Free tier: 0.25 vCPU is sufficient for lab
      memory = "0.5Gi"

      env {
        name  = "SERVICE_NAME"
        value = "consent-service"
      }
      env {
        name  = "LOG_LEVEL"
        value = "INFO"
      }
    }

    # Scale to zero when not in use — critical for free tier cost management
    min_replicas = 0
    max_replicas = 1
  }

  ingress {
    external_enabled = true
    target_port      = 8081
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }

  tags = azurerm_resource_group.govfinex.tags
}

# Data Retrieval Service — Azure Container App
resource "azurerm_container_app" "data_retrieval_service" {
  name                         = "ca-data-retrieval"
  container_app_environment_id = azurerm_container_app_environment.govfinex.id
  resource_group_name          = azurerm_resource_group.govfinex.name
  revision_mode                = "Single"

  template {
    container {
      name   = "data-retrieval-service"
      image  = "govfinexlab/data-retrieval-service:latest"
      cpu    = 0.5    # Java service needs slightly more CPU for JVM startup
      memory = "1Gi"

      env {
        name  = "SPRING_PROFILES_ACTIVE"
        value = "azure"
      }
    }

    min_replicas = 0
    max_replicas = 1
  }

  ingress {
    external_enabled = true
    target_port      = 8080
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }

  tags = azurerm_resource_group.govfinex.tags
}
```

```hcl
# infra/terraform/variables.tf

variable "resource_group_name" {
  description = "Azure Resource Group name for all GovFinEx lab resources"
  type        = string
  default     = "rg-govfinex-lab"
}

variable "location" {
  description = "Azure region. Southeast Asia for Singapore-context labs."
  type        = string
  default     = "southeastasia"
  # Southeast Asia = Singapore region
  # For India context: use "centralindia" or "southindia"
}

variable "environment" {
  description = "Deployment environment label"
  type        = string
  default     = "lab"
}
```

```hcl
# infra/terraform/outputs.tf

output "consent_service_url" {
  description = "Public URL for the consent service"
  value       = azurerm_container_app.consent_service.ingress[0].fqdn
}

output "data_retrieval_service_url" {
  description = "Public URL for the data retrieval service"
  value       = azurerm_container_app.data_retrieval_service.ingress[0].fqdn
}

output "resource_group_name" {
  description = "Resource group containing all lab resources"
  value       = azurerm_resource_group.govfinex.name
}
```

---

# L4. STEP-BY-STEP EXECUTION GUIDE

---

**STEP 1 of 6: Start services and verify OpenAPI UI**

🎯 OBJECTIVE: Confirm that API-First worked — the OpenAPI spec drives the UI before any business logic is called.

CONCEPT LINK: API-First Design (Block 2) — the contract is visible and testable before integration.

```bash
$ docker compose up --build -d
$ sleep 15  # Allow Java service JVM startup time
$ curl -s http://localhost:8081/health
```

Expected Output:
```json
{"status": "healthy", "service": "consent-service", "version": "0.1.0"}
```

Now open in browser: `http://localhost:8081/docs`

🔍 WHAT TO OBSERVE: FastAPI auto-generates this interactive UI directly from the OpenAPI YAML spec in `contracts/openapi/consent-api-v1.yaml`. Every field, enum, and example you defined in the YAML appears here. This IS API-First — the contract is the UI.

⚠️ COMMON MISTAKE: Candidates change the FastAPI Pydantic model (schemas.py) instead of updating the YAML spec first. The YAML is the source of truth. Always update YAML first, then update code to match.

🔧 FIX: If the UI does not match the YAML, check that `main.py` loads the spec from the contracts directory, not from auto-generated Pydantic schemas.

🤖 COPILOT PROMPT:
"Explain the difference between code-first and API-first OpenAPI approaches. What are the production risks of code-first API design in a government platform with 500+ API consumers?"

---

**STEP 2 of 6: Create a consent grant via API**

🎯 OBJECTIVE: Exercise the full Hexagonal Architecture stack — REST adapter → application service → domain aggregate → repository adapter → event publisher adapter.

CONCEPT LINK: Hexagonal Architecture full flow (Block 1, Diagram 1).

```bash
$ curl -X POST http://localhost:8081/consents \
  -H "Content-Type: application/json" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "citizenId": "S1234567A",
    "institutions": ["DBS", "OCBC", "CPF"],
    "dataCategories": ["ACCOUNT_BALANCE", "CPF_CONTRIBUTIONS"],
    "purposeCode": "FINANCIAL_PLANNING",
    "durationDays": 30
  }' | python3 -m json.tool
```

Expected Output:
```json
{
  "consentId": "a1b2c3d4-...",
  "citizenId": "S1234567A",
  "status": "ACTIVE",
  "institutions": ["DBS", "OCBC", "CPF"],
  "dataCategories": ["ACCOUNT_BALANCE", "CPF_CONTRIBUTIONS"],
  "purposeCode": "FINANCIAL_PLANNING",
  "createdAt": "2026-06-12T09:30:00+00:00",
  "expiresAt": "2026-07-12T09:30:00+00:00",
  "revokedAt": null
}
```

🔍 WHAT TO OBSERVE: Copy the `consentId` value — you need it for Step 3. Note that `revokedAt` is `null` — in the OpenAPI spec, this field is marked `nullable: true`. This was a deliberate contract decision. If you had marked it non-nullable, this response would fail schema validation.

---

**STEP 3 of 6: Trigger financial data retrieval**

🎯 OBJECTIVE: Show the Anti-Corruption Layer in action — three different mock bank adapters translate their "native" formats into the same domain model.

```bash
# Use the consentId from Step 2
$ CONSENT_ID="paste-consent-id-here"

$ curl -X POST http://localhost:8080/retrievals \
  -H "Content-Type: application/json" \
  -d "{
    \"consentId\": \"$CONSENT_ID\",
    \"citizenId\": \"S1234567A\",
    \"institutions\": [\"DBS\", \"OCBC\", \"CPF\"]
  }" | python3 -m json.tool
```

Expected Output (truncated):
```json
{
  "retrievals": [
    {
      "institutionId": "DBS",
      "status": "COMPLETED",
      "accountSummaries": [
        {
          "accountNumber": "****6789",
          "accountType": "SAVINGS",
          "availableBalance": {"amount": 12450.75, "currency": "SGD"},
          "institutionCode": "DBS"
        }
      ]
    },
    {
      "institutionId": "OCBC",
      "status": "COMPLETED",
      "accountSummaries": [...]
    },
    {
      "institutionId": "CPF",
      "status": "COMPLETED",
      "accountSummaries": [...]
    }
  ]
}
```

🔍 WHAT TO OBSERVE: All three institution adapters return data in identical domain format — `accountNumber`, `accountType`, `availableBalance`. The DBS adapter internally used `acctNo` and `avlBal` — translated in the ACL. OCBC uses different field names. CPF uses a completely different model (contributions, not accounts). All three produce the same domain `AccountSummary` record.

⚠️ COMMON MISTAKE: Candidates try to add bank-specific fields to the response because "the client might need them." This breaks the bounded context separation. If a consumer needs bank-specific data, they build a separate integration with that bank's native API — not through this abstraction layer.

---

**STEP 4 of 6: Test domain invariants in isolation**

🎯 OBJECTIVE: Demonstrate that domain tests run with zero infrastructure. This is the proof that Hexagonal Architecture is working correctly.

CONCEPT LINK: Hexagonal Architecture — testability as a first-class concern (Block 1, Layer 3).

```bash
$ cd consent-service
$ source venv/bin/activate
$ python3 -m pytest tests/test_domain.py -v
```

Expected Output:
```
tests/test_domain.py::test_new_consent_is_active PASSED                     [ 20%]
tests/test_domain.py::test_revoke_active_consent PASSED                     [ 40%]
tests/test_domain.py::test_cannot_revoke_already_revoked PASSED             [ 60%]
tests/test_domain.py::test_duration_over_90_days_raises PASSED              [ 80%]
tests/test_domain.py::test_domain_events_emitted_on_grant PASSED            [100%]

5 passed in 0.047s
```

🔍 WHAT TO OBSERVE: 0.047 seconds. No database. No HTTP. No Docker. Pure Python business logic tested at the speed of thought. Now run the Java domain tests:

```bash
$ cd ../data-retrieval-service
$ ./mvnw test -pl . -Dtest="FinancialDataTest" -q
```

Expected Output:
```
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time:  3.241 s
```

⚠️ COMMON MISTAKE: If this test requires `@SpringBootTest`, your domain layer has leaked framework dependencies. Check imports in `FinancialData.java` — there must be zero Spring imports.

---

**STEP 5 of 6: Demonstrate adapter swap without code change**

🎯 OBJECTIVE: Swap the in-memory event publisher for a logging publisher — without changing a single line of domain or application service code. This proves the adapter pattern works.

```bash
# Create a simple logging publisher adapter
$ cat > consent-service/src/adapters/logging_publisher.py << 'EOF'
import json
from src.ports.event_publisher import EventPublisher

class LoggingEventPublisher(EventPublisher):
    """
    Alternative driven adapter: logs events to stdout instead of storing them.
    Useful for debugging. Can be activated via configuration.
    Zero changes to domain or application service required.
    """
    def publish(self, event: dict) -> None:
        print(f"[EVENT LOG] {json.dumps(event, indent=2)}")
    
    def get_published_events(self) -> list[dict]:
        raise NotImplementedError("LoggingEventPublisher does not store events")
EOF
```

Now in `consent-service/main.py`, swap the publisher:

```python
# Change this line:
publisher = InMemoryEventPublisher()

# To this line:
from src.adapters.logging_publisher import LoggingEventPublisher
publisher = LoggingEventPublisher()
```

Restart and create a consent — observe events printed to stdout instead of stored.

🔍 WHAT TO OBSERVE: The domain (`consent.py`), the application service (`consent_service.py`), and the API adapter (`router.py`) required zero modifications. Only the dependency wiring in `main.py` changed. This is the entire value of Hexagonal Architecture demonstrated in a single 5-line change.

---

**STEP 6 of 6: Deploy to Azure with Terraform**

🎯 OBJECTIVE: Provision the Azure Container Apps environment using IaC.

```bash
$ az login
# Follow browser authentication flow

$ cd infra/terraform
$ terraform init
```

Expected Output:
```
Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "~> 3.85"...
- Installed hashicorp/azurerm v3.85.0
Terraform has been successfully initialized!
```

```bash
$ terraform plan -out=tfplan
# Review the plan — should show 5 resources to create

$ terraform apply tfplan
```

Expected Output (last 3 lines):
```
azurerm_container_app.consent_service: Creation complete after 45s
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
Outputs:
consent_service_url = "ca-consent-service.bluefield-abc123.southeastasia.azurecontainerapps.io"
```

✅ VERIFY:
```bash
$ curl -s https://$(terraform output -raw consent_service_url)/health
# Expected: {"status": "healthy", "service": "consent-service", "version": "0.1.0"}
```

Teardown (important — free tier has monthly limits):
```bash
$ terraform destroy -auto-approve
```

---

# L5. TRAINER DEMO SCRIPT

---

🎬 TRAINER DEMO SCRIPT: The Architecture Lives in the Code

SETUP CHECK (Before showing screen):
□ `docker compose up --build -d` completed successfully
□ Browser tab open at `http://localhost:8081/docs`
□ Terminal ready in project root directory
□ `consent-service/src/domain/consent.py` open in code editor

---

TALKING POINT 1 (While showing consent.py):

"Look at the top of this file. Count the imports with me. `uuid`, `dataclasses`, `datetime`, `enum`, `typing`. That is it. No FastAPI. No SQLAlchemy. No Kafka. No Spring. This file has no idea it is running inside a web server. It does not know if it is Monday or if it is Budget Day with 50,000 concurrent requests. It only knows about consent grants. That is the entire point."

👉 POINT AT: The import block at the top of consent.py

ASK AUDIENCE: "If I add `from fastapi import HTTPException` to this file — what architectural principle have I violated and what is the practical consequence?"

EXPECTED RESPONSES: Hexagonal Architecture violated. Now you cannot test the domain without starting FastAPI. Domain is coupled to HTTP transport layer. Change in FastAPI version can break domain tests.

---

BREAK IT (Intentional failure for learning):

Add this to `ConsentDuration.__post_init__` in consent.py:
```python
from fastapi import HTTPException
raise HTTPException(status_code=400, detail="test")
```

Run domain tests:
```bash
$ python3 -m pytest tests/test_domain.py -v
```

🔴 SHOW: Import error — FastAPI not installed in test virtual environment (because it should not be).

EXPLAIN: "The test environment does not have FastAPI installed — deliberately. The domain should never need it. The moment your domain imports infrastructure libraries, your test environment must have those libraries. Your test speed degrades. Your CI pipeline gets slower. Your architecture has leaked."

FIX IT:
Remove the FastAPI import. Tests pass in 47ms again.

EXPLAIN: "The fix is the point. Hexagonal Architecture's value is not just theoretical elegance. It is this: your domain tests run in 47 milliseconds on a laptop with no Docker, no database, no network. That is 10,000 test runs per minute. That is confidence."

---

🤖 COPILOT LIVE PROMPT (Run while tests run):

"I have a Python domain class with zero framework imports. All business logic is tested with plain pytest in 47ms. I use dependency injection to wire FastAPI and SQLAlchemy adapters at startup. What are the production benefits and potential drawbacks of this strict Hexagonal Architecture approach for a Singapore government financial data platform?"

Expected Copilot output: Benefits around testability, adaptability. Drawbacks around initial complexity, learning curve, potential over-engineering for simple CRUD systems. Use this to spark class discussion.

---

# L6. VERIFICATION CHECKLIST

---

Lab Completion Verification:

□ consent-service running on port 8081 — verify:
  ```bash
  curl http://localhost:8081/health
  ```

□ data-retrieval-service running on port 8080 — verify:
  ```bash
  curl http://localhost:8080/actuator/health
  ```

□ aggregation-service running on port 8082 — verify:
  ```bash
  curl http://localhost:8082/health
  ```

□ OpenAPI UI shows all 4 endpoints from YAML spec — verify: open `http://localhost:8081/docs`

□ Consent creation returns ACTIVE status with 30-day expiry — verify: run Step 2 curl command

□ Data retrieval returns COMPLETED status from all 3 mock adapters — verify: run Step 3 curl command

□ Domain tests pass with zero infrastructure in < 1 second — verify: run Step 4 pytest command

□ Adapter swap completed without domain code change — verify: run Step 5 and observe stdout events

□ Terraform plan shows 5 resources (optional — requires Azure CLI login) — verify: `terraform plan`

□ Teardown completed to preserve free tier quota — verify: `terraform destroy -auto-approve`

---

# L7. EXTENSIVE DOCUMENTATION

---

## L7.1 Architecture Decision Documentation

**Why this technology stack:**

FastAPI for Python services was chosen over Flask because FastAPI generates OpenAPI schemas directly from Pydantic models — making API-First a natural default, not an afterthought. Pydantic v2 provides compile-time-equivalent type checking that enforces the Published Language schema at the Python layer.

Spring Boot 3.x for the Java service demonstrates enterprise-grade Hexagonal Architecture with constructor injection. The `@Component` annotation on adapters combined with interface injection in the application service is the idiomatic Spring implementation of the ports-and-adapters pattern.

**What the code demonstrates:**

The lab concretely demonstrates that the Anti-Corruption Layer is not a pattern in a book — it is the `translateToAccountSummary` method in `MockDBSAdapter`. That single method is where all the complexity of translating a legacy bank's data model into your clean domain model lives. In production systems, that method is 200 lines long and has 50 unit tests. In legacy integration projects, the absence of that method means the legacy model pollutes your entire codebase.

**How this maps to production scale:**

At SGFinDex scale (700+ organisations), each organisation is a separate adapter. The `InstitutionDataPort` interface is the contract every adapter must implement. Adding organisation number 701 = write one new adapter class implementing the existing interface. The orchestration layer (DataRetrievalService) does not change. The domain does not change. This is the O in SOLID (Open-Closed Principle) made concrete.

---

## L7.2 Pattern Deep-Dive

**Pattern: Hexagonal Architecture (Ports and Adapters)**

Formal Definition: An application architecture that isolates the core domain logic from external concerns (databases, HTTP, message queues, third-party APIs) through explicitly named ports (interfaces) and adapters (implementations). Introduced by Alistair Cockburn in 2005.

Variants:
- Clean Architecture (Robert Martin): Adds explicit layer naming (Entities, Use Cases, Interface Adapters, Frameworks)
- Onion Architecture (Jeffrey Palermo): Same concept, different naming
- The practical implementation is nearly identical across all three

When this pattern fails (anti-pattern zone):
- Simple CRUD services with no domain logic: Hexagonal Architecture adds unnecessary abstraction layers for a service that is purely a database wrapper
- Teams without DDD understanding: Without Bounded Contexts, the ports become arbitrary and the pattern degrades into meaningless interface proliferation
- When adapters are shared across bounded contexts: the entire isolation benefit disappears

Famous implementations:
- NPCI UPI architecture separates the payment routing core from bank CBS adapters — each bank is an adapter behind the same switching port
- Netflix Hystrix was designed as an adapter over circuit-breaking logic that the core streaming service never needed to know about directly

---

## L7.3 Production Readiness Gap Analysis

| Lab Has                   | Production Needs                                  | Effort to Add |
| ------------------------- | ------------------------------------------------- | ------------- |
| In-memory event bus       | Apache Kafka with Schema Registry                 | High          |
| Mock institution adapters | HTTPS adapters with mTLS certificates             | High          |
| No authentication         | OAuth 2.0 + PKCE + SingPass federation            | High          |
| SQLite/in-memory store    | PostgreSQL with read replicas + PgBouncer         | Medium        |
| Single instance           | Auto-scaling (KEDA for event-driven scaling)      | Medium        |
| stdout logging            | Azure Monitor + structured JSON logs + SIEM       | Medium        |
| No encryption             | AES-256 for citizenId at rest, TLS 1.3 in transit | High          |
| Single region             | Multi-region with Azure Traffic Manager           | High          |
| No rate limiting          | Azure API Management with per-consumer quotas     | Low           |
| Manual Terraform apply    | GitOps with Terraform Cloud + PR-gated applies    | Low           |

---

## L7.4 Security Review

Vulnerabilities intentionally absent from lab:

The `citizenId` field (`S1234567A`) is stored and returned in plaintext. In production under PDPA (Singapore), citizen NRIC/FIN is personal data requiring encryption at rest and in transit. The production implementation would store a pseudonymous identifier derived from the SingPass-issued subject identifier — never the raw NRIC.

OWASP Top 10 mapping:
- A01 Broken Access Control: Lab has no authentication. Production requires consent token validation on every data retrieval call.
- A02 Cryptographic Failures: citizenId plaintext. Production: field-level encryption.
- A03 Injection: FastAPI's Pydantic validation prevents injection at the API boundary. Lab correctly demonstrates this.
- A04 Insecure Design: The idempotency key pattern prevents duplicate consent grants — correct security design.

Singapore context (MAS TRM, PDPA):

MAS TRM 2021, Section 9.2 requires financial institutions to implement API authentication using industry-standard protocols (OAuth 2.0). The consent service's `/consents` endpoint requires SingPass-backed OAuth 2.0 tokens in production — the lab omits this for simplicity.

PDPA Section 13: Consent must be explicit, informed, and purpose-specific. The `purposeCode` field in the OpenAPI spec directly implements purpose limitation. The `dataCategories` field implements data minimisation. Both are PDPA requirements coded into the API contract.

India context (DPDP Act 2023):

For the ONDC use case from the assignment, DPDP Act 2023 Section 6 requires that consent for personal data processing must specify: the purpose, the data fiduciary, the data collected, and the duration. The `CreateConsentRequest` schema covers all four of these requirements — demonstrating that a well-designed API contract can be a compliance instrument, not just a technical specification.

---

## L7.5 Cost Architecture (Azure Free Tier)

| Resource                        | Free Tier Limit              | Used in Lab                  | Cost     |
| ------------------------------- | ---------------------------- | ---------------------------- | -------- |
| Azure Container Apps — vCPU     | 180,000 vCPU-seconds/month   | ~2,000 seconds (lab session) | ₹0 / S$0 |
| Azure Container Apps — memory   | 360,000 GB-seconds/month     | ~4,000 GB-seconds            | ₹0 / S$0 |
| Azure Container Apps — requests | 2 million/month              | ~500 (lab)                   | ₹0 / S$0 |
| Log Analytics Workspace         | 5 GB/day free ingestion      | < 100 MB (lab)               | ₹0 / S$0 |
| Azure Container Registry        | Free tier: 1 registry, 10 GB | < 1 GB                       | ₹0 / S$0 |

Production Equivalent (Estimated for Singapore SGFinDex scale):

| Resource at Scale                                   | Production Spec           | Monthly Cost |
| --------------------------------------------------- | ------------------------- | ------------ |
| Azure Container Apps (consent-service, 10 replicas) | 2 vCPU, 4 GB RAM          | S$180/month  |
| Azure Container Apps (data-retrieval, 20 replicas)  | 4 vCPU, 8 GB RAM          | S$720/month  |
| Azure Database for PostgreSQL (Flexible)            | General Purpose, 4 vCores | S$280/month  |
| Azure Service Bus Premium (Kafka alternative)       | 1 messaging unit          | S$380/month  |
| Azure API Management (Standard)                     | 1 unit                    | S$480/month  |

Cost Optimisation Opportunities:
- Scale-to-zero on Container Apps during off-peak hours: estimated 40% compute saving
- Reserved instances for PostgreSQL (1-year): 30% saving vs. pay-as-you-go
- Azure Service Bus Standard tier if message volume < 13 million/month: 70% saving vs. Premium

---

## L7.6 Regulatory Compliance Notes

| Regulation                 | Relevance to This Lab                                                           | Gap in Lab                                         | What's Needed                                                 |
| -------------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------- |
| PDPA Singapore, Section 13 | Consent must be explicit and purpose-specific — implemented in ConsentAggregate | citizenId stored in plaintext                      | AES-256 field-level encryption; pseudonymous ID from SingPass |
| MAS TRM 2021, Section 9.2  | API authentication via OAuth 2.0                                                | No authentication on any endpoint                  | OAuth 2.0 + PKCE; SingPass federation                         |
| MAS TRM 2021, Section 9.3  | API version management and documentation                                        | API versioning exists in spec; no enforcement gate | Enforce version header validation in API Management           |
| DPDP Act 2023, Section 6   | Consent must specify purpose, duration, data fiduciary                          | Purpose and duration implemented                   | Add data fiduciary declaration field to consent request       |
| CERT-In 2022 (India)       | 6-hour incident reporting for data breaches                                     | No incident detection mechanism                    | Integrate Azure Sentinel with automated incident detection    |

---

## L7.7 Alternative Approaches

THIS LAB USES: Hexagonal Architecture (Ports and Adapters) with explicit interface segregation

ALTERNATIVE 1: Layered Architecture (Controller → Service → Repository)
WOULD WORK WHEN: Simple CRUD applications with no domain complexity, small team, short-lived system
TRADE-OFF vs. Hexagonal: Easier to understand initially, but testing requires infrastructure, adapter swap requires touching multiple layers, domain logic tends to leak into service or controller layer over time

ALTERNATIVE 2: CQRS from the Start (separate read and write models)
WOULD WORK WHEN: High read-to-write ratio (SGFinDex: 100 reads per consent created), complex query requirements, need for separate scaling of read and write paths
TRADE-OFF vs. Hexagonal: More complexity upfront. Hexagonal and CQRS are not mutually exclusive — Day 3 will show how CQRS sits on top of a Hexagonal core. Start with Hexagonal, add CQRS when read performance demands it.

ALTERNATIVE 3: GraphQL instead of OpenAPI REST
WOULD WORK WHEN: Consumer-driven data requirements, multiple consumer types needing different data shapes (financial planning app vs. loan application)
TRADE-OFF vs. OpenAPI: GraphQL gives consumers flexibility but makes API contract governance harder. For a government platform with strict PDPA data minimisation requirements, OpenAPI's explicit scoping is safer than GraphQL's flexible field selection.

---

## L7.8 Copilot Lab Prompts

UNDERSTANDING PROMPT:
"Explain the `ConsentAggregate` class in `consent.py`. What architectural pattern does it implement? What would break architecturally if I moved the `revoke()` method into the FastAPI router instead of keeping it in the domain class?"

EXTENSION PROMPT:
"How would I extend the GovFinEx consent-service to support delegated consent — where a person can grant consent on behalf of another (e.g., a financial advisor acting on behalf of a client)? What changes are needed in the domain model, the OpenAPI spec, and the database schema?"

CRITIQUE PROMPT:
"Review this Python domain class for DDD violations, Hexagonal Architecture violations, and PDPA Singapore compliance gaps: [paste consent.py]"

INDIA/SINGAPORE CONTEXT PROMPT:
"How would the ConsentAggregate pattern need to be modified to comply with India's DPDP Act 2023 consent requirements versus Singapore's PDPA consent requirements? What are the specific differences in how consent duration, purpose limitation, and withdrawal are handled under each regulation?"

---

✅ DAY 2 COMPLETE

📘 Document 1: Training Content — API-First Design, DDD & Hexagonal Architecture ✅ Generated

🔬 Document 2: Lab Manual — GovFinEx: Hexagonal Architecture + API-First in Practice ✅ Generated

📊 Quality Gates: All 5 passed ✅

Geography: 🇸🇬 Singapore (SGFinDex primary) + 🇮🇳 India (ONDC/GSTN secondary) + 🌏 Cross-border (UPI-PayNow) ✅

Copilot Integration: 6 embedded prompts across both documents ✅

Documentation Templates: RFC, Stakeholder Email, API Governance Policy ✅

Java 17 Features Used: Records, Sealed Classes, Text Blocks, Pattern Matching Switch, var ✅

Python 3.10+ Features Used: match-case, X|Y union types, dataclasses, type hints throughout ✅

👀 Day 3 Preview: Distributed Systems & Event-Driven Architecture

"Tomorrow we find out what actually happens inside Kafka when 10 billion UPI transactions flow through it every month. We build event sourcing, CQRS, and a workflow engine — and then we deliberately break them in the most instructive ways possible."
---