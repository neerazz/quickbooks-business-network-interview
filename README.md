# QuickBooks Business Network - System Design Concept

> **Craft Demo for Principal Software Engineer Role**  
> **Candidate**: [Neeraj Kumar Singh](https://www.linkedin.com/in/neerajkumarsinghb)  
> **Date**: December 2025

---

## 🚀 60-Second Overview

This repository contains the system design and architectural proof-of-concept for the **QuickBooks Business Network**, a platform designed to map the global small business economy.

**The Problem**: A network of **1M+ businesses** generates **12M+ events/month**, but their relationships are hidden in disparate data.  
**The Solution**: An event-driven, polyglot architecture that resolves business identities with **99.99% accuracy** and enables sub-second graph exploration.  
**Key Tech Stack**: Java (Spring Boot), Kafka, Neo4j, PostgreSQL, Elasticsearch, Python (ML).

### 🏆 Key Scalability Targets
| Metric | Design Target | Achieved Via |
| :--- | :--- | :--- |
| **Entities** | **1,000,000** | Horizontal Sharding |
| **Relationships** | **50,000,000** | Neo4j (Index-Free Adjacency) |
| **Search Latency** | **<500ms** (p95) | Elasticsearch + Redis L2 Cache |
| **Graph Query** | **<2s** (p95) | Optimized Cypher + Read Replicas |
| **Resolution Accuracy** | **99.99%** | 4-Stage ML Pipeline (Block/Compare) |

---

## 🏗️ Architecture

The system follows a **Microservices** pattern with **Polyglot Persistence**, decoupled by an **Event-Driven** backbone (Kafka).

### High-Level Design
![Architecture Diagram](docs/architecture/high_level_design.png)
*(See `docs/architecture` for detailed mermaid diagrams)*

### Core Services
1.  **Ingest Service**: High-throughput event ingestion (Invoice/Payment) via **Transactional Outbox**.
2.  **Entity Resolution Service**: The "Brain" - dedupes businesses using ML.
3.  **Network Service**: Manages the graph (Neo4j) and relationship weights.
4.  **Search Service**: Hybrid search (Keyword + Vector) using Graph RAG.

---

## 🧠 The "Killer Feature": Entity Resolution

Naive string matching fails at scale ($O(n^2)$). We implemented a **4-Stage Pipeline**:

1.  **Standardize**: `Corp` -> `Corporation`, `TX` -> `Texas`.
2.  **Block**: Reduces comparison space by **99.5%** using Prefix+State keys.
3.  **Compare**: 7-Feature Vector (Levenshtein, Jaro-Winkler, Address Sim).
4.  **Classify**: XGBoost Model (>0.85 Confidence).

---

## 🛡️ Reliability & Trade-offs

### Decisions
*   **Polyglot Persistence**: We chose to maintain 5 data stores (Neo4j, Postgres, ES, Redis, Milvus) to meet the <2s graph traversal SLA, accepting higher operational complexity over the poor performance of a single-DB solution (Postgres CTEs took 3.2s).
*   **Eventual Consistency**: Relationship edges are consistent within **5 seconds** (CAP Theorem AP mode), while Financial Transactions remain ACID (CP mode) in Postgres.

### Patterns Implemented
*   **Transactional Outbox**: Guarantees simple exactly-once event processing.
*   **Circuit Breakers**: Resilience4j prevents cascading failures to the Graph DB.
*   **Dead Letter Queues**: Ensures no event is ever lost, only delayed.

---

## 📂 Repository Structure

```bash
/
├── docs/
│   ├── architecture/        # Mermaid diagrams & System flows
│   ├── deep-dives/          # Entity Resolution logic details
│   └── interviews/          # Presentation Deck & Strategy
├── src/
│   ├── main/java/           # Service definitions (Proto/Interfaces)
│   └── scripts/             # Capacity planning calculators
└── README.md                # This file
```

---

*Designed & Architected by Neeraj Kumar Singh.*
