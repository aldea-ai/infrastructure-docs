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
        DNS[Domain Names<br/>api.aldea.ai<br/>backend.aldea.ai<br/>platform.aldea.ai]
        GA[Global Accelerator<br/>Fast worldwide routing]
        WAF[Web Firewall<br/>Blocks bad traffic]
    end

    subgraph LB["Load Balancer - Traffic Director"]
        ALB[Distributes requests<br/>to the right service]
    end

    subgraph ECS["ECS Cluster - Where Services Run"]
        direction LR
        WS[WebSocket Proxy<br/>Live audio streaming]
        HTTP[HTTP Proxy<br/>File uploads]
        BE[Backend API<br/>User accounts]
        FE[Frontend<br/>Web dashboard]
    end

    subgraph Backend["Backend Services"]
        REDIS[Redis Cache<br/>API keys & limits]
        STT[Speech Engine<br/>Does the transcription]
        DB[(Database<br/>User data)]
    end

    USER --> DNS --> GA --> WAF --> ALB
    ALB --> WS & HTTP & BE & FE
    WS & HTTP --> REDIS
    WS & HTTP --> STT
    BE --> DB
```

---

## Production Network Architecture (Multi-AZ)

Production and staging environments use **Multi-AZ deployments** for high availability. All critical resources are distributed across multiple Availability Zones.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Internet["Internet"]
        IGW[Internet Gateway]
    end

    subgraph VPC["Production VPC (10.0.0.0/16)"]
        subgraph AZ_A["Availability Zone A (us-west-2a)"]
            subgraph PubA["Public Subnet A<br/>10.0.1.0/24"]
                ALB_A[ALB Node]
                NAT_A[NAT Gateway A]
            end
            subgraph PrivA["Private Subnet A<br/>10.0.10.0/24"]
                ECS_A[ECS Tasks]
                REDIS_A[Redis Primary]
                RDS_A[RDS Primary]
            end
        end

        subgraph AZ_B["Availability Zone B (us-west-2b)"]
            subgraph PubB["Public Subnet B<br/>10.0.2.0/24"]
                ALB_B[ALB Node]
                NAT_B[NAT Gateway B]
            end
            subgraph PrivB["Private Subnet B<br/>10.0.20.0/24"]
                ECS_B[ECS Tasks]
                REDIS_B[Redis Replica]
                RDS_B[RDS Standby]
            end
        end

        subgraph AZ_C["Availability Zone C (us-west-2c)"]
            subgraph PubC["Public Subnet C<br/>10.0.3.0/24"]
                ALB_C[ALB Node]
            end
            subgraph PrivC["Private Subnet C<br/>10.0.30.0/24"]
                ECS_C[ECS Tasks]
            end
        end
    end

    IGW --> PubA & PubB & PubC
    ALB_A & ALB_B & ALB_C --> PrivA & PrivB & PrivC
    ECS_A --> NAT_A
    ECS_B --> NAT_B
    ECS_C --> NAT_A

    REDIS_A <-.->|Replication| REDIS_B
    RDS_A <-.->|Sync Replication| RDS_B
```

---

## Multi-AZ Resource Distribution

AWS runs services across multiple data centers (Availability Zones) for reliability. If one zone fails, others keep running.

| Resource | Multi-AZ in Prod/Staging | Single-AZ in Dev |
|----------|--------------------------|------------------|
| **ALB** | ✅ Nodes in 3 AZs | ✅ Nodes in 2 AZs |
| **ECS Tasks** | ✅ Distributed across 3 AZs | ⚠️ Single AZ |
| **ElastiCache Redis** | ✅ Primary + Replica in different AZs | ⚠️ Single node |
| **RDS PostgreSQL** | ✅ Multi-AZ with standby | ⚠️ Single instance |
| **NAT Gateway** | ✅ One per AZ (2-3 NATs) | ⚠️ Single NAT |
| **Subnets** | ✅ Public + Private in each AZ | ⚠️ Minimal subnets |

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph ALB["Application Load Balancer (Multi-AZ)"]
        LB[Distributes traffic<br/>across all zones]
    end

    subgraph VPC["Virtual Private Cloud"]
        subgraph AZ_A["Zone A (us-west-2a)"]
            subgraph SubA["Private Subnet A"]
                WS_A[WS-Proxy Task]
                HTTP_A[HTTP-Proxy Task]
                BE_A[Backend Task]
                FE_A[Frontend Task]
            end
        end

        subgraph AZ_B["Zone B (us-west-2b)"]
            subgraph SubB["Private Subnet B"]
                WS_B[WS-Proxy Task]
                HTTP_B[HTTP-Proxy Task]
                BE_B[Backend Task]
                FE_B[Frontend Task]
            end
        end

        subgraph AZ_C["Zone C (us-west-2c)"]
            subgraph SubC["Private Subnet C"]
                SQS_C[DB Sync Task]
            end
        end

        subgraph Data["Shared Data Layer (Multi-AZ)"]
            subgraph RedisCluster["ElastiCache Redis"]
                REDIS_P[(Primary<br/>Zone A)]
                REDIS_R[(Replica<br/>Zone B)]
            end
            subgraph RDSCluster["RDS PostgreSQL"]
                RDS_P[(Primary<br/>Zone A)]
                RDS_S[(Standby<br/>Zone B)]
            end
        end
    end

    LB --> SubA & SubB & SubC
    SubA & SubB --> REDIS_P
    REDIS_P <-.-> REDIS_R
    BE_A & BE_B --> RDS_P
    RDS_P <-.->|Automatic Failover| RDS_S
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

