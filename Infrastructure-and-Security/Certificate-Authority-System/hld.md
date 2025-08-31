# High-Level Design: Certificate Authority (CA) System

## 1. Document Information

- **Document Title**: Certificate Authority (CA) System - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a Certificate Authority (CA) system. A CA is a critical component of a Public Key Infrastructure (PKI) responsible for issuing, managing, and revoking digital certificates. These certificates (e.g., X.509) bind a public key to an identity (like a person, a server, or a device), forming the foundation of trust for protocols like TLS/SSL. This design focuses on a hierarchical model with a secure offline root CA and one or more online intermediate CAs.

### 2.2 Scope

**In Scope:**
-   A secure Root CA, kept offline and air-gapped.
-   An online Intermediate CA, which is signed by the Root CA and handles day-to-day certificate issuance.
-   An API for requesting new certificates (via a Certificate Signing Request - CSR).
-   A mechanism for certificate revocation, including the generation of Certificate Revocation Lists (CRLs) and/or an Online Certificate Status Protocol (OCSP) responder.
-   A database to store all issued certificates and their status.
-   The process for certificate renewal.

**Out of Scope:**
-   The clients or services that use the issued certificates.
-   The physical security of the hardware (e.g., Hardware Security Modules - HSMs).
-   The detailed implementation of cryptographic algorithms.
-   The user interface for managing the CA.

### 2.3 High-Level Architecture Overview
The architecture follows a best-practice, two-tier hierarchy to protect the all-important Root CA.
1.  **Offline Root CA:** The ultimate source of trust. Its private key is generated and stored on a **Hardware Security Module (HSM)** in a physically secure, air-gapped environment. It is only brought online to sign certificates for Intermediate CAs or to generate a new CRL.
2.  **Online Intermediate CA:** This is the day-to-day workhorse. Its certificate is signed by the Root CA. It runs as a highly available online service.
    -   It exposes an **Issuance API** that accepts Certificate Signing Requests (CSRs).
    -   After validating a CSR, it signs it with its own private key (also stored in an HSM) to create a new end-entity certificate.
    -   It records the issued certificate in a **Certificate Database**.
    -   It provides certificate status through an **OCSP Responder** and/or by publishing a **Certificate Revocation List (CRL)**.

---

## 3. System Overview

### 3.1 Business Context
In a zero-trust world, verifying identity is paramount. Digital certificates are the standard mechanism for authenticating servers (TLS), clients (mTLS), and code (code signing). A private CA allows an organization to manage its own internal PKI, issuing certificates for internal services, devices, and employees without relying on (and paying for) public CAs. This enhances security and provides greater control over the organization's trust infrastructure.

### 3.2 System Purpose
To provide a secure and robust infrastructure for issuing and managing the lifecycle of digital certificates, thereby establishing a chain of trust for various security protocols.

### 3.3 Success Criteria
-   **Security**: The private keys of the Root and Intermediate CAs must never be compromised. This is the single most important criterion.
-   **Availability**: The online components (Intermediate CA, OCSP/CRL services) must be highly available.
-   **Correctness**: The CA must correctly implement the X.509 standard and associated RFCs.
-   **Auditability**: Every action taken by the CA (issuance, revocation) must be securely logged for auditing purposes.
-   **Scalability**: The system must be able to issue and manage millions of certificates.

### 3.4 Assumptions
-   A Hardware Security Module (HSM) is available for secure key generation and storage.
-   Strict operational security procedures (e.g., for the Root CA signing ceremony) will be followed.
-   The system is for a private, internal PKI, not a public, web-trusted CA.

### 3.5 Constraints
-   **Extreme Security Requirements**: The security of the entire infrastructure depends on the security of the CA's private keys. The design must prioritize security above all else.
-   **Complexity of PKI Standards**: The X.509 and related standards are complex and unforgiving.
-   **Offline Nature of the Root**: The air-gapped nature of the Root CA makes operations like signing a new Intermediate CA certificate a slow, manual, and highly ceremonial process.
-   **Revocation Overhead**: Providing real-time revocation status via OCSP can create a high-traffic service that must be very fast and highly available.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (Root CA Creation)**: The system must provide a secure, offline process for creating the Root CA's key pair and self-signed certificate.
-   **FR-002 (Intermediate CA Creation)**: The system must allow the Root CA to sign a CSR from an Intermediate CA.
-   **FR-003 (Certificate Issuance)**: The Intermediate CA must provide an API to accept CSRs, validate them, and issue signed end-entity certificates.
-   **FR-004 (Certificate Revocation)**: An administrator must be able to revoke a certificate before its natural expiry (e.g., if its key is compromised).
-   **FR-005 (CRL Generation)**: The CAs must periodically publish a signed Certificate Revocation List (CRL), which contains the serial numbers of all revoked certificates.
-   **FR-006 (OCSP Service)**: The system must provide an Online Certificate Status Protocol (OCSP) responder that clients can query to get the real-time revocation status of a single certificate.
-   **FR-007 (Certificate Storage)**: The system must maintain a database of every certificate it has issued, including its status and lifecycle information.

