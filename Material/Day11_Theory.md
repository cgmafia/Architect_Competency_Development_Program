# Day 11: DevSecOps, Deployment & Excellence - Theory Document

## PART 2 OF 5

---

### Section B: Architecture and Design

#### High-Level Design (HLD) - Secure CI/CD Pipeline with SAST, DAST, SCA

```mermaid
flowchart TB
    subgraph Developer["Developer Workspace"]
        IDE["IDE with SonarLint<br/>(Real-time SAST)"]
        PreCommit["Pre-commit Hook<br/>(Secret Scanning)"]
        IDE --> PreCommit
    end

    subgraph SCM["Source Control - Git/GitHub"]
        Push["git push"]
        Webhook["Webhook Trigger"]
        Push --> Webhook
    end

    subgraph Pipeline["CI/CD Pipeline - GitHub Actions"]
        subgraph BuildStage["Build Stage"]
            Checkout["Checkout Code"]
            Build["Maven Build"]
            UnitTest["Unit Tests"]
            Checkout --> Build --> UnitTest
        end

        subgraph SASTStage["SAST Stage"]
            SonarScan["SonarQube Scanner"]
            QualityGate["Quality Gate Check"]
            SonarScan --> QualityGate
        end

        subgraph SCASTage["SCA Stage"]
            DepCheck["OWASP Dependency-Check"]
            TrivyFS["Trivy Filesystem Scan"]
            SBOMGen["Generate SBOM<br/>(CycloneDX)"]
            DepCheck --> SBOMGen
            TrivyFS --> SBOMGen
        end

        subgraph ContainerStage["Container Security"]
            DockerBuild["Docker Build"]
            TrivyImage["Trivy Image Scan"]
            CosignSign["Cosign Sign Image"]
            DockerBuild --> TrivyImage --> CosignSign
        end

        subgraph DeployStage["Deploy to Staging"]
            K8sDeploy["Deploy to K8s<br/>(Staging Namespace)"]
            HealthCheck["Health Check Wait"]
            K8sDeploy --> HealthCheck
        end

        subgraph DASTStage["DAST Stage"]
            ZAPScan["OWASP ZAP Baseline Scan"]
            ZAPReport["Generate DAST Report"]
            ZAPScan --> ZAPReport
        end

        subgraph ProdStage["Production Deployment"]
            PolicyCheck["OPA Policy Gate"]
            CanaryDeploy["Canary Deployment<br/>(5% traffic)"]
            Monitor["Monitor 15 min"]
            FullDeploy["Full Rollout"]
            PolicyCheck --> CanaryDeploy --> Monitor --> FullDeploy
        end

        BuildStage --> SASTStage --> SCASTage --> ContainerStage --> DeployStage --> DASTStage --> ProdStage
    end

    subgraph ExternalTools["External Security Tools"]
        SonarServer["SonarQube Server<br/>(Rules + History)"]
        NVD["NVD / OSV Database"]
        TrivyDB["Trivy Vulnerability DB"]
        ZAPProxy["ZAP Proxy"]
        OPAServer["OPA Server<br/>(Policy as Code)"]
        Vault["HashiCorp Vault<br/>(Secrets)"]
    end

    SonarScan -.-> SonarServer
    DepCheck -.-> NVD
    TrivyFS -.-> TrivyDB
    TrivyImage -.-> TrivyDB
    ZAPScan -.-> ZAPProxy
    PolicyCheck -.-> OPAServer
    BuildStage -.-> Vault

    subgraph Artifacts["Artifact Storage"]
        Registry["Container Registry<br/>(Signed Images)"]
        SBOMStore["SBOM Archive"]
        Reports["Security Reports"]
        CosignSign --> Registry
        SBOMGen --> SBOMStore
        SonarScan --> Reports
        ZAPScan --> Reports
    end

    PreCommit --> Push
```

**Design Decision Annotations:**

| Component                   | Decision                                 | Rationale                                                                     |
| --------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------- |
| SonarLint in IDE            | Shift-left SAST to developer desktop     | Catch issues before commit; reduces pipeline failures by 60%                  |
| Pre-commit secret scanning  | Block secrets before reaching remote     | Prevents credential exposure; GitHub reported 85% reduction in leaked secrets |
| SonarQube over alternatives | Enterprise-grade with quality gates      | Free tier supports 100K LOC; integrates with Maven natively                   |
| OWASP Dependency-Check      | Open-source SCA with NVD                 | No license cost; government-acceptable (NIST-maintained DB)                   |
| Trivy for containers        | Fast, comprehensive, OCI-compliant       | Scans in seconds vs. minutes for Clair; supports SBOM generation              |
| ZAP Baseline for DAST       | Automated, CI-friendly                   | Runs headless; produces JUnit-compatible reports for gating                   |
| OPA for policy enforcement  | Declarative, version-controlled policies | Separate policy from code; auditable; supports complex rules                  |
| Cosign for image signing    | Supply chain security                    | Sigstore ecosystem; keyless signing with OIDC                                 |

#### Design Rationale and Trade-off Analysis

**Recommended Approach: Layered Security Scanning Pipeline**

This approach implements defense-in-depth by applying multiple scanning techniques at different pipeline stages, each catching vulnerabilities the others miss.

**Alternative Approaches:**

| Aspect                 | Recommended (Layered)                      | Alternative 1 (DAST-Only) | Alternative 2 (All-in-One Platform)         |
| ---------------------- | ------------------------------------------ | ------------------------- | ------------------------------------------- |
| Description            | Separate best-of-breed tools at each stage | Only run DAST in staging  | Single platform (e.g., Snyk, Checkmarx One) |
| Vulnerability Coverage | 95%+                                       | 40-50%                    | 85-90%                                      |
| Time to Feedback       | Minutes (SAST) to Hours (DAST)             | Hours                     | Minutes                                     |
| Setup Complexity       | High (multiple integrations)               | Low                       | Medium                                      |
| License Cost           | Low (mostly OSS)                           | Low                       | High ($50-100K/year)                        |
| False Positive Rate    | Medium (tunable per tool)                  | Low                       | Medium-High                                 |
| Government Compliance  | Excellent (NIST-aligned)                   | Poor (no SAST/SCA)        | Good (certifications)                       |

**Trade-off Analysis:**

```
Trade-off 1: [Feedback Speed] vs [Coverage Completeness]
├── Faster feedback (SAST-only) = Developers fix quickly but miss runtime issues
├── Complete coverage (all three) = Slower pipeline but comprehensive security
└── Optimal: SAST/SCA in PR (fast), DAST in merge-to-main (slower but thorough)

Trade-off 2: [Setup Complexity] vs [Tool Optimization]
├── Single platform = Easy setup but limited tuning per scanning type
├── Best-of-breed = Complex setup but optimal configuration per tool
└── Optimal: Start with platform, migrate to best-of-breed at scale

Trade-off 3: [False Positive Rate] vs [False Negative Rate]
├── Aggressive rules = Catch more real issues but developer fatigue
├── Relaxed rules = Happy developers but missed vulnerabilities
└── Optimal: Baseline + incremental tightening with developer training
```

**ATAM Utility Tree Reference:**

```
                    Security Pipeline Quality Attributes
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
     [Security]               [Efficiency]            [Maintainability]
           │                       │                       │
     ┌─────┴─────┐           ┌─────┴─────┐           ┌─────┴─────┐
     │           │           │           │           │           │
 [Coverage]  [Compliance] [Speed]  [Cost]   [Flexibility][Usability]
   H(0.9)      H(0.8)     M(0.6)   H(0.8)     M(0.5)      M(0.6)

H = High priority (weight 0.7-1.0)
M = Medium priority (weight 0.4-0.6)
```

> **Architect's Note:** For US FedRAMP authorization, SAST/SCA are mandatory (Control SC-7, SI-2). Singapore IM8 requires "secure-by-design" with automated scanning. India's CERT-In advisories mandate vulnerability scanning within 24 hours of CVE disclosure. The layered approach satisfies all three.

---

### Section C: Code Walkthrough

#### Implementation Walkthrough - SAST Integration with SonarQube

We'll implement a Spring Boot 3.x microservice with SAST configuration, demonstrating what SonarQube detects and how to remediate findings.

**Project Context:** A simple Citizen Document Service that accepts document uploads and metadata. We'll intentionally introduce vulnerabilities for SAST detection.

##### File 1: pom.xml - Maven Configuration with SonarQube Plugin

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

    <groupId>gov.sg.dcs</groupId>
    <artifactId>citizen-document-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>Citizen Document Service</name>
    <description>Secure document management for GovTech Singapore</description>

    <properties>
        <java.version>17</java.version>
        <!-- SonarQube configuration properties -->
        <sonar.organization>govtech-sg</sonar.organization>
        <sonar.projectKey>gov.sg.dcs:citizen-document-service</sonar.projectKey>
        <sonar.sourceEncoding>UTF-8</sonar.sourceEncoding>
        <sonar.java.source>17</sonar.java.source>
        <sonar.java.target>17</sonar.java.target>
        <sonar.coverage.jacoco.xmlReportPaths>
            ${project.build.directory}/site/jacoco/jacoco.xml
        </sonar.coverage.jacoco.xmlReportPaths>
        <!-- Quality gate thresholds - enforced in pipeline -->
        <sonar.qualitygate.wait>true</sonar.qualitygate.wait>
        <!-- Fail build if quality gate fails -->
        <sonar.qualitygate.fail>true</sonar.qualitygate.fail>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- PostgreSQL Driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- NOTE: Intentionally using older version with known CVE for SCA demo -->
        <!-- In production, always use latest stable versions -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.13.0</version>
            <!-- SCA will flag: CVE-2022-42003, CVE-2022-42004 -->
        </dependency>

        <!-- Test Dependencies -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Spring Boot Maven Plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!-- JaCoCo for Code Coverage - Required by SonarQube -->
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.11</version>
                <executions>
                    <!-- Prepare agent for coverage data collection -->
                    <execution>
                        <id>prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <!-- Generate report after tests -->
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- SonarQube Scanner Plugin -->
            <plugin>
                <groupId>org.sonarsource.scanner.maven</groupId>
                <artifactId>sonar-maven-plugin</artifactId>
                <version>3.10.0.2594</version>
            </plugin>

            <!-- OWASP Dependency-Check for SCA -->
            <plugin>
                <groupId>org.owasp</groupId>
                <artifactId>dependency-check-maven</artifactId>
                <version>9.0.8</version>
                <configuration>
                    <!-- Fail build on CVSS >= 7 (High severity) -->
                    <failBuildOnCVSS>7</failBuildOnCVSS>
                    <!-- Use NVD as primary source -->
                    <nvdApiDelay>5000</nvdApiDelay>
                    <!-- Generate HTML and JSON reports -->
                    <format>HTML,JSON</format>
                    <!-- Suppress false positives -->
                    <suppressionFile>
                        ${project.basedir}/dependency-check-suppressions.xml
                    </suppressionFile>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>check</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- Maven Compiler Plugin - Explicit Java 17 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <!-- Enable preview features if needed -->
                    <compilerArgs>
                        <arg>-Xlint:all</arg>
                        <arg>-Werror</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

**What happens under the hood:**
- `sonar-maven-plugin` attaches to the `verify` phase and sends compiled bytecode, source files, and JaCoCo reports to the SonarQube server
- `sonar.qualitygate.wait=true` makes Maven block until SonarQube finishes analysis
- `sonar.qualitygate.fail=true` exits with non-zero code if quality gate fails
- JaCoCo prepares a Java agent that instruments bytecode during test execution, recording which lines/branches were covered

##### File 2: VulnerableController.java - Intentional SAST Violations

