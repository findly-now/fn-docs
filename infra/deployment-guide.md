# fn-infra Deployment Guide

**Complete infrastructure deployment guide for setting up the cloud-native Kubernetes infrastructure and CI/CD pipelines.**

## Prerequisites

### Cloud Provider Setup
- **Google Cloud Platform** account with billing enabled
- **GCP Project** with required APIs enabled:
  - Kubernetes Engine API
  - Container Registry API
  - Cloud Resource Manager API
  - IAM Service Account Credentials API
  - Cloud DNS API
  - Cloud Monitoring API

### Required Tools
- **gcloud CLI** 400.0.0+ for GCP management
- **kubectl** 1.27+ for Kubernetes operations
- **Terraform** 1.5+ for infrastructure provisioning
- **Helm** 3.12+ for application deployment
- **Docker** 20.10+ for container operations

### Platform Compatibility
- **Apple Silicon Support**: Local development uses `platform: linux/amd64` in docker-compose files for service compatibility
- **Multi-platform Builds**: Infrastructure supports both AMD64 and ARM64 architectures
- **See**: [Platform Compatibility Guide](../CLOUD-SETUP.md#platform-compatibility)

### Access Requirements
- GCP Owner or Editor role for initial setup
- GitHub repository admin access for CI/CD setup
- Domain name for production ingress (optional)

## Initial Infrastructure Setup

### 1. GCP Project Configuration

**Create and Configure Project**:
```bash
# Set project variables
export PROJECT_ID="findly-now-production"
export BILLING_ACCOUNT_ID="your-billing-account-id"
export REGION="us-central1"

# Create project
gcloud projects create $PROJECT_ID
gcloud config set project $PROJECT_ID

# Link billing account
gcloud billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID

# Enable required APIs
gcloud services enable container.googleapis.com
gcloud services enable containerregistry.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com
gcloud services enable iamcredentials.googleapis.com
gcloud services enable dns.googleapis.com
gcloud services enable monitoring.googleapis.com
```

**Create Service Accounts**:
```bash
# Terraform service account
gcloud iam service-accounts create terraform-sa \
    --description="Terraform service account" \
    --display-name="Terraform SA"

# Grant necessary permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:terraform-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/editor"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:terraform-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountAdmin"

# Create and download key
gcloud iam service-accounts keys create terraform-key.json \
    --iam-account=terraform-sa@$PROJECT_ID.iam.gserviceaccount.com
```

### 2. Terraform State Backend Setup

**Create GCS Bucket for Terraform State**:
```bash
# Create bucket for Terraform state
gsutil mb -p $PROJECT_ID -c REGIONAL -l $REGION gs://$PROJECT_ID-terraform-state

# Enable versioning
gsutil versioning set on gs://$PROJECT_ID-terraform-state

# Set lifecycle policy
cat > lifecycle.json << EOF
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "Delete"},
        "condition": {
          "age": 30,
          "isLive": false
        }
      }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://$PROJECT_ID-terraform-state
```

### 3. Initial Infrastructure Deployment

**Clone and Configure fn-infra**:
```bash
# Clone repository
git clone https://github.com/findly-now/fn-infra.git
cd fn-infra

# Copy and configure environment
cp terraform/environments/production/terraform.tfvars.example \
   terraform/environments/production/terraform.tfvars

# Edit configuration
vi terraform/environments/production/terraform.tfvars
```

**Production Terraform Configuration**:
```hcl
# terraform/environments/production/terraform.tfvars
project_id = "findly-now-production"
region     = "us-central1"
environment = "production"

# Cluster configuration
cluster_name = "findly-production"
cluster_zones = ["us-central1-a", "us-central1-b", "us-central1-c"]

# Node pool configuration
node_pools = {
  application_pool = {
    machine_type = "e2-standard-8"
    disk_size    = 200
    disk_type    = "pd-ssd"
    min_nodes    = 5
    max_nodes    = 50
    preemptible  = false
  }

  monitoring_pool = {
    machine_type = "e2-standard-4"
    disk_size    = 100
    min_nodes    = 2
    max_nodes    = 8
    preemptible  = false
    taints = [{
      key    = "monitoring"
      value  = "true"
      effect = "NO_SCHEDULE"
    }]
  }
}

# Network configuration
vpc_cidr = "10.0.0.0/16"
subnet_cidr = "10.0.1.0/24"
pod_cidr = "10.1.0.0/16"
service_cidr = "10.2.0.0/16"

# Security configuration
authorized_networks = [
  {
    cidr_block   = "0.0.0.0/0"  # Restrict this in production
    display_name = "all-networks"
  }
]

# Monitoring configuration
enable_monitoring = true
enable_logging    = true

# Backup configuration
backup_enabled = true
backup_retention_days = 30
```

**Deploy Infrastructure**:
```bash
# Initialize Terraform
cd terraform/environments/production
terraform init

# Plan deployment
terraform plan -out=tfplan

# Apply infrastructure
terraform apply tfplan

# Verify cluster creation
gcloud container clusters list
```

## Kubernetes Configuration

### 1. Cluster Access Setup

**Configure kubectl Access**:
```bash
# Get cluster credentials
gcloud container clusters get-credentials findly-production \
    --region us-central1 \
    --project findly-now-production

# Verify access
kubectl cluster-info
kubectl get nodes
```

### 2. Base Kubernetes Resources

**Create Namespaces and RBAC**:
```bash
# Apply base Kubernetes configuration
kubectl apply -f k8s/base/namespaces/
kubectl apply -f k8s/base/rbac/
kubectl apply -f k8s/base/network-policies/

# Verify namespace creation
kubectl get namespaces
kubectl get networkpolicies -n findly-now
```

**Install Essential Add-ons**:
```bash
# Install ingress controller
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=LoadBalancer \
    --set controller.service.loadBalancerIP=STATIC_IP_ADDRESS

# Install cert-manager for SSL certificates
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true

# Create ClusterIssuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@findlynow.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Monitoring Stack Deployment

### 1. Prometheus and Grafana Setup

**Install Monitoring Stack**:
```bash
# Add monitoring Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --values helm/monitoring/prometheus-values.yaml \
    --wait

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

**Prometheus Values Configuration**:
```yaml
# helm/monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

    additionalScrapeConfigs:
    - job_name: 'findly-services'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [findly-now]
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

grafana:
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: fast-ssd

  adminPassword: "secure-admin-password"

  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'findly-dashboards'
        orgId: 1
        folder: 'Findly Now'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/findly

  dashboards:
    findly:
      findly-overview:
        gnetId: 1860
        revision: 27
        datasource: Prometheus

alertmanager:
  config:
    global:
      slack_api_url: 'SLACK_WEBHOOK_URL'
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
    receivers:
    - name: 'web.hook'
      slack_configs:
      - channel: '#alerts'
        title: 'Findly Now Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

### 2. Logging Stack Setup

**Install Fluent Bit for Log Aggregation**:
```bash
# Add Fluent Bit Helm repository
helm repo add fluent https://fluent.github.io/helm-charts

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
    --namespace logging \
    --create-namespace \
    --values helm/logging/fluent-bit-values.yaml
```

**Fluent Bit Configuration**:
```yaml
# helm/logging/fluent-bit-values.yaml
config:
  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

  outputs: |
    [OUTPUT]
        Name  stackdriver
        Match kube.*
        google_service_credentials /tmp/credentials.json
        export_to_project_id findly-now-production
        resource  k8s_container
        k8s_cluster_name findly-production
        k8s_cluster_location us-central1
```

## Application Deployment

### 1. Helm Chart Installation

**Deploy Main Application**:
```bash
# Install/upgrade the main application
helm upgrade --install findly-app ./helm/findly-app \
    --namespace findly-now \
    --create-namespace \
    --values helm/findly-app/values-production.yaml \
    --wait \
    --timeout=10m

# Verify deployment
kubectl get pods -n findly-now
kubectl get svc -n findly-now
kubectl get ingress -n findly-now
```

**Production Values Override**:
```yaml
# helm/findly-app/values-production.yaml
global:
  environment: production
  imageRegistry: gcr.io/findly-now-production

services:
  fn-posts:
    replicas: 5
    image:
      tag: "1.2.0"
    resources:
      requests:
        memory: "1Gi"
        cpu: "1000m"
      limits:
        memory: "2Gi"
        cpu: "2000m"
    autoscaling:
      enabled: true
      minReplicas: 5
      maxReplicas: 20

  fn-notifications:
    replicas: 3
    image:
      tag: "1.1.0"
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
  - host: api.findlynow.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: findly-tls-cert
    hosts:
    - api.findlynow.com

monitoring:
  serviceMonitor:
    enabled: true
    interval: 30s
    path: /metrics

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
```

### 2. Database Setup

**PostgreSQL with PostGIS**:
```bash
# Install PostgreSQL operator
kubectl apply -f https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Create PostgreSQL cluster
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: findly-now
spec:
  instances: 3
  postgresql:
    parameters:
      shared_preload_libraries: "postgis"
  bootstrap:
    initdb:
      database: findly_production
      owner: findly_user
      secret:
        name: postgres-credentials
  storage:
    size: 100Gi
    storageClass: fast-ssd
  monitoring:
    enabled: true
EOF
```

## CI/CD Pipeline Setup

### 1. GitHub Actions Configuration

**Setup Repository Secrets**:
```bash
# Add secrets to GitHub repository
gh secret set GCP_PROJECT_ID --body "findly-now-production"
gh secret set GCP_SA_KEY --body "$(cat terraform-key.json | base64)"
gh secret set KUBE_CONFIG --body "$(cat ~/.kube/config | base64)"
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/..."
```

**Environment Protection Rules**:
```yaml
# .github/environments/production.yml
name: production
protection_rules:
  required_reviewers:
    users: ["devops-lead", "security-lead"]
  prevent_self_review: true
  required_status_checks:
    strict: true
    contexts: ["security-scan", "integration-tests"]
deployment_branch_policy:
  protected_branches: true
  custom_branch_policies: false
```

### 2. Automated Deployment Pipeline

**Service Deployment Workflow**:
```yaml
# .github/workflows/deploy-service.yml
name: Deploy Service

on:
  workflow_call:
    inputs:
      service:
        required: true
        type: string
      environment:
        required: true
        type: string
      version:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup GCloud
      uses: google-github-actions/setup-gcloud@v1
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ secrets.GCP_PROJECT_ID }}

    - name: Configure kubectl
      run: |
        gcloud container clusters get-credentials findly-${{ inputs.environment }} \
          --region us-central1

    - name: Deploy with Helm
      run: |
        helm upgrade --install findly-app ./helm/findly-app \
          --namespace findly-now \
          --set services.${{ inputs.service }}.image.tag=${{ inputs.version }} \
          --set global.environment=${{ inputs.environment }} \
          --wait \
          --timeout=600s

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/${{ inputs.service }} -n findly-now
        kubectl wait --for=condition=ready pod \
          -l app=${{ inputs.service }} \
          -n findly-now \
          --timeout=300s

    - name: Run health checks
      run: |
        ./scripts/health-check.sh ${{ inputs.service }} ${{ inputs.environment }}

    - name: Notify success
      if: success()
      uses: 8398a7/action-slack@v3
      with:
        status: success
        channel: '#deployments'
        message: "✅ ${{ inputs.service }} v${{ inputs.version }} deployed to ${{ inputs.environment }}"
