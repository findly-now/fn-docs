# Findly Now - Migration Guide

**Document Ownership**: This document OWNS all migration procedures, breaking changes, and upgrade paths for the Findly Now ecosystem.

## Overview

This migration guide documents the major configuration standardization and system improvements implemented across the entire Findly Now ecosystem. These changes establish a unified, production-ready foundation for all microservices.

## Migration Summary

### What Changed
- **Kafka Configuration Standardization**: Unified environment variable names and topic naming across all services
- **Algorithm Weight Configuration**: Added configurable matching weights for fn-matcher service
- **Topic Naming Convention**: Established consistent topic naming pattern across the platform
- **Environment Variable Consistency**: Standardized variable names for better operational consistency

### Breaking Changes
- Kafka environment variable names changed in multiple services
- Topic names updated to follow new naming convention
- fn-matcher algorithm weights now configurable via environment variables

### Impact Assessment
- **Downtime Required**: Yes, for environment variable updates
- **Data Loss Risk**: None (topic data preserved, only configuration changes)
- **Rollback Complexity**: Medium (requires reverting environment variables)

## Pre-Migration Checklist

### 1. Backup Current Configuration
```bash
# Backup current Kubernetes deployment configurations
kubectl get deployments -n findly-now -o yaml > pre-migration-deployments.yaml

# Backup environment variables for all services
kubectl get configmaps -n findly-now -o yaml > pre-migration-configmaps.yaml
kubectl get secrets -n findly-now -o yaml > pre-migration-secrets.yaml

# Document current Kafka consumer group positions
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --describe --all-groups > pre-migration-consumer-positions.txt
```

### 2. Validate Current System Health
```bash
# Ensure all services are healthy before migration
kubectl get pods -n findly-now
kubectl exec deployment/fn-posts -- curl -f http://localhost:8080/health
kubectl exec deployment/fn-notifications -- curl -f http://localhost:4000/health
kubectl exec deployment/fn-media-ai -- curl -f http://localhost:8000/health
kubectl exec deployment/fn-matcher -- curl -f http://localhost:3000/health
```

### 3. Prepare New Configuration
Create new environment variable files with updated configurations for each service.

## Service-by-Service Migration

### 1. fn-posts (Go Service)

#### Changes Required
- Add missing `KAFKA_TOPIC` environment variable
- Ensure compliance with schema requirements

#### Migration Steps
```bash
# 1. Add new environment variable
kubectl patch deployment fn-posts -n findly-now -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-posts",
          "env": [
            {"name": "KAFKA_TOPIC", "value": "posts.events"}
          ]
        }]
      }
    }
  }
}'

# 2. Verify deployment rollout
kubectl rollout status deployment/fn-posts -n findly-now

# 3. Test event publishing
kubectl logs deployment/fn-posts | grep -i "kafka\|event\|publish"
```

#### Validation
```bash
# Verify posts are being published to correct topic
kafka-console-consumer --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --consumer.config kafka.properties \
  --topic posts.events \
  --from-beginning --max-messages 1
```

### 2. fn-notifications (Elixir Service)

#### Changes Required
- Update all event processors to use configurable topic names
- Add support for all standardized topic environment variables

#### Migration Steps
```bash
# 1. Update environment variables
kubectl patch deployment fn-notifications -n findly-now -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-notifications",
          "env": [
            {"name": "KAFKA_POSTS_TOPIC", "value": "posts.events"},
            {"name": "KAFKA_MATCHER_TOPIC", "value": "posts.matching"},
            {"name": "KAFKA_ENRICHMENT_TOPIC", "value": "media-ai.enrichment"},
            {"name": "KAFKA_USERS_TOPIC", "value": "users.lifecycle"}
          ]
        }]
      }
    }
  }
}'

# 2. Restart to pick up new configuration
kubectl rollout restart deployment/fn-notifications -n findly-now
kubectl rollout status deployment/fn-notifications -n findly-now

# 3. Monitor Broadway consumer startup
kubectl logs deployment/fn-notifications | grep -i "broadway\|consumer\|topic"
```

#### Validation
```bash
# Check that service is consuming from all expected topics
kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --group fn-notifications-production \
  --describe
```

### 3. fn-media-ai (Python Service)

#### Changes Required
- Change `KAFKA_BROKERS` to `KAFKA_BOOTSTRAP_SERVERS`
- Add missing `KAFKA_SASL_USERNAME` and `KAFKA_SASL_PASSWORD`
- Update topic variable names

