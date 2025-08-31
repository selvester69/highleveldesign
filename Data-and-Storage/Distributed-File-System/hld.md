# High-Level Design: Distributed File System

## 1. Document Information

- **Document Title**: Distributed File System - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a Distributed File System (DFS), architecturally similar to the Google File System (GFS) or the Hadoop Distributed File System (HDFS). The system is designed to store and manage extremely large files (multi-gigabyte to terabyte scale) across a large cluster of commodity hardware, providing high-throughput, fault-tolerant access to the data.

### 2.2 Scope

**In Scope:**
-   Storing massive files by splitting them into fixed-size "chunks".
-   A central metadata server (or "NameNode") to manage the file system namespace and chunk locations.
-   A set of data servers (or "ChunkServers" / "DataNodes") to store the actual data chunks.
-   Replication of chunks across multiple data servers for fault tolerance.
-   Client interactions for creating, reading, writing, and deleting files.
-   A heartbeat mechanism for the metadata server to monitor the health of data servers.

**Out of Scope:**
-   A POSIX-compliant file system interface. The system will expose its own simplified API.
-   Low-latency data access for small files (the system is optimized for large files and high throughput).
-   Strict security controls and fine-grained permissions (a simplified model will be used).
-   Distributed consensus algorithms for the metadata server itself (we assume a single-point-of-contact NameNode with its own persistence strategy).

### 2.3 High-Level Architecture Overview
The architecture consists of a single **Master Node** (also called NameNode) and many **ChunkServers** (or DataNodes). The Master Node stores all file system metadata, including the file hierarchy, permissions, and the mapping of each file to its constituent chunks and their locations. It does *not* store the file data itself. The ChunkServers store the actual data, broken into fixed-size chunks (e.g., 64MB or 128MB). For reliability, each chunk is replicated on multiple (typically 3) ChunkServers. Clients communicate with the Master to get metadata (e.g., "where are the chunks for file X?") and then interact directly with the ChunkServers to read or write the file data.

---

## 3. System Overview

### 3.1 Business Context
Many large-scale data processing applications, such as MapReduce, data warehousing, and scientific computing, need to process datasets that are far too large to fit on a single machine. A DFS provides a scalable, reliable, and cost-effective foundation for storing and accessing this data by leveraging clusters of inexpensive commodity hardware.

### 3.2 System Purpose
To provide a fault-tolerant, high-throughput file system optimized for large files and sequential read/write workloads. The system prioritizes performance and reliability over low latency and POSIX compliance.

### 3.3 Success Criteria
-   **Scalability**: The system should be able to manage petabytes of data across thousands of servers.
-   **High Throughput**: The system must support high-speed sequential reads and writes, aggregating the bandwidth of many machines.
-   **Fault Tolerance**: The failure of a single ChunkServer should not result in data loss or unavailability. The system should automatically re-replicate under-replicated chunks.
-   **Cost-Effectiveness**: The system must run on standard, off-the-shelf hardware.
-   **Data Integrity**: Data should not be corrupted in transit or at rest.

### 3.4 Assumptions
-   Component failures are the norm, not the exception. The system must be built from the ground up to handle frequent hardware failures.
-   Files are huge by traditional standards. Small files are supported but are not the optimization target.
-   Workloads consist primarily of large, sequential reads and appends. Random writes are not a primary use case.
-   The system runs in a trusted environment with limited security threats.

### 3.5 Constraints
-   **Single Point of Failure (SPOF)**: The centralized Master Node can be a performance bottleneck and a potential SPOF. Its design must be carefully considered for high availability.
-   **Large Chunk Size**: The large chunk size (e.g., 64MB) is great for sequential reads but can lead to wasted space if the system stores many small files (internal fragmentation).
-   **Client-Side Complexity**: The client is "smart" and handles a significant amount of logic, including talking to the Master, then the ChunkServers, and handling retries.
-   **Metadata Scale**: The Master must hold all metadata in memory to provide fast responses, which limits the total number of files and chunks the system can store based on the Master's RAM.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (File Creation)**: Clients must be able to create new, empty files in the file system namespace.
-   **FR-002 (File Deletion)**: Clients must be able to delete existing files.
-   **FR-003 (File Reading)**: Clients must be able to read the entire content of a file or a specific byte range within it.
-   **FR-004 (File Appending)**: Clients must be able to append data to the end of an existing file. (Atomic record appends are a key feature).
-   **FR-005 (Namespace Operations)**: Clients must be able to list directories and get file metadata.
-   **FR-006 (Chunking)**: The system must automatically split large files into fixed-size chunks.
-   **FR-007 (Replication)**: The system must automatically create a configurable number of replicas for each chunk on different ChunkServers.
-   **FR-008 (Fault Detection)**: The Master must be able to detect when a ChunkServer has failed via a heartbeat mechanism.
-   **FR-009 (Automatic Re-replication)**: Upon detecting a ChunkServer failure, the Master must identify all under-replicated chunks and instruct other ChunkServers to create new replicas.

