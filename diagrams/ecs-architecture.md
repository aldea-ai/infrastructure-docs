# Aldea ECS Architecture

## What is This?

This document shows how Aldea's speech-to-text (ASR) services run in the cloud using AWS ECS (Elastic Container Service). Think of ECS as a way to run applications in isolated containers without managing servers.

---

## How Users Connect to Aldea

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'border1': '#2d3748', 'border2': '#2d3748', 'note1': '#5a6577', 'note2': '#000', 'text1': '#000', 'text2': '#000', 'critical': '#4a5568', 'done': '#5a6577', 'arrowheadColor': '#2d3748', 'fontFamily': 'arial', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Users["Your Application"]
        APP[Your App]
    end

    subgraph Domains["Web Addresses"]
        API[api.aldea.ai<br/>Speech-to-Text API]
        BACK[backend.aldea.ai<br/>Platform API]
        PLAT[platform.aldea.ai<br/>Web Dashboard]
    end

    subgraph Services["What Each Does"]
        S1[Real-time transcription<br/>& audio processing]
        S2[Account management<br/>& billing]
        S3[Web interface to<br/>manage your account]
    end

    APP --> API --> S1
    APP --> BACK --> S2
    APP --> PLAT --> S3
```

---

## Production Architecture Overview

A simplified view of how requests flow through the system:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Internet["The Internet"]
        USER[Your Application]
    end

    subgraph Edge["Edge Layer - Security & Speed"]
        DNS[Domain Names<br/>api.aldea.ai]
        GA[Global Accelerator<br/>Fast worldwide routing]
        WAF[Web Firewall<br/>Blocks bad traffic]
    end

    subgraph LB["Load Balancer - Traffic Director"]
        ALB[Distributes requests<br/>to the right service]
    end

    subgraph ECS["ECS Cluster - Services"]
        direction LR
        WS[WebSocket Proxy<br/>Live audio streaming]
        HTTP[HTTP Proxy<br/>File uploads]
        BE[Backend API<br/>Account & billing]
        FE[Frontend<br/>Web dashboard]
        SYNC[DB Sync<br/>Redis to Postgres]
    end

    subgraph Data["Data Layer"]
        REDIS[Redis Cache<br/>API keys & limits]
        RDS[(PostgreSQL<br/>User data)]
        SQS[[SQS Queue]]
    end

    subgraph Serverless["Serverless"]
        LAMBDA[Lambda<br/>Queue processor]
    end

    subgraph STT_Live["STT Live Servers (GPU VMs)"]
        LIVE1[stt-api-live-1]
        LIVE2[stt-api-live-2]
    end

    subgraph STT_Dev["STT Dev Servers (GPU VMs)"]
        DEV1[stt-api-dev-1]
        DEV2[stt-api-dev-2]
    end

    USER --> DNS --> GA --> WAF --> ALB
    ALB --> WS & HTTP & BE & FE
    WS & HTTP --> REDIS
    BE --> REDIS & RDS
    FE --> BE
    RDS -->|NOTIFY| SYNC
    SYNC --> SQS
    SQS --> LAMBDA
    LAMBDA --> REDIS
    HTTP --> LIVE1 & LIVE2
    WS --> DEV1 & DEV2
```

---

## Production Network Architecture (Multi-AZ)

Production and staging environments use **Multi-AZ deployments** for high availability. All critical resources are distributed across multiple Availability Zones.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    IGW[Internet Gateway]

    subgraph AZ_A["Zone A (us-west-2a)"]
        PubA["Public Subnet 10.0.1.0/24"]
        PrivA["Private Subnet 10.0.10.0/24"]
        ALB_A[ALB Node]
        NAT_A[NAT Gateway]
        ECS_A[ECS Tasks]
        REDIS_A[(Redis Primary)]
        RDS_A[(RDS Primary)]
    end

    subgraph AZ_B["Zone B (us-west-2b)"]
        PubB["Public Subnet 10.0.2.0/24"]
        PrivB["Private Subnet 10.0.20.0/24"]
        ALB_B[ALB Node]
        NAT_B[NAT Gateway]
        ECS_B[ECS Tasks]
        REDIS_B[(Redis Replica)]
        RDS_B[(RDS Standby)]
    end

    IGW --> ALB_A & ALB_B
    ALB_A --> ECS_A
    ALB_B --> ECS_B
    ECS_A --> NAT_A
    ECS_B --> NAT_B

    REDIS_A <-.->|Replication| REDIS_B
    RDS_A <-.->|Sync Replication| RDS_B
