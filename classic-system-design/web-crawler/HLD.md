# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: Web Crawler - High-Level Design
- **Version**: 1.0
- **Date**: 2025-08-22
- **Author(s)**: Jules (AI Agent)
- **Reviewers**: [Reviewer Names and Roles]
- **Approvers**: [Approver Names and Roles]
- **Document Status**: Draft

### Revision History
| Version | Date       | Author          | Changes         |
|---------|------------|-----------------|-----------------|
| 1.0     | 2025-08-22 | Jules (AI Agent)| Initial version |

### Distribution List
- Stakeholders
- Development Team
- Operations Team

---

## 2. Executive Summary

### 2.1 Purpose
This document provides the high-level design for a scalable, distributed Web Crawler. The system is designed to discover and download web pages from the internet, process their content, and extract links to discover new pages, forming the foundation of a search engine's indexing pipeline.

### 2.2 Scope
**In Scope:**
- A distributed, multi-threaded crawling architecture.
- A sophisticated URL Frontier to manage the list of URLs to visit.
- Handling of `robots.txt` to ensure polite crawling.
- Content parsing to extract text and hyperlinks.
- Deduplication of crawled content.
- Scalable storage for raw page content.

**Out of Scope:**
- A full search engine index (e.g., inverted index). This design focuses only on the crawling and page processing part.
- Page rendering (e.g., executing JavaScript). The crawler will only process raw HTML.
- Advanced page ranking algorithms (like PageRank).
- A user-facing search interface.

### 2.3 Key Benefits
- **Scalability**: Designed to crawl billions of web pages by distributing the load across many machines.
- **Politeness**: Built-in respect for `robots.txt` and rate limiting to avoid overloading web servers.
- **Resilience**: The system is designed to be fault-tolerant, handling network errors and unresponsive servers gracefully.
- **Extensibility**: The modular design allows for easy addition of new content processors or parsers.

### 2.4 High-Level Architecture Overview
The architecture is a large-scale, distributed pipeline. It begins with a set of seed URLs that are fed into a central **URL Frontier**. The Frontier manages the immense list of discovered URLs, prioritizing them and ensuring politeness. Multiple **Fetcher** nodes retrieve pages from the Frontier, download the content, and pass it to **Parser** nodes. The Parsers extract text content and new links. The extracted links are sent back to the URL Frontier for future crawling, and the page content is stored in a scalable object store. This entire process is asynchronous and designed for high throughput and resilience.

---

## 3. System Overview

### 3.1 Business Context
Web crawlers are the fundamental data collection backbone of the internet. They empower services ranging from search engines (like Google) to market research tools and data archiving projects. A robust, scalable, and polite crawler is a critical asset for any organization that needs to process web-scale data.

### 3.2 System Purpose
The purpose of this system is to crawl the web in a scalable and efficient manner, starting from a set of seed URLs. It will download web pages, parse them for new links to discover more of the web, and store the page content for downstream processing.

### 3.3 Success Criteria
- **Crawl Rate**: Achieve a sustained crawl rate of 1,000 pages per second.
- **Scalability**: The system should be able to scale to store and manage 10 billion URLs in the frontier.
- **Politeness**: The system must strictly adhere to `robots.txt` rules and implement per-host rate limiting to avoid overwhelming servers.
- **Coverage**: The system should be designed to eventually crawl a significant portion of the public web.

### 3.4 Assumptions
- The web contains many "crawler traps" (infinite loops of generated pages) that must be handled.
- Network latency and server errors are common and must be handled gracefully.
- The content of the web is highly redundant; deduplication is critical.
- A significant portion of the web is dynamically generated with JavaScript, but this version will focus on static HTML.

### 3.5 Constraints
- The system must be cost-efficient in terms of network bandwidth, CPU, and storage.
- The system must be a "good citizen" of the internet, avoiding any behavior that could be perceived as a denial-of-service attack.
- The legal and ethical implications of storing web content must be considered.

### 3.6 Dependencies
- A large-scale object storage system (e.g., HDFS, S3).
- A distributed database or queuing system for the URL frontier.
- A DNS resolution service.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements
- **FR-001**: The system must be able to take a list of seed URLs to start crawling.
- **FR-002**: The system must fetch and parse `robots.txt` for each new domain before crawling it.
- **FR-003**: The system must download the content of a given URL.
- **FR-004**: The system must parse HTML content to extract all hyperlinks.
- **FR-005**: The system must add newly discovered URLs to the URL frontier.
- **FR-006**: The system must detect and store a hash of page content to identify duplicates.
- **FR-007**: The system must maintain a record of when each page was last crawled.