```java
package gov.sg.dcs.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import jakarta.servlet.http.HttpServletRequest;
import java.io.*;
import java.sql.*;

/**
 * VULNERABLE CONTROLLER - For SAST Demonstration Only
 * 
 * This controller intentionally contains multiple security vulnerabilities
 * that SonarQube will detect. Each vulnerability is annotated with:
 * - VULN-ID: Reference for discussion
 * - EXPECTED-SAST-FINDING: What SonarQube should report
 * - REMEDIATION: How to fix it
 */
@RestController
@RequestMapping("/api/v1/documents")
public class VulnerableController {

    // ========================================================================
    // VULN-001: SQL INJECTION
    // EXPECTED-SAST-FINDING: "A prepared statement is used to execute a 
    // query constructed with untrusted input" or similar SQL injection rule
    // CWE-89: Improper Neutralization of Special Elements in SQL
    // ========================================================================
    
    @GetMapping("/search")
    public String searchDocuments(
            @RequestParam String citizenId,
            @RequestParam(required = false) String docType,
            HttpServletRequest request) throws Exception {
        
        // Anti-pattern: Direct string concatenation with user input
        // SonarQube Rule: java:S2077 - Strings should not be concatenated 
        // with SQL queries
        
        String query = "SELECT * FROM documents WHERE citizen_id = '" 
                       + citizenId + "'";
        
        if (docType != null) {
            // Additional injection point
            query += " AND doc_type = '" + docType + "'";
        }
        
        // Simulated database connection (in real code, use connection pool)
        Connection conn = DriverManager.getConnection(
            "jdbc:postgresql://localhost:5432/dcs", "admin", "admin123");
        
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery(query); // VULNERABLE EXECUTION
        
        StringBuilder result = new StringBuilder();
        while (rs.next()) {
            result.append(rs.getString("doc_name")).append(",");
        }
        
        conn.close();
        return result.toString();
    }

    // ========================================================================
    // VULN-002: COMMAND INJECTION
    // EXPECTED-SAST-FINDING: "Operating system command executed with 
    // untrusted input"
    // CWE-78: Improper Neutralization of Special Elements in OS Command
    // ========================================================================
    
    @PostMapping("/convert")
    public String convertDocument(
            @RequestParam String fileName,
            @RequestParam String outputFormat) throws Exception {
        
        // Anti-pattern: User input directly in command execution
        // SonarQube Rule: java:S4822 - Call to Runtime.exec() should 
        // be sanitized
        
        String command = "libreoffice --headless --convert-to " 
                         + outputFormat + " --outdir /tmp " 
                         + "/uploads/" + fileName;
        
        // VULNERABLE: Attacker could pass:
        // fileName = "doc.pdf; rm -rf /"
        // outputFormat = "pdf; cat /etc/passwd"
        
        Process process = Runtime.getRuntime().exec(command);
        process.waitFor();
        
        return "Conversion complete";
    }

    // ========================================================================
    // VULN-003: PATH TRAVERSAL
    // EXPECTED-SAST-FINDING: "File path constructed with untrusted input"
    // CWE-22: Improper Limitation of a Pathname to a Restricted Directory
    // ========================================================================
    
    @GetMapping("/download")
    public byte[] downloadDocument(
            @RequestParam String documentName,
            @RequestParam String category) throws Exception {
        
        // Anti-pattern: User input in file path without validation
        // SonarQube Rule: java:S2756 - File paths should not be constructed 
        // from untrusted input
        
        String basePath = "/data/documents/";
        String fullPath = basePath + category + "/" + documentName;
        
        // VULNERABLE: Attacker could pass:
        // category = "../../etc"
        // documentName = "passwd"
        // Result: /data/documents/../../etc/passwd = /etc/passwd
        
        File file = new File(fullPath);
        FileInputStream fis = new FileInputStream(file);
        byte[] content = fis.readAllBytes();
        fis.close();
        
        return content;
    }

    // ========================================================================
    // VULN-004: HARD-CODED CREDENTIALS
    // EXPECTED-SAST-FINDING: "Credentials should not be hard-coded"
    // CWE-798: Use of Hard-coded Credentials
    // ========================================================================
    
    // SonarQube Rule: java:S2068 - Credentials should not be hard-coded
    
    private static final String DB_PASSWORD = "SuperSecret123!"; // VULNERABLE
    private static final String API_KEY = "AKIAIOSFODNN7EXAMPLE"; // VULNERABLE
    private static final String ENCRYPTION_KEY = "MySecretKey12345678"; // VULNERABLE

    @PostMapping("/encrypt")
    public String encryptData(@RequestBody String data) {
        // Using hard-coded encryption key
        return "encrypted:" + data.hashCode(); // Simplified for demo
    }

    // ========================================================================
    // VULN-005: INSECURE RANDOM
    // EXPECTED-SAST-FINDING: "Secure random should be used instead of 
    // random"
    // CWE-330: Use of Insufficiently Random Values
    // ========================================================================
    
    @PostMapping("/generate-token")
    public String generateToken(@RequestParam String citizenId) {
        // Anti-pattern: java.util.Random is not cryptographically secure
        // SonarQube Rule: java:S2245 - Privilege escalation possible 
        // when using non-cryptographic random
        
        java.util.Random random = new java.util.Random(citizenId.hashCode());
        long token = random.nextLong();
        
        // VULNERABLE: Predictable token generation
        // Should use java.security.SecureRandom
        return Long.toHexString(token);
    }

    // ========================================================================
    // VULN-006: EXCEPTION INFORMATION LEAKAGE
    // EXPECTED-SAST-FINDING: "Exceptions should not be exposed to clients"
    // CWE-209: Generation of Error Message Containing Sensitive Info
    // ========================================================================
    
    @GetMapping("/metadata/{id}")
    public String getMetadata(@PathVariable Long id) {
        try {
            // Business logic that might throw
            if (id < 0) {
                throw new IllegalArgumentException("Invalid document ID: " + id);
            }
            return "Metadata for document " + id;
        } catch (Exception e) {
            // Anti-pattern: Returning full exception details to client
            // SonarQube Rule: java:S5669 - Exception details should not be 
            // logged or returned in responses
            
            return "Error: " + e.getClass().getName() 
                   + " - " + e.getMessage()
                   + " at " + e.getStackTrace()[0];
        }
    }

    // ========================================================================
    // VULN-007: LOG INJECTION
    // EXPECTED-SAST-FINDING: "User input should not be logged directly"
    // CWE-117: Improper Output Neutralization for Logs
    // ========================================================================
    
    private static final org.slf4j.Logger log = 
        org.slf4j.LoggerFactory.getLogger(VulnerableController.class);
    
    @PostMapping("/upload")
    public String uploadDocument(
            @RequestParam String citizenId,
            @RequestParam MultipartFile file) {
        
        // Anti-pattern: User input directly in log without sanitization
        // SonarQube Rule: java:S5164 - User input should be sanitized 
        // before logging
        
        log.info("Document upload requested by citizen: {}", citizenId);
        // If citizenId contains newlines, attacker can inject fake log entries
        // e.g., citizenId = "validUser\n2024-01-15 ERROR Database corrupted"
        
        log.debug("File details: name={}, size={}, contentType={}", 
                  file.getOriginalFilename(), 
                  file.getSize(), 
                  file.getContentType());
        
        return "Upload received";
    }
}
```

**What SonarQube Detects Under the Hood:**

1. **SQL Injection Detection (java:S2077):**
   - SonarQube parses the string concatenation
   - Identifies `citizenId` and `docType` as tainted (from `@RequestParam`)
   - Traces data flow to `stmt.executeQuery(query)`
   - Flags as critical vulnerability

2. **Command Injection Detection (java:S4822):**
   - Detects `Runtime.exec()` call
   - Traces `fileName` and `outputFormat` as tainted
   - Flags as critical vulnerability

3. **Hard-coded Credentials (java:S2068):**
   - Pattern matching for variables with names containing "password", "key", "secret"
   - Checks literal string values against common patterns
   - Flags regardless of variable visibility (even `private`)

##### File 3: SecureController.java - Remediated Version

```java
package gov.sg.dcs.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Pattern;
import java.io.*;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.SecureRandom;
import java.util.List;
import java.util.UUID;

/**
 * SECURE CONTROLLER - Remediated Version
 * 
 * This controller demonstrates the fixed versions of all vulnerabilities
 * found in VulnerableController. Compare with VulnerableController to
 * understand SAST remediation patterns.
 */
@RestController
@RequestMapping("/api/v1/documents/secure")
public class SecureController {

    private final JdbcTemplate jdbcTemplate;
    private final SecureRandom secureRandom;
    
    // REMEDIATION VULN-004: Externalized configuration via Spring
    @Value("${dcs.encryption.key-vault-ref}")
    private String encryptionKeyVaultRef; // Reference to Vault, not actual key
    
    @Value("${dcs.documents.base-path}")
    private String documentsBasePath;
    
    // Allowed file extensions whitelist
    private static final List<String> ALLOWED_FORMATS = List.of("pdf", "docx", "xlsx");
    private static final List<String> ALLOWED_CATEGORIES = List.of("identity", "tax", "health");

    public SecureController(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        // REMEDIATION VULN-005: Use SecureRandom for cryptographic operations
        this.secureRandom = new SecureRandom();
    }

    // ========================================================================
    // REMEDIATION VULN-001: Parameterized Query
    // SonarQube will NOT flag this - data flow is safe
    // ========================================================================
    
    @GetMapping("/search")
    public List<DocumentDTO> searchDocuments(
            @RequestParam 
            @Pattern(regexp = "^[A-Z0-9]{8,12}$", message = "Invalid citizen ID format")
            String citizenId,
            @RequestParam(required = false) 
            @Pattern(regexp = "^[A-Z]{3,20}$", message = "Invalid document type")
            String docType) {
        
        // REMEDIATION: Use parameterized query with JdbcTemplate
        // The ? placeholder is safely bound by the JDBC driver
        StringBuilder sql = new StringBuilder(
            "SELECT id, citizen_id, doc_type, doc_name, created_at " +
            "FROM documents WHERE citizen_id = ?"
        );
        
        if (docType != null) {
            sql.append(" AND doc_type = ?");
        }
        
        RowMapper<DocumentDTO> rowMapper = (rs, rowNum) -> new DocumentDTO(
            rs.getLong("id"),
            rs.getString("citizen_id"),
            rs.getString("doc_type"),
            rs.getString("doc_name"),
            rs.getTimestamp("created_at").toLocalDateTime()
        );
        
        if (docType != null) {
            return jdbcTemplate.query(sql.toString(), rowMapper, citizenId, docType);
        } else {
            return jdbcTemplate.query(sql.toString(), rowMapper, citizenId);
        }
    }

    // ========================================================================
    // REMEDIATION VULN-002: Input Validation + No Shell Execution
    // ========================================================================
    
    @PostMapping("/convert")
    public String convertDocument(
            @RequestParam 
            @Pattern(regexp = "^[a-zA-Z0-9_\\-\\.]+$", message = "Invalid filename")
            String fileName,
            @RequestParam 
            String outputFormat) throws ConversionException {
        
        // REMEDIATION STEP 1: Strict input validation
        if (!ALLOWED_FORMATS.contains(outputFormat.toLowerCase())) {
            throw new ConversionException("Unsupported format: " + outputFormat);
        }
        
        // REMEDIATION STEP 2: Use safe path construction
        Path safeFileName = Paths.get(fileName).getFileName(); // Strips path components
        Path safePath = Paths.get("/uploads").resolve(safeFileName);
        
        // REMEDIATION STEP 3: Verify file exists and is within allowed directory
        if (!safePath.startsWith("/uploads")) {
            throw new ConversionException("Path traversal detected");
        }
        
        if (!safePath.toFile().exists()) {
            throw new ConversionException("File not found");
        }
        
        // REMEDIATION STEP 4: Use library instead of shell command
        // In production, use Apache POI, PDFBox, or similar libraries
        // This avoids shell injection entirely
        DocumentConverter converter = new DocumentConverter(outputFormat);
        String resultPath = converter.convert(safePath);
        
        return "Conversion complete: " + resultPath;
    }

    // ========================================================================
    // REMEDIATION VULN-003: Canonical Path Validation
    // ========================================================================
    
    @GetMapping("/download")
    public byte[] downloadDocument(
            @RequestParam 
            @Pattern(regexp = "^[a-zA-Z0-9_\\-\\.]+$", message = "Invalid filename")
            String documentName,
            @RequestParam String category) throws SecureFileException {
        
        // REMEDIATION STEP 1: Validate category against whitelist
        if (!ALLOWED_CATEGORIES.contains(category.toLowerCase())) {
            throw new SecureFileException("Invalid category");
        }
        
        // REMEDIATION STEP 2: Use Path API with canonical path check
        Path requestedPath = Paths.get(documentsBasePath, category, documentName)
                                  .normalize()
                                  .toAbsolutePath();
        
        Path basePath = Paths.get(documentsBasePath).toAbsolutePath();
        
        // REMEDIATION STEP 3: Verify the resolved path is within base directory
        if (!requestedPath.startsWith(basePath)) {
            throw new SecureFileException("Path traversal attempt detected");
        }
        
        // REMEDIATION STEP 4: Additional check - file must exist
        File file = requestedPath.toFile();
        if (!file.exists() || !file.isFile()) {
            throw new SecureFileException("Document not found");
        }
        
        try (FileInputStream fis = new FileInputStream(file)) {
            return fis.readAllBytes();
        } catch (IOException e) {
            throw new SecureFileException("Failed to read document");
        }
    }

    // ========================================================================
    // REMEDIATION VULN-005: SecureRandom for Tokens
    // ========================================================================
    
    @PostMapping("/generate-token")
    public String generateToken(@RequestParam String citizenId) {
        // REMEDIATION: Use SecureRandom for security-sensitive operations
        byte[] bytes = new byte[16];
        secureRandom.nextBytes(bytes);
        
        // Convert to hex string
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));
        }
        
        // Additional: Use UUID for request tracking (not security)
        String requestId = UUID.randomUUID().toString();
        
        return sb.toString();
    }

    // ========================================================================
    // REMEDIATION VULN-006: Generic Error Response
    // ========================================================================
    
    @GetMapping("/metadata/{id}")
    public DocumentMetadataResponse getMetadata(@PathVariable Long id) {
        try {
            if (id == null || id <= 0) {
                throw new BusinessException("DOC-001", "Invalid document identifier");
            }
            
            // Business logic here
            return new DocumentMetadataResponse(id, "Metadata retrieved");
            
        } catch (BusinessException e) {
            // REMEDIATION: Return structured error without stack trace
            throw e; // Handled by @ControllerAdvice
        } catch (Exception e) {
            // REMEDIATION: Log full details internally, return generic message
            log.error("Unexpected error retrieving metadata for id={}", id, e);
            throw new BusinessException("SYS-001", "An internal error occurred");
        }
    }

    // ========================================================================
    // REMEDIATION VULN-007: Sanitized Logging
    // ========================================================================
    
    private static final org.slf4j.Logger log = 
        org.slf4j.LoggerFactory.getLogger(SecureController.class);
    
    @PostMapping("/upload")
    public UploadResponse uploadDocument(
            @RequestParam 
            @Pattern(regexp = "^[A-Z0-9]{8,12}$", message = "Invalid citizen ID")
            String citizenId,
            @RequestParam MultipartFile file) {
        
        // REMEDIATION: Sanitize user input before logging
        // Option 1: Log only identifiers, not raw input
        log.info("Document upload requested, citizenId length={}, file size={}", 
                 citizenId.length(), 
                 file.getSize());
        
        // Option 2: If you must log the value, sanitize newlines
        String safeCitizenId = citizenId.replace("\n", "\\n")
                                        .replace("\r", "\\r");
        log.debug("Processing upload for citizen: {}", safeCitizenId);
        
        // REMEDIATION: Validate file name
        String originalName = file.getOriginalFilename();
        if (originalName != null && !originalName.matches("^[a-zA-Z0-9_\\-\\. ]+$")) {
            throw new BusinessException("DOC-002", "Invalid file name");
        }
        
        return new UploadResponse("RECEIVED", "Document queued for processing");
    }
}

// Supporting DTOs and Exceptions (SonarQube-compliant)

record DocumentDTO(Long id, String citizenId, String docType, 
                   String docName, java.time.LocalDateTime createdAt) {}

record DocumentMetadataResponse(Long documentId, String status) {}

record UploadResponse(String status, String message) {}

class ConversionException extends RuntimeException {
    public ConversionException(String message) {
        super(message);
    }
}

class SecureFileException extends RuntimeException {
    public SecureFileException(String message) {
        super(message);
    }
}

class BusinessException extends RuntimeException {
    private final String code;
    
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }
    
    public String getCode() {
        return code;
    }
}

// Stub for document converter
class DocumentConverter {
    private final String format;
    
    public DocumentConverter(String format) {
        this.format = format;
    }
    
    public String convert(Path input) {
        return "converted_" + input.getFileName() + "." + format;
    }
}
```

