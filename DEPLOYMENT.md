# Findly Now - Deployment Guide

**Document Ownership**: This document OWNS deployment procedures, environment configuration, and operational runbooks.

## Quick Start Deployment

### Prerequisites Check

```bash
# Verify required tools
gcloud version          # Google Cloud SDK
terraform --version     # >= 1.5.0
helm version           # >= 3.12.0
kubectl version        # >= 1.28.0

# Authenticate with Google Cloud
gcloud auth login
gcloud config set project findly-production
```

### One-Command Deployment

```bash
# Development environment
make -C fn-infra initial-setup ENV=dev

# Production environment (requires approval)
make -C fn-infra initial-setup ENV=prod
```

## Environment Configuration

### Environment Tiers

| Environment | Purpose | Auto-Deploy | Scaling | Cost |
|------------|---------|-------------|---------|------|
| **dev** | Active development | On push to main | Min 1 pod | ~$50/month |
| **staging** | Pre-production testing | On release tag | Min 2 pods | ~$75/month |
| **production** | Live traffic | Manual approval | Min 3 pods | ~$150/month |

### Environment Variables

Each environment uses Google Secret Manager for sensitive configuration:

```bash
# View current secrets
gcloud secrets list --filter="labels.env=${ENV}"

# Update a secret
echo -n "new-value" | gcloud secrets versions add ${SECRET_NAME} --data-file=-

# Secrets automatically sync to pods via External Secrets Operator
```

## Deployment Workflows

### 1. Automated CI/CD Pipeline

**GitHub Actions Workflow** (Recommended):

```yaml
# Triggered automatically on code push
main branch → dev environment (automatic)
v*.*.* tag → staging environment (automatic)
production/* branch → production (manual approval)
```

**Pipeline Stages**:
1. **Build & Test** - Run unit and integration tests
2. **Security Scan** - Vulnerability scanning with Trivy
3. **Container Build** - Multi-arch Docker images
4. **Registry Push** - Google Artifact Registry
5. **Helm Deploy** - Update Kubernetes manifests
6. **Smoke Tests** - Verify deployment health
7. **Rollback Ready** - Automatic rollback on failure

### 2. Manual Deployment

For emergency deployments or specific service updates:

```bash
# Deploy specific service
cd fn-infra
make deploy-service SERVICE=fn-posts ENV=staging

# Deploy all services
make deploy-all ENV=staging

# Emergency deployment (skips some checks)
make emergency-deploy ENV=prod
```

### 3. Helm Chart Deployment

Direct Helm deployment for granular control:

```bash
# Upgrade a specific service
helm upgrade fn-posts ./helm/fn-posts \
  -f ./helm/fn-posts/values-${ENV}.yaml \
  --namespace findly \
  --create-namespace \
  --wait \
  --timeout 10m

# Rollback if needed
helm rollback fn-posts --namespace findly
```

## Secret Management

### Google Secret Manager Integration

All sensitive data is stored in Google Secret Manager and synced to Kubernetes:

```bash
# Create application secret
gcloud secrets create kafka-api-key-${ENV} \
  --replication-policy="automatic" \
  --labels="env=${ENV},service=shared"

# Add secret version
echo -n "your-secret-value" | \
  gcloud secrets versions add kafka-api-key-${ENV} --data-file=-

# Verify secret in cluster
kubectl get secret kafka-credentials -n findly -o yaml
```

### Secret Rotation

Automated secret rotation every 90 days:

```bash
# Manual rotation
make -C fn-infra rotate-secrets ENV=${ENV}

# Verify rotation
kubectl rollout restart deployment -n findly
```

## Workload Identity Configuration

### Service Account Binding

Each pod uses Workload Identity for secure Google Cloud access:

```bash
# Create service account
gcloud iam service-accounts create ${SERVICE}-sa \
  --display-name="${SERVICE} Workload Identity"

# Grant permissions
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SERVICE}-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectUser"

# Bind to Kubernetes
kubectl annotate serviceaccount ${SERVICE} \
  iam.gke.io/gcp-service-account=${SERVICE}-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  -n findly
```

### Permission Matrix

| Service | GCS | Secret Manager | Cloud SQL | Pub/Sub |
|---------|-----|---------------|-----------|---------|
| fn-posts | Read/Write | Read | Write | - |
| fn-notifications | - | Read | Read | - |
| fn-media-ai | Read | Read | - | - |
| fn-matcher | - | Read | Write | - |

## Monitoring and Observability

### Health Checks

All services expose standardized health endpoints:

```bash
# Check service health
kubectl exec -it deploy/fn-posts -n findly -- curl localhost:8080/health

# Check all services
for service in posts notifications media-ai matcher; do
  echo "Checking fn-${service}..."
  kubectl exec -it deploy/fn-${service} -n findly -- \
    curl -s localhost:8080/health | jq .status
done
```

### Monitoring Stack

**Prometheus + Grafana** for metrics and visualization:

```bash
# Access Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80
# Open http://localhost:3000 (admin/admin)

# Access Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090
```

### Log Aggregation

**Google Cloud Logging** with structured JSON logs:

```bash
# View logs for a service
gcloud logging read "resource.labels.cluster_name=findly-${ENV}-cluster \
  AND resource.labels.namespace_id=findly \
  AND resource.labels.pod_id:fn-posts" \
  --limit 50 \
  --format json

# Stream logs in real-time
kubectl logs -f deployment/fn-posts -n findly
```

### Alerts

Critical alerts configured in Google Cloud Monitoring:

