# Helm Charts Mastery - Unified Metering Service Analysis

## Table of Contents
1. [Helm Overview](#helm-overview)
2. [Repository Structure Analysis](#repository-structure-analysis)
3. [Chart Components Deep Dive](#chart-components-deep-dive)
4. [Dependency Management](#dependency-management)
5. [Values Architecture](#values-architecture)
6. [Template Analysis](#template-analysis)
7. [Deployment Strategies](#deployment-strategies)
8. [Monitoring & Observability](#monitoring--observability)
9. [Security Implementation](#security-implementation)
10. [Interview Questions & Answers](#interview-questions--answers)
11. [Practical Commands](#practical-commands)
12. [Best Practices](#best-practices)

---

## Helm Overview

### What is Helm?
Helm is a **Kubernetes Package and Deployment Manager** that provides:
- **Package Management**: Like apt or yum in Linux world
- **Automated Version Handling**: During upgrade/rollback
- **Template System**: Share deployment instructions as scripts
- **Repository Management**: Store and reuse templates
- **Lifecycle Management**: Complete application lifecycle (Create, Install, Upgrade, Rollback, Delete, Status, Versioning)

### Benefits
- **Repeatability**: Same template across environments
- **Reliability**: Automated deployment processes
- **Multi-Environment**: Manage complex environments easily
- **Collaboration**: Everything defined in files
- **Version Control**: Track changes and rollback capability

---

## Repository Structure Analysis

### Project Architecture
```
unified-metering-service-deploy/
├── argo/                          # CI/CD Workflow Charts
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── k8s/helm/                      # Application Deployment Charts
│   ├── values.yaml                # Global values
│   ├── Stage/
│   │   ├── values.yaml            # Stage environment values
│   │   └── va6/
│   │       ├── Chart.yaml         # Stage region chart
│   │       └── values.yaml        # Stage region values
│   └── Prod/
│       ├── values.yaml            # Prod environment values
│       └── va6/
│           ├── Chart.yaml         # Prod region chart
│           └── values.yaml        # Prod region values
└── charts/
    └── enm-client/                # Custom monitoring chart
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

### Repository Types
1. **Remote Repositories**: Adobe Artifactory
2. **Local Repositories**: File-based charts
3. **Git Repository**: GitOps deployment

---

## Chart Components Deep Dive

### 1. Chart Component
**Definition**: A Chart is a collection of files that describe a related set of Kubernetes resources.

**Implementation**:
```yaml
# Chart.yaml - Chart metadata
apiVersion: v2
name: Stage-va6
description: A Helm chart for Kubernetes
type: application
version: 0.0.1
appVersion: "0.0.1"
```

### 2. Release Component
**Definition**: A Release is an instance of a chart deployed to a Kubernetes cluster.

**Implementation**: Managed through Argo Workflows
```yaml
# Release lifecycle through Argo
- name: execute-pipeline-stages
  templateRef:
    name: pipeline-stages-all-template
    template: pipeline-stages-all
```

### 3. Repository Component
**Definition**: A Repository is a collection of charts that can be accessed by Helm.

**Implementation**:
```yaml
# External repositories
- name: ethos-k8s-helm-templates
  repository: "https://artifactory-uw2.adobeitc.com/artifactory/helm-ethos-k8s-helm-templates-release"

# Local repositories
- name: enm-client
  repository: "file://../../../../charts/enm-client"
```

### 4. Template Component
**Definition**: Templates are Kubernetes manifest files with Go templating syntax.

**Implementation**:
```yaml
# Template with conditional rendering
{{- if .Values.prometheusRulesConfig.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: '{{ include "enm-client.name" . }}-rules'
{{- end }}
```

### 5. Values Component
**Definition**: Values are configuration parameters that customize chart behavior.

**Implementation**: Hierarchical values structure
```yaml
# Global values
global:
  gitOrg: meterix
  gitRepo: unified-metering-service

# Environment-specific overrides
global:
  clusterType: STAGE
  serviceEnvironment: Stage
```

### 6. Dependencies Component
**Definition**: Dependencies are external charts required by a chart.

**Implementation**:
```yaml
# Aliased dependencies
- name: ethos-k8s-helm-deployment-templates
  alias: cont1-deployment-templates
  version: "3.12.0"

# Conditional dependencies
- name: enm-prom-operator
  condition: enm-prom-operator.enabled
  version: "58.7.2"
```

### 7. Hooks Component
**Definition**: Hooks are special resources that run at specific lifecycle events.

**Implementation**: Argo Workflow hooks
```yaml
- name: exit-handler
  dag:
    tasks:
      - name: "demote"
        when: "{{`{{ workflow.status }}`}} != Succeeded"
```

### 8. Tests Component
**Definition**: Tests are resources that validate chart functionality.

**Implementation**: Health checks and monitoring
```yaml
livenessProbe:
  path: /ping
  port: 8080
  gracePeriod: 30s
  timeout: 5s
```

---

## Dependency Management

### Dependency Categories
1. **Core Infrastructure**: `ethos-k8s-helm-templates`
2. **Application Deployment**: `ethos-k8s-helm-deployment-templates` (aliased)
3. **Monitoring**: `enm-prom-operator`, `ethos-enm-helper-templates`
4. **Networking**: `cgw-flex-templates`
5. **Core Services**: `emo`
6. **Custom Charts**: `enm-client`

### Dependency Strategies
```yaml
# Pattern 1: Aliasing for Multiple Instances
- name: ethos-k8s-helm-deployment-templates
  alias: cont1-deployment-templates    # Main app
- name: ethos-k8s-helm-deployment-templates  
  alias: ums-worker-deployment-templates # Worker

# Pattern 2: Conditional Dependencies
- name: enm-prom-operator
  condition: enm-prom-operator.enabled

# Pattern 3: Version Management
version: "3.4.0"        # Pinned versions
version: "^1.0.0-beta" # Caret ranges
```

---

## Values Architecture

### Hierarchical Values Override Pattern
```yaml
# Level 1: Global Values (k8s/helm/values.yaml)
global:
  gitOrg: meterix
  gitRepo: unified-metering-service

# Level 2: Environment Values (k8s/helm/Stage/values.yaml)
global:
  clusterType: STAGE
  serviceEnvironment: Stage

# Level 3: Region Values (k8s/helm/Stage/va6/values.yaml)
global:
  clusterName: ethos504-stage-va6
  serviceRegion: va6
```

### Values Types
- **Global Values**: Shared across all templates
- **Chart Values**: Specific to chart dependencies
- **Environment Values**: Environment-specific overrides
- **Region Values**: Region-specific configurations

---

## Template Analysis

### Template Helper Functions
```yaml
# Naming conventions
{{- define "enm-client.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

# Conditional logic
{{- define "environmentName"}}
{{- if or (eq .Values.global.serviceEnvironment "Production") (eq .Values.global.serviceEnvironment "Production8") -}}
prod
{{- else if eq (lower .Values.global.serviceEnvironment) "stage8" -}}
stage
{{- else -}}
{{- print .Values.global.serviceEnvironment | lower }}
{{- end }}
{{- end }}

# Region-specific values
{{- define "cpuLimitWarning" -}}
{{- if eq .Values.global.serviceRegion "va6" -}}
432
{{- else -}}
288
{{- end }}
{{- end }}
```

### Template Patterns
1. **Conditional Rendering**: `{{- if .Values.enabled }}`
2. **Variable Substitution**: `{{ include "chart.name" . }}`
3. **Function Calls**: `{{ include "environmentName" . }}`
4. **Built-in Objects**: `{{ $.Release.Namespace }}`

---

## Deployment Strategies

### Rolling Update Configuration
```yaml
rollout:
  strategy:
    rolling:
      maxSurge: 25%          # Can create 25% extra pods during update
      maxUnavailable: 0%     # Zero downtime - no pods unavailable
  revisionHistoryLimit: 5    # Keep 5 old ReplicaSets for rollback
```

### Multi-Service Deployment
```yaml
# Main Application Service
cont1-deployment-templates:
  deployment:
    name: deploy1
    replicas: 2

# Worker Service
ums-worker-deployment-templates:
  deployment:
    name: ums-worker
    replicas: 1
```

### Environment Promotion
```yaml
stages:
  - name: "Stage"
    steps: [va6]
  - name: "approve-PROD-promotion"
    steps: [manual-wait]  # Manual approval gate
  - name: "Prod"
    steps: [va6]
```

---

## Monitoring & Observability

### Prometheus Rules
```yaml
# Pod Health
- alert: UnifiedMeteringservice-KubernetesPodCrashLooping
  expr: increase(kube_pod_container_status_restarts_total[1m]) > 2
  for: 5m

# Resource Utilization
- alert: UnifiedMeteringService-HighCPUUtilization
  expr: (sum(rate(container_cpu_usage_seconds_total{container="cont1"}[3m]))by (pod) /4*100) > 60
  for: 5m

# Auto-scaling
- alert: UnifiedMeteringService-HPA-HighContainerCount
  expr: ( kube_horizontalpodautoscaler_status_desired_replicas > (kube_horizontalpodautoscaler_spec_max_replicas*70/100) ) > 0
  for: 5m
```

### Alerting Configuration
- **Severity Levels**: sev2 (Critical), sev3 (Warning)
- **Slack Integration**: `ums-stage-alert` channel
- **Runbook Links**: Adobe internal documentation

### Logging Configuration
```yaml
logging:
  index: unified_metering_service_nonprod
  sourceType: ums-stage-va6
  enableAccessLogging: true
  envoyHttpFilters: all
```

---

## Security Implementation

### Vault Integration
```yaml
# Database Credentials
secretRefPath: dme_meterix/unifiedmeteringservice/stage/ethos/runtime/va6/database/aurora_auth

# IMS Credentials
secretRefPath: dme_meterix/unifiedmeteringservice/stage/IMS_CREDENTIALS

# Redis Authentication
secretRefPath: dme_meterix/unifiedmeteringservice/stage/ethos/runtime/va6/database/redis_auth
```

### IRSA (IAM Roles for Service Accounts)
```yaml
useServiceAccountInContainer:
  enabled: true
  name: irsa-sa
```

### API Gateway Security
```yaml
authPolicy:
  apiKey:
    enabled: true
    priority: [queryParam, header, cookie]
  oAuth:
    ims:
      enabled: true
      tokenType: service/user
```

---

## Interview Questions & Answers

### Q1: Explain the difference between Chart.yaml and values.yaml
**Answer**:
```yaml
# Chart.yaml - Metadata and Dependencies
apiVersion: v2
name: my-chart
version: 1.0.0
dependencies:
  - name: nginx
    version: "1.0.0"
    repository: "https://charts.bitnami.com/bitnami"

# values.yaml - Configuration Values
nginx:
  replicaCount: 3
  image:
    repository: nginx
    tag: "1.21"
```

**Key Differences**:
- **Chart.yaml**: Defines chart metadata, dependencies, and structure
- **values.yaml**: Defines configurable parameters and their default values
- **Chart.yaml**: Required for chart definition
- **values.yaml**: Optional, provides default values

### Q2: How do you handle secrets in Helm charts?
**Answer**:
```yaml
# Method 1: External Secrets Operator
secrets:
  database:
    vaultPath: "secret/database"
    fields:
      - username
      - password
    secretStoreRef: "vault-backend"

# Method 2: Kubernetes Secrets
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "my-chart.fullname" . }}-secret
type: Opaque
data:
  username: {{ .Values.database.username | b64enc }}
  password: {{ .Values.database.password | b64enc }}
```

### Q3: Explain Helm template functions and their usage
**Answer**:
```yaml
# String Functions
{{ .Values.name | upper }}                    # Convert to uppercase
{{ .Values.name | lower }}                    # Convert to lowercase
{{ .Values.name | trunc 10 }}                 # Truncate to 10 characters
{{ .Values.name | replace "old" "new" }}      # Replace text

# Conditional Functions
{{- if .Values.enabled }}
enabled: true
{{- else }}
enabled: false
{{- end }}

# Default Values
{{ .Values.replicas | default 3 }}            # Default to 3 if not set

# Template Functions
{{ include "my-chart.fullname" . }}           # Call template function
{{ template "my-chart.labels" . }}            # Include template
```

### Q4: How do you implement environment-specific configurations?
**Answer**:
```yaml
# values.yaml (Global defaults)
global:
  environment: dev
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

# values-staging.yaml (Staging overrides)
global:
  environment: staging
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 512Mi

# values-production.yaml (Production overrides)
global:
  environment: production
  replicas: 5
  resources:
    requests:
      cpu: 1
      memory: 2Gi
    limits:
      cpu: 2
      memory: 4Gi
```

### Q5: Explain Helm hooks and their usage
**Answer**:
```yaml
# Pre-install hook
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-chart.fullname" . }}-pre-install
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
spec:
  template:
    spec:
      containers:
      - name: pre-install
        image: busybox
        command: ["sh", "-c", "echo 'Pre-install hook executed'"]

# Post-install hook
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-chart.fullname" . }}-post-install
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "5"
```

---

## Practical Commands

### Repository Management
```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# List repositories
helm repo list

# Update repositories
helm repo update

# Search charts
helm search repo nginx
```

### Chart Management
```bash
# Install chart
helm install my-release ./charts/my-chart

# Upgrade chart
helm upgrade my-release ./charts/my-chart

# List releases
helm list

# Check release status
helm status my-release

# Uninstall release
helm uninstall my-release
```

### Dependency Management
```bash
# Update dependencies
helm dependency update

# Build dependencies
helm dependency build

# List dependencies
helm dependency list
```

### Template Operations
```bash
# Render templates
helm template my-chart ./charts/my-chart

# Dry run
helm install my-release ./charts/my-chart --dry-run

# Debug template
helm template my-chart ./charts/my-chart --debug
```

---

## Best Practices

### Chart Design
- ✅ **Version Pinning**: Specific versions for stability  
- ✅ **Template Reusability**: Helper functions and common templates  
- ✅ **Environment Separation**: Different charts for different environments  
- ✅ **Dependency Management**: Clear dependency definitions  

### Security
- ✅ **Secrets Management**: Use Vault or external secret operators  
- ✅ **RBAC Integration**: IAM roles for service accounts  
- ✅ **Network Policies**: Proper ingress and egress controls  
- ✅ **Image Security**: Use trusted base images  

### Operations
- ✅ **Health Checks**: Proper liveness and readiness probes  
- ✅ **Resource Limits**: Set appropriate CPU and memory limits  
- ✅ **Monitoring**: Comprehensive observability stack  
- ✅ **Rollback Strategy**: Keep revision history for rollbacks  

### GitOps Integration
- ✅ **Git-based**: All configurations in version control  
- ✅ **Automated Deployments**: CI/CD pipeline integration  
- ✅ **Environment Promotion**: Controlled promotion process  
- ✅ **Audit Trail**: Track all changes in Git  

---

## Key Takeaways

1. **Helm Components**: Chart, Release, Repository, Template, Values, Dependencies, Hooks, Tests
2. **Dependency Management**: Aliasing, conditional dependencies, version pinning
3. **Values Architecture**: Hierarchical override pattern
4. **Template Patterns**: Conditional rendering, helper functions, built-in objects
5. **Deployment Strategies**: Rolling updates, zero-downtime deployments
6. **Security**: Vault integration, IRSA, network policies
7. **Monitoring**: Prometheus rules, alerting, logging
8. **Best Practices**: Version control, automation, security, observability

---

## Repository vs Chart Differences

| Aspect | Repository | Chart |
|--------|------------|-------|
| **Purpose** | Storage/Collection | Package/Application |
| **Contains** | Multiple charts | Single application |
| **Location** | Remote server or local directory | Single directory |
| **Access** | `helm repo add/update` | `helm install/upgrade` |
| **Example** | Artifactory, GitHub | Your application |

### Repository Examples in Your Project
```yaml
# Remote Repositories
- name: ethos-k8s-helm-templates
  repository: "https://artifactory-uw2.adobeitc.com/artifactory/helm-ethos-k8s-helm-templates-release"

# Local Repositories  
- name: enm-client
  repository: "file://../../../../charts/enm-client"
```

---

## Deployment Architecture

### GitOps Deployment Pattern
Your Unified Metering Service is deployed using **GitOps** (Argo CD) rather than direct Helm commands:

```
Git Repository → Argo CD → Kubernetes Cluster → Namespace: ns-team-meterix--unified-metering-service-deploy--4be-e2ad9e37
```

### Why No Direct Helm Releases?
- **Resources exist** but no Helm releases
- **Argo Workflow templates** suggest GitOps deployment
- **Namespace naming** follows GitOps pattern
- **Automated deployments** through Git changes

---

## Contributing

This documentation is based on the analysis of the Unified Metering Service deployment repository. For updates or corrections, please refer to the actual implementation in the repository.

---

## License

This documentation is for educational and interview preparation purposes based on the Unified Metering Service implementation.