### 4.2 Non-Functional Requirements
-   **Security**: Private keys must be stored in an HSM. The Root CA must be physically and logically isolated.
-   **High Availability**: The OCSP and CRL distribution points must be highly available and low-latency. The Intermediate CA issuance service must also be highly available.
-   **Performance**: The OCSP responder must be able to handle a very high volume of requests with low latency.
-   **Auditability**: All significant events must be logged to a secure, tamper-evident audit log.
-   **Interoperability**: Certificates and CRLs must conform to the X.509 standard to be compatible with standard clients and libraries.
-   **Scalability**: The certificate database and OCSP responders must scale to handle millions of certificates.

---

### 5. Core Architecture

The architecture is strictly layered to enforce security boundaries.

- **Tier 0: Root CA**
  - **Hardware:** A server and a Hardware Security Module (HSM) stored in a secure physical vault.
  - **Network:** Completely air-gapped. It is never connected to any network.
  - **Software:** A minimal OS and CA software (e.g., OpenSSL, CFSSL).
  - **Process:** It is only powered on for specific, highly controlled "signing ceremonies" to:
    1.  Sign a certificate for a new Intermediate CA.
    2.  Sign a new Certificate Revocation List (CRL) for the Root CA itself.
  - The signed artifacts are then physically transferred (e.g., via a trusted USB drive) to the online environment.

- **Tier 1: Online Intermediate CA**
  - **Hardware:** A cluster of servers connected to network-attached HSMs.
  - **Network:** Runs in a highly secured, isolated network zone with strict firewall rules.
  - **Issuance Service:** An API service that receives Certificate Signing Requests (CSRs). It validates the request against policies (e.g., "is this domain allowed to get a certificate?") and, if valid, uses the Intermediate CA's private key (in the HSM) to sign the certificate.
  - **Certificate Database:** A replicated database (e.g., PostgreSQL) that stores all issued certificates, their serial numbers, expiry dates, and revocation status.
  - **Revocation Service:** An internal API for administrators to revoke a certificate. This updates the certificate's status in the database.

- **Tier 2: Public-Facing Services**
  - These services run in a less-secured network zone and provide public information about certificate status. They read data from the Tier 1 database but cannot write to it.
  - **CRL Distribution Point:** A simple web server fronted by a Content Delivery Network (CDN). It hosts the CRL files generated by the Root and Intermediate CAs.
  - **OCSP Responder:** A high-performance, read-only service that receives OCSP requests from clients. It looks up the requested certificate's serial number in a cached copy of the certificate database and returns a signed response indicating its status (`good`, `revoked`, or `unknown`).

