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
