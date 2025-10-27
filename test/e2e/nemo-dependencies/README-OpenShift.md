# NVIDIA NIM Operator E2E Dependencies - OpenShift Deployment

Quick guide for deploying NVIDIA NIM Operator E2E test dependencies on OpenShift.

## Table of Contents

- [Prerequisites](#prerequisites)

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

- [NeMo Operator Installation](#nemo-operator-installation)
  - [Prerequisites](#nemo-operator-prerequisites)
  - [Volcano Scheduler Installation](#volcano-scheduler-installation)
  - [NeMo Operator Deployment](#nemo-operator-deployment)
  - [NEMO Resources Deployment](#nemo-resources-deployment)
  - [Verification](#nemo-operator-verification)
  - [Troubleshooting NeMo Operator](#troubleshooting-nemo-operator)

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

## Datastore Component Deployment

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

## Entity-Store Component Deployment

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

## Customizer Component Deployment

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

## Jupyter Component Deployment

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

## Guardrail Component Deployment

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

## Evaluator Component Deployment

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

## NeMo Operator Installation

The NVIDIA NeMo Operator enables training and deployment of Large Language Models (LLMs) on Kubernetes. This section covers installing the operator in the `<your-namespace>` namespace with namespace-scoped volcano scheduler for enhanced cluster safety.

### Prerequisites {#nemo-operator-prerequisites}

Before installing the NeMo Operator, ensure you have:

1. **NGC API Key**: Valid NVIDIA GPU Cloud API key for accessing NeMo operator images
2. **OpenShift Cluster**: With sufficient GPU nodes for training workloads
3. **Namespace**: Use existing `<your-namespace>` namespace (already created for dependencies)
   - Replace `<your-namespace>` with your actual namespace name throughout this guide
4. **Dependencies Deployed**: All NEMO dependencies (datastore, entity-store, customizer, etc.) should be operational
5. **Helm**: Helm 3.x installed and configured

**NGC Credentials Setup:**
```bash
# Create NGC API secret for operator authentication
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

### Volcano Scheduler Installation

The NeMo Operator requires volcano scheduler for distributed training workloads. We install it with namespace-scoped webhook selectors to minimize cluster-wide impact.

1. **Install Volcano CRDs First** (Cluster-scoped requirement):
   ```bash
   # Install volcano CRDs only
   helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
   helm repo update

   # Install CRDs without the scheduler components
   oc apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml --dry-run=client -o yaml | grep -A1000 "kind: CustomResourceDefinition" | oc apply -f -
   ```

2. **Configure Namespace-Scoped Volcano**:

   Create `volcano-values.yaml`:
   ```yaml
   custom:
     admission_enable: true
     controller_enable: true
     scheduler_enable: true
     webhooks_namespace_selector_expressions:
       - key: "kubernetes.io/metadata.name"
         operator: "In"
         values:
           - "<your-namespace>"
   ```

3. **Deploy Volcano with Namespace Targeting**:
   ```bash
   # Install volcano with namespace-scoped webhook configuration
   helm install volcano volcano-sh/volcano \
     -n volcano-system \
     --create-namespace \
     -f volcano-values.yaml
   ```

4. **Verification**:
   ```bash
   # Check volcano components
   oc get pods -n volcano-system

   # Verify webhook is targeting our namespace
   oc get validatingwebhookconfiguration volcano-admission-webhook -o yaml | grep -A5 namespaceSelector
   ```

   Expected output:
   ```
   NAME                                READY   STATUS    RESTARTS   AGE
   volcano-admission-xxx               1/1     Running   0          2m
   volcano-controller-xxx              1/1     Running   0          2m
   volcano-scheduler-xxx               1/1     Running   0          2m
   ```

### NeMo Operator Deployment

1. **Add NVIDIA Helm Repository**:
   ```bash
   helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
   helm repo update
   ```

2. **Deploy NeMo Operator**:
   ```bash
   # Install NVIDIA NeMo Operator in AllNamespaces mode (required)
   helm install nemo-operator nvidia/nemo-operator \
     -n nemo-operator-system \
     --create-namespace \
     --set image.repository=nvcr.io/nvidia/nemo-operator \
     --set image.tag=25.08 \
     --set imagePullSecrets[0].name=ngc-secret \
     --wait --timeout=300s
   ```

3. **Verify Operator Installation**:
   ```bash
   # Check operator deployment
   oc get pods -n nemo-operator-system

   # Verify Custom Resource Definitions
   oc get crd | grep nemo

   # Check operator logs
   oc logs -n nemo-operator-system deployment/nemo-operator-controller-manager
   ```

   Expected CRDs:
   ```
   nemocaches.apps.nvidia.com
   nemocustomizers.apps.nvidia.com
   nemodatastores.apps.nvidia.com
   nemoentitystores.apps.nvidia.com
   nemoevaluators.apps.nvidia.com
   nemoguardrails.apps.nvidia.com
   nimcaches.apps.nvidia.com
   nimpipelines.apps.nvidia.com
   ```

### NEMO Resources Deployment

1. **Apply NEMO Custom Resources**:
   ```bash
   # Deploy all NEMO resources using the sample configuration
   # Note: Update the namespace references in your YAML file to match <your-namespace>
   oc apply -f /tmp/modified_samples.yaml
   ```

   This deploys:
   - **NemoCustomizer**: Fine-tuning and customization API service
   - **NemoDatastore**: Data management and storage service
   - **NemoEntitystore**: Entity and model metadata management
   - **NemoEvaluator**: Model evaluation and benchmarking service
   - **NemoGuardrail**: Safety and content filtering service
   - **NIMCache**: Model caching for inference optimization
   - **NIMPipeline**: Inference pipeline for Llama 3.2 1B Instruct model

2. **Monitor Resource Creation**:
   ```bash
   # Check NEMO custom resources
   oc get nemocustomizer,nemodatastore,nemoentitystore,nemoevaluator,nemoguardrail -n <your-namespace>

   # Check NIM resources
   oc get nimcache,nimpipeline -n <your-namespace>

   # Monitor pod creation
   watch oc get pods -n <your-namespace>
   ```

### Verification {#nemo-operator-verification}

**Complete System Verification:**

1. **Check All NEMO Resources**:
   ```bash
   # Verify all custom resources are created
   oc get nemocustomizer nemocustomizer-sample -n <your-namespace> -o yaml
   oc get nemodatastore nemodatastore-sample -n <your-namespace> -o yaml
   oc get nemoentitystore nemoentitystore-sample -n <your-namespace> -o yaml
   oc get nemoevaluator nemoevaluator-sample -n <your-namespace> -o yaml
   oc get nemoguardrail nemoguardrails-sample -n <your-namespace> -o yaml
   ```

2. **Verify Infrastructure Services**:
   ```bash
   # Check all dependency pods are running
   oc get pods -n <your-namespace> | grep -E "(postgresql|opentelemetry|mlflow|argo|milvus)"

   # Verify database connectivity
   oc exec datastore-pg-postgresql-0 -n <your-namespace> -- pg_isready
   oc exec entity-store-pg-postgresql-0 -n <your-namespace> -- pg_isready
   oc exec customizer-pg-postgresql-0 -n <your-namespace> -- pg_isready
   oc exec evaluator-pg-postgresql-0 -n <your-namespace> -- pg_isready
   oc exec guardrail-pg-postgresql-0 -n <your-namespace> -- pg_isready
   ```

3. **Test NEMO Service Endpoints**:
   ```bash
   # Port forward to test services locally
   oc port-forward svc/nemodatastore-sample -n <your-namespace> 8000:8000 &
   curl -X GET http://localhost:8000/health

   oc port-forward svc/nemoentitystore-sample -n <your-namespace> 8001:8000 &
   curl -X GET http://localhost:8001/health

   oc port-forward svc/nemocustomizer-sample -n <your-namespace> 8002:8000 &
   curl -X GET http://localhost:8002/health
   ```

4. **Verify NIM Pipeline**:
   ```bash
   # Check NIM cache status
   oc get nimcache meta-llama3-1b-instruct -n <your-namespace> -o jsonpath='{.status.state}'

   # Check NIM pipeline status
   oc get nimpipeline llama3-1b-pipeline -n <your-namespace> -o jsonpath='{.status.state}'

   # Test inference endpoint when ready
   oc port-forward svc/meta-llama3-1b-instruct -n <your-namespace> 8080:8000 &
   curl -X POST http://localhost:8080/v1/completions \
     -H "Content-Type: application/json" \
     -d '{"model": "meta/llama-3.2-1b-instruct", "prompt": "Hello", "max_tokens": 10}'
   ```

**Expected Final State:**
```bash
# All components should be running
NAME                                           READY   STATUS    RESTARTS   AGE
datastore-pg-postgresql-0                      1/1     Running   0          45m
entity-store-pg-postgresql-0                   1/1     Running   0          40m
customizer-pg-postgresql-0                     1/1     Running   0          35m
evaluator-pg-postgresql-0                      1/1     Running   0          30m
guardrail-pg-postgresql-0                      1/1     Running   0          25m
argo-workflows-server-xxx                      1/1     Running   0          30m
argo-workflows-workflow-controller-xxx         1/1     Running   0          30m
milvus-xxx                                     1/1     Running   0          30m
mlflow-tracking-xxx                            1/1     Running   0          35m
mlflow-minio-xxx                               1/1     Running   0          35m
opentelemetry-collector-xxx                    1/1     Running   0          30m
jupyter-notebook-xxx                           1/1     Running   0          20m
nemocustomizer-sample-xxx                      1/1     Running   0          15m
nemodatastore-sample-xxx                       1/1     Running   0          15m
nemoentitystore-sample-xxx                     1/1     Running   0          15m
nemoevaluator-sample-xxx                       1/1     Running   0          15m
nemoguardrails-sample-xxx                      1/1     Running   0          15m
meta-llama3-1b-instruct-xxx                    1/1     Running   0          10m
```

### Troubleshooting NeMo Operator

**Common NeMo Operator Issues:**

**Issue**: NeMo Operator pods stuck in ImagePullBackOff
```bash
# Error: Failed to pull image "nvcr.io/nvidia/nemo-operator:25.08"
```
- **Root Cause**: Missing or incorrect NGC credentials
- **Solution**: Verify NGC secrets are correctly created and contain valid API key:
```bash
oc get secret ngc-secret -n nemo-operator-system -o yaml
oc get secret ngc-api-secret -n <your-namespace> -o yaml
```

**Issue**: Volcano webhook affecting cluster-wide pod creation
```bash
# Error: admission webhook "volcano-admission-webhook" denied the request
```
- **Root Cause**: Volcano webhook misconfigured for cluster-wide operation
- **Solution**: Verify namespace selector in webhook configuration:
```bash
oc get validatingwebhookconfiguration volcano-admission-webhook -o yaml | grep -A10 namespaceSelector
```

**Issue**: NEMO resources stuck in "Pending" state
```bash
# Custom resources created but no pods appear
```
- **Root Cause**: Missing dependencies or operator not watching namespace
- **Solution**:
  1. Check operator logs: `oc logs -n nemo-operator-system deployment/nemo-operator-controller-manager`
  2. Verify all prerequisite services are running
  3. Check RBAC permissions for service accounts

**Issue**: NIM Pipeline fails to start inference service
```bash
# Error: Failed to mount NIM cache volume
```
- **Root Cause**: NIMCache not fully populated or storage issues
- **Solution**:
  1. Check NIMCache status: `oc get nimcache meta-llama3-1b-instruct -n <your-namespace> -o yaml`
  2. Verify storage class and PVC creation: `oc get pvc -n <your-namespace>`
  3. Monitor cache population: `oc logs -f <nimcache-pod> -n <your-namespace>`

**Issue**: Training jobs fail with volcano scheduler errors
```bash
# Error: volcano.batch/v1alpha1, Kind=Job
```
- **Root Cause**: Volcano CRDs not installed or incompatible version
- **Solution**:
  1. Verify volcano CRDs: `oc get crd | grep volcano`
  2. Check volcano controller logs: `oc logs -n volcano-system deployment/volcano-controller`
  3. Ensure volcano is properly scoped to <your-namespace> namespace

**Performance Considerations:**

- **GPU Resources**: Ensure sufficient GPU nodes for training workloads
- **Storage**: Use high-performance storage classes (e.g., gp3-csi) for model caching
- **Networking**: Verify InfiniBand/high-speed networking for multi-node training
- **Memory**: Training jobs require significant memory - monitor node resource availability

**Security Notes:**

⚠️ **IMPORTANT**: Replace `<YOUR_NGC_API_KEY>` with your actual NGC API key. The API key should be rotated regularly in production environments.

**Production Security Checklist:**
- [ ] Rotate NGC API keys regularly
- [ ] Use Kubernetes secrets with proper RBAC
- [ ] Enable TLS for all service communications
- [ ] Review volcano webhook scope and permissions
- [ ] Implement network policies for service isolation
- [ ] Monitor operator logs for security events

## Architectural Decisions & Component Analysis

This section documents the analysis and decisions made regarding component enablement/disablement for NVIDIA NIM Operator v3.0.0 compatibility and cluster safety.

### Volcano Batch Scheduler Analysis

**Decision**: Volcano is **disabled** (`volcano.enabled: false`) in the main installation workflow.

**Why Volcano was Originally Included:**
- **Batch Job Scheduling**: Advanced scheduling for GPU-intensive AI/ML workloads
- **Gang Scheduling**: Coordinated scheduling of distributed training jobs
- **Resource Management**: Efficient GPU sharing and queue management
- **NVIDIA Ecosystem**: Common component in NVIDIA's AI platform stack

**Why Volcano is Disabled:**
```yaml
# install.yaml override (CHANGED for safety)
volcano:
  enabled: false  # SAFETY: Disabled to prevent cluster-wide webhook issues
```

**Cluster-Wide Risk Analysis:**
| Component | Risk Level | Impact Scope | Consequence if Failed |
|-----------|------------|--------------|----------------------|
| **Admission Webhooks** | 🚨 Critical | Entire Cluster | All pod creation blocked |
| **ClusterRole/ClusterRoleBinding** | 🟡 Medium | Cluster-wide permissions | Security/access issues |
| **Custom Resource Definitions** | 🟡 Medium | Cluster-wide API | API extension conflicts |

**Real-World Impact Experienced:**
- **Orphaned webhooks** from previous volcano installation blocked entire cluster
- **All pod creation failed** across all namespaces due to webhook failures
- **Required cluster-wide intervention** to remove webhook configurations

**Functional Impact Assessment:**
| Workload Type | Impact without Volcano | Mitigation |
|---------------|------------------------|------------|
| **Infrastructure Services** (MLflow, PostgreSQL, etc.) | 🟢 **Verified: No impact** - Standard Kubernetes scheduler sufficient | Use default scheduler |
| **GPU Inference (NIMs)** | 🟡 **Unknown** - Basic GPU scheduling may work, requires testing | Enable volcano per-workload when needed |
| **Distributed Training** | 🟠 **Likely impact** - No gang scheduling available | Use Job/CronJob or enable volcano selectively |
| **Batch Processing** | 🟠 **Likely impact** - No advanced queueing | Implement application-level queueing |

**When to Re-enable Volcano:**
- Deploying compute-intensive NIMs requiring GPU scheduling
- Running distributed training workloads
- Need for advanced batch job management
- Production AI/ML pipeline requiring resource optimization

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

## Notes

- Based on NVIDIA NIM Operator v3.0.0
- Tested on OpenShift 4.x with AWS EBS storage
- PostgreSQL StatefulSet uses 500Mi EBS volume (per component)
- Database credentials are auto-generated in `secrets.yaml`
- Evaluator uses namespace-scoped Argo Workflows (enhanced security model)
- All components support standalone deployment or combined installation

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