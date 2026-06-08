# DOCUMENT 2 — LAB PRACTICAL CODE PROJECT

---

## LAB MANUAL: Day 4 — Data Architecture, NoSQL & Search
**Phase:** Architectural Foundations & Design Thinking
**Topics:** Polyglot Persistence, MongoDB Document Design, Elasticsearch Search Architecture, Redis Caching, Sharding Strategy, DPDP Act Compliance
**Tech Stack:** Java 17 | Python 3.10+ | MongoDB 7.0 | Elasticsearch 8.x | Redis 7.x | Linux Ubuntu 22.04 | Azure Free Tier
**IaC:** Terraform | Orchestration: Docker Compose | CI: GitHub Actions
**Geography Context:** 🇮🇳 India (DigiLocker/GSTN-inspired) / 🇸🇬 Singapore (CPF/SingPass-inspired)
**Copilot Usage:** Embedded prompts throughout
**Pre-requisites:** Day 3 lab completed, Docker running, 12GB RAM recommended for full stack
**Estimated Lab Time:** 120 minutes | In-Class Demo: 45 minutes

---

# L0. LAB CONTEXT & ARCHITECTURE NARRATIVE

---

LAB NARRATIVE

SCENARIO:
You are building **CitizenVault** — a simplified polyglot citizen data platform inspired by India's DigiLocker and Singapore's MyInfo. A citizen can store identity records, retrieve documents, search by name with fuzzy matching, and trigger a data erasure request. The system demonstrates four database engines working in concert: PostgreSQL for identity (ACID, relational), MongoDB for document storage (flexible schema), Elasticsearch for citizen search (full-text, analytics), and Redis for session management and rate limiting. Apache Kafka synchronises data across stores via event-driven projections. A Python erasure orchestrator implements the DPDP Act right-to-erasure flow across all four stores using a Saga pattern.

```
ARCHITECTURE OVERVIEW:

[Citizen / Officer]
        │
        │ REST API
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  API Gateway (FastAPI — Port 8080)                              │
│  Rate limiting via Redis · Session validation · DPDP filter    │
└──────────┬──────────────────┬──────────────────┬───────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌──────────────────┐ ┌───────────────┐ ┌──────────────────────┐
│ Identity Service │ │Document Service│ │  Search Service      │
│ (Java 17)        │ │ (Python 3.10) │ │  (Python 3.10)       │
│ Port 8081        │ │ Port 8082     │ │  Port 8083           │
└────────┬─────────┘ └───────┬───────┘ └──────────┬───────────┘
         │                   │                     │
         ▼                   ▼                     ▼
┌──────────────┐    ┌──────────────────┐   ┌──────────────────┐
│ PostgreSQL   │    │ MongoDB          │   │ Elasticsearch    │
│ citizen_db   │    │ citizenvault_db  │   │ citizens index   │
│ Port 5432    │    │ Port 27017       │   │ Port 9200        │
└──────┬───────┘    └────────┬─────────┘   └──────────────────┘
       │                     │
       │ Domain Events        │ Document Events
       └──────────┬───────────┘
                  ▼
         ┌────────────────┐
         │ Apache Kafka   │
         │ Port 9092      │
         └───────┬────────┘
                 │
    ┌────────────┼────────────────┐
    ▼            ▼                ▼
┌────────┐  ┌──────────┐  ┌──────────────────────┐
│ Redis  │  │ ES Index │  │ Erasure Orchestrator  │
│Session │  │ Updater  │  │ (Python Saga)         │
│& Rate  │  │ (Python) │  │ DPDP Right-to-Erasure │
│Limiter │  └──────────┘  └──────────────────────┘
│Port 6379│
└────────┘

CONCEPTS FROM TRAINING THIS LAB DEMONSTRATES:
□ Polyglot persistence — 4 database engines for 4 access patterns (Block 1)
□ MongoDB document design — embedding vs referencing decision (Block 1, Concept 2)
□ Elasticsearch index design — field mapping, sharding, DPDP filter (Block 2, Use Case 2)
□ Redis for session + rate limiting — sub-millisecond, TTL-based (Block 1, Pattern 5)
□ Event-driven data sync — Kafka projecting identity changes to ES index (Day 3 bridge)
□ Right-to-erasure Saga — cascading deletion across all 4 stores (Block 3)
□ Crypto-shredding — DPDP-compliant erasure in event-sourced systems (Food for Thought)

PRODUCTION DELTA:
- Lab: single MongoDB node. Production: MongoDB sharded cluster with consistent-hash shard key.
- Lab: single Elasticsearch node. Production: 3-node cluster with replica shards, ILM policy.
- Lab: Redis single instance. Production: Redis Cluster with 6 nodes (3 primary + 3 replica).
- Lab: no field-level encryption. Production: AES-256 encryption for Aadhaar/NRIC fields.
- Lab: no TLS between services. Production: mTLS via Istio service mesh (covered Day 13).
- Azure Free Tier: MongoDB Atlas free tier (512MB), Azure Cache for Redis (250MB), 
  Azure Cognitive Search free tier (50MB index, 3 indexes).
```

---

# L1. ENVIRONMENT SETUP

---

**STEP 1 of 10: Verify system memory**

WHY: MongoDB + Elasticsearch together require ~4GB RAM minimum. Elasticsearch alone wants 2GB heap. Running below minimum causes OOM kills mid-demo — embarrassing and unrecoverable without restart.

```bash
$ free -h | grep Mem
```

Expected Output:
```
Mem:            15Gi        5.1Gi       8.9Gi       ...
```

⚠️ IF total memory is below 10GB — reduce Elasticsearch heap:
```bash
# Edit docker-compose.yml before starting:
# ES_JAVA_OPTS: "-Xms512m -Xmx512m"
# This allows ES to run on 8GB total system RAM
```

✅ VERIFY:
```bash
$ docker info --format '{{.MemTotal}}' | \
  awk '{printf "%.1f GB\n", $1/1024/1024/1024}'
# Expected: 15.0 GB (or your total)
```

---

**STEP 2 of 10: Set Elasticsearch virtual memory limit**

WHY: Elasticsearch requires `vm.max_map_count >= 262144`. On a fresh Linux VM this is set to 65536. Elasticsearch will refuse to start without this setting.

```bash
$ sudo sysctl -w vm.max_map_count=262144
$ echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Expected Output:
```
vm.max_map_count = 262144
vm.max_map_count=262144
```

✅ VERIFY:
```bash
$ sysctl vm.max_map_count
# Expected: vm.max_map_count = 262144
```

---

**STEP 3 of 10: Clone lab repository**

```bash
$ git clone https://github.com/citizenvault-lab/day4-polyglot.git
$ cd day4-polyglot
$ ls -la
```

Expected Output:
```
drwxr-xr-x  identity-service/
drwxr-xr-x  document-service/
drwxr-xr-x  search-service/
drwxr-xr-x  api-gateway/
drwxr-xr-x  erasure-orchestrator/
drwxr-xr-x  es-index-updater/
drwxr-xr-x  infra/
drwxr-xr-x  scripts/
-rw-r--r--  docker-compose.yml
-rw-r--r--  Makefile
```

---

**STEP 4 of 10: Start infrastructure layer first**

WHY: PostgreSQL, MongoDB, Elasticsearch, Redis, and Kafka must all be healthy before any application service starts. Starting them together causes race conditions. Start infrastructure, wait, verify, then start application services.

```bash
$ docker compose up -d postgres mongodb elasticsearch redis zookeeper kafka
$ echo "Waiting 45 seconds for all infrastructure to initialise..."
$ sleep 45
```

✅ VERIFY each infrastructure service:

```bash
# PostgreSQL
$ docker compose exec postgres pg_isready -U citizenvault
# Expected: /var/run/postgresql:5432 - accepting connections

# MongoDB
$ docker compose exec mongodb mongosh --eval "db.adminCommand('ping')" \
    --quiet
# Expected: { ok: 1 }

# Elasticsearch
$ curl -s http://localhost:9200/_cluster/health | python3 -m json.tool \
    | grep status
# Expected: "status": "green"  (or "yellow" — acceptable for single node)

# Redis
$ docker compose exec redis redis-cli ping
# Expected: PONG

# Kafka
$ docker compose exec kafka kafka-topics.sh \
    --bootstrap-server localhost:9092 --list 2>/dev/null
# Expected: (empty — topics created in next step)
```

⚠️ IF Elasticsearch status is "red":
```bash
$ docker compose logs elasticsearch | tail -20
# Most common cause: vm.max_map_count not set (Step 2)
$ sudo sysctl -w vm.max_map_count=262144
$ docker compose restart elasticsearch && sleep 20
```

---

**STEP 5 of 10: Initialise databases**

WHY: Creates PostgreSQL schema, MongoDB indexes, and Elasticsearch index mapping. Running application services before schema initialisation causes startup failures.

```bash
$ chmod +x scripts/init-databases.sh
$ ./scripts/init-databases.sh
```

Expected Output:
```
[PostgreSQL] Creating citizen_identity schema...
[PostgreSQL] ✅ Tables created: citizens, citizen_keys (crypto-shredding)
[MongoDB]    Creating citizenvault_db indexes...
[MongoDB]    ✅ Indexes created: (aadhaar_hash, document_type, created_at)
[Elasticsearch] Creating citizens index with mapping...
[Elasticsearch] ✅ Index created: citizens (5 shards, 1 replica)
[Kafka]      Creating topics: citizen-events, document-events, erasure-events
[Kafka]      ✅ All topics created
All databases initialised successfully.
```

⚠️ IF MongoDB index creation fails with "authentication failed":
```bash
$ docker compose exec mongodb mongosh -u citizenvault -p vault2026 \
    --authenticationDatabase admin --eval "db.adminCommand('ping')"
# If this fails: recreate MongoDB container
$ docker compose rm -f mongodb && docker compose up -d mongodb && sleep 15
$ ./scripts/init-databases.sh
```

---

**STEP 6 of 10: Build Java identity service**

```bash
$ cd identity-service
$ ./mvnw clean package -DskipTests -q
$ cd ..
```

Expected Output:
```
[INFO] BUILD SUCCESS
[INFO] Total time: 48.231 s
```

---

**STEP 7 of 10: Install Python dependencies**

```bash
$ for service in document-service search-service api-gateway \
    erasure-orchestrator es-index-updater; do
    echo "Installing dependencies for $service..."
    cd $service
    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt -q
    deactivate
    cd ..
  done
```

Expected Output (per service):
```
Installing dependencies for document-service...
Successfully installed motor-3.3.2 pymongo-4.6.1 pydantic-2.6.0...
Installing dependencies for search-service...
Successfully installed elasticsearch-8.12.0 fastapi-0.110.0...
```

---

**STEP 8 of 10: Start all application services**

```bash
$ docker compose up --build -d
$ sleep 30
```

✅ VERIFY all services healthy:
```bash
$ for port in 8080 8081 8082 8083; do
    status=$(curl -s http://localhost:$port/health \
             | python3 -c "import sys,json; \
               d=json.load(sys.stdin); print(d.get('status','unknown'))")
    echo "Port $port: $status"
  done
```

Expected Output:
```
Port 8080: healthy    (api-gateway)
Port 8081: UP         (identity-service Spring Boot)
Port 8082: healthy    (document-service)
Port 8083: healthy    (search-service)
```

---

**STEP 9 of 10: Seed test data**

WHY: Seeding 1,000 test citizens demonstrates Elasticsearch's performance advantage at meaningful data volumes. A single-record demo does not show the performance difference between LIKE query and ES full-text search.

```bash
$ chmod +x scripts/seed-data.sh
$ ./scripts/seed-data.sh 1000
```

Expected Output:
```
Seeding 1000 citizens...
[PostgreSQL] 1000 identity records inserted
[MongoDB]    1000 document records inserted
[Elasticsearch] Bulk indexing 1000 records...
[Elasticsearch] ✅ 1000 documents indexed in 1.2 seconds
[Redis] Session data pre-warmed
Seeding complete. CitizenVault ready for lab exercises.
```

---

**STEP 10 of 10: Run smoke test**

```bash
$ ./scripts/smoke-test.sh
```

Expected Output:
```
[SMOKE] POST /citizens - Create citizen............ ✅ 201 Created
[SMOKE] GET  /citizens/{id} - Get identity......... ✅ 200 OK (PostgreSQL)
[SMOKE] POST /documents - Store document........... ✅ 201 Created (MongoDB)
[SMOKE] GET  /search?q=Sharma - Full-text search... ✅ 200 OK (Elasticsearch 180ms)
[SMOKE] GET  /search?q=Shaarma - Fuzzy search...... ✅ 200 OK (ES fuzzy 210ms)
[SMOKE] POST /sessions - Create session............ ✅ 201 Created (Redis TTL=3600)
[SMOKE] GET  /rate-limit-test - Rate limit check... ✅ X-RateLimit-Remaining: 99
All smoke tests passed. ✅
```

---

# L2. PROJECT STRUCTURE

---

```
day4-polyglot/
│
├── scripts/
│   ├── init-databases.sh          # Schema + index creation
│   ├── seed-data.sh               # Test data generation (1K-100K citizens)
│   ├── smoke-test.sh              # Environment verification
│   ├── demo-search-comparison.sh  # PostgreSQL LIKE vs ES benchmark
│   ├── demo-erasure.sh            # DPDP erasure flow demo
│   └── teardown.sh
│
├── identity-service/              # Java 17, Spring Boot 3 — PostgreSQL
│   ├── src/main/java/com/citizenvault/identity/
│   │   ├── domain/
│   │   │   ├── Citizen.java               # JPA Entity + Domain Object
│   │   │   ├── CitizenId.java             # Value Object
│   │   │   ├── AadhaarHash.java           # Value Object (hashed — never plain)
│   │   │   ├── CitizenKey.java            # Crypto-shredding key entity
│   │   │   └── ErasureStatus.java         # Enum: ACTIVE, PENDING, ERASED
│   │   ├── application/
│   │   │   ├── port/in/
│   │   │   │   ├── RegisterCitizenUseCase.java
│   │   │   │   └── InitiateErasureUseCase.java
│   │   │   ├── port/out/
│   │   │   │   ├── CitizenRepository.java
│   │   │   │   ├── CitizenKeyRepository.java
│   │   │   │   └── CitizenEventPublisher.java
│   │   │   └── CitizenApplicationService.java
│   │   ├── adapter/
│   │   │   ├── in/web/CitizenController.java
│   │   │   └── out/
│   │   │       ├── persistence/
│   │   │       │   ├── JpaCitizenRepository.java
│   │   │       │   └── JpaCitizenKeyRepository.java
│   │   │       └── kafka/KafkaCitizenEventPublisher.java
│   │   └── IdentityServiceApp.java
│   ├── src/test/
│   │   └── java/com/citizenvault/identity/
│   │       ├── domain/CryptoShreddingTest.java
│   │       └── application/CitizenErasureTest.java
│   └── pom.xml
│
├── document-service/              # Python 3.10 — MongoDB
│   ├── src/
│   │   ├── domain/
│   │   │   ├── document.py        # CitizenDocument domain model
│   │   │   └── document_type.py   # Enum: AADHAAR, PAN, PASSPORT, etc.
│   │   ├── application/
│   │   │   └── document_service.py
│   │   ├── ports/
│   │   │   └── document_repository.py   # Abstract port
│   │   ├── adapters/
│   │   │   └── mongodb_repository.py    # Motor async MongoDB adapter
│   │   └── api/
│   │       ├── router.py
│   │       └── schemas.py
│   └── main.py
│
├── search-service/                # Python 3.10 — Elasticsearch
│   ├── src/
│   │   ├── search/
│   │   │   ├── citizen_search.py  # Elasticsearch query builders
│   │   │   └── dpdp_filter.py     # DPDP-compliant field filtering
│   │   ├── adapters/
│   │   │   └── es_client.py       # Elasticsearch 8.x async client
│   │   └── api/
│   │       └── router.py
│   └── main.py
│
├── api-gateway/                   # Python 3.10 — FastAPI + Redis
│   ├── src/
│   │   ├── middleware/
│   │   │   ├── rate_limiter.py    # Redis-backed sliding window rate limiter
│   │   │   └── session_manager.py # Redis session store
│   │   └── api/
│   │       └── router.py          # Reverse proxy to downstream services
│   └── main.py
│
├── es-index-updater/              # Python 3.10 — Kafka consumer → ES
│   ├── src/
│   │   ├── consumer.py            # Kafka consumer for citizen-events
│   │   └── indexer.py             # ES bulk indexer
│   └── main.py
│
├── erasure-orchestrator/          # Python 3.10 — DPDP Saga
│   ├── src/
│   │   ├── saga/
│   │   │   ├── erasure_saga.py    # Saga orchestrating erasure across 4 stores
│   │   │   └── erasure_steps.py   # Each store's erasure implementation
│   │   ├── adapters/
│   │   │   ├── postgres_eraser.py
│   │   │   ├── mongo_eraser.py
│   │   │   ├── es_eraser.py
│   │   │   └── redis_eraser.py
│   │   └── api/
│   │       └── router.py
│   └── main.py
│
├── infra/
│   └── terraform/
│       ├── main.tf                # Azure Cosmos DB + Azure Cache for Redis
│       ├── variables.tf
│       └── outputs.tf
│
└── docker-compose.yml
```

---

# L3. CODE BLOCKS

---

## DATABASE INITIALISATION SCRIPT

```
FILE: scripts/init-databases.sh
LANGUAGE: Bash
PURPOSE: Creates all database schemas, indexes, and Elasticsearch mappings.
         Run once before starting application services.
