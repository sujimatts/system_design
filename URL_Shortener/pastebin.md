# System Design: Pastebin (Document Store)

A Pastebin-like service allows users to store plain text or source code and retrieve it via a unique, short URL (e.g., `https://pastebin.com/aB34eFg`).

---

## 1. System Requirements & Scope

### Functional Requirements
* **Create Paste**: Upload text/code (up to 10MB) and get a unique short URL.
* **Retrieve Paste**: View the paste using the unique short URL.
* **Expiration (TTL)**: Pastes automatically expire and delete after a user-specified time.

### Non-Functional Requirements
* **High Availability**: 99.99% uptime so users can always access their pastes.
* **Low Latency**: Retrieving a paste should be super fast (sub-100ms).
* **Scalability**: Handle heavy read/write traffic (Read-heavy, 10:1 ratio).

---

## 2. Thought Process & Core Architecture (Why and How?)

This section explains the design decisions, trade-offs, and technology choices for each component.

### A. How We Generate Unique Keys (The Key Problem & Key Generation Service)
* **The Goal**: Every paste needs a short, unique URL key (e.g., `aB34eFg`).
* **The Problem with Runtime Generation**: 
  * If the Application Server generates a random key at runtime, it must query the database to check if that key is already in use (collision check). 
  * As the database grows, collisions increase, requiring multiple database roundtrips for a single paste creation, slowing down writes.
* **The Solution (Key Generation Service - KGS)**:
  * **What is KGS?** KGS is a dedicated microservice that pre-generates millions of unique strings (e.g., 7-character Base62 keys) *offline* and stores them in a database. At runtime, when an App Server needs a key, it simply requests one from KGS, which hands it out from memory instantly ($O(1)$ lookup) with **zero collision checks at runtime**.
  * **KGS DB Tech Stack**: KGS stores the pre-generated keys in a **Relational SQL Database (like MySQL or PostgreSQL)**.
  * **Why SQL for KGS?**: We need **ACID transaction guarantees and row-level locking**. When multiple Application Servers concurrently ask KGS for keys, the database must guarantee that the same key is never handed out to two different servers. SQL database transactions allow us to read a key and mark it as `used` in a single, safe atomic operation.

### B. The Application Server Stack
* **Role**: Stateless backend API layer that handles HTTP requests, authentication, and coordinate reads/writes.
* **Tech Stack**: Built using highly concurrent backend stacks like **Golang** (efficient concurrency via Goroutines), **Node.js** (non-blocking I/O), or **Java/Spring Boot**.
* **Why Stateless?**: It allows us to easily scale horizontally by adding more servers behind a Load Balancer.

### C. Large Payload Storage: Why AWS S3?
* **Role**: Stores the raw text files.
* **Tech Stack**: **Object Storage** (e.g., AWS S3, Google Cloud Storage).
* **Why S3?**:
  * **Database Bloat**: Storing raw text/code payloads (up to 10MB) directly in a database row is highly inefficient. It bloats database disk usage, degrades indexes, and slows down queries.
  * **Scale and Cost**: S3 is designed to store massive, unstructured files cheaply and reliably, offering 99.999999999% durability and scaling automatically. We store the raw paste in S3 under `/pastes/{uniqueKey}`.

### D. The Metadata Database: Why and What?
* **Role**: Serves as the "Card Catalog" / Index. It stores details *about* the paste (who owns it, when it expires, and the path to where the actual file is stored in S3).
* **Tech Stack**: **NoSQL Key-Value/Document Store** (e.g., AWS DynamoDB or MongoDB).
* **Why NoSQL?**: 
  * Paste retrieval is a simple key-value lookup: *"Give me metadata for key `aB34eFg`"*. 
  * There are no complex relationships or SQL Joins. NoSQL databases are optimized for simple lookups, providing sub-millisecond query responses, and scale horizontally by partitioning keys.

### E. Caching Layer: Why Redis?
* **Role**: Caches hot (popular) paste payloads.
* **Tech Stack**: **Redis** (in-memory key-value store).
* **Why?**: The system is read-heavy. Caching popular paste payloads in memory allows us to serve reads in under 5ms, avoiding hitting S3 or the Metadata DB repeatedly during traffic spikes.

---

## 3. High-Level Architecture Diagram

Here is how all the components interact:

