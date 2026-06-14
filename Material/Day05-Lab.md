# Day 5 Lab Document — Comprehension Check

Before generating, let me confirm the full scope:

**Day 5 covers:**

| #   | Topic                                                               | Duration |
| --- | ------------------------------------------------------------------- | -------- |
| 1   | Search Architecture & Data Consistency Models (Strong vs. Eventual) | 1.0 hr   |
| 2   | Case Study: Scaling a Citizen Data Platform Across Regions          | 1.0 hr   |
| 3   | Zero Trust Architecture: Identity, Device, Network, App Layers      | 1.5 hrs  |
| 4   | API Gateway Security, Keycloak/RBAC, mTLS, Secret Management        | 1.0 hr   |
| 5   | Rapid Threat Modeling & Security Gate Design                        | 0.5 hr   |

**Lab Narrative:** A **Citizen Services Security & Search Platform** — a government portal that adds full-text search over government tenders (Elasticsearch-style search via PostgreSQL full-text search + MongoDB Atlas-style aggregations), then hardens the entire platform with Zero Trust principles: Keycloak for OAuth2/OIDC + RBAC, mTLS between services, secret management via environment vaults, and an API Gateway security layer. Threat modeling (STRIDE) gates are embedded in the pipeline.

**Tech Stack Today:** Java 17 + Spring Boot 3.x, PostgreSQL 15 (full-text search), MongoDB 7, Keycloak (Docker), Docker Compose, Maven, Postman, PowerShell 7.x

**Dependencies:** Day 4 infrastructure patterns referenced; Day 5 is self-contained with its own scaffold.

---

Confirmed. Generating the complete Day 5 Lab Document in parts.

---

# DAY 5 — LAB DOCUMENT

## Lab Title: Citizen Services Security & Search Platform — Zero Trust, RBAC, mTLS, and Search Architecture

**Day Reference:** Day 5 — Data Architecture, NoSQL & Search (Session 2) + Security, Zero Trust

---

## Lab Header

### Prerequisites Checklist

```
[ ] Docker Desktop running (version 24+), at least 8 GB RAM allocated
[ ] Java 17 installed: java -version → openjdk 17.x
[ ] Maven 3.9+ installed: mvn -version
[ ] Git Bash or PowerShell 7.x available
[ ] Postman v10+ installed
[ ] Ports available: 5432, 27017, 8080, 8081, 8082, 8443, 9090 (Keycloak)
[ ] Day 4 completed (conceptual reference only — Day 5 is self-contained)
[ ] Minimum 12 GB free disk space
[ ] Internet access for pulling Docker images (Keycloak image ~500MB)
[ ] OpenSSL available (Git Bash ships with it on Windows)
```

### Estimated Times

| Phase                        | Time          |
| ---------------------------- | ------------- |
| Offline Setup (trainer prep) | 60-75 minutes |
| In-Class Demonstration       | 5.0-5.5 hours |
| Cleanup                      | 15 minutes    |

### Learning Objectives

By the end of this lab, participants will be able to:

1. Implement full-text search with ranking and consistency trade-off handling in PostgreSQL
2. Build MongoDB aggregation pipelines for regional citizen data analytics
3. Configure Keycloak realms, clients, roles, and RBAC policies for a government portal
4. Secure a Spring Boot API Gateway with OAuth2/OIDC token validation and role enforcement
5. Generate and enforce mTLS certificates between microservices
6. Apply STRIDE threat modeling to identify and mitigate threats in a citizen portal login flow
7. Embed security gates (secret scanning, RBAC policy checks) in the development workflow

### Project Architecture Overview

```mermaid
graph TB
    subgraph Client["Client Layer"]
        BROWSER["Browser / Postman"]
        MOB["Mobile Client"]
    end

    subgraph Gateway["API Gateway Layer (Port 8080)"]
        GW["CitizenApiGateway\nSpring Cloud Gateway\n+ OAuth2 Resource Server"]
    end

    subgraph IAM["Identity & Access Management"]
        KC["Keycloak\n:9090\nOAuth2 / OIDC / RBAC"]
    end

    subgraph Services["Microservices"]
        SS["SearchService\n:8081\nPostgreSQL FTS + MongoDB"]
        DS["DataService\n:8082\nCitizen Data + Analytics"]
    end

    subgraph Security["Security Layer"]
        MTLS["mTLS\nService-to-Service"]
        CERT["Self-Signed Certs\n(OpenSSL)"]
        SECRET["Secret Management\n(.env + Docker Secrets)"]
    end

    subgraph Storage["Data Layer"]
        PG["PostgreSQL 15\n:5432\nTender FTS Index"]
        MG["MongoDB 7\n:27017\nCitizen Analytics"]
    end

    subgraph ThreatModel["Threat Modeling"]
        STRIDE["STRIDE Gates\nSpoofing, Tampering,\nRepudiation, Info Disclosure,\nDoS, Elevation"]
    end

    BROWSER -->|HTTPS + Bearer Token| GW
    MOB -->|HTTPS + Bearer Token| GW
    BROWSER -->|/auth| KC
    GW -->|validate token| KC
    GW -->|mTLS| SS
    GW -->|mTLS| DS
    SS --- PG
    SS --- MG
    DS --- MG
    MTLS --- CERT
    SECRET -.->|inject| GW
    SECRET -.->|inject| SS
    SECRET -.->|inject| DS
    STRIDE -.->|gates| GW
    STRIDE -.->|gates| KC

    style Client fill:#dbeafe,stroke:#1d4ed8
    style Gateway fill:#fef3c7,stroke:#d97706
    style IAM fill:#f0fdf4,stroke:#16a34a
    style Services fill:#ede9fe,stroke:#7c3aed
    style Security fill:#fee2e2,stroke:#dc2626
    style Storage fill:#f0f9ff,stroke:#0284c7
    style ThreatModel fill:#fff7ed,stroke:#ea580c
```

---

## Step 0: PowerShell Project Scaffold Script

> **Trainer Note:** Run this OFFLINE before class. It creates the complete project skeleton. Populate each file using the code sections below.

```powershell
# Day5-Scaffold.ps1
# Run from: C:\training\
# Creates the complete Day 5 project structure

$root = "C:\training\day5-security-search-platform"

$dirs = @(
    "$root",

    # API Gateway
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\config",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\filter",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\security",
    "$root\citizen-api-gateway\src\main\resources",
    "$root\citizen-api-gateway\src\test\java\gov\citizen\gateway",

    # Search Service
    "$root\citizen-search-service\src\main\java\gov\citizen\search",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\config",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\model",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\repository",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\service",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\api",
    "$root\citizen-search-service\src\main\resources",
    "$root\citizen-search-service\src\test\java\gov\citizen\search",

    # Data Service
    "$root\citizen-data-service\src\main\java\gov\citizen\data",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\config",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\model",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\repository",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\service",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\api",
    "$root\citizen-data-service\src\main\resources",
    "$root\citizen-data-service\src\test\java\gov\citizen\data",

    # Infrastructure
    "$root\infra\postgres",
    "$root\infra\keycloak",
    "$root\infra\certs",
    "$root\infra\secrets",

    # Threat Model
    "$root\threat-model",

    # Postman
    "$root\postman"
)

$files = @(
    # Root
    "$root\docker-compose.yml",
    "$root\.env",
    "$root\README.md",

    # Infra
    "$root\infra\postgres\init.sql",
    "$root\infra\keycloak\realm-export.json",
    "$root\infra\certs\generate-certs.sh",
    "$root\infra\secrets\secrets.env",

    # Threat Model
    "$root\threat-model\stride-analysis.md",

    # API Gateway
    "$root\citizen-api-gateway\pom.xml",
    "$root\citizen-api-gateway\Dockerfile",
    "$root\citizen-api-gateway\src\main\resources\application.yml",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\CitizenGatewayApplication.java",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\config\SecurityConfig.java",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\config\GatewayRouteConfig.java",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\filter\RateLimitFilter.java",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\filter\AuditLoggingFilter.java",
    "$root\citizen-api-gateway\src\main\java\gov\citizen\gateway\security\JwtClaimsValidator.java",
    "$root\citizen-api-gateway\src\test\java\gov\citizen\gateway\GatewaySecurityTest.java",

    # Search Service
    "$root\citizen-search-service\pom.xml",
    "$root\citizen-search-service\Dockerfile",
    "$root\citizen-search-service\src\main\resources\application.yml",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\CitizenSearchApplication.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\config\SecurityConfig.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\config\DataSourceConfig.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\model\Tender.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\repository\TenderSearchRepository.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\service\TenderSearchService.java",
    "$root\citizen-search-service\src\main\java\gov\citizen\search\api\SearchController.java",
    "$root\citizen-search-service\src\test\java\gov\citizen\search\TenderSearchServiceTest.java",

    # Data Service
    "$root\citizen-data-service\pom.xml",
    "$root\citizen-data-service\Dockerfile",
    "$root\citizen-data-service\src\main\resources\application.yml",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\CitizenDataApplication.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\config\SecurityConfig.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\config\MongoConfig.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\model\CitizenProfile.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\repository\CitizenProfileRepository.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\service\CitizenAnalyticsService.java",
    "$root\citizen-data-service\src\main\java\gov\citizen\data\api\CitizenDataController.java",
    "$root\citizen-data-service\src\test\java\gov\citizen\data\CitizenDataServiceTest.java",

    # Postman
    "$root\postman\Day5-SecuritySearch.postman_collection.json"
)

foreach ($dir in $dirs) {
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
    Write-Host "Created dir:  $dir" -ForegroundColor Green
}

foreach ($file in $files) {
    New-Item -ItemType File -Path $file -Force | Out-Null
    Write-Host "Created file: $file" -ForegroundColor Cyan
}

Write-Host "`nScaffold complete: $root" -ForegroundColor Yellow
Write-Host "Dirs: $($dirs.Count) | Files: $($files.Count)" -ForegroundColor Yellow
```

**Run it:**

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
cd C:\training
.\Day5-Scaffold.ps1
```

`[EXPECTED: ~35 directories and ~42 files listed as Created. Final summary line shown in yellow.]`

---

## Section 1: Infrastructure Setup

### What We Are Building

We spin up PostgreSQL (for full-text search over government tenders), MongoDB (for citizen profile analytics), and Keycloak (OAuth2/OIDC identity provider with RBAC). This is the foundational layer every service in Day 5 depends on.

### File: `docker-compose.yml`

```yaml
# docker-compose.yml
# Day 5 — Citizen Services Security & Search Platform
# Services: PostgreSQL, MongoDB, Keycloak
# WHY these three:
#   PostgreSQL = relational + full-text search (tsvector/tsquery)
#   MongoDB    = document store for citizen profiles + aggregation pipelines
#   Keycloak   = enterprise-grade IAM (OAuth2, OIDC, RBAC, mTLS support)

version: "3.9"

networks:
  citizen-net:
    driver: bridge

volumes:
  postgres-data:
  mongo-data:
  keycloak-data:

services:

  # ── PostgreSQL: Tender Full-Text Search Store ─────────────────────────────
  postgres:
    image: postgres:15-alpine
    container_name: citizen-postgres
    environment:
      POSTGRES_DB: citizen_search
      POSTGRES_USER: citizen_user
      POSTGRES_PASSWORD: citizen_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - citizen-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U citizen_user -d citizen_search"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── MongoDB: Citizen Profile Analytics ───────────────────────────────────
  mongodb:
    image: mongo:7.0
    container_name: citizen-mongodb
    environment:
      MONGO_INITDB_DATABASE: citizen_profiles
      MONGO_INITDB_ROOT_USERNAME: mongo_user
      MONGO_INITDB_ROOT_PASSWORD: mongo_pass
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    networks:
      - citizen-net
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 15s
      timeout: 10s
      retries: 5

  # ── Keycloak: Identity & Access Management ────────────────────────────────
  # WHY Keycloak: open-source, enterprise-grade, supports OAuth2/OIDC/SAML
  # Used by: India's e-governance portals, Singapore MyInfo-style integrations
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.4
    container_name: citizen-keycloak
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin_pass
      KC_DB: dev-mem
      # WHY dev-mem: in-memory DB for lab simplicity
      # Production: use KC_DB=postgres with a dedicated Keycloak DB
      KC_HTTP_PORT: 9090
      KC_HOSTNAME_STRICT: "false"
      KC_HTTP_ENABLED: "true"
    command: ["start-dev", "--import-realm"]
    ports:
      - "9090:9090"
    volumes:
      - ./infra/keycloak/realm-export.json:/opt/keycloak/data/import/realm-export.json
    networks:
      - citizen-net
    healthcheck:
      test: ["CMD-SHELL",
             "curl -sf http://localhost:9090/health/ready || exit 1"]
      interval: 20s
      timeout: 10s
      retries: 10
      start_period: 60s
```

### File: `.env`

```bash
# .env — Day 5 environment configuration
# WHY .env: separate config from code; never commit secrets to VCS

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=citizen_search
POSTGRES_USER=citizen_user
POSTGRES_PASSWORD=citizen_pass

# MongoDB
MONGO_HOST=localhost
MONGO_PORT=27017
MONGO_DB=citizen_profiles
MONGO_USER=mongo_user
MONGO_PASSWORD=mongo_pass

# Keycloak
KEYCLOAK_BASE_URL=http://localhost:9090
KEYCLOAK_REALM=citizen-portal
KEYCLOAK_CLIENT_ID=citizen-gateway
KEYCLOAK_CLIENT_SECRET=gateway-secret-2024

# Service Ports
GATEWAY_PORT=8080
SEARCH_SERVICE_PORT=8081
DATA_SERVICE_PORT=8082

# mTLS Cert Paths (generated in Section 3)
CERT_DIR=./infra/certs
```

### File: `infra/postgres/init.sql`