**What Changed Under the Hood (SAST Perspective):**

| Vulnerability          | Taint Source       | Original Sink         | Remediation                       | Why SAST is Satisfied                                                   |
| ---------------------- | ------------------ | --------------------- | --------------------------------- | ----------------------------------------------------------------------- |
| SQL Injection          | `@RequestParam`    | `Statement.execute()` | `JdbcTemplate.query()` with `?`   | JdbcTemplate binds parameters at JDBC level, untainted before execution |
| Command Injection      | `@RequestParam`    | `Runtime.exec()`      | Library-based conversion          | No shell execution path exists                                          |
| Path Traversal         | `@RequestParam`    | `FileInputStream`     | `Path.normalize()` + prefix check | Canonical path guaranteed within base directory                         |
| Hard-coded Credentials | N/A                | N/A                   | `@Value` from external config     | No literal string contains credential patterns                          |
| Insecure Random        | N/A                | Token generation      | `SecureRandom`                    | Class is on SonarQube's "safe" list for crypto                          |
| Exception Leak         | Internal exception | HTTP response         | `@ControllerAdvice` pattern       | Exception object never reaches response writer                          |
| Log Injection          | `@RequestParam`    | Logger                | Newline sanitization              | Sanitizer breaks injection pattern                                      |

---


### Section C: Code Walkthrough (Continued)

#### Implementation Walkthrough - SCA Integration

##### File 4: dependency-check-suppressions.xml - False Positive Management

SCA tools inevitably produce false positives. Suppression files allow architects to document why specific findings are accepted, creating an audit trail required for FedRAMP and IM8 compliance.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    
    <!--
        SUPPRESSION FILE FOR OWASP DEPENDENCY-CHECK
        =============================================
        Each suppression MUST include:
        - A unique packageUrl regex to identify the dependency
        - A CVE identifier to suppress
        - A justification explaining WHY this is acceptable
        - An expiry date after which the suppression must be reviewed
        
        REVIEW SCHEDULE: This file is reviewed quarterly by the Security Architect
        Last Review: 2024-01-15
        Next Review: 2024-04-15
        Reviewer: [Security Architect Name]
    -->

    <!-- 
        Suppression 1: H2 Database Test Scope
        Reason: H2 is only used in test scope, never reaches production
        CVE: CVE-2023-44487 (HTTP/2 Rapid Reset) - affects H2's web console
        Expiry: 2024-06-30 (upgrade to H2 2.2.224 planned)
    -->
    <suppress>
        <packageUrl regex="true">^pkg:maven/com\.h2database/h2@.*$</packageUrl>
        <cve>CVE-2023-44487</cve>
        <justification>
            H2 database is exclusively used in src/test scope for unit testing.
            The H2 web console is never enabled (spring.h2.console.enabled=false).
            This vulnerability cannot be exploited in our deployment model.
            Upgrade to H2 2.2.224 scheduled for Q2 2024 sprint.
            Approved by Security Architect on 2024-01-15.
        </justification>
        <expiryDate>2024-06-30</expiryDate>
    </suppress>

    <!--
        Suppression 2: Jackson-Databind (Intentional for Demo)
        Reason: This is intentionally vulnerable for SCA training demonstration
        CVE: CVE-2022-42003, CVE-2022-42004
        Expiry: NEVER (training dependency only)
        NOTE: This suppression would NEVER be approved in production
    -->
    <suppress>
        <packageUrl regex="true">^pkg:maven/com\.fasterxml\.jackson\.core/jackson-databind@2\.13\.0$</packageUrl>
        <cve>CVE-2022-42003</cve>
        <cve>CVE-2022-42004</cve>
        <justification>
            TRAINING ENVIRONMENT ONLY - This version is intentionally vulnerable
            for SCA demonstration purposes. This suppression is explicitly 
            blocked from merging to main branch via .gitignore and branch protection.
            In production, jackson-databind is managed by Spring Boot BOM.
        </justification>
        <expiryDate>2099-12-31</expiryDate>
    </suppress>

    <!--
        Suppression 3: Spring Framework CVE (Remediated by upgrade)
        Reason: Spring Boot 3.2.0 includes Spring Framework 6.1.1 which
                contains the fix for CVE-2023-6378
        CVE: CVE-2023-6378
        Expiry: 2024-12-31
    -->
    <suppress>
        <packageUrl regex="true">^pkg:maven/org\.springframework/spring-.*@6\.1\.1$</packageUrl>
        <cve>CVE-2023-6378</cve>
        <justification>
            False positive: Spring Framework 6.1.1 (included in Spring Boot 3.2.0)
            contains the fix for this DoS vulnerability. The NVD has not yet 
            updated the affected versions range to exclude 6.1.1+.
            Verified by checking Spring Security Advisory: 
            https://spring.io/security/cve-2023-6378
        </justification>
        <expiryDate>2024-12-31</expiryDate>
    </suppress>
</suppressions>
```

> **Architect's Note:** Every suppression in production MUST have an expiry date and review schedule. Singapore's IM8 guidelines require "evidence-based risk acceptance" - a suppression file without justification is not acceptable evidence. The US DISA STIG requires suppression review "at least annually or when major version changes occur."

##### File 5: trivy-config.yaml - Container Image Scanning Configuration

```yaml
# Trivy Configuration for Container Image Scanning
# Documentation: https://aquasecurity.github.io/trivy/latest/docs/configuration/

# Scan severity levels to report
severity:
  - CRITICAL
  - HIGH
  - MEDIUM

# Exit code when vulnerabilities are found
# 0: No vulnerabilities or only LOW
# 1: MEDIUM or higher found
exit-code: 1

# Vulnerability types to scan
vulnerability:
  # Include OS packages (Alpine, Debian, Ubuntu base images)
  type:
    - os
    - library

# Ignore unfixed vulnerabilities (no patch available)
# Set to false for government compliance - must track all known vulns
ignore-unfixed: false

# Output format for CI/CD parsing
format: json

# Output file path
output: trivy-report.json

# File to write results in SARIF format for GitHub integration
# sarif: trivy-report.sarif

# Ignore table for false positives (similar to dependency-check suppressions)
ignore-file: .trivyignore

# Cache directory for faster scans
cache-dir: .trivy-cache

# Timeout for scanning
timeout: 10m

# Skip directories/files
skip-dirs:
  - node_modules
  - .git
  - vendor

# Security misconfiguration scanning
misconfig:
  type:
    - dockerfile
    - kubernetes
    - terraform
```

##### File 6: .trivyignore - Trivy False Positive Management

```
# Trivy Ignore File
# Format: CVE_ID or CVE_ID:PATH
# Documentation: https://aquasecurity.github.io/trivy/latest/docs/configuration/filtering/

# Alpine Linux apk-tools (base image dependency, no upgrade path)
# CVE-2021-36159: Integer overflow in apk-tools
# Justification: Alpine 3.18 includes fix; this is for legacy images only
CVE-2021-36159

# libxml2 in Alpine base (affects XML parsing)
# CVE-2022-29824: Integer overflow in xmlBuf and xmlBuffer
# Justification: Our service does not parse untrusted XML; upgrade pending
CVE-2022-29824:usr/lib/libxml2.so.2

# Go stdlib net/http (transitive dependency)
# CVE-2023-39325: HTTP/2 rapid reset attack
# Justification: Go 1.21.3+ contains fix; verify go.mod version
CVE-2023-39325
```

#### Implementation Walkthrough - DAST with OWASP ZAP

##### File 7: zap-baseline-config.yaml - ZAP Baseline Scan Configuration

```yaml
# OWASP ZAP Baseline Scan Configuration
# Used for automated CI/CD scanning
# Documentation: https://www.zaproxy.org/docs/docker/baseline-scan/

# Target URL (passed via environment variable in CI)
# target: ${ZAP_TARGET_URL}

# Number of minutes to scan
# Baseline scans should complete in 5-10 minutes for CI/CD
minutes: 5

# AJAX spider enabled for SPAs
ajaxSpider: true

# Traditional spider configuration
spider:
  # Maximum depth to crawl
  maxDepth: 5
  # Number of threads
  threads: 5
  # URL patterns to exclude
  exclude: 
    - ".*logout.*"
    - ".*admin.*"
    - ".*\\.css$"
    - ".*\\.js$"
    - ".*\\.png$"
    - ".*\\.ico$"

# Active scan configuration
activeScan:
  # Strength of active scan: LOW, MEDIUM, HIGH, INSANE
  # Use MEDIUM for CI/CD (balances coverage vs. time)
  strength: MEDIUM
  # Rules to exclude (reduce false positives in CI)
  excludeRules:
    - "10038"  # CSRF - requires manual verification
    - "10055"  # CSP - often misconfigured intentionally for CDN
    - "10096"  # X-Frame-Options - may be intentional for embedding
  # Scan only specific contexts
  context:
    name: "citizen-document-service"
    includeUrls:
      - "http://staging-dcs.internal:8080/api/.*"

# Authentication configuration (for scanning authenticated endpoints)
authentication:
  # Form-based authentication
  type: "form"
  # Login URL
  loginUrl: "http://staging-dcs.internal:8080/api/v1/auth/login"
  # Login request body
  loginRequest: "username=${ZAP_USERNAME}&password=${ZAP_PASSWORD}"
  # Verification regex - confirms successful login
  verificationRegex: ".*\"token\".*"
  # CSRF token field (if required)
  csrfField: "_csrf"

# Users for authenticated scanning
users:
  - name: "citizen-user"
    credentials:
      username: "${ZAP_TEST_USER}"
      password: "${ZAP_TEST_PASS}"
  - name: "admin-user"
    credentials:
      username: "${ZAP_ADMIN_USER}"
      password: "${ZAP_ADMIN_PASS}"

# Report configuration
report:
  # Output format
  format: "html"
  # Output file
  file: "zap-report.html"
  # Directory for report
  dir: "reports/dast"

# Alerts configuration
alerts:
  # Thresholds for build failure
  thresholds:
    - high: 0      # Fail if any HIGH severity found
      medium: 5    # Allow up to 5 MEDIUM severity
      low: 10      # Allow up to 10 LOW severity
```

##### File 8: GitHub Actions Workflow - Complete Secure Pipeline

```yaml
# .github/workflows/secure-pipeline.yml
# Complete DevSecOps Pipeline for Citizen Document Service
# Implements SAST, SCA, Container Security, DAST with quality gates

name: Secure CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Environment variables available to all jobs
env:
  JAVA_VERSION: '17'
  JAVA_DISTRIBUTION: 'temurin'
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/citizen-document-service
  # Staging environment for DAST
  STAGING_NAMESPACE: dcs-staging
  ZAP_TARGET_URL: 'http://localhost:8080/api/v1'