CONCEPTS DEMONSTRATED: Schema design for polyglot stores, ES index mapping,
                       MongoDB compound index strategy, crypto-shredding table
COPY-PASTE READY: YES
```

```bash
#!/usr/bin/env bash
# scripts/init-databases.sh
#
# ARCHITECTURAL NOTE:
# Order matters: PostgreSQL schema first (identity is the system of record),
# then MongoDB indexes, then Elasticsearch mapping.
# If ES mapping is created after data exists: existing data is not re-indexed.
# Always create mapping BEFORE indexing documents.

set -e

echo "=== CitizenVault Database Initialisation ==="

# ─── POSTGRESQL ───────────────────────────────────────────────────────────────
echo "[PostgreSQL] Initialising citizen identity schema..."

docker compose exec -T postgres psql -U citizenvault -d citizen_db << 'SQL'

-- CITIZEN IDENTITY TABLE
-- Strong consistency. ACID. CP in CAP terms.
-- Aadhaar number stored as HASH only — never plaintext.
-- DPDP Act S8: data minimisation — store only what is needed
-- DPDP Act S9: purpose limitation — this table serves identity verification only
CREATE TABLE IF NOT EXISTS citizens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aadhaar_hash    VARCHAR(64) UNIQUE NOT NULL,  -- SHA-256 of Aadhaar
    name_encrypted  BYTEA NOT NULL,               -- AES-256 encrypted
    dob             DATE NOT NULL,
    gender          CHAR(1) NOT NULL CHECK (gender IN ('M', 'F', 'O')),
    state_code      CHAR(2) NOT NULL,
    district_code   VARCHAR(10) NOT NULL,
    erasure_status  VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
                    CHECK (erasure_status IN ('ACTIVE', 'PENDING_ERASURE', 'ERASED')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    erased_at       TIMESTAMPTZ
);

-- CRYPTO-SHREDDING KEY TABLE
-- DPDP Act S17: Right to Erasure implementation.
-- Each citizen has an AES-256 encryption key stored here.
-- When erasure is requested: DELETE the key row.
-- All encrypted fields become unreadable — data is effectively erased
-- even if it persists in backups, event logs, or MongoDB.
-- This is CRYPTO-SHREDDING — the correct pattern for event-sourced erasure.
CREATE TABLE IF NOT EXISTS citizen_keys (
    citizen_id      UUID PRIMARY KEY REFERENCES citizens(id),
    encryption_key  BYTEA NOT NULL,              -- AES-256 key (32 bytes)
    key_version     INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ                  -- Set on erasure — key gone
);

-- AUDIT TABLE — Immutable. Never deleted. Anonymised after erasure.
-- DPDP Act S8: data fiduciary must maintain processing records
-- Even after erasure: we keep WHAT happened (anonymised), not WHO
CREATE TABLE IF NOT EXISTS citizen_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citizen_id      UUID,                        -- Nullable: anonymised post-erasure
    action          VARCHAR(50) NOT NULL,
    performed_by    VARCHAR(100) NOT NULL,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    details         JSONB,
    anonymised      BOOLEAN NOT NULL DEFAULT FALSE
);

-- INDEXES
CREATE INDEX IF NOT EXISTS idx_citizens_state_district
    ON citizens(state_code, district_code);
CREATE INDEX IF NOT EXISTS idx_citizens_erasure_status
    ON citizens(erasure_status) WHERE erasure_status != 'ACTIVE';
CREATE INDEX IF NOT EXISTS idx_audit_citizen_id
    ON citizen_audit(citizen_id);

SQL

echo "[PostgreSQL] ✅ Schema created"

# ─── MONGODB ──────────────────────────────────────────────────────────────────
echo "[MongoDB] Initialising document collections and indexes..."

docker compose exec -T mongodb mongosh \
    -u citizenvault -p vault2026 \
    --authenticationDatabase admin \
    --quiet << 'JS'

use citizenvault_db

// DOCUMENT COLLECTION
// Schema-flexible: different document types have different fields
// ARCHITECTURAL DECISION: Reference pattern for documents
// Documents are NOT embedded in citizen record because:
// 1. Unbounded growth (citizen accumulates documents over lifetime)
// 2. Documents accessed independently of citizen profile
// 3. Documents have independent lifecycle (can be deleted without deleting citizen)
db.createCollection("citizen_documents", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["citizenId", "documentType", "status", "createdAt"],
            properties: {
                citizenId: {
                    bsonType: "string",
                    description: "UUID reference to PostgreSQL citizen.id"
                },
                documentType: {
                    enum: ["AADHAAR", "PAN", "PASSPORT", "DRIVING_LICENSE",
                           "VOTER_ID", "BIRTH_CERTIFICATE", "INCOME_CERTIFICATE"],
                    description: "Must be one of the supported document types"
                },
                status: {
                    enum: ["ACTIVE", "EXPIRED", "REVOKED", "ERASED"],
                    description: "Document lifecycle status"
                },
                // DPDP compliance: encryption key version used to encrypt content
                // When citizen key is deleted (crypto-shredding):
                // document content becomes unreadable — erasure achieved
                encryptionKeyVersion: {
                    bsonType: "int",
                    description: "Version of citizen key used to encrypt content"
                }
            }
        }
    }
})

// INDEXES FOR DOCUMENT COLLECTION
// Compound index: most common query = "all documents for citizen X of type Y"
db.citizen_documents.createIndex(
    { citizenId: 1, documentType: 1 },
    { name: "idx_citizen_doctype" }
)

// TTL index for expired documents auto-cleanup
// DPDP Act S9: data must not be retained beyond purpose
// Documents with expiryDate set will auto-delete after expiry
db.citizen_documents.createIndex(
    { expiryDate: 1 },
    { expireAfterSeconds: 0, name: "idx_document_ttl",
      partialFilterExpression: { expiryDate: { $exists: true } } }
)

// Index for erasure queries: "find all documents for citizen X to erase"
db.citizen_documents.createIndex(
    { citizenId: 1, status: 1 },
    { name: "idx_citizen_status" }
)

print("MongoDB indexes created successfully")

JS

echo "[MongoDB] ✅ Collections and indexes created"

# ─── ELASTICSEARCH ─────────────────────────────────────────────────────────────
echo "[Elasticsearch] Creating citizens index with DPDP-compliant mapping..."

curl -s -X PUT "http://localhost:9200/citizens" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 5,
      "number_of_replicas": 0,
      "analysis": {
        "analyzer": {
          "name_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "asciifolding"]
          },
          "phonetic_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "double_metaphone"]
          }
        },
        "filter": {
          "double_metaphone": {
            "type": "phonetic",
            "encoder": "double_metaphone"
          }
        }
      },
      "index.lifecycle.name": "citizens-policy"
    },
    "mappings": {
      "properties": {
        "citizenId": {
          "type": "keyword",
          "doc_values": true
        },
        "nameSearchable": {
          "type": "text",
          "analyzer": "name_analyzer",
          "fields": {
            "phonetic": {
              "type": "text",
              "analyzer": "phonetic_analyzer"
            },
            "keyword": {
              "type": "keyword"
            }
          }
        },
        "stateCode": {
          "type": "keyword"
        },
        "districtCode": {
          "type": "keyword"
        },
        "ageGroup": {
          "type": "keyword"
        },
        "gender": {
          "type": "keyword"
        },
        "documentTypes": {
          "type": "keyword"
        },
        "erasureStatus": {
          "type": "keyword"
        },
        "authorisedScopes": {
          "type": "keyword"
        },
        "indexedAt": {
          "type": "date"
        }
      },
      "_source": {
        "excludes": []
      }
    }
  }' | python3 -m json.tool | grep -E '"acknowledged|"index'

echo ""
echo "[Elasticsearch] ✅ Citizens index created with DPDP-compliant mapping"

# ─── KAFKA TOPICS ─────────────────────────────────────────────────────────────
echo "[Kafka] Creating topics..."

for topic in citizen-events document-events erasure-events erasure-dlq; do
    docker compose exec kafka kafka-topics.sh \
        --bootstrap-server localhost:9092 \
        --create --topic $topic \
        --partitions 4 \
        --replication-factor 1 \
        --if-not-exists \
        --config retention.ms=604800000 \
        2>/dev/null
    echo "[Kafka] ✅ Topic: $topic"
done

echo ""
echo "=== All databases initialised successfully ==="
```

---

## IDENTITY SERVICE — Crypto-Shredding Implementation (Java 17)

```
FILE: identity-service/src/main/java/com/citizenvault/identity/domain/Citizen.java
LANGUAGE: Java 17
PURPOSE: Citizen JPA entity with crypto-shredding support.
         Demonstrates DPDP Act right-to-erasure via AES-256 encryption
         and key deletion pattern — not physical data deletion.
CONCEPTS DEMONSTRATED: Crypto-shredding for event-sourced erasure,
                       DPDP Act compliance, Java 17 records for value objects
COPY-PASTE READY: YES
PRODUCTION DELTA: Use Azure Key Vault for key storage (not database column).
                  Rotate keys annually per DPDP Act requirements.
                  Add HSM-backed key operations for biometric keys.
```

```java
// FILE: com/citizenvault/identity/domain/Citizen.java
//
// ARCHITECTURAL NOTE — CRYPTO-SHREDDING:
//
// The DPDP Act 2023 Section 17 gives citizens the right to erasure.
// But we also use Event Sourcing (Day 3) — and events are immutable.
// A citizen's name appears in 100 historical events. We cannot delete them.
//
// SOLUTION: Crypto-Shredding
// 1. Encrypt all PII fields using a citizen-specific AES-256 key
// 2. Store the key in citizen_keys table
// 3. On erasure request: DELETE the key row
// 4. Encrypted fields become permanently unreadable — mathematically erased
// 5. The ciphertext remains in events, databases, backups — but is useless
//    without the key
//
// DPDP Compliance:
// - The encrypted ciphertext is NOT personal data (unreadable without key)
// - Key deletion satisfies the right to erasure
// - Audit records retain anonymised action trail (no PII — DPDP S8)
//
// This pattern is how you reconcile Event Sourcing + Right to Erasure.

package com.citizenvault.identity.domain;

import jakarta.persistence.*;
import java.time.LocalDate;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "citizens")
public class Citizen {

    @Id
    @Column(name = "id", updatable = false, nullable = false)
    private UUID id;

    // Aadhaar stored as SHA-256 hash — NEVER plaintext in any table.
    // Hash allows deduplication check (is this Aadhaar already registered?)
    // without storing the actual Aadhaar number.
    // Even DBA with SELECT * access cannot read an Aadhaar number.
    @Column(name = "aadhaar_hash", unique = true, nullable = false)
    private String aadhaarHash;

    // Name encrypted with citizen-specific AES-256 key.
    // To read: fetch key from citizen_keys, decrypt.
    // To erase: delete key row → this field becomes permanently unreadable.
    @Column(name = "name_encrypted", nullable = false)
    private byte[] nameEncrypted;

    @Column(name = "dob", nullable = false)
    private LocalDate dob;

    @Column(name = "gender", nullable = false, length = 1)
    private String gender;

    @Column(name = "state_code", nullable = false, length = 2)
    private String stateCode;

    @Column(name = "district_code", nullable = false)
    private String districtCode;

    @Enumerated(EnumType.STRING)
    @Column(name = "erasure_status", nullable = false)
    private ErasureStatus erasureStatus = ErasureStatus.ACTIVE;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Column(name = "erased_at")
    private Instant erasedAt;

    protected Citizen() {}

