# High-Level Design: SMS Gateway Service

## 1. Document Information

- **Document Title**: SMS Gateway Service - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a centralized SMS Gateway Service. The purpose of this service is to provide a single, unified, and reliable API for sending and receiving SMS messages, abstracting away the complexities of integrating with multiple third-party SMS providers (e.g., Twilio, Vonage, Sinch). The system is designed to improve reliability by enabling automatic failover between providers and to optimize costs by routing messages through the most cost-effective provider for a given destination.

### 2.2 Scope

**In Scope:**
-   A single API endpoint for sending SMS messages.
-   Integration with multiple downstream SMS provider APIs.
-   A routing engine to select the best provider for a given message.
-   A webhook endpoint to receive incoming messages and delivery status updates from providers.
-   A message queue for resilient, asynchronous message processing.
-   Centralized logging and analytics for all sent and received messages.

**Out of Scope:**
-   The third-party SMS provider services themselves.
-   The internal client applications that use this gateway service.
-   A user interface for managing the service or viewing logs.
-   Billing and account management for the clients of this service.

### 2.3 High-Level Architecture Overview
The system is an API gateway with intelligent routing and a resilient, queue-based architecture. An internal **Client Application** makes a single API call to the **SMS Gateway API** to send a message. The API service performs basic validation and enqueues the send request into a **Message Queue**. A fleet of **Sending Workers** consumes from this queue. For each message, a **Routing Engine** component consults a **Provider Ruleset** to select the best provider (e.g., based on destination country, cost, or current provider health). The worker then formats the request for the selected provider's specific API and makes the call. Delivery status updates are sent from the provider back to a **Callback Service**, which processes them and updates the message status in an **Analytics Database**.

---

## 3. System Overview

### 3.1 Business Context
Many applications rely on SMS for critical user interactions like two-factor authentication (2FA), appointment reminders, and delivery notifications. Relying on a single third-party SMS provider creates a single point of failure and vendor lock-in. An in-house SMS Gateway provides a layer of abstraction that increases reliability through multi-provider failover, optimizes costs through intelligent routing, and simplifies integration for all other internal services by providing one consistent API.

### 3.2 System Purpose
To provide a highly available, cost-effective, and unified platform for all programmatic SMS communication, abstracting the details of multiple external providers.

### 3.3 Success Criteria
-   **High Reliability**: The service must achieve a higher message delivery success rate than any single provider by intelligently routing around failures.
-   **High Availability**: The gateway's API endpoint must be highly available (99.99%).
-   **Low Latency**: The time from API call to the message being accepted by a third-party provider should be low (e.g., <500ms at p95).
-   **Cost-Effectiveness**: The routing engine should demonstrably reduce overall SMS costs.
-   **Observability**: The system must provide centralized logs and metrics for every message sent.

### 3.4 Assumptions
-   The organization has accounts with multiple third-party SMS providers.
-   The volume of SMS messages is high enough to justify the complexity of building and maintaining this gateway.
-   A mechanism exists to compare the costs of different providers for different regions.

### 3.5 Constraints
-   **Provider API Differences**: Each SMS provider has a unique API, different capabilities, and different error codes, which must be normalized by the gateway.
-   **Complex Routing Logic**: The logic for choosing the optimal provider can become complex, involving factors like cost, deliverability rates, and real-time provider health.
-   **Asynchronous Status Updates**: SMS delivery is asynchronous. The system must be able to correlate incoming delivery status webhooks with the original outbound message.
-   **Regulatory Compliance**: SMS regulations vary significantly by country (e.g., rules around sender IDs, opt-outs). The gateway must be able to handle this complexity.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Unified Send API)**: The system must expose a single, simple API endpoint for sending an SMS message, taking a destination number and message body as input.
-   **FR-002 (Multi-Provider Integration)**: The system must integrate with at least two different third-party SMS provider APIs.
-   **FR-003 (Intelligent Routing)**: The system must have a configurable routing engine to select a provider for each message based on rules (e.g., destination country, cost).
-   **FR-004 (Asynchronous Sending)**: The API call should return quickly to the client. The actual sending of the message should happen asynchronously.
-   **FR-005 (Status Callback Handling)**: The system must provide a webhook endpoint to receive delivery status updates (`sent`, `delivered`, `failed`) from the providers.
-   **FR-006 (Message Status Tracking)**: The system must store and provide the status of every message sent through it.
-   **FR-007 (Automatic Failover)**: If the selected provider for a message fails to send it, the system should automatically retry with a different provider.
-   **FR-008 (Centralized Logging)**: The system must log every message request and status update in a standardized format.

