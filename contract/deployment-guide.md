# fn-contract Deployment Guide

**Complete deployment guide for the Contract domain service schema governance and validation across all environments.**

## Prerequisites

### Infrastructure Requirements
- **Git Repository**: GitHub repository with branch protection
- **Confluent Schema Registry**: Managed schema storage
- **CI/CD Pipeline**: GitHub Actions or similar
- **Documentation Hosting**: S3 + CloudFront or similar
- **Package Registry**: npm registry for CLI tools

### Required Tools
- **Node.js** 18+ for validation tools
- **Java** 11+ for Avro tools
- **Docker** for containerized validation
- **kubectl** for Kubernetes deployments (if applicable)

### Platform Compatibility
- **Apple Silicon Support**: Validation tools work natively on both Intel and Apple Silicon Macs
- **Docker Integration**: Uses `platform: linux/amd64` when needed for tool compatibility
- **See**: [Platform Compatibility Guide](../CLOUD-SETUP.md#platform-compatibility)

## Repository Structure Setup

```bash
# Initialize fn-contract repository
git clone https://github.com/findly-now/fn-contract.git
cd fn-contract

# Install dependencies
npm install

# Setup directory structure
mkdir -p {api,events,avro}/{posts,matcher,media-ai,notifications}
mkdir -p {docs,tools,tests,scripts}
```

## Schema Registry Configuration

### Confluent Cloud Setup

```bash
# Install Confluent CLI
curl -sL --http1.1 https://cnfl.io/cli | sh -s -- latest

# Configure Schema Registry
confluent schema-registry cluster create \
  --cloud gcp \
  --geo us \
  --environment $ENVIRONMENT_ID

# Set compatibility levels
confluent schema-registry compatibility set \
  --level BACKWARD \
  --subject posts.created

confluent schema-registry compatibility set \
  --level FORWARD \
  --subject media-ai.enhanced
```

### Schema Publishing Pipeline

```yaml
# .github/workflows/schema-publish.yml
name: Schema Publishing
on:
  push:
    branches: [main]
    paths: ['avro/**']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Publish Schemas
      env:
        SCHEMA_REGISTRY_URL: ${{ secrets.SCHEMA_REGISTRY_URL }}
        SCHEMA_REGISTRY_KEY: ${{ secrets.SCHEMA_REGISTRY_KEY }}
        SCHEMA_REGISTRY_SECRET: ${{ secrets.SCHEMA_REGISTRY_SECRET }}
      run: |
        npm run publish-schemas
```

## CI/CD Pipeline Setup

### Validation Workflow

```yaml
# .github/workflows/validate.yml
name: Contract Validation
on:
  pull_request:
    paths: ['api/**', 'events/**', 'avro/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Validate schemas
      run: npm run validate

    - name: Check compatibility
      run: npm run compatibility-check

    - name: Generate impact report
      run: npm run impact-analysis

    - name: Upload reports
      uses: actions/upload-artifact@v3
      with:
        name: validation-reports
        path: reports/
```

## CLI Tools Deployment

### Package Publishing

```bash
# Build and publish contract validator
cd tools/contract-validator
npm version patch
npm publish --access public

# Install globally
npm install -g @findly/contract-validator
```

### Usage in Services

```bash
# Add to service package.json
npm install --save-dev @findly/contract-validator

# Pre-commit hook
npx contract-validator validate api/posts/v1.yaml
```

## Documentation Generation

### API Documentation

```bash
# Generate OpenAPI docs
npx redoc-cli build api/posts/v1.yaml --output docs/posts-api.html

# Generate AsyncAPI docs
npx @asyncapi/generator events/posts-events/v1.yaml @asyncapi/html-template \
  --output docs/events/

# Deploy to S3
aws s3 sync docs/ s3://findly-contracts-docs/
```

## Monitoring Setup

### Schema Usage Metrics

```yaml
# monitoring/dashboard.json
{
  "dashboard": {
    "title": "Contract Management",
    "panels": [
      {
        "title": "Schema Validation Rate",
        "type": "stat",
        "targets": ["schema_validations_success_rate"]
      },
      {
        "title": "Breaking Changes",
        "type": "graph",
        "targets": ["breaking_changes_detected"]
      }
    ]
  }
}
```

## Security Configuration

### Access Controls

```yaml
# schema-registry-rbac.yml
subjects:
  posts.*:
    permissions:
      - principals: ["fn-posts-service"]
        operations: ["WRITE"]
      - principals: ["fn-notifications-service", "fn-matcher-service"]
        operations: ["READ"]

  matcher.*:
    permissions:
      - principals: ["fn-matcher-service"]
        operations: ["WRITE"]
      - principals: ["fn-posts-service", "fn-notifications-service"]
        operations: ["READ"]
```

---

*For complete contract management and schema governance. See [domain-architecture.md](domain-architecture.md) for design decisions and [api-documentation.md](api-documentation.md) for detailed specifications.*