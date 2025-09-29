# Findly Now - Troubleshooting Guide

**Document Ownership**: This document OWNS troubleshooting procedures, common issues resolution, and diagnostic tools for the entire Findly Now ecosystem.

## Overview

This comprehensive troubleshooting guide provides solutions for common issues across all Findly Now microservices, with special focus on Kafka configuration problems, database connectivity issues, and service integration challenges.

## Quick Diagnostic Checklist

Before diving into specific troubleshooting sections, run through this quick checklist:

### 1. Service Health Check
```bash
# Check all service health endpoints
curl -f http://fn-posts:8080/health || echo "Posts service unhealthy"
curl -f http://fn-notifications:4000/health || echo "Notifications service unhealthy"
curl -f http://fn-media-ai:8000/health || echo "Media AI service unhealthy"
curl -f http://fn-matcher:3000/health || echo "Matcher service unhealthy"
```

### 2. Kafka Connectivity Check
```bash
# Verify Kafka cluster is reachable
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties --list
```

### 3. Database Connectivity Check
```bash
# Test PostgreSQL connections
psql $DATABASE_URL -c "SELECT 1;" || echo "Database connection failed"
```

### 4. Environment Variable Validation
```bash
# Verify critical environment variables are set
echo "Kafka Bootstrap: ${KAFKA_BOOTSTRAP_SERVERS:-NOT_SET}"
echo "Database URL: ${DATABASE_URL:-NOT_SET}"
echo "Environment: ${ENVIRONMENT:-NOT_SET}"
```

## Kafka Configuration Issues

### Common Kafka Problems

#### 1. Connection Authentication Failures

**Symptoms:**
- Services fail to start with Kafka authentication errors
- Error messages like "SASL authentication failed" or "Invalid credentials"
- Consumer groups not showing up in Kafka management tools

**Diagnosis:**
```bash
# Test Kafka authentication manually
kafka-console-consumer --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --consumer.config kafka.properties \
  --topic posts.events \
  --from-beginning --max-messages 1

# Check if credentials are correctly set
echo "Username: ${KAFKA_SASL_USERNAME:-NOT_SET}"
echo "Password: ${KAFKA_SASL_PASSWORD:0:8}..." # Show only first 8 characters
```

**Solutions:**
```bash
# 1. Verify environment variables use correct names
export KAFKA_BOOTSTRAP_SERVERS="pkc-xxxxx.region.aws.confluent.cloud:9092"
export KAFKA_SASL_USERNAME="your-confluent-api-key"
export KAFKA_SASL_PASSWORD="your-confluent-api-secret"
export KAFKA_SECURITY_PROTOCOL="SASL_SSL"
export KAFKA_SASL_MECHANISM="PLAIN"

# 2. Create kafka.properties file for CLI tools
cat > kafka.properties << EOF
bootstrap.servers=${KAFKA_BOOTSTRAP_SERVERS}
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${KAFKA_SASL_USERNAME}" password="${KAFKA_SASL_PASSWORD}";
EOF

# 3. For fn-media-ai service (common issue), ensure variables match expected names
# Change from KAFKA_BROKERS to KAFKA_BOOTSTRAP_SERVERS
```

#### 2. Topic Not Found Errors

**Symptoms:**
- Services fail with "Topic 'xyz' does not exist" errors
- Consumer groups show no activity
- Producer errors about unknown topics

**Diagnosis:**
```bash
# List all available topics
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties --list

# Check specific topic details
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --describe --topic posts.events
```

**Solutions:**
```bash
# 1. Create missing topics (if auto-creation is disabled)
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --create --topic posts.events \
  --partitions 3 --replication-factor 3

kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --create --topic media-ai.enrichment \
  --partitions 3 --replication-factor 3

kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --create --topic posts.matching \
  --partitions 3 --replication-factor 3

# 2. Verify topic names in environment variables match actual topics
echo "Expected topics:"
echo "  posts.events (not fn-posts.post.created)"
echo "  media-ai.enrichment (not fn-media-ai.post.enhanced)"
echo "  posts.matching (not fn-matcher.post.matched)"
```

#### 3. Consumer Group Lag Issues

**Symptoms:**
- Events are not being processed in real-time
- Consumer lag continuously increasing
- Services appear healthy but data is stale

