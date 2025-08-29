# High-Level Design Document

## 1. Document Information

### Document Metadata

- **Document Title**: File Storage Service - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-22
- **Author(s)**: Jules (AI Agent)
- **Reviewers**: [Reviewer Names and Roles]
- **Approvers**: [Approver Names and Roles]
- **Document Status**: Draft

### Revision History

| Version | Date       | Author          | Changes         |
|---------|------------|-----------------|-----------------|
| 1.0     | 2025-08-22 | Jules (AI Agent)| Initial version |

### Distribution List

- Stakeholders
- Development Team
- Operations Team

---

## 2. Executive Summary

### 2.1 Purpose

This document provides the high-level design for a scalable File Storage and Synchronization Service, similar to Dropbox or Google Drive. The system allows users to store files in the cloud, manage them in folders, and synchronize them across multiple devices.

### 2.2 Scope

**In Scope:**

- User accounts and authentication.
- Uploading, downloading, and deleting files.
- Organizing files into a folder hierarchy.
- A client application that monitors a local directory and synchronizes changes with the cloud.
- Storing file metadata and version history.
- Sharing files and folders with other users.

**Out of Scope:**

- Advanced collaborative features like real-time document editing (e.g., Google Docs).
- Full-text search of file contents.
- Enterprise-level features like data loss prevention (DLP) or advanced access controls.

### 2.3 Key Benefits

- **Durability**: Files are stored reliably with multiple replicas to prevent data loss.
- **Availability**: Users can access their files from anywhere at any time.
- **Synchronization**: Provides a seamless experience by keeping files up-to-date across all of a user's devices.
- **Scalability**: Designed to store exabytes of data and handle millions of concurrent users.

### 2.4 High-Level Architecture Overview

The architecture is composed of several key services. A **Client Application** runs on the user's device, monitoring the local file system for changes. When a change occurs (e.g., a file is added or modified), the client communicates with a **Metadata Service** to record the change and uploads the actual file data to a **Block Storage Service**. The file is broken down into smaller, content-addressed blocks to enable deduplication. The Metadata Service then notifies other clients for the same user about the change via a **Notification Service**, prompting them to download the updated blocks. This synchronization is often managed by the clients maintaining a persistent connection to the Notification Service.

---

## 3. System Overview

### 3.1 Business Context

Cloud file storage services are essential tools for both consumers and businesses, providing a centralized, accessible, and reliable way to store and synchronize files. They form the foundation of the modern "work from anywhere" environment.

### 3.2 System Purpose

The system's purpose is to provide a durable, highly available storage solution for user files. It must allow users to upload, download, and manage their files, and critically, it must synchronize any changes to those files across all of the user's connected devices.

### 3.3 Success Criteria

- **Durability**: 99.999999999% (11 9s) durability for stored files. No data should ever be lost.
- **Availability**: 99.99% availability for file uploads, downloads, and metadata operations.
- **Sync Speed**: Changes made on one device should be reflected on other devices within seconds.
- **Scalability**: The system must scale to store exabytes of data.

### 3.4 Assumptions

- Users will install a client application on their devices (desktop, mobile).
- A small number of large files will contribute a significant portion of the total storage, but a large number of small files will dominate the number of operations.
- Files are treated as opaque blobs; the service does not need to understand their content.

### 3.5 Constraints

- The cost of storage is a major concern. The system must be designed for cost-efficiency.
- Efficiently handling file synchronization is a complex distributed systems problem.

### 3.6 Dependencies

- A large-scale, low-cost object storage system (e.g., S3).
- A database for storing file metadata.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements

- **FR-001**: Users must be able to upload files.
- **FR-002**: Users must be able to download files.
- **FR-003**: Users must be able to create, rename, and delete files and folders.
- **FR-004**: The system must track file version history.
- **FR-005**: A client application must monitor a local folder and synchronize its state with the cloud.
- **FR-006**: The system must break large files into smaller chunks for efficient uploading and downloading.
- **FR-007**: The system must deduplicate file chunks to save storage space.

