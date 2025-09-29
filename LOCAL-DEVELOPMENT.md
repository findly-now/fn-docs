# Local Development Guide - Findly Now Platform

This comprehensive guide helps you run the complete Findly Now platform locally with all microservices and infrastructure. Perfect for development, testing, and understanding the full system.

## 🎯 Quick Start

**Get the complete environment running in under 5 minutes:**

```bash
# 1. Start everything
make local-all-up

# 2. Check status
make local-all-status

# 3. Generate test data
./scripts/generate-test-data.sh --full

# 4. Open the dashboard
make open-health
```

**Access Points:**
- **Health Dashboard**: http://localhost:8082
- **Kafka UI**: http://localhost:8081
- **Posts API**: http://localhost:8080/api/v1
- **Notifications**: http://localhost:4000
- **Media AI API**: http://localhost:8000/docs
- **Matcher API**: http://localhost:3000/api/v1

## 📋 Prerequisites

### System Requirements

| Requirement | Minimum | Recommended | Check Command |
|-------------|---------|-------------|---------------|
| **RAM** | 6GB available | 8GB+ available | `free -h` (Linux) / Activity Monitor (macOS) |
| **Storage** | 5GB free | 10GB+ free | `df -h` |
| **Docker** | 20.10+ | Latest | `docker --version` |
| **Docker Compose** | v2.0+ | Latest | `docker compose version` |

### Required Software

```bash
# Check all requirements
make requirements

# Install missing tools (macOS with Homebrew)
brew install docker docker-compose curl make

# Install missing tools (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install docker.io docker-compose curl make
```

## 🏗️ Architecture Overview

The local environment replicates the complete production architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Local Environment                        │
├─────────────────────────────────────────────────────────────────┤
│  🖥️  Web Interfaces                                             │
│  • Health Dashboard (8082)  • Kafka UI (8081)                  │
│  • MinIO Console (9001)     • Service APIs                     │
├─────────────────────────────────────────────────────────────────┤
│  🚀 Microservices                                               │
│  • fn-posts (8080)          • fn-notifications (4000)          │
│  • fn-media-ai (8000)       • fn-matcher (3000)                │
├─────────────────────────────────────────────────────────────────┤
│  🏗️  Infrastructure                                             │
│  • PostgreSQL (5432)        • Kafka (9092)                     │
│  • MinIO S3 (9000)          • Redis (6379)                     │
└─────────────────────────────────────────────────────────────────┘
```

### Service Dependencies

```mermaid
graph TB
    subgraph "Infrastructure"
        PG[PostgreSQL]
        K[Kafka]
        M[MinIO]
        R[Redis]
    end

    subgraph "Microservices"
        P[fn-posts]
        N[fn-notifications]
        AI[fn-media-ai]
        MT[fn-matcher]
    end

    subgraph "Events"
        PC[post.created]
        PE[post.enhanced]
        PM[post.matched]
        NS[notification.sent]
    end

    P --> PC
    PC --> AI
    AI --> PE
    PE --> MT
    MT --> PM
    PM --> N
    N --> NS

    P -.-> PG
    N -.-> PG
    AI -.-> R
    MT -.-> PG

    P -.-> M
    AI -.-> M

    P -.-> K
    N -.-> K
    AI -.-> K
    MT -.-> K
```

## 🚀 Starting the Environment

### Method 1: Complete Environment (Recommended)

Start everything with proper sequencing and health checks:

```bash
# Start with build and logs
make local-all-up

# Start in background (faster)
make local-all-up-detached

# Start without rebuilding (fastest)
make local-all-up-fast
```

### Method 2: Infrastructure Only (for Service Development)

Start only shared services for individual service development:

```bash
# Start infrastructure
make local-infra-up

# Then start individual services manually
cd fn-posts && make run         # Terminal 1
cd fn-notifications && make up  # Terminal 2
cd fn-media-ai && make run      # Terminal 3
cd fn-matcher && make run       # Terminal 4
```

### Method 3: Service-Specific Development

Quick setup for developing a specific service:

```bash
# Setup for Posts development
make dev-posts
# Then: cd fn-posts && make run

# Setup for Notifications development
make dev-notifications
# Then: cd fn-notifications && make up

# Setup for Media AI development
make dev-media-ai
# Then: cd fn-media-ai && make run

