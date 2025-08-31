# High-Level Design: Distributed Lock Manager

## 1. Document Information

- **Document Title**: Distributed Lock Manager - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a Distributed Lock Manager service. The system is designed to provide a reliable mechanism for enforcing mutual exclusion among processes in a distributed environment, ensuring that only one process can access a critical resource at a time. The design is heavily based on leveraging an underlying, strongly consistent key-value store, such as etcd or ZooKeeper, to handle the core consensus and fault tolerance.

### 2.2 Scope

**In Scope:**
-   An API for acquiring and releasing locks.
-   Support for both exclusive (write) and shared (read) locks.
-   A fault-tolerant locking mechanism that can survive client and server failures.
-   A "lease" mechanism to automatically release locks held by failed clients.
-   A solution to the "thundering herd" problem, where many clients waiting for a lock are notified at once.

**Out of Scope:**
-   The underlying distributed key-value store (e.g., etcd). We are designing the locking logic *on top* of such a system.
-   The client applications that use the lock manager.
-   Deadlock detection and resolution within the lock manager itself.

### 2.3 High-Level Architecture Overview
The system is implemented as a service layer on top of a **Distributed Key-Value Store** (like etcd). To acquire a lock for a resource named `my-lock`, a client attempts to create a unique, ephemeral key (e.g., `/locks/my-lock/1`) in the key-value store. The key with the lowest sequential number in the directory is considered the lock holder. Other clients "watch" for the deletion of the key preceding their own. When a client releases the lock (or its session expires and the ephemeral key is deleted), the next client in the queue is notified and acquires the lock. This approach avoids polling and provides fairness. **Leases** (a feature of the underlying store) are used to tie the lifetime of the lock key to the client's session, ensuring locks are automatically released if a client crashes.

---

## 3. System Overview

### 3.1 Business Context
In distributed systems, multiple processes often need to coordinate access to a shared resource (e.g., updating a record in a non-transactional database, performing a maintenance task on a singleton service). A distributed lock manager provides a fundamental synchronization primitive to prevent "split-brain" scenarios and ensure data consistency in these situations. It is a critical building block for creating reliable distributed applications.

### 3.2 System Purpose
To provide a simple, fault-tolerant, and highly available API for distributed mutual exclusion, allowing clients to safely coordinate access to shared resources.

### 3.3 Success Criteria
-   **Mutual Exclusion**: At any given time, for a given lock, only one client can hold an exclusive lock.
-   **No Deadlocks**: The basic locking protocol must be free from deadlocks (though client application logic can still create them).
-   **Fault Tolerance**: The failure of a lock-holding client must not cause the lock to be held indefinitely. The lock manager service itself must remain available even if some of its nodes fail.
-   **Fairness**: Locks should be granted in a fair, first-come, first-served manner.

### 3.4 Assumptions
-   A highly available, strongly consistent key-value store (like etcd or ZooKeeper) is available as a dependency.
-   Clients are reasonably well-behaved and will attempt to release locks they acquire.
-   Network latency between clients and the lock service is relatively low.

### 3.5 Constraints
-   **Performance Overhead**: Acquiring and releasing a lock involves at least one round-trip to the underlying consensus-based store, which has inherent latency. This is not a high-performance, fine-grained locking mechanism.
-   **Dependency on Key-Value Store**: The availability and performance of the lock manager are entirely dependent on the availability and performance of the underlying key-value store.
-   **Clock Skew**: The system should not rely on synchronized clocks for correctness, as clock drift is a reality in distributed systems.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Exclusive Lock Acquisition)**: A client must be able to request an exclusive lock on a named resource. The call should block until the lock is acquired or a timeout occurs.
-   **FR-002 (Lock Release)**: A client that holds a lock must be able to release it.
-   **FR-003 (Automatic Release on Failure)**: If a client holding a lock fails (crashes or is partitioned from the network), the system must automatically release the lock after a specified lease period expires.
-   **FR-004 (Fair Queuing)**: If multiple clients are waiting for a lock, they should be granted the lock in the order they requested it (FIFO).
-   **FR-005 (Shared Lock Support)**: The system should support shared (read) locks, allowing multiple clients to hold a read lock simultaneously, but blocking any exclusive (write) locks.
-   **FR-006 (Non-Blocking Acquisition)**: The API should provide a non-blocking `try_lock` variant that immediately returns success or failure.