### 4.2 Non-Functional Requirements
#### 4.2.1 Performance Requirements
- **Page Download Latency**: Highly variable, but timeouts should be enforced (e.g., 10 seconds).
- **Throughput**: 1,000 pages/sec, which translates to ~86 million pages/day.
- **URL Frontier Latency**: Adding and retrieving URLs from the frontier should take < 10ms.

#### 4.2.2 Scalability Requirements
- The URL frontier must scale to hold tens of billions of URLs.
- The content store must scale to petabytes of data.
- The number of Fetcher and Parser nodes must be horizontally scalable to thousands of instances.

#### 4.2.3 Availability Requirements
- The system should be fault-tolerant. Failure of a single crawler node should not halt the entire system.
- The URL frontier must be highly available (99.95%).
- **RTO**: < 30 minutes.
- **RPO**: < 1 hour for the URL frontier state.

#### 4.2.4 Security Requirements
- The crawler must identify itself with a clear User-Agent string.
- The system must not crawl firewalled or private networks.
- Internal components must communicate over a secure channel.

#### 4.2.5 Politeness Requirements
- **robots.txt**: Strictly adhere to `Disallow` rules.
- **Crawl-Delay**: Respect the `Crawl-Delay` directive in `robots.txt`.
- **Rate Limiting**: If no `Crawl-Delay` is specified, enforce a default delay between requests to the same host (e.g., 1-2 seconds).

#### 4.2.6 Reliability Requirements
- The system must implement robust retry logic for transient network or server errors.
- URL processing should be idempotent. Crawling the same URL twice should not create duplicate entries.

---

## 5. Architecture Design

### 5.1 Architecture Principles
- **Asynchronous & Loosely Coupled**: The pipeline stages (Fetch, Parse, Index) are decoupled via message queues.
- **Scalability & Parallelism**: The system is designed for massive parallelism, with many nodes performing the same tasks.
- **Politeness as a Priority**: The design must prevent the crawler from overwhelming any single web server.
- **Design for Failure**: Any component can fail without bringing down the entire crawling operation.

### 5.2 Architecture Patterns
- **Distributed Pipeline**: The crawling process is a multi-stage pipeline (Frontier -> Fetcher -> Parser -> Frontier).
- **Service-Oriented Architecture**: The system is composed of specialized services (e.g., DNS Service, robots.txt Service).

### 5.3 High-Level Architecture Diagram
```
                               +-----------------+
                               |   Seed URLs     |
                               +-------+---------+
                                       |
                                       v
+-------------------------------------------------------------------------+
|                               URL Frontier                              |
|                                                                         |
|  +-----------------+  +-----------------+  +--------------------------+ |
|  |  URL Prioritizer  |  |  URL Queues     |  | URL Deduplication Filter | |
|  | (By Host/Rank)    |  | (Per-Host)      |  | (e.g., Bloom Filter)     | |
|  +-----------------+  +-------+---------+  +--------------------------+ |
|                               |                                         |
+-------------------------------|-----------------------------------------+
                                | (URLs to Fetch)
                                v
+-------------------------------------------------------------------------+
|                           Fetcher Nodes (Pool)                          |
|                                                                         |
| [ F1 ] [ F2 ] [ F3 ] ... [ Fn ]                                          |
| - Fetches robots.txt (via Robots Service)                               |
| - Fetches page content                                                  |
| - Passes (URL, Content) to Parser                                       |
+-------------------------------+-----------------------------------------+
                                | (Raw HTML Content)
                                v
+-------------------------------------------------------------------------+
|                           Parser Nodes (Pool)                           |
|                                                                         |
| [ P1 ] [ P2 ] [ P3 ] ... [ Pn ]                                          |
| - Parses HTML to extract text and links                                 |
| - Normalizes and cleans extracted links                                 |
+-------------------------------+-----------------------------------------+
      | (Extracted Text)          | (Extracted Links)
      v                           v
+-----------------+     +----------------------------------+
|  Content Store  |     | Back to URL Frontier for         |
| (e.g., S3/HDFS) |     | processing and potential queuing |
+-----------------+     +----------------------------------+
```

