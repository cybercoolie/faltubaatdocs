---
layout: default
title: Getting Started
nav_order: 2
description: "Getting started with FaltuBaat - Overview, features, and quick start guide"
permalink: /docs/getting-started/
---

# Getting Started with FaltuBaat
{: .no_toc }

Real-time chat application with video calling and live streaming capabilities
{: .fs-6 .fw-300 }

---

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 🎯 Overview

FaltuBaat is a full-featured real-time communication platform that provides:

- **💬 Real-time Chat** - Instant messaging with Socket.io
- **📹 Video Calling** - Peer-to-peer video communication
- **🎬 Live Streaming** - RTMP-based live broadcasting with HLS playback
- **🔐 Secure Authentication** - JWT-based user authentication
- **📱 Responsive Design** - Works on desktop and mobile

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      FaltuBaat Application                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │   Node.js Server    │    │   Nginx + RTMP Module       │ │
│  │                     │    │                             │ │
│  │   • Express API     │    │   • RTMP Ingest (1935)      │ │
│  │   • Socket.io       │◄───│   • HLS Output (8080)       │ │
│  │   • JWT Auth        │    │   • Stream Callbacks        │ │
│  │   • SQLite DB       │    │                             │ │
│  │                     │    │                             │ │
│  │   Port: 3000/3443   │    │   Port: 1935/8080           │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| **Backend** | Node.js 20, Express.js |
| **Real-time** | Socket.io |
| **Database** | SQLite (better-sqlite3) |
| **Authentication** | JWT (jsonwebtoken), bcrypt |
| **Streaming** | Nginx + RTMP Module, HLS |
| **Security** | Helmet, express-rate-limit |
| **Container** | Docker, Docker Compose |
| **Orchestration** | Kubernetes, AWS ECS |

---

## 📋 Features

### Chat & Communication
- ✅ Real-time messaging
- ✅ User presence (online/offline)
- ✅ Typing indicators
- ✅ Message history

### Video & Streaming
- ✅ Live video streaming (RTMP)
- ✅ HLS playback for viewers
- ✅ Stream status tracking
- ✅ OBS/FFmpeg compatible

### Security
- ✅ JWT authentication
- ✅ Password hashing (bcrypt)
- ✅ Rate limiting
- ✅ HTTPS support
- ✅ Security headers (Helmet)

---

## 🚀 Deployment Options

Choose the deployment method that fits your needs:

### 🐳 Docker Deployments

| Option | Description | Guide |
|--------|-------------|-------|
| **Single Container** | All-in-one container with app + nginx | [Docker Guide](../docker/) |
| **Multi Container** | Separate app and nginx containers | [Docker Guide](../docker/) |

### ☁️ AWS Deployments

| Option | Description | Guide |
|--------|-------------|-------|
| **EC2 Single** | One EC2 with app + nginx | [EC2 Guide](../ec2/) |
| **EC2 Multi** | Separate EC2 for app and RTMP | [EC2 Guide](../ec2/) |
| **ECS Single** | Single Fargate container | [ECS Single](../ecs-single/) |
| **ECS Multi** | Multi-container Fargate task | [ECS Multi](../ecs-multi/) |
| **EKS** | Kubernetes on AWS | [EKS Guide](../eks/) |

### 🌐 Other Cloud Providers

The Docker images are portable and can be deployed on:
- **Azure**: Container Apps, AKS, Azure VMs
- **GCP**: Cloud Run, GKE, Compute Engine
- **Any Cloud**: Use the Docker image with your provider's container service

{: .note }
> 💡 Just push the image to your registry and configure ports (3000, 1935, 8080)

---

## 📁 Project Structure

```
faltubaat/
├── server-https.js          # Main application server
├── db.js                     # Database initialization
├── package.json              # Node.js dependencies
├── public/                   # Frontend assets
│   ├── index.html
│   ├── chat.js
│   ├── styles.css
│   └── security.js
├── deploy/
│   ├── docker/
│   │   ├── single-container/ # All-in-one Docker setup
│   │   └── multi-container/  # Separate containers setup
│   ├── ec2/
│   │   ├── single-ec2/       # Single EC2 install scripts
│   │   └── multi-ec2/        # Multi EC2 install scripts
│   └── ecs/
│       ├── single-container/ # ECS task definitions
│       └── multi-container/  # ECS multi-container tasks
├── EKS/                      # Kubernetes manifests
└── Documentation/            # This documentation
```

---

## 🏃 Quick Start

### Option 1: Docker (Recommended)

```bash
# Clone the repository
git clone https://github.com/faltudeploy/faltubaat.git

# Run with Docker Compose
cd faltubaat/deploy/docker/single-container
docker-compose up -d

# Access the app
open http://localhost:3000
```

### Option 2: Local Development

```bash
# Install dependencies
npm install

# Initialize database
npm run init-db

# Start development server
npm run dev

# Access the app
open http://localhost:3000
```

### Option 3: EC2 Deployment

```bash
# Upload to EC2
scp -i your-key.pem -r ./* ec2-user@your-ec2-ip:~/faltubaat/

# SSH and install
ssh -i your-key.pem ec2-user@your-ec2-ip
cd ~/faltubaat/deploy/ec2/single-ec2
sudo ./ec2-install.sh
```

---

## 🔧 Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_ENV` | development | Environment mode |
| `PORT` | 3000 | HTTP port |
| `HTTPS_PORT` | 3443 | HTTPS port |
| `JWT_SECRET` | (required) | Token signing secret |
| `DB_PATH` | ./data/faltubaat.db | Database path |

### Ports

| Port | Protocol | Service |
|------|----------|---------|
| 3000 | HTTP | Web application |
| 3443 | HTTPS | Secure web application |
| 1935 | RTMP | Live streaming ingest |
| 8080 | HTTP | HLS stream playback |

---

## 📺 Streaming

### Start a Stream (OBS Studio)

1. Open OBS Studio
2. Go to **Settings** → **Stream**
3. Configure:
   - **Service**: Custom
   - **Server**: `rtmp://YOUR_SERVER:1935/live`
   - **Stream Key**: `your-stream-key`
4. Click **Start Streaming**

### Watch a Stream

Open in browser or VLC:
```
http://YOUR_SERVER:8080/hls/your-stream-key.m3u8
```

---

## 🔒 Security Considerations

{: .warning }
> Always use HTTPS in production and configure proper security settings.

- **JWT_SECRET**: Use a strong, unique secret in production
- **HTTPS**: Always use HTTPS in production
- **Rate Limiting**: Configured by default to prevent abuse
- **Secrets Management**: Use AWS Secrets Manager, Azure Key Vault, or similar

---

## 📞 Support

If you encounter issues:
1. Check the relevant deployment guide
2. Review logs for error messages
3. Open an issue on GitHub

---

**Made with ❤️ by FaltuBaat Team**
