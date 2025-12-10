# EKS to ECS Migration: Side-by-Side Comparison

## What Changed?

We moved from Kubernetes (EKS) to AWS's simpler container service (ECS) to reduce complexity and operational overhead.

---

## Before & After Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart LR
    subgraph Before["BEFORE: Kubernetes (EKS)"]
        direction TB
        B1[Complex orchestration]
        B2[Managed EC2 servers]
        B3[Many K8s components]
        B4[Self-managed Redis]
    end

    subgraph After["AFTER: Containers (ECS)"]
        direction TB
        A1[Simple task definitions]
        A2[Serverless Fargate]
        A3[AWS-native services]
        A4[Managed ElastiCache]
    end

    Before -->|Simplified| After
```

---

## Architecture Comparison

### EKS (Before)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart TB
    subgraph User["Your App"]
        APP[Application]
    end

    subgraph K8s["Kubernetes Cluster"]
        ING[Ingress Controller]
        SVC[K8s Services]

        subgraph Nodes["EC2 Worker Nodes"]
            N1[Node 1]
            N2[Node 2]
            N3[Node 3]
        end

        PODS[Application Pods]
        REDIS_K8S[Redis StatefulSet<br/>Self-managed]
    end

    APP --> ING --> SVC --> PODS
    PODS --> REDIS_K8S
    Nodes --> PODS
```

### ECS (After)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart TB
    subgraph User["Your App"]
        APP[Application]
    end

    subgraph AWS["AWS Managed"]
        GA[Global Accelerator]
        WAF[Web Firewall]
        ALB[Load Balancer]

        subgraph ECS["ECS Fargate"]
            T1[Task 1]
            T2[Task 2]
            T3[Task 3]
        end

        REDIS_AWS[ElastiCache Redis<br/>AWS-managed]
    end

    APP --> GA --> WAF --> ALB --> ECS
    ECS --> REDIS_AWS
```

---

## What Got Simpler

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart TB
    subgraph Removed["Things We No Longer Manage"]
        R1[Kubernetes API Server]
        R2[Worker Node EC2s]
        R3[Node Auto-scaling]
        R4[Ingress Controller]
        R5[CoreDNS]
        R6[VPC CNI Plugin]
        R7[Redis StatefulSet]
        R8[Helm Charts]
    end

    subgraph Added["What We Use Instead"]
        A1[ECS Cluster - managed]
        A2[Fargate - serverless]
        A3[Auto-scaling - native]
        A4[ALB - native]
        A5[Route53 - native]
        A6[awsvpc - simple]
        A7[ElastiCache - managed]
        A8[Task Definitions - simple]
    end

    R1 --> A1
    R2 --> A2
    R3 --> A3
    R4 --> A4
    R5 --> A5
    R6 --> A6
    R7 --> A7
    R8 --> A8
```

---

## Deployment Comparison

### EKS Deployment (Complex)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart LR
    subgraph Steps["7 Steps"]
        S1[Build image]
        S2[Push to ECR]
        S3[Update Helm values]
        S4[Helm upgrade]
        S5[kubectl apply]
        S6[Wait for rollout]
        S7[Verify pods]
    end

    S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7
```

### ECS Deployment (Simple)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart LR
    subgraph Steps["4 Steps"]
        S1[Build image]
        S2[Push to ECR]
        S3[Update service]
        S4[Auto rolling update]
    end

    S1 --> S2 --> S3 --> S4
```

---

## Service Mapping

What became what:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart LR
    subgraph EKS["EKS Components"]
        E1[Pod]
        E2[Deployment]
        E3[Service]
        E4[Ingress]
        E5[StatefulSet]
        E6[ConfigMap]
        E7[Secret]
    end

    subgraph ECS["ECS Equivalents"]
        C1[Task]
        C2[Service]
        C3[Target Group]
        C4[ALB Rules]
        C5[ElastiCache]
        C6[Env Variables]
        C7[Secrets Manager]
    end

    E1 --> C1
    E2 --> C2
    E3 --> C3
    E4 --> C4
    E5 --> C5
    E6 --> C6
    E7 --> C7
```

---

## Cost Model Change

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart TB
    subgraph EKS_Cost["EKS Costs"]
        EC1[EKS Control Plane<br/>$73/month]
        EC2[EC2 Worker Nodes<br/>Always running]
        EC3[Node over-provisioning<br/>Wasted capacity]
    end

    subgraph ECS_Cost["ECS Costs"]
        CC1[No cluster fee<br/>$0/month]
        CC2[Fargate Tasks<br/>Pay per second]
        CC3[Right-sized tasks<br/>No waste]
    end
```

---

## Reliability Comparison

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart TB
    subgraph EKS_HA["EKS High Availability"]
        EH1[Control plane - AWS managed]
        EH2[Worker nodes - You manage]
        EH3[Node failures - You handle]
        EH4[Pod scheduling - Complex]
    end

    subgraph ECS_HA["ECS High Availability"]
        CH1[Cluster - AWS managed]
        CH2[Tasks - AWS manages]
        CH3[Failures - Auto-replaced]
        CH4[Placement - Automatic]
    end
```

---

## Quick Reference

| What | EKS Term | ECS Term |
|------|----------|----------|
| Running container | Pod | Task |
| Keep N running | Deployment | Service |
| Traffic routing | Ingress | ALB Listener Rules |
| Internal networking | K8s Service | Target Group |
| Configuration | ConfigMap | Environment Variables |
| Secrets | K8s Secret | Secrets Manager |
| Logs | kubectl logs | CloudWatch Logs |
| Scaling | HPA | Service Auto Scaling |

---

## Summary

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#1e3a5f', 'primaryTextColor': '#fff', 'primaryBorderColor': '#38bdf8', 'lineColor': '#38bdf8', 'secondaryColor': '#4c1d95', 'tertiaryColor': '#064e3b', 'textColor': '#fff', 'clusterBkg': '#0f172a', 'clusterBorder': '#38bdf8'}}}%%
flowchart LR
    subgraph Before["Before: EKS"]
        B[Complex<br/>Many moving parts<br/>K8s expertise needed]
    end

    subgraph After["After: ECS"]
        A[Simple<br/>AWS-native<br/>Just works]
    end

    Before -->|Migration| After
```

**Bottom line:** We went from managing a complex Kubernetes cluster to using AWS-native services that "just work."