| Alert | Threshold | Action |
|-------|-----------|--------|
| Pod CPU > 80% | 5 minutes | Auto-scale horizontally |
| Pod Memory > 90% | 5 minutes | Alert on-call engineer |
| Error rate > 1% | 2 minutes | Page on-call engineer |
| Latency P95 > 1s | 5 minutes | Investigate performance |
| Pod restarts > 3 | 10 minutes | Check logs and health |

## Scaling Strategies

### Horizontal Pod Autoscaling

```yaml
# Configured in Helm values
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### Vertical Pod Autoscaling

GKE Autopilot automatically adjusts pod resources based on usage:

```bash
# View VPA recommendations
kubectl describe vpa fn-posts-vpa -n findly

# Apply recommendations
kubectl patch deployment fn-posts -n findly \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources", "value": {...}}]'
```

### Manual Scaling

```bash
# Scale specific service
kubectl scale deployment fn-posts -n findly --replicas=5

# Scale all services
for service in posts notifications media-ai matcher; do
  kubectl scale deployment fn-${service} -n findly --replicas=3
done
```

## Rollback Procedures

### Automated Rollback

Deployments automatically rollback on failure:

```yaml
# Helm deployment with automatic rollback
helm upgrade fn-posts ./fn-posts \
  --atomic \
  --cleanup-on-fail \
  --timeout 10m
```

### Manual Rollback

```bash
# Rollback to previous version
helm rollback fn-posts -n findly

# Rollback to specific revision
helm rollback fn-posts 3 -n findly

# View rollout history
kubectl rollout history deployment/fn-posts -n findly
```

## Disaster Recovery

### Backup Strategy

Daily automated backups of critical data:

```bash
# Database backups (Cloud SQL)
gcloud sql backups create \
  --instance=findly-postgres-${ENV} \
  --description="Daily backup $(date +%Y%m%d)"

# Secret backups
gcloud secrets versions list ${SECRET_NAME} \
  --format="table(name,createTime,state)"
```

### Recovery Procedures

**Database Recovery**:
```bash
# Restore from backup
gcloud sql backups restore ${BACKUP_ID} \
  --backup-instance=findly-postgres-${ENV} \
  --target-instance=findly-postgres-${ENV}-restore
```

**Cluster Recovery**:
```bash
# Recreate cluster from Terraform
cd fn-infra/terraform
terraform apply -var="environment=${ENV}"

# Restore application state
helm install findly ./helm/findly \
  -f ./helm/findly/values-${ENV}.yaml
```

## Troubleshooting

### Common Issues

#### Pods Not Starting

```bash
# Check pod status
kubectl describe pod ${POD_NAME} -n findly

# Check events
kubectl get events -n findly --sort-by='.lastTimestamp'

# Check resource quotas
kubectl describe resourcequota -n findly
```

#### Service Unavailable

```bash
# Check service endpoints
kubectl get endpoints -n findly

# Test service connectivity
kubectl run debug --image=nicolaka/netshoot -it --rm
```

#### High Memory Usage

```bash
# Check memory usage
kubectl top pods -n findly

# Increase memory limits
kubectl set resources deployment fn-posts \
  --limits=memory=2Gi -n findly
```

### Debug Commands

```bash
# Interactive shell in pod
kubectl exec -it deployment/fn-posts -n findly -- /bin/sh

# Copy files from pod
kubectl cp findly/fn-posts-xxx:/app/logs/app.log ./app.log

# Port forward for local debugging
kubectl port-forward deployment/fn-posts 8080:8080 -n findly
```

## Security Checklist

- [ ] All secrets in Google Secret Manager
- [ ] Workload Identity configured for all pods
- [ ] Network policies restricting pod communication
- [ ] Binary Authorization enabled for container images
- [ ] Pod Security Standards enforced
- [ ] RBAC configured with least privilege
- [ ] Regular vulnerability scanning with Trivy
- [ ] TLS certificates auto-renewed with cert-manager
- [ ] Audit logs enabled and monitored
- [ ] Private GKE cluster with authorized networks

## Cost Optimization

### Resource Right-Sizing

```bash
# View resource recommendations
gcloud container clusters describe findly-${ENV}-cluster \
  --region=${REGION} \
  --format="value(verticalPodAutoscaling)"

# Apply recommendations via Helm values
helm upgrade fn-posts ./fn-posts \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=512Mi
```

### Cost Monitoring

```bash
# View cluster costs
gcloud billing accounts list
gcloud beta billing budgets list

# Cost breakdown by service
kubectl top pods -n findly --containers
```

### Optimization Tips

1. **Use Spot VMs** for non-critical workloads (dev/staging)
2. **Enable cluster autoscaling** to reduce idle resources
3. **Set resource requests** accurately to avoid overprovisioning
4. **Use regional storage** instead of multi-regional when possible
5. **Configure lifecycle policies** for old container images
6. **Implement pod disruption budgets** for safe node pool updates

## Operational Runbooks

### Daily Operations

```bash
# Morning health check
make -C fn-infra health-check ENV=${ENV}

# Review overnight alerts
gcloud alpha monitoring policies list --filter="enabled=true"

# Check backup status
gcloud sql operations list --instance=findly-postgres-${ENV}
```

### Weekly Maintenance

```bash
# Update dependencies
make -C fn-infra update-deps ENV=${ENV}

# Review security patches
gcloud container images scan ${IMAGE}

# Cleanup old resources
make -C fn-infra cleanup ENV=${ENV}
```

### Monthly Reviews

- Review and optimize resource allocations
- Audit access logs and permissions
- Update disaster recovery documentation
- Test backup restoration procedures
- Review cost trends and optimize

---

*For infrastructure code details, see [fn-infra repository](../fn-infra/). For service-specific deployment guides, see individual service repositories.*