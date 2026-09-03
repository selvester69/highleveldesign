# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: YouTube Processor - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-24
- **Author(s)**: Jules (AI Software Engineer)
- **Reviewers**: Architecture Review Board, Engineering Leads
- **Approvers**: VP of Engineering, Chief Architect
- **Document Status**: Draft

### Revision History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-08-24 | Jules | Initial creation of YouTube Processor HLD |

### Distribution List
- Media Infrastructure Team
- Platform Engineering Team
- Operations & SRE Team
- Product Management

---

## 2. Executive Summary

### 2.1 Purpose
The YouTube Processor is a distributed, high-throughput, fault-tolerant video processing platform. It ingests raw user-uploaded video files, orchestrates parallel chunking and encoding pipelines, generates multi-bitrate adaptive streams (HLS/DASH), extracts audio for automatic speech recognition (ASR) closed-captioning, applies watermarks/thumbnails, and performs automated content moderation and copyright fingerprinting before publishing to global CDNs.

### 2.2 Scope
**In Scope:**
- Resumable multi-part upload of high-definition/4K/8K video files.
- Video chunking and DAG-driven asynchronous workflow execution.
- Multi-codec encoding (H.264, H.265/HEVC, VP9, AV1) and multi-resolution adaptive bitrate generation.
- Automated thumbnail extraction, AI frame selection, and dynamic watermarking.
- Audio extraction, noise reduction, and AI-powered closed-caption / subtitle generation.
- Content moderation (NSFW detection, toxicity scanning) and digital fingerprinting (Content ID matching).
- Real-time processing status updates via WebSockets and webhooks.
- Asset lifecycle management, multi-tiered cold/hot storage, and CDN integration.

**Out of Scope:**
- Real-time live-streaming ingestion (e.g., RTMP live streaming).
- Frontend video player client UI development (focus is backend processing and delivery API).
- Payment processing and subscription management.

### 2.3 Key Benefits
- **Sub-linear Processing Time**: Video chunking allows parallel transcoding, reducing total processing latency for a 1-hour 4K video from hours to under 3 minutes.
- **Cost & Bandwidth Efficiency**: Advanced codecs (AV1/VP9) reduce bandwidth consumption by up to 30-50% while preserving visual quality.
- **Robust Fault Tolerance**: Idempotent chunk-level processing ensures failed worker tasks resume from the last completed chunk without re-processing the entire file.
- **Automated Compliance & Rights Management**: Automated Content ID fingerprinting and visual/audio moderation prevent copyright infringement and inappropriate content distribution prior to public playback.

### 2.4 High-Level Architecture Overview
The YouTube Processor separates the upload/ingestion plane, asynchronous DAG processing orchestration plane, distributed compute execution plane, and metadata/delivery plane. Uploads occur via direct pre-signed S3 multi-part uploads. Upon upload completion, an event triggers the **Video DAG Orchestrator**, which decomposes the video into GOP-aligned (Group of Pictures) chunks. Independent worker pools process chunks in parallel across encoding, audio extraction, thumbnail generation, AI transcription, and fingerprinting tasks. Aggregated results are consolidated into master HLS/DASH manifests (`.m3u8` / `.mpd`) and pushed to origin storage backed by a global CDN.

---

## 3. System Overview

### 3.1 Business Context
Video content represents over 80% of consumer internet traffic. Platforms ingesting user-generated content (UGC) must process petabytes of raw video uploaded daily across diverse device types, aspect ratios, network conditions, and resolutions. The YouTube Processor solves the computational, storage, and copyright challenges of scale.

### 3.2 System Purpose
To provide an enterprise-grade, elastic video processing system that reliably ingests raw media files of arbitrary length and format, transforms them into optimized streaming formats, enforces content policies, and makes streams available globally with minimal delay.

### 3.3 Success Criteria
- **Processing SLA**: 95% of uploaded videos available for streaming within 1.5x of the video duration (e.g., a 10-minute video processed in under 15 minutes; chunked parallel paths < 2 minutes).
- **Upload Reliability**: 99.99% success rate for chunked resumable uploads up to 100 GB.
- **Streaming Quality**: Support 8K, 4K, 1080p, 720p, 480p, 360p, 240p in HLS and DASH.
- **Copyright & Moderation Latency**: Complete Content ID scanning within 60 seconds of upload completion.
- **System Availability**: 99.95% control plane uptime; 99.999% CDN playback availability.

