# HLD: Auction System

## 1. Document Information
- **Document Title**: High-Level Design for a Real-Time Auction System
- **Version**: 1.0
- **Date**: 2025-08-31
- **Author**: Jules (AI Agent)
- **Status**: Draft

---

## 2. Executive Summary
This document outlines the high-level design for a real-time Auction System, similar to the core functionality of eBay. The system is designed to manage time-bound bidding on items, handle concurrent bid placements from many users, and reliably determine a winner. The architecture prioritizes low-latency communication for a responsive bidding experience, strong consistency for processing bids, and temporal accuracy for managing auction lifecycles.

---

## 3. Scope

### 3.1 In-Scope
-   **Auction Creation**: Allowing sellers to list items for auction with a start price, duration, and end time.
-   **Live Bidding**: Allowing buyers to place bids on active auctions.
-   **Real-time Updates**: Broadcasting new bids to all participants of an auction in real-time.
-   **Proxy Bidding**: Automatically placing bids for a user up to their specified maximum amount.
-   **Winner Determination**: Identifying and declaring the highest bidder as the winner when the auction concludes.
-   **Notifications**: Notifying the winner and seller after the auction ends.

### 3.2 Out-of-Scope
-   **Payment & Shipping**: This is handed off to the Order Management and Fulfillment services post-auction.
-   **User Reputation System**: A separate service would manage seller and buyer ratings.
-   **Different Auction Types**: This design focuses on standard English auctions. Other types like Dutch or Vickrey auctions are not included.

---

## 4. Core Functional Requirements
-   **FR-001**: A seller must be able to create an auction for an item with a defined end time.
-   **FR-002**: A buyer must be able to place a bid on an active auction.
-   **FR-003**: The system must reject any bid that is not higher than the current winning bid.
-   **FR-004**: All users viewing an auction must receive immediate updates when a new higher bid is placed.
-   **FR-005**: A buyer can place a "max bid" (proxy bid), and the system will automatically bid on their behalf up to that amount.
-   **FR-006**: At the exact end time, the auction must be closed, and the highest bidder declared the winner.

---

## 5. Key Non-Functional Requirements (NFRs)
-   **Latency**: **Very low latency is critical for bidding.** Bid placement and the subsequent broadcast to other clients should have a p99 latency of < 200ms.
-   **Consistency**: **Strong consistency** is required for bid processing to avoid race conditions and ensure there is only one valid high bidder at any time.
-   **Temporal Accuracy**: The system's time must be synchronized and accurate to end auctions fairly and precisely.
-   **Availability**: 99.95% uptime for the bidding and auction services.
-   **Scalability**: Must handle many concurrent auctions and high bidding traffic, especially in the final minutes of popular auctions ("sniping").

---

## 6. High-Level Architecture Diagram

```mermaid
graph TD
    subgraph "Clients"
        Client["Web/Mobile Client"]
    end

    subgraph "Real-time Bidding Layer"
        BiddingGateway["Bidding Gateway (WebSocket)"]
    end

    subgraph "Backend Services"
        AuctionService["Auction Service"]
        ProxyBidEngine["Proxy Bidding Engine"]
        AuctionScheduler["Auction Ending Scheduler"]
    end

    subgraph "Data Stores"
        AuctionDB["Auction DB (PostgreSQL)"]
        ActiveAuctionCache["Active Auction Cache (Redis)"]
    end

    %% Connections
    Client -- "WebSocket Connection" <--> BiddingGateway
    Client -- "HTTP for initial data" --> AuctionService

    BiddingGateway -- "Forwards valid bids" --> AuctionService
    AuctionService -- "Processes bids" --> ProxyBidEngine
    ProxyBidEngine -- "Reads/Writes" --> AuctionDB
    AuctionService -- "Reads/Writes" --> AuctionDB
    AuctionService -- "Caches hot data in" --> ActiveAuctionCache

    AuctionService -- "Schedules end_auction job" --> AuctionScheduler
    AuctionScheduler -- "Triggers AuctionService at end_time" --> AuctionService
```

---

## 7. Component Breakdown
-   **Bidding Gateway (WebSocket)**: A stateful service that maintains persistent WebSocket connections with clients. It's responsible for receiving bids and broadcasting real-time price updates to subscribed users.
-   **Auction Service**: The core backend service. It handles the business logic of creating auctions, validating bids, and executing the winner determination logic.
-   **Proxy Bidding Engine**: A component within the Auction Service that manages the logic for proxy bids. When a new bid is placed, this engine checks against existing proxy bids to see if a counter-bid needs to be automatically placed.
-   **Auction Ending Scheduler**: A distributed, time-based scheduler (e.g., built on Quartz, or using Kafka with delayed topics). It is responsible for invoking the Auction Service to end an auction at its precise `end_time`.
-   **Auction DB (PostgreSQL)**: The source of truth for all auction and bid data. A relational database is chosen for its strong transactional capabilities.
-   **Active Auction Cache (Redis)**: Caches details of currently active auctions to reduce database read load.

---

