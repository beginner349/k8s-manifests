# k8s-manifests

[![Deploy to EKS](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml/badge.svg)](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml)

This repository contains the Kubernetes manifests and GitOps configuration that deploy the **beginner349-app** (a Spring Boot service) to an Amazon EKS cluster running **EKS Auto Mode**, whose built-in `eks.amazonaws.com/alb` controller provisions ALBs for ingress. Deployment is driven by **ArgoCD** using the **app-of-apps** pattern: a CI workflow bootstraps ArgoCD, and ArgoCD then reconciles the whole stack from this repo — the app, the **External Secrets Operator (ESO)** (syncing credentials from **AWS Secrets Manager**), **Grafana Alloy** (forwarding OTLP telemetry to **Grafana Cloud**), a self-hosted **Keycloak** (via the Keycloak Operator) backed by an **Amazon Aurora PostgreSQL** cluster, and **external-dns** (managing **Route 53** records).

## Project Overview

The configuration sets up a complete environment including namespace isolation, application scaling, and external access via an Application Load Balancer (ALB).

## Resources & Architecture

The application deployment is composed of the following components under `base/`, orchestrated via **Kustomize**:

* **Namespace (`base/01-namespace.yaml`)**: Creates the `beginner349-dev` environment to isolate project resources.
* **Deployment (`base/02-deployment.yaml`)**: Manages 3 replicas of the `spring-boot-app`. It is configured for the `ap-southeast-1` region and exposes port `8080`.
* **Service (`base/03-service.yaml`)**: An internal service named `service-beginner349` that routes traffic on port `80` to the application's port `8080`.
* **Ingress Configuration**:
    - **IngressClassParams (`base/04-ingressclassparams.yaml`)**: Configures the ALB to be `internet-facing`.
    - **IngressClass (`base/05-ingressclass.yaml`)**: Sets `alb` as the default ingress class for the cluster, powered by the `eks.amazonaws.com/alb` controller.
    - **Ingress (`base/06-ingress.yaml`)**: Defines the routing rules, directing all traffic (`/*`) to the backend service.
* **ServiceAccount (`base/07-serviceaccount.yaml`)**: IRSA-annotated service account assumed by the application pods.

## GitOps with ArgoCD (app-of-apps)

`argocd/root-app.yaml.` watches `argocd/applications/`, which holds one `Application` per stack component; the `beginner349 AppProject` constrains allowed repos/namespaces.

| Path                                          | Kind        | Purpose                                                                  |
|-----------------------------------------------|-------------|--------------------------------------------------------------------------|
| `argocd/root-app.yaml`                        | Application | Root app: watches `argocd/applications/` and auto-syncs children         |
| `argocd/projects/project.yaml`                | AppProject  | `beginner349` project; whitelists source repos + destination namespaces  |
| `argocd/bootstrap/values.yaml`                | Helm values | ArgoCD install values: `server.insecure`, public NLB for the server      |
| `argocd/applications/external-secrets.yaml`   | Application | ESO Helm chart (wave 0); SA with IRSA annotation                         |
| `argocd/applications/spring-boot-app.yaml`    | Application | Kustomize `base/` (wave 0)                                               |
| `argocd/applications/keycloak-operator.yaml`  | Application | Keycloak Operator (remote Kustomize base), ns `keycloak` (wave 0)        |
| `argocd/applications/external-dns.yaml`       | Application | external-dns Helm chart (Route 53), ns `external-dns` (wave 0)           |
| `argocd/applications/secret-resources.yaml`   | Application | `secrets/` ClusterSecretStore + ExternalSecret (wave 1)                  |
| `argocd/applications/keycloak.yaml`           | Application | `keycloak/` (`keycloak-stack`): DB ExternalSecret + RDS CA bundle + Keycloak CR + realm import + Ingress |
| `argocd/applications/grafana-alloy.yaml`      | Application | Alloy chart + `alloy-values.yaml` (wave 2)                               |

wave 0 = operators + app base (ESO, external-dns, Keycloak Operator, the Spring Boot app); wave 1 = resources that depend on those operators/CRDs (the ESO `secret-resources`, and the Keycloak stack — DB ExternalSecret + CA bundle, then the Keycloak CR, then the realm import + Ingress); wave 2 = Alloy (needs the synced `alloy-grafana-credentials` Secret). All child apps use `automated` sync with `prune` + `selfHeal`.

## Observability & Secrets Management

Telemetry from the Spring Boot app is shipped to Grafana Cloud, with the Grafana Cloud API token sourced securely from AWS Secrets Manager — no static secrets in git. ESO and Alloy are deployed by ArgoCD (the `external-secrets`, `secret-resources`, and `grafana-alloy` Applications), not the CI workflow.

**Flow:** `AWS Secrets Manager` → **ESO** (IRSA auth) → k8s `Secret` → **Grafana Alloy** → **Grafana Cloud (OTLP)**. The app sends OTLP to Alloy on ports `4317` (gRPC) / `4318` (HTTP).

| Manifest                             | Purpose                                                                 |
|--------------------------------------|-------------------------------------------------------------------------|
| `secrets/cluster-secret-store.yaml`  | `ClusterSecretStore` pointing ESO at AWS Secrets Manager via the SA.    |
| `secrets/external-secret.yaml`       | Syncs the Grafana token into the `alloy-grafana-credentials` Secret.    |
| `alloy-values.yaml`                  | Helm values for Grafana Alloy (OTLP receiver → Grafana Cloud exporter). |

The ESO `ServiceAccount` (with its IRSA annotation) is declared inline in `argocd/applications/external-secrets.yaml` via the chart's `serviceAccount` values, and the `monitoring` / `external-secrets` namespaces are created by ArgoCD (`CreateNamespace=true`).

