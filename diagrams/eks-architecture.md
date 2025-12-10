# Aldea EKS Architecture (Previous)

## What is This?

This document shows how Aldea's services **previously** ran using AWS EKS (Elastic Kubernetes Service). We migrated to ECS for simpler operations. This is kept for historical reference.

**Key Difference:** EKS uses Kubernetes (a complex container orchestration system) while ECS is AWS's simpler native container service.

---

## How Users Connected

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Users["Your Application"]
        APP[Your App]
    end

    subgraph Domains["Web Addresses"]
        API[api.aldea.ai<br/>Speech API]
        BACK[backend.aldea.ai<br/>Platform API]
        PLAT[platform.aldea.ai<br/>Web Dashboard]
    end

    subgraph K8s["Kubernetes Cluster"]
        ING[Ingress Controller<br/>Routes traffic]
        PODS[Application Pods]
    end

    APP --> API & BACK & PLAT --> ING --> PODS
```

---

## EKS Architecture Overview

A simplified view of the Kubernetes-based architecture:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Internet["The Internet"]
        USER[Your Application]
    end

    subgraph DNS["Domain Names"]
        DOMAINS[api.aldea.ai<br/>backend.aldea.ai<br/>platform.aldea.ai]
    end

    subgraph LB["Load Balancer"]
        ALB[Application Load Balancer<br/>Managed by K8s Ingress]
    end

    subgraph EKS["Kubernetes Cluster"]
        subgraph CP["Control Plane (AWS Managed)"]
            API[Kubernetes API<br/>Manages everything]
        end

        subgraph Workers["Worker Nodes (We Managed)"]
            CPU[CPU Nodes<br/>General workloads]
            GPU[GPU Nodes<br/>ML processing]
            RNODE[Redis Nodes<br/>Cache workloads]
        end

        subgraph Apps["Applications (Pods)"]
            WS[WebSocket Proxy]
            STT[Speech API]
            BE[Backend API]
            FE[Frontend Web]
        end
    end

    subgraph Backend["Backend Services"]
        REDIS[Redis Cluster<br/>Self-managed in K8s]
        DB[(Database)]
    end

    USER --> DOMAINS --> ALB --> Apps
    Apps --> REDIS
    BE --> DB
    Workers --> API
```

---

## Pods Distributed Across Availability Zones

In Kubernetes, applications run in "Pods" which are scheduled across worker nodes in different availability zones:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph ALB["Load Balancer"]
        LB[Distributes traffic<br/>across all pods]
    end

    subgraph EKS["Kubernetes Cluster"]
        subgraph AZ_A["Zone A (us-west-2a)"]
            subgraph NodeA["Worker Node 1"]
                WS_A1[WS-Proxy Pod]
                BE_A1[Backend Pod]
            end
        end

        subgraph AZ_B["Zone B (us-west-2b)"]
            subgraph NodeB["Worker Node 2"]
                WS_B1[WS-Proxy Pod]
                STT_B1[STT-API Pod]
            end
        end

        subgraph AZ_C["Zone C (us-west-2c)"]
            subgraph NodeC["Worker Node 3"]
                FE_C1[Frontend Pod]
                BE_C1[Backend Pod]
            end
        end

        subgraph Redis["Redis Namespace"]
            R1[Redis Pod 1<br/>Zone A]
            R2[Redis Pod 2<br/>Zone B]
            R3[Redis Pod 3<br/>Zone C]
        end
    end

    LB --> NodeA & NodeB & NodeC
    WS_A1 & WS_B1 --> R1 & R2 & R3