# Cancel in-progress runs for same branch/PR
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # =========================================================================
  # JOB 1: BUILD AND UNIT TEST
  # =========================================================================
  build:
    name: Build & Unit Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for SonarQube blame data

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven

      - name: Cache SonarQube Packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Build and Test
        run: mvn clean verify -DskipTests=false -B
        env:
          # Test database configuration
          SPRING_DATASOURCE_URL: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
          SPRING_DATASOURCE_USERNAME: sa
          SPRING_DATASOURCE_PASSWORD: 

      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/
          retention-days: 7

      - name: Upload JaCoCo Report
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/
          retention-days: 7

  # =========================================================================
  # JOB 2: SAST - SonarQube Analysis
  # =========================================================================
  sast:
    name: SAST Analysis
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven

      - name: Download JaCoCo Report
        uses: actions/download-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/

      - name: SonarQube Scan
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }} \
            -Dsonar.projectKey=gov.sg.dcs:citizen-document-service \
            -Dsonar.branch.name=${{ github.head_ref || github.ref_name }} \
            -Dsonar.scm.revision=${{ github.sha }} \
            -Dsonar.pullrequest.key=${{ github.event.pull_request.number }} \
            -Dsonar.pullrequest.branch=${{ github.head_ref }} \
            -Dsonar.pullrequest.base=${{ github.base_ref }}
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

      - name: SonarQube Quality Gate
        uses: sonarsource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # =========================================================================
  # JOB 3: SCA - Dependency Vulnerability Scan
  # =========================================================================
  sca:
    name: SCA Analysis
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: ${{ env.JAVA_DISTRIBUTION }}
          cache: maven

      - name: OWASP Dependency-Check
        uses: dependency-check/Dependency-Check_Action@main
        id: dependency-check
        with:
          project: 'citizen-document-service'
          path: '.'
          format: 'HTML,JSON'
          args: >
            --suppression dependency-check-suppressions.xml
            --failOnCVSS 7
            --nvdApiDelay 5000
        continue-on-error: true  # Don't fail immediately, capture report

      - name: Upload Dependency-Check Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: dependency-check-report
          path: reports/
          retention-days: 30

      - name: Evaluate SCA Results
        run: |
          echo "Evaluating SCA scan results..."
          
          # Parse JSON report for critical/high vulnerabilities
          CRITICAL_COUNT=$(jq '[.dependencies[].vulnerabilities[]? | select(.severity == "CRITICAL")] | length' dependency-check-report.json 2>/dev/null || echo "0")
          HIGH_COUNT=$(jq '[.dependencies[].vulnerabilities[]? | select(.severity == "HIGH")] | length' dependency-check-report.json 2>/dev/null || echo "0")
          
          echo "Critical: $CRITICAL_COUNT"
          echo "High: $HIGH_COUNT"
          
          # Quality gate: Fail on critical, allow up to 2 high (with suppressions)
          if [ "$CRITICAL_COUNT" -gt 0 ]; then
            echo "::error::$CRITICAL_COUNT CRITICAL vulnerabilities found. Build failed."
            exit 1
          fi
          
          if [ "$HIGH_COUNT" -gt 2 ]; then
            echo "::warning::$HIGH_COUNT HIGH vulnerabilities found. Review required."
            # Don't fail, but warn - allows exceptions with approval
          fi
          
          echo "SCA quality gate passed."

      - name: Generate SBOM
        run: |
          # Install CycloneDX Maven Plugin run
          mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom \
            -DoutputFormat=json \
            -DoutputName=sbom \
            -DincludeCompileScope=true \
            -DincludeTestScope=false
          
          # Upload SBOM as artifact
          echo "SBOM generated: target/sbom.json"

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: target/sbom.json
          retention-days: 365  # SBOMs should be retained long-term

  # =========================================================================
  # JOB 4: CONTAINER SECURITY
  # =========================================================================
  container-security:
    name: Container Build & Scan
    needs: [sast, sca]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build Container Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          load: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # Build arguments for security
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}

      - name: Run Trivy Vulnerability Scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'image'
          image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'
          exit-code: '1'
          ignore-unfixed: false

      - name: Upload Trivy Scan Results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'trivy-container-scan'

      - name: Run Trivy Misconfiguration Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'table'
          severity: 'CRITICAL,HIGH,MEDIUM'
          exit-code: '1'
          ignore-unfixed: false

      - name: Sign Container with Cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign the Container Image
        run: |
          cosign sign --yes \
            --key env://COSIGN_PRIVATE_KEY \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        env:
          COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
          COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

      - name: Push Signed Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # =========================================================================
  # JOB 5: DAST - Deploy to Staging and Scan
  # =========================================================================
  dast:
    name: DAST Analysis
    needs: container-security
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: dcs_test
          POSTGRES_USER: dcs_test
          POSTGRES_PASSWORD: test_password_123
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Deploy to Staging (Local K8s via Docker Compose)
        run: |
          # In production, this would use kubectl apply to staging namespace
          # For CI, we use Docker Compose to simulate staging
          
          export IMAGE_TAG=${{ github.sha }}
          export SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/dcs_test
          export SPRING_DATASOURCE_USERNAME=dcs_test
          export SPRING_DATASOURCE_PASSWORD=test_password_123
          
          docker compose -f docker-compose.staging.yml up -d --wait
          
          # Wait for application to be ready
          echo "Waiting for application to start..."
          timeout 120 bash -c 'until curl -sf http://localhost:8080/actuator/health; do sleep 5; done'
          
          echo "Application is ready for DAST scanning"

      - name: Run ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: 'http://localhost:8080/api/v1'
          rules_file_name: 'zap-rules.tsv'
          cmd_options: '-a -j -t 5'  # Active scan, JSON output, 5 min timeout
          fail_action: true
          issue_report: 'zap-report.html'
          full_report: 'zap-full-report.html'
        continue-on-error: true

      - name: Upload ZAP Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: zap-dast-report
          path: |
            zap-report.html
            zap-full-report.html
          retention-days: 30

      - name: Evaluate DAST Results
        run: |
          # Parse ZAP report for high/critical alerts
          if [ -f "zap-report.html" ]; then
            # Count alerts by risk level from JSON (if available)
            HIGH_ALERTS=$(grep -o "High" zap-report.html | wc -l || echo "0")
            
            echo "High severity DAST findings: $HIGH_ALERTS"
            
            if [ "$HIGH_ALERTS" -gt 0 ]; then
              echo "::warning::DAST scan found $HIGH_ALERTS high-severity issues"
              echo "Review the ZAP report in Artifacts for details"
              # In production, this might fail the build
              # For now, we warn and require manual review
            fi
          fi

      - name: Teardown Staging
        if: always()
        run: |
          docker compose -f docker-compose.staging.yml down -v --remove-orphans

  # =========================================================================
  # JOB 6: SECURITY SUMMARY
  # =========================================================================
  security-summary:
    name: Security Gate Summary
    needs: [sast, sca, container-security, dast]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Collect Security Results
        run: |
          echo "============================================"
          echo "SECURITY PIPELINE SUMMARY"
          echo "============================================"
          echo ""
          echo "SAST (SonarQube):       ${{ needs.sast.result }}"
          echo "SCA (Dep-Check):        ${{ needs.sca.result }}"
          echo "Container (Trivy):      ${{ needs.container-security.result }}"
          echo "DAST (ZAP):             ${{ needs.dast.result }}"
          echo ""
          
          # Determine overall security gate status
          if [[ "${{ needs.sast.result }}" == "failure" ]] || \
             [[ "${{ needs.sca.result }}" == "failure" ]] || \
             [[ "${{ needs.container-security.result }}" == "failure" ]]; then
            echo "SECURITY GATE: FAILED"
            echo "::error::One or more security scans failed. Manual review required."
            exit 1
          fi
          
          if [[ "${{ needs.dast.result }}" == "failure" ]]; then
            echo "SECURITY GATE: WARNING (DAST findings require review)"
            # Don't fail on DAST - requires manual verification
          else
            echo "SECURITY GATE: PASSED"
          fi

      - name: Create Security Summary Issue (if failures)
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const title = `Security Scan Failures - ${context.sha.substring(0, 7)}`;
            const body = `## Security Pipeline Summary
            
            | Scan Type         | Status                                 |
            | ----------------- | -------------------------------------- |
            | SAST (SonarQube)  | ${{ needs.sast.result }}               |
            | SCA (Dep-Check)   | ${{ needs.sca.result }}                |
            | Container (Trivy) | ${{ needs.container-security.result }} |
            | DAST (ZAP)        | ${{ needs.dast.result }}               |
            
            ### Action Required
            1. Review the scan reports in the Actions run artifacts
            2. Fix or suppress findings with appropriate justification
            3. Re-run the pipeline after fixes
            
            **Commit:** ${context.sha}
            **Branch:** ${context.ref}
            **Run:** [View Details](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})
            `;
            
            await github.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: title,
              body: body,
              labels: ['security', 'automated', 'pipeline-failure']
            });
```

**What happens under the hood in this pipeline:**

1. **Job Dependencies (DAG):**
   - `build` runs first (no dependencies)
   - `sast` and `sca` run in parallel after `build` (independent scans)
   - `container-security` runs only after both `sast` and `sca` pass (security gate)
   - `dast` runs only after container is built and only on main branch (expensive)
   - `security-summary` always runs to provide consolidated status

2. **SonarQube Integration:**
   - `fetch-depth: 0` fetches full Git history for blame annotation
   - JaCoCo data is downloaded from build job artifact
   - Quality gate action polls SonarQube API until analysis completes

3. **Trivy in CI:**
   - Uses SARIF format to integrate with GitHub Security tab
   - `exit-code: 1` fails the job if vulnerabilities found
   - `ignore-unfixed: false` ensures all known CVEs are reported (government requirement)

4. **ZAP DAST:**
   - Uses PostgreSQL service container for realistic testing
   - `zaproxy/action-baseline` runs headless ZAP in container
   - `-a` flag enables active scanning (not just passive crawl)
   - `-t 5` limits scan to 5 minutes for CI time constraints

---

### Secret Scanning Deep Dive

#### Concept Foundation

**What is Secret Scanning?**

Secret scanning is the automated detection of sensitive data (API keys, passwords, tokens, certificates, private keys) in source code, configuration files, commit history, and build artifacts. Unlike SAST which looks for vulnerability patterns, secret scanning looks for specific data formats that represent credentials.

**Types of Secrets in Government Systems:**

| Secret Type                      | Format Pattern                   | Risk if Exposed                  |
| -------------------------------- | -------------------------------- | -------------------------------- |
| AWS Access Key                   | `AKIA[0-9A-Z]{16}`               | Full cloud infrastructure access |
| Azure Service Principal          | `GUID` + `secret`                | Azure tenant access              |
| PostgreSQL Connection String     | `postgresql://user:pass@host/db` | Database breach                  |
| JWT Signing Key                  | Base64 encoded 256+ bits         | Token forgery, identity spoofing |
| SSH Private Key                  | `-----BEGIN.*PRIVATE KEY-----`   | Server access, lateral movement  |
| India Aadhaar API Key            | Varies by provider               | PII of 1.4B citizens             |
| US FedRAMP Authorization Token   | Agency-specific                  | Federal system access            |
| Singapore SingPass Client Secret | UUID format                      | National ID system access        |
| mTLS Certificate + Key           | PEM encoded                      | Service mesh access              |

**Where Secrets Hide:**

```
High Visibility Locations (Easy to detect):
├── Source code files (*.java, *.py)
├── Configuration files (application.yml, *.properties)
├── Dockerfiles (ENV, ARG directives)
├── CI/CD pipeline files (*.yml)
└── IaC files (*.tf, *.json)

Hidden Locations (Require advanced scanning):
├── Git commit history (removed in later commits but still in .git)
├── Binary files (embedded in JARs, compiled binaries)
├── Encoded values (base64, URL-encoded, hex-encoded secrets)
├── Environment variable files (.env, .env.local)
├── Test fixtures and mock data
├── Documentation files (README.md with "example" keys that are real)
├── Build logs (echo statements, debug output)
└── Container image layers (secrets in intermediate layers)
```

**Scanning Approaches:**

**1. Pattern Matching (Regex-Based)**
- Fast, low false-positive rate for well-defined formats
- Examples: `AKIA[0-9A-Z]{16}` for AWS keys
- Tools: git-secrets, detect-secrets, trufflehog (legacy mode)
- Limitation: Misses custom secret formats, encoded secrets

**2. Entropy Analysis**
- Measures randomness of strings to identify potential secrets
- High entropy (>4.5 bits per character) suggests encrypted/random data
- Useful for detecting custom API keys without known patterns
- Limitation: High false-positive rate (UUIDs, hashes, encrypted data)

**3. Contextual Analysis**
- Examines surrounding code for secret-indicating patterns
- Variable names: `password`, `api_key`, `secret`, `token`
- Function calls: `setPassword()`, `withApiKey()`, `authenticate()`
- Comments: `// TODO: remove before commit`, `# temp credentials`
- Tools: GitHub Advanced Security, GitLab Secret Detection

