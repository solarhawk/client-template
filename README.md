# Dorkomen Client Template

A Kustomize-based client template for deploying custom applications alongside the [Dorkomen Base Chart](https://github.com/solarhawk/base). This template is designed to be deployed via **ArgoCD** (included in the base chart) and enables clients to add their own applications while leveraging the shared platform infrastructure.

## What's Included

- **Microservices Demo** - Google's Online Boutique (11-service e-commerce demo)
- **App Template** - Scaffolding for adding your own applications
- **Environment Overlays** - Dev and Prod configurations with domain patching
- **Traefik Ingress** - Pre-configured IngressRoutes with TLS

---

## Prerequisites

Before deploying this client template, ensure:

1. **Dorkomen Base Chart is deployed** via FluxCD
   - See [Base Chart README](https://github.com/solarhawk/base) for installation
2. **ArgoCD is running** and accessible
3. **DNS configured** for your domain (or hosts file for local development)

---

## Quick Start

### Step 1: Create Your Repository

**Option A: Fork on GitHub**

1. Click "Fork" on GitHub to create your own copy
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/YOUR_FORK.git
   cd YOUR_FORK
   ```

**Option B: Create a new repository**

```bash
# Clone this template
git clone https://github.com/solarhawk/client-template.git my-client
cd my-client

# Remove the original remote and add your own
git remote remove origin
git remote add origin https://github.com/YOUR_ORG/YOUR_REPO.git

# Push to your repository
git push -u origin main
```

### Step 2: Configure Your Environment

Edit the overlay for your environment:

**Development:** `overlays/dev/configmap-values.yaml`
**Production:** `overlays/prod/configmap-values.yaml`

---

## Deploying with ArgoCD

### Step 1: Access ArgoCD

Open your browser and navigate to ArgoCD:

```
https://argocd.<your-domain>
```

**Get the admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Login credentials:**
- Username: `admin`
- Password: (from command above)

---

### Step 2: Configure Repository Access

#### For Public Repositories

No additional configuration needed. Skip to Step 3.

#### For Private Repositories

You must configure ArgoCD to access your private repository.

**Option A: Via ArgoCD UI**

1. Navigate to **Settings** → **Repositories**
2. Click **+ Connect Repo**
3. Choose connection method:

   **HTTPS with Token:**
   | Field | Value |
   |-------|-------|
   | Repository URL | `https://github.com/YOUR_ORG/YOUR_REPO.git` |
   | Username | Your GitHub username or `oauth2` for tokens |
   | Password | Your GitHub Personal Access Token |

   **SSH:**
   | Field | Value |
   |-------|-------|
   | Repository URL | `git@github.com:YOUR_ORG/YOUR_REPO.git` |
   | SSH private key | Paste your SSH private key |

4. Click **Connect**

**Option B: Via kubectl (Recommended for GitOps)**

Create a secret for repository credentials:

```bash
# For HTTPS with GitHub Token
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: my-client-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/YOUR_ORG/YOUR_REPO.git
  username: oauth2
  password: YOUR_GITHUB_PAT
EOF
```

```bash
# For SSH Key
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: my-client-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:YOUR_ORG/YOUR_REPO.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    YOUR_PRIVATE_KEY_HERE
    -----END OPENSSH PRIVATE KEY-----
EOF
```

**Creating a GitHub Personal Access Token:**

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click **Generate new token (classic)**
3. Select scopes:
   - `repo` (Full control of private repositories)
4. Copy the token immediately (it won't be shown again)

---

### Step 3: Create ArgoCD Project

Projects provide logical grouping and access control for applications.

**Via ArgoCD UI:**

1. Navigate to **Settings** → **Projects**
2. Click **+ New Project**
3. Configure:

   | Field | Value |
   |-------|-------|
   | Name | `my-client` |
   | Description | `My Client Applications` |

4. Under **Source Repositories**, click **Add** and enter:
   ```
   https://github.com/YOUR_ORG/YOUR_REPO.git
   ```
   Or use `*` to allow all repositories

5. Under **Destinations**, click **Add**:

   | Field | Value |
   |-------|-------|
   | Server | `https://kubernetes.default.svc` |
   | Namespace | `*` (or specific namespaces) |

6. Click **Create**

**Via kubectl:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-client
  namespace: argocd
spec:
  description: My Client Applications

  # Allow manifests from this repository
  sourceRepos:
    - https://github.com/YOUR_ORG/YOUR_REPO.git

  # Allow deployment to these destinations
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc

  # Allow all cluster-scoped resources (namespaces, etc.)
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

  # Allow all namespaced resources
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
EOF
```

---

### Step 4: Create ArgoCD Application

**Via ArgoCD UI:**

1. Click **+ New App**
2. Configure **General** section:

   | Field | Value |
   |-------|-------|
   | Application Name | `my-client-dev` |
   | Project Name | `my-client` |
   | Sync Policy | `Automatic` (recommended) |

3. Enable sync options:
   - ✅ **Auto-Create Namespace**
   - ✅ **Prune Resources** (removes deleted resources)
   - ✅ **Self Heal** (reverts manual changes)

4. Configure **Source** section:

   | Field | Value |
   |-------|-------|
   | Repository URL | `https://github.com/YOUR_ORG/YOUR_REPO.git` |
   | Revision | `HEAD` or `main` |
   | Path | `overlays/dev` |

5. Configure **Destination** section:

   | Field | Value |
   |-------|-------|
   | Cluster URL | `https://kubernetes.default.svc` |
   | Namespace | `dorkomen` |

6. Click **Create**

**Via kubectl:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-client-dev
  namespace: argocd
spec:
  project: my-client

  source:
    repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
    targetRevision: HEAD
    path: overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: dorkomen

  syncPolicy:
    automated:
      prune: true      # Delete resources removed from git
      selfHeal: true   # Revert manual changes
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
EOF
```

---

### Step 5: Verify Deployment

**Via ArgoCD UI:**

1. Navigate to **Applications**
2. Click on your application
3. View the resource tree and sync status
4. All resources should show green ✓ when healthy

**Via kubectl:**

```bash
# Check ArgoCD Application status
kubectl get applications -n argocd

# Check deployed resources
kubectl get all -n microservices-demo

# Check HelmReleases
kubectl get helmreleases -n dorkomen
```

---

## Accessing the Demo Application

Once deployed, access the Online Boutique:

| Environment | URL |
|-------------|-----|
| Development | `https://boutique.dev.dorkomen.local` |
| Production | `https://boutique.dorkomen.example.com` |

**For local development**, add to your hosts file:

**Windows:** `C:\Windows\System32\drivers\etc\hosts`
**Linux/Mac:** `/etc/hosts`

```
127.0.0.1 boutique.dev.dorkomen.local
```

---

## Adding Your Own Applications

### Step 1: Copy the App Template

```bash
cp -r apps/_app-template apps/my-new-app
```

### Step 2: Configure Your Application

Edit the files in `apps/my-new-app/`:

**namespace.yaml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-app
  labels:
    app.kubernetes.io/name: my-new-app
```

**helmrelease.yaml:**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-new-app
  namespace: dorkomen
spec:
  targetNamespace: my-new-app
  chart:
    spec:
      chart: my-chart
      version: "1.0.0"
      sourceRef:
        kind: HelmRepository
        name: my-repo
        namespace: dorkomen
  values:
    # Your chart values here
```

### Step 3: Enable the Application

Edit `apps/kustomization.yaml`:

```yaml
resources:
  - microservices-demo
  - my-new-app  # Add your app here
```

### Step 4: Add Environment Patches (Optional)

Edit `overlays/dev/kustomization.yaml` to add environment-specific patches:

```yaml
patches:
  # ... existing patches ...

  # Patch for your new app
  - target:
      kind: HelmRelease
      name: my-new-app
    patch: |
      - op: replace
        path: /spec/values/replicas
        value: 1
```

### Step 5: Commit and Push

```bash
git add .
git commit -m "Add my-new-app"
git push
```

ArgoCD will automatically sync the changes (if auto-sync is enabled).

---

## Managing Secrets Securely

**Never commit secrets to git.** Use one of these approaches:

### Option 1: Kubernetes Secrets (Manual)

Create secrets directly in the cluster:

```bash
kubectl create secret generic my-app-secrets \
  --namespace my-new-app \
  --from-literal=api-key=YOUR_API_KEY \
  --from-literal=db-password=YOUR_PASSWORD
```

Reference in your HelmRelease:

```yaml
values:
  existingSecret: my-app-secrets
```

### Option 2: Sealed Secrets (GitOps-friendly)

Install Sealed Secrets controller, then create sealed secrets that can be safely committed:

```bash
# Install kubeseal CLI
brew install kubeseal  # macOS

# Create a sealed secret
kubectl create secret generic my-app-secrets \
  --namespace my-new-app \
  --from-literal=api-key=YOUR_API_KEY \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > apps/my-new-app/sealed-secret.yaml
```

### Option 3: External Secrets Operator

For production, consider using External Secrets Operator with:
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- Google Secret Manager

---

## Environment Configuration

### Development Overlay

`overlays/dev/kustomization.yaml`:
- Points base chart to `develop` branch
- Uses `dev.dorkomen.local` domain
- Relaxed resource limits

### Production Overlay

`overlays/prod/kustomization.yaml`:
- Points base chart to specific version tag (e.g., `v0.1.0`)
- Uses `dorkomen.example.com` domain
- Production-ready resource limits

### Customizing Domains

Edit the patches in your overlay's `kustomization.yaml`:

```yaml
patches:
  - target:
      kind: IngressRoute
      name: microservices-demo
    patch: |
      - op: replace
        path: /spec/routes/0/match
        value: Host(`boutique.your-domain.com`)
```

---

## Sync Strategies

### Automatic Sync (Recommended)

ArgoCD automatically applies changes when git is updated:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### Manual Sync

Require explicit sync actions:

1. Click **Sync** in ArgoCD UI
2. Select resources to sync
3. Click **Synchronize**

Or via CLI:

```bash
argocd app sync my-client-dev
```

---

## Troubleshooting

### Check Application Status

```bash
# List all applications
kubectl get applications -n argocd

# Describe specific application
kubectl describe application my-client-dev -n argocd

# Get sync status
argocd app get my-client-dev
```

### View Application Logs

In ArgoCD UI:
1. Click on your application
2. Click on a specific resource
3. Click **Logs** tab

### Force Sync

```bash
# Via ArgoCD CLI
argocd app sync my-client-dev --force

# Via kubectl
kubectl patch application my-client-dev -n argocd \
  --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"prune": true}}}'
```

### Check Resource Health

```bash
# Check all pods in the app namespace
kubectl get pods -n microservices-demo

# Check events
kubectl get events -n microservices-demo --sort-by='.lastTimestamp'

# Check HelmRelease status
kubectl describe helmrelease microservices-demo -n dorkomen
```

### Common Issues

| Issue | Solution |
|-------|----------|
| App stuck in "Syncing" | Check for resource conflicts or missing CRDs |
| "ComparisonError" | Ensure Kustomize path is correct |
| "Unable to connect to repository" | Verify repository credentials |
| Resources not pruning | Enable `prune: true` in sync policy |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                  │
│                    (GitOps Controller)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Watches & Syncs
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Client Template Repository                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ overlays/dev or overlays/prod                           │   │
│  │   ├── Base Chart Reference (HelmRelease)                │   │
│  │   └── Client Apps (microservices-demo, custom apps)     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│  ┌──────────────────┐  ┌──────────────────────────────────┐    │
│  │  dorkomen        │  │  microservices-demo              │    │
│  │  (Base Chart)    │  │  (Client App)                    │    │
│  │  - ArgoCD        │  │  - 11 microservices              │    │
│  │  - GitLab        │  │  - Traefik IngressRoute          │    │
│  │  - Elastic       │  │  - TLS Certificate               │    │
│  │  - cert-manager  │  │                                  │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
client-template/
├── kustomization.yaml          # Main kustomization (base)
├── namespace.yaml              # Dorkomen namespace
├── gitrepository.yaml          # FluxCD GitRepository for base chart
├── helmrelease.yaml            # HelmRelease for base chart
├── transformer.yaml            # Kustomize name transformer
│
├── apps/                       # Client applications
│   ├── kustomization.yaml      # Apps manifest
│   ├── _app-template/          # Template for new apps
│   └── microservices-demo/     # Demo application
│       ├── namespace.yaml
│       ├── gitrepository.yaml
│       ├── helmrelease.yaml
│       ├── certificate.yaml
│       ├── ingressroute.yaml
│       └── http-redirect.yaml
│
└── overlays/                   # Environment configurations
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── configmap-values.yaml
    │   └── secret-values.yaml
    └── prod/
        ├── kustomization.yaml
        ├── configmap-values.yaml
        └── secret-values.yaml
```

---

## Multi-Environment Deployment

For managing multiple environments, create separate ArgoCD Applications:

```bash
# Development
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-client-dev
  namespace: argocd
spec:
  project: my-client
  source:
    repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
    path: overlays/dev
    targetRevision: develop
  destination:
    server: https://kubernetes.default.svc
    namespace: dorkomen
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Production
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-client-prod
  namespace: argocd
spec:
  project: my-client
  source:
    repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git
    path: overlays/prod
    targetRevision: main  # Or specific tag like v1.0.0
  destination:
    server: https://kubernetes.default.svc
    namespace: dorkomen
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

---

## Updating the Base Chart Version

To update the base chart version, edit the GitRepository reference in your overlay:

**For Production (pinned version):**

Edit `overlays/prod/kustomization.yaml`:

```yaml
patches:
  - target:
      kind: GitRepository
      name: dorkomen
    patch: |
      - op: replace
        path: /spec/ref
        value:
          tag: v0.2.0  # Update to new version
```

**For Development (follow branch):**

Edit `overlays/dev/kustomization.yaml`:

```yaml
patches:
  - target:
      kind: GitRepository
      name: dorkomen
    patch: |
      - op: replace
        path: /spec/ref
        value:
          branch: develop
```

Commit and push—ArgoCD will sync automatically.

---

## License

See [LICENSE](LICENSE) for details.