### 3.4 Assumptions
- Cloud object storage (e.g., AWS S3, Google Cloud Storage) provides high durability (99.999999999%) and multi-part upload support.
- Specialized compute workloads (e.g., FFmpeg encoding, AI ML inference) utilize GPU-accelerated worker instances (NVIDIA T4/A10G or AWS Inferentia).
- Client devices support standard HLS and MPEG-DASH streaming specifications.

### 3.5 Constraints
- High compute cost during peak upload spikes requires dynamic spot instance scaling and queue management.
- Frame-exact video chunking must occur strictly on Keyframes / I-frames (GOP boundaries) to avoid visual artifacts during segment concatenation.

### 3.6 Dependencies
- Cloud Object Storage (S3 / GCS) for raw, intermediate, and processed assets.
- Distributed Message Queue & Event Stream (Apache Kafka / AWS SQS).
- Relational Metadata DB (PostgreSQL) + NoSQL Cache (Redis).
- AI ML Services (Whisper/ASR for transcriptions, ResNet/Vision Transformers for NSFW detection, Acoustic Fingerprinting libraries).

---

## 4. Requirements Analysis

### 4.1 Functional Requirements

#### 4.1.1 Core Features
- **FR-001**: **Resumable Multi-part Upload**: Allow clients to upload raw video files up to 100 GB using signed parallel chunk uploads.
- **FR-002**: **GOP-aligned Video Chunking**: Automatically slice raw video files into 5-second to 10-second chunks on I-frame boundaries.
- **FR-003**: **Parallel Multi-Format Transcoding**: Transcode chunks into H.264, HEVC, VP9, and AV1 formats across resolutions (240p to 8K) and variable bitrates.
- **FR-004**: **Manifest Generation**: Dynamically construct HLS (`master.m3u8`) and DASH (`manifest.mpd`) manifest files linking all transcoded stream segments.
- **FR-005**: **Thumbnail & Preview GIF Generation**: Extract static thumbnails at regular time intervals, select best-frame thumbnail using AI scoring, and generate 5-second animated preview GIFs/WebPs.
- **FR-006**: **Dynamic Watermarking**: Burn-in or overlay customizable user/brand watermarks or invisible digital watermarking.
- **FR-007**: **Audio Extraction & Speech-to-Text Transcription**: Extract audio tracks, run noise cancellation, and generate VTT/SRT closed captions using AI speech recognition across multiple languages.
- **FR-008**: **Content Moderation & Copyright Fingerprinting**: Generate video/audio digital signatures and check against a copyright database; flag adult, violent, or hate speech content.
- **FR-009**: **Real-Time Notification & Status Tracking**: Stream processing milestones and final completion/rejection events via WebSockets and webhooks.

#### 4.1.2 User Stories
- **US-001**: As a creator, I want to upload a 4K 60fps video so that my viewers can stream it in highest quality on smart TVs and low bitrates on mobile phones.
- **US-002**: As a creator, I want auto-generated closed captions so that my content is accessible without manually creating subtitle files.
- **US-003**: As a content owner, I want automated copyright detection so that stolen re-uploaded videos are flagged instantly.
- **US-004**: As a developer, I want webhook notifications when processing completes so that my platform can publish the video immediately.

### 4.2 Non-Functional Requirements

#### 4.2.1 Performance Requirements
- **Upload Throughput**: Support up to 10,000 concurrent video uploads globally.
- **Transcoding Parallelism**: Process individual 10-second video chunks within 15 seconds.
- **Metadata Query Latency**: < 20 ms (p99) for video playback metadata requests.

#### 4.2.2 Scalability Requirements
- Scale storage horizontally to hundreds of petabytes.
- Auto-scale transcode worker pool from 100 to 10,000+ GPU/CPU instances based on queue depth.

#### 4.2.3 Availability Requirements
- Processing pipeline availability: 99.9%.
- Storage & Delivery availability: 99.999%.

#### 4.2.4 Security Requirements
- End-to-end encryption (TLS 1.3 in transit, AES-256 at rest).
- Signed URLs / Signed Cookies for private content access.
- Role-Based Access Control (RBAC) and OAuth 2.0 API authentication.

#### 4.2.5 Usability Requirements
- Clear processing progress metrics (e.g., "Transcoding 65% [720p complete, 4K in progress]").

#### 4.2.6 Reliability Requirements
- Automatic retry with exponential backoff for failed chunk processing jobs.
- Dead Letter Queue (DLQ) for non-retryable malformed video chunks.

