# MinIO Community Edition: Single-Node Kubernetes Deployment Guide

## Table of Contents
1. [Overview](#overview)
2. [Deployment Modes](#deployment-modes)
3. [Kubernetes Deployment Configuration](#kubernetes-deployment-configuration)
4. [Access Key Configuration](#access-key-configuration)
5. [Storage Configuration](#storage-configuration)
6. [Accessing MinIO](#accessing-minio)
7. [Limitations](#limitations)
8. [Upgrading to Distributed/HA Deployment](#upgrading-to-distributedha-deployment)

---

## Overview

This guide provides step-by-step instructions for deploying MinIO Community Edition on Kubernetes as a **single-node, single-replica** setup. This configuration is suitable for:

- Development and testing environments
- Non-production workloads
- Lightweight applications with minimal storage requirements
- Learning and experimentation with object storage

**Important**: This setup is **not recommended** for production environments that require high availability, data durability, or horizontal scaling.

---

## Deployment Modes

MinIO supports two primary deployment modes:

### 1. Standalone Mode (Single-Node)
- Runs on a single server/node
- Single MinIO instance
- Suitable for development, testing, or low-traffic scenarios
- **No redundancy** - data loss risk if the node fails
- Minimal resource requirements

### 2. Distributed Mode (Multi-Node)
- Runs across multiple nodes/servers
- Provides high availability and data durability
- Supports erasure coding for data protection
- Recommended for production environments
- Requires minimum 4 nodes for optimal erasure coding

### Why Single-Replica is Acceptable for Non-Production

A single-replica MinIO deployment is acceptable when:
- **Data durability is not critical** - Development/test data can be regenerated
- **Availability requirements are low** - Temporary downtime is acceptable
- **Resource constraints exist** - Limited compute/storage resources available
- **Cost optimization** - Minimizing infrastructure costs for non-critical workloads
- **Simplified operations** - Easier to manage and troubleshoot

---

## Kubernetes Deployment Configuration

### Minimal Deployment YAML

Create a file named `minio-deployment.yaml` with the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  labels:
    app: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        - --console-address
        - ":9001"
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: root-user
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: root-password
        ports:
        - containerPort: 9000
          name: api
        - containerPort: 9001
          name: console
        volumeMounts:
        - name: storage
          mountPath: /data
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app: minio
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP
    name: api
  - port: 9001
    targetPort: 9001
    protocol: TCP
    name: console
  selector:
    app: minio
```

### Key Configuration Details

- **Image**: Uses the official `minio/minio:latest` image
- **Replicas**: Set to `1` for single-node deployment
- **Storage Path**: `/data` - where MinIO stores objects
- **Console Port**: `9001` - MinIO web console for management
- **API Port**: `9000` - S3-compatible API endpoint
- **Resource Limits**: Configurable based on your cluster capacity

---

## Access Key Configuration

MinIO requires root access credentials (username and password) to be set via environment variables. These should be stored securely using Kubernetes Secrets.

### Step 1: Create Access Credentials Secret

Create a Kubernetes secret containing the root user credentials:

```bash
kubectl create secret generic minio-credentials \
  --from-literal=root-user=minioadmin \
  --from-literal=root-password=minioadmin123
```

**Security Best Practice**: Use strong, randomly generated passwords in production. For example:

```bash
# Generate a random password
PASSWORD=$(openssl rand -base64 32)

# Create secret with generated password
kubectl create secret generic minio-credentials \
  --from-literal=root-user=minioadmin \
  --from-literal=root-password=$PASSWORD
```

### Step 2: Alternative - Create Secret from YAML

You can also create the secret using a YAML file for version control (with base64-encoded values):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
type: Opaque
data:
  root-user: bWluaW9hZG1pbg==  # base64 encoded: minioadmin
  root-password: bWluaW9hZG1pbiMxMjM=  # base64 encoded: minioadmin123
```

Apply it with:
```bash
kubectl apply -f minio-secret.yaml
```

**Note**: The deployment YAML references this secret via `secretKeyRef` in the environment variables section.

### Step 3: Verify Secret Creation

```bash
kubectl get secret minio-credentials
kubectl describe secret minio-credentials
```

---

## Storage Configuration

MinIO requires persistent storage to retain data across pod restarts. This is achieved using Kubernetes PersistentVolumes (PV) and PersistentVolumeClaims (PVC).

### PersistentVolumeClaim Configuration

Create a file named `minio-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard  # Adjust based on your cluster's storage class
```

### Storage Considerations

1. **Storage Size**: Adjust the `storage` value based on your needs (e.g., `50Gi`, `100Gi`)
2. **Storage Class**: 
   - Use `standard` for most cloud providers (GKE, EKS, AKS)
   - Use `local-path` for local Kubernetes clusters (k3s, kind, minikube)
   - Check available storage classes: `kubectl get storageclass`
3. **Access Mode**: `ReadWriteOnce` is sufficient for single-replica deployment
4. **Volume Binding**: Ensure your cluster has available PersistentVolumes or dynamic provisioning enabled

### Apply Storage Configuration

```bash
kubectl apply -f minio-pvc.yaml
```

Verify the PVC is bound:
```bash
kubectl get pvc minio-pvc
```

Expected output:
```
NAME       STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-pvc  Bound    pvc-xxx  10Gi       RWO            standard       1m
```

---

## Accessing MinIO

### Step 1: Deploy MinIO

Apply the deployment and service:

```bash
kubectl apply -f minio-deployment.yaml
```

### Step 2: Verify Deployment

Check that the pod is running:

```bash
kubectl get pods -l app=minio
kubectl get svc minio
```

Wait for the pod to be in `Running` state:
```bash
kubectl wait --for=condition=ready pod -l app=minio --timeout=300s
```

### Step 3: Access Methods

#### Option A: Port Forwarding (Recommended for Testing)

**Access the MinIO API (S3 endpoint):**
```bash
kubectl port-forward svc/minio 9000:9000
```

**Access the MinIO Console (Web UI):**
```bash
kubectl port-forward svc/minio 9001:9001
```

Then open your browser to:
- **Console**: http://localhost:9001
- **API Endpoint**: http://localhost:9000

Login with the credentials from your secret (default: `minioadmin` / `minioadmin123`).

#### Option B: NodePort Service (Cluster Access)

Modify the service to use NodePort:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  type: NodePort
  ports:
  - port: 9000
    targetPort: 9000
    nodePort: 30090
    name: api
  - port: 9001
    targetPort: 9001
    nodePort: 30091
    name: console
  selector:
    app: minio
```

Access via: `http://<node-ip>:30091` (console) or `http://<node-ip>:30090` (API)

#### Option C: LoadBalancer Service (Cloud Providers)

Change service type to `LoadBalancer`:

```yaml
spec:
  type: LoadBalancer
```

The cloud provider will assign an external IP automatically.

### Step 4: Test MinIO Connection

Using the MinIO client (`mc`):

```bash
# Configure MinIO alias
mc alias set myminio http://localhost:9000 minioadmin minioadmin123

# List buckets
mc ls myminio

# Create a test bucket
mc mb myminio/test-bucket

# Upload a file
mc cp /path/to/file myminio/test-bucket/
```

Using AWS CLI (S3-compatible):

```bash
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin123
export AWS_ENDPOINT_URL=http://localhost:9000

aws --endpoint-url=$AWS_ENDPOINT_URL s3 ls
aws --endpoint-url=$AWS_ENDPOINT_URL s3 mb s3://test-bucket
```

---

## Limitations

### Single-Replica MinIO Limitations

1. **No High Availability**
   - Pod failure results in complete service unavailability
   - No automatic failover or redundancy
   - Downtime during node maintenance or failures

2. **No Data Durability**
   - Single point of failure for data storage
   - No erasure coding or replication
   - Data loss risk if the persistent volume fails
   - No protection against disk corruption

3. **No Horizontal Scaling**
   - Cannot scale beyond a single pod
   - Performance limited by single node resources
   - No load distribution across multiple instances

4. **Limited Performance**
   - Throughput limited by single node capabilities
   - No parallel processing across nodes
   - I/O bottleneck on single storage volume

5. **No Multi-Zone Support**
   - Cannot distribute data across availability zones
   - Vulnerable to zone-level failures

6. **Backup Dependency**
   - Requires external backup solutions for data protection
   - No built-in replication or snapshot capabilities

### When to Use Single-Replica

✅ **Appropriate for:**
- Development and testing environments
- Proof of concept projects
- Learning and experimentation
- Non-critical internal tools
- Low-traffic applications

❌ **Not appropriate for:**
- Production workloads
- Critical business data
- High-traffic applications
- Compliance-sensitive data
- Multi-user production environments

---

## Upgrading to Distributed/HA Deployment

To upgrade from single-replica to a distributed, high-availability MinIO deployment, you'll need to make several key changes:

### 1. Multiple Pods/Replicas

Change from `replicas: 1` to multiple replicas (minimum 4 for erasure coding):

```yaml
spec:
  replicas: 4  # Minimum 4 for erasure coding
```

### 2. StatefulSet Instead of Deployment

Use a `StatefulSet` for stable network identities and ordered deployment:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
spec:
  serviceName: minio
  replicas: 4
  # ... rest of configuration
```

### 3. Multiple Storage Volumes

Each pod needs its own persistent volume:

```yaml
volumeClaimTemplates:
- metadata:
    name: storage
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi
```

### 4. Distributed Mode Arguments

Update MinIO server arguments to use distributed mode:

```yaml
args:
- server
- http://minio-{0..3}.minio.default.svc.cluster.local/data
- --console-address
- ":9001"
```

### 5. Headless Service

Create a headless service for StatefulSet pod discovery:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  clusterIP: None  # Headless service
  ports:
  - port: 9000
    name: api
  - port: 9001
    name: console
  selector:
    app: minio
```

### 6. Data Migration Considerations

**Important**: You cannot directly migrate from standalone to distributed mode. You'll need to:

1. **Export data** from the single-replica MinIO instance
2. **Deploy the new distributed setup**
3. **Import data** into the distributed cluster
4. **Update application endpoints** to point to the new service

### 7. Additional Configuration

- **Erasure Coding**: Automatically enabled with 4+ nodes
- **Load Balancing**: Use a LoadBalancer or Ingress for external access
- **Monitoring**: Set up Prometheus metrics and alerting
- **Backup Strategy**: Implement regular backups even with HA

### Example Distributed Deployment Structure

```
minio-0.minio.default.svc.cluster.local
minio-1.minio.default.svc.cluster.local
minio-2.minio.default.svc.cluster.local
minio-3.minio.default.svc.cluster.local
```

All pods work together as a single distributed MinIO cluster, providing:
- High availability (survives node failures)
- Data durability (erasure coding)
- Horizontal scaling
- Better performance

---

## Quick Start Summary

1. **Create secret**: `kubectl create secret generic minio-credentials --from-literal=root-user=minioadmin --from-literal=root-password=minioadmin123`

2. **Create PVC**: `kubectl apply -f minio-pvc.yaml`

3. **Deploy MinIO**: `kubectl apply -f minio-deployment.yaml`

4. **Port forward**: `kubectl port-forward svc/minio 9001:9001`

5. **Access console**: Open http://localhost:9001 and login

---

## Additional Resources

- [MinIO Documentation](https://min.io/docs/)
- [MinIO Kubernetes Operator](https://github.com/minio/operator) (for production deployments)
- [MinIO Client (mc) Documentation](https://min.io/docs/minio/linux/reference/minio-mc.html)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

---

## Troubleshooting

### Pod Not Starting

```bash
# Check pod status
kubectl describe pod -l app=minio

# Check logs
kubectl logs -l app=minio
```

### PVC Not Binding

```bash
# Check PVC status
kubectl describe pvc minio-pvc

# Check available storage classes
kubectl get storageclass
```

### Cannot Access Service

```bash
# Verify service endpoints
kubectl get endpoints minio

# Check service configuration
kubectl describe svc minio
```

---

**Document Version**: 1.0  
**Last Updated**: 2024  
**MinIO Version**: Latest (Community Edition)

