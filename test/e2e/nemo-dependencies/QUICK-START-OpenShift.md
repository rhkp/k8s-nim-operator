# NVIDIA NEMO OpenShift Quick Start Guide

Get NVIDIA NEMO running on OpenShift in ~45 minutes with minimal configuration.

> **For detailed troubleshooting and configuration options, see [README-OpenShift.md](./README-OpenShift.md)**

## Prerequisites

- **OpenShift cluster** 4.x with ~200Gi storage available
- **Storage class** supporting `ReadWriteOnce` (e.g., `gp3-csi` for AWS EBS)
- **CLI tools**: `oc`, `helm`, `ansible` installed and authenticated
- **NGC API key** from [NVIDIA GPU Cloud](https://ngc.nvidia.com/)
- **Cluster permissions**: namespace-admin or cluster-admin
- **GPU nodes**: At least one GPU node (g5.xlarge, g5.2xlarge, etc.) for inference workloads

## Step 1: Configuration Setup

### 1.1 Create Target Namespace
```bash
# Replace 'my-nemo' with your preferred namespace name
export NEMO_NAMESPACE="my-nemo"
oc create namespace $NEMO_NAMESPACE
```

### 1.2 Validate Cluster Prerequisites
```bash
# Check for GPU nodes (should show nodes with nvidia.com/gpu labels)
oc get nodes -l 'feature.node.kubernetes.io/pci-10de.present=true' \
  -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu

# Verify storage class exists
oc get storageclass gp3-csi

# Expected: At least one GPU node and gp3-csi storage class available
```

### 1.3 Create NGC Secrets
```bash
# Replace <YOUR_NGC_API_KEY> with your actual NGC API key
export NGC_API_KEY="<YOUR_NGC_API_KEY>"

# NGC Image Pull Secret
oc create secret docker-registry ngc-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=$NGC_API_KEY \
  -n $NEMO_NAMESPACE

# NGC API Secret
oc create secret generic ngc-api-secret \
  --from-literal=NGC_API_KEY=$NGC_API_KEY \
  -n $NEMO_NAMESPACE
```

### 1.4 Configure values.yaml
```bash
# Navigate to the deployment directory
cd deploy-on-openshift/test/e2e/nemo-dependencies

# Update values.yaml with your configuration
# Edit these 3 key values:
cat > values.yaml << EOF
installation_namespace: $NEMO_NAMESPACE

pvc:
  storage_class: "gp3-csi"              # Replace with your storage class
  volume_access_mode: ReadWriteOnce

localPathProvisioner:
  enabled: false

install:
  datastore: yes
  entity_store: yes
  customizer: yes
  jupyter: yes
  guardrail: yes
  evaluator: yes
EOF
```

## Step 2: Deploy Infrastructure Components

```bash
# Deploy all infrastructure components (PostgreSQL, MLflow, Argo, etc.)
ansible-playbook -c local -i localhost install.yaml

# Wait for infrastructure to be ready (~15-20 minutes)
echo "Waiting for infrastructure deployment..."
oc get pods -n $NEMO_NAMESPACE -w
```

**Expected**: All pods showing `Running` status (may take 15-20 minutes)

## Step 3: Install Volcano Scheduler

```bash
# Add Volcano Helm repository
helm repo add volcano https://volcano-sh.github.io/helm-charts
helm repo update

# Pre-grant privileged SCC for OpenShift (prevents scheduler pod failure)
oc adm policy add-scc-to-user privileged system:serviceaccount:$NEMO_NAMESPACE:volcano-scheduler

# Install Volcano scheduler
helm install volcano volcano/volcano \
  --namespace $NEMO_NAMESPACE \
  --version 1.9.0 \
  --wait --timeout=300s
```

**Expected**: 4 volcano pods running (admission, controllers, scheduler + init completed)

## Step 4: Install NeMo Operator (v25.06)

```bash
# Add NeMo Helm repository
helm repo add nvidia-nemo https://helm.ngc.nvidia.com/nvidia-nemo
helm repo update

# Install NeMo Operator
helm install nemo-operator nvidia-nemo/nemo-operator \
  -n $NEMO_NAMESPACE \
  --set manager.resources.limits.memory=512Mi \
  --set manager.resources.requests.memory=256Mi \
  --wait --timeout=300s

# CRITICAL: Patch service account to use NGC image pull secret
oc patch serviceaccount nemo-operator-controller-manager -n $NEMO_NAMESPACE -p '{"imagePullSecrets": [{"name": "ngc-secret"}]}'

# Restart the deployment to pick up the new secret
oc delete pod -n $NEMO_NAMESPACE -l app.kubernetes.io/name=nemo-operator
```

**Expected**: NeMo operator pod running with 2/2 containers ready

## Step 5: Install NIM Operator (v3.0.1)

```bash
# Install NIM Operator using OpenShift-specific Helm chart
helm install k8s-nim-operator /tmp/k8s-nim-operator/deployments/helm/k8s-nim-operator \
  -n $NEMO_NAMESPACE \
  --set operator.resources.limits.memory=512Mi \
  --set operator.resources.requests.memory=256Mi \
  --wait --timeout=300s
```

**Expected**: NIM operator pod running with 1/1 container ready

> **Critical**: OpenShift requires the local Helm chart from the `deploy-v3.0-on-openshift` branch. The official NVIDIA Helm repository chart (`nvidia/k8s-nim-operator`) is not compatible with OpenShift due to security context and permission differences.

## Step 6: Deploy NEMO Samples

```bash
# Use the OpenShift-optimized NEMO samples with GPU tolerations pre-configured
oc apply -f nemo-samples.yaml
```

> **Note**: The provided `nemo-samples.yaml` includes GPU node tolerations for common OpenShift configurations (`g5-gpu` and `nvidia.com/gpu` taints). No additional configuration needed.

**Expected**: 7 custom resources created (NemoCustomizer, NemoDatastore, NemoEntitystore, NemoEvaluator, NemoGuardrail, NIMCache, NIMPipeline)

## Step 7: Verify Deployment

### 7.1 Check Infrastructure Services
```bash
# Verify infrastructure pods are running
oc get pods -n $NEMO_NAMESPACE | grep -E "(postgresql|opentelemetry|mlflow|argo|milvus)"
```

**Expected**: All infrastructure pods showing `Running` status

### 7.2 Check NEMO Microservices
```bash
# Verify NEMO microservices are Ready
oc get -n $NEMO_NAMESPACE nemoentitystore,nemodatastore,nemoguardrails,nemocustomizer,nemoevaluator
```

**Expected**: All 5 services showing `STATUS: Ready`

### 7.3 Check NIM Services
```bash
# Verify NIM services are Ready
oc get -n $NEMO_NAMESPACE nimpipeline,nimcache,nimservice
```

**Expected**: All 3 services showing `STATUS: Ready` (NIM cache may be pending without GPU nodes)

### 7.4 Test API Endpoint
```bash
# Test NEMO Customizer API
oc run test-api --image=curlimages/curl:latest -n $NEMO_NAMESPACE --restart=Never -- \
  curl -X GET "http://nemocustomizer-sample.$NEMO_NAMESPACE:8000/v1/customization/configs"

# Check result and cleanup
sleep 5
oc logs test-api -n $NEMO_NAMESPACE
oc delete pod test-api -n $NEMO_NAMESPACE
```

**Expected**: JSON response with available model configurations

## Quick Reference - Complete Command Sequence

```bash
# 1. Setup
export NEMO_NAMESPACE="my-nemo"
export NGC_API_KEY="<YOUR_NGC_API_KEY>"
oc create namespace $NEMO_NAMESPACE

# 2. Create secrets
oc create secret docker-registry ngc-secret \
  --docker-server=nvcr.io --docker-username='$oauthtoken' \
  --docker-password=$NGC_API_KEY -n $NEMO_NAMESPACE
oc create secret generic ngc-api-secret \
  --from-literal=NGC_API_KEY=$NGC_API_KEY -n $NEMO_NAMESPACE

# 3. Configure and deploy infrastructure
cd deploy-on-openshift/test/e2e/nemo-dependencies
# Edit values.yaml with your namespace and storage class
ansible-playbook -c local -i localhost install.yaml

# 4. Install operators
helm repo add nvidia-nemo https://helm.ngc.nvidia.com/nvidia-nemo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install nemo-operator nvidia-nemo/nemo-operator -n $NEMO_NAMESPACE \
  --set manager.resources.limits.memory=512Mi --wait --timeout=300s

# CRITICAL: Patch service account for NGC image pull secret
oc patch serviceaccount nemo-operator-controller-manager -n $NEMO_NAMESPACE -p '{"imagePullSecrets": [{"name": "ngc-secret"}]}'
oc delete pod -n $NEMO_NAMESPACE -l app.kubernetes.io/name=nemo-operator

helm install k8s-nim-operator /tmp/k8s-nim-operator/deployments/helm/k8s-nim-operator -n $NEMO_NAMESPACE \
  --set operator.resources.limits.memory=512Mi --wait --timeout=300s

# 5. Deploy samples
curl -o nemo-samples.yaml https://raw.githubusercontent.com/NVIDIA/k8s-nim-operator/main/config/samples/nemo/latest/all_in_one.yaml
sed -i "s/namespace: nemo/namespace: $NEMO_NAMESPACE/g" nemo-samples.yaml
oc apply -f nemo-samples.yaml

# 6. Verify
oc get -n $NEMO_NAMESPACE nemoentitystore,nemodatastore,nemoguardrails,nemocustomizer,nemoevaluator
```

## Success Criteria

✅ **Infrastructure**: ~15 pods running (PostgreSQL, MLflow, Argo, Milvus, OpenTelemetry)
✅ **Operators**: 2 operator pods running (nemo-operator, k8s-nim-operator)
✅ **NEMO Services**: 5 microservices showing `Ready` status
✅ **NIM Services**: 3 NIM components deployed (cache may be pending without GPU)
✅ **API Test**: Successful JSON response from customizer endpoint

## Timeline

- **Configuration**: 5 minutes
- **Infrastructure deployment**: 15-20 minutes
- **Operators installation**: 5-10 minutes
- **NEMO samples deployment**: 10-15 minutes
- **Verification**: 5 minutes

**Total**: ~45 minutes to working NEMO deployment

## Troubleshooting

For detailed troubleshooting, configuration options, and architectural information, see [README-OpenShift.md](./README-OpenShift.md).

**Prerequisites Check:**
- **Storage class**: Verify `gp3-csi` (or equivalent) supports `ReadWriteOnce`
- **Memory resources**: Operators require 512Mi memory limits (pre-configured)
- **NGC credentials**: Ensure API key is valid and secrets created correctly
- **GPU nodes**: The samples include tolerations for `g5-gpu` and `nvidia.com/gpu` taints (pre-configured)

> **Proactive Design**: This deployment is designed to succeed on first attempt. All common OpenShift configurations are pre-configured.

## What's Deployed

### Infrastructure Components
- **Datastore**: PostgreSQL database for data management
- **Entity Store**: PostgreSQL database for entity metadata
- **Customizer**: MLflow + PostgreSQL + OpenTelemetry for model customization
- **Jupyter**: Notebook server for development
- **Guardrail**: PostgreSQL database for safety controls
- **Evaluator**: Argo Workflows + Milvus + PostgreSQL + OpenTelemetry for evaluation

### NEMO Microservices
- **NemoCustomizer**: Model fine-tuning and customization service
- **NemoDatastore**: Data management and storage service
- **NemoEntitystore**: Entity and model metadata management
- **NemoEvaluator**: Model evaluation and benchmarking service
- **NemoGuardrail**: Safety and content filtering service

### NIM Services
- **NIMCache**: Model caching (meta-llama3-1b-instruct)
- **NIMPipeline**: Inference pipeline for Llama 3.2 1B model
- **NIMService**: Production inference service

## Next Steps

With NEMO deployed, you can:
- **Upload datasets** via NemoDatastore API
- **Fine-tune models** using NemoCustomizer
- **Set up guardrails** for content filtering
- **Run evaluations** on model performance
- **Deploy inference** using NIM services

For production use, review security considerations and customize configurations as described in the full [README-OpenShift.md](./README-OpenShift.md).