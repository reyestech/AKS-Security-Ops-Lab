# AKS-Security-Ops-Lab

----

# This script will generate a ready-to-use AKS Azure-only starter pack
# matching the user's repo style, with a Quickstart and copy/paste-ready scripts.
#
# It creates a small folder tree and zips it so the user can download everything at once.

import os, json, textwrap, zipfile, pathlib

root = "/mnt/data/aks-azure-only-starter"
docs = os.path.join(root, "docs")
infra = os.path.join(root, "infra")
manifests = os.path.join(root, "manifests")
workflows = os.path.join(root, ".github", "workflows")

for d in [root, docs, infra, manifests, workflows]:
    os.makedirs(d, exist_ok=True)

quickstart = r"""# AKS Security & Ops Lab — **Azure-Only Quickstart** (3-node)

> This mirrors the **Azure Phase** of your repo (no bare-metal). Everything below is **copy/paste-ready**.  
> Tools: **Azure CLI ≥ 2.60**, **kubectl ≥ 1.30**, **Helm 3**. Shell: **PowerShell**.

---

<details>
<summary><b>0) One-time prerequisites</b> — Click to expand</summary>

```powershell
# Check Azure CLI
az version

# Install kubectl (if needed)
az aks install-cli
kubectl version --client

# Optional: Helm
choco install kubernetes-helm -y  # Windows + Chocolatey (or use your preferred method)
```
</details>

<details> <summary><b>1) Variables (edit to taste)</b></summary>
$RG       = "rg-aks-homelab"
$LOC      = "eastus"
$CLUSTER  = "aks-homelab"
$NODECNT  = 3
$SIZE     = "Standard_B2s"           # bump if you need more headroom
$LA       = "la-aks-homelab"
$ACR      = "youruniqueacr1234"      # lowercase letters/numbers only
</details>

<details open> <summary><b>2) Create Resource Group, Log Analytics, and ACR</b></summary>
az group create --name $RG --location $LOC
az monitor log-analytics workspace create -g $RG -n $LA
az acr create -g $RG -n $ACR --sku Basic
</details>



<details> <summary><b>3) Create AKS (Option A: simple & fast)</b></summary>

Matches your repo’s "Cluster Provisioning" style.
az aks create `
  --resource-group $RG `
  --name $CLUSTER `
  --node-count $NODECNT `
  --node-vm-size $SIZE `
  --enable-addons monitoring `
  --generate-ssh-keys

az aks get-credentials --resource-group $RG --name $CLUSTER
kubectl get nodes -o wide
</details>


<details> <summary><b>3B) Create AKS (Option B: modern flags)</b></summary>
Enables OIDC/Workload Identity and Azure Monitor metrics from day one.

az aks create `
  -g $RG -n $CLUSTER `
  --node-count $NODECNT --node-vm-size $SIZE `
  --enable-managed-identity `
  --enable-oidc-issuer --enable-workload-identity `
  --attach-acr $ACR `
  --enable-azure-monitor-metrics

az aks get-credentials -g $RG -n $CLUSTER
kubectl get nodes -o wide
</details>



<details open> <summary><b>4) Deploy baseline app (+ Service)</b></summary>

kubectl create ns demo
kubectl -n demo apply -f manifests/deployment.yaml
kubectl -n demo apply -f manifests/service-lb.yaml
kubectl -n demo get svc web -o wide
</details>




<details> <summary><b>5) (Optional) Ingress + TLS</b></summary>

Keep the LoadBalancer service for speed. Later, swap to Ingress by applying manifests/ingress.yaml and changing the Service to ClusterIP.
# Example (if you decide to use NGINX Ingress later)
# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
# helm repo update
# helm install ingress ingress-nginx/ingress-nginx -n ingress-basic --create-namespace
# kubectl -n demo apply -f manifests/ingress.yaml
</details>

<details> <summary><b>6) (Optional) Connect Sentinel</b></summary>

In Azure Portal: Microsoft Sentinel → + Add → select your $LA workspace.

Add the Azure Kubernetes/Container Insights connector.

Verify Container Insights charts are populating and LA has data flowing.

</details>


<details> <summary><b>8) Clean up (deletes all resources)</b></summary>
  # This removes the entire lab resource group
az group delete --name $RG --yes --no-wait

</details>


Repo layout suggestion
infra/            # scripts for RG/LA/ACR/AKS create/destroy
manifests/        # sample app + service (+ optional ingress)
docs/             # this quickstart + screenshots
.github/workflows # (optional) CI/CD to AKS



----[ Screenshots to capture for my README will go here ]

kubectl get nodes showing 3 Ready nodes

Azure Monitor / Container Insights overview

The web service external IP responding in a browser

Tag suggestion when done: v0.1-azure-phase
"""

create_ps1 = r"""<# AKS Azure-Only Lab — create.ps1
Usage: .\create.ps1 (edit variables below first)
#>

$ErrorActionPreference = "Stop"

===== Variables =====

$RG = $env:RG ; if (-not $RG) { $RG = "rg-aks-homelab" }
$LOC = $env:LOC ; if (-not $LOC) { $LOC = "eastus" }
$CLUSTER = $env:CLUSTER ; if (-not $CLUSTER) { $CLUSTER = "aks-homelab" }
$NODECNT = $env:NODECNT ; if (-not $NODECNT) { $NODECNT = 3 }
$SIZE = $env:SIZE ; if (-not $SIZE) { $SIZE = "Standard_B2s" }
$LA = $env:LA ; if (-not $LA) { $LA = "la-aks-homelab" }
$ACR = $env:ACR ; if (-not $ACR) { $ACR = "youruniqueacr1234" }

Write-Host "Creating RG/LA/ACR in $LOC..." -ForegroundColor Cyan
az group create --name $RG --location $LOC | Out-Null
az monitor log-analytics workspace create -g $RG -n $LA | Out-Null
az acr create -g $RG -n $ACR --sku Basic | Out-Null

Write-Host "Creating AKS cluster ($NODECNT nodes)..." -ForegroundColor Cyan
az aks create --resource-group $RG
--name $CLUSTER --node-count $NODECNT
--node-vm-size $SIZE --enable-addons monitoring
--generate-ssh-keys | Out-Null

Write-Host "Getting kubeconfig..." -ForegroundColor Cyan
az aks get-credentials --resource-group $RG --name $CLUSTER | Out-Null

Write-Host "Validating nodes..." -ForegroundColor Cyan
kubectl get nodes -o wide
"""

