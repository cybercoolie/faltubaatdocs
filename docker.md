# Docker Deployment Guide

This guide covers Docker deployment options for FaltuBaat - a live chat application with RTMP streaming.

## Folder Structure

```
deploy/docker/
├── single-container/          # All-in-one Docker container
│   ├── Dockerfile             # Node.js + Nginx + RTMP in one image
│   ├── run.sh                 # Build and run script
│   ├── start-services.sh      # Container entrypoint (starts both services)
│   └── s3-db-sync.sh          # Optional S3 database sync
│
└── multi-container/           # Separate containers with docker-compose
    ├── docker-compose.yml     # Service definitions
    ├── Dockerfile.app         # Node.js chat application
    ├── Dockerfile.nginx-rtmp  # Nginx RTMP streaming server
    ├── nginx-rtmp.conf        # Nginx configuration
    ├── start.sh               # Build and start script
    ├── start-app.sh           # App container entrypoint
    └── s3-db-sync.sh          # Optional S3 database sync
```

---

## Single Container Deployment

Runs Node.js + Nginx/RTMP in **one container**. Simpler but less flexible.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│              faltubaat-single container                 │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │   Node.js App   │    │   Nginx + RTMP Module       │ │
│  │   Port 3000     │◀───│   Port 1935 (RTMP)          │ │
│  │   Port 3443     │    │   Port 8080 (HLS)           │ │
│  └─────────────────┘    └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Deploy

```bash
cd deploy/docker/single-container
./run.sh
```

### Access

| Service | URL |
|---------|-----|
| Chat App (HTTP) | http://localhost:3000 |
| Chat App (HTTPS) | https://localhost:3443 |
| RTMP Server | rtmp://localhost:1935/live |
| HLS Streams | http://localhost:8080/hls/ |

### Management

```bash
docker logs -f faltubaat-single      # View logs
docker stop faltubaat-single         # Stop
docker start faltubaat-single        # Start
docker rm faltubaat-single           # Remove
docker exec -it faltubaat-single bash  # Shell access
```

### Pros/Cons

| Pros | Cons |
|------|------|
| ✅ Simple setup | ❌ Larger image size |
| ✅ Single container to manage | ❌ Can't scale services independently |
| ✅ All services bundled | ❌ Single health check for both |
| ✅ Easy to debug | ❌ Mixed logs |

---

## Multi-Container Deployment

Runs Node.js and Nginx/RTMP as **separate containers** via Docker Compose.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    chat-network (bridge)                    │
├─────────────────────────┬───────────────────────────────────┤
│      chat-app           │         nginx-rtmp                │
│  ┌─────────────────┐    │    ┌─────────────────────────┐    │
│  │   Node.js       │    │    │   Nginx + RTMP module   │    │
│  │   Port 3000     │◄───┼────│   on_publish callback   │    │
│  │   Port 3443     │    │    │   Port 1935 (RTMP)      │    │
│  │   SQLite DB     │    │    │   Port 8080 (HLS)       │    │
│  └─────────────────┘    │    └─────────────────────────┘    │
└─────────────────────────┴───────────────────────────────────┘
```

### Deploy

```bash
cd deploy/docker/multi-container
./start.sh
```

Or manually:

```bash
cd deploy/docker/multi-container
docker compose up -d --build
```

### Access

| Service | URL |
|---------|-----|
| Chat App (HTTP) | http://localhost:3000 |
| Chat App (HTTPS) | https://localhost:3443 |
| RTMP Server | rtmp://localhost:1935/live |
| HLS Streams | http://localhost:8080/hls/ |

### Management

```bash
docker compose logs -f              # All logs
docker compose logs -f chat-app     # App logs only
docker compose logs -f nginx-rtmp   # Nginx logs only
docker compose down                 # Stop all
docker compose restart              # Restart all
docker compose ps                   # List containers
```

### Pros/Cons

| Pros | Cons |
|------|------|
| ✅ Independent scaling | ❌ More complex setup |
| ✅ Separate logs per service | ❌ Requires networking knowledge |
| ✅ Individual health checks | ❌ Multiple images to manage |
| ✅ Update services independently | |
| ✅ Production-ready architecture | |

---

## Quick Comparison

| Feature | Single Container | Multi-Container |
|---------|------------------|-----------------|
| Command | `./run.sh` | `./start.sh` |
| Containers | 1 | 2 |
| Image Size | ~800MB | ~200MB + ~150MB |
| Scaling | All or nothing | Per service |
| Logs | Mixed | Separate |
| ECS/K8s Ready | ⚠️ Limited | ✅ Yes |

---

## Persistent Storage

Both deployments use Docker volumes for persistence:

| Volume | Purpose |
|--------|---------|
| `faltubaat-db` or `db-data` | SQLite database |
| `faltubaat-hls` or `hls-data` | HLS stream files |

### View Volumes

```bash
docker volume ls
docker volume inspect faltubaat-db
```

### Backup Database

```bash
# Single container
docker cp faltubaat-single:/app/data/faltubaat.db ./backup.db

# Multi-container
docker cp faltubaat-app:/app/data/faltubaat.db ./backup.db
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_ENV` | production | Environment mode |
| `PORT` | 3000 | HTTP port |
| `HTTPS_PORT` | 3443 | HTTPS port |
| `JWT_SECRET` | (auto-generated) | Token signing secret |
| `DB_PATH` | /app/data/faltubaat.db | Database path |
| `S3_SYNC_ENABLED` | false | Enable S3 database sync |
| `S3_DB_BUCKET` | - | S3 bucket for database |

---

## Troubleshooting

### Container won't start

```bash
# Check logs
docker logs faltubaat-single

# Check if ports are in use
netstat -an | findstr "3000 1935 8080"
```

### RTMP not working

```bash
# Check nginx is running (single-container)
docker exec faltubaat-single pgrep nginx

# Check nginx logs
docker exec faltubaat-single cat /var/log/nginx/error.log
```

### Database issues

```bash
# Check if database exists
docker exec faltubaat-single ls -la /app/data/

# Reinitialize database
docker exec faltubaat-single npm run init-db
```

---

## Related Documentation

- [EC2 Deployment](ec2.md) - Deploy directly on EC2/VM
- [ECS Single Container](ecs-single-container.md) - AWS ECS with single container
- [ECS Multi-Container](ecs_multi-container.md) - AWS ECS with multiple containers
- [EKS Deployment](eks.md) - Kubernetes on AWS