```
                      ┌────────────────────────┐
                      │  Client Browser / API  │
                      └───────────┬────────────┘
                                  │
                                  ▼
                      ┌────────────────────────┐
                      │     Load Balancer      │
                      └───────────┬────────────┘
                                  │
                                  ▼
                      ┌────────────────────────┐       ┌───────────────────────┐
                      │  Application Servers   │◄─────►│ Key Generation (KGS)  │
                      └───────┬───┬───┬────────┘       └───────────┬───────────┘
                              │   │   │                            │
             ┌────────────────┘   │   └────────────────┐           │
             ▼                    ▼                    ▼           ▼
     ┌───────────────┐    ┌───────────────┐    ┌───────────────┐ ┌─────────────┐
     │  Redis Cache  │    │  Metadata DB  │    │ Object Storage│ │   KGS DB    │
     │ (Hot Pastes)  │    │  (NoSQL/SQL)  │    │   (AWS S3)    │ │ (Relational)│
     └───────────────┘    └───────▲───────┘    └───────▲───────┘ └─────────────┘
                                  │                    │
                                  └────────┬───────────┘
                                           │
                                  ┌────────┴───────┐
                                  │ Cleanup Daemon │
                                  └────────────────┘
```

---

## 4. Metadata Database Schema

#### Schema: `PasteMetadata`
| Field Name | Type | Description |
| :--- | :--- | :--- |
| **ShortKey (PK)** | `String` | Unique 7-character key (e.g., `aB34eFg`) |
| **StoragePath** | `String` | S3 URI (e.g., `s3://pastebin-blobs/pastes/aB34eFg`) |
| **ExpirationTime**| `Timestamp` | Expiration time (null = never) |
| **CreatedAt** | `Timestamp` | Creation time |

---

## 5. System Workflows (Step-by-Step)

### A. The Simplified Workflow
* **Write**: Get key from KGS $\rightarrow$ Save text file to S3 $\rightarrow$ Store S3 link and expiration in Metadata DB $\rightarrow$ Return short URL.
* **Read**: Check Redis $\rightarrow$ if not found, check Metadata DB to get S3 link $\rightarrow$ Fetch text file from S3 $\rightarrow$ Cache it $\rightarrow$ Show to user.

---

### B. Detailed Write Workflow (Uploading a Paste)
1. **Client** uploads paste text/code and sets optional expiration.
2. **Load Balancer** routes the request to an available **App Server**.
3. The **App Server** queries the **Key Generation Service (KGS)** to fetch a pre-allocated unused short key (e.g., `x7K2p9w`).
4. The **App Server** compresses the text payload and writes it to **Object Storage (S3)** at `s3://pastebin-blobs/pastes/x7K2p9w`.
5. The **App Server** writes a metadata record in the **Metadata DB** mapping key `x7K2p9w` to `s3://pastebin-blobs/pastes/x7K2p9w`, along with expiration timestamps.
6. The **App Server** returns the short URL `https://pastebin.com/x7K2p9w` to the user.

---

### C. Detailed Read Workflow (Accessing a Paste)
1. **Client** requests `https://pastebin.com/x7K2p9w`.
2. The request is routed to an **App Server**.
3. The **App Server** checks **Redis Cache** for key `x7K2p9w`.
   * **Cache Hit**: Return text payload instantly.
   * **Cache Miss**:
     1. App Server queries **Metadata DB** using `x7K2p9w` as the primary key.
     2. If key is not found or is expired, return `404 Not Found`.
     3. If valid, read `StoragePath` (S3 URL) from metadata.
     4. Fetch raw text payload from **Object Storage (S3)**, decompress it.
     5. Write the payload to **Redis Cache** (with TTL).
     6. Return text payload to client.

---

## 6. Key Concepts & Interview Cheat Sheet

* **TTL (Time To Live / Data Expiration)**:
  * **Passive Cleanup**: When a user visits a paste, check if it's expired. If yes, delete it from S3/Metadata DB and return a 404.
  * **Active Cleanup**: A daily background cron job scans the Metadata DB for expired records, deletes them from the DB, and deletes the corresponding file from S3.
* **Write/Read Path Decoupling**: For extremely high write volumes, write tasks can be pushed to a Message Queue (like Kafka) so that files are uploaded asynchronously without blocking the user.
* **Security**: Rate limit requests per IP to prevent spamming/filling up S3. Run automated scanners on S3 uploads to detect malware.