```

## Security Configuration

### 1. Pod Security Standards

**Enable Pod Security Standards**:
```bash
# Label namespaces with security standards
kubectl label namespace findly-now \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Apply security context constraints
kubectl apply -f k8s/base/security/pod-security-policy.yaml
```

### 2. Network Security

**Implement Network Policies**:
```yaml
# k8s/base/network-policies/findly-network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: findly-app-network-policy
  namespace: findly-now
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: findly-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: findly-app
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### 3. Secret Management

**Setup External Secrets Operator**:
```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets-system \
    --create-namespace

# Create SecretStore for Google Secret Manager
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcpsm-secret-store
  namespace: findly-now
spec:
  provider:
    gcpsm:
      projectId: findly-now-production
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: findly-production
          serviceAccountRef:
            name: external-secrets-sa
EOF
```

## Backup and Disaster Recovery

### 1. Database Backup Setup

**Automated PostgreSQL Backups**:
```bash
# Create backup CronJob
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: findly-now
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h postgres-cluster-rw -U findly_user findly_production | \
              gzip > /backup/backup-$(date +%Y%m%d-%H%M%S).sql.gz
              gsutil cp /backup/backup-*.sql.gz gs://findly-backups/postgres/
              find /backup -name "*.sql.gz" -mtime +7 -delete
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            emptyDir: {}
          restartPolicy: OnFailure
EOF
```

