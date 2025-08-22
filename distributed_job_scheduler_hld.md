# High-Level Design Document

## Document Metadata
- **Document Title**: Distributed Job Scheduler - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-22
- **Author(s)**: Jules (AI Software Engineer)
- **Reviewers**: N/A
- **Approvers**: N/A
- **Document Status**: Draft

## Revision History
| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-08-22 | Jules | Initial version |

## Distribution List
- Development Team
- Operations Team
- Data Engineering Team

---

## 2. Executive Summary

### 2.1 Purpose
This document describes the high-level design for a distributed, fault-tolerant job scheduling and workflow orchestration platform. The system allows users to define, schedule, execute, and monitor complex data pipelines and other workflows as code. It is heavily inspired by platforms like Apache Airflow.

### 2.2 Scope
**Included**:
- Workflow definition as Directed Acyclic Graphs (DAGs).
- A scheduler to trigger workflows based on time or external events.
- A distributed pool of workers to execute tasks.
- A web-based user interface for monitoring and managing workflows.
- A metadata database to store the state of all workflows and tasks.
- Extensible architecture for defining custom operators and sensors.

**Excluded**:
- The actual business logic of the tasks themselves (provided by users).
- Big data processing frameworks (the scheduler orchestrates them, but does not implement them).

### 2.3 Key Benefits
- **Reliability**: Ensures that tasks are executed and retried upon failure.
- **Scalability**: Scales to handle thousands of concurrent workflows and tasks.
- **Flexibility**: Workflows are defined in code (e.g., Python), allowing for dynamic generation and complex logic.
- **Observability**: Provides a central UI to visualize dependencies, monitor progress, and inspect logs.

### 2.4 High-Level Architecture Overview
The architecture is composed of several key components: a Web Server (UI), a Scheduler, a Metadata Database, and a pool of Workers. The Scheduler periodically polls the metadata database to identify tasks that are ready to run and sends them to a message queue. Workers pick up tasks from the queue, execute them, and update their status in the metadata database.

---

## 3. System Overview

### 3.1 Business Context
In many organizations, complex business processes, data pipelines (ETL/ELT), and infrastructure management tasks need to be run on a regular schedule. A job scheduler automates this process, providing a robust and visible way to manage these workflows, handle dependencies, and recover from failures, which is critical for data-driven decision-making and operational efficiency.

### 3.2 System Purpose
To provide a platform for programmatically authoring, scheduling, and monitoring workflows. It enables users to build and run complex data pipelines that can interact with a wide variety of external systems.

### 3.3 Success Criteria
- **Reliability**: No missed schedules; >99.9% task execution success rate (including retries).
- **Scalability**: Support for 10,000+ active DAGs and 100,000+ daily task instances.
- **Latency**: Task scheduling latency (from trigger time to execution) should be < 1 minute.
- **Usability**: The UI should provide a clear, intuitive way to manage and monitor workflows.

### 3.4 Assumptions
- Workflows can be represented as Directed Acyclic Graphs (DAGs).
- Tasks are idempotent, meaning they can be re-run without causing unintended side effects.
- The metadata database is a reliable, high-availability component.

### 3.5 Constraints
- The scheduler is designed for batch-oriented workflows, not low-latency streaming applications.

### 3.6 Dependencies
- A relational database for metadata storage (e.g., PostgreSQL, MySQL).
- A message queue for task distribution (e.g., RabbitMQ, Redis, Celery).
- A reliable storage layer for logs (e.g., a network file system or cloud storage).

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
- **FR-001**: Users shall define workflows as DAGs in Python code.
- **FR-002**: The system shall parse DAG definition files to populate the metadata database.
- **FR-003**: The scheduler shall trigger DAG runs based on a defined schedule (e.g., cron expression).
- **FR-004**: The system shall execute tasks in the order defined by their dependencies within the DAG.
- **FR-005**: The system shall allow tasks to be retried automatically upon failure.
- **FR-006**: The UI shall display DAGs, their run history, and the status of each task.
- **FR-007**: Users shall be able to view logs for each executed task via the UI.
- **FR-008**: The system shall provide a mechanism for passing data between tasks (XComs).

