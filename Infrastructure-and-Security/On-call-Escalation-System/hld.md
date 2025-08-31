# High-Level Design: On-call Escalation System

## 1. Document Information

- **Document Title**: On-call Escalation System - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for an On-call Escalation System, architecturally similar to PagerDuty or Opsgenie. The system's primary purpose is to receive alerts from various monitoring systems, determine who is currently on-call for a given service, notify them reliably through multiple channels, and automatically escalate the alert if it is not acknowledged within a specified timeframe.

### 2.2 Scope

**In Scope:**
-   An API endpoint to receive alerts from monitoring systems (e.g., Prometheus, Datadog).
-   A scheduling component for creating and managing on-call rotations for different teams.
-   A notification engine that can send alerts via multiple channels (SMS, push notification, phone call, email).
-   An escalation policy engine to define rules for escalating unacknowledged alerts.
-   A mechanism for on-call engineers to acknowledge or resolve alerts.
-   A central data store for schedules, escalation policies, and incident history.

**Out of Scope:**
-   The monitoring systems that generate the initial alerts.
-   The user interface for creating schedules and policies.
-   The mobile application for receiving push notifications.
-   Detailed incident management features like post-mortems or runbook integration.

### 2.3 High-Level Architecture Overview
The system is an event-driven workflow. An **Ingestion Service** receives an alert and publishes it to a **Message Queue**. An **Alert Processor** service consumes the alert. It queries a **Scheduling Service** to determine the current on-call user for the alert's service. It then passes the alert and user information to a **Notification Service**, which attempts to notify the user via their preferred channels. The Alert Processor also creates a delayed "escalation" job in a **Job Queue**. If the user **acknowledges** the alert via an API call, the escalation job is cancelled. If the alert is not acknowledged within the timeout, the escalation job is executed, which triggers the Alert Processor to find the next person in the escalation policy and repeat the process.

---

## 3. System Overview

### 3.1 Business Context
Modern software systems are complex and require 24/7 monitoring to ensure reliability. When an issue occurs, it's critical to notify the correct on-call engineer immediately to minimize downtime and business impact. An automated escalation system ensures that alerts are never missed, even if the primary on-call person is unavailable, by providing reliable, multi-channel notifications and a clear escalation path.

### 3.2 System Purpose
To ensure that critical alerts from monitoring systems are reliably and quickly delivered to the correct on-call engineer, and to automate the escalation process to guarantee a human response.

### 3.3 Success Criteria
-   **Notification Reliability**: Notifications must have an extremely high delivery rate (>99.99%) across all channels.
-   **Notification Latency**: The time from alert ingestion to the first notification being sent should be less than 1 minute.
-   **Scalability**: The system must handle thousands of incoming alerts per minute from many different monitoring sources.
-   **Availability**: The alert ingestion and notification pipeline must be highly available.
-   **Correctness**: The system must always route an alert to the correct person based on the current on-call schedule.

### 3.4 Assumptions
-   Third-party services (e.g., Twilio for SMS/calls, APNS/FCM for push notifications) will be used for sending notifications.
-   On-call schedules and escalation policies are defined and managed by users.
-   Monitoring systems are configured to send alerts to the system's ingestion API.

### 3.5 Constraints
-   **Dependency on Third-Party Providers**: The system's reliability is heavily dependent on the reliability of the external notification providers.
-   **Time-Critical Operations**: Scheduling and escalation logic is highly sensitive to time. The system must have a consistent and reliable way of handling time zones and scheduled jobs.
-   **Alert Storms**: A major outage can cause a "storm" of thousands of alerts, which can overwhelm the system and the on-call engineers. The system needs a way to handle this.
-   **State Management**: The system is highly stateful, tracking the status (triggered, acknowledged, resolved) of every active incident.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Alert Ingestion)**: The system must provide a secure API endpoint for monitoring services to send alerts.
-   **FR-002 (On-call Scheduling)**: Users must be able to create on-call schedules with rotations (e.g., daily, weekly) for a team.
-   **FR-003 (Escalation Policies)**: Users must be able to define multi-level escalation policies (e.g., "1. Notify the on-call person. 2. If no acknowledgement in 15 minutes, notify their manager.").
-   **FR-004 (Service-Policy Mapping)**: Users must be able to map an incoming alert (based on its source or service name) to a specific escalation policy.
-   **FR-005 (Multi-Channel Notification)**: The system must be able to notify users via Email, SMS, Push Notification, and automated Phone Call.
-   **FR-006 (Alert Acknowledgement)**: An on-call user must be able to acknowledge an alert (e.g., via the mobile app or by replying to an SMS), which stops further notifications and escalations for that alert.
-   **FR-007 (Alert Resolution)**: An on-call user must be able to resolve an alert once the underlying issue is fixed.
-   **FR-008 (Alert De-duplication)**: The system should be able to group multiple related alerts into a single incident to avoid flooding the user with notifications.