---

## 5. Architecture Design

### 5.1 Architecture Principles
- **Asynchronous & Event-Driven**: Decouple ingestion, orchestration, execution, and delivery via messaging topologies.
- **Stateless Compute Workers**: All processing nodes pull tasks from queues, fetch chunks from object storage, and push output without holding local state.
- **Data Locality & Caching**: Cache hot video segments near the network edge using a tiered CDN strategy.
- **Idempotency**: Re-executing any job or chunk process produces identical outputs without side effects.

### 5.2 Architecture Patterns
- **Saga / DAG Orchestration Pattern**: Manage multi-stage asynchronous processing workflows.
- **MapReduce / Fan-Out Fan-In Pattern**: Fan-out video chunk processing across hundreds of nodes, then fan-in to merge manifest files and trigger publication.
- **Microservices Architecture**: Separate services for Upload, DAG Control, Transcoding, AI Processing, Moderation, and Delivery.

### 5.3 High-Level Architecture Diagram

```text
                                +-----------------------------+
                                |      Client Application     |
                                +--------------+--------------+
                                               |
                                (1. Init Upload / Get Signed URLs)
                                               v
                                +--------------+--------------+
                                |      API Gateway / Auth     |
                                +--------------+--------------+
                                               |
                                               v
+------------------+ (2. Direct Upload) +------+----------------------+
| Raw Video Bucket | <----------------- | Upload Service              |
|  (Amazon S3)     |                    +--------------+--------------+
+--------+---------+                                   |
         | (3. Object Created Event)                   v
         v                                      +------+----------------------+
+--------+---------+                            | Metadata DB (PostgreSQL)    |
| Event Bus / Kafka|                            +-----------------------------+
+--------+---------+
         |
         +---------------------------------------+
         |                                       |
         v                                       v
+--------+--------------------+        +---------+--------------------+
| Video DAG Orchestrator      |        | Content ID & Moderation    |
| (Temporal.io / Custom DAG)  |        | Service (GPU Workers)       |
+--------+--------------------+        +------------------------------+
         |
         | (4. Dispatch Chunk Jobs)
         v
+--------+-----------------------------------------------------------------------------------+
|                               Distributed Processing Queue (Kafka / SQS)                         |
+---+----------------------------+-----------------------------+----------------------------+
    |                            |                             |                            |
    v                            v                             v                            v
+---+------------------+ +-------+-------------------+ +-------+-------------------+ +------+--------------------+
| Transcoding Workers  | | Thumbnail & Watermark     | | Audio & Subtitle ASR      | | Manifest Synthesizer    |
| (FFmpeg / NVENC / AV1| | Workers (OpenCV / PIL)    | | Workers (Whisper AI)    | | (HLS / DASH Builder)     |
+---+------------------+ +-------+-------------------+ +-------+-------------------+ +------+--------------------+
    |                            |                             |                            |
    +----------------------------+-----------------------------+----------------------------+
                                 | (5. Write Artifacts)
                                 v
                     +-----------+------------------+
                     | Processed Assets Bucket      |
                     | (S3 / CDN Origin)            |
                     +-----------+------------------+
                                 |
                                 v
                     +-----------+------------------+
                     | Global CDN Network           |
                     | (Cloudflare / Fastly)        |
                     +-----------+------------------+
                                 |
                                 v
                     +-----------+------------------+
                     | End-User Video Player        |
                     +------------------------------+
```

### 5.4 Component Overview
1. **Upload Service**: Manages multi-part upload sessions, issues pre-signed S3 URLs, validates upload integrity (MD5/SHA256 checksums), and updates upload state.
2. **Video DAG Orchestrator**: Maintains the execution state of processing pipelines. Splits video into GOP chunks, tracks task dependencies, handles retries, and coordinates fan-in completion.
3. **Transcoding & Encoding Engine**: A cluster of containerized workers executing FFmpeg and hardware-accelerated encoders (NVENC/QuickSync) to generate H.264, HEVC, VP9, and AV1 renditions.
4. **Thumbnail & Watermark Processor**: Generates preview sprites, poster images, dynamic overlays, and visual watermarks.
5. **Audio & Subtitle Processor**: Extracts multi-track audio, executes noise suppression, and runs deep-learning speech recognition (Whisper AI) to produce synchronized subtitle files (`.vtt`, `.srt`).
6. **Content Moderation & Copyright Service**: Extracts acoustic and visual feature vectors (fingerprints), matches against the database, and inspects video frames for safety compliance.
7. **Manifest Synthesizer**: Merges output segment references into HLS master playlists (`master.m3u8`) and DASH MPD manifests.
8. **Notification Service**: Sends Webhook events, SSE, or WebSocket messages notifying client applications of status changes.