```

---

## Multi-AZ Resource Distribution

AWS runs services across multiple data centers (Availability Zones) for reliability. If one zone fails, others keep running.

| Resource | Multi-AZ in Prod/Staging | Single-AZ in Dev |
|----------|--------------------------|------------------|
| **ALB** | ✅ Nodes in 2 AZs | ✅ Nodes in 2 AZs |
| **ECS Tasks** | ✅ Distributed across 2 AZs | ⚠️ Single AZ |
| **ElastiCache Redis** | ✅ Primary + Replica in different AZs | ⚠️ Single node |
| **RDS PostgreSQL** | ✅ Multi-AZ with standby | ⚠️ Single instance |
| **NAT Gateway** | ✅ One per AZ (2 NATs) | ⚠️ Single NAT |
| **Subnets** | ✅ Public + Private in each AZ | ⚠️ Minimal subnets |

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    LB[Application Load Balancer]

    subgraph AZ_A["Zone A"]
        WS_A[WS-Proxy]
        HTTP_A[HTTP-Proxy]
        BE_A[Backend]
        FE_A[Frontend]
        SYNC_A[DB Sync]
    end

    subgraph AZ_B["Zone B"]
        WS_B[WS-Proxy]
        HTTP_B[HTTP-Proxy]
        BE_B[Backend]
        FE_B[Frontend]
        SYNC_B[DB Sync]
    end

    subgraph Data["Data Layer (Multi-AZ)"]
        REDIS_P[(Redis Primary)]
        REDIS_R[(Redis Replica)]
        RDS_P[(RDS Primary)]
        RDS_R[(RDS Standby)]
        SQS[[SQS Queue]]
    end

    subgraph Serverless["Serverless"]
        LAMBDA[Lambda]
    end

    subgraph STT_Live["STT Live Servers (GPU VMs)"]
        LIVE1[stt-api-live-1]
        LIVE2[stt-api-live-2]
    end

    subgraph STT_Dev["STT Dev Servers (GPU VMs)"]
        DEV1[stt-api-dev-1]
        DEV2[stt-api-dev-2]
    end

    LB --> AZ_A & AZ_B
    WS_A & WS_B & HTTP_A & HTTP_B --> REDIS_P
    BE_A & BE_B --> REDIS_P & RDS_P
    FE_A & FE_B --> BE_A & BE_B
    RDS_P -->|NOTIFY| SYNC_A & SYNC_B
    SYNC_A & SYNC_B --> SQS
    SQS --> LAMBDA
    LAMBDA --> REDIS_P
    REDIS_P <-.-> REDIS_R
    RDS_P <-.-> RDS_R
    HTTP_A & HTTP_B --> STT_Live
    WS_A & WS_B --> STT_Dev
```

---

## WebSocket Flow - Live Audio Streaming

