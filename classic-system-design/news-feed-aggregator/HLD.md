# High-Level Design Document

## 1. Document Information

### Document Metadata

- **Document Title**: News Feed Aggregator - High-Level Design
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

This document provides the high-level design for a News Feed Aggregator service, similar to Reddit or Hacker News. The system allows users to submit links, engage in discussions, and view a ranked feed of popular content.

### 2.2 Scope

**In Scope:**

- User accounts and authentication.
- Submission of new posts (links with titles).
- A system for users to upvote (or downvote) posts.
- A ranking algorithm to determine a post's score and position on the front page.
- Threaded comments and discussions on posts.
- Creation of different communities or topics (e.g., subreddits).

**Out of Scope:**

- Real-time features like live comment updates.
- User profiles with detailed karma history.
- Moderation tools for community managers.
- A sophisticated recommendation engine for personalized feeds.

### 2.3 Key Benefits

- **Community Engagement**: Fosters discussion and content discovery through a voting mechanism.
- **Scalability**: Designed to handle a high volume of submissions, votes, and comments.
- **Timeliness**: The ranking algorithm ensures that fresh and popular content is surfaced to users quickly.
- **Extensibility**: The design supports the addition of new communities and content types.

### 2.4 High-Level Architecture Overview

The architecture is built around a core **Ranking Service** that constantly recalculates the score of recent posts based on votes and time decay. Users can submit new links and post comments via a **Post & Comment Service**. Votes are ingested by a high-throughput **Voting Service** that updates vote counts and triggers re-ranking calculations. A **Feed Generation Service** is responsible for creating the main ranked feed that users see. This feed is often heavily cached due to its high read volume. Data is stored across multiple databases: a relational database for user and community data, and a NoSQL database for posts, comments, and votes, which are subject to very high write loads.

---

## 3. System Overview

### 3.1 Business Context

News feed aggregators create vibrant communities around shared interests by allowing users to submit content and collectively determine its visibility through voting. The core value lies in the community-driven curation and the resulting discussions.

### 3.2 System Purpose

The system's purpose is to allow users to post links, vote on submissions, and view a feed of posts ranked by popularity and timeliness. It must also support threaded comments to facilitate discussion.

### 3.3 Success Criteria

- **Engagement**: High number of votes and comments per post.
- **Feed Freshness**: The front page should update frequently with new, popular content.
- **Scalability**: The system must handle "viral" events where a single post receives a massive number of views and votes in a short period.
- **Availability**: 99.9% uptime for core features (viewing, voting, commenting).

### 3.4 Assumptions

- A simple, time-decay-based ranking algorithm is sufficient for the initial version.
- Users are grouped into communities (subreddits), and feeds are generated per-community.

### 3.5 Constraints

- The voting system must be resilient to manipulation (e.g., bots, vote brigades).
- The ranking algorithm can be computationally intensive and must be optimized.

### 3.6 Dependencies

- A database for storing posts, comments, and user data.
- A caching layer for hot posts and generated feeds.

---

## 4. Requirements Analysis

### 4.1 Functional Requirements

- **FR-001**: Users must be able to create posts (links or text) within a specific community.
- **FR-002**: Users must be able to upvote or downvote posts.
- **FR-003**: The system must calculate a score for each post based on its votes and age.
- **FR-004**: The system must display a ranked list of posts for a community.
- **FR-005**: Users must be able to add comments to a post.
- **FR-006**: Comments must be displayed in a threaded (hierarchical) structure.
- **FR-007**: Users must be able to vote on comments.

### 4.2 Non-Functional Requirements

#### 4.2.1 Performance Requirements

- **Feed Load Time**: < 300ms for the main feed.
- **Vote/Comment Latency**: < 200ms.
- **Post Submission Latency**: < 500ms.

#### 4.2.2 Scalability Requirements

- The system must handle millions of users and posts.
- The voting service must handle tens of thousands of votes per second during peak events.
- The comment system must support deeply nested threads with thousands of individual comments.

#### 4.2.3 Availability Requirements

- Core services (viewing feeds, voting) must have 99.9% uptime.

#### 4.2.4 Security Requirements

- The system must have mechanisms to detect and rate-limit suspicious voting patterns.
- User authentication is required for all actions (posting, voting, commenting).

---

## 5. Architecture Design

### 5.1 Architecture Principles

- **Write-Heavy**: The system must be designed to handle a very high volume of writes, especially for votes.
- **Read-Optimized Cache**: While writes are heavy, the main feed is read far more often. The generated feed must be heavily cached.
- **Eventual Consistency**: It is acceptable for vote counts and rankings to have a slight delay in updating. Strict consistency is not required for these features.

### 5.2 Architecture Patterns

