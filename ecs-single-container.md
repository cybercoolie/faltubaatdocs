# FaltuBaat Single-Container ECS Deployment

This deployment runs Node.js + Nginx/RTMP in a single container on AWS ECS.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Cloud                                │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐     ┌─────────────────────────────────────┐   │
│  │     ALB     │────▶│          ECS Fargate                │   │
│  │  Port 80/443│     │  ┌───────────────────────────────┐  │   │
│  └─────────────┘     │  │   faltubaat-single container  │  │   │
│                      │  │  ┌─────────┐  ┌────────────┐  │  │   │
│  ┌─────────────┐     │  │  │ Node.js │  │   Nginx    │  │  │   │
│  │     NLB     │────▶│  │  │  :3000  │◀─│   :1935    │  │  │   │
│  │  Port 1935  │     │  │  │  :3443  │  │   :8080    │  │  │   │
│  └─────────────┘     │  │  └─────────┘  └────────────┘  │  │   │
│                      │  └───────────────────────────────┘  │   │
│                      └─────────────────────────────────────┘   │
│                                     │                          │
│                      ┌──────────────┴──────────────┐           │
│                      │         EFS Storage         │           │
│                      │   /data (SQLite)            │           │
│                      │   /hls (streams)            │           │
│                      └─────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

1. **AWS CLI** configured with appropriate credentials
2. **Docker** installed locally
3. **ECR Repository** created:
   ```bash
   aws ecr create-repository --repository-name faltubaat-single --region YOUR_REGION
   ```
4. **ECS Cluster** created:
   ```bash
   aws ecs create-cluster --cluster-name faltubaat-cluster --region YOUR_REGION
   ```
5. **EFS File System** for persistent storage (or use S3 for testing)
6. **JWT Secret in Secrets Manager** (see below)

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

The secret is referenced in `task-definition.json`:
```json
"secrets": [
  {
    "name": "JWT_SECRET",
    "valueFrom": "arn:aws:secretsmanager:YOUR_REGION:YOUR_ACCOUNT_ID:secret:faltubaat/jwt-secret"
  }
]
```

> **Note**: ECS automatically fetches the secret at container startup. The app never sees the Secrets Manager ARN, only the actual secret value.

## Configuration

Update the following in `task-definition.json`:

| Placeholder | Description |
|-------------|-------------|
| `YOUR_ACCOUNT_ID` | Your AWS Account ID |
| `YOUR_REGION` | AWS Region (e.g., us-east-1) |
| `fs-XXXXXXXXX` | EFS File System ID |

## S3 Database Storage (Optional - For Testing)

Instead of EFS, you can use S3 for database storage during testing:

### 1. Create S3 Bucket
```bash
aws s3 mb s3://faltubaat-test-db --region YOUR_REGION
```

### 2. Attach IAM Policy
Use the policy in `deploy/ecs/iam-policy-s3-db.json`:
```bash
aws iam put-role-policy \
  --role-name ecsTaskRole \
  --policy-name FaltuBaatS3DBSync \
  --policy-document file://deploy/ecs/iam-policy-s3-db.json
```

### 3. Enable S3 Sync in Task Definition
Update environment variables:
```json
{
  "name": "S3_SYNC_ENABLED",
  "value": "true"
},
{
  "name": "S3_DB_BUCKET",
  "value": "faltubaat-test-db"
}
```

### 4. Remove EFS Volume (Optional)
For testing without EFS, remove the `app-data` volume and mount point.

### How It Works
- **On startup**: Downloads database from S3 (if exists)
- **Every 5 min**: Syncs database to S3
- **On shutdown**: Uploads final database state

⚠️ **Warning**: S3 sync is for testing only. Use EFS for production (data consistency).

Update in `service-definition.json`:

| Placeholder | Description |
|-------------|-------------|
| `subnet-XXXXXXXXX` | VPC Subnet IDs |
| `sg-XXXXXXXXX` | Security Group ID |

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
| Inbound | 3000 | ALB SG | HTTP traffic |
| Inbound | 3443 | ALB SG | HTTPS traffic |
| Inbound | 1935 | 0.0.0.0/0 | RTMP streaming |
| Inbound | 8080 | ALB SG | HLS streams |
| Inbound | 2049 | VPC CIDR | EFS mount |

## Limitations

- ⚠️ Single health check for both processes
- ⚠️ Cannot scale Node.js and Nginx independently
- ⚠️ Mixed CloudWatch logs
- ⚠️ If Nginx crashes, container may stay "healthy"

For production, consider using the **multi-container** deployment instead.
