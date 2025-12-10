# Aldea ECS Architecture

## Overview

Current production architecture using AWS ECS Fargate for all ASR (Automatic Speech Recognition) services.

## Production ECS Cluster (prod-ws-proxy)

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

    subgraph GA["Global Accelerator"]
        GA_EP[Anycast IPs<br/>75.2.x.x / 99.83.x.x]
    end

    subgraph WAF["AWS WAF"]
        WAF_RULES[Rate Limiting<br/>Geo Blocking<br/>IP Allowlist]
    end

    subgraph ALB["Application Load Balancer"]
        HTTPS_LIS[HTTPS Listener :443]

        subgraph Routing["Routing Rules"]
            R1[Priority 10: platform.aldea.ai → Frontend]
            R2[Priority 20: backend.aldea.ai → Backend]
            R3[Priority 100: WebSocket + /listen/* → WS-Proxy]
            R4[Priority 200: HTTP + /listen/* → HTTP-Proxy]
            R5[Priority 300: /transcribe/* → HTTP-Proxy]
        end
    end

    subgraph ECS["ECS Cluster: prod-ws-proxy"]
        subgraph Services["Fargate Services"]
            WS1[ws-proxy-live1<br/>WebSocket Streaming]
            WS2[ws-proxy-live2<br/>WebSocket Streaming]
            HTTP1[http-proxy-live1<br/>HTTP Transcription]
            HTTP2[http-proxy-live2<br/>HTTP Transcription]
            BE[imp-backend<br/>Platform API]
            FE[imp-frontend<br/>Platform UI]
            SQS[sqs-updater<br/>DB Sync]
        end

        subgraph TG["Target Groups"]
            TG_WS1[ws-proxy-live1 TG]
            TG_WS2[ws-proxy-live2 TG]
            TG_HTTP1[http-proxy-live1 TG]
            TG_HTTP2[http-proxy-live2 TG]
            TG_BE[backend TG]
            TG_FE[frontend TG]
        end
    end

    subgraph Backend_Services["Backend Services"]
        subgraph Redis["ElastiCache Redis"]
            REDIS[Redis 7.1<br/>TLS Enabled<br/>Multi-AZ]
        end

        subgraph STT["STT API Backends"]
            STT1[stt-api-live-1.aldea.ai:8800]
            STT2[stt-api-live-2.aldea.ai:8800]
        end

        subgraph RDS["RDS PostgreSQL"]
            DB[(aldea-prod<br/>PostgreSQL)]
        end

        subgraph SQS["SQS Queue"]
            QUEUE[usage-updates-queue]
        end
    end

    subgraph ECR["ECR Repositories"]
        ECR_WS[prod-ws-proxy-ws-proxy]
        ECR_HTTP[prod-ws-proxy-http-proxy]
        ECR_BE[prod-ws-proxy-backend]
        ECR_FE[prod-ws-proxy-frontend]
        ECR_SQS[prod-ws-proxy-sqs-updater]
    end

    subgraph Secrets["Secrets Manager"]
        SEC_REDIS[redis-auth-token]
        SEC_BE[imp-backend/env]
        SEC_FE[imp-frontend/env]
    end

    Users --> api & backend & platform
    api & backend & platform --> GA_EP
    GA_EP --> WAF_RULES
    WAF_RULES --> HTTPS_LIS
    HTTPS_LIS --> Routing

    R1 --> TG_FE --> FE
    R2 --> TG_BE --> BE
    R3 --> TG_WS1 & TG_WS2
    TG_WS1 --> WS1
    TG_WS2 --> WS2
    R4 & R5 --> TG_HTTP1 & TG_HTTP2
    TG_HTTP1 --> HTTP1
    TG_HTTP2 --> HTTP2

    WS1 & WS2 --> REDIS
    WS1 --> STT1
    WS2 --> STT2
    HTTP1 --> STT1
    HTTP2 --> STT2

    BE --> DB
    BE --> QUEUE
    SQS --> QUEUE
    SQS --> DB

    WS1 & WS2 & HTTP1 & HTTP2 & BE & FE & SQS -.-> SEC_REDIS & SEC_BE & SEC_FE
```

## Dev ECS Cluster (aldea-ecs-dev)

```mermaid
flowchart TB
    subgraph DNS["Route53 DNS"]
        dev_api[dev-api.aldea.ai]
        dev_backend[dev-backend.aldea.ai]
        dev_platform[dev-platform.aldea.ai]
    end

    subgraph ALB["Application Load Balancer"]
        DEV_LIS[HTTPS Listener :443]
    end

    subgraph ECS["ECS Cluster: aldea-ecs-dev"]
        WS[ws-proxy<br/>WebSocket]
        TRANS[transcribe<br/>HTTP API]
        BE_DEV[backend<br/>Platform API]
        FE_DEV[frontend<br/>Platform UI]
        SQS_DEV[sqs-updater<br/>DB Sync]
    end

    subgraph Backend["Backend Services"]
        REDIS_DEV[ElastiCache Redis<br/>dev-ws-proxy]
        STT_DEV[STT API Backend]
        DB_DEV[(Staging DB<br/>PostgreSQL)]
    end

    dev_api --> DEV_LIS
    dev_backend --> DEV_LIS
    dev_platform --> DEV_LIS

    DEV_LIS --> WS & TRANS & BE_DEV & FE_DEV

    WS --> REDIS_DEV --> STT_DEV
    TRANS --> STT_DEV
    BE_DEV --> DB_DEV
    SQS_DEV --> DB_DEV
```

## Service Communication Flow

```mermaid
sequenceDiagram
    participant Client
    participant GA as Global Accelerator
    participant WAF as AWS WAF
    participant ALB as Load Balancer
    participant WS as WS-Proxy
    participant Redis as ElastiCache
    participant STT as STT API

    Note over Client,STT: WebSocket Streaming Flow

    Client->>GA: wss://api.aldea.ai/listen/v1
    GA->>WAF: Route to origin
    WAF->>ALB: Pass (if allowed)
    ALB->>ALB: Detect WebSocket header
    ALB->>WS: Forward to ws-proxy
    WS->>Redis: Validate API key
    Redis-->>WS: Key valid + rate limit OK
    WS->>STT: Open upstream connection

    loop Audio Streaming
        Client->>WS: Audio chunks
        WS->>STT: Forward audio
        STT-->>WS: Transcription results
        WS-->>Client: JSON results
    end

    WS->>Redis: Update usage metrics
```

## CI/CD Pipeline Flow

```mermaid
flowchart LR
    subgraph GitHub["GitHub Repositories"]
        PROXY[aldea-proxy-services]
        BACKEND[imp-backend]
        FRONTEND[imp-frontend]
        INFRA[aldea-eks-platform]
    end

    subgraph GHA["GitHub Actions"]
        BUILD[Build & Push to ECR]
        DEPLOY[Deploy to ECS]
    end

    subgraph AWS["AWS"]
        ECR[ECR Registry]
        ECS[ECS Service Update]
        ROLL[Rolling Deployment]
    end

    PROXY & BACKEND & FRONTEND -->|push to dev/main| BUILD
    BUILD -->|Docker image| ECR
    ECR --> DEPLOY
    DEPLOY -->|Update task definition| ECS
    ECS --> ROLL

    INFRA -->|Terraform| AWS
```
