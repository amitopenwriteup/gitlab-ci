# GitLab CI/CD × Kubernetes Deployment Workshop

> **Duration:** Full Day (6–8 hrs) | **Level:** Intermediate–Advanced  
> **Prerequisites:** Basic Git, Docker knowledge, Docker Engine or Docker Desktop installed

---

## Table of Contents

1. [Workshop Overview](#overview)
2. [Module 1 — Environment Setup](#module-1)
3. [Module 2 — GitLab CI/CD Fundamentals](#module-2)
4. [Module 3 — Dockerizing Your Application](#module-3)
5. [Module 4 — Kubernetes Crash Course](#module-4)
6. [Module 5 — GitLab + Kubernetes Integration](#module-5)
7. [Module 6 — Full Pipeline: Build → Push → Deploy](#module-6)
8. [Module 7 — Advanced Patterns](#module-7)
9. [Module 8 — Monitoring & Troubleshooting](#module-8)
10. [Capstone Project](#capstone)
11. [Reference Cheat Sheet](#cheatsheet)

---

## Workshop Overview <a name="overview"></a>

By the end of this workshop you will be able to:

- Write multi-stage `.gitlab-ci.yml` pipelines
- Build and push Docker images to GitLab Container Registry
- Deploy applications to Kubernetes using `kubectl` and Helm from CI
- Implement environment-specific deployments (dev / staging / production)
- Use GitLab Environments, rollbacks, and protected branches
- Apply advanced patterns: Helm, canary deployments, secrets management

### Tools We Will Use

| Tool | Purpose |
|------|---------|
| GitLab (SaaS or self-hosted) | Source control + CI/CD |
| GitLab Container Registry | Docker image storage |
| Docker | Containerisation |
| kind (Kubernetes IN Docker) | Local container orchestration |
| kubectl | Kubernetes CLI |
| Helm | Kubernetes package manager |
| kubeconfig / Service Accounts | Auth between GitLab and K8s |

---

## Module 1 — Environment Setup <a name="module-1"></a>

**Duration: 45 min**

### 1.1 Local Kubernetes Cluster — kind (Kubernetes IN Docker)

kind runs a full Kubernetes cluster inside Docker containers — no VM overhead, fast startup, perfect for CI/CD workshops.

**Install kind**

```bash
# Linux / macOS (via Go)
go install sigs.k8s.io/kind@v0.22.0

# macOS via Homebrew
brew install kind

# Windows via Chocolatey
choco install kind
```

**Install kubectl**

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# macOS
brew install kubectl

# Verify
kubectl version --client
```

**Create a multi-node workshop cluster**

Create `kind-config.yaml`:
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: workshop
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
# Create the cluster
kind create cluster --config kind-config.yaml

# Verify — you should see 1 control-plane + 2 workers
kubectl get nodes

# Check context was set automatically
kubectl cluster-info --context kind-workshop
```

**Useful kind commands**

```bash
# List clusters
kind get clusters

# Load a local Docker image into kind (no registry needed for local testing)
kind load docker-image workshop-app:local --name workshop

# Delete cluster when done
kind delete cluster --name workshop
```

> **Note:** kind uses your existing Docker daemon — make sure Docker Desktop or Docker Engine is running before starting kind.

### 1.2 GitLab Project Setup

1. Create a new GitLab project (or use an existing one)
2. Go to **Settings → CI/CD → Runners** — note your project URL
3. Enable **Container Registry**: **Settings → Packages & Registries → Container Registry**

### 1.3 Required Variables (add under Settings → CI/CD → Variables)

| Variable | Value | Protected | Masked |
|----------|-------|-----------|--------|
| `KUBE_CONFIG` | Base64-encoded kubeconfig | ✅ | ✅ |
| `REGISTRY_USER` | GitLab username or token name | — | — |
| `REGISTRY_PASSWORD` | GitLab Personal Access Token | ✅ | ✅ |

**Generate `KUBE_CONFIG`:**
```bash
cat ~/.kube/config | base64 -w 0
# Copy output → paste into GitLab variable
```

### 1.4 Verify kubectl Access from GitLab

Create a quick test job to confirm connectivity (we'll build on this later):
```yaml
# Quick connectivity test — not a final pipeline
test-kube-access:
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
    - kubectl --kubeconfig=/tmp/kubeconfig get nodes
```

---

## Module 2 — GitLab CI/CD Fundamentals <a name="module-2"></a>

**Duration: 60 min**

### 2.1 Pipeline Anatomy

A GitLab pipeline lives in `.gitlab-ci.yml` at the root of your repository.

```
Repository
└── .gitlab-ci.yml       ← Pipeline definition
    ├── stages            ← Ordered list of phases
    ├── variables         ← Global variables
    └── jobs              ← Individual tasks
        ├── build
        ├── test
        └── deploy
```

### 2.2 Core Concepts

**Stages** define the order of execution. Jobs in the same stage run in parallel.

```yaml
stages:
  - build
  - test
  - deploy
```

**Jobs** are the actual work units:

```yaml
my-job:
  stage: build
  image: node:20-alpine
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
```

**Rules** control when a job runs:

```yaml
deploy-production:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual          # Require human approval
    - if: $CI_COMMIT_TAG    # Auto-deploy on git tags
      when: on_success
```

### 2.3 Predefined Variables (Key Ones)

| Variable | Value |
|----------|-------|
| `$CI_COMMIT_BRANCH` | Current branch name |
| `$CI_COMMIT_SHA` | Full commit SHA |
| `$CI_COMMIT_SHORT_SHA` | Short commit SHA (8 chars) |
| `$CI_REGISTRY` | GitLab Container Registry URL |
| `$CI_REGISTRY_IMAGE` | Full image path for this project |
| `$CI_ENVIRONMENT_NAME` | Active environment name |
| `$CI_PROJECT_NAME` | Project name |

### 2.4 Exercise 2A — Your First Pipeline

Create `.gitlab-ci.yml`:

```yaml
stages:
  - greet
  - info

hello-world:
  stage: greet
  image: alpine:3.19
  script:
    - echo "Hello from GitLab CI!"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Commit: $CI_COMMIT_SHORT_SHA"

environment-info:
  stage: info
  image: alpine:3.19
  script:
    - echo "Runner OS: $(uname -a)"
    - echo "Registry: $CI_REGISTRY_IMAGE"
```

Push to GitLab and watch it run under **CI/CD → Pipelines**.

---

## Module 3 — Dockerizing Your Application <a name="module-3"></a>

**Duration: 45 min**

### 3.1 Sample Application

We'll use a minimal Node.js web server throughout this workshop:

**`app.js`**
```javascript
const http = require('http');
const PORT = process.env.PORT || 8080;
const VERSION = process.env.APP_VERSION || 'unknown';

http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', version: VERSION }));
    return;
  }
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end(`Hello from version ${VERSION}!\n`);
}).listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

**`package.json`**
```json
{
  "name": "workshop-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node app.js",
    "test": "echo 'Tests passed'"
  }
}
```

### 3.2 Dockerfile (Multi-Stage)

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Production image
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# Non-root user for security
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup app.js .

USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["node", "app.js"]
```

### 3.3 Build & Test Locally

```bash
# Build
docker build -t workshop-app:local .

# Run
docker run -p 8080:8080 -e APP_VERSION=local workshop-app:local

# Test
curl http://localhost:8080/health
```

### 3.4 Push to GitLab Registry

```bash
# Login
docker login registry.gitlab.com

# Tag (replace with your project path)
docker tag workshop-app:local registry.gitlab.com/YOUR_GROUP/YOUR_PROJECT:latest

# Push
docker push registry.gitlab.com/YOUR_GROUP/YOUR_PROJECT:latest
```

---

## Module 4 — Kubernetes Crash Course <a name="module-4"></a>

**Duration: 60 min**

### 4.1 Core Resources

```
Kubernetes Cluster
├── Namespace (logical isolation)
│   ├── Deployment (desired state for pods)
│   │   └── Pod (running container)
│   ├── Service (network access to pods)
│   ├── ConfigMap (non-secret config)
│   ├── Secret (sensitive config)
│   └── Ingress (external HTTP routing)
```

### 4.2 Deployment Manifest

**`k8s/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: workshop
  labels:
    app: workshop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # Zero-downtime rollout
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      containers:
        - name: app
          image: registry.gitlab.com/YOUR_GROUP/YOUR_PROJECT:latest
          ports:
            - containerPort: 8080
          env:
            - name: APP_VERSION
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['version']
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### 4.3 Service Manifest

**`k8s/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: workshop-app-svc
  namespace: workshop
spec:
  selector:
    app: workshop-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### 4.4 Namespace & Registry Secret

```bash
# Create namespace
kubectl create namespace workshop

# Create image pull secret (so K8s can pull from GitLab Registry)
kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<your-gitlab-username> \
  --docker-password=<your-personal-access-token> \
  --namespace=workshop
```

Add to deployment spec:
```yaml
spec:
  imagePullSecrets:
    - name: gitlab-registry-secret
```

### 4.5 Exercise 4A — Manual Deploy

```bash
# Apply manifests
kubectl apply -f k8s/

# Watch rollout
kubectl rollout status deployment/workshop-app -n workshop

# Port-forward to test locally
kubectl port-forward svc/workshop-app-svc 8080:80 -n workshop

# In another terminal
curl http://localhost:8080/health
```

### 4.6 Key kubectl Commands

```bash
# Get resources
kubectl get pods,svc,deployments -n workshop

# Describe (great for debugging)
kubectl describe pod <pod-name> -n workshop

# Logs
kubectl logs -f deployment/workshop-app -n workshop

# Execute into pod
kubectl exec -it <pod-name> -n workshop -- sh

# Delete and re-apply
kubectl delete -f k8s/ && kubectl apply -f k8s/

# Rollback
kubectl rollout undo deployment/workshop-app -n workshop
```

---

## Module 5 — GitLab + Kubernetes Integration <a name="module-5"></a>

**Duration: 45 min**

### 5.1 Service Account for CI

Rather than giving GitLab your full kubeconfig, create a least-privilege service account:

**`k8s/ci-service-account.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-ci
  namespace: workshop
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-ci-role
  namespace: workshop
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch", "create"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-ci-rolebinding
  namespace: workshop
subjects:
  - kind: ServiceAccount
    name: gitlab-ci
    namespace: workshop
roleRef:
  kind: Role
  name: gitlab-ci-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f k8s/ci-service-account.yaml
```

### 5.2 Extract Service Account Token

```bash
# Create a long-lived token (K8s 1.24+)
kubectl create token gitlab-ci -n workshop --duration=8760h

# Or create a Secret-based token
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-ci-token
  namespace: workshop
  annotations:
    kubernetes.io/service-account.name: gitlab-ci
type: kubernetes.io/service-account-token
EOF

# Get the token
kubectl get secret gitlab-ci-token -n workshop -o jsonpath='{.data.token}' | base64 -d
```

### 5.3 Build kubeconfig for CI

```bash
# Get cluster CA
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'

# Get API server URL
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}'
```

Create a minimal kubeconfig:
```yaml
# ci-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
  - cluster:
      certificate-authority-data: <BASE64_CA>
      server: https://<API_SERVER>
    name: workshop-cluster
contexts:
  - context:
      cluster: workshop-cluster
      namespace: workshop
      user: gitlab-ci
    name: workshop
current-context: workshop
users:
  - name: gitlab-ci
    user:
      token: <SERVICE_ACCOUNT_TOKEN>
```

```bash
# Base64 encode and store as CI variable
cat ci-kubeconfig.yaml | base64 -w 0
# → paste as KUBE_CONFIG in GitLab CI/CD Variables
```

---

## Module 6 — Full Pipeline: Build → Push → Deploy <a name="module-6"></a>

**Duration: 90 min**

### 6.1 Complete `.gitlab-ci.yml`

```yaml
# ==================================================
# GitLab CI/CD Pipeline — Kubernetes Deployment
# ==================================================

stages:
  - build
  - test
  - deploy-dev
  - deploy-staging
  - deploy-production

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest
  DOCKER_BUILDKIT: "1"

# ──────────────────────────────────────────────────
# TEMPLATES (reusable anchors)
# ──────────────────────────────────────────────────

.kubectl-setup: &kubectl-setup
  before_script:
    - echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - kubectl version --client

.deploy-template: &deploy-template
  image: bitnami/kubectl:1.29
  <<: *kubectl-setup
  script:
    - >
      kubectl set image deployment/workshop-app
      app=$IMAGE_TAG
      -n $K8S_NAMESPACE
    - kubectl rollout status deployment/workshop-app -n $K8S_NAMESPACE --timeout=120s
    - kubectl get pods -n $K8S_NAMESPACE

# ──────────────────────────────────────────────────
# STAGE: build
# ──────────────────────────────────────────────────

build-image:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    # Login to GitLab Container Registry
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    # Build with cache from latest
    - |
      docker build \
        --cache-from $IMAGE_LATEST \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        --label "git.commit=$CI_COMMIT_SHA" \
        --label "git.branch=$CI_COMMIT_BRANCH" \
        -t $IMAGE_TAG \
        -t $IMAGE_LATEST \
        .
    # Push both tags
    - docker push $IMAGE_TAG
    - docker push $IMAGE_LATEST
  rules:
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG

# ──────────────────────────────────────────────────
# STAGE: test
# ──────────────────────────────────────────────────

unit-tests:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test
  coverage: '/Coverage: \d+\.\d+%/'

container-scan:
  stage: test
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  variables:
    TRIVY_EXIT_CODE: "1"
    TRIVY_SEVERITY: "HIGH,CRITICAL"
  script:
    - trivy image --exit-code $TRIVY_EXIT_CODE $IMAGE_TAG
  allow_failure: true   # Warn but don't block (adjust for production)

# ──────────────────────────────────────────────────
# STAGE: deploy-dev
# ──────────────────────────────────────────────────

deploy-dev:
  stage: deploy-dev
  <<: *deploy-template
  variables:
    K8S_NAMESPACE: workshop-dev
  environment:
    name: development
    url: https://dev.your-app.example.com
    on_stop: stop-dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_BRANCH =~ /^feature\/.*/

stop-dev:
  stage: deploy-dev
  image: bitnami/kubectl:1.29
  <<: *kubectl-setup
  variables:
    K8S_NAMESPACE: workshop-dev
    GIT_STRATEGY: none
  script:
    - kubectl scale deployment/workshop-app --replicas=0 -n $K8S_NAMESPACE
  environment:
    name: development
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
      when: manual

# ──────────────────────────────────────────────────
# STAGE: deploy-staging
# ──────────────────────────────────────────────────

deploy-staging:
  stage: deploy-staging
  <<: *deploy-template
  variables:
    K8S_NAMESPACE: workshop-staging
  environment:
    name: staging
    url: https://staging.your-app.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success

# ──────────────────────────────────────────────────
# STAGE: deploy-production
# ──────────────────────────────────────────────────

deploy-production:
  stage: deploy-production
  <<: *deploy-template
  variables:
    K8S_NAMESPACE: workshop-prod
  environment:
    name: production
    url: https://your-app.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual      # Always require human approval for prod
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
      when: on_success
```

### 6.2 Kubernetes Manifests with Environment Substitution

Use `envsubst` for dynamic image tags in manifests:

**`k8s/deployment.yaml`** (updated)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: ${REPLICAS}
  template:
    spec:
      containers:
        - name: app
          image: ${IMAGE_TAG}          # Replaced at deploy time
```

In your CI script:
```bash
- envsubst < k8s/deployment.yaml | kubectl apply -f -
```

### 6.3 Exercise 6A — Trigger a Full Pipeline

1. Create branches: `develop`, `main`
2. Push a change to `develop` — watch build + dev deploy
3. Merge to `main` — watch build + staging deploy
4. Manually trigger production deploy from GitLab UI
5. Go to **Deployments → Environments** to see all active environments

---

## Module 7 — Advanced Patterns <a name="module-7"></a>

**Duration: 60 min**

### 7.1 Helm-Based Deployment

Helm is the preferred way to manage Kubernetes applications at scale.

**Install Helm in CI:**
```yaml
.helm-setup: &helm-setup
  before_script:
    - echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - helm version
```

**Helm deploy job:**
```yaml
deploy-helm:
  stage: deploy-production
  image: alpine/helm:3.14
  <<: *helm-setup
  script:
    - >
      helm upgrade --install workshop-app ./helm/workshop-app
      --namespace workshop-prod
      --create-namespace
      --set image.tag=$CI_COMMIT_SHORT_SHA
      --set image.repository=$CI_REGISTRY_IMAGE
      --set replicaCount=3
      --set ingress.host=your-app.example.com
      --wait
      --timeout 5m
      --atomic                    # Rollback on failure
```

### 7.2 Canary Deployments

```yaml
# k8s/canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app-canary
  namespace: workshop-prod
  labels:
    app: workshop-app
    track: canary
spec:
  replicas: 1             # 1 canary vs 9 stable = 10% traffic
  selector:
    matchLabels:
      app: workshop-app
      track: canary
  template:
    metadata:
      labels:
        app: workshop-app
        track: canary
    spec:
      containers:
        - name: app
          image: ${IMAGE_TAG}   # New version
```

CI pipeline with canary stage:
```yaml
stages:
  - build
  - test
  - canary       # 10% traffic
  - promote      # 100% traffic (manual)
  - cleanup

deploy-canary:
  stage: canary
  script:
    - envsubst < k8s/canary-deployment.yaml | kubectl apply -f -
    - echo "Canary deployed. Monitor metrics before promoting."
  environment:
    name: production/canary
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

promote-to-stable:
  stage: promote
  script:
    - kubectl set image deployment/workshop-app app=$IMAGE_TAG -n workshop-prod
    - kubectl rollout status deployment/workshop-app -n workshop-prod
    - kubectl delete deployment/workshop-app-canary -n workshop-prod
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  needs: ["deploy-canary"]
```

### 7.3 Secrets Management with GitLab + K8s Secrets

**Never store secrets in manifests.** Use this pattern:

```yaml
# In CI: create/update a K8s secret from GitLab CI variables
create-k8s-secret:
  stage: deploy-dev
  image: bitnami/kubectl:1.29
  script:
    - echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
    - >
      kubectl create secret generic app-secrets
      --from-literal=DB_PASSWORD=$DB_PASSWORD
      --from-literal=API_KEY=$API_KEY
      --namespace workshop-dev
      --dry-run=client -o yaml | kubectl apply -f -
```

Reference in deployment:
```yaml
containers:
  - name: app
    envFrom:
      - secretRef:
          name: app-secrets
```

---

## Module 8 — Monitoring & Troubleshooting <a name="module-8"></a>

**Duration: 45 min**

### 8.1 Pipeline Debugging

**Common CI failures and fixes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `denied: access forbidden` | Registry auth failed | Check `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` vars |
| `cannot connect to the Docker daemon` | Missing `docker:dind` service | Add `services: [docker:dind]` |
| `connection refused` to K8s | Bad kubeconfig | Re-generate and re-encode `KUBE_CONFIG` |
| `ImagePullBackOff` in K8s | Missing image pull secret | Create registry secret in namespace |
| `ErrImageNeverPull` | Image tag mismatch | Verify `$IMAGE_TAG` resolves correctly |
| Timeout on rollout | App unhealthy | Check liveness/readiness probe endpoints |

**Debug a stuck job:**
```yaml
debug-job:
  stage: build
  image: alpine:3.19
  script:
    - env | sort                    # Print all CI variables
    - echo "$KUBE_CONFIG" | base64 -d | head -5   # Check kubeconfig structure
    - cat /etc/hosts
```

### 8.2 Kubernetes Debugging Commands

```bash
# Pod status
kubectl get pods -n workshop -o wide

# Detailed pod info (events at the bottom are gold)
kubectl describe pod <pod-name> -n workshop

# Container logs
kubectl logs <pod-name> -n workshop
kubectl logs <pod-name> -n workshop --previous    # Crashed container logs

# Deployment events
kubectl describe deployment workshop-app -n workshop

# All events in namespace (sorted by time)
kubectl get events -n workshop --sort-by='.lastTimestamp'

# Resource usage
kubectl top pods -n workshop
kubectl top nodes

# Interactive debug container
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Check service endpoints
kubectl get endpoints workshop-app-svc -n workshop

# Test DNS resolution inside cluster
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup workshop-app-svc.workshop.svc.cluster.local
```

### 8.3 Rollback Strategy

**Automatic rollback** (already in Helm with `--atomic`).

**Manual rollback:**
```bash
# Check rollout history
kubectl rollout history deployment/workshop-app -n workshop

# Rollback to previous version
kubectl rollout undo deployment/workshop-app -n workshop

# Rollback to specific revision
kubectl rollout undo deployment/workshop-app -n workshop --to-revision=3

# GitLab also supports rollback from Environments UI
# Deployments → Environments → [env] → Re-deploy previous version
```

### 8.4 Health Check Endpoint Pattern

Always implement `/health` and `/ready`:

```javascript
// Liveness — am I alive?
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// Readiness — am I ready to serve traffic?
app.get('/ready', async (req, res) => {
  try {
    await db.ping();   // Check database connection
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not ready', error: err.message });
  }
});
```

---

## Capstone Project <a name="capstone"></a>

**Duration: 60 min**

Build a complete GitLab CI/CD pipeline for a multi-tier app. 

### Requirements

1. **Repository structure:**
   ```
   ├── app.js
   ├── package.json
   ├── Dockerfile
   ├── .gitlab-ci.yml
   └── k8s/
       ├── namespace.yaml
       ├── deployment.yaml
       ├── service.yaml
       └── configmap.yaml
   ```

2. **Pipeline must:**
   - Build a Docker image on every push
   - Run at least one test job
   - Auto-deploy to `dev` on `develop` branch
   - Deploy to `staging` on merge to `main`
   - Require manual approval for `production`
   - Use GitLab Environments with working URLs

3. **Kubernetes must have:**
   - Resource requests and limits
   - Liveness and readiness probes
   - At least 2 replicas in staging/prod
   - Image pull secret for GitLab Registry

4. **Bonus challenges:**
   - Add Trivy container scanning
   - Implement canary deployment
   - Add `helm upgrade --install` instead of raw `kubectl`
   - Add Slack notification on production deploy

---

## Reference Cheat Sheet <a name="cheatsheet"></a>

### GitLab CI Quick Reference

```yaml
# Full job template
job-name:
  stage: deploy
  image: alpine:3.19
  services: []
  variables:
    MY_VAR: "value"
  before_script:
    - echo "Setup"
  script:
    - echo "Main work"
  after_script:
    - echo "Cleanup"
  artifacts:
    paths: [dist/]
    expire_in: 1 day
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths: [node_modules/]
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  needs: ["build"]        # DAG: don't wait for other stages
  allow_failure: false
  retry: 2
  timeout: 10 minutes
  tags: [kubernetes]      # Runner selector
```

### kubectl Quick Reference

```bash
# CRUD
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl get all -n <ns>
kubectl describe <resource> <name> -n <ns>

# Deployments
kubectl set image deployment/<name> <container>=<image> -n <ns>
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns>
kubectl scale deployment/<name> --replicas=3 -n <ns>

# Logs & Debug
kubectl logs -f <pod> -n <ns>
kubectl exec -it <pod> -n <ns> -- sh
kubectl port-forward svc/<name> 8080:80 -n <ns>
kubectl top pods -n <ns>

# Secrets
kubectl create secret generic <name> --from-literal=key=value -n <ns>
kubectl get secret <name> -n <ns> -o jsonpath='{.data.key}' | base64 -d
```

### Docker Quick Reference

```bash
# Build
docker build -t image:tag .
docker build --no-cache --build-arg FOO=bar -t image:tag .

# Registry
docker login registry.gitlab.com
docker push image:tag
docker pull image:tag

# Run
docker run -d -p 8080:8080 --env-file .env image:tag
docker exec -it <container> sh
docker logs -f <container>

# Cleanup
docker system prune -af
```

---

*Workshop materials prepared for the GitLab × Kubernetes Deployment Workshop.*  
*Keep this document handy — every command here is battle-tested.*
