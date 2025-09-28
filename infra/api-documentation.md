# fn-infra API Documentation

**Infrastructure automation APIs, deployment scripts, and DevOps tooling for managing the Findly Now cloud infrastructure.**

## Overview

fn-infra provides infrastructure as code (IaC) and DevOps automation rather than traditional REST APIs. The "API" consists of:

1. **Terraform APIs**: Infrastructure provisioning interfaces
2. **Kubernetes APIs**: Container orchestration interfaces
3. **Helm APIs**: Application packaging and deployment
4. **CI/CD APIs**: GitHub Actions and deployment automation
5. **Monitoring APIs**: Prometheus, Grafana, and alerting

## Repository Structure

### Infrastructure as Code Organization

```
fn-infra/
├── terraform/                     # Infrastructure provisioning
│   ├── modules/
│   │   ├── gke-cluster/           # GKE cluster module
│   │   ├── networking/            # VPC and networking
│   │   ├── monitoring/            # Prometheus/Grafana setup
│   │   └── security/              # IAM and security policies
│   └── environments/
│       ├── dev/                   # Development environment
│       ├── staging/               # Staging environment
│       └── production/            # Production environment
├── k8s/                          # Kubernetes manifests
│   ├── base/                     # Base configurations
│   │   ├── namespaces/
│   │   ├── rbac/
│   │   └── network-policies/
│   └── overlays/                 # Environment-specific overrides
│       ├── dev/
│       ├── staging/
│       └── production/
├── helm/                         # Helm charts
│   ├── findly-app/              # Main application chart
│   ├── monitoring/              # Monitoring stack
│   ├── security/                # Security tools
│   └── ingress/                 # Ingress controllers
├── scripts/                     # Automation scripts
│   ├── deploy.sh               # Deployment automation
│   ├── rollback.sh             # Rollback procedures
│   ├── backup.sh               # Backup automation
│   └── disaster-recovery.sh    # DR procedures
└── .github/workflows/          # CI/CD pipelines
    ├── terraform-plan.yml
    ├── terraform-apply.yml
    ├── deploy-dev.yml
    ├── deploy-staging.yml
    └── deploy-production.yml
```

## Terraform Infrastructure APIs

### Cluster Management Module

**GKE Cluster Configuration**:
```hcl
# terraform/modules/gke-cluster/main.tf
module "gke_cluster" {
  source = "./modules/gke-cluster"

  cluster_name = var.cluster_name
  region       = var.region
  environment  = var.environment

  # Node pool configuration
  node_pools = {
    application_pool = {
      machine_type = "e2-standard-4"
      disk_size    = 100
      disk_type    = "pd-ssd"
      min_nodes    = 3
      max_nodes    = 10
      auto_scaling = true
      zones        = ["us-central1-a", "us-central1-b", "us-central1-c"]
    }

    monitoring_pool = {
      machine_type = "e2-standard-2"
      disk_size    = 50
      min_nodes    = 2
      max_nodes    = 4
      taints = [{
        key    = "monitoring"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  # Security configuration
  enable_network_policy = true
  enable_pod_security_policy = true
  master_authorized_networks = var.authorized_networks

  # Monitoring and logging
  enable_monitoring = true
  enable_logging    = true
  log_config = {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
}
```

**Networking Module**:
```hcl
# terraform/modules/networking/main.tf
module "vpc_network" {
  source = "./modules/networking"

  network_name = "${var.environment}-vpc"
  region       = var.region

  # Subnet configuration
  subnets = {
    private_subnet = {
      cidr_range = "10.0.0.0/24"
      purpose    = "PRIVATE_RFC_1918"
    }

    public_subnet = {
      cidr_range = "10.0.1.0/24"
      purpose    = "PRIVATE_RFC_1918"
    }
  }

  # Security configuration
  firewall_rules = {
    allow_internal = {
      direction = "INGRESS"
      priority  = 1000
      source_ranges = ["10.0.0.0/16"]
      allowed = [{
        protocol = "tcp"
        ports    = ["1-65535"]
      }]
    }

    allow_health_checks = {
      direction = "INGRESS"
      priority  = 1000
      source_ranges = ["130.211.0.0/22", "35.191.0.0/16"]
      allowed = [{
        protocol = "tcp"
        ports    = ["80", "443", "8080"]
      }]
    }
  }

  # NAT gateway for outbound internet access
  enable_nat_gateway = true
  nat_config = {
    source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
    log_config = {
      enable = true
      filter = "ERRORS_ONLY"
    }
  }
}
```