**4. Historical Scanning**
- Scans entire Git history, not just current HEAD
- Detects secrets that were "removed" but exist in past commits
- Critical for incident response (secret may have been cached)
- Tools: trufflehog, gitleaks

#### Implementation: Pre-Commit Hook for Secret Scanning

##### File 9: .pre-commit-config.yaml - Pre-Commit Configuration

```yaml
# .pre-commit-config.yaml
# Pre-commit hooks for secret scanning and basic security checks
# Documentation: https://pre-commit.com/

minimum_pre_commit_version: '2.20.0'
default_stages: [commit, push]
fail_fast: false

repos:
  # =================================================================
  # REPO 1: Detect Secrets (Yelp's detect-secrets)
  # Scans for 50+ secret patterns using regex + entropy
  # =================================================================
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        name: Detect Secrets (Yelp)
        args: ['--baseline', '.secrets.baseline']
        exclude: |
          (?x)^(
            package-lock\.json|
            \.secrets\.baseline|
            tests/.*\.json|
            src/test/resources/.*
          )$
        # Exit code 1 blocks the commit
        # Developers must run: detect-secrets scan --update-baseline
        # to add legitimate false positives

  # =================================================================
  # REPO 2: Gitleaks
  # Specialized for Git repositories, scans history awareness
  # =================================================================
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.1
    hooks:
      - id: gitleaks
        name: Gitleaks Secret Scan
        # Uses default gitleaks rules + custom rules
        # Custom rules defined in .gitleaks.toml

  # =================================================================
  # REPO 3: TruffleHog
  # Scans for secrets with high-accuracy verification
  # Can verify if AWS keys are still active
  # =================================================================
  - repo: https://github.com/trufflesecurity/trufflehog
    rev: v3.63.1
    hooks:
      - id: trufflehog
        name: TruffleHog Secret Scan
        args:
          - '--only-verified'  # Only report verified active secrets
          - '--filter=entropy' # Use entropy detection

  # =================================================================
  # REPO 4: Basic Security Checks
  # =================================================================
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-yaml
        name: Validate YAML Files
        args: ['--unsafe']  # Allow custom YAML tags (e.g., Spring)
      - id: check-json
        name: Validate JSON Files
      - id: check-merge-conflict
        name: Check for Merge Conflicts
      - id: detect-private-key
        name: Detect Private Keys
      - id: no-commit-to-branch
        name: Protect Main Branch
        args: ['--branch', 'main', '--branch', 'release/*']

  # =================================================================
  # REPO 5: Java-Specific Checks
  # =================================================================
  - repo: https://github.com/jgitzel/pre-commit-java
    rev: v0.2.0
    hooks:
      - id: check-java-compile
        name: Verify Java Compiles
        args: ['--source', '17', '--target', '17']
```

##### File 10: .gitleaks.toml - Custom Gitleaks Rules for Government Context

```toml
# .gitleaks.toml
# Custom Gitleaks configuration for Government of Singapore projects
# Documentation: https://github.com/gitleaks/gitleaks#configuration

title = 'GovTech Singapore Secret Detection Rules'
version = '0.1'

[[rules]]
id = "singpass-client-secret"
description = "SingPass Client Secret"
regex = '''\b[sS][iI][nN][gG][pP][aA][sS][sS][_\-]?[cC][lL][iI][eE][nN][tT][_\-]?[sS][eE][cC][rR][eE][tT][\s]*[=:]\s*['\"]?[a-zA-Z0-9\-_]{32,}['\"]?'''
tags = ["singpass", "government", "singapore", "api-key"]
secretGroup = 1

[[rules]]
id = "singpass-partner-id"
description = "SingPass Partner ID"
regex = '''\b[sS][iI][nN][gG][pP][aA][sS][sS][_\-]?[pP][aA][rR][tT][nN][eE][rR][_\-]?[iI][dD][\s]*[=:]\s*['\"]?[A-Z0-9]{10,20}['\"]?'''
tags = ["singpass", "government", "singapore"]
secretGroup = 1

[[rules]]
id = "corppass-api-key"
description = "CorpPass API Key"
regex = '''\b[cC][oO][rR][pP][pP][aA][sS][sS][_\-]?[aA][pP][iI][_\-]?[kK][eE][yY][\s]*[=:]\s*['\"]?[a-zA-Z0-9]{40,}['\"]?'''
tags = ["corppass", "government", "singapore", "api-key"]
secretGroup = 1

[[rules]]
id = "india-aadhaar-api-key"
description = "India Aadhaar Authentication API Key"
regex = '''\b[aA][aA][dD][hH][aA][aA][rR][_\-]?[aA][pP][iI][_\-]?[kK][eE][yY][\s]*[=:]\s*['\"]?[a-zA-Z0-9]{32,64}['\"]?'''
tags = ["aadhaar", "government", "india", "api-key", "pii"]
secretGroup = 1

[[rules]]
id = "india-upi-merchant-key"
description = "India UPI Merchant Secret Key"
regex = '''\b[uU][pP][iI][_\-]?[mM][eE][rR][cC][hH][aA][nN][tT][_\-]?[kK][eE][yY][\s]*[=:]\s*['\"]?[a-zA-Z0-9]{32,}['\"]?'''
tags = ["upi", "payment", "india", "merchant"]
secretGroup = 1

[[rules]]
id = "us-fedramp-authorization"
description = "US FedRAMP Authorization Token"
regex = '''\b[fF][eE][dD][rR][aA][mM][pP][_\-]?[tT][oO][kK][eE][nN][\s]*[=:]\s*['\"]?[a-zA-Z0-9\-_.]{50,}['\"]?'''
tags = ["fedramp", "government", "us", "authorization"]
secretGroup = 1

[[rules]]
id = "azure-service-principal-password"
description = "Azure Service Principal Password"
regex = '''[a-zA-Z0-9_-]{8,40}\.(?:[a-zA-Z0-9_-]{8,40}\.)?[a-zA-Z0-9_-]{20,}'''
tags = ["azure", "cloud", "service-principal"]
# Additional context filter in allowlist
additionalRegex = '''(?i)(password|secret|credential|key)'''

# =================================================================
# ALLOWLIST - Exceptions for known false positives
# =================================================================

[allowlist]
description = "Global Allowlist"
paths = [
    '''\.secrets\.baseline''',
    '''\.gitleaks\.toml''',
    '''node_modules''',
    '''\.git''',
    '''vendor''',
    '''dist''',
    '''build''',
    '''target''',
    '''\.class''',
    '''\.jar''',
    '''tests/.*\.json''',
    '''src/test/resources/.*''',
    '''\.env\.example''',
    '''README\.md''',
]

# Specific strings that are NOT secrets (example values, placeholders)
regexes = [
    '''YOUR_[A-Z_]+_HERE''',
    '''<replace-[a-z-]+>''',
    '''example\.com''',
    '''localhost''',
    '''test[_\-]?password''',
    '''changeme''',
    '''TODO.*''',
]

# Commits that are allowed to contain secrets (e.g., rotating old keys)
commits = [
    '''Rotate old credentials''',
    '''Update example configuration''',
]
```

##### File 11: .secrets.baseline - Detect-Secrets Baseline

```json
{
  "version": "1.4.0",
  "plugins_used": [
    {
      "name": "Base64HighEntropyString",
      "max_entropy": 4.5
    },
    {
      "name": "HexHighEntropyString",
      "max_entropy": 3.0
    },
    {
      "name": "KeywordDetector",
      "keyword_exclude": "password"
    },
    {
      "name": "PrivateKeyDetector"
    },
    {
      "name": "BasicAuthDetector"
    },
    {
      "name": "AWSKeyDetector"
    },
    {
      "name": "AzureKeyDetector"
    },
    {
      "name": "GitHubTokenDetector"
    },
    {
      "name": "SlackTokenDetector"
    },
    {
      "name": "StripeDetector"
    }
  ],
  "results": {
    "src/test/resources/application-test.properties": [
      {
        "type": "Basic Auth",
        "filename": "src/test/resources/application-test.properties",
        "hashed_secret": "3c9909afec25354d551dae21590bb26e38d53f2173b8d3dc3eee4c047e7ab1c1",
        "is_verified": false,
        "line_number": 12,
        "human_secret": "testuser:testpassword",
        "allowlist_reasons": [
          "Test resource - never deployed to production"
        ]
      },
      {
        "type": "Secret Keyword",
        "filename": "src/test/resources/application-test.properties",
        "hashed_secret": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        "is_verified": false,
        "line_number": 15,
        "human_secret": "",
        "allowlist_reasons": [
          "Empty password for H2 in-memory test database"
        ]
      }
    ],
    "README.md": [
      {
        "type": "AWS Key",
        "filename": "README.md",
        "hashed_secret": "iB8eXoGxXlQrYzAbCdEfGhIjKlMnOpQr",
        "is_verified": false,
        "line_number": 45,
        "human_secret": "AKIAIOSFODNN7EXAMPLE",
        "allowlist_reasons": [
          "Documented example key from AWS documentation"
        ]
      }
    ]
  },
  "generated_at": "2024-01-15T10:30:00Z",
  "baseline_author": "security-architect@govtech.sg",
  "review_date": "2024-01-15",
  "next_review_date": "2024-04-15"
}
```

> **Production Insight:** GitHub's secret scanning automatically notifies cloud providers (AWS, Azure, GCP) when active secrets are detected. The provider may automatically revoke the credential. In 2023, a US federal agency had their Azure Service Principal revoked mid-deployment because a developer committed a `.env` file. The deployment failed, and recovery took 4 hours. Always use GitHub Secrets or HashiCorp Vault for credentials.

---

### Section D: Real-World Case Study

#### Case Study: SingHealth Data Breach - Lessons for DevSecOps

**Disclaimer:** The following case study is based on publicly available information from the Committee of Inquiry (COI) report on the SingHealth cyberattack (2018). Specific technical details are illustrative and based on common patterns observed in healthcare data breaches.

**Background:**

In June 2018, Singapore's SingHealth Group, comprising public hospitals and specialty centers, experienced a sophisticated cyberattack that resulted in the exfiltration of personal and medical records of approximately 1.5 million patients, including outpatient prescription data for 160,000 individuals, including the Prime Minister.

**Initial (Flawed) Architecture and Problems:**

```mermaid
flowchart TB
    subgraph Perimeter["Perimeter Security (Pre-Breach)"]
        FW["Firewall<br/>(Port-based rules)"]
        IDS["IDS<br/>(Signature-based)"]
        AV["Antivirus<br/>(Endpoint)"]
    end

    subgraph DMZ["DMZ"]
        WebServer["Web Server<br/>(Sun Solaris - EOL)"]
    end

    subgraph Internal["Internal Network (Flat)"]
        EHR["EHR Database<br/>(Patient Records)"]
        AD["Active Directory<br/>(Single Domain)"]
        Workstation["Admin Workstation<br/>(No MFA)"]
        Backup["Backup Server<br/>(No encryption)"]
    end

    FW --> WebServer
    WebServer --> EHR
    WebServer --> AD
    Workstation --> AD
    Workstation --> EHR
    EHR --> Backup

    style Perimeter fill:#ff6b6b,color:#fff
    style Internal fill:#ff6b6b,color:#fff
    style DMZ fill:#ffa726,color:#fff
```

**Problems Identified:**

| Problem               | Technical Detail                                         | Impact                                               |
| --------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| End-of-Life OS        | Web server running Sun Solaris (unsupported since 2015)  | No security patches for known vulnerabilities        |
| Flat Network          | No network segmentation between DMZ and clinical systems | Lateral movement unrestricted                        |
| Weak Authentication   | Admin accounts without MFA                               | Credential theft enabled full access                 |
| Signature-Based IDS   | Could not detect novel attack patterns                   | Advanced persistent threat went undetected for weeks |
| No DAST               | Web applications never tested for vulnerabilities        | SQL injection vulnerability in patient portal        |
| No Secret Scanning    | Database credentials hardcoded in application config     | Credentials extracted from source code repository    |
| No Encryption at Rest | Backup tapes unencrypted                                 | Data exfiltration included backup copies             |
| Insufficient Logging  | No centralized log aggregation                           | Attack timeline reconstructed manually over months   |

**Quantifiable Impact:**

| Metric                     | Value                                                          |
| -------------------------- | -------------------------------------------------------------- |
| Records Exfiltrated        | 1.5 million patient records                                    |
| Individuals Affected       | 1.5 million (27% of Singapore population)                      |
| Sensitive Records          | 160,000 outpatient prescriptions                               |
| Time to Detection          | 7 days (external party notification)                           |
| Time to Containment        | 4 additional days                                              |
| Investigation Duration     | 8 months (COI proceedings)                                     |
| Financial Penalty          | SGD 250,000 (PDPA fine - maximum at the time)                  |
| Estimated Remediation Cost | SGD 50-100 million (infrastructure overhaul)                   |
| Organizational Impact      | Complete leadership change at SingHealth IT; IHiS CEO resigned |

**Remediation Architecture - Applying Day 11 Concepts:**