### 5.4 Component Overview
- **URL Frontier**: The brain of the crawler. It's responsible for storing all known URLs, deciding which ones to crawl next, and enforcing politeness.
  - **URL Queues**: The Frontier maintains many queues, typically one for each hostname, to enforce per-host politeness delays.
  - **URL Prioritizer**: A component that scores URLs based on factors like PageRank, update frequency, etc., to decide which queues to service.
  - **Deduplication Filter**: A probabilistic data structure like a Bloom Filter or Cuckoo Filter to quickly check if a URL has already been seen, preventing redundant processing.
- **Fetcher Nodes**: A large pool of worker nodes whose only job is to download content. A Fetcher gets a URL from the Frontier, makes an HTTP request, and retrieves the raw page content. It also handles DNS lookups and respects `robots.txt` rules.
- **Robots.txt Service**: A dedicated service that fetches and caches `robots.txt` files to avoid re-fetching them for every request to a given host.
- **Parser Nodes**: A pool of worker nodes that take raw page content (e.g., HTML) from Fetchers. They parse the document to extract two key things: the actual text content and all the outgoing hyperlinks.
- **Content Store**: A highly scalable, durable object store like Amazon S3 or HDFS. It stores the raw and/or processed page content, often compressed, for later use by indexing systems.

### 5.5 Technology Stack
#### 5.5.1 Backend Technologies
- **Programming Language**: **C++** or **Go**.
  - **Justification**: For a high-performance system like a web crawler where efficiency and control over memory and networking are paramount, a language like C++ or Go is a strong choice.
- **Framework**: Custom frameworks are common, but libraries like `libcurl` (C++) or Go's standard `net/http` would be used.

#### 5.5.2 Database Technologies
- **URL Frontier**: Often a custom solution built on top of a distributed database like **Cassandra** or a transactional key-value store like **FoundationDB**. Using a standard message queue is often not sufficient due to the need for complex prioritization and scheduling.
- **URL Deduplication**: **Redis** (for its Bloom Filter implementation) or an in-memory data structure within the Frontier service itself.
- **Content Storage**: **Amazon S3**, **Google Cloud Storage**, or **HDFS**.
- **Robots.txt Cache**: **Redis** or **Memcached**.

#### 5.5.3 Infrastructure & DevOps
- **Deployment**: The system is typically deployed across multiple data centers to be closer to web servers around the world and for resilience.
- **Orchestration**: **Kubernetes** is a good fit for managing the pools of Fetcher and Parser nodes.

### 5.6 Architecture Decision Records (ADRs)
#### 5.6.1 ADR-001: URL Frontier Implementation
- **Status**: Accepted
- **Context**: The URL Frontier must manage billions of URLs, enforce politeness on millions of hosts simultaneously, and allow for flexible prioritization. A simple database or message queue is insufficient.
- **Decision**: We will implement a **custom URL Frontier service**. It will use a combination of data stores. A distributed key-value store (like Cassandra) will hold the main URL metadata. Each host will have its own queue for URLs to be crawled. A scheduler process will manage these queues, moving URLs to a "ready to be fetched" state based on politeness delays and priority scores. Fetcher nodes will pull from this ready state.
- **Consequences**: This is a complex component to build but is essential for a high-performance, polite crawler. It isolates the complex scheduling logic from the simple Fetcher nodes.

---

## 6. Detailed Component Design

### 6.1 Component 1: URL Frontier
#### 6.1.1 Purpose
To manage the lifecycle of all URLs known to the crawler.
#### 6.1.2 Responsibilities
- Accept new URLs from the Parser nodes.
- Deduplicate incoming URLs against already-seen URLs.
- Prioritize URLs for crawling based on a scoring algorithm.
- Group URLs by hostname and dispense them to Fetchers while respecting politeness delays.
- Persist the state of all URLs.
#### 6.1.3 Interfaces
- **Input**: A stream of newly discovered URLs.
- **Output**: A stream of URLs that are ready to be fetched.

### 6.2 Component 2: Fetcher Node
#### 6.2.1 Purpose
To download web page content.
#### 6.2.2 Responsibilities
- Request a batch of URLs from the URL Frontier.
- For each URL, perform a DNS lookup to get the IP address.
- For each new host, request the `robots.txt` file from the Robots.txt service.
- Make an HTTP GET request to download the page content.
- Handle redirects, timeouts, and connection errors.
- Emit the downloaded content for the Parser nodes.
#### 6.2.3 Dependencies
- URL Frontier, DNS Service, Robots.txt Service.