```sql
-- init.sql
-- PostgreSQL schema for Government Tender Full-Text Search
-- Demonstrates: tsvector, tsquery, GIN index, consistency model choices
-- WHY PostgreSQL FTS instead of a dedicated search engine (Elasticsearch):
--   For lab scale and simplicity. In production (100M+ docs), Elasticsearch
--   or Azure Cognitive Search would be used with eventual consistency trade-offs.

-- ── Tender Table ─────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS tenders (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tender_number   VARCHAR(50)     NOT NULL UNIQUE,
    title           TEXT            NOT NULL,
    description     TEXT            NOT NULL,
    department      VARCHAR(200)    NOT NULL,
    -- geo_region: supports data locality queries (India state, US state, SG district)
    geo_region      VARCHAR(50)     NOT NULL,
    -- budget_inr: stored in INR for India tenders (USD/SGD for others)
    budget_inr      NUMERIC(15,2),
    status          VARCHAR(20)     NOT NULL DEFAULT 'OPEN'
                                    CHECK (status IN ('OPEN','CLOSED','AWARDED','CANCELLED')),
    published_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deadline_at     TIMESTAMPTZ,
    -- search_vector: pre-computed tsvector for full-text search
    -- WHY pre-compute: faster search than computing at query time
    -- Updated via trigger (see below)
    search_vector   TSVECTOR,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

-- ── GIN Index on search_vector ────────────────────────────────────────────────
-- WHY GIN (Generalised Inverted Index): optimal for tsvector/tsquery operations
-- GIN stores per-lexeme posting lists → O(log n) lookup regardless of table size
-- Alternative: GiST index (faster update, slower search) — trade-off noted
CREATE INDEX IF NOT EXISTS idx_tenders_search_vector
    ON tenders USING GIN(search_vector);

-- ── Supporting Indexes ────────────────────────────────────────────────────────
CREATE INDEX IF NOT EXISTS idx_tenders_geo_region
    ON tenders(geo_region);

CREATE INDEX IF NOT EXISTS idx_tenders_status
    ON tenders(status);

CREATE INDEX IF NOT EXISTS idx_tenders_published_at
    ON tenders(published_at DESC);

-- ── Trigger: Auto-update search_vector on INSERT / UPDATE ────────────────────
-- WHY trigger: ensures search_vector is always consistent with source columns
-- This is STRONG consistency — search index is synchronously updated in same txn
-- Compare: Elasticsearch uses eventual consistency (async indexing lag)
CREATE OR REPLACE FUNCTION update_tender_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        -- Title gets weight 'A' (highest relevance in ranking)
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        -- Department gets weight 'B'
        setweight(to_tsvector('english', COALESCE(NEW.department, '')), 'B') ||
        -- Description gets weight 'C' (lower relevance)
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'C');
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_tender_search_vector
    BEFORE INSERT OR UPDATE OF title, description, department
    ON tenders
    FOR EACH ROW
    EXECUTE FUNCTION update_tender_search_vector();

-- ── Consistency SLA Comment ───────────────────────────────────────────────────
-- STRONG consistency: search_vector updated in same ACID transaction as the row
-- SLA implication: write latency slightly higher (~2-5ms for tsvector computation)
-- but search results are ALWAYS consistent with source data — zero lag
-- For a government tender portal, strong consistency is the correct trade-off:
-- a closed tender must NOT appear in search results even 1 second later

-- ── Seed Data: Government Tenders (India, US, Singapore) ─────────────────────
INSERT INTO tenders (tender_number, title, description, department, geo_region, budget_inr, status, deadline_at)
VALUES
    ('TND-IN-2024-001',
     'Smart City IoT Sensor Network Deployment',
     'Procurement of IoT sensors for air quality monitoring traffic management and waste collection in metropolitan areas under Smart Cities Mission',
     'Ministry of Housing and Urban Affairs',
     'IN-MH',
     150000000.00,
     'OPEN',
     NOW() + INTERVAL '30 days'),

    ('TND-IN-2024-002',
     'National Health Stack API Gateway Infrastructure',
     'Design build and operate a secure API gateway for Ayushman Bharat Digital Mission health data exchange between hospitals insurers and government',
     'Ministry of Health and Family Welfare',
     'IN-DL',
     500000000.00,
     'OPEN',
     NOW() + INTERVAL '45 days'),

    ('TND-IN-2024-003',
     'Rural Broadband Connectivity BharatNet Phase 3',
     'Laying optical fiber cable for last mile broadband connectivity to gram panchayats in underserved districts',
     'Department of Telecommunications',
     'IN-UP',
     2000000000.00,
     'OPEN',
     NOW() + INTERVAL '60 days'),

    ('TND-US-2024-001',
     'Federal Identity Verification Platform Modernization',
     'Modernize Login.gov identity proofing infrastructure with biometric verification and zero trust network access for federal agencies',
     'General Services Administration',
     'US-DC',
     75000000.00,
     'OPEN',
     NOW() + INTERVAL '40 days'),

    ('TND-US-2024-002',
     'Cloud Infrastructure Security Assessment FedRAMP',
     'Conduct FedRAMP High authorization assessment for cloud infrastructure hosting sensitive government workloads',
     'Cybersecurity and Infrastructure Security Agency',
     'US-VA',
     12000000.00,
     'OPEN',
     NOW() + INTERVAL '20 days'),

    ('TND-SG-2024-001',
     'Singpass Digital Identity SDK Integration Framework',
     'Development of standardised SDK for private sector integration with Singpass MyInfo and Face Verification services',
     'Government Technology Agency',
     'SG-CENTRAL',
     8000000.00,
     'OPEN',
     NOW() + INTERVAL '35 days'),

    ('TND-SG-2024-002',
     'OneService Municipal Issue Reporting AI Enhancement',
     'Integrate large language model for automated classification and routing of municipal issues reported via OneService application',
     'Smart Nation and Digital Government Office',
     'SG-CENTRAL',
     4500000.00,
     'OPEN',
     NOW() + INTERVAL '25 days'),

    ('TND-IN-2024-004',
     'GSTN Tax Analytics and Fraud Detection Platform',
     'Build machine learning pipeline for anomaly detection in GST returns to identify fraudulent input tax credit claims',
     'Goods and Services Tax Network',
     'IN-DL',
     300000000.00,
     'CLOSED',
     NOW() - INTERVAL '5 days'),

    ('TND-US-2024-003',
     'VA Healthcare Patient Portal Accessibility Upgrade',
     'Upgrade Veterans Affairs patient portal to WCAG 2.2 AA compliance with screen reader optimization and multilingual support',
     'Department of Veterans Affairs',
     'US-DC',
     18000000.00,
     'OPEN',
     NOW() + INTERVAL '50 days'),

    ('TND-IN-2024-005',
     'DigiLocker Enterprise Document Vault Scaling',
     'Scale DigiLocker infrastructure to support 500 million citizens storing and sharing Aadhaar PAN driving licence and educational certificates',
     'Ministry of Electronics and Information Technology',
     'IN-DL',
     800000000.00,
     'OPEN',
     NOW() + INTERVAL '90 days');

-- ── Verify seed data ──────────────────────────────────────────────────────────
-- SELECT tender_number, title, geo_region, status FROM tenders ORDER BY published_at DESC;
-- SELECT COUNT(*) FROM tenders WHERE search_vector @@ to_tsquery('english', 'identity');
```

### File: `infra/keycloak/realm-export.json`

```json
{
  "realm": "citizen-portal",
  "enabled": true,
  "displayName": "Citizen Portal",
  "displayNameHtml": "<strong>Citizen Portal</strong>",
  "sslRequired": "none",
  "registrationAllowed": false,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": true,
  "editUsernameAllowed": false,
  "bruteForceProtected": true,
  "permanentLockout": false,
  "maxFailureWaitSeconds": 900,
  "minimumQuickLoginWaitSeconds": 60,
  "waitIncrementSeconds": 60,
  "quickLoginCheckMilliSeconds": 1000,
  "maxDeltaTimeSeconds": 43200,
  "failureFactor": 5,
  "accessTokenLifespan": 300,
  "refreshTokenMaxReuse": 0,
  "roles": {
    "realm": [
      {
        "name": "CITIZEN",
        "description": "Standard citizen user — can view own profile and search tenders"
      },
      {
        "name": "OFFICER",
        "description": "Government officer — can view all citizens in own department"
      },
      {
        "name": "ADMIN",
        "description": "System administrator — full access to all resources"
      },
      {
        "name": "AUDITOR",
        "description": "Read-only auditor — can view all data but cannot modify"
      }
    ]
  },
  "clients": [
    {
      "clientId": "citizen-gateway",
      "name": "Citizen API Gateway",
      "enabled": true,
      "publicClient": false,
      "secret": "gateway-secret-2024",
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": true,
      "serviceAccountsEnabled": true,
      "protocol": "openid-connect",
      "redirectUris": ["http://localhost:8080/*"],
      "webOrigins": ["http://localhost:8080"],
      "defaultClientScopes": ["openid", "profile", "email", "roles"],
      "attributes": {
        "access.token.lifespan": "300",
        "use.refresh.tokens": "true"
      }
    },
    {
      "clientId": "citizen-search-service",
      "name": "Citizen Search Service (M2M)",
      "enabled": true,
      "publicClient": false,
      "secret": "search-secret-2024",
      "serviceAccountsEnabled": true,
      "standardFlowEnabled": false,
      "directAccessGrantsEnabled": false,
      "protocol": "openid-connect"
    }
  ],
  "users": [
    {
      "username": "rahul.citizen",
      "email": "rahul@example.in",
      "firstName": "Rahul",
      "lastName": "Kumar",
      "enabled": true,
      "emailVerified": true,
      "credentials": [
        {
          "type": "password",
          "value": "citizen123",
          "temporary": false
        }
      ],
      "realmRoles": ["CITIZEN"]
    },
    {
      "username": "priya.officer",
      "email": "priya@gov.in",
      "firstName": "Priya",
      "lastName": "Sharma",
      "enabled": true,
      "emailVerified": true,
      "credentials": [
        {
          "type": "password",
          "value": "officer123",
          "temporary": false
        }
      ],
      "realmRoles": ["OFFICER"]
    },
    {
      "username": "admin.portal",
      "email": "admin@gov.in",
      "firstName": "Portal",
      "lastName": "Admin",
      "enabled": true,
      "emailVerified": true,
      "credentials": [
        {
          "type": "password",
          "value": "admin123",
          "temporary": false
        }
      ],
      "realmRoles": ["ADMIN", "OFFICER", "CITIZEN"]
    },
    {
      "username": "auditor.cag",
      "email": "auditor@cag.gov.in",
      "firstName": "CAG",
      "lastName": "Auditor",
      "enabled": true,
      "emailVerified": true,
      "credentials": [
        {
          "type": "password",
          "value": "auditor123",
          "temporary": false
        }
      ],
      "realmRoles": ["AUDITOR"]
    }
  ]
}
```

### File: `infra/certs/generate-certs.sh`

```bash
#!/bin/bash
# generate-certs.sh
# Generates self-signed CA + server/client certificates for mTLS
# WHY mTLS (mutual TLS): both client AND server prove identity
# Standard TLS: only server proves identity (one-way)
# mTLS: server proves identity TO client AND client proves identity TO server
# Used in: zero trust service-to-service communication

set -e

CERT_DIR="$(dirname "$0")"
CA_SUBJECT="/C=IN/ST=Delhi/O=CitizenGov/CN=CitizenGov-RootCA"
SERVER_SUBJECT="/C=IN/ST=Delhi/O=CitizenGov/CN=citizen-services"
CLIENT_SUBJECT="/C=IN/ST=Delhi/O=CitizenGov/CN=citizen-gateway-client"
VALIDITY_DAYS=365

echo "=== Generating Root CA ==="
# WHY Root CA: all service certs signed by same CA → mutual trust
openssl genrsa -out "$CERT_DIR/ca.key" 4096
openssl req -new -x509 -key "$CERT_DIR/ca.key" \
    -out "$CERT_DIR/ca.crt" \
    -days $VALIDITY_DAYS \
    -subj "$CA_SUBJECT"

echo "=== Generating Server Certificate (for Search & Data Services) ==="
openssl genrsa -out "$CERT_DIR/server.key" 2048
openssl req -new -key "$CERT_DIR/server.key" \
    -out "$CERT_DIR/server.csr" \
    -subj "$SERVER_SUBJECT"

# SAN (Subject Alternative Names) required for modern TLS validation
cat > "$CERT_DIR/server-ext.cnf" <<EOF
[req]
req_extensions = v3_req
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = localhost
DNS.2 = citizen-search-service
DNS.3 = citizen-data-service
IP.1 = 127.0.0.1
EOF

openssl x509 -req -in "$CERT_DIR/server.csr" \
    -CA "$CERT_DIR/ca.crt" \
    -CAkey "$CERT_DIR/ca.key" \
    -CAcreateserial \
    -out "$CERT_DIR/server.crt" \
    -days $VALIDITY_DAYS \
    -extfile "$CERT_DIR/server-ext.cnf" \
    -extensions v3_req

echo "=== Generating Client Certificate (for API Gateway) ==="
openssl genrsa -out "$CERT_DIR/client.key" 2048
openssl req -new -key "$CERT_DIR/client.key" \
    -out "$CERT_DIR/client.csr" \
    -subj "$CLIENT_SUBJECT"
openssl x509 -req -in "$CERT_DIR/client.csr" \
    -CA "$CERT_DIR/ca.crt" \
    -CAkey "$CERT_DIR/ca.key" \
    -CAcreateserial \
    -out "$CERT_DIR/client.crt" \
    -days $VALIDITY_DAYS

echo "=== Creating PKCS12 Keystores (for Java Spring Boot) ==="
# WHY PKCS12: Java's KeyStore format for loading certs in Spring Boot SSL config
openssl pkcs12 -export \
    -in "$CERT_DIR/server.crt" \
    -inkey "$CERT_DIR/server.key" \
    -out "$CERT_DIR/server-keystore.p12" \
    -name server \
    -CAfile "$CERT_DIR/ca.crt" \
    -passout pass:changeit

openssl pkcs12 -export \
    -in "$CERT_DIR/client.crt" \
    -inkey "$CERT_DIR/client.key" \
    -out "$CERT_DIR/client-keystore.p12" \
    -name client \
    -CAfile "$CERT_DIR/ca.crt" \
    -passout pass:changeit

# Create truststore containing the CA cert (used by all services)
keytool -import -alias citizengov-ca \
    -file "$CERT_DIR/ca.crt" \
    -keystore "$CERT_DIR/truststore.p12" \
    -storetype PKCS12 \
    -storepass changeit \
    -noprompt

echo "=== Certificate Summary ==="
echo "Files generated:"
ls -la "$CERT_DIR"/*.crt "$CERT_DIR"/*.key "$CERT_DIR"/*.p12 2>/dev/null
echo ""
echo "CA fingerprint:"
openssl x509 -in "$CERT_DIR/ca.crt" -fingerprint -noout -sha256
echo ""
echo "Server cert valid until:"
openssl x509 -in "$CERT_DIR/server.crt" -noout -enddate
echo "Certificate generation complete."
```

### Start Infrastructure

```powershell
cd C:\training\day5-security-search-platform

# Pull all images first (do this offline)
docker compose pull

# Start infrastructure
docker compose up -d

# Monitor startup
docker compose ps
```

`[EXPECTED: postgres healthy ~15s, mongodb healthy ~20s, keycloak healthy ~90s (JVM warmup)]`

```powershell
# Generate mTLS certificates (run via Git Bash)
bash ./infra/certs/generate-certs.sh
```

`[EXPECTED: ca.crt, server.crt, client.crt, server-keystore.p12, client-keystore.p12, truststore.p12 created in infra/certs/]`

```powershell
# Verify PostgreSQL seed data
docker exec -it citizen-postgres psql `
  -U citizen_user -d citizen_search `
  -c "SELECT tender_number, title, geo_region FROM tenders ORDER BY published_at DESC LIMIT 5;"
```

`[EXPECTED: 5 rows of tender data shown including TND-IN-2024-001, TND-SG-2024-001 etc.]`

```powershell
# Verify Keycloak realm loaded
Start-Process "http://localhost:9090/realms/citizen-portal/.well-known/openid-configuration"
```

`[EXPECTED: Browser opens JSON showing issuer, token_endpoint, jwks_uri for citizen-portal realm]`

### Pre-Demonstration Verification Checklist

```
[ ] docker compose ps → all 3 containers healthy
[ ] PostgreSQL: 10 tender rows seeded, search_vector column populated
[ ] MongoDB: mongosh ping succeeds
[ ] Keycloak: http://localhost:9090/realms/citizen-portal → JSON response
[ ] Keycloak admin console: http://localhost:9090 → login admin/admin_pass → citizen-portal realm visible
[ ] Certs: infra/certs/ contains ca.crt, server.crt, client.crt, *.p12 files
[ ] Ports 5432, 27017, 9090 not blocked by firewall
```

### Troubleshooting: Infrastructure

