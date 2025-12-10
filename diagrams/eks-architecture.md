# Aldea EKS Architecture (Previous)

## Overview

Previous production architecture using AWS EKS with Kubernetes for workload orchestration. This architecture was replaced with ECS Fargate for simplified operations.

## Production EKS Cluster (prod-eks)

```mermaid
flowchart TB
    subgraph Internet
        Users[Users/Clients]
    end

    subgraph DNS["Route53 DNS"]
        api[api.aldea.ai]
        backend[backend.aldea.ai]
        platform[platform.aldea.ai]
    end

    subgraph EKS["EKS Cluster: prod-eks"]
        subgraph ControlPlane["EKS Control Plane"]
            API_SERVER[Kubernetes API Server]
            ETCD[(etcd)]
        end

        subgraph NodeGroups["EC2 Node Groups"]
            CPU[cpu-nodes<br/>t3.medium]
            GPU[gpu-nodes<br/>g4dn.xlarge]
            REDIS_NODE[redis-nodes<br/>r6g.large]
        end

        subgraph K8sAddons["Kubernetes Add-ons"]
            ALB_CTRL[AWS Load Balancer Controller]
            EBS_CSI[EBS CSI Driver]
            COREDNS[CoreDNS]
            KUBE_PROXY[kube-proxy]
            VPC_CNI[VPC CNI]
        end

        subgraph Helm["Helm Releases"]
            EXT_SEC[External Secrets]
            SEC_STORE[Secrets Store CSI]
        end

        subgraph Namespaces["Kubernetes Namespaces"]
            subgraph Default["default namespace"]
                WS_DEPLOY[ws-proxy<br/>Deployment]
                WS_SVC[ws-proxy<br/>Service]
                STT_DEPLOY[stt-api<br/>Deployment]
                STT_SVC[stt-api<br/>Service]
                BE_DEPLOY[imp-backend-api<br/>Deployment]
                BE_SVC[imp-backend-api<br/>Service]
                FE_DEPLOY[imp-frontend<br/>Deployment]
                FE_SVC[imp-frontend<br/>Service]
                SQS_DEPLOY[sqs-data-updator<br/>Deployment]
            end

            subgraph RedisNS["redis namespace"]
                REDIS_CLUSTER[Redis Cluster<br/>StatefulSet]
                REDIS_SVC[Redis Service]
            end
        end
    end

    subgraph ALB["Application Load Balancer"]
        INGRESS[ALB Ingress<br/>Managed by Controller]
    end

    subgraph Backend_Services["Backend Services"]
        subgraph RDS["RDS PostgreSQL"]
            DB[(aldea-prod<br/>PostgreSQL)]
        end

        subgraph SQS["SQS Queue"]
            QUEUE[usage-updates-queue]
        end

        subgraph Cognito["Cognito"]
            USER_POOL[User Pool<br/>Authentication]
        end
    end

    subgraph ECR["ECR Repositories"]
        ECR_WS[ws-proxy]
        ECR_STT[stt-api]
        ECR_BE[imp-backend-api]
        ECR_FE[imp-frontend]
        ECR_SQS[sqs-data-updator]
    end

    subgraph IAM["IAM OIDC"]
        OIDC[OIDC Provider]
        SA_ROLES[Service Account IAM Roles]
    end

    Users --> api & backend & platform
    api & backend & platform --> INGRESS
    INGRESS --> WS_SVC & BE_SVC & FE_SVC
    WS_SVC --> WS_DEPLOY
    BE_SVC --> BE_DEPLOY
    FE_SVC --> FE_DEPLOY

    WS_DEPLOY --> REDIS_SVC
    WS_DEPLOY --> STT_SVC
    STT_SVC --> STT_DEPLOY

    BE_DEPLOY --> DB
    BE_DEPLOY --> USER_POOL
    BE_DEPLOY --> QUEUE
    SQS_DEPLOY --> QUEUE
    SQS_DEPLOY --> DB

    ALB_CTRL --> INGRESS
    EXT_SEC --> SA_ROLES
    SEC_STORE --> SA_ROLES
    SA_ROLES --> OIDC

    CPU & GPU & REDIS_NODE --> API_SERVER
```

## EKS Networking Architecture

