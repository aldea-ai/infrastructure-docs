# EKS to ECS Migration Comparison

## Architecture Comparison

```mermaid
flowchart TB
    subgraph Before["BEFORE: EKS Architecture"]
        direction TB
        subgraph EKS_Infra["Infrastructure Layer"]
            EKS_CP[EKS Control Plane<br/>Managed by AWS]
            EKS_NODES[EC2 Node Groups<br/>cpu-nodes / gpu-nodes / redis-nodes]
            EKS_ADDONS[K8s Add-ons<br/>CoreDNS / VPC-CNI / ALB Controller]
        end

        subgraph EKS_K8s["Kubernetes Layer"]
            EKS_HELM[Helm Releases<br/>External Secrets / CSI Driver]
            EKS_DEPLOY[Deployments<br/>ws-proxy / stt-api / backend / frontend]
            EKS_SVC[Services<br/>ClusterIP / LoadBalancer]
            EKS_ING[Ingress<br/>ALB Ingress Controller]
        end

        subgraph EKS_Data["Data Layer"]
            EKS_REDIS[Redis StatefulSet<br/>In-cluster]
            EKS_RDS[(RDS PostgreSQL)]
        end

        EKS_CP --> EKS_NODES
        EKS_NODES --> EKS_DEPLOY
        EKS_HELM --> EKS_DEPLOY
        EKS_DEPLOY --> EKS_SVC --> EKS_ING
        EKS_DEPLOY --> EKS_REDIS & EKS_RDS
    end

    subgraph After["AFTER: ECS Architecture"]
        direction TB
        subgraph ECS_Infra["Infrastructure Layer"]
            ECS_CLUSTER[ECS Cluster<br/>Fully Managed]
            ECS_FARGATE[Fargate Tasks<br/>Serverless Compute]
            ECS_GA[Global Accelerator<br/>Anycast IPs]
        end

        subgraph ECS_Services["Service Layer"]
            ECS_WS[ws-proxy Service<br/>WebSocket Streaming]
            ECS_HTTP[http-proxy Service<br/>HTTP Transcription]
            ECS_BE[Backend Service<br/>Platform API]
            ECS_FE[Frontend Service<br/>Platform UI]
        end

        subgraph ECS_Data["Data Layer"]
            ECS_REDIS[ElastiCache Redis<br/>Managed / Multi-AZ]
            ECS_RDS[(RDS PostgreSQL)]
        end

        ECS_CLUSTER --> ECS_FARGATE
        ECS_FARGATE --> ECS_WS & ECS_HTTP & ECS_BE & ECS_FE
        ECS_GA --> ECS_WS & ECS_HTTP
        ECS_WS & ECS_HTTP --> ECS_REDIS
        ECS_BE --> ECS_RDS
    end

    Before -.->|Migration| After
```

## Component Mapping

```mermaid
flowchart LR
    subgraph EKS["EKS Components"]
        E1[EKS Control Plane]
        E2[EC2 Node Groups]
        E3[Kubernetes Deployments]
        E4[K8s Services]
        E5[ALB Ingress Controller]
        E6[Redis StatefulSet]
        E7[Helm Charts]
        E8[eksctl / kubectl]
    end

    subgraph ECS["ECS Components"]
        C1[ECS Cluster]
        C2[Fargate Tasks]
        C3[ECS Services]
        C4[Target Groups]
        C5[ALB + WAF + GA]
        C6[ElastiCache Redis]
        C7[Task Definitions]
        C8[aws ecs CLI]
    end

    E1 -->|Replaced by| C1
    E2 -->|Replaced by| C2
    E3 -->|Replaced by| C3
    E4 -->|Replaced by| C4
    E5 -->|Replaced by| C5
    E6 -->|Replaced by| C6
    E7 -->|Replaced by| C7
    E8 -->|Replaced by| C8
```

## Complexity Comparison

```mermaid
graph TB
    subgraph EKS_Complexity["EKS: Higher Complexity"]
        EKS_1[Kubernetes API Server]
        EKS_2[etcd State Management]
        EKS_3[Node Group Scaling]
        EKS_4[Pod Scheduling]
        EKS_5[Service Mesh Options]
        EKS_6[RBAC Configuration]
        EKS_7[Helm Release Management]
        EKS_8[CNI Plugin Management]
        EKS_9[CSI Driver Setup]
        EKS_10[Ingress Controller]
    end

    subgraph ECS_Simplicity["ECS: Simplified"]
        ECS_1[Task Definitions]
        ECS_2[Service Auto-scaling]
        ECS_3[ALB Integration]
        ECS_4[Secrets Manager]
    end

    EKS_Complexity -->|Simplified to| ECS_Simplicity
```