### 4.2 Non-Functional Requirements
-   **High Availability**: The lock service must be as available as its underlying key-value store.
-   **Fault Tolerance**: The system must tolerate client failures and a minority of server failures.
-   **Performance**: Lock acquisition latency should be low, dominated primarily by the underlying store's write latency.
-   **Scalability**: The system should be able to manage tens of thousands of concurrent locks.
-   **Correctness**: The mutual exclusion property must be strictly guaranteed.
-   **Ease of Use**: The client-facing API should be simple and abstract away the complexity of the underlying protocol.

---

### 5. Core Architecture

The architecture is a client-side library and a thin service layer that implements the locking recipe on top of a foundational distributed key-value store like etcd or ZooKeeper.

- **Underlying Key-Value Store (e.g., etcd):**
  - Provides the core primitives needed:
    - **Strong Consistency:** All operations are linearizable.
    - **Atomic Compare-and-Swap (CAS):** Operations can be made conditional on the current state of a key.
    - **Ephemeral Nodes/Keys:** Keys can be created with a "lease" attached. If the client fails to renew the lease (e.g., because it crashed), the key is automatically deleted. This is the key to fault tolerance.
    - **Watch Mechanism:** Clients can watch a key or prefix for changes (creations, deletions, updates).
    - **Sequential Keys:** The ability to create a key with a monotonically increasing sequence number appended.

- **Lock Service / Client Library:**
  - This contains the logic for the locking protocol. It can be a standalone service that clients call, or more commonly, a library that clients embed directly.
  - **Lock Acquisition (`acquire(lock_name)`) Workflow:**
    1.  The client connects to the key-value store and creates a new, sequential, ephemeral key under the desired lock path. For example, `PUT /locks/{lock_name}/<uuid>-<seq> with lease`. The UUID ensures uniqueness, and the sequence number is assigned by the store.
    2.  The client then gets a list of all keys currently under `/locks/{lock_name}/`.
    3.  If the client's key has the lowest sequence number, it has successfully acquired the lock, and the function returns.
    4.  If its key is *not* the lowest, the client has *not* acquired the lock. It enters a waiting state.
    5.  To avoid the "thundering herd" problem (where all waiters poll the lock), the client places a **watch** specifically on the key immediately preceding its own in the sequence.
    6.  The client's `acquire` call now blocks. When the preceding key is deleted (because that client released the lock or crashed), the watch is triggered.
    7.  The watcher wakes up, checks if it is now the new lowest-sequence key holder, and if so, acquires the lock and returns.

- **Lock Release (`release(lock_name)`) Workflow:**
  - The client simply deletes its ephemeral key from the key-value store (e.g., `DELETE /locks/{lock_name}/<uuid>-<seq>`). This action triggers the watch for the next client in the queue.

