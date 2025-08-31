# High-Level Design: Graph Database

## 1. Document Information

- **Document Title**: Graph Database - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a native graph database, architecturally similar to Neo4j. The system is designed to store data as a "property graph," consisting of nodes, relationships (edges), and properties (key-value pairs) attached to both. Its primary purpose is to provide an efficient and intuitive way to model, store, and query highly connected data, making it ideal for use cases like social networks, fraud detection, and recommendation engines.

### 2.2 Scope

**In Scope:**
-   A data model based on the Labeled Property Graph model (nodes, relationships, properties, labels).
-   A native graph storage engine that enables "index-free adjacency."
-   A declarative query language designed for graph traversal (e.g., Cypher-like).
-   A transactional engine supporting ACID properties for database operations.
-   Indexing on node/relationship properties for fast lookups of starting points for traversals.

**Out of Scope:**
-   Massively distributed, sharded graph processing (the design focuses on a single-server or primary/secondary cluster model).
-   The client driver for connecting to the database.
-   Graph visualization tools.
-   Graph analytics algorithms (e.g., PageRank) as a built-in feature, though they can be implemented using the query language.

### 2.3 High-Level Architecture Overview
The core of the system is a **Native Graph Storage Engine**. Unlike other databases that represent relationships by joining tables, this engine stores nodes and relationships as specific records with direct physical pointers to each other. A node record contains a pointer to its first relationship, and each relationship record points to the next relationship for its start and end nodes, forming a linked list of relationships for each node. This "index-free adjacency" allows for extremely fast traversal from one node to its neighbors. On top of this storage engine sits a **Transaction Manager** ensuring ACID compliance and a **Query Processor** that executes a declarative graph query language (like Cypher) by translating it into a series of efficient, low-level traversal operations on the storage engine.

---

## 3. System Overview

### 3.1 Business Context
Many modern datasets are not flat tables but are rich, complex networks of interconnected entities. Examples include social graphs, financial transaction graphs, and knowledge graphs. Traditional relational databases struggle to handle these datasets, as "join-heavy" queries become exponentially slower as the depth of the query increases. A graph database provides a natural and high-performance way to work with this connected data, unlocking insights that would otherwise be impractical to find.

### 3.2 System Purpose
To provide a database management system specifically designed to store and query graph data efficiently. The system prioritizes the performance of traversal queries (e.g., "find friends of friends") over the large-scale analytical aggregations that a relational or columnar database would excel at.

### 3.3 Success Criteria
-   **High-Performance Traversals**: The time taken to traverse from one node to its neighbors should be constant (O(1)), regardless of the total number of nodes in the graph.
-   **ACID Compliance**: The database must support fully ACID transactions.
-   **Expressive Query Language**: The query language should make it easy and intuitive to express complex graph patterns and traversals.
-   **Scalability**: The system should be able to store graphs with billions of nodes and relationships on a single server.
-   **Data Integrity**: The graph model must remain consistent, with no dangling relationships or corrupted records.

### 3.4 Assumptions
-   The data is highly connected. If the data has few relationships, a relational database might be a better choice.
-   The primary workload consists of transactional queries and graph traversals, not large-scale scans and aggregations of the entire dataset.
-   The entire graph dataset can fit on a single, powerful server (vertical scaling).

### 3.5 Constraints
-   **Traversal-Focused**: The storage engine is highly optimized for traversals. Queries that require scanning all nodes or relationships to find a property (a "global scan") can be slow if not supported by an index.
-   **Sharding Complexity**: Sharding a graph across multiple machines is notoriously difficult because random traversals can lead to a high number of network hops between servers ("supernode" problem). This design focuses on a single-server model first.
-   **Data Model**: The property graph model is flexible but less structured than a relational schema, requiring careful data modeling from the application side.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Node Management)**: The system must allow for the creation, retrieval, updating, and deletion (CRUD) of nodes.
-   **FR-002 (Relationship Management)**: The system must allow for CRUD operations on relationships between nodes. A relationship must have a start node, an end node, and a type.
-   **FR-003 (Property Management)**: The system must allow for setting and removing key-value properties on both nodes and relationships.
-   **FR-004 (Labels)**: The system must allow one or more labels to be assigned to a node (e.g., `:Person`, `:User`), which can be used for grouping and indexing.
-   **FR-005 (Graph Query Language)**: The system must provide a declarative query language to express graph patterns. Example: `MATCH (user:User {name: 'Alice'})-[:FRIENDS_WITH]->(friend) RETURN friend.name`.
-   **FR-006 (Indexing)**: The system must allow the creation of indexes on node properties (e.g., on `User.name`) to provide fast entry points into the graph.
-   **FR-007 (Transactions)**: All data modification operations must be performed within an ACID-compliant transaction.

