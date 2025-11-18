# MinIO Single-Node Kubernetes Deployment

This directory contains the configuration files and documentation for deploying MinIO Community Edition on Kubernetes as a single-node, single-replica setup.

## Files

- `deployment-guide.md` - Comprehensive technical documentation
- `minio-deployment.yaml` - Kubernetes Deployment and Service configuration
- `minio-pvc.yaml` - PersistentVolumeClaim for MinIO storage
- `minio-secret.yaml.example` - Example secret configuration (for reference)

## Quick Start

1. **Create the access credentials secret:**
   ```bash
   kubectl create secret generic minio-credentials \
     --from-literal=root-user=minioadmin \
     --from-literal=root-password=minioadmin123
   ```

2. **Create the persistent volume claim:**
   ```bash
   kubectl apply -f minio-pvc.yaml
   ```

3. **Deploy MinIO:**
   ```bash
   kubectl apply -f minio-deployment.yaml
   ```

4. **Access MinIO Console:**
   ```bash
   kubectl port-forward svc/minio 9001:9001
   ```
   Then open http://localhost:9001 in your browser.

5. **Access MinIO API (S3 endpoint):**
   ```bash
   kubectl port-forward svc/minio 9000:9000
   ```

## Documentation

For detailed information, configuration options, limitations, and upgrade paths, see [deployment-guide.md](./deployment-guide.md).

## Important Notes

⚠️ **This is a single-replica deployment suitable for development and testing only.**

- ❌ Not recommended for production
- ❌ No high availability
- ❌ No data durability guarantees
- ✅ Suitable for development, testing, and learning

For production deployments, consider using the [MinIO Kubernetes Operator](https://github.com/minio/operator) or deploying a distributed MinIO cluster.

