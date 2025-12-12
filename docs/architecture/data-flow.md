# QuickBooks Business Network - Visual Service Architecture

## High-Level Service Connections Diagram

[Miro](https://miro.com/app/board/uXjVGc_CdZ4=/?share_link_id=443014834206)

```mermaid
graph TB
    subgraph "Client Layer"
        UI[React UI :3000]
    end

    subgraph "Gateway Layer"
        GW[API Gateway :8080]
        AUTH[Auth Service :8092]
    end

    subgraph "Business Logic Layer"
        BS[Business Service :8081]
        TS[Transaction Service :8084]
        PS[Payment Service :8084]
        NS[Network Service :8082<br/>ORCHESTRATOR]
        ER[Entity Resolution :8085]
        SS[Search Service :8083]
        RAG[Graph RAG :8086]
        TC[Training Consumer :8089]
        DG[Data Generator :8090]
        SLA[SLA Metrics :8091]
    end

    subgraph "AI Layer"
        AI[AI Service :8000]
    end

    subgraph "Data Layer"
        PG[(PostgreSQL :5432)]
        NEO[(Neo4j :7687)]
        ES[(Elasticsearch :9200)]
        KAFKA[Kafka :9092]
        REDIS[(Redis :6379)]
        VECTOR[(Vector DB :19530)]
    end

    %% Client to Gateway
    UI -->|HTTPS| GW

    %% Gateway to Services
    GW -->|Auth Check| AUTH
    GW -->|Route Requests| BS
    GW -->|Route Requests| NS
    GW -->|Route Requests| SS
    GW -->|Route Requests| TS
    GW -->|Route Requests| RAG

    %% Business Service Connections
    BS -->|Read/Write| PG
    BS -->|Produce business.events| KAFKA
    BS -->|Cache| REDIS

    %% Transaction Service Connections
    TS -->|Store Transactions| PG
    TS -->|Produce transaction.events| KAFKA

    %% Network Service Connections (ORCHESTRATOR)
    NS -->|Graph Queries| NEO
    NS -->|Cache| REDIS
    NS -->|Consume business/transaction events| KAFKA
    NS -->|Enrich Data| BS
    NS -->|HTTP: Trigger Resolution| ER
    NS -->|Produce search.index.requests| KAFKA

    %% Entity Resolution Connections (KEY: Calls Search Service)
    ER -->|HTTP: Fuzzy Search Matching| SS
    ER -->|ML Inference| AI
    ER -->|Semantic Search| VECTOR

    %% Search Service Connections
    SS -->|Fuzzy Search Queries| ES
    SS -->|Graph Context| NEO
    SS -->|Cache Results| REDIS
    SS -->|Consume search.index.requests| KAFKA
    SS -->|Index New Businesses| ES

    %% Payment Service Connections
    PS -->|Consume transaction.events| KAFKA
    PS -->|Update Status| PG

    %% Graph RAG Connections
    RAG -->|Embeddings| AI
    RAG -->|Vector Search| VECTOR
    RAG -->|Subgraph Query| NEO
    RAG -->|LLM Generate| AI

    %% Training Consumer Connections
    TC -->|Consume Events| KAFKA
    TC -->|Store Training Data| PG
    TC -->|Train Models| AI

    %% AI Service Connections
    AI -->|Update Embeddings| VECTOR

    %% SLA Metrics Connections
    SLA -.->|Scrape Metrics| BS
    SLA -.->|Scrape Metrics| NS
    SLA -.->|Scrape Metrics| SS
    SLA -.->|Scrape Metrics| TS
    SLA -.->|Scrape Metrics| ER
    SLA -->|Store Metrics| REDIS
    SLA -->|Monitor Lag| KAFKA

    %% Data Generator Connections
    DG -->|Generate Test Data| KAFKA
    DG -.->|Verify| PG
    DG -.->|Verify| NEO
    DG -.->|Verify| ES

    %% Auth Service
    AUTH -->|Session Cache| REDIS

    style UI fill:#e1f5ff
    style GW fill:#fff4e1
    style AUTH fill:#fff4e1
    style NS fill:#ffcccc
    style ER fill:#ffe6cc
    style AI fill:#ffe1f5
    style PG fill:#e8f5e9
    style NEO fill:#e8f5e9
    style ES fill:#e8f5e9
    style KAFKA fill:#fff3e0
    style REDIS fill:#e8f5e9
    style VECTOR fill:#e8f5e9
```

## Critical Data Flow: Business Creation → Entity Resolution

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as API Gateway
    participant AUTH as Auth Service
    participant BS as Business Service
    participant PG as PostgreSQL
    participant KAFKA as Kafka
    participant NS as Network Service
    participant ER as Entity Resolution
    participant SS as Search Service
    participant ES as Elasticsearch
    participant AI as AI Service
    participant VDB as Vector DB
    participant NEO as Neo4j

    UI->>GW: POST /businesses {data}
    GW->>AUTH: Validate token
    AUTH-->>GW: Valid ✓
    GW->>BS: Create business
    BS->>PG: INSERT business
    PG-->>BS: business_id=12345
    BS->>KAFKA: Publish BusinessCreated (business.events)
    BS-->>GW: Success
    GW-->>UI: 201 Created

    Note over KAFKA,NS: Async Orchestration Begins

    KAFKA->>NS: Consume event (Group: network-service)
    
    Note over NS,ER: Network Service orchestrates Entity Resolution
    
    NS->>ER: HTTP POST /resolve {business_id: 12345}
    
    Note over ER,SS: Entity Resolution 4-Stage Pipeline
    
    ER->>AI: Stage 1: Standardize name
    AI-->>ER: "ACME CORPORATION"
    
    ER->>SS: Stage 2: HTTP GET /search?fuzzy=true&q=ACME CORPORATION
    SS->>ES: Fuzzy query (n-gram + phonetic)
    ES-->>SS: Top 10 candidates with scores
    SS-->>ER: Candidate list [{id, name, score}, ...]
    
    ER->>AI: Stage 3: Generate embeddings
    AI-->>ER: 384-dim vector
    ER->>VDB: Semantic similarity search
    VDB-->>ER: Ranked candidates
    
    ER->>AI: Stage 4: ML predict (XGBoost)
    AI-->>ER: {match_prob: 0.92, target_id: 54321}
    
    ER-->>NS: Response {is_duplicate: true, matched_id: 54321, confidence: 0.92}
    
    alt Duplicate Found (confidence > 0.85)
        NS->>NEO: UPDATE existing node (link duplicate)
        NEO-->>NS: Updated
        Note over NS: No new node created, no indexing needed
    else New Business
        NS->>NEO: CREATE new node (Business:12345)
        NEO-->>NS: Created
        NS->>KAFKA: Publish IndexBusiness (search.index.requests)
        
        KAFKA->>SS: Consume (Group: search-service)
        SS->>ES: POST /businesses/_doc/12345
        ES-->>SS: Indexed
    end
```

## Transaction-Based Auto Node Creation Flow

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as API Gateway
    participant TS as Transaction Service
    participant PG as Payment Gateway
    participant PGDB as PostgreSQL
    participant KAFKA as Kafka
    participant NS as Network Service
    participant ER as Entity Resolution
    participant SS as Search Service
    participant AI as AI Service
    participant NEO as Neo4j
    participant ES as Elasticsearch

    UI->>GW: POST /transactions {to: "New Supplier", amount: 15000}
    GW->>TS: Record transaction
    TS->>PG: Process payment
    PG-->>TS: Approved ✓
    TS->>PGDB: INSERT transaction
    PGDB-->>TS: txn_id=9999
    TS->>KAFKA: Publish TransactionApproved (transaction.events)
    TS-->>UI: 201 Created

    Note over KAFKA,NS: Auto Node Creation Workflow

    KAFKA->>NS: Consume event (Group: network-service)
    NS->>NEO: Check if "New Supplier" exists
    NEO-->>NS: Not found
    
    NS->>ER: HTTP POST /resolve {name: "New Supplier", ...}
    ER->>AI: Standardize
    AI-->>ER: "NEW SUPPLIER CORPORATION"
    ER->>SS: HTTP GET /search?fuzzy=true
    SS->>ES: Fuzzy search
    ES-->>SS: No similar businesses
    SS-->>ER: Empty candidates []
    ER->>AI: ML predict
    AI-->>ER: {is_duplicate: false, confidence: 0.10}
    ER-->>NS: No match found
    
    NS->>NEO: CREATE (Business:67890 {name: "New Supplier", auto_created: true})
    NEO-->>NS: Created
    NS->>NEO: CREATE relationship (12345)-[:TRANSACTS_WITH]->(67890)
    NEO-->>NS: Relationship created
    
    NS->>KAFKA: Publish IndexBusiness (search.index.requests)
    KAFKA->>SS: Consume
    SS->>ES: Index new business
    ES-->>SS: Indexed
```

## Event Flow: Transaction Processing

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as API Gateway
    participant TS as Transaction Service
    participant PG as PostgreSQL
    participant KAFKA as Kafka
    participant PS as Payment Service
    participant NS as Network Service
    participant NEO as Neo4j
    participant SS as Search Service
    participant ES as Elasticsearch
    participant TC as Training Consumer

    UI->>GW: POST /transactions
    GW->>TS: Record transaction
    TS->>PG: INSERT transaction
    PG-->>TS: txn_id=9999
    TS->>KAFKA: TransactionRecorded event
    TS-->>UI: 201 Created

    par Payment Processing
        KAFKA->>PS: Consume event (Group: payment-processing)
        PS->>PS: Process payment
        PS->>PG: UPDATE settlement_status
    and Network Update
        KAFKA->>NS: Consume event (Group: network-service)
        NS->>NEO: UPDATE relationship weight
        NEO-->>NS: Updated
        NS->>SS: HTTP POST /reindex (if significant change)
        SS->>ES: Update rankings
        ES-->>SS: Updated
    and Search Indexing
        KAFKA->>SS: Consume event (Group: search-service)
        SS->>ES: Index transaction
        ES-->>SS: Indexed
    and Training Pipeline
        KAFKA->>TC: Consume event (Group: training-pipeline)
        TC->>PG: INSERT training_data
    end
```

## Read Path: Network Map Query

```mermaid
sequenceDiagram
    participant UI as React UI
    participant GW as API Gateway
    participant NS as Network Service
    participant REDIS as Redis Cache
    participant NEO as Neo4j
    participant BS as Business Service
    participant PG as PostgreSQL

    UI->>GW: GET /network/12345?depth=2
    GW->>NS: Get network map
    NS->>REDIS: GET cache
    
    alt Cache Hit
        REDIS-->>NS: Cached graph
        NS-->>UI: Return graph (5ms)
    else Cache Miss
        REDIS-->>NS: null
        NS->>NEO: Cypher query (depth=2)
        NEO-->>NS: Graph structure
        NS->>BS: Batch get business details
        BS->>PG: SELECT * WHERE id IN (...)
        PG-->>BS: Business data
        BS-->>NS: Enriched data
        NS->>REDIS: SET cache (TTL=30min)
        NS-->>UI: Return graph (280ms)
    end
```

## Connection Matrix by Protocol

```mermaid
graph LR
    subgraph "HTTP/REST Connections"
        GW[API Gateway]
        BS[Business Service]
        NS[Network Service<br/>ORCHESTRATOR]
        SS[Search Service]
        TS[Transaction Service]
        ER[Entity Resolution]
        RAG[Graph RAG]
        AI[AI Service]
        
        GW -.->|HTTP| BS
        GW -.->|HTTP| NS
        GW -.->|HTTP| SS
        NS -.->|HTTP| BS
        NS -.->|HTTP| ER
        NS -.->|HTTP| SS
        ER -.->|HTTP| AI
        RAG -.->|HTTP| AI
    end

    subgraph "Kafka Event Bus"
        K[Kafka :9092]
        BS -->|Produce| K
        TS -->|Produce| K
        NS -->|Consume + Produce| K
        SS -->|Consume| K
        PS -->|Consume| K
        TC -->|Consume| K
    end

    subgraph "Database Connections (JDBC/Bolt/Native)"
        PG[(PostgreSQL)]
        NEO[(Neo4j)]
        ES[(Elasticsearch)]
        REDIS[(Redis)]
        VDB[(Milvus)]
        
        BS -->|JDBC| PG
        TS -->|JDBC| PG
        TC -->|JDBC| PG
        NS -->|Bolt| NEO
        SS -->|Bolt| NEO
        RAG -->|Bolt| NEO
        SS -->|HTTP| ES
        ER -->|HTTP| ES
        NS -->|Native| REDIS
        SS -->|Native| REDIS
        AUTH -->|Native| REDIS
        SLA -->|Native| REDIS
        ER -->|gRPC| VDB
        RAG -->|gRPC| VDB
        AI -->|gRPC| VDB
    end
```

---

## Architecture Pattern: Orchestration Model

### Why This Architecture?

**Previous Architecture Issues:**
- Entity Resolution consuming directly from Kafka created tight coupling
- No coordination between graph updates and entity matching
- Search Service manually polling for updates
- Difficult to ensure consistency across Neo4j, Elasticsearch, and ML models

**Improved Architecture Benefits:**

1. **Network Service as Orchestrator**
   - Single point of control for graph operations
   - Coordinates entity resolution synchronously
   - Ensures Neo4j is updated before entity matching begins
   - Controls Kafka event flow

2. **Search Service Real-time Indexing**
   - Consumes from Kafka for immediate indexing
   - No polling required
   - Parallel processing with Network Service
   - Independent scaling

3. **Entity Resolution as Stateless Service**
   - HTTP-based (synchronous) invocation
   - No Kafka consumer lag to manage
   - Easier to scale horizontally
   - Network Service controls when/how it's called

### Data Flow Summary

```
Business Created Event Flow:
1. Business Service → Kafka (business.events)
2. Network Service consumes event
   ├─→ Creates Neo4j node
   ├─→ Calls Entity Resolution (HTTP)
   │   └─→ Entity Resolution: Elasticsearch + Milvus + AI Service
   ├─→ If match found: Creates relationship in Neo4j
   ├─→ Publishes match event to Kafka (entity.matches)
   └─→ Invalidates Redis cache
3. Search Service consumes event (parallel)
   └─→ Indexes in Elasticsearch

Transaction Event Flow:
1. Transaction Service → Kafka (transaction.events)
2. Network Service consumes event
   ├─→ Updates relationship weight in Neo4j
   ├─→ If significant change: Calls Search Service (HTTP)
   └─→ Invalidates Redis cache
3. Search Service consumes event (parallel)
   └─→ Indexes transaction in Elasticsearch
4. Payment Service consumes event (parallel)
   └─→ Processes payment
```

### Consumer Groups

| Consumer Group | Topics | Services | Purpose |
|----------------|--------|----------|---------|
| network-service | business.events, transaction.events | Network Service | Orchestration & graph updates |
| search-service | business.events, transaction.events | Search Service | Real-time indexing |
| payment-processing | transaction.events | Payment Service | Payment processing |
| training-pipeline | business.events, entity.matches, user.feedback | Training Consumer | ML training data |

**Total Kafka Consumer Groups**: 4 (down from 5 in previous architecture)
**Entity Resolution**: HTTP-based (no consumer group needed)