## Key Differences: WebSocket vs HTTP

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph WebSocket["WebSocket (Live Streaming)"]
        direction TB
        W1[Persistent connection]
        W2[Audio sent in small chunks]
        W3[Results arrive in real-time]
        W4[Best for: live calls, meetings]
    end

    subgraph HTTP["HTTP (File Upload)"]
        direction TB
        H1[Single request/response]
        H2[Complete file uploaded]
        H3[Results after processing]
        H4[Best for: recorded files]
    end
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
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart TB
    subgraph Internet["Internet"]
        IGW[Internet Gateway]
    end

    subgraph VPC["Dev VPC (10.1.0.0/16)"]
        subgraph AZ_A["Single Availability Zone (us-west-2a)"]
            subgraph PubDev["Public Subnet<br/>10.1.1.0/24"]
                ALB_DEV[ALB]
                NAT_DEV[Single NAT Gateway]
            end
            subgraph PrivDev["Private Subnet<br/>10.1.10.0/24"]
                WS_DEV[WS-Proxy]
                HTTP_DEV[HTTP-Proxy]
                BE_DEV[Backend]
                FE_DEV[Frontend]
            end
        end

        subgraph DataDev["Data Layer (Single-AZ)"]
            REDIS_DEV[(Redis<br/>Single Node)]
            RDS_DEV[(RDS<br/>No Standby)]
        end
    end

    subgraph Domains["Dev Domains"]
        D1[dev-api.aldea.ai]
        D2[dev-backend.aldea.ai]
        D3[dev-platform.aldea.ai]
    end

    Domains --> IGW --> PubDev
    ALB_DEV --> PrivDev
    PrivDev --> NAT_DEV
    WS_DEV & HTTP_DEV --> REDIS_DEV
    BE_DEV --> RDS_DEV
```

### Production vs Dev Comparison

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Prod["Production/Staging (Multi-AZ)"]
        direction TB
        P1[ALB in 3 AZs]
        P2[ECS tasks in 3 AZs]
        P3[Redis Primary + Replica]
        P4[RDS Multi-AZ Standby]
        P5[NAT Gateway per AZ]
        P6[6 Subnets total]
    end

    subgraph Dev["Dev (Single-AZ)"]
        direction TB
        D1[ALB in 1-2 AZs]
        D2[ECS tasks in 1 AZ]
        D3[Redis Single Node]
        D4[RDS Single Instance]
        D5[Single NAT Gateway]
        D6[2 Subnets total]
    end

    Prod -.-|Cost optimized| Dev
```

---

## How Code Gets Deployed

When developers push code changes:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4a5568', 'primaryTextColor': '#000', 'primaryBorderColor': '#2d3748', 'lineColor': '#2d3748', 'secondaryColor': '#5a6577', 'tertiaryColor': '#6b7280', 'background': '#1a202c', 'mainBkg': '#4a5568', 'secondBkg': '#5a6577', 'clusterBkg': '#3d4852', 'clusterBorder': '#2d3748', 'titleColor': '#000'}}}%%
flowchart LR
    subgraph Dev["Developer"]
        CODE[Push code<br/>to GitHub]
    end

    subgraph CI["Automated Build"]
        BUILD[Build container<br/>image]
        TEST[Run tests]
        PUSH[Push to<br/>image registry]
    end

    subgraph Deploy["Deployment"]
        UPDATE[Update service<br/>with new image]
        ROLL[Rolling update<br/>zero downtime]
    end

    subgraph Live["Production"]
        NEW[New version<br/>running]
    end

    CODE --> BUILD --> TEST --> PUSH --> UPDATE --> ROLL --> NEW
```