### 4.2 Non-Functional Requirements
-   **High Availability**: The alert processing pipeline must be highly available.
-   **Reliability**: Notifications must be reliable. The system should handle failures from notification providers gracefully.
-   **Low Latency**: Alerts must be processed and sent out quickly.
-   **Scalability**: The system must handle a high volume of concurrent alerts.
-   **Durability**: Incident history and configurations must be stored durably.
-   **Security**: PII (phone numbers, emails) must be stored securely. Access to the system should be authenticated.

---

### 5. Core Architecture

The architecture is an event-driven system built around a central incident store and a delayed job queue for managing time-based escalations.

- **Ingestion Layer:**
  - **Alert API:** A highly available HTTP endpoint that receives alerts from external monitoring systems. It performs basic validation and pushes the raw alert into a message queue.
  - **Raw Alerts Queue (e.g., SQS/RabbitMQ):** A message queue that decouples the ingestion API from the processing logic, providing a durable buffer.

- **Processing Layer:**
  - **Alert Processor:** A worker service that consumes alerts from the raw queue.
    1.  **De-duplicates/Correlates:** Checks if the alert belongs to an existing active incident.
    2.  **Looks up Policy:** Determines which service and escalation policy to use.
    3.  **Finds On-call User:** Queries the Scheduler service for the current on-call person at level 1.
    4.  **Creates Incident:** Creates a new incident record in the Incident Store with a `triggered` status.
    5.  **Triggers Notifications:** Sends a job to the Notification service.
    6.  **Schedules Escalation:** Pushes a delayed job to the Escalation Job Queue (e.g., "escalate incident #123 in 15 minutes").
  - **Scheduler Service:** A service that manages on-call schedules and rotations. It exposes an API `GET /on-call?service=X` which returns the user ID for the specified service.
  - **Escalation Worker:** A service that consumes from the delayed job queue. When a job is delivered, it checks the incident's status. If it's still `triggered` (not acknowledged), it tells the Alert Processor to escalate to the next level.

- **Data Stores:**
  - **Configuration DB (e.g., PostgreSQL):** Stores user data, schedules, teams, and escalation policies.
  - **Incident Store (e.g., Cassandra/DynamoDB):** A highly available NoSQL database that stores the state of all active and historical incidents. It is optimized for high write throughput.
  - **Escalation Job Queue (e.g., Redis with sorted sets, or a managed service like SQS with delay):** Stores the scheduled escalation tasks.

- **Notification Layer:**
  - **Notification Service:** An API service that takes a request (`user_id`, `message`, `channel`) and sends it to the appropriate third-party provider.
  - **Third-Party Gateways:** Integrations with Twilio (SMS, calls), SendGrid (email), and APNS/FCM (push notifications). This layer handles provider-specific logic and retries.

- **User Interaction Layer:**
  - **Callback API:** An endpoint that receives user actions (e.g., from a link in an SMS, a button in a push notification).
  - **Acknowledge/Resolve Handler:** When a user acknowledges an incident, this handler updates the Incident Store's record to `acknowledged` and cancels the pending escalation job in the job queue.

