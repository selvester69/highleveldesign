# High-Level Design: Event Streaming Platform

## 1. Document Information

- **Document Title**: Event Streaming Platform - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a distributed, partitioned, replicated commit log service, architecturally similar to Apache Kafka. The platform is designed for high-throughput, low-latency ingestion and consumption of event streams. It will serve as a foundational infrastructure component for real-time data pipelines, event-driven architectures, and microservices communication.

### 2.2 Scope

**In Scope:**
-   A distributed cluster of "brokers" that manage the data.
-   The concept of "topics" as feeds of events.
-   Partitioning of topics for parallel processing and scalability.
-   Replication of partitions for durability and high availability.
-   A producer API for publishing messages to topics.
-   A consumer API for subscribing to and reading messages from topics.
-   A mechanism for coordinating the cluster and managing partition leadership.

**Out of Scope:**
-   A stream processing library (like Kafka Streams or Flink).
-   A schema registry.
-   Connectors to other systems (like Kafka Connect).
-   The client-side producer and consumer libraries themselves (we design the APIs they will use).

### 2.3 High-Level Architecture Overview
The system is a distributed cluster of **Brokers**. Data is organized into **Topics**. Each topic is split into multiple **Partitions**, which are ordered, immutable commit logs. For fault tolerance, each partition is replicated across multiple brokers. One broker acts as the **Leader** for a given partition, handling all reads and writes, while other brokers act as **Followers**, passively replicating the data. A **Coordination Service** (e.g., ZooKeeper or a self-managed Raft-based quorum) is used to manage cluster membership, elect partition leaders, and store metadata. **Producers** write data to the leader broker for a partition, and **Consumers** read from it, tracking their position in the log via an "offset".

---

## 3. System Overview

### 3.1 Business Context
In modern data-intensive applications, the ability to reliably move massive volumes of data in real-time is a core business need. An event streaming platform acts as the central nervous system for a company's data, enabling everything from website activity tracking and operational metrics to complex event-driven microservices. Building this as a platform provides a unified, scalable, and reliable solution for all teams.

### 3.2 System Purpose
To provide a durable, scalable, and high-performance platform for publishing and subscribing to streams of events (a "distributed commit log"). The system must guarantee message ordering within a partition and provide configurable durability.

### 3.3 Success Criteria
-   **High Throughput**: The system must support a sustained throughput of millions of messages per second on a moderately sized cluster.
-   **Low Latency**: P99 latency for a producer to publish a message and receive acknowledgement should be under 10ms.
-   **Durability**: Once a message is acknowledged by the leader and its in-sync replicas, it must not be lost, even in the event of a leader failure.
-   **Scalability**: The system must be able to scale horizontally by adding more brokers to handle more topics, partitions, and higher throughput.
-   **High Availability**: The failure of a single broker should not make any data unavailable.

### 3.4 Assumptions
-   The system will run on commodity hardware in a trusted data center environment.
-   Producers and consumers will be "smart" clients that can handle routing requests to the correct broker.
-   A small amount of downtime for the coordination service is acceptable, though it may temporarily prevent leader elections.

### 3.5 Constraints
-   **Disk I/O is the Bottleneck**: The system's performance is fundamentally bound by how fast it can write to and read from disk. The design must optimize for sequential disk access.
-   **Network Bandwidth**: Replication traffic between brokers can be substantial and must be factored into network capacity planning.
-   **Consistency vs. Availability**: The system must provide strong guarantees, which requires careful management of the trade-offs between consistency and availability, especially regarding producer acknowledgements.
-   **Complexity of Distributed Consensus**: Maintaining cluster state, handling failures, and electing leaders is a complex distributed systems problem.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Publishing)**: Producers must be able to publish messages to a specific topic. The producer should be able to choose the partition or have it assigned automatically.
-   **FR-002 (Subscribing)**: Consumers must be able to subscribe to one or more topics.
-   **FR-003 (Partitioning)**: Topics must be divisible into multiple partitions to allow for parallel consumption.
-   **FR-004 (Ordering Guarantee)**: The system must guarantee that all messages within a single partition are stored and delivered to consumers in the order they were produced.
-   **FR-005 (Replication)**: The system must replicate each partition to a configurable number of brokers.
-   **FR-006 (Leadership)**: For each partition, one replica must be designated the "leader". Only the leader can handle writes from producers.
-   **FR-007 (Offset Tracking)**: Consumers must be responsible for tracking their position (offset) in each partition's log. The system should provide a mechanism for consumers to commit their offsets.
-   **FR-008 (Data Retention)**: The system must retain data for a configurable period of time (e.g., 7 days) or until a topic reaches a certain size.

