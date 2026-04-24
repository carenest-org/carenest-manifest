# CareNest — Manifest Repository (GitOps Source of Truth)

This repository contains the **Helm chart** that defines the entire CareNest platform deployment. ArgoCD watches this repo and auto-syncs any changes to the Kubernetes cluster.

---

## 📁 Repository Structure

```
helm/
└── carenest/
    ├── Chart.yaml                    # Helm chart metadata
    ├── values.yaml                   # All configurable values (images, resources, etc.)
    └── templates/
        ├── auth-deployment.yaml      # Auth microservice deployment
        ├── auth-service.yaml         # Auth ClusterIP service
        ├── appointment-deployment.yaml
        ├── appointment-service.yaml
        ├── pharmacy-deployment.yaml
        ├── pharmacy-service.yaml
        ├── notify-deployment.yaml
        ├── notify-service.yaml
        ├── frontend-deployment.yaml
        ├── frontend-service.yaml
        ├── gateway.yaml              # Envoy Gateway + EnvoyProxy CRD
        ├── httproute.yaml            # HTTP routing rules
        ├── gateway-nodeport.yaml     # NodePort config (disabled)
        ├── configmap.yaml            # Non-sensitive config
        ├── secrets.yaml              # Sensitive config (base64)
        ├── mongo-statefulset.yaml    # MongoDB StatefulSet
        ├── mongo-service.yaml        # MongoDB headless service
        ├── redis-deployment.yaml     # Redis cache
        ├── redis-service.yaml        # Redis ClusterIP service
        ├── haproxy-deployment.yaml   # In-cluster HAProxy (disabled)
        ├── networkpolicy.yaml        # Network security policies
        ├── rbac.yaml                 # ServiceAccount, Role, RoleBinding
        ├── pdb.yaml                  # Pod Disruption Budgets
        ├── pvc.yaml                  # Persistent Volume Claims
        └── hpa-frontend.yaml        # Frontend Horizontal Pod Autoscaler
```

---

## 🐳 Per-Service Image Configuration

Each microservice has its own `image.repository` and `image.tag` in `values.yaml`:

```yaml
auth:
  image:
    repository: jayadevarun2003/carenest-auth-service
    tag: latest          # ← Updated by CI/CD pipeline

appointment:
  image:
    repository: jayadevarun2003/carenest-appointment-service
    tag: latest

pharmacy:
  image:
    repository: jayadevarun2003/carenest-pharmacy-service
    tag: latest

notify:
  image:
    repository: jayadevarun2003/carenest-notify-service
    tag: latest

frontend:
  image:
    repository: jayadevarun2003/carenest-frontend
    tag: latest
```

**Why this structure?**
- The CD pipeline uses `yq` to update **only** the `tag` field for a single service
- No risk of accidentally modifying other services during automated updates
- Each deployment template references: `"{{ .Values.<service>.image.repository }}:{{ .Values.<service>.image.tag }}"`

---

## 🔄 GitOps Workflow

```
┌──────────────┐     ┌───────────────────┐     ┌──────────────┐
│  CI Pipeline │────►│  This Manifest    │────►│   ArgoCD     │
│  (per-svc)   │     │  Repository       │     │  Auto-Sync   │
└──────────────┘     └───────────────────┘     └──────┬───────┘
                                                       │
                     ┌─────────────────────────────────▼──────┐
                     │         Kubernetes Cluster              │
                     │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
                     │  │  Auth   │ │ Appoint  │ │Pharmacy │  │
                     │  └─────────┘ └─────────┘ └─────────┘  │
                     │  ┌─────────┐ ┌─────────┐              │
                     │  │ Notify  │ │Frontend │              │
                     │  └─────────┘ └─────────┘              │
                     └────────────────────────────────────────┘
```

### Flow:
1. Developer pushes code to a microservice repo (e.g., `carenest-auth-service`)
2. CI pipeline runs: SonarQube → Build → Snyk → Docker+Trivy → Push
3. CD step clones this manifest repo
4. Updates `auth.image.tag` in `values.yaml` to the new commit SHA
5. Commits and pushes the change
6. **ArgoCD detects the change** and auto-syncs the deployment
7. Kubernetes performs a rolling update with the new image

### Example CD Commit:
```
chore(cd): update auth image to sha-abc1234

Service: auth
Image Tag: sha-abc1234
Triggered by: org/carenest-auth-service@full-sha
```

---

## 🏗️ Helm Architecture

### Infrastructure Layer
| Component | Type | Managed By CI/CD? |
|---|---|---|
| MongoDB | StatefulSet | ❌ (infra image) |
| Redis | Deployment | ❌ (infra image) |
| Envoy Gateway | CRD + Gateway | ❌ (platform) |
| HAProxy | Deployment (disabled) | ❌ (external EC2) |

### Application Layer
| Component | Type | Managed By CI/CD? |
|---|---|---|
| Auth Service | Deployment | ✅ |
| Appointment Service | Deployment | ✅ |
| Pharmacy Service | Deployment | ✅ |
| Notify Service | Deployment | ✅ |
| Frontend | Deployment | ✅ |

### Security Layer
| Component | Purpose |
|---|---|
| NetworkPolicies | Zero-trust pod-to-pod communication |
| RBAC | Least-privilege service account |
| Secrets | Base64-encoded sensitive config |
| PDB | Disruption budgets for availability |

---

## 🛠️ Manual Operations

### Deploy the chart
```bash
helm upgrade --install carenest helm/carenest \
  --namespace carenest-dev \
  --create-namespace
```

### Override a value
```bash
helm upgrade carenest helm/carenest \
  --namespace carenest-dev \
  --set auth.image.tag=sha-abc1234
```

### Lint the chart
```bash
helm lint helm/carenest
```

### Template render (dry run)
```bash
helm template carenest helm/carenest
```

---

## ⚠️ Important Notes

- **Never manually edit image tags** — let the CI/CD pipeline handle it
- ArgoCD must have read access to this repository
- The `production` branch can be used for production deployments with separate `values-prod.yaml`
- Infrastructure images (`mongo`, `redis`, `busybox`) are defined under `infraImages:` and are NOT updated by CI/CD