![Architecture Diagram](https://i.imgur.com/3D1vY2M.png)

### 6. Data Schema and Models

- **Key Structure in the KV Store:**
  - A base path for all locks, e.g., `/locks/`.
  - Under this, a directory for each named lock, e.g., `/locks/database-migration/`.
  - Within each lock directory, a set of sequential, ephemeral keys represent the queue of waiters.
    - `/locks/database-migration/client-abc-0000000001` (Lock holder)
    - `/locks/database-migration/client-def-0000000002` (Waiter 1)
    - `/locks/database-migration/client-ghi-0000000003` (Waiter 2)
- **Lock Data:** The value of the key can store metadata, such as the client's hostname, the time of acquisition, and the lock type (exclusive/shared).

### 7. API Design

The client library would expose a simple, intuitive API.

- **`lock = LockManager.new_lock('my-resource')`**
- **`lock.acquire(timeout=30)`:**
  - Attempts to acquire the lock, blocking for up to 30 seconds. Returns `True` on success, `False` on timeout.
- **`lock.release()`:**
  - Releases the lock if held.
- **`with LockManager.new_lock('my-resource') as lock:`**
  - A context manager for automatically acquiring and releasing the lock.

### 8. Scalability and Performance

- **Scalability:** The system's scalability is primarily that of the underlying key-value store. It can handle a large number of different locks. However, contention on a single lock creates a serialized bottleneck by design.
- **Performance:**
  - **Lock Acquisition:** Requires several round trips to the KV store (create key, get children, set watch). Latency is directly tied to the KV store's performance.
  - **Lock Release:** A single, fast delete operation.
  - The watch-based mechanism is highly performant as it avoids client polling.

### 9. Reliability and Fault Tolerance

- **Server-Side Fault Tolerance:** Inherited directly from the underlying key-value store (e.g., etcd is tolerant to `(N-1)/2` failures).
- **Client-Side Fault Tolerance:** This is the key feature. By using ephemeral keys tied to a client's session/lease, the lock is **guaranteed** to be released if the client crashes or is partitioned from the network. The KV store's lease mechanism acts as a "dead man's switch." This prevents locks from being held indefinitely, a major problem in distributed systems.

### 10. Security Considerations

- **Authentication/Authorization:** The client must authenticate with the key-value store. ACLs within the store can be used to control which clients are allowed to request which locks (e.g., by controlling access to key prefixes).
- **Denial of Service:** A malicious client could attempt to create millions of locks, potentially overwhelming the KV store. Rate limiting and monitoring are required.

### 11. Monitoring and Observability

- **Lock Contention:** Monitor the queue depth (number of keys) for popular locks. High contention may indicate a performance bottleneck in the application logic.
- **Lock Hold Time:** Log and monitor the duration for which locks are held. Long-held locks can reduce system throughput.
- **KV Store Metrics:** Closely monitor the health, latency, and error rates of the underlying etcd/ZooKeeper cluster.

### 12. Deployment and Operations

- **Dependency Management:** The primary operational task is managing the health of the underlying key-value store. The lock manager itself is often just a library within client applications.
- **Client-Side Logic:** Ensuring all clients use a correct and tested implementation of the locking recipe is crucial. A small bug in the client library can violate mutual exclusion guarantees.

### 13. Cost Estimation

- **Primary Driver:** The cost of running and maintaining the highly available key-value store cluster (e.g., 3-5 dedicated server nodes).

### 14. Risks and Mitigations

- **Underlying Store Outage:**
  - **Risk:** The etcd/ZooKeeper cluster goes down.
  - **Mitigation:** No new locks can be acquired or released. The system fails "safe" in that it doesn't grant multiple exclusive locks, but it violates liveness. The only mitigation is to make the underlying store as resilient as possible.
- **Application-Level Deadlock:**
  - **Risk:** Client A acquires lock 1 and waits for lock 2. Client B acquires lock 2 and waits for lock 1.
  - **Mitigation:** This is a classic deadlock that the lock manager cannot solve. The application developers must ensure a consistent lock acquisition order across all clients to prevent this.
- **Long-Held Locks:**
  - **Risk:** A client acquires a lock and performs a very long-running or stalled operation, blocking all other clients.
  - **Mitigation:** Implement timeouts on lock acquisition. Add monitoring and alerting for locks held longer than a defined threshold.

### 15. Final Considerations

A distributed lock manager is a powerful but "heavy" tool. It should be used for coarse-grained locking where strong correctness guarantees are needed, and the overhead of network round-trips is acceptable. The recipe of using sequential, ephemeral nodes in a system like ZooKeeper or etcd is a standard, proven pattern for building fault-tolerant distributed locks. It elegantly solves the problems of fairness (FIFO queueing) and fault tolerance (automatic release on client failure) by leveraging the powerful primitives provided by the underlying consensus-based store.