![Architecture Diagram](https://i.imgur.com/vH9tJ4A.png)

### 6. Data Schema and Models

- **Incidents Table (in Incident Store):**
  - `incident_id` (Primary Key)
  - `service_id`
  - `status` ('triggered', 'acknowledged', 'resolved')
  - `current_escalation_level` (integer)
  - `triggered_at`, `acknowledged_at`, `resolved_at` (timestamps)
  - `payload` (the raw alert details)
- **Schedules Table (in Config DB):**
  - `schedule_id`, `team_id`, `rotation_type` (daily, weekly), `start_time`, `list_of_user_ids`.
- **Escalation Policies Table (in Config DB):**
  - `policy_id`, `service_id`, `levels` (JSON array, e.g., `[{"level": 1, "target": "on-call", "wait_minutes": 15}, {"level": 2, "target": "manager", "wait_minutes": 30}]`).

### 7. API Design

- **`POST /api/v1/ingest` (Alert Ingestion):**
  - **Usage:** Called by monitoring systems.
  - **Body:** `{ "service_key": "...", "event_type": "trigger", "description": "...", "details": {...} }`
  - **Response:** `202 Accepted`.

- **`POST /api/v1/incidents/{id}/acknowledge` (User Interaction):**
  - **Usage:** Called when a user acknowledges an alert.
  - **Body:** `{ "user_id": "..." }`
  - **Response:** `200 OK`.

- **`POST /api/v1/incidents/{id}/resolve` (User Interaction):**
  - **Usage:** Called when a user resolves an alert.
  - **Response:** `200 OK`.

### 8. Scalability and Performance

- **Stateless Services:** All processing services (Alert Processor, Scheduler, Notification Service) are stateless and can be scaled horizontally.
- **Queue-Based Architecture:** The use of message queues provides a scalable buffer for incoming alerts, allowing the system to handle large "alert storms" without dropping requests.
- **Database Choice:** The Incident Store uses a NoSQL database optimized for high write throughput and horizontal scaling to handle the load of creating and updating incidents. The Configuration DB can be a standard relational database as its workload is less intensive.

### 9. Reliability and Fault Tolerance

- **Notification Redundancy:** The core value proposition. If a push notification is not acknowledged, the system automatically tries SMS, then a phone call. The Notification Service should also handle retries if a third-party provider's API fails.
- **Idempotent Ingestion:** The ingestion layer should generate a unique key for each incoming alert to allow for safe retries from clients and to de-duplicate alerts.
- **Durable Queues:** All queues used in the system are durable to prevent loss of tasks during a service restart.
- **Active-Active Deployment:** The entire system can be deployed in an active-active configuration across multiple regions to provide extreme availability.

### 10. Security Considerations

- **Ingestion Security:** The ingestion API endpoint must be protected. Each service should have a unique, revocable API key.
- **User Data:** The system stores sensitive user PII (phone numbers, email). This data must be encrypted at rest and access must be tightly controlled.
- **Callback Security:** The endpoints for acknowledging/resolving alerts must be secured (e.g., using signed, single-use URLs) to prevent unauthorized actions.

### 11. Monitoring and Observability

- **End-to-End Latency:** Track the time from ingestion to first notification (Ttn) and time to acknowledgement (TTA).
- **Notification Delivery Rates:** Monitor the success and failure rates for each notification channel (SMS, push, etc.).
- **Queue Depths:** Monitor the depth of the alert and escalation queues. A growing queue indicates a processing bottleneck.
- **Incident Funnel:** Track the number of incidents triggered, acknowledged, and resolved.

### 12. Deployment and Operations

- **Third-Party Provider Management:** Have contracts with multiple SMS/voice providers and a mechanism to fail over between them if one has an outage.
- **On-call for the On-call System:** This system is a top-tier critical service and must have its own on-call rotation.

### 13. Cost Estimation

- **Primary Drivers:**
  - **Third-Party Notification Costs:** SMS and phone calls are a significant operational cost, billed on a per-use basis.
  - **Compute:** The cost of running the various microservices 24/7.
  - **Databases:** The cost of the managed database services.

### 14. Risks and Mitigations

- **"Alert Storm" / Cascading Failure:**
  - **Risk:** A major upstream outage triggers thousands of alerts, which in turn trigger thousands of notifications, potentially hitting API rate limits of notification providers or overwhelming on-call staff.
  - **Mitigation:** Implement intelligent de-duplication and grouping of alerts at the ingestion layer. Provide a global "maintenance mode" to temporarily pause notifications.
- **Clock Sync Issues:**
  - **Risk:** The distributed nature of the system makes it sensitive to clock skew, which could affect the timing of escalations.
  - **Mitigation:** Use NTP on all servers to ensure clocks are synchronized. Design the escalation logic to be resilient to minor clock drift.
- **Missed Escalation:**
  - **Risk:** A bug in the delayed job queue or the escalation worker causes an escalation to be missed.
  - **Mitigation:** Implement a separate "janitor" service that periodically scans the Incident Store for incidents that have been `triggered` for too long without being acknowledged or escalated, and re-triggers the escalation logic for them.

### 15. Final Considerations

An on-call escalation system is a critical component for operational excellence. Its design prioritizes reliability and fault tolerance above all else. The event-driven, queue-based architecture provides the necessary scalability and resilience to handle unpredictable alert volumes. The core challenges are not just in the software, but also in managing dependencies on external notification providers and in creating a system that is robust enough that the company can trust it to be the last line of defense during a major outage.