### 6.3 Component 3: Parser Node
#### 6.3.1 Purpose
To process raw web content to extract links and text.
#### 6.3.2 Responsibilities
- Consume raw page content from Fetcher nodes.
- Parse the HTML document structure.
- Extract all hyperlink (`<a>`) tags.
- Normalize extracted URLs (e.g., convert relative URLs to absolute URLs).
- Extract plain text content from the page.
- Send extracted links back to the URL Frontier.
- Send extracted text to the Content Store.

---

## 7. Data Design

### 7.1 Data Architecture
Data is stored in several specialized systems: a high-throughput database for the URL frontier state, a simple cache for `robots.txt` rules, and a massive object store for the crawled content itself.

### 7.2 Data Models
#### 7.2.1 Logical Data Model
- **URL Record**: `url`, `url_hash`, `host`, `last_crawled_timestamp`, `priority_score`, `content_hash`.
- **Content**: The raw HTML or text content of a page.
- **Robots Rule**: `host`, `disallowed_paths`, `crawl_delay`.

### 7.3 Database Design
#### 7.3.1 Database Schema (Illustrative)
**Table: `url_metadata` (Cassandra)**
- `host` (TEXT, Partition Key)
- `url_hash` (BIGINT, Clustering Key): Hash of the full URL.
- `full_url` (TEXT)
- `priority` (FLOAT)
- `last_crawled` (TIMESTAMP)
- `status` (TEXT): e.g., 'UNCRAWLED', 'IN_FLIGHT', 'CRAWLED', 'ERROR'.

**Data Structure: `seen_urls` (Bloom Filter in Redis/Memory)**
- A probabilistic filter to quickly answer "Have we ever seen this URL before?".

**Object Store: `page_content` (S3)**
- **Path/Key**: `s3://crawl-data/{YYYY-MM-DD}/{content_hash}.html.gz`
- Content is stored compressed, named by its content hash for automatic deduplication.

#### 7.3.2 Data Flow
1. A **Parser** extracts a new URL.
2. It sends the URL to the **Frontier**.
3. The **Frontier** checks the **Bloom Filter**. If the URL is new, it's added to the `url_metadata` table with 'UNCRAWLED' status.
4. A scheduler in the **Frontier** selects a host, checks its politeness delay, and if ready, pulls a high-priority URL from that host's queue.
5. The URL is given to a **Fetcher**. The URL's status is updated to 'IN_FLIGHT'.
6. The **Fetcher** downloads the page and sends the content to a **Parser**.
7. The **Parser** extracts text, calculates a `content_hash`, and stores the content in S3.
8. The **Frontier** is notified that the URL is 'CRAWLED' and its `last_crawled` and `content_hash` fields are updated.

---

## 8. API Design

### 8.1 API Architecture
The system is primarily internally driven and does not have a large public API surface. The main external interaction is with the web itself. Internal APIs between microservices will be based on gRPC for high performance.

### 8.2 API Specifications
#### 8.2.1 Core APIs (Internal gRPC)
##### Service: `URLFrontier`
- `GetWork(request) returns (stream URLInfo)`: A streaming RPC for Fetchers to request work.
- `SubmitNewURLs(stream URL) returns (SubmitResult)`: A streaming RPC for Parsers to submit newly found links.

##### Service: `RobotsService`
- `IsAllowed(request) returns (bool)`: Checks if a given URL is allowed to be crawled for a given user-agent.

### 8.3 API Documentation Strategy
Internal APIs will be documented using Protocol Buffer (`.proto`) definitions, which serve as a contract and documentation.

---

## 9. Security Design

### 9.1 Security Architecture
The crawler's security posture is primarily defensive and aims to protect the crawler infrastructure itself, while also ensuring the crawler is a good actor on the internet.

### 9.2 Authentication & Authorization
- All internal gRPC APIs will be secured with mTLS to ensure only authenticated services can communicate.
- Access to operational dashboards and control planes will be behind SSO and be role-based.

