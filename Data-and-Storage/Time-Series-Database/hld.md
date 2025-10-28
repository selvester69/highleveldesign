# High-Level Design: Time Series Database

## 1. Document Information

- **Document Title**: Time Series Database (TSDB) - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a Time Series Database (TSDB), a database explicitly optimized for storing and querying time-stamped data. The system is designed for high-throughput data ingestion, efficient long-term storage via advanced compression, and fast analytical queries that aggregate data over time ranges. It is architecturally similar to systems like InfluxDB or Prometheus.

### 2.2 Scope

**In Scope:**
-   A data model optimized for time series data (timestamp, measurement, tags, fields).
-   A high-throughput write API for ingesting millions of data points per second.
-   A query engine and language (e.g., a SQL-like or functional language) for filtering and aggregating data by time.
-   A storage engine that uses log-structured merge-tree (LSM Tree) principles for writes and a columnar format for long-term storage.
-   Data compression techniques tailored for time series data.
-   Data lifecycle management, including retention policies and downsampling.

**Out of Scope:**
-   A full-text search engine.
-   Support for transactions or complex joins found in relational databases.
-   The agents or collectors that gather and send data to the TSDB (e.g., Telegraf).
-   Advanced alerting and visualization layers (these are applications built on top of the TSDB).

### 2.3 High-Level Architecture Overview
The architecture is designed to separate high-volume, recent data from historical, compressed data. **Writes** are first appended to a **Write-Ahead Log (WAL)** for durability and then stored in an **in-memory cache (Memtable)** for fast access. Periodically, the Memtable is flushed to disk as a new, sorted data file. The storage engine, inspired by Log-Structured Merge-Trees (LSM Trees), continuously **compacts** these smaller, sorted files into larger, highly-optimized, columnar-format files in the background. **Queries** must intelligently access both the in-memory cache for very recent data and the on-disk columnar files for historical data, merging the results to provide a unified view.

---

## 3. System Overview

### 3.1 Business Context
The proliferation of IoT devices, application monitoring, and financial systems has led to an explosion of time-stamped data. Standard databases are not equipped to handle the relentless ingestion rates and specific query patterns of this data. A TSDB provides the necessary infrastructure to store, analyze, and derive value from these massive data streams, enabling real-time monitoring, anomaly detection, and capacity planning.

### 3.2 System Purpose
To provide a purpose-built solution for efficiently storing and querying large volumes of time series data. The system must handle high write throughput and provide fast, flexible time-based aggregation queries.

### 3.3 Success Criteria
-   **Ingestion Rate**: The system must sustain an ingestion rate of at least 1 million data points per second on a single node.
-   **Query Performance**: Typical analytical queries (e.g., "average CPU usage for server X over the last 24 hours, grouped by minute") should complete in under a second.
-   **Storage Efficiency**: The system must achieve high compression ratios (e.g., 10x or better) on time series data.
-   **Scalability**: The system should be able to scale horizontally to support larger data volumes and higher query loads.
-   **Reliability**: Data written to the database must be durable and protected against corruption.

### 3.4 Assumptions
-   Data arrives in roughly chronological order.
-   Updates to existing data points are rare or disallowed. The workload is overwhelmingly append-only.
-   Queries are typically analytical, covering a time range and aggregating data, rather than point lookups.
-   The number of unique time series (cardinality) can be very high.

### 3.5 Constraints
-   **High Cardinality**: A large number of unique time series (e.g., millions of different combinations of measurements and tags) can create a massive index, stressing memory and disk I/O.
-   **Write-Heavy Workload**: The system must be optimized for writes, as ingestion rates are often orders of magnitude higher than query rates.
-   **Data Lifecycle**: Time series data loses value over time. The system must efficiently manage this by downsampling or deleting old data.
-   **Index Management**: The index must allow for fast lookups of series based on their tags, as well as fast lookups of data points based on time.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Data Ingestion)**: The system must provide a high-throughput write API (e.g., over HTTP) that accepts batches of data points.
-   **FR-002 (Data Model)**: The system must support a data model consisting of a timestamp, a measurement name, a set of key-value tags for indexing, and a set of key-value fields for the actual values.
-   **FR-003 (Querying)**: The system must provide a rich query language to select data by measurement, tags, and time range, and to perform aggregate functions (`mean`, `max`, `sum`), transformations, and grouping.
-   **FR-004 (Indexing)**: The system must automatically index the tags for all incoming data to allow for fast lookups.
-   **FR-005 (Data Retention)**: Users must be able to define retention policies that automatically delete data older than a certain age.
-   **FR-006 (Downsampling/Rollups)**: The system must support continuous queries that automatically create aggregated, lower-resolution rollups of data (e.g., creating hourly averages from minutely data).
-   **FR-007 (Data Compression)**: The system must automatically compress data on disk to save space.