```

---

## EKS Network Layout

How network traffic flows through subnets:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Internet
        IGW[Internet Gateway]
    end

    subgraph VPC["Virtual Private Cloud"]
        subgraph Public["Public Subnets (Load Balancer)"]
            PUB_A[Zone A<br/>Load Balancer]
            PUB_B[Zone B<br/>Load Balancer]
            NAT_A[NAT Gateway A]
            NAT_B[NAT Gateway B]
        end

        subgraph Private["Private Subnets (Worker Nodes)"]
            subgraph PrivA["Zone A"]
                NODE_A[Worker Node<br/>with Pods]
            end
            subgraph PrivB["Zone B"]
                NODE_B[Worker Node<br/>with Pods]
            end
            subgraph PrivC["Zone C"]
                NODE_C[Worker Node<br/>with Pods]
            end
        end
    end

    IGW --> PUB_A & PUB_B
    PUB_A --> NAT_A
    PUB_B --> NAT_B
    NODE_A --> NAT_A
    NODE_B --> NAT_B
    NODE_C --> NAT_A

    PUB_A & PUB_B --> Private
```

---

## Kubernetes Components Explained

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Simple["What You Need to Know"]
        direction TB
        S1[Pods = Your running application]
        S2[Nodes = Servers that run pods]
        S3[Services = How pods talk to each other]
        S4[Ingress = How traffic gets in]
    end

    subgraph Technical["Technical Details"]
        direction TB
        T1[ReplicaSet = Keeps N pods running]
        T2[Deployment = Manages updates]
        T3[StatefulSet = For databases/caches]
        T4[ConfigMap/Secret = Configuration]
    end

    Simple --> Technical
```

---

## How Services Communicated

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Ingress["Traffic Entry"]
        ALB[Load Balancer]
    end

    subgraph Services["Kubernetes Services"]
        SVC_WS[ws-proxy:8000]
        SVC_STT[stt-api:8800]
        SVC_BE[backend:8000]
        SVC_FE[frontend:3000]
        SVC_REDIS[redis:6379]
    end

    subgraph Pods["Running Applications"]
        WS[WS-Proxy<br/>2 replicas]
        STT[STT-API<br/>2 replicas]
        BE[Backend<br/>2 replicas]
        FE[Frontend<br/>2 replicas]
        REDIS[Redis<br/>3 replicas]
    end

    ALB --> SVC_WS & SVC_BE & SVC_FE
    SVC_WS --> WS
    SVC_STT --> STT
    SVC_BE --> BE
    SVC_FE --> FE
    SVC_REDIS --> REDIS

    WS --> SVC_STT & SVC_REDIS
```

---

## Why We Moved Away from EKS

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph EKS_Problems["EKS Challenges"]
        E1[Complex Kubernetes<br/>learning curve]
        E2[Node management<br/>& patching]
        E3[Many moving parts<br/>to configure]
        E4[Higher operational<br/>overhead]
    end

    subgraph ECS_Benefits["ECS Advantages"]
        C1[Simpler AWS-native<br/>service]
        C2[No servers to<br/>manage - Fargate]
        C3[Native AWS<br/>integrations]
        C4[Lower operational<br/>overhead]
    end

    EKS_Problems -->|Migrated to| ECS_Benefits
```

---

## Deployment Pipeline (EKS)

How code was deployed in the EKS era:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Dev["Developer"]
        CODE[Push code]
    end

    subgraph CI["Build"]
        BUILD[Build image]
        PUSH[Push to ECR]
    end

    subgraph K8s["Kubernetes"]
        HELM[Helm chart<br/>update]
        KUBECTL[kubectl apply]
        DEPLOY[Rolling update]
    end

    CODE --> BUILD --> PUSH --> HELM --> KUBECTL --> DEPLOY
```

---

## EKS vs ECS Comparison

| Aspect | EKS (Before) | ECS (Now) |
|--------|--------------|-----------|
| **Complexity** | High - need K8s knowledge | Low - AWS native |
| **Servers** | Managed EC2 nodes | Serverless (Fargate) |
| **Scaling** | Pod autoscaler + node autoscaler | Simple service autoscaling |
| **Networking** | VPC CNI + kube-proxy | Simple awsvpc mode |
| **Load Balancing** | Ingress controller needed | Native ALB integration |
| **Secrets** | External Secrets operator | Direct Secrets Manager |
| **Cost Model** | Pay for nodes + EKS fee | Pay per task |