### 4.2 Non-Functional Requirements
-   **High Availability**: Data should remain available for reads even if a ChunkServer fails.
-   **Durability**: Data should not be lost. The system should be able to withstand the failure of multiple ChunkServers (up to the replication factor).
-   **Scalability**: The number of ChunkServers should be able to scale into the thousands.
-   **Performance**: Optimized for high aggregate throughput from many clients reading/writing large files.
-   **Data Integrity**: The system must use checksums to detect data corruption at rest or in transit.
-   **Simplicity**: The design should aim for simplicity, offloading complexity to the client where possible to keep the server components manageable.

---

### 5. Core Architecture

The architecture is defined by a single Master coordinating many ChunkServers. The key design choice is to separate metadata management from data flow.

- **Client Library:** A library linked into the user's application. It handles all communication with the Master and ChunkServers, implementing the logic for read, write, and control operations.
- **Master (NameNode):**
  - Holds all file system metadata in memory: the namespace tree, file-to-chunk mappings, and chunk locations (`chunk_handle` -> `list_of_chunkservers`).
  - Manages system-wide activities: chunk lease management for writes, garbage collection of orphaned chunks, and orchestrating re-replication between ChunkServers.
  - Communicates with ChunkServers via a heartbeat mechanism to monitor their health and collect stats.
  - Persists its state to disk via a transaction log and periodic checkpoints.
- **ChunkServer (DataNode):**
  - Stores fixed-size (e.g., 64MB) data chunks on its local disk, each identified by a unique 64-bit `chunk_handle`.
  - Reads and writes chunk data as requested by clients or other ChunkServers.
  - Each chunk is stored as a plain file. A checksum is stored for each 64KB block within the chunk to detect corruption.
  - Periodically sends a heartbeat to the Master to report its status and the list of chunks it holds.

**Read Workflow:**
1.  Client asks the Master for the locations of chunks for a given file and byte range.
2.  Master replies with an ordered list of `chunk_handles` and the corresponding lists of ChunkServer locations for each chunk. The client caches this information.
3.  Client contacts the closest ChunkServer for the first chunk and requests the data.
4.  ChunkServer reads the data from its disk, verifies the checksum, and sends it to the client.
5.  Client repeats for all subsequent chunks.

**Append Workflow:**
1.  Client asks the Master for the primary replica holder for the last chunk of the file.
2.  Master grants a "lease" to one replica (the Primary) and tells the client the locations of the Primary and other (Secondary) replicas.
3.  Client pushes the data to all replicas, starting with the Primary. The data is stored in an internal buffer on the ChunkServers.
4.  Once all replicas acknowledge receiving the data, the client sends a "commit" request to the Primary.
5.  The Primary assigns a serial order to the commits and applies the write to its local chunk in that order.
6.  The Primary forwards the commit request (with the assigned order) to all Secondaries, who apply the write in the same order.
7.  The Primary replies to the client after receiving acknowledgements from all Secondaries.