### Environment-Specific Configurations

**Production Environment**:
```hcl
# terraform/environments/production/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }

  backend "gcs" {
    bucket = "findly-terraform-state-prod"
    prefix = "infrastructure/production"
  }
}

module "production_cluster" {
  source = "../../modules/gke-cluster"

  cluster_name = "findly-production"
  region       = "us-central1"
  environment  = "production"

  # Production-specific overrides
  node_pools = {
    application_pool = {
      machine_type = "e2-standard-8"  # Larger instances for production
      disk_size    = 200
      min_nodes    = 5                # Higher minimum for availability
      max_nodes    = 50               # Higher scaling limit
    }
  }

  # Enhanced security for production
  master_authorized_networks = [
    {
      cidr_block   = "10.0.0.0/8"
      display_name = "internal-network"
    }
  ]

  # Backup and disaster recovery
  backup_config = {
    enabled                = true
    backup_retain_days     = 30
    point_in_time_recovery = true
  }
}

# Production monitoring configuration
module "production_monitoring" {
  source = "../../modules/monitoring"

  cluster_name = module.production_cluster.cluster_name
  environment  = "production"

  # Enhanced monitoring for production
  monitoring_config = {
    prometheus_retention = "30d"
    grafana_persistence = true
    alertmanager_config = {
      slack_webhook_url = var.slack_webhook_url
      pagerduty_service_key = var.pagerduty_service_key
    }
  }
}
```

## Kubernetes Deployment APIs

### Application Deployment Manifests

**Service Deployment Template**:
```yaml
# k8s/base/deployment-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: SERVICE_NAME
  namespace: findly-now
  labels:
    app: SERVICE_NAME
    version: VERSION
spec:
  replicas: REPLICA_COUNT
  selector:
    matchLabels:
      app: SERVICE_NAME
  template:
    metadata:
      labels:
        app: SERVICE_NAME
        version: VERSION
    spec:
      serviceAccountName: SERVICE_NAME-sa
      containers:
      - name: SERVICE_NAME
        image: gcr.io/findly-now/SERVICE_NAME:VERSION
        ports:
        - containerPort: SERVICE_PORT
        env:
        - name: ENVIRONMENT
          value: ENVIRONMENT_NAME
        - name: LOG_LEVEL
          value: LOG_LEVEL
        resources:
          requests:
            memory: MEMORY_REQUEST
            cpu: CPU_REQUEST
          limits:
            memory: MEMORY_LIMIT
            cpu: CPU_LIMIT
        livenessProbe:
          httpGet:
            path: /health/live
            port: SERVICE_PORT
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: SERVICE_PORT
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /app/config
        - name: secrets
          mountPath: /app/secrets
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: SERVICE_NAME-config
      - name: secrets
        secret:
          secretName: SERVICE_NAME-secrets
```

**Horizontal Pod Autoscaler**:
```yaml
# k8s/base/hpa-template.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: SERVICE_NAME-hpa
  namespace: findly-now
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: SERVICE_NAME
  minReplicas: MIN_REPLICAS
  maxReplicas: MAX_REPLICAS
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: CPU_TARGET
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: MEMORY_TARGET
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: REQUEST_TARGET
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Environment Overlays

**Production Overlay**:
```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: findly-now

resources:
- ../../base

patchesStrategicMerge:
- deployment-patches.yaml
- hpa-patches.yaml

