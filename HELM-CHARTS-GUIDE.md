# Official vs Community Helm Charts for MinIO on Kubernetes

## Overview

MinIO provides multiple deployment options for Kubernetes, including official and community-maintained Helm charts. This document compares the available Helm chart options and provides guidance on deploying MinIO in standalone mode using Helm.

---

## Available Helm Charts

### 1. Official MinIO Community Helm Chart

**Repository**: `https://helm.min.io/`

The official MinIO community Helm chart is maintained by MinIO and provides a straightforward way to deploy MinIO Server on any Kubernetes cluster.

**Key Features:**
- Default mode: Standalone (single-server)
- PersistentVolumeClaim enabled by default
- Default storage size: 500Gi
- Supports both standalone and distributed modes
- Configurable authentication and resource limits

### 2. Bitnami MinIO Helm Chart

**Repository**: Bitnami Helm repository

Bitnami maintains a popular, well-documented MinIO Helm chart that also supports standalone and distributed deployments.

**Key Features:**
- Default mode: Standalone
- Similar configuration options to official chart
- Well-maintained and regularly updated
- Comprehensive documentation

---

## Single-Node (Standalone) Mode

### Default Configuration

Both Helm charts explicitly support single-server mode. The default configuration (`mode: standalone`) deploys one MinIO pod serving one volume with `replicas: 1`.

### Example: Official MinIO Chart Installation

```bash
# Add the MinIO Helm repository
helm repo add minio https://helm.min.io/
helm repo update

# Install MinIO in standalone mode
helm install myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=5Gi \
  --set resources.requests.memory=512Mi \
  --set accessKey=admin \
  --set secretKey=admin123
```

**Note**: You can omit `mode=standalone` as it's the default, or explicitly set `mode=standalone` and `replicas=1`.

### Example: Bitnami Chart Installation

```bash
# Add Bitnami Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install MinIO in standalone mode
helm install myminio bitnami/minio \
  --set mode=standalone \
  --set persistence.size=5Gi \
  --set auth.rootUser=admin \
  --set auth.rootPassword=admin123
```

---

## Configuration Values: Standalone vs Distributed

### Standalone Mode Configuration

For a single-node deployment, you can override values in `values.yaml` or via `--set` flags:

#### Key Configuration Parameters

```yaml
# Mode configuration
mode: standalone  # Default, can be omitted
replicas: 1       # Only relevant if distributed mode

# Persistence
persistence:
  enabled: true
  size: 10Gi      # Adjust based on needs (5Gi for dev, 10Gi+ for production-like)
  storageClass: "" # Uses cluster default if empty

# Authentication
accessKey: "admin"        # Or use existingSecret
secretKey: "s3cret123"    # Or use existingSecret

# Alternative authentication (newer format)
rootUser: "minioadmin"
rootPassword: "minioadmin123"

# Resource limits
resources:
  requests:
    memory: 512Mi
    cpu: 200m
  limits:
    memory: 2Gi
    cpu: 1000m

# Service configuration
service:
  type: ClusterIP  # Or LoadBalancer/NodePort for external access

# Ingress (optional)
ingress:
  enabled: false
  annotations: {}
  hosts: []
```

### Distributed Mode Configuration

To switch to distributed mode:

```yaml
mode: distributed
replicas: 4  # Minimum 4 for erasure coding (recommended: 4, 8, 16)
```

**Important**: Changing from standalone to distributed is not seamless. MinIO's data format differs (FS mode vs erasure-coded), so you typically need to:
1. Back up existing data
2. Redeploy with distributed mode
3. Restore data to the new cluster

---

## Storage and Kubernetes Considerations

### Cloud Provider Storage

#### AWS EKS
- Default StorageClass: `gp2` or `gp3` (EBS volumes)
- PVC automatically provisions EBS volumes
- Can override with `persistence.storageClass`

#### Google GKE
- Default StorageClass: `pd-standard` or `pd-ssd` (Persistent Disks)
- PVC automatically provisions Google PD volumes
- Can override with `persistence.storageClass`

#### Azure AKS
- Default StorageClass: `managed-premium` or `managed`
- PVC automatically provisions Azure Disks

### Storage Best Practices

1. **Development/Testing**:
   - Use smaller PVC sizes (5-10Gi)
   - Consider ephemeral storage for pure testing (`persistence.enabled: false`)
   - Note: Data will be lost on pod restart if persistence is disabled

2. **Production**:
   - Use distributed mode with multiple volumes
   - Plan for sufficient storage capacity
   - Consider storage class performance tiers (SSD vs standard)

3. **PVC Lifecycle**:
   - **Standalone mode**: PVC is deleted on Helm uninstall by default
   - **Distributed mode**: PVCs persist unless explicitly deleted
   - Always backup data before uninstalling