### 2. Disaster Recovery Testing

**DR Testing Script**:
```bash
#!/bin/bash
# scripts/disaster-recovery-test.sh

set -euo pipefail

echo "🧪 Starting Disaster Recovery Test..."

# 1. Create test namespace
kubectl create namespace dr-test-$(date +%s)
TEST_NAMESPACE="dr-test-$(date +%s)"

# 2. Deploy application to test namespace
helm install findly-app-dr ./helm/findly-app \
    --namespace $TEST_NAMESPACE \
    --set global.environment=dr-test \
    --wait

# 3. Restore database from backup
LATEST_BACKUP=$(gsutil ls gs://findly-backups/postgres/ | tail -1)
gsutil cp $LATEST_BACKUP /tmp/latest-backup.sql.gz
gunzip /tmp/latest-backup.sql.gz

# Create test database
kubectl exec -n $TEST_NAMESPACE postgres-0 -- \
    createdb -U findly_user findly_dr_test

# Restore data
kubectl exec -i -n $TEST_NAMESPACE postgres-0 -- \
    psql -U findly_user findly_dr_test < /tmp/latest-backup.sql

# 4. Run smoke tests
./scripts/smoke-tests.sh dr-test $TEST_NAMESPACE

# 5. Cleanup
kubectl delete namespace $TEST_NAMESPACE
rm /tmp/latest-backup.sql

echo "✅ Disaster Recovery Test Completed"
```