```mermaid
flowchart TB
    subgraph DevSecOps["DevSecOps Pipeline (Post-Breach)"]
        SAST["SAST<br/>(SonarQube)"]
        SCA["SCA<br/>(Dependency-Check)"]
        SecretScan["Secret Scanning<br/>(Pre-commit + GitHub)"]
        ContainerScan["Container Scan<br/>(Trivy)"]
        DAST["DAST<br/>(ZAP in Staging)"]
        SAST --> SCA --> SecretScan --> ContainerScan --> DAST
    end

    subgraph ZeroTrust["Zero Trust Network Architecture"]
        subgraph Zone1["Zone 1: Public Interface"]
            WAF["WAF + Bot Protection"]
            APIGW["API Gateway<br/>(Rate Limiting)"]
            WAF --> APIGW
        end

        subgraph Zone2["Zone 2: Application Tier"]
            App1["Patient Portal<br/>(Containerized)"]
            App2["Admin Console<br/>(Containerized)"]
            APIGW --> App1
            APIGW --> App2
        end

        subgraph Zone3["Zone 3: Data Tier"]
            EHR["EHR Database<br/>(Encrypted at Rest)"]
            KeyVault["Azure Key Vault<br/>(Secrets)"]
            App1 --> EHR
            App2 --> EHR
            KeyVault -.-> App1
            KeyVault -.-> App2
        end

        subgraph Zone4["Zone 4: Management"]
            JumpHost["Bastion Host<br/>(MFA + Session Recording)"]
            SIEM["SIEM<br/>(Sentinel + Logs)"]
            SOC["SOC 24/7<br/>(Managed)"]
            JumpHost --> SIEM
            SIEM --> SOC
        end
    end

    DevSecOps --> ZeroTrust

    subgraph Monitoring["Observability Stack"]
        Prometheus["Prometheus<br/>(Metrics)"]
        Jaeger["Jaeger<br/>(Tracing)"]
        ELK["ELK Stack<br/>(Logging)"]
        AlertManager["AlertManager<br/>(SLO-Based)"]
        Prometheus --> AlertManager
        ELK --> AlertManager
    end

    ZeroTrust --> Monitoring

    style DevSecOps fill:#4caf50,color:#fff
    style ZeroTrust fill:#2196f3,color:#fff
    style Monitoring fill:#ff9800,color:#fff
```

**Specific DevSecOps Controls Implemented:**

| Control Type               | Implementation                                 | Prevents                                   |
| -------------------------- | ---------------------------------------------- | ------------------------------------------ |
| SAST (SonarQube)           | Quality gate blocks PRs with critical findings | SQL injection, command injection in code   |
| SCA (Dependency-Check)     | Weekly automated scans, CVSS >= 7 fails build  | Known vulnerabilities in dependencies      |
| Secret Scanning            | Pre-commit hooks + GitHub Advanced Security    | Hardcoded credentials in source code       |
| Container Scanning (Trivy) | Images scanned before registry push            | Vulnerable base images, misconfigurations  |
| DAST (ZAP)                 | Automated scan on staging deployment           | Runtime vulnerabilities, misconfigurations |
| Infrastructure as Code     | Terraform with security modules                | Drift from secure baseline                 |
| Policy as Code (OPA)       | K8s admission controllers                      | Non-compliant deployments                  |

**Lessons Learned and Architectural Principles Reinforced:**

1. **Defense in Depth is Non-Negotiable**
   - Perimeter security alone is insufficient
   - Every layer (code, container, network, application, data) needs security controls
   - Assume breach; design for containment

2. **Shift-Left Security Prevents Vulnerabilities at Source**
   - The SQL injection in the patient portal would have been caught by SAST
   - Hardcoded credentials would have been caught by secret scanning
   - Cost of fixing in development: ~$100; cost post-breach: ~$100 million

3. **Automated Security Gates Remove Human Error**
   - Manual code reviews miss 40-60% of vulnerabilities (OWASP data)
   - Automated gates enforce policy consistently
   - Security becomes part of the definition of done, not an afterthought

4. **Observability is a Security Requirement**
   - The 7-day detection gap could have been hours with proper logging and alerting
   - SLO-based alerting on authentication failures would have triggered immediate response
   - Distributed tracing would have shown the data exfiltration path in real-time

5. **Supply Chain Security is Critical**
   - EOL operating systems are single points of failure
   - Dependency vulnerabilities (like Log4Shell, discovered after this breach) require automated SCA
   - SBOM requirements (US EO 14028) are direct responses to these scenarios

> **Architect's Note:** After the SingHealth breach, Singapore's Cyber Security Agency (CSA) and the Ministry of Health (MOH) mandated the "Healthcare Cybersecurity Essentials" framework, which explicitly requires SAST, DAST, and SCA for all healthcare IT systems. India's CERT-In issued advisory CVE-2021-44228 (Log4Shell) requiring all government agencies to patch within 24 hours - impossible without SCA automation. The US CISA's Binding Operational Directive (BOD) 22-01 requires federal agencies to remediate known exploited vulnerabilities within specific timeframes, again requiring SCA.

---


# Day 11: DevSecOps, Deployment & Excellence - Theory Document

## PART 5 OF 5 (FINAL)

---

### Section E: Engagement and Assessment

#### Food for Thought / Provocation

**The Supply Chain Paradox:**

You are the Solution Architect for India's Ministry of Rural Development, designing the public distribution system (PDS) 2.0 platform that will serve 800 million ration card holders. Your team has identified a critical dependency:

```
com.gov.india:aadhaar-ekyc-sdk:3.2.1
├── Direct dependency for Aadhaar biometric verification
├── Used by 12 microservices in your architecture
├── No alternative SDK exists (government-mandated)
└── OWASP Dependency-Check reports: CVE-2024-XXXXX (CVSS 9.8 - Critical)
    - Vulnerability: Buffer overflow in fingerprint image processing
    - Exploitable: Remotely, without authentication
    - Impact: Remote code execution on the verifying server
    - Fix available: Version 3.2.2 (released 2 days ago)
    - Fix status: UIDAI has NOT certified version 3.2.2 for production use
```

**The Dilemma:**

1. **Option A: Upgrade to 3.2.2 immediately**
   - Fixes the critical vulnerability
   - Violates UIDAI certification requirement (illegal under Aadhaar Act)
   - Your team cannot legally process Aadhaar data with uncertified SDK
   - Estimated time to UIDAI certification: 6-8 months

2. **Option B: Stay on 3.2.1 with compensating controls**
   - Maintain legal compliance
   - Requires network isolation, strict input validation, runtime monitoring
   - Accepts residual risk that may not satisfy your CISO or CERT-In auditors
   - Must be documented and accepted at Director-level

3. **Option C: Disable biometric verification temporarily**
   - Removes the vulnerable code path entirely
   - 800 million citizens cannot access subsidized food
   - Political and humanitarian crisis
   - Violates Supreme Court order on PDS digitization

**Your Challenge (Before You Sleep Tonight):**

Draft a one-page risk acceptance document that:
1. Quantifies the risk in terms of citizen impact (not just technical CVSS)
2. Proposes specific compensating controls with implementation timeline
3. Identifies the single person who should sign the risk acceptance and why
4. Defines the trigger condition that would force Option C (service shutdown)
5. Includes a "kill switch" architecture that can disable the vulnerable component in <5 minutes if exploited

**Prompt for AI Exploration:**
```
"I am a Solution Architect for an Indian government system that has a 
critical vulnerability (CVSS 9.8) in a mandatory dependency where no 
certified fix exists. What compensating controls can reduce the 
exploitability of a buffer overflow in a fingerprint processing library 
when I cannot upgrade the library? Consider: memory-safe wrappers, 
seccomp profiles, gVisor sandboxing, and network micro-segmentation. 
Provide a risk reduction estimate for each control."
```

> **Trade-off Alert:** This scenario is not hypothetical. In 2022, a US Department of Defense system faced a similar situation with a radar processing library that had a critical vulnerability but no certified alternative. The risk acceptance document was signed by a 3-star general, and compensating controls cost $2.3 million to implement. The alternative (system shutdown) would have affected air defense coverage for 3 states.

---

#### Questionnaire

**Instructions:** Answer all questions. Time allocation: 20 minutes. Questions are categorized by cognitive level per Bloom's taxonomy.

---

**CONCEPTUAL QUESTIONS (Understanding)**

**Q1.** Explain the fundamental difference between how SAST and DAST detect SQL injection vulnerabilities. Why can DAST find SQL injection instances that SAST misses, and vice versa?

**Q2.** What is a Software Bill of Materials (SBOM), and why has it become a regulatory requirement for US federal suppliers (Executive Order 14028)? Name two SBOM formats and their governing standards.

**Q3.** Explain the concept of "taint analysis" in SAST tools. Define the three components of taint analysis: source, sink, and propagator. Provide an example of each in a Java Spring Boot application.

---

**APPLICATION QUESTIONS (Applying)**

**Q4.** You are configuring OWASP Dependency-Check for a Spring Boot 3.x application. The scan reports 15 vulnerabilities:
- 8 in transitive dependencies of `spring-boot-starter-web`
- 4 in `jackson-databind:2.13.0` (direct dependency you control)
- 3 in `hibernate-core` (transitive, no upgrade path in current Spring Boot version)

Write the `dependency-check-suppressions.xml` entry for ONE of these, including all required elements for government audit compliance.

**Q5.** A developer pushes a commit that contains `export DB_PASSWORD="SuperSecret123!"` in a shell script. The pre-commit hook with `detect-secrets` catches it. The developer removes the line in the next commit. Explain the two remaining security concerns and the specific commands/tools to address each.

**Q6.** Design the quality gate thresholds for a GitHub Actions pipeline deploying to a Singapore government production environment (IM8 compliance). Specify:
- Maximum allowed SAST critical/high findings
- Minimum code coverage percentage
- SCA CVSS threshold for automatic failure
- DAST behavior (fail vs. warn) for high findings
- Justify each threshold based on risk tolerance

---

**ANALYSIS QUESTIONS (Evaluating Trade-offs)**

**Q7.** Compare running Trivy container scanning in the CI pipeline (before push to registry) versus scanning images already in the registry. Create a trade-off analysis table covering: detection timing, attack surface window, false positive handling, registry pollution, and incident response implications. Recommend an approach for a system processing US federal tax data (FedRAMP High).

**Q8.** Your organization is evaluating two SAST tools:
- **Tool A (Open Source - SonarQube):** Free, 500 rules, 15% false positive rate, no support SLA, community plugins
- **Tool B (Commercial - Checkmarx):** $80K/year, 2000 rules, 8% false positive rate, 4-hour support SLA, native CI/CD integration

Perform a 5-year TCO analysis considering: license cost, developer time wasted on false positives (assume 30 min per false positive, $75/hour developer cost, 50 scans/month), missed vulnerability cost (estimate based on industry breach cost data), and compliance requirements for Singapore's MAS TRM (Technology Risk Management) guidelines.

---

**SCENARIO-BASED QUESTIONS (Architectural Judgment)**

**Q9.** You are the architect for a US state DMV (Department of Motor Vehicles) modernization project. The legacy system processes 50 million transactions/year. Your CI/CD pipeline currently takes 45 minutes. Adding full SAST, SCA, and DAST increases it to 2.5 hours. Development velocity has dropped 40%, and the delivery team is pushing back.

The CTO asks you: "We have a compliance deadline in 6 months. We can't afford to slow down this much. What do you recommend?"

Provide your recommendation with:
- Specific pipeline optimizations (what runs when)
- Risk-based approach for the 6-month transition
- Metrics you would track to prove the investment is worthwhile
- What you would NOT compromise on regardless of deadline pressure

**Q10.** At 2:00 AM, your on-call engineer receives a PageDuty alert: "DAST scan detected 3 new high-severity findings in production." Investigation reveals:
1. A new API endpoint `/api/v2/bulk-export` was deployed 6 hours ago
2. The endpoint lacks authentication (anyone can call it)
3. The endpoint exposes PII of 2 million citizens (name, address, SSN last 4)
4. DAST also shows the endpoint has no rate limiting (can be called 1000x/second)
5. You check: the endpoint was not in the OpenAPI spec submitted for security review
6. The PR was approved by a junior developer who misread the diff

Describe your incident response process for the next 60 minutes, including:
- Immediate containment actions
- Communication chain (who to notify, in what order)
- Evidence preservation steps
- Root cause analysis questions
- Long-term prevention measures

**Q11.** India's CERT-In has issued an advisory about a new zero-day vulnerability (CVE-2024-NEW0DAY) in a logging library used by 80% of Java applications. No patch is available. CVSS is 10.0 (unauthenticated RCE). Your citizen services portal uses this library.

Evaluate these three response options using a structured decision matrix:
- **Option A:** Shut down all affected services immediately
- **Option B:** Implement WAF rules to block known exploit patterns
- **Option C:** Replace the library with an alternative (estimated 2 weeks)

Your matrix must include: time to implement, effectiveness against known exploits, effectiveness against unknown variants, service disruption, and compliance implications under India's IT Act.

