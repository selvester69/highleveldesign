# High-Level Design: Two-Factor Authentication (2FA) System

## 1. Document Information

- **Document Title**: Two-Factor Authentication (2FA) System - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-30
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary

### 2.1 Purpose
This document outlines the high-level design for a Two-Factor Authentication (2FA) service. This system is designed to provide a second layer of security on top of a primary authentication mechanism (like username/password). It ensures that even if a user's password is compromised, an attacker cannot gain access to their account without possessing the second factor. This design covers two common 2FA methods: Time-based One-Time Passwords (TOTP) and SMS-based codes.

### 2.2 Scope

**In Scope:**
-   An API for enrolling a user in 2FA (e.g., generating a TOTP secret, registering a phone number).
-   A service for generating and sending one-time codes via SMS.
-   An API for verifying a user-submitted code (either from a TOTP app or SMS).
-   Secure storage of user 2FA configuration and secrets.
-   Generation of single-use recovery codes for users who lose their second factor.

**Out of Scope:**
-   The primary authentication system (username/password). This 2FA service is called *after* primary authentication succeeds.
-   Other 2FA methods like push notifications, hardware U2F keys, or biometrics.
-   The user interface for enrolling in 2FA or entering a code.
-   The third-party SMS gateway provider.

### 2.3 High-Level Architecture Overview
The system is a centralized service that integrates into an existing login flow. After a user successfully authenticates with their primary factor (e.g., password), the **Login Service** calls the **2FA Service** to check if the user is enrolled. If so, the 2FA Service challenges the user for their second factor.
-   For **TOTP**, the client application calculates the current code and submits it. The 2FA Service uses the user's stored secret key to independently calculate the code and verify the match.
-   For **SMS**, the 2FA Service generates a random code, stores it temporarily, and uses a **Notification Gateway** to send it to the user's registered phone number. The user then submits this code for verification.
Upon successful verification, the 2FA Service gives a final approval to the Login Service, which can then complete the login process.

---

## 3. System Overview

### 3.1 Business Context
Stolen passwords are one of the most common causes of security breaches. Two-Factor Authentication provides a powerful, industry-standard defense against this threat. By requiring a second, separate verification step, it dramatically increases the security of user accounts, protects user data, and builds trust in the platform.

### 3.2 System Purpose
To enhance user account security by requiring a second piece of evidence (a "factor") to prove their identity, in addition to their password. The system must be reliable, secure, and easy to integrate into existing login workflows.

### 3.3 Success Criteria
-   **Security**: The 2FA secrets (TOTP keys, recovery codes) must be stored with extreme security. The verification process must be resilient to attacks. This is the top priority.
-   **Reliability**: The system, especially the SMS delivery component, must be highly reliable. Users being unable to log in because of a 2FA failure is a critical issue.
-   **Low Latency**: The verification process must be very fast (<200ms) to not add significant friction to the login experience.
-   **High Availability**: The 2FA service is in the critical path for user login and must be highly available.

### 3.4 Assumptions
-   The system is called by a trusted, internal login service after primary authentication has succeeded.
-   A third-party gateway (like Twilio) is used for sending SMS messages.
-   Users have access to a smartphone capable of running an authenticator app (for TOTP) or receiving SMS messages.

### 3.5 Constraints
-   **Dependency on SMS Providers**: SMS delivery is not guaranteed and can be subject to delays or failures outside of the system's control.
-   **Cost**: Sending SMS messages incurs a direct, per-message cost.
-   **User Experience Friction**: Adding a 2FA step inherently adds friction to the login process. The system must be as seamless as possible.
-   **Secret Management**: The system manages highly sensitive secrets that, if compromised, would undermine the entire security benefit.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
-   **FR-001 (TOTP Enrollment)**: The system must be able to generate a unique, cryptographically secure secret key for a user, to be shared via a QR code for enrollment in an authenticator app.
-   **FR-002 (SMS Enrollment)**: The system must be able to register and verify a user's phone number.
-   **FR-003 (TOTP Code Verification)**: The system must be able to verify a TOTP code submitted by a user against the stored secret. It should account for potential clock drift.
-   **FR-004 (SMS Code Generation & Sending)**: The system must be able to generate a random one-time code, send it to the user's phone number via SMS, and store it for verification.
-   **FR-005 (SMS Code Verification)**: The system must be able to verify the SMS code submitted by the user. Codes must expire after a short period.
-   **FR-006 (Recovery Code Generation)**: The system must be able to generate a set of single-use recovery codes for the user to store in a safe place.
-   **FR-007 (Recovery Code Verification)**: The system must allow a user to use one of their recovery codes in place of a TOTP/SMS code.
-   **FR-008 (Disenrollment)**: The system must provide a secure way for a user to disable 2FA on their account.

