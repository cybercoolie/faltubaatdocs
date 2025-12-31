---
layout: home
title: Home
nav_order: 1
description: "FaltuBaat - Real-time chat application with video calling and live streaming capabilities"
permalink: /
---

# FaltuBaat Documentation
{: .fs-9 }

Real-time chat application with video calling and live streaming capabilities.
{: .fs-6 .fw-300 }

[Get Started]({{ site.baseurl }}/docs/getting-started/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/yourusername/faltubaat){: .btn .fs-5 .mb-4 .mb-md-0 }

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

## 🚀 Deployment Options

Choose the deployment method that fits your needs:

### 🐳 Docker Deployments

| Option | Description | Guide |
|--------|-------------|-------|
| **Single Container** | All-in-one container with app + nginx | [Docker Guide]({{ site.baseurl }}/docs/docker/) |
| **Multi Container** | Separate app and nginx containers | [Docker Guide]({{ site.baseurl }}/docs/docker/) |

### ☁️ AWS Deployments

| Option | Description | Guide |
|--------|-------------|-------|
| **EC2 Single** | One EC2 with app + nginx | [EC2 Guide]({{ site.baseurl }}/docs/ec2/) |
| **EC2 Multi** | Separate EC2 for app and RTMP | [EC2 Guide]({{ site.baseurl }}/docs/ec2/) |
| **ECS Single** | Single Fargate container | [ECS Single]({{ site.baseurl }}/docs/ecs-single/) |
| **ECS Multi** | Multi-container Fargate task | [ECS Multi]({{ site.baseurl }}/docs/ecs-multi/) |
| **EKS** | Kubernetes on AWS | [EKS Guide]({{ site.baseurl }}/docs/eks/) |

### 🌐 Other Cloud Providers

The Docker images are portable and can be deployed on:
- **Azure**: Container Apps, AKS, Azure VMs
- **GCP**: Cloud Run, GKE, Compute Engine
- **Any Cloud**: Use the Docker image with your provider's container service

> 💡 Just push the image to your registry and configure ports (3000, 1935, 8080)

---

## 📋 Quick Start

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

---

## 📖 Documentation Index

{: .important }
> Select a guide from the sidebar or use the links below to get started.

| Document | Description |
|----------|-------------|
| [Getting Started]({{ site.baseurl }}/docs/getting-started/) | Overview, features, and quick start |
| [Docker]({{ site.baseurl }}/docs/docker/) | Docker single & multi-container deployment |
| [EC2]({{ site.baseurl }}/docs/ec2/) | AWS EC2 single & multi-instance deployment |
| [ECS Single Container]({{ site.baseurl }}/docs/ecs-single/) | AWS ECS Fargate single container |
| [ECS Multi-Container]({{ site.baseurl }}/docs/ecs-multi/) | AWS ECS Fargate multi-container |
| [EKS]({{ site.baseurl }}/docs/eks/) | AWS EKS Kubernetes deployment |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing`)
5. Open a Pull Request

---

<p align="center">
  <strong>Made with ❤️ by FaltuBaat Team</strong>
</p>
