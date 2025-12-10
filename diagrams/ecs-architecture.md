# Aldea ECS Architecture

## What is This?

This document shows how Aldea's speech-to-text (ASR) services run in the cloud using AWS ECS (Elastic Container Service). Think of ECS as a way to run applications in isolated containers without managing servers.

---

## How Users Connect to Aldea

```mermaid
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

## Tasks Distributed Across Availability Zones

AWS runs services across multiple data centers (Availability Zones) for reliability. If one zone fails, others keep running.

```mermaid
flowchart TB
    subgraph ALB["Load Balancer"]
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

        subgraph Data["Shared Data Layer"]
            REDIS[(Redis<br/>Multi-AZ)]
            RDS[(Database<br/>Multi-AZ)]
        end
    end

    LB --> SubA & SubB & SubC
    SubA & SubB --> REDIS
    BE_A & BE_B --> RDS
```

---

## WebSocket Flow - Live Audio Streaming

For real-time transcription where audio streams continuously:

```mermaid
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

    rect rgb(240, 248, 255)
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

## Dev Environment

A smaller version for testing and development:

```mermaid
flowchart TB
    subgraph Domains["Dev Domains"]
        D1[dev-api.aldea.ai]
        D2[dev-backend.aldea.ai]
        D3[dev-platform.aldea.ai]
    end

    subgraph ECS["Dev ECS Cluster"]
        subgraph AZ_A["Zone A"]
            WS[WebSocket Proxy]
            TRANS[HTTP Transcribe]
        end
        subgraph AZ_B["Zone B"]
            BE[Backend API]
            FE[Frontend Web]
        end
    end

    subgraph Backend["Backend Services"]
        REDIS[Redis Cache]
        STT[Speech Engine]
        DB[(Staging Database)]
    end

    D1 --> WS & TRANS
    D2 --> BE
    D3 --> FE

    WS & TRANS --> REDIS --> STT
    BE --> DB
```

---

## How Code Gets Deployed

When developers push code changes:

```mermaid
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