destroy_ps1 = r"""<# AKS Azure-Only Lab — destroy.ps1
WARNING: Deletes the entire resource group.
Usage: .\destroy.ps1
#>

$ErrorActionPreference = "Stop"
$RG = $env:RG ; if (-not $RG) { $RG = "rg-aks-homelab" }

Write-Host "Deleting resource group $RG ..." -ForegroundColor Yellow
az group delete --name $RG --yes --no-wait
Write-Host "Delete initiated."
"""

deployment_yaml = r"""apiVersion: apps/v1
kind: Deployment
metadata:
name: web
namespace: demo
labels:
app: web
spec:
replicas: 2
selector:
matchLabels:
app: web
template:
metadata:
labels:
app: web
spec:
containers:
- name: web
image: mcr.microsoft.com/azuredocs/aks-helloworld
ports:
- containerPort: 80
resources:
requests:
cpu: "100m"
memory: "128Mi"
limits:
cpu: "250m"
memory: "256Mi"
"""

service_lb_yaml = r"""apiVersion: v1
kind: Service
metadata:
name: web
namespace: demo
labels:
app: web
spec:
type: LoadBalancer
selector:
app: web
ports:

name: http
port: 80
targetPort: 80
"""

ingress_yaml = r"""apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: web
namespace: demo
annotations:
nginx.ingress.kubernetes.io/rewrite-target: /
spec:
ingressClassName: nginx
rules:

http:
paths:

path: /
pathType: Prefix
backend:
service:
name: web
port:
number: 80
"""

workflow_yaml = r"""name: build-and-deploy-aks
on:
push:
branches: [ "main" ]
permissions:
id-token: write
contents: read

jobs:
cd:
runs-on: ubuntu-latest
env:
RG: rg-aks-homelab
CLUSTER: aks-homelab
NAMESPACE: demo
steps:
- uses: actions/checkout@v4

  - name: Azure login (OIDC)
    uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

  - name: Set AKS context
    uses: Azure/aks-set-context@v4
    with:
      resource-group: ${{ env.RG }}
      cluster-name: ${{ env.CLUSTER }}

  - name: Deploy manifests
    uses: azure/k8s-deploy@v5
    with:
      namespace: ${{ env.NAMESPACE }}
      manifests: |
        manifests/deployment.yaml
        manifests/service-lb.yaml
"""

Write files

with open(os.path.join(docs, "quickstart.md"), "w", encoding="utf-8") as f:
f.write(quickstart)

with open(os.path.join(infra, "create.ps1"), "w", encoding="utf-8") as f:
f.write(create_ps1)

with open(os.path.join(infra, "destroy.ps1"), "w", encoding="utf-8") as f:
f.write(destroy_ps1)

with open(os.path.join(manifests, "deployment.yaml"), "w", encoding="utf-8") as f:
f.write(deployment_yaml)

with open(os.path.join(manifests, "service-lb.yaml"), "w", encoding="utf-8") as f:
f.write(service_lb_yaml)

with open(os.path.join(manifests, "ingress.yaml"), "w", encoding="utf-8") as f:
f.write(ingress_yaml)

with open(os.path.join(workflows, "build-and-deploy-aks.yml"), "w", encoding="utf-8") as f:
f.write(workflow_yaml)

Zip it

zip_path = "/mnt/data/aks-azure-only-starter.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
for folder, _, files in os.walk(root):
for file in files:
full = os.path.join(folder, file)
rel = os.path.relpath(full, root)
z.write(full, arcname=os.path.join("aks-azure-only-starter", rel))

zip_path

Result
'/mnt/data/aks-azure-only-starter.zip'


Done! I packaged a clean AKS Azure-only starter you can drop into your repo, with a Quickstart and copy-paste–ready scripts/YAML.

--[Download the ZIP]


What’s inside:
aks-azure-only-starter/
├─ docs/
│  └─ quickstart.md                # step-by-step with copyable code blocks (GitHub shows copy buttons)
├─ infra/
│  ├─ create.ps1                   # one-shot Azure setup (RG, LA, ACR, AKS)
│  └─ destroy.ps1                  # clean-up script (deletes RG)
├─ manifests/
│  ├─ deployment.yaml              # baseline app (2 replicas)
│  ├─ service-lb.yaml              # LoadBalancer service
│  └─ ingress.yaml                 # optional NGINX ingress
└─ .github/workflows/
   └─ build-and-deploy-aks.yml     # optional GitHub Actions deploy


   How to use (fast path):

Unzip into your azure-kubernetes-hlab/ repo root.

Edit variables in infra/create.ps1 (ACR name, region, etc.).

Run:
cd infra
.\create.ps1
kubectl create ns demo
kubectl -n demo apply -f ../manifests/deployment.yaml
kubectl -n demo apply -f ../manifests/service-lb.yaml
kubectl -n demo get svc web -o wide