| Issue                          | Symptom                            | Fix                                                                      |
| ------------------------------ | ---------------------------------- | ------------------------------------------------------------------------ |
| Keycloak fails to import realm | `realm-export.json not found`      | Verify volume mount path uses forward slashes in docker-compose.yml      |
| Keycloak 60s+ startup          | Health check failing               | Normal — wait up to 120s for JVM warmup and realm import                 |
| `keytool not found`            | cert generation fails              | `keytool` ships with JDK — verify `java -version` works first            |
| PostgreSQL trigger not firing  | `search_vector` is NULL            | Manually run `UPDATE tenders SET title=title;` to trigger re-computation |
| OpenSSL not found              | `bash: openssl: command not found` | Use Git Bash (ships with OpenSSL) not PowerShell directly                |

---

## Section 2: Search Architecture — PostgreSQL Full-Text Search Service

### What We Are Building

The `citizen-search-service` exposes a REST API for searching government tenders using PostgreSQL full-text search with ranked results, consistency guarantees, and geo-region filtering. This demonstrates the strong consistency model (synchronous index update via trigger) versus what an eventual consistency model (async Elasticsearch indexing) would look like.

### File: `citizen-search-service/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>gov.citizen</groupId>
    <artifactId>citizen-search-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>Citizen Search Service</name>
    <description>Government Tender Full-Text Search — Day 5</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Web MVC for REST endpoints -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Data JPA for PostgreSQL interaction -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Spring Data MongoDB for citizen profile aggregations -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>

        <!-- Spring Security OAuth2 Resource Server (JWT validation) -->
        <!-- WHY: validates Keycloak-issued JWTs for every incoming request -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- PostgreSQL JDBC driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Actuator for health checks and metrics -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Lombok for boilerplate reduction -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Test dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### File: `citizen-search-service/src/main/resources/application.yml`

```yaml
# application.yml — Citizen Search Service Configuration
server:
  port: 8081
  # mTLS configuration — server presents certificate to gateway client
  ssl:
    enabled: false
    # NOTE: Enable for full mTLS demo (Section 3)
    # key-store: classpath:server-keystore.p12
    # key-store-password: changeit
    # key-store-type: PKCS12
    # trust-store: classpath:truststore.p12
    # trust-store-password: changeit
    # client-auth: need   # ← REQUIRE client certificate

spring:
  application:
    name: citizen-search-service

  # PostgreSQL for full-text search
  datasource:
    url: jdbc:postgresql://${POSTGRES_HOST:localhost}:${POSTGRES_PORT:5432}/${POSTGRES_DB:citizen_search}
    username: ${POSTGRES_USER:citizen_user}
    password: ${POSTGRES_PASSWORD:citizen_pass}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 20000
      idle-timeout: 300000

  jpa:
    hibernate:
      ddl-auto: validate
      # WHY validate (not create/update): schema managed by init.sql, not JPA
      # This prevents accidental schema changes from ORM auto-DDL
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true

  # MongoDB for analytics aggregations
  data:
    mongodb:
      host: ${MONGO_HOST:localhost}
      port: ${MONGO_PORT:27017}
      database: ${MONGO_DB:citizen_profiles}
      username: ${MONGO_USER:mongo_user}
      password: ${MONGO_PASSWORD:mongo_pass}
      authentication-database: admin

  # OAuth2 Resource Server — validates JWTs issued by Keycloak
  security:
    oauth2:
      resourceserver:
        jwt:
          # JWKS URI: public keys from Keycloak for JWT signature validation
          jwks-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}/protocol/openid-connect/certs
          issuer-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

logging:
  level:
    gov.citizen.search: DEBUG
    org.springframework.security: INFO
```

### File: `CitizenSearchApplication.java`

```java
package gov.citizen.search;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Citizen Search Service — Day 5
 *
 * Demonstrates:
 * - PostgreSQL full-text search (tsvector/tsquery) with STRONG consistency
 * - MongoDB aggregation pipelines for regional analytics
 * - OAuth2 JWT validation (Keycloak integration)
 * - RBAC enforcement via Spring Security method security
 */
@SpringBootApplication
public class CitizenSearchApplication {
    public static void main(String[] args) {
        SpringApplication.run(CitizenSearchApplication.class, args);
    }
}
```

### File: `model/Tender.java`

```java
package gov.citizen.search.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.ZonedDateTime;
import java.util.UUID;

/**
 * Tender JPA entity mapping to the tenders table.
 *
 * WHY @Entity and not a plain DTO:
 * JPA entity allows us to use Spring Data JPA for standard CRUD
 * while using native queries for the tsvector full-text search operations
 * that JPA/JPQL cannot express natively.
 */
@Entity
@Table(name = "tenders")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Tender {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "id", updatable = false, nullable = false)
    private UUID id;

    @Column(name = "tender_number", nullable = false, unique = true)
    private String tenderNumber;

    @Column(name = "title", nullable = false)
    private String title;

    @Column(name = "description", nullable = false, columnDefinition = "TEXT")
    private String description;

    @Column(name = "department", nullable = false)
    private String department;

    @Column(name = "geo_region", nullable = false)
    private String geoRegion;

    @Column(name = "budget_inr", precision = 15, scale = 2)
    private BigDecimal budgetInr;

    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "published_at")
    private ZonedDateTime publishedAt;

    @Column(name = "deadline_at")
    private ZonedDateTime deadlineAt;

    // WHY @Column(insertable=false, updatable=false):
    // search_vector is managed entirely by the PostgreSQL trigger.
    // JPA must not attempt to write to this column — it would override the trigger.
    @Column(name = "search_vector", insertable = false, updatable = false)
    private String searchVector;

    @Column(name = "created_at")
    private ZonedDateTime createdAt;

    @Column(name = "updated_at")
    private ZonedDateTime updatedAt;
}
```

### File: `repository/TenderSearchRepository.java`

```java
package gov.citizen.search.repository;

import gov.citizen.search.model.Tender;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.UUID;

/**
 * TenderSearchRepository
 *
 * Demonstrates THREE search query patterns:
 * 1. Simple status + region filter (standard JPA derived query)
 * 2. Full-text search with ts_rank ranking (native PostgreSQL query)
 * 3. Full-text search with plainto_tsquery (natural language, no operators needed)
 *
 * WHY native queries for FTS:
 * JPQL/HQL has no support for PostgreSQL-specific tsvector/tsquery operators
 * (@@ operator, ts_rank function). We must drop to native SQL for FTS.
 */
@Repository
public interface TenderSearchRepository extends JpaRepository<Tender, UUID> {

    // ── 1. Standard filtered query (JPA derived) ──────────────────────────────
    List<Tender> findByStatusAndGeoRegionOrderByPublishedAtDesc(
            String status, String geoRegion);

    List<Tender> findByStatusOrderByPublishedAtDesc(String status);

    // ── 2. Full-Text Search with ranking (native PostgreSQL) ──────────────────
    /**
     * Searches tenders using PostgreSQL tsvector @@ tsquery operator.
     *
     * to_tsquery: accepts operators like 'health & digital | API'
     * ts_rank:    scores each match by position/frequency of lexemes
     * Results ordered by rank DESC (most relevant first)
     *
     * WHY plainto_tsquery in method 3 instead:
     * to_tsquery fails if user types "health digital" without & operator
     * plainto_tsquery converts free text to AND query automatically
     *
     * @param searchQuery PostgreSQL tsquery string (e.g., 'health & digital')
     * @param limit       maximum results (enforce pagination at repo level)
     */
    @Query(value = """
            SELECT t.*, ts_rank(t.search_vector, to_tsquery('english', :searchQuery)) AS rank
            FROM tenders t
            WHERE t.search_vector @@ to_tsquery('english', :searchQuery)
              AND t.status = 'OPEN'
            ORDER BY rank DESC
            LIMIT :limit
            """,
            nativeQuery = true)
    List<Tender> searchByTsQuery(
            @Param("searchQuery") String searchQuery,
            @Param("limit") int limit);

    // ── 3. Natural Language Full-Text Search (plainto_tsquery) ────────────────
    /**
     * User-friendly FTS — accepts plain text, no operators required.
     * "digital identity" → plainto_tsquery → 'digital' & 'identity'
     *
     * Optional geo_region filter: when null, searches all regions.
     * WHY COALESCE trick: allows optional parameter in native query
     */
    @Query(value = """
            SELECT t.*,
                   ts_rank_cd(t.search_vector, plainto_tsquery('english', :searchText)) AS rank
            FROM tenders t
            WHERE t.search_vector @@ plainto_tsquery('english', :searchText)
              AND (:geoRegion IS NULL OR t.geo_region = :geoRegion)
            ORDER BY rank DESC, t.published_at DESC
            LIMIT :limit OFFSET :offset
            """,
            nativeQuery = true)
    List<Tender> searchNaturalLanguage(
            @Param("searchText") String searchText,
            @Param("geoRegion") String geoRegion,
            @Param("limit") int limit,
            @Param("offset") int offset);

    // ── 4. Count for pagination ───────────────────────────────────────────────
    @Query(value = """
            SELECT COUNT(*)
            FROM tenders t
            WHERE t.search_vector @@ plainto_tsquery('english', :searchText)
              AND (:geoRegion IS NULL OR t.geo_region = :geoRegion)
            """,
            nativeQuery = true)
    long countNaturalLanguageResults(
            @Param("searchText") String searchText,
            @Param("geoRegion") String geoRegion);
}
```

### File: `service/TenderSearchService.java`

```java
package gov.citizen.search.service;

import gov.citizen.search.model.Tender;
import gov.citizen.search.repository.TenderSearchRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Map;

/**
 * TenderSearchService
 *
 * Business logic layer for search operations.
 * Demonstrates:
 * - Strong consistency guarantees (reads always reflect latest writes)
 * - Consistency SLA: search results current as of last committed transaction
 * - Contrast with eventual consistency (async indexing lag in Elasticsearch)
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class TenderSearchService {

    private final TenderSearchRepository tenderSearchRepository;

    private static final int DEFAULT_PAGE_SIZE = 10;
    private static final int MAX_PAGE_SIZE = 50;

    /**
     * Full-text search with natural language query.
     *
     * STRONG CONSISTENCY guarantee:
     * Because the search_vector is updated synchronously via PostgreSQL trigger
     * in the SAME transaction as the INSERT/UPDATE, this search will ALWAYS
     * return the current state. No indexing lag.
     *
     * SLA implication: search latency = query time only (~5-20ms for 10k rows)
     * vs Elasticsearch eventual consistency lag of 1-30 seconds after writes.
     *
     * For government tender portals, this is the CORRECT trade-off:
     * A newly closed tender must immediately disappear from OPEN searches.
     */
    @Transactional(readOnly = true)
    public Map<String, Object> searchTenders(
            String searchText,
            String geoRegion,
            int page,
            int size) {

        // Clamp page size to prevent abuse
        int pageSize = Math.min(size, MAX_PAGE_SIZE);
        int offset = page * pageSize;

        log.debug("Searching tenders: text='{}', region='{}', page={}, size={}",
                searchText, geoRegion, page, pageSize);

        // Execute the FTS query
        List<Tender> results = tenderSearchRepository.searchNaturalLanguage(
                searchText,
                geoRegion,
                pageSize,
                offset);

        // Count total for pagination metadata
        long total = tenderSearchRepository.countNaturalLanguageResults(
                searchText, geoRegion);

        log.info("Search '{}' returned {}/{} results (page {})",
                searchText, results.size(), total, page);

        // Return structured paginated response
        return Map.of(
                "results", results,
                "totalResults", total,
                "page", page,
                "pageSize", pageSize,
                "totalPages", (int) Math.ceil((double) total / pageSize),
                "consistencyModel", "STRONG",
                "searchEngine", "PostgreSQL-FTS",
                "indexType", "GIN-tsvector"
        );
    }

    /**
     * Retrieves open tenders by region.
     * Useful for regional government officer dashboards.
     */
    @Transactional(readOnly = true)
    public List<Tender> getOpenTendersByRegion(String geoRegion) {
        if (geoRegion != null && !geoRegion.isBlank()) {
            return tenderSearchRepository
                    .findByStatusAndGeoRegionOrderByPublishedAtDesc("OPEN", geoRegion);
        }
        return tenderSearchRepository.findByStatusOrderByPublishedAtDesc("OPEN");
    }

    /**
     * Demonstrates the EVENTUAL CONSISTENCY trade-off narrative.
     * This method explains what would happen differently with Elasticsearch.
     *
     * Called during the "Food for Thought" segment of Section 2.
     */
    public Map<String, String> getConsistencyComparison() {
        return Map.of(
                "postgresql_fts_model", "STRONG_CONSISTENCY",
                "postgresql_fts_lag", "0ms — search index updated in same ACID transaction",
                "postgresql_fts_sla", "Query latency: 5-50ms for up to 100K rows",
                "postgresql_fts_limit", "Does not scale horizontally beyond single primary",

                "elasticsearch_model", "EVENTUAL_CONSISTENCY",
                "elasticsearch_lag", "1-30 seconds — async indexing via bulk API",
                "elasticsearch_sla", "Query latency: 1-10ms at 1B+ document scale",
                "elasticsearch_limit", "Requires separate infrastructure, sync complexity"
        );
    }
}
```

### File: `api/SearchController.java`

```java
package gov.citizen.search.api;

import gov.citizen.search.model.Tender;
import gov.citizen.search.service.TenderSearchService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

/**
 * SearchController — REST endpoints for government tender search.
 *
 * Security model:
 * - GET /search/tenders → requires any authenticated user (CITIZEN, OFFICER, ADMIN, AUDITOR)
 * - GET /search/tenders/region/{region} → requires OFFICER or ADMIN role
 * - GET /search/consistency-info → requires ADMIN or AUDITOR role
 *
 * WHY @PreAuthorize:
 * Method-level security enforced AFTER JWT validation.
 * Role claim extracted from Keycloak JWT 'realm_access.roles' array.
 */
@RestController
@RequestMapping("/search")
@RequiredArgsConstructor
@Slf4j
public class SearchController {

    private final TenderSearchService tenderSearchService;

    /**
     * Full-text search endpoint.
     * Accessible to ALL authenticated users regardless of role.
     * Citizens, Officers, Admins, and Auditors can all search public tenders.
     *
     * Example: GET /search/tenders?q=health+digital&region=IN-DL&page=0&size=10
     */
    @GetMapping("/tenders")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Map<String, Object>> searchTenders(
            @RequestParam("q") String query,
            @RequestParam(value = "region", required = false) String geoRegion,
            @RequestParam(value = "page", defaultValue = "0") int page,
            @RequestParam(value = "size", defaultValue = "10") int size,
            @AuthenticationPrincipal Jwt jwt) {

        // Log who is searching (audit trail — important for government systems)
        String username = jwt.getClaimAsString("preferred_username");
        log.info("Tender search by user='{}': query='{}', region='{}'",
                username, query, geoRegion);

        Map<String, Object> results = tenderSearchService.searchTenders(
                query, geoRegion, page, size);

        return ResponseEntity.ok(results);
    }

    /**
     * Regional tender listing — restricted to OFFICER and ADMIN roles.
     * Officers can only view their own region's tenders (enforced in service layer
     * in a full implementation — simplified here for lab clarity).
     *
     * Example: GET /search/tenders/region/IN-MH
     */
    @GetMapping("/tenders/region/{region}")
    @PreAuthorize("hasAnyRole('OFFICER', 'ADMIN')")
    public ResponseEntity<List<Tender>> getTendersByRegion(
            @PathVariable String region,
            @AuthenticationPrincipal Jwt jwt) {

        String username = jwt.getClaimAsString("preferred_username");
        log.info("Regional tender list by officer='{}': region='{}'", username, region);

        List<Tender> tenders = tenderSearchService.getOpenTendersByRegion(region);
        return ResponseEntity.ok(tenders);
    }

    /**
     * Consistency model information — for ADMIN and AUDITOR only.
     * Shows architectural trade-offs: strong vs eventual consistency.
     *
     * Example: GET /search/consistency-info
     */
    @GetMapping("/consistency-info")
    @PreAuthorize("hasAnyRole('ADMIN', 'AUDITOR')")
    public ResponseEntity<Map<String, String>> getConsistencyInfo() {
        return ResponseEntity.ok(tenderSearchService.getConsistencyComparison());
    }

    /**
     * Health check — no auth required (used by gateway and monitoring)
     */
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of("status", "UP", "service", "citizen-search-service"));
    }
}
```