### 4.2 Non-Functional Requirements

#### 4.2.1 Performance Requirements

- **Upload/Download Speed**: Should be limited only by the user's network bandwidth.
- **Sync Latency**: < 5 seconds for a metadata change (like a file rename) to propagate.
- **Metadata API Latency**: < 100ms for operations like listing a folder's contents.

#### 4.2.2 Scalability Requirements

- The metadata service must scale to handle billions of files and folders.
- The storage service must scale to exabytes.
- The notification service must handle millions of concurrent connections from clients.

#### 4.2.3 Availability Requirements

- 99.99% for the data path (uploads/downloads).
- 99.9% for the metadata path.

#### 4.2.4 Security Requirements

- All files must be encrypted at rest and in transit.
- The system must provide a way to share files securely with other users.

---

## 5. Architecture Design

### 5.1 Architecture Principles

- **Separation of Metadata and Data**: The file metadata (names, hierarchy, versions) is managed separately from the file content (data blocks). This allows each to be scaled independently with the right tools.
- **Content-Addressable Storage**: File blocks are stored using a hash of their content as the key. This provides automatic, efficient, block-level deduplication.
- **Client-Orchestrated Sync**: The client is intelligent. The server provides notifications of change, but the client is responsible for fetching the metadata and data it needs to reconcile its local state.

### 5.2 Architecture Patterns

- **Block Storage**: Files are split into smaller, fixed-size or variable-size blocks.
- **Long Polling / WebSockets**: The client maintains a near-real-time connection to the Notification Service to learn about changes immediately.
- **Database per Service**: The Metadata Service uses a transactional database, while the Block Storage service uses a simple key-value object store.

### 5.3 High-Level Architecture Diagram

```text
+----------------+  (1. File change detected)
| Client App     | -------------------------------------+
+----------------+                                      |
   |         ^                                          v
   | (5. Download blocks)  +------------------+   (2. Update Metadata)
   |         |             | Block Storage    | <--------------------+
   |         +----------- | (S3)             |   (3. Upload Blocks) |
   |                       +------------------+                      |
   v                                                                 |
+----------------+   (4. Long Poll for changes)   +------------------+
| Notification Svc |<---------------------------- | Metadata Service |
| (Real-time sync) |                              | (Manages file   |
+----------------+                              | hierarchy,     |
                                                | versions)        |
                                                +--------+---------+
                                                         |
                                                         v
                                                +------------------+
                                                | Metadata DB      |
                                                | (PostgreSQL, etc)|
                                                +------------------+
```

### 5.4 Component Overview

- **Client Application**: A desktop or mobile client that monitors the user's local "Dropbox" folder. It communicates with all backend services.
- **Metadata Service**: The brain of the system. It manages the file system hierarchy (files, folders, permissions, versions). It does NOT store the file content itself.
- **Metadata DB**: A transactional database (like PostgreSQL) that stores all file metadata. This needs to support a hierarchical structure and transactions for operations like moving a folder.
- **Block Storage Service**: A simple service that provides an API to upload and download file blocks. It sits in front of a massive key-value object store.
- **Object Store (S3)**: The durable, scalable storage for the actual file blocks. Blocks are keyed by the hash of their content.
- **Notification Service**: A highly scalable service that manages persistent connections from millions of clients. When the Metadata Service records a change, it informs the Notification Service, which in turn pushes a notification to all relevant clients for that user.

### 5.5 Technology Stack

- **Client Application**: Written in a native language for each OS (e.g., C++ on Windows, Swift on macOS, Kotlin on Android).
- **Backend Services**: **Go** is an excellent choice for its performance and concurrency, which is useful for the high-throughput Notification and Block Storage services.
- **Metadata Database**: **PostgreSQL** or **MySQL Cluster**. A relational database is a good choice for managing the file system tree and enforcing transactional consistency.
- **Block Storage**: **Amazon S3** or a custom object store built on top of a distributed file system.

### 5.6 Architecture Decision Records (ADRs)

