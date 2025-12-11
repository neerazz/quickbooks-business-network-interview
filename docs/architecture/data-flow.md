# Data Flow Documentation

## Overview

This document describes the data flow patterns in the QuickBooks Business Network platform.

## High-Level Data Flow

```mermaid
flowchart LR
    subgraph Client
        UI[React UI]
    end
    
    subgraph Gateway
        GW[API Gateway]
    end
    
    subgraph Services
        BS[Business]
        NS[Network]
        SS[Search]
        TS[Transaction]
        ER[Entity Resolution]
        GR[Graph RAG]
    end
    
    subgraph DataStores
        PG[(PostgreSQL)]
        NEO[(Neo4j)]
        ES[(Elasticsearch)]
        REDIS[(Redis)]
    end
    
    subgraph Messaging
        KAFKA[Kafka]
    end
    
    UI --> GW
    GW --> BS & NS & SS & TS & ER & GR
    
    BS --> PG & NEO
    NS --> NEO & REDIS
    SS --> ES
    TS --> PG & KAFKA
    ER --> PG & KAFKA
    GR --> NEO & ES
    
    KAFKA --> ER
```

## Flow 1: Business Creation

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as Gateway
    participant BS as Business Service
    participant PG as PostgreSQL
    participant NEO as Neo4j
    participant ES as Elasticsearch
    participant KAFKA as Kafka
    
    UI->>GW: POST /businesses
    GW->>BS: Forward request
    BS->>PG: Insert business record
    BS->>NEO: Create business node
    BS->>KAFKA: Publish business.created
    BS-->>GW: 201 Created
    GW-->>UI: Business details
    
    Note over KAFKA,ES: Async Processing
    KAFKA->>ES: Index business (via consumer)
```

## Flow 2: Transaction Recording

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as Gateway
    participant TS as Transaction Service
    participant PG as PostgreSQL
    participant KAFKA as Kafka
    participant NS as Network Service
    participant NEO as Neo4j
    
    UI->>GW: POST /transactions
    GW->>TS: Forward request
    TS->>PG: Insert transaction
    TS->>KAFKA: Publish transaction.created
    TS-->>GW: 201 Created
    GW-->>UI: Transaction details
    
    Note over KAFKA,NEO: Async Weight Update
    KAFKA->>NS: Consume event
    NS->>NEO: Update relationship weight
```

## Flow 3: Network Visualization

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as Gateway
    participant NS as Network Service
    participant REDIS as Redis Cache
    participant NEO as Neo4j
    
    UI->>GW: GET /networks/{id}?depth=2
    GW->>NS: Forward request
    
    alt Cache Hit
        NS->>REDIS: Check cache
        REDIS-->>NS: Return cached data
    else Cache Miss
        NS->>NEO: MATCH path query
        NEO-->>NS: Return graph data
        NS->>REDIS: Store in cache (TTL 10min)
    end
    
    NS-->>GW: Network map
    GW-->>UI: Nodes and edges
```

## Flow 4: Entity Resolution Pipeline

```mermaid
sequenceDiagram
    participant IF as Ingest Flow
    participant KAFKA as Kafka
    participant ER as Entity Resolution
    participant AI as AI Service
    participant PG as PostgreSQL
    participant RF as Resolve Flow
    
    IF->>KAFKA: Publish entity (entity.pending)
    
    Note over ER: Standardize Stage
    KAFKA->>ER: Consume entity
    ER->>ER: Normalize name, address
    
    Note over ER: Block Stage
    ER->>ER: Generate blocking keys
    ER->>PG: Find candidates in block
    
    Note over ER: Compare Stage
    ER->>AI: Generate feature vectors
    AI-->>ER: Similarity scores
    
    Note over ER: Classify Stage
    ER->>AI: Apply ML classifier
    AI-->>ER: Match/Non-match/Review
    
    alt High Confidence Match (>=0.85)
        ER->>KAFKA: Publish merge event
        KAFKA->>RF: Process merge
        RF->>PG: Merge entities
    else Low Confidence (0.50-0.85)
        ER->>PG: Flag for review
    else Non-Match (<0.50)
        ER->>PG: Keep separate
    end
```

## Flow 5: Graph RAG Query

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as Gateway
    participant GR as Graph RAG
    participant NEO as Neo4j
    participant VDB as Milvus
    participant AI as AI Service
    
    UI->>GW: POST /graph-rag/query
    GW->>GR: Forward query
    
    Note over GR: Query Parsing
    GR->>AI: Parse natural language
    AI-->>GR: Entities and intent
    
    Note over GR: Context Retrieval
    GR->>NEO: Graph traversal
    NEO-->>GR: Related nodes
    GR->>VDB: Vector similarity
    VDB-->>GR: Similar entities
    
    Note over GR: Response Generation
    GR->>AI: Generate response
    AI-->>GR: Natural language answer
    
    GR-->>GW: Response with sources
    GW-->>UI: Display answer
```

## Flow 6: Search

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as Gateway
    participant SS as Search Service
    participant ES as Elasticsearch
    
    UI->>GW: GET /search?query=acme
    GW->>SS: Forward request
    
    SS->>ES: Multi-match query
    Note over ES: Search fields:<br/>name, address, category
    ES-->>SS: Scored results
    
    SS->>SS: Apply business rules
    SS-->>GW: Paginated results
    GW-->>UI: Display results
```

## Data Consistency Patterns

### Eventually Consistent Flows

- Search indexing (async via Kafka)
- Relationship weight updates (async via Kafka)
- Entity resolution processing (async via Kafka)

### Strongly Consistent Flows

- Business creation (sync to PostgreSQL + Neo4j)
- Transaction recording (sync to PostgreSQL)
- User feedback submission (sync to PostgreSQL)

## Caching Strategy

| Data Type | Cache Location | TTL |
|-----------|----------------|-----|
| Network maps | Redis | 10 minutes |
| Search results | Redis | 5 minutes |
| Business details | Redis | 15 minutes |
| SLA metrics | In-memory | 1 minute |

## Event Topics

| Topic | Producer | Consumer(s) |
|-------|----------|-------------|
| business.created | Business Service | Search, Entity Resolution |
| transaction.created | Transaction Service | Network Service |
| entity.pending | Ingest Flow | Entity Resolution |
| entity.resolved | Entity Resolution | Resolve Flow |
| feedback.received | UI | Training Consumer |