- **CQRS (Command Query Responsibility Segregation)**: The write path (casting a vote) can be separated from the read path (viewing a post's score). Votes can be sent to a queue for asynchronous processing, while reads are served from a denormalized view.
- **Lambda Architecture (Simplified)**: A batch layer (Ranking Service) periodically recalculates scores for all recent posts, while a speed layer (Voting Service) provides real-time vote increments.

### 5.3 High-Level Architecture Diagram

```text
+--------------+   (Submit, Comment, Vote)   +-----------------+
| User's Client| --------------------------> |   API Gateway   |
+--------------+                             +-------+---------+
                                                     |
             +---------------------------------------+---------------------------------------+
             | (Writes)                              | (Reads)                               |
             v                                       v                                       v
+-----------------+                          +--------------------+                   +--------------------+
|  Post/Comment Svc |                          | Feed Generation Svc  |                   | Post/Comment Read Svc|
+--------+--------+                          +---------+----------+                   +--------------------+
         |                                              |                                       ^
         v                                              v                                       |
+-----------------+                          +--------------------+                   +--------------------+
| DB (PostgreSQL) |                          |  Feed Cache (Redis)|<------------------+ Ranking Service      |
| (Users, Posts, |                          +--------------------+                   | (Batch Job)        |
|  Comments)      |                                                                   +---------+----------+
+-----------------+                                                                             ^
         ^                                                                                      |
         |                                                                                      |
+--------+--------+                                                                             |
|  Voting Service | <---------------------------------------------------------------------------+
| (High-throughput|
| queue & workers)|
+-----------------+
```

### 5.4 Component Overview

- **API Gateway**: Handles all incoming requests for posting, commenting, and voting.
- **Post/Comment Service**: Manages the storage and retrieval of posts and the comment hierarchy in the primary database.
- **Voting Service**: A high-throughput service designed to ingest a massive number of votes. It accepts vote events, queues them, and has workers that asynchronously update vote counts in a separate data store or directly in the main DB.
- **Ranking Service**: A batch process that runs periodically (e.g., every few minutes). It queries for recent and active posts, applies a ranking algorithm (like the Hacker News algorithm: `Score = (P-1) / (T+2)^G`), and stores the ranked list of post IDs.
- **Feed Generation Service**: This service takes the ranked list of post IDs from the Ranking Service, hydrates them with post details from the Post service, and writes the fully rendered feed into a cache.
- **Feed Cache**: A distributed cache (like Redis) that stores the pre-generated front-page feed. This allows for extremely fast reads.

### 5.5 Technology Stack

- **Backend Services**: **Go** or **Rust** for performance-critical services like Voting. **Python/Django** or **Ruby/Rails** could be used for the less performance-critical Post/Comment service.
- **Primary Database**: **PostgreSQL** is a good choice for storing the structured data of users, posts, and the comment tree (using a recursive CTE).
- **Voting System**: A message queue like **Kafka** to handle the firehose of votes, with consumers that aggregate counts into a fast data store like **Redis**.
- **Feed Cache**: **Redis** or **Memcached**.

### 5.6 Architecture Decision Records (ADRs)

#### 5.6.1 ADR-001: Ranking Algorithm

- **Status**: Accepted
- **Context**: We need an algorithm to rank posts that balances popularity (votes) with freshness (time). A simple vote count would favor old, popular posts forever.
- **Decision**: We will adopt a time-decaying ranking algorithm similar to Hacker News or Reddit's "hot" ranking. The score of a post will be a function of its raw vote count and the time elapsed since submission. The ranking will be recalculated periodically, not on every vote.
- **Consequences**: This makes the ranking logic more complex than a simple sort. It requires a dedicated Ranking Service. The feed will not be perfectly real-time, but this is an acceptable trade-off for scalability and performance.

---

## 6. Detailed Component Design

### 6.1 Component 1: Voting Service

#### 6.1.1 Purpose

To reliably ingest and process a high volume of user votes.

#### 6.1.2 Responsibilities

- Provide a lightweight API endpoint to accept votes.
- Publish each vote event to a message queue (e.g., Kafka).
- Have a pool of consumers that read from the queue.
- Consumers update vote counts in a fast cache (e.g., Redis Hashes `post:votes`) and periodically flush updates to the primary DB.

#### 6.1.3 Internal Design

- The endpoint is a simple, stateless service. The core logic is in the asynchronous consumers. This decouples the user's action from the database write.

### 6.2 Component 2: Ranking Service

#### 6.2.1 Purpose

To calculate the ranked order of posts for the main feed.

#### 6.2.2 Responsibilities

- Run as a scheduled batch job (e.g., every 5 minutes).
- Fetch all posts submitted or voted on within a time window (e.g., the last 48 hours).
- Fetch their current vote counts from the cache/DB.
- Apply the ranking algorithm to calculate a score for each post.
- Store the resulting ranked list of post IDs in a place accessible to the Feed Generation Service.

---

## 7. Data Design

### 7.1 Data Models

- **Post**: `post_id`, `title`, `url`, `submitter_id`, `community_id`, `created_at`.
- **Comment**: `comment_id`, `post_id`, `parent_comment_id`, `author_id`, `text`.
- **Vote**: `vote_id`, `post_or_comment_id`, `user_id`, `direction` (up/down).

### 7.2 Database Design

- **Primary DB (PostgreSQL)**:
  - `posts` table.
  - `comments` table with a self-referencing `parent_comment_id` to model the thread tree.
  - `users` and `communities` tables.
- **Vote & Score Cache (Redis)**:
  - Use Redis Hashes to store vote counts: `HINCRBY post:123:votes up 1`.
  - Use Redis Sorted Sets to store the ranked feeds: `ZADD community:tech:feed <score> <post_id>`.

---

## 8. API Design

### 8.1 API Architecture

- A standard REST API for all user actions.

### 8.2 Core APIs (RESTful)

- `POST /api/v1/posts`: Submit a new post to a community.
- `GET /api/v1/c/{community_name}`: Get the ranked feed for a community.
- `POST /api/v1/posts/{post_id}/vote`: Cast a vote on a post.
- `GET /api/v1/posts/{post_id}/comments`: Get the comment tree for a post.
- `POST /api/v1/comments`: Post a new comment.

---

## 9. Security Design

### 9.1 Security Architecture

The system relies on standard web security practices. All user-generated content must be sanitized before being rendered to prevent XSS attacks.

### 9.2 Authentication & Authorization

- All actions that modify state (posting, commenting, voting) will require a valid user session token (e.g., JWT).
- Reading feeds and comments can be done anonymously.

### 9.3 Vote & Ranking Security

- **Vote Manipulation**: This is a key challenge.
  - **Rate Limiting**: Limit the number of votes per user per unit of time.
  - **IP-based restrictions**: Limit votes from a single IP address to prevent simple bot attacks.
  - **Account Age**: New accounts may have their votes weighted less or be unable to vote for a short period.
  - **Behavioral Analysis**: A background job can analyze voting patterns to detect suspicious "brigades" of users all voting on the same posts.

---

## 10. Scalability & Performance

### 10.1 Performance Requirements

- As defined in section 4.2.1. The system must feel fast and responsive, especially when casting a vote or loading the main feed.

### 10.2 Scalability Strategy

- **Voting Service**: This is the most critical service to scale. Using a message queue like Kafka allows the front-end API to simply accept the vote and return, while the actual processing happens asynchronously. The number of consumers can be scaled up to handle the load.
- **Feed Generation**: The front page feed is the hottest read path. Generating it offline with the Ranking Service and storing it in a cache like Redis is a crucial optimization. This avoids expensive database queries on every page load.
- **Database**: The primary PostgreSQL database can be scaled with read replicas for comment-heavy posts. At a very large scale, sharding by `community_id` or `post_id` might be necessary.

---

## 11. Deployment & Operations

### 11.1 Deployment Architecture

- The system will be deployed as a set of microservices on Kubernetes.
- The Ranking Service will be deployed as a Kubernetes CronJob.

### 11.2 Monitoring & Alerting

- **Key Metrics**: `votes_per_second`, `kafka_queue_lag`, `p99_feed_load_time`, `ranking_job_duration`, `cache_hit_ratio`.
- **Alerting**: Alerts for high Kafka lag (indicating the voting consumers can't keep up), high feed load latency, or failures in the ranking job.

---

## 12. Testing Strategy

### 12.1 Test Types

- **A/B Testing**: The ranking algorithm is a perfect candidate for A/B testing. Different versions of the algorithm can be run in parallel, and the resulting feeds can be served to different segments of users to measure which one drives more engagement (clicks, comments, time on site).
- **Load Testing**: Simulating a "viral post" scenario with a massive influx of votes and comments in a short period is a critical test case.

---

## 13. Risk Analysis

### 13.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Ranking Algorithm is Flawed | High | Medium | A bad algorithm can lead to a stale or uninteresting front page. A/B test all changes to the algorithm. Have a simple, fallback ranking method (e.g., sort by date) if the main one fails. |
| Comment Thread Performance | Medium | High | Displaying a deeply nested comment thread can require many database queries. Use techniques like materialized paths to store the tree structure, and heavily cache the rendered comment threads. |
| "Thundering Herd" on Cache Expiry | High | Medium | When the cached front page feed expires, many requests might hit the Feed Generation Service at once. Use a cache regeneration strategy like "stale-while-revalidate". |

### 13.2 Business Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Community Quality Degrades | High | High | Unmoderated communities can become toxic. While moderation tools are out of scope for V1, the architecture should allow for them to be added easily. |
| Lack of Content | High | Medium | The site needs a critical mass of users to be interesting. Seed the site with content and engage in community building in the early stages. |

---

## 14. Implementation Plan (High-Level)

- **Phase 1**: Build the core User, Post, and Comment services with a PostgreSQL backend.
- **Phase 2**: Implement the high-throughput Voting Service using Kafka and Redis.
- **Phase 3**: Develop the first version of the Ranking Service as a batch job.
- **Phase 4**: Implement the Feed Generation Service and the Feed Cache.
- **Phase 5**: Launch with a few initial communities and begin iterating on the ranking algorithm.

---

## 15. Appendices

### Appendix A: Glossary

- **Ranking Algorithm**: The formula used to calculate a score for a post, determining its position in the feed.
- **Time Decay**: A property of a ranking algorithm where the age of a post negatively affects its score.
- **CQRS (Command Query Responsibility Segregation)**: An architectural pattern that separates read and write operations for a data store.
- **Materialized Path**: A way to represent a tree structure in a relational database by storing the full path to a node in a string.