### 4.2 Non-Functional Requirements
-   **High Availability**: The API endpoints and the sending workers must be highly available.
-   **Scalability**: The system must be able to scale to handle a high throughput of messages.
-   **Reliability**: The system should guarantee at-least-once delivery semantics for messages.
-   **Security**: API keys and provider credentials must be stored securely.
-   **Extensibility**: It should be easy to add a new provider integration to the system.
-   **Low Latency**: The API response time should be fast.

---

### 5. Core Architecture

The architecture is a classic decoupled, queue-based system designed for resilience and scalability.

- **API Layer:**
  - **SMS API Service:** A stateless REST API that exposes the `/send` endpoint. It validates the request and the client's API key. Upon successful validation, it generates a unique `message_id`, creates a job payload, and enqueues it into the `Send Queue`. It then immediately returns the `message_id` to the client.

- **Processing Layer:**
  - **Send Queue (e.g., SQS, RabbitMQ):** A durable, highly available message queue that buffers all outgoing SMS requests. This is key to the system's resilience.
  - **Sending Workers:** A fleet of stateless workers that consume jobs from the Send Queue. Each worker performs the core logic:
    1.  **Routing:** It passes the message details (e.g., destination country) to the Routing Engine to get a prioritized list of providers.
    2.  **Provider Integration:** It attempts to send the message using the first provider from the list. It uses a specific "provider adapter" that knows how to format the API request for that provider (e.g., Twilio adapter, Vonage adapter).
    3.  **Failover Logic:** If the API call to the primary provider fails or times out, the worker catches the exception and automatically retries the send using the second provider in the list.
    4.  **Status Update:** Once the message is successfully accepted by a provider, the worker updates the message's status in the Analytics Database to `sent`.
  - **Routing Engine:** A component (likely a library) that contains the logic for selecting the best provider. It can be configured with rules like `IF country == 'US' THEN providers = [Twilio, Vonage]` or `IF cost(Twilio) < cost(Vonage) THEN ...`. It can also incorporate real-time provider health data.

- **Feedback & Status Layer:**
  - **Callback Service:** A public-facing API endpoint that receives HTTP webhooks from the third-party providers about delivery status changes (e.g., `delivered`, `failed`).
  - **Status Update Queue:** The Callback Service places the incoming status updates into another queue for asynchronous processing.
  - **Status Worker:** A worker that consumes from the status queue, parses the provider-specific payload, normalizes the status, and updates the final status of the message in the Analytics Database using its `message_id`.

- **Data Stores & Supporting Services:**
  - **Analytics Database (e.g., Cassandra, DynamoDB):** A highly scalable NoSQL database optimized for writes. It stores a record for every single message sent, including its status, the provider used, cost, etc.
  - **Provider Rules DB/Config:** A database or configuration file that stores the routing rules and provider credentials.
  - **Secrets Manager:** Securely stores the API keys for all third-party providers.