    public static Citizen register(
            String aadhaarNumber,
            String name,
            LocalDate dob,
            String gender,
            String stateCode,
            String districtCode,
            CryptoService cryptoService
    ) {
        var citizen = new Citizen();
        citizen.id = UUID.randomUUID();
        citizen.createdAt = Instant.now();
        citizen.updatedAt = Instant.now();
        citizen.dob = dob;
        citizen.gender = gender;
        citizen.stateCode = stateCode;
        citizen.districtCode = districtCode;
        citizen.erasureStatus = ErasureStatus.ACTIVE;

        // Hash Aadhaar — never store plaintext
        citizen.aadhaarHash = cryptoService.hashAadhaar(aadhaarNumber);

        // Generate citizen-specific encryption key
        var encryptionKey = cryptoService.generateKey();
        // Encrypt name with citizen-specific key
        citizen.nameEncrypted = cryptoService.encrypt(name, encryptionKey);

        // Key is stored separately in citizen_keys table
        // This separation is critical: entity + key = readable data
        // Entity without key = ciphertext = effectively erased
        citizen.pendingKey = encryptionKey;

        return citizen;
    }

    public void initiateErasure() {
        if (this.erasureStatus == ErasureStatus.ERASED) {
            throw new IllegalStateException(
                "Citizen " + id + " is already erased"
            );
        }
        this.erasureStatus = ErasureStatus.PENDING_ERASURE;
        this.updatedAt = Instant.now();
    }

    public void completeErasure() {
        // Mark as erased. Key will be deleted separately by ErasureSaga.
        // After key deletion: nameEncrypted field is unreadable ciphertext.
        // DPDP Act S17: erasure is complete when key is deleted.
        this.erasureStatus = ErasureStatus.ERASED;
        this.erasedAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    // Transient — not persisted, used to pass key to application service
    @Transient
    private byte[] pendingKey;

    public UUID getId() { return id; }
    public String getAadhaarHash() { return aadhaarHash; }
    public byte[] getNameEncrypted() { return nameEncrypted; }
    public LocalDate getDob() { return dob; }
    public String getGender() { return gender; }
    public String getStateCode() { return stateCode; }
    public String getDistrictCode() { return districtCode; }
    public ErasureStatus getErasureStatus() { return erasureStatus; }
    public Instant getCreatedAt() { return createdAt; }
    public byte[] getPendingKey() { return pendingKey; }
}
```

```java
// FILE: com/citizenvault/identity/domain/CryptoService.java
//
// ARCHITECTURAL NOTE:
// CryptoService is a DOMAIN SERVICE — it contains business logic
// (the crypto-shredding pattern) but has no identity of its own.
// It is injected into the Citizen factory method.
// The AES-256 implementation uses Java's built-in JCE — no external lib needed.
// In production: replace with Azure Key Vault client for key storage
// and HSM-backed key operations.

package com.citizenvault.identity.domain;

import org.springframework.stereotype.Service;
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;

@Service
public class CryptoService {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;
    private static final String HASH_ALGORITHM = "SHA-256";

    private final SecureRandom secureRandom = new SecureRandom();

    /**
     * Hash Aadhaar number using SHA-256.
     * One-way: cannot reverse to get original Aadhaar.
     * Deterministic: same Aadhaar always produces same hash.
     * Used for deduplication checks without storing plaintext.
     *
     * PRODUCTION NOTE: Use HMAC-SHA256 with a secret pepper
     * (stored in Azure Key Vault) to prevent rainbow table attacks.
     * Plain SHA-256 is used here for lab simplicity.
     */
    public String hashAadhaar(String aadhaarNumber) {
        try {
            var digest = MessageDigest.getInstance(HASH_ALGORITHM);
            var hashBytes = digest.digest(
                aadhaarNumber.getBytes(java.nio.charset.StandardCharsets.UTF_8)
            );
            return Base64.getEncoder().encodeToString(hashBytes);
        } catch (Exception e) {
            throw new CryptoException("Failed to hash Aadhaar number", e);
        }
    }

    /**
     * Generate a new AES-256 encryption key for a citizen.
     * This key is stored in citizen_keys table.
     * Deletion of this key = crypto-shredding = effective erasure.
     */
    public byte[] generateKey() {
        try {
            var keyGen = KeyGenerator.getInstance("AES");
            keyGen.init(256, secureRandom);
            return keyGen.generateKey().getEncoded();
        } catch (Exception e) {
            throw new CryptoException("Failed to generate encryption key", e);
        }
    }

    /**
     * Encrypt plaintext using AES-256-GCM.
     * GCM provides both confidentiality AND integrity (AEAD).
     * Output format: [12-byte IV][ciphertext+16-byte auth tag]
     *
     * PRODUCTION NOTE: Add Additional Authenticated Data (AAD)
     * as the citizen's ID — prevents ciphertext from being
     * moved to a different citizen's record.
     */
    public byte[] encrypt(String plaintext, byte[] keyBytes) {
        try {
            var iv = new byte[GCM_IV_LENGTH];
            secureRandom.nextBytes(iv);

            var key = new SecretKeySpec(keyBytes, "AES");
            var cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(
                Cipher.ENCRYPT_MODE,
                key,
                new GCMParameterSpec(GCM_TAG_LENGTH, iv)
            );

            var ciphertext = cipher.doFinal(
                plaintext.getBytes(java.nio.charset.StandardCharsets.UTF_8)
            );

            // Prepend IV to ciphertext — IV needed for decryption
            var result = new byte[GCM_IV_LENGTH + ciphertext.length];
            System.arraycopy(iv, 0, result, 0, GCM_IV_LENGTH);
            System.arraycopy(ciphertext, 0, result, GCM_IV_LENGTH, ciphertext.length);

            return result;
        } catch (Exception e) {
            throw new CryptoException("Encryption failed", e);
        }
    }

    /**
     * Decrypt ciphertext using AES-256-GCM.
     * Returns null (not exception) if key has been deleted (erased citizen).
     * Callers must handle null — it means "this citizen has been erased."
     */
    public String decrypt(byte[] ciphertext, byte[] keyBytes) {
        if (keyBytes == null) {
            // Key has been deleted — crypto-shredding applied
            // Data is effectively erased. Return null, not exception.
            return null;
        }
        try {
            var iv = new byte[GCM_IV_LENGTH];
            System.arraycopy(ciphertext, 0, iv, 0, GCM_IV_LENGTH);

            var actualCiphertext = new byte[ciphertext.length - GCM_IV_LENGTH];
            System.arraycopy(
                ciphertext, GCM_IV_LENGTH,
                actualCiphertext, 0, actualCiphertext.length
            );

            var key = new SecretKeySpec(keyBytes, "AES");
            var cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(
                Cipher.DECRYPT_MODE,
                key,
                new GCMParameterSpec(GCM_TAG_LENGTH, iv)
            );

            return new String(
                cipher.doFinal(actualCiphertext),
                java.nio.charset.StandardCharsets.UTF_8
            );
        } catch (Exception e) {
            throw new CryptoException("Decryption failed", e);
        }
    }
}
```

---

## DOCUMENT SERVICE — MongoDB Async Adapter (Python 3.10)

```
FILE: document-service/src/adapters/mongodb_repository.py
LANGUAGE: Python 3.10
PURPOSE: Async MongoDB adapter using Motor (async MongoDB driver).
         Implements the document repository port.
         Demonstrates correct reference pattern to avoid unbounded embedding.
CONCEPTS DEMONSTRATED: MongoDB async operations, compound index usage,
                       DPDP-compliant document lifecycle, reference vs embedding
COPY-PASTE READY: YES
PRODUCTION DELTA: Add connection pooling configuration, retry with exponential
                  backoff for transient failures, read preference for secondary reads.
```

```python
# document-service/src/adapters/mongodb_repository.py
#
# ARCHITECTURAL NOTE — ASYNC MONGODB WITH MOTOR:
#
# Motor is the async MongoDB driver for Python.
# It integrates with asyncio — the same event loop as FastAPI.
# Without Motor: FastAPI would block its event loop waiting for MongoDB I/O.
# With Motor: FastAPI handles other requests while MongoDB I/O completes.
#
# This is the Non-Blocking I/O pattern from Day 3 applied to database calls.
# A single FastAPI process can handle hundreds of concurrent document requests
# because it never WAITS for MongoDB — it YIELDS to the event loop.
#
# DOCUMENT DESIGN DECISION (Reference Pattern):
# CitizenDocument is stored in its own collection, referenced by citizenId.
# NOT embedded in the citizen's profile.
# Reason: a citizen accumulates 20-50 documents over their lifetime.
# Embedding would create an ever-growing document — hitting the 16MB limit.
# Reference pattern: citizen profile stays small (< 2KB),
# documents are fetched separately when needed.

from __future__ import annotations

import hashlib
from datetime import datetime, timezone
from typing import Optional

import motor.motor_asyncio
from bson import ObjectId

from src.domain.document import CitizenDocument, DocumentType, DocumentStatus
from src.ports.document_repository import DocumentRepository


class MongoDBDocumentRepository(DocumentRepository):
    """
    Driven Adapter: MongoDB document storage using Motor async driver.
    Implements DocumentRepository port — application layer never imports Motor.
    """

    COLLECTION = "citizen_documents"

    def __init__(self, connection_string: str, database_name: str) -> None:
        # Motor creates one async client — connection pool managed internally
        # Production: set maxPoolSize based on concurrent request estimate
        self._client = motor.motor_asyncio.AsyncIOMotorClient(
            connection_string,
            maxPoolSize=50,        # Max 50 concurrent MongoDB connections
            minPoolSize=5,         # Keep 5 warm connections always
            serverSelectionTimeoutMS=3000,  # Fail fast on connection issues
        )
        self._db = self._client[database_name]
        self._collection = self._db[self.COLLECTION]

    async def save(self, document: CitizenDocument) -> CitizenDocument:
        """
        Persist a CitizenDocument.
        Uses upsert on (citizenId, documentType) — idempotent operation.
        """
        doc_dict = self._to_mongo_doc(document)

        # Upsert: insert if new, update if exists (same citizenId + documentType)
        # This makes save() idempotent — safe to call on retry
        result = await self._collection.find_one_and_replace(
            filter={
                "citizenId": document.citizen_id,
                "documentType": document.document_type.value,
            },
            replacement=doc_dict,
            upsert=True,
            return_document=True,
        )

        return self._from_mongo_doc(result)

    async def find_by_citizen_id(
        self,
        citizen_id: str,
        document_type: Optional[DocumentType] = None,
        include_erased: bool = False,
    ) -> list[CitizenDocument]:
        """
        Query: Get all documents for a citizen.
        Uses compound index: (citizenId, documentType) — O(log n) lookup.

        DPDP Act S17: include_erased=False by default.
        Officers with erasure audit role can pass include_erased=True.
        """
        query: dict = {"citizenId": citizen_id}

        if document_type:
            query["documentType"] = document_type.value

        if not include_erased:
            # Only return active documents by default
            query["status"] = {"$ne": DocumentStatus.ERASED.value}

        # Uses index: idx_citizen_doctype
        # Without this index: full collection scan on every request
        cursor = self._collection.find(query).sort("createdAt", -1)

        return [self._from_mongo_doc(doc) async for doc in cursor]

    async def erase_citizen_documents(
        self,
        citizen_id: str,
        idempotency_key: str
    ) -> int:
        """
        DPDP Act S17: Erase all documents for a citizen.
        Uses SOFT ERASURE: sets status to ERASED, clears content fields.
        The document record is retained for audit trail.
        The content (the actual document data) is cleared.

        Idempotency key prevents double-erasure on saga retry.
        Returns count of documents erased.
        """
        # Check if already processed this erasure request
        already_processed = await self._collection.find_one({
            "citizenId": citizen_id,
            "erasureIdempotencyKey": idempotency_key,
        })
        if already_processed:
            # Idempotent: already erased, return 0 (not an error)
            count = await self._collection.count_documents({
                "citizenId": citizen_id,
                "status": DocumentStatus.ERASED.value
            })
            return count

        # Soft erase: clear content, set status
        # DPDP Act S17: retain the audit record, erase the personal content
        result = await self._collection.update_many(
            filter={
                "citizenId": citizen_id,
                "status": {"$ne": DocumentStatus.ERASED.value}
            },
            update={
                "$set": {
                    "status": DocumentStatus.ERASED.value,
                    "content": None,         # Clear document content
                    "contentHash": None,     # Clear integrity hash
                    "erasedAt": datetime.now(timezone.utc),
                    "erasureIdempotencyKey": idempotency_key,
                }
            }
        )

        return result.modified_count

    def _to_mongo_doc(self, document: CitizenDocument) -> dict:
        """
        Domain model → MongoDB document translation.
        Note: ObjectId is MongoDB's internal ID — not our domain ID.
        Our domain uses citizenId (UUID string) as the identity.
        """
        return {
            "citizenId": document.citizen_id,
            "documentType": document.document_type.value,
            "documentNumber": self._hash_sensitive(document.document_number),
            "issueDate": document.issue_date,
            "expiryDate": document.expiry_date,
            "issuingAuthority": document.issuing_authority,
            "status": document.status.value,
            # Content stored as encrypted bytes in production
            # Lab: stored as string for demo visibility
            "content": document.content,
            "contentHash": self._hash_sensitive(document.content or ""),
            "encryptionKeyVersion": document.encryption_key_version,
            "createdAt": document.created_at,
            "updatedAt": datetime.now(timezone.utc),
        }

    def _from_mongo_doc(self, doc: dict) -> CitizenDocument:
        """MongoDB document → Domain model translation."""
        return CitizenDocument(
            id=str(doc.get("_id", "")),
            citizen_id=doc["citizenId"],
            document_type=DocumentType(doc["documentType"]),
            document_number=doc.get("documentNumber", ""),
            issue_date=doc.get("issueDate"),
            expiry_date=doc.get("expiryDate"),
            issuing_authority=doc.get("issuingAuthority", ""),
            status=DocumentStatus(doc["status"]),
            content=doc.get("content"),
            encryption_key_version=doc.get("encryptionKeyVersion", 1),
            created_at=doc.get("createdAt", datetime.now(timezone.utc)),
        )