#### Migration Steps
```bash
# 1. Update Kafka configuration variables
kubectl patch deployment fn-media-ai -n findly-now -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-media-ai",
          "env": [
            {"name": "KAFKA_BOOTSTRAP_SERVERS", "valueFrom": {"secretKeyRef": {"name": "kafka-secrets", "key": "bootstrap-servers"}}},
            {"name": "KAFKA_SASL_USERNAME", "valueFrom": {"secretKeyRef": {"name": "kafka-secrets", "key": "sasl-username"}}},
            {"name": "KAFKA_SASL_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "kafka-secrets", "key": "sasl-password"}}},
            {"name": "KAFKA_CONSUMER_TOPICS", "value": "posts.events"},
            {"name": "KAFKA_POST_ENHANCED_TOPIC", "value": "media-ai.enrichment"}
          ]
        }]
      }
    }
  }
}'

# 2. Remove old environment variable (if it exists)
kubectl patch deployment fn-media-ai -n findly-now --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/env", "value": {"name": "KAFKA_BROKERS"}}
]' || echo "KAFKA_BROKERS not found, continuing..."

# 3. Restart service
kubectl rollout restart deployment/fn-media-ai -n findly-now
kubectl rollout status deployment/fn-media-ai -n findly-now
```

#### Validation
```bash
# Verify service connects to Kafka successfully
kubectl logs deployment/fn-media-ai | grep -i "kafka\|bootstrap\|connection"

# Check photo analysis events are being published
kafka-console-consumer --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --consumer.config kafka.properties \
  --topic media-ai.enrichment \
  --property print.timestamp=true
```

### 4. fn-matcher (Rust Service)

#### Changes Required
- Update Kafka topic configuration to use `KAFKA_TOPICS_*` variables
- Add configurable algorithm weights via `MATCHING_*_WEIGHT` variables

#### Migration Steps
```bash
# 1. Update Kafka topic configuration
kubectl patch deployment fn-matcher -n findly-now -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "fn-matcher",
          "env": [
            {"name": "KAFKA_TOPICS_POSTS", "value": "posts.events"},
            {"name": "KAFKA_TOPICS_ENRICHMENT", "value": "media-ai.enrichment"},
            {"name": "KAFKA_TOPICS_MATCHING", "value": "posts.matching"},
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

# 2. Restart service to apply new algorithm weights
kubectl rollout restart deployment/fn-matcher -n findly-now
kubectl rollout status deployment/fn-matcher -n findly-now
```

#### Validation
```bash
# Verify algorithm weights are loaded correctly
kubectl logs deployment/fn-matcher | grep -i "weight\|algorithm\|config"

# Test matching functionality
kubectl logs deployment/fn-matcher | grep -i "match\|confidence\|score"
```

### 5. fn-contract (Schema Management)

#### Changes Required
- Update AsyncAPI specifications with correct topic names
- Add MatchExpired event definition

#### Migration Steps
```bash
# 1. Navigate to fn-contract repository
cd ../fn-contract

# 2. Update AsyncAPI specifications (already updated by contracts-maintainer)
# Verify schemas are using correct topic names:
grep -r "posts.events\|media-ai.enrichment\|posts.matching" schemas/

# 3. Validate and publish updated schemas
make validate-schemas
make publish-schemas
```

#### Validation
```bash
# Verify schema registry contains updated schemas
curl -X GET "http://schema-registry:8081/subjects" | jq .
```

## Topic Migration

### Topic Name Changes
The following topic name standardization was implemented:

| Service | Old Topic Name | New Topic Name | Purpose |
|---------|---------------|----------------|---------|
| fn-posts | `fn-posts.post.created` | `posts.events` | Posts lifecycle events |
| fn-media-ai | `fn-media-ai.post.enhanced` | `media-ai.enrichment` | AI photo analysis results |
| fn-matcher | `fn-matcher.post.matched` | `posts.matching` | Matching results and claims |
| External | Various | `users.lifecycle` | User management events |

### Topic Migration Process

#### Option 1: Zero-Downtime Migration (Recommended)
```bash
# 1. Create new topics alongside existing ones
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

# 2. Update producers to publish to new topics (service migrations above)
# 3. Update consumers to read from new topics (service migrations above)
# 4. Wait for old topics to drain (24-48 hours)
# 5. Delete old topics
kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --command-config kafka.properties \
  --delete --topic fn-posts.post.created
```

