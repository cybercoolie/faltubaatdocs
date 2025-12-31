# FaltuBaat Kubernetes Deployment Guide

## Prerequisites

1. **EKS Cluster** - Running Kubernetes cluster on AWS
2. **kubectl** - Configured to connect to your cluster
3. **AWS Load Balancer Controller** - Installed in cluster
4. **EFS CSI Driver** - For persistent storage
5. **ECR Repository** - For Docker images

## Files Overview

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates faltubaat namespace |
| `configmap-app.yaml` | App environment configuration |
| `configmap-nginx.yaml` | Nginx RTMP configuration |
| `secret.yaml` | Sensitive data (JWT secret, etc.) |
| `storage-class.yaml` | EFS storage class |
| `pvc.yaml` | Persistent volume claims |
| `deployment-app.yaml` | Node.js app deployment |
| `deployment-rtmp.yaml` | Nginx RTMP deployment |
| `service-app.yaml` | App load balancer service |
| `service-rtmp.yaml` | RTMP load balancer service |
| `ingress.yaml` | ALB Ingress (optional, for HTTPS) |
| `hpa.yaml` | Auto-scaling configuration |

## Step-by-Step Deployment

### 1. Setup AWS Resources

```bash
# Create EFS filesystem
aws efs create-file-system --creation-token faltubaat-efs

# Note the FileSystemId and update storage-class.yaml
```

### 2. Build and Push Docker Images

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com

# Build and push App image
docker build -t faltubaat-app .
docker tag faltubaat-app:latest <account>.dkr.ecr.us-east-1.amazonaws.com/faltubaat-app:latest
docker push <account>.dkr.ecr.us-east-1.amazonaws.com/faltubaat-app:latest

# Update deployment-app.yaml with your ECR image URL
```

### 3. Update Configuration

Before deploying, update these files:

- **`storage-class.yaml`**: Replace `fs-xxxxxxxxx` with your EFS ID
- **`deployment-app.yaml`**: Replace `<YOUR_ECR_REPO>` with your ECR URL
- **`secret.yaml`**: Change `JWT_SECRET` to a secure value
- **`ingress.yaml`**: Update domain and ACM certificate ARN

### 4. Deploy to Kubernetes

```bash
# Apply in order
kubectl apply -f namespace.yaml
kubectl apply -f configmap-app.yaml
kubectl apply -f configmap-nginx.yaml
kubectl apply -f secret.yaml
kubectl apply -f storage-class.yaml
kubectl apply -f pvc.yaml
kubectl apply -f deployment-app.yaml
kubectl apply -f deployment-rtmp.yaml
kubectl apply -f service-app.yaml
kubectl apply -f service-rtmp.yaml
kubectl apply -f ingress.yaml  # Optional
kubectl apply -f hpa.yaml

# Or apply all at once
kubectl apply -f .
```

### 5. Get Load Balancer URLs

```bash
# Get App URL (ALB)
kubectl get ingress -n faltubaat

# Get RTMP URL (NLB)
kubectl get svc faltubaat-rtmp-service -n faltubaat
```

### 6. Update App Configuration

After getting the NLB URL for RTMP, update your frontend to point to:
- **RTMP Ingest**: `rtmp://<NLB-URL>:1935/live`
- **HLS Playback**: `http://<NLB-URL>:8080/hls`

## Architecture

```
                    Internet
                        │
            ┌───────────┴───────────┐
            │                       │
        ┌───▼───┐               ┌───▼───┐
        │  ALB  │               │  NLB  │
        │ (443) │               │(1935) │
        └───┬───┘               └───┬───┘
            │                       │
    ┌───────▼───────┐       ┌───────▼───────┐
    │   App Pods    │       │   RTMP Pods   │
    │  (Node.js)    │◄──────│  (Nginx)      │
    │   x2-10       │       │    x1-5       │
    └───────┬───────┘       └───────┬───────┘
            │                       │
            └───────────┬───────────┘
                        │
                    ┌───▼───┐
                    │  EFS  │
                    │(Data) │
                    └───────┘
```

## Monitoring

```bash
# Check pod status
kubectl get pods -n faltubaat

# View logs
kubectl logs -f deployment/faltubaat-app -n faltubaat
kubectl logs -f deployment/faltubaat-rtmp -n faltubaat

# Check HPA status
kubectl get hpa -n faltubaat
```

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod <pod-name> -n faltubaat
```

### Service not accessible
```bash
kubectl get svc -n faltubaat
kubectl describe svc faltubaat-app-service -n faltubaat
```

### Storage issues
```bash
kubectl get pvc -n faltubaat
kubectl describe pvc faltubaat-data-pvc -n faltubaat
```

## Cleanup

```bash
kubectl delete -f .
```