# Setup for Matcher development
make dev-matcher
# Then: cd fn-matcher && make run
```

## 📊 Monitoring and Status

### Quick Status Check

To check the current status of your local development environment:

```bash
# Check all container status
cd fn-infra && docker compose -f local/docker-compose.yml ps

# Quick status overview
make local-all-status

# Comprehensive health check
make local-all-health

# Individual service health
curl http://localhost:8080/health      # Posts
curl http://localhost:4000/api/health  # Notifications
curl http://localhost:8000/health      # Media AI
curl http://localhost:3000/health      # Matcher
```

### Infrastructure Services Status

All infrastructure services should be properly configured and healthy:
- **PostgreSQL**: `findly-postgres:5432` with databases: `posts_db`, `notifications_db`, `matcher_db`
- **Kafka**: `findly-kafka:29092` in KRaft mode (no Zookeeper)
- **Redis**: `findly-redis:6379` for caching
- **MinIO**: `findly-minio:9000` for S3-compatible storage

### Service Connections

All services now use container networking instead of localhost:
- **Database**: `postgresql://service_user:service_password@findly-postgres:5432/service_db`
- **Kafka**: `findly-kafka:29092`
- **Redis**: `findly-redis:6379`
- **MinIO**: `findly-minio:9000`

### Viewing Logs

```bash
# All services logs
make local-all-logs

# Recent logs only
make local-all-logs-tail

# Individual service logs
make logs-posts
make logs-notifications
make logs-media-ai
make logs-matcher
make logs-kafka
make logs-postgres

# Individual service logs with Docker
docker logs findly-[service] --tail 50 -f
```

### Web Interfaces

```bash
# Open all monitoring interfaces
make open-all

# Individual interfaces
make open-health        # Health dashboard
make open-kafka-ui      # Event monitoring
make open-minio         # File storage
make open-posts         # Posts API
make open-notifications # Notifications dashboard
make open-media-ai      # AI API docs
make open-matcher       # Matcher API
```

## 🧪 Testing and Development

### Generate Test Data

Create realistic test data for development and testing:

```bash
# Generate complete dataset
./scripts/generate-test-data.sh --full

# Generate specific data
./scripts/generate-test-data.sh --posts 20
./scripts/generate-test-data.sh --posts 10 --matches
./scripts/generate-test-data.sh --notifications

# Clean up test data
./scripts/generate-test-data.sh --cleanup
```

### Run End-to-End Tests

Verify the complete system works together:

```bash
# Quick connectivity tests
./scripts/run-e2e-tests.sh --quick

# Complete test suite
./scripts/run-e2e-tests.sh --full

# Verbose testing with cleanup
./scripts/run-e2e-tests.sh --full --verbose --cleanup
```

### Manual Testing Workflows

#### Test Event Flow

1. **Create a Lost Post:**
```bash
curl -X POST http://localhost:8080/api/v1/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Lost iPhone 14 Pro",
    "description": "Space Gray iPhone 14 Pro with purple case",
    "item_type": "electronics",
    "post_type": "lost",
    "location": {
      "latitude": 37.7749,
      "longitude": -122.4194,
      "address": "Union Square, San Francisco"
    }
  }'
```

2. **Monitor Event Propagation:**
   - Visit Kafka UI: http://localhost:8081
   - Check `post.created` topic for the event
   - Watch for `post.enhanced` event from Media AI
   - Look for potential matches in Matcher

3. **Create Matching Found Post:**
```bash
curl -X POST http://localhost:8080/api/v1/posts \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Found iPhone",
    "description": "Found Space Gray iPhone with purple case near Union Square",
    "item_type": "electronics",
    "post_type": "found",
    "location": {
      "latitude": 37.7750,
      "longitude": -122.4195,
      "address": "Near Union Square, San Francisco"
    }
  }'
```

4. **Check for Matches:**
```bash
curl http://localhost:3000/api/v1/matches
```

#### Test API Endpoints

```bash
# Posts API
curl http://localhost:8080/api/v1/posts
curl http://localhost:8080/api/v1/posts/{id}

# Media AI API
curl http://localhost:8000/api/v1/models/status

# Matcher API
curl http://localhost:3000/api/v1/matches

# Notifications API
curl http://localhost:4000/api/notifications
```

## 🔧 Database Access

### PostgreSQL Connections

Each service has its own database for domain isolation:

```bash
# Quick database shell
make local-shell-postgres

# Connect to specific databases
psql postgresql://postgres:postgres@localhost:5432/posts_db
psql postgresql://postgres:postgres@localhost:5432/notifications_db
psql postgresql://postgres:postgres@localhost:5432/matcher_db

# Database access from Docker
docker exec -it findly-postgres psql -U postgres
```

### Database Users & Permissions

Each service has its own database user:
- `posts_user:posts_password` → `posts_db`
- `notifications_user:notifications_password` → `notifications_db`
- `matcher_user:matcher_password` → `matcher_db`

### Useful Database Queries

```sql
-- Posts database
\c posts_db
SELECT title, post_type, created_at FROM posts ORDER BY created_at DESC LIMIT 10;

-- Check spatial data
SELECT title, ST_AsText(location) as location FROM posts WHERE location IS NOT NULL;

-- Notifications database
\c notifications_db
SELECT type, title, status, created_at FROM notifications ORDER BY created_at DESC LIMIT 10;

-- Matcher database
\c matcher_db
SELECT lost_post_id, found_post_id, confidence_score, created_at FROM matches ORDER BY confidence_score DESC;
```

## 📡 Event Streaming (Kafka)

### Monitor Events

Access Kafka UI at http://localhost:8081 to monitor:

- **Topic Activity**: Real-time event flow
- **Consumer Groups**: Service consumption status
- **Message Content**: Event payload inspection

### Key Topics

| Topic | Purpose | Producers | Consumers |
|-------|---------|-----------|-----------|
| `post.created` | New posts | fn-posts | fn-media-ai, fn-matcher |
| `post.enhanced` | AI analysis done | fn-media-ai | fn-matcher |
| `post.matched` | Matches found | fn-matcher | fn-notifications |
| `notification.sent` | Notifications sent | fn-notifications | Analytics |

### Manual Event Testing

```bash
# Get into Kafka container
make local-shell-kafka

# List topics
kafka-topics --bootstrap-server localhost:9092 --list

# Watch events in real-time
kafka-console-consumer --bootstrap-server localhost:9092 --topic post.created --from-beginning

# Send test event
kafka-console-producer --bootstrap-server localhost:9092 --topic post.created

# Kafka topics with Docker
docker exec findly-kafka kafka-topics --list --bootstrap-server localhost:9092
```

## 💾 Object Storage (MinIO)

### Access MinIO Console

- **URL**: http://localhost:9001
- **Username**: `findly`
- **Password**: `findly123`

### Buckets

- `posts-photos-dev`: Development photos (public read)
- `posts-photos-staging`: Staging environment
- `posts-photos-prod`: Production testing

### Command Line Access

```bash
# Using MinIO client in container
docker exec -it findly-minio mc ls myminio/

# Using AWS CLI (if installed)
aws --endpoint-url=http://localhost:9000 \
    --access-key-id=findly \
    --secret-access-key=findly123 \
    s3 ls s3://posts-photos-dev
```

## 🛠️ Development Workflows

### Starting Fresh

```bash
# 1. Start infrastructure
cd fn-infra && make local-up

# 2. Verify infrastructure health
make local-health

# 3. Start individual services (in separate terminals)
cd fn-posts && make dev
cd fn-notifications && make up
cd fn-matcher && make dev
cd fn-media-ai && make dev
```

### Hot Reload Development

Each service supports hot reload for faster development:

```bash
# Go (fn-posts) - uses Air
cd fn-posts && make dev

# Elixir (fn-notifications) - built-in Phoenix hot reload
cd fn-notifications && make up

# Python (fn-media-ai) - uses uvicorn reload
cd fn-media-ai && make dev

# Rust (fn-matcher) - uses cargo-watch
cd fn-matcher && make dev
```

### Code Quality

```bash
# Format all services
make local-format-all

# Lint all services
make local-lint-all

# Update dependencies
make local-update-deps
```

### Building and Rebuilding

```bash
# Build all containers
make local-build-all

# Rebuild everything (no cache)
make local-rebuild-all

# Build specific service
docker-compose -f fn-infra/local/docker-compose.full.yml build fn-posts
```

## 🔍 Troubleshooting

### Service Status & Solutions

#### fn-posts ✅ HEALTHY
- **Status**: Working properly with container database connections
- **Database**: Successfully connecting to `posts_db` via `findly-postgres:5432`
- **Health Check**: `curl http://localhost:8080/health` returns 200 OK