#### Option 2: Maintenance Window Migration
```bash
# 1. Stop all services
kubectl scale deployment fn-notifications --replicas=0 -n findly-now
kubectl scale deployment fn-media-ai --replicas=0 -n findly-now
kubectl scale deployment fn-matcher --replicas=0 -n findly-now
kubectl scale deployment fn-posts --replicas=0 -n findly-now

# 2. Create new topics
# (same as Option 1)

# 3. Apply all service configurations
# (same as service migrations above)

# 4. Scale services back up
kubectl scale deployment fn-posts --replicas=2 -n findly-now
kubectl scale deployment fn-media-ai --replicas=1 -n findly-now
kubectl scale deployment fn-matcher --replicas=2 -n findly-now
kubectl scale deployment fn-notifications --replicas=2 -n findly-now
```

## Environment-Specific Migration

### Development Environment
```bash
# Local development uses the same topic names but simpler configuration
export KAFKA_BOOTSTRAP_SERVERS="localhost:9094"
export KAFKA_SECURITY_PROTOCOL="PLAINTEXT"

# No SASL credentials needed for local Kafka
# Topic names remain the same
export KAFKA_TOPICS_POSTS="posts.events"
export KAFKA_TOPICS_ENRICHMENT="media-ai.enrichment"
export KAFKA_TOPICS_MATCHING="posts.matching"
```

### Staging Environment
```bash
# Staging mirrors production configuration
# Use staging-specific consumer group IDs:
KAFKA_CONSUMER_GROUP_ID="fn-notifications-staging"
KAFKA_CONSUMER_GROUP_ID="fn-media-ai-staging"
KAFKA_CONSUMER_GROUP_ID="fn-matcher-staging"
```

### Production Environment
```bash
# Production uses full Confluent Cloud configuration
# All variables as documented in ENVIRONMENT-VARIABLES.md
# Consumer group IDs use "-production" suffix
```

## Algorithm Weight Tuning

### New Configurable Weights (fn-matcher)
The fn-matcher service now supports runtime algorithm weight configuration:

#### Default Production Weights
```bash
MATCHING_LOCATION_WEIGHT=0.3  # 30% - Geographic proximity
MATCHING_VISUAL_WEIGHT=0.4    # 40% - AI-powered visual similarity
MATCHING_TEXT_WEIGHT=0.2      # 20% - Text description matching
MATCHING_TEMPORAL_WEIGHT=0.1  # 10% - Time-based proximity
```

#### Environment-Specific Tuning Examples
```bash
# Urban/High-Density Areas (reduce location importance)
MATCHING_LOCATION_WEIGHT=0.15
MATCHING_VISUAL_WEIGHT=0.5
MATCHING_TEXT_WEIGHT=0.25
MATCHING_TEMPORAL_WEIGHT=0.1

# Rural/Low-Density Areas (increase location importance)
MATCHING_LOCATION_WEIGHT=0.5
MATCHING_VISUAL_WEIGHT=0.3
MATCHING_TEXT_WEIGHT=0.15
MATCHING_TEMPORAL_WEIGHT=0.05

# Testing/Development (favor visual for AI testing)
MATCHING_LOCATION_WEIGHT=0.2
MATCHING_VISUAL_WEIGHT=0.6
MATCHING_TEXT_WEIGHT=0.15
MATCHING_TEMPORAL_WEIGHT=0.05
```

#### Weight Validation
All weights must sum to 1.0 (within 0.01 tolerance) or the service will fail to start with a clear error message.

## Post-Migration Validation

### System Health Verification
```bash
#!/bin/bash
# Post-migration validation script

echo "=== Post-Migration Validation ==="

# 1. Check all services are healthy
echo "Checking service health..."
for service in fn-posts fn-notifications fn-media-ai fn-matcher; do
  if kubectl exec deployment/$service -- curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "✓ $service healthy"
  else
    echo "✗ $service unhealthy"
    exit 1
  fi
done

# 2. Verify Kafka connectivity
echo "Checking Kafka connectivity..."
if kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
   --command-config kafka.properties --list > /dev/null 2>&1; then
  echo "✓ Kafka accessible"
else
  echo "✗ Kafka connection failed"
  exit 1
fi

# 3. Check topic existence
echo "Verifying topics exist..."
for topic in posts.events media-ai.enrichment posts.matching; do
  if kafka-topics --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
     --command-config kafka.properties \
     --describe --topic $topic > /dev/null 2>&1; then
    echo "✓ Topic $topic exists"
  else
    echo "✗ Topic $topic missing"
    exit 1
  fi
done

# 4. Check consumer groups are active
echo "Checking consumer group activity..."
for group in fn-notifications-production fn-media-ai-production fn-matcher-production; do
  if kafka-consumer-groups --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
     --command-config kafka.properties \
     --describe --group $group | grep -q "STABLE"; then
    echo "✓ Consumer group $group active"
  else
    echo "⚠ Consumer group $group not active or has lag"
  fi
done

# 5. Test end-to-end flow
echo "Testing end-to-end event flow..."
# This would involve creating a test post and verifying it flows through all services
echo "Manual testing required: Create a test post and verify notifications/matching work"

echo "=== Validation Complete ==="
```