## 8. Technology Stack
-   **Backend Services**: **Java/Kotlin (Spring)** or **Go**.
-   **Real-time Layer**: A WebSocket library like `gorilla/websocket` (Go) or Spring WebFlux.
-   **Database**: **PostgreSQL**.
-   **Cache**: **Redis**.
-   **Scheduler**: A robust library like **Quartz**, or a custom implementation using a message broker.

---

## 9. Data Schema / Data Model

```mermaid
erDiagram
    AUCTIONS {
        UUID auction_id PK
        UUID item_id
        UUID seller_id
        Timestamp start_time
        Timestamp end_time
        Decimal start_price
        Decimal current_price
        UUID current_high_bidder_id
        String status "ACTIVE, ENDED, CANCELLED"
    }

    BIDS {
        UUID bid_id PK
        UUID auction_id FK
        UUID user_id FK
        Decimal amount
        Timestamp created_at
    }

    PROXY_BIDS {
        UUID proxy_bid_id PK
        UUID auction_id FK
        UUID user_id FK
        Decimal max_amount
        Boolean is_active
    }

    AUCTIONS ||--|{ BIDS : "has"
    AUCTIONS ||--|{ PROXY_BIDS : "has"
```

---

## 10. API Endpoints & WebSocket Events
-   `POST /api/v1/auctions`: (HTTP) Create a new auction.
-   `GET /api/v1/auctions/{auctionId}`: (HTTP) Get static details about an auction.
-   `POST /api/v1/auctions/{auctionId}/proxy_bid`: (HTTP) Set or update a user's max proxy bid.

-   **WebSocket Events (Client -> Server)**:
    -   `subscribe_auction { "auctionId": "..." }`
    -   `place_bid { "auctionId": "...", "amount": 120.50 }`

-   **WebSocket Events (Server -> Client)**:
    -   `bid_update { "auctionId": "...", "newPrice": 120.50, "highBidder": "user123" }`
    -   `auction_ended { "auctionId": "...", "finalPrice": 150.00, "winner": "user456" }`
    -   `error { "message": "Bid too low" }`

---

## 11. Core Logic / User Flows (Proxy Bidding)

```mermaid
sequenceDiagram
    participant UserA
    participant BiddingGateway
    participant AuctionService
    participant DB

    Note over UserA, DB: UserB has an active proxy bid with max_amount = $50. Current price is $20.

    UserA->>+BiddingGateway: place_bid { amount: 25 }
    BiddingGateway->>+AuctionService: Process bid for $25
    AuctionService->>+DB: Lock auction record
    AuctionService->>DB: Check for higher proxy bids
    DB-->>AuctionService: Found UserB's proxy bid (max $50)

    AuctionService->>AuctionService: Logic: UserB's proxy bid must counter.
    AuctionService->>DB: 1. Place bid for UserB at $26 (UserA's $25 + increment)

    AuctionService->>AuctionService: Logic: UserA's bid is now outbid.

    DB-->>AuctionService: Update successful
    AuctionService->>+DB: Unlock auction record

    AuctionService-->>-BiddingGateway: Bid processed. New price is $26 by UserB.
    BiddingGateway-->>UserA: bid_update { newPrice: 26.00, highBidder: "UserB" }
```

---

## 12. Scalability and Performance Design
-   **Separation of Concerns**: The WebSocket gateways are separated from the core auction logic, allowing each to be scaled independently.
-   **Database Contention**: For very popular auctions, the corresponding row in the `auctions` table will be a hot spot. Operations must be very fast, using row-level locking. For extreme scale, consider sharding auctions by ID.
-   **Reliable Scheduler**: The scheduler must be horizontally scalable and fault-tolerant to manage millions of auction end times.
-   **Fan-out on Write**: A single bid is written to the DB, but the update is fanned out to many clients via the WebSocket gateway.

---

## 13. Security and Resilience
-   **Security**:
    -   All bid placements must be authenticated.
    -   The system must prevent users from bidding on their own items.
    -   Implement rate limiting to prevent bidding bots from overwhelming the system.
-   **Resilience**:
    -   **Idempotent End-Auction**: The process to end an auction must be idempotent. If a worker fails mid-process, another can safely re-run it.
    -   **Scheduler Reliability**: Use a persistent job store for the scheduler so that jobs are not lost if a scheduler node goes down.
    -   **Transactional Integrity**: All bid processing and state changes must occur within a single database transaction.

---

## 14. Monitoring and Observability
-   **Metrics**:
    -   WebSocket connection count.
    -   Bid processing latency (from receipt to broadcast).
    -   Scheduler job execution lag and failure rate.
    -   Database lock wait times.
-   **Alerting**:
    -   Alert if an auction is not processed within a few seconds of its scheduled end time.
    -   Alert on high bid rejection rates.

---

## 15. Deployment and CI/CD
-   The Bidding Gateway and Auction Service can be deployed independently.
-   Rigorous testing of the proxy bidding logic and the auction-ending process is critical. This includes tests for race conditions and temporal accuracy.
-   A canary deployment strategy can be used to roll out new versions of the bidding logic to a small subset of users first.
