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

### Ingress
* **API Gateway**
  * Authenticates requests and enforces rate limits
  * Routes job submission to the Ingestion layer and video view requests to the Playback layer

### Ingestion & State Layer
* **Request Ingestion Service**
  * Validates and persists incoming job requests to the SQL Primary DB
  * Enqueues a job task to the Job Ingestion Queue
* **SQL Primary DB**
  * Single source of truth for job metadata and workflow state across all pipeline stages
* **Job Ingestion Queue**
  * Decouples the HTTP request path from the orchestration layer
* **Job Workflow Coordinator (Temporal State Machine)**
  * Manages the full job lifecycle: Created → Generating → Compiling → Transcoding → Ready
  * Dispatches tasks to downstream queues and handles retries on stage failure

### GPU Execution Pool (Pull-Based)
* **GPU Task Queue**
  * Buffers pending generation tasks; workers pull when GPU capacity is available
  * Enables SQS-style visibility timeout for automatic retry on worker crash
* **GPU Generation Worker Pool**
  * Autoscaled pool of GPU instances that pull tasks and run the generative AI model
  * Streams output chunks to Object Storage and emits progress events to Redis
* **Autoscaling Controller (KEDA)**
  * Monitors GPU Task Queue depth and scales the worker pool up or down accordingly
* **Model Warm Cache / Registry**
  * Stores pre-loaded AI model weights to eliminate cold-start latency on new worker instances
* **NoSQL DB (Worker Checkpoints)**
  * Persists in-progress generation snapshots so interrupted jobs can resume from the last checkpoint
* **Object Storage (Raw Chunks)**
  * Stores raw video chunks produced by the generation workers

### Compilation & Packaging Pipeline
* **Compilation Queue**
  * Buffers compile jobs; triggered by the Coordinator once all generation chunks are confirmed
* **Video Compiler Service**
  * Pulls chunk lists from Object Storage and merges them into a single contiguous video file
* **Transcoding Queue**
  * Buffers transcode jobs; triggered by the Coordinator once compilation succeeds
* **Video Transcoding Service (FFmpeg / ABR Packaging)**
  * Transcodes the compiled video into multi-bitrate HLS/DASH adaptive streaming segments
  * Registers the output manifest URL in the SQL Primary DB on completion
* **Object Storage (HLS/DASH Segments)**
  * Stores the final adaptive bitrate video segments and manifests served to the CDN

### Real-time Status Delivery
* **Redis Cache (Live Status & Progress)**
  * Holds per-job progress state written by GPU workers; fan-out source for the Notification Service
* **Notification & Event Service**
  * Consumes status events from Redis and fans out to the WebSocket gateway and third-party channels
* **WebSocket / SSE Gateway**
  * Maintains persistent connections with clients and pushes real-time job progress updates
* **Third Party Services (SMS, E-mail, Webhooks)**
  * External notification channels for job completion or failure alerts

### Delivery & Playback
* **Video Playback Service**
  * Resolves the HLS manifest URL for a completed job and returns it to the client
* **Redis Metadata Cache**
  * Caches manifest URLs and job completion metadata to reduce SQL read load on playback requests
* **CDN (Akamai/Cloudflare)**
  * Caches and serves HLS/DASH segment files at the edge for low-latency global video streaming

## 4. System Design Diagram

