# End-to-End Lab: Vault + Terraform + Kubernetes + GitLab CI

## Architecture

```text
Ubuntu VM
   |
   +-- Kubernetes Cluster
   |
   +-- HashiCorp Vault
          |
          +-- kubeconfig
          +-- Grafana Password
          |
          v
GitLab CI
          |
          v
Terraform
          |
          +-- Kubernetes Provider
          +-- Helm Provider
          |
          v
Prometheus + Grafana
```

---

# Step 1: Install Vault on Ubuntu

Update packages:

```bash
sudo apt update
sudo apt install -y gpg wget curl unzip
```

Add HashiCorp repository:

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg >/dev/null
```

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
```

Install Vault:

```bash
sudo apt update
sudo apt install -y vault
```

Verify:

```bash
vault version
```

---

# Step 2: Configure Vault

Create configuration:

```bash
sudo mkdir -p /opt/vault/data
```

```bash
sudo tee -a /etc/vault.d/vault.hcl > /dev/null <<'EOF'
ui = true

storage "raft" {
  path = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}
EOF

```

Enable service:

```bash
sudo systemctl enable vault
sudo systemctl restart vault
sudo systemctl status vault
```

---

# Step 3: Initialize Vault

Set Vault address:

```bash
export VAULT_ADDR=http://<VM-IP>:8200
```

Initialize:

```bash
vault operator init
```

Example output:

```text
Unseal Key 1: xxxxx
Initial Root Token: hvs.xxxxx
```

Save:

* Unseal Key
* Root Token

securely.

---

# Step 4: Unseal Vault

```bash
vault operator unseal
```

Paste key.

Login:

```bash
vault login
```

Paste root token.

---

# Step 5: Enable KV Secret Engine

```bash
vault secrets enable -path=secret kv-v2
```

Verify:

```bash
vault secrets list
```

---

# Step 6: Store kubeconfig in Vault

Locate kubeconfig:

```bash
cat ~/.kube/config
```

Store:

```bash
vault kv put secret/k8s/cluster \
kubeconfig="$(cat ~/.kube/config)"
```

Verify:

```bash
vault kv get secret/k8s/cluster
```

---

# Step 7: Store Grafana Password

```bash
vault kv put secret/monitoring/grafana \
admin_password="StrongPassword123!"
```

Verify:

```bash
vault kv get secret/monitoring/grafana
```

---

# Step 8: Terraform Project Structure

```text
terraform-monitoring/
├── providers.tf
├── namespace.tf
├── vault.tf
├── prometheus.tf
├── grafana.tf
├── variables.tf
├── outputs.tf
└── .gitlab-ci.yml
```

---

# providers.tf

```hcl
terraform {
  required_version = ">=1.5"

  required_providers {

    kubernetes = {
      source = "hashicorp/kubernetes"
      version = "~>2.32"
    }

    helm = {
      source = "hashicorp/helm"
      version = "~>2.13"
    }

    vault = {
      source = "hashicorp/vault"
      version = "~>4.3"
    }

    local = {
      source = "hashicorp/local"
    }
  }
}

provider "vault" {
  address = var.vault_addr
  token   = var.vault_token
}
```

---

# vault.tf

Read kubeconfig:

```hcl
data "vault_kv_secret_v2" "cluster" {
  mount = "secret"
  name  = "k8s/cluster"
}
```

Read Grafana secret:

```hcl
data "vault_kv_secret_v2" "grafana" {
  mount = "secret"
  name  = "monitoring/grafana"
}
```

Create local kubeconfig:

```hcl
resource "local_file" "kubeconfig" {

  filename = "${path.module}/kubeconfig"

  content = data.vault_kv_secret_v2.cluster.data["kubeconfig"]
}
```

---

# Kubernetes Provider

```hcl
provider "kubernetes" {
  config_path = local_file.kubeconfig.filename
}

provider "helm" {
  kubernetes {
    config_path = local_file.kubeconfig.filename
  }
}
```

---

# namespace.tf

```hcl
resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}
```

---

# prometheus.tf

```hcl
resource "helm_release" "prometheus" {

  name       = "prometheus"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name

  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "prometheus"

  values = [
    yamlencode({
      server = {
        service = {
          type = "NodePort"
        }

        persistentVolume = {
          enabled = true
          size    = "20Gi"
        }
      }
    })
  ]
}
```

---

# grafana.tf

```hcl
resource "helm_release" "grafana" {

  name      = "grafana"
  namespace = kubernetes_namespace.monitoring.metadata[0].name

  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"

  values = [
    yamlencode({

      adminUser     = "admin"
      adminPassword = data.vault_kv_secret_v2.grafana.data["admin_password"]

      service = {
        type = "NodePort"
      }

      persistence = {
        enabled = true
        size    = "10Gi"
      }

    })
  ]

  depends_on = [
    helm_release.prometheus
  ]
}
```

---

# variables.tf

```hcl
variable "vault_addr" {}
variable "vault_token" {
  sensitive = true
}
```

---

# outputs.tf

```hcl
output "namespace" {
  value = kubernetes_namespace.monitoring.metadata[0].name
}
```

---

# GitLab Variables

Configure:

```text
VAULT_ADDR
VAULT_TOKEN
```

Protected and masked.

---

# .gitlab-ci.yml

```yaml
stages:
  - validate
  - plan
  - apply

variables:
  TF_IN_AUTOMATION: "true"
  TF_VAR_vault_addr: $VAULT_ADDR
  TF_VAR_vault_token: $VAULT_TOKEN
default:
  image:
    name: hashicorp/terraform:1.8
    entrypoint: [""]

before_script:
  - terraform version

terraform_validate:
  stage: validate

  script:
    - terraform init
    - terraform validate

terraform_plan:
  stage: plan

  script:
    - terraform init
    - terraform plan -out=tfplan

  artifacts:
    paths:
      - tfplan

terraform_apply:
  stage: apply
  when: manual

  script:
    - terraform init
    - terraform apply -auto-approve tfplan
```

---

# Deploy

```bash
terraform init
```

```bash
terraform plan
```

```bash
terraform apply
```

---

# Verify

```bash
kubectl get pods -n monitoring
```

```bash
kubectl get svc -n monitoring
```

---

# Access Grafana

Find NodePort:

```bash
kubectl get svc -n monitoring
```

Example:

```text
grafana  NodePort  3000:32000/TCP
```

Open:

```text
http://<VM-IP>:32000
```

Login:

```text
admin
```

Password comes automatically from Vault:

```bash
vault kv get secret/monitoring/grafana
```

---

# Production Improvements

1. Replace root token with AppRole or JWT authentication.
2. Enable TLS on Vault.
3. Use GitLab OIDC/JWT login to Vault.
4. Use Vault Agent Injector for Kubernetes workloads.
5. Replace separate charts with kube-prometheus-stack.
6. Enable Vault Raft snapshots and backups.