### 5.5 Technology Stack

#### 5.5.1 Frontend / Client Integration
- Protocols: REST API, WebSockets, gRPC.
- SDKs: JavaScript/TypeScript, Swift, Kotlin.

#### 5.5.2 Backend Services
- **Languages**: Go (high-concurrency control plane, API Gateway), C++ / Rust (media tools wrapper, custom FFmpeg extensions), Python (AI ML models).
- **Frameworks**: Gin / gRPC-Go, PyTorch, Temporal.io Go SDK.
- **Media Libraries**: FFmpeg, libvpx, libaom (AV1), OpenCV, GPU NVENC/NVDEC drivers.

#### 5.5.3 Database & Cache
- **Primary Relational DB**: PostgreSQL 16 (partitioned tables for video metadata, job states).
- **Cache & Real-time State**: Redis Cluster (session state, upload chunk tracking, pub/sub).
- **Vector / Fingerprint Search DB**: Milvus or pgvector for high-dimensional video/audio fingerprint matching.

#### 5.5.4 Infrastructure & Middleware
- **Message Bus**: Apache Kafka (high-throughput event streaming).
- **Containerization & Orchestration**: Docker, Kubernetes (K8s) with Karpenter / KEDA for custom metric auto-scaling.
- **Storage**: Amazon S3 / Ceph Object Storage.
- **CDN**: Cloudflare / Fastly / AWS CloudFront.

### 5.6 Architecture Decision Records (ADRs)

#### 5.6.1 ADR-001: Keyframe / GOP-Based Chunking Strategy
- **Status**: Accepted
- **Context**: Slicing arbitrary raw video files at arbitrary timestamps causes broken Group of Pictures (GOP), resulting in visual corruption or requirement of slow re-encoding at boundaries.
- **Decision**: Use `ffprobe` to detect keyframe (I-frame) locations and slice raw videos strictly at GOP boundaries (typically every 2–10 seconds).
- **Consequences**: Video chunks can be independently encoded without cross-chunk frame dependencies, enabling true parallel transcode fan-out.

#### 5.6.2 ADR-002: Workflow Orchestration Engine Selection
- **Status**: Accepted
- **Context**: A custom state machine in database tables is prone to race conditions, retry bugs, and difficult dependency tracking.
- **Decision**: Adopt Temporal.io as the DAG workflow orchestrator.
- **Consequences**: Guarantees durable execution, automatic state management, robust retry policies, and clear visibility into complex pipeline executions.

---

## 6. Detailed Component Design

### 6.1 Component 1: API Gateway & Upload Service

#### 6.1.1 Purpose
Provides client authentication, rate limiting, upload session management, and issues pre-signed URLs for parallel direct-to-S3 multi-part uploads.

#### 6.1.2 Responsibilities
- Authenticate upload requests via JWT/OAuth2.
- Generate multi-part S3 pre-signed upload URLs (e.g., 5MB–50MB chunk size).
- Track chunk upload completion and verify checksums.
- Finalize multi-part upload and issue `VideoIngested` event.

#### 6.1.3 Interfaces
- **Inputs**: HTTP POST `/api/v1/videos/upload-session`
- **Outputs**: Upload Session ID, List of Signed URLs, Kafka `video.upload.completed` message.

#### 6.1.4 Internal Design
Stateless Go service backed by Redis for multi-part chunk tracking. Communicates with AWS S3 API via AWS SDK Go v2.

#### 6.1.5 Dependencies
PostgreSQL, Redis, AWS S3.

---

### 6.2 Component 2: Video DAG Orchestrator & Chunking Engine

#### 6.2.1 Purpose
Ingests `video.upload.completed` events, inspects media container properties, executes Keyframe-aligned chunking, and spawns the parallel workflow graph.

#### 6.2.2 Responsibilities
- Probe raw file metadata (`ffprobe` for codec, resolution, frame rate, GOP distribution, audio channels).
- Generate a Chunking Plan (e.g., 100 chunks for a 1000-second video).
- Emit individual task messages (`transcode.chunk`, `extract.audio`, `generate.thumbnails`, `fingerprint.content`).
- Monitor completion of all child tasks; trigger `synthesize.manifest` upon completion.

