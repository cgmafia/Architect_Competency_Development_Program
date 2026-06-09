# LAB MANUAL: Day 2 — Domain-Driven Design, API-First & Interoperability Workbench

**Phase:** Architectural Foundations & Design Thinking
**Day:** 2 of 14 | Date: Thursday, June 12, 2025
**Topics:** API-First Design, Domain-Driven Design, Hexagonal Architecture, Interoperability Patterns
**Tech Stack:** Java 17 | Spring Boot 3.3 | PostgreSQL 15 | Python 3.10+ | Terraform 1.5+
**Orchestration:** Docker Desktop | Docker Compose
**Cloud:** Azure Free Tier
**OS:** Windows 11 | PowerShell 7+ | Git Bash
**Geography Context:** 🇮🇳 India / 🇸🇬 Singapore — Nivesh Gateway Scenario
**Builds On:** Day 1 Architecture Decision Workbench (PostgreSQL, Docker Compose patterns)
**Estimated Lab Time:** 90 minutes | In-Class Demo: 25 minutes

---

## IMPORTANT: HOW TO USE THIS DOCUMENT

This document is fully self-contained. Every command, expected output, error fix, and verification step is written here. You do not need internet access after completing the one-time setup in L1. Follow every step in sequence. Do not skip sections.

Commands are written for both PowerShell and Git Bash where they differ. When a command works identically in both, only one version is shown. Git Bash is preferred for curl commands. PowerShell is used for file creation and Windows-specific operations.

This lab builds the next layer of the Nivesh Gateway platform. Day 1 built the governance workbench. Day 2 builds the actual domain services using DDD bounded contexts, hexagonal architecture, and API-First design principles.

---

## L0. LAB CONTEXT AND ARCHITECTURE NARRATIVE

### What This Lab Demonstrates

Day 2 training covered three interconnected concepts: Domain-Driven Design with bounded contexts and aggregates, Hexagonal (Ports and Adapters) architecture, and API-First design using OpenAPI contracts. This lab makes those concepts tangible by building two bounded domain services for the Nivesh Gateway platform, connected through a well-defined API contract.

### Scenario

Nivesh Gateway needs two clearly separated domain services:

The **Investor Profile Service** (Java / Spring Boot) owns the Investor bounded context. It handles investor identity, KYC status, and investment preferences. This is the core domain — the heart of the business.

The **Product Catalogue Service** (Python / FastAPI) owns the Product bounded context. It handles Singapore-regulated investment product listings, eligibility rules, and risk classifications.

These two services must never share a database. They must communicate only through their published APIs. They are deployed in the same Docker Compose stack but are architecturally independent — a concrete demonstration of bounded context isolation.

A third component, the **API Gateway Stub** (Nginx), sits in front of both services and routes requests by path prefix. This demonstrates the interoperability pattern from Block 2 of today's training.

### Architecture Overview

```mermaid
graph TD
    CLIENT["Client\nPowerShell / Git Bash / Browser"] --> GW["API Gateway Stub\nNginx\nPort 8090"]

    GW -->|"/api/investors/*"| INV["Investor Profile Service\nJava 17 / Spring Boot 3\nPort 8080\nHexagonal Architecture"]
    GW -->|"/api/products/*"| PROD["Product Catalogue Service\nPython 3.10 / FastAPI\nPort 8000\nHexagonal Architecture"]

    INV -->|"JDBC / HikariCP"| INVDB["investor_db\nPostgreSQL 15\nPort 5432"]
    PROD -->|"SQLAlchemy"| PRODDB["product_db\nPostgreSQL 15\nPort 5432"]

    INV -.->|"OpenAPI Contract\n(published, not shared DB)"| PROD

    TF["Terraform\n(host)"] -->|"az login + apply"| AZ["Azure Free Tier\nResource Group\n2 Storage Containers"]

    style INV fill:#004488,color:#fff
    style PROD fill:#006622,color:#fff
    style GW fill:#884400,color:#fff
    style INVDB fill:#336699,color:#fff
    style PRODDB fill:#336699,color:#fff
```

### Hexagonal Architecture Applied to Investor Profile Service

```mermaid
graph LR
    subgraph DRIVING ["Driving Adapters (Left)"]
        REST["REST Controller\n(Primary Adapter)"]
        TEST["Integration Tests\n(Primary Adapter)"]
    end

    subgraph CORE ["Domain Core (Hexagon)"]
        PORT_IN["InvestorUseCase\n(Input Port)"]
        APP["InvestorApplicationService\n(Use Case Impl)"]
        DOM["Investor Aggregate\nKycStatus ValueObject\nRiskProfile ValueObject"]
        PORT_OUT["InvestorRepository\n(Output Port)"]
    end

    subgraph DRIVEN ["Driven Adapters (Right)"]
        JPA["JPA Repository Adapter\n(Secondary Adapter)"]
        DB["PostgreSQL\ninvestor_db"]
    end

    REST --> PORT_IN
    TEST --> PORT_IN
    PORT_IN --> APP
    APP --> DOM
    APP --> PORT_OUT
    PORT_OUT --> JPA
    JPA --> DB
```

### Bounded Context Separation

```mermaid
graph TD
    subgraph BC1 ["Bounded Context: Investor (Java)"]
        I1["Investor Aggregate"]
        I2["KycStatus Value Object"]
        I3["RiskProfile Value Object"]
        I4["investor_db schema"]
    end

    subgraph BC2 ["Bounded Context: Product (Python)"]
        P1["InvestmentProduct Aggregate"]
        P2["RiskClassification Value Object"]
        P3["EligibilityCriteria Value Object"]
        P4["product_db schema"]
    end

    BC1 -. "API Contract Only\n(OpenAPI)\nNo shared DB\nNo shared classes" .-> BC2

    style BC1 fill:#003366,color:#fff
    style BC2 fill:#004400,color:#fff
```

### Data Flow: Cross-Context Eligibility Check

```mermaid
sequenceDiagram
    participant CLI as Client
    participant GW as Nginx Gateway :8090
    participant INV as Investor Service :8080
    participant PROD as Product Service :8000

    CLI->>GW: GET /api/investors/INV-001/eligible-products
    GW->>INV: GET /api/investors/INV-001/eligible-products
    INV->>INV: Load Investor Aggregate from investor_db
    INV->>PROD: GET /api/products?riskProfile=MODERATE (via HTTP client)
    PROD->>PROD: Filter products by risk classification
    PROD-->>INV: Product list (JSON)
    INV->>INV: Apply investor-side eligibility rules
    INV-->>GW: Eligible product list
    GW-->>CLI: 200 OK (filtered product list)
```

### Concepts From Training This Lab Demonstrates

- Bounded contexts with strict database isolation (DDD — Block 1, Concept 1)
- Hexagonal architecture — ports and adapters in Java (Block 1, Concept 2)
- API-First design — OpenAPI contract as the source of truth (Block 1, Concept 3)
- Interoperability via HTTP API contracts, not shared databases (Block 2, Use Case 1)
- Versioning strategy — API path versioning (Block 2, Advanced Concepts)
- Ubiquitous language — domain terms in code, not generic CRUD names (Block 1, Concept 1)

### Production Delta

In production: services communicate via Azure API Management instead of Nginx. PostgreSQL runs on Azure Database for PostgreSQL Flexible Server with per-schema access controls. Service-to-service calls use managed identity and mTLS. The OpenAPI contract is published to an internal developer portal. Event-driven integration (Kafka) supplements synchronous calls for high-volume scenarios. Azure Free Tier limits mean we use a single PostgreSQL instance with two databases, and Nginx as a lightweight gateway stub.

---

## L1. PREREQUISITES AND ONE-TIME ENVIRONMENT SETUP

### L1.1 Required Software

All tools from Day 1 remain required. Verify they are still available:

```powershell
Write-Host "=== Day 2 Environment Check ===" -ForegroundColor Cyan
docker --version
java -version
mvn -version
python --version
terraform version
git --version
jq --version
Write-Host "=== Check complete ===" -ForegroundColor Green
```

Additional requirement for Day 2:

| Software          | Purpose                 | Verify                |
| ----------------- | ----------------------- | --------------------- |
| Python sqlalchemy | ORM for Product service | `pip show sqlalchemy` |
| Python asyncpg    | Async PostgreSQL driver | `pip show asyncpg`    |
| Python alembic    | Database migrations     | `pip show alembic`    |

These are installed automatically by the requirements.txt in this lab.

### L1.2 Docker Desktop Check

```powershell
docker info | Select-String "Server Version"
# Expected: Server Version: 24.x.x or higher
```

If Day 1 containers are still running, stop them first:

```powershell
# Only if Day 1 is still running
Set-Location "$HOME\projects\architecture-decision-workbench\docker"
docker-compose down
Set-Location "$HOME\projects"
```

### L1.3 Project Scaffold Creation

```powershell
New-Item -ItemType Directory -Path "$HOME\projects\nivesh-gateway-ddd" -Force
Set-Location "$HOME\projects\nivesh-gateway-ddd"
```

Create the full directory structure:

```powershell
$dirs = @(
    # Investor Service (Java / Hexagonal)
    "investor-service\src\main\java\com\nivesh\investor\domain\model",
    "investor-service\src\main\java\com\nivesh\investor\domain\port\in",
    "investor-service\src\main\java\com\nivesh\investor\domain\port\out",
    "investor-service\src\main\java\com\nivesh\investor\application",
    "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest",
    "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto",
    "investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence",
    "investor-service\src\main\java\com\nivesh\investor\adapter\out\external",
    "investor-service\src\main\java\com\nivesh\investor\config",
    "investor-service\src\main\resources\db\migration",
    "investor-service\src\test\java\com\nivesh\investor",
    # Product Service (Python / Hexagonal)
    "product-service\src\domain\model",
    "product-service\src\domain\port",
    "product-service\src\application",
    "product-service\src\adapter\in_rest",
    "product-service\src\adapter\out_persistence",
    "product-service\src\migrations\versions",
    "product-service\tests",
    # Infrastructure and gateway
    "gateway\nginx",
    "infra\terraform",
    "docker",
    "scripts",
    "contracts\openapi",
    "docs\architecture\ADRs",
    ".github\workflows"
)
foreach ($dir in $dirs) {
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
}
Write-Host "Directory structure created." -ForegroundColor Green
```

---

## L2. PROJECT STRUCTURE

```
nivesh-gateway-ddd\
├── investor-service\                          ← Java / Spring Boot / Hexagonal
│   └── src\
│       ├── main\
│       │   ├── java\com\nivesh\investor\
│       │   │   ├── domain\
│       │   │   │   ├── model\
│       │   │   │   │   ├── Investor.java          ← Aggregate Root
│       │   │   │   │   ├── KycStatus.java         ← Value Object (enum)
│       │   │   │   │   └── RiskProfile.java       ← Value Object (enum)
│       │   │   │   ├── port\
│       │   │   │   │   ├── in\
│       │   │   │   │   │   └── InvestorUseCase.java   ← Input Port
│       │   │   │   │   └── out\
│       │   │   │   │       ├── InvestorRepository.java ← Output Port
│       │   │   │   │       └── ProductCataloguePort.java ← Anti-Corruption Layer Port
│       │   │   ├── application\
│       │   │   │   └── InvestorApplicationService.java ← Use Case Impl
│       │   │   ├── adapter\
│       │   │   │   ├── in\rest\
│       │   │   │   │   ├── InvestorController.java
│       │   │   │   │   ├── GlobalExceptionHandler.java
│       │   │   │   │   └── dto\
│       │   │   │   │       ├── CreateInvestorRequest.java
│       │   │   │   │       └── InvestorResponse.java
│       │   │   │   └── out\
│       │   │   │       ├── persistence\
│       │   │   │       │   ├── InvestorJpaEntity.java
│       │   │   │       │   ├── InvestorJpaRepository.java
│       │   │   │       │   └── InvestorPersistenceAdapter.java
│       │   │   │       └── external\
│       │   │   │           └── ProductCatalogueAdapter.java ← HTTP Anti-Corruption Layer
│       │   │   ├── config\
│       │   │   │   └── AppConfig.java
│       │   │   └── InvestorServiceApp.java
│       │   └── resources\
│       │       ├── application.yml
│       │       └── db\migration\
│       │           └── V1__create_investor_tables.sql
│       └── test\java\com\nivesh\investor\
│           └── InvestorControllerIntegrationTest.java
│
├── product-service\                           ← Python / FastAPI / Hexagonal
│   └── src\
│       ├── domain\
│       │   ├── model\
│       │   │   └── investment_product.py      ← Aggregate + Value Objects
│       │   └── port\
│       │       └── product_repository_port.py ← Output Port (ABC)
│       ├── application\
│       │   └── product_use_cases.py           ← Use Case Impl
│       ├── adapter\
│       │   ├── in_rest\
│       │   │   └── product_router.py          ← FastAPI Router (Primary Adapter)
│       │   └── out_persistence\
│       │       └── postgres_product_adapter.py ← SQLAlchemy Adapter
│       └── main.py                            ← FastAPI App Entry Point
│   ├── migrations\                            ← Alembic migrations
│   │   ├── env.py
│   │   └── versions\
│   │       └── 001_create_products_table.py
│   ├── tests\
│   │   └── test_product_use_cases.py
│   ├── alembic.ini
│   └── requirements.txt
│
├── gateway\nginx\
│   └── nginx.conf                             ← API Gateway routing rules
│
├── contracts\openapi\
│   ├── investor-api-v1.yaml                   ← Investor Service OpenAPI spec
│   └── product-api-v1.yaml                    ← Product Service OpenAPI spec
│
├── infra\terraform\
│   ├── providers.tf
│   ├── variables.tf
│   ├── main.tf
│   └── outputs.tf
│
├── docker\
│   ├── Dockerfile.investor                    ← Java multi-stage build
│   ├── Dockerfile.product                     ← Python slim build
│   └── docker-compose.yml                     ← Full stack
│
├── scripts\
│   ├── setup.ps1
│   ├── demo.ps1
│   ├── verify.ps1
│   └── teardown.ps1
│
└── .github\workflows\
    └── ci.yml
```

---

## L3. COMPLETE FILE CONTENTS

Create every file exactly as shown. Start from the project root: `$HOME\projects\nivesh-gateway-ddd`

---

### L3.1 OpenAPI Contracts — Write First (API-First Principle)

The contracts are written before a single line of service code. This is the API-First principle in practice.

**File: contracts\openapi\investor-api-v1.yaml**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"

@'
openapi: 3.0.3
info:
  title: Nivesh Gateway — Investor Profile API
  description: |
    Manages the Investor bounded context for the Nivesh Gateway cross-border
    investment platform. Handles investor identity, KYC status, risk profile,
    and eligible product discovery.

    Compliance: DPDP Act 2023 (India) | PDPA Singapore | MAS TRM
  version: 1.0.0
  contact:
    name: Nivesh Gateway Architecture Team

servers:
  - url: http://localhost:8080
    description: Local development
  - url: http://localhost:8090/api/investors
    description: Via API Gateway (local)

paths:
  /api/investors:
    post:
      operationId: createInvestor
      summary: Onboard a new investor
      tags: [Investors]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateInvestorRequest'
      responses:
        '201':
          description: Investor created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvestorResponse'
        '400':
          $ref: '#/components/responses/ValidationError'
        '409':
          $ref: '#/components/responses/Conflict'

    get:
      operationId: listInvestors
      summary: List all investors
      tags: [Investors]
      parameters:
        - name: kycStatus
          in: query
          schema:
            $ref: '#/components/schemas/KycStatus'
        - name: riskProfile
          in: query
          schema:
            $ref: '#/components/schemas/RiskProfile'
      responses:
        '200':
          description: Investor list
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/InvestorResponse'

  /api/investors/{investorId}:
    get:
      operationId: getInvestor
      summary: Get investor by ID
      tags: [Investors]
      parameters:
        - name: investorId
          in: path
          required: true
          schema:
            type: string
            pattern: '^INV-[0-9]{6}$'
      responses:
        '200':
          description: Investor found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvestorResponse'
        '404':
          $ref: '#/components/responses/NotFound'

  /api/investors/{investorId}/kyc:
    patch:
      operationId: updateKycStatus
      summary: Update KYC verification status
      tags: [Investors]
      parameters:
        - name: investorId
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/KycUpdateRequest'
      responses:
        '200':
          description: KYC status updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvestorResponse'
        '404':
          $ref: '#/components/responses/NotFound'

  /api/investors/{investorId}/eligible-products:
    get:
      operationId: getEligibleProducts
      summary: Get investment products eligible for this investor
      tags: [Investors, Cross-Context]
      parameters:
        - name: investorId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Eligible product list from Product bounded context
          content:
            application/json:
              schema:
                type: object
                properties:
                  investorId:
                    type: string
                  riskProfile:
                    $ref: '#/components/schemas/RiskProfile'
                  eligibleProducts:
                    type: array
                    items:
                      type: object
        '404':
          $ref: '#/components/responses/NotFound'
        '503':
          description: Product Catalogue Service unavailable