![Architecture Diagram](https://i.imgur.com/7gK7x0Y.png)

### 6. Data Schema and Models

- **Certificate Database Schema:**
  - `certificates` table: `serial_number` (Primary Key), `subject_name`, `issuer_id`, `valid_from`, `valid_to`, `status` ('valid', 'revoked', 'expired'), `public_key`, `raw_certificate_pem`.
  - `revocation_info` table: `serial_number`, `revocation_date`, `revocation_reason`.
- **Certificate Signing Request (CSR):** A standard format containing the public key and subject information (e.g., common name) for the certificate being requested.
- **Certificate Revocation List (CRL):** A signed file containing a list of serial numbers of all non-expired certificates that have been revoked by the CA.

### 7. API Design

- **`POST /api/v1/issue` (Intermediate CA Issuance Service):**
  - **Usage:** Request a new certificate.
  - **Body:** `{ "csr_pem": "..." }`
  - **Response:** `200 OK` with `{ "certificate_pem": "...", "chain_pem": "..." }`.

- **`POST /api/v1/revoke` (Intermediate CA Revocation Service):**
  - **Usage:** Revoke an existing certificate.
  - **Body:** `{ "serial_number": "...", "reason": "keyCompromise" }`
  - **Response:** `200 OK`.

- **`GET /ocsp/{base64_request}` (OCSP Responder):**
  - **Usage:** Standard OCSP protocol endpoint for clients to check certificate status.

### 8. Scalability and Performance

- **Issuance Scalability:** The Intermediate CA Issuance Service is a standard API and can be scaled by adding more nodes behind a load balancer. The database and HSM can become bottlenecks but can be scaled with standard techniques (read replicas, HSM clustering).
- **OCSP Scalability:** This is the most performance-critical component.
  - It is a read-only service, making it easy to scale horizontally.
  - It uses an in-memory cache of certificate statuses for low-latency responses.
  - It can be globally distributed using a CDN and Anycast routing to provide fast responses to clients worldwide.

### 9. Reliability and Fault Tolerance

- **Root CA:** Reliability is procedural. The key is stored on multiple, redundant HSMs in different geographic locations.
- **Intermediate CA:** The services and database are run in a high-availability cluster across multiple availability zones.
- **OCSP/CRL:** These services are critical for clients. If a client cannot check the revocation status of a certificate, it may refuse to trust it (hard-fail) or trust a potentially revoked certificate (soft-fail). They must be extremely reliable, hence the use of CDNs and global distribution.

### 10. Security Considerations

- **Key Protection:** This is paramount. Using HSMs is non-negotiable, as they ensure private keys can be used for signing but can never be exported.
- **Physical Security:** Strict physical access controls are required for the Root CA and Intermediate CA HSMs.
- **"Signing Ceremony":** The process of using the Root CA private key is a formal, audited procedure involving multiple people to prevent insider threats.
- **Separation of Duties:** Different roles and permissions are used for certificate requesters, approvers, and CA administrators.

### 11. Monitoring and Observability

- **Audit Log:** A secure, tamper-evident log of every certificate issued or revoked.
- **HSM Monitoring:** Monitor the health and utilization of the HSMs.
- **OCSP Metrics:** Request volume, response latency, and error rates for the OCSP responders.
- **CRL/OCSP Freshness:** Monitor to ensure that CRLs are being published on schedule and that OCSP responders have up-to-date information.

### 12. Deployment and Operations

- **Key Generation Ceremony:** A well-documented, audited procedure for generating the Root key pair.
- **Disaster Recovery:** A plan must be in place for recovering the CA in the event of a disaster, including secure backups of the HSM data and the certificate database.

### 13. Cost Estimation

- **Primary Drivers:**
  - **HSMs:** Hardware Security Modules are expensive pieces of specialized hardware.
  - **Secure Infrastructure:** The cost of physically secure data centers and isolated networks.
  - **Personnel:** The cost of the specialized security and operations personnel required to manage the CA.

### 14. Risks and Mitigations

- **Root Key Compromise:**
  - **Risk:** This is the worst-case scenario. If the Root CA private key is compromised, the entire PKI is untrustworthy.
  - **Mitigation:** There is no technical mitigation. The only solution is to destroy the entire PKI and build a new one from scratch, distributing the new root certificate to all relying parties. Prevention through extreme physical and logical security is the only viable strategy.
- **Intermediate Key Compromise:**
  - **Risk:** An Intermediate CA key is compromised.
  - **Mitigation:** The Root CA must revoke the Intermediate CA's certificate. A new Intermediate CA must be created and signed by the Root. All certificates issued by the compromised Intermediate CA are now invalid.
- **Mis-issuance:**
  - **Risk:** A bug or procedural error leads to the issuance of an incorrect or unauthorized certificate.
  - **Mitigation:** Implement strong policy checks and approval workflows before issuance. Use tools like Certificate Transparency logs (even for private CAs) to make all issued certificates publicly auditable.

### 15. Final Considerations

Building and operating a Certificate Authority is one of the most security-critical tasks in an IT infrastructure. The design must be conservative and prioritize security and procedural rigor above all else. The hierarchical model with an offline root is the industry standard for a reason: it provides the strongest possible protection for the root of trust. While the technical components are important, the operational procedures, audits, and ceremonies surrounding the CA are just as critical to its long-term security and trustworthiness.
