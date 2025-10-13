# Claim Status API — Repository Guide & Runbook

> Java (Spring Boot) ➜ containerized ➜ **Azure Container Apps** (public ingress) ➜ **API Management (APIM)** ➜ **Azure OpenAI**  
> CI/CD: **Azure DevOps** (Windows self‑hosted agent), image scanning (optional **Trivy** or **Defender for Cloud**)

---

## 1) Overview

This repository contains a Spring Boot service that exposes:

- `GET /claims/{id}`: reads mock data (JSON) and returns claim details.  
- `POST /claims/{id}/summarize`: reads adjuster notes and calls **Azure OpenAI** to return:
  ```json
  { "summary": "...", "customerSummary": "...", "adjusterSummary": "...", "nextStep": "..." }
  ```

The service is containerized and deployed to **Azure Container Apps (ACA)**; **API Management (APIM)** provides a stable public gateway and policy (subscription key, rate‑limit). CI/CD builds and pushes images to **Azure Container Registry (ACR)** and updates the Container App.

**Your environment**

- **Resource Group**: `introspect-2b`  
- **Region**: `eastus2`  
- **ACR**: `claimsintrospect2b.azurecr.io`  
- **ACA Environment**: `managedEnvironment-introspect2b-b108`  
- **ACA App**: `claims-aca`  
- **Azure OpenAI**: endpoint `https://claims-openaires.services.ai.azure.com/`, deployment `gpt-5-mini`  
- **Azure DevOps variable group**: `Claims` (contains secret `AZURE_OPENAI_KEY`)  
- **Service connections**:  
  - Docker Registry ➜ `acr-claimsintrospect2b-sc`  
  - Azure Resource Manager ➜ `arm-introspect2b-rg-sc`

**References:**  
ACA quickstart & env creation, Log Analytics integration, ACR quickstart, APIM docs, Azure DevOps tasks (**Docker@2**, **AzureCLI@2**), Git push from CLI.  
- ACA: https://learn.microsoft.com/azure/container-apps/tutorial-deploy-first-app-cli · https://learn.microsoft.com/azure/container-apps/log-monitoring  
- ACR: https://learn.microsoft.com/azure/container-registry/container-registry-get-started-azure-cli  
- APIM: https://learn.microsoft.com/azure/api-management/  
- Pipelines: https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/docker-v2?view=azure-pipelines · https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/azure-cli-v2?view=azure-pipelines  
- Git (push/clone): https://learn.microsoft.com/azure/devops/repos/git/share-your-code-in-git-cmdline?view=azure-devops

---

## 2) Repository Structure

```
.
├─ src/main/java/com/example/claims/         # Spring Boot code
│  ├─ api/ClaimController.java               # GET /claims/{id}, POST /claims/{id}/summarize
│  ├─ model/{Claim, SummaryResponse}.java
│  └─ service/{ClaimService, NotesService, AzureOpenAISummarizer}.java
├─ src/main/resources/
│  ├─ application.yaml                       # Reads AZURE_OPENAI_* env vars
│  └─ mocks/{claims.json, notes.json}        # Lab data
├─ apim/
│  ├─ openapi.yaml                           # Import in APIM
│  └─ api-policy.xml                         # Inbound policy: key + rate-limit; set backend to ACA FQDN
├─ iac/aca.bicep                             # Optional infra-as-code (ACA env+app pattern)
├─ observability/*.kql                       # KQL queries for APIM & Container Apps logs
├─ pipelines/azure-pipelines.yml             # Windows agent pipeline (CI build, image push, ACA deploy)
├─ Dockerfile                                # Multi-stage build (Maven build ➜ Temurin JRE image)
├─ .dockerignore
└─ README.md (or docs/README-lab.md)         # This guide
```

Design goals:

- **Separation of concerns**: code, contracts (OpenAPI, APIM policies), infra (Bicep), ops (KQL), and CI remain modular.  
- **12‑factor**: secrets as env/secrets in ACA; no secrets in source.  
- **Traceability**: pipeline adds build ID as image tag; APIM & ACA logs stream to **Log Analytics** (ACA logging).

---

## 3) Prerequisites

- **Azure**: owner/contributor on RG; provider registrations: `Microsoft.App`, `Microsoft.OperationalInsights`, `Microsoft.ApiManagement`, `Microsoft.CognitiveServices`.  
  Docs: Providers & types; registration troubleshooting.
- **ACA environment** (`managedEnvironment-introspect2b-b108`) already created; link diagnostics to **Log Analytics** workspace to query logs (`ContainerAppConsoleLogs_CL`, `ContainerAppSystemLogs_CL`).  
- **Azure OpenAI** resource with a deployed model (`gpt-5-mini`), keep **Endpoint** and **Key** handy (key stored in DevOps variable group).  
- **APIM** instance (Developer SKU is fine for labs).  
- **ACR**: `claimsintrospect2b.azurecr.io`.  
- **Azure DevOps**:  
  - Variable group **`Claims`** with secret `AZURE_OPENAI_KEY`.  
  - Service connections: `acr-claimsintrospect2b-sc` (Docker Registry) & `arm-introspect2b-rg-sc` (ARM, scope RG). Assign **AcrPush** on ACR and **Contributor** on RG.  
- **Agent**: Windows self‑hosted agent with **Docker** and **Azure CLI** installed.

> ⚠️ Never commit secrets. Store `AZURE_OPENAI_KEY` only as a **secret** in DevOps variable group or pipeline variable.

---

## 4) Local Development