#### fn-notifications 🔍 POTENTIAL ISSUES
- **Potential Issue**: Compilation issues with Elixir dependencies
- **Root Cause**: Mix compilation errors in beam_lib
- **Solution**:
  ```bash
  cd fn-notifications
  mix deps.clean --all
  mix deps.get
  mix compile
  ```

#### fn-matcher ⚠️ POTENTIAL ISSUES
- **Potential Issue**: Cargo permission errors accessing registry cache
- **Root Cause**: Docker volume permission issues with Rust cargo cache
- **Solution**:
  ```bash
  cd fn-matcher
  # Clear cargo cache and rebuild
  docker compose down
  docker volume prune -f
  docker compose up --build
  ```

#### fn-media-ai ⚠️ POTENTIAL ISSUES
- **Potential Issue**: Health check endpoint returning 404
- **Root Cause**: Health endpoint might be at different path
- **Investigation**: Service is running but health check URL incorrect
- **Solution**: Check if health endpoint is at `/api/v1/health` instead of `/health`

### Common Issues & Solutions

#### Services Won't Start

```bash
# Check Docker status
docker info

# Check for port conflicts
lsof -i :5432 -i :9092 -i :9000 -i :6379 -i :8080 -i :4000 -i :8000 -i :3000

# Clean restart
make local-all-clean
make local-all-up

# Check container logs
docker logs findly-[service-name] --tail 50

# Common fixes:
# 1. Environment file issues
cp fn-infra/local/env/fn-[service].env fn-[service]/.env

# 2. Permission issues
sudo chown -R $USER:$USER fn-[service]/

# 3. Port conflicts
lsof -i :8080  # Check if port is in use
```

#### Database Connection Issues

```bash
# Check PostgreSQL container
docker logs findly-postgres

# Verify database is ready
make local-shell-postgres

# Reset databases
make local-all-clean  # WARNING: Deletes all data
make local-all-up

# Test database connection from container
docker exec findly-postgres psql -U postgres -c "\l"

# Verify service .env files use container names
grep DATABASE_URL fn-*/env | grep findly-postgres
```

#### Kafka/Event Issues

```bash
# Check Kafka logs
make logs-kafka

# List topics
docker exec findly-kafka kafka-topics --bootstrap-server localhost:9092 --list

# Check consumer groups
docker exec findly-kafka kafka-consumer-groups --bootstrap-server localhost:9092 --list

# Verify topics exist
docker exec findly-kafka kafka-topics --list --bootstrap-server localhost:9092

# Check service .env files
grep KAFKA_BROKERS fn-*/.env | grep findly-kafka
```

#### Performance Issues

```bash
# Check resource usage
docker stats

# Check system resources
make requirements

# Reduce memory usage (edit docker-compose.full.yml)
# Kafka: KAFKA_HEAP_OPTS="-Xmx256M -Xms256M"

# Check Docker resource usage
docker stats

# Increase Docker memory/CPU limits in Docker Desktop
# Check for port conflicts
netstat -an | grep LISTEN | grep -E "(5432|9092|6379|9000)"
```

#### Individual Service Issues

```bash
# Check service logs
make logs-[service-name]

# Get into service container
make local-shell-[service-name]

# Restart specific service
docker-compose -f fn-infra/local/docker-compose.full.yml restart fn-posts
```

### When Things Break

```bash
# Nuclear option - complete reset
cd fn-infra
make local-clean  # WARNING: Deletes all data
make local-up

# Gentler option - restart services
docker compose -f local/docker-compose.yml restart
```

### Debug Commands

```bash
# Show all container status
make local-all-status

# Detailed health check
make local-all-health

# Test connectivity between services
make local-test-connectivity

# Test complete event flow
make local-test-event-flow

# View all logs
cd fn-infra && make local-logs
```

### Getting Help

1. **Check logs first**: `make local-all-logs`
2. **Verify health**: `make local-all-health`
3. **Test connectivity**: `make local-test-connectivity`
4. **Review configuration**: Check `fn-infra/local/env/` files
5. **Clean restart**: `make local-all-clean && make local-all-up`

## 🔄 Stopping and Cleanup

### Stop Environment

