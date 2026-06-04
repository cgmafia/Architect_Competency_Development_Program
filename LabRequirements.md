# Lab Requirements Document

### 🖥️ 1. Linux VDI Specifications (The Participant Workspace)
Since participants are connecting via Linux VDIs, the environment must be lightweight enough to stream smoothly but powerful enough to run local CLIs, IDEs, and lightweight containers without lag.

*   **OS:** Ubuntu 22.04 LTS or RHEL 9 (Standard Enterprise Linux).
*   **Compute:** Minimum 4 vCPU, 16 GB RAM (8 vCPU / 32 GB RAM preferred if local Docker builds are heavy).
*   **Storage:** 50 GB SSD (Persistent home directory for user configs and code).
*   **Network:** Low-latency connection to the **Azure Sweden Central** region.
*   **Pre-Installed Local Tooling (Linux Native):**
    *   **IDE:** Visual Studio Code (with pre-configured profiles).
    *   **CLIs:** Azure CLI (`az`), Kubernetes CLI (`kubectl`, `helm`), Terraform CLI, Docker CLI, Git.
    *   **Runtimes:** Node.js (LTS), Python 3.10+, Java 17+ (OpenJDK), .NET 8 SDK.
    *   **Utilities:** `jq`, `yq`, `curl`, `wget`, `httpie`, `tree`.

### ☁️ 2. Azure Cloud Infrastructure & Governance
To prevent runaway costs and ensure security during the training, the Azure environment must be strictly governed.

*   **Azure Region:** `swedencentral` (Sweden Central) for all resource deployments.
*   **Subscription Model:** Dedicated "Training" Pay-As-You-Go or Enterprise Agreement (EA) subscription.
*   **Identity & Access (Entra ID / Azure AD):**
    *   Dedicated Security Group for the batch (e.g., `EY-Architect-Batch-Jun26`).
    *   SSO enabled for both VDI access and Azure Portal.
*   **RBAC (Role-Based Access Control):**
    *   Trainees get **"Contributor"** access *only* to their specific, pre-provisioned Training Resource Groups.
    *   **No Subscription-level access** (prevents them from creating unauthorized global resources).
*   **Azure Policies (Mandatory Guardrails):**
    *   Restrict allowed VM sizes (e.g., block G-series/GPU instances).
    *   Restrict deployments to `swedencentral` only.
    *   Auto-tag all resources with `Project=EY-Architect`, `Batch=Jun26`, `CostCenter=Training`.
    *   Enforce auto-shutdown for any accidental VM creations.

### 🛠️ 3. Core Azure Services to Pre-Provision (or Quota Approved)
To save time during the 4.5-hour daily blocks, PaaS and managed services should be pre-provisioned via Terraform before the labs begin.

| Category                 | Azure Services Required                                                    | Purpose in Labs                                                                        |
| :----------------------- | :------------------------------------------------------------------------- | :------------------------------------------------------------------------------------- |
| **Compute & Containers** | **AKS** (Azure Kubernetes Service), **ACR** (Azure Container Registry)     | Core deployment target for Microservices, Istio/Service Mesh, and DevSecOps pipelines. |
| **Databases & Data**     | **Cosmos DB** (NoSQL/Mongo API), **Azure Cache for Redis**, **Azure SQL**  | Polyglot persistence labs, caching strategies, legacy migration targets.               |
| **Messaging & Events**   | **Azure Event Hubs**, **Azure Service Bus**                                | Event-driven architecture, CQRS, and Kafka/RabbitMQ equivalent labs.                   |
| **DevOps & CI/CD**       | **Azure DevOps Services** (or GitHub Enterprise), **Azure Container Apps** | Shift-left security pipelines, IaC deployments, artifact management.                   |
| **Observability**        | **Azure Monitor**, **Log Analytics Workspace**, **App Insights**           | ELK stack alternative/integration, distributed tracing, metrics.                       |
| **Security & Identity**  | **Entra ID (Azure AD)**, **Key Vault**, **Azure API Management**           | Zero Trust labs, mTLS, secret management, API Gateway security.                        |

### 🧠 4. Specialized Tooling & Licenses
Since this is an advanced architecture and AI-assisted program, specific commercial and open-source licenses are required.

*   **AI-Assisted Development:**
    *   **GitHub Copilot Business/Enterprise Licenses:** 1 per participant. (Crucial for the "Vibe Coding" and AI-Assisted Validation modules).
    *   *Alternative:* Azure OpenAI Service endpoints pre-provisioned with `gpt-4` and `text-embedding-ada-002` models if Copilot is not approved.
*   **Architecture & Design:**
    *   **Draw.io / Diagrams.net:** Desktop version pre-installed on VDI, or enterprise web license.
    *   **PlantUML / C4-Model Plugins:** Pre-installed in VS Code.
*   **API & Load Testing:**
    *   **Postman for Linux:** Pre-installed on VDI with a shared "Training Workspace" synced via API keys.
    *   **k6 / JMeter:** CLI versions pre-installed on VDI for performance engineering labs.

### 📦 5. "Zero-Toil" Lab Delivery Strategy (Crucial for 14-Day Crunch)
Because the schedule is highly compressed, the traditional "build from scratch" lab model will fail. You must implement a **Starter Kit & GitOps** approach.

1.  **Central Training Repository:** A master GitHub/Azure DevOps repo containing:
    *   `/starter-kits`: Pre-written boilerplate code for the microservices, mobile-backend, and legacy monoliths.
    *   `/infra-templates`: Pre-validated Terraform modules for AKS, CosmosDB, etc.
    *   `/datasets`: Mock "citizen data" and "legacy database" SQL/JSON dumps.
2.  **Pre-Built Container Images:** All heavy base images (e.g., ELK stack, Istio control plane, Java/Node runtimes) must be pre-pulled and pushed to the **Azure Container Registry (ACR)** so VDIs don't waste 20 minutes downloading them from Docker Hub.
3.  **VS Code DevContainers:** Provide `.devcontainer` configurations in the starter kits so that when a user opens a project in VS Code on their Linux VDI, all exact dependencies, extensions, and CLI tools are automatically injected into the workspace.

### 🔐 6. Security, Compliance & Data Isolation
Aligning with your strict requirements for security and data isolation (GDPR/Zero Leakage):

*   **No Production Data:** All labs must use strictly synthetic/mock data.
*   **VDI Data Loss Prevention (DLP):** Disable clipboard copying from VDI to local machine, and disable local drive mapping via the VDI client (Citrix/VMware/Windows App).
*   **Secret Management:** Trainees must *never* hardcode credentials. All labs must enforce fetching secrets from **Azure Key Vault** using Managed Identities.
*   **Network Isolation:** AKS clusters should be deployed as **Private Clusters** (API server only accessible via VDI virtual network). The VDI acts as the secure jump box/bastion.

### 📋 7. Pre-Flight Checklist for IT/DevOps Team
*   [ ] Provision Linux VDIs and apply baseline security hardening.
*   [ ] Deploy Azure Landing Zone / Training Resource Groups in `swedencentral`.
*   [ ] Apply Azure Policies for cost and resource guardrails.
*   [ ] Provision GitHub Copilot licenses and assign to trainee Entra ID accounts.
*   [ ] Run Terraform scripts to pre-deploy AKS, ACR, CosmosDB, and KeyVault.
*   [ ] Push all pre-built Docker images to ACR.
*   [ ] Seed the central Git repo with Starter Kits, Mock Data, and Terraform templates.
*   [ ] **Dry Run:** Trainer logs into a VDI and runs the Day 1 and Day 14 labs end-to-end to verify network routing and tool functionality.