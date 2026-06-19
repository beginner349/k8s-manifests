# k8s-manifests

[![Deploy to EKS](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml/badge.svg)](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml)

This repository contains the Kubernetes manifests and GitOps configuration that deploy the **beginner349-app** (a Spring Boot service) to an Amazon EKS cluster running **EKS Auto Mode** with the **AWS Load Balancer Controller** for ingress. Deployment is driven by **ArgoCD** using the **app-of-apps** pattern: a CI workflow bootstraps ArgoCD, and ArgoCD then reconciles the whole stack from this repo — the app, the **External Secrets Operator (ESO)** (syncing credentials from **AWS Secrets Manager**), and **Grafana Alloy** (forwarding OTLP telemetry to **Grafana Cloud**).

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

`root-app.yaml` watches `argocd/applications/`, which holds one `Application` per stack component; the `beginner349 AppProject` constrains allowed repos/namespaces.

| Path                                          | Kind        | Purpose                                                                  |
|-----------------------------------------------|-------------|--------------------------------------------------------------------------|
| `argocd/root-app.yaml`                        | Application | Root app: watches `argocd/applications/` and auto-syncs children         |
| `argocd/projects/project.yaml`                | AppProject  | `beginner349` project; whitelists source repos + destination namespaces  |
| `argocd/bootstrap/values.yaml`                | Helm values | ArgoCD install values: `server.insecure`, public NLB for the server      |
| `argocd/applications/external-secrets.yaml`   | Application | ESO Helm chart (wave 0); SA with IRSA annotation                         |
| `argocd/applications/spring-boot-app.yaml`    | Application | Kustomize `base/` (wave 0)                                               |
| `argocd/applications/secret-resources.yaml`   | Application | `secrets/` ClusterSecretStore + ExternalSecret (wave 1)                  |
| `argocd/applications/grafana-alloy.yaml`      | Application |  Alloy chart + `alloy-values.yaml` (wave 2)                              |

wave 0 = ESO operator + app; wave 1 = secret resources (ClusterSecretStore/ExternalSecret, needs ESO CRDs first); wave 2 = Alloy (needs the synced `alloy-grafana-credentials` Secret). All child apps use `automated` sync with `prune` + `selfHeal`.

## Observability & Secrets Management

Telemetry from the Spring Boot app is shipped to Grafana Cloud, with the Grafana Cloud API token sourced securely from AWS Secrets Manager — no static secrets in git. ESO and Alloy are deployed by ArgoCD (the `external-secrets`, `secret-resources`, and `grafana-alloy` Applications), not the CI workflow.

**Flow:** `AWS Secrets Manager` → **ESO** (IRSA auth) → k8s `Secret` → **Grafana Alloy** → **Grafana Cloud (OTLP)**. The app sends OTLP to Alloy on ports `4317` (gRPC) / `4318` (HTTP).

| Manifest                             | Purpose                                                                 |
|--------------------------------------|-------------------------------------------------------------------------|
| `secrets/cluster-secret-store.yaml`  | `ClusterSecretStore` pointing ESO at AWS Secrets Manager via the SA.    |
| `secrets/external-secret.yaml`       | Syncs the Grafana token into the `alloy-grafana-credentials` Secret.    |
| `alloy-values.yaml`                  | Helm values for Grafana Alloy (OTLP receiver → Grafana Cloud exporter). |

 The ESO `ServiceAccount` (with its IRSA annotation) is now declared inline in `argocd/applications/external-secrets.yaml` via the chart's `serviceAccount` values, and the `monitoring` / `external-secrets` namespaces are created by ArgoCD (`CreateNamespace=true`) — the standalone `external-secrets-sa.yaml` and `monitoring-namespace.yaml` manifests were removed.

**Helm charts:**
- `external-secrets/external-secrets` (ESO operator + CRDs) — namespace `external-secrets`
- `grafana/alloy` — namespace `monitoring`

> Real values (IRSA role ARN, AWS account, Grafana instance ID / OTLP endpoint, and the Secrets Manager key `dev/grafana-cloud/api-token`) live in the manifests listed above.

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
3. **Install ArgoCD** — `helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --version 9.5.19 -f argocd/bootstrap/values.yaml --wait`.
4. **Hand off to GitOps** — `kubectl apply -f argocd/projects/project.yaml` then `-f argocd/root-app.yaml`. From here ArgoCD reconciles the app, ESO, secret resources, and Alloy by sync-wave.
5. **Print admin password** — read from the `argocd-initial-admin-secret` Secret.

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
| OTLP ports                | `4317` (gRPC) / `4318` (HTTP)    |
| Telemetry backend         | Grafana Cloud (OTLP)             |
| ArgoCD namespace          | argocd                           |
| ArgoCD server exposure    | public NLB (`internet-facing`)   |

## TODO
- [x] Setting Up OpenID Connect (OIDC) in AWS for GitHub Actions
- [x] add Grafana Alloy + ESO charts for Grafana Cloud telemetry on EKS
- [x] Install and configure ArgoCD for GitOps in EKS
- [ ] Deploy Keycloak Operator on EKS backed by CloudNativePG
- [ ] Implement TLS/SSL for the spring boot ingress with a custom domain using ExternalDNS and Route 53
- [ ] Implement Kustomize overlays for environment-specific configurations