#### 6.2.3 Workflow Execution Graph
```text
                  +------------------------+
                  |  Raw Video Uploaded    |
                  +-----------+------------+
                              |
                              v
                  +-----------+------------+
                  |  Probe Metadata &      |
                  |  GOP-Aligned Chunking  |
                  +-----------+------------+
                              |
        +---------------------+---------------------+---------------------+
        |                     |                     |                     |
        v                     v                     v                     v
+-------+-------+     +-------+-------+     +-------+-------+     +-------+-------+
| Transcode     |     | Audio         |     | Thumbnail &   |     | Content ID    |
| Chunks (1..N) |     | Extraction &  |     | Preview GIF   |     | Fingerprint   |
| (Parallel)    |     | AI Speech ASR |     | Generation    |     | & Moderation  |
+-------+-------+     +-------+-------+     +-------+-------+     +-------+-------+
        |                     |                     |                     |
        +---------------------+---------------------+---------------------+
                              |
                              v
                  +-----------+------------+
                  |  Manifest Synthesizer   |
                  |  & Publication Event   |
                  +------------------------+
```

---

### 6.3 Component 3: Transcoding & Encoding Workers

#### 6.3.1 Purpose
Processes individual video chunks into target codecs, resolutions, and bitrates.

#### 6.3.2 Responsibilities
- Pull chunk task from Kafka topic `transcode.chunk.jobs`.
- Download raw chunk from S3 staging area.
- Transcode chunk using FFmpeg pipeline into multiple outputs:
  - **AV1** (high efficiency, 4K/1080p)
  - **VP9** (web compatibility, 4K/1080p/720p)
  - **HEVC / H.265** (mobile / TV compatibility)
  - **H.264 / AVC** (legacy fallback)
- Upload transcoded segment files (`.ts` / `.m4s`) to Processed S3 Bucket.
- Report chunk completion to Orchestrator.

---

### 6.4 Component 4: Thumbnail & Watermark Generator

#### 6.4.1 Purpose
Creates static poster images, scrub previews (storyboards/sprites), animated GIFs/WebP, and applies visual watermarks.

#### 6.4.2 Responsibilities
- Capture frame images at 1-second intervals.
- Generate a 10x10 storyboard tile sprite (`sprite_1.jpg`) and corresponding VTT track for player seek previews.
- Run AI quality scoring model (Laplacian variance for blurriness, aesthetics model) to pick the best auto-thumbnail.
- Overlay dynamic text/image watermarks using FFmpeg video filters.

---

### 6.5 Component 5: Audio & Subtitle ASR Processor

#### 6.5.1 Purpose
Extracts audio tracks, normalizes volume, runs AI speech recognition, and produces multi-language subtitles.

#### 6.5.2 Responsibilities
- Extract audio into AAC / Opus format.
- Run voice activity detection (VAD) and Whisper AI model to generate timestamped text transcription.
- Format transcripts into WebVTT (`.vtt`) and SubRip (`.srt`) files.
- Translate subtitles into target languages using Machine Translation API.

---

### 6.6 Component 6: Content Moderation & Copyright Fingerprinting Service

#### 6.6.1 Purpose
Protects platform compliance by identifying copyright violations and policy-violating content.

#### 6.6.2 Responsibilities
- Compute Chromaprint / AcoustID audio fingerprints for audio streams.
- Extract visual video frame feature vectors (using CNN/Vision Transformer) at 1 fps.
- Query Milvus vector database for Content ID matches against proprietary database.
- Scan frames with NSFW / Violence image classification model.
- If violation confidence > 0.90, flag video state as `BLOCKED_COPYRIGHT` or `FLAGGED_MODERATION`.

---

### 6.7 Component 7: Manifest Synthesizer & Publication Service

#### 6.7.1 Purpose
Combines transcoded chunk segments into standard HLS and DASH streaming manifests.

#### 6.7.2 Responsibilities
- Collect all segment paths (`chunk_001_1080p.m4s`, `chunk_002_1080p.m4s`, etc.).
- Generate Master Playlist (`master.m3u8`) and Media Playlists (`1080p.m3u8`, `720p.m3u8`, `audio.m3u8`).
- Generate DASH MPD manifest (`manifest.mpd`).
- Write manifests to Processed S3 Bucket and purge CDN cache for updated manifests.
- Mark video status as `READY_FOR_PLAYBACK`.