configMapGenerator:
- name: environment-config
  literals:
  - ENVIRONMENT=production
  - LOG_LEVEL=info
  - METRICS_ENABLED=true

secretGenerator:
- name: production-secrets
  envs:
  - secrets.env

images:
- name: SERVICE_NAME
  newTag: PRODUCTION_TAG

replicas:
- name: fn-posts
  count: 5
- name: fn-notifications
  count: 3
- name: fn-matcher
  count: 3
- name: fn-media-ai
  count: 2
```

## Helm Chart APIs

### Main Application Chart

**Chart Structure**:
```yaml
# helm/findly-app/Chart.yaml
apiVersion: v2
name: findly-app
description: Findly Now microservices application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
- name: postgresql
  version: 12.x.x
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled

- name: redis
  version: 17.x.x
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled

- name: prometheus
  version: 15.x.x
  repository: https://prometheus-community.github.io/helm-charts
  condition: monitoring.prometheus.enabled

- name: grafana
  version: 6.x.x
  repository: https://grafana.github.io/helm-charts
  condition: monitoring.grafana.enabled
```

**Values Configuration**:
```yaml
# helm/findly-app/values.yaml
global:
  environment: production
  imageRegistry: gcr.io/findly-now
  imagePullPolicy: IfNotPresent

  # Global security settings
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000

services:
  fn-posts:
    enabled: true
    image:
      repository: fn-posts
      tag: "1.2.0"
    replicas: 5
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
    autoscaling:
      enabled: true
      minReplicas: 3
      maxReplicas: 15
      targetCPUUtilizationPercentage: 70

  fn-notifications:
    enabled: true
    image:
      repository: fn-notifications
      tag: "1.1.0"
    replicas: 3
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"

  fn-matcher:
    enabled: true
    image:
      repository: fn-matcher
      tag: "1.0.0"
    replicas: 3
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"

  fn-media-ai:
    enabled: true
    image:
      repository: fn-media-ai
      tag: "0.1.0"
    replicas: 2
    resources:
      requests:
        memory: "1Gi"
        cpu: "1000m"
      limits:
        memory: "4Gi"
        cpu: "4000m"

ingress:
  enabled: true
  className: "gce"
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "findly-global-ip"
    kubernetes.io/ingress.allow-http: "false"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
  - host: api.findlynow.com
    paths:
    - path: /api/posts
      pathType: Prefix
      service: fn-posts
    - path: /api/notifications
      pathType: Prefix
      service: fn-notifications
    - path: /api/matches
      pathType: Prefix
      service: fn-matcher
  tls:
  - secretName: findly-tls-cert
    hosts:
    - api.findlynow.com

monitoring:
  prometheus:
    enabled: true
    persistence:
      enabled: true
      size: 50Gi
    retention: 30d

  grafana:
    enabled: true
    persistence:
      enabled: true
      size: 10Gi
    adminPassword: "secure-admin-password"
    dashboards:
      enabled: true
      configMaps:
        findly-dashboards: findly-dashboards

security:
  networkPolicies:
    enabled: true
  podSecurityPolicies:
    enabled: true
  rbac:
    enabled: true
```

## CI/CD Pipeline APIs

### GitHub Actions Workflows

**Main Deployment Pipeline**:
```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to deploy (or "all" for full deployment)'
        required: true
        default: 'all'
        type: choice
        options: ['all', 'fn-posts', 'fn-notifications', 'fn-matcher', 'fn-media-ai']
      version:
        description: 'Version to deploy'
        required: true
        type: string

env:
  GCP_PROJECT_ID: findly-now-production
  GKE_CLUSTER: findly-production
  GKE_REGION: us-central1