![Architecture Diagram](https://i.imgur.com/kS9lY4W.png)

### 6. Data Schema and Models

- **Send API Request Body:**
  ```json
  {
    "to": "+15551234567",
    "from": "+15557654321",
    "body": "Your verification code is 123456."
  }
  ```
- **Message Job Schema (in Send Queue):**
  ```json
  {
    "message_id": "uuid",
    "to": "+15551234567",
    "from": "+15557654321",
    "body": "...",
    "client_id": "...",
    "retry_count": 0
  }
  ```
- **Analytics DB Schema:**
  - `messages` table: `message_id` (Primary Key), `to`, `from`, `body`, `status` ('pending', 'sent', 'delivered', 'failed'), `provider_used`, `cost`, `created_at`, `sent_at`, `delivered_at`.

### 7. API Design

- **`POST /v1/messages` (Send API):**
  - **Usage:** Send a new SMS message.
  - **Auth:** Requires a client API key.
  - **Body:** `{ "to": "...", "from": "...", "body": "..." }`
  - **Response:** `202 Accepted` with `{ "message_id": "..." }`.
- **`GET /v1/messages/{id}` (Status API):**
  - **Usage:** Check the status of a previously sent message.
  - **Response:** `200 OK` with the full message record from the Analytics DB.
- **`POST /v1/callbacks/{provider}` (Callback Service):**
  - **Usage:** The public endpoint configured in the provider's dashboard to receive status webhooks.

### 8. Scalability and Performance

- **High Throughput:** The system can be scaled by increasing the number of Sending Worker instances. The queue-based architecture allows for massive parallelism.
- **Stateless Services:** The API and worker services are stateless, making them easy to scale horizontally with a load balancer or in an auto-scaling group.
- **Database Choice:** The Analytics DB is write-heavy and uses a NoSQL database designed for horizontal scaling to avoid becoming a bottleneck.

### 9. Reliability and Fault Tolerance

- **Provider Failover:** This is a core feature. The retry logic in the Sending Workers provides resilience against the failure of a single external provider.
- **Durable Queues:** The use of queues for both sending and status updates ensures that requests are not lost if a worker process crashes. The job can be re-processed by another worker.
- **API and Callback HA:** The public-facing API services are deployed across multiple availability zones for high availability.

### 10. Security Considerations

- **Credential Management:** Provider API keys and secrets must be stored in a secure secrets management system (e.g., Vault) and never in code or configuration files.
- **Webhook Security:** The public callback endpoint must be secured. A common method is for the provider to sign its webhook requests (e.g., with an HMAC signature), which the Callback Service must validate to ensure the request is legitimate.
- **Rate Limiting:** The Send API should be rate-limited to prevent abuse by a single client.

### 11. Monitoring and Observability

- **Provider Performance:** Monitor the latency, success rate, and cost for each integrated provider. This data is crucial for the Routing Engine.
- **Queue Depths:** The length of the Send Queue and Status Update Queue are key indicators of system health.
- **End-to-End Latency:** Track the time from the initial API call to the message being successfully delivered.

### 12. Deployment and Operations

- **Provider Integration:** Adding a new provider involves creating a new "adapter" for the Sending Worker and a new parsing module for the Status Worker, then updating the routing configuration.
- **Cost Monitoring:** Requires building dashboards to track spending across different providers and regions.

### 13. Cost Estimation

- **Primary Drivers:**
  - **Third-Party Provider Costs:** The per-message fees charged by Twilio, Vonage, etc.
  - **Compute:** The cost of the worker fleet.
  - **Database & Queues:** The cost of the managed services for data storage.

### 14. Risks and Mitigations

- **Cascading Failures:**
  - **Risk:** The primary provider has a major outage, causing all initial send attempts to fail and time out. The massive volume of retries then overwhelms the secondary provider.
  - **Mitigation:** Implement a circuit breaker in the Sending Workers. If a provider's error rate exceeds a threshold, the circuit breaker trips, and workers immediately fail over to the next provider without waiting for a timeout.
- **Incorrect Status Correlation:**
  - **Risk:** The system cannot match an incoming delivery status webhook to the original outbound message.
  - **Mitigation:** When sending a message, most provider APIs allow you to pass a custom `MessageID` or `CallbackData` field. The gateway should set this to its internal `message_id` to ensure a reliable correlation.
- **Regulatory Issues:**
  - **Risk:** Sending messages that violate local regulations leads to fines or blocking.
  - **Mitigation:** The Routing Engine needs to be aware of country-specific rules (e.g., allowed sender IDs, required opt-out language) and enforce them.

### 15. Final Considerations

An SMS Gateway Service is a powerful abstraction layer that turns a set of unreliable, heterogeneous external APIs into a single, reliable, and cost-optimized internal platform. The core principles of its design are resilience through redundancy (multiple providers) and scalability through a decoupled, queue-based architecture. While the initial implementation is straightforward, the complexity grows in the sophistication of the routing engine and the need to handle the unique quirks and regulatory requirements of each provider and destination country.
