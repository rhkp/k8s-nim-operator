# NVIDIA NIM Operator E2E Dependencies - OpenShift Deployment

Quick guide for deploying NVIDIA NIM Operator E2E test dependencies on OpenShift.

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

#### Generated Files

The playbook creates sensitive files that are git-ignored:
- `customizer-postgresql-values.yaml` - Helm values with PostgreSQL configuration
- `customizer-opentelemetry-values.yaml` - OpenTelemetry collector configuration
- `secrets.yaml` - Customizer PostgreSQL password

#### MLflow Enabled

**MLflow is now enabled** (`mlflow.enabled: true`) with working image alternatives compatible with NVIDIA NIM Operator v3.0.0.

**Image Configuration for v3.0.0 Compatibility:**
- **MLflow Server**: `docker.io/library/python:3.9-slim` with runtime installation
- **PostgreSQL**: `docker.io/bitnami/postgresql:latest`
- **Git Init Container**: `docker.io/alpine/git:latest`
- **Volume Permissions**: `docker.io/library/busybox:latest`
- **MinIO**: Disabled (incompatible official image with Bitnami configuration)

**Custom Installation Method:**
MLflow is installed at runtime using `pip install --prefix=/tmp/pip-install mlflow==2.12.2 psycopg2-binary` to work around read-only filesystem restrictions in OpenShift security contexts.

**Additional Generated Files:**
- `mlflow.yaml` - MLflow Helm values with working image overrides (git-ignored)

### Verification

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

Expected output:
```
NAME                                           READY   STATUS    RESTARTS   AGE
customizer-pg-postgresql-0                     1/1     Running   0          10m
opentelemetry-collector-xxxxx                  1/1     Running   0          10m
mlflow-tracking-xxxxx                          1/1     Running   0          10m
mlflow-postgresql-0                            1/1     Running   0          10m
```

## Common Troubleshooting {#common-troubleshooting}

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
- PostgreSQL StatefulSet uses 500Mi EBS volume
- Database credentials are auto-generated in `secrets.yaml`