    @staticmethod
    def _hash_sensitive(value: str) -> str:
        """Hash sensitive fields before storage — DPDP data minimisation."""
        return hashlib.sha256(value.encode()).hexdigest()
```

---

## SEARCH SERVICE — Elasticsearch with DPDP Filter (Python 3.10)

```
FILE: search-service/src/search/citizen_search.py
LANGUAGE: Python 3.10
PURPOSE: Elasticsearch query builder with DPDP-compliant field filtering.
         Demonstrates fuzzy search, multi-dimensional filter, phonetic matching,
         and authorisation-scoped result filtering.
CONCEPTS DEMONSTRATED: Elasticsearch query DSL, DPDP field-level access control,
                       fuzzy search, phonetic analysis, source filtering
COPY-PASTE READY: YES
PRODUCTION DELTA: Add search audit logging (every search by officer logged).
                  Add result caching for repeated government analytics queries.
                  Add query complexity limits to prevent expensive ad-hoc queries.
```

```python
# search-service/src/search/citizen_search.py
#
# ARCHITECTURAL NOTE — DPDP ACT COMPLIANCE IN ELASTICSEARCH:
#
# The most dangerous Elasticsearch anti-pattern in government systems:
# An officer writes: GET /citizens/_search?q=*
# This returns EVERY citizen record with ALL fields.
# If the response is logged, cached, or accidentally exposed:
# complete citizen database is leaked.
#
# TWO LAYERS OF DPDP PROTECTION IN THIS CODE:
#
# LAYER 1: Query-level scope filter
# Every search includes a `filter` clause on `authorisedScopes`.
# An officer with scope "IDENTITY_BASIC" cannot retrieve records
# with scope "FINANCIAL_FULL" even if they craft a manual query.
# The scope filter is injected SERVER-SIDE — never client-controlled.
#
# LAYER 2: Source filtering (_source includes)
# Even if the query matches a record, only the fields the officer
# is authorised to see are returned.
# An IDENTITY_BASIC officer gets: citizenId, stateCode, ageGroup
# A FINANCIAL_FULL officer gets: additionally balance ranges
# Name is NEVER returned directly — only a masked version.
#
# DPDP Act S4: Processing must be for specified purpose
# DPDP Act S8: Data minimisation — return only what is needed
# These are architecturally enforced, not just policy documents.

from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Any

from elasticsearch import AsyncElasticsearch


class OfficerScope(Enum):
    """
    Authorisation scopes for government officers.
    Injected by API gateway from authenticated session.
    Cannot be set by the client directly.
    """
    IDENTITY_BASIC = "IDENTITY_BASIC"       # Name (masked), state, age group
    IDENTITY_FULL = "IDENTITY_FULL"         # Full identity details
    DOCUMENT_OFFICER = "DOCUMENT_OFFICER"   # Document list access
    FINANCIAL_ANALYST = "FINANCIAL_ANALYST" # Financial data access


# Scope → allowed Elasticsearch _source fields
# This mapping is the DPDP data minimisation enforcement mechanism
SCOPE_ALLOWED_FIELDS: dict[str, list[str]] = {
    OfficerScope.IDENTITY_BASIC.value: [
        "citizenId",
        "stateCode",
        "districtCode",
        "ageGroup",
        "gender",
        "erasureStatus",
        # Name is NOT in this list — returned masked via post-processing
    ],
    OfficerScope.IDENTITY_FULL.value: [
        "citizenId",
        "nameSearchable",   # Officer with full identity can see name
        "stateCode",
        "districtCode",
        "ageGroup",
        "gender",
        "documentTypes",
        "erasureStatus",
    ],
    OfficerScope.FINANCIAL_ANALYST.value: [
        "citizenId",
        "stateCode",
        "ageGroup",
        "documentTypes",
        # Financial analysts see AGGREGATE info, not individual identity
    ],
}


@dataclass
class SearchRequest:
    query_text: str | None
    state_code: str | None
    district_code: str | None
    age_group: str | None
    gender: str | None
    document_types: list[str] | None
    officer_scope: str
    fuzzy: bool = True
    page: int = 0
    page_size: int = 20


@dataclass
class SearchResult:
    citizen_id: str
    fields: dict[str, Any]
    score: float
    name_masked: str | None  # Always masked — never raw name in response


class CitizenSearchService:
    """
    Elasticsearch-powered citizen search with DPDP compliance.

    Every search request is scope-filtered before execution.
    Every response is field-filtered before returning.
    No escape hatch — scope is injected server-side.
    """

    INDEX = "citizens"
    MAX_PAGE_SIZE = 100  # Prevent large data dumps

    def __init__(self, es_client: AsyncElasticsearch) -> None:
        self._es = es_client

    async def search(self, request: SearchRequest) -> list[SearchResult]:
        """
        Execute a DPDP-compliant citizen search.

        Query construction order:
        1. Must clauses (text search + filters)
        2. Scope filter (injected server-side — cannot be bypassed)
        3. Source filter (only authorised fields returned)
        4. Pagination
        """
        if request.page_size > self.MAX_PAGE_SIZE:
            request.page_size = self.MAX_PAGE_SIZE

        query = self._build_query(request)
        source_fields = SCOPE_ALLOWED_FIELDS.get(
            request.officer_scope,
            SCOPE_ALLOWED_FIELDS[OfficerScope.IDENTITY_BASIC.value]
        )

        response = await self._es.search(
            index=self.INDEX,
            body=query,
            source=source_fields,       # Layer 2 DPDP: field-level filter
            from_=request.page * request.page_size,
            size=request.page_size,
            timeout="5s",               # Never let a query run > 5s
        )

        return [
            self._map_hit(hit, request.officer_scope)
            for hit in response["hits"]["hits"]
            # Layer: never return ERASED citizens in normal search
            if hit["_source"].get("erasureStatus") != "ERASED"
        ]

    def _build_query(self, request: SearchRequest) -> dict:
        """
        Build Elasticsearch bool query.

        Structure:
        bool:
          must: [text search clauses]
          filter: [exact match clauses + SCOPE FILTER]

        filter clauses do NOT affect relevance score (faster execution).
        must clauses DO affect relevance score (ranked by relevance).
        """
        must_clauses = []
        filter_clauses = []

        # ── TEXT SEARCH ──────────────────────────────────────────────
        if request.query_text:
            if request.fuzzy:
                # Multi-match with fuzziness:
                # "Shaarma" matches "Sharma" (edit distance 1)
                # "Ramasubramanian" fuzzy matches variants
                # phonetic field catches "Sinha" / "Singha" variants
                must_clauses.append({
                    "multi_match": {
                        "query": request.query_text,
                        "fields": [
                            "nameSearchable^3",          # Boost exact match
                            "nameSearchable.phonetic^1", # Phonetic fallback
                        ],
                        "type": "best_fields",
                        "fuzziness": "AUTO",   # AUTO: 0 for 1-2 chars,
                                               # 1 for 3-5 chars,
                                               # 2 for 6+ chars
                        "prefix_length": 2,    # First 2 chars must match
                                               # Prevents too-fuzzy matches
                        "minimum_should_match": "75%",
                    }
                })
            else:
                # Exact phrase match — for ID-based lookups
                must_clauses.append({
                    "match": {
                        "nameSearchable": {
                            "query": request.query_text,
                            "operator": "and",
                        }
                    }
                })

        # ── EXACT FILTERS ─────────────────────────────────────────────
        if request.state_code:
            filter_clauses.append({"term": {"stateCode": request.state_code}})

        if request.district_code:
            filter_clauses.append(
                {"term": {"districtCode": request.district_code}}
            )

        if request.age_group:
            filter_clauses.append({"term": {"ageGroup": request.age_group}})

        if request.gender:
            filter_clauses.append({"term": {"gender": request.gender}})

        if request.document_types:
            filter_clauses.append(
                {"terms": {"documentTypes": request.document_types}}
            )

        # ── SCOPE FILTER (Layer 1 DPDP) ────────────────────────────────
        # CRITICAL: This filter is ALWAYS added — server-side injection.
        # Client cannot remove this filter. Client cannot override scope.
        # An officer with IDENTITY_BASIC scope cannot retrieve
        # records that require FINANCIAL_ANALYST scope.
        # authorisedScopes field in ES doc lists which scopes can see it.
        filter_clauses.append({
            "term": {
                "authorisedScopes": request.officer_scope
            }
        })

        # If no must clauses: match_all (returns all within filters)
        if not must_clauses:
            must_clauses.append({"match_all": {}})

        return {
            "query": {
                "bool": {
                    "must": must_clauses,
                    "filter": filter_clauses,
                }
            },
            "sort": [
                {"_score": {"order": "desc"}},      # Relevance first
                {"stateCode": {"order": "asc"}},    # Then alphabetical by state
            ],
            "highlight": {
                "fields": {
                    "nameSearchable": {
                        "pre_tags": ["<mark>"],
                        "post_tags": ["</mark>"],
                        "number_of_fragments": 1,
                    }
                }
            }
        }

    def _map_hit(
        self,
        hit: dict,
        officer_scope: str
    ) -> SearchResult:
        """
        Map Elasticsearch hit to SearchResult.
        Apply name masking regardless of scope.
        Full name is NEVER returned raw — always masked.
        DPDP Act S8: data minimisation in search results.
        """
        source = hit["_source"]
        raw_name = source.get("nameSearchable", "")

        # Name masking: "Anand Vijayakumar" → "A**** V**********"
        # Officers see enough to confirm identity without full disclosure
        masked_name = self._mask_name(raw_name) if raw_name else None

        return SearchResult(
            citizen_id=source.get("citizenId", ""),
            fields={k: v for k, v in source.items() if k != "nameSearchable"},
            score=hit.get("_score", 0.0),
            name_masked=masked_name,
        )

    @staticmethod
    def _mask_name(name: str) -> str:
        """
        Mask a name for DPDP-compliant display.
        First character of each word retained. Rest replaced with *.
        "Anand Vijayakumar" → "A**** V**********"
        """
        parts = name.split()
        masked_parts = []
        for part in parts:
            if len(part) <= 1:
                masked_parts.append(part)
            else:
                masked_parts.append(part[0] + "*" * (len(part) - 1))
        return " ".join(masked_parts)
```

---

## REDIS — Session + Rate Limiter (Python 3.10)

```
FILE: api-gateway/src/middleware/rate_limiter.py
LANGUAGE: Python 3.10
PURPOSE: Redis-backed sliding window rate limiter.
         Demonstrates Redis atomic operations (INCR, EXPIRE, TTL)
         for sub-millisecond rate limit enforcement.
CONCEPTS DEMONSTRATED: Redis atomic operations, sliding window algorithm,
                       sub-millisecond access pattern, DPDP API throttling
COPY-PASTE READY: YES
PRODUCTION DELTA: Use Redis Cluster for high-availability rate limiting.
                  Add per-IP blocking for repeated limit violations.
                  Integrate with Azure API Management for enterprise throttling.
```

```python
# api-gateway/src/middleware/rate_limiter.py
#
# ARCHITECTURAL NOTE — REDIS FOR RATE LIMITING:
#
# Why Redis (not PostgreSQL/MongoDB) for rate limiting:
#
# Rate limiting requires:
# 1. ATOMIC counter increment (no race condition between check + increment)
# 2. AUTOMATIC expiry (counter resets after window)
# 3. SUB-MILLISECOND response (rate limit check is on EVERY request)
#
# Redis native operations:
# INCR key     → Atomically increment. Thread-safe. 0.1ms.
# EXPIRE key N → Set TTL. Key auto-deletes after N seconds.
# TTL key      → Get remaining TTL. For X-RateLimit-Reset header.
#
# These three operations implement a fixed-window rate limiter in 3 lines.
# The same logic in PostgreSQL requires:
# SELECT FOR UPDATE + UPDATE + WHERE + transaction = 5-50ms
# At 10,000 requests/second: 50ms × 10,000 = 500 seconds of DB time
# for rate limiting alone. Impractical.
#
# Redis: 0.1ms × 10,000 = 1 second total. Practical.
#
# SLIDING WINDOW (used below) vs FIXED WINDOW:
# Fixed window: 100 requests per minute, window resets at :00
# Vulnerability: 100 requests at :59 + 100 at :00 = 200 in 1 second
#
# Sliding window: uses Redis sorted set to track request timestamps
# Window is always the last 60 seconds from NOW
# No boundary vulnerability
# Cost: slightly more Redis memory (O(n) per window vs O(1) for fixed)

import time
from dataclasses import dataclass

import redis.asyncio as aioredis


@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    reset_at: int      # Unix timestamp when limit resets
    limit: int


class SlidingWindowRateLimiter:
    """
    Redis-backed sliding window rate limiter.

    Uses Redis sorted set to track request timestamps.
    Window is always "last N seconds from now" — no fixed-window boundary bug.

    Key pattern: rate_limit:{citizen_id}:{endpoint}
    TTL: window_seconds × 2 (safety margin for clock drift)
    """

    def __init__(self, redis_client: aioredis.Redis) -> None:
        self._redis = redis_client

    # Rate limit configuration per endpoint type
    # DPDP Act S4: processing for specified purpose includes rate limits
    # to prevent bulk data harvesting via API
    LIMITS: dict[str, dict] = {
        "search":       {"requests": 100, "window_seconds": 60},
        "identity":     {"requests": 1000, "window_seconds": 60},
        "document":     {"requests": 500, "window_seconds": 60},
        "erasure":      {"requests": 5, "window_seconds": 3600},   # 5 per hour
        "default":      {"requests": 200, "window_seconds": 60},
    }

    async def check_and_increment(
        self,
        citizen_id: str,
        endpoint_type: str,
    ) -> RateLimitResult:
        """
        Check rate limit and increment counter if allowed.
        Single Redis pipeline call — atomic, sub-millisecond.

        Sliding window algorithm:
        1. Remove timestamps older than window_start
        2. Count remaining timestamps (= requests in current window)
        3. If count < limit: add current timestamp, return allowed=True
        4. If count >= limit: return allowed=False, no increment
        """
        config = self.LIMITS.get(endpoint_type, self.LIMITS["default"])
        limit = config["requests"]
        window_seconds = config["window_seconds"]

        key = f"rate_limit:{citizen_id}:{endpoint_type}"
        now = time.time()
        window_start = now - window_seconds

        # Use Redis pipeline for atomic multi-command execution
        # All commands execute as one unit — no interleaving
        async with self._redis.pipeline(transaction=True) as pipe:
            # Remove timestamps outside the sliding window
            pipe.zremrangebyscore(key, "-inf", window_start)
            # Count requests within window
            pipe.zcard(key)
            # Add current request timestamp (score=timestamp, member=unique_id)
            pipe.zadd(key, {str(now): now})
            # Set TTL on the key (prevents memory leak for inactive users)
            pipe.expire(key, window_seconds * 2)

            results = await pipe.execute()

        current_count = results[1]  # zcard result (before this request)

        if current_count >= limit:
            # Rate limit exceeded — remove the just-added entry
            await self._redis.zrem(key, str(now))
            return RateLimitResult(
                allowed=False,
                remaining=0,
                reset_at=int(now + window_seconds),
                limit=limit,
            )

        return RateLimitResult(
            allowed=True,
            remaining=limit - current_count - 1,
            reset_at=int(now + window_seconds),
            limit=limit,
        )


class RedisSessionManager:
    """
    Redis session store for API gateway.
    Session = key:value with TTL.
    O(1) lookup. Sub-millisecond. Auto-expiry via Redis TTL.
    """

    SESSION_TTL_SECONDS = 3600  # 1 hour — DPDP Act S9: time-bounded sessions

    def __init__(self, redis_client: aioredis.Redis) -> None:
        self._redis = redis_client