**Diagnosis:**
```bash
# Check consumer group lag for all services
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-notifications-production --describe

kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-media-ai-production --describe

kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-matcher-production --describe
```

**Solutions:**
```bash
# 1. Scale up consumer instances
kubectl scale deployment fn-notifications --replicas=3

# 2. Reset consumer group to latest (if safe to lose messages)
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-notifications-production \
  --reset-offsets --to-latest --topic posts.events --execute

# 3. Check for processing bottlenecks in service logs
kubectl logs deployment/fn-notifications | grep -i "error\|timeout\|slow"
```

#### 4. Variable Name Mismatches

**Problem**: Services using inconsistent environment variable names for Kafka configuration.

**Service-Specific Fixes:**

**fn-media-ai (Python) Common Issue:**
```bash
# ❌ Old/Incorrect variables
KAFKA_BROKERS="kafka-cluster.confluent.cloud:9092"

# ✅ Correct variables
KAFKA_BOOTSTRAP_SERVERS="kafka-cluster.confluent.cloud:9092"
KAFKA_SASL_USERNAME="your-api-key"
KAFKA_SASL_PASSWORD="your-api-secret"
```

**fn-matcher (Rust) Common Issue:**
```bash
# ❌ Old topic variable names
KAFKA_TOPICS_POSTS_LIFECYCLE="posts.events"

# ✅ Correct variable names
KAFKA_TOPICS_POSTS="posts.events"
KAFKA_TOPICS_ENRICHMENT="media-ai.enrichment"
KAFKA_TOPICS_MATCHING="posts.matching"
```

## Database Connectivity Issues

### PostgreSQL Connection Problems

#### 1. Connection Pool Exhaustion

**Symptoms:**
- "Too many connections" errors
- Services hanging on database operations
- Intermittent connection failures

**Diagnosis:**
```bash
# Check current connection count
psql $DATABASE_URL -c "
  SELECT count(*) as active_connections,
         max_conn,
         max_conn - count(*) as available_connections
  FROM pg_stat_activity,
       (SELECT setting::int as max_conn FROM pg_settings WHERE name='max_connections') mc
  GROUP BY max_conn;"
```

**Solutions:**
```bash
# 1. Adjust connection pool settings per service
# fn-posts
DATABASE_MAX_CONNECTIONS=10
DATABASE_MIN_CONNECTIONS=2

# fn-matcher
DATABASE_MAX_CONNECTIONS=20
DATABASE_MIN_CONNECTIONS=5

# 2. Check for connection leaks in application logs
kubectl logs deployment/fn-posts | grep -i "connection.*timeout\|pool.*exhausted"

# 3. Monitor connection patterns
psql $DATABASE_URL -c "
  SELECT application_name, state, count(*)
  FROM pg_stat_activity
  WHERE datname = current_database()
  GROUP BY application_name, state;"
```

#### 2. PostGIS Extension Issues (fn-matcher)

**Symptoms:**
- "function ST_DWithin does not exist" errors
- Spatial query failures
- Migration failures

**Diagnosis:**
```bash
# Check if PostGIS is installed and available
psql $DATABASE_URL -c "SELECT PostGIS_Full_Version();"

# Check available spatial functions
psql $DATABASE_URL -c "
  SELECT proname
  FROM pg_proc
  WHERE proname LIKE 'st_%'
  LIMIT 10;"
```

**Solutions:**
```bash
# 1. Install PostGIS extension (requires superuser)
psql $DATABASE_URL -c "CREATE EXTENSION IF NOT EXISTS postgis;"
psql $DATABASE_URL -c "CREATE EXTENSION IF NOT EXISTS postgis_topology;"

# 2. Grant permissions to application user
psql $DATABASE_URL -c "
  GRANT USAGE ON SCHEMA public TO your_app_user;
  GRANT CREATE ON SCHEMA public TO your_app_user;"

# 3. Verify spatial indexes are working
psql $DATABASE_URL -c "
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT * FROM posts
  WHERE ST_DWithin(location, ST_Point(-74.006, 40.7128), 1000);"
```

#### 3. Connection Timeout Issues

**Symptoms:**
- "Connection timeout" errors during startup
- Services failing health checks
- Database queries hanging