## Deployment Pipeline Comparison

```mermaid
flowchart LR
    subgraph EKS_Deploy["EKS Deployment"]
        ED1[GitHub Push] --> ED2[Build Docker Image]
        ED2 --> ED3[Push to ECR]
        ED3 --> ED4[Update Helm Values]
        ED4 --> ED5[eksctl/kubectl apply]
        ED5 --> ED6[Wait for Rollout]
        ED6 --> ED7[Verify Pods Ready]
    end

    subgraph ECS_Deploy["ECS Deployment"]
        CD1[GitHub Push] --> CD2[Build Docker Image]
        CD2 --> CD3[Push to ECR]
        CD3 --> CD4[Register Task Def]
        CD4 --> CD5[Update Service]
        CD5 --> CD6[Wait for Stable]
    end

    EKS_Deploy -.->|Simplified| ECS_Deploy
```

## Network Architecture Comparison

```mermaid
flowchart TB
    subgraph EKS_Net["EKS Networking"]
        EN_USER[Users] --> EN_R53[Route53]
        EN_R53 --> EN_ALB[ALB]
        EN_ALB --> EN_ING[Ingress Controller]
        EN_ING --> EN_SVC[K8s Service]
        EN_SVC --> EN_POD[Pod]
        EN_POD --> EN_CNI[VPC CNI]
        EN_CNI --> EN_VPC[VPC Subnet]
    end

    subgraph ECS_Net["ECS Networking"]
        CN_USER[Users] --> CN_GA[Global Accelerator]
        CN_GA --> CN_WAF[AWS WAF]
        CN_WAF --> CN_ALB[ALB]
        CN_ALB --> CN_TG[Target Group]
        CN_TG --> CN_TASK[Fargate Task]
        CN_TASK --> CN_ENI[ENI in VPC]
    end
```

## Key Differences Summary

| Aspect | EKS | ECS |
|--------|-----|-----|
| **Compute** | EC2 Node Groups | Fargate (Serverless) |
| **Container Orchestration** | Kubernetes | ECS Native |
| **State Management** | etcd | AWS Managed |
| **Scaling** | HPA + Cluster Autoscaler | Service Auto Scaling |
| **Networking** | VPC CNI + kube-proxy | awsvpc mode |
| **Load Balancing** | ALB Ingress Controller | Native ALB Integration |
| **Secrets** | External Secrets + CSI | Secrets Manager Direct |
| **Service Discovery** | CoreDNS | Cloud Map (optional) |
| **Monitoring** | Prometheus + Grafana | CloudWatch + X-Ray |
| **Cost Model** | EC2 instances + EKS fee | Per-task pricing |
| **Complexity** | High (K8s knowledge required) | Low (AWS native) |
| **Flexibility** | Very High | Moderate |

## Benefits of Migration

```mermaid
mindmap
  root((ECS Benefits))
    Operational
      No cluster management
      No node patching
      Simplified deployments
      Native AWS integration
    Cost
      Pay per task
      No idle node costs
      No EKS control plane fee
    Security
      Fargate isolation
      Managed updates
      Native IAM integration
    Reliability
      Multi-AZ by default
      Global Accelerator
      AWS WAF integration
```

## Services Mapping

### Production Environment

| Service | EKS (Before) | ECS (After) |
|---------|--------------|-------------|
| WebSocket Proxy | ws-proxy Deployment | ws-proxy-live1/live2 Services |
| HTTP Transcription | stt-api Deployment | http-proxy-live1/live2 Services |
| Platform Backend | imp-backend-api Deployment | imp-backend Service |
| Platform Frontend | imp-frontend Deployment | imp-frontend Service |
| DB Sync | sqs-data-updator Deployment | sqs-updater Service |
| Redis | Redis StatefulSet | ElastiCache Replication Group |

### Dev Environment

| Service | EKS (Before) | ECS (After) |
|---------|--------------|-------------|
| WebSocket Proxy | N/A | aldea-ecs-dev Service |
| HTTP Transcription | N/A | aldea-ecs-dev-transcribe Service |
| Platform Backend | N/A | aldea-ecs-dev-backend Service |
| Platform Frontend | N/A | aldea-ecs-dev-frontend Service |
| DB Sync | N/A | aldea-ecs-dev-sqs-updater Service |