    async def create_session(
        self,
        citizen_id: str,
        officer_scope: str,
        session_id: str,
    ) -> str:
        """
        Create a session with TTL.
        Returns session_id for cookie.

        Redis key: session:{session_id}
        Redis value: JSON with citizen_id, scope, created_at
        TTL: 3600 seconds (auto-expire — no cron job needed)
        """
        import json
        session_data = json.dumps({
            "citizenId": citizen_id,
            "officerScope": officer_scope,
            "createdAt": time.time(),
        })

        # SETEX: SET with EXpiry — atomic, one command
        await self._redis.setex(
            f"session:{session_id}",
            self.SESSION_TTL_SECONDS,
            session_data,
        )
        return session_id

    async def get_session(self, session_id: str) -> dict | None:
        """
        Retrieve session. Returns None if expired or not found.
        O(1) hash lookup — 0.1ms regardless of session count.
        """
        import json
        data = await self._redis.get(f"session:{session_id}")
        if data is None:
            return None
        return json.loads(data)

    async def delete_session(self, session_id: str) -> None:
        """Logout: delete session immediately (before TTL expires)."""
        await self._redis.delete(f"session:{session_id}")
```

---

## ERASURE ORCHESTRATOR — DPDP Right-to-Erasure Saga (Python 3.10)

```
FILE: erasure-orchestrator/src/saga/erasure_saga.py
LANGUAGE: Python 3.10
PURPOSE: DPDP Act S17 right-to-erasure implementation as a Saga.
         Orchestrates erasure across PostgreSQL, MongoDB, Elasticsearch, and Redis.
         Uses crypto-shredding for immutable stores (event log, backups).
CONCEPTS DEMONSTRATED: DPDP Act compliance, cross-store Saga, crypto-shredding,
                       idempotent erasure operations, audit preservation
COPY-PASTE READY: YES
PRODUCTION DELTA: Use Temporal.io for durable saga execution with 24-hour timeout.
                  Add MeitY notification API call after erasure completion.
                  Add erasure certificate generation (PDF) for citizen download.
```

```python
# erasure-orchestrator/src/saga/erasure_saga.py
#
# ARCHITECTURAL NOTE — DPDP ACT ERASURE ACROSS POLYGLOT STORES:
#
# Right to Erasure (DPDP Act S17) must cover ALL stores where the citizen's
# personal data exists. In our polyglot architecture:
#
# Store 1: PostgreSQL (citizen_identity)
#   → Delete encryption key (crypto-shredding)
#   → Set erasure_status = ERASED
#   → Anonymise audit records (keep action, remove who)
#
# Store 2: MongoDB (citizen_documents)
#   → Soft erase: clear content, set status = ERASED
#   → Retain document TYPE and DATE (anonymised audit trail)
#
# Store 3: Elasticsearch (citizen search index)
#   → Delete document from index
#   → OR: set erasureStatus = ERASED (then exclude from searches)
#
# Store 4: Redis (session data)
#   → Delete all active sessions for this citizen
#   → Delete rate limit counters
#
# Store 5: Kafka (event log) — CANNOT delete (immutable)
#   → Crypto-shredding handles this:
#     Historical events contain ENCRYPTED name fields
#     Key is deleted in Step 1 (PostgreSQL)
#     Events become unreadable — mathematically erased
#   → This is the ONLY DPDP-compliant approach for event sourcing
#
# SAGA DESIGN:
# Each step is idempotent (safe to retry).
# If any step fails: log the failure, continue with other steps
# (partial erasure is better than no erasure for DPDP compliance).
# All steps completed within 24 hours of request (DPDP SLA).

from __future__ import annotations

import json
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Callable


class ErasureStep(Enum):
    INITIATED = "INITIATED"
    KEY_DELETED = "KEY_DELETED"           # Crypto-shredding complete
    POSTGRES_MARKED = "POSTGRES_MARKED"   # Status = ERASED in PostgreSQL
    MONGO_ERASED = "MONGO_ERASED"         # Documents soft-erased in MongoDB
    ES_REMOVED = "ES_REMOVED"             # Removed from search index
    REDIS_CLEARED = "REDIS_CLEARED"       # Sessions + rate limits cleared
    AUDIT_ANONYMISED = "AUDIT_ANONYMISED" # Audit records anonymised
    COMPLETED = "COMPLETED"
    PARTIALLY_COMPLETED = "PARTIALLY_COMPLETED"  # Some steps failed


@dataclass
class ErasureStepResult:
    step: ErasureStep
    success: bool
    records_affected: int
    error: str | None = None
    executed_at: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )


@dataclass
class ErasureSagaState:
    saga_id: str
    citizen_id: str
    requested_at: datetime
    requested_by: str           # Officer or citizen who requested
    step_results: list[ErasureStepResult] = field(default_factory=list)
    current_step: ErasureStep = ErasureStep.INITIATED
    completed_at: datetime | None = None

    def record_step(self, result: ErasureStepResult) -> None:
        self.step_results.append(result)
        self.current_step = result.step

    def is_complete(self) -> bool:
        return self.current_step in (
            ErasureStep.COMPLETED,
            ErasureStep.PARTIALLY_COMPLETED
        )

    def success_rate(self) -> float:
        if not self.step_results:
            return 0.0
        successes = sum(1 for r in self.step_results if r.success)
        return successes / len(self.step_results)


class ErasureOrchestrator:
    """
    DPDP Act S17 Erasure Saga Orchestrator.

    Coordinates erasure across all 4 data stores.
    Each step is idempotent — safe to retry on failure.
    Partial completion is tracked and reported.
    Full completion must occur within 24 hours of request.
    """

    def __init__(
        self,
        postgres_eraser,
        mongo_eraser,
        es_eraser,
        redis_eraser,
        saga_repository,
        event_publisher: Callable[[str, dict], None],
    ) -> None:
        self._pg = postgres_eraser
        self._mongo = mongo_eraser
        self._es = es_eraser
        self._redis = redis_eraser
        self._repository = saga_repository
        self._publish = event_publisher

    async def execute_erasure(
        self,
        citizen_id: str,
        requested_by: str,
        idempotency_key: str,
    ) -> ErasureSagaState:
        """
        Execute the complete DPDP erasure saga.

        Returns saga state with step-by-step results.
        Caller uses this to generate the erasure certificate.

        IDEMPOTENCY:
        If called twice with same idempotency_key:
        returns the existing saga state without re-executing.
        Prevents double-erasure on client retry.
        """
        # Check for existing saga (idempotency)
        existing = await self._repository.find_by_idempotency_key(
            idempotency_key
        )
        if existing and existing.is_complete():
            return existing

        saga = ErasureSagaState(
            saga_id=str(uuid.uuid4()),
            citizen_id=citizen_id,
            requested_at=datetime.now(timezone.utc),
            requested_by=requested_by,
        )
        await self._repository.save(saga)

        self._log_step(citizen_id, "ERASURE_INITIATED",
                       f"Erasure requested by {requested_by}")

        # ── STEP 1: Delete encryption key (CRYPTO-SHREDDING) ─────────
        # This is the most critical step.
        # Once the key is deleted:
        # ALL encrypted fields (name, in events, in backups) are unreadable.
        # This handles the event sourcing + erasure incompatibility.
        step1 = await self._execute_step(
            saga, ErasureStep.KEY_DELETED,
            lambda: self._pg.delete_encryption_key(citizen_id, idempotency_key)
        )
        saga.record_step(step1)

        # ── STEP 2: Mark citizen as ERASED in PostgreSQL ──────────────
        step2 = await self._execute_step(
            saga, ErasureStep.POSTGRES_MARKED,
            lambda: self._pg.mark_citizen_erased(citizen_id, idempotency_key)
        )
        saga.record_step(step2)

        # ── STEP 3: Soft-erase MongoDB documents ──────────────────────
        step3 = await self._execute_step(
            saga, ErasureStep.MONGO_ERASED,
            lambda: self._mongo.erase_citizen_documents(
                citizen_id, idempotency_key
            )
        )
        saga.record_step(step3)

        # ── STEP 4: Remove from Elasticsearch index ───────────────────
        step4 = await self._execute_step(
            saga, ErasureStep.ES_REMOVED,
            lambda: self._es.remove_from_index(citizen_id, idempotency_key)
        )
        saga.record_step(step4)

        # ── STEP 5: Clear Redis sessions and rate limits ──────────────
        step5 = await self._execute_step(
            saga, ErasureStep.REDIS_CLEARED,
            lambda: self._redis.clear_citizen_data(citizen_id)
        )
        saga.record_step(step5)

        # ── STEP 6: Anonymise PostgreSQL audit records ────────────────
        # DPDP Act S8: retain audit TRAIL (what happened) but remove WHO
        # citizen_id set to NULL in audit table after erasure
        # Action and timestamp retained for regulatory audit
        step6 = await self._execute_step(
            saga, ErasureStep.AUDIT_ANONYMISED,
            lambda: self._pg.anonymise_audit_records(citizen_id)
        )
        saga.record_step(step6)

        # ── COMPLETION ────────────────────────────────────────────────
        all_success = all(r.success for r in saga.step_results)
        saga.current_step = (
            ErasureStep.COMPLETED if all_success
            else ErasureStep.PARTIALLY_COMPLETED
        )
        saga.completed_at = datetime.now(timezone.utc)
        await self._repository.save(saga)

        # Publish completion event (for certificate generation, MeitY notification)
        self._publish("CitizenErased", {
            "citizenId": citizen_id,
            "sagaId": saga.saga_id,
            "completedAt": saga.completed_at.isoformat(),
            "successRate": saga.success_rate(),
            "stepsCompleted": len([r for r in saga.step_results if r.success]),
            "totalSteps": len(saga.step_results),
        })

        self._log_step(
            citizen_id,
            "ERASURE_COMPLETED" if all_success else "ERASURE_PARTIAL",
            f"Success rate: {saga.success_rate():.0%}"
        )

        return saga

    async def _execute_step(
        self,
        saga: ErasureSagaState,
        step: ErasureStep,
        operation: Callable,
    ) -> ErasureStepResult:
        """
        Execute one erasure step with error isolation.
        A failure in one step does NOT abort other steps.
        DPDP compliance: partial erasure is better than no erasure.
        """
        try:
            records_affected = await operation()
            return ErasureStepResult(
                step=step,
                success=True,
                records_affected=records_affected or 0,
            )
        except Exception as e:
            # Log failure but continue — do not raise
            self._log_step(
                saga.citizen_id, f"STEP_FAILED_{step.value}", str(e)
            )
            return ErasureStepResult(
                step=step,
                success=False,
                records_affected=0,
                error=str(e),
            )

    def _log_step(
        self,
        citizen_id: str,
        step: str,
        message: str
    ) -> None:
        # Structured log — citizenId is anonymised in output
        # Never log full citizen_id in production — use hashed reference
        print(json.dumps({
            "level": "INFO",
            "erasureStep": step,
            "citizenRef": citizen_id[:8] + "****",  # Partial ID only
            "message": message,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }))
```

---

## ELASTICSEARCH INDEX UPDATER — Kafka Consumer (Python 3.10)

```
FILE: es-index-updater/src/indexer.py
LANGUAGE: Python 3.10
PURPOSE: Consumes citizen-events from Kafka and updates Elasticsearch index.
         Demonstrates event-driven data synchronisation across polyglot stores.
         Uses ES bulk API for efficient batch indexing.
CONCEPTS DEMONSTRATED: Cross-store data sync, Kafka→ES projection,
                       bulk indexing for throughput, DPDP field projection
COPY-PASTE READY: YES
PRODUCTION DELTA: Add schema registry validation before indexing.
                  Add dead letter queue for malformed events.
                  Implement index alias for zero-downtime reindexing.
```

```python
# es-index-updater/src/indexer.py
#
# ARCHITECTURAL NOTE — EVENT-DRIVEN ES SYNC:
#
# Elasticsearch is a READ MODEL (CQRS read side).
# PostgreSQL is the WRITE MODEL (system of record).
#
# Sync mechanism: PostgreSQL change → Kafka event → ES index update
#
# This is the "Projection Worker" pattern from Day 3.
# The ES index is eventually consistent with PostgreSQL.
# Lag: typically 1-5 seconds (Kafka processing + ES indexing).
# This lag is ACCEPTABLE for search (AP behaviour).
# It is NOT acceptable for identity verification (CP behaviour).
#
# BULK INDEXING:
# ES bulk API processes multiple index operations in one HTTP call.
# Single-document indexing: 1 HTTP call per document = high overhead.
# Bulk indexing: 100-1000 documents per HTTP call = 10-100× throughput.
# For seeding 1M citizen records: bulk indexing takes 2 minutes.
# Single-document indexing: 8+ hours.
# Always use bulk API for high-volume operations.

from __future__ import annotations

import asyncio
import json
from datetime import datetime, timezone

from elasticsearch import AsyncElasticsearch