### 4.2 Non-Functional Requirements
-   **High Write Throughput**: Must handle millions of writes per second.
-   **Low Query Latency**: Queries should return within milliseconds to a few seconds, depending on complexity.
-   **Storage Efficiency**: Must be highly efficient in storing data on disk.
-   **Scalability**: Must be able to scale out to a multi-node cluster.
-   **Durability**: Acknowledged writes must be durable and survive a server crash.
-   **Availability**: The system should be highly available, especially in a clustered setup.

---

### 5. Core Architecture

The architecture uses a Log-Structured Merge-Tree (LSM Tree) design, which is ideal for write-heavy workloads, combined with a columnar storage format for query efficiency.

- **Write Path:**
  1.  **Ingestion API:** An HTTP endpoint receives batches of points in a specific format (e.g., InfluxDB Line Protocol).
  2.  **Write-Ahead Log (WAL):** The raw data is immediately written to a WAL on disk. This ensures durability; if the server crashes, the WAL can be replayed to recover any data that wasn't flushed to the main storage.
  3.  **Memtable:** The data is also written to an in-memory, sorted data structure (the Memtable). This allows for very fast writes and makes recent data immediately available for querying.
  4.  **Flushing:** Once the Memtable reaches a certain size, or after a certain time, it is flushed to disk as a new, immutable, sorted file called a **Time-Structured Merge-Tree (TSM) file**.
- **Storage Engine:**
  - **TSM Files:** These are memory-mapped, columnar-format files. Data for a single series (timestamps and values) is stored contiguously and compressed using techniques like Gorilla, Simple8b, and run-length encoding.
  - **Index:** A separate, inverted index (the "Series File") maps measurement and tag sets to the series data stored across the various TSM files. This is critical for handling high cardinality.
- **Compaction:**
  - A background process continuously merges smaller TSM files into larger ones. This process has several benefits:
    - It combines data for the same series, improving compression.
    - It removes deleted data points.
    - It keeps the number of files manageable, which improves query performance (fewer files to check).
- **Query Path:**
  1.  **Query Engine:** Parses and plans the query.
  2.  The engine first checks the Memtable for very recent data that matches the query's time range and tag filters.
  3.  It then uses the Series File index to identify which TSM files on disk might contain relevant data.
  4.  It reads the compressed, columnar data from the relevant TSM files, decompresses it on the fly, and performs the required aggregations.
  5.  Finally, it merges the results from the Memtable and the TSM files before returning the final result to the client.

