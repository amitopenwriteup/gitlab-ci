If Kubernetes is already running on your Linux VM and your `kubeconfig` is available, the simplest approach is to use the Terraform Helm provider to deploy Prometheus and Grafana.

## Directory Structure

```text
terraform-monitoring/
├── providers.tf
├── namespace.tf
├── prometheus.tf
├── grafana.tf
├── variables.tf
└── terraform.tfvars
```

---

## providers.tf

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.32"
    }

    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.13"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}
```

---

## namespace.tf

```hcl
resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}
```

---

## prometheus.tf

Deploy Prometheus using the community Helm chart.

```hcl
resource "helm_release" "prometheus" {
  name       = "prometheus"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name

  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "prometheus"

  create_namespace = false

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

      alertmanager = {
        enabled = true
      }

      pushgateway = {
        enabled = true
      }
    })
  ]
}
```

---

## grafana.tf

Deploy Grafana.

```hcl
resource "helm_release" "grafana" {
  name       = "grafana"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name

  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"

  create_namespace = false

  values = [
    yamlencode({

      adminUser     = var.grafana_admin_user
      adminPassword = var.grafana_admin_password

      service = {
        type = "NodePort"
      }

      persistence = {
        enabled = true
        size    = "10Gi"
      }

      datasources = {
        "datasources.yaml" = {
          apiVersion = 1

          datasources = [
            {
              name      = "Prometheus"
              type      = "prometheus"
              access    = "proxy"
              isDefault = true
              url       = "http://prometheus-server"
            }
          ]
        }
      }
    })
  ]

  depends_on = [
    helm_release.prometheus
  ]
}
```

---

## variables.tf

```hcl
variable "grafana_admin_user" {
  type        = string
  default     = "admin"
}

variable "grafana_admin_password" {
  type      = string
  sensitive = true
}
```

---

## terraform.tfvars

```hcl
grafana_admin_password = "StrongPassword123!"
```

---

## Outputs

```hcl
output "grafana_namespace" {
  value = kubernetes_namespace.monitoring.metadata[0].name
}
```

---

## Deploy

```bash
terraform init

terraform plan

terraform apply -auto-approve
```

---

## Verify

```bash
kubectl get pods -n monitoring

kubectl get svc -n monitoring
```

Expected services:

```text
NAME                TYPE       PORT
prometheus-server   NodePort   80
grafana             NodePort   80
```

Get Grafana admin password:

```bash
kubectl get secret grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" | base64 -d
```

Access:

```text
http://<VM-IP>:<Grafana-NodePort>
http://<VM-IP>:<Prometheus-NodePort>
```

For production, consider using the Prometheus Operator stack via the Helm chart `kube-prometheus-stack`, which deploys Prometheus, Grafana, Alertmanager, node exporters, and Kubernetes monitoring components together.