### 4.2 Non-Functional Requirements
-   **Security**: All secrets must be encrypted at rest. The system must protect against brute-force attacks on the verification endpoint.
-   **High Availability**: The service must be deployed in a highly available configuration.
-   **Low Latency**: Verification requests must be handled with very low latency.
-   **Fault Tolerance**: The system should be able to handle failures of the SMS gateway provider.
-   **Data Integrity**: The 2FA configuration data for users must be stored durably and consistently.
-   **Auditability**: Every 2FA enrollment, verification attempt (success or failure), and recovery event must be logged for security auditing.

---

### 5. Core Architecture

The system is a centralized, API-based service designed to be called by other internal services as part of a larger authentication workflow.

- **2FA Service:** A stateless API service that orchestrates all 2FA logic. It is the single entry point for all 2FA-related operations.
- **2FA Database (e.g., PostgreSQL):**
  - Stores the 2FA configuration for each user.
  - **`user_2fa_settings` table:** `user_id`, `is_totp_enabled`, `is_sms_enabled`, `phone_number`.
  - **`totp_secrets` table:** `user_id`, `encrypted_secret_key`. The secret key is encrypted at the application layer before being stored.
  - **`recovery_codes` table:** `user_id`, `hashed_code`. The recovery codes are hashed (like passwords) so they cannot be read from the database.
- **One-Time Code Cache (e.g., Redis):**
  - A short-lived cache used to store generated SMS codes.
  - When an SMS code is generated for a user, it's stored in Redis with a short TTL (e.g., 5 minutes). The verification logic checks against this cache.
- **Notification Gateway:**
  - An internal service that abstracts away the details of third-party notification providers.
  - The 2FA Service calls this gateway with a request like `send_sms(phone_number, message)`.
  - This gateway handles the integration with one or more external SMS providers (e.g., Twilio, Vonage). This abstraction allows for failing over to a backup provider if the primary one is down.
- **Secrets Management (e.g., Vault, AWS KMS):**
  - A secure service that stores the master encryption key used to encrypt and decrypt the TOTP secrets in the database. The 2FA service requests this key at startup.

**Verification Flows:**
- **TOTP Verification:**
  1.  2FA Service receives a `user_id` and a `totp_code`.
  2.  It retrieves the `encrypted_secret_key` for the user from the database.
  3.  It calls the Secrets Management service to get the decryption key.
  4.  It decrypts the user's secret key in memory.
  5.  It uses a standard TOTP library to generate the current expected code (and codes for the previous and next time windows, to allow for clock skew).
  6.  It compares the user's submitted code to the expected codes and returns a pass/fail result.
- **SMS Verification:**
  1.  2FA Service receives a request to `send_sms_code` for a `user_id`.
  2.  It generates a cryptographically random 6-digit code.
  3.  It stores a hash of the code in the One-Time Code Cache with a 5-minute TTL: `SET user_id:sms_code HASH(code) EX 300`.
  4.  It calls the Notification Gateway to send the code to the user's registered phone number.
  5.  When the user submits the code, the service hashes the submission and compares it to the hash stored in the cache.