## Monitoring and Alerting Setup

### 1. Custom Metrics Configuration

**Application Metrics**:
```yaml
# monitoring/prometheus/findly-rules.yaml
groups:
- name: findly-now.rules
  rules:
  - alert: ServiceDown
    expr: up{job="findly-services"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Service {{ $labels.instance }} is down"
      description: "{{ $labels.instance }} has been down for more than 1 minute"

  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High error rate on {{ $labels.instance }}"
      description: "Error rate is {{ $value | humanizePercentage }}"

  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High latency on {{ $labels.instance }}"
      description: "95th percentile latency is {{ $value }}s"
```

### 2. Dashboard Configuration

**Grafana Dashboard Provisioning**:
```json
{
  "dashboard": {
    "title": "Findly Now Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service)",
            "legendFormat": "{{ service }} errors"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "{{ service }} p95"
          }
        ]
      }
    ]
  }
}
```

## Troubleshooting Guide

### Common Issues

**Pod Startup Issues**:
```bash
# Check pod status
kubectl get pods -n findly-now

# Check pod logs
kubectl logs -f deployment/fn-posts -n findly-now

# Describe pod for events
kubectl describe pod <pod-name> -n findly-now

# Check resource constraints
kubectl top pods -n findly-now
```

**Networking Issues**:
```bash
# Test service connectivity
kubectl exec -it <pod-name> -n findly-now -- curl http://service-name:port/health

# Check network policies
kubectl get networkpolicies -n findly-now

# Check ingress configuration
kubectl describe ingress -n findly-now
```

**Resource Issues**:
```bash
# Check node resources
kubectl top nodes

# Check resource quotas
kubectl describe resourcequota -n findly-now

# Check persistent volume claims
kubectl get pvc -n findly-now
```

### Emergency Procedures

**Emergency Rollback**:
```bash
# Rollback all services
for service in fn-posts fn-notifications fn-matcher fn-media-ai; do
  kubectl rollout undo deployment/$service -n findly-now
done

# Monitor rollback status
kubectl rollout status deployment/fn-posts -n findly-now
```

**Emergency Scale Down**:
```bash
# Scale down to minimum replicas
kubectl scale deployment fn-posts --replicas=1 -n findly-now
kubectl scale deployment fn-notifications --replicas=1 -n findly-now
```

## Performance Optimization

### Resource Tuning

**Cluster Autoscaler Configuration**:
```yaml
# cluster-autoscaler-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status-config
  namespace: kube-system
data:
  nodes.max: "50"
  cores.max: "500"
  memory.max: "2000Gi"
  scale-down-delay-after-add: "10m"
  scale-down-unneeded-time: "10m"
  skip-nodes-with-local-storage: "false"
```

**Vertical Pod Autoscaler**:
```bash
# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-install.sh

# Apply VPA to services
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: fn-posts-vpa
  namespace: findly-now
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fn-posts
  updatePolicy:
    updateMode: "Auto"
EOF
```

---

*This deployment guide provides comprehensive instructions for setting up the complete Findly Now infrastructure. For architectural details, see [domain-architecture.md](domain-architecture.md). For API specifications, see [api-documentation.md](api-documentation.md).*