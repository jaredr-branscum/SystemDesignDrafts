# Video AI Generation System Design

## 1. Overview

This document outlines the system design for a scalable and resilient video AI generation system. The system allows customers to submit requests for video generation, which are processed asynchronously. The design emphasizes fault tolerance, performance, and cost-effectiveness.

## 2. Requirements

* **Functional:**
    * Customers should be able to submit AI video generation requests.
    * Customers should be able to track the status of their jobs.
    * The system should be resilient to failures.
* Out of Scope:
  * Real-time Video Editing
  * Live Video Streaming
  * Custom AI Model Upload
  * Direct Social Media Posting
  * User Management System
  * Support for multiple video formats (abstracted as something spooky)
* **Non-Functional:**
    * Scalability: The system should handle a large number of concurrent requests.
    * Reliability: The system should minimize data loss and ensure job completion.
    * Performance: The system should provide a responsive user experience.
    * Cost-effectiveness: The system should optimize resource utilization.
* Out of Scope:
  * On-Premise Deployment
  * Security Compliance
  * Adjusting video quality resolution for customer device (abstracted as something spooky)

## 3. Components

### Ingress & Gateway Layer
* **DNS & CDN**: Resolves client requests and caches video segment deliveries at the edge.
* **Ingress Load Balancer**: Distributes incoming HTTP and WebSocket traffic across gateway instances.
* **API Gateway**: Handles authentication, rate limiting, and routes requests to downstream services (e.g. video playback vs. job creation).

### Ingestion & State Layer
* **Request Ingestion Service**: Receives video generation requests, writes initial job metadata, and places them into the job ingestion queue.
* **SQL Primary DB**: The primary system of record for job metadata, customer settings, and durable workflow state.
* **Job Ingestion Queue**: Decouples incoming requests from down-stream execution nodes.
* **Job Workflow Coordinator**: A Temporal-style orchestrator managing the multi-stage pipeline state machine (Created -> Processing Chunks -> Compiling -> Transcoding -> Ready).

### GPU Execution Pool (Pull-Based)
* **GPU Task Queue**: Buffers generation tasks. GPU workers pull from this queue based on current concurrency limits.
* **GPU Generation Worker Pool**: Autoscaled container pool hosting specialized hardware instances running generative AI models to yield raw video chunks.
* **Autoscaling Controller (KEDA)**: Scale-to-zero/minimum metrics agent scaling workers based on the GPU Task Queue depth.
* **Model Cache / Registry**: Warm repository for local, fast weight loading across dynamic GPU workers.
* **NoSQL DB**: Captures worker-level state checkpoints to facilitate mid-generation resumes.
* **Object Storage (Chunks)**: Key-value object repository containing raw output video chunks.

### Compilation & Packaging Pipeline
* **Compilation Queue & Video Compiler Service**: Coordinates merging individual raw video chunks from object storage into standard raw source media.
* **Transcoding Queue & Video Transcoding Service**: Orchestrates FFmpeg / Adaptive Bitrate (ABR) pipelines to encode raw video into HLS/DASH manifests and segments.
* **Object Storage (HLS/DASH Segments)**: Durable repository housing streamable segment files and index playlists.

### Real-time Status Delivery
* **Redis (Live Job Status)**: Caches rapid progress percentages and heartbeats.
* **Notification & Event Service**: Translates progress events to external notification adapters.
* **WebSocket / SSE Gateway**: Maintains persistent connections with clients to deliver real-time percentage updates.
* **Third Party Channels**: External integration adapters for SMS, Email, and Webhook endpoints.

### Delivery & Playback
* **Video Playback Service**: Serves HLS playlist manifests to streaming clients.
* **Redis (Manifest Metadata)**: Caches resolved manifest locations to bypass SQL primary DB requests for active players.

---

## 4. System Design Diagram