### 9.3 Application Security
- **User-Agent**: The crawler will identify itself with a clear `User-Agent` string that includes a link to a page with more information about the crawler's purpose and policies.
- **Crawler Traps**: The system must have mechanisms to detect and avoid crawler traps (e.g., calendar pages that generate infinite future links). This can be done by tracking URL path depth and detecting repetitive URL patterns.
- **Resource Consumption**: The crawler must be careful about how it consumes resources on remote servers. This is managed by the politeness mechanisms in the URL Frontier.

---

## 10. Scalability & Performance

### 10.1 Performance Requirements
- As defined in section 4.2.1. The main goal is high throughput (pages/sec).

### 10.2 Scalability Strategy
- **Fetcher/Parser Scaling**: The Fetcher and Parser pools are stateless and can be scaled horizontally nearly infinitely. Auto-scaling rules will be based on the size of the input queues.
- **URL Frontier Scaling**: This is the most complex component to scale. It will be sharded, likely by hostname, so that the responsibility for managing queues for different hosts is distributed across multiple Frontier nodes.
- **DNS Scaling**: A distributed, caching DNS resolver will be used to handle the massive number of DNS lookups required.

### 10.3 Caching Strategy
- **robots.txt Cache**: A distributed cache (Redis) will store `robots.txt` rules for millions of hosts, with a TTL of ~24 hours.
- **DNS Cache**: The DNS resolver will heavily cache DNS records to reduce latency and load on upstream DNS servers.
- **Content Cache**: Not applicable in the same way as other systems. The goal is to fetch fresh content, not serve cached content.

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture
- The system will be deployed on Kubernetes clusters, potentially across multiple geographic regions to reduce latency to target web servers.

### 11.2 Monitoring & Alerting
- **Key Metrics**:
  - `pages_crawled_per_second`
  - `frontier_size` (total URLs, and by host)
  - `http_status_codes` (2xx, 3xx, 4xx, 5xx rates)
  - `dns_latency`
  - `robots_cache_hit_ratio`
- **Dashboards**: Grafana dashboards will provide a real-time view of the crawler's health and performance.
- **Alerting**: Alerts will be triggered for high error rates, stalled queues in the frontier, or low crawl throughput.

---

## 12. Testing Strategy

### 12.1 Test Types
- **Unit & Integration Tests**: For individual components.
- **E2E Testing**: A dedicated test environment will be set up with a small, local "web" of a few hundred pages. The test will seed the crawler with one URL and verify that it correctly crawls all the pages, respects `robots.txt`, and extracts the links.
- **Performance Testing**: A "dark launch" approach can be used, where the crawler is pointed at a small fraction of the live web to measure performance before scaling up.

---

## 13. Risk Analysis

### 13.1 Technical Risks
| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| URL Frontier becomes a bottleneck | High | High | A sharded, distributed design is critical. The logic must be highly optimized. Continuous performance monitoring is essential. |
| Handling "bad" HTML | Medium | High | Use robust, battle-tested HTML parsing libraries that can handle malformed documents without crashing. |
| Storage cost explosion | High | Medium | Implement aggressive compression on stored content. Use intelligent storage tiering (e.g., S3 Standard-IA) for older data. |

### 13.2 Operational Risks
| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Getting Blocked/Banned | High | High | Strictly adhere to politeness rules. Rotate IP addresses for Fetcher nodes. Maintain a clear `User-Agent` and contact info. |
| Legal/Ethical Issues | High | Medium | Have a clear data usage policy. Respect copyright. Provide a mechanism for site owners to request removal of their content. |

---

## 14. Implementation Plan (High-Level)

- **Phase 1**: Build the core data stores and a single-node version of the Frontier, Fetcher, and Parser to prove the logic.
- **Phase 2**: Implement the distributed URL Frontier with sharding and politeness.
- **Phase 3**: Scale out the Fetcher and Parser pools on Kubernetes.
- **Phase 4**: Implement robust monitoring, alerting, and operational controls.
- **Phase 5**: Begin crawling a small seed set and gradually expand.

---

## 15. Appendices

### Appendix A: Glossary
- **URL Frontier**: The data structure that manages the list of URLs to be crawled.
- **Politeness**: A set of policies to ensure a web crawler does not overwhelm a web server.
- **Crawler Trap**: A part of a website that is designed to cause a crawler to make an infinite number of requests.
- **Bloom Filter**: A space-efficient probabilistic data structure used to test whether an element is a member of a set.