For real-time transcription where audio streams continuously:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'actorBkg': '#4a5568', 'actorBorder': '#2d3748', 'actorTextColor': '#000', 'actorLineColor': '#2d3748', 'signalColor': '#2d3748', 'signalTextColor': '#000', 'labelBoxBkgColor': '#5a6577', 'labelBoxBorderColor': '#2d3748', 'labelTextColor': '#000', 'loopTextColor': '#000', 'noteBkgColor': '#5a6577', 'noteTextColor': '#000', 'noteBorderColor': '#2d3748', 'activationBkgColor': '#6b7280', 'activationBorderColor': '#2d3748', 'sequenceNumberColor': '#000'}}}%%
sequenceDiagram
    participant App as Your App
    participant GA as Global Accelerator
    participant LB as Load Balancer
    participant WS as WebSocket Proxy
    participant Redis as Redis Cache
    participant STT as Speech Engine

    Note over App,STT: Real-time Audio Streaming

    App->>GA: Connect to wss://api.aldea.ai/listen
    GA->>LB: Route to nearest server
    LB->>WS: Open WebSocket connection

    WS->>Redis: Check API key valid?
    Redis-->>WS: Yes, allowed 10 connections

    WS->>STT: Open audio stream

    rect rgb(74, 85, 104)
        Note over App,STT: Continuous streaming loop
        App->>WS: Send audio chunk
        WS->>STT: Forward audio
        STT-->>WS: "Hello world"
        WS-->>App: {"text": "Hello world"}
    end

    App->>WS: Close connection
    WS->>Redis: Log: used 45 seconds
```

---

## HTTP Flow - File Upload Transcription

For uploading audio files to get transcription back:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'actorBkg': '#4a5568', 'actorBorder': '#2d3748', 'actorTextColor': '#000', 'actorLineColor': '#2d3748', 'signalColor': '#2d3748', 'signalTextColor': '#000', 'labelBoxBkgColor': '#5a6577', 'labelBoxBorderColor': '#2d3748', 'labelTextColor': '#000', 'loopTextColor': '#000', 'noteBkgColor': '#5a6577', 'noteTextColor': '#000', 'noteBorderColor': '#2d3748', 'activationBkgColor': '#6b7280', 'activationBorderColor': '#2d3748', 'sequenceNumberColor': '#000'}}}%%
sequenceDiagram
    participant App as Your App
    participant GA as Global Accelerator
    participant LB as Load Balancer
    participant HTTP as HTTP Proxy
    participant Redis as Redis Cache
    participant STT as Speech Engine

    Note over App,STT: File Upload Transcription

    App->>GA: POST api.aldea.ai/v1/listen<br/>+ audio file (meeting.wav)
    GA->>LB: Route request
    LB->>HTTP: Forward to HTTP service

    HTTP->>Redis: Check API key valid?
    Redis-->>HTTP: Yes, allowed

    HTTP->>STT: Send complete audio file

    Note over STT: Processing audio...<br/>This may take a few seconds

    STT-->>HTTP: Complete transcription

    HTTP->>Redis: Log: processed 5 minutes of audio

    HTTP-->>App: {"text": "Meeting transcript..."}
```

---

## Dev Environment (Single-AZ - Cost Optimized)

The dev environment uses **single-AZ deployments** to reduce costs. This is acceptable for development since uptime is not critical.

**Cost savings in Dev:**
- No Multi-AZ RDS standby (~50% RDS cost savings)
- Single Redis node instead of cluster (~50% ElastiCache savings)
- Single NAT Gateway (~66% NAT cost savings)
- Fewer ECS tasks

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Domains["Dev Domains"]
        D1[dev-api.aldea.ai]
    end

    IGW[Internet Gateway]
    ALB[Load Balancer]
    NAT[NAT Gateway]

    subgraph AZ_A["Single Zone (us-west-2a)"]
        WS[WS-Proxy]
        HTTP[HTTP-Proxy]
        BE[Backend]
        FE[Frontend]
        SYNC[DB Sync]
    end

    subgraph Data["Data Layer (Single-AZ)"]
        REDIS[(Redis Single Node)]
        RDS[(RDS Single Instance)]
        SQS[[SQS Queue]]
    end

    subgraph Serverless["Serverless"]
        LAMBDA[Lambda]
    end

    subgraph STT_Live["STT Live Servers (GPU VMs)"]
        LIVE1[stt-api-live-1]
        LIVE2[stt-api-live-2]
    end

    subgraph STT_Dev["STT Dev Servers (GPU VMs)"]
        DEV1[stt-api-dev-1]
        DEV2[stt-api-dev-2]
    end

    Domains --> IGW --> ALB --> AZ_A
    AZ_A --> NAT
    WS & HTTP --> REDIS
    BE --> REDIS & RDS
    FE --> BE
    RDS -->|NOTIFY| SYNC
    SYNC --> SQS
    SQS --> LAMBDA
    LAMBDA --> REDIS
    HTTP --> STT_Live
    WS --> STT_Dev