### Storage Class Configuration

```yaml
persistence:
  enabled: true
  size: 10Gi
  storageClass: "gp3"  # EKS example
  # storageClass: "pd-ssd"  # GKE example
  # storageClass: ""  # Use cluster default
```

---

## Development Caveats and Limitations

### Standalone Mode Limitations

⚠️ **Standalone mode is intended for evaluation and development, NOT for production.**

#### Key Limitations:

1. **No High Availability**
   - Single pod = single point of failure
   - Pod or node failure = service downtime
   - No automatic failover

2. **No Data Durability**
   - No node-level redundancy
   - Single disk/node failure = data loss risk
   - No erasure coding protection

3. **No Multi-Node Features**
   - Bucket versioning works, but no cross-node replication
   - High-availability features require multiple servers
   - Performance limited to single node

4. **Data Persistence**
   - PVC deleted on Helm uninstall (standalone mode)
   - All data lost when release is removed
   - Must backup before uninstalling

### What Works in Standalone Mode

✅ **Fully Functional Features:**
- Complete S3-compatible API
- Web console (UI) on ports 9000/9001
- Bucket management
- User and policy management
- Server-side encryption
- Bucket policies
- Object operations (upload/download/delete)

✅ **Suitable For:**
- Development environments
- Testing and evaluation
- Proof of concept projects
- Learning MinIO features
- Low-traffic internal tools

❌ **Not Suitable For:**
- Production workloads
- Critical business data
- High-availability requirements
- Multi-user production systems
- Compliance-sensitive data

---

## Complete Helm Installation Examples

### Example 1: Minimal Development Setup

```bash
helm install myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=5Gi \
  --set persistence.enabled=true \
  --set resources.requests.memory=512Mi \
  --set resources.requests.cpu=200m \
  --set accessKey=minioadmin \
  --set secretKey=minioadmin123
```

### Example 2: Development with Custom Storage Class

```bash
helm install myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=10Gi \
  --set persistence.storageClass=gp3 \
  --set rootUser=admin \
  --set rootPassword=securepassword123
```

### Example 3: Standalone with LoadBalancer Service

```bash
helm install myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=10Gi \
  --set service.type=LoadBalancer \
  --set accessKey=admin \
  --set secretKey=admin123
```

### Example 4: Using Values File

Create `minio-values.yaml`:

```yaml
mode: standalone
replicas: 1

persistence:
  enabled: true
  size: 10Gi
  storageClass: ""

auth:
  rootUser: minioadmin
  rootPassword: minioadmin123

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 2Gi
    cpu: 1000m

service:
  type: ClusterIP

ingress:
  enabled: false
```

Install with values file:

```bash
helm install myminio minio/minio -f minio-values.yaml
```

---

## Accessing MinIO After Helm Installation

### 1. Get Service Information

```bash
# Check service
kubectl get svc myminio

# Get service details
kubectl describe svc myminio
```

### 2. Port Forwarding (ClusterIP Service)

```bash
# Access Web Console (port 9001)
kubectl port-forward svc/myminio 9001:9001

# Access API (port 9000)
kubectl port-forward svc/myminio 9000:9000
```

Then access:
- **Console**: http://localhost:9001
- **API**: http://localhost:9000

### 3. LoadBalancer Service

If you set `service.type=LoadBalancer`, get the external IP:

```bash
kubectl get svc myminio
# Wait for EXTERNAL-IP to be assigned
```

Access via:
- **Console**: http://<EXTERNAL-IP>:9001
- **API**: http://<EXTERNAL-IP>:9000

### 4. Ingress Configuration

If ingress is enabled, configure your ingress controller and access via the configured hostname.

---

## Upgrading and Maintenance

### Upgrade MinIO Release

```bash
# Update Helm repository
helm repo update

# Upgrade release
helm upgrade myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=10Gi \
  --reuse-values  # Reuse existing values
```

### View Current Configuration

```bash
# Get current values
helm get values myminio

# Get all release information
helm get all myminio
```

### Uninstall (⚠️ Data Loss Warning)

```bash
# Uninstall release
helm uninstall myminio

# WARNING: In standalone mode, this deletes the PVC and all data!
# Always backup data before uninstalling
```

---

## Migrating from Standalone to Distributed

### Step-by-Step Migration

1. **Backup Existing Data**
   ```bash
   # Use MinIO client (mc) to backup buckets
   mc mirror myminio/backup-source /local/backup
   ```

2. **Uninstall Standalone Deployment**
   ```bash
   helm uninstall myminio
   ```

3. **Deploy Distributed Mode**
   ```bash
   helm install myminio minio/minio \
     --set mode=distributed \
     --set replicas=4 \
     --set persistence.size=10Gi
   ```