---

## 7. Data Design

### 7.1 Data Architecture
A polyglot persistence design:
- **PostgreSQL**: Transactional storage for video metadata, job states, user ownership, and ACLs.
- **Redis**: Fast caching, active upload session tracking, and real-time processing metrics.
- **Milvus / Vector Index**: High-dimensional vector embeddings for Content ID matching.
- **Object Storage (S3)**: Immutable media storage (Raw Bucket, Staging Bucket, Public Processed Bucket).

### 7.2 Data Models

```sql
-- Video Master Table
CREATE TABLE videos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    raw_s3_key VARCHAR(512) NOT NULL,
    duration_seconds NUMERIC(10, 2),
    resolution_width INT,
    resolution_height INT,
    frame_rate NUMERIC(5, 2),
    aspect_ratio VARCHAR(16),
    status VARCHAR(32) NOT NULL DEFAULT 'PENDING_UPLOAD',
    master_manifest_url VARCHAR(512),
    thumbnail_url VARCHAR(512),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Processing Jobs DAG Table
CREATE TABLE processing_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    workflow_id VARCHAR(128) NOT NULL,
    total_chunks INT NOT NULL,
    completed_chunks INT NOT NULL DEFAULT 0,
    status VARCHAR(32) NOT NULL DEFAULT 'PROCESSING',
    error_message TEXT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at TIMESTAMPTZ
);

-- Video Renditions Table
CREATE TABLE video_renditions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    codec VARCHAR(16) NOT NULL, -- 'h264', 'hevc', 'vp9', 'av1'
    resolution VARCHAR(16) NOT NULL, -- '1080p', '720p', '4k'
    bitrate_kbps INT NOT NULL,
    playlist_s3_key VARCHAR(512) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Subtitles & Transcripts Table
CREATE TABLE video_subtitles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    language_code VARCHAR(8) NOT NULL,
    subtitle_s3_key VARCHAR(512) NOT NULL,
    is_auto_generated BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 8. API Design

### 8.1 API Architecture
RESTful JSON API for management and control plane operations; gRPC for internal microservice communication; WebSockets for status updates.

### 8.2 API Specifications

#### 8.2.1 Initiate Upload Session
- **Method**: `POST`
- **Path**: `/api/v1/videos/upload`
- **Headers**: `Authorization: Bearer <JWT>`
- **Request Body**:
```json
{
  "title": "My High Definition Vlog",
  "description": "4K video upload test",
  "file_size_bytes": 10737418240,
  "file_name": "vlog_4k.mp4",
  "chunk_size_bytes": 52428800
}
```
- **Response**: `201 Created`
```json
{
  "video_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "upload_id": "s3_upload_session_982347923",
  "chunk_size_bytes": 52428800,
  "total_chunks": 200,
  "part_urls": [
    {
      "part_number": 1,
      "upload_url": "https://raw-bucket.s3.amazonaws.com/raw/vlog_4k.mp4?partNumber=1&uploadId=..."
    }
  ]
}
```

#### 8.2.2 Complete Upload Session
- **Method**: `POST`
- **Path**: `/api/v1/videos/{video_id}/upload/complete`
- **Request Body**:
```json
{
  "upload_id": "s3_upload_session_982347923",
  "parts": [
    { "part_number": 1, "etag": "\"e1a2b3c4...\"" },
    { "part_number": 2, "etag": "\"f5g6h7i8...\"" }
  ]
}
```
- **Response**: `200 OK`
```json
{
  "video_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "INGESTED_PROCESSING_QUEUED",
  "websocket_tracking_url": "wss://api.processor.tv/ws/videos/9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d/status"
}
```

#### 8.2.3 Query Video Status & Playback Manifest
- **Method**: `GET`
- **Path**: `/api/v1/videos/{video_id}`
- **Response**: `200 OK`
```json
{
  "video_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "READY_FOR_PLAYBACK",
  "duration_seconds": 600.5,
  "manifests": {
    "hls": "https://cdn.processor.tv/videos/9b1deb4d/master.m3u8",
    "dash": "https://cdn.processor.tv/videos/9b1deb4d/manifest.mpd"
  },
  "thumbnails": {
    "poster": "https://cdn.processor.tv/videos/9b1deb4d/poster.jpg",
    "storyboard": "https://cdn.processor.tv/videos/9b1deb4d/storyboard.vtt"
  },
  "subtitles": [
    { "language": "en", "url": "https://cdn.processor.tv/videos/9b1deb4d/sub_en.vtt" }
  ]
}
```

---

## 9. Security Design

### 9.1 Security Architecture
The platform isolates public ingress from internal batch processing. Direct S3 access is strictly limited via short-lived signed URLs with embedded hash verification.

### 9.2 Authentication & Authorization
- **Client Auth**: OAuth2 / JWT tokens issued by Identity Provider.
- **Service Auth**: Mutual TLS (mTLS) between internal gRPC microservices via Service Mesh (Istio).
- **CDN Auth**: Signed URLs with token authentication (HMAC-SHA256) to prevent unauthorized hotlinking.

### 9.3 Data Security
- **Encryption at Rest**: AWS S3 SSE-KMS for object storage; AES-256 for PostgreSQL database disk.
- **Encryption in Transit**: TLS 1.3 across all HTTP/gRPC endpoints.
- **DRM Integration**: Support FairPlay (Apple), Widevine (Google), and PlayReady (Microsoft) key server integration for premium paid content.

### 9.4 Application & Infrastructure Security
- Antivirus & Malicious Code Scanning on raw uploaded containers before transcoding execution.
- Sandboxed Transcode Containers: Workers run in unprivileged Docker containers with restricted system calls (Seccomp profiles) to prevent FFmpeg buffer overflow exploits.

---

## 10. Scalability & Performance

### 10.1 Performance Metrics & Target SLAs
- **Chunk Encoding Speed**: 10-second 1080p chunk processed in < 3 seconds on GPU.
- **End-to-End Latency**: 10-minute 4K video ready in all resolutions in < 2 minutes.

### 10.2 Scalability Strategy
- **Horizontal Scaling**: Transcoding workers scale horizontally across K8s worker groups using Kubernetes Event-driven Autoscaling (KEDA) based on Kafka topic consumer lag.
- **GPU Acceleration**: Utilize hardware acceleration (NVIDIA NVENC for H.264/HEVC, Intel QuickSync) to increase single-node encoding density by 10x over pure CPU encoding.
- **Tiered CDN Caching**:
  - **Edge Caching**: Manifest files (`.m3u8`) cached with low TTL (2–5 seconds during live/processing; long TTL once static).
  - **Segment Caching**: Media segments (`.ts`, `.m4s`) cached indefinitely at edge nodes (high cache hit ratio > 98%).

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture
Kubernetes cluster spanning multiple availability zones (AZs) with dedicated worker node pools:
- `control-plane-pool`: CPU-optimized instances (c6i.xlarge) for API, Gateway, Orchestrator.
- `cpu-worker-pool`: Compute-heavy instances (c6i.8xlarge) for standard CPU transcoding and chunk slicing.
- `gpu-worker-pool`: GPU instances (g5.2xlarge - NVIDIA A10G) for 4K/8K AI-enhanced encoding, ASR speech model execution, and Content ID vector generation.

### 11.2 Environment Strategy
- **Dev / Staging / Production**: Isolated AWS accounts and Kubernetes namespaces.
- **Feature Flagging**: Launch Darkly / OpenFeature to safely roll out new encoder configurations (e.g., enabling AV1 encoding).

### 11.3 CI/CD Pipeline
- Automated build pipeline using GitHub Actions / ArgoCD:
  - Code Linting & Static Analysis (`golangci-lint`, `ruff`).
  - Unit & Integration Tests.
  - Container Image Vulnerability Scanning (`Trivy`).
  - GitOps Deployment to Kubernetes via ArgoCD.

### 11.4 Monitoring & Alerting
- **Metrics**: Prometheus collecting custom exporter metrics:
  - `youtube_processor_active_uploads`
  - `youtube_processor_queue_depth`
  - `youtube_processor_chunk_transcode_duration_seconds`
  - `youtube_processor_failed_jobs_total`
- **Dashboards**: Grafana dashboards displaying pipeline lag, worker GPU utilization, and encoding cost per video minute.
- **Alerting**: PagerDuty alerts for queue backlog > 10,000 items or DLQ message count > 50 in 5 minutes.

---

## 12. Testing Strategy

### 12.1 Test Types
- **Unit Testing**: Tests for chunking logic, manifest parser, and S3 URL builder (>85% code coverage).
- **Integration Testing**: Spin up LocalStack (S3, SQS) and test complete multi-part upload flow and message publication.
- **End-to-End System Testing**: Automated pipeline test that uploads a standard test video sample (`big_buck_bunny_4k.mp4`), verifies manifest generation, checks segment playability with `ffprobe`, and validates subtitle alignment.
- **Load & Stress Testing**: Use Locust / k6 to simulate 5,000 concurrent video uploads and verify KEDA autoscaling response.

---

## 13. Risk Analysis

### 13.1 Technical & Operational Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| **Malicious / Malformed Video Container Exploit** | High | Medium | Execute FFmpeg inside isolated, unprivileged containers with strict seccomp/AppArmor profiles and memory limits. |
| **GPU Instance Shortage in Cloud Region** | High | Low | Multi-region deployment failover; fallback to CPU-based encoding (libx264/libvpx) with temporary higher latency SLA. |
| **Poison Pill Video File Crash Loop** | Medium | Medium | Limit task retries to 3 attempts. Route failing tasks to Dead Letter Queue (DLQ) and flag video for manual inspection. |
| **High Cloud Storage & egress costs** | High | High | Implement Lifecycle Policies: delete raw uploaded videos after 30 days; store intermediate chunks in ephemeral storage; deliver exclusively via CDN. |

---

## 14. Implementation Plan

### 14.1 Project Phases

#### Phase 1: Ingestion & Basic Transcoding (Weeks 1–4)
- Implement Upload Service with S3 multi-part pre-signed URL generation.
- Build basic Video DAG Orchestrator and FFmpeg H.264/720p/1080p single-node transcoder.
- Generate basic HLS manifests (`master.m3u8`).

#### Phase 2: Chunking Fan-Out & Parallel Processing (Weeks 5–8)
- Implement GOP-aligned keyframe chunking.
- Implement distributed Kafka task queue and worker pool autoscaling.
- Support HEVC, VP9, and AV1 multi-bitrate profiles.

#### Phase 3: AI Extensions & Content Protection (Weeks 9–12)
- Integrate Whisper AI for automatic speech recognition and subtitle generation.
- Implement thumbnail score model and storyboard sprite builder.
- Integrate Content ID fingerprinting and NSFW content moderation service.

#### Phase 4: Production Hardening & Global Scale (Weeks 13–16)
- Setup Multi-AZ Kubernetes clusters, KEDA GPU autoscaling, and Cloudflare CDN origin configuration.
- Perform load testing, security penetration testing, and failure mode chaos testing.

---

## 15. Appendices

### Appendix A: Glossary
- **GOP (Group of Pictures)**: A collection of consecutive frames within a coded video stream starting with an I-frame (keyframe).
- **HLS (HTTP Live Streaming)**: An HTTP-based media streaming communications protocol developed by Apple.
- **DASH (Dynamic Adaptive Streaming over HTTP)**: An adaptive bitrate streaming technique enabling high quality streaming over HTTP.
- **Transcoding**: The direct digital-to-digital conversion of one encoding format to another.
- **ASR (Automatic Speech Recognition)**: AI-driven conversion of spoken audio into text.
- **Content ID**: Automated audio/video fingerprinting system used to identify copyrighted media.

### Appendix B: Sample FFmpeg Encoding Commands

#### Keyframe Detection & GOP Chunking
```bash
ffprobe -select_streams v:0 -show_frames \
  -show_entries frame=pkt_pts_time,key_frame \
  -of csv input_video.mp4 | grep "1$"
```

#### H.264 1080p Segment Encoding
```bash
ffmpeg -i chunk_001.mp4 -c:v libx264 -preset veryfast \
  -b:v 4500k -maxrate 5000k -bufsize 9000k \
  -vf "scale=1920:1080:force_original_aspect_ratio=decrease,pad=1920:1080:(ow-iw)/2:(oh-ih)/2" \
  -c:a aac -b:a 128k -f hls -hls_time 4 -hls_playlist_type vod \
  -hls_segment_filename "segment_1080p_%03d.ts" 1080p.m3u8
```

#### AV1 4K Segment Encoding (libaom-av1)
```bash
ffmpeg -i chunk_001.mp4 -c:v libaom-av1 -crf 24 -b:v 0 -cpu-used 4 \
  -vf "scale=3840:2160" -c:a opus -b:a 160k \
  -f stream_segment -segment_time 4 -reset_timestamps 1 chunk_001_av1_%03d.m4s
```
