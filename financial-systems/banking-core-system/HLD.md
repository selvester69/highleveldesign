# High-Level Design: Core Banking System

## 1. Document Information

- **Document Title**: Core Banking System (CBS) - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-28
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document provides the high-level design for a modern Core Banking System (CBS). The CBS is the central nervous system of a bank, acting as the authoritative system of record for all customer accounts, financial transactions, and the bank's general ledger. The design prioritizes data integrity, security, and reliability above all else.

### 2.2 Scope

**In Scope:**
-   A central **Customer Information File (CIF)** for managing customer data.
-   **Account Management** for various product types (e.g., savings accounts, checking/current accounts, loan accounts).
-   A **General Ledger (GL)** system based on double-entry bookkeeping principles.
-   A **Transaction Processor** for handling debits and credits to customer accounts (e.g., deposits, withdrawals, transfers).
-   A batch processing system for **end-of-day operations**, such as interest calculation and fee application.
-   A secure, internal API layer to allow other banking channels (Online Banking, Mobile App, ATMs, Branch Tellers) to interact with the core.

**Out of Scope:**
-   The user-facing applications themselves (e.g., the Online Banking website, the Mobile App).
-   Connections to external payment networks (e.g., SWIFT, ACH) - these are handled by separate payment hub systems that integrate with the CBS.
-   Loan origination, credit scoring, and treasury management systems.
-   Customer Relationship Management (CRM) and marketing platforms.

### 2.3 High-Level Architecture Overview
The architecture is designed as a highly available, fault-tolerant, and secure set of microservices built around a powerful relational database. The core of the system is the **General Ledger**, which serves as the immutable source of truth. All operations are performed via a secure **API Layer** and are processed by dedicated services for **Account Management** and **Transaction Processing**. Data integrity is ensured through strict transactional control and a double-entry accounting model for every financial movement.

---

## 3. System Overview

### 3.1 Business Context
A Core Banking System is the software that a bank uses to manage its most fundamental financial operations. It is one of the most critical systems in any financial institution, as its accuracy and availability directly impact the bank's ability to function and maintain regulatory compliance.

### 3.2 System Purpose
The primary purpose of the CBS is to provide a single, authoritative source of truth for the bank's financial data. It must securely and reliably manage customer accounts, process transactions with perfect accuracy, and ensure the overall financial integrity of the bank.

### 3.3 Success Criteria
-   **Data Integrity & Accuracy**: Zero data loss or corruption. The General Ledger must always be perfectly balanced. Every transaction must be processed exactly once.
-   **Availability**: 99.99% availability for the API layer during business hours, with well-defined and reliable windows for end-of-day batch processing.
-   **Auditability & Compliance**: The system must provide a complete, tamper-evident audit trail for every transaction and change, sufficient to pass all regulatory and financial audits.
-   **Reliability**: The system must be fault-tolerant and able to recover from failures without losing data or compromising integrity.

### 3.4 Assumptions
-   The bank operates in a strict regulatory environment.
-   The CBS will be the central hub, and other systems will integrate with it via its APIs.
-   The system will have a dedicated operations team to manage end-of-day processes and reconciliation.

### 3.5 Constraints
-   **ACID Compliance is Non-Negotiable**: All transactions that modify the financial state of the system must be fully ACID-compliant (Atomic, Consistent, Isolated, Durable).
-   **Double-Entry Bookkeeping**: Every financial transaction must be recorded as corresponding debit and credit entries in the General Ledger.
-   **Security**: Access to the CBS must be strictly controlled, and sensitive customer data must be encrypted at rest and in transit.
-   **Long-Term Data Archival**: The system must be able to archive decades of financial data in a secure and accessible manner for regulatory purposes.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Customer Management)**: The system shall create and maintain a unique profile for each customer (CIF), containing their personal details and a list of their accounts.
-   **FR-002 (Account Management)**: The system shall support the lifecycle of various account types, including opening, updating status (e.g., active, dormant, closed), and closing accounts.
-   **FR-003 (Transaction Processing)**: The system shall process financial transactions, including deposits, withdrawals, and internal transfers between accounts. Each transaction must validate funds availability and update account balances atomically.
-   **FR-004 (General Ledger)**: The system must maintain a real-time General Ledger. Every transaction must generate a balanced set of entries (debits and credits) in the GL.
-   **FR-005 (Interest Calculation)**: The system shall have a batch process to accurately calculate interest payable or receivable on accounts and post the corresponding entries.
-   **FR-006 (Fee Application)**: The system shall support the configuration and automatic application of standard account fees (e.g., monthly service fees).
-   **FR-007 (Internal API Layer)**: The system must expose a secure set of APIs for other banking systems (e.g., mobile banking) to query account balances, view transaction history, and initiate transactions.