#### 5.6.1 ADR-001: Synchronization Mechanism

- **Status**: Accepted
- **Context**: Clients need to be notified of changes made by other clients for the same user in near-real-time.
- **Decision**: The client will use **HTTP Long Polling** to connect to the Notification Service. When a change occurs, the server immediately returns a response with a "changes occurred" message. The client then contacts the Metadata Service to get the specific details of the change. This is a simple, robust, and scalable pattern.
- **Consequences**: This requires a highly scalable Notification Service that can hold millions of open HTTP connections. It's more resource-intensive than a pure push mechanism (like WebSockets) but is simpler to implement and firewall-friendly. The client becomes more complex, as it has to drive the synchronization logic.

---

## 6. Detailed Component Design

### 6.1 Component 1: Client Application

#### 6.1.1 Purpose

To provide a seamless file synchronization experience for the user.

#### 6.1.2 Responsibilities

- Monitor the local file system for changes.
- Break files into blocks, hash them, and upload new blocks to the Block Storage Service.
- Communicate with the Metadata Service to create/update file metadata.
- Maintain a long-polling connection to the Notification Service.
- When notified of a change, fetch the new metadata and download the required blocks to reconstruct the updated file locally.

### 6.2 Component 2: Metadata Service

#### 6.2.1 Purpose

To be the source of truth for all file and folder metadata.

#### 6.2.2 Responsibilities

- Provide APIs for all metadata operations (create, read, update, delete).
- Manage file versions by maintaining a pointer to the root of the block list for each version.
- Handle sharing and permissions logic.
- After a successful metadata update, notify the Notification Service.

### 6.3 Component 3: Notification Service

#### 6.3.1 Purpose

To inform connected clients that a change has occurred.

#### 6.3.2 Responsibilities

- Manage millions of persistent (long-polling) connections.
- Maintain a mapping of `user_id` to their active connections.
- Receive notifications from the Metadata Service.
- Send a simple "a change happened" message to the relevant clients.

---

## 7. Data Design

### 7.1 Data Models

- **File**: `file_id`, `name`, `owner_id`, `parent_folder_id`, `current_version_id`.
- **FileVersion**: `version_id`, `file_id`, `created_at`, `root_block_list_pointer`.
- **Block**: The actual binary data, stored in an object store.
- **Metadata**: The hierarchy is modeled like a traditional file system tree, with each user having a root folder.

### 7.2 Database Design

- **Metadata DB (PostgreSQL)**:
  - `files` table: `file_id`, `parent_id`, `name`, `owner_id`. A self-referencing table to represent the tree structure.
  - `file_versions` table: `version_id`, `file_id`, `block_list_id`.
  - `block_lists` table: A table that maps a `block_list_id` to an ordered list of `block_hashes`.
- **Block Store (S3)**:
  - Objects are named by the hash of their content, e.g., `s3://blocks/{sha256_hash}`.

---

## 8. API Design

### 8.1 API Architecture

- A REST API for the Metadata Service.
- A separate endpoint for the long-polling Notification Service.
- Direct uploads/downloads from the client to the Block Store (S3), often using pre-signed URLs.

### 8.2 Core APIs (RESTful)

- `GET /api/v1/metadata?path=/my/folder`: List the contents of a folder.
- `POST /api/v1/metadata`: Create a new file or folder.
- `GET /api/v1/events`: The long-polling endpoint for the Notification Service.
- `POST /api/v1/blocks/upload-url`: Get a pre-signed URL to upload a block to S3.

---

## 9. Security Design

### 9.1 Security Architecture

The architecture separates metadata and data paths. Access to both must be independently secured. Pre-signed URLs for direct S3 access is a key pattern, offloading bandwidth from our services while maintaining security.

### 9.2 Authentication & Authorization

- **User Authentication**: Standard OAuth 2.0 flow.
- **Client Authentication**: Each installed client application may have its own unique ID to be tracked and managed.
- **Permissions**: The Metadata Service is the source of truth for all permissions, including for shared files and folders. It must authorize every request.

