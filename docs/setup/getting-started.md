# Getting Started Guide

## Prerequisites

Before you begin, ensure you have the following installed:

- **JDK 17+** - [Download](https://adoptium.net/)
- **Maven 3.8+** - [Download](https://maven.apache.org/download.cgi)
- **Node.js 18+** - [Download](https://nodejs.org/)
- **Docker Desktop** - [Download](https://www.docker.com/products/docker-desktop)
- **Git** - [Download](https://git-scm.com/)

### Verify Installation

```bash
java -version   # Should show 17+
mvn -version    # Should show 3.8+
node -v         # Should show 18+
docker --version
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/quickbooks-business-network.git
cd quickbooks-business-network
```

### 2. Start Infrastructure Services

```bash
# Start all backing services (PostgreSQL, Neo4j, Elasticsearch, Kafka, Redis, Milvus)
docker-compose up -d

# Wait for services to be ready (about 60 seconds)
docker-compose ps  # All should show "healthy" or "running"
```

### 3. Build the Backend

```bash
# Build all services
mvn clean install -DskipTests

# Or build a specific service
mvn -pl src/services/business clean package
```

### 4. Run Backend Services

Start services in separate terminals or use your IDE:

```bash
# From the project root
java -jar src/services/gateway/target/gateway-service-1.0.0-SNAPSHOT.jar &
java -jar src/services/business/target/business-service-1.0.0-SNAPSHOT.jar &
java -jar src/services/network/target/network-service-1.0.0-SNAPSHOT.jar &
java -jar src/services/search/target/search-service-1.0.0-SNAPSHOT.jar &
java -jar src/services/transaction/target/transaction-service-1.0.0-SNAPSHOT.jar &
java -jar src/services/entity-resolution/target/entity-resolution-service-1.0.0-SNAPSHOT.jar &
# ... start other services as needed
```

### 5. Start the Frontend

```bash
cd src/frontend/react-ui
npm install
npm run dev
```

### 7. Seed Test Data (Optional)

To populate the system with test data and verify the complete pipeline flow:

```powershell
# Windows (PowerShell)
.\scripts\seed-data.ps1

# With options
.\scripts\seed-data.ps1 -EventCount 50 -WaitTimeout 180
```

```bash
# Linux/macOS
./scripts/seed-data.sh

# With options
./scripts/seed-data.sh --event-count 50 --wait-timeout 180
```

This script will:

1. Check all infrastructure services are healthy
2. Verify required microservices are running
3. Seed test data via the Ingest API
4. Wait for events to flow through the pipeline
5. Verify data appears in PostgreSQL, Neo4j, and Elasticsearch

### 8. Access the Application

- **Frontend UI**: <http://localhost:3000>
- **API Gateway**: <http://localhost:8080>
- **Neo4j Browser**: <http://localhost:7474>
- **Elasticsearch**: <http://localhost:9200>

## Development Workflow

### Running Tests

```bash
# Run all tests
mvn test

# Run tests for a specific service
mvn -pl src/services/network test

# Run property tests with 100 iterations
mvn test -Djqwik.tries=100

# Run frontend tests
cd src/frontend/react-ui && npm test
```

### Running with Hot Reload

For backend development with Spring Boot DevTools:

```bash
mvn -pl src/services/business spring-boot:run
```

For frontend with Vite:

```bash
cd src/frontend/react-ui
npm run dev
```

## Environment Configuration

Copy the example environment file:

```bash
cp .env.example .env
```

Configure the following variables:

```bash
# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=quickbooks
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# Neo4j
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password

# Elasticsearch
ELASTICSEARCH_HOST=localhost
ELASTICSEARCH_PORT=9200

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# AI Service
AI_SERVICE_URL=http://localhost:8000
```

## Troubleshooting

### Docker Services Won't Start

```bash
# Check memory allocation (increase to at least 8GB)
docker system info | grep Memory

# Clear old containers
docker-compose down -v
docker system prune -f

# Restart services
docker-compose up -d
```

### Elasticsearch Startup Issues

```bash
# Increase virtual memory for Elasticsearch
# On Linux/Mac:
sudo sysctl -w vm.max_map_count=262144

# On Windows (in PowerShell as Admin):
wsl -d docker-desktop
sysctl -w vm.max_map_count=262144
```

### Port Conflicts

Check for conflicting processes:

```bash
# Linux/Mac
lsof -i :8080

# Windows
netstat -ano | findstr :8080
```

### Build Failures

```bash
# Clear Maven cache
mvn dependency:purge-local-repository

# Rebuild with fresh dependencies
mvn clean install -U
```

## Next Steps

1. **Explore the API**: Check the [API Documentation](../api/openapi-v1.yaml)
2. **Understand the Architecture**: Read the [Architecture Overview](../architecture/overview.md)
3. **Learn the User Flows**: See the [User Guide](../user-guide/usage.md)
4. **Contribute**: Review the [Contributing Guidelines](../../CONTRIBUTING.md)
