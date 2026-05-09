# k8s-manifests

[![Deploy to EKS](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml/badge.svg)](https://github.com/beginner349/k8s-manifests/actions/workflows/kubernetes-deployment.yml)

This repository contains the Kubernetes configuration files and CI/CD workflow required to deploy the **beginner349-app** (a Spring Boot application) to an Amazon EKS cluster. The deployment utilizes **EKS Auto Mode** and the **AWS Load Balancer Controller** for automated ingress management.

## Project Overview

The configuration sets up a complete environment including namespace isolation, application scaling, and external access via an Application Load Balancer (ALB).

## Resources & Architecture

The deployment is composed of the following components, orchestrated via **Kustomize**:

* **Namespace (`01-namespace.yaml`)**: Creates the `beginner349-dev` environment to isolate project resources.
* **Deployment (`02-deployment.yml`)**: Manages 3 replicas of the `spring-boot-app`. It is configured for the `ap-southeast-1` region and exposes port `8080`.
* **Service (`03-service.yaml`)**: An internal service named `service-beginner349` that routes traffic on port `80` to the application's port `8080`.
* **Ingress Configuration**:
    - **IngressClassParams (`04-ingressclassparams.yaml`)**: Configures the ALB to be `internet-facing`.
    - **IngressClass (`05-ingressclass.yaml`)**: Sets `alb` as the default ingress class for the cluster, powered by the `eks.amazonaws.com/alb` controller.
    - **Ingress (`06-ingress.yaml`)**: Defines the routing rules, directing all traffic (`/*`) to the backend service.
* **Service account (`07-serviceaccount.yaml`)**: 


## Deployment Instructions

### 1. Local Deployment (Kustomize)

The `kustomization.yaml` file tracks all resources in the correct order [5]. To deploy the full stack to your cluster, run:

```bash
kubectl apply -k .
```

### 2. CI/CD via GitHub Actions

A manual deployment workflow is available for automated updates via GitHub Actions
* Authentication: The workflow uses OIDC (OpenID Connect) to securely authenticate with AWS, requiring id-token: write permissions
* Trigger: This is a manual workflow (workflow_dispatch), allowing you to trigger deployments directly from the Actions tab in GitHub

## Configuration Details

| Feature     | Setting           |
|-------------|-------------------|
| Namespace   | `beginner349-dev` |
| Replicas    | 3                 |
| ALB Scheme  | `internet-facing` |
| Target Type | `ip`              |
| AWS Region  | `ap-southeast-1`  |

## TODO
- [x] Setting Up OpenID Connect (OIDC) in AWS for GitHub Actions
- [ ] Implement Kustomize overlays for environment-specific configurations
- [ ] Implement TLS/SSL for the ingress with a custom domain using ExternalDNS and Route 53
- [ ] Install and configure ArgoCD for GitOps in EKS
- [ ] Install and configure Prometheus and Grafana in EKS