components:
  schemas:
    CreateInvestorRequest:
      type: object
      required: [investorId, fullName, email, nationality, riskProfile]
      properties:
        investorId:
          type: string
          pattern: '^INV-[0-9]{6}$'
          example: INV-000001
        fullName:
          type: string
          minLength: 2
          maxLength: 200
          example: Priya Krishnamurthy
        email:
          type: string
          format: email
          example: priya.k@example.com
        nationality:
          type: string
          enum: [INDIAN, SINGAPOREAN, OTHER]
          example: INDIAN
        riskProfile:
          $ref: '#/components/schemas/RiskProfile'

    KycUpdateRequest:
      type: object
      required: [kycStatus]
      properties:
        kycStatus:
          $ref: '#/components/schemas/KycStatus'

    InvestorResponse:
      type: object
      properties:
        id:
          type: integer
        investorId:
          type: string
        fullName:
          type: string
        email:
          type: string
        nationality:
          type: string
        kycStatus:
          $ref: '#/components/schemas/KycStatus'
        riskProfile:
          $ref: '#/components/schemas/RiskProfile'
        createdAt:
          type: string
          format: date-time

    KycStatus:
      type: string
      enum: [PENDING, IN_PROGRESS, VERIFIED, REJECTED, EXPIRED]

    RiskProfile:
      type: string
      enum: [CONSERVATIVE, MODERATE, AGGRESSIVE]

  responses:
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            type: object
            properties:
              status: { type: integer }
              error: { type: string }
              fieldErrors: { type: object }

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            type: object
            properties:
              status: { type: integer }
              message: { type: string }

    Conflict:
      description: Resource already exists
      content:
        application/json:
          schema:
            type: object
            properties:
              status: { type: integer }
              message: { type: string }
'@ | Set-Content -Path "contracts\openapi\investor-api-v1.yaml" -Encoding UTF8
```

**File: contracts\openapi\product-api-v1.yaml**

```powershell
@'
openapi: 3.0.3
info:
  title: Nivesh Gateway — Investment Product Catalogue API
  description: |
    Manages the Product bounded context for the Nivesh Gateway platform.
    Handles Singapore-regulated investment product listings, risk classifications,
    and investor eligibility criteria.

    Compliance: MAS Capital Markets Services | Singapore Securities Act
  version: 1.0.0

servers:
  - url: http://localhost:8000
    description: Local development
  - url: http://localhost:8090/api/products
    description: Via API Gateway (local)

paths:
  /api/products:
    post:
      operationId: createProduct
      summary: Register a new investment product
      tags: [Products]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProductRequest'
      responses:
        '201':
          description: Product created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '400':
          $ref: '#/components/responses/ValidationError'
        '409':
          $ref: '#/components/responses/Conflict'

    get:
      operationId: listProducts
      summary: List products, optionally filtered by risk classification
      tags: [Products]
      parameters:
        - name: riskProfile
          in: query
          schema:
            type: string
            enum: [CONSERVATIVE, MODERATE, AGGRESSIVE]
          description: Filter by investor risk profile compatibility
        - name: minRoi
          in: query
          schema:
            type: number
      responses:
        '200':
          description: Product list
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/ProductResponse'

  /api/products/{productCode}:
    get:
      operationId: getProduct
      summary: Get product by code
      tags: [Products]
      parameters:
        - name: productCode
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Product found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    CreateProductRequest:
      type: object
      required: [productCode, productName, riskClassification, expectedRoiPercent, currency]
      properties:
        productCode:
          type: string
          pattern: '^PRD-[A-Z0-9]{6}$'
          example: PRD-SG0001
        productName:
          type: string
          example: Singapore Government Bond Fund
        riskClassification:
          type: string
          enum: [CONSERVATIVE, MODERATE, AGGRESSIVE]
        expectedRoiPercent:
          type: number
          minimum: 0
          maximum: 100
          example: 4.5
        currency:
          type: string
          enum: [SGD, USD, INR]
          example: SGD
        minimumInvestmentSgd:
          type: number
          example: 1000.00
        masRegulated:
          type: boolean
          example: true

    ProductResponse:
      type: object
      properties:
        id:
          type: integer
        productCode:
          type: string
        productName:
          type: string
        riskClassification:
          type: string
        expectedRoiPercent:
          type: number
        currency:
          type: string
        minimumInvestmentSgd:
          type: number
        masRegulated:
          type: boolean
        createdAt:
          type: string
          format: date-time

  responses:
    ValidationError:
      description: Validation failed
    NotFound:
      description: Not found
    Conflict:
      description: Already exists