jobs:
  pre-deployment-checks:
    runs-on: ubuntu-latest
    outputs:
      deployment_approved: ${{ steps.approval.outputs.approved }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Validate deployment inputs
      run: |
        echo "Service: ${{ github.event.inputs.service }}"
        echo "Version: ${{ github.event.inputs.version }}"

        # Validate version format (semantic versioning)
        if [[ ! "${{ github.event.inputs.version }}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
          echo "Invalid version format. Use semantic versioning (x.y.z)"
          exit 1
        fi

    - name: Check staging deployment status
      run: |
        # Verify that the version is deployed and stable in staging
        kubectl config set-context staging
        kubectl get deployment ${{ github.event.inputs.service }} -o jsonpath='{.status.readyReplicas}'

    - name: Security scan
      uses: securecodewarrior/github-action-vulnerability-scan@v1
      with:
        image: gcr.io/${{ env.GCP_PROJECT_ID }}/${{ github.event.inputs.service }}:${{ github.event.inputs.version }}

    - name: Request deployment approval
      id: approval
      uses: trstringer/manual-approval@v1
      with:
        secret: ${{ github.TOKEN }}
        approvers: devops-team,security-team
        minimum-approvals: 2
        issue-title: "Production Deployment Approval Required"
        issue-body: |
          **Service**: ${{ github.event.inputs.service }}
          **Version**: ${{ github.event.inputs.version }}

          Please review and approve this production deployment.

          - [ ] Security scan passed
          - [ ] Staging validation completed
          - [ ] Rollback plan reviewed

  deploy:
    needs: pre-deployment-checks
    if: needs.pre-deployment-checks.outputs.deployment_approved == 'true'
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Google Cloud
      uses: google-github-actions/setup-gcloud@v1
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ env.GCP_PROJECT_ID }}

    - name: Configure kubectl
      run: |
        gcloud container clusters get-credentials ${{ env.GKE_CLUSTER }} \
          --region ${{ env.GKE_REGION }} \
          --project ${{ env.GCP_PROJECT_ID }}

    - name: Deploy with Helm
      run: |
        helm upgrade --install findly-app ./helm/findly-app \
          --namespace findly-now \
          --set services.${{ github.event.inputs.service }}.image.tag=${{ github.event.inputs.version }} \
          --set global.environment=production \
          --wait \
          --timeout=600s

    - name: Verify deployment
      run: |
        # Wait for rollout to complete
        kubectl rollout status deployment/${{ github.event.inputs.service }} -n findly-now

        # Run health checks
        kubectl wait --for=condition=ready pod \
          -l app=${{ github.event.inputs.service }} \
          -n findly-now \
          --timeout=300s

    - name: Run smoke tests
      run: |
        # Run critical path tests against production
        ./scripts/smoke-tests.sh ${{ github.event.inputs.service }}

    - name: Notify deployment success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        channel: '#deployments'
        message: |
          ✅ Production deployment successful!
          **Service**: ${{ github.event.inputs.service }}
          **Version**: ${{ github.event.inputs.version }}
          **Deployed by**: ${{ github.actor }}

  rollback:
    needs: deploy
    if: failure()
    runs-on: ubuntu-latest
    steps:
    - name: Emergency rollback
      run: |
        # Automatic rollback on deployment failure
        kubectl rollout undo deployment/${{ github.event.inputs.service }} -n findly-now

        # Wait for rollback to complete
        kubectl rollout status deployment/${{ github.event.inputs.service }} -n findly-now

    - name: Notify rollback
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        channel: '#deployments'
        message: |
          🚨 Production deployment failed and was rolled back!
          **Service**: ${{ github.event.inputs.service }}
          **Failed version**: ${{ github.event.inputs.version }}
          **Action**: Automatic rollback executed
```

### Infrastructure Pipeline

**Terraform Apply Workflow**:
```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths: ['terraform/**']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to apply changes'
        required: true
        default: 'dev'
        type: choice
        options: ['dev', 'staging', 'production']

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment || 'dev' }}
    defaults:
      run:
        working-directory: terraform/environments/${{ github.event.inputs.environment || 'dev' }}

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0

    - name: Setup Google Cloud
      uses: google-github-actions/setup-gcloud@v1
      with:
        service_account_key: ${{ secrets.GCP_SA_KEY }}
        project_id: ${{ secrets.GCP_PROJECT_ID }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: |
        terraform plan -out=tfplan
        terraform show -json tfplan > plan.json

    - name: Security scan for Terraform
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'config'
        scan-ref: 'terraform/'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply tfplan

    - name: Update cluster credentials
      if: github.ref == 'refs/heads/main'
      run: |
        # Update kubeconfig for the newly created/updated cluster
        gcloud container clusters get-credentials \
          $(terraform output -raw cluster_name) \
          --region $(terraform output -raw cluster_region)
```

## Monitoring APIs

### Prometheus Configuration

**Service Discovery and Scraping**:
```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # Kubernetes API server
  - job_name: 'kubernetes-apiserver'
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https

  # Application services
  - job_name: 'findly-services'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod_name

  # Node exporter
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
    - role: node
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: kubernetes.default.svc:443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics
```

### AlertManager Configuration

**Alert Routing and Notification**:
```yaml
# monitoring/alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@findlynow.com'
  slack_api_url: '$SLACK_API_URL'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty'
    group_wait: 10s
    repeat_interval: 5m

  - match:
      severity: warning
    receiver: 'slack'
    group_wait: 30s
    repeat_interval: 30m

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://webhook-service:5001/'

- name: 'pagerduty'
  pagerduty_configs:
  - service_key: '$PAGERDUTY_SERVICE_KEY'
    description: "{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}"

- name: 'slack'
  slack_configs:
  - channel: '#alerts'
    title: 'Findly Now Alert'
    text: "{{ range .Alerts }}{{ .Annotations.description }}{{ end }}"
    send_resolved: true

inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  equal: ['alertname', 'cluster', 'service']
```

## Deployment Scripts

### Automated Deployment Script

**Master Deployment Script**:
```bash
#!/bin/bash
# scripts/deploy.sh

set -euo pipefail

# Configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

# Default values
ENVIRONMENT="dev"
SERVICE="all"
VERSION=""
DRY_RUN=false
ROLLBACK=false

# Parse command line arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    -e|--environment)
      ENVIRONMENT="$2"
      shift 2
      ;;
    -s|--service)
      SERVICE="$2"
      shift 2
      ;;
    -v|--version)
      VERSION="$2"
      shift 2
      ;;
    --dry-run)
      DRY_RUN=true
      shift
      ;;
    --rollback)
      ROLLBACK=true
      shift
      ;;
    -h|--help)
      echo "Usage: $0 [OPTIONS]"
      echo "Options:"
      echo "  -e, --environment    Target environment (dev|staging|production)"
      echo "  -s, --service        Service name or 'all' for all services"
      echo "  -v, --version        Version to deploy"
      echo "  --dry-run           Show what would be deployed without applying"
      echo "  --rollback          Rollback to previous version"
      echo "  -h, --help          Show this help message"
      exit 0
      ;;
    *)
      echo "Unknown option $1"
      exit 1
      ;;
  esac