```bash
# Stop all services (keeps data)
make local-all-down

# Stop and remove volumes (deletes data)
make local-all-clean

# Stop individual services
docker-compose -f fn-infra/local/docker-compose.full.yml stop fn-posts
```

### Cleanup

```bash
# Remove test data
./scripts/generate-test-data.sh --cleanup

# Clean Docker system
docker system prune -f

# Remove all volumes (WARNING: deletes all data)
docker volume prune -f
```

## 🎯 Development Tips

### Efficient Development Cycle

1. **Start infrastructure only**: `make local-infra-up`
2. **Develop one service at a time** with hot reload
3. **Use test data**: `./scripts/generate-test-data.sh --full`
4. **Monitor events**: Keep Kafka UI open
5. **Watch logs**: `make logs-[service]` in separate terminal

### Debugging Strategies

- **Start simple**: Test individual services before full integration
- **Follow events**: Use Kafka UI to trace event flow
- **Check health endpoints**: Each service has `/health`
- **Use verbose logging**: Set `LOG_LEVEL=debug` in env files
- **Test incrementally**: Create simple test cases first

### Performance Optimization

- **Allocate enough RAM**: 8GB+ recommended
- **Use SSD storage**: Faster container startup
- **Close unused applications**: Free up system resources
- **Monitor Docker stats**: `docker stats`

## 🌐 Access URLs

### Application Services
- **fn-posts**: http://localhost:8080
- **fn-notifications**: http://localhost:4000
- **fn-matcher**: http://localhost:3000
- **fn-media-ai**: http://localhost:8000

### Infrastructure UIs
- **Kafka UI**: http://localhost:8081 (monitoring)
- **MinIO Console**: http://localhost:9001 (admin/admin123)
- **Health Dashboard**: http://localhost:8082

### Direct Infrastructure Access
- **PostgreSQL**: localhost:5432
- **Redis**: localhost:6379
- **Kafka**: localhost:9092
- **MinIO API**: localhost:9000

### Service URLs (Container Networking)

Services communicate using container names:
- **fn-posts**: `http://findly-posts:8080`
- **fn-notifications**: `http://findly-notifications:4000`
- **fn-matcher**: `http://findly-matcher:3000`
- **fn-media-ai**: `http://findly-media-ai:8000`

## 📁 Environment Configuration

### Environment Files (Container Networking)
- `fn-posts/.env` → Uses `findly-postgres:5432`
- `fn-notifications/.env` → Uses `findly-postgres:5432`
- `fn-matcher/.env` → Uses `findly-postgres:5432`
- `fn-media-ai/.env` → Uses `findly-kafka:29092`, `findly-redis:6379`

### Infrastructure Configuration
- `fn-infra/local/docker-compose.yml` → Infrastructure services
- `fn-infra/local/init-scripts/01-init-databases.sql` → Database setup
- `fn-infra/local/env/` → Pre-configured environment templates

## ✅ Success Criteria

The local development environment is considered ready when:

1. **Infrastructure**: All containers healthy (`docker ps` shows healthy status)
2. **Networking**: Services use container names (no localhost connections)
3. **Databases**: Each service connects to its dedicated database
4. **fn-posts**: Returns 200 from `curl http://localhost:8080/health`
5. **Environment**: All .env files copied from `fn-infra/local/env/`

## 📚 Next Steps

### Learn More

- **System Architecture**: [fn-docs/ARCHITECTURE.md](fn-docs/ARCHITECTURE.md)
- **Business Vision**: [fn-docs/VISION.md](fn-docs/VISION.md)
- **Service Documentation**: Individual README files in each service
- **API Documentation**: Available at service endpoints

### Contribute

- **Run tests**: `./scripts/run-e2e-tests.sh --full`
- **Follow code standards**: `make local-format-all && make local-lint-all`
- **Write tests**: Add E2E test cases
- **Update documentation**: Keep this guide current

### Deploy to Cloud

- **Cloud Setup Guide**: [fn-docs/CLOUD-SETUP.md](fn-docs/CLOUD-SETUP.md)
- **Infrastructure**: [fn-infra/k8s/](fn-infra/k8s/) Kubernetes manifests
- **CI/CD**: GitHub Actions workflows

---

## 🎉 You're Ready!

You now have the complete Findly Now platform running locally. The system is designed to help people reunite with their lost items through intelligent matching and notifications.

**Happy coding!** 🚀

For questions or issues, check the troubleshooting section above or review individual service documentation.