**Diagnosis:**
```bash
# Test connection latency
time psql $DATABASE_URL -c "SELECT 1;"

# Check for long-running queries
psql $DATABASE_URL -c "
  SELECT pid, now() - pg_stat_activity.query_start AS duration, query
  FROM pg_stat_activity
  WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';"
```

**Solutions:**
```bash
# 1. Adjust timeout settings
DATABASE_CONNECT_TIMEOUT=30s
DATABASE_IDLE_TIMEOUT=600s
DATABASE_MAX_LIFETIME=1800s

# 2. Check network connectivity
ping your-database-host

# 3. Review connection string SSL settings
# For Supabase/cloud databases, ensure sslmode=require
DATABASE_URL="postgresql://user:pass@host:port/db?sslmode=require"
```

## Service Integration Issues

### Inter-Service Communication Problems

#### 1. Event Processing Delays

**Symptoms:**
- Notifications not sent promptly after post creation
- Matching results delayed or missing
- AI analysis not triggering

**Diagnosis:**
```bash
# Check event flow across topics
kafka-console-consumer --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --consumer.config kafka.properties \
  --topic posts.events \
  --property print.timestamp=true

# Monitor service processing logs
kubectl logs deployment/fn-notifications | grep -E "Processing.*event|Error.*processing"
kubectl logs deployment/fn-media-ai | grep -E "Analyzing.*photo|Error.*analysis"
kubectl logs deployment/fn-matcher | grep -E "Matching.*posts|Error.*matching"
```

**Solutions:**
```bash
# 1. Verify all services are consuming from correct topics
# Check fn-notifications consumer config
kubectl exec deployment/fn-notifications -- env | grep KAFKA_.*_TOPIC

# 2. Check for consumer group conflicts
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --list | grep -E "fn-notifications|fn-media-ai|fn-matcher"

# 3. Restart services in dependency order
kubectl rollout restart deployment/fn-posts
sleep 30
kubectl rollout restart deployment/fn-media-ai
kubectl rollout restart deployment/fn-matcher
kubectl rollout restart deployment/fn-notifications
```

#### 2. Algorithm Weight Configuration Issues (fn-matcher)

**Symptoms:**
- Matching results seem incorrect or biased
- Low match confidence scores
- Algorithm startup failures

**Diagnosis:**
```bash
# Check current algorithm weights
kubectl exec deployment/fn-matcher -- env | grep MATCHING_.*_WEIGHT

# Verify weights sum to 1.0
python3 -c "
weights = [0.3, 0.4, 0.2, 0.1]  # location, visual, text, temporal
total = sum(weights)
print(f'Total weight: {total}')
print('Valid:', abs(total - 1.0) < 0.01)
"
```

**Solutions:**
```bash
# 1. Set correct algorithm weights (must sum to 1.0)
kubectl patch deployment fn-matcher -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-matcher",
          "env": [
            {"name": "MATCHING_LOCATION_WEIGHT", "value": "0.3"},
            {"name": "MATCHING_VISUAL_WEIGHT", "value": "0.4"},
            {"name": "MATCHING_TEXT_WEIGHT", "value": "0.2"},
            {"name": "MATCHING_TEMPORAL_WEIGHT", "value": "0.1"}
          ]
        }]
      }
    }
  }
}'

# 2. Test different weight configurations for specific environments
# Urban environment (reduce location weight)
MATCHING_LOCATION_WEIGHT=0.15
MATCHING_VISUAL_WEIGHT=0.5
MATCHING_TEXT_WEIGHT=0.25
MATCHING_TEMPORAL_WEIGHT=0.1
```

## Environment-Specific Issues

### Development Environment

#### 1. Local Kafka Setup Issues

**Symptoms:**
- Services can't connect to local Kafka
- "Connection refused" errors
- Topic creation failures

**Solutions:**
```bash
# 1. Start local Kafka with correct ports
docker-compose up -d kafka

# 2. Use correct local Kafka configuration
export KAFKA_BOOTSTRAP_SERVERS="localhost:9094"
export KAFKA_SECURITY_PROTOCOL="PLAINTEXT"
# No SASL credentials needed for local development

# 3. Create topics manually if auto-creation is disabled
kafka-topics --bootstrap-server localhost:9094 \
  --create --topic posts.events --partitions 1 --replication-factor 1
```

#### 2. Database Migration Issues