```mermaid
%%{
  init: {
    "theme": "base",
    "themeVariables": {
      "background": "#ffffff",
      "primaryColor": "#e8f0fe",
      "primaryTextColor": "#1d1d1f",
      "primaryBorderColor": "#1a73e8",
      "lineColor": "#5f6368",
      "secondaryColor": "#f3e8fd",
      "tertiaryColor": "#f5f5f7",
      "edgeLabelBackground": "#ffffff"
    }
  }
}%%
flowchart TD
    %% Styling Classes with explicit dark text colors for high readability
    classDef client fill:#f5f5f7,stroke:#1d1d1f,stroke-width:2px,color:#1d1d1f;
    classDef lb fill:#fef7e0,stroke:#f9ab00,stroke-width:2px,color:#1d1d1f;
    classDef gateway fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px,color:#1d1d1f;
    classDef service fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px,color:#1d1d1f;
    classDef queue fill:#f3e8fd,stroke:#a142f4,stroke-width:2px,color:#1d1d1f;
    classDef database fill:#e6f4ea,stroke:#137333,stroke-width:2px,color:#1d1d1f;
    classDef external fill:#f1f3f4,stroke:#5f6368,stroke-width:2px,color:#1d1d1f;
    classDef thirdparty fill:#f1f3f4,stroke:#5f6368,stroke-width:2px,stroke-dasharray: 5 5,color:#1d1d1f;
    classDef orchestrator fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,color:#1d1d1f;

    %% Node Definitions
    Client["Client (Web/App)"]:::client
    DNS["DNS Lookup"]:::external
    CDN["CDN (Akamai/Cloudflare)"]:::external
    LB_Ingress["Ingress Load Balancer"]:::lb
    APIGW["API Gateway (Auth & Rate Limiting)"]:::gateway

    subgraph Ingestion_Layer["Ingestion & State Layer"]
        ReqHandler["Request Ingestion Service"]:::service
        JobDb[("SQL Primary DB<br/>(Job Metadata & Workflow State)")]:::database
        JobQueue["Job Ingestion Queue"]:::queue
        Coordinator[["Job Workflow Coordinator<br/>(Temporal State Machine)"]]:::orchestrator
    end

    subgraph GPU_Generation_Layer["GPU Execution Pool (Pull-Based)"]
        TaskQueue["GPU Task Queue (e.g. SQS/RabbitMQ)"]:::queue
        GenWorker["GPU Generation Worker Pool<br/>(Autoscaled)"]:::service
        Autoscaler["Autoscaling Controller (KEDA)"]:::gateway
        ModelCache[("Model Warm Cache / Registry")]:::database
        NoSQL[("NoSQL DB<br/>(Worker Checkpoints)")]:::database
        ObjStore[("Object Storage<br/>(Raw Chunks)")]:::database
    end

    subgraph Post_Processing_Layer["Compilation & Packaging Pipeline"]
        CompileQueue["Compilation Queue"]:::queue
        Compiler["Video Compiler Service"]:::service
        TranscodeQueue["Transcoding Queue"]:::queue
        Transcoder["Video Transcoding Service<br/>(FFmpeg / ABR Packaging)"]:::service
        ObjStoreOut[("Object Storage<br/>(HLS/DASH Segments)")]:::database
    end

    subgraph Notification_Layer["Real-time Status Delivery"]
        RedisStatus[("Redis Cache<br/>(Live Status & Progress)")]:::database
        Notification["Notification & Event Service"]:::service
        WSAgent["WebSocket / SSE Gateway"]:::gateway
        ThirdParty["Third Party Services<br/>(SMS, E-mail, Webhooks)"]:::thirdparty
    end

    subgraph Delivery_Layer["Delivery & Playback"]
        Playback["Video Playback Service"]:::service
        RedisPlayback[("Redis Metadata Cache")]:::database
    end

    %% Client entry
    Client -->|1. Resolve| DNS
    Client -->|2. Request HTTP| LB_Ingress
    LB_Ingress --> APIGW
    Client <-->|Live Updates| WSAgent

    %% Ingestion Flow
    APIGW -->|3. Submit Job| ReqHandler
    ReqHandler -->|4. Write Pending State| JobDb
    ReqHandler -->|5. Enqueue| JobQueue
    JobQueue --> Coordinator
    Coordinator <-->|Manage State| JobDb

    %% Generation Flow
    Coordinator -->|6. Dispatch Tasks| TaskQueue
    TaskQueue -.->|7. Pull Task| GenWorker
    Autoscaler -->|Monitor Queue Depth| TaskQueue
    Autoscaler -->|Scale Workers| GenWorker
    GenWorker <-->|Load AI Model| ModelCache
    GenWorker <-->|Save Snapshots| NoSQL
    GenWorker -->|8. Upload Chunks| ObjStore
    GenWorker -->|9. Update Progress| RedisStatus

    %% Real-time update push
    RedisStatus -.->|Sync| Notification
    Notification --> WSAgent
    Notification --> ThirdParty

    %% Compilation Flow
    GenWorker -->|10. Task Completed| Coordinator
    Coordinator -->|11. Trigger Merge| CompileQueue
    CompileQueue -.->|Pull| Compiler
    ObjStore -->|12. Read Chunks| Compiler
    Compiler -->|13. Upload Raw Video| ObjStoreOut

    %% Transcoding Flow
    Compiler -->|14. Success Event| Coordinator
    Coordinator -->|15. Trigger Transcode| TranscodeQueue
    TranscodeQueue -.->|Pull| Transcoder
    ObjStoreOut -->|16. Read Raw Video| Transcoder
    Transcoder -->|17. Upload HLS/DASH Segments| ObjStoreOut
    Transcoder -->|18. Register Manifest| JobDb
    Transcoder -->|19. Complete Event| Coordinator

    %% Playback Flow
    APIGW -->|View Video Request| Playback
    Playback <-->|Fetch manifest link| RedisPlayback
    RedisPlayback <-->|Sync metadata| JobDb
    Playback -->|Return HLS Playback URL| Client
    Client -->|Stream Video Segments| CDN
    CDN -->|Cache & Serve| ObjStoreOut
```