### 4.2 Non-Functional Requirements
-   **Data Integrity**: This is the most critical NFR. The system must guarantee that the financial ledger is always consistent and correct. This is achieved through database-level constraints and transactional logic.
-   **Security**:
    -   **Access Control**: Strict role-based access control (RBAC) must be enforced for all bank employees interacting with the system.
    -   **Data Encryption**: All Personally Identifiable Information (PII) and sensitive financial data must be encrypted at rest and in transit.
-   **Availability**: 99.99% uptime for transactional APIs. The system must have a comprehensive disaster recovery plan with a low RTO and an RPO of zero.
-   **Recoverability**: The system must be able to recover to a consistent state from backups in the event of a catastrophic failure.
-   **Auditability**: Every action that creates or modifies financial data must be logged in an immutable audit trail, including who performed the action and when.
-   **Performance**: While not a low-latency system like an exchange, the system must be able to process the bank's daily transaction volume and complete its end-of-day batch processing within the designated overnight window.

---

## 5. Architecture Design

### 5.1 Architecture Principles
- **Ledger as the Source of Truth**: The General Ledger (GL) is the immutable and definitive record of all financial facts. All other financial data is a projection or summary of the GL.
- **Transactional Consistency**: Every operation that alters the financial state of the bank must be executed within a single, atomic transaction to ensure data integrity.
- **Security by Default**: The system is designed with a "deny all" security posture. Access to data and functionality is only granted through explicit, role-based permissions.
- **Auditability**: Every action is logged. It must be possible to trace every transaction from its origin to its final entries in the GL.

### 5.2 Architecture Patterns
- **Layered, Service-Oriented Architecture**: The system is designed as a set of logical services (e.g., Account Service, Customer Service) that can be deployed together as a "modular monolith" or as separate microservices. This provides a clear separation of concerns.
- **Double-Entry Bookkeeping**: This fundamental accounting principle is enforced by the architecture. For every transaction, debits must equal credits, ensuring the ledger always remains balanced.
- **Strangler Fig Pattern (for migration)**: When replacing a legacy CBS, this pattern is often used to incrementally move functionality from the old system to the new system, reducing migration risk.

### 5.3 High-Level Architecture Diagram

```mermaid
graph TD
    subgraph "Bank Channels (Clients)"
        OnlineBanking["Online Banking"]
        MobileApp["Mobile App"]
        ATM_Network["ATM Network"]
        BranchSystem["Branch Teller System"]
    end

    subgraph "Core Banking System"
        API_Layer["Secure API Layer"]

        subgraph "Online Transaction Processing (OLTP) Services"
            CustomerService["Customer Service (CIF)"]
            AccountService["Account Service"]
            TransactionProcessor["Transaction Processor"]
            LedgerService["Ledger Service"]
        end

        subgraph "Batch Processing Services"
            InterestService["Interest & Fee Service"]
            ReportingService["Reporting Service"]
        end

        DB["Core Relational Database <br/> (PostgreSQL / Oracle)"]
    end

    Bank Channels -- "Secure API Calls" --> API_Layer
    API_Layer --> CustomerService
    API_Layer --> AccountService
    API_Layer --> TransactionProcessor

    TransactionProcessor --> LedgerService
    CustomerService --> DB
    AccountService --> DB
    LedgerService --> DB

    BatchProcessing["Batch Scheduler"] -- "Triggers" --> InterestService
    BatchProcessing -- "Triggers" --> ReportingService
    InterestService --> DB
    ReportingService --> DB
```