![Architecture Diagram](https://i.imgur.com/G5gDk7S.png)

### 6. Data Schema and Models

- **Line Protocol (for ingestion):** A human-readable text format for a single data point.
  `measurement,tag_key1=tag_value1,tag_key2=tag_value2 field_key1=field_value1,field_key2=field_value2 timestamp`
  Example: `cpu,host=serverA,region=us-west usage_user=99.8,usage_system=0.2 1465839830100400200`
- **On-Disk Columnar Format (TSM File):**
  - Data is organized by series (a unique combination of measurement and tags).
  - Within a series, timestamps and values for each field are stored in separate, contiguous blocks.
  - This columnar layout is extremely efficient for queries that only touch a subset of fields and allows for the best compression algorithms to be used for each data type.

### 7. API Design

- **`POST /write`:**
  - **Usage:** Ingest a batch of data points.
  - **Body:** Data points formatted in Line Protocol, separated by newlines.
  - **Response:** `204 No Content` on success.

- **`POST /query`:**
  - **Usage:** Execute a query.
  - **Body:** A JSON object containing the query string, e.g., `{ "q": "SELECT mean(usage_user) FROM cpu WHERE host = 'serverA' AND time > now() - 1h GROUP BY time(1m)" }`
  - **Response:** `200 OK` with a JSON body containing the query results.

### 8. Scalability and Performance

- **Write Performance:** The LSM Tree design excels at writes. Appending to the WAL and updating the in-memory Memtable are extremely fast operations. Disk flushes and compactions are handled in the background, isolating them from the write path latency.
- **Query Performance:** The columnar storage format is key. A query that only needs one field (`usage_user`) doesn't need to read the data for other fields (`usage_system`), dramatically reducing I/O.
- **Compression:** Time series data is highly compressible. Timestamps often have a fixed delta, and values often don't change much between points. Specialized algorithms like Gorilla compression (for floats) and Simple8b (for integers) achieve very high compression ratios.
- **Clustering:** For horizontal scaling, data can be sharded across multiple nodes based on a hash of the time series (measurement + tags). A distributed query engine would be responsible for sending sub-queries to the relevant nodes and merging the results.

### 9. Reliability and Fault Tolerance

- **Durability:** The Write-Ahead Log (WAL) ensures that once a write is acknowledged, it will survive a crash. On restart, the node replays the WAL to rebuild its Memtable.
- **Data Integrity:** Checksums are used within the TSM files to detect block corruption.
- **Clustering and Replication:** In a clustered setup, data shards would be replicated across multiple nodes using a consensus protocol like Raft. If a node holding the master shard fails, a replica is promoted to master, ensuring high availability.

### 10. Security Considerations

- **Authentication:** The API should be protected by a token-based authentication system.
- **Authorization:** An ACL system can provide fine-grained permissions, allowing users/roles to read or write only to specific measurements.

### 11. Monitoring and Observability

- **Internal Metrics:** The TSDB should expose its own internal metrics as a time series (eating its own dog food). Key metrics include:
  - Ingestion rate (points/sec).
  - Query latency (p95, p99).
  - Compaction throughput.
  - WAL size, Memtable size.
  - Index cardinality.

### 12. Deployment and Operations

- **Backup and Restore:** A backup tool can be used to create a snapshot of all TSM and WAL files. Restoration involves replacing the files and restarting the node.
- **Downsampling:** Continuous queries run in the background to pre-compute rollups. For example, a query runs every hour to calculate the hourly average of the last hour's minute-level data and writes the result back into a new measurement.
- **Retention Policies:** A background process periodically checks for data that is older than its retention policy and deletes the entire TSM files that contain only expired data.

### 13. Cost Estimation

- **Primary Driver:** Storage (fast SSDs are recommended for performance). The high compression ratio is a key feature for managing this cost.
- **Memory:** The size of the index is directly related to the series cardinality. High-cardinality datasets require significant RAM.
- **Compute:** CPU is needed for ingestion, querying, and especially for background compaction.

### 14. Risks and Mitigations

- **High Cardinality "Tsunami":**
  - **Risk:** An application bug or a new workload writes data with highly dynamic tags (e.g., using a request ID as a tag), causing the number of unique time series to explode. This can exhaust memory and bring the database to a halt.
  - **Mitigation:** Implement cardinality limits and monitoring. Provide tools to help users identify and drop high-cardinality series. Design the index (Series File) to be highly efficient and partially disk-based.
- **Compaction Lag:**
  - **Risk:** The ingestion rate is so high that the background compaction process cannot keep up. This leads to a large number of small TSM files, which severely degrades query performance.
  - **Mitigation:** Provision sufficient CPU and I/O resources for compaction. Allow operators to manually trigger compactions or tune the compaction strategy.

### 15. Final Considerations

A Time Series Database is a highly specialized system that makes specific trade-offs to solve a specific problem well. Its architecture, combining a Write-Ahead Log, an LSM-Tree-like structure, and a columnar storage format, is a proven model for achieving massive write throughput and fast analytical queries. The single biggest challenge in operating a TSDB at scale is managing series cardinality, which requires both a robust technical design and good user practices.