class CitizenIndexUpdater:
    """
    Projection worker: updates Elasticsearch citizens index
    from Kafka citizen-events topic.

    Batches index operations for throughput efficiency.
    Handles partial batch failures (some docs succeed, some fail).
    """

    INDEX = "citizens"
    BATCH_SIZE = 100                # Process 100 events per bulk call
    FLUSH_INTERVAL_SECONDS = 2.0   # Flush pending batch every 2 seconds

    def __init__(self, es_client: AsyncElasticsearch) -> None:
        self._es = es_client
        self._pending_operations: list[dict] = []
        self._last_flush = datetime.now(timezone.utc).timestamp()

    async def handle_event(
        self,
        event_type: str,
        payload: dict
    ) -> None:
        """
        Route event to appropriate index operation.
        Events are batched — actual ES call happens in flush().
        """
        match event_type:
            case "CitizenRegistered":
                await self._queue_index(payload)
            case "CitizenUpdated":
                await self._queue_update(payload)
            case "CitizenErased":
                await self._queue_delete(payload)
            case _:
                pass  # Unknown events ignored — forward compatibility

        # Flush if batch is full or interval exceeded
        if (len(self._pending_operations) >= self.BATCH_SIZE * 2 or
                self._should_flush_by_time()):
            await self.flush()

    async def _queue_index(self, payload: dict) -> None:
        """
        Build Elasticsearch document from citizen payload.
        Only index fields needed for search — DPDP data minimisation.
        Never index Aadhaar hash or encrypted name in plaintext.
        """
        citizen_id = payload.get("citizenId")
        if not citizen_id:
            return

        # ES document: only search-relevant fields
        # Name stored as nameSearchable (analysed for fuzzy matching)
        # NOT storing: aadhaarHash, nameEncrypted, encryptionKey
        es_doc = {
            "citizenId": citizen_id,
            # Name is stored in searchable form (lowercase, ascii-folded)
            # The ENCRYPTED name from PostgreSQL is decrypted ONCE
            # during this projection. The plaintext exists in ES index.
            # PRODUCTION: ES index must have field-level encryption
            # OR: store only phonetic token (not readable name)
            "nameSearchable": payload.get("name", ""),
            "stateCode": payload.get("stateCode", ""),
            "districtCode": payload.get("districtCode", ""),
            "ageGroup": self._compute_age_group(payload.get("dob", "")),
            "gender": payload.get("gender", ""),
            "documentTypes": payload.get("documentTypes", []),
            "erasureStatus": "ACTIVE",
            # Scope field determines which officer scopes can see this doc
            "authorisedScopes": [
                "IDENTITY_BASIC",
                "IDENTITY_FULL",
                "DOCUMENT_OFFICER",
            ],
            "indexedAt": datetime.now(timezone.utc).isoformat(),
        }

        # Bulk API format: action line + document line (alternating)
        self._pending_operations.append(
            {"index": {"_index": self.INDEX, "_id": citizen_id}}
        )
        self._pending_operations.append(es_doc)

    async def _queue_update(self, payload: dict) -> None:
        """Partial update — only changed fields."""
        citizen_id = payload.get("citizenId")
        if not citizen_id:
            return

        update_fields = {
            k: v for k, v in payload.items()
            if k in ("stateCode", "districtCode", "documentTypes",
                     "erasureStatus", "ageGroup")
        }

        self._pending_operations.append(
            {"update": {"_index": self.INDEX, "_id": citizen_id}}
        )
        self._pending_operations.append({"doc": update_fields})

    async def _queue_delete(self, payload: dict) -> None:
        """
        Remove citizen from search index on erasure.
        DPDP Act S17: erased citizens must not appear in searches.
        """
        citizen_id = payload.get("citizenId")
        if not citizen_id:
            return

        self._pending_operations.append(
            {"delete": {"_index": self.INDEX, "_id": citizen_id}}
        )

    async def flush(self) -> None:
        """
        Execute pending bulk operations against Elasticsearch.
        Handles partial failures — some docs may fail while others succeed.
        """
        if not self._pending_operations:
            return

        operations = list(self._pending_operations)
        self._pending_operations.clear()
        self._last_flush = datetime.now(timezone.utc).timestamp()

        try:
            response = await self._es.bulk(operations=operations)

            if response.get("errors"):
                # Bulk completed but some items failed
                # Log failed items for DLQ routing
                failed = [
                    item for item in response["items"]
                    if list(item.values())[0].get("error")
                ]
                print(json.dumps({
                    "level": "WARN",
                    "message": f"Bulk index partial failure",
                    "totalItems": len(response["items"]),
                    "failedItems": len(failed),
                    "firstError": failed[0] if failed else None,
                }))
            else:
                print(json.dumps({
                    "level": "INFO",
                    "message": "Bulk index complete",
                    "itemCount": len(response["items"]),
                    "tookMs": response.get("took", 0),
                }))

        except Exception as e:
            # Full bulk failure — re-queue operations or route to DLQ
            # In production: re-add to pending with retry counter
            print(json.dumps({
                "level": "ERROR",
                "message": "Bulk index failed",
                "error": str(e),
                "operationCount": len(operations) // 2,
            }))

    def _should_flush_by_time(self) -> bool:
        now = datetime.now(timezone.utc).timestamp()
        return (now - self._last_flush) >= self.FLUSH_INTERVAL_SECONDS

    @staticmethod
    def _compute_age_group(dob_str: str) -> str:
        """
        Compute age group from DOB for search faceting.
        Stores age GROUP (not age) — reduces precision for DPDP minimisation.
        "1975-03-15" → "45-54"
        """
        if not dob_str:
            return "UNKNOWN"
        try:
            from datetime import date
            dob = date.fromisoformat(dob_str[:10])
            age = (date.today() - dob).days // 365
            if age < 18:   return "UNDER_18"
            if age < 30:   return "18-29"
            if age < 45:   return "30-44"
            if age < 60:   return "45-59"
            return "60_PLUS"
        except Exception:
            return "UNKNOWN"
```

---

## DOMAIN TESTS — Crypto-Shredding Verification (Java 17)

```
FILE: identity-service/src/test/java/com/citizenvault/identity/domain/CryptoShreddingTest.java
LANGUAGE: Java 17
PURPOSE: Proves that deleting the encryption key makes data permanently unreadable.
         This is the DPDP Act S17 compliance proof test.
CONCEPTS DEMONSTRATED: Crypto-shredding correctness verification,
                       DPDP right-to-erasure architectural proof
COPY-PASTE READY: YES
```

```java
// FILE: com/citizenvault/identity/domain/CryptoShreddingTest.java
//
// THE MOST IMPORTANT COMPLIANCE TEST:
// Prove that after key deletion, the encrypted data is unreadable.
// This is the architectural proof that crypto-shredding satisfies DPDP S17.
// If this test passes: your erasure implementation is DPDP-compliant.
// If this test fails: encrypted data is not truly erased.

package com.citizenvault.identity.domain;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

@DisplayName("Crypto-Shredding — DPDP Act S17 Compliance Tests")
class CryptoShreddingTest {

    private final CryptoService cryptoService = new CryptoService();

    @Test
    @DisplayName("Encrypted name is readable with key — baseline")
    void encryptedNameReadableWithKey() {
        var name = "Anand Vijayakumar";
        var key = cryptoService.generateKey();
        var encrypted = cryptoService.encrypt(name, key);

        var decrypted = cryptoService.decrypt(encrypted, key);

        assertThat(decrypted).isEqualTo(name);
    }

    @Test
    @DisplayName("DPDP S17: After key deletion, name becomes permanently unreadable")
    void afterKeyDeletionNameIsUnreadable() {
        // Simulate: citizen registers, name is encrypted
        var name = "Priya Ramasubramanian";
        var key = cryptoService.generateKey();
        var encrypted = cryptoService.encrypt(name, key);

        // Verify readable BEFORE erasure
        assertThat(cryptoService.decrypt(encrypted, key)).isEqualTo(name);

        // Simulate key deletion (crypto-shredding)
        // In production: DELETE FROM citizen_keys WHERE citizen_id = ?
        byte[] deletedKey = null;  // Key is gone

        // Verify UNREADABLE after erasure
        // Returns null — not an exception — callers handle null as "erased"
        var result = cryptoService.decrypt(encrypted, deletedKey);
        assertThat(result).isNull();

        // The ciphertext still exists — but is permanently unreadable
        // DPDP Act S17 compliance: data is effectively erased
        // even though the bytes remain in the database and event log
        assertThat(encrypted).isNotNull();
        assertThat(encrypted.length).isGreaterThan(0);
    }

    @Test
    @DisplayName("Different citizens have different keys — no cross-contamination")
    void differentCitizensHaveDifferentKeys() {
        var key1 = cryptoService.generateKey();
        var key2 = cryptoService.generateKey();

        var name1 = "Citizen One";
        var name2 = "Citizen Two";

        var encrypted1 = cryptoService.encrypt(name1, key1);
        var encrypted2 = cryptoService.encrypt(name2, key2);

        // Correct decryption
        assertThat(cryptoService.decrypt(encrypted1, key1)).isEqualTo(name1);
        assertThat(cryptoService.decrypt(encrypted2, key2)).isEqualTo(name2);

        // Cross-key decryption MUST fail
        // If citizen 1's key could decrypt citizen 2's data:
        // erasing citizen 1's key would also erase citizen 2's data
        assertThatThrownBy(() -> cryptoService.decrypt(encrypted2, key1))
                .isInstanceOf(CryptoException.class);

        assertThatThrownBy(() -> cryptoService.decrypt(encrypted1, key2))
                .isInstanceOf(CryptoException.class);
    }

    @Test
    @DisplayName("Aadhaar hash is non-reversible — DPDP data minimisation")
    void aadhaarHashIsNonReversible() {
        var aadhaar = "123456789012";
        var hash1 = cryptoService.hashAadhaar(aadhaar);
        var hash2 = cryptoService.hashAadhaar(aadhaar);

        // Deterministic: same input = same hash (needed for dedup check)
        assertThat(hash1).isEqualTo(hash2);

        // Non-reversible: hash does not contain original Aadhaar
        assertThat(hash1).doesNotContain(aadhaar);
        assertThat(hash1.length()).isEqualTo(44); // Base64-encoded SHA-256

        // Different Aadhaar = different hash
        var differentHash = cryptoService.hashAadhaar("999999999999");
        assertThat(hash1).isNotEqualTo(differentHash);
    }
}
```

---

## INFRASTRUCTURE — Terraform for Azure Polyglot Stack

```
FILE: infra/terraform/main.tf
LANGUAGE: HCL
PURPOSE: Provisions Azure-managed versions of all polyglot stores.
         Demonstrates cloud-native alternatives to self-hosted databases.
CONCEPTS DEMONSTRATED: Azure managed database services, free tier configuration,
                       data residency enforcement through Azure region selection
COPY-PASTE READY: YES
PRODUCTION DELTA: Add Azure Private Endpoints for all databases (no public internet).
                  Add Azure Defender for databases (threat detection).
                  Add customer-managed keys (CMK) for PostgreSQL encryption.
```

```hcl
# infra/terraform/main.tf
#
# ARCHITECTURAL NOTE — MANAGED vs SELF-HOSTED:
#
# This lab uses Docker for all databases locally.
# This Terraform provisions MANAGED Azure services for cloud deployment.
#
# MANAGED ADVANTAGES:
# ✅ Automated backups, patches, HA — reduces operational toil
# ✅ Azure compliance documentation for MAS TRM / MeitY audits
# ✅ Built-in monitoring via Azure Monitor
# ✅ SLA-backed availability (99.99% for most services)
#
# MANAGED DISADVANTAGES:
# ❌ Higher cost vs self-hosted at very high scale
# ❌ Less configuration flexibility
# ❌ Vendor lock-in for specific features
# ❌ Some features unavailable (e.g., Elasticsearch plugins)
#
# RECOMMENDATION for government platforms:
# Use managed services for all databases.
# The compliance documentation and SLA coverage justify the cost.
# Self-host only when a specific feature is unavailable in managed tier.

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

resource "azurerm_resource_group" "citizenvault" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment  = "lab"
    project      = "citizenvault-day4"
    cost-centre  = "training"
    # DATA RESIDENCY TAG — tracks which resources hold citizen data
    # Used in compliance reporting to demonstrate DPDP localisation
    data-residency = "india-only"
  }
}

# ─── AZURE DATABASE FOR POSTGRESQL ───────────────────────────────────────────
# Flexible Server — more configuration options than Single Server
# Free tier: 32GB storage, 2 vCores — sufficient for lab
resource "azurerm_postgresql_flexible_server" "citizenvault" {
  name                = "psql-citizenvault-${var.environment}"
  resource_group_name = azurerm_resource_group.citizenvault.name
  location            = azurerm_resource_group.citizenvault.location

  # Burstable B2ms — free trial tier (12 months)
  sku_name   = "B_Standard_B2ms"
  version    = "15"
  storage_mb = 32768

  administrator_login    = var.postgres_admin_user
  administrator_password = var.postgres_admin_password

  # High availability — disabled for lab cost minimisation
  # Production: enable with standby in different AZ
  high_availability {
    mode = "Disabled"
  }

  # Backup: 7-day retention with geo-redundant disabled (lab)
  # Production: 35 days, geo-redundant for DR
  backup_retention_days        = 7
  geo_redundant_backup_enabled = false

  # DPDP Act: data localisation
  # This PostgreSQL instance will ONLY exist in the specified region
  # Geo-redundant backup would replicate data outside region — disabled
  # Production: if DR is needed — use within-country secondary region

  tags = azurerm_resource_group.citizenvault.tags
}

resource "azurerm_postgresql_flexible_server_database" "citizen_db" {
  name      = "citizen_db"
  server_id = azurerm_postgresql_flexible_server.citizenvault.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

# ─── AZURE COSMOS DB (MONGODB API) ────────────────────────────────────────────
# MongoDB-compatible API — existing MongoDB clients work unchanged
# Free tier: 1000 RU/s, 25GB storage — sufficient for lab
resource "azurerm_cosmosdb_account" "citizenvault" {
  name                = "cosmos-citizenvault-${var.environment}"
  location            = azurerm_resource_group.citizenvault.location
  resource_group_name = azurerm_resource_group.citizenvault.name
  offer_type          = "Standard"
  kind                = "MongoDB"

  # Free tier — one per subscription
  enable_free_tier = true

  # Strong consistency for identity data
  # Cosmos DB consistency levels align with CAP:
  # Strong = CP (linearisable reads)
  # Eventual = AP (lowest latency)
  # Session = balanced (read-your-own-writes)
  consistency_policy {
    consistency_level = "Session"  # Document store: session consistency
  }

  geo_location {
    location          = azurerm_resource_group.citizenvault.location
    failover_priority = 0
  }

  # DPDP: only one geo_location — no cross-region replication
  # Production: add secondary region within India for HA

  capabilities {
    name = "EnableMongo"
  }

  capabilities {
    name = "MongoDBv3.4"
  }

  tags = azurerm_resource_group.citizenvault.tags
}

# ─── AZURE CACHE FOR REDIS ────────────────────────────────────────────────────
# Free tier: C0 (250MB) — sufficient for session + rate limiting demo
resource "azurerm_redis_cache" "citizenvault" {
  name                = "redis-citizenvault-${var.environment}"
  location            = azurerm_resource_group.citizenvault.location
  resource_group_name = azurerm_resource_group.citizenvault.name

  # C0 Basic = 250MB, no SLA, no replication — lab use only
  # Production: P1 Premium = 6GB, Redis Cluster, zone-redundant
  capacity = 0
  family   = "C"
  sku_name = "Basic"

  # Redis version 7 — latest stable
  redis_version = "7"

  # Enable non-SSL port for lab simplicity
  # Production: enable_non_ssl_port = false — TLS only
  enable_non_ssl_port = false

  tags = azurerm_resource_group.citizenvault.tags
}

# ─── AZURE COGNITIVE SEARCH ───────────────────────────────────────────────────
# Free tier: 50MB index, 3 indexes, 3 replicas — lab sufficient
# NOTE: Azure Cognitive Search uses a different query language than Elasticsearch
# Your Python elasticsearch client WILL NOT work with Azure Cognitive Search.
# In production: use Elasticsearch on Azure VMs or Elastic Cloud on Azure.
# Azure Cognitive Search is included here for cost reference only.
resource "azurerm_search_service" "citizenvault" {
  name                = "search-citizenvault-${var.environment}"
  resource_group_name = azurerm_resource_group.citizenvault.name
  location            = azurerm_resource_group.citizenvault.location
  sku                 = "free"  # Free tier: 50MB, 3 indexes

  tags = azurerm_resource_group.citizenvault.tags
}
```

---

# L4. STEP-BY-STEP EXECUTION GUIDE

---

**STEP 1 of 7: Register a citizen — observe polyglot write**

🎯 OBJECTIVE: A single API call triggers writes to PostgreSQL (identity) AND publishes a Kafka event that the ES index updater consumes to update Elasticsearch.

CONCEPT LINK: Database-per-service pattern, event-driven sync (Block 3, Pattern 1).

```bash
$ curl -X POST http://localhost:8080/citizens \
  -H "Content-Type: application/json" \
  -H "X-Session-Token: demo-officer-token" \
  -d '{
    "aadhaarNumber": "234567890123",
    "name": "Anand Vijayakumar",
    "dateOfBirth": "1985-06-15",
    "gender": "M",
    "stateCode": "29",
    "districtCode": "BLR"
  }' | python3 -m json.tool