```bash
mvn -q -DskipTests package
java -jar target/claim-status-api-1.0.0.jar

# Local env for Azure OpenAI
export AZURE_OPENAI_ENDPOINT="https://claims-openaires.services.ai.azure.com/"
export AZURE_OPENAI_KEY="<your key>"
export AZURE_OPENAI_DEPLOYMENT="gpt-5-mini"
export AZURE_OPENAI_API_VERSION="2024-02-15-preview"
```

**Endpoints**  
- `GET http://localhost:8080/claims/C-1001`  
- `POST http://localhost:8080/claims/C-1001/summarize`

---

## 5) Azure Portal Setup (once)

1) **Container App (claims-aca)** ➜ **Ingress**: **External**, **Target port**: `8080`, **Accept traffic: Anywhere**.  
2) **ACA Environment** ➜ **Monitoring → Diagnostic settings** ➜ **Send to Log Analytics** (workspace).  
3) **Azure OpenAI** ➜ **Keys and Endpoint**; **Deployments** ➜ `gpt-5-mini`.  
4) **APIM** ➜ import later after first pipeline deploy.

---

## 6) Push the Code to Azure DevOps

**New empty repo**
```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://dev.azure.com/<org>/<project>/_git/<repo>
git push -u origin main
```

**Existing clone**
```bash
git checkout main
git pull origin main
git add .
git commit -m "Update"
git push origin main
```

---

## 7) CI/CD — Azure Pipelines (Windows Agent)

Pipeline at `pipelines/azure-pipelines.yml`:
1) Build JAR (**Maven@4**).  
2) Build & push image to ACR (**Docker@2**).  
3) (Optional) **Trivy** scan on Windows.  
4) Deploy/update **ACA app** (**AzureCLI@2**), set env & secret, ensure external ingress 8080, print ACA FQDN.

**Variables already wired:**
```
- group: Claims
ACR_LOGIN_SERVER=claimsintrospect2b.azurecr.io
RESOURCE_GROUP=introspect-2b    LOCATION=eastus2
ACA_ENV=managedEnvironment-introspect2b-b108
ACA_APP=claims-aca
AZURE_OPENAI_ENDPOINT=https://claims-openaires.services.ai.azure.com/
AZURE_OPENAI_DEPLOYMENT=gpt-5-mini
imageName=claim-status-api      tag=$(Build.BuildId)
TRIVY_VERSION=0.57.1
ACR_SERVICE_CONNECTION=acr-claimsintrospect2b-sc
AZURE_SERVICE_CONNECTION=arm-introspect2b-rg-sc
SUBSCRIPTION_ID=f685f234-7872-4fbe-9f8c-5143fe0bb1e0
```

---

## 8) API Management (APIM)

1) Copy **ACA FQDN** from pipeline logs (e.g., `claims-aca.<region>.azurecontainerapps.io`).  
2) **APIs → + Add API → OpenAPI**: import `apim/openapi.yaml`.  
3) Backend URL = `https://<ACA_FQDN>`.  
4) Update `apim/api-policy.xml` (`<YOUR-ACA-INGRESS-HOST>`) and apply.  
5) Test via APIM (subscription key).

---

## 9) Observability

- ACA → Logs (Console/System) via Log Analytics (`ContainerAppConsoleLogs_CL`, `ContainerAppSystemLogs_CL`).  
- APIM → Analytics; optional export to Log Analytics.  
- Use KQL samples in `observability/`.

---

## 10) Security & Scanning

- CI: **Trivy** (`HIGH,CRITICAL`, `--exit-code 1`).  
- Registry: **Defender for Containers** scans ACR on push/import/pull.  
- RBAC: SPN has **AcrPush** (ACR) and **Contributor** (RG).

---

## 11) Architecture

```
Developer -> Git push (Azure Repos)
          -> Pipeline (Windows agent)
             - mvn package
             - docker build & push (ACR)
             - deploy/update ACA (image + env + secret)
APIM (subscription key) -> Container Apps (Public Ingress 8080) -> Spring Boot API
                                                   |
                                                   -> Azure OpenAI (endpoint+key)
Logs: APIM Analytics; ACA -> Log Analytics (KQL)
```

---

## 12) Troubleshooting

- **ACR push denied**: grant **AcrPush** to Docker Registry connection SPN.  
- **Provider errors**: register `Microsoft.App`, `Microsoft.OperationalInsights`.  
- **App unreachable**: Ingress **External**, port **8080**.  
- **No logs**: enable Environment **Diagnostic settings** to Log Analytics.  
- **APIM 401**: include subscription key; backend URL is ACA FQDN.  
- **Git push prompts/fails**: use **GCM** (HTTPS SSO) or **SSH**.

---

## 13) Operations

- **Roll forward**: push to `main`; image tagged with `$(Build.BuildId)`.  
- **Roll back**: switch ACA revision/image; or deploy older ACR tag.  
- **Scale**: `az containerapp update --min-replicas/--max-replicas`.  
- **Rotate OpenAI key**: update variable group `Claims`; rerun deploy.

---

## 14) Appendix

### Create ACR (CLI)
```bash
az acr create -n claimsintrospect2b -g introspect-2b --sku Standard
```

### Push local folder to Azure Repos
```bash
git init && git add . && git commit -m "Initial commit"
git branch -M main
git remote add origin https://dev.azure.com/<org>/<project>/_git/<repo>
git push -u origin main
```

---

## 15) Checklist

- [ ] ACR exists; SPN has **AcrPush**.  
- [ ] ACA env linked to Log Analytics (Diagnostic settings).  
- [ ] ACA app ingress **External**, **8080**.  
- [ ] OpenAI endpoint & deployment exist; key in variable group `Claims`.  
- [ ] APIM imported OpenAPI; backend → ACA FQDN; policy applied.  
- [ ] DevOps service connections authorized.  
- [ ] Pipeline built/pushed image; ACA updated; FQDN printed.