```mermaid
%%{init: {
  'theme': 'default',
  'themeVariables': {
    'tertiaryColor': '#f5f5f7',
    'edgeLabelBackground': '#ffffff'
  }
}}%%
flowchart TD
    %% Styling Classes with explicit dark text colors for readability
    classDef client fill:#f5f5f7,stroke:#1d1d1f,stroke-width:2px,color:#1d1d1f;
    classDef lb fill:#fef7e0,stroke:#f9ab00,stroke-width:2px,color:#1d1d1f;
    classDef gateway fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px,color:#1d1d1f;
    classDef service fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px,color:#1d1d1f;
    classDef queue fill:#f3e8fd,stroke:#a142f4,stroke-width:2px,color:#1d1d1f;
    classDef database fill:#e6f4ea,stroke:#137333,stroke-width:2px,color:#1d1d1f;
    classDef external fill:#f1f3f4,stroke:#5f6368,stroke-width:2px,color:#1d1d1f;
    classDef thirdparty fill:#f1f3f4,stroke:#5f6368,stroke-width:2px,stroke-dasharray: 5 5,color:#1d1d1f;
    classDef orchestrator fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,color:#1d1d1f;

    Client["Client (Web/App)"]:::client
    DNS["DNS Lookup"]:::external
    CDN["CDN"]:::external
    LB_Ingress["Ingress Load Balancer"]:::lb
    APIGW["API Gateway<br/>(Auth & Rate Limiting)"]:::gateway

    subgraph Ingestion_Layer["Ingestion & State Layer"]
        ReqHandler["Request Ingestion Service"]:::service
        JobDb[("SQL Primary DB<br/>(Job Metadata & State)")]:::database
        JobQueue["Job Ingestion Queue"]:::queue
        Coordinator[["Job Workflow Coordinator<br/>(Temporal State Machine)"]]:::orchestrator
    end

    subgraph GPU_Generation_Layer["GPU Execution Pool (Pull-Based)"]
        TaskQueue["GPU Task Queue"]:::queue
        GenWorker["GPU Generation Worker Pool<br/>(Autoscaled)"]:::service
        Autoscaler["Autoscaling Controller (KEDA)"]:::gateway
        ModelCache[("Model Cache / Registry")]:::database
        NoSQL[("NoSQL DB<br/>(Worker Checkpoints)")]:::database
        ObjStore[("Object Storage<br/>(Chunks)")]:::database
    end

    subgraph Post_Processing_Layer["Compilation & Packaging Pipeline"]
        CompileQueue["Compilation Queue"]:::queue
        Compiler["Video Compiler Service"]:::service
        TranscodeQueue["Transcoding Queue"]:::queue
        Transcoder["Video Transcoding Service<br/>(FFmpeg / ABR)"]:::service
        ObjStoreOut[("Object Storage<br/>(HLS/DASH Segments)")]:::database
    end

    subgraph Notification_Layer["Real-time Status Delivery"]
        RedisStatus[("Redis<br/>(Live Job Status)")]:::database
        Notification["Notification & Event Service"]:::service
        WSAgent["WebSocket / SSE Gateway"]:::gateway
        ThirdParty["Third Party Channels<br/>(SMS, E-mail, Webhooks)"]:::thirdparty
    end

    subgraph Delivery_Layer["Delivery & Playback"]
        Playback["Video Playback Service"]:::service
        RedisPlayback[("Redis<br/>(Manifest Metadata)")]:::database
    end

    Client --> DNS
    Client --> LB_Ingress
    LB_Ingress --> APIGW
    Client <-->|Live Status| WSAgent

    APIGW -->|Submit Job| ReqHandler
    ReqHandler -->|Write State| JobDb
    ReqHandler -->|Enqueue| JobQueue
    JobQueue --> Coordinator
    Coordinator <-->|State R/W| JobDb

    Coordinator -->|Dispatch| TaskQueue
    TaskQueue -.->|Pull| GenWorker
    Autoscaler -->|Monitor Depth| TaskQueue
    Autoscaler -->|Scale| GenWorker
    GenWorker <-->|Load Model| ModelCache
    GenWorker <-->|Checkpoint| NoSQL
    GenWorker -->|Write Chunks| ObjStore
    GenWorker -->|Progress Event| RedisStatus

    RedisStatus -.->|Fan-out| Notification
    Notification --> WSAgent
    Notification --> ThirdParty

    GenWorker -->|Job Done| Coordinator
    Coordinator -->|Trigger Compile| CompileQueue
    CompileQueue -.->|Pull| Compiler
    ObjStore -->|Read Chunks| Compiler
    Compiler -->|Merged Video| ObjStoreOut

    Compiler -->|Compile Done| Coordinator
    Coordinator -->|Trigger Transcode| TranscodeQueue
    TranscodeQueue -.->|Pull| Transcoder
    ObjStoreOut -->|Read Video| Transcoder
    Transcoder -->|Write Segments| ObjStoreOut
    Transcoder -->|Register Manifest| JobDb
    Transcoder -->|Transcode Done| Coordinator

    APIGW -->|View Request| Playback
    Playback <-->|Manifest Lookup| RedisPlayback
    RedisPlayback <-->|Sync| JobDb
    Playback -->|HLS URL| Client
    Client -->|Stream| CDN
    CDN -->|Serve Segments| ObjStoreOut
```