### 4.2 Non-Functional Requirements
-   **Performance (Index-Free Adjacency)**: The performance of traversing from a node to its adjacent nodes should be independent of the total graph size.
-   **Durability**: Committed transactions must be durable and survive a database crash.
-   **Concurrency**: The system must handle multiple concurrent readers and writers using a concurrency control mechanism like Multi-Version Concurrency Control (MVCC).
-   **Scalability**: The system must scale vertically to take advantage of machines with large amounts of RAM and fast storage.
-   **Data Consistency**: The database must enforce schema constraints where specified and ensure the structural integrity of the graph.

---

### 5. Core Architecture

The architecture is built around a native graph storage engine that physically stores graph structures, enabling high-performance traversals.

- **Query Processor:**
  - **Parser:** Parses the declarative query string (e.g., Cypher) into an Abstract Syntax Tree (AST).
  - **Planner:** Converts the AST into a physical execution plan. It uses statistics and indexes to find the most efficient way to execute the query (e.g., "should I start the traversal from the `:User` node or the `:Movie` node?").
  - **Executor:** Executes the plan against the transaction manager, producing a stream of results.
- **Transaction Manager:**
  - Manages the lifecycle of transactions (begin, commit, rollback).
  - Ensures ACID properties, typically using Multi-Version Concurrency Control (MVCC) to allow concurrent reads and writes without locks.
  - Manages the Write-Ahead Log (WAL) for durability.
- **Cache Manager:**
  - Manages an in-memory page cache that holds frequently accessed parts of the graph (nodes, relationships, properties). This is crucial for performance.
- **Storage Engine (The Core):**
  - Consists of several fixed-size record files stored on disk.
  - **Node Store (`nodes.db`):** A file containing fixed-size node records. A node's ID is its byte offset into this file.
  - **Relationship Store (`rels.db`):** A file containing fixed-size relationship records.
  - **Property Store (`props.db`):** A file containing property data, linked from node and relationship records.
  - **Index Files:** B-tree or Lucene indexes for fast lookups of nodes by their properties.
- **Index-Free Adjacency:**
  - A node record at a specific ID (e.g., ID 5) contains a pointer (another record ID) to its first relationship.
  - That relationship record points to the start and end nodes, its type, and has two pointers: one to the *next* relationship for the start node and one to the *next* relationship for the end node.
  - This creates two doubly linked lists of relationships for every node, allowing traversals by following these pointers directly, without needing a separate index lookup (like a foreign key join in SQL).