### Functional Testing
```bash
# 1. Test posts creation and event publishing
curl -X POST http://fn-posts/api/posts \
  -H "Content-Type: application/json" \
  -d '{"title": "Test Migration Post", "description": "Testing migration", "location": {"lat": 40.7128, "lng": -74.0060}}'

# 2. Verify event appears in posts.events topic
kafka-console-consumer --bootstrap-server $KAFKA_BOOTSTRAP_SERVERS \
  --consumer.config kafka.properties \
  --topic posts.events \
  --from-beginning --max-messages 1

# 3. Test matching algorithm with new weights
# Monitor fn-matcher logs for weight loading confirmation
kubectl logs deployment/fn-matcher | grep -i "algorithm.*weight"
```

## Rollback Procedures

### Emergency Rollback
If issues are discovered after migration:

```bash
#!/bin/bash
# Emergency rollback script

echo "=== EMERGENCY ROLLBACK ==="

# 1. Restore previous deployment configuration
kubectl apply -f pre-migration-deployments.yaml

# 2. Restore previous ConfigMaps and Secrets
kubectl apply -f pre-migration-configmaps.yaml
kubectl apply -f pre-migration-secrets.yaml

# 3. Reset consumer groups to pre-migration positions
# (Only if pre-migration-consumer-positions.txt was created)
echo "Manual consumer group reset required using pre-migration-consumer-positions.txt"

# 4. Restart all services
kubectl rollout restart deployment/fn-posts -n findly-now
kubectl rollout restart deployment/fn-notifications -n findly-now
kubectl rollout restart deployment/fn-media-ai -n findly-now
kubectl rollout restart deployment/fn-matcher -n findly-now

echo "=== ROLLBACK COMPLETE - VERIFY SYSTEM HEALTH ==="
```

### Partial Rollback
For rolling back individual services:

```bash
# Example: Rollback fn-media-ai only
kubectl rollout undo deployment/fn-media-ai -n findly-now
kubectl rollout status deployment/fn-media-ai -n findly-now
```

## Monitoring and Alerting Updates

### New Metrics to Monitor
After migration, monitor these additional metrics:

```bash
# Algorithm weight effectiveness (fn-matcher)
curl http://fn-matcher:3000/metrics | grep matching_confidence_distribution

# Topic throughput on new topic names
kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list $KAFKA_BOOTSTRAP_SERVERS \
  --topic posts.events \
  --time -1
```

### Updated Alert Rules
```yaml
# Update Prometheus alert rules for new topic names
- alert: PostsEventLag
  expr: kafka_consumer_lag{topic="posts.events"} > 100
  for: 5m
  annotations:
    summary: "High lag on posts.events topic"

- alert: EnrichmentEventLag
  expr: kafka_consumer_lag{topic="media-ai.enrichment"} > 50
  for: 5m
  annotations:
    summary: "High lag on media-ai.enrichment topic"
```

## Success Criteria

The migration is considered successful when:

1. ✅ All services start successfully with new configuration
2. ✅ All Kafka consumers are actively processing events
3. ✅ No consumer lag > 100 messages on any topic
4. ✅ End-to-end flow works (post creation → AI analysis → matching → notifications)
5. ✅ Algorithm weights are correctly applied and validated
6. ✅ No error logs related to configuration or connectivity
7. ✅ All health checks pass across all services
8. ✅ Performance metrics remain within acceptable ranges

## Timeline and Coordination

### Recommended Migration Schedule
1. **Preparation Phase** (1-2 days): Backup, validation, testing in staging
2. **Migration Window** (2-4 hours): Service updates and testing
3. **Validation Phase** (24 hours): Monitoring and issue resolution
4. **Cleanup Phase** (1 week): Remove old topics and configuration

### Team Coordination
- **DevOps**: Infrastructure updates and monitoring
- **Backend Engineers**: Service-specific testing and validation
- **QA**: End-to-end functional testing
- **Product**: User-facing validation and acceptance testing

---

*This migration guide ensures a smooth transition to the standardized Findly Now configuration. For issues during migration, consult [TROUBLESHOOTING.md](TROUBLESHOOTING.md) or contact the DevOps team immediately.*