### 5.4 Component Overview
- **API Layer**: A secure gateway that exposes the core banking functionalities to other internal bank systems. It handles authentication, authorization, and request routing.
- **Customer Service (CIF)**: Manages the master record of all customer information.
- **Account Service**: Manages the lifecycle and state of all customer accounts (savings, checking, loans).
- **Transaction Processor**: Executes real-time financial transactions. It validates requests, orchestrates the movement of funds between accounts, and ensures the corresponding entries are posted to the ledger.
- **Ledger Service**: Owns and protects the integrity of the General Ledger. It is the only service that can write to the core ledger tables.
- **Batch Processing System**: A set of services that run on a schedule (typically overnight) to perform non-real-time tasks like calculating interest, applying fees, and generating statements.
- **Core Relational Database**: The authoritative data store for the entire system, enforcing data integrity through schemas, constraints, and transactions.

### 5.5 Technology Stack
- **Programming Language**: **Java (with Spring Boot)** is a very common and robust choice for large-scale enterprise systems, offering a mature ecosystem for security, transactions, and batch processing.
- **Database**: **PostgreSQL** or **Oracle**. A powerful, ACID-compliant relational database is non-negotiable.
- **API**: **REST** or **gRPC** for synchronous internal communication.
- **Batch Processing**: **Spring Batch** or a similar robust framework.

### 5.6 Architecture Decision Records (ADRs)

#### 5.6.1 ADR-001: Relational Database as the Single Source of Truth
- **Decision**: To build the system around a single, powerful, and highly-available relational database cluster.
- **Justification**: Data integrity is the most critical requirement. A traditional RDBMS provides the strongest guarantees for ACID compliance, which is essential for a financial system of record.
- **Consequences**: The database can become a scalability bottleneck. This is mitigated by using a powerful database, efficient queries, and offloading read-heavy workloads (like analytics) to read replicas.

---

## 6. Detailed Component Design

### 6.1 Component 1: API Layer
- **Purpose**: To provide a stable, secure, and versioned interface to the core system.
- **Responsibilities**:
    -   Authenticate and authorize the calling system (e.g., the Mobile Banking backend).
    -   Validate incoming requests against a defined schema.
    -   Route requests to the appropriate backend service.
    -   Transform data between the public API format and the internal domain models.

### 6.2 Component 2: Transaction Processor & Ledger Service
- **Purpose**: To execute transactions atomically and ensure the ledger is always balanced.
- **Responsibilities**:
    1.  Wrap the entire operation in a single database transaction.
    2.  Validate the transaction request (e.g., check account status, verify sufficient funds).
    3.  Instruct the Ledger Service to create the corresponding double-entry bookkeeping records (e.g., debit customer A's account, credit customer B's account).
    4.  Update the balances on the customer-facing account records.
    5.  Commit the transaction. If any step fails, the entire operation is rolled back, leaving the system in a consistent state.

### 6.3 Component 3: Account Service
- **Purpose**: To manage the state and business rules associated with customer accounts.
- **Responsibilities**:
    -   Handle CRUD (Create, Read, Update, Delete) operations for accounts.
    -   Enforce business rules, such as preventing the balance from dropping below a minimum on certain account types.
    -   Provide other services with information about account status and details.

### 6.4 Component 4: End-of-Day (EOD) Batch Service
- **Purpose**: To perform large-scale, periodic processing across many or all accounts.
- **Responsibilities**:
    -   Run on a schedule (e.g., daily at midnight).
    -   Read account data, calculate interest or fees based on configured rules, and post the resulting transactions back into the system via the Transaction Processor.
    -   Generate data for customer statements and regulatory reports.

---

## 7. Data Design

### 7.1 Data Models
The data model is highly normalized and relational to ensure integrity and consistency. It is built around the principles of double-entry bookkeeping.

- **`customers` table (CIF)**:
  - `customer_id` (PK), `first_name`, `last_name`, `tax_identifier`, `address_details`, `status`, `created_at`, `updated_at`.
- **`accounts` table**:
  - `account_id` (PK), `customer_id` (FK), `product_id` (FK), `account_number` (UNIQUE), `current_balance`, `available_balance`, `status`.