```

---

## Branching Strategy

All repositories follow this branching strategy:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Branches["Git Branches"]
        FEAT[feature/*]
        DEV[dev]
        MAIN[main]
        REL[Release Tag]
    end

    subgraph Environments["Deployment Targets"]
        D[Dev ECS]
        S[Staging ECS]
        P[Prod ECS]
    end

    FEAT -->|PR + Tests| DEV
    DEV -->|Auto deploy| D
    DEV -->|Merge| MAIN
    MAIN -->|Auto deploy| S
    MAIN -->|Create release| REL
    REL -->|Auto deploy| P
```

---

## CI Pipeline - Pull Requests

When a PR is opened, tests run automatically:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Trigger["PR Opened"]
        PR[Pull Request]
    end

    subgraph Tests["CI Tests"]
        PROXY[Proxy Tests<br/>Rust]
        WS[WebSocket Tests<br/>Python]
        LAMBDA[Lambda Tests]
    end

    subgraph Result["Outcome"]
        PASS[✓ Ready to merge]
        FAIL[✗ Fix required]
    end

    PR --> PROXY & WS & LAMBDA
    PROXY & WS & LAMBDA -->|All pass| PASS
    PROXY & WS & LAMBDA -->|Any fail| FAIL
```

---

## Deployment Pipeline - aldea-proxy-services

Services: **ws-proxy**, **transcribe**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Triggers["Trigger"]
        DEV_PUSH[Push to dev]
        MAIN_PUSH[Push to main]
        RELEASE[GitHub Release]
    end

    subgraph Build["Build"]
        DOCKER[Docker Build<br/>linux/amd64]
        ECR[Push to ECR]
    end

    subgraph DevDeploy["Dev Deployment"]
        DEV_ECS[ECS Rolling Update<br/>dev-ws-proxy]
    end

    subgraph StagingDeploy["Staging Deployment"]
        STG_WS[ws-proxy<br/>dev1 + dev2]
        STG_HTTP[transcribe<br/>live1 + live2]
        VALIDATE_S[Validation Tests<br/>st-api.aldea.ai]
    end

    subgraph ProdDeploy["Production Deployment"]
        PROD_WS[ws-proxy<br/>CodeDeploy Blue/Green]
        PROD_HTTP[transcribe<br/>CodeDeploy Blue/Green]
        VALIDATE_P[Validation Tests<br/>api.aldea.ai]
    end

    DEV_PUSH --> DOCKER --> ECR --> DEV_ECS
    MAIN_PUSH --> DOCKER --> ECR --> STG_WS & STG_HTTP --> VALIDATE_S
    RELEASE --> DOCKER --> ECR --> PROD_WS & PROD_HTTP --> VALIDATE_P
```

---

## Deployment Pipeline - imp-backend

Service: **Backend API**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Triggers["Trigger"]
        MAIN_PUSH[Push to main]
        RELEASE[GitHub Release]
    end

    subgraph Build["Build"]
        DOCKER[Docker Build<br/>linux/amd64]
        ECR[Push to ECR]
    end

    subgraph StagingDeploy["Staging"]
        STG_TASK[Update Task Def]
        STG_ECS[ECS Rolling Update<br/>staging-ws-proxy-backend]
        SLACK_S[Slack Notification]
    end

    subgraph ProdDeploy["Production"]
        PROD_TASK[Update Task Def]
        CODEDEPLOY[CodeDeploy<br/>Blue/Green]
        SLACK_P[Slack Notification]
    end

    MAIN_PUSH --> DOCKER --> ECR --> STG_TASK --> STG_ECS --> SLACK_S
    RELEASE --> DOCKER --> ECR --> PROD_TASK --> CODEDEPLOY --> SLACK_P