**Q12.** A developer on your team proposes the following to "speed up the pipeline":

```yaml
# Proposed optimization
sast:
  only-on: [main]  # Only run SAST on main branch merges
  skip-files: ["**/test/**", "**/generated/**"]  # Skip test code
  
sca:
  skip-transitive: true  # Only scan direct dependencies
  nvd-cache: 30 days  # Cache NVD data for 30 days instead of daily
  
dast:
  run-on: [release]  # Only run DAST on release tags
```

For each proposed optimization, identify:
1. The specific vulnerability class that would go undetected
2. A real-world breach that exploited exactly this gap
3. Whether you would accept this optimization, reject it, or modify it (with specifics)

---

### Answer Key with Explanations

**A1. SAST vs. DAST for SQL Injection:**

SAST detects SQL injection by analyzing source code statically. It parses the code, identifies tainted data sources (e.g., `@RequestParam`), traces data flow through the application, and checks if tainted data reaches dangerous sinks (e.g., `Statement.execute()`) without sanitization.

DAST detects SQL injection by actively attacking a running application. It sends crafted HTTP requests with SQL injection payloads (e.g., `' OR 1=1 --`) and observes the application's response for signs of successful exploitation (error messages, time delays, changed behavior).

**Why DAST finds what SAST misses:**
- Configuration vulnerabilities (SQL injection via misconfigured ORM settings)
- Vulnerabilities in dynamically generated queries not visible in source
- Vulnerabilities introduced by framework behavior at runtime
- Business logic flaws where legitimate queries become dangerous in specific contexts

**Why SAST finds what DAST misses:**
- Code paths that require specific authentication states DAST cannot reach
- Dead code or unused endpoints that contain vulnerabilities (future risk)
- Vulnerabilities in code not yet deployed (shift-left advantage)
- Exact line numbers and code context for immediate remediation

**A2. SBOM and EO 14028:**

A Software Bill of Materials (SBOM) is a machine-readable inventory of all components (libraries, frameworks, modules) that make up a software application, including their versions, suppliers, and dependencies (including transitive dependencies). It functions like an ingredients list for software, enabling rapid vulnerability impact assessment when a new CVE is disclosed.

EO 14028 (May 2021) requires SBOMs for all software sold to the US federal government because:
- The SolarWinds supply chain attack (2020) demonstrated that malicious code can be inserted deep in the supply chain
- Without an SBOM, organizations cannot quickly determine if they are affected by a new vulnerability
- Log4Shell (CVE-2021-44282) affected 60%+ of Java applications; organizations with SBOMs responded in hours, others took weeks

Two SBOM formats:
1. **SPDX (Software Package Data Exchange)** - ISO/IEC 5962:2021 standard, maintained by Linux Foundation, originally designed for license compliance, now includes security metadata
2. **CycloneDX** - OWASP project, lightweight JSON/XML format, designed specifically for security use cases, supports vulnerability tracking and service SBOMs

**A3. Taint Analysis Components:**

**Source:** A point where untrusted data enters the application. In Spring Boot:
```java
// HTTP request parameters
@RequestParam String userId
@RequestBody UserDTO userDto
// HTTP headers
@RequestHeader String authToken
// Cookies
Cookie[] cookies = request.getCookies()
// Environment variables (if containing user-influenced data)
String config = System.getenv("USER_INPUT_CONFIG");
```

**Sink:** A point where untrusted data could cause harm if it reaches that point without sanitization. In Spring Boot:
```java
// SQL execution
jdbcTemplate.query("SELECT * FROM users WHERE id = " + userId);
// Command execution
Runtime.getRuntime().exec("process " + fileName);
// File operations
new FileOutputStream("/data/" + userProvidedPath);
// Reflection
Class.forName(userProvidedClassName).newInstance();
// HTML output (XSS sink)
response.getWriter().write("<div>" + userComment + "</div>");
```

**Propagator:** Code that passes tainted data from one point to another without sanitizing it. In Spring Boot:
```java
// String concatenation
String query = "SELECT * FROM users WHERE id = " + userId;  // Propagates taint
// Method calls
public void process(String input) {
    saveToDatabase(input);  // input is still tainted
}
// Collection operations
List<String> params = List.of(userId);  // Taint propagates into collection
params.stream().forEach(this::query);   // Taint propagates out
// Object wrapping
UserDTO dto = new UserDTO(userId);  // Taint wraps into object
repository.save(dto);                // Taint still present
```

**A4. Dependency-Check Suppression Example:**

For `jackson-databind:2.13.0` (direct dependency with known CVEs):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <suppress>
        <packageUrl regex="true">^pkg:maven/com\.fasterxml\.jackson\.core/jackson-databind@2\.13\.0$</packageUrl>
        <cve>CVE-2022-42003</cve>
        <cve>CVE-2022-42004</cve>
        <justification>
            Jackson-databind 2.13.0 is a direct dependency managed by 
            Spring Boot 3.0.x BOM. Upgrade path requires migration to 
            Spring Boot 3.1.x, which is scheduled for Sprint 14 (2024-02-15).
            Compensating controls in place: (1) Input validation via 
            @Valid annotations prevents malformed JSON parsing, (2) WAF 
            rules block known exploit patterns, (3) Application runs in 
            restricted network zone without outbound internet access.
            Risk accepted by: [CISO Name], Date: [Date], 
            Review expiry: 2024-03-15.
        </justification>
        <expiryDate>2024-03-15</expiryDate>
    </suppress>
</suppressions>
```

**Required elements for government audit:**
- `packageUrl`: Precise dependency identification (regex for flexibility)
- `cve`: Specific CVEs being suppressed (not blanket suppression)
- `justification`: Detailed explanation of why suppression is acceptable
- `expiryDate`: Mandatory review date (prevents permanent suppressions)
- (Implied) Approval documentation: Who approved, when, and review schedule

**A5. Remaining Security Concerns After Commit Removal:**

**Concern 1: The secret still exists in Git history.**
Even though the line was removed in a subsequent commit, the original commit containing the password remains in `.git/objects/`. Anyone with repository access can retrieve it. If the repository is ever made public (even temporarily), the secret is exposed.

**Mitigation:**
```bash
# Option A: Use git-filter-repo to rewrite history (destructive)
git filter-repo --path shell-script.sh --invert-paths
git push --force --all

# Option B: Use BFG Repo Cleaner (faster for large repos)
java -jar bfg.jar --replace-text passwords.txt
git push --force --all

# Option C: If secret was already pushed to remote, assume it's compromised
# 1. Rotate the credential IMMEDIATELY in the database
# 2. Do NOT rely on history rewriting alone
```

**Concern 2: The secret may have been cached or logged.**
CI/CD pipelines log build output. If the secret was printed in logs (e.g., `echo $DB_PASSWORD`), it exists in CI log storage. Application runtime may have cached it. Monitoring tools may have captured it.

**Mitigation:**
```bash
# 1. Purge CI/CD logs (GitHub Actions example)
# Go to Actions -> Run -> Delete log

# 2. Check for secrets in CI artifacts
gh run download --name artifacts  # Review for leaked secrets

# 3. Rotate credential in target system (PostgreSQL)
ALTER USER app_user WITH PASSWORD 'new_secure_password';

# 4. Scan for secret in external systems
# - Check GitHub secret scanning alerts
# - Check if secret appears in any issue/PR comments
# - Check monitoring/alerting systems that may have logged it
```

**A6. Quality Gate Thresholds for IM8 Compliance:**

| Gate            | Threshold                 | Justification                                                                                                                                                                            |
| --------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SAST Critical   | 0                         | Zero tolerance for critical vulnerabilities (injection, auth bypass). IM8 Section 5.3 requires "no critical vulnerabilities in production code."                                         |
| SAST High       | ≤ 2                       | Limited tolerance with documented risk acceptance. High findings must have compensating controls and CISO sign-off within 48 hours.                                                      |
| Code Coverage   | ≥ 80% line, ≥ 70% branch  | IM8 requires "evidence of thorough testing." Spring Boot applications with lower coverage have correlated 3x more production incidents (GovTech internal data, 2023).                    |
| SCA CVSS        | Fail on CVSS ≥ 7.0 (High) | NIST NVD defines 7.0+ as high severity. IM8 requires "vulnerabilities with CVSS ≥ 7.0 to be remediated within 7 days." Failing the build enforces this.                                  |
| SCA Transitive  | Include transitive        | Log4Shell was a transitive dependency. Excluding transitive dependencies would have missed it.                                                                                           |
| DAST High       | Warn (not fail)           | DAST has higher false positive rate than SAST/SCA. High findings require manual verification within 24 hours. Automated failure would cause excessive false positives blocking releases. |
| DAST Critical   | Fail                      | Critical DAST findings (e.g., auth bypass, data exposure) are almost never false positives. Immediate block.                                                                             |
| Secret Scanning | Fail on any               | Zero tolerance. Secrets in code is a categorical failure.                                                                                                                                |

**A7. Trivy Scanning: Before Push vs. After Push:**

| Criterion                   | Before Push (CI)                               | After Push (Registry)                             |
| --------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| **Detection Timing**        | Earliest possible (during build)               | After image exists in registry                    |
| **Attack Surface Window**   | Zero - vulnerable image never reaches registry | Window exists between push and scan completion    |
| **Registry Pollution**      | None - only clean images pushed                | Vulnerable images exist in registry until deleted |
| **False Positive Handling** | Blocks developer, immediate feedback           | Requires separate cleanup process                 |
| **Rollback Complexity**     | Simple - don't push                            | Must delete image, re-tag, handle existing pulls  |
| **Multi-Registry Sync**     | Clean from the start                           | May sync vulnerable image to other registries     |
| **Audit Trail**             | Build log shows rejection                      | Must correlate scan results with image digests    |
| **Incident Response**       | Prevention-focused                             | Detection-focused (requires IR process)           |

**Recommendation for FedRAMP High:**
Implement **both** with different purposes:
1. **CI Gate (Before Push):** Primary control. Block any image with CVSS ≥ 7.0. This is your preventative control.
2. **Registry Scan (After Push):** Defense-in-depth. Continuous scanning catches new CVEs disclosed after image was pushed. Also scans images from third parties that bypassed CI.

Additionally:
- Implement image signing (Cosign) so only CI-scanned images can be deployed
- Use OPA policy to reject unsigned images in Kubernetes
- Set up automated alerts for new CVEs in existing registry images

**A8. 5-Year TCO Analysis:**

**Assumptions:**
- 50 scans/month = 600 scans/year = 3,000 scans over 5 years
- Developer cost: $75/hour
- False positive investigation time: 30 minutes
- Missed vulnerability cost: Based on Ponemon Institute 2023 data, average cost of a data breach is $4.45M, with 12% attributable to vulnerabilities that should have been caught by SAST

**Tool A (SonarQube - Open Source):**

| Cost Category       | Calculation                                         | 5-Year Total             |
| ------------------- | --------------------------------------------------- | ------------------------ |
| License             | $0                                                  | $0                       |
| Infrastructure      | $500/month server                                   | $30,000                  |
| Maintenance         | 10 hrs/month admin @ $75/hr                         | $45,000                  |
| False Positive Time | 600 scans × 15% FP rate × 0.5 hrs × $75             | $337,500                 |
| Missed Vuln Risk    | 3,000 scans × 0.5% miss rate × $4.45M × 12% / 3,000 | $26,700 (expected value) |
| **Total**           |                                                     | **$439,200**             |

**Tool B (Checkmarx - Commercial):**

| Cost Category       | Calculation                                         | 5-Year Total            |
| ------------------- | --------------------------------------------------- | ----------------------- |
| License             | $80,000/year × 5                                    | $400,000                |
| Infrastructure      | Included in license                                 | $0                      |
| Maintenance         | 2 hrs/month admin @ $75/hr                          | $9,000                  |
| False Positive Time | 600 scans × 8% FP rate × 0.5 hrs × $75              | $180,000                |
| Missed Vuln Risk    | 3,000 scans × 0.1% miss rate × $4.45M × 12% / 3,000 | $5,340 (expected value) |
| **Total**           |                                                     | **$594,340**            |

**Analysis:**
- Tool A is $155,140 cheaper over 5 years
- However, Tool B catches 5x more vulnerabilities (0.5% vs 0.1% miss rate)
- MAS TRM Section 4.5 requires "commercial support SLA for security tools" for systems classified as "Significant" or "Major"
- If the system processes >$10M transactions/day, the missed vulnerability risk becomes unacceptable

**Recommendation:** 
- For "Minor" systems (MAS TRM): Use Tool A (SonarQube) with custom rules
- For "Significant/Major" systems (MAS TRM): Use Tool B (Checkmarx) despite higher TCO
- Hybrid approach: SonarQube for all developers, Checkmarx for critical path code

**A9. DMV Pipeline Optimization:**

**Immediate Recommendation to CTO:**

"I understand the velocity concern. Here's my proposal: we don't slow down development; we change WHEN and WHERE security runs."

**Pipeline Optimizations:**

```
CURRENT STATE (2.5 hours, all on main branch):
[Build 10m] → [Unit Test 15m] → [SAST 45m] → [SCA 20m] → [DAST 60m] → [Deploy 10m]

