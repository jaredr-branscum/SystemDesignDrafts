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
* Request Handler
  * Enqueues requests for generating videos to the message queue
* Job Processing Service
  * Orchestrates spinning up jobs to produce videos and manages the job lifecycle
  * Retains the job status
* Video Generation Service
  * The specialized hardware instances that run generative AI models to generate videos
* Video Compiler Service
  * Assembles video chunks from object storage into media files that can be downloaded
  * Subscribed to Pub/Sub Topic
* Video Playback Service
  * Retrieves the viewable video files from Object Storage and other complex system components for supporting video that are out of scope
* Notification Service
  * Responsible for pushing notifications to third-party customer channels

## 4. System Design Diagram

```mermaid
flowchart TD
    %% Styling Classes
    classDef client fill:#f5f5f7,stroke:#1d1d1f,stroke-width:2px;
    classDef lb fill:#fef7e0,stroke:#f9ab00,stroke-width:2px;
    classDef gateway fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px;
    classDef service fill:#e8f0fe,stroke:#1a73e8,stroke-width:2px;
    classDef queue fill:#f3e8fd,stroke:#a142f4,stroke-width:2px;
    classDef database fill:#e6f4ea,stroke:#137333,stroke-width:2px;
    classDef external fill:#f1f3f4,stroke:#5f6368,stroke-width:2px;
    classDef thirdparty fill:#f1f3f4,stroke:#5f6368,stroke-width:2px,stroke-dasharray: 5 5;
    classDef cloud fill:#fce8e6,stroke:#ea4335,stroke-width:2px,stroke-dasharray: 5 5;

    %% Node Definitions
    Client["Client"]:::client
    DNS["DNS"]:::external
    CDN_Top["CDN"]:::external
    LB_Ingress["Load Balancer"]:::lb
    APIGW["API Gateway<br/>(Multiple Instances)"]:::gateway

    %% Creation Path
    LB_Req["Load Balancer"]:::lb
    ReqHandler["Request Handler<br/>(Multiple Instances)"]:::service
    MQ["Message Queue"]:::queue
    JobProc["Job Processing Service"]:::service
    SQL[("SQL Database<br/>(Job metadata & status)")]:::database
    LB_Gen["Load Balancer"]:::lb
    GenService["Video Generation Service<br/>(Hardware Specialized - Runs GenAI)<br/>(Multiple Instances)"]:::service
    NoSQL[("NoSQL Database<br/>(Video Generated Snapshot Data)")]:::database
    ObjStore[("Object Storage")]:::database

    %% Events & Notifications
    PubSub["Pub/Sub"]:::queue
    Notification["Notification Service"]:::service
    ThirdParty["Third Party Services<br/>(SMS, e-mail, etc.)"]:::thirdparty

    %% Compilation Path
    Compiler["Video Compiler Service<br/>(Multiple Instances)"]:::service

    %% Playback Path
    LB_Playback["Load Balancer"]:::lb
    Playback["Video Playback Service<br/>(Multiple Instances)"]:::service
    MemCache[("Memory Cache<br/>(Video Metadata)")]:::database
    CDN_Left["CDN"]:::external

    %% Spooky Transcoding Cloud (Using stadium shape for maximum compatibility with markdown renderers)
    Spooky(["Spooky Video Transcoding Magic /<br/>Netflix System Design Model 😂😂😂😂😂😂"]):::cloud

    %% Connections
    Client --> LB_Ingress
    Client --> DNS
    Client --> CDN_Top
    LB_Ingress --> APIGW

    %% Ingress Routing
    APIGW -->|"Request to create AI Video"| LB_Req
    APIGW -->|"View Generated Video"| LB_Playback

    %% Creation Flow
    LB_Req --> ReqHandler
    ReqHandler --> MQ
    MQ --> JobProc
    JobProc <--> SQL
    JobProc --> LB_Gen
    LB_Gen --> GenService
    GenService <--> NoSQL
    GenService <-->|"Read/Write Generated Video Chunks"| ObjStore
    GenService -->|"Job Success/Failure Event"| PubSub

    %% Event Loops
    PubSub -->|"Job Status Event"| JobProc
    PubSub -->|"Send Customer Notifications"| Notification
    Notification --> ThirdParty

    %% Compilation Flow
    PubSub -->|"Video Generation Completed"| Compiler
    ObjStore -->|"Assembles Video Chunks into downloadable media"| Compiler

    %% Playback Flow
    LB_Playback --> Playback
    Playback <-->|"Video metadata"| MemCache
    Playback --> CDN_Left
    Playback -->|"Transcoded Video Request"| Spooky
    ObjStore --> Spooky
    Spooky --> Playback
```