```mermaid
flowchart TB
    subgraph VPC["VPC: 10.0.0.0/16"]
        subgraph PublicSubnets["Public Subnets"]
            PUB_A[10.0.1.0/24<br/>us-west-2a]
            PUB_B[10.0.2.0/24<br/>us-west-2b]
            PUB_C[10.0.3.0/24<br/>us-west-2c]
            NAT_A[NAT Gateway A]
            NAT_B[NAT Gateway B]
        end

        subgraph PrivateSubnets["Private Subnets"]
            PRIV_A[10.0.11.0/24<br/>us-west-2a]
            PRIV_B[10.0.12.0/24<br/>us-west-2b]
            PRIV_C[10.0.13.0/24<br/>us-west-2c]
        end

        subgraph Workers["EKS Worker Nodes"]
            NODE1[Node 1<br/>us-west-2a]
            NODE2[Node 2<br/>us-west-2b]
            NODE3[Node 3<br/>us-west-2c]
        end

        subgraph VPCEndpoints["VPC Endpoints"]
            EP_ECR_API[ecr.api]
            EP_ECR_DKR[ecr.dkr]
            EP_S3[S3 Gateway]
            EP_STS[STS]
            EP_LOGS[CloudWatch Logs]
            EP_SSM[SSM]
        end
    end

    subgraph IGW["Internet Gateway"]
        IG[IGW]
    end

    PUB_A & PUB_B & PUB_C --> IG
    NODE1 --> PRIV_A --> NAT_A --> PUB_A
    NODE2 --> PRIV_B --> NAT_B --> PUB_B
    NODE3 --> PRIV_C --> NAT_A

    NODE1 & NODE2 & NODE3 -.-> EP_ECR_API & EP_ECR_DKR & EP_S3 & EP_STS & EP_LOGS & EP_SSM
```

## EKS Deployment Model

```mermaid
flowchart LR
    subgraph GitHub["GitHub"]
        REPO[Code Repository]
        GHA[GitHub Actions]
    end

    subgraph CI_CD["CI/CD Pipeline"]
        BUILD[Docker Build]
        PUSH[Push to ECR]
        EKSCTL[eksctl Deploy]
    end

    subgraph EKS["EKS Cluster"]
        DEPLOY[Kubernetes Deployment]
        RS[ReplicaSet]
        PODS[Pods]
    end

    REPO -->|Push| GHA
    GHA --> BUILD --> PUSH
    GHA --> EKSCTL
    EKSCTL -->|kubectl apply| DEPLOY
    DEPLOY --> RS --> PODS
    PUSH -.->|Pull image| PODS
```

## Kubernetes Service Architecture

```mermaid
flowchart TB
    subgraph K8s["Kubernetes Cluster"]
        subgraph Ingress["Ingress Layer"]
            ALB_ING[ALB Ingress]
        end

        subgraph Services["Service Layer"]
            SVC_WS[ws-proxy:8000<br/>ClusterIP]
            SVC_STT[stt-api:8800<br/>ClusterIP]
            SVC_BE[imp-backend:8000<br/>ClusterIP]
            SVC_FE[imp-frontend:3000<br/>ClusterIP]
            SVC_REDIS[redis:6379<br/>ClusterIP]
        end

        subgraph Deployments["Deployment Layer"]
            DEP_WS[ws-proxy<br/>replicas: 2]
            DEP_STT[stt-api<br/>replicas: 2]
            DEP_BE[imp-backend<br/>replicas: 2]
            DEP_FE[imp-frontend<br/>replicas: 2]
        end

        subgraph StatefulSets["StatefulSet Layer"]
            SS_REDIS[redis-cluster<br/>replicas: 3]
        end
    end

    ALB_ING --> SVC_WS & SVC_BE & SVC_FE
    SVC_WS --> DEP_WS
    SVC_STT --> DEP_STT
    SVC_BE --> DEP_BE
    SVC_FE --> DEP_FE
    SVC_REDIS --> SS_REDIS

    DEP_WS --> SVC_REDIS & SVC_STT
    DEP_BE --> SVC_REDIS
```

## EKS IAM Architecture

```mermaid
flowchart TB
    subgraph IAM["IAM"]
        subgraph Roles["IAM Roles"]
            CLUSTER_ROLE[EKS Cluster Role]
            NODE_ROLE[EKS Node Role]
            ALB_ROLE[ALB Controller Role]
            EBS_ROLE[EBS CSI Role]
            BACKEND_ROLE[imp-backend-api Role]
            FRONTEND_ROLE[imp-frontend Role]
            SQS_ROLE[sqs-data-updator Role]
        end

        subgraph OIDC["OIDC Provider"]
            OIDC_PROV[EKS OIDC<br/>oidc.eks.us-west-2.amazonaws.com]
        end
    end

    subgraph K8s["Kubernetes"]
        subgraph SA["Service Accounts"]
            SA_ALB[aws-load-balancer-controller]
            SA_EBS[ebs-csi-controller]
            SA_BE[imp-backend-api]
            SA_FE[imp-frontend]
            SA_SQS[sqs-data-updator]
        end
    end

    SA_ALB -->|AssumeRoleWithWebIdentity| ALB_ROLE
    SA_EBS -->|AssumeRoleWithWebIdentity| EBS_ROLE
    SA_BE -->|AssumeRoleWithWebIdentity| BACKEND_ROLE
    SA_FE -->|AssumeRoleWithWebIdentity| FRONTEND_ROLE
    SA_SQS -->|AssumeRoleWithWebIdentity| SQS_ROLE

    ALB_ROLE & EBS_ROLE & BACKEND_ROLE & FRONTEND_ROLE & SQS_ROLE --> OIDC_PROV
```