### 4.2 Non-Functional Requirements
-   **Throughput**: Sustain >1 million messages/sec per broker for sequential writes.
-   **Latency**: P99 produce latency < 10ms.
-   **Durability**: No data loss for acknowledged messages.
-   **Availability**: 99.99% uptime for the data plane.
-   **Scalability**: The cluster must be able to scale to hundreds of brokers.
-   **Efficiency**: The system must be highly efficient, leveraging OS features like the page cache and zero-copy reads.
-   **Fault Tolerance**: The system must automatically handle broker failures by electing a new leader for the partitions it hosted.

---

### 5. Core Architecture

The system is a cluster of brokers managed by a coordination service. Clients (producers/consumers) are smart and talk directly to the appropriate brokers.

- **Broker:** A server instance in the cluster. Each broker stores a set of partitions on its local disk. It handles all read/write requests for the partitions for which it is the leader and replicates data for partitions for which it is a follower.
- **Topic, Partition, and Log:** A topic is a logical feed name. It is composed of one or more partitions. A partition is a strictly ordered, append-only log. The log is implemented as a directory of segment files on disk. Messages are never modified.
- **Offset:** Each message in a partition log is assigned a unique, sequential integer ID called its offset.
- **Coordination Service (ZooKeeper):** A separate, highly available cluster used to manage the state of the broker cluster.
  - **Cluster Membership:** Which brokers are currently alive and part of the cluster.
  - **Topic Configuration:** The list of topics, their partition counts, and replication factors.
  - **Partition Leadership:** For each partition, which replica is the current leader. ZooKeeper's watch mechanism is used to notify brokers of changes (e.g., a new leader election).
- **Producer:** A client that publishes messages. The producer fetches metadata (which broker is the leader for its target partition) from a broker and then sends data directly to that leader. It can wait for an acknowledgement (`ack`) based on its durability requirements.
- **Consumer & Consumer Group:** A client that reads messages. Consumers subscribe to topics and are part of a `consumer group`. The system distributes the partitions of a topic among the consumers in a group, so each partition is consumed by exactly one consumer in the group. This allows for parallel, load-balanced consumption.

**Write Path:**
1.  A producer sends a message to the leader broker for a partition.
2.  The leader appends the message to its local log.
3.  Follower brokers send fetch requests to the leader to replicate the new message.
4.  Once the message has been replicated to the configured number of "in-sync replicas" (ISRs), the leader sends an acknowledgement (`ack`) to the producer.

**Read Path:**
1.  A consumer requests messages from the leader broker for a partition, specifying an offset.
2.  The broker reads the messages sequentially from the log file on disk.
3.  To optimize performance, the broker uses the OS's `sendfile` system call to transfer bytes directly from the page cache to the network socket (zero-copy), avoiding copying data into the application's user space.