```

---

## Deployment Pipeline - imp-frontend

Service: **Frontend Web App**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Triggers["Trigger"]
        MAIN_PUSH[Push to main]
        RELEASE[GitHub Release]
    end

    subgraph Build["Build"]
        SECRETS[Fetch NEXT_PUBLIC_*<br/>from Secrets Manager]
        DOCKER[Docker Build<br/>with build args]
        ECR[Push to ECR]
    end

    subgraph StagingDeploy["Staging"]
        STG_TASK[Update Task Def]
        STG_ECS[ECS Rolling Update<br/>staging-ws-proxy-frontend]
        SLACK_S[Slack Notification]
    end

    subgraph ProdDeploy["Production"]
        PROD_TASK[Update Task Def]
        CODEDEPLOY[CodeDeploy<br/>Blue/Green]
        SLACK_P[Slack Notification]
    end

    MAIN_PUSH --> SECRETS --> DOCKER --> ECR --> STG_TASK --> STG_ECS --> SLACK_S
    RELEASE --> SECRETS --> DOCKER --> ECR --> PROD_TASK --> CODEDEPLOY --> SLACK_P
```

---

## CodeDeploy Blue/Green Deployment

Production services use CodeDeploy for zero-downtime deployments:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Before["Before Deployment"]
        BLUE[Blue Tasks<br/>Current version]
        ALB_B[ALB routing<br/>100% traffic]
    end

    subgraph During["During Deployment"]
        BLUE2[Blue Tasks]
        GREEN[Green Tasks<br/>New version]
        ALB_D[ALB testing<br/>Green target group]
    end

    subgraph After["After Success"]
        GREEN2[Green Tasks<br/>New version]
        ALB_A[ALB routing<br/>100% traffic]
        TERM[Blue terminated<br/>after 5 min]
    end

    Before --> During --> After
```

---

## Deployment Summary by Service

| Service | Repository | Dev | Staging | Production |
|---------|------------|-----|---------|------------|
| **ws-proxy** | aldea-proxy-services | ECS Rolling | ECS Rolling (×2) | CodeDeploy B/G |
| **transcribe** | aldea-proxy-services | ECS Rolling | ECS Rolling (×2) | CodeDeploy B/G |
| **Backend** | imp-backend | ECS Rolling | ECS Rolling | CodeDeploy B/G |
| **Frontend** | imp-frontend | ECS Rolling | ECS Rolling | CodeDeploy B/G |
| **DB Sync** | aldea-proxy-services | ECS Rolling | ECS Rolling | ECS Rolling |

---

## Environment Domains

Each environment has dedicated domains for all services:

| Service | Production | Staging | Dev |
|---------|------------|---------|-----|
| **ASR API** | api.aldea.ai | st-api.aldea.ai | dev-api.aldea.ai |
| **Backend API** | backend.aldea.ai | st-backend.aldea.ai | - |
| **Frontend** | platform.aldea.ai | st-frontend.aldea.ai | - |
| **Grafana** | grafana.aldea.ai | - | - |

### Domain Routing

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Domains["Public Domains"]
        API_PROD[api.aldea.ai]
        API_STG[st-api.aldea.ai]
        API_DEV[dev-api.aldea.ai]
    end

    subgraph GA["Global Accelerator"]
        GA_PROD[Prod GA<br/>Anycast IPs]
        GA_STG[Staging GA<br/>Anycast IPs]
    end

    subgraph ECS["ECS Clusters"]
        PROD[prod-ws-proxy]
        STG[staging-ws-proxy]
        DEV[dev-ws-proxy]
    end

    API_PROD --> GA_PROD --> PROD
    API_STG --> GA_STG --> STG
    API_DEV --> DEV
```