'@ | Set-Content -Path "contracts\openapi\product-api-v1.yaml" -Encoding UTF8
```

---

### L3.2 Investor Service — Java / Spring Boot / Hexagonal Architecture

**File: investor-service\pom.xml**

```powershell
@'
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
           http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.nivesh</groupId>
  <artifactId>investor-service</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
    <relativePath/>
  </parent>

  <properties>
    <java.version>17</java.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <testcontainers.version>1.19.8</testcontainers.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.flywaydb</groupId>
      <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <scope>runtime</scope>
    </dependency>
    <!-- WHY WebClient instead of RestTemplate:
         RestTemplate is in maintenance mode. WebClient supports both
         synchronous and async calls and is the Spring 6 standard. -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>${testcontainers.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.testcontainers</groupId>
      <artifactId>postgresql</artifactId>
      <version>${testcontainers.version}</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.projectreactor</groupId>
      <artifactId>reactor-test</artifactId>
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
'@ | Set-Content -Path "investor-service\pom.xml" -Encoding UTF8
```

**File: investor-service\src\main\resources\application.yml**

```powershell
@'
server:
  port: 8080

spring:
  application:
    name: investor-service

  datasource:
    url: jdbc:postgresql://postgres:5432/investor_db
    username: nivesh_user
    password: nivesh_pass
    driver-class-name: org.postgresql.Driver
    hikari:
      connection-timeout: 20000
      maximum-pool-size: 5
      pool-name: InvestorHikariPool

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always

# WHY external service URL as configuration:
# The Product Catalogue URL is injected from environment/config,
# not hardcoded in business logic. This is the hexagonal adapter principle --
# the domain does not know where the external service runs.
product-catalogue:
  base-url: http://product-service:8000

logging:
  level:
    com.nivesh: DEBUG
    org.springframework: INFO
'@ | Set-Content -Path "investor-service\src\main\resources\application.yml" -Encoding UTF8
```

**File: investor-service\src\main\resources\db\migration\V1__create_investor_tables.sql**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\resources\db\migration" -Force | Out-Null

@'
-- V1__create_investor_tables.sql
-- WHY separate investor_db schema:
-- Bounded contexts must not share a database. The investor_db schema belongs
-- exclusively to the Investor Service. No other service has credentials
-- to this database. This enforces the bounded context isolation at the
-- infrastructure level, not just at the application level.

CREATE TABLE IF NOT EXISTS investors (
    id               BIGSERIAL    PRIMARY KEY,
    investor_id      VARCHAR(20)  NOT NULL UNIQUE,
    full_name        VARCHAR(200) NOT NULL,
    email            VARCHAR(255) NOT NULL UNIQUE,
    nationality      VARCHAR(30)  NOT NULL
                         CHECK (nationality IN (''INDIAN'', ''SINGAPOREAN'', ''OTHER'')),
    kyc_status       VARCHAR(20)  NOT NULL DEFAULT ''PENDING''
                         CHECK (kyc_status IN (''PENDING'', ''IN_PROGRESS'', ''VERIFIED'', ''REJECTED'', ''EXPIRED'')),
    risk_profile     VARCHAR(20)  NOT NULL
                         CHECK (risk_profile IN (''CONSERVATIVE'', ''MODERATE'', ''AGGRESSIVE'')),
    created_at       TIMESTAMP    NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_investor_id   ON investors(investor_id);
CREATE UNIQUE INDEX IF NOT EXISTS idx_investor_email ON investors(email);
CREATE INDEX        IF NOT EXISTS idx_kyc_status    ON investors(kyc_status);
CREATE INDEX        IF NOT EXISTS idx_risk_profile  ON investors(risk_profile);

COMMENT ON TABLE investors IS
    ''Investor aggregate root for Nivesh Gateway Investor bounded context'';
COMMENT ON COLUMN investors.kyc_status IS
    ''KYC lifecycle: PENDING -> IN_PROGRESS -> VERIFIED | REJECTED. VERIFIED required for trading.'';
'@ | Set-Content -Path "investor-service\src\main\resources\db\migration\V1__create_investor_tables.sql" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\model\KycStatus.java**

```powershell
@'
package com.nivesh.investor.domain.model;

/**
 * WHY an enum and not a String:
 * KycStatus is a Value Object with a finite set of valid states.
 * Using an enum makes invalid states unrepresentable in the domain model.
 * A String field could hold "VERIFEID" (typo) -- this cannot.
 *
 * WHY these specific states:
 * These mirror the RBI KYC lifecycle and MAS CDD requirements.
 * EXPIRED is added for the India context where KYC has a validity period.
 */
public enum KycStatus {
    PENDING,
    IN_PROGRESS,
    VERIFIED,
    REJECTED,
    EXPIRED;

    public boolean allowsTrading() {
        return this == VERIFIED;
    }

    public boolean requiresRenewal() {
        return this == EXPIRED;
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\model\KycStatus.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\model\RiskProfile.java**

```powershell
@'
package com.nivesh.investor.domain.model;

/**
 * WHY RiskProfile as a Value Object:
 * Risk profile is not just a label -- it carries eligibility rules.
 * The isEligibleFor method encodes the business rule about which profiles
 * can access which risk classifications. This rule lives in the domain,
 * not in a controller or a database query.
 *
 * In DDD terms: business rules that belong to the domain must live in
 * the domain model, not leak into application services or adapters.
 */
public enum RiskProfile {
    CONSERVATIVE,
    MODERATE,
    AGGRESSIVE;

    /**
     * Determines if this investor risk profile is eligible for a
     * product with the given risk classification.
     * Business rule: investors can access products at or below their risk tolerance.
     */
    public boolean isEligibleFor(final String productRiskClassification) {
        return switch (this) {
            case CONSERVATIVE -> productRiskClassification.equals("CONSERVATIVE");
            case MODERATE     -> productRiskClassification.equals("CONSERVATIVE")
                              || productRiskClassification.equals("MODERATE");
            case AGGRESSIVE   -> true; // Aggressive investors can access all risk levels
        };
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\model\RiskProfile.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\model\Investor.java**

```powershell
@'
package com.nivesh.investor.domain.model;

import java.time.LocalDateTime;

/**
 * WHY Investor is the Aggregate Root:
 *
 * The Investor aggregate is the consistency boundary for all investor-related
 * state changes. KycStatus and RiskProfile are part of this aggregate --
 * they cannot be changed independently of the Investor aggregate root.
 *
 * WHY this is a plain Java object (not a JPA entity):
 * In hexagonal architecture, the domain model is framework-agnostic.
 * JPA annotations belong in the persistence adapter (InvestorJpaEntity),
 * not in the domain. This keeps the domain model testable without Spring.
 *
 * WHY the business methods updateKycStatus and canAccessProducts:
 * Domain behaviour lives on the aggregate, not in services or controllers.
 * This is DDD's "tell, don't ask" principle.
 */
public class Investor {

    private Long id;
    private final String investorId;
    private final String fullName;
    private final String email;
    private final String nationality;
    private KycStatus kycStatus;
    private final RiskProfile riskProfile;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public Investor(final String investorId,
                    final String fullName,
                    final String email,
                    final String nationality,
                    final RiskProfile riskProfile) {
        this.investorId  = investorId;
        this.fullName    = fullName;
        this.email       = email;
        this.nationality = nationality;
        this.riskProfile = riskProfile;
        this.kycStatus   = KycStatus.PENDING;
        this.createdAt   = LocalDateTime.now();
        this.updatedAt   = LocalDateTime.now();
    }

    // Reconstitution constructor (used by persistence adapter)
    public Investor(final Long id,
                    final String investorId,
                    final String fullName,
                    final String email,
                    final String nationality,
                    final KycStatus kycStatus,
                    final RiskProfile riskProfile,
                    final LocalDateTime createdAt,
                    final LocalDateTime updatedAt) {
        this.id          = id;
        this.investorId  = investorId;
        this.fullName    = fullName;
        this.email       = email;
        this.nationality = nationality;
        this.kycStatus   = kycStatus;
        this.riskProfile = riskProfile;
        this.createdAt   = createdAt;
        this.updatedAt   = updatedAt;
    }

    /**
     * Domain behaviour: transition KYC status.
     * The aggregate enforces the lifecycle rule -- you cannot VERIFY
     * an already-EXPIRED investor without going through IN_PROGRESS.
     */
    public void updateKycStatus(final KycStatus newStatus) {
        if (this.kycStatus == KycStatus.VERIFIED && newStatus == KycStatus.PENDING) {
            throw new IllegalStateException(
                "Cannot revert a VERIFIED investor to PENDING. " +
                "Use EXPIRED or REJECTED status transitions instead."
            );
        }
        this.kycStatus  = newStatus;
        this.updatedAt  = LocalDateTime.now();
    }

    /**
     * Domain behaviour: eligibility check.
     * The investor knows their own risk profile and delegates to the
     * RiskProfile value object for the eligibility business rule.
     */
    public boolean canAccessProductWithRisk(final String productRiskClassification) {
        return kycStatus.allowsTrading()
            && riskProfile.isEligibleFor(productRiskClassification);
    }

    // Getters
    public Long getId()                  { return id; }
    public String getInvestorId()        { return investorId; }
    public String getFullName()          { return fullName; }
    public String getEmail()             { return email; }
    public String getNationality()       { return nationality; }
    public KycStatus getKycStatus()      { return kycStatus; }
    public RiskProfile getRiskProfile()  { return riskProfile; }
    public LocalDateTime getCreatedAt()  { return createdAt; }
    public LocalDateTime getUpdatedAt()  { return updatedAt; }
    public void setId(final Long id)     { this.id = id; }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\model\Investor.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\port\in\InvestorUseCase.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\domain\port\in" -Force | Out-Null

@'
package com.nivesh.investor.domain.port.in;

import com.nivesh.investor.domain.model.Investor;
import com.nivesh.investor.domain.model.KycStatus;

import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * WHY this interface exists as an Input Port:
 *
 * The Input Port defines what operations the domain exposes to the outside world.
 * The REST controller (primary adapter) depends on THIS interface, not on the
 * application service implementation directly. This means:
 * 1. The domain can be tested without Spring (use a mock or stub)
 * 2. The controller never knows how the use case is implemented
 * 3. The implementation can change without touching the controller
 *
 * This is the LEFT side of the hexagon.
 */
public interface InvestorUseCase {

    Investor createInvestor(String investorId, String fullName, String email,
                             String nationality, String riskProfile);

    Optional<Investor> findByInvestorId(String investorId);

    List<Investor> findAll();

    List<Investor> findByKycStatus(String kycStatus);

    List<Investor> findByRiskProfile(String riskProfile);

    Investor updateKycStatus(String investorId, KycStatus newStatus);

    /**
     * Cross-context operation: fetches eligible products for an investor.
     * The domain port defines the contract; the adapter calls the Product Service.
     * Returns a map containing investorId, riskProfile, and eligibleProducts.
     */
    Map<String, Object> getEligibleProducts(String investorId);
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\port\in\InvestorUseCase.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\port\out\InvestorRepository.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\domain\port\out" -Force | Out-Null

@'
package com.nivesh.investor.domain.port.out;

import com.nivesh.investor.domain.model.Investor;
import com.nivesh.investor.domain.model.KycStatus;
import com.nivesh.investor.domain.model.RiskProfile;

import java.util.List;
import java.util.Optional;

/**
 * WHY this interface exists as an Output Port:
 *
 * The Output Port defines what the domain needs from its persistence infrastructure.
 * The domain defines this interface; the infrastructure adapter implements it.
 * The domain never imports JPA, JDBC, or any persistence framework.
 *
 * This is the RIGHT side of the hexagon. The domain tells infrastructure
 * what it needs. Infrastructure does not dictate to the domain.
 */
public interface InvestorRepository {

    Investor save(Investor investor);

    Optional<Investor> findByInvestorId(String investorId);

    List<Investor> findAll();

    List<Investor> findByKycStatus(KycStatus kycStatus);

    List<Investor> findByRiskProfile(RiskProfile riskProfile);

    boolean existsByInvestorId(String investorId);

    boolean existsByEmail(String email);
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\port\out\InvestorRepository.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\domain\port\out\ProductCataloguePort.java**

```powershell
@'
package com.nivesh.investor.domain.port.out;

import java.util.List;
import java.util.Map;

/**
 * WHY a separate port for the Product Catalogue:
 *
 * The Investor domain needs product data to answer "what can this investor buy?"
 * But the Investor domain must NOT know that the product data comes from an
 * HTTP call to a separate service. It just defines what it needs.
 *
 * This port is the Anti-Corruption Layer (ACL) boundary. The adapter
 * (ProductCatalogueAdapter) translates between the external Product Service
 * API response and the domain's needs. If the Product Service changes its
 * API format, only the adapter changes -- the domain is untouched.
 *
 * This is one of the most important patterns in bounded context integration.
 */
public interface ProductCataloguePort {

    /**
     * Fetch products compatible with the given investor risk profile.
     * Returns a list of product maps -- the domain does not import Product classes.
     */
    List<Map<String, Object>> findProductsByRiskProfile(String riskProfile);
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\domain\port\out\ProductCataloguePort.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\application\InvestorApplicationService.java**

```powershell
@'
package com.nivesh.investor.application;

import com.nivesh.investor.domain.model.Investor;
import com.nivesh.investor.domain.model.KycStatus;
import com.nivesh.investor.domain.model.RiskProfile;
import com.nivesh.investor.domain.port.in.InvestorUseCase;
import com.nivesh.investor.domain.port.out.InvestorRepository;
import com.nivesh.investor.domain.port.out.ProductCataloguePort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

/**
 * WHY this class lives in the application layer, not the domain:
 *
 * The application service orchestrates use cases. It:
 * 1. Validates business-level pre-conditions (duplicate investor)
 * 2. Calls domain methods (investor.updateKycStatus)
 * 3. Delegates to output ports (repository, product catalogue)
 * 4. Does NOT contain business rules -- those live in the domain model
 *
 * This separation ensures domain rules are testable without Spring context
 * and the application service is testable by mocking ports.
 */
@Service
@Transactional
public class InvestorApplicationService implements InvestorUseCase {

    private final InvestorRepository investorRepository;
    private final ProductCataloguePort productCataloguePort;

    public InvestorApplicationService(
            final InvestorRepository investorRepository,
            final ProductCataloguePort productCataloguePort) {
        this.investorRepository   = investorRepository;
        this.productCataloguePort = productCataloguePort;
    }

    @Override
    public Investor createInvestor(final String investorId,
                                    final String fullName,
                                    final String email,
                                    final String nationality,
                                    final String riskProfile) {
        if (investorRepository.existsByInvestorId(investorId)) {
            throw new InvestorAlreadyExistsException(
                "Investor ID " + investorId + " already exists in the registry."
            );
        }
        if (investorRepository.existsByEmail(email)) {
            throw new InvestorAlreadyExistsException(
                "An investor with email " + email + " is already registered."
            );
        }
        var investor = new Investor(
            investorId, fullName, email, nationality,
            RiskProfile.valueOf(riskProfile)
        );
        return investorRepository.save(investor);
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<Investor> findByInvestorId(final String investorId) {
        return investorRepository.findByInvestorId(investorId);
    }

    @Override
    @Transactional(readOnly = true)
    public List<Investor> findAll() {
        return investorRepository.findAll();
    }

    @Override
    @Transactional(readOnly = true)
    public List<Investor> findByKycStatus(final String kycStatus) {
        return investorRepository.findByKycStatus(KycStatus.valueOf(kycStatus));
    }

    @Override
    @Transactional(readOnly = true)
    public List<Investor> findByRiskProfile(final String riskProfile) {
        return investorRepository.findByRiskProfile(RiskProfile.valueOf(riskProfile));
    }

    @Override
    public Investor updateKycStatus(final String investorId, final KycStatus newStatus) {
        var investor = investorRepository.findByInvestorId(investorId)
            .orElseThrow(() -> new InvestorNotFoundException(
                "Investor " + investorId + " not found."
            ));
        investor.updateKycStatus(newStatus);
        return investorRepository.save(investor);
    }

    @Override
    @Transactional(readOnly = true)
    public Map<String, Object> getEligibleProducts(final String investorId) {
        var investor = investorRepository.findByInvestorId(investorId)
            .orElseThrow(() -> new InvestorNotFoundException(
                "Investor " + investorId + " not found."
            ));

        // Fetch all products for investor's risk profile from Product Catalogue (via ACL)
        var products = productCataloguePort.findProductsByRiskProfile(
            investor.getRiskProfile().name()
        );

        // Apply investor-side eligibility filter using domain logic
        var eligible = products.stream()
            .filter(p -> investor.canAccessProductWithRisk(
                String.valueOf(p.get("riskClassification"))
            ))
            .toList();

        var result = new LinkedHashMap<String, Object>();
        result.put("investorId",       investor.getInvestorId());
        result.put("fullName",         investor.getFullName());
        result.put("kycStatus",        investor.getKycStatus().name());
        result.put("riskProfile",      investor.getRiskProfile().name());
        result.put("eligibleProducts", eligible);
        result.put("productCount",     eligible.size());
        return result;
    }

    // Domain exception classes -- live in application layer as they are use-case concerns
    public static class InvestorNotFoundException extends RuntimeException {
        public InvestorNotFoundException(final String message) { super(message); }
    }

    public static class InvestorAlreadyExistsException extends RuntimeException {
        public InvestorAlreadyExistsException(final String message) { super(message); }
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\application\InvestorApplicationService.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorJpaEntity.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence" -Force | Out-Null

@'
package com.nivesh.investor.adapter.out.persistence;

import jakarta.persistence.*;
import java.time.LocalDateTime;

/**
 * WHY a separate JPA entity and not annotating the domain Investor class:
 *
 * The domain Investor class must be framework-agnostic. Annotating it with
 * JPA annotations (@Entity, @Column, @GeneratedValue) would make the domain
 * depend on Jakarta Persistence -- a persistence framework concern.
 *
 * InvestorJpaEntity is an infrastructure detail. It lives in the persistence
 * adapter. It maps to the database schema. The adapter translates between
 * InvestorJpaEntity and the domain Investor class.
 *
 * This pattern is called the Repository Translation pattern in hexagonal architecture.
 */
@Entity
@Table(name = "investors")
public class InvestorJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "investor_id", unique = true, nullable = false, updatable = false)
    private String investorId;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String nationality;

    @Column(name = "kyc_status", nullable = false)
    private String kycStatus;

    @Column(name = "risk_profile", nullable = false, updatable = false)
    private String riskProfile;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }

    protected InvestorJpaEntity() {}

    // Getters and setters
    public Long getId()                           { return id; }
    public void setId(Long id)                   { this.id = id; }
    public String getInvestorId()                { return investorId; }
    public void setInvestorId(String investorId) { this.investorId = investorId; }
    public String getFullName()                  { return fullName; }
    public void setFullName(String fullName)     { this.fullName = fullName; }
    public String getEmail()                     { return email; }
    public void setEmail(String email)           { this.email = email; }
    public String getNationality()               { return nationality; }
    public void setNationality(String nat)       { this.nationality = nat; }
    public String getKycStatus()                 { return kycStatus; }
    public void setKycStatus(String kycStatus)   { this.kycStatus = kycStatus; }
    public String getRiskProfile()               { return riskProfile; }
    public void setRiskProfile(String rp)        { this.riskProfile = rp; }
    public LocalDateTime getCreatedAt()          { return createdAt; }
    public void setCreatedAt(LocalDateTime t)    { this.createdAt = t; }
    public LocalDateTime getUpdatedAt()          { return updatedAt; }
    public void setUpdatedAt(LocalDateTime t)    { this.updatedAt = t; }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorJpaEntity.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorJpaRepository.java**

```powershell
@'
package com.nivesh.investor.adapter.out.persistence;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
interface InvestorJpaRepository extends JpaRepository<InvestorJpaEntity, Long> {
    Optional<InvestorJpaEntity> findByInvestorId(String investorId);
    List<InvestorJpaEntity> findByKycStatus(String kycStatus);
    List<InvestorJpaEntity> findByRiskProfile(String riskProfile);
    boolean existsByInvestorId(String investorId);
    boolean existsByEmail(String email);
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorJpaRepository.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorPersistenceAdapter.java**

```powershell
@'
package com.nivesh.investor.adapter.out.persistence;

import com.nivesh.investor.domain.model.Investor;
import com.nivesh.investor.domain.model.KycStatus;
import com.nivesh.investor.domain.model.RiskProfile;
import com.nivesh.investor.domain.port.out.InvestorRepository;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Optional;

/**
 * WHY this adapter exists as a separate class:
 *
 * The persistence adapter implements the domain's InvestorRepository output port.
 * It translates between:
 *   - InvestorJpaEntity (infrastructure model with JPA annotations)
 *   - Investor (domain model, framework-agnostic)
 *
 * The translation (toEntity, toDomain) isolates database concerns from domain logic.
 * If the database schema changes, only this adapter changes. The domain is untouched.
 */
@Component
public class InvestorPersistenceAdapter implements InvestorRepository {

    private final InvestorJpaRepository jpaRepository;

    public InvestorPersistenceAdapter(final InvestorJpaRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public Investor save(final Investor investor) {
        var entity = toEntity(investor);
        var saved  = jpaRepository.save(entity);
        investor.setId(saved.getId());
        return toDomain(saved);
    }

    @Override
    public Optional<Investor> findByInvestorId(final String investorId) {
        return jpaRepository.findByInvestorId(investorId).map(this::toDomain);
    }

    @Override
    public List<Investor> findAll() {
        return jpaRepository.findAll().stream().map(this::toDomain).toList();
    }

    @Override
    public List<Investor> findByKycStatus(final KycStatus kycStatus) {
        return jpaRepository.findByKycStatus(kycStatus.name())
                            .stream().map(this::toDomain).toList();
    }

    @Override
    public List<Investor> findByRiskProfile(final RiskProfile riskProfile) {
        return jpaRepository.findByRiskProfile(riskProfile.name())
                            .stream().map(this::toDomain).toList();
    }

    @Override
    public boolean existsByInvestorId(final String investorId) {
        return jpaRepository.existsByInvestorId(investorId);
    }

    @Override
    public boolean existsByEmail(final String email) {
        return jpaRepository.existsByEmail(email);
    }

    private InvestorJpaEntity toEntity(final Investor investor) {
        var entity = new InvestorJpaEntity();
        entity.setId(investor.getId());
        entity.setInvestorId(investor.getInvestorId());
        entity.setFullName(investor.getFullName());
        entity.setEmail(investor.getEmail());
        entity.setNationality(investor.getNationality());
        entity.setKycStatus(investor.getKycStatus().name());
        entity.setRiskProfile(investor.getRiskProfile().name());
        entity.setCreatedAt(investor.getCreatedAt());
        entity.setUpdatedAt(investor.getUpdatedAt());
        return entity;
    }

    private Investor toDomain(final InvestorJpaEntity entity) {
        return new Investor(
            entity.getId(),
            entity.getInvestorId(),
            entity.getFullName(),
            entity.getEmail(),
            entity.getNationality(),
            KycStatus.valueOf(entity.getKycStatus()),
            RiskProfile.valueOf(entity.getRiskProfile()),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\persistence\InvestorPersistenceAdapter.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\out\external\ProductCatalogueAdapter.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\external" -Force | Out-Null

@'
package com.nivesh.investor.adapter.out.external;

import com.nivesh.investor.domain.port.out.ProductCataloguePort;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;

import java.time.Duration;
import java.util.Collections;
import java.util.List;
import java.util.Map;

/**
 * WHY this is an Anti-Corruption Layer (ACL) adapter:
 *
 * The Product Catalogue Service is a separate bounded context with its own
 * domain model. This adapter translates between:
 *   - The external HTTP API response format (Product Service language)
 *   - The domain concept the Investor Service needs (just product data as maps)
 *
 * WHY we return List<Map<String, Object>> and not a ProductDto class:
 * The Investor domain must not import types from the Product bounded context.
 * Using generic maps keeps the boundary clean. In a real system, a shared
 * kernel or published language DTO might be justified -- but that is a
 * deliberate architectural decision requiring an ADR.
 *
 * WHY circuit breaker / timeout here and not elsewhere:
 * External calls must be isolated. A slow Product Service must not block
 * the Investor Service. The timeout and fallback are the ACL's responsibility.
 */
@Component
public class ProductCatalogueAdapter implements ProductCataloguePort {

    private static final Logger log = LoggerFactory.getLogger(ProductCatalogueAdapter.class);
    private static final Duration TIMEOUT = Duration.ofSeconds(5);

    private final WebClient webClient;

    public ProductCatalogueAdapter(
            @Value("${product-catalogue.base-url}") final String productCatalogueBaseUrl) {
        this.webClient = WebClient.builder()
            .baseUrl(productCatalogueBaseUrl)
            .build();
    }

    @Override
    @SuppressWarnings("unchecked")
    public List<Map<String, Object>> findProductsByRiskProfile(final String riskProfile) {
        try {
            var result = webClient.get()
                .uri(uriBuilder -> uriBuilder
                    .path("/api/products")
                    .queryParam("riskProfile", riskProfile)
                    .build())
                .retrieve()
                .bodyToFlux(Map.class)
                .cast((Class<Map<String, Object>>) (Class<?>) Map.class)
                .timeout(TIMEOUT)
                .collectList()
                .block();
            return result != null ? result : Collections.emptyList();
        } catch (Exception ex) {
            // WHY we return empty and log instead of throwing:
            // The Product Catalogue is a supporting context. If it is
            // temporarily unavailable, the Investor Service degrades gracefully --
            // it returns an empty eligible product list rather than failing completely.
            // This is the circuit-breaker principle applied at the ACL boundary.
            log.warn("Product Catalogue unreachable for riskProfile={}: {}",
                     riskProfile, ex.getMessage());
            return Collections.emptyList();
        }
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\out\external\ProductCatalogueAdapter.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto\CreateInvestorRequest.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto" -Force | Out-Null

@'
package com.nivesh.investor.adapter.in.rest.dto;

import jakarta.validation.constraints.*;

/**
 * WHY a DTO and not the domain Investor class directly:
 *
 * The REST adapter receives external data in the API protocol format.
 * The domain Investor class is an aggregate with business state.
 * Mixing them creates coupling: API changes break the domain, or domain
 * changes alter the API contract unexpectedly.
 *
 * The DTO is the API contract representation. It validates API-level
 * constraints. The application service converts it to domain objects.
 */
public record CreateInvestorRequest(
    @NotBlank(message = "investorId is required")
    @Pattern(regexp = "^INV-[0-9]{6}$",
             message = "investorId must match format INV-NNNNNN (e.g. INV-000001)")
    String investorId,

    @NotBlank(message = "fullName is required")
    @Size(min = 2, max = 200, message = "fullName must be between 2 and 200 characters")
    String fullName,

    @NotBlank(message = "email is required")
    @Email(message = "email must be a valid email address")
    String email,

    @NotBlank(message = "nationality is required")
    @Pattern(regexp = "INDIAN|SINGAPOREAN|OTHER",
             message = "nationality must be INDIAN, SINGAPOREAN, or OTHER")
    String nationality,

    @NotBlank(message = "riskProfile is required")
    @Pattern(regexp = "CONSERVATIVE|MODERATE|AGGRESSIVE",
             message = "riskProfile must be CONSERVATIVE, MODERATE, or AGGRESSIVE")
    String riskProfile
) {}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto\CreateInvestorRequest.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto\InvestorResponse.java**

```powershell
@'
package com.nivesh.investor.adapter.in.rest.dto;

import com.nivesh.investor.domain.model.Investor;
import java.time.LocalDateTime;

/**
 * WHY a dedicated response record:
 * The response contract must be stable and explicitly defined.
 * It prevents internal domain fields from accidentally leaking into the API
 * (e.g., internal IDs, audit fields, sensitive computed properties).
 */
public record InvestorResponse(
    Long id,
    String investorId,
    String fullName,
    String email,
    String nationality,
    String kycStatus,
    String riskProfile,
    boolean tradingAllowed,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {
    public static InvestorResponse from(final Investor investor) {
        return new InvestorResponse(
            investor.getId(),
            investor.getInvestorId(),
            investor.getFullName(),
            investor.getEmail(),
            investor.getNationality(),
            investor.getKycStatus().name(),
            investor.getRiskProfile().name(),
            investor.getKycStatus().allowsTrading(),
            investor.getCreatedAt(),
            investor.getUpdatedAt()
        );
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\dto\InvestorResponse.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\InvestorController.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest" -Force | Out-Null

@'
package com.nivesh.investor.adapter.in.rest;

import com.nivesh.investor.adapter.in.rest.dto.CreateInvestorRequest;
import com.nivesh.investor.adapter.in.rest.dto.InvestorResponse;
import com.nivesh.investor.domain.model.KycStatus;
import com.nivesh.investor.domain.port.in.InvestorUseCase;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

/**
 * WHY the controller depends on InvestorUseCase (interface) and not InvestorApplicationService:
 *
 * This is the hexagonal architecture principle in practice.
 * The primary adapter (controller) depends on the INPUT PORT (interface).
 * It never imports the application service implementation.
 * This means the controller can be tested by injecting a mock InvestorUseCase
 * without loading Spring context or the database.
 */
@RestController
@RequestMapping("/api/investors")
public class InvestorController {

    private final InvestorUseCase investorUseCase;

    public InvestorController(final InvestorUseCase investorUseCase) {
        this.investorUseCase = investorUseCase;
    }

    @PostMapping
    public ResponseEntity<InvestorResponse> create(
            @Valid @RequestBody final CreateInvestorRequest request) {
        var investor = investorUseCase.createInvestor(
            request.investorId(), request.fullName(), request.email(),
            request.nationality(), request.riskProfile()
        );
        return ResponseEntity.status(HttpStatus.CREATED)
                             .body(InvestorResponse.from(investor));
    }

    @GetMapping
    public ResponseEntity<List<InvestorResponse>> findAll(
            @RequestParam(required = false) final String kycStatus,
            @RequestParam(required = false) final String riskProfile) {
        List<InvestorResponse> result;
        if (kycStatus != null) {
            result = investorUseCase.findByKycStatus(kycStatus)
                                    .stream().map(InvestorResponse::from).toList();
        } else if (riskProfile != null) {
            result = investorUseCase.findByRiskProfile(riskProfile)
                                    .stream().map(InvestorResponse::from).toList();
        } else {
            result = investorUseCase.findAll()
                                    .stream().map(InvestorResponse::from).toList();
        }
        return ResponseEntity.ok(result);
    }

    @GetMapping("/{investorId}")
    public ResponseEntity<InvestorResponse> findById(@PathVariable final String investorId) {
        return investorUseCase.findByInvestorId(investorId)
            .map(investor -> ResponseEntity.ok(InvestorResponse.from(investor)))
            .orElse(ResponseEntity.notFound().build());
    }

    @PatchMapping("/{investorId}/kyc")
    public ResponseEntity<InvestorResponse> updateKyc(
            @PathVariable final String investorId,
            @RequestBody final Map<String, String> body) {
        var status   = KycStatus.valueOf(body.get("kycStatus"));
        var investor = investorUseCase.updateKycStatus(investorId, status);
        return ResponseEntity.ok(InvestorResponse.from(investor));
    }

    @GetMapping("/{investorId}/eligible-products")
    public ResponseEntity<Map<String, Object>> getEligibleProducts(
            @PathVariable final String investorId) {
        return ResponseEntity.ok(investorUseCase.getEligibleProducts(investorId));
    }

    @GetMapping("/health-check")
    public ResponseEntity<Map<String, String>> healthCheck() {
        return ResponseEntity.ok(Map.of("service", "investor-service", "status", "UP"));
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\InvestorController.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\GlobalExceptionHandler.java**

```powershell
@'
package com.nivesh.investor.adapter.in.rest;

import com.nivesh.investor.application.InvestorApplicationService.InvestorAlreadyExistsException;
import com.nivesh.investor.application.InvestorApplicationService.InvestorNotFoundException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            final MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = new HashMap<>();
        for (FieldError e : ex.getBindingResult().getFieldErrors()) {
            fieldErrors.put(e.getField(), e.getDefaultMessage());
        }
        return ResponseEntity.badRequest().body(Map.of(
            "timestamp",   LocalDateTime.now().toString(),
            "status",      400,
            "error",       "Validation Failed",
            "fieldErrors", fieldErrors
        ));
    }

    @ExceptionHandler(InvestorNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(
            final InvestorNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of(
            "timestamp", LocalDateTime.now().toString(),
            "status",    404,
            "error",     "Investor Not Found",
            "message",   ex.getMessage()
        ));
    }

    @ExceptionHandler(InvestorAlreadyExistsException.class)
    public ResponseEntity<Map<String, Object>> handleConflict(
            final InvestorAlreadyExistsException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of(
            "timestamp", LocalDateTime.now().toString(),
            "status",    409,
            "error",     "Investor Already Exists",
            "message",   ex.getMessage()
        ));
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, Object>> handleIllegalArg(
            final IllegalArgumentException ex) {
        return ResponseEntity.badRequest().body(Map.of(
            "timestamp", LocalDateTime.now().toString(),
            "status",    400,
            "error",     "Invalid Input",
            "message",   ex.getMessage()
        ));
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<Map<String, Object>> handleIllegalState(
            final IllegalStateException ex) {
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(Map.of(
            "timestamp", LocalDateTime.now().toString(),
            "status",    422,
            "error",     "Invalid State Transition",
            "message",   ex.getMessage()
        ));
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\adapter\in\rest\GlobalExceptionHandler.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\config\AppConfig.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\main\java\com\nivesh\investor\config" -Force | Out-Null

@'
package com.nivesh.investor.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class AppConfig {

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\config\AppConfig.java" -Encoding UTF8
```

**File: investor-service\src\main\java\com\nivesh\investor\InvestorServiceApp.java**

```powershell
@'
package com.nivesh.investor;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class InvestorServiceApp {
    public static void main(final String[] args) {
        SpringApplication.run(InvestorServiceApp.class, args);
    }
}
'@ | Set-Content -Path "investor-service\src\main\java\com\nivesh\investor\InvestorServiceApp.java" -Encoding UTF8
```

**File: investor-service\src\test\java\com\nivesh\investor\InvestorControllerIntegrationTest.java**

```powershell
New-Item -ItemType Directory -Path "investor-service\src\test\java\com\nivesh\investor" -Force | Out-Null

@'
package com.nivesh.investor;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.Map;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class InvestorControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("investor_test")
        .withUsername("test_user")
        .withPassword("test_pass");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url",      postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
        // Use a stub URL for product catalogue -- it is not under test here
        r.add("product-catalogue.base-url", () -> "http://localhost:19999");
    }

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper mapper;

    private Map<String, String> validInvestorPayload(String investorId) {
        return Map.of(
            "investorId",  investorId,
            "fullName",    "Priya Krishnamurthy",
            "email",       investorId.toLowerCase().replace("-","") + "@test.com",
            "nationality", "INDIAN",
            "riskProfile", "MODERATE"
        );
    }

    @Test
    void createInvestor_validPayload_returns201() throws Exception {
        mockMvc.perform(post("/api/investors")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(validInvestorPayload("INV-000001"))))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.investorId").value("INV-000001"))
            .andExpect(jsonPath("$.kycStatus").value("PENDING"))
            .andExpect(jsonPath("$.tradingAllowed").value(false));
    }

    @Test
    void createInvestor_invalidIdFormat_returns400() throws Exception {
        var payload = Map.of(
            "investorId", "INVALID",
            "fullName", "Test", "email", "t@t.com",
            "nationality", "INDIAN", "riskProfile", "MODERATE"
        );
        mockMvc.perform(post("/api/investors")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(payload)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.investorId").exists());
    }

    @Test
    void createInvestor_duplicate_returns409() throws Exception {
        var payload = mapper.writeValueAsString(validInvestorPayload("INV-000002"));
        mockMvc.perform(post("/api/investors")
                .contentType(MediaType.APPLICATION_JSON).content(payload))
            .andExpect(status().isCreated());
        mockMvc.perform(post("/api/investors")
                .contentType(MediaType.APPLICATION_JSON).content(payload))
            .andExpect(status().isConflict());
    }

    @Test
    void updateKycStatus_toVerified_allowsTrading() throws Exception {
        mockMvc.perform(post("/api/investors")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(validInvestorPayload("INV-000003"))))
            .andExpect(status().isCreated());

        mockMvc.perform(patch("/api/investors/INV-000003/kyc")
                .contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(Map.of("kycStatus", "VERIFIED"))))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.kycStatus").value("VERIFIED"))
            .andExpect(jsonPath("$.tradingAllowed").value(true));
    }

    @Test
    void getInvestor_notFound_returns404() throws Exception {
        mockMvc.perform(get("/api/investors/INV-999999"))
            .andExpect(status().isNotFound());
    }

    @Test
    void listInvestors_filterByKycStatus_returnsList() throws Exception {
        mockMvc.perform(get("/api/investors?kycStatus=PENDING"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$").isArray());
    }

    @Test
    void healthCheck_returnsUp() throws Exception {
        mockMvc.perform(get("/api/investors/health-check"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("UP"));
    }
}
'@ | Set-Content -Path "investor-service\src\test\java\com\nivesh\investor\InvestorControllerIntegrationTest.java" -Encoding UTF8
```

---

### L3.3 Product Service — Python / FastAPI / Hexagonal Architecture

**File: product-service\requirements.txt**

```powershell
@'
fastapi==0.111.0
uvicorn==0.30.1
pydantic==2.7.4
sqlalchemy==2.0.30
asyncpg==0.29.0
alembic==1.13.1
psycopg2-binary==2.9.9
pytest==8.2.2
pytest-asyncio==0.23.7
httpx==0.27.0
'@ | Set-Content -Path "product-service\requirements.txt" -Encoding UTF8
```

**File: product-service\src\domain\model\investment_product.py**

```powershell
New-Item -ItemType Directory -Path "product-service\src\domain\model" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\domain\__init__.py" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\domain\model\__init__.py" -Force | Out-Null

@'
"""
investment_product.py -- Domain model for the Product bounded context.

WHY this module has no database imports, no FastAPI imports, no SQLAlchemy:
The domain model is framework-agnostic. It expresses business concepts and
rules in pure Python. This makes it testable without any infrastructure
and readable by a domain expert who does not know FastAPI or SQLAlchemy.

WHY RiskClassification is a separate class and not a raw string:
Value Objects encapsulate business rules about their own validity.
RiskClassification knows which risk tiers are compatible with each other.
That rule lives here in the domain, not in a database query or a controller.
"""

from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional


class RiskClassification(str, Enum):
    """
    Value Object: Risk classification for investment products.
    Mirrors the RiskProfile enum in the Investor bounded context
    through the shared language of the cross-context API contract.
    """
    CONSERVATIVE = "CONSERVATIVE"
    MODERATE     = "MODERATE"
    AGGRESSIVE   = "AGGRESSIVE"

    def is_suitable_for_investor_profile(self, investor_risk_profile: str) -> bool:
        """
        Domain rule: determine if this product risk level suits an investor profile.
        Conservative products suit all profiles.
        Moderate products suit MODERATE and AGGRESSIVE.
        Aggressive products suit only AGGRESSIVE.
        """
        match investor_risk_profile:
            case "CONSERVATIVE":
                return self == RiskClassification.CONSERVATIVE
            case "MODERATE":
                return self in (RiskClassification.CONSERVATIVE, RiskClassification.MODERATE)
            case "AGGRESSIVE":
                return True
            case _:
                return False


class Currency(str, Enum):
    SGD = "SGD"
    USD = "USD"
    INR = "INR"


@dataclass
class InvestmentProduct:
    """
    Aggregate Root for the Product bounded context.

    WHY dataclass and not Pydantic BaseModel:
    The domain model should not be coupled to Pydantic (an API/validation library).
    Pydantic models live in the adapter layer (in_rest).
    The domain uses Python dataclasses -- lightweight, framework-agnostic.
    """
    product_code:           str
    product_name:           str
    risk_classification:    RiskClassification
    expected_roi_percent:   float
    currency:               Currency
    minimum_investment_sgd: float      = 1000.0
    mas_regulated:          bool       = True
    id:                     Optional[int]      = field(default=None, repr=False)
    created_at:             Optional[datetime] = field(default_factory=datetime.utcnow)

    def __post_init__(self):
        """Domain invariants enforced at construction time."""
        if not self.product_code.startswith("PRD-"):
            raise ValueError(
                f"Product code must start with 'PRD-'. Got: {self.product_code}"
            )
        if not (0 <= self.expected_roi_percent <= 100):
            raise ValueError(
                f"Expected ROI must be between 0 and 100. Got: {self.expected_roi_percent}"
            )
        if self.minimum_investment_sgd < 0:
            raise ValueError("Minimum investment cannot be negative.")

    def is_suitable_for(self, investor_risk_profile: str) -> bool:
        """Domain behaviour: eligibility check delegated to the Value Object."""
        return self.risk_classification.is_suitable_for_investor_profile(
            investor_risk_profile
        )
'@ | Set-Content -Path "product-service\src\domain\model\investment_product.py" -Encoding UTF8
```

**File: product-service\src\domain\port\product_repository_port.py**

```powershell
New-Item -ItemType Directory -Path "product-service\src\domain\port" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\domain\port\__init__.py" -Force | Out-Null

@'
"""
product_repository_port.py -- Output Port (Abstract Base Class).

WHY an ABC and not a Protocol:
ABC enforces explicit registration -- adapters must declare they implement
the port. Protocol uses structural subtyping which can hide unintended
implementations. For a governance-critical port, explicit is better.

This port defines what the domain needs from persistence infrastructure.
SQLAlchemy, asyncpg, and PostgreSQL are invisible to the domain.
"""

from __future__ import annotations
from abc import ABC, abstractmethod
from typing import List, Optional

from ..model.investment_product import InvestmentProduct, RiskClassification


class ProductRepositoryPort(ABC):

    @abstractmethod
    def save(self, product: InvestmentProduct) -> InvestmentProduct:
        """Persist a new product and return it with assigned ID."""
        ...

    @abstractmethod
    def find_by_product_code(self, product_code: str) -> Optional[InvestmentProduct]:
        """Find a product by its unique code. Returns None if not found."""
        ...

    @abstractmethod
    def find_all(self) -> List[InvestmentProduct]:
        """Return all registered products."""
        ...

    @abstractmethod
    def find_by_risk_classification(
        self, classification: RiskClassification
    ) -> List[InvestmentProduct]:
        """Return products matching a specific risk classification."""
        ...

    @abstractmethod
    def exists_by_product_code(self, product_code: str) -> bool:
        """Check if a product with this code already exists."""
        ...
'@ | Set-Content -Path "product-service\src\domain\port\product_repository_port.py" -Encoding UTF8
```

**File: product-service\src\application\product_use_cases.py**

```powershell
New-Item -ItemType Directory -Path "product-service\src\application" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\application\__init__.py" -Force | Out-Null

@'
"""
product_use_cases.py -- Application layer: use case orchestration.

WHY application layer is separate from domain:
Use cases orchestrate: they call domain methods and output ports.
They do not contain business rules -- those are in the domain model.
They do not contain HTTP or database concerns -- those are in adapters.
"""

from __future__ import annotations
from typing import List, Optional

from ..domain.model.investment_product import (
    Currency,
    InvestmentProduct,
    RiskClassification,
)
from ..domain.port.product_repository_port import ProductRepositoryPort


class ProductAlreadyExistsError(Exception):
    pass


class ProductNotFoundError(Exception):
    pass


class ProductUseCase:
    """
    Application service implementing product management use cases.

    WHY injected port and not direct import of PostgreSQL adapter:
    The use case depends on the abstraction (ProductRepositoryPort),
    not the implementation. This enables testing with an in-memory
    stub without touching the database.
    """

    def __init__(self, repository: ProductRepositoryPort) -> None:
        self._repo = repository

    def create_product(
        self,
        product_code:           str,
        product_name:           str,
        risk_classification:    str,
        expected_roi_percent:   float,
        currency:               str,
        minimum_investment_sgd: float = 1000.0,
        mas_regulated:          bool  = True,
    ) -> InvestmentProduct:

        if self._repo.exists_by_product_code(product_code):
            raise ProductAlreadyExistsError(
                f"Product code {product_code} already exists in the catalogue."
            )

        product = InvestmentProduct(
            product_code           = product_code,
            product_name           = product_name,
            risk_classification    = RiskClassification(risk_classification),
            expected_roi_percent   = expected_roi_percent,
            currency               = Currency(currency),
            minimum_investment_sgd = minimum_investment_sgd,
            mas_regulated          = mas_regulated,
        )
        return self._repo.save(product)

    def get_product(self, product_code: str) -> InvestmentProduct:
        product = self._repo.find_by_product_code(product_code)
        if product is None:
            raise ProductNotFoundError(
                f"Product {product_code} not found in the catalogue."
            )
        return product

    def list_products(
        self,
        risk_profile: Optional[str] = None,
    ) -> List[InvestmentProduct]:
        if risk_profile:
            # Filter using domain logic -- is_suitable_for applies the business rule
            all_products = self._repo.find_all()
            return [p for p in all_products if p.is_suitable_for(risk_profile)]
        return self._repo.find_all()
'@ | Set-Content -Path "product-service\src\application\product_use_cases.py" -Encoding UTF8
```

**File: product-service\src\adapter\out_persistence\postgres_product_adapter.py**

```powershell
New-Item -ItemType Directory -Path "product-service\src\adapter\out_persistence" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\adapter\__init__.py" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\adapter\out_persistence\__init__.py" -Force | Out-Null

@'
"""
postgres_product_adapter.py -- Persistence adapter implementing ProductRepositoryPort.

WHY synchronous SQLAlchemy and not async:
For a demo lab with a simple CRUD pattern, synchronous SQLAlchemy reduces
complexity significantly. The FastAPI endpoints run in a thread pool for
sync operations. Production systems with high I/O concurrency would use
async SQLAlchemy 2.0 + asyncpg.

WHY a separate ORM model (ProductOrm) and not the domain dataclass:
Same reason as in the Java service -- the persistence model carries
SQLAlchemy metadata that the domain must not know about.
"""

from __future__ import annotations
from datetime import datetime
from typing import List, Optional

from sqlalchemy import Boolean, Column, DateTime, Float, Integer, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

from ...domain.model.investment_product import (
    Currency,
    InvestmentProduct,
    RiskClassification,
)
from ...domain.port.product_repository_port import ProductRepositoryPort


class Base(DeclarativeBase):
    pass


class ProductOrm(Base):
    __tablename__ = "investment_products"

    id                     = Column(Integer, primary_key=True, autoincrement=True)
    product_code           = Column(String(20), unique=True, nullable=False, index=True)
    product_name           = Column(String(255), nullable=False)
    risk_classification    = Column(String(20), nullable=False)
    expected_roi_percent   = Column(Float, nullable=False)
    currency               = Column(String(5), nullable=False)
    minimum_investment_sgd = Column(Float, nullable=False, default=1000.0)
    mas_regulated          = Column(Boolean, nullable=False, default=True)
    created_at             = Column(DateTime, default=datetime.utcnow)


class PostgresProductAdapter(ProductRepositoryPort):

    def __init__(self, database_url: str) -> None:
        self._engine        = create_engine(database_url, echo=False)
        self._session_maker = sessionmaker(bind=self._engine)
        Base.metadata.create_all(self._engine)  # Creates table if not exists

    def save(self, product: InvestmentProduct) -> InvestmentProduct:
        with self._session_maker() as session:
            orm = self._to_orm(product)
            session.add(orm)
            session.commit()
            session.refresh(orm)
            return self._to_domain(orm)

    def find_by_product_code(self, product_code: str) -> Optional[InvestmentProduct]:
        with self._session_maker() as session:
            orm = session.query(ProductOrm).filter_by(
                product_code=product_code
            ).first()
            return self._to_domain(orm) if orm else None

    def find_all(self) -> List[InvestmentProduct]:
        with self._session_maker() as session:
            return [self._to_domain(o) for o in session.query(ProductOrm).all()]

    def find_by_risk_classification(
        self, classification: RiskClassification
    ) -> List[InvestmentProduct]:
        with self._session_maker() as session:
            rows = session.query(ProductOrm).filter_by(
                risk_classification=classification.value
            ).all()
            return [self._to_domain(r) for r in rows]

    def exists_by_product_code(self, product_code: str) -> bool:
        with self._session_maker() as session:
            return session.query(ProductOrm).filter_by(
                product_code=product_code
            ).count() > 0

    @staticmethod
    def _to_orm(product: InvestmentProduct) -> ProductOrm:
        return ProductOrm(
            product_code           = product.product_code,
            product_name           = product.product_name,
            risk_classification    = product.risk_classification.value,
            expected_roi_percent   = product.expected_roi_percent,
            currency               = product.currency.value,
            minimum_investment_sgd = product.minimum_investment_sgd,
            mas_regulated          = product.mas_regulated,
        )

    @staticmethod
    def _to_domain(orm: ProductOrm) -> InvestmentProduct:
        return InvestmentProduct(
            id                     = orm.id,
            product_code           = orm.product_code,
            product_name           = orm.product_name,
            risk_classification    = RiskClassification(orm.risk_classification),
            expected_roi_percent   = orm.expected_roi_percent,
            currency               = Currency(orm.currency),
            minimum_investment_sgd = orm.minimum_investment_sgd,
            mas_regulated          = orm.mas_regulated,
            created_at             = orm.created_at,
        )
'@ | Set-Content -Path "product-service\src\adapter\out_persistence\postgres_product_adapter.py" -Encoding UTF8
```

**File: product-service\src\adapter\in_rest\product_router.py**

```powershell
New-Item -ItemType Directory -Path "product-service\src\adapter\in_rest" -Force | Out-Null
New-Item -ItemType File -Path "product-service\src\adapter\in_rest\__init__.py" -Force | Out-Null

@'
"""
product_router.py -- REST adapter (Primary Adapter) for Product bounded context.

WHY FastAPI APIRouter and not a flat app:
APIRouter allows this adapter to be mounted on the main app with a prefix.
It keeps routing declarations separate from application wiring.
The router knows nothing about the database -- it depends only on ProductUseCase.
"""

from __future__ import annotations
from datetime import datetime
from typing import List, Optional

from fastapi import APIRouter, Depends, HTTPException, Query
from pydantic import BaseModel, Field, field_validator

from ...application.product_use_cases import (
    ProductAlreadyExistsError,
    ProductNotFoundError,
    ProductUseCase,
)
from ...domain.model.investment_product import InvestmentProduct

router = APIRouter(prefix="/api/products", tags=["Products"])


# --- Request/Response DTOs (Pydantic -- adapter concern, not domain) ---

class CreateProductRequest(BaseModel):
    productCode:           str   = Field(pattern=r"^PRD-[A-Z0-9]{6}$",
                                         examples=["PRD-SG0001"])
    productName:           str   = Field(min_length=3, max_length=255)
    riskClassification:    str
    expectedRoiPercent:    float = Field(ge=0, le=100)
    currency:              str
    minimumInvestmentSgd:  float = Field(default=1000.0, ge=0)
    masRegulated:          bool  = True

    @field_validator("riskClassification")
    @classmethod
    def validate_risk(cls, v: str) -> str:
        allowed = {"CONSERVATIVE", "MODERATE", "AGGRESSIVE"}
        if v not in allowed:
            raise ValueError(f"riskClassification must be one of {allowed}")
        return v

    @field_validator("currency")
    @classmethod
    def validate_currency(cls, v: str) -> str:
        allowed = {"SGD", "USD", "INR"}
        if v not in allowed:
            raise ValueError(f"currency must be one of {allowed}")
        return v


class ProductResponse(BaseModel):
    id:                    Optional[int]
    productCode:           str
    productName:           str
    riskClassification:    str
    expectedRoiPercent:    float
    currency:              str
    minimumInvestmentSgd:  float
    masRegulated:          bool
    createdAt:             Optional[datetime]

    @classmethod
    def from_domain(cls, p: InvestmentProduct) -> "ProductResponse":
        return cls(
            id                   = p.id,
            productCode          = p.product_code,
            productName          = p.product_name,
            riskClassification   = p.risk_classification.value,
            expectedRoiPercent   = p.expected_roi_percent,
            currency             = p.currency.value,
            minimumInvestmentSgd = p.minimum_investment_sgd,
            masRegulated         = p.mas_regulated,
            createdAt            = p.created_at,
        )


# --- Dependency injection (use case bound to router via app wiring) ---

_use_case: Optional[ProductUseCase] = None

def get_use_case() -> ProductUseCase:
    if _use_case is None:
        raise RuntimeError("ProductUseCase not initialised. Check app startup.")
    return _use_case

def set_use_case(uc: ProductUseCase) -> None:
    global _use_case
    _use_case = uc


# --- Route handlers ---

@router.post("", response_model=ProductResponse, status_code=201)
def create_product(
    request: CreateProductRequest,
    uc: ProductUseCase = Depends(get_use_case)
) -> ProductResponse:
    try:
        product = uc.create_product(
            product_code           = request.productCode,
            product_name           = request.productName,
            risk_classification    = request.riskClassification,
            expected_roi_percent   = request.expectedRoiPercent,
            currency               = request.currency,
            minimum_investment_sgd = request.minimumInvestmentSgd,
            mas_regulated          = request.masRegulated,
        )
        return ProductResponse.from_domain(product)
    except ProductAlreadyExistsError as e:
        raise HTTPException(status_code=409, detail=str(e))
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.get("", response_model=List[ProductResponse])
def list_products(
    riskProfile: Optional[str] = Query(default=None),
    uc: ProductUseCase         = Depends(get_use_case)
) -> List[ProductResponse]:
    products = uc.list_products(risk_profile=riskProfile)
    return [ProductResponse.from_domain(p) for p in products]


@router.get("/{product_code}", response_model=ProductResponse)
def get_product(
    product_code: str,
    uc: ProductUseCase = Depends(get_use_case)
) -> ProductResponse:
    try:
        return ProductResponse.from_domain(uc.get_product(product_code))
    except ProductNotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
'@ | Set-Content -Path "product-service\src\adapter\in_rest\product_router.py" -Encoding UTF8
```

**File: product-service\src\main.py**

```powershell
@'
"""
main.py -- FastAPI application entry point for Product Service.

WHY dependency wiring happens here and not in the router:
The adapter (router) depends on an abstraction (ProductUseCase).
The concrete adapter (PostgresProductAdapter) is wired here at startup.
This is the Composition Root pattern -- all dependency wiring happens
in one place, at the application boundary.
"""

from __future__ import annotations
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI

from src.adapter.in_rest.product_router import router, set_use_case
from src.adapter.out_persistence.postgres_product_adapter import PostgresProductAdapter
from src.application.product_use_cases import ProductUseCase


DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql://nivesh_user:nivesh_pass@postgres:5432/product_db"
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Composition Root: wire concrete adapters to use cases
    repo     = PostgresProductAdapter(DATABASE_URL)
    use_case = ProductUseCase(repo)
    set_use_case(use_case)

    # Seed demo data if catalogue is empty
    _seed_products(use_case)

    print(f"[Product Service] Started. DB: {DATABASE_URL.split('@')[1]}")
    yield
    print("[Product Service] Shutdown.")


def _seed_products(uc: ProductUseCase) -> None:
    """Seed demo investment products on first start."""
    seed_data = [
        {
            "product_code": "PRD-SG0001", "product_name": "Singapore Government Bond Fund",
            "risk_classification": "CONSERVATIVE", "expected_roi_percent": 3.5,
            "currency": "SGD", "minimum_investment_sgd": 500.0, "mas_regulated": True
        },
        {
            "product_code": "PRD-SG0002", "product_name": "STI Index ETF",
            "risk_classification": "MODERATE", "expected_roi_percent": 7.2,
            "currency": "SGD", "minimum_investment_sgd": 1000.0, "mas_regulated": True
        },
        {
            "product_code": "PRD-SG0003", "product_name": "Asia Growth Equity Fund",
            "risk_classification": "AGGRESSIVE", "expected_roi_percent": 14.5,
            "currency": "SGD", "minimum_investment_sgd": 5000.0, "mas_regulated": True
        },
        {
            "product_code": "PRD-US0001", "product_name": "US Technology Index Fund",
            "risk_classification": "AGGRESSIVE", "expected_roi_percent": 18.0,
            "currency": "USD", "minimum_investment_sgd": 10000.0, "mas_regulated": True
        },
        {
            "product_code": "PRD-IN0001", "product_name": "India Government Securities Fund",
            "risk_classification": "CONSERVATIVE", "expected_roi_percent": 6.8,
            "currency": "INR", "minimum_investment_sgd": 250.0, "mas_regulated": False
        },
    ]
    for data in seed_data:
        try:
            uc.create_product(**data)
        except Exception:
            pass  # Already seeded


app = FastAPI(
    title="Nivesh Gateway — Investment Product Catalogue",
    version="1.0.0",
    lifespan=lifespan
)

app.include_router(router)


@app.get("/health")
async def health():
    return {"status": "UP", "service": "product-service"}
'@ | Set-Content -Path "product-service\src\main.py" -Encoding UTF8
```

**File: product-service\src\__init__.py**

```powershell
New-Item -ItemType File -Path "product-service\src\__init__.py" -Force | Out-Null
```

**File: product-service\tests\test_product_use_cases.py**

```powershell
New-Item -ItemType Directory -Path "product-service\tests" -Force | Out-Null
New-Item -ItemType File -Path "product-service\tests\__init__.py" -Force | Out-Null

@'
"""
test_product_use_cases.py -- Unit tests for product use cases.

WHY in-memory stub and not a real database:
These are UNIT tests for the application layer.
They test that the use case orchestrates correctly.
The repository port is stubbed -- no database, no Docker, runs in milliseconds.
Integration tests (requiring a database) live separately.
"""

import pytest
from typing import Dict, List, Optional

from src.domain.model.investment_product import Currency, InvestmentProduct, RiskClassification
from src.domain.port.product_repository_port import ProductRepositoryPort
from src.application.product_use_cases import (
    ProductAlreadyExistsError,
    ProductNotFoundError,
    ProductUseCase,
)


class InMemoryProductRepository(ProductRepositoryPort):
    """In-memory stub implementing the output port for testing."""

    def __init__(self) -> None:
        self._store: Dict[str, InvestmentProduct] = {}
        self._next_id = 1

    def save(self, product: InvestmentProduct) -> InvestmentProduct:
        product.id = self._next_id
        self._next_id += 1
        self._store[product.product_code] = product
        return product

    def find_by_product_code(self, product_code: str) -> Optional[InvestmentProduct]:
        return self._store.get(product_code)

    def find_all(self) -> List[InvestmentProduct]:
        return list(self._store.values())

    def find_by_risk_classification(self, classification: RiskClassification) -> List[InvestmentProduct]:
        return [p for p in self._store.values() if p.risk_classification == classification]

    def exists_by_product_code(self, product_code: str) -> bool:
        return product_code in self._store


@pytest.fixture
def repo():
    return InMemoryProductRepository()


@pytest.fixture
def use_case(repo):
    return ProductUseCase(repo)


def make_product_args(**overrides):
    defaults = dict(
        product_code="PRD-SG0001",
        product_name="Singapore Bond Fund",
        risk_classification="CONSERVATIVE",
        expected_roi_percent=3.5,
        currency="SGD",
    )
    return {**defaults, **overrides}


class TestCreateProduct:
    def test_create_valid_product_returns_with_id(self, use_case):
        product = use_case.create_product(**make_product_args())
        assert product.id is not None
        assert product.product_code == "PRD-SG0001"
        assert product.risk_classification == RiskClassification.CONSERVATIVE

    def test_duplicate_product_code_raises_error(self, use_case):
        use_case.create_product(**make_product_args())
        with pytest.raises(ProductAlreadyExistsError, match="PRD-SG0001"):
            use_case.create_product(**make_product_args())

    def test_invalid_product_code_format_raises_error(self, use_case):
        with pytest.raises(ValueError, match="PRD-"):
            use_case.create_product(**make_product_args(product_code="INVALID"))

    def test_roi_out_of_range_raises_error(self, use_case):
        with pytest.raises(ValueError):
            use_case.create_product(**make_product_args(expected_roi_percent=150))


class TestListProducts:
    def test_list_all_returns_all_products(self, use_case):
        use_case.create_product(**make_product_args(product_code="PRD-SG0001"))
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0002", risk_classification="MODERATE"))
        assert len(use_case.list_products()) == 2

    def test_filter_by_conservative_returns_only_conservative(self, use_case):
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0001", risk_classification="CONSERVATIVE"))
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0002", risk_classification="AGGRESSIVE"))
        result = use_case.list_products(risk_profile="CONSERVATIVE")
        assert all(p.risk_classification == RiskClassification.CONSERVATIVE for p in result)

    def test_moderate_investor_sees_conservative_and_moderate(self, use_case):
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0001", risk_classification="CONSERVATIVE"))
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0002", risk_classification="MODERATE"))
        use_case.create_product(**make_product_args(
            product_code="PRD-SG0003", risk_classification="AGGRESSIVE"))
        result = use_case.list_products(risk_profile="MODERATE")
        assert len(result) == 2

    def test_aggressive_investor_sees_all_products(self, use_case):
        for code, risk in [("PRD-SG0001", "CONSERVATIVE"),
                           ("PRD-SG0002", "MODERATE"),
                           ("PRD-SG0003", "AGGRESSIVE")]:
            use_case.create_product(**make_product_args(
                product_code=code, risk_classification=risk))
        result = use_case.list_products(risk_profile="AGGRESSIVE")
        assert len(result) == 3


class TestGetProduct:
    def test_get_existing_product_returns_product(self, use_case):
        use_case.create_product(**make_product_args())
        product = use_case.get_product("PRD-SG0001")
        assert product.product_name == "Singapore Bond Fund"

    def test_get_nonexistent_product_raises_error(self, use_case):
        with pytest.raises(ProductNotFoundError, match="PRD-NONE"):
            use_case.get_product("PRD-NONE")
'@ | Set-Content -Path "product-service\tests\test_product_use_cases.py" -Encoding UTF8
```

---

### L3.4 Nginx API Gateway Configuration

**File: gateway\nginx\nginx.conf**

```powershell
New-Item -ItemType Directory -Path "gateway\nginx" -Force | Out-Null

@'
# nginx.conf -- API Gateway routing for Nivesh Gateway local demo
#
# WHY Nginx as a gateway stub:
# In production, Azure API Management handles routing, auth, rate limiting,
# and policy enforcement. Nginx here demonstrates the routing concept
# without requiring a paid Azure service.
#
# WHY path-based routing and not host-based:
# For local development, all services share localhost. Path prefix routing
# (/api/investors/* vs /api/products/*) demonstrates bounded context
# separation at the gateway level.

events {
    worker_connections 1024;
}

http {
    upstream investor_service {
        server investor-service:8080;
    }

    upstream product_service {
        server product-service:8000;
    }

    server {
        listen 8090;
        server_name localhost;

        # Gateway headers -- demonstrates request tracing pattern
        add_header X-Gateway "Nivesh-API-Gateway-Stub" always;
        add_header X-Request-ID $request_id always;

        # Health check for the gateway itself
        location /gateway/health {
            return 200 '{"status":"UP","service":"api-gateway-stub"}';
            add_header Content-Type application/json;
        }

        # Route investor bounded context
        location /api/investors {
            proxy_pass         http://investor_service;
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Request-ID      $request_id;
            proxy_read_timeout 30s;
            proxy_connect_timeout 5s;
        }

        # Route product bounded context
        location /api/products {
            proxy_pass         http://product_service;
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Request-ID      $request_id;
            proxy_read_timeout 30s;
            proxy_connect_timeout 5s;
        }

        # Error responses
        error_page 502 503 504 @gateway_error;
        location @gateway_error {
            return 503 '{"status":"ERROR","message":"Upstream service unavailable"}';
            add_header Content-Type application/json;
        }
    }
}
'@ | Set-Content -Path "gateway\nginx\nginx.conf" -Encoding UTF8
```

---

### L3.5 Docker Files

**File: docker\Dockerfile.investor**

```powershell
@'
# Multi-stage build for Investor Service
# Stage 1: Build the JAR
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /build
RUN apk add --no-cache maven
COPY investor-service/pom.xml .
# Download dependencies first (layer caching optimisation)
RUN mvn --no-transfer-progress dependency:go-offline -q
COPY investor-service/src ./src
RUN mvn --no-transfer-progress -q package -DskipTests

# Stage 2: Runtime image
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=builder /build/target/investor-service-1.0.0.jar app.jar
ENV JAVA_OPTS="-Xms256m -Xmx512m -XX:+UseG1GC"
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=5 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
'@ | Set-Content -Path "docker\Dockerfile.investor" -Encoding UTF8
```

**File: docker\Dockerfile.product**

```powershell
@'
FROM python:3.10-slim AS runtime
WORKDIR /app

# Install dependencies
COPY product-service/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source
COPY product-service/src ./src
COPY product-service/src/main.py ./main.py

RUN useradd -r -s /bin/false appuser
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
'@ | Set-Content -Path "docker\Dockerfile.product" -Encoding UTF8
```

**File: docker\docker-compose.yml**

```powershell
@'
version: "3.9"

networks:
  nivesh-net:
    driver: bridge
    # WHY a named network:
    # Services communicate by container name (e.g., postgres:5432).
    # Named networks make service discovery explicit and auditable.

volumes:
  postgres-data:
    # Named volume for PostgreSQL data persistence across restarts

services:

  postgres:
    image: postgres:15-alpine
    container_name: nivesh-postgres
    environment:
      POSTGRES_USER: nivesh_user
      POSTGRES_PASSWORD: nivesh_pass
      POSTGRES_MULTIPLE_DATABASES: investor_db,product_db
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh
    ports:
      - "5432:5432"
    networks:
      - nivesh-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nivesh_user -d investor_db && pg_isready -U nivesh_user -d product_db"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 15s

  investor-service:
    build:
      context: ..
      dockerfile: docker/Dockerfile.investor
    container_name: nivesh-investor-service
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/investor_db
      SPRING_DATASOURCE_USERNAME: nivesh_user
      SPRING_DATASOURCE_PASSWORD: nivesh_pass
      PRODUCT_CATALOGUE_BASE_URL: http://product-service:8000
      SPRING_PROFILES_ACTIVE: local
    networks:
      - nivesh-net
    depends_on:
      postgres:
        condition: service_healthy
    restart: on-failure

  product-service:
    build:
      context: ..
      dockerfile: docker/Dockerfile.product
    container_name: nivesh-product-service
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://nivesh_user:nivesh_pass@postgres:5432/product_db
    networks:
      - nivesh-net
    depends_on:
      postgres:
        condition: service_healthy
    restart: on-failure

  api-gateway:
    image: nginx:1.25-alpine
    container_name: nivesh-api-gateway
    ports:
      - "8090:8090"
    volumes:
      - ../gateway/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - nivesh-net
    depends_on:
      - investor-service
      - product-service
    restart: on-failure
'@ | Set-Content -Path "docker\docker-compose.yml" -Encoding UTF8
```

**File: docker\init-multiple-dbs.sh** — Creates two separate databases at PostgreSQL startup.

```powershell
@'
#!/bin/bash
# init-multiple-dbs.sh
# Creates investor_db and product_db as separate databases.
# WHY separate databases: Bounded context isolation at the infrastructure level.

set -e

create_db() {
    local db=$1
    echo "Creating database: $db"
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE DATABASE $db;
        GRANT ALL PRIVILEGES ON DATABASE $db TO $POSTGRES_USER;
EOSQL
}

# investor_db and product_db are created automatically from the env var
# if using the POSTGRES_MULTIPLE_DATABASES pattern, but we use explicit creation:
for db in investor_db product_db; do
    if psql -U "$POSTGRES_USER" -lqt | cut -d \| -f 1 | grep -qw "$db"; then
        echo "Database $db already exists, skipping."
    else
        create_db "$db"
    fi
done
'@ | Set-Content -Path "docker\init-multiple-dbs.sh" -Encoding UTF8
```

Convert to Unix line endings (required for shell scripts in Docker):

```powershell
# Convert CRLF to LF for the shell script (critical for Linux containers)
$content = Get-Content "docker\init-multiple-dbs.sh" -Raw
$content = $content -replace "`r`n", "`n"
[System.IO.File]::WriteAllText(
    (Resolve-Path "docker\init-multiple-dbs.sh").Path,
    $content,
    [System.Text.Encoding]::UTF8
)
Write-Host "Line endings converted for init-multiple-dbs.sh" -ForegroundColor Green
```

---

### L3.6 Terraform Files

**File: infra\terraform\providers.tf**

```powershell
@'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
}

provider "azurerm" {
  features {}
}
'@ | Set-Content -Path "infra\terraform\providers.tf" -Encoding UTF8
```

**File: infra\terraform\variables.tf**

```powershell
@'
variable "location" {
  description = "Azure region -- southeastasia for Singapore proximity"
  type        = string
  default     = "southeastasia"
}

variable "resource_group_name" {
  description = "Resource group for Nivesh Gateway DDD services"
  type        = string
  default     = "rg-nivesh-gateway-ddd"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "demo"
}

variable "storage_account_name" {
  description = "Globally unique storage account name"
  type        = string
  default     = "niveshgatewayddd001"
}
'@ | Set-Content -Path "infra\terraform\variables.tf" -Encoding UTF8
```

**File: infra\terraform\main.tf**

```powershell
@'
# Day 2 Terraform: Two bounded contexts need two separate storage containers.
# This mirrors the architectural decision: bounded contexts have separate
# data ownership at every layer -- code, database, and artifact storage.

resource "azurerm_resource_group" "ddd" {
  name     = var.resource_group_name
  location = var.location
  tags = {
    environment  = var.environment
    project      = "nivesh-gateway"
    day          = "day2-ddd"
    managed-by   = "terraform"
  }
}

resource "azurerm_storage_account" "artifacts" {
  name                            = var.storage_account_name
  resource_group_name             = azurerm_resource_group.ddd.name
  location                        = azurerm_resource_group.ddd.location
  account_tier                    = "Standard"
  account_replication_type        = "LRS"
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
  tags = {
    environment = var.environment
    managed-by  = "terraform"
  }
}

# Separate container per bounded context -- architectural isolation at the artifact level
resource "azurerm_storage_container" "investor_context" {
  name                  = "investor-context"
  storage_account_name  = azurerm_storage_account.artifacts.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "product_context" {
  name                  = "product-context"
  storage_account_name  = azurerm_storage_account.artifacts.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "openapi_contracts" {
  name                  = "openapi-contracts"
  storage_account_name  = azurerm_storage_account.artifacts.name
  container_access_type = "private"
}
'@ | Set-Content -Path "infra\terraform\main.tf" -Encoding UTF8
```

**File: infra\terraform\outputs.tf**

```powershell
@'
output "resource_group_name"          { value = azurerm_resource_group.ddd.name }
output "storage_account_name"         { value = azurerm_storage_account.artifacts.name }
output "investor_container_name"      { value = azurerm_storage_container.investor_context.name }
output "product_container_name"       { value = azurerm_storage_container.product_context.name }
output "openapi_contracts_container"  { value = azurerm_storage_container.openapi_contracts.name }
'@ | Set-Content -Path "infra\terraform\outputs.tf" -Encoding UTF8
```

---

### L3.7 GitHub Actions CI

**File: .github\workflows\ci.yml**

```powershell
@'
name: day2-ci

on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["main"]

jobs:

  investor-service-build:
    name: Investor Service -- Java Build and Test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: investor_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    defaults:
      run:
        working-directory: investor-service
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
          cache: maven
      - name: Build and test
        run: mvn --no-transfer-progress clean test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/investor_test
          SPRING_DATASOURCE_USERNAME: test_user
          SPRING_DATASOURCE_PASSWORD: test_pass
          PRODUCT_CATALOGUE_BASE_URL: http://localhost:19999

  product-service-test:
    name: Product Service -- Python Unit Tests
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: product-service
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
          cache: pip
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run unit tests
        run: |
          cd ..
          PYTHONPATH=. pytest product-service/tests/ -v --tb=short

  terraform-validate:
    name: Terraform Validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.5.7"
      - run: terraform init -backend=false
        working-directory: infra/terraform
      - run: terraform validate
        working-directory: infra/terraform
'@ | Set-Content -Path ".github\workflows\ci.yml" -Encoding UTF8
```

---

### L3.8 Automation Scripts

**File: scripts\setup.ps1**

```powershell
@'
# setup.ps1 -- Day 2 Lab one-click setup
# Run from project root: .\scripts\setup.ps1

param([switch]$SkipBuild, [switch]$SkipPython)

$ErrorActionPreference = "Stop"
$projectRoot = Split-Path -Parent $PSScriptRoot
Set-Location $projectRoot

function Write-Step { param($msg) Write-Host "`n[STEP] $msg" -ForegroundColor Cyan }
function Write-OK   { param($msg) Write-Host "[OK]   $msg" -ForegroundColor Green }
function Write-FAIL { param($msg) Write-Host "[FAIL] $msg" -ForegroundColor Red; exit 1 }

Write-Step "Checking tools..."
foreach ($cmd in @("docker","java","mvn","python","terraform")) {
    try { Invoke-Expression "$cmd --version 2>&1" | Out-Null; Write-OK $cmd }
    catch { Write-FAIL "$cmd not found" }
}

Write-Step "Checking Docker Desktop..."
try { docker info | Out-Null; Write-OK "Docker running" }
catch { Write-FAIL "Docker Desktop not running" }

if (-not $SkipBuild) {
    Write-Step "Building Java Investor Service..."
    Set-Location "investor-service"
    mvn --no-transfer-progress -q clean package -DskipTests
    if ($LASTEXITCODE -ne 0) { Write-FAIL "Maven build failed" }
    Write-OK "Java build complete"
    Set-Location $projectRoot
}

if (-not $SkipPython) {
    Write-Step "Installing Python dependencies for Product Service..."
    pip install -r product-service/requirements.txt -q
    Write-OK "Python dependencies installed"
}

Write-Step "Fixing line endings for shell script..."
$content = Get-Content "docker\init-multiple-dbs.sh" -Raw
$content = $content -replace "`r`n", "`n"
[System.IO.File]::WriteAllText(
    (Resolve-Path "docker\init-multiple-dbs.sh").Path,
    $content, [System.Text.Encoding]::UTF8
)
Write-OK "Line endings fixed"

Write-Step "Validating Terraform..."
Set-Location "infra\terraform"
terraform init -input=false | Out-Null
terraform validate
Set-Location $projectRoot

Write-Host "`n=== Day 2 Setup Complete ===" -ForegroundColor Green
Write-Host "Next: .\scripts\demo.ps1" -ForegroundColor Yellow
'@ | Set-Content -Path "scripts\setup.ps1" -Encoding UTF8
```

**File: scripts\demo.ps1**

```powershell
@'
# demo.ps1 -- Day 2 full demo playbook (automated)
# Run from project root: .\scripts\demo.ps1

$ErrorActionPreference = "Continue"
$projectRoot = Split-Path -Parent $PSScriptRoot
Set-Location "$projectRoot\docker"

function Write-Step   { param($m) Write-Host "`n[DEMO] $m" -ForegroundColor Cyan }
function Write-OK     { param($m) Write-Host "[OK]   $m" -ForegroundColor Green }
function Write-Result { param($m) Write-Host "[OUT]  $m" -ForegroundColor White }

Write-Step "Starting full stack..."
docker-compose up -d --build
Write-Host "Waiting 90 seconds for all services to be healthy..." -ForegroundColor Yellow
Start-Sleep -Seconds 90

Write-Step "Checking all containers..."
docker-compose ps

Write-Step "Verifying gateway health..."
$gw = Invoke-RestMethod "http://localhost:8090/gateway/health"
Write-Result ($gw | ConvertTo-Json)

Write-Step "Creating Investor INV-000001 (Priya -- MODERATE risk, INDIAN)"
$inv1 = @{
    investorId  = "INV-000001"; fullName = "Priya Krishnamurthy"
    email       = "priya.k@nivesh.com"; nationality = "INDIAN"
    riskProfile = "MODERATE"
} | ConvertTo-Json
$r1 = Invoke-RestMethod "http://localhost:8090/api/investors" -Method POST -ContentType "application/json" -Body $inv1
Write-OK "Created: $($r1.investorId) | KYC: $($r1.kycStatus) | Trading: $($r1.tradingAllowed)"

Write-Step "Creating Investor INV-000002 (Wei Ming -- AGGRESSIVE, SINGAPOREAN)"
$inv2 = @{
    investorId  = "INV-000002"; fullName = "Wei Ming Tan"
    email       = "weiming@nivesh.com.sg"; nationality = "SINGAPOREAN"
    riskProfile = "AGGRESSIVE"
} | ConvertTo-Json
$r2 = Invoke-RestMethod "http://localhost:8090/api/investors" -Method POST -ContentType "application/json" -Body $inv2
Write-OK "Created: $($r2.investorId) | Risk: $($r2.riskProfile)"

Write-Step "DEMO: Verifying Priya's KYC status..."
$kyc = @{ kycStatus = "VERIFIED" } | ConvertTo-Json
$r3 = Invoke-RestMethod "http://localhost:8090/api/investors/INV-000001/kyc" -Method PATCH -ContentType "application/json" -Body $kyc
Write-OK "KYC updated: $($r3.kycStatus) | Trading now allowed: $($r3.tradingAllowed)"

Write-Step "DEMO: Showing ALL investment products in catalogue..."
$products = Invoke-RestMethod "http://localhost:8090/api/products"
Write-Result "Total products: $($products.Count)"
$products | ForEach-Object {
    Write-Result "  $($_.productCode) | $($_.riskClassification) | ROI: $($_.expectedRoiPercent)% | $($_.currency)"
}

Write-Step "DEMO: Filtering products for MODERATE risk profile..."
$modProducts = Invoke-RestMethod "http://localhost:8090/api/products?riskProfile=MODERATE"
Write-Result "Products eligible for MODERATE investor: $($modProducts.Count)"
$modProducts | ForEach-Object { Write-Result "  $($_.productCode): $($_.productName)" }

Write-Step "DEMO: Cross-context call -- Priya's eligible products (via Investor Service -> Product Service)"
$eligible = Invoke-RestMethod "http://localhost:8090/api/investors/INV-000001/eligible-products"
Write-Result "Investor: $($eligible.investorId) | Risk: $($eligible.riskProfile) | KYC: $($eligible.kycStatus)"
Write-Result "Eligible products: $($eligible.productCount)"
$eligible.eligibleProducts | ForEach-Object {
    Write-Result "  $($_.productCode): $($_.productName)"
}

Write-Step "DEMO: Wei Ming's eligible products (AGGRESSIVE -- should see all)"
$eligible2 = Invoke-RestMethod "http://localhost:8090/api/investors/INV-000002/eligible-products"
Write-Result "Wei Ming eligible products: $($eligible2.productCount) (KYC status: $($eligible2.kycStatus))"

Write-Step "DEMO: Intentional failure -- invalid investor ID format (expect 400)"
try {
    Invoke-RestMethod "http://localhost:8090/api/investors" -Method POST -ContentType "application/json" `
        -Body (@{investorId="INVALID"; fullName="X"; email="x@x.com"; nationality="INDIAN"; riskProfile="MODERATE"} | ConvertTo-Json)
} catch {
    Write-OK "Correctly rejected invalid format: $($_.Exception.Response.StatusCode)"
}

Write-Step "DEMO: Intentional failure -- duplicate investor (expect 409)"
try {
    Invoke-RestMethod "http://localhost:8090/api/investors" -Method POST -ContentType "application/json" -Body $inv1
} catch {
    Write-OK "Correctly rejected duplicate: $($_.Exception.Response.StatusCode)"
}

Write-Step "DEMO: Intentional failure -- invalid KYC state transition (VERIFIED -> PENDING)"
try {
    Invoke-RestMethod "http://localhost:8090/api/investors/INV-000001/kyc" -Method PATCH -ContentType "application/json" `
        -Body (@{kycStatus="PENDING"} | ConvertTo-Json)
} catch {
    Write-OK "Domain rule enforced: $($_.Exception.Response.StatusCode)"
}

Write-Host "`n=== Day 2 Demo Complete ===" -ForegroundColor Green
Write-Host "Gateway: http://localhost:8090" -ForegroundColor Yellow
Write-Host "Investor Service: http://localhost:8080" -ForegroundColor Yellow
Write-Host "Product Service: http://localhost:8000/docs (Swagger UI)" -ForegroundColor Yellow
'@ | Set-Content -Path "scripts\demo.ps1" -Encoding UTF8
```

**File: scripts\verify.ps1**

```powershell
@'
# verify.ps1 -- Day 2 Lab verification
# Run from project root: .\scripts\verify.ps1

$ErrorActionPreference = "Continue"
$passed = 0; $failed = 0

function Test-Check {
    param($name, $scriptBlock)
    try { & $scriptBlock; Write-Host "[PASS] $name" -ForegroundColor Green; $script:passed++ }
    catch { Write-Host "[FAIL] $name -- $_" -ForegroundColor Red; $script:failed++ }
}

Write-Host "`n=== Day 2 Lab Verification ===" -ForegroundColor Cyan

Test-Check "Docker Desktop running" { docker info | Out-Null }

Test-Check "PostgreSQL healthy" {
    $r = docker exec nivesh-postgres pg_isready -U nivesh_user -d investor_db 2>&1
    if ($r -notmatch "accepting") { throw "Not ready" }
}

Test-Check "investor_db exists" {
    docker exec nivesh-postgres psql -U nivesh_user -d investor_db -c "\dt" 2>&1 | Out-Null
}

Test-Check "product_db exists" {
    docker exec nivesh-postgres psql -U nivesh_user -d product_db -c "\dt" 2>&1 | Out-Null
}

Test-Check "Investor Service health UP" {
    $r = Invoke-RestMethod "http://localhost:8080/actuator/health"
    if ($r.status -ne "UP") { throw "Status: $($r.status)" }
}

Test-Check "Product Service health UP" {
    $r = Invoke-RestMethod "http://localhost:8000/health"
    if ($r.status -ne "UP") { throw "Status: $($r.status)" }
}

Test-Check "API Gateway health UP" {
    $r = Invoke-RestMethod "http://localhost:8090/gateway/health"
    if ($r.status -ne "UP") { throw "Status: $($r.status)" }
}

Test-Check "Create investor via gateway returns 201" {
    $body = @{
        investorId="INV-VER-001"; fullName="Verify Test"; email="ver001@test.com"
        nationality="INDIAN"; riskProfile="MODERATE"
    } | ConvertTo-Json
    Invoke-RestMethod "http://localhost:8090/api/investors" -Method POST -ContentType "application/json" -Body $body | Out-Null
}

Test-Check "Get investor via gateway returns 200" {
    Invoke-RestMethod "http://localhost:8090/api/investors/INV-VER-001" | Out-Null
}

Test-Check "Products catalogue has seeded products" {
    $r = Invoke-RestMethod "http://localhost:8000/api/products"
    if ($r.Count -lt 5) { throw "Expected at least 5 seeded products, got $($r.Count)" }
}

Test-Check "Product filter by CONSERVATIVE returns subset" {
    $r = Invoke-RestMethod "http://localhost:8000/api/products?riskProfile=CONSERVATIVE"
    if ($r.Count -lt 1) { throw "No conservative products returned" }
}

Test-Check "KYC update changes status" {
    Invoke-RestMethod "http://localhost:8090/api/investors/INV-VER-001/kyc" `
        -Method PATCH -ContentType "application/json" `
        -Body (@{kycStatus="VERIFIED"} | ConvertTo-Json) | Out-Null
    $r = Invoke-RestMethod "http://localhost:8090/api/investors/INV-VER-001"
    if ($r.kycStatus -ne "VERIFIED") { throw "KYC not updated" }
}

Test-Check "Eligible products endpoint responds" {
    Invoke-RestMethod "http://localhost:8090/api/investors/INV-VER-001/eligible-products" | Out-Null
}

Test-Check "Terraform valid" {
    Set-Location "infra\terraform"
    terraform validate | Out-Null
    if ($LASTEXITCODE -ne 0) { throw "Terraform invalid" }
    Set-Location "..\.."}

Test-Check "Python unit tests pass" {
    $env:PYTHONPATH = "$PWD"
    $result = pytest product-service/tests/ -v --tb=short 2>&1
    if ($LASTEXITCODE -ne 0) { throw "Python tests failed" }
}

Test-Check "Java investors table exists in DB" {
    $r = docker exec nivesh-postgres psql -U nivesh_user -d investor_db `
        -c "SELECT COUNT(*) FROM investors;" 2>&1
    if ($r -notmatch "\d+") { throw "investors table query failed" }
}

Write-Host "`n=== Verification Results ===" -ForegroundColor Cyan
Write-Host "Passed: $passed" -ForegroundColor Green
Write-Host "Failed: $failed" -ForegroundColor $(if ($failed -gt 0) { "Red" } else { "Green" })
if ($failed -gt 0) { Write-Host "`nSome checks failed. See above." -ForegroundColor Red; exit 1 }
else { Write-Host "`nAll checks passed. Lab ready for demo." -ForegroundColor Green }
'@ | Set-Content -Path "scripts\verify.ps1" -Encoding UTF8
```

**File: scripts\teardown.ps1**

```powershell
@'
# teardown.ps1 -- Stop and clean up Day 2 Lab
param([switch]$RemoveVolumes, [switch]$RemoveImages)

$projectRoot = Split-Path -Parent $PSScriptRoot
Set-Location "$projectRoot\docker"

Write-Host "`n[TEARDOWN] Stopping all services..." -ForegroundColor Yellow
if ($RemoveVolumes) { docker-compose down -v }
else                { docker-compose down }

if ($RemoveImages) {
    Write-Host "[TEARDOWN] Removing built images..." -ForegroundColor Yellow
    docker rmi nivesh-investor-service nivesh-product-service 2>$null
}

Write-Host "[OK] Teardown complete." -ForegroundColor Green
Write-Host "Full reset: .\scripts\teardown.ps1 -RemoveVolumes -RemoveImages" -ForegroundColor Yellow
'@ | Set-Content -Path "scripts\teardown.ps1" -Encoding UTF8
```

---

## L4. STEP-BY-STEP EXECUTION GUIDE

### L4.1 First-Time Build

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"
.\scripts\setup.ps1
```

Expected output ends with: `=== Day 2 Setup Complete ===`

If Maven build fails inside investor-service:

```powershell
Set-Location investor-service
mvn --no-transfer-progress dependency:go-offline
mvn --no-transfer-progress -q clean package -DskipTests
Set-Location ..
```

### L4.2 Starting the Stack

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd\docker"
docker-compose up -d --build
```

Wait 90 seconds for all services. Spring Boot takes time to connect to PostgreSQL and run Flyway migrations.

```powershell
Start-Sleep -Seconds 90
docker-compose ps
```

Expected: four containers all showing healthy or running.

If investor-service restarts:

```powershell
docker-compose logs investor-service --tail=40
# Most common cause: PostgreSQL not ready. Fix:
docker-compose restart investor-service
Start-Sleep -Seconds 30
```

### L4.3 Run Python Unit Tests (No Docker Needed)

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"
$env:PYTHONPATH = "$PWD"
pytest product-service/tests/ -v
```

Expected: All 12 tests pass.

Git Bash:

```bash
cd ~/projects/nivesh-gateway-ddd
PYTHONPATH=. pytest product-service/tests/ -v
```

### L4.4 Run Java Tests

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd\investor-service"
mvn --no-transfer-progress test
```

Expected: 7 tests pass.

### L4.5 Core Demo Commands — Individual Execution

**Create Investor (via API Gateway):**

```bash
curl -s -X POST http://localhost:8090/api/investors \
  -H "Content-Type: application/json" \
  -d '{
    "investorId":  "INV-000001",
    "fullName":    "Priya Krishnamurthy",
    "email":       "priya.k@nivesh.com",
    "nationality": "INDIAN",
    "riskProfile": "MODERATE"
  }' | jq
```

Expected: HTTP 201 with kycStatus: PENDING and tradingAllowed: false.

**Verify KYC:**

```bash
curl -s -X PATCH http://localhost:8090/api/investors/INV-000001/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus": "VERIFIED"}' | jq '{investorId,kycStatus,tradingAllowed}'
```

Expected: kycStatus: VERIFIED, tradingAllowed: true.

**List all products:**

```bash
curl -s http://localhost:8090/api/products | jq '[.[] | {productCode, riskClassification, expectedRoiPercent}]'
```

Expected: 5 seeded products.

**Filter products by risk profile:**

```bash
curl -s "http://localhost:8090/api/products?riskProfile=MODERATE" | jq length
# Expected: 2 (CONSERVATIVE + MODERATE products)

curl -s "http://localhost:8090/api/products?riskProfile=AGGRESSIVE" | jq length
# Expected: 5 (all products)
```

**Cross-context eligible products call:**

```bash
curl -s http://localhost:8090/api/investors/INV-000001/eligible-products | jq \
  '{investorId, kycStatus, riskProfile, productCount}'
```

Expected: MODERATE investor with VERIFIED KYC sees conservative and moderate products.

**Intentional failure — invalid KYC state transition:**

```bash
# Priya is already VERIFIED -- reverting to PENDING is a domain violation
curl -s -X PATCH http://localhost:8090/api/investors/INV-000001/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus": "PENDING"}' | jq
```

Expected: HTTP 422 Unprocessable Entity with domain rule message.

**View Product Service Swagger UI:**

Open browser: `http://localhost:8000/docs`

This shows the auto-generated OpenAPI documentation — the API-First contract made visible.

**Verify separate databases:**

```powershell
# investor_db -- shows investors table
docker exec nivesh-postgres psql -U nivesh_user -d investor_db `
    -c "SELECT investor_id, full_name, kyc_status, risk_profile FROM investors;"

# product_db -- shows investment_products table
docker exec nivesh-postgres psql -U nivesh_user -d product_db `
    -c "SELECT product_code, product_name, risk_classification FROM investment_products;"
```

Teaching point: Two databases. Two schemas. Zero shared tables. This is bounded context isolation at the infrastructure level.

**Validate Terraform:**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd\infra\terraform"
terraform init -input=false
terraform validate
terraform plan -input=false
Set-Location "$HOME\projects\nivesh-gateway-ddd"
```

---

## L5. PRE-DEMO CHECKS — TRAINER CHECKLIST

Allow 15 minutes before class starts. Run all checks in sequence.

**Start the stack:**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd\docker"
docker-compose up -d
Start-Sleep -Seconds 90
```

**Run full verification:**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"
.\scripts\verify.ps1
```

All checks must show PASS before proceeding.

**Manual pre-demo checklist:**

| Check                | Command                                                                | Expected              |
| -------------------- | ---------------------------------------------------------------------- | --------------------- |
| 4 containers running | `docker-compose ps`                                                    | 4 rows, all Up        |
| investor_db ready    | `docker exec nivesh-postgres pg_isready -U nivesh_user -d investor_db` | accepting connections |
| product_db ready     | `docker exec nivesh-postgres pg_isready -U nivesh_user -d product_db`  | accepting connections |
| Investor Service UP  | `curl -s http://localhost:8080/actuator/health`                        | `{"status":"UP"}`     |
| Product Service UP   | `curl -s http://localhost:8000/health`                                 | `{"status":"UP"}`     |
| API Gateway UP       | `curl -s http://localhost:8090/gateway/health`                         | `{"status":"UP"}`     |
| Products seeded      | `curl -s http://localhost:8000/api/products \| jq length`              | 5                     |
| Swagger UI loads     | Browser: http://localhost:8000/docs                                    | Swagger UI renders    |

**Clean investor data before demo if needed:**

```powershell
docker exec nivesh-postgres psql -U nivesh_user -d investor_db `
    -c "DELETE FROM investors WHERE investor_id LIKE 'INV-VER%';"
```

---

## L6. DEMO PLAYBOOK — STEP-BY-STEP FOR TRAINER

**DEMO STEP 1 — Show the architecture (3 minutes)**

Show docker-compose ps. Four containers running. Point out the key architectural decision on screen:

"Notice there is no shared database. investor-service talks to investor_db. product-service talks to product_db. The API Gateway routes requests to the right service based on path prefix. This is not a microservices trend — this is bounded context isolation enforced at the infrastructure level."

Show both databases have separate tables:

```powershell
docker exec nivesh-postgres psql -U nivesh_user -d investor_db -c "\dt"
docker exec nivesh-postgres psql -U nivesh_user -d product_db -c "\dt"
```

"Two databases. Different schemas. Different ownership. This is what 'separate bounded context' means in practice."

---

**DEMO STEP 2 — API-First: Show the contract before the code (2 minutes)**

Open browser: `http://localhost:8000/docs`

"Before I wrote a single line of service code today, we wrote two OpenAPI contracts — one for investors, one for products. What you see here is that contract rendered live. An architect who cannot show you the API contract of a service they designed has not designed it — they have implemented it."

Show the POST /api/products endpoint in Swagger. Point out the field-level validation rules visible in the schema.

---

**DEMO STEP 3 — Hexagonal Architecture: Show the dependency direction (2 minutes)**

Open VS Code or a terminal and show the package structure of investor-service:

"The domain model — Investor, KycStatus, RiskProfile — has zero imports from Spring, JPA, or PostgreSQL. The controller depends on InvestorUseCase, the interface. Not on InvestorApplicationService, the implementation. The JPA entity is in the adapter, not the domain. This is hexagonal architecture: dependencies always point inward, toward the domain."

Show this quickly by `cat` or `type` of KycStatus.java:

```powershell
Get-Content "investor-service\src\main\java\com\nivesh\investor\domain\model\KycStatus.java"
```

"Pure Java. No Spring annotation. No JPA annotation. Testable in isolation."

---

**DEMO STEP 4 — Create investors (3 minutes)**

Run the create investor command via the API Gateway (port 8090 — not 8080 directly):

```bash
curl -s -X POST http://localhost:8090/api/investors \
  -H "Content-Type: application/json" \
  -d '{"investorId":"INV-000001","fullName":"Priya Krishnamurthy","email":"priya.k@nivesh.com","nationality":"INDIAN","riskProfile":"MODERATE"}' | jq
```

Point at tradingAllowed: false. "A newly onboarded investor cannot trade. KYC is PENDING. The domain enforces this — not a database flag, not a controller if-statement. The `allowsTrading()` method on KycStatus makes it impossible to represent a trading-eligible investor without VERIFIED KYC."

---

**DEMO STEP 5 — Domain business rule: KYC lifecycle (3 minutes)**

Verify KYC:

```bash
curl -s -X PATCH http://localhost:8090/api/investors/INV-000001/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus": "VERIFIED"}' | jq '{kycStatus,tradingAllowed}'
```

Now attempt an illegal state transition:

```bash
curl -s -X PATCH http://localhost:8090/api/investors/INV-000001/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus": "PENDING"}' | jq
```

Show the 422 error. "The aggregate root — the Investor class — enforces this rule in its `updateKycStatus` method. Not the database. Not the controller. The domain model. If someone removes that check from the domain, the tests break. That is how you govern business rules."

---

**DEMO STEP 6 — Cross-context call with ACL (4 minutes)**

```bash
curl -s http://localhost:8090/api/investors/INV-000001/eligible-products | jq
```

Walk through what just happened:

"The client called the Investor Service. The Investor Service loaded the investor from its own database. It called the Product Service through the Anti-Corruption Layer — `ProductCatalogueAdapter` — using a plain HTTP call. It received a product list. It applied its own eligibility filter using `investor.canAccessProductWithRisk()`. Then it returned the result. Two bounded contexts. Zero shared databases. One clean API boundary."

Show what happens when you create an AGGRESSIVE investor and compare eligible product counts:

```bash
curl -s -X POST http://localhost:8090/api/investors \
  -H "Content-Type: application/json" \
  -d '{"investorId":"INV-000002","fullName":"Wei Ming Tan","email":"weiming@test.com","nationality":"SINGAPOREAN","riskProfile":"AGGRESSIVE"}' | jq .investorId

curl -s http://localhost:8090/api/investors/INV-000002/eligible-products | jq .productCount
# Wei Ming sees 0 -- KYC not verified yet. Demonstrate domain rule.
```

---

**DEMO STEP 7 — Candidate exercise: live ADR creation (4 minutes)**

Ask one candidate to articulate ADR-004: "Why did we choose separate databases for bounded contexts instead of separate schemas in one database?"

Facilitate the group discussion. Expected points: blast radius, credential isolation, independent scaling, no accidental cross-context queries, separate migration history.

Write the ADR on the whiteboard using the ADR template from Day 1.

---

**DEMO STEP 8 — Run verification script (2 minutes)**

```powershell
.\scripts\verify.ps1
```

All checks pass. "This is the fitness function for today's architecture. Every check maps to a decision we made — separate databases, KYC enforcement, cross-context ACL, API-First contract. If any check fails, an architectural decision was violated."

---

## L7. VERIFICATION CHECKLIST

Run each check in sequence. All must pass before marking lab complete.

**All containers healthy:**

```powershell
docker-compose ps
# Expected: 4 containers, all Up
```

**investor_db and product_db both exist:**

```powershell
docker exec nivesh-postgres psql -U nivesh_user -c "\l" | Select-String "investor_db|product_db"
```

**Investor Service health:**

```bash
curl -s http://localhost:8080/actuator/health | jq .status
# Expected: "UP"
```

**Product Service health:**

```bash
curl -s http://localhost:8000/health | jq .status
# Expected: "UP"
```

**API Gateway health:**

```bash
curl -s http://localhost:8090/gateway/health | jq .status
# Expected: "UP"
```

**Investor creation via gateway returns 201:**

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8090/api/investors \
  -H "Content-Type: application/json" \
  -d '{"investorId":"INV-CHK-01","fullName":"Check User","email":"chk01@test.com","nationality":"INDIAN","riskProfile":"CONSERVATIVE"}'
# Expected: 201
```

**Invalid investor ID format returns 400:**

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:8090/api/investors \
  -H "Content-Type: application/json" \
  -d '{"investorId":"BADFORMAT","fullName":"X","email":"x@x.com","nationality":"INDIAN","riskProfile":"MODERATE"}'
# Expected: 400
```

**Products catalogue is seeded:**

```bash
curl -s http://localhost:8000/api/products | jq length
# Expected: 5
```

**MODERATE filter returns subset:**

```bash
curl -s "http://localhost:8000/api/products?riskProfile=MODERATE" | jq length
# Expected: 2
```

**KYC state transition enforced:**

```bash
# Update to VERIFIED
curl -s -X PATCH http://localhost:8090/api/investors/INV-CHK-01/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus":"VERIFIED"}' | jq .tradingAllowed
# Expected: true

# Try illegal revert
curl -s -o /dev/null -w "%{http_code}" \
  -X PATCH http://localhost:8090/api/investors/INV-CHK-01/kyc \
  -H "Content-Type: application/json" \
  -d '{"kycStatus":"PENDING"}'
# Expected: 422
```

**investor_db investors table populated:**

```powershell
docker exec nivesh-postgres psql -U nivesh_user -d investor_db `
    -c "SELECT investor_id, kyc_status FROM investors ORDER BY id;"
```

**product_db products table populated:**

```powershell
docker exec nivesh-postgres psql -U nivesh_user -d product_db `
    -c "SELECT product_code, risk_classification FROM investment_products ORDER BY id;"
```

**Python unit tests all pass:**

```powershell
$env:PYTHONPATH = "$HOME\projects\nivesh-gateway-ddd"
pytest "$HOME\projects\nivesh-gateway-ddd\product-service\tests\" -v
# Expected: 12 passed
```

**Terraform valid:**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd\infra\terraform"
terraform validate
# Expected: Success! The configuration is valid.
Set-Location "$HOME\projects\nivesh-gateway-ddd"
```

---

## L8. TROUBLESHOOTING GUIDE

**Problem: investor-service fails with "Connection refused to postgres:5432"**

The container started before PostgreSQL was ready despite depends_on. Fix:

```powershell
docker-compose restart investor-service
Start-Sleep -Seconds 45
docker-compose logs investor-service --tail=20
```

**Problem: init-multiple-dbs.sh fails with "/bin/bash^M: bad interpreter"**

Windows line endings in the shell script. Fix:

```powershell
$content = Get-Content "docker\init-multiple-dbs.sh" -Raw
$content = $content -replace "`r`n", "`n"
[System.IO.File]::WriteAllText(
    (Resolve-Path "docker\init-multiple-dbs.sh").Path,
    $content, [System.Text.Encoding]::UTF8
)
docker-compose down -v
docker-compose up -d
```

**Problem: product_db does not exist after startup**

The init script did not run. This happens when the postgres volume already exists from a previous run. Fix:

```powershell
docker-compose down -v   # removes the volume
docker-compose up -d     # recreates from scratch with init script
```

**Problem: Python unit tests fail with "ModuleNotFoundError: No module named 'src'"**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"
$env:PYTHONPATH = "$PWD"
pytest product-service/tests/ -v
```

Git Bash:

```bash
cd ~/projects/nivesh-gateway-ddd
PYTHONPATH=. pytest product-service/tests/ -v
```

**Problem: Product Service returns 500 on startup**

Usually a database connection issue. Check:

```powershell
docker-compose logs product-service --tail=30
docker exec nivesh-postgres psql -U nivesh_user -d product_db -c "\dt"
```

If product_db has no tables, the SQLAlchemy `create_all` failed. Restart:

```powershell
docker-compose restart product-service
Start-Sleep -Seconds 20
```

**Problem: API Gateway returns 502 Bad Gateway**

One of the upstream services is not ready. Check:

```powershell
curl -s http://localhost:8080/actuator/health
curl -s http://localhost:8000/health
docker-compose logs api-gateway --tail=10
```

Wait for both services to be UP, then retry through the gateway.

**Problem: eligible-products returns empty list unexpectedly**

Two possible causes:

Cause 1: KYC not VERIFIED. Check investor KYC status:

```bash
curl -s http://localhost:8090/api/investors/{investorId} | jq .kycStatus
```

Cause 2: Product Service unreachable from Investor Service. The ACL returns empty on failure. Check:

```powershell
docker-compose logs investor-service | Select-String "Product Catalogue unreachable"
```

Fix: ensure product-service is healthy, then retry.

**Problem: Port 8090 conflicts with another application**

Change the gateway port in docker-compose.yml from `8090:8090` to `9090:8090`. Update all demo commands accordingly.

**Problem: Maven build fails with "package com.nivesh.investor does not exist"**

Ensure all Java source files are created before running mvn. Run setup.ps1 which creates all files and then builds.

---

## L9. POST-DEMO CLEANUP

**Standard cleanup — keeps PostgreSQL data:**

```powershell
Set-Location "$HOME\projects\nivesh-gateway-ddd"
.\scripts\teardown.ps1
```

**Full reset — removes all data:**

```powershell
.\scripts\teardown.ps1 -RemoveVolumes
```

**Full reset including images:**

```powershell
.\scripts\teardown.ps1 -RemoveVolumes -RemoveImages
```

**Verify cleanup:**

```powershell
docker ps -a | Select-String "nivesh"
# Expected: no results
```

---

## L10. DOCUMENTATION SECTION

### L10.1 Architecture Decision Documentation

Why separate databases instead of separate schemas in one PostgreSQL instance:

A separate database enforces bounded context isolation at every layer: separate credentials prevent accidental cross-context queries at the connection level, separate Flyway migration histories prevent schema coupling, separate backup and restore policies are possible, and separate scaling decisions can be made independently in production. A separate schema in one database would work but requires discipline to maintain — the infrastructure does not enforce the boundary. Separate databases make the boundary physically impossible to violate without an explicit cross-database query.

Why hexagonal architecture in the Investor Service:

The Investor domain contains complex business rules (KYC lifecycle, eligibility logic, risk profile constraints) that must be testable without Spring, PostgreSQL, or any HTTP framework. Hexagonal architecture achieves this by keeping the domain model free of framework annotations and making all infrastructure dependencies flow through port interfaces. The 7 integration tests in InvestorControllerIntegrationTest cover the full path from HTTP to database. The domain rules (KycStatus.allowsTrading, RiskProfile.isEligibleFor) can be tested in plain JUnit without any infrastructure setup.

Why the Anti-Corruption Layer pattern for cross-context calls:

The Investor Service needs product data to answer eligibility questions. But it must not import Product Service classes, share a database with the Product context, or assume the Product Service API format will never change. The ProductCataloguePort defines what the domain needs. The ProductCatalogueAdapter translates the external API response into that format. When the Product Service changes its API (adding a field, renaming a property), only the adapter changes. The domain, application service, and controller are untouched.

### L10.2 Production Readiness Gap Analysis

| Lab Has                     | Production Needs                                                              | Effort |
| --------------------------- | ----------------------------------------------------------------------------- | ------ |
| Nginx gateway stub          | Azure API Management with policies, throttling, auth, caching                 | High   |
| WebClient with timeout only | Resilience4j circuit breaker, retry, bulkhead on all external calls           | Medium |
| Basic error handling in ACL | Dead letter queue for failed cross-context calls, compensation events         | High   |
| No service-to-service auth  | Mutual TLS between services, Azure Managed Identity, short-lived tokens       | High   |
| SQLAlchemy create_all       | Alembic migration history, rollback scripts, zero-downtime migration strategy | Medium |
| Seeded data in startup code | Separate data migration pipelines, not embedded in application startup        | Medium |
| Single PostgreSQL instance  | Azure Database for PostgreSQL Flexible Server per bounded context             | High   |

### L10.3 Regulatory Compliance Notes

| Regulation     | Relevance                                                                                      | Gap in Lab                             | Production Fix                                                              |
| -------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------- | --------------------------------------------------------------------------- |
| DPDP Act 2023  | Investor PII (name, email, nationality) requires consent, purpose limitation, right to erasure | No consent model, no data deletion API | Add consent_given field, soft-delete, purpose-limited access                |
| MAS TRM 2021   | KYC status is a regulated data element — changes must be audited                               | No audit log for KYC changes           | Add kyc_audit_log table with actor, timestamp, before/after                 |
| RBI Guidelines | Cross-border data transfers (investor data crossing India-Singapore boundary)                  | No data residency control              | Tag investor records by nationality, apply residency rules at storage layer |
| PDPA Singapore | Product catalogue data accessible to investors is a consent-scoped operation                   | No access logging                      | Add access event log per investor per product fetch                         |

### L10.4 Copilot Prompts for Self-Study

Understanding prompt:
"Explain the difference between InvestorRepository (output port) and InvestorJpaRepository (Spring Data interface) in this hexagonal architecture. Why are they separate? What breaks architecturally if we merge them into one interface?"

Extension prompt:
"How would I add an event publishing mechanism to the Investor Service so that when KYC status changes to VERIFIED, an event is published to a message queue for downstream services to consume? Describe where in the hexagonal architecture this belongs and what new ports and adapters are needed."

Critique prompt:
"Review the ProductCatalogueAdapter code. Identify: resilience gaps (what happens on timeout), security gaps (no auth on cross-service calls), and observability gaps (what is missing for production monitoring)."

India and Singapore context prompt:
"How would the Investor bounded context need to change to enforce India's DPDP Act 2023 data residency requirements while simultaneously supporting Singapore investors whose data must comply with MAS CDD guidelines? What are the architectural options and their trade-offs?"

---

✅ **DAY 2 LAB MANUAL COMPLETE**

Document 2: Lab Manual — Domain-Driven Design, API-First & Interoperability Workbench — Written for Windows 11 / PowerShell / Git Bash / Docker Desktop / PostgreSQL 15 / Java 17 / Spring Boot 3.3 / Python FastAPI / Nginx / Terraform 1.5+ / Azure Free Tier

Quality Gates QG-2 and QG-3: Passed