![Architecture Diagram](https://i.imgur.com/vHqgCgL.png)

### 6. Data Schema and Models

- **Message Format:** A binary format containing:
  - `offset` (int64)
  - `size` (int32)
  - `crc32` (int32)
  - `magic_byte` (int8) - for versioning
  - `attributes` (int8) - e.g., for compression type
  - `timestamp` (int64)
  - `key` (bytes)
  - `value` (bytes)
- **Log Segments:** The log for a partition is broken into segment files on disk. A new segment is created when the current one reaches a certain size (e.g., 1GB) or age. This simplifies data retention; to delete old data, the system just deletes the entire segment file. Each segment has an associated index file that maps offsets to byte positions in the segment file for fast lookups.

### 7. API Design

The API is a custom binary protocol over TCP, designed for performance.

- **`Produce` Request:**
  - **Sent by:** Producer
  - **To:** Leader broker for a partition.
  - **Payload:** Topic name, partition ID, and a batch of messages.
  - **Response:** Success/error code. For success, the offset of the first message in the batch.

- **`Fetch` Request:**
  - **Sent by:** Consumer (or follower replica)
  - **To:** Leader broker for a partition.
  - **Payload:** Topic name, partition ID, fetch offset, max bytes to return.
  - **Response:** A batch of messages starting from the fetch offset.

- **`Metadata` Request:**
  - **Sent by:** Any client.
  - **To:** Any broker.
  - **Payload:** List of topics to get metadata for.
  - **Response:** The set of partitions for the topics, the leader for each partition, and the list of replicas.

- **`OffsetCommit` Request:**
  - **Sent by:** Consumer.
  - **To:** A designated "group coordinator" broker.
  - **Payload:** Consumer group ID, topic, partition, and the committed offset.
  - **Response:** Success/error code.

### 8. Scalability and Performance

- **Sequential I/O:** The core design principle is to turn all writes into sequential appends to the disk log. This is much faster than random writes and allows the system to achieve performance comparable to an in-memory queue.
- **Zero-Copy:** The read path is optimized using the `sendfile` system call, which avoids expensive data copies between kernel space and user space.
- **Partitioning:** Throughput scales linearly with the number of partitions. A topic with 100 partitions can be consumed by a consumer group with up to 100 parallel consumers.
- **Batching:** Producers and consumers are designed to work with batches of messages, which amortizes network and disk I/O overhead over many messages.

### 9. Reliability and Fault Tolerance

- **Replication & ISRs:** A partition's "in-sync replica" (ISR) list is the set of follower replicas that are caught up with the leader. A write is only considered committed when it has been replicated to all replicas in the ISR list.
- **Leader Election:** If a leader broker fails, the coordination service (ZooKeeper) detects its absence. A new leader is elected from the ISR list. This ensures that the new leader has all the committed messages.
- **Producer Acknowledgements:** Producers can configure the level of durability required:
  - `acks=0`: Fire and forget. Lowest latency, no durability guarantee.
  - `acks=1`: The leader acknowledges the write. Durable if the leader doesn't fail.
  - `acks=all`: The leader and all ISRs acknowledge the write. Highest durability, highest latency.

### 10. Security Considerations

- **Network Security:** Brokers should be run in a trusted network. Communication can be encrypted using TLS.
- **Authentication & Authorization:** The platform can integrate with SASL for authentication. Access Control Lists (ACLs) can be defined to control which users/applications can read from or write to which topics.

### 11. Monitoring and Observability

- **Broker Metrics:** Disk usage, network I/O, CPU usage.
- **Topic/Partition Metrics:** Message rate, byte rate, log size.
- **Consumer Metrics:** Consumer lag (the difference between the last offset in the log and the consumer's committed offset) is a critical metric to monitor. High lag indicates that consumers cannot keep up with producers.
- **Replication Metrics:** The number of under-replicated partitions and the size of the ISR list for each partition.

### 12. Deployment and Operations

- **Cluster Management:** Adding a broker to the cluster is as simple as starting the broker process with the correct configuration.
- **Topic Management:** Tools are provided to create, delete, and alter topics (e.g., change the number of partitions).
- **Rebalancing:** When a new consumer joins or leaves a group, the partitions are automatically rebalanced among the remaining consumers.

### 13. Cost Estimation

- **Primary Drivers:**
  - **Storage:** The cost of the disks on the broker machines, determined by the message volume and retention period.
  - **Compute:** The number of broker instances needed to handle the I/O and CPU load.
  - **Network:** The cost of bandwidth for replication and client traffic.

### 14. Risks and Mitigations

- **Coordination Service Failure:**
  - **Risk:** If ZooKeeper is down, the cluster cannot elect new leaders or change configuration. The data plane (producing/consuming) will continue to function for existing leaders.
  - **Mitigation:** Run ZooKeeper as a highly available cluster itself (e.g., a 5-node ensemble).
- **"Hot" Partitions:**
  - **Risk:** A single partition receives a disproportionate amount of traffic, overwhelming its leader broker.
  - **Mitigation:** Choose a good partitioning key to distribute data evenly. Monitor for hot spots and consider increasing the partition count for the topic.
- **Loss of Quorum in ISRs:**
  - **Risk:** Multiple replicas for a partition fail, leaving fewer than the minimum required number of in-sync replicas. The system may have to choose between halting writes (prioritizing consistency) or allowing writes to an under-replicated partition (prioritizing availability).
  - **Mitigation:** This is a fundamental trade-off. The system provides the configuration (`unclean.leader.election.enable`) to let the operator make this choice.

### 15. Final Considerations

This design for an event streaming platform mirrors the core principles of Apache Kafka. The key innovation is the abstraction of a partitioned, replicated commit log. This simple but powerful abstraction provides a foundation for a huge variety of use cases. The performance comes from optimizing for sequential disk I/O and leveraging OS-level features, while the reliability comes from the in-sync replica and leader election protocol managed by a coordination service.