### 4.2 Non-Functional Requirements
#### 4.2.1 Scalability Requirements
- The worker pool must be horizontally scalable to increase task execution capacity.
- The scheduler should be configurable for high availability (active-active mode).

#### 4.2.2 Availability Requirements
- **Uptime**: 99.9% for the control plane (Scheduler, Web UI).
- **Fault Tolerance**: The system must be able to tolerate worker node failures without losing task state.

#### 4.2.3 Security Requirements
- Role-Based Access Control (RBAC) for the UI to control user permissions.
- Secure storage for sensitive information like passwords and API keys (Connections & Variables).

---

## 5. Architecture Design

### 5.1 Architecture Principles
- **Configuration as Code**: Workflows are defined in code, enabling version control, collaboration, and testing.
- **Decoupled Components**: Components like the scheduler and workers are stateless and communicate via the metadata database and message queue, allowing them to be scaled independently.
- **Extensibility**: The system should be easy to extend with custom operators, hooks, and executors.

### 5.2 Architecture Patterns
- **Producer-Consumer**: The scheduler produces tasks, and workers consume them from a queue.
- **Stateful Core, Stateless Periphery**: The metadata database is the stateful core, while other components are largely stateless.

### 5.3 High-Level Architecture Diagram
```
+----------------+       +-------------------+
|      User      |------>|    Web Server     |
| (via Browser)  |       | (UI / API)        |
+----------------+       +---------+---------+
                                   |
                                   | (Read/Write State)
+----------------+       +---------v---------+       +-------------------+
|   DAG Files    |<----->|     Scheduler     |------>|   Message Queue   |
| (Python Code)  |       +---------+---------+       +---------+---------+
+----------------+                 |                           |
                                   | (Read/Write State)        | (Consume Tasks)
                         +---------v---------+       +---------v---------+
                         | Metadata Database |<----->|      Workers      |
                         |  (PostgreSQL)     |       | (Celery Executor) |
                         +-------------------+       +-------------------+
```

### 5.4 Component Overview
- **Web Server**: A web application that provides the UI for users to monitor DAGs, manage connections, and view logs.
- **Scheduler**: The heart of the system. It periodically scans the DAG files, checks schedules, and sends tasks that are ready to run to the message queue.
- **Metadata Database**: A relational database that stores all state, including DAG definitions, task instances, run history, connections, and variables.
- **Workers**: A pool of processes that listen to the message queue, execute the tasks they receive, and update the status in the metadata database.
- **Message Queue**: A broker that decouples the scheduler from the workers, providing a buffer for tasks.

### 5.5 Technology Stack
- **Backend Language**: Python.
- **Metadata Database**: PostgreSQL or MySQL.
- **Message Queue**: RabbitMQ or Redis (with Celery).
- **Web Framework**: Flask.

### 5.6 Architecture Decision Records (ADRs)
#### 5.6.1 ADR-001: Choice of Executor
- **Status**: Accepted
- **Context**: We need a mechanism to distribute tasks to workers.
- **Decision**: Use the Celery Executor. Celery is a mature, feature-rich distributed task queue that integrates well with Python and supports multiple message brokers.
- **Consequences**: Provides robust task distribution, scaling, and monitoring capabilities out of the box. Adds Celery and a message broker (e.g., RabbitMQ) as system dependencies.

---

## 6. Detailed Component Design

### 6.1 Component 1: Scheduler
#### 6.1.1 Purpose
To schedule and queue tasks for execution.
#### 6.1.2 Responsibilities
- Parse DAG definition files from the DAGs folder.
- Create DAG Run instances in the metadata database for scheduled DAGs.
- Evaluate task dependencies and identify "runnable" tasks.
- Enqueue runnable tasks on the message queue for workers to pick up.
- Can run in an active-active HA configuration for resilience.