**Solutions:**
```bash
# 1. Ensure development database is running
docker-compose up -d postgres

# 2. Run migrations for each service
cd fn-posts && make migrate-up
cd fn-matcher && sqlx migrate run
cd fn-notifications && mix ecto.migrate
```

### Production Environment

#### 1. Memory and Performance Issues

**Symptoms:**
- Services restarting due to OOM kills
- High CPU usage
- Slow response times

**Diagnosis:**
```bash
# Check resource usage
kubectl top pods -n findly-now

# Check memory limits and requests
kubectl describe deployment fn-media-ai | grep -A5 -B5 "Limits\|Requests"
```

**Solutions:**
```bash
# 1. Adjust resource limits based on actual usage
kubectl patch deployment fn-media-ai -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-media-ai",
          "resources": {
            "requests": {"memory": "512Mi", "cpu": "250m"},
            "limits": {"memory": "1Gi", "cpu": "500m"}
          }
        }]
      }
    }
  }
}'

# 2. Enable horizontal pod autoscaling
kubectl autoscale deployment fn-media-ai --cpu-percent=70 --min=1 --max=5
```

## Emergency Recovery Procedures

### Service Recovery

#### Complete System Recovery
```bash
#!/bin/bash
# Emergency recovery script

echo "Starting emergency recovery..."

# 1. Check Kafka cluster health
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties --list > /dev/null
if [ $? -eq 0 ]; then
  echo "✓ Kafka cluster accessible"
else
  echo "✗ Kafka cluster issues - contact Confluent support"
  exit 1
fi

# 2. Restart services in dependency order
echo "Restarting core services..."
kubectl rollout restart deployment/fn-posts -n findly-now
kubectl rollout status deployment/fn-posts -n findly-now --timeout=300s

echo "Restarting dependent services..."
kubectl rollout restart deployment/fn-media-ai -n findly-now
kubectl rollout restart deployment/fn-matcher -n findly-now
kubectl rollout restart deployment/fn-notifications -n findly-now

# 3. Wait for all services to be ready
kubectl wait --for=condition=available deployment --all -n findly-now --timeout=600s

# 4. Verify system health
echo "Verifying system health..."
for service in fn-posts fn-notifications fn-media-ai fn-matcher; do
  if kubectl exec deployment/$service -- curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "✓ $service healthy"
  else
    echo "✗ $service unhealthy"
  fi
done

echo "Emergency recovery complete"
```

### Consumer Group Reset
```bash
#!/bin/bash
# Reset consumer groups if events are stuck

# WARNING: This will lose unprocessed messages

# Reset all consumer groups to latest
for group in fn-notifications-production fn-media-ai-production fn-matcher-production; do
  echo "Resetting consumer group: $group"
  kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
    --command-config kafka.properties \
    --group $group \
    --reset-offsets --to-latest --all-topics --execute
done

# Restart services to pick up from latest offsets
kubectl rollout restart deployment/fn-notifications
kubectl rollout restart deployment/fn-media-ai
kubectl rollout restart deployment/fn-matcher
```

## Monitoring and Alerting

### Key Metrics to Monitor

#### Kafka Metrics
```bash
# Consumer lag monitoring
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --describe --all-groups | grep LAG

# Topic throughput monitoring
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list $KAFKA_BOOTSTRAP_SERVERS \
  --topic posts.events \
  --time -1
```

#### Service Health Metrics
```bash
# Health check all services
for service in fn-posts fn-notifications fn-media-ai fn-matcher; do
  response=$(kubectl exec deployment/$service -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)
  echo "$service: HTTP $response"
done
```

### Automated Monitoring Setup

#### Prometheus AlertManager Rules
```yaml
groups:
- name: findly-now-alerts
  rules:
  - alert: KafkaConsumerLag
    expr: kafka_consumer_lag_sum > 1000
    for: 5m
    annotations:
      summary: "High consumer lag detected"
      description: "Consumer lag is {{ $value }} messages"

  - alert: ServiceDown
    expr: up{job=~"fn-.*"} == 0
    for: 2m
    annotations:
      summary: "Service {{ $labels.job }} is down"

  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 5m
    annotations:
      summary: "High error rate in {{ $labels.service }}"
```

---

*This troubleshooting guide covers the most common issues encountered in the Findly Now ecosystem. For service-specific issues, see individual service documentation. For emergency support, contact the DevOps team.*