---

## Staging Environment

Staging mirrors production architecture but uses different STT server routing for testing.

### Staging Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'clusterBkg': 'transparent', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Domains["Staging Domains"]
        ST_API[st-api.aldea.ai]
        ST_BACK[st-backend.aldea.ai]
        ST_FRONT[st-frontend.aldea.ai]
    end

    subgraph GA["Global Accelerator"]
        ST_GA[Staging GA]
    end

    subgraph ALB["Load Balancer"]
        ST_ALB[staging-ws-proxy ALB]
    end

    subgraph ECS["ECS Cluster (staging-ws-proxy)"]
        direction LR
        ST_WS[ws-proxy<br/>WebSocket]
        ST_HTTP[transcribe<br/>HTTP]
        ST_BE[Backend API]
        ST_FE[Frontend]
    end

    subgraph Data["Data Layer"]
        ST_REDIS[(Redis)]
        ST_RDS[(PostgreSQL)]
    end

    subgraph STT["STT Servers"]
        DEV1[stt-api-dev-1]
        DEV2[stt-api-dev-2]
    end

    ST_API --> ST_GA --> ST_ALB
    ST_BACK --> ST_ALB
    ST_FRONT --> ST_ALB
    ST_ALB --> ST_WS & ST_HTTP & ST_BE & ST_FE
    ST_WS & ST_HTTP --> ST_REDIS
    ST_BE --> ST_REDIS & ST_RDS
    ST_FE --> ST_BE
    ST_WS --> DEV1 & DEV2
    ST_HTTP --> DEV1 & DEV2
```

### Staging vs Production

| Aspect | Production | Staging |
|--------|------------|---------|
| **STT Servers** | stt-api-live-1, stt-api-live-2 | stt-api-dev-1, stt-api-dev-2 |
| **Multi-AZ** | Yes | Yes |
| **CodeDeploy** | Blue/Green | Rolling updates |
| **Global Accelerator** | Yes | Yes |
| **Redis** | Cluster mode | Cluster mode |

---

## Redis Seeding and Org Sync

API keys are cached in Redis for fast validation. The sync process ensures Redis stays up-to-date with the source database.

### Sync Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'actorBkg': '#4a5568', 'actorBorder': '#2d3748', 'actorTextColor': '#000', 'signalColor': '#2d3748', 'signalTextColor': '#000'}}}%%
sequenceDiagram
    participant RDS as PostgreSQL
    participant SYNC as DB Sync Service
    participant SQS as SQS Queue
    participant LAMBDA as Lambda Processor
    participant REDIS as Redis Cache

    Note over RDS,REDIS: Real-time Org/API Key Sync

    RDS->>SYNC: NOTIFY on org changes
    SYNC->>SQS: Queue update message
    SQS->>LAMBDA: Trigger processor
    LAMBDA->>RDS: Fetch org details
    LAMBDA->>REDIS: Update cache
```

### Components

| Component | Description |
|-----------|-------------|
| **DB Sync Service** | ECS service listening for PostgreSQL NOTIFY events |
| **SQS Queue** | `prod-ws-proxy-org-sync` - buffers update messages |
| **Lambda Processor** | Processes queue messages and updates Redis |
| **Redis** | Caches org data with API keys and rate limits |

### Redis Data Structure

```
org:{org_id}:api_keys     # Set of API key hashes
org:{org_id}:limits       # Hash of rate limits
apikey:{key_hash}         # Points to org_id
```

### Initial Seeding

On deployment or Redis reset, initial seeding is triggered:

```bash
# Invoke Lambda to seed all orgs
AWS_PROFILE=aldea-prod aws lambda invoke \
  --function-name prod-ws-proxy-redis-seeder \
  --payload '{"seed_all": true}' \
  --region us-west-2 \
  /tmp/response.json
```