![Architecture Diagram](https://i.imgur.com/k2E0h1O.png)

### 6. Data Schema and Models

- **Master's In-Memory Metadata:**
  - **Namespace:** A tree data structure mapping full pathnames to file/directory metadata.
  - **File-to-Chunk Mapping:** A map `filename` -> `array of chunk_handles`.
  - **Chunk-to-Location Mapping:** A map `chunk_handle` -> `{ list_of_chunkservers, primary_replica, lease_expiry }`.
- **On-Disk Metadata (Master):**
  - **Transaction Log:** An append-only log of every metadata change (file creation, new chunk, etc.). This is used for recovery.
  - **Checkpoint:** A snapshot of the Master's in-memory data structures, stored as a file.
- **ChunkServer Data:**
  - Each chunk is a file on the local filesystem.
  - A separate metadata file for each chunk stores its checksums.

### 7. API Design

The API is exposed via the client library.

- `create(path)`
- `delete(path)`
- `read(path, offset, length) -> data`
- `append(path, data) -> offset`
- `ls(path) -> list_of_files`

### 8. Scalability and Performance

- **Read Scalability:** Reads are highly scalable as they are parallelized across many ChunkServers, with no involvement from the Master after the initial metadata lookup.
- **Write Scalability:** Appends are also parallel, but the Master's involvement in lease management can be a bottleneck. The key is that data flows directly from the client to the ChunkServers, not through the Master.
- **Master Bottleneck:** The single Master is the main scalability challenge. Its performance is limited by its CPU and memory. For very large systems, sharding the namespace across multiple Masters (Federation) is a possible extension.
- **Network Topology:** The Master uses network topology information (e.g., rack awareness) when placing replicas to maximize resilience and read/write performance. For example, it places one replica on a different rack from the others.

### 9. Reliability and Fault Tolerance

- **High Availability of Data:** Achieved through chunk replication. If a ChunkServer goes down, the data is still available from its other replicas.
- **Master Recovery:** If the Master fails, it can be restarted. It recovers its state by replaying the transaction log on top of the last checkpoint. A hot standby Master can be maintained for faster failover.
- **Data Integrity:** Checksums are used to detect data corruption. If a ChunkServer detects a checksum mismatch on a read, it returns an error to the client and reports the corruption to the Master. The client then reads from another replica, and the Master re-replicates the chunk from a good replica.
- **Garbage Collection:** When a file is deleted, the Master logs the deletion but does not immediately reclaim the resources. It renames the file to a hidden name. During a regular scan, it garbage collects the chunks for any such hidden files that have existed for a configurable time (e.g., 3 days), providing a safety net against accidental deletion.

### 10. Security Considerations

- **Simplified Model:** The system assumes a trusted environment. Security is not a primary focus.
- **Access Control:** A simple permission model (e.g., file ownership and read/write permissions) is stored in the Master's metadata.
- **Network Security:** Communication between components is not encrypted by default to maximize performance, but could be layered on top.

### 11. Monitoring and Observability

- **Master Metrics:** Number of files, chunks, live/dead ChunkServers, and memory usage.
- **ChunkServer Metrics:** Disk space usage, network I/O, number of active reads/writes.
- **Heartbeats:** The heartbeat mechanism is the core of monitoring. The Master uses it to track the health of the entire cluster.

### 12. Deployment and Operations

- **Commodity Hardware:** The system is designed to run on standard Linux servers with standard hard drives.
- **Cluster Management:** Adding a new ChunkServer involves simply starting the process on a new machine. It will automatically contact the Master and join the cluster.
- **Balancing:** The Master can automatically rebalance chunks across the cluster to ensure even disk space usage and to spread load.

### 13. Cost Estimation

- **Primary Driver:** Storage costs (the total number and size of disks across all ChunkServers).
- **Compute:** The Master node requires a machine with a large amount of RAM. ChunkServers are typically not CPU-intensive.
- **Network:** Network infrastructure to support high-bandwidth traffic between nodes, especially within a rack.

### 14. Risks and Mitigations

- **Master Failure (SPOF):**
  - **Risk:** If the Master is down, the entire file system is unusable as clients cannot get metadata.
  - **Mitigation:** A hot standby Master that shadows the primary's state by tailing its transaction log. Failover can be automated using a system like ZooKeeper.
- **Metadata Scalability Limit:**
  - **Risk:** The Master runs out of RAM and cannot store any more file metadata.
  - **Mitigation:** For extreme scale, implement HDFS Federation, where multiple Master nodes each manage a slice of the file system namespace.
- **Cascading Failures:**
  - **Risk:** A network partition or rack failure causes many ChunkServers to become unavailable simultaneously, leading to massive re-replication traffic that congests the network.
  - **Mitigation:** Throttle re-replication traffic. Make replica placement rack-aware to survive a single rack failure.

### 15. Final Considerations

This design for a Distributed File System deliberately makes trade-offs to excel at its specific goal: storing and providing high-throughput access to very large files. It sacrifices low-latency access and POSIX compliance for simplicity, scalability, and fault tolerance. The separation of control plane (Master) from the data plane (Client -> ChunkServer) is the key architectural principle that allows the system to scale its bandwidth capacity by simply adding more ChunkServers. Managing the Master as a single point of failure is the system's greatest operational challenge.