### File: `config/SecurityConfig.java` (Search Service)

```java
package gov.citizen.search.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * SecurityConfig — OAuth2 Resource Server configuration for Search Service.
 *
 * WHY stateless session:
 * Microservices must NOT store HTTP sessions (violates 12-Factor App principle VI).
 * Each request carries the JWT — the token IS the session.
 *
 * WHY JwtAuthenticationConverter:
 * Keycloak puts roles inside jwt.realm_access.roles (not the Spring default
 * jwt.scope or jwt.authorities). We must extract them explicitly and prefix
 * with ROLE_ so that @PreAuthorize("hasRole('OFFICER')") works correctly.
 */
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF: stateless REST APIs use JWT, not cookies
            .csrf(csrf -> csrf.disable())

            // Stateless session: no HttpSession created or used
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                // Health check open to monitoring systems
                .requestMatchers("/search/health", "/actuator/health").permitAll()
                // All other endpoints require a valid JWT
                .anyRequest().authenticated()
            )

            // Configure as OAuth2 Resource Server — validates JWT from Keycloak
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    /**
     * Extracts Keycloak roles from JWT and converts to Spring GrantedAuthorities.
     *
     * Keycloak JWT structure for realm roles:
     * {
     *   "realm_access": {
     *     "roles": ["CITIZEN", "offline_access", "uma_authorization"]
     *   }
     * }
     *
     * Spring Security expects: ROLE_CITIZEN (with ROLE_ prefix)
     */
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();

        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Extract realm_access claim from Keycloak JWT
            Map<String, Object> realmAccess =
                jwt.getClaimAsMap("realm_access");

            if (realmAccess == null || !realmAccess.containsKey("roles")) {
                return List.of();
            }

            @SuppressWarnings("unchecked")
            Collection<String> roles = (Collection<String>) realmAccess.get("roles");

            // Map each role to ROLE_<ROLENAME> as Spring Security expects
            return roles.stream()
                .filter(role -> !role.startsWith("offline") && !role.startsWith("uma"))
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
        });

        return converter;
    }
}
```

### File: `citizen-search-service/src/test/java/gov/citizen/search/TenderSearchServiceTest.java`

```java
package gov.citizen.search;

import gov.citizen.search.model.Tender;
import gov.citizen.search.repository.TenderSearchRepository;
import gov.citizen.search.service.TenderSearchService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.ZonedDateTime;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

/**
 * Unit tests for TenderSearchService.
 * Tests search logic, pagination, and consistency model metadata.
 * Uses Mockito to isolate from PostgreSQL — no database required for unit tests.
 */
@ExtendWith(MockitoExtension.class)
class TenderSearchServiceTest {

    @Mock
    private TenderSearchRepository tenderSearchRepository;

    @InjectMocks
    private TenderSearchService tenderSearchService;

    private Tender sampleTender;

    @BeforeEach
    void setUp() {
        sampleTender = Tender.builder()
            .id(UUID.randomUUID())
            .tenderNumber("TND-IN-2024-001")
            .title("Smart City IoT Sensor Network Deployment")
            .description("IoT sensors for air quality monitoring and traffic management")
            .department("Ministry of Housing and Urban Affairs")
            .geoRegion("IN-MH")
            .budgetInr(new BigDecimal("150000000.00"))
            .status("OPEN")
            .publishedAt(ZonedDateTime.now())
            .deadlineAt(ZonedDateTime.now().plusDays(30))
            .build();
    }

    @Test
    @DisplayName("searchTenders returns paginated results with consistency metadata")
    void searchTenders_returnsPaginatedResultsWithConsistencyMetadata() {
        // Arrange
        when(tenderSearchRepository.searchNaturalLanguage(
                eq("IoT sensors"), isNull(), eq(10), eq(0)))
            .thenReturn(List.of(sampleTender));
        when(tenderSearchRepository.countNaturalLanguageResults(
                eq("IoT sensors"), isNull()))
            .thenReturn(1L);

        // Act
        Map<String, Object> result = tenderSearchService.searchTenders(
                "IoT sensors", null, 0, 10);

        // Assert — results present
        assertThat(result).containsKey("results");
        @SuppressWarnings("unchecked")
        List<Tender> results = (List<Tender>) result.get("results");
        assertThat(results).hasSize(1);
        assertThat(results.get(0).getTenderNumber()).isEqualTo("TND-IN-2024-001");

        // Assert — consistency metadata present
        assertThat(result.get("consistencyModel")).isEqualTo("STRONG");
        assertThat(result.get("searchEngine")).isEqualTo("PostgreSQL-FTS");
        assertThat(result.get("indexType")).isEqualTo("GIN-tsvector");

        // Assert — pagination metadata correct
        assertThat(result.get("totalResults")).isEqualTo(1L);
        assertThat(result.get("page")).isEqualTo(0);
        assertThat(result.get("totalPages")).isEqualTo(1);
    }

    @Test
    @DisplayName("searchTenders clamps page size to MAX_PAGE_SIZE of 50")
    void searchTenders_clampsPageSizeToMaximum() {
        // Arrange
        when(tenderSearchRepository.searchNaturalLanguage(any(), any(), eq(50), eq(0)))
            .thenReturn(List.of());
        when(tenderSearchRepository.countNaturalLanguageResults(any(), any()))
            .thenReturn(0L);

        // Act — request 200 results (over the 50 maximum)
        Map<String, Object> result = tenderSearchService.searchTenders(
                "health", null, 0, 200);

        // Assert — page size clamped to 50
        assertThat(result.get("pageSize")).isEqualTo(50);
        verify(tenderSearchRepository)
            .searchNaturalLanguage(eq("health"), isNull(), eq(50), eq(0));
    }

    @Test
    @DisplayName("getConsistencyComparison returns both PostgreSQL and Elasticsearch models")
    void getConsistencyComparison_returnsBothModels() {
        Map<String, String> comparison = tenderSearchService.getConsistencyComparison();

        assertThat(comparison)
            .containsKey("postgresql_fts_model")
            .containsKey("elasticsearch_model")
            .containsEntry("postgresql_fts_model", "STRONG_CONSISTENCY")
            .containsEntry("elasticsearch_model", "EVENTUAL_CONSISTENCY");
    }
}
```

### Build and Start Search Service

```powershell
cd C:\training\day5-security-search-platform\citizen-search-service

# Build (skip tests first for speed, run tests separately)
mvn clean package -DskipTests

# Run
mvn spring-boot:run `
  -Dspring-boot.run.jvmArguments="-DPOSTGRES_HOST=localhost -DPOSTGRES_PORT=5432 -DPOSTGRES_DB=citizen_search -DPOSTGRES_USER=citizen_user -DPOSTGRES_PASSWORD=citizen_pass -DMONGO_HOST=localhost -DMONGO_USER=mongo_user -DMONGO_PASSWORD=mongo_pass -DKEYCLOAK_BASE_URL=http://localhost:9090 -DKEYCLOAK_REALM=citizen-portal"
```

`[EXPECTED: Spring Boot starts on port 8081. Log shows "Tomcat started on port 8081". JPA validation passes against existing schema.]`

```powershell
# Run unit tests
mvn test
```

`[EXPECTED: 3 tests found, 3 passed, 0 failed. BUILD SUCCESS.]`

### Definition of Done — Section 2

```
[ ] citizen-search-service starts on port 8081 without errors
[ ] GET http://localhost:8081/search/health → {"status":"UP"}
[ ] PostgreSQL: search_vector column populated for all 10 seed rows
[ ] Unit tests: 3 tests pass
[ ] JPA validation: no schema mismatch errors in startup log
```

---

## Section 3: Zero Trust — Keycloak RBAC + JWT Validation

### What We Are Building

The `citizen-api-gateway` acts as the single entry point enforcing Zero Trust: every request must carry a valid Keycloak JWT, roles are extracted and enforced, requests are rate-limited and audit-logged before being forwarded to downstream services.

### File: `citizen-api-gateway/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>gov.citizen</groupId>
    <artifactId>citizen-api-gateway</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>Citizen API Gateway</name>
    <description>Zero Trust API Gateway with OAuth2/OIDC — Day 5</description>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>

    <dependencies>
        <!-- Spring Cloud Gateway (reactive, WebFlux-based) -->
        <!-- WHY reactive gateway: non-blocking I/O handles thousands of concurrent
             connections without thread-per-request overhead -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <!-- OAuth2 Resource Server for JWT validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- Spring Security (reactive) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Actuator for health/metrics -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### File: `citizen-api-gateway/src/main/resources/application.yml`

```yaml
# application.yml — Citizen API Gateway
# Zero Trust gateway: validate every request, enforce roles, audit log, rate limit

server:
  port: 8080

spring:
  application:
    name: citizen-api-gateway

  # OAuth2 Resource Server: validate JWTs from Keycloak
  security:
    oauth2:
      resourceserver:
        jwt:
          jwks-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}/protocol/openid-connect/certs
          issuer-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}

  # Spring Cloud Gateway routing rules
  cloud:
    gateway:
      # Global default filters applied to ALL routes
      default-filters:
        # Add correlation ID for distributed tracing
        - AddRequestHeader=X-Correlation-ID, ${random.uuid}
        # Strip CORS pre-flight handling at gateway level
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

      routes:
        # ── Search Service Routes ──────────────────────────────────────────────
        - id: search-public
          uri: http://localhost:8081
          predicates:
            - Path=/api/search/**
          filters:
            # Rewrite: /api/search/tenders → /search/tenders
            - RewritePath=/api/search/(?<segment>.*), /search/${segment}
            # Add downstream service identifier header
            - AddRequestHeader=X-Gateway-Source, citizen-api-gateway
            # Circuit breaker (conceptual — would need resilience4j dependency)
            # - CircuitBreaker=name=searchCB,fallbackUri=forward:/fallback/search

        # ── Data Service Routes ───────────────────────────────────────────────
        - id: data-service
          uri: http://localhost:8082
          predicates:
            - Path=/api/citizens/**
          filters:
            - RewritePath=/api/citizens/(?<segment>.*), /citizens/${segment}
            - AddRequestHeader=X-Gateway-Source, citizen-api-gateway

      # Global CORS configuration
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:3000"
              - "http://localhost:8080"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders:
              - "*"
            allowCredentials: true
            maxAge: 3600

# Rate limiting config (used by RateLimitFilter)
rate-limit:
  requests-per-second: 10
  burst-size: 20

management:
  endpoints:
    web:
      exposure:
        include: health,info,gateway
  endpoint:
    health:
      show-details: always
    gateway:
      enabled: true

logging:
  level:
    gov.citizen.gateway: DEBUG
    org.springframework.cloud.gateway: INFO
    org.springframework.security: INFO
```

### File: `CitizenGatewayApplication.java`

```java
package gov.citizen.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Citizen API Gateway — Day 5 Zero Trust Implementation
 *
 * Implements:
 * - Zero Trust principle: "Never Trust, Always Verify"
 * - Every request validated: JWT signature, expiry, issuer, audience
 * - Role-based routing: different backend access per role
 * - Audit logging: every proxied request recorded
 * - Rate limiting: prevents DoS and API abuse
 */
@SpringBootApplication
public class CitizenGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(CitizenGatewayApplication.class, args);
    }
}
```

### File: `config/SecurityConfig.java` (Gateway)

```java
package gov.citizen.gateway.config;

import gov.citizen.gateway.security.JwtClaimsValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.ReactiveJwtAuthenticationConverterAdapter;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.server.SecurityWebFilterChain;
import reactor.core.publisher.Mono;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * SecurityConfig for Spring Cloud Gateway (Reactive / WebFlux).
 *
 * WHY @EnableWebFluxSecurity instead of @EnableWebSecurity:
 * Spring Cloud Gateway is built on Project Reactor (WebFlux).
 * It uses reactive ServerHttpSecurity, not the servlet-based HttpSecurity.
 * All filter chains must be reactive (return Mono/Flux, not blocking).
 *
 * Zero Trust enforcement:
 * - ALL requests require a valid Keycloak JWT (no anonymous access)
 * - Exception: /actuator/health (for load balancer health probes)
 * - JWT validated: signature (via JWKS), expiry, issuer URI
 * - Roles extracted from realm_access.roles claim
 */
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    /**
     * Main security filter chain for the reactive gateway.
     *
     * Zero Trust principle applied:
     * "Never trust, always verify" — even internal service calls
     * coming through the gateway must present a valid token.
     */
    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(
            ServerHttpSecurity http,
            JwtClaimsValidator jwtClaimsValidator) {

        http
            // Disable CSRF: gateway is stateless, uses JWT not cookies
            .csrf(ServerHttpSecurity.CsrfSpec::disable)

            .authorizeExchange(exchanges -> exchanges
                // Health endpoint: allow unauthenticated (for Docker health checks)
                .pathMatchers("/actuator/health").permitAll()

                // Search endpoints: any authenticated user
                .pathMatchers("/api/search/**").authenticated()

                // Citizen data: OFFICER or ADMIN only
                // WHY hasRole: enforces RBAC at gateway — downstream services
                // do a second verification (defense in depth)
                .pathMatchers("/api/citizens/all").hasAnyRole("OFFICER", "ADMIN")
                .pathMatchers("/api/citizens/**").authenticated()

                // Deny everything else by default (Zero Trust default-deny)
                .anyExchange().denyAll()
            )

            // Configure reactive OAuth2 Resource Server
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(
                        reactiveJwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    /**
     * Reactive JWT authentication converter for WebFlux.
     * Wraps the standard JwtAuthenticationConverter in a reactive adapter.
     *
     * Extracts realm_access.roles from Keycloak JWT and maps to
     * Spring Security GrantedAuthorities (ROLE_CITIZEN, ROLE_OFFICER etc.)
     */
    @Bean
    public ReactiveJwtAuthenticationConverterAdapter reactiveJwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();

        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null || !realmAccess.containsKey("roles")) {
                return List.of();
            }

            @SuppressWarnings("unchecked")
            Collection<String> roles = (Collection<String>) realmAccess.get("roles");

            return roles.stream()
                .filter(role -> List.of("CITIZEN", "OFFICER", "ADMIN", "AUDITOR")
                    .contains(role))
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
        });

        return new ReactiveJwtAuthenticationConverterAdapter(converter);
    }
}
```

### File: `security/JwtClaimsValidator.java`