done

# Validation
if [[ "$ENVIRONMENT" != "dev" && "$ENVIRONMENT" != "staging" && "$ENVIRONMENT" != "production" ]]; then
  echo "Error: Invalid environment. Must be dev, staging, or production."
  exit 1
fi

if [[ -z "$VERSION" && "$ROLLBACK" == false ]]; then
  echo "Error: Version is required for deployment."
  exit 1
fi

# Setup kubectl context
setup_kubectl() {
  local env=$1
  echo "Setting up kubectl for $env environment..."

  case $env in
    dev)
      gcloud container clusters get-credentials findly-dev --region us-central1
      ;;
    staging)
      gcloud container clusters get-credentials findly-staging --region us-central1
      ;;
    production)
      gcloud container clusters get-credentials findly-production --region us-central1
      ;;
  esac
}

# Deploy service
deploy_service() {
  local service=$1
  local version=$2
  local env=$3

  echo "Deploying $service version $version to $env..."

  if [[ "$DRY_RUN" == true ]]; then
    echo "[DRY RUN] Would deploy $service:$version to $env"
    helm template findly-app "$PROJECT_ROOT/helm/findly-app" \
      --set services.$service.image.tag=$version \
      --set global.environment=$env \
      --namespace findly-now
    return 0
  fi

  # Actual deployment
  helm upgrade --install findly-app "$PROJECT_ROOT/helm/findly-app" \
    --namespace findly-now \
    --create-namespace \
    --set services.$service.image.tag=$version \
    --set global.environment=$env \
    --wait \
    --timeout=600s

  # Verify deployment
  echo "Verifying deployment..."
  kubectl rollout status deployment/$service -n findly-now --timeout=300s

  # Health check
  echo "Running health checks..."
  kubectl wait --for=condition=ready pod \
    -l app=$service \
    -n findly-now \
    --timeout=300s

  echo "✅ $service deployed successfully!"
}