![Architecture Diagram](https://i.imgur.com/nJgqL8V.png)

### 6. Data Schema and Models

- **On-Disk Records:**
  - **Node Record (~15 bytes):**
    - `in_use` (1 bit)
    - `first_relationship_id` (pointer)
    - `first_property_id` (pointer)
    - `labels` (pointer to a label store)
  - **Relationship Record (~34 bytes):**
    - `in_use` (1 bit)
    - `start_node_id` (pointer)
    - `end_node_id` (pointer)
    - `type` (pointer to a type store)
    - `first_property_id` (pointer)
    - `next_rel_id_for_start_node` (pointer)
    - `prev_rel_id_for_start_node` (pointer)
    - `next_rel_id_for_end_node` (pointer)
    - `prev_rel_id_for_end_node` (pointer)
  - **Property Record:** A variable-length record containing key-value data, linked together in a list for a given node/relationship.

### 7. API Design

- **Primary API:** A TCP-based protocol (e.g., Bolt) for executing queries.
- **Query Language (Cypher-like):**
  - **Read Query:**
    `MATCH (p:Person {name: 'Alice'})-[:KNOWS]->(f:Person) WHERE f.age > 30 RETURN f.name`
  - **Write Query:**
    `CREATE (p:Person {name: 'Bob', age: 35})`
  - **Pattern Matching:** The core of the language is `MATCH`, which describes a graph pattern to find.

### 8. Scalability and Performance

- **Vertical Scaling:** The primary scaling strategy. The system is designed to perform best when the graph data fits into the main memory (page cache) of a single large server.
- **Index-Free Adjacency:** This is the key to performance. Traversing a relationship is a simple pointer-chasing operation, which is extremely fast and has O(1) complexity. A query like "friends of friends of friends" remains fast, whereas it would require multiple expensive joins in a relational database.
- **Indexes:** B-tree indexes are crucial for finding the starting points of a traversal (e.g., finding the "Alice" node quickly). They are not used for the traversal itself.

### 9. Reliability and Fault Tolerance

- **ACID Transactions:** The use of a Write-Ahead Log (WAL) and a proper transaction manager ensures durability and consistency. A transaction is first written to the WAL, and only then is the main database file modified. If the server crashes, the WAL is replayed on restart to bring the database to a consistent state.
- **High Availability (Clustering):** While not sharded, the database can be deployed in a high-availability (HA) cluster.
  - A Primary/Secondary (Leader/Follower) model is used.
  - The Primary handles all write operations.
  - Transactions are replicated from the Primary to the Secondaries.
  - Secondaries can handle read queries, providing read scaling.
  - If the Primary fails, a Secondary is promoted to be the new Primary (automated failover).

### 10. Security Considerations

- **Authentication and Authorization:** The database should have a built-in security model with users and roles. Roles can be granted specific privileges (e.g., read-only, write access to certain data, schema admin).
- **Encryption:** Support for encryption of data at rest (the database files) and in transit (TLS for the query protocol).

### 11. Monitoring and Observability

- **Transaction Metrics:** Transaction logs, number of open transactions, lock contention.
- **Cache Metrics:** Page cache hit/miss ratio. A high hit ratio is critical for performance.
- **Query Log:** Log slow queries to identify performance bottlenecks.
- **Store File Sizes:** Monitor the size of the node, relationship, and property stores.

### 12. Deployment and Operations

- **Backup/Restore:** A robust backup mechanism is needed that can take a consistent snapshot of the database files, even while the database is online.
- **Memory Tuning:** Configuring the page cache size is the most important operational task for performance.

### 13. Cost Estimation

- **Primary Driver:** A single, powerful server with a very large amount of RAM. This can be more expensive upfront than a cluster of smaller commodity machines.
- **Storage:** Fast storage (SSDs/NVMe) is required for good performance.

### 14. Risks and Mitigations

- **"Super Nodes" (or "Hub Nodes"):**
  - **Risk:** A node with millions of relationships (e.g., a celebrity in a social network). Traversing from this node can be slow as it involves iterating over a very long linked list of relationships.
  - **Mitigation:** This is a data modeling problem. The model should be adjusted to avoid super nodes, for example by splitting a single `FOLLOWS` relationship type into `FOLLOWS_2023_Q1`, `FOLLOWS_2023_Q2`, etc., to break up the linked lists.
- **Hitting the Vertical Scaling Limit:**
  - **Risk:** The graph grows too large to fit on a single server.
  - **Mitigation:** For read-heavy workloads, scale reads with more replicas. For write-heavy workloads that outgrow a single machine, the solution is much harder and involves moving to a sharded graph database, which has its own set of complex trade-offs (like distributed transactions and cross-shard traversals).

### 15. Final Considerations

A native graph database is a specialized tool designed for a specific type of data and workload. Its power comes from "index-free adjacency," which makes it unparalleled for deep, complex traversal queries. This architectural choice, however, makes horizontal sharding very difficult, leading to a "scale-up" (vertical scaling) rather than "scale-out" philosophy. For use cases with highly connected data where the relationships are as important as the data itself, this design provides a massive performance advantage over other database models.
