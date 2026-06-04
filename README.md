# k8s-manifests

[![Deploy to EKS](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml/badge.svg)](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml)

This repository contains the Kubernetes manifests and CI/CD workflow that deploy the **beginner349-app** (a Spring Boot service) to an Amazon EKS cluster running **EKS Auto Mode** with the **AWS Load Balancer Controller** for ingress. Beyond the app itself, it provisions an **observability and secrets stack**: the **External Secrets Operator (ESO)** syncs credentials from **AWS Secrets Manager**, and **Grafana Alloy** forwards application telemetry (OTLP) to **Grafana Cloud**.

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

## Observability & Secrets Management

Telemetry from the Spring Boot app is shipped to Grafana Cloud, with the Grafana Cloud API token sourced securely from AWS Secrets Manager — no static secrets in git. These components are installed via Helm by the CI workflow, on top of the kustomize app deploy.

**Flow:** `AWS Secrets Manager` → **ESO** (IRSA auth) → k8s `Secret` → **Grafana Alloy** → **Grafana Cloud (OTLP)**. The app sends OTLP to Alloy on ports `4317` (gRPC) / `4318` (HTTP).

| Manifest                      | Purpose |
|-------------------------------|-----------------------------------------------------------------------------|
| `monitoring-namespace.yaml`   | Creates the `monitoring` and `external-secrets` namespaces. |
| `external-secrets-sa.yaml`    | ServiceAccount for ESO, IRSA-annotated (`<eso-irsa-role-arn>`). |
| `cluster-secret-store.yaml`   | `ClusterSecretStore` pointing ESO at AWS Secrets Manager via the SA. |
| `external-secret.yaml`        | Syncs the Grafana token into the `alloy-grafana-credentials` Secret. |
| `alloy-values.yaml`           | Helm values for Grafana Alloy (OTLP receiver → Grafana Cloud exporter). |

**Helm charts:**
- `external-secrets/external-secrets` (ESO operator + CRDs) — namespace `external-secrets`
- `grafana/alloy` — namespace `monitoring`

> Real values (IRSA role ARN, AWS account, Grafana instance ID / OTLP endpoint, and the Secrets Manager key `dev/grafana-cloud/api-token`) live in the manifests listed above.

## Deployment Instructions

### 1. Local Deployment (Kustomize)

The `kustomization.yaml` file tracks all resources in the correct order. To deploy the full stack to your cluster, run:

```bash
kubectl apply -k base
```

### 2. CI/CD via GitHub Actions

A manual deployment workflow (`workflow_dispatch`) runs the full stack, in order:

1. **AWS auth** — OIDC (`id-token: write`), then `aws eks update-kubeconfig`.
2. **Deploy app** — `kustomize edit set image` (ECR tag) + `kubectl apply -k base`.
3. **Helm setup** — install Helm, add `external-secrets` and `grafana` repos.
4. **Namespaces + ESO SA** — apply `monitoring-namespace.yaml` and `external-secrets-sa.yaml`.
5. **Install ESO** — `helm upgrade --install` with `installCRDs=true`, reusing `external-secrets-sa`.
6. **ESO resources** — apply `cluster-secret-store.yaml` + `external-secret.yaml`, then `kubectl wait` for the ExternalSecret `Ready` condition (guarantees the Secret exists before Alloy starts).
7. **Install Grafana Alloy** — `helm upgrade --install grafana/alloy -f alloy-values.yaml`.

## Configuration Details

| Feature              | Setting                          |
|----------------------|----------------------------------|
| App namespace        | `beginner349-dev`                |
| Replicas             | 3                                |
| ALB Scheme           | `internet-facing`                |
| Target Type          | `ip`                             |
| AWS Region           | `ap-southeast-1`                 |
| Monitoring namespace | `monitoring`                     |
| ESO namespace        | `external-secrets`               |
| OTLP ports           | `4317` (gRPC) / `4318` (HTTP)    |
| Telemetry backend    | Grafana Cloud (OTLP)             |

## TODO
- [x] Setting Up OpenID Connect (OIDC) in AWS for GitHub Actions
- [x] add Grafana Alloy + ESO charts for Grafana Cloud telemetry on EKS
- [ ] Implement Kustomize overlays for environment-specific configurations
- [ ] Implement TLS/SSL for the ingress with a custom domain using ExternalDNS and Route 53
- [ ] Install and configure ArgoCD for GitOps in EKS