![Architecture Diagram](https://i.imgur.com/k9v0v0n.png)

### 6. Data Schema and Models

- **TOTP Secret:** A randomly generated secret key (e.g., 160 bits), typically encoded in Base32 for user-friendliness.
- **SMS Code:** A 6-8 digit random number.
- **Recovery Code:** A set of 8-10 longer, high-entropy alphanumeric codes. They are treated like single-use passwords and should be hashed in the database.

### 7. API Design

The API is internal, called by other services like a login or account management service.

- **`POST /api/v1/2fa/enroll/totp`**:
  - **Body:** `{ "user_id": "..." }`
  - **Response:** `{ "secret_key": "...", "qr_code_image": "data:image/png;base64,..." }`
- **`POST /api/v1/2fa/enroll/sms`**:
  - **Body:** `{ "user_id": "...", "phone_number": "..." }`
- **`POST /api/v1/2fa/challenge/sms`**:
  - **Usage:** Triggers an SMS code to be sent.
  - **Body:** `{ "user_id": "..." }`
- **`POST /api/v1/2fa/verify`**:
  - **Usage:** The core verification endpoint.
  - **Body:** `{ "user_id": "...", "type": "totp|sms|recovery", "code": "..." }`
  - **Response:** `{ "verified": true/false }`

### 8. Scalability and Performance

- **Stateless Service:** The 2FA Service itself is stateless, allowing it to be scaled horizontally to handle high request volumes during peak login times.
- **Database Performance:** The database workload is simple key-value lookups (`WHERE user_id = ...`), which are easily indexed and scaled.
- **Cache:** The Redis cache for SMS codes is also highly scalable.

### 9. Reliability and Fault Tolerance

- **High Availability:** The 2FA Service, database, and cache are all deployed in a highly available configuration across multiple AZs.
- **SMS Gateway Failover:** The Notification Gateway is designed to automatically fail over to a secondary SMS provider if the primary provider's API returns errors.
- **Recovery Codes:** Recovery codes are the ultimate fallback. They provide a reliable way for users to regain access if they lose their phone, ensuring the 2FA system doesn't permanently lock users out.

### 10. Security Considerations

- **Secret Encryption:** The application-layer encryption of TOTP secrets is critical. Even if the database is compromised, the secrets remain protected without the master key from the Secrets Management service.
- **Brute-Force Protection:** The `/verify` endpoint must be strictly rate-limited on a per-user basis. After a small number of failed attempts (e.g., 5), the user's account should be temporarily locked.
- **Code Expiry:** SMS and recovery codes must be invalidated immediately after use. SMS codes must have a short expiry time.
- **Secure Generation:** All secrets (TOTP keys, SMS codes, recovery codes) must be generated using a cryptographically secure pseudo-random number generator (CSPRNG).

### 11. Monitoring and Observability

- **Verification Rates:** Monitor the success and failure rates for each 2FA method. A sudden spike in failures could indicate an attack or a system malfunction.
- **SMS Delivery Rates:** Monitor the success/failure rates reported by the SMS gateway provider.
- **Audit Trail:** A tamper-evident log of all 2FA events (enrollment, verification, recovery code use) is essential for security investigations.

### 12. Deployment and Operations

- **Secrets Management:** The operational security of the master encryption key in the secrets management system is of the utmost importance.
- **Provider Management:** Manage relationships and billing with third-party SMS providers.

### 13. Cost Estimation

- **Primary Driver:** The cost of sending SMS messages, which is billed per message.
- **Compute and Database:** The cost of running the service and its database, which is generally modest as the service is not resource-intensive.

### 14. Risks and Mitigations

- **SMS Pumping Fraud:**
  - **Risk:** Attackers use the SMS 2FA feature on a large scale to send messages to premium-rate numbers that they control, generating revenue for themselves and a large bill for the company.
  - **Mitigation:** Implement strict rate limiting on phone number verification and code generation endpoints. Use fraud detection services from the SMS provider.
- **SIM Swapping Attack:**
  - **Risk:** An attacker convinces a mobile carrier to transfer a user's phone number to a SIM card they control, thereby intercepting the SMS 2FA code.
  - **Mitigation:** This is a difficult-to-solve, real-world attack. The best mitigation is to encourage users to use more secure TOTP-based authentication instead of SMS. The system can also look for signals of a recent SIM swap.
- **Compromise of Secrets:**
  - **Risk:** The database or the secrets manager is compromised.
  - **Mitigation:** Defense-in-depth. Encrypt secrets in the database. Tightly control access to the database and the secrets manager. Have a plan to force a 2FA re-enrollment for all users in a worst-case scenario.

### 15. Final Considerations

A Two-Factor Authentication system is a relatively simple but powerful security control. The design must be defensive at every layer, from protecting against brute-force attacks at the API to securely storing secrets at rest. While providing multiple 2FA methods like TOTP and SMS is user-friendly, it's also important to recognize their different security levels and guide users toward the most secure options available. The reliability of the system is just as important as its security, as failures can lock legitimate users out of their accounts.
