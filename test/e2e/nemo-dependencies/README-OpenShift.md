# NVIDIA NIM Operator E2E Dependencies - OpenShift Deployment

Complete guide for deploying NVIDIA NIM Operator E2E test dependencies on OpenShift.

> **🚀 New to NVIDIA NEMO? Start with the [Quick Start Guide](./QUICK-START-OpenShift.md) for a streamlined 45-minute deployment.**
>
> This document provides comprehensive configuration options, troubleshooting, and architectural details.

## Table of Contents

- [Prerequisites](#prerequisites)

- [Infrastructure Components](#infrastructure-components)
  - [Datastore Component Deployment](#datastore-component-deployment)
    - [Troubleshooting](#troubleshooting)
      - [Generated Files](#generated-files)
    - [Verification](#verification)
  - [Entity-Store Component Deployment](#entity-store-component-deployment)
    - [Troubleshooting](#troubleshooting-1)
      - [Generated Files](#generated-files-1)
    - [Verification](#verification-1)
  - [Customizer Component Deployment](#customizer-component-deployment)
    - [Troubleshooting](#troubleshooting-2)
    - [Generated Files](#generated-files-2)
    - [MLflow Deployment Options](#mlflow-deployment-options)
      - [Option 1: MLflow without MinIO (Simple Setup)](#option-1-mlflow-without-minio-simple-setup)
      - [Option 2: MLflow with MinIO (Enhanced Setup)](#option-2-mlflow-with-minio-enhanced-setup)
    - [Customizer Verification](#customizer-verification)
    - [Customizer Post-Deployment Setup for MinIO (Option 2)](#customizer-post-deployment-setup-for-minio-option-2)
    - [Choosing Between MLflow Options (Customizer)](#choosing-between-mlflow-options-customizer)
  - [Jupyter Component Deployment](#jupyter-component-deployment)
    - [Generated Files](#generated-files-3)
    - [Verification](#verification-3)
  - [Guardrail Component Deployment](#guardrail-component-deployment)
    - [Generated Files](#generated-files-4)
    - [Verification](#verification-4)
  - [Evaluator Component Deployment](#evaluator-component-deployment)
    - [Generated Files](#generated-files-5)
    - [Argo Workflows Configuration](#argo-workflows-configuration)
      - [Cluster Conflict Resolution](#cluster-conflict-resolution)
      - [Namespaced Mode Benefits](#namespaced-mode-benefits)
    - [Component Architecture](#component-architecture)
    - [Verification](#verification-5)

- [NeMo Operator Installation (v25.06)](#nemo-operator-installation-v2506)
  - [Prerequisites](#nemo-operator-prerequisites)
  - [NeMo Operator Deployment](#nemo-operator-deployment)
  - [Verification](#nemo-operator-verification)
  - [Troubleshooting NeMo Operator](#troubleshooting-nemo-operator)

- [NIM Operator Installation (v3.0.1)](#nim-operator-installation-v301)
  - [Prerequisites](#nim-operator-prerequisites)
  - [NIM Operator Deployment](#nim-operator-deployment)
  - [Verification](#nim-operator-verification)
  - [Troubleshooting NIM Operator](#troubleshooting-nim-operator)

- [NEMO Samples Deployment](#nemo-samples-deployment)

- [NEMO Microservices Verification](#nemo-microservices-verification)
  - [Overview](#overview)
  - [Step 1: Check NEMO Microservices Status](#step-1-check-nemo-microservices-status)
  - [Step 2: Verify ConfigMaps](#step-2-verify-configmaps)
  - [Step 3: Check NIM Services Status](#step-3-check-nim-services-status)
  - [Step 4: Verify Service Endpoints](#step-4-verify-service-endpoints)
  - [Step 5: Test API Endpoints](#step-5-test-api-endpoints)
  - [Verification Summary](#verification-summary)
  - [Troubleshooting Verification Issues](#troubleshooting-verification-issues)

- [Architectural Decisions & Component Analysis](#architectural-decisions--component-analysis)
  - [Volcano Batch Scheduler Analysis](#volcano-batch-scheduler-analysis)
  - [Bitnami Init Container Analysis](#bitnami-init-container-analysis)
  - [Trade-offs Summary](#trade-offs-summary)
  - [Re-enabling Components Safely](#re-enabling-components-safely)
  - [Summary: Current vs. Original v3.0.0 Configuration](#summary-current-vs-original-v300-configuration)

- [Common Troubleshooting](#common-troubleshooting)
  - [MLflow-Specific Issues](#mlflow-specific-issues)
  - [Pod Stuck in Pending](#pod-stuck-in-pending)

- [Notes](#notes)

- [Security Considerations](#security-considerations)

## Prerequisites

- OpenShift cluster with AWS EBS storage
- `oc` CLI authenticated to cluster
- Ansible installed locally
- Target namespace created (update `installation_namespace` in `values.yaml`)

## Infrastructure Components

The infrastructure components provide the foundational services required for the NVIDIA NEMO ecosystem. These components should be deployed first before installing the operators.

### Datastore Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for datastore only**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     datastore: yes
     # Set all others to 'no'
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Troubleshooting

For common issues (pod stuck in pending, storage classes, access modes), see [Common Troubleshooting](#common-troubleshooting) below.

#### Generated Files

The playbook creates sensitive files that are git-ignored:
- `ds-postgresql-values.yaml` - Helm values
- `secrets.yaml` - Database passwords and JWT tokens

### Verification

```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify PostgreSQL is running
oc logs datastore-pg-postgresql-0 -n <your-namespace>
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
datastore-pg-postgresql-0   1/1     Running   0          10m
```

### Entity-Store Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for entity-store**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     entity_store: yes
     # Set all others to 'no' for standalone deployment
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Troubleshooting

For common issues (pod stuck in pending, storage classes, access modes), see [Common Troubleshooting](#common-troubleshooting) below.

#### Generated Files

The playbook creates sensitive files that are git-ignored:
- `entity-store-db-values.yaml` - Helm values with PostgreSQL configuration
- `secrets.yaml` - Entity-store PostgreSQL password

### Verification

```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify PostgreSQL is running
oc logs entity-store-pg-postgresql-0 -n <your-namespace>
```

Expected output:
```
NAME                            READY   STATUS    RESTARTS   AGE
entity-store-pg-postgresql-0    1/1     Running   0          10m
```

### Customizer Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for customizer**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     customizer: yes
     # Set all others to 'no' for standalone deployment
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Troubleshooting

For common issues (pod stuck in pending, storage classes, access modes), see [Common Troubleshooting](#common-troubleshooting) below.

### Generated Files

The playbook creates sensitive files that are git-ignored:
- `customizer-postgresql-values.yaml` - Helm values with PostgreSQL configuration
- `customizer-opentelemetry-values.yaml` - OpenTelemetry collector configuration
- `secrets.yaml` - Customizer PostgreSQL password

### MLflow Deployment Options

MLflow can be deployed in two configurations depending on your artifact storage requirements:

#### Option 1: MLflow without MinIO (Simple Setup)

**Configuration**: Basic MLflow tracking with PostgreSQL backend only.

**Image Configuration for v3.0.0 Compatibility:**
- **MLflow Server**: `docker.io/library/python:3.9-slim` with runtime installation
- **PostgreSQL**: `docker.io/bitnami/postgresql:latest`
- **Git Init Container**: `docker.io/alpine/git:latest`
- **Volume Permissions**: `docker.io/library/busybox:latest`
- **MinIO**: Disabled (for artifact storage simplicity)

**Configuration in `customizer/defaults/main.yml`:**
```yaml
mlflow:
  enabled: true
  # MinIO will be disabled in generated mlflow.yaml
```

**Generated `mlflow.yaml` template** includes MinIO disabled configuration:
```yaml
minio:
  enabled: false
```

#### Option 2: MLflow with MinIO (Enhanced Setup)

**Configuration**: Full MLflow tracking with MinIO object storage for artifacts.

**Image Configuration for v3.0.0 Compatibility:**
- **MLflow Server**: `docker.io/library/python:3.9-slim` with runtime installation
- **PostgreSQL**: `docker.io/bitnami/postgresql:latest`
- **MinIO**: `quay.io/minio/minio:latest` (official MinIO image)
- **Git Init Container**: `docker.io/alpine/git:latest`
- **Volume Permissions**: Disabled (to avoid Bitnami utility dependencies)

**Configuration in `customizer/templates/mlflow.yaml.j2`:**
```yaml
minio:
  enabled: true
  image:
    registry: quay.io
    repository: minio/minio
    tag: latest
  # Disable init containers that expect Bitnami utilities
  enableDefaultInitContainers: false
  # Disable bucket provisioning job that fails with wait-for-available-minio
  provisioning:
    enabled: false
  auth:
    rootUser: "minioadmin"      # ⚠️ SECURITY: Change in production
    rootPassword: "minioadmin"  # ⚠️ SECURITY: Change in production

volumePermissions:
  enabled: false  # Disable to avoid Bitnami-specific utilities

postgresql:
  # Disable init containers that may expect Bitnami utilities
  enableDefaultInitContainers: false
```

**Custom Installation Method (Both Options):**
MLflow is installed at runtime using `pip install --prefix=/tmp/pip-install mlflow==2.12.2 psycopg2-binary` to work around read-only filesystem restrictions in OpenShift security contexts.

**Additional Generated Files:**
- `mlflow.yaml` - MLflow Helm values with working image overrides (git-ignored)

### Customizer Verification

**Common verification commands:**
```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify PostgreSQL is running
oc logs customizer-pg-postgresql-0 -n <your-namespace>

# Verify OpenTelemetry collector is running
oc logs -l app.kubernetes.io/name=opentelemetry-collector -n <your-namespace>

# Verify MLflow tracking server is running
oc logs -l app.kubernetes.io/name=mlflow -n <your-namespace>
```

**Expected output for Option 1 (MLflow without MinIO):**
```
NAME                                           READY   STATUS    RESTARTS   AGE
customizer-pg-postgresql-0                     1/1     Running   0          10m
opentelemetry-collector-xxxxx                  1/1     Running   0          10m
mlflow-tracking-xxxxx                          1/1     Running   0          10m
mlflow-postgresql-0                            1/1     Running   0          10m
```

**Expected output for Option 2 (MLflow with MinIO):**
```
NAME                                           READY   STATUS    RESTARTS   AGE
customizer-pg-postgresql-0                     1/1     Running   0          10m
opentelemetry-collector-xxxxx                  1/1     Running   0          10m
mlflow-tracking-xxxxx                          1/1     Running   0          10m
mlflow-postgresql-0                            1/1     Running   0          10m
mlflow-minio-xxxxx                             1/1     Running   0          10m
```

**MinIO-specific verification (Option 2 only):**
```bash
# Verify MinIO is running and accessible
oc logs -l app.kubernetes.io/name=minio -n <your-namespace>

# Check MinIO service
oc get svc mlflow-minio -n <your-namespace>

# Access MinIO console (optional)
oc port-forward svc/mlflow-minio -n <your-namespace> 9001:9001
# Then open http://localhost:9001 with credentials: minioadmin/minioadmin
# ⚠️ SECURITY: Change default credentials in production
```

**MLflow artifact storage configuration:**
- **Option 1**: Artifacts stored in MLflow server filesystem
- **Option 2**: Artifacts stored in MinIO at `s3://mlflow/` bucket

### Customizer Post-Deployment Setup for MinIO (Option 2)

When using MLflow with MinIO, the bucket provisioning job is disabled to avoid Bitnami utility dependencies. **Manual bucket setup is required:**

```bash
# Method 1: Using MinIO Client (mc) from within MinIO pod
oc exec -n <your-namespace> deployment/mlflow-minio -- sh -c "
  mc alias set myminio http://localhost:9000 minioadmin minioadmin
  mc mb myminio/mlflow --ignore-existing
  mc policy set public myminio/mlflow
"
# ⚠️ SECURITY: Uses default credentials - change in production

# Method 2: Using kubectl and MinIO pod directly
oc exec -n <your-namespace> deployment/mlflow-minio -- mkdir -p /data/mlflow

# Verify bucket creation
oc exec -n <your-namespace> deployment/mlflow-minio -- ls -la /data/
```

**Expected output after bucket setup:**
```
total 12
drwxr-xr-x    3 1001     1001          4096 Oct 16 10:30 .
drwxr-xr-x    1 root     root          4096 Oct 16 10:25 ..
drwxr-xr-x    2 1001     1001          4096 Oct 16 10:30 mlflow
```

**MLflow configuration for MinIO:**
MLflow is automatically configured to use MinIO for artifact storage when the minio component is enabled. The connection details are handled via environment variables in the MLflow tracking server.

### Choosing Between MLflow Options (Customizer)

| Feature | Option 1 (without MinIO) | Option 2 (with MinIO) |
|---------|--------------------------|----------------------|
| **Complexity** | Simple | Moderate |
| **Artifact Storage** | Local filesystem | S3-compatible object storage |
| **Scalability** | Limited | High |
| **Post-Deploy Setup** | None required | Manual bucket creation |
| **Resource Usage** | Lower (fewer pods) | Higher (additional MinIO pod) |
| **Production Ready** | Basic workflows | Enterprise workflows |
| **Backup/Restore** | File-based | Object-based |

**Recommendation:**
- **Choose Option 1** for development, testing, or simple MLflow tracking needs
- **Choose Option 2** for production environments requiring scalable artifact storage and enterprise features

**Current Implementation**: This deployment uses **Option 2 (MLflow with MinIO)** for enhanced artifact storage capabilities.

### Jupyter Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for jupyter**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     jupyter: yes
     # Set all others to 'no' for standalone deployment
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Generated Files

The playbook creates files that are git-ignored:
- No sensitive configuration files generated for Jupyter (uses default settings)

**Configuration Notes:**
- **Image**: Uses default Jupyter notebook image
- **Authentication**: Default token-based access (`jupyter_token: token`) ⚠️ SECURITY: Change in production
- **Storage**: 5Gi persistent volume for notebook storage
- **Port**: NodePort service on port 30036
- **No modifications**: Uses original v3.0.0 configuration

### Verification

```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify Jupyter notebook is running
oc logs -l app=jupyter-notebook -n <your-namespace>

# Get service details for access
oc get svc jupyter-service -n <your-namespace>
```

Expected output:
```
NAME                            READY   STATUS    RESTARTS   AGE
jupyter-notebook-xxxxx          1/1     Running   0          10m
```

**Access Jupyter Notebook:**
```bash
# Port forward to access locally
oc port-forward svc/jupyter-service -n <your-namespace> 8888:8888

# Then open http://localhost:8888
# Default token: token (⚠️ SECURITY: Change in production)
```

### Guardrail Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for guardrail**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     guardrail: yes
     # Set all others to 'no' for standalone deployment
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Generated Files

The playbook creates sensitive files that are git-ignored:
- `guardrail-postgresql-values.yaml` - Helm values with PostgreSQL configuration
- `secrets.yaml` - Guardrail PostgreSQL password

**Configuration Notes:**
- **Database**: PostgreSQL backend (similar to other components)
- **Credentials**: Database credentials auto-generated in secrets.yaml (⚠️ Review generated passwords)
- **Image**: Uses default Guardrail image from v3.0.0
- **No modifications**: Uses original v3.0.0 configuration

### Verification

```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify PostgreSQL is running
oc logs guardrail-pg-postgresql-0 -n <your-namespace>
```

Expected output:
```
NAME                            READY   STATUS    RESTARTS   AGE
guardrail-pg-postgresql-0       1/1     Running   0          10m
```

### Evaluator Component Deployment

1. **Update values.yaml**:

```yaml
pvc:
  storage_class: "gp3-csi"
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany

localPathProvisioner:
  enabled: false  # Use EBS instead of local-path
```

2. **Configure for evaluator**:
   ```yaml
   # values.yaml
   installation_namespace: <your-namespace>

   install:
     evaluator: yes
     # Set all others to 'no' for standalone deployment
   ```

3. **Deploy**:
   ```bash
   ansible-playbook -c local -i localhost install.yaml
   ```

### Generated Files

The playbook creates sensitive files that are git-ignored:
- `evaluator-postgresql-values.yaml` - Helm values with PostgreSQL configuration
- `secrets.yaml` - Evaluator PostgreSQL password
- `argo-values.yaml` - Argo Workflows configuration with namespace-scoped permissions
- `argo-sa.yaml` - Service account for workflow execution
- `milvus-values.yaml` - Milvus vector database configuration
- `opentelemetry-values.yaml` - OpenTelemetry collector configuration
- `milvus-oc-rbac.yaml` - OpenShift RBAC for Milvus operations
- `argo-validation.yaml` - Argo Workflows validation rules

**Configuration Notes:**
- **Database**: PostgreSQL backend (similar to other components)
- **Vector Database**: Milvus for embeddings and similarity search
- **Workflow Engine**: Argo Workflows for evaluation pipeline orchestration
- **Observability**: OpenTelemetry for metrics and tracing
- **Credentials**: Database and service account credentials auto-generated in secrets.yaml
- **RBAC**: Namespace-scoped permissions only (enhanced security)

### Argo Workflows Configuration

The evaluator uses **Argo Workflows** for orchestrating evaluation pipelines. Special configuration was required for OpenShift compatibility and cluster safety.

#### Cluster Conflict Resolution

**Problem Encountered:**
- Existing orphaned ClusterRoles from previous installations blocked deployment
- CRD conflicts with Data Science Pipelines operator
- Namespace ownership conflicts (`<some-namespace>` vs `<your-namespace>`)

**Solution Implemented:**
```yaml
# argo-values.yaml configuration
crds:
  install: false  # Use existing CRDs from Data Science Pipelines operator

singleNamespace: true  # Restrict to namespace-scoped resources only

createAggregateRoles: false  # Disable cluster-wide aggregate roles

# Disable cluster template access to avoid conflicts
controller:
  clusterWorkflowTemplates:
    enabled: false

server:
  clusterWorkflowTemplates:
    enabled: false
```

#### Namespaced Mode Benefits

**Enhanced Security Model:**
- **No ClusterRoles**: Uses namespace-scoped Roles only
- **Zero Cluster Impact**: Cannot affect other namespaces or components
- **Isolation**: Complete workflow isolation within evaluator namespace
- **RBAC Precision**: Minimal required permissions only

**Functional Impact Assessment:**
```
✅ Preserved: All workflow execution, templates, UI, artifacts
✅ Enhanced: Security isolation, deployment safety, conflict avoidance
⚠️ Limited: No access to ClusterWorkflowTemplates (not used by evaluator)
```

**Resource Creation Comparison:**
```
Standard Mode (Risky):        Namespaced Mode (Safe):
├── 9 ClusterRoles           ├── 3 Roles (namespace-scoped)
├── 4 ClusterRoleBindings    ├── 3 RoleBindings (namespace-scoped)
└── Cluster-wide permissions └── Zero cluster impact
```

### Component Architecture

The evaluator component deploys a comprehensive AI evaluation stack:

**Core Components:**
- **Argo Workflows Server & Controller**: Pipeline orchestration and workflow management
- **PostgreSQL**: Metadata storage for evaluation results and configuration
- **Milvus**: Vector database for embeddings, similarity search, and retrieval evaluation
- **OpenTelemetry Collector**: Metrics collection and observability

**Data Flow:**
1. **Evaluation Requests** → Argo Workflows (pipeline orchestration)
2. **Workflow Execution** → Milvus (vector operations) + PostgreSQL (metadata)
3. **Metrics & Traces** → OpenTelemetry Collector
4. **Results Storage** → PostgreSQL (structured data) + Milvus (embeddings)

**Integration Points:**
- **NGC Integration**: Pre-configured image pull secrets (`ngc-secret`, `nvcrimagepullsecret`)
- **OpenShift Security**: Uses `anyuid` SCC for workflow pods
- **Namespace Isolation**: All components operate within evaluator namespace only

### Verification

```bash
# Check deployment status
oc get pods -n <your-namespace>
oc get pvc -n <your-namespace>

# Verify PostgreSQL is running
oc logs evaluator-pg-postgresql-0 -n <your-namespace>

# Verify Argo Workflows components
oc logs -l app.kubernetes.io/name=argo-workflows-server -n <your-namespace>
oc logs -l app.kubernetes.io/name=argo-workflows-workflow-controller -n <your-namespace>

# Verify Milvus vector database
oc logs -l app.kubernetes.io/name=milvus -n <your-namespace>

# Verify OpenTelemetry collector
oc logs -l app.kubernetes.io/name=opentelemetry-collector -n <your-namespace>

# Check Argo Workflows UI access
oc port-forward svc/argo-workflows-server -n <your-namespace> 2746:2746
# Then open https://localhost:2746
```

Expected output:
```
NAME                                           READY   STATUS    RESTARTS   AGE
evaluator-pg-postgresql-0                      1/1     Running   0          10m
argo-workflows-server-xxxxx                    1/1     Running   0          10m
argo-workflows-workflow-controller-xxxxx       1/1     Running   0          10m
milvus-xxxxx                                   1/1     Running   0          10m
opentelemetry-collector-xxxxx                  1/1     Running   0          10m
```

**Post-Deployment Validation:**
```bash
# Test Argo Workflows functionality
oc get workflows -n <your-namespace>
oc get workflowtemplates -n <your-namespace>

# Test Milvus connectivity (optional)
oc exec -n <your-namespace> deployment/milvus -- curl -s localhost:19530/health

# Verify service accounts and RBAC
oc get serviceaccounts | grep argo-workflows
oc get roles,rolebindings | grep argo-workflows
```

## NeMo Operator Installation (v25.06)

The NVIDIA NeMo Operator v25.06 is required for building, training, and customizing generative AI models. This operator manages the infrastructure components (dependencies) that the NEMO microservices need. Install this operator **first** before proceeding with the NIM Operator.

### Prerequisites {#nemo-operator-prerequisites}

Before installing the NeMo Operator, ensure you have:

1. **All Infrastructure Components Deployed**: Complete infrastructure from previous sections:
   - ✅ Datastore, Entity-store, Customizer, Evaluator, Guardrail components deployed via Ansible
   - ✅ PostgreSQL databases, OpenTelemetry, MLflow, Argo Workflows, Milvus running
   - ✅ All dependency services should be in "Ready" state

2. **NGC Credentials**: Valid NVIDIA GPU Cloud API key
3. **OpenShift Cluster**: Version 4.x with sufficient resources
4. **Target Namespace**: Use existing `<your-namespace>` from dependency deployment
5. **Helm**: Version 3.x installed and configured

**Pre-Installation Verification:**
```bash
# Verify all infrastructure dependencies are ready
oc get pods -n <your-namespace> | grep -E "(postgresql|opentelemetry|mlflow|argo|milvus)"

# Should show all infrastructure services running
```

### Volcano Scheduler Installation (Required for NeMo Operator)

**IMPORTANT**: The NeMo Operator requires Volcano scheduler for advanced GPU job scheduling and PodGroup CRDs.

1. **Install Volcano Scheduler**:
   ```bash
   # Add Volcano Helm repository
   helm repo add volcano https://volcano-sh.github.io/helm-charts
   helm repo update

   # Install Volcano scheduler
   helm install volcano volcano/volcano \
     --namespace <your-namespace> \
     --version 1.9.0 \
     --wait --timeout=300s

   # Grant privileged SCC to volcano-scheduler for hostPath volumes
   oc adm policy add-scc-to-user privileged system:serviceaccount:<your-namespace>:volcano-scheduler

   # Restart volcano-scheduler to apply SCC permissions
   oc rollout restart deployment volcano-scheduler -n <your-namespace>
   ```

2. **Verify Volcano Installation**:
   ```bash
   # Check all Volcano pods are running
   oc get pods -n <your-namespace> | grep volcano

   # Verify PodGroup CRD is installed
   oc get crd | grep podgroup
   ```

   Expected output:
   ```
   volcano-admission-xxxxx                         1/1     Running     0          2m
   volcano-controllers-xxxxx                       1/1     Running     0          2m
   volcano-scheduler-xxxxx                         1/1     Running     0          2m
   volcano-admission-init-xxxxx                    0/1     Completed   0          2m

   podgroups.scheduling.volcano.sh                 2025-XX-XX
   ```

### NeMo Operator Deployment

1. **Add NVIDIA NeMo Helm Repository**:
   ```bash
   helm repo add nvidia-nemo https://helm.ngc.nvidia.com/nvidia-nemo
   helm repo update
   ```

2. **Install NeMo Operator v25.06**:
   ```bash
   # Install NeMo Operator with proper resource limits
   helm install nemo-operator nvidia-nemo/nemo-operator \
     -n <your-namespace> \
     --set manager.resources.limits.memory=512Mi \
     --set manager.resources.requests.memory=256Mi \
     --wait --timeout=300s

   # CRITICAL: Patch service account to use NGC image pull secret
   # This step is required for OpenShift to pull NGC container images
   oc patch serviceaccount nemo-operator-controller-manager -n <your-namespace> -p '{"imagePullSecrets": [{"name": "ngc-secret"}]}'

   # Restart the deployment to pick up the new secret
   oc delete pod -n <your-namespace> -l app.kubernetes.io/name=nemo-operator
   ```

3. **Verify Operator Installation**:
   ```bash
   # Check operator pod status
   oc get pods -n <your-namespace> | grep nemo-operator

   # Check operator logs
   oc logs -l app.kubernetes.io/name=nemo-operator -n <your-namespace>

   # Verify CRDs are installed
   oc get crd | grep -E "(nemocustomizer|nemodatastore|nemoentitystore)"
   ```

   Expected output:
   ```
   NAME                                            READY   STATUS    RESTARTS   AGE
   nemo-operator-controller-manager-xxxxx          2/2     Running   0          2m
   ```

### Verification {#nemo-operator-verification}

**Check NeMo Operator Status:**
```bash
# Verify operator is running and healthy
oc get pods -n <your-namespace> | grep nemo-operator
oc get deployment nemo-operator-controller-manager -n <your-namespace>

# Check CRDs are available
oc get crd | grep nemo
```

Expected output:
```
NAME                                            READY   STATUS    RESTARTS   AGE
nemo-operator-controller-manager-xxxxx          2/2     Running   0          5m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
nemo-operator-controller-manager         1/1     1            1           5m
```

**Verify CRDs are Installed:**
```bash
oc get crd | grep nemo
```

Expected output should include:
```
nemocustomizers.apps.nvidia.com
nemodatastores.apps.nvidia.com
nemoentitystores.apps.nvidia.com
nemoevaluators.apps.nvidia.com
nemoguardrails.apps.nvidia.com
```

### Troubleshooting NeMo Operator

**Issue**: NeMo Operator pod stuck in `ImagePullBackOff` status
```bash
# Error: Failed to pull image "nvcr.io/nvidia/nemo-operator:v25.06"
```
- **Root Cause**: Service account lacks NGC image pull secret for accessing NVIDIA container registry
- **Solution**: Patch the service account to use NGC secret:
```bash
# Add NGC image pull secret to service account
oc patch serviceaccount nemo-operator-controller-manager -n <your-namespace> -p '{"imagePullSecrets": [{"name": "ngc-secret"}]}'

# Restart the deployment to pick up the new secret
oc delete pod -n <your-namespace> -l app.kubernetes.io/name=nemo-operator

# Verify pod is now running
oc get pods -n <your-namespace> | grep nemo-operator
```

**Issue**: NeMo Operator fails with "no matches for kind 'PodGroup' in version 'scheduling.volcano.sh/v1beta1'"
```bash
# Error: unable to create controller: no matches for kind "PodGroup" in version "scheduling.volcano.sh/v1beta1"
```
- **Root Cause**: Volcano scheduler not installed or PodGroup CRDs missing
- **Solution**: Install Volcano scheduler first:
```bash
# Install Volcano scheduler
helm install volcano volcano/volcano --namespace <your-namespace> --version 1.9.0
oc adm policy add-scc-to-user privileged system:serviceaccount:<your-namespace>:volcano-scheduler
oc rollout restart deployment volcano-scheduler -n <your-namespace>

# Verify PodGroup CRD exists
oc get crd | grep podgroup
```

**Issue**: Volcano scheduler pod stuck in pending with SCC violations
```bash
# Error: pods "volcano-scheduler-xxx-" is forbidden: unable to validate against any security context constraint
# [...] spec.volumes[1]: Invalid value: "hostPath": hostPath volumes are not allowed to be used
```
- **Root Cause**: Volcano scheduler requires privileged SCC for hostPath volumes (`/tmp/klog-socks`)
- **Solution**: Grant privileged SCC to volcano-scheduler service account:
```bash
oc adm policy add-scc-to-user privileged system:serviceaccount:<your-namespace>:volcano-scheduler
oc rollout restart deployment volcano-scheduler -n <your-namespace>
```

**Issue**: NeMo Operator CrashLoopBackOff with OOMKilled
```bash
# Error: pod killed due to memory limit
```
- **Root Cause**: Default memory limit (256Mi) insufficient for operator
- **Solution**: Increase memory limits during installation:
```bash
helm upgrade nemo-operator nvidia-nemo/nemo-operator \
  -n <your-namespace> \
  --set manager.resources.limits.memory=512Mi \
  --set manager.resources.requests.memory=256Mi
```

**Issue**: CRDs not found or conflicts
```bash
# Error: CRD conflicts or missing custom resource definitions
```
- **Root Cause**: NeMo Operator CRDs conflict with existing resources
- **Solution**: Check for existing CRDs and clean up if necessary:
```bash
oc get crd | grep nemo
# Remove conflicting CRDs if safe to do so
```

**Issue**: Helm repository not found
```bash
# Error: chart "nemo-operator" matching not found
```
- **Root Cause**: Wrong Helm repository or repository not added
- **Solution**: Ensure correct repository is added:
```bash
helm repo add nvidia-nemo https://helm.ngc.nvidia.com/nvidia-nemo
helm repo update
helm search repo nvidia-nemo/nemo-operator
```

**Success Criteria:**
- ✅ NeMo Operator pod running and healthy (2/2 containers)
- ✅ All NeMo CRDs installed and available
- ✅ Operator responding to custom resource changes
- ✅ No error logs in operator container

**Important Notes:**
- **Purpose**: NeMo Operator manages the **infrastructure and training** aspects of NEMO
- **Sequence**: Install NeMo Operator **before** NIM Operator
- **Dependencies**: NeMo Operator works with the infrastructure components deployed via Ansible
- **Complementary**: Works alongside NIM Operator for complete NEMO ecosystem

## NIM Operator Installation (v3.0.1)

The NVIDIA NIM Operator v3.0.1 is required for deploying, scaling, and serving models for inference in production. This operator works **alongside** the NeMo Operator and should be installed **second**.

**⚠️ Critical Architecture Note**: The NIM Operator actually manages the NEMO microservice workloads (nemocustomizer, nemodatastore, nemoentitystore, nemoguardrail, nemoevaluator), while the NeMo Operator handles training jobs (nemotrainingjobs, nemovalidationjobs). The NIM operator may experience restarts on some clusters due to cluster health issues.

### Prerequisites {#nim-operator-prerequisites}

Before installing the NIM Operator, ensure you have:

1. **NeMo Operator Installed**: The NeMo Operator v25.06 must be installed first:
   - ✅ NeMo Operator pod running and healthy (2/2 containers)
   - ✅ All NeMo CRDs available (nemocustomizers, nemodatastores, etc.)
   - ✅ Operator responding to custom resource changes

2. **All NEMO Dependencies Deployed**: Complete infrastructure from previous sections:
   - ✅ Datastore, Entity-store, Customizer, Evaluator, Guardrail components
   - ✅ PostgreSQL databases, OpenTelemetry, MLflow, Argo Workflows, Milvus
   - ✅ All services should be in "Ready" state

3. **NGC Credentials**: Valid NVIDIA GPU Cloud API key
4. **OpenShift Cluster**: Version 4.x with sufficient resources
5. **Target Namespace**: Use existing `<your-namespace>` from dependency deployment
6. **Helm**: Version 3.x installed and configured

**Pre-Installation Verification:**
```bash
# Verify NeMo Operator is running
oc get pods -n <your-namespace> | grep nemo-operator

# Verify all dependencies are ready
oc get pods -n <your-namespace> | grep -E "(postgresql|opentelemetry|mlflow|argo|milvus)"

# Should show both operators and all infrastructure services running
```

### NIM Operator Deployment

1. **Create Required Secrets**:
   ```bash
   # Create NGC API secret for NEMO services
   oc create secret generic ngc-api-secret \
     --from-literal=NGC_API_KEY=<YOUR_NGC_API_KEY> \
     -n <your-namespace>

   # Create NGC image pull secret
   oc create secret docker-registry ngc-secret \
     --docker-server=nvcr.io \
     --docker-username='$oauthtoken' \
     --docker-password=<YOUR_NGC_API_KEY> \
     -n <your-namespace>
   ```

2. **Use OpenShift-Specific Helm Chart**:
   ```bash
   # CRITICAL: OpenShift requires the local Helm chart from deploy-v3.0-on-openshift branch
   # The official NVIDIA Helm repository chart is NOT compatible with OpenShift
   cd /tmp/k8s-nim-operator/test/e2e/nemo-dependencies
   ```

3. **Install NIM Operator v3.0.1 (OpenShift-Compatible)**:
   ```bash
   # Install NIM Operator using OpenShift-specific Helm chart
   helm install k8s-nim-operator /tmp/k8s-nim-operator/deployments/helm/k8s-nim-operator \
     -n <your-namespace> \
     --set operator.resources.limits.memory=512Mi \
     --set operator.resources.requests.memory=256Mi \
     --wait --timeout=300s
   ```

4. **Verify Operator Installation**:
   ```bash
   # Check operator pod status
   oc get pods -n <your-namespace> | grep k8s-nim-operator

   # Check operator logs
   oc logs -l app.kubernetes.io/name=k8s-nim-operator -n <your-namespace>

   # Verify CRDs are installed
   oc get crd | grep -E "(nimcache|nimpipeline|nemocustomizer)"
   ```

   Expected output:
   ```
   NAME                                         READY   STATUS    RESTARTS   AGE
   k8s-nim-operator-xxxx-xxxx                   1/1     Running   0          2m
   ```

**⚠️ Important**: The NIM operator requires namespace scoping (`WATCH_NAMESPACE`) for OpenShift stability. This operator manages the actual NEMO microservice workloads.


### Verification {#nim-operator-verification}

**Check NEMO Resource Status:**
```bash
# Verify all custom resources
oc get nemocustomizer,nemodatastore,nemoentitystore,nemoevaluator,nemoguardrail,nimcache,nimpipeline -n <your-namespace>
```

Expected output:
```
NAME                                                   STATUS   AGE
nemocustomizer.apps.nvidia.com/nemocustomizer-sample   Ready    10m
nemodatastore.apps.nvidia.com/nemodatastore-sample     Ready    10m
nemoentitystore.apps.nvidia.com/nemoentitystore-sample Ready    10m
nemoevaluator.apps.nvidia.com/nemoevaluator-sample     Ready    10m
nemoguardrail.apps.nvidia.com/nemoguardrails-sample    Ready    10m
nimcache.apps.nvidia.com/meta-llama3-1b-instruct       Ready    10m
nimpipeline.apps.nvidia.com/llama3-1b-pipeline         Ready    10m
```

**Test NEMO Service Endpoints:**
```bash
# Test Customizer API
oc port-forward svc/nemocustomizer-sample -n <your-namespace> 8000:8000 &
curl -X GET http://localhost:8000/health

# Test Datastore API
oc port-forward svc/nemodatastore-sample -n <your-namespace> 8001:8000 &
curl -X GET http://localhost:8001/health

# Test Entity Store API
oc port-forward svc/nemoentitystore-sample -n <your-namespace> 8002:8000 &
curl -X GET http://localhost:8002/health

# Test Guardrail API
oc port-forward svc/nemoguardrails-sample -n <your-namespace> 8003:8000 &
curl -X GET http://localhost:8003/health
```

**Test NIM Inference (when GPU nodes available):**
```bash
# Check NIM cache status
oc get nimcache meta-llama3-1b-instruct -n <your-namespace> -o jsonpath='{.status.state}'

# Test inference endpoint (when ready)
oc port-forward svc/meta-llama3-1b-instruct -n <your-namespace> 8080:8000 &
curl -X POST http://localhost:8080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "meta/llama-3.2-1b-instruct", "prompt": "Hello", "max_tokens": 10}'
```

### Troubleshooting NIM Operator

**Issue**: NIM Operator CrashLoopBackOff with OOMKilled
```bash
# Error: pod killed due to memory limit
```
- **Root Cause**: NIM operator requires significantly more memory than documented (8Gi vs 512Mi)
- **Solution**: Use high memory limits during installation:
```bash
helm upgrade k8s-nim-operator nvidia/k8s-nim-operator \
  -n <your-namespace> \
  --set manager.resources.limits.memory=8Gi \
  --set manager.resources.requests.memory=4Gi \
  --set manager.resources.limits.cpu=4 \
  --set manager.resources.requests.cpu=2
```

**Issue**: NIM Operator Excessive Restarts (Cluster Health Related)
```bash
# Symptoms: Multiple restarts vs stable behavior on other clusters
k8s-nim-operator-xxxxx   0/1   CrashLoopBackOff   6+   10m
```
- **Root Cause**: Cluster-level health issues affecting operator stability
- **Diagnosis Commands**:
```bash
# Check overall cluster health
oc get clusterversion
oc get co | grep DEGRADED
oc get nodes | grep -E "(NotReady|SchedulingDisabled)"
```
- **Known Problematic Configurations**:
  - OpenShift 4.18.x with degraded cluster operators
  - Kubernetes v1.31.x with machine-config operator in DEGRADED state
  - Clusters with nodes in NotReady/SchedulingDisabled status
- **Verified Working Configurations**:
  - OpenShift 4.19+ / Kubernetes v1.32+
  - All cluster operators healthy (no DEGRADED status)
  - All nodes in Ready state
- **Solutions**:
  - **Preferred**: Use healthy clusters with recent OpenShift/Kubernetes versions
  - **Workaround**: Accept restart behavior as operational (services may still function)
  - **Investigation**: Review cluster upgrade path if operator stability is critical

**Issue**: NEMO services show "NotReady" status
```bash
# Services created but not ready
```
- **Root Cause**: Missing NGC API secret
- **Solution**: Ensure NGC secrets are created:
```bash
oc get secret ngc-api-secret ngc-secret -n <your-namespace>
# If missing, recreate using commands in deployment section
```

**Issue**: NIM Cache pods stuck in Pending
```bash
# Error: no nodes available for GPU workloads
```
- **Root Cause**: NIM cache requires GPU nodes with proper scheduling
- **Expected Behavior**: This is normal in environments without GPU nodes
- **Solution**: Either deploy to GPU-enabled cluster or disable GPU-dependent resources for infrastructure testing

**Issue**: OpenTelemetry DNS resolution errors
```bash
# Error: StatusCode.UNAVAILABLE encountered while exporting logs to *.nemo.svc.cluster.local
```
- **Root Cause**: NEMO resources may reference wrong namespace in DNS
- **Status**: Services remain functional; this is a logging issue only
- **Impact**: Does not affect core functionality

**Issue**: Database connection failures
```bash
# Error: connection to database failed
```
- **Root Cause**: Database services not ready
- **Solution**: Verify all PostgreSQL services are running:
```bash
oc get pods -n <your-namespace> | grep postgresql
# All should show 1/1 Running status
```

**Performance Optimization:**
- **Memory**: Use at least 512Mi memory limit for NIM operator
- **Storage**: Ensure high-performance storage classes (gp3-csi) for database volumes
- **Networking**: Verify service mesh and DNS resolution for inter-service communication

**Success Criteria:**
- ✅ NIM Operator pod running and healthy
- ✅ 4-7 NEMO microservices showing "Ready" status
- ✅ All dependency services operational
- ✅ API endpoints responding to health checks
- ⚠️ GPU-dependent services may remain NotReady without GPU nodes (expected)

**Production Considerations:**
- Monitor operator resource usage and adjust limits as needed
- Implement proper TLS termination for API services
- Configure authentication and authorization for production workloads
- Set up monitoring and alerting for NEMO service health
- Regular backup of PostgreSQL databases and MLflow artifacts

## NEMO Samples Deployment

The NEMO Samples deployment creates the actual NEMO microservices using the infrastructure deployed earlier and the operators. This should be deployed after both the NeMo Operator and NIM Operator are successfully installed.

### Prerequisites

Before deploying NEMO samples, ensure you have:

1. **Both Operators Installed**: The NeMo Operator v25.06 and NIM Operator v3.0.1 must be running
2. **All Infrastructure Components Deployed**: Complete infrastructure from Infrastructure Components section
3. **NGC Credentials**: Valid NVIDIA GPU Cloud API key configured
4. **Target Namespace**: Use existing `<your-namespace>` from previous deployments

### Deploy NEMO Custom Resources

1. **Prepare Samples Configuration**:
   ```bash
   # Clone the official NIM operator repository and checkout v3.0.1
   cd /tmp
   git clone https://github.com/NVIDIA/k8s-nim-operator.git
   cd k8s-nim-operator && git checkout v3.0.1

   # Copy and modify samples for our namespace
   cp -r config/samples/nemo/latest /tmp/nemo-samples-modified
   cd /tmp/nemo-samples-modified
   sed -i "s/namespace: nemo/namespace: <your-namespace>/g" *.yaml
   sed -i "s/\.nemo\.svc\.cluster\.local/.<your-namespace>.svc.cluster.local/g" *.yaml
   ```

2. **Deploy NEMO Custom Resources**:
   ```bash
   # Apply all NEMO samples
   oc apply -n <your-namespace> -f /tmp/nemo-samples-modified/
   ```

   This deploys:
   - **NemoCustomizer**: Fine-tuning and model customization service
   - **NemoDatastore**: Data management and storage service
   - **NemoEntitystore**: Entity and model metadata management
   - **NemoEvaluator**: Model evaluation and benchmarking service
   - **NemoGuardrail**: Safety and content filtering service
   - **NIMCache**: Model caching (meta-llama3-1b-instruct)
   - **NIMPipeline**: Inference pipeline for Llama 3.2 1B model

3. **Monitor Deployment Progress**:
   ```bash
   # Watch NEMO resources status
   watch oc get nemocustomizer,nemodatastore,nemoentitystore,nemoevaluator,nemoguardrail,nimcache,nimpipeline -n <your-namespace>

   # Monitor pod creation
   watch oc get pods -n <your-namespace>
   ```

### Verification

The verification of NEMO samples deployment is covered in detail in the [NEMO Microservices Verification](#nemo-microservices-verification) section below.

## NEMO Microservices Verification

This section provides comprehensive verification steps to ensure all NEMO microservices are operational according to NVIDIA's official documentation standards.

### Overview

Following deployment of the NIM Operator and NEMO samples, perform these verification steps to confirm the complete NVIDIA NEMO ecosystem is functioning correctly:

1. **NEMO Microservices Status Check**
2. **ConfigMaps Verification**
3. **NIM Services Status Check**
4. **Service Endpoints Verification**
5. **API Endpoints Testing**

### Step 1: Check NEMO Microservices Status

Verify all NEMO microservices show `Ready` status:

```bash
oc get -n <your-namespace> nemoentitystore,nemodatastore,nemoguardrails,nemocustomizer,nemoevaluator
```

**Expected Output:**
```
NAME                                                     STATUS   AGE
nemoentitystore.apps.nvidia.com/nemoentitystore-sample   Ready    22h
nemodatastore.apps.nvidia.com/nemodatastore-sample       Ready    22h
nemoguardrail.apps.nvidia.com/nemoguardrails-sample      Ready    22h
nemocustomizer.apps.nvidia.com/nemocustomizer-sample     Ready    22h
nemoevaluator.apps.nvidia.com/nemoevaluator-sample       Ready    22h
```

**✅ Success Criteria:** All 5 NEMO microservices showing `STATUS: Ready`

### Step 2: Verify ConfigMaps

Check for NEMO-related ConfigMaps:

```bash
oc get -n <your-namespace> configmap | grep "nemo"
```

**Expected Output:**
```
nemo-model-config                              2      22h
nemo-training-config                           1      22h
nemocustomizer-sample                          1      22h
```

**✅ Success Criteria:** All expected ConfigMaps present with data entries

### Step 3: Check NIM Services Status

Verify NIM Pipeline, Cache, and Service components:

```bash
oc get -n <your-namespace> nimpipeline,nimcache,nimservice
```

**Expected Output:**
```
NAME                                             STATUS   AGE
nimpipeline.apps.nvidia.com/llama3-1b-pipeline   Ready    22h

NAME                                               STATUS   PVC                           AGE
nimcache.apps.nvidia.com/meta-llama3-1b-instruct   Ready    meta-llama3-1b-instruct-pvc   22h

NAME                                                 STATUS   AGE
nimservice.apps.nvidia.com/meta-llama3-1b-instruct   Ready    22h
```

**✅ Success Criteria:** All NIM components showing `STATUS: Ready`

### Step 4: Verify Service Endpoints

Check that all services have proper ClusterIP endpoints:

```bash
# Check NEMO microservices
oc get services -n <your-namespace> | grep "nemo"

# Check NIM inference service
oc get services -n <your-namespace> | grep "meta-llama3-1b-instruct"
```

**Expected Output:**
```
# NEMO Microservices (all on port 8000/TCP)
nemocustomizer-sample                     ClusterIP   172.30.134.54    <none>        8000/TCP,9009/TCP
nemodatastore-sample                      ClusterIP   172.30.89.50     <none>        8000/TCP
nemoentitystore-sample                    ClusterIP   172.30.232.110   <none>        8000/TCP
nemoevaluator-sample                      ClusterIP   172.30.4.133     <none>        8000/TCP
nemoguardrails-sample                     ClusterIP   172.30.82.148    <none>        8000/TCP

# NIM Inference Service
meta-llama3-1b-instruct                   ClusterIP   172.30.62.18     <none>        8000/TCP
```

**✅ Success Criteria:** All services properly exposed with ClusterIP endpoints on port 8000

### Step 5: Test API Endpoints

Test each microservice API to verify functionality. Create temporary test pods to validate API responses:

#### Test NeMo Customizer API

```bash
oc run test-nemo-customizer --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://nemocustomizer-sample.<your-namespace>:8000/v1/customization/configs"

# Get results and cleanup
sleep 5 && oc logs test-nemo-customizer -n <your-namespace> && oc delete pod test-nemo-customizer -n <your-namespace>
```

**Expected:** JSON response with customization configurations for available models (llama-3.2-1b, llama-3.1-8b with training options)

#### Test NeMo Entity Store API

```bash
oc run test-nemo-entitystore --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://nemoentitystore-sample.<your-namespace>:8000/v1/base-urls"

# Get results and cleanup
sleep 5 && oc logs test-nemo-entitystore -n <your-namespace> && oc delete pod test-nemo-entitystore -n <your-namespace>
```

**Expected:** JSON response with datastore base URLs

#### Test NeMo Data Store API

```bash
oc run test-nemo-datastore --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://nemodatastore-sample.<your-namespace>:8000/v1/hf/api/datasets"

# Get results and cleanup
sleep 5 && oc logs test-nemo-datastore -n <your-namespace> && oc delete pod test-nemo-datastore -n <your-namespace>
```

**Expected:** JSON array (empty `[]` if no datasets configured)

#### Test NeMo Guardrails API

```bash
oc run test-nemo-guardrails --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://nemoguardrails-sample.<your-namespace>:8000/v1/guardrail/configs"

# Get results and cleanup
sleep 5 && oc logs test-nemo-guardrails -n <your-namespace> && oc delete pod test-nemo-guardrails -n <your-namespace>
```

**Expected:** JSON response with pagination structure (empty data array if no guardrails configured)

#### Test NeMo Evaluator API

```bash
oc run test-nemo-evaluator --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://nemoevaluator-sample.<your-namespace>:8000/v1/evaluation/configs"

# Get results and cleanup
sleep 5 && oc logs test-nemo-evaluator -n <your-namespace> && oc delete pod test-nemo-evaluator -n <your-namespace>
```

**Expected:** JSON response with pagination structure (empty data array if no evaluation configs)

#### Test NIM Inference Service

```bash
oc run test-nim-service --image=curlimages/curl:latest -n <your-namespace> --restart=Never -- curl -X GET "http://meta-llama3-1b-instruct.<your-namespace>:8000/v1/health/ready"

# Get results and cleanup
sleep 5 && oc logs test-nim-service -n <your-namespace> && oc delete pod test-nim-service -n <your-namespace>
```

**Expected:** `{"object":"health.response","message":"Service is ready."}`

### Verification Summary

**✅ Complete Success Criteria:**

| Component | Verification | Expected Result |
|-----------|-------------|-----------------|
| **NEMO Microservices** | Status check | 5 services showing `Ready` |
| **ConfigMaps** | Resource check | 3 ConfigMaps with proper data |
| **NIM Services** | Status check | Pipeline, Cache, Service all `Ready` |
| **Service Endpoints** | Network check | All services on ClusterIP:8000 |
| **API Functionality** | HTTP requests | All APIs responding with valid JSON |

**🚀 Deployment Status: FULLY OPERATIONAL**

When all verification steps pass, the complete NVIDIA NEMO ecosystem is successfully deployed and ready for:
- **Model Training & Fine-tuning** (Customizer)
- **Data Management** (Datastore & Entity Store)
- **Model Evaluation** (Evaluator)
- **Safety & Content Filtering** (Guardrails)
- **Production Inference** (NIM Service)

### Troubleshooting Verification Issues

**Issue**: Services show `NotReady` status
- **Check**: Pod logs for specific errors: `oc logs -l app.kubernetes.io/name=<service-name> -n <your-namespace>`
- **Verify**: All infrastructure dependencies (PostgreSQL, MLflow, etc.) are running

**Issue**: API tests fail with connection refused
- **Check**: Service endpoints: `oc get svc -n <your-namespace>`
- **Verify**: Pods are Running: `oc get pods -n <your-namespace>`

**Issue**: GPU-dependent services pending
- **Expected**: NIM cache/service may remain pending without GPU nodes
- **Solution**: Add tolerations for GPU node taints (see [NIM cache pod stuck pending on GPU nodes](#pod-stuck-in-pending))

## Architectural Decisions & Component Analysis

This section documents the analysis and decisions made regarding component enablement/disablement for NVIDIA NIM Operator v3.0.0 compatibility and cluster safety.

### Volcano Batch Scheduler Analysis

**Updated Decision**: Volcano is **REQUIRED** for NeMo Operator v25.06 and must be installed with proper OpenShift SCC configuration.

**Why Volcano was Originally Included:**
- **Batch Job Scheduling**: Advanced scheduling for GPU-intensive AI/ML workloads
- **Gang Scheduling**: Coordinated scheduling of distributed training jobs
- **Resource Management**: Efficient GPU sharing and queue management
- **NVIDIA Ecosystem**: Common component in NVIDIA's AI platform stack

**Critical Discovery**: NeMo Operator v25.06 **REQUIRES** Volcano for PodGroup CRDs and will fail without it.

**OpenShift-Specific Requirements:**
- **Volcano scheduler needs privileged SCC** for hostPath volumes (`/tmp/klog-socks`)
- **Without proper SCC**: volcano-scheduler pod fails with security constraint violations
- **Without Volcano**: NeMo operator fails with "no matches for kind 'PodGroup'"

**Tested Working Configuration:**
```bash
# Install Volcano with proper OpenShift configuration
helm install volcano volcano/volcano --namespace <your-namespace> --version 1.9.0
oc adm policy add-scc-to-user privileged system:serviceaccount:<your-namespace>:volcano-scheduler
oc rollout restart deployment volcano-scheduler -n <your-namespace>
```

**Verified Impact Assessment:**
| Component | Actual Risk | Impact Scope | Solution |
|-----------|-------------|--------------|----------|
| **Volcano Installation** | 🟢 **Low** - Namespace-scoped deployment | Target namespace only | Use proper SCC configuration |
| **NeMo Operator Without Volcano** | 🚨 **Critical** - Complete failure | NeMo operator non-functional | Install Volcano first |
| **Missing SCC for volcano-scheduler** | 🟡 **Medium** - Pod stuck pending | Volcano scheduler only | Grant privileged SCC |

**Updated Functional Impact Assessment:**
| Workload Type | Impact without Volcano | Status |
|---------------|------------------------|--------|
| **Infrastructure Services** (MLflow, PostgreSQL, etc.) | 🟢 **No impact** - Standard Kubernetes scheduler sufficient | ✅ Verified working |
| **NeMo Operator** | 🚨 **Critical failure** - Cannot start without PodGroup CRDs | ❌ Completely blocked |
| **GPU Inference (NIMs)** | 🟠 **Blocked** - Depends on NeMo operator being functional | ❌ Cannot deploy without NeMo operator |
| **Distributed Training** | 🟠 **Blocked** - Depends on NeMo operator being functional | ❌ Cannot deploy without NeMo operator |

**Updated Requirements:**
- **Always required**: For any NVIDIA NeMo Operator v25.06 deployment
- **OpenShift clusters**: Must grant privileged SCC to volcano-scheduler
- **Production environments**: Volcano provides advanced GPU scheduling capabilities needed by NeMo

### Bitnami Init Container Analysis

**Decision**: Multiple Bitnami-specific init containers are **disabled** for compatibility.

**Components Disabled:**
```yaml
# MLflow Configuration
minio:
  enableDefaultInitContainers: false
  provisioning:
    enabled: false

postgresql:
  enableDefaultInitContainers: false

tracking:
  enableDefaultInitContainers: false

volumePermissions:
  enabled: false
```

**Why Bitnami Init Containers were Disabled:**

**1. Missing Utility Dependencies:**
```bash
# Error encountered:
/bin/bash: line 4: wait-for-port: command not found
```
- **Cause**: Bitnami charts expect proprietary utilities (`wait-for-port`, `wait-for-available-minio`)
- **Alternative Images**: Standard images (python:3.9-slim, quay.io/minio/minio) lack these utilities
- **Solution**: Disable dependency checking init containers

**2. OpenShift Security Context Conflicts:**
```bash
# Volume permission errors in OpenShift
```
- **Cause**: OpenShift automatically handles security contexts and file permissions
- **Bitnami Approach**: Uses privileged init containers for volume setup
- **Solution**: Disable volume permission init containers (`volumePermissions.enabled: false`)

**3. Registry Rate Limiting Avoidance:**
```bash
# Docker Hub rate limiting
toomanyrequests: You have reached your unauthenticated pull rate limit
```
- **Alternative Strategy**: Use multiple registries (docker.io, quay.io, ghcr.io)
- **Bitnami Images**: Often subject to rate limiting on Docker Hub
- **Solution**: Mix of registries with alternative image sources

**Functional Impact Assessment:**

| Feature | Impact without Init Containers | Risk Level | Mitigation |
|---------|-------------------------------|------------|------------|
| **Database Readiness** | 🟡 Services may start before DB ready | Medium | Application-level retries |
| **Volume Permissions** | 🟢 Appears to work - OpenShift handles automatically | Low | None needed |
| **Bucket Provisioning** | 🟡 Manual setup required (verified) | Medium | Post-deployment scripts |
| **Dependency Ordering** | 🟡 Race conditions possible (not fully tested) | Medium | Health checks in applications |

### Trade-offs Summary

**✅ Benefits of Current Configuration:**
- **Cluster Stability**: Reduces risk of cluster-wide webhook failures
- **Deployment Reliability**: Avoids image pull failures from rate limiting
- **OpenShift Compatibility**: Works with OpenShift security contexts
- **Operational Safety**: Namespaced impact only, minimizes cluster-wide changes

**⚠️ Limitations of Current Configuration:**
- **Manual Setup**: MinIO buckets require manual creation
- **Basic Scheduling**: No advanced GPU scheduling for compute workloads (untested impact)
- **Reduced Automation**: Some Bitnami convenience features disabled
- **Startup Dependencies**: Services may start before dependencies fully ready (potential race conditions)

**🎯 Recommendation Matrix:**

| Use Case | Configuration | Justification |
|----------|--------------|---------------|
| **Development/Testing** | Keep current (volcano disabled) | Safety and simplicity prioritized |
| **Production Infrastructure** | Keep current | Proven stability for API services |
| **GPU Inference Workloads** | Selectively enable volcano | Enable only for compute-intensive NIMs |
| **Distributed Training** | Enable volcano + init containers | Full functionality needed |

### Re-enabling Components Safely

**For Volcano (when needed):**
```yaml
# install.yaml - Enable only for specific workloads
volcano:
  enabled: true
  # Consider namespace-scoped deployments where possible
```

**For Bitnami Init Containers (if needed):**
```yaml
# Use alternative registries to avoid rate limiting
postgresql:
  enableDefaultInitContainers: true
  initImage:
    registry: ghcr.io  # Alternative registry
    repository: bitnami/postgresql
```

**Best Practices for Re-enabling:**
1. **Test in non-production** clusters first
2. **Enable per-namespace** where possible
3. **Monitor cluster-wide resources** for conflicts
4. **Have rollback plan** ready
5. **Document changes** and impact assessment

### Summary: Current vs. Original v3.0.0 Configuration

**✅ What's Currently Working:**
- **Core Infrastructure Services**: All dependency services (PostgreSQL, OpenTelemetry, MLflow) deployed and running
- **MLflow Tracking**: Full experiment tracking with MinIO artifact storage
- **Data Services**: Datastore, Entity-store, Customizer, Guardrail components deployed
- **Evaluation Pipeline**: Evaluator with Argo Workflows, Milvus vector database, and namespace-scoped security
- **Development Workflow**: Jupyter notebooks for AI/ML development
- **Basic Functionality**: Infrastructure services operate as expected

**⚠️ What's Modified for Safety:**
- **Volcano Scheduler**: Disabled cluster-wide to prevent webhook conflicts
- **Bitnami Init Containers**: Disabled to avoid utility dependencies
- **Automatic Provisioning**: Manual MinIO bucket setup required

**🎯 Net Impact Assessment:**
```
Infrastructure Services: Working as expected ✅
GPU Scheduling: Impact unknown - requires testing with actual NIMs ⚠️
Cluster Stability: Improved for webhook-related issues ✅
Setup Complexity: Increased due to manual steps ⚠️
```

**Important Limitations:**
- **Untested with actual NIMs**: Impact on GPU inference workloads not verified
- **No advanced scheduling**: Distributed training capabilities not available
- **Manual setup required**: Some operational convenience lost

**Recommended Approach:**
1. **Use current configuration** for infrastructure deployment
2. **Test with actual NIM workloads** to identify any gaps
3. **Enable additional components** based on actual requirements
4. **Document any issues** encountered with real workloads

**This configuration provides a stable foundation but requires validation with actual NVIDIA NIM Operator v3.0.0 workloads to confirm full compatibility.**

## Common Troubleshooting {#common-troubleshooting}

### Evaluator-Specific Issues

**Issue**: Evaluator PostgreSQL fails to deploy with template generation errors
```bash
# Error: Template file not found or configuration file not generated
```
- **Root Cause**: Copy-paste error in `/evaluator/tasks/postgresql.yaml` using wrong destination filename
- **Symptoms**: `evaluator-postgresql-values.yaml` file not created, no `evaluator-pg` Helm release
- **Solution**: Ensure `dest: evaluator-postgresql-values.yaml` (not `ds-postgresql-values.yaml`) in postgresql.yaml task file

**Issue**: Evaluator OpenTelemetry deployment fails with version mismatch
```bash
# Error: Chart version conflicts or deployment timeout
```
- **Root Cause**: Evaluator uses outdated OpenTelemetry chart version `0.78.1` vs working `0.93.3`
- **Solution**: Update `/evaluator/defaults/main.yml` to use `chart_version: "0.93.3"`

**Issue**: MLflow PostgreSQL fails with non-existent image tag
```bash
# Error: initializing source docker://bitnami/postgresql:16.3.0-debian-12-r5: manifest unknown
```
- **Root Cause**: MLflow chart v1.0.6 defaults to non-existent PostgreSQL image tag
- **Solution**: Add image override to `/customizer/templates/mlflow.yaml.j2`:
```yaml
postgresql:
  enabled: true
  image:
    registry: registry-1.docker.io
    repository: bitnami/postgresql
    tag: latest
```

**Issue**: Evaluator OpenTelemetry pod crashes with configuration errors
```bash
# Error: unknown type: "zipkin" for id: "zipkin" (valid values: [nop otlp otlphttp file loadbalancing debug])
```
- **Root Cause**: OpenTelemetry collector doesn't support zipkin exporter in current version
- **Solution**: Update `/evaluator/templates/opentelemetry-values.yaml.j2` with compatible configuration:
```yaml
mode: deployment
image:
  repository: "otel/opentelemetry-collector-k8s"
  tag: "0.102.1"
config:
  exporters:
    debug:
      verbosity: detailed
  # Remove zipkin exporter and incompatible processors
```

**Issue**: Milvus connectivity check fails on repeated deployments
```bash
# Error: pods "milvus-check" already exists
```
- **Root Cause**: Leftover pod from previous deployment attempts
- **Solution**: Add cleanup step to `/evaluator/tasks/milvus.yaml`:
```yaml
- name: Clean up any existing milvus-check pod
  command: kubectl delete pod milvus-check -n {{ namespace }} --ignore-not-found=true
  changed_when: false
```

**Issue**: OpenTelemetry template missing required image configuration
```bash
# Error: 'image.repository' must be set
```
- **Root Cause**: Evaluator OpenTelemetry template lacks image specification that other components have
- **Solution**: Add image configuration to `/evaluator/templates/opentelemetry-values.yaml.j2`

### MLflow-Specific Issues

**Issue**: MinIO provisioning job fails with "wait-for-available-minio" error
```bash
# Error: Back-off restarting failed container wait-for-available-minio in pod mlflow-minio-provisioning-xxx
```
- **Cause**: Bitnami chart expects `wait-for-port` utility not available in standard images
- **Solution**: Bucket provisioning is disabled in Customizer Option 2. Use manual bucket setup commands in the [Customizer Post-Deployment Setup](#customizer-post-deployment-setup-for-minio-option-2) section.

**Issue**: MLflow init containers fail with "wait-for-port: command not found"
```bash
# Error: /bin/bash: line 4: wait-for-port: command not found
```
- **Cause**: Bitnami-specific utilities missing in alternative images
- **Solution**: All init containers are disabled in Customizer Option 2 configuration (`enableDefaultInitContainers: false`)

**Issue**: Docker Hub rate limiting during image pulls
```bash
# Error: toomanyrequests: You have reached your unauthenticated pull rate limit
```
- **Solution**: Customizer Option 2 uses alternative registries:
  - MinIO: `quay.io/minio/minio:latest`
  - MLflow: `docker.io/library/python:3.9-slim` (cached/commonly available)
  - PostgreSQL: `docker.io/bitnami/postgresql:latest` (generally available)

**Issue**: Volume permissions errors in OpenShift
- **Cause**: OpenShift security contexts restrict file permissions
- **Solution**: Volume permissions init containers are disabled (`volumePermissions.enabled: false`)
- **Note**: OpenShift handles security contexts automatically

### Pod Stuck in Pending

**Issue**: Multiple default storage classes
```bash
oc get storageclass
# If you see multiple "(default)" classes, remove conflicts:
oc patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

**Issue**: Storage access mode
- Error: `does not support access mode ReadWriteMany`
- Solution: Use `ReadWriteOnce` in `values.yaml`

**Issue**: NIM cache pod stuck pending on GPU nodes
```bash
# Error: 0/15 nodes are available: 7 node(s) had untolerated taint {g5-gpu: true}
```
- **Root Cause**: GPU nodes have taints that NIM cache pod cannot tolerate
- **Symptoms**: `meta-llama3-1b-instruct-pod` shows `FailedScheduling` events
- **Solution**: Add toleration for GPU node taints to NIMCache resource:
```bash
# Add GPU node toleration to NIMCache
oc patch nimcache meta-llama3-1b-instruct -n <your-namespace> --type='merge' \
  -p='{"spec":{"tolerations":[{"key":"g5-gpu","operator":"Equal","value":"true","effect":"NoSchedule"}]}}'

# Delete existing pod to recreate with new toleration
oc delete pod meta-llama3-1b-instruct-pod -n <your-namespace>

# Verify pod is now scheduled on GPU node
oc get pods -n <your-namespace> | grep llama
```
- **Expected Result**: Pod transitions from Pending to Running on GPU node
- **Note**: Adjust taint key/value to match your cluster's GPU node taints

## Notes

- **Dual-Operator Architecture**: Based on both NeMo Operator v25.06 and NIM Operator v3.0.1
  - **NeMo Operator**: Manages infrastructure, training, and customization (Development & Fine-Tuning)
  - **NIM Operator**: Manages inference, scaling, and production deployment (Inference & Production)
- Tested on OpenShift 4.x with AWS EBS storage
- PostgreSQL StatefulSet uses 500Mi EBS volume (per component)
- Database credentials are auto-generated in `secrets.yaml`
- Evaluator uses namespace-scoped Argo Workflows (enhanced security model)
- All components support standalone deployment or combined installation
- Both operators require 512Mi memory limit (increased from default 256Mi)
- NEMO microservices successfully deployed and operational (5/5 Ready status achieved)
- GPU-dependent services (NIM cache/pipeline) require GPU nodes for full functionality
- **Installation Order**: Deploy infrastructure → NeMo Operator → NIM Operator → NEMO Samples

## Security Considerations

⚠️ **IMPORTANT**: This configuration uses default credentials for development/testing purposes.

### Production Security Checklist:

**For MinIO (MLflow Option 2):**
- [ ] Change default MinIO credentials (`minioadmin/minioadmin`)
- [ ] Use strong passwords for `rootUser` and `rootPassword`
- [ ] Consider using Kubernetes secrets for credentials
- [ ] Review bucket access policies

**For Jupyter:**
- [ ] Change default token (`token`) to a strong, random value
- [ ] Consider implementing proper authentication (OAuth, LDAP)
- [ ] Review notebook access permissions
- [ ] Enable HTTPS for production access

**For Evaluator (Argo Workflows + Milvus):**
- [ ] Review auto-generated evaluator database passwords in `secrets.yaml`
- [ ] Configure Argo Workflows server authentication for production
- [ ] Secure Milvus vector database access with proper authentication
- [ ] Review workflow execution permissions and service account scope
- [ ] Enable TLS for Argo Workflows server in production
- [ ] Configure OpenTelemetry collector security for metrics collection

**For All Components:**
- [ ] Review auto-generated database passwords in `secrets.yaml`
- [ ] Use strong, unique passwords for each component
- [ ] Implement network policies to restrict inter-pod communication
- [ ] Enable TLS/SSL for database connections
- [ ] Regular security updates for all container images

**File Security:**
- [ ] Ensure `secrets.yaml` is never committed to version control
- [ ] Restrict access to generated configuration files
- [ ] Use Kubernetes RBAC to limit service account permissions

### Development vs Production:

| Aspect | Development (Current) | Production (Recommended) |
|--------|----------------------|-------------------------|
| **MinIO Credentials** | `minioadmin/minioadmin` | Strong, unique credentials |
| **Jupyter Token** | `token` | Strong, random token or OAuth |
| **Database Passwords** | Auto-generated | Strong, managed secrets |
| **TLS/SSL** | Not configured | Required for all services |
| **Network Policies** | Open | Restricted communication |

## OpenShift Compatibility Adaptations

This section documents the specific modifications and workarounds implemented to make NVIDIA's official NeMo deployment work on OpenShift. These adaptations address platform-specific requirements, security constraints, and compatibility issues encountered during the porting process.

### Key Differences Summary

| Category | NVIDIA O/fficial | OpenShift Implementation | Key Changes |
|----------|----------------|-------------------------|-------------|
| **Default Namespace** | `nemo` | `<your-namespace>` (configurable) | Namespace flexibility for multi-tenant environments |
| **Storage Class** | `local-path` provisioner | `gp3-csi` (AWS EBS) | Enterprise-grade persistent storage |
| **Volume Access Mode** | Default (ReadWriteMany) | `ReadWriteOnce` | AWS EBS compatibility |
| **MLflow Installation** | Standard Bitnami chart | Custom runtime installation | OpenShift security context compliance |
| **MinIO Image** | Bitnami MinIO | `quay.io/minio/minio:latest` | Official MinIO image for reliability |
| **Init Containers** | Enabled (Bitnami) | Disabled | Avoid Bitnami utility dependencies |
| **Volcano Scheduler** | Optional | Required with privileged SCC | NeMo Operator dependency requirement |
| **Operators Required** | NIM Operator only | NeMo Operator + NIM Operator | Dual-operator architecture for complete functionality |

### Container Image Modifications

#### MLflow Component
**Official NVIDIA**: Uses standard Bitnami MLflow Helm chart with default images
**OpenShift Changes**:
```yaml
MLflow Server: docker.io/library/python:3.9-slim
PostgreSQL: docker.io/bitnami/postgresql:latest
Installation Method: pip install --prefix=/tmp/pip-install mlflow==2.12.2 psycopg2-binary
Reason: Work around read-only filesystem restrictions in OpenShift security contexts
```

#### MinIO Component
**Official NVIDIA**: Standard Bitnami MinIO setup
**OpenShift Changes**:
```yaml
minio:
  image:
    registry: quay.io
    repository: minio/minio
    tag: latest
  enableDefaultInitContainers: false
  provisioning:
    enabled: false
```

### OpenShift-Specific Configurations

#### Storage Configuration
**Storage Changes**:
```yaml
pvc:
  storage_class: "gp3-csi"           # AWS EBS instead of local-path
  volume_access_mode: ReadWriteOnce  # Not ReadWriteMany
localPathProvisioner:
  enabled: false                     # Use EBS instead of local-path
```

#### Security Context Constraints
**Required SCC Configurations**:
```bash
# Volcano scheduler requires privileged SCC for hostPath volumes
oc adm policy add-scc-to-user privileged system:serviceaccount:<your-namespace>:volcano-scheduler

# Service account patching for NGC access
oc patch serviceaccount nemo-operator-controller-manager -p '{"imagePullSecrets": [{"name": "ngc-secret"}]}'
```

#### GPU Node Tolerations
**Added for OpenShift GPU Clusters**:
```yaml
tolerations:
  - key: "g5-gpu"              # OpenShift-specific GPU node taint
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  - key: "nvidia.com/gpu"      # Standard NVIDIA GPU taint
    operator: "Exists"
    effect: "NoSchedule"
```

### Deployment Process Differences

#### Operator Installation Sequence
**NVIDIA Official**: Single operator deployment
**OpenShift Implementation**: Dual-operator sequence
```
1. Infrastructure Components (Ansible)
2. Volcano Scheduler (Required for NeMo Operator)
3. NeMo Operator v25.06 (Infrastructure management)
4. NIM Operator v3.0.1 (Microservice workloads)
5. NEMO Samples (Custom resources)
```

#### Helm Chart Sources
**NVIDIA Official**: NVIDIA NGC Helm repository
**OpenShift Changes**:
- **NeMo Operator**: Uses NGC Helm repository
- **NIM Operator**: Uses local chart from deploy-v3.0-on-openshift branch
- **Reason**: Official NGC chart not compatible with OpenShift

### Disabled Components for Compatibility

#### Bitnami Init Containers
**Components Disabled**:
```yaml
minio:
  enableDefaultInitContainers: false
  provisioning:
    enabled: false
postgresql:
  enableDefaultInitContainers: false
volumePermissions:
  enabled: false
```

**Reasons for Disabling**:
1. **Missing Utilities**: Bitnami charts expect proprietary utilities (`wait-for-port`, `wait-for-available-minio`)
2. **Security Context Conflicts**: OpenShift handles security contexts automatically
3. **Registry Rate Limiting**: Avoid Docker Hub rate limiting issues

### Manual Setup Requirements

#### Post-Deployment Steps Not Required in Official
1. **MinIO Bucket Creation**: Manual setup required
2. **Service Account Patching**: NGC image pull secrets
3. **SCC Configuration**: Privileged permissions for Volcano
4. **Operator Resource Limits**: Memory increase to 512Mi

### Namespace Inconsistency Note

**Important Finding**: The nemo-samples.yaml file shows:
- **Namespace**: `arhkp-nemo`
- **DNS References**: Still uses `.nemo.svc.cluster.local`

This suggests the samples may require additional updates for complete namespace consistency.

### Verification Differences

#### NVIDIA Official
- Basic pod and service checks
- Single namespace (`nemo`) verification

#### OpenShift Implementation
- Multi-component verification across customizable namespace
- Additional SCC and toleration verification
- Dual-operator status checks
- API endpoint testing with curl pods

### Production Considerations

#### Additional OpenShift Requirements
- **Resource Quotas**: Consider namespace-level resource limits
- **Network Policies**: OpenShift-specific networking security
- **Image Registry**: Configure for enterprise image registries
- **Backup Strategy**: Account for EBS volume backups vs local storage
- **RBAC**: OpenShift-specific role-based access controls

### Sources and References

This comparison is based on analysis of:
- **OpenShift Documentation**: This README-OpenShift.md file
- **NVIDIA Official**: https://docs.nvidia.com/nim-operator/latest/nemo-prerequisites.html
- **NVIDIA Official**: https://docs.nvidia.com/nim-operator/latest/deploy-nemo-microservices.html
- **Sample Configuration**: nemo-samples.yaml in this directory

**Note**: This comparison reflects the current state of the OpenShift implementation and may require updates as both NVIDIA's official documentation and the OpenShift adaptation evolve.