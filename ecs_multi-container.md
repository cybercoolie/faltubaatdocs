# FaltuBaat Multi-Container ECS Deployment

This deployment runs Node.js and Nginx/RTMP as **separate containers** on AWS ECS, enabling independent scaling and better observability.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐         ┌────────────────────────────────────────┐    │
│  │     ALB     │────────▶│            ECS Fargate Task            │    │
│  │  Port 80/443│         │  ┌─────────────┐   ┌────────────────┐  │    │
│  └─────────────┘         │  │  chat-app   │   │   nginx-rtmp   │  │    │
│                          │  │   :3000     │◀──│     :1935      │  │    │
│  ┌─────────────┐         │  │   :3443     │   │     :8080      │  │    │
│  │     NLB     │────────▶│  └──────┬──────┘   └───────┬────────┘  │    │
│  │  Port 1935  │         │         │                  │           │    │
│  └─────────────┘         └─────────┼──────────────────┼───────────┘    │
│                                    │                  │                │
│                          ┌─────────▼──────┐  ┌────────▼────────┐       │
│                          │   EFS /data    │  │    EFS /hls     │       │
│                          │   (SQLite DB)  │  │   (HLS streams) │       │
│                          └────────────────┘  └─────────────────┘       │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    CloudWatch Log Groups                          │  │
│  │  /ecs/faltubaat/chat-app    │    /ecs/faltubaat/nginx-rtmp       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

## Container Communication

Containers in the same ECS task communicate via `localhost` in `awsvpc` network mode.

However, we use **ECS Service Connect** to provide DNS-based discovery (`chat-app:3000`), which:
- Works if you scale to multiple tasks
- Provides service mesh capabilities
- Enables traffic routing

## Prerequisites

1. **AWS CLI** configured with appropriate credentials
2. **Docker** installed locally
3. **ECR Repositories** created:
   ```bash
   aws ecr create-repository --repository-name faltubaat-app --region YOUR_REGION
   aws ecr create-repository --repository-name faltubaat-nginx-rtmp --region YOUR_REGION
   ```
4. **ECS Cluster** with Service Connect namespace:
   ```bash
   aws ecs create-cluster \
     --cluster-name faltubaat-cluster \
     --service-connect-defaults namespace=faltubaat \
     --region YOUR_REGION
   ```
5. **Cloud Map Namespace** for service discovery:
   ```bash
   aws servicediscovery create-http-namespace \
     --name faltubaat \
     --region YOUR_REGION
   ```
6. **EFS File System** for persistent storage (or use S3 for testing)
7. **CloudWatch Log Groups**:
   ```bash
   aws logs create-log-group --log-group-name /ecs/faltubaat/chat-app --region YOUR_REGION
   aws logs create-log-group --log-group-name /ecs/faltubaat/nginx-rtmp --region YOUR_REGION
   ```
8. **JWT Secret in Secrets Manager** (see below)

## JWT Secret Setup

The application uses JWT (JSON Web Tokens) for user authentication. The **JWT_SECRET** is a private key used to sign and verify tokens.

### Why Store in Secrets Manager?

| Without Secrets Manager | With Secrets Manager |
|------------------------|---------------------|
| Random secret on each restart | Consistent secret across restarts |
| Users logged out when container restarts | Users stay logged in |
| Different secret per container | Same secret for all containers |
| Token from container A fails on B | Tokens work everywhere |

### Create JWT Secret

```bash
# Generate a secure random secret and store in Secrets Manager
aws secretsmanager create-secret \
  --name faltubaat/jwt-secret \
  --description "JWT signing secret for FaltuBaat authentication" \
  --secret-string "$(openssl rand -hex 32)" \
  --region YOUR_REGION
```

### Verify Secret Was Created

```bash
# List secrets
aws secretsmanager list-secrets --region YOUR_REGION

# Retrieve secret value (for debugging only)
aws secretsmanager get-secret-value \
  --secret-id faltubaat/jwt-secret \
  --region YOUR_REGION \
  --query 'SecretString' --output text
```

### Rotate Secret (Optional)

```bash
# Update with a new random secret (will log out all users)
aws secretsmanager update-secret \
  --secret-id faltubaat/jwt-secret \
  --secret-string "$(openssl rand -hex 32)" \
  --region YOUR_REGION
```

### Task Definition Reference

The secret is referenced in `task-definition.json` for the `chat-app` container:
```json
"secrets": [
  {
    "name": "JWT_SECRET",
    "valueFrom": "arn:aws:secretsmanager:YOUR_REGION:YOUR_ACCOUNT_ID:secret:faltubaat/jwt-secret"
  }
]
```

> **Note**: ECS automatically fetches the secret at container startup. The app never sees the Secrets Manager ARN, only the actual secret value.

## S3 Database Storage (Optional - For Testing)

Use S3 instead of EFS for database storage during testing:

### Setup
```bash
# 1. Create S3 bucket
aws s3 mb s3://faltubaat-test-db --region YOUR_REGION

# 2. Attach IAM policy to task role
aws iam put-role-policy \
  --role-name ecsTaskRole \
  --policy-name FaltuBaatS3DBSync \
  --policy-document file://deploy/ecs/iam-policy-s3-db.json
```

### Enable in Task Definition
Set these environment variables for the `chat-app` container:
```json
{ "name": "S3_SYNC_ENABLED", "value": "true" },
{ "name": "S3_DB_BUCKET", "value": "faltubaat-test-db" }
```

### Behavior
| Event | Action |
|-------|--------|
| Container start | Download DB from S3 |
| Every 5 minutes | Sync DB to S3 |
| Container stop | Upload final DB state |

⚠️ **Production Warning**: Use EFS for production. S3 sync may lose recent data on crashes.
   ```

## Configuration

Update the following in `task-definition.json`:

| Placeholder | Description |
|-------------|-------------|
| `YOUR_ACCOUNT_ID` | Your AWS Account ID |
| `YOUR_REGION` | AWS Region (e.g., us-east-1) |
| `fs-XXXXXXXXX` | EFS File System ID |

Update in `service-definition.json`:

| Placeholder | Description |
|-------------|-------------|
| `subnet-XXXXXXXXX` | VPC Subnet IDs (private recommended) |
| `sg-XXXXXXXXX` | Security Group ID |
| Target Group ARNs | Your ALB/NLB target groups |

## Deployment

```bash
# Set environment variables
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=123456789012

# Run deployment
./deploy.sh
```

## Security Group Rules

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| Inbound | 3000 | ALB SG | HTTP traffic to chat-app |
| Inbound | 3443 | ALB SG | HTTPS traffic to chat-app |
| Inbound | 1935 | 0.0.0.0/0 | RTMP streaming input |
| Inbound | 8080 | ALB SG | HLS stream output |
| Inbound | 2049 | VPC CIDR | EFS mount |

## Scaling

```bash
# Scale to 3 tasks (each with chat-app + nginx-rtmp)
aws ecs update-service \
  --cluster faltubaat-cluster \
  --service faltubaat-multi-service \
  --desired-count 3

# Auto-scaling based on CPU
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/faltubaat-cluster/faltubaat-multi-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10
```

## Advantages over Single-Container

| Feature | Multi-Container | Single-Container |
|---------|-----------------|------------------|
| Health Checks | ✅ Per container | ⚠️ Single check |
| Logging | ✅ Separate streams | ⚠️ Mixed logs |
| Resource Control | ✅ Per container | ⚠️ Shared |
| Failure Isolation | ✅ Independent | ⚠️ Coupled |
| Updates | ✅ Independent | ⚠️ Full redeploy |