- **`gl_transactions` table**: An immutable, append-only log of all financial events.
  - `transaction_id` (PK), `transaction_type` (e.g., 'DEPOSIT', 'TRANSFER'), `description`, `timestamp`.
- **`gl_entries` table**: The double-entry ledger itself. Every transaction has at least two entries.
  - `entry_id` (PK), `transaction_id` (FK), `gl_account_id` (FK), `debit_amount`, `credit_amount`. The sum of debits must equal the sum of credits for any given `transaction_id`.

### 7.2 Data Storage Strategy
- **Primary Database**: A highly available, active-passive relational database cluster (e.g., PostgreSQL with streaming replication, Oracle RAC). The database is the source of truth and enforces data integrity via constraints and transactions.
- **Backup and Archival**: A rigorous backup strategy is essential. This includes point-in-time recovery capabilities, daily full backups, and long-term archival of data to meet regulatory requirements (which can be 7+ years).
- **Read Replicas**: Read-only replicas of the database are used to serve non-transactional workloads, such as internal reporting and analytics, to avoid impacting the performance of the primary OLTP database.

---

## 8. API Design

### 8.1 API Architecture
The CBS exposes a set of secure, internal-only, service-oriented APIs. These APIs are not available to the public internet and are only accessible from within the bank's trusted network.

### 8.2 API Specifications
- **Protocol**: A combination of synchronous and asynchronous patterns.
    - **Synchronous**: **gRPC** or **REST** for requests that require an immediate response (e.g., `getAccountBalance`).
    - **Asynchronous**: A **Message Queue** (e.g., RabbitMQ) for commands that can be processed asynchronously (e.g., `processDeposit`).
- **Idempotency**: All command-based APIs (those that change state) must support an idempotency key. This ensures that if a client sends the same request twice due to a network error, the transaction is only processed once.
- **Core Endpoints (Example REST-like definitions)**:
    - `POST /v1/transactions`: The primary endpoint to initiate a financial transfer.
    - `GET /v1/customers/{id}`: Retrieve customer details.
    - `GET /v1/accounts/{id}`: Retrieve account details, including balance.
    - `GET /v1/accounts/{id}/history`: Retrieve a list of recent transactions for an account.

---

## 9. Security Design

### 9.1 Security Architecture
The security model is designed to protect sensitive customer data and prevent unauthorized financial movements, adhering to the principle of least privilege.

### 9.2 Authentication & Authorization
- **System-to-System Authentication**: Calling systems (e.g., the Online Banking backend) must authenticate to the CBS API layer using mutual TLS (mTLS), ensuring that only trusted internal systems can make requests.
- **End-User Authorization (Role-Based Access Control - RBAC)**: The identity of the end-user (whether a customer or a bank employee) is securely passed along with the API request. The CBS enforces a strict RBAC model. For example:
    - A customer can only view their own accounts.
    - A bank teller can initiate a deposit but cannot approve a large wire transfer.
    - A branch manager has higher-level permissions than a teller.
- **Multi-Factor Authentication**: All access for bank employees with privileged access to the CBS requires MFA.

### 9.3 Data Security
- **Encryption**:
    - **At Rest**: All sensitive data in the database, particularly Customer PII, is encrypted using Transparent Data Encryption (TDE).
    - **In Transit**: All network communication uses TLS 1.2+.
- **Data Masking**: In non-production environments (like testing and development), sensitive data is masked or anonymized to prevent data leakage.

### 9.4 Auditability
- **Immutable Audit Log**: A critical security control. Every action that reads or modifies data is logged to a tamper-evident, append-only audit log.
- **Log Contents**: Each audit log entry includes the timestamp, the service that performed the action, the specific data that was accessed or changed, and the identity of the end-user and/or system that initiated the request.

---

## 10. Scalability & Performance

### 10.1 Performance Requirements
- **Batch Processing Window**: The end-of-day batch jobs (interest calculation, fee application, reporting) must complete within a strict overnight window (e.g., 6 hours) to ensure the system is available for the next business day.
- **Transactional Throughput**: The system must support the peak transaction volume of the bank during business hours.