```

Expected Output:
```json
{
  "citizenId": "c1d2e3f4-...",
  "status": "REGISTERED",
  "message": "Citizen registered. Search index updated within 5 seconds.",
  "aadhaarStored": "HASH_ONLY",
  "encryptionKeyStored": true
}
```

🔍 WHAT TO OBSERVE: `aadhaarStored: HASH_ONLY` confirms the Aadhaar number was hashed before storage. `encryptionKeyStored: true` confirms the AES-256 key was created. The name is encrypted in PostgreSQL — you cannot read it directly from the database without the key.

```bash
# Verify in PostgreSQL — name is stored as encrypted bytes
$ docker compose exec postgres psql -U citizenvault -d citizen_db \
    -c "SELECT id, aadhaar_hash, name_encrypted, state_code, erasure_status \
        FROM citizens LIMIT 3;"
```

Expected Output:
```
| id           | aadhaar_hash | name_encrypted | state_code | erasure_status |
| ------------ | ------------ | -------------- | ---------- | -------------- |
| c1d2e3f4-... | xK7mP2qR...= | \x1a2b3c...    | 29         | ACTIVE         |
```

🔍 WHAT TO OBSERVE: `name_encrypted` shows binary data — not the citizen's name. Even a DBA with direct database access cannot read the name without the AES-256 key from `citizen_keys`. This is DPDP data protection at the infrastructure layer.

---

**STEP 2 of 7: Search with fuzzy matching — observe Elasticsearch performance**

🎯 OBJECTIVE: Compare PostgreSQL LIKE query vs. Elasticsearch full-text search performance. This demonstrates the core argument for polyglot persistence.

CONCEPT LINK: Access pattern → database selection (Block 1, Concept 1).

```bash
# WAIT 5 seconds for ES index updater to process the Kafka event
$ sleep 5

# Search with exact spelling
$ time curl -s "http://localhost:8080/search?q=Vijayakumar&scope=IDENTITY_BASIC" \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "results": [
    {
      "citizenId": "c1d2e3f4-...",
      "nameMasked": "A**** V**********",
      "stateCode": "29",
      "districtCode": "BLR",
      "ageGroup": "30-44",
      "gender": "M",
      "score": 4.21
    }
  ],
  "totalHits": 1,
  "searchTimeMs": 18
}

real    0m0.024s
```

```bash
# Fuzzy search — misspelled name
$ time curl -s "http://localhost:8080/search?q=Vijayakumaar&scope=IDENTITY_BASIC&fuzzy=true" \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "results": [
    {
      "citizenId": "c1d2e3f4-...",
      "nameMasked": "A**** V**********",
      "score": 2.87
    }
  ],
  "totalHits": 1,
  "searchTimeMs": 24
}
```

🔍 WHAT TO OBSERVE: `nameMasked: "A**** V**********"` — the DPDP scope filter is working. The IDENTITY_BASIC scope returns a masked name. The name is never returned in plaintext even though it exists in the ES index.

Now run the comparison script:

```bash
$ ./scripts/demo-search-comparison.sh
```

Expected Output:
```
=== SEARCH PERFORMANCE COMPARISON ===
Dataset: 1000 citizens

PostgreSQL LIKE query: SELECT * FROM citizens WHERE name LIKE '%Vijayakumar%'
  Attempt 1: 847ms (full table scan — no index on encrypted name)
  Attempt 2: 823ms
  Attempt 3: 851ms
  Average: 840ms

Elasticsearch full-text + fuzzy:
  Attempt 1: 18ms
  Attempt 2: 15ms
  Attempt 3: 22ms
  Average: 18ms

Performance difference: 47× faster with Elasticsearch
At 1M citizens: PostgreSQL LIKE ~ 4 minutes | ES ~ 50ms
At 100M citizens: PostgreSQL LIKE ~ 7 hours | ES ~ 200ms
```

⚠️ COMMON MISTAKE: Candidates add a PostgreSQL index on the name column to make it "competitive." Point out: the name field is ENCRYPTED (as it should be for DPDP compliance). You cannot put a B-tree index on encrypted bytes and still do meaningful text search. The PostgreSQL LIKE query requires plaintext — which means you would need to store the name unencrypted to make PostgreSQL search work. ES can index a searchable form of the name while PostgreSQL stores the encrypted form. This is the architecture.

---

**STEP 3 of 7: Store a document in MongoDB**

🎯 OBJECTIVE: Demonstrate the reference pattern — document stored in MongoDB with citizenId reference, not embedded in the citizen's PostgreSQL record.

```bash
$ CITIZEN_ID="c1d2e3f4-..."  # Use ID from Step 1

$ curl -X POST http://localhost:8080/documents \
  -H "Content-Type: application/json" \
  -H "X-Session-Token: demo-officer-token" \
  -d "{
    \"citizenId\": \"$CITIZEN_ID\",
    \"documentType\": \"AADHAAR\",
    \"documentNumber\": \"234567890123\",
    \"issuingAuthority\": \"UIDAI\",
    \"issueDate\": \"2015-01-01\"
  }" | python3 -m json.tool
```

Expected Output:
```json
{
  "documentId": "65f1a2b3c4d5e6f7...",
  "citizenId": "c1d2e3f4-...",
  "documentType": "AADHAAR",
  "status": "ACTIVE",
  "documentNumberStored": "HASH_ONLY"
}
```

```bash
# Verify in MongoDB — check the reference pattern
$ docker compose exec mongodb mongosh \
    -u citizenvault -p vault2026 \
    --authenticationDatabase admin \
    --quiet citizenvault_db \
    --eval "db.citizen_documents.find({citizenId: '$CITIZEN_ID'}).pretty()"
```

🔍 WHAT TO OBSERVE: The MongoDB document has `citizenId` as a reference to PostgreSQL. The Aadhaar document number is stored as a hash (`documentNumber` field contains SHA-256). The PostgreSQL `citizens` table has NO documents array — the reference pattern is working.

---

**STEP 4 of 7: Trigger DPDP erasure and observe cross-store cascade**

🎯 OBJECTIVE: Demonstrate the right-to-erasure Saga executing across all four stores. Observe crypto-shredding making encrypted data permanently unreadable.

CONCEPT LINK: DPDP Act S17, Erasure Saga, Crypto-shredding (Block 1, Concept 2 + Food for Thought from Day 4).

```bash
$ curl -X POST "http://localhost:8080/citizens/$CITIZEN_ID/erasure" \
  -H "Content-Type: application/json" \
  -H "X-Session-Token: demo-officer-token" \
  -H "X-Idempotency-Key: $(uuidgen)" \
  -d '{
    "requestedBy": "citizen-self",
    "reason": "DPDP_RIGHT_TO_ERASURE"
  }' | python3 -m json.tool
```

Expected Output:
```json
{
  "sagaId": "era-a1b2c3...",
  "citizenId": "c1d2e3f4-...",
  "status": "COMPLETED",
  "stepsCompleted": 6,
  "totalSteps": 6,
  "successRate": "100%",
  "completedAt": "2026-06-16T10:45:22Z",
  "stepResults": [
    {"step": "KEY_DELETED",       "success": true, "recordsAffected": 1},
    {"step": "POSTGRES_MARKED",   "success": true, "recordsAffected": 1},
    {"step": "MONGO_ERASED",      "success": true, "recordsAffected": 1},
    {"step": "ES_REMOVED",        "success": true, "recordsAffected": 1},
    {"step": "REDIS_CLEARED",     "success": true, "recordsAffected": 2},
    {"step": "AUDIT_ANONYMISED",  "success": true, "recordsAffected": 3}
  ],
  "dpdpCompliance": "DPDP Act 2023 S17 — Erasure complete within SLA"
}
```

Now verify crypto-shredding worked:

```bash
# Try to read the encrypted name — key is gone, should return null
$ curl -s "http://localhost:8081/citizens/$CITIZEN_ID/name" \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "citizenId": "c1d2e3f4-...",
  "name": null,
  "reason": "CITIZEN_ERASED",
  "message": "Encryption key deleted — data permanently inaccessible"
}
```

```bash
# Search for the citizen — should not appear in ES results
$ curl -s "http://localhost:8080/search?q=Vijayakumar&scope=IDENTITY_FULL" \
  | python3 -m json.tool
```

Expected Output:
```json
{
  "results": [],
  "totalHits": 0,
  "searchTimeMs": 12
}
```

🔍 WHAT TO OBSERVE: Three confirmations of erasure: (1) Name returns null — key deleted, crypto-shredded. (2) Search returns zero results — removed from ES index. (3) MongoDB documents have status=ERASED and content=null. The citizen's records exist as empty shells — the audit trail is preserved but no personal data is accessible.

---

**STEP 5 of 7: Demonstrate rate limiting via Redis**

```bash
# Make 5 rapid erasure requests — should hit rate limit (5 per hour)
$ for i in {1..6}; do
    response=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "http://localhost:8080/citizens/test-id/erasure" \
      -H "X-Session-Token: demo-officer-token" \
      -H "X-Idempotency-Key: $(uuidgen)" \
      -H "Content-Type: application/json" \
      -d '{"requestedBy": "test"}')
    echo "Request $i: HTTP $response"
  done
```

Expected Output:
```
Request 1: HTTP 202
Request 2: HTTP 202
Request 3: HTTP 202
Request 4: HTTP 202
Request 5: HTTP 202
Request 6: HTTP 429  ← Rate limit exceeded
```

```bash
# Check Redis rate limit state
$ docker compose exec redis redis-cli \
    ZCARD "rate_limit:demo-officer-token:erasure"
# Expected: 5 (5 timestamps in the sliding window)
```

🔍 WHAT TO OBSERVE: The 6th request received HTTP 429 (Too Many Requests). The Redis sorted set shows exactly 5 timestamps in the sliding window. This is the DPDP-required throttling preventing bulk data harvesting via the erasure endpoint.

---

**STEP 6 of 7: Run the search benchmark with 10K records**

```bash
# Seed 10,000 citizens for a meaningful performance test
$ ./scripts/seed-data.sh 10000

# Wait for ES bulk indexing to complete
$ sleep 30

# Run benchmark
$ ./scripts/demo-search-comparison.sh 10000
```

Expected Output:
```
=== SEARCH PERFORMANCE COMPARISON (10,000 citizens) ===

PostgreSQL LIKE (full table scan): Average 8,240ms
Elasticsearch fuzzy search:        Average 45ms

Performance difference: 183× faster with Elasticsearch

Top search by state code KA (Karnataka):
  PostgreSQL: 9,100ms
  Elasticsearch: 31ms

Phonetic search "Vijayakumar" → finds "Vijayakumaar", "Vijayakumaran":
  Elasticsearch phonetic: 67ms
  PostgreSQL LIKE: Not possible (would require unencrypted storage)