4. **Restore Data**
   ```bash
   # Configure new MinIO alias
   mc alias set newminio http://<new-service>:9000 admin admin123
   
   # Restore data
   mc mirror /local/backup newminio/
   ```

---

## Comparison: Helm Chart vs Manual YAML Deployment

### Helm Chart Advantages

✅ **Pros:**
- Easy installation and configuration
- Built-in best practices
- Easy upgrades and rollbacks
- Configurable via values
- Community support and documentation
- Handles service, ingress, and storage automatically

### Manual YAML Advantages

✅ **Pros:**
- Full control over configuration
- No Helm dependency
- Easier to customize complex scenarios
- Better for learning Kubernetes internals
- Version control friendly

### When to Use Each

**Use Helm Chart When:**
- Quick deployment needed
- Standard configuration is sufficient
- Want easy upgrades
- Prefer declarative configuration
- Team familiar with Helm

**Use Manual YAML When:**
- Need fine-grained control
- Custom requirements not supported by chart
- Learning Kubernetes
- Want to avoid Helm dependency
- Complex multi-environment setup

---

## Troubleshooting

### Common Issues

#### 1. PVC Not Binding

```bash
# Check PVC status
kubectl get pvc

# Check available storage classes
kubectl get storageclass

# Verify PVC details
kubectl describe pvc <pvc-name>
```

**Solution**: Ensure storage class exists or set `persistence.storageClass: ""` to use default.

#### 2. Pod Not Starting

```bash
# Check pod status
kubectl get pods -l app=minio

# View pod logs
kubectl logs -l app=minio

# Describe pod for events
kubectl describe pod <pod-name>
```

#### 3. Cannot Access Service

```bash
# Verify service exists
kubectl get svc myminio

# Check service endpoints
kubectl get endpoints myminio

# Test port forwarding
kubectl port-forward svc/myminio 9001:9001
```

#### 4. Authentication Issues

```bash
# Verify secret exists (if using existingSecret)
kubectl get secret <secret-name>

# Check environment variables in pod
kubectl exec <pod-name> -- env | grep MINIO
```

---

## Best Practices

### 1. Development Environment

```yaml
mode: standalone
persistence:
  size: 5Gi
resources:
  requests:
    memory: 512Mi
    cpu: 200m
```

### 2. Staging Environment

```yaml
mode: standalone  # Or distributed with 4 replicas
persistence:
  size: 50Gi
  storageClass: "gp3"
resources:
  requests:
    memory: 1Gi
    cpu: 500m
```

### 3. Production Environment

```yaml
mode: distributed
replicas: 4  # Or 8, 16 for larger deployments
persistence:
  size: 100Gi
  storageClass: "gp3"  # Or high-performance storage
resources:
  requests:
    memory: 2Gi
    cpu: 1000m
```

### 4. Security Best Practices

- Use strong, randomly generated passwords
- Store credentials in Kubernetes secrets
- Enable TLS/SSL for production
- Use network policies to restrict access
- Regularly rotate access keys

---

## References and Sources

### Official Documentation

- **MinIO Helm Chart**: https://github.com/harshavardhana/charts
- **MinIO Helm Chart Values**: https://github.com/harshavardhana/charts/blob/main/minio/values.yaml
- **Bitnami MinIO Chart**: https://github.com/bitnami/charts/tree/main/bitnami/minio
- **Bitnami Chart Values**: https://raw.githubusercontent.com/bitnami/charts/main/bitnami/minio/values.yaml

### Community Resources

- **SUSE/Rancher MinIO Guide**: https://docs.apps.rancher.io/reference-guides/minio/
- **MinIO Operator Issues**: https://github.com/minio/operator/issues/1025
- **MinIO Documentation**: https://docs.min.io

### Related Articles

- **Tencent Cloud Developer Article**: https://cloud.tencent.com/developer/article/2353439

---

## Summary

### Key Takeaways

1. **Both official and Bitnami charts support standalone mode** - Default configuration deploys single-node MinIO
2. **Standalone mode is for dev/test only** - Not suitable for production workloads
3. **Easy to configure** - Use `--set` flags or values files
4. **PVC management** - Standalone mode deletes PVC on uninstall (data loss)
5. **Migration path** - Can upgrade to distributed mode with backup/restore
6. **Storage integration** - Works seamlessly with cloud provider storage (EBS, PD, Azure Disks)

### Quick Reference

```bash
# Install standalone MinIO
helm install myminio minio/minio \
  --set mode=standalone \
  --set persistence.size=10Gi \
  --set accessKey=admin \
  --set secretKey=admin123

# Access console
kubectl port-forward svc/myminio 9001:9001

# Uninstall (⚠️ deletes data)
helm uninstall myminio
```

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**Helm Chart Version**: Latest from official repositories