**Helm charts:**
- `external-secrets/external-secrets` (ESO operator + CRDs) — namespace `external-secrets`
- `grafana/alloy` — namespace `monitoring`

> Real values (IRSA role ARN, AWS account, Grafana instance ID / OTLP endpoint, and the Secrets Manager key `dev/grafana-cloud/api-token`) live in the manifests listed above.

## Database & Keycloak (Aurora PostgreSQL)

A self-hosted **Keycloak** runs in the `keycloak` namespace, deployed via the **Keycloak Operator** CR, and is backed by an external **Amazon Aurora PostgreSQL** cluster (RDS). The CR connects over the standard JDBC URL (`jdbc:postgresql://keycloak-db.cluster-….ap-southeast-1.rds.amazonaws.com:5432/keycloak`) with `db-tls-mode: verify-server`, validating the server certificate against the **Amazon RDS CA bundle** mounted from the `rds-ca-bundle` ConfigMap. Database credentials are synced by **ESO** from the RDS-managed master secret in AWS Secrets Manager into the `keycloak-db-credentials` Secret — no static DB passwords in git.

On first sync the realm is bootstrapped declaratively via a **`KeycloakRealmImport`** (a sample realm with a public OIDC client and a test user). Keycloak is exposed via an ALB `Ingress` at `https://auth.beginner349.com` (ACM cert, HTTP→HTTPS redirect, health check on port `9000` `/health/ready`); the DNS record is managed by **external-dns** against the `beginner349.com` hosted zone.

| Manifest                          | Purpose                                                                           |
|-----------------------------------|-----------------------------------------------------------------------------------|
| `keycloak-operator/`              | Kustomize remote base installing the Keycloak Operator + CRDs (v26.6.3).          |
| `keycloak/db-external-secret.yaml`| `ExternalSecret` syncing the RDS master creds → `keycloak-db-credentials` Secret. |
| `keycloak/rds-ca-bundle.yaml`     | `ConfigMap` with the Amazon RDS CA bundle (used for `verify-server` TLS).         |
| `keycloak/keycloak-cr.yaml`       | Keycloak CR; Aurora JDBC URL, TLS `verify-server`, 3 instances.                   |
| `keycloak/realm-import.yaml`      | `KeycloakRealmImport`: realm + OIDC client + user.                                |
| `keycloak/ingress.yaml`           | ALB Ingress for `auth.beginner349.com` (ACM TLS, HTTP→HTTPS).                     |

## Deployment Instructions

### 1. Local Deployment (Kustomize)

ArgoCD normally syncs `base/` for you. To apply just the app manifests directly (e.g. for local testing), run:

```bash
kubectl apply -k base
```

The image tag is pinned in base/kustomization.yaml (images[].newTag).

### 2. CI/CD via GitHub Actions

The `Bootstrap ArgoCD` workflow (`workflow_dispatch`) only installs ArgoCD and hands the rest off to GitOps — it does **not** deploy the app or charts directly:

1. **AWS auth** — OIDC (`id-token: write`), assume `AWS_ROLE_ARN`.
2. **kubeconfig** — `aws eks update-kubeconfig --name my-eks-cluster`.
3. **Install Keycloak Operator CRDs** — `kubectl apply` the `keycloaks` + `keycloakrealmimports` CRDs (keycloak-k8s-resources `26.6.3`) so the operator's CRs are recognized on first sync.
4. **Install ArgoCD** — `helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --version 9.5.19 -f argocd/bootstrap/values.yaml --wait`.
5. **Hand off to GitOps** — `kubectl apply -f argocd/projects/project.yaml` then `-f argocd/root-app.yaml`. From here ArgoCD reconciles the app, ESO, secret resources, Keycloak, and Alloy by sync-wave.
6. **Print admin password** — read from the `argocd-initial-admin-secret` Secret.

The ArgoCD server is reachable over a public NLB; log in as `admin` with the printed password.

## Configuration Details

| Feature                   | Setting                          |
|---------------------------|----------------------------------|
| App namespace             | `beginner349-dev`                |
| Replicas                  | 3                                |
| ALB Scheme                | `internet-facing`                |
| Target Type               | `ip`                             |
| AWS Region                | `ap-southeast-1`                 |
| Monitoring namespace      | `monitoring`                     |
| ESO namespace             | `external-secrets`               |
| Keycloak namespace        | `keycloak`                       |
| external-dns namespace    | `external-dns`                   |
| Keycloak database         | Amazon Aurora PostgreSQL (RDS)   |
| DB TLS                    | `verify-server` (RDS CA bundle)  |
| Keycloak instances        | 3                                |
| Keycloak hostname         | `auth.beginner349.com`           |
| OTLP ports                | `4317` (gRPC) / `4318` (HTTP)    |
| Telemetry backend         | Grafana Cloud (OTLP)             |
| ArgoCD namespace          | argocd                           |
| ArgoCD server exposure    | public NLB (`internet-facing`)   |

## TODO
- [x] Setting Up OpenID Connect (OIDC) in AWS for GitHub Actions
- [x] add Grafana Alloy + ESO charts for Grafana Cloud telemetry on EKS
- [x] Install and configure ArgoCD for GitOps in EKS
- [x] Deploy Keycloak Operator on EKS
- [x] Migrate Keycloak database from CloudNativePG to Aurora Postgres
- [x] Automate Keycloak realm/client/user provisioning via `KeycloakRealmImport`
- [ ] Implement TLS/SSL for the spring boot ingress with a custom domain using ExternalDNS and Route 53 (external-dns + Route 53 + ACM are now in place for Keycloak; reuse for the app ingress)
- [ ] Implement Kustomize overlays for environment-specific configurations