PROPOSED STATE (45 minutes effective, 2.5 hours total parallel):
Developer PR (15 min):
[Build 5m] → [Unit Test 10m] → [Incremental SAST 5m] → [PR Feedback]

Merge to Main (30 min, parallel):
[SAST Full 45m] ─────────────────┐
[SCA 20m] ───────────────────────┼→ [Gate Check 5m] → [Deploy to Staging 10m]
[DAST 60m - async, non-blocking] ┘→ [Results in 1 hour, separate gate]
```

**Specific Optimizations:**

| Technique               | Time Saved      | Risk Mitigation                                                    |
| ----------------------- | --------------- | ------------------------------------------------------------------ |
| Incremental SAST on PRs | 40 min/PR       | Only scan changed files; full scan on merge                        |
| Parallel SAST + SCA     | 20 min          | Independent scans, no dependency                                   |
| Async DAST              | 60 min blocking | Run against staging, alert if findings, don't block initial deploy |
| SCA with cached NVD     | 15 min          | Cache vulnerability DB, update daily not per-scan                  |
| SAST with SonarLint IDE | Shift-left      | Catch 60% of issues before PR (zero pipeline time)                 |

**6-Month Transition Plan:**

| Month | Phase        | SAST                 | SCA                    | DAST                     |
| ----- | ------------ | -------------------- | ---------------------- | ------------------------ |
| 1-2   | Foundation   | IDE only (SonarLint) | None                   | None                     |
| 3-4   | Partial Gate | PR incremental       | Monthly manual scan    | Staging only, advisory   |
| 5-6   | Full Gate    | PR + merge full      | Per-build, CVSS≥9 fail | Per-release, CVSS≥9 fail |
| 7+    | Maturity     | Full gate            | Full gate              | Full gate                |

**Metrics to Track:**

1. **Vulnerability Escape Rate:** Vulnerabilities found in production vs. found in pipeline
   - Target: <5% escape rate by month 6
2. **Mean Time to Remediate (MTTR):** Time from vulnerability detection to fix
   - Target: Critical <24 hours, High <7 days
3. **Pipeline Rejection Rate:** % of builds failing security gates
   - Target: <10% (indicates either poor code quality or misconfigured gates)
4. **Developer Feedback Time:** Time from PR creation to security feedback
   - Target: <15 minutes
5. **False Positive Rate:** % of security findings dismissed as false positives
   - Target: <15% (indicates well-tuned rules)

**Non-Negotiable Items:**

1. **Secret scanning must be enforced from Day 1.** No exceptions. A leaked credential is an immediate incident, not a technical debt item.
2. **Critical CVEs (CVSS ≥ 9.0) must block deployment.** This is a compliance requirement, not a preference.
3. **Production deployment requires at least SAST + SCA completion.** DAST can be async for the transition period, but SAST/SCA are non-negotiable.

**A10. Incident Response - First 60 Minutes:**

**T+0 to T+5 minutes: CONTAINMENT**

```
Actions:
1. Block the endpoint immediately (DO NOT wait for analysis)
   kubectl patch virtualservice dcs-api \
     --type='json' -p='[{"op":"replace","path":"/spec/http/2/match/0/uri/exact","value":"/api/v2/bulk-export-blocked"}]'
   
   OR if using API Gateway:
   az apim api delete --resource-group dcs-rg --service-name dcs-apim --api-id bulk-export-v2

2. Enable rate limiting on all /api/v2/ endpoints (defense in depth)

3. Verify the endpoint returns 404/403 from multiple locations

DO NOT:
- Shut down the entire service (affects legitimate users)
- Delete the code (destroys evidence)
- Start fixing the bug (preservation first)
```

**T+5 to T+15 minutes: ASSESSMENT**

```
Actions:
1. Check access logs for the endpoint
   - How many times was it called?
   - From which IPs?
   - What data was returned?

2. Check if DAST was the first caller or if external actors found it first
   - If DAST was first: incident contained, no data exposure
   - If external IPs called it: assume data breach, escalate immediately

3. Capture current state for forensics
   - Snapshot the pod: kubectl cp dcs-pod:/app /forensics/app-snapshot
   - Export logs: kubectl logs dcs-pod --since=6h > /forensics/pod-logs.txt
   - Database query log for the endpoint's queries
```

**T+15 to T+30 minutes: COMMUNICATION**

```
Notification Chain (in order):
1. On-call Security Engineer (already notified via PagerDuty)
2. CISO / Security Director (phone call, not email)
3. Engineering Manager (Slack + phone)
4. CTO (if CISO confirms potential data exposure)
5. Legal/Compliance (if CISO confirms data exposure)
6. Government Liaison (if citizen PII confirmed exposed)

Initial Message Template:
"SECURITY INCIDENT - PRIORITY P1
Time: [Timestamp]
System: Citizen Services Portal
Issue: Unauthenticated PII exposure endpoint detected
Status: Endpoint blocked, investigation in progress
Data at Risk: Citizen PII (name, address, SSN-4)
Scope: Unknown, under investigation
Next Update: 30 minutes"
```

**T+30 to T+45 minutes: EVIDENCE PRESERVATION**

```
Actions:
1. Preserve all logs with tamper-evident hashing
   sha256sum pod-logs.txt > pod-logs.txt.sha256
   
2. Capture database state
   - Query: SELECT COUNT(*) FROM audit_log WHERE endpoint='/api/v2/bulk-export'
   - Export relevant audit records

3. Document timeline
   - Exact time PR was merged
   - Exact time deployment completed
   - Exact time DAST scan started
   - Exact time endpoint was blocked

4. Take screenshots of:
   - The problematic PR approval
   - DAST scan results
   - Current endpoint status (404)
```

**T+45 to T+60 minutes: ROOT CAUSE QUESTIONS**

```
Questions to Investigate (not necessarily answer now):
1. Why was this endpoint not in the OpenAPI spec?
   - Did the developer bypass the spec-first requirement?
   - Was the spec review process bypassed?

2. Why did a junior developer approve a PR with security implications?
   - Was CODEOWNERS file properly configured?
   - Was the junior developer aware they were approving security-sensitive code?

3. Why didn't existing security controls catch this?
   - Was the endpoint missing authentication annotations (@PreAuthorize)?
   - Did SAST have a rule for missing auth on @GetMapping?
   - Was the diff too large for effective review?

4. Why was DAST the first to detect this (not SAST or code review)?
   - SAST doesn't detect missing authentication (design flaw, not code flaw)
   - Code review missed it (human error)
   - No architecture validation in pipeline
```

**Long-term Prevention Measures:**

1. **Architecture Validation:** Add an OPA policy that rejects any K8s deployment where endpoints are not registered in the API catalog
2. **CODEOWNERS:** Enforce that any PR touching authentication/authorization code requires senior developer + security team approval
3. **SAST Custom Rule:** Create a SonarQube custom rule that flags any `@GetMapping`/`@PostMapping` without corresponding `@PreAuthorize` or `@Secured`
4. **API Discovery:** Implement runtime API discovery (e.g., Traceable) to detect shadow APIs
5. **DAST Expansion:** Run DAST on every deployment to staging, not just releases

**A11. Zero-Day Response Decision Matrix:**

| Criterion                              | Option A: Shutdown        | Option B: WAF Rules               | Option C: Replace Library    |
| -------------------------------------- | ------------------------- | --------------------------------- | ---------------------------- |
| **Time to Implement**                  | 5-30 minutes              | 1-4 hours                         | 2 weeks                      |
| **Effectiveness vs. Known Exploits**   | 100% (no code running)    | 80-90% (pattern-based)            | 100% (vulnerability removed) |
| **Effectiveness vs. Unknown Variants** | 100%                      | 30-50% (novel bypasses)           | 100%                         |
| **Service Disruption**                 | 100% (complete outage)    | 0-5% (false positives)            | 0% (gradual migration)       |
| **IT Act Compliance**                  | Compliant (protects data) | Partially compliant (best effort) | Compliant (permanent fix)    |
| **Citizen Impact**                     | Severe (no services)      | Minimal                           | None                         |
| **Political Impact**                   | High (media scrutiny)     | Low                               | None                         |
| **Technical Risk**                     | None (safe)               | Medium (bypass possible)          | Low (testing required)       |
| **Overall Score (1-10)**               | 4                         | 6                                 | 8                            |

**Recommendation: Phased Approach**

```
Phase 1 (T+0 to T+4 hours): Implement Option B (WAF Rules)
- Deploy emergency WAF rules blocking known exploit patterns
- Enable enhanced logging on all application servers
- Activate incident response team

Phase 2 (T+4 to T+48 hours): Assess + Prepare Option C
- Begin library replacement in emergency branch
- Set up parallel test environment with new library
- Run full regression test suite

Phase 3 (T+48 hours to T+2 weeks): Execute Option C
- Deploy replacement library to canary (1% traffic)
- Monitor for 24 hours
- Gradual rollout to 100%

Trigger Condition for Option A (Shutdown):
IF (WAF logs show successful exploitation attempts > 10/minute)
OR (CERT-In issues emergency directive to shut down)
OR (new variant discovered that bypasses WAF rules)
THEN execute Option A immediately
```

**A12. Pipeline Optimization Evaluation:**

**Optimization 1: `only-on: [main]` - Only run SAST on main branch**

| Aspect                         | Analysis                                                                                                                                                                                                                    |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Vulnerability Class Missed** | Vulnerabilities introduced in feature branches that exist for days/weeks without scanning. Merging to main with accumulated tech debt.                                                                                      |
| **Real-World Breach**          | **SolarWinds (2020):** Malicious code was introduced in a feature branch, passed code review (social engineering), and merged to main. If SAST had run on the branch, the suspicious network calls might have been flagged. |
| **Decision**                   | **MODIFY:** Run incremental SAST on all PRs (changed files only, ~5 min), full SAST on merge to main. This catches issues early without full scan overhead.                                                                 |

**Optimization 2: `skip-files: ["**/test/**"]` - Skip test code**

| Aspect                         | Analysis                                                                                                                                                                                                                                                              |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Vulnerability Class Missed** | Test code often contains hardcoded credentials, insecure test configurations, and vulnerabilities in test utilities that get copy-pasted to production code.                                                                                                          |
| **Real-World Breach**          | **Uber (2022):** The attacker gained initial access through a hardcoded credential in a PowerShell script that was part of a test/automation suite. The script was on a shared network drive, not in production, but the credential was valid for production systems. |
| **Decision**                   | **REJECT:** Do not skip test files entirely. Instead, apply different rules: lower severity thresholds for test code, but still scan for secrets and critical vulnerabilities.                                                                                        |
| **Modification:**              |
```yaml
sast:
  skip-files: []  # Don't skip
  test-code-rules:
    - secrets: BLOCK  # Still block secrets
    - sql-injection: WARN  # Warn on SQL injection in tests
    - code-style: SKIP  # Skip style issues in tests
```

**Optimization 3: `skip-transitive: true` - Only scan direct dependencies**

| Aspect                                                                                                                                  | Analysis                                                                                                                                                                                                                                                                                                                                |
| --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Vulnerability Class Missed**                                                                                                          | Transitive dependency vulnerabilities. This is EXACTLY how Log4Shell spread - Log4j was a transitive dependency of Spring Boot, not a direct dependency for most applications.                                                                                                                                                          |
| **Real-World Breach**                                                                                                                   | **Log4Shell (CVE-2021-44282):** Affected an estimated 93% of cloud environments. Most affected applications did not directly declare Log4j as a dependency - it was pulled in transitively through Spring Boot, Apache Struts, Elasticsearch, etc. Organizations that only scanned direct dependencies were blind to the vulnerability. |
| **Decision**                                                                                                                            | **ABSOLUTELY REJECT:** This optimization is unacceptable for any government system. The entire purpose of SCA is to catch transitive dependency vulnerabilities.                                                                                                                                                                        |
| **Alternative:** If scan time is the concern, cache the dependency tree and only re-scan when `pom.xml` or `package-lock.json` changes. |

**Optimization 4: `nvd-cache: 30 days` - Cache NVD data for 30 days**

| Aspect                                                                                                                                                                | Analysis                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Vulnerability Class Missed**                                                                                                                                        | New CVEs disclosed within the 30-day cache window. A critical vulnerability disclosed on Day 1 would not be detected until Day 30.                                                                                                                                                 |
| **Real-World Breach**                                                                                                                                                 | **Log4Shell Response Timeline:** CVE-2021-44282 was disclosed December 9, 2021. Exploits were in the wild within hours. By December 15, mass exploitation was occurring. A 30-day cache would have meant not detecting Log4Shell until January 8, 2022 - a full month of exposure. |
| **Decision**                                                                                                                                                          | **REJECT:** For government systems, NVD data must be no older than 24-48 hours. CERT-In advisories require assessment within 24 hours of CVE disclosure.                                                                                                                           |
| **Modification:** Cache locally but update daily. Use a mirror/proxy (like Dependency-Track) that updates the NVD database centrally and serves it to all CI runners. |

---

## Day 11 Theory Document Completion Gate