### 9.3 Data Security

- **Client-Side Encryption**: For maximum security, the client could encrypt file blocks before uploading them. The server would never have access to the unencrypted data. This complicates features like web access, which would require the key to be sent to the browser.
- **Server-Side Encryption**: All data in S3 and in the database will be encrypted at rest. All communication is over TLS.

---

## 10. Scalability & Performance

### 10.1 Performance Requirements

- As defined in section 4.2.1. Sync latency is a key user-facing metric.

### 10.2 Scalability Strategy

- **Metadata Service**: This is the most likely bottleneck. The database can be scaled with read replicas. For write scaling, sharding by `user_id` is a viable strategy, as a user's file system is self-contained.
- **Notification Service**: This service must handle millions of concurrent, long-lived connections. It can be built on an asynchronous, non-blocking I/O framework (like Netty, or Go's net/http) and scaled horizontally. A load balancer will distribute connections, and a shared backend (like Redis Pub/Sub) can be used to broadcast notifications to the correct server holding the client's connection.
- **Block Storage**: S3 provides virtually infinite scalability for the file blocks themselves.

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture

- The backend services will be deployed on Kubernetes.
- The client applications will be deployed through standard OS-specific app stores or installers.

### 11.2 Monitoring & Alerting

- **Key Metrics**: `sync_latency`, `metadata_api_p99_latency`, `active_notification_connections`, `storage_growth_rate`, `deduplication_ratio`.
- **Alerting**: Alerts for high sync latency, high error rates in metadata operations, or a sudden drop in active connections.

---

## 12. Testing Strategy

### 12.1 Test Types

- **Client Application Testing**: Extensive testing across different operating systems (Windows, macOS, Linux, iOS, Android) is required.
- **Synchronization Logic Testing**: This is the most critical area. Automated tests will create complex scenarios:
  - **Conflict Resolution**: Two clients modify the same file at the same time. The system should create a "conflicted copy".
  - **Offline Changes**: A client goes offline, makes many changes, and then comes back online. The changes must sync correctly.
  - **Large Scale Changes**: A user moves a folder with 10,000 files in it. This should be a single, efficient metadata operation.
- **Durability Testing**: Regularly test the backup and recovery process for the metadata database.

---

## 13. Risk Analysis

### 13.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Client-side Sync Bugs | Critical | High | A bug in the client could delete user data locally or remotely. Rigorous testing is required. Implement a "trash" feature to allow users to recover accidentally deleted files. |
| Metadata DB Failure | Critical | Low | This is the single point of failure for the file system structure. Use a highly available, replicated database configuration. Perform regular, tested backups. |
| "Thundering Herd" on Notification | High | Medium | A widely shared folder is modified, causing a notification to be sent to thousands of users at once. This could overload the Metadata service as all clients request the update. Use exponential backoff for clients and cache popular shared metadata. |

### 13.2 Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| High Storage Costs | High | High | The block-level deduplication is the primary mitigation. Also, implement storage quotas for users and tiered pricing. |
| Abuse of Service | Medium | High | Users storing illegal or copyrighted material. Implement a reporting mechanism and a clear acceptable use policy. |

---

## 14. Implementation Plan (High-Level)

- **Phase 1**: Build the Metadata service and DB with basic file/folder operations.
- **Phase 2**: Build the Block Storage service and integrate with S3.
- **Phase 3**: Develop the first version of the desktop client with basic sync logic.
- **Phase 4**: Build the Notification service and integrate it with the client and metadata service.
- **Phase 5**: Implement versioning and sharing features.

---

## 15. Appendices

### Appendix A: Glossary

- **Block**: A small chunk of a file.
- **Content-Addressable Storage**: Storing data based on a hash of its content, which allows for automatic deduplication.
- **Long Polling**: A technique where a client holds an HTTP request open until the server has new data to send. Used to simulate a server push.
- **Deduplication**: The process of eliminating duplicate copies of data to save storage space.