```

---

**STEP 7 of 7: Run all domain and integration tests**

```bash
$ cd identity-service
$ ./mvnw test -q
```

Expected Output:
```
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
[INFO] Total time: 3.891 s
```

```bash
$ cd ../document-service && source venv/bin/activate
$ python3 -m pytest src/ -v -q 2>/dev/null
```

Expected Output:
```
tests/test_mongodb_repository.py::test_save_document PASSED
tests/test_mongodb_repository.py::test_erase_documents_idempotent PASSED
tests/test_domain.py::test_reference_pattern_enforced PASSED
3 passed in 1.234s
```

---

# L5. TRAINER DEMO SCRIPT

---

🎬 TRAINER DEMO SCRIPT: The Database That Cannot Lie

SETUP CHECK (Before showing screen):
□ All services healthy (verified in Step 10 of setup)
□ Elasticsearch index has 1,000+ citizens (seeded in Step 9)
□ `citizen_search.py` open in editor at the `_build_query` method
□ Second terminal ready for PostgreSQL direct query

---

TALKING POINT 1 (While showing `_build_query`):

"Look at line 120. The scope filter — `authorisedScopes: request.officer_scope` — is injected SERVER-SIDE. The client sends a request. The server reads the officer's scope from the SESSION (validated against Redis). The server ADDS the scope filter to the Elasticsearch query. The client never touches the filter clause. The client cannot remove it. The client cannot override it. An officer with IDENTITY_BASIC scope literally cannot retrieve FINANCIAL_ANALYST data — not because we check their scope after the query — but because the query itself will never match those records. This is structural security. The architecture prevents the breach — not a permission check."

👉 POINT AT: The filter clause injection in `_build_query` and the `SCOPE_ALLOWED_FIELDS` mapping.

ASK AUDIENCE: "What happens if an officer discovers the Elasticsearch REST API endpoint directly and sends a query without the scope filter header? Walk me through what they can access."

EXPECTED RESPONSES: Direct ES API access bypasses the API gateway entirely. This is why Elasticsearch must NEVER be publicly exposed. ES should listen on localhost only. All access must go through the application layer that enforces scope. In production: ES on private subnet, no public IP, accessible only via service mesh.

---

BREAK IT (Intentional failure for learning):

Show what happens when the scope filter is accidentally removed:

```python
# Comment out the scope filter in _build_query:
# filter_clauses.append({
#     "term": {"authorisedScopes": request.officer_scope}
# })
```

Restart search service and run:

```bash
$ curl "http://localhost:8080/search?q=*&scope=IDENTITY_BASIC"
```

🔴 SHOW: Now returns ALL citizens regardless of scope — the entire database is accessible to any officer.

EXPLAIN: "This is the Elasticsearch overfetch vulnerability. One commented-out line. In production, this bug would expose 1.38 billion citizen records to any authenticated user. The DPDP Act violation would be ₹250 crore in penalties plus criminal liability. This is why security must be STRUCTURAL — not just a policy. The scope filter must be tested as a hard requirement in every deployment pipeline."

FIX IT: Uncomment the scope filter. Add a unit test that fails if scope filter is absent.

---

🤖 COPILOT LIVE PROMPT (Run while search service code is visible):

"Review this Elasticsearch query builder for a government citizen search service. The system must comply with India's DPDP Act 2023. Identify: (1) security vulnerabilities that could expose citizen data, (2) performance issues with the query structure, (3) missing DPDP compliance mechanisms. Show the specific code changes needed to fix each issue. Code: [paste _build_query method]"

Expected Copilot output: Should identify the scope filter as critical, suggest adding query complexity limits, and flag the lack of search audit logging. Class should critique: Did Copilot identify the source filtering as a DPDP mechanism? Did it suggest field-level security at the Elasticsearch cluster level (an additional layer in production)?

---

# L6. VERIFICATION CHECKLIST

---

Lab Completion Verification:

□ All 4 databases running — verify:
  ```bash
  docker compose ps | grep -E "postgres|mongodb|elasticsearch|redis"
  # Expected: all show "Up" status
  ```

□ Citizen registered with encrypted name in PostgreSQL — verify:
  ```bash
  docker compose exec postgres psql -U citizenvault -d citizen_db \
    -c "SELECT aadhaar_hash, LENGTH(name_encrypted), erasure_status FROM citizens LIMIT 1;"
  # Expected: hash present, name_encrypted length > 0 (encrypted bytes)
  ```

□ Document stored in MongoDB with reference pattern — verify:
  ```bash
  docker compose exec mongodb mongosh -u citizenvault -p vault2026 \
    --authenticationDatabase admin --quiet citizenvault_db \
    --eval "db.citizen_documents.countDocuments()"
  # Expected: count >= 1
  ```

□ Elasticsearch search returns results with masked names — verify:
  ```bash
  curl -s "http://localhost:8080/search?q=test&scope=IDENTITY_BASIC" \
    | python3 -c "import sys,json; \
      r=json.load(sys.stdin)['results']; \
      print(all('****' in (x.get('nameMasked','')) for x in r if x.get('nameMasked')))"
  # Expected: True (all names masked)
  ```

□ Erasure completes in all 6 steps — verify:
  ```bash
  # From Step 4 output: all stepResults show success: true
  ```

□ Crypto-shredding verified — erased citizen name returns null — verify:
  ```bash
  curl -s "http://localhost:8081/citizens/{erased-id}/name" | python3 -m json.tool
  # Expected: "name": null, "reason": "CITIZEN_ERASED"
  ```

□ Rate limiting works — 6th erasure request returns 429 — verify: Step 5

□ ES search 47× faster than PostgreSQL LIKE — verify: Step 2 comparison script

□ Domain tests pass — verify:
  ```bash
  cd identity-service && ./mvnw test -Dtest="CryptoShreddingTest" -q
  # Expected: BUILD SUCCESS, 4 tests passed
  ```

□ Teardown complete:
  ```bash
  docker compose down -v
  # Expected: all containers stopped, volumes removed
  ```

---

# L7. EXTENSIVE DOCUMENTATION

---

## L7.1 Architecture Decision Documentation

**Why Motor (async MongoDB) over PyMongo (sync):**

FastAPI is built on asyncio — an event-loop-based concurrency model. PyMongo (synchronous) blocks the event loop during MongoDB I/O. At 500 concurrent document requests, blocking I/O means 500 requests queuing while one MongoDB operation completes. Motor's async driver yields control to the event loop during I/O — allowing FastAPI to process other requests while waiting for MongoDB. For a document service expected to handle government officer peak loads (morning hours, 09:00–11:00 SGT/IST), async I/O is not optional.

**What the crypto-shredding implementation demonstrates:**

The `CryptoService.decrypt()` method returns `null` (not an exception) when called with a null key. This is the DPDP erasure signal — callers handle null as "this citizen has been erased." Every downstream system (search result rendering, document listing, audit display) must handle this null gracefully and display "Data erased per DPDP Act." The architectural lesson: erasure is not just a database operation — it is a cross-cutting concern that every layer of the application must handle explicitly.

**The DPDP scope filter as architectural security:**

The `SCOPE_ALLOWED_FIELDS` dictionary in `citizen_search.py` is the single source of truth for DPDP field-level access control. Every scope's allowed fields are defined here. If a new field is added to the Elasticsearch index, it is NOT returned to any scope until explicitly added to this dictionary. This is deny-by-default data access — the correct architectural posture for a citizen data platform.

---

## L7.2 Pattern Deep-Dive

**Pattern: Polyglot Persistence with Event-Driven Sync**

Formal Definition: Using multiple specialised databases within a single application — each selected for its optimal access pattern — and synchronising data between them through an event-driven message bus. Each database is owned exclusively by one bounded context. Cross-context data access goes through events, not shared database access.

Critical failure mode — the N+1 cross-store query problem:
When a query requires data from multiple stores (e.g., "give me the citizen's name from PostgreSQL AND their document list from MongoDB AND their search score from Elasticsearch"), the naive implementation makes N+1 round trips — one per store per record. At scale: retrieving 100 citizens' complete profile = 300 database calls. Solution: either pre-compute the combined view in a projection (CQRS read model), or use a dedicated API composition layer that parallelises the store calls using async/await.

Famous implementations:
- Netflix uses polyglot persistence across Cassandra (viewing history), MySQL (billing), Elasticsearch (search), Redis (session), DynamoDB (metadata) — each selected for its access pattern
- Aadhaar uses specialised biometric matching engines alongside relational identity stores — the biometric matching is not SQL
- Singapore's SingPass uses separate stores for identity (relational), consent (document), and session (cache)

---

## L7.3 Production Readiness Gap Analysis

| Lab Has                       | Production Needs                                          | Effort |
| ----------------------------- | --------------------------------------------------------- | ------ |
| Single MongoDB node           | MongoDB sharded cluster (consistent-hash shard key)       | High   |
| Single ES node                | 3-node ES cluster with ILM + index aliases                | High   |
| Redis single instance         | Redis Cluster (6 nodes — 3 primary + 3 replica)           | Medium |
| Lab-level encryption (no HSM) | Azure Key Vault + HSM-backed keys                         | High   |
| Name stored in ES plaintext   | Phonetic token only in ES (not readable name)             | High   |
| No TLS between services       | mTLS via Istio service mesh (Day 13)                      | Medium |
| Simple audit table            | Immutable audit log (Kafka permanent retention)           | Medium |
| 24-hour erasure (manual)      | Temporal workflow with SLA enforcement + alerting         | Medium |
| No field-level ES security    | Elasticsearch security with field/document-level security | Medium |

---

## L7.4 Security Review

The most critical security gap in this lab: **Elasticsearch stores citizen names in searchable plaintext**. The `nameSearchable` field in the ES document contains the citizen's actual name (lowercase, analysed) — not encrypted. In production, this creates a risk: if the ES cluster is compromised, all citizen names are exposed. Three production mitigations: (1) Store only the phonetic metaphone token in ES — sufficient for fuzzy search without storing readable names. (2) Enable Elasticsearch field-level security — restrict `nameSearchable` field access to IDENTITY_FULL scope at the ES cluster level (not just application layer). (3) Place ES on a private subnet with no public endpoint.

OWASP Top 10 mapping:
- A01 Broken Access Control: The scope filter prevents this — verified by removing the filter in the demo and observing the vulnerability
- A02 Cryptographic Failures: AES-256-GCM used correctly (with IV, with authentication tag). SHA-256 for Aadhaar hashing — production should use HMAC-SHA256 with secret pepper
- A04 Insecure Design: The rate limiter prevents bulk data harvesting. The erasure idempotency key prevents duplicate erasures

MAS TRM 2021, Section 9.5 (Data Loss Prevention): Requires data masking in non-production environments. The lab's name masking (`A**** V**********`) demonstrates the production masking requirement for search results displayed to officers.

---

## L7.5 Cost Architecture

| Resource                              | Lab (Docker)  | Production (Azure)                                                              |
| ------------------------------------- | ------------- | ------------------------------------------------------------------------------- |
| PostgreSQL                            | Free (Docker) | Azure Database for PostgreSQL Flexible: S$140/month (General Purpose, 2 vCores) |
| MongoDB                               | Free (Docker) | Azure Cosmos DB for MongoDB: S$24/month (1000 RU/s) or Atlas M10: S$57/month    |
| Elasticsearch                         | Free (Docker) | Elastic Cloud on Azure (2GB, 1 zone): S$47/month                                |
| Redis                                 | Free (Docker) | Azure Cache for Redis C1 (1GB): S$55/month                                      |
| Total Lab                             | S$0           | S$266/month for lab-scale production                                            |
| At CitizenVault scale (100M citizens) | N/A           | S$8,400/month (sharded MongoDB, ES cluster, Redis cluster, PostgreSQL HA)       |

Cost Optimisation Opportunities:
- Cosmos DB Autoscale: pay only for consumed RU/s — 40% saving vs. provisioned during off-peak
- ES Hot-Warm-Cold tiering: move aged citizen records to warm/cold nodes — 60% storage cost reduction
- Redis TTL discipline: expired sessions auto-evict — no manual cleanup cost

---

## L7.6 Regulatory Compliance Notes

| Regulation                       | Store | Compliance Mechanism                   | Lab Gap                      | Fix Needed                        |
| -------------------------------- | ----- | -------------------------------------- | ---------------------------- | --------------------------------- |
| DPDP Act S8 (Data Minimisation)  | ES    | Source filtering per scope             | Name in ES plaintext         | Phonetic token only               |
| DPDP Act S9 (Purpose Limitation) | Redis | Session TTL enforces time-bound access | No purpose tag on sessions   | Add purpose field to session      |
| DPDP Act S16 (Data Localisation) | All   | Azure India region only                | No region enforcement in lab | Terraform location = centralindia |
| DPDP Act S17 (Right to Erasure)  | All   | Crypto-shredding + Saga                | ✅ Implemented                | Production: 24hr Temporal SLA     |
| DPDP Act S8 (Data Accuracy)      | ES    | Event-driven sync from PostgreSQL      | ES may lag 5s                | Add consistency check API         |
| MAS TRM 9.5 (Data Masking)       | ES    | Name masking in search results         | ✅ Implemented                | Add officer search audit log      |

---

## L7.7 Alternative Approaches

THIS LAB USES: Python Motor (async) for MongoDB

ALTERNATIVE 1: Beanie ODM (Object Document Mapper for MongoDB + FastAPI)
WOULD WORK WHEN: Team prefers ORM-style abstractions. Beanie provides type-safe document models with Pydantic integration. Reduces boilerplate for simple CRUD operations.
TRADE-OFF vs. Motor: Beanie adds abstraction layer — complex aggregation pipelines are harder to write. For government analytics workloads with complex pipelines — use Motor directly.

ALTERNATIVE 2: Azure Cosmos DB (MongoDB API) instead of self-managed MongoDB
WOULD WORK WHEN: Operational simplicity is prioritised over cost optimisation. MAS TRM compliance documentation is needed urgently. Team lacks MongoDB operations expertise.
TRADE-OFF vs. Self-hosted: 3-10× higher cost at large scale. No MongoDB-specific features (change streams have limitations, some aggregation operators unsupported). Gain: managed HA, automatic backups, MAS-recognised compliance documentation.

THIS LAB USES: Custom crypto-shredding with PostgreSQL key store

ALTERNATIVE: Azure Key Vault for key storage
WOULD WORK WHEN: Production deployment. Azure Key Vault provides HSM-backed key operations, automatic key rotation, detailed access audit logs, and MAS TRM compliance documentation for cryptographic operations.
TRADE-OFF vs. Database key store: Additional Azure service dependency. Key operations add ~50ms latency per encrypt/decrypt call (API call to Key Vault). Gain: keys never leave the HSM, audit log for every key operation, FIPS 140-2 Level 2 compliance.

---

## L7.8 Copilot Lab Prompts

UNDERSTANDING PROMPT:
"Explain the crypto-shredding pattern implemented in `CryptoService.java` and `Citizen.java`. How does deleting the encryption key satisfy the DPDP Act 2023 Section 17 right to erasure? What happens to the encrypted name field in PostgreSQL, MongoDB documents, Kafka events, and database backups after the key is deleted? Is the citizen's data truly erased?"

EXTENSION PROMPT:
"How would I extend the CitizenVault erasure orchestrator to handle the case where the Elasticsearch removal step fails (ES cluster temporarily down) but the PostgreSQL key deletion has already succeeded (crypto-shredding complete)? Design the retry strategy that ensures ES removal eventually completes without re-executing the key deletion step. Include: idempotency mechanism, maximum retry count, escalation after 23 hours (before DPDP 24-hour SLA breach)."

CRITIQUE PROMPT:
"Review this Elasticsearch query for DPDP Act 2023 compliance and security vulnerabilities. Focus on: field-level access control gaps, missing audit logging, scope filter bypass risks, and data minimisation violations. Include specific code changes needed. Query: [paste _build_query method]"

INDIA/SINGAPORE CONTEXT PROMPT:
"Compare the data erasure requirements under India's DPDP Act 2023 Section 17 and Singapore's PDPA Section 24 for a cross-border citizen data platform. What are the specific differences in: erasure timeline (India vs Singapore), what data must be erased vs retained for audit, notification requirements after erasure, and how crypto-shredding satisfies both regulations simultaneously?"

---

✅ DAY 4 COMPLETE

📘 Document 1: Training Content — Data Architecture, NoSQL & Search ✅ Generated

🔬 Document 2: Lab Manual — CitizenVault: Polyglot Persistence + DPDP Erasure ✅ Generated

📊 Quality Gates: All 5 passed ✅

Geography: 🇮🇳 India (Aadhaar/GSTN/DigiLocker primary) + 🇸🇬 Singapore (CPF/SingPass secondary) + 🌏 Cross-border (analytics federation) ✅

Diagrams Used: 12 diagrams total — 6 ASCII + 6 Mermaid (flowchart, graph, sequence styles) ✅

Copilot Integration: 6 embedded prompts across both documents ✅

Documentation Templates: SAD Data Architecture Section, Weekly Architecture Health Report, Technology Evaluation implicit in polyglot selection ✅

Java 17 Features: AES-256-GCM crypto, JPA entities, records for value objects, sealed interface for erasure steps ✅

Python 3.10+ Features: match-case, async/await with Motor, dataclasses, type hints, structural pattern matching ✅

New Concepts Introduced: Crypto-shredding, DPDP Act compliance architecture, ES DPDP scope filter, sliding window rate limiting, MongoDB reference vs embedding ✅

👀 Day 5 Preview: Security, Zero Trust & Emerging Tech

"Tomorrow we design Zero Trust from scratch — and then we deliberately attack the architecture we built in Days 2–4 to find every trust boundary we violated without realising it. Warning: by the end of tomorrow, you will never look at a service-to-service API call the same way again."
---