### 6.2 Component 2: Worker
#### 6.2.1 Purpose
To execute the business logic of a single task.
#### 6.2.2 Responsibilities
- Dequeue a task message from the message queue.
- Execute the task's defined logic.
- Emit logs to a centralized location.
- Update the task's final status (success, failure, etc.) in the metadata database.

---

## 7. Data Design

### 7.1 Data Architecture
A normalized relational schema in the metadata database is used to track all system state.

### 7.2 Data Models (Key Tables)
- **dag**: Stores information about each DAG.
- **dag_run**: Represents a specific instance of a DAG running.
- **task_instance**: Represents a specific run of a task for a given `dag_run`. This is the central table for state tracking.
- **connection**: Stores connection details (host, user, password) for external systems.
- **variable**: Stores key-value pairs for configuration.

---

## 8. API Design

### 8.1 API Architecture
A RESTful API is exposed by the web server for programmatic interaction with the system.

### 8.2 Core APIs
- **`POST /api/v1/dags/{dag_id}/dagRuns`**: Trigger a new DAG run.
- **`GET /api/v1/dags/{dag_id}/dagRuns/{dag_run_id}`**: Get the status of a DAG run.
- **`GET /api/v1/dags/{dag_id}/dagRuns/{dag_run_id}/taskInstances`**: Get the status of all task instances in a DAG run.

---

## 9. Security Design

### 9.1 Authentication & Authorization
- The web server implements user authentication (e.g., via a local user database, LDAP, or OAuth).
- Fine-grained RBAC is applied to control which users can view, edit, or trigger specific DAGs.

### 9.2 Data Security
- Sensitive connection passwords and variables are encrypted at rest in the metadata database.
- TLS should be enabled for the web UI and for communication with the database and message broker.

---

## 10. Scalability & Performance

### 10.1 Scalability Strategy
- **Workers**: The primary axis of scaling. The number of worker nodes can be increased to handle more concurrent tasks.
- **Scheduler**: Can be run in an active-active HA configuration on multiple nodes to handle scheduling for a very large number of DAGs.

### 10.2 Performance Monitoring
- Monitor the queue length of the message broker to ensure tasks are not backing up.
- Track scheduler "heartbeat" and DAG parsing times to detect performance issues.
- Monitor task instance duration to identify slow or stuck tasks.

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture
- A typical production deployment involves dedicated nodes for the web server, scheduler, workers, database, and message broker.
- Containerization (e.g., using Docker and Kubernetes) is highly recommended to simplify deployment and management. The Kubernetes Executor is a first-class alternative to the Celery Executor for cloud-native deployments.

### 11.2 CI/CD Pipeline
- DAG code should be managed in a Git repository.
- CI pipelines should run static analysis and tests on DAGs.
- CD pipelines should automate the deployment of new/updated DAG code to the production environment.

### 11.3 Logging Strategy
- Task logs are written to a temporary location on the worker and then streamed to a centralized, persistent location (e.g., S3, Google Cloud Storage) and associated with the task instance in the metadata database.

---

## 12. Testing Strategy

### 12.1 Test Types
- **DAG Integrity Tests**: Static tests to ensure DAGs are acyclic, have no syntax errors, and load correctly.
- **Unit Tests**: For custom operators and hooks.
- **Integration Tests**: Test that a DAG can successfully interact with external systems in a staging environment.

---

## 13. Risk Analysis

### 13.1 Technical Risks
| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Scheduler is a Single Point of Failure | High | Medium | Run scheduler in an active-active HA configuration. |
| Metadata Database Bottleneck | High | Medium | Use a robust, well-tuned database. Monitor performance and scale the database instance as needed. |
| Misconfigured DAGs | Medium | High | Implement CI/CD with static analysis and testing for all DAG code. |

---

## 14. Implementation Plan

### 14.1 Project Phases
- **Phase 1**: Core components: metadata database schema, basic scheduler, and a local executor for single-node execution.
- **Phase 2**: Develop the web server UI for basic monitoring.
- **Phase 3**: Implement a distributed executor (Celery/Kubernetes) and scale-out workers.
- **Phase 4**: Add advanced features: RBAC security, HA scheduler, and a comprehensive API.