# Rollback service
rollback_service() {
  local service=$1
  local env=$2

  echo "Rolling back $service in $env..."

  if [[ "$DRY_RUN" == true ]]; then
    echo "[DRY RUN] Would rollback $service in $env"
    return 0
  fi

  kubectl rollout undo deployment/$service -n findly-now
  kubectl rollout status deployment/$service -n findly-now --timeout=300s

  echo "✅ $service rolled back successfully!"
}

# Main execution
main() {
  echo "🚀 Starting deployment process..."
  echo "Environment: $ENVIRONMENT"
  echo "Service: $SERVICE"
  echo "Version: $VERSION"
  echo "Dry run: $DRY_RUN"
  echo "Rollback: $ROLLBACK"
  echo

  # Setup kubectl
  setup_kubectl "$ENVIRONMENT"

  # Production safety check
  if [[ "$ENVIRONMENT" == "production" && "$DRY_RUN" == false ]]; then
    echo "⚠️  This is a PRODUCTION deployment!"
    read -p "Are you sure you want to continue? (yes/no): " confirm
    if [[ "$confirm" != "yes" ]]; then
      echo "Deployment cancelled."
      exit 0
    fi
  fi

  # Deploy or rollback
  if [[ "$ROLLBACK" == true ]]; then
    if [[ "$SERVICE" == "all" ]]; then
      for svc in fn-posts fn-notifications fn-matcher fn-media-ai; do
        rollback_service "$svc" "$ENVIRONMENT"
      done
    else
      rollback_service "$SERVICE" "$ENVIRONMENT"
    fi
  else
    if [[ "$SERVICE" == "all" ]]; then
      for svc in fn-posts fn-notifications fn-matcher fn-media-ai; do
        deploy_service "$svc" "$VERSION" "$ENVIRONMENT"
      done
    else
      deploy_service "$SERVICE" "$VERSION" "$ENVIRONMENT"
    fi
  fi

  echo "🎉 Deployment process completed!"
}

# Run main function
main "$@"
```

## Security APIs

### RBAC Configuration

**Service Account and Role Bindings**:
```yaml
# k8s/base/rbac/service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fn-posts-sa
  namespace: findly-now
  annotations:
    iam.gke.io/gcp-service-account: fn-posts@findly-now.iam.gserviceaccount.com

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: fn-posts-role
  namespace: findly-now
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: fn-posts-rolebinding
  namespace: findly-now
subjects:
- kind: ServiceAccount
  name: fn-posts-sa
  namespace: findly-now
roleRef:
  kind: Role
  name: fn-posts-role
  apiGroup: rbac.authorization.k8s.io
```

### Network Policies

**Security Network Segmentation**:
```yaml
# k8s/base/network-policies/default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: findly-now
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-namespace-communication
  namespace: findly-now
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: findly-now
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: findly-now
  - to: []  # Allow egress to external services
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
    - protocol: UDP
      port: 53
```

---

*This infrastructure API documentation provides comprehensive guidance for managing cloud infrastructure, deployments, and DevOps automation. For architectural details, see [domain-architecture.md](domain-architecture.md). For deployment procedures, see [deployment-guide.md](deployment-guide.md).*