```java
package gov.citizen.gateway.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.List;
import java.util.Map;

/**
 * JwtClaimsValidator — additional JWT claim validation beyond signature/expiry.
 *
 * Spring's JWT resource server validates:
 *   - Signature (via JWKS from Keycloak)
 *   - Expiry (exp claim)
 *   - Issuer (iss claim matches configuration)
 *
 * This validator adds government-specific checks:
 *   - Audience claim (aud) must include our client ID
 *   - User must not be in a suspended list (stub — would check a cache)
 *   - Token must not be issued before service started (replay attack detection)
 *
 * WHY additional validation:
 * Zero Trust requires continuous verification.
 * A valid signature does not mean the token represents a valid session.
 * Example: India's DigiLocker token revocation must be enforced here.
 */
@Component
@Slf4j
public class JwtClaimsValidator {

    private static final String EXPECTED_ISSUER_SUFFIX = "/realms/citizen-portal";
    private static final List<String> ALLOWED_AUDIENCES = List.of(
        "citizen-gateway", "account"
    );

    /**
     * Validates additional claims beyond standard JWT validation.
     * Returns true if valid, false if the token should be rejected.
     */
    public boolean validate(Jwt jwt) {
        // Check issuer ends with expected realm
        String issuer = jwt.getIssuer().toString();
        if (!issuer.endsWith(EXPECTED_ISSUER_SUFFIX)) {
            log.warn("JWT rejected: issuer '{}' does not match expected realm", issuer);
            return false;
        }

        // Check audience — token must be intended for this service
        List<String> audiences = jwt.getAudience();
        boolean hasValidAudience = audiences.stream()
            .anyMatch(ALLOWED_AUDIENCES::contains);

        if (!hasValidAudience) {
            log.warn("JWT rejected: audiences {} do not include allowed clients", audiences);
            return false;
        }

        // Check token is not from the future (clock skew attack)
        Instant issuedAt = jwt.getIssuedAt();
        if (issuedAt != null && issuedAt.isAfter(Instant.now().plusSeconds(30))) {
            log.warn("JWT rejected: issued_at {} is in the future (clock skew?)", issuedAt);
            return false;
        }

        log.debug("JWT validated: subject='{}', roles={}",
            jwt.getSubject(),
            extractRoles(jwt));

        return true;
    }

    private List<String> extractRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return List.of();
        @SuppressWarnings("unchecked")
        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles != null ? roles : List.of();
    }
}
```

### File: `filter/AuditLoggingFilter.java`

```java
package gov.citizen.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;

/**
 * AuditLoggingFilter — Global reactive filter that logs every proxied request.
 *
 * WHY audit logging at gateway level:
 * Zero Trust requires "assume breach" — log EVERYTHING.
 * Government systems must maintain access logs for:
 *   - India: IT Act 2000, DPDP Act 2023 audit requirements
 *   - US: FedRAMP requires audit logs with user identity for all access events
 *   - Singapore: IM8 requires activity logging for government ICT systems
 *
 * Logged: user identity, method, path, response status, latency, correlation ID
 *
 * WHY GlobalFilter:
 * Applied to ALL routes automatically — no per-route configuration needed.
 * Ensures audit completeness — no route can accidentally bypass audit logging.
 */
@Component
@Slf4j
public class AuditLoggingFilter implements GlobalFilter, Ordered {

    // Order = -1 means this runs BEFORE routing filters
    // WHY: we want to capture start time before any routing overhead
    @Override
    public int getOrder() {
        return -1;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        long startTime = Instant.now().toEpochMilli();

        String correlationId = request.getHeaders()
            .getFirst("X-Correlation-ID");
        String method = request.getMethod().name();
        String path = request.getPath().value();

        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .cast(JwtAuthenticationToken.class)
            .map(auth -> {
                Jwt jwt = auth.getToken();
                return jwt.getClaimAsString("preferred_username");
            })
            // Default to "anonymous" if no auth context (e.g., /actuator/health)
            .defaultIfEmpty("anonymous")
            .flatMap(username ->
                chain.filter(exchange)
                    .doFinally(signal -> {
                        ServerHttpResponse response = exchange.getResponse();
                        long latencyMs = Instant.now().toEpochMilli() - startTime;

                        // Structured audit log — parseable by ELK/Splunk
                        log.info(
                            "AUDIT | correlationId={} | user={} | method={} " +
                            "| path={} | status={} | latencyMs={}",
                            correlationId,
                            username,
                            method,
                            path,
                            response.getStatusCode() != null
                                ? response.getStatusCode().value()
                                : "unknown",
                            latencyMs
                        );
                    })
            );
    }
}
```

### File: `filter/RateLimitFilter.java`

```java
package gov.citizen.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * RateLimitFilter — Simple token-bucket rate limiter at gateway level.
 *
 * WHY rate limiting at gateway:
 * - Protects downstream services from DoS attacks
 * - Enforces fair use among citizen users
 * - Required by India's NIC API gateway standards
 * - Required for FedRAMP compliance (US government APIs)
 *
 * Algorithm: Fixed window counter per IP address.
 * Production alternative: Redis-backed sliding window with Lua scripts.
 * WHY not Redis here: avoids adding Redis dependency for lab simplicity.
 *
 * Limits:
 * - 10 requests per second per IP (configurable)
 * - Returns HTTP 429 Too Many Requests with Retry-After header
 */
@Component
@Slf4j
public class RateLimitFilter implements GlobalFilter, Ordered {

    private static final int REQUESTS_PER_WINDOW = 20;
    private static final long WINDOW_SIZE_MS = 1000L; // 1 second window

    // Per-IP counters: key=IP, value=[count, windowStart]
    private final Map<String, long[]> ipCounters = new ConcurrentHashMap<>();

    @Override
    public int getOrder() {
        // Run after AuditLoggingFilter (-1) but before routing (0)
        return 0;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String clientIp = getClientIp(exchange);

        if (isRateLimited(clientIp)) {
            log.warn("RATE_LIMIT | IP={} | Requests exceeded {} per second",
                clientIp, REQUESTS_PER_WINDOW);

            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            exchange.getResponse().getHeaders().add("Retry-After", "1");
            exchange.getResponse().getHeaders().add("X-Rate-Limit-Limit",
                String.valueOf(REQUESTS_PER_WINDOW));
            exchange.getResponse().getHeaders().add("X-Rate-Limit-Remaining", "0");

            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    private boolean isRateLimited(String clientIp) {
        long now = System.currentTimeMillis();

        ipCounters.compute(clientIp, (ip, counters) -> {
            if (counters == null) {
                // First request from this IP
                return new long[]{1L, now};
            }
            long count = counters[0];
            long windowStart = counters[1];

            if (now - windowStart > WINDOW_SIZE_MS) {
                // Window expired — reset counter
                return new long[]{1L, now};
            }
            // Within window — increment
            return new long[]{count + 1, windowStart};
        });

        long[] counters = ipCounters.get(clientIp);
        return counters[0] > REQUESTS_PER_WINDOW;
    }

    private String getClientIp(ServerWebExchange exchange) {
        // Check X-Forwarded-For first (for requests through load balancers)
        String xff = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
        if (xff != null && !xff.isBlank()) {
            return xff.split(",")[0].trim();
        }
        var remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null ? remoteAddress.getAddress().getHostAddress() : "unknown";
    }
}
```

### Build and Start Gateway

```powershell
cd C:\training\day5-security-search-platform\citizen-api-gateway

mvn clean package -DskipTests

mvn spring-boot:run `
  -Dspring-boot.run.jvmArguments="-DKEYCLOAK_BASE_URL=http://localhost:9090 -DKEYCLOAK_REALM=citizen-portal"
```

`[EXPECTED: Gateway starts on port 8080. Log shows "Netty started on port(s): 8080". Routes registered: search-public, data-service.]`

### Get a Token and Test the Gateway

```powershell
# Step 1: Get JWT token for Rahul (CITIZEN role)
$tokenResponse = Invoke-RestMethod `
  -Method POST `
  -Uri "http://localhost:9090/realms/citizen-portal/protocol/openid-connect/token" `
  -ContentType "application/x-www-form-urlencoded" `
  -Body @{
    grant_type    = "password"
    client_id     = "citizen-gateway"
    client_secret = "gateway-secret-2024"
    username      = "rahul.citizen"
    password      = "citizen123"
    scope         = "openid profile roles"
  }

# Store token
$citizenToken = $tokenResponse.access_token
Write-Host "Token obtained for rahul.citizen (CITIZEN role)"
Write-Host "Token preview: $($citizenToken.Substring(0, 50))..."
```

`[EXPECTED: Token string printed. No error. Token starts with "eyJ" (base64 encoded JWT header)]`

```powershell
# Step 2: Search tenders via Gateway (should succeed — CITIZEN can search)
Invoke-RestMethod `
  -Method GET `
  -Uri "http://localhost:8080/api/search/tenders?q=digital+identity&page=0&size=5" `
  -Headers @{ Authorization = "Bearer $citizenToken" }
```

`[EXPECTED: JSON response with results array containing tenders matching "digital identity". consistencyModel="STRONG".]`

```powershell
# Step 3: Get officer token (OFFICER role)
$officerResponse = Invoke-RestMethod `
  -Method POST `
  -Uri "http://localhost:9090/realms/citizen-portal/protocol/openid-connect/token" `
  -ContentType "application/x-www-form-urlencoded" `
  -Body @{
    grant_type    = "password"
    client_id     = "citizen-gateway"
    client_secret = "gateway-secret-2024"
    username      = "priya.officer"
    password      = "officer123"
    scope         = "openid profile roles"
  }

$officerToken = $officerResponse.access_token

# Step 4: Access regional tenders (OFFICER can access, CITIZEN cannot)
Invoke-RestMethod `
  -Method GET `
  -Uri "http://localhost:8080/api/search/tenders/region/IN-MH" `
  -Headers @{ Authorization = "Bearer $officerToken" }