### 10.2 Scalability Strategy
- **Vertical Scaling**: The core database is the main candidate for vertical scaling (i.e., running on very powerful servers with large amounts of RAM and high-speed storage). This is because sharding a transactional banking ledger is extremely complex.
- **Horizontal Scaling**: The stateless service layer (API Layer, Transaction Processor) can be scaled horizontally to handle more requests from the bank's various channels. Read-heavy workloads are offloaded to database read replicas.

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture
The system is deployed in a high-availability configuration across two or more data centers. An active-passive setup is common, with a primary data center handling all live traffic and a secondary DR site ready for a rapid failover.

### 11.2 Environment Strategy
- **Testing**: A dedicated, full-scale testing environment that is a mirror of production.
- **UAT (User Acceptance Testing)**: An environment for business users (e.g., bank operations staff) to test new features and sign off on them before deployment.
- **Production**: The live system of record.

### 11.3 CI/CD Pipeline
Deployments to the Core Banking System are infrequent, high-risk events. They are planned months in advance and are not part of a typical rapid CI/CD flow. The process involves extensive testing, UAT, a detailed deployment plan, and often requires a scheduled downtime window.

### 11.4 Operations & Monitoring
- **IT Operations Team**: A dedicated team is responsible for monitoring the CBS 24/7, managing backups, and executing the EOD batch jobs.
- **Reconciliation**: A daily operational process where the finance team reconciles the CBS ledger against other systems and statements from partner banks to ensure perfect financial alignment.

---

## 12. Testing Strategy

### 12.1 Parallel Run Testing
When migrating from a legacy CBS, the most critical testing phase is the "parallel run." The new CBS is run alongside the old system for an extended period (weeks or months), processing the same inputs. The outputs (e.g., account balances, reports) are compared daily to ensure the new system is 100% accurate before the final cutover.

### 12.2 Disaster Recovery Testing
The bank must conduct regular, full-scale tests of its disaster recovery plan, where the entire system is failed over to the DR site. This is a regulatory requirement in many jurisdictions.

### 12.3 End-to-End Transactional Testing
Automated tests that simulate complex, multi-leg financial transactions and verify that all account balances and ledger entries are correct at the end of the process.

---

## 13. Risk Analysis

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data Corruption / Inconsistency | Catastrophic | Very Low | Strict ACID guarantees from the RDBMS, double-entry bookkeeping, extensive testing, and daily reconciliation. |
| EOD Batch Job Failure | Critical | Low | Make all batch jobs idempotent and restartable. Extensive monitoring and alerting. A dedicated operations team on standby during the batch window. |
| Migration from Legacy System | Critical | High | A phased migration using the Strangler Fig pattern. A long parallel run testing period. Extensive planning and data validation. |
| Insider Threat | High | Medium | Strict Role-Based Access Control (RBAC), separation of duties, and a complete, tamper-evident audit trail for every action. |

---

## 14. Implementation Plan (High-Level)

A CBS replacement is a multi-year project.
- **Phase 1 (Foundation)**: Build the core data model, the General Ledger service, and the Customer Information service.
- **Phase 2 (Core Products)**: Implement the Account Management service for one or two core products (e.g., Savings, Checking).
- **Phase 3 (Transactions)**: Build the Transaction Processor and the internal API layer.
- **Phase 4 (Parallel Run)**: Begin a parallel run, feeding live data to the new CBS and comparing its output with the legacy system. This phase is the longest and most critical.
- **Phase 5 (Migration)**: Once the parallel run is consistently successful, plan the "go-live" event. This involves a carefully managed data migration and cutover from the old system to the new one.

---

## 15. Appendices

### Appendix A: Glossary
- **CBS**: Core Banking System.
- **CIF**: Customer Information File. The master record of a bank's customers.
- **GL**: General Ledger. The central book of accounts for the bank.
- **ACID**: Atomicity, Consistency, Isolation, Durability. Properties of database transactions.
- **Double-Entry Bookkeeping**: An accounting principle where every transaction results in at least one debit and one credit entry, and the total debits must equal the total credits.
