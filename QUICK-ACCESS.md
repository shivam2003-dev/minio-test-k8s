# 🚀 MinIO Quick Access Guide

## ✅ Deployment Status: RUNNING

Your MinIO instance is **deployed and running** on Kubernetes!

---

## 🌐 Access the Web UI (Console)

**Yes, MinIO has a beautiful web UI!**

### Step 1: Start Port Forwarding
```bash
kubectl port-forward svc/minio 9001:9001
```

### Step 2: Open in Browser
Open: **http://localhost:9001**

### Step 3: Login
- **Username**: `minioadmin`
- **Password**: `minioadmin123`

---

## 🔌 Access the S3 API

### Start Port Forwarding
```bash
kubectl port-forward svc/minio 9000:9000
```

### Use with S3 Clients
- **Endpoint**: `http://localhost:9000`
- **Access Key**: `minioadmin`
- **Secret Key**: `minioadmin123`

---

## 📊 Current Status

```
✅ Pod: Running (1/1)
✅ Service: Active
✅ Storage: 10Gi allocated
✅ Web UI: Port 9001
✅ S3 API: Port 9000
```

---

## 🎯 What You Can Do in the UI

1. **Create Buckets** - Organize your objects
2. **Upload/Download Files** - Drag and drop interface
3. **Manage Users** - Create access keys
4. **Set Policies** - Configure bucket permissions
5. **View Metrics** - Monitor usage and performance
6. **Configure Lifecycle** - Set up retention policies

---

## 🛠️ Quick Commands

```bash
# Check status
kubectl get pods -l app=minio

# View logs
kubectl logs -l app=minio

# Restart if needed
kubectl rollout restart deployment/minio
```

---

**Ready to use!** 🎉