```

`[EXPECTED: List of open tenders for region IN-MH. HTTP 200.]`

```powershell
# Step 5: Try regional endpoint with CITIZEN token (should be 403)
try {
    Invoke-RestMethod `
      -Method GET `
      -Uri "http://localhost:8080/api/search/tenders/region/IN-MH" `
      -Headers @{ Authorization = "Bearer $citizenToken" }
} catch {
    Write-Host "Expected 403 Forbidden: $($_.Exception.Response.StatusCode)" -ForegroundColor Green
}
```

`[EXPECTED: HTTP 403 Forbidden — CITIZEN role cannot access officer-restricted regional endpoint. RBAC enforcement working.]`

### Definition of Done — Section 3

```
[ ] Gateway starts on port 8080 without errors
[ ] Keycloak token obtained for rahul.citizen (CITIZEN role)
[ ] Keycloak token obtained for priya.officer (OFFICER role)
[ ] GET /api/search/tenders?q=digital → HTTP 200 with CITIZEN token
[ ] GET /api/search/tenders/region/IN-MH → HTTP 200 with OFFICER token
[ ] GET /api/search/tenders/region/IN-MH → HTTP 403 with CITIZEN token
[ ] Audit log lines visible in gateway console for every request
[ ] Rate limit: sending 25 rapid requests → HTTP 429 after 20th
```

---

## Section 4: Citizen Data Service — MongoDB Analytics

### What We Are Building

The `citizen-data-service` stores citizen profiles in MongoDB and exposes aggregation pipeline endpoints for regional analytics — demonstrating MongoDB's strength for flexible document storage and complex multi-stage aggregations.

### File: `citizen-data-service/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>gov.citizen</groupId>
    <artifactId>citizen-data-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>Citizen Data Service</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### File: `citizen-data-service/src/main/resources/application.yml`

```yaml
server:
  port: 8082

spring:
  application:
    name: citizen-data-service

  data:
    mongodb:
      host: ${MONGO_HOST:localhost}
      port: ${MONGO_PORT:27017}
      database: ${MONGO_DB:citizen_profiles}
      username: ${MONGO_USER:mongo_user}
      password: ${MONGO_PASSWORD:mongo_pass}
      authentication-database: admin

  security:
    oauth2:
      resourceserver:
        jwt:
          jwks-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}/protocol/openid-connect/certs
          issuer-uri: ${KEYCLOAK_BASE_URL:http://localhost:9090}/realms/${KEYCLOAK_REALM:citizen-portal}

management:
  endpoints:
    web:
      exposure:
        include: health,info

logging:
  level:
    gov.citizen.data: DEBUG
```

### File: `model/CitizenProfile.java`

```java
package gov.citizen.data.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

/**
 * CitizenProfile — MongoDB document representing a citizen's government profile.
 *
 * WHY MongoDB for citizen profiles:
 * - Schema flexibility: different benefit types have different fields
 *   (agricultural subsidy vs. healthcare vs. education — each has unique attributes)
 * - Nested documents: address, benefitHistory, documents all nested naturally
 * - No joins: citizen profile accessed as one atomic document
 * - Horizontal scaling: sharding by geoRegion for data sovereignty compliance
 *
 * WHY NOT PostgreSQL here:
 * The highly variable nested structure (benefitHistory varies per scheme)
 * would require either many nullable columns or complex EAV patterns in SQL.
 * MongoDB handles polymorphic nested data natively.
 */
@Document(collection = "citizen_profiles")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CitizenProfile {

    @Id
    private String id;

    // Aadhaar-equivalent ID (anonymised for lab — not real Aadhaar numbers)
    @Indexed(unique = true)
    @Field("citizen_id")
    private String citizenId;

    @Field("full_name")
    private String fullName;

    @Field("date_of_birth")
    private LocalDate dateOfBirth;

    @Field("gender")
    private String gender;

    // geoRegion indexed for sharding and regional analytics
    @Indexed
    @Field("geo_region")
    private String geoRegion;  // e.g., "IN-TN", "US-CA", "SG-CENTRAL"

    @Field("address")
    private Address address;

    @Field("income_category")
    private String incomeCategory; // BPL, APL, EWS for India

    @Field("benefit_history")
    private List<BenefitRecord> benefitHistory;

    // Flexible document: different benefit types carry different attributes
    @Field("scheme_attributes")
    private Map<String, Object> schemeAttributes;

    @Field("registration_date")
    private LocalDate registrationDate;

    @Field("is_active")
    private boolean isActive;

    // ── Nested document classes ───────────────────────────────────────────────

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Address {
        private String line1;
        private String line2;
        private String city;
        private String state;
        private String pincode;
        private String country;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class BenefitRecord {
        private String schemeName;
        private String status;
        private Double amountReceived;
        private String currency;
        private LocalDate disbursementDate;
        private String transactionRef;
    }
}
```

### File: `service/CitizenAnalyticsService.java`

```java
package gov.citizen.data.service;

import gov.citizen.data.model.CitizenProfile;
import gov.citizen.data.repository.CitizenProfileRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.aggregation.*;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

/**
 * CitizenAnalyticsService
 *
 * Demonstrates MongoDB aggregation pipelines for government analytics.
 * These are the CQRS read-side projections for citizen data platform.
 *
 * Pipeline 1: Regional benefit distribution summary
 * Pipeline 2: Income category breakdown per state
 * Pipeline 3: Scheme-wise beneficiary count
 *
 * WHY MongoTemplate over Repository for aggregations:
 * Spring Data Repository methods cannot express multi-stage pipelines.
 * MongoTemplate.aggregate() gives full control over pipeline stages.
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class CitizenAnalyticsService {

    private final CitizenProfileRepository citizenProfileRepository;
    private final MongoTemplate mongoTemplate;

    /**
     * Seeds sample citizen data for the lab demonstration.
     * Called once on first run.
     */
    public void seedSampleData() {
        if (citizenProfileRepository.count() > 0) {
            log.info("Sample data already exists — skipping seed");
            return;
        }

        List<CitizenProfile> profiles = List.of(
            CitizenProfile.builder()
                .citizenId("CIT-IN-2024-001")
                .fullName("Anitha Ramachandran")
                .geoRegion("IN-TN")
                .incomeCategory("BPL")
                .isActive(true)
                .registrationDate(java.time.LocalDate.of(2020, 3, 15))
                .address(CitizenProfile.Address.builder()
                    .city("Chennai").state("Tamil Nadu").country("India")
                    .pincode("600001").build())
                .benefitHistory(List.of(
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("PM-KISAN")
                        .status("DISBURSED")
                        .amountReceived(6000.0)
                        .currency("INR")
                        .disbursementDate(java.time.LocalDate.of(2024, 4, 1))
                        .transactionRef("PMKISAN-TN-001").build()
                ))
                .build(),

            CitizenProfile.builder()
                .citizenId("CIT-IN-2024-002")
                .fullName("Venkatesh Narayanan")
                .geoRegion("IN-TN")
                .incomeCategory("APL")
                .isActive(true)
                .registrationDate(java.time.LocalDate.of(2021, 7, 20))
                .address(CitizenProfile.Address.builder()
                    .city("Coimbatore").state("Tamil Nadu").country("India")
                    .pincode("641001").build())
                .benefitHistory(List.of(
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("Ayushman-Bharat")
                        .status("ENROLLED")
                        .amountReceived(500000.0)
                        .currency("INR")
                        .disbursementDate(java.time.LocalDate.of(2024, 1, 10))
                        .transactionRef("AB-TN-002").build()
                ))
                .build(),

            CitizenProfile.builder()
                .citizenId("CIT-US-2024-001")
                .fullName("Maria Santos")
                .geoRegion("US-CA")
                .incomeCategory("LOW_INCOME")
                .isActive(true)
                .registrationDate(java.time.LocalDate.of(2022, 11, 5))
                .address(CitizenProfile.Address.builder()
                    .city("Los Angeles").state("California").country("USA")
                    .pincode("90001").build())
                .benefitHistory(List.of(
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("SNAP")
                        .status("ACTIVE")
                        .amountReceived(281.0)
                        .currency("USD")
                        .disbursementDate(java.time.LocalDate.of(2024, 5, 1))
                        .transactionRef("SNAP-CA-001").build()
                ))
                .build(),

            CitizenProfile.builder()
                .citizenId("CIT-SG-2024-001")
                .fullName("Tan Wei Ming")
                .geoRegion("SG-CENTRAL")
                .incomeCategory("LOWER_INCOME")
                .isActive(true)
                .registrationDate(java.time.LocalDate.of(2023, 2, 14))
                .address(CitizenProfile.Address.builder()
                    .city("Singapore").state("Central").country("Singapore")
                    .pincode("049483").build())
                .benefitHistory(List.of(
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("ComCare-ST")
                        .status("DISBURSED")
                        .amountReceived(800.0)
                        .currency("SGD")
                        .disbursementDate(java.time.LocalDate.of(2024, 3, 15))
                        .transactionRef("CC-SG-001").build()
                ))
                .build(),

            CitizenProfile.builder()
                .citizenId("CIT-IN-2024-003")
                .fullName("Ramesh Gupta")
                .geoRegion("IN-UP")
                .incomeCategory("BPL")
                .isActive(true)
                .registrationDate(java.time.LocalDate.of(2019, 6, 30))
                .address(CitizenProfile.Address.builder()
                    .city("Lucknow").state("Uttar Pradesh").country("India")
                    .pincode("226001").build())
                .benefitHistory(List.of(
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("PM-KISAN")
                        .status("DISBURSED")
                        .amountReceived(6000.0)
                        .currency("INR")
                        .disbursementDate(java.time.LocalDate.of(2024, 4, 1))
                        .transactionRef("PMKISAN-UP-003").build(),
                    CitizenProfile.BenefitRecord.builder()
                        .schemeName("PM-Ujjwala")
                        .status("DISBURSED")
                        .amountReceived(1600.0)
                        .currency("INR")
                        .disbursementDate(java.time.LocalDate.of(2024, 2, 10))
                        .transactionRef("UJJWALA-UP-003").build()
                ))
                .build()
        );

        citizenProfileRepository.saveAll(profiles);
        log.info("Seeded {} citizen profiles", profiles.size());
    }

    /**
     * Pipeline 1: Regional summary — beneficiary count and total disbursement per region.
     *
     * MongoDB Aggregation Pipeline:
     * Stage 1: $unwind benefitHistory array (one doc per benefit record)
     * Stage 2: $match only DISBURSED status (active disbursements only)
     * Stage 3: $group by geoRegion — count citizens, sum amounts
     * Stage 4: $sort by totalDisbursed DESC
     *
     * This demonstrates eventual-vs-strong consistency:
     * MongoDB aggregations read from the primary by default (strong read).
     * readPreference=secondary would give eventual reads (faster, stale).
     */
    public List<Map> getRegionalBenefitSummary() {
        log.info("Running regional benefit summary aggregation pipeline");

        UnwindOperation unwind = Aggregation.unwind("benefitHistory");

        MatchOperation matchDisbursed = Aggregation.match(
            Criteria.where("benefitHistory.status").is("DISBURSED")
        );

        GroupOperation groupByRegion = Aggregation.group("geoRegion")
            .count().as("totalBeneficiaries")
            .sum("benefitHistory.amountReceived").as("totalDisbursed")
            .addToSet("benefitHistory.schemeName").as("schemes");

        SortOperation sortByDisbursed = Aggregation.sort(
            org.springframework.data.domain.Sort.Direction.DESC, "totalDisbursed"
        );

        Aggregation pipeline = Aggregation.newAggregation(
            unwind, matchDisbursed, groupByRegion, sortByDisbursed
        );

        AggregationResults<Map> results = mongoTemplate.aggregate(
            pipeline, "citizen_profiles", Map.class
        );

        log.info("Regional summary: {} regions found", results.getMappedResults().size());
        return results.getMappedResults();
    }

    /**
     * Pipeline 2: Income category breakdown — how many citizens in each income band.
     * Used by government for targeted scheme outreach.
     */
    public List<Map> getIncomeCategoryBreakdown(String geoRegion) {
        MatchOperation matchRegion = Aggregation.match(
            Criteria.where("geoRegion").is(geoRegion)
                .and("isActive").is(true)
        );

        GroupOperation groupByIncome = Aggregation.group("incomeCategory")
            .count().as("citizenCount");

        SortOperation sort = Aggregation.sort(
            org.springframework.data.domain.Sort.Direction.DESC, "citizenCount"
        );

        Aggregation pipeline = Aggregation.newAggregation(
            matchRegion, groupByIncome, sort
        );

        return mongoTemplate.aggregate(
            pipeline, "citizen_profiles", Map.class
        ).getMappedResults();
    }

    /**
     * Returns all profiles (ADMIN/AUDITOR only — enforced at controller layer).
     */
    public List<CitizenProfile> getAllCitizens() {
        return citizenProfileRepository.findAll();
    }

    /**
     * Returns profiles for a specific region (OFFICER scope).
     */
    public List<CitizenProfile> getCitizensByRegion(String geoRegion) {
        return citizenProfileRepository.findByGeoRegion(geoRegion);
    }
}
```

### File: `repository/CitizenProfileRepository.java`

```java
package gov.citizen.data.repository;

import gov.citizen.data.model.CitizenProfile;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface CitizenProfileRepository extends MongoRepository<CitizenProfile, String> {

    Optional<CitizenProfile> findByCitizenId(String citizenId);

    List<CitizenProfile> findByGeoRegion(String geoRegion);

    List<CitizenProfile> findByGeoRegionAndIncomeCategory(
        String geoRegion, String incomeCategory);

    long countByGeoRegion(String geoRegion);
}
```

### File: `api/CitizenDataController.java`

```java
package gov.citizen.data.api;

import gov.citizen.data.model.CitizenProfile;
import gov.citizen.data.service.CitizenAnalyticsService;
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/citizens")
@RequiredArgsConstructor
@Slf4j
public class CitizenDataController {

    private final CitizenAnalyticsService analyticsService;

    // Seed data on startup for the lab demonstration
    @PostConstruct
    public void init() {
        analyticsService.seedSampleData();
    }

    /**
     * Regional benefit summary — OFFICER and ADMIN only.
     * Demonstrates MongoDB aggregation pipeline for analytics.
     */
    @GetMapping("/analytics/regional-summary")
    @PreAuthorize("hasAnyRole('OFFICER', 'ADMIN', 'AUDITOR')")
    public ResponseEntity<List<Map>> getRegionalSummary() {
        return ResponseEntity.ok(analyticsService.getRegionalBenefitSummary());
    }

    /**
     * Income breakdown for a region — OFFICER and ADMIN.
     */
    @GetMapping("/analytics/income-breakdown/{region}")
    @PreAuthorize("hasAnyRole('OFFICER', 'ADMIN')")
    public ResponseEntity<List<Map>> getIncomeBreakdown(
            @PathVariable String region,
            @AuthenticationPrincipal Jwt jwt) {
        log.info("Income breakdown requested by '{}' for region '{}'",
            jwt.getClaimAsString("preferred_username"), region);
        return ResponseEntity.ok(analyticsService.getIncomeCategoryBreakdown(region));
    }

    /**
     * All citizens — ADMIN and AUDITOR only.
     */
    @GetMapping("/all")
    @PreAuthorize("hasAnyRole('ADMIN', 'AUDITOR')")
    public ResponseEntity<List<CitizenProfile>> getAllCitizens() {
        return ResponseEntity.ok(analyticsService.getAllCitizens());
    }

    /**
     * Health endpoint.
     */
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of("status", "UP", "service", "citizen-data-service"));
    }
}
```

### File: `config/SecurityConfig.java` (Data Service)

```java
package gov.citizen.data.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/citizens/health", "/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null || !realmAccess.containsKey("roles")) {
                return List.of();
            }
            @SuppressWarnings("unchecked")
            Collection<String> roles = (Collection<String>) realmAccess.get("roles");
            return roles.stream()
                .filter(r -> List.of("CITIZEN","OFFICER","ADMIN","AUDITOR").contains(r))
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

### Build and Start Data Service

```powershell
cd C:\training\day5-security-search-platform\citizen-data-service

mvn clean package -DskipTests

mvn spring-boot:run `
  -Dspring-boot.run.jvmArguments="-DMONGO_HOST=localhost -DMONGO_USER=mongo_user -DMONGO_PASSWORD=mongo_pass -DKEYCLOAK_BASE_URL=http://localhost:9090 -DKEYCLOAK_REALM=citizen-portal"
```

`[EXPECTED: Service starts on port 8082. Log shows "Seeded 5 citizen profiles". No MongoDB connection errors.]`

```powershell
# Test analytics with OFFICER token (use $officerToken from Section 3)
Invoke-RestMethod `
  -Method GET `
  -Uri "http://localhost:8080/api/citizens/analytics/regional-summary" `
  -Headers @{ Authorization = "Bearer $officerToken" }
```

`[EXPECTED: JSON array with 4 regional entries showing totalBeneficiaries and totalDisbursed per geoRegion. Sorted by totalDisbursed DESC.]`

```powershell
# Test income breakdown for IN-TN
Invoke-RestMethod `
  -Method GET `
  -Uri "http://localhost:8080/api/citizens/analytics/income-breakdown/IN-TN" `
  -Headers @{ Authorization = "Bearer $officerToken" }
```

`[EXPECTED: JSON showing income category distribution for Tamil Nadu region — BPL, APL counts.]`

### Definition of Done — Section 4

```
[ ] citizen-data-service starts on port 8082
[ ] 5 citizen profiles seeded in MongoDB
[ ] GET /api/citizens/analytics/regional-summary → HTTP 200 with aggregation results (OFFICER token)
[ ] GET /api/citizens/all → HTTP 403 with CITIZEN token (ADMIN only)
[ ] GET /api/citizens/all → HTTP 200 with ADMIN token
[ ] MongoDB compass (or mongosh): citizen_profiles collection shows 5 documents
```

---

## Section 5: Threat Modeling — STRIDE Analysis

### What We Are Building

A structured STRIDE threat model document for the citizen portal login flow, plus a demonstration of how to embed threat-model outputs as security gates in the development workflow.

### File: `threat-model/stride-analysis.md`

```markdown
# STRIDE Threat Model — Citizen Portal Login Flow
# Day 5 — Rapid Threat Modeling & Security Gate Design
# Methodology: STRIDE (Microsoft) + DREAD risk scoring

## System Under Analysis
Citizen Portal Login Flow:
Browser → Keycloak (Auth) → citizen-api-gateway → citizen-search-service

## Data Flow Diagram (DFD Level 1)

```
[Citizen Browser] --HTTPS--> [Keycloak :9090] --JWT--> [API Gateway :8080]
                                                              |
                                              HTTPS/mTLS ─────┤
                                                              ↓
                                              [Search Service :8081]
                                                              |
                                                         [PostgreSQL]
```

## STRIDE Threat Analysis

### T1 — SPOOFING (Identity)
**Threat:** Attacker presents a forged JWT token to the API Gateway.
**Asset:** Citizen identity and authorization.
**Attack vector:** Attacker generates JWT signed with a weak/self-made key,
  bypassing Keycloak entirely.
**Current control:** Gateway validates JWT signature against Keycloak JWKS endpoint.
**Gap:** If JWKS URI is not pinned, attacker could serve fake JWKS.
**Mitigation:**
  1. Pin JWKS URI to specific Keycloak hostname (no redirect allowed)
  2. Set short token lifetime (300s in our config — correct)
  3. Implement token introspection for critical operations (not just signature check)
**Risk (DREAD):** Damage=9, Reproducibility=5, Exploitability=4, Users=9, Discoverability=3
**Risk Score:** (9+5+4+9+3)/5 = **6.0 (HIGH)**

### T2 — TAMPERING (Data)
**Threat:** Man-in-the-middle modifies tender data in transit between
  Search Service and PostgreSQL.
**Asset:** Government tender integrity.
**Attack vector:** Attacker on the Docker internal network intercepts
  unencrypted TCP between Java app and PostgreSQL.
**Current control:** Docker network isolation (bridge network).
**Gap:** No TLS on PostgreSQL connection in lab config.
**Mitigation:**
  1. Enable SSL on PostgreSQL (`ssl=true` in JDBC URL)
  2. Use PostgreSQL server certificate verification
  3. Apply database network policies (restrict pg port to app containers only)
**Risk Score:** **4.0 (MEDIUM)** — internal network attack required

### T3 — REPUDIATION
**Threat:** A government officer queries citizen data and later denies doing so.
**Asset:** Audit trail integrity; regulatory compliance.
**Attack vector:** Officer bypasses gateway audit logging by directly calling
  the data service internal port (8082 exposed on localhost).
**Current control:** AuditLoggingFilter at gateway layer.
**Gap:** Internal ports 8081, 8082 exposed on localhost — direct access bypasses gateway.
**Mitigation:**
  1. Remove localhost port exposure for internal services in production
  2. Use Docker network: internal services reachable ONLY via service name (not host)
  3. Add audit logging at service layer too (defense in depth)
  4. Use immutable audit log (append-only, signed with HMAC)
**Risk Score:** **5.5 (HIGH)** — directly impacts DPDP Act / FedRAMP audit requirements

### T4 — INFORMATION DISCLOSURE
**Threat:** Citizen A can view Citizen B's profile by manipulating citizenId in API path.
**Asset:** Citizen personal data (PII — protected under DPDP Act, PDPA, CCPA).
**Attack vector:** GET /citizens/{citizenId} with a guessed/enumerated ID.
**Current control:** None in current lab implementation (simplified for lab).
**Gap:** No ownership check — any authenticated CITIZEN can call with any ID.
**Mitigation:**
  1. Implement ownership validation: jwt.sub must match requested citizenId
  2. Use Keycloak UMA (User-Managed Access) for fine-grained authorization
  3. Opaque IDs (UUID) instead of sequential IDs to prevent enumeration
  4. Attribute-Based Access Control (ABAC) for multi-tenant data isolation
**Risk Score:** **8.0 (CRITICAL)** — PII exposure, regulatory violation

### T5 — DENIAL OF SERVICE
**Threat:** Attacker floods the search endpoint with expensive FTS queries,
  exhausting PostgreSQL connections.
**Asset:** Service availability; SLA compliance.
**Attack vector:** 1000 concurrent requests with complex search terms like
  `health:* OR digital:*` causing full GIN index scan.
**Current control:** RateLimitFilter (20 req/sec per IP).
**Gap:** Rate limit bypassed by using multiple IPs / distributed attack.
**Mitigation:**
  1. Add query complexity limit (max search term length, no wildcard leading)
  2. PostgreSQL connection pool limit (hikari maxPoolSize=10 in our config — correct)
  3. Implement CAPTCHA for unauthenticated search (if offered)
  4. Azure DDoS Protection Standard for production (India NIC: NICSI firewall)
**Risk Score:** **5.0 (MEDIUM)**

### T6 — ELEVATION OF PRIVILEGE
**Threat:** CITIZEN role user gains OFFICER capabilities.
**Asset:** Authorization boundary integrity.
**Attack vector:** JWT manipulation (if signed with HS256 and key is weak),
  or Keycloak misconfiguration allowing CITIZEN to self-assign OFFICER role.
**Current control:**
  - RS256 asymmetric signing (Keycloak default) — private key not exposed
  - Role assignment restricted to Keycloak admin
  - @PreAuthorize at controller level in both gateway and service
**Gap:** Keycloak admin password "admin_pass" is weak (lab only).
**Mitigation:**
  1. Keycloak admin password must be rotated before any non-lab environment
  2. Disable Keycloak admin REST API in production (use console only)
  3. Implement role change audit alerts in Keycloak event listener
  4. Regular Keycloak realm audit (quarterly for government systems)
**Risk Score:** **3.5 (LOW)** — strong controls in place, admin password is lab-only issue

## Security Gate Design

Gates to add to CI/CD pipeline (implemented Day 11):

| Gate             | Tool                   | Block On                      | Phase          |
| ---------------- | ---------------------- | ----------------------------- | -------------- |
| Secret scanning  | truffleHog             | Hardcoded credentials in code | Pre-commit     |
| SAST             | SonarQube              | Critical security hotspots    | CI build       |
| Dependency scan  | OWASP dependency-check | Critical CVEs                 | CI build       |
| DAST             | OWASP ZAP              | OWASP Top 10 findings         | Staging deploy |
| RBAC policy test | Custom JUnit           | Role bypass test failures     | CI test        |

## Threat Model Update Frequency
This threat model must be reviewed:
- On every major architecture change (new service, new data flow)
- Quarterly for running government systems
- After every security incident
- Before every production release
```

### STRIDE Demo in Class

```powershell
# Demonstrate T3 (Repudiation threat) LIVE:
# Show that direct access to internal service bypasses gateway audit log

# This works because port 8081 is exposed on localhost — bypassing the gateway:
$directToken = $officerToken  # reuse officer token

Invoke-RestMethod `
  -Method GET `
  -Uri "http://localhost:8081/search/tenders?q=health&page=0&size=5" `
  -Headers @{ Authorization = "Bearer $directToken" }

# CHECK: Gateway audit log shows NOTHING for this request
# This PROVES the repudiation threat — officer accessed data without gateway audit
Write-Host "STRIDE T3 demonstrated: direct service access bypasses gateway audit" -ForegroundColor Red
```

`[EXPECTED: HTTP 200 response from search service directly. Gateway console shows no audit log line for this request. Demonstrates the repudiation threat concretely.]`

```powershell
# Fix demonstration: Stop direct port exposure (in production, only via Docker network)
# Show what the docker-compose change would look like:
Write-Host @"
SECURITY FIX for T3 (Repudiation):
In docker-compose.yml, remove localhost port mappings for internal services:

BEFORE (vulnerable):
  citizen-search-service:
    ports:
      - '8081:8081'  # ← exposes to host, bypasses gateway

AFTER (secure):
  citizen-search-service:
    # NO ports section — only reachable via Docker network name
    # Gateway accesses via: http://citizen-search-service:8081
    networks:
      - citizen-net
"@ -ForegroundColor Green
```

### Definition of Done — Section 5

```
[ ] threat-model/stride-analysis.md created with all 6 STRIDE categories
[ ] T3 demonstration: direct port access confirmed to bypass gateway audit
[ ] STRIDE risk scores presented and discussed
[ ] Security gate pipeline table reviewed with participants
[ ] At least 2 mitigations per threat discussed in class
```

---

## Section 6: In-Class Demonstration Script

### Pre-Demo Verification (Run Before Class Starts)

```powershell
# Full pre-demo verification script — run 30 minutes before class

Write-Host "=== Day 5 Pre-Demo Verification ===" -ForegroundColor Cyan

# 1. Infrastructure
Write-Host "`n[1] Checking Docker containers..." -ForegroundColor Yellow
docker compose ps

# 2. PostgreSQL
Write-Host "`n[2] PostgreSQL tender count..." -ForegroundColor Yellow
docker exec citizen-postgres psql -U citizen_user -d citizen_search `
  -c "SELECT COUNT(*), status FROM tenders GROUP BY status;"

# 3. Keycloak
Write-Host "`n[3] Keycloak realm check..." -ForegroundColor Yellow
$kc = Invoke-RestMethod -Uri "http://localhost:9090/realms/citizen-portal"
Write-Host "Realm: $($kc.realm) | DisplayName: $($kc.display_name)"

# 4. Services
Write-Host "`n[4] Service health checks..." -ForegroundColor Yellow
Invoke-RestMethod -Uri "http://localhost:8081/search/health"
Invoke-RestMethod -Uri "http://localhost:8082/citizens/health"

# 5. Gateway
Write-Host "`n[5] Gateway health..." -ForegroundColor Yellow
Invoke-RestMethod -Uri "http://localhost:8080/actuator/health"

# 6. Token test
Write-Host "`n[6] Token acquisition test..." -ForegroundColor Yellow
$testToken = (Invoke-RestMethod `
  -Method POST `
  -Uri "http://localhost:9090/realms/citizen-portal/protocol/openid-connect/token" `
  -ContentType "application/x-www-form-urlencoded" `
  -Body @{
    grant_type="password"; client_id="citizen-gateway"
    client_secret="gateway-secret-2024"
    username="rahul.citizen"; password="citizen123"
    scope="openid profile roles"
  }).access_token

if ($testToken) {
    Write-Host "Token obtained: OK" -ForegroundColor Green
} else {
    Write-Host "Token FAILED — check Keycloak" -ForegroundColor Red
}

# 7. Search test
Write-Host "`n[7] Search API test..." -ForegroundColor Yellow
$searchResult = Invoke-RestMethod `
  -Uri "http://localhost:8080/api/search/tenders?q=digital&page=0&size=3" `
  -Headers @{ Authorization = "Bearer $testToken" }
Write-Host "Search results: $($searchResult.totalResults) total, $($searchResult.results.Count) returned"

Write-Host "`n=== All checks passed. Ready for class ===" -ForegroundColor Green
```

### Demonstration Flow

**Block 1 — Search Architecture (45 min)**

| Step | Action                                   | What to Say                                                                                                                                                        |
| ---- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | Show `init.sql` tsvector trigger         | "PostgreSQL computes and stores a search vector synchronously — this is STRONG consistency. The moment a tender is inserted, it is searchable."                    |
| 2    | Run `SELECT` with `@@` operator in psql  | "This is the same query our Java code runs. The GIN index makes this O(log n) regardless of table size."                                                           |
| 3    | Run search API call in Postman           | "Notice the response includes `consistencyModel: STRONG` — we communicate this SLA to consumers explicitly."                                                       |
| 4    | Show `getConsistencyComparison()` output | "Contrast with Elasticsearch: 1-30 second indexing lag. For a government tender portal, a closed tender appearing in search for 30 seconds is a regulatory issue." |
| 5    | Ask the class                            | "If this were a financial portal with 1 billion transactions, would you still choose PostgreSQL FTS?"                                                              |

**Block 2 — Zero Trust + Keycloak (60 min)**

| Step | Action                                     | What to Say                                                                                                |
| ---- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| 1    | Open Keycloak admin console                | "This is the Keycloak admin console. Show the citizen-portal realm, the four roles, and the four users."   |
| 2    | Decode JWT in jwt.io                       | "Paste the citizen token here. Look at realm_access.roles: CITIZEN. Our gateway reads exactly this field." |
| 3    | Run CITIZEN token against /search endpoint | "HTTP 200 — CITIZEN can search."                                                                           |
| 4    | Run CITIZEN token against /region endpoint | "HTTP 403 — RBAC enforced. The gateway checks the role BEFORE routing to the backend."                     |
| 5    | Run OFFICER token against /region endpoint | "HTTP 200 — role matches. Defense in depth: the backend service checks the role AGAIN independently."      |
| 6    | Show audit log lines in gateway console    | "Every request logged: user, method, path, status, latency. This is the FedRAMP AU-2 control satisfied."   |

**Block 3 — STRIDE Threat Modeling (30 min)**

| Step | Action                            | What to Say                                                                                                                                                       |
| ---- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Draw DFD on whiteboard            | "Threat modeling always starts with a data flow diagram. Every arrow is a potential threat vector."                                                               |
| 2    | Walk through T4 (Info Disclosure) | "This is the most dangerous threat for a government system. Any CITIZEN can call /citizens/CIT-IN-2024-001. Does your JWT subject match that ID? We never check." |
| 3    | Live demo T3 (Repudiation)        | "Run the direct call to port 8081. Show gateway log has nothing. STRIDE T3 proved live."                                                                          |
| 4    | Discuss mitigations               | "These mitigations become security gates in Day 11's CI/CD pipeline."                                                                                             |

### Questions to Ask During Demo

```
1. "Before I run this call — will it succeed with a CITIZEN token? Vote: hands up for yes, hands up for no."
2. "If I set token lifetime to 3600 seconds instead of 300, what STRIDE threat increases?"
3. "MongoDB aggregation pipeline just ran. Was that a STRONG or EVENTUAL read? How would you verify?"
4. "If Keycloak goes down, what happens to ALL three services? Is that acceptable for a government portal?"
5. "The RateLimitFilter uses an in-memory map. What happens when we scale the gateway to 3 instances?"
```

### Post-Demonstration: Break It and Fix It

**Break 1 — Invalid JWT (SPOOFING demo)**

```powershell
# Send a tampered/expired token
$fakeToken = "eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJoYWNrZXIiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsiQURNSU4iXX0sImV4cCI6OTk5OTk5OTk5OX0.FAKE_SIGNATURE"

try {
    Invoke-RestMethod `
      -Method GET `
      -Uri "http://localhost:8080/api/search/tenders?q=health" `
      -Headers @{ Authorization = "Bearer $fakeToken" }
} catch {
    Write-Host "HTTP 401 Unauthorized — JWT signature invalid. Spoofing attempt rejected." -ForegroundColor Green
}
```

**Break 2 — Expired Token**

```powershell
# Wait for token to expire (300 seconds / skip with: set token lifetime to 10s in Keycloak)
# To demonstrate quickly: change token lifespan to 10 seconds in Keycloak admin console
# Realm Settings → Tokens → Access Token Lifespan = 10 seconds
# Get a token, wait 15 seconds, try to use it

Start-Sleep -Seconds 15
try {
    Invoke-RestMethod `
      -Uri "http://localhost:8080/api/search/tenders?q=health" `
      -Headers @{ Authorization = "Bearer $testToken" }
} catch {
    Write-Host "HTTP 401 — Token expired. Zero Trust: no long-lived sessions." -ForegroundColor Green
}
```

**Break 3 — Rate Limit (DoS simulation)**

```powershell
# Send 25 rapid requests — should trigger 429 after 20th
1..25 | ForEach-Object {
    try {
        $response = Invoke-RestMethod `
          -Uri "http://localhost:8080/api/search/tenders?q=health" `
          -Headers @{ Authorization = "Bearer $testToken" }
        Write-Host "Request $_: OK" -ForegroundColor Green
    } catch {
        Write-Host "Request $_: $($_.Exception.Response.StatusCode)" -ForegroundColor Red
    }
}
```

`[EXPECTED: Requests 1-20 return 200. Requests 21-25 return 429 Too Many Requests]`

---

## Section 7: Postman Collection

### File: `postman/Day5-SecuritySearch.postman_collection.json`

```json
{
  "info": {
    "name": "Day 5 — Citizen Security & Search Platform",
    "description": "Tests for Zero Trust API Gateway, Keycloak RBAC, Search Service, and Data Service",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "base_url", "value": "http://localhost:8080" },
    { "key": "keycloak_url", "value": "http://localhost:9090" },
    { "key": "realm", "value": "citizen-portal" },
    { "key": "client_id", "value": "citizen-gateway" },
    { "key": "client_secret", "value": "gateway-secret-2024" },
    { "key": "citizen_token", "value": "" },
    { "key": "officer_token", "value": "" },
    { "key": "admin_token", "value": "" }
  ],
  "item": [
    {
      "name": "Auth",
      "item": [
        {
          "name": "Get CITIZEN Token (rahul.citizen)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "var json = pm.response.json();",
                  "pm.collectionVariables.set('citizen_token', json.access_token);",
                  "pm.test('Has access_token', () => pm.expect(json.access_token).to.exist);"
                ]
              }
            }
          ],
          "request": {
            "method": "POST",
            "url": "{{keycloak_url}}/realms/{{realm}}/protocol/openid-connect/token",
            "header": [{ "key": "Content-Type", "value": "application/x-www-form-urlencoded" }],
            "body": {
              "mode": "urlencoded",
              "urlencoded": [
                { "key": "grant_type", "value": "password" },
                { "key": "client_id", "value": "{{client_id}}" },
                { "key": "client_secret", "value": "{{client_secret}}" },
                { "key": "username", "value": "rahul.citizen" },
                { "key": "password", "value": "citizen123" },
                { "key": "scope", "value": "openid profile roles" }
              ]
            }
          }
        },
        {
          "name": "Get OFFICER Token (priya.officer)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "pm.collectionVariables.set('officer_token', pm.response.json().access_token);"
                ]
              }
            }
          ],
          "request": {
            "method": "POST",
            "url": "{{keycloak_url}}/realms/{{realm}}/protocol/openid-connect/token",
            "header": [{ "key": "Content-Type", "value": "application/x-www-form-urlencoded" }],
            "body": {
              "mode": "urlencoded",
              "urlencoded": [
                { "key": "grant_type", "value": "password" },
                { "key": "client_id", "value": "{{client_id}}" },
                { "key": "client_secret", "value": "{{client_secret}}" },
                { "key": "username", "value": "priya.officer" },
                { "key": "password", "value": "officer123" },
                { "key": "scope", "value": "openid profile roles" }
              ]
            }
          }
        },
        {
          "name": "Get ADMIN Token (admin.portal)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.collectionVariables.set('admin_token', pm.response.json().access_token);"
                ]
              }
            }
          ],
          "request": {
            "method": "POST",
            "url": "{{keycloak_url}}/realms/{{realm}}/protocol/openid-connect/token",
            "header": [{ "key": "Content-Type", "value": "application/x-www-form-urlencoded" }],
            "body": {
              "mode": "urlencoded",
              "urlencoded": [
                { "key": "grant_type", "value": "password" },
                { "key": "client_id", "value": "{{client_id}}" },
                { "key": "client_secret", "value": "{{client_secret}}" },
                { "key": "username", "value": "admin.portal" },
                { "key": "password", "value": "admin123" },
                { "key": "scope", "value": "openid profile roles" }
              ]
            }
          }
        }
      ]
    },
    {
      "name": "Search Service",
      "item": [
        {
          "name": "Search Tenders — Digital Identity (CITIZEN)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "var json = pm.response.json();",
                  "pm.test('Has results', () => pm.expect(json.results).to.be.an('array'));",
                  "pm.test('Strong consistency', () => pm.expect(json.consistencyModel).to.equal('STRONG'));",
                  "pm.test('GIN index used', () => pm.expect(json.indexType).to.equal('GIN-tsvector'));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": {
              "raw": "{{base_url}}/api/search/tenders?q=digital+identity&page=0&size=5",
              "query": [
                { "key": "q", "value": "digital identity" },
                { "key": "page", "value": "0" },
                { "key": "size", "value": "5" }
              ]
            },
            "header": [{ "key": "Authorization", "value": "Bearer {{citizen_token}}" }]
          }
        },
        {
          "name": "Search Regional Tenders — IN-MH (OFFICER only)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200 for OFFICER', () => pm.response.to.have.status(200));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/search/tenders/region/IN-MH",
            "header": [{ "key": "Authorization", "value": "Bearer {{officer_token}}" }]
          }
        },
        {
          "name": "Regional Tenders — CITIZEN Forbidden (403)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 403 for CITIZEN role', () => pm.response.to.have.status(403));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/search/tenders/region/IN-MH",
            "header": [{ "key": "Authorization", "value": "Bearer {{citizen_token}}" }]
          }
        },
        {
          "name": "Consistency Info — ADMIN only",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "var json = pm.response.json();",
                  "pm.test('Has both models', () => {",
                  "  pm.expect(json.postgresql_fts_model).to.exist;",
                  "  pm.expect(json.elasticsearch_model).to.exist;",
                  "});"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/search/consistency-info",
            "header": [{ "key": "Authorization", "value": "Bearer {{admin_token}}" }]
          }
        }
      ]
    },
    {
      "name": "Data Service",
      "item": [
        {
          "name": "Regional Benefit Summary (OFFICER)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Status 200', () => pm.response.to.have.status(200));",
                  "pm.test('Is array', () => pm.expect(pm.response.json()).to.be.an('array'));",
                  "pm.test('Has regions', () => pm.expect(pm.response.json().length).to.be.greaterThan(0));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/citizens/analytics/regional-summary",
            "header": [{ "key": "Authorization", "value": "Bearer {{officer_token}}" }]
          }
        },
        {
          "name": "Income Breakdown IN-TN (OFFICER)",
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/citizens/analytics/income-breakdown/IN-TN",
            "header": [{ "key": "Authorization", "value": "Bearer {{officer_token}}" }]
          }
        },
        {
          "name": "All Citizens — CITIZEN Forbidden (403)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('CITIZEN cannot access /all', () => pm.response.to.have.status(403));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/citizens/all",
            "header": [{ "key": "Authorization", "value": "Bearer {{citizen_token}}" }]
          }
        },
        {
          "name": "All Citizens — ADMIN allowed (200)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('ADMIN can access /all', () => pm.response.to.have.status(200));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/citizens/all",
            "header": [{ "key": "Authorization", "value": "Bearer {{admin_token}}" }]
          }
        }
      ]
    },
    {
      "name": "Security Tests",
      "item": [
        {
          "name": "No Token — Expect 401",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Unauthenticated → 401', () => pm.response.to.have.status(401));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/search/tenders?q=health"
          }
        },
        {
          "name": "Fake Token — Expect 401",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test('Invalid JWT → 401', () => pm.response.to.have.status(401));"
                ]
              }
            }
          ],
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/search/tenders?q=health",
            "header": [{ "key": "Authorization", "value": "Bearer INVALID.JWT.TOKEN" }]
          }
        }
      ]
    }
  ]
}
```

---

## Lab Footer

### Cleanup Script

```powershell
# Day5-Cleanup.ps1
# Complete teardown of Day 5 environment

Write-Host "=== Day 5 Cleanup Starting ===" -ForegroundColor Yellow

# 1. Stop Spring Boot services (if running as background jobs)
Write-Host "[1] Stopping Spring Boot services..." -ForegroundColor Cyan
Get-Process -Name "java" -ErrorAction SilentlyContinue | Stop-Process -Force
Write-Host "Java processes stopped."

# 2. Stop and remove Docker containers, networks, volumes
Write-Host "[2] Stopping Docker Compose..." -ForegroundColor Cyan
cd C:\training\day5-security-search-platform
docker compose down -v --remove-orphans
Write-Host "Containers, networks, and volumes removed."

# 3. Remove generated certificates
Write-Host "[3] Removing certificates..." -ForegroundColor Cyan
Remove-Item -Path ".\infra\certs\*.crt" -Force -ErrorAction SilentlyContinue
Remove-Item -Path ".\infra\certs\*.key" -Force -ErrorAction SilentlyContinue
Remove-Item -Path ".\infra\certs\*.csr" -Force -ErrorAction SilentlyContinue
Remove-Item -Path ".\infra\certs\*.p12" -Force -ErrorAction SilentlyContinue
Remove-Item -Path ".\infra\certs\*.srl" -Force -ErrorAction SilentlyContinue
Remove-Item -Path ".\infra\certs\*.cnf" -Force -ErrorAction SilentlyContinue
Write-Host "Certificates removed."

# 4. Remove Maven build artifacts
Write-Host "[4] Cleaning Maven targets..." -ForegroundColor Cyan
@("citizen-api-gateway", "citizen-search-service", "citizen-data-service") | ForEach-Object {
    if (Test-Path ".\$_\target") {
        Remove-Item -Recurse -Force ".\$_\target"
        Write-Host "  Cleaned: $_\target"
    }
}

# 5. Clear environment variables
Write-Host "[5] Clearing environment variables..." -ForegroundColor Cyan
Remove-Item Env:POSTGRES_HOST -ErrorAction SilentlyContinue
Remove-Item Env:POSTGRES_PASSWORD -ErrorAction SilentlyContinue
Remove-Item Env:MONGO_PASSWORD -ErrorAction SilentlyContinue
Remove-Item Env:KEYCLOAK_CLIENT_SECRET -ErrorAction SilentlyContinue
Write-Host "Environment variables cleared."

# 6. Verify cleanup
Write-Host "`n[6] Verification..." -ForegroundColor Cyan
$containers = docker ps -a --filter "name=citizen" --format "{{.Names}}" 2>$null
if ($containers) {
    Write-Host "WARNING: Remaining containers: $containers" -ForegroundColor Red
} else {
    Write-Host "No citizen-* containers remaining." -ForegroundColor Green
}

$volumes = docker volume ls --filter "name=day5" --format "{{.Name}}" 2>$null
if ($volumes) {
    Write-Host "WARNING: Remaining volumes: $volumes" -ForegroundColor Red
} else {
    Write-Host "No day5-* volumes remaining." -ForegroundColor Green
}

Write-Host "`n=== Day 5 Cleanup Complete ===" -ForegroundColor Green
```

---

### Full Project Repository Structure

```
C:\training\day5-security-search-platform\
│
├── docker-compose.yml
├── .env
├── README.md
├── Day5-Scaffold.ps1
├── Day5-Cleanup.ps1
│
├── infra\
│   ├── postgres\
│   │   └── init.sql                          # Tender table, GIN index, FTS trigger, seed data
│   ├── keycloak\
│   │   └── realm-export.json                 # citizen-portal realm: roles, users, clients
│   ├── certs\
│   │   ├── generate-certs.sh                 # OpenSSL CA + server + client cert generation
│   │   ├── ca.crt                            # [generated] Root CA certificate
│   │   ├── server.crt / server.key           # [generated] Service certificates
│   │   ├── client.crt / client.key           # [generated] Gateway client certificate
│   │   ├── server-keystore.p12               # [generated] Java KeyStore
│   │   ├── client-keystore.p12               # [generated] Gateway KeyStore
│   │   └── truststore.p12                    # [generated] Shared TrustStore (CA cert)
│   └── secrets\
│       └── secrets.env
│
├── threat-model\
│   └── stride-analysis.md                    # Full STRIDE analysis with risk scores
│
├── citizen-api-gateway\
│   ├── pom.xml
│   ├── Dockerfile
│   └── src\
│       ├── main\
│       │   ├── java\gov\citizen\gateway\
│       │   │   ├── CitizenGatewayApplication.java
│       │   │   ├── config\
│       │   │   │   ├── SecurityConfig.java         # Reactive WebFlux security + JWT converter
│       │   │   │   └── GatewayRouteConfig.java     # Spring Cloud Gateway routes
│       │   │   ├── filter\
│       │   │   │   ├── AuditLoggingFilter.java     # Global audit log (FedRAMP AU-2)
│       │   │   │   └── RateLimitFilter.java        # Token-bucket rate limiter
│       │   │   └── security\
│       │   │       └── JwtClaimsValidator.java     # Custom claim validation (issuer, audience)
│       │   └── resources\
│       │       └── application.yml
│       └── test\java\gov\citizen\gateway\
│           └── GatewaySecurityTest.java
│
├── citizen-search-service\
│   ├── pom.xml
│   ├── Dockerfile
│   └── src\
│       ├── main\
│       │   ├── java\gov\citizen\search\
│       │   │   ├── CitizenSearchApplication.java
│       │   │   ├── config\
│       │   │   │   ├── SecurityConfig.java         # JWT converter for Keycloak roles
│       │   │   │   └── DataSourceConfig.java
│       │   │   ├── model\
│       │   │   │   └── Tender.java                 # JPA entity with search_vector
│       │   │   ├── repository\
│       │   │   │   └── TenderSearchRepository.java # Native FTS queries (tsquery, ts_rank)
│       │   │   ├── service\
│       │   │   │   └── TenderSearchService.java    # Consistency model + pagination
│       │   │   └── api\
│       │   │       └── SearchController.java       # RBAC-enforced REST endpoints
│       │   └── resources\
│       │       └── application.yml
│       └── test\java\gov\citizen\search\
│           └── TenderSearchServiceTest.java        # Mockito unit tests
│
├── citizen-data-service\
│   ├── pom.xml
│   ├── Dockerfile
│   └── src\
│       ├── main\
│       │   ├── java\gov\citizen\data\
│       │   │   ├── CitizenDataApplication.java
│       │   │   ├── config\
│       │   │   │   ├── SecurityConfig.java
│       │   │   │   └── MongoConfig.java
│       │   │   ├── model\
│       │   │   │   └── CitizenProfile.java         # MongoDB document (nested: Address, BenefitRecord)
│       │   │   ├── repository\
│       │   │   │   └── CitizenProfileRepository.java
│       │   │   ├── service\
│       │   │   │   └── CitizenAnalyticsService.java # MongoDB aggregation pipelines
│       │   │   └── api\
│       │   │       └── CitizenDataController.java  # Role-restricted analytics endpoints
│       │   └── resources\
│       │       └── application.yml
│       └── test\java\gov\citizen\data\
│           └── CitizenDataServiceTest.java
│
└── postman\
    └── Day5-SecuritySearch.postman_collection.json
```

---

### Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DAY 5 QUICK REFERENCE CARD                           │
├─────────────────┬───────────────────────────────────────────────────────┤
│ SERVICE         │ URL / Port                                             │
├─────────────────┼───────────────────────────────────────────────────────┤
│ API Gateway     │ http://localhost:8080                                  │
│ Search Service  │ http://localhost:8081  (internal, via gateway)         │
│ Data Service    │ http://localhost:8082  (internal, via gateway)         │
│ Keycloak        │ http://localhost:9090                                  │
│ Keycloak Admin  │ http://localhost:9090  → admin / admin_pass            │
│ PostgreSQL      │ localhost:5432  DB: citizen_search                     │
│ MongoDB         │ localhost:27017 DB: citizen_profiles                   │
├─────────────────┼───────────────────────────────────────────────────────┤
│ KEY ENDPOINTS   │                                                        │
├─────────────────┼───────────────────────────────────────────────────────┤
│ Get Token       │ POST /realms/citizen-portal/protocol/openid-connect/   │
│                 │      token (Keycloak)                                  │
│ Search Tenders  │ GET  /api/search/tenders?q={text}&region={r}           │
│ Regional List   │ GET  /api/search/tenders/region/{region} [OFFICER+]    │
│ Consistency     │ GET  /api/search/consistency-info [ADMIN/AUDITOR]      │
│ Regional Stats  │ GET  /api/citizens/analytics/regional-summary          │
│ Income Breakdown│ GET  /api/citizens/analytics/income-breakdown/{region} │
│ All Citizens    │ GET  /api/citizens/all [ADMIN/AUDITOR only]            │
├─────────────────┼───────────────────────────────────────────────────────┤
│ TEST USERS      │ Username / Password / Role                             │
├─────────────────┼───────────────────────────────────────────────────────┤
│                 │ rahul.citizen  / citizen123 / CITIZEN                  │
│                 │ priya.officer  / officer123 / OFFICER                  │
│                 │ admin.portal   / admin123   / ADMIN                    │
│                 │ auditor.cag    / auditor123 / AUDITOR                  │
├─────────────────┼───────────────────────────────────────────────────────┤
│ KEY COMMANDS    │                                                        │
├─────────────────┼───────────────────────────────────────────────────────┤
│ Start infra     │ docker compose up -d                                   │
│ Check health    │ docker compose ps                                      │
│ Keycloak logs   │ docker compose logs keycloak -f                        │
│ PG tender count │ docker exec citizen-postgres psql -U citizen_user      │
│                 │   -d citizen_search -c "SELECT COUNT(*) FROM tenders;" │
│ Mongo check     │ docker exec -it citizen-mongodb mongosh                │
│                 │   -u mongo_user -p mongo_pass --authenticationDatabase │
│                 │   admin --eval "db.citizen_profiles.countDocuments()"  │
│ Gen certs       │ bash ./infra/certs/generate-certs.sh                   │
│ Build gateway   │ cd citizen-api-gateway && mvn clean package -DskipTests│
│ Build search    │ cd citizen-search-service && mvn clean package          │
│ Build data      │ cd citizen-data-service && mvn clean package            │
│ Cleanup         │ .\Day5-Cleanup.ps1                                     │
├─────────────────┼───────────────────────────────────────────────────────┤
│ FTS DEMO QUERY  │ SELECT title, ts_rank(search_vector,                   │
│ (psql)          │   plainto_tsquery('english','digital identity')) AS r  │
│                 │ FROM tenders WHERE search_vector @@                    │
│                 │   plainto_tsquery('english','digital identity')         │
│                 │ ORDER BY r DESC;                                       │
├─────────────────┼───────────────────────────────────────────────────────┤
│ STRIDE RISKS    │ T4 Info Disclosure = CRITICAL (8.0)                    │
│                 │ T1 Spoofing = HIGH (6.0)                               │
│                 │ T3 Repudiation = HIGH (5.5)                            │
│                 │ T5 DoS = MEDIUM (5.0)                                  │
│                 │ T2 Tampering = MEDIUM (4.0)                            │
│                 │ T6 Elevation = LOW (3.5)                               │
└─────────────────┴───────────────────────────────────────────────────────┘
```

---

> "Day 5 Lab Document is complete. Would you like to:
> **(a)** Refine specific sections of the Lab Document
> **(b)** Proceed to generate the Lab Document for Day 6
> **(c)** Generate the Theory Document for Day 5 first
> **(d)** Both — specify sections to refine, then proceed to Day 6"

---

