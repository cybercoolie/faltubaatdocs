---
layout: default
title: EC2 Deployment
nav_order: 4
description: "AWS EC2/VM deployment guide for FaltuBaat - Single and multi-instance options"
permalink: /docs/ec2/
---

# EC2/VM Deployment Guide
{: .no_toc }

Deploy FaltuBaat on AWS EC2 instances with native service management.
{: .fs-6 .fw-300 }

---

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Deployment Options

Choose based on your scale and requirements:

| Option | Description | Location | Best For |
|--------|-------------|----------|----------|
| **Single EC2** | App + RTMP on one instance | `deploy/ec2/single-ec2/` | Development, small scale |
| **Multi EC2** | Separate App and RTMP servers | `deploy/ec2/multi-ec2/` | Production, high traffic |

### Quick Comparison

| Feature | Single EC2 | Multi EC2 |
|---------|------------|-----------|
| **Instances** | 1 | 2 |
| **Setup Complexity** | Simple | Moderate |
| **Cost** | Lower | Higher |
| **Scaling** | Limited | Independent |
| **Fault Tolerance** | Low | Higher |
| **Maintenance** | Easier | More flexible |

### When to Use Each

| Choose Single EC2 if... | Choose Multi EC2 if... |
|-------------------------|------------------------|
| Development or testing | Production workload |
| Low traffic (< 100 users) | High traffic (100+ users) |
| Limited budget | Need independent scaling |
| Simple maintenance | Want fault isolation |
| Few streams (< 5) | Many streams (5+) |

---

## Prerequisites

### Supported Operating Systems

| OS | Version | Status |
|----|---------|--------|
| Amazon Linux | 2023, 2 | ✅ Supported |
| Ubuntu | 22.04, 24.04 | ✅ Supported |
| Debian | 11, 12 | ✅ Supported |
| RHEL/CentOS | 8, 9 | ✅ Supported |

### Minimum EC2 Instance Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| Instance Type | t2.micro | t3.medium |
| vCPU | 1 | 2+ |
| RAM | 1 GB | 4 GB |
| Storage | 8 GB | 20 GB |

### Required Ports

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 3000 | TCP | HTTP (Node.js) |
| 3443 | TCP | HTTPS (Node.js) |
| 1935 | TCP | RTMP (Streaming) |
| 8080 | TCP | HLS (Video playback) |

---

## Single EC2 Deployment

Deploy FaltuBaat on a single EC2 instance with Node.js and Nginx/RTMP running as native services.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      EC2 Instance                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────┐    ┌─────────────────────────────┐ │
│  │   Node.js App       │    │   Nginx + RTMP Module       │ │
│  │   (systemd)         │    │   (systemd)                 │ │
│  │                     │    │                             │ │
│  │   Port 3000 (HTTP)  │◄───│   on_publish callback       │ │
│  │   Port 3443 (HTTPS) │    │   Port 1935 (RTMP)          │ │
│  │                     │    │   Port 8080 (HLS)           │ │
│  └──────────┬──────────┘    └──────────────┬──────────────┘ │
│             │                              │                │
│  ┌──────────▼──────────┐    ┌──────────────▼──────────────┐ │
│  │  /opt/faltubaat/    │    │   /var/www/html/hls/        │ │
│  │  data/faltubaat.db  │    │   (HLS stream files)        │ │
│  └─────────────────────┘    └─────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Files

| File | Purpose |
|------|---------|
| `deploy/ec2/single-ec2/ec2-install.sh` | Full installation script |
| `deploy/ec2/single-ec2/ec2-uninstall.sh` | Remove application completely |
| `deploy/ec2/single-ec2/nginx-ec2.conf` | Nginx configuration |

### Security Group

| Port | Protocol | Source | Description |
|------|----------|--------|-------------|
| 22 | TCP | Your IP | SSH access |
| 3000 | TCP | 0.0.0.0/0 | HTTP Chat |
| 3443 | TCP | 0.0.0.0/0 | HTTPS Chat |
| 1935 | TCP | 0.0.0.0/0 | RTMP Streaming |
| 8080 | TCP | 0.0.0.0/0 | HLS Streams |

### Installation

#### Step 1: Connect to EC2

```bash
ssh -i your-key.pem ec2-user@your-ec2-ip    # Amazon Linux
ssh -i your-key.pem ubuntu@your-ec2-ip       # Ubuntu
```

#### Step 2: Upload Application Files

From your local machine:
```bash
# Upload entire project
scp -i your-key.pem -r ./* ec2-user@your-ec2-ip:~/faltubaat/

# Or use rsync for faster updates
rsync -avz -e "ssh -i your-key.pem" ./ ec2-user@your-ec2-ip:~/faltubaat/
```

#### Step 3: Run Installation

```bash
git clone https://github.com/faltubaatdeploy/faltubaat.git 
cd faltubaat/deploy/ec2/single-ec2
chmod +x *.sh
sudo ./ec2-install.sh
```

The script will:
- Detect your OS (Amazon Linux or Ubuntu)
- Install Node.js 20, Nginx with RTMP module
- Create application user and directories
- Configure systemd services
- Generate SSL certificates
- Initialize the database
- Start all services

#### Step 4: Access Your Application

| Service | URL |
|---------|-----|
| Chat App (HTTP) | `http://YOUR_EC2_IP:3000` |
| Chat App (HTTPS) | `https://YOUR_EC2_IP:3443` |
| RTMP Server | `rtmp://YOUR_EC2_IP:1935/live` |
| HLS Streams | `http://YOUR_EC2_IP:8080/hls/` |

---

## Multi EC2 Deployment

Deploy FaltuBaat across **two EC2 instances** for better performance, scaling, and fault isolation.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Cloud                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────┐      ┌─────────────────────────────────┐  │
│  │      APP SERVER (EC2)       │      │       RTMP SERVER (EC2)         │  │
│  │                             │      │                                 │  │
│  │  ┌───────────────────────┐  │      │  ┌───────────────────────────┐  │  │
│  │  │      Node.js App      │  │◄─────│  │   Nginx + RTMP Module     │  │  │
│  │  │                       │  │      │  │                           │  │  │
│  │  │   Port 3000 (HTTP)    │  │ HTTP │  │   Port 1935 (RTMP)        │  │  │
│  │  │   Port 3443 (HTTPS)   │  │ call │  │   Port 8080 (HLS)         │  │  │
│  │  │                       │  │back  │  │                           │  │  │
│  │  └───────────┬───────────┘  │      │  └─────────────┬─────────────┘  │  │
│  │              │              │      │                │                │  │
│  │  ┌───────────▼───────────┐  │      │  ┌─────────────▼─────────────┐  │  │
│  │  │   /opt/faltubaat/     │  │      │  │   /var/www/html/hls/      │  │  │
│  │  │   data/faltubaat.db   │  │      │  │   (HLS stream files)      │  │  │
│  │  └───────────────────────┘  │      │  └───────────────────────────┘  │  │
│  │                             │      │                                 │  │
│  │  Instance: t3.small         │      │  Instance: c5.large             │  │
│  │  (General purpose)          │      │  (Compute optimized)            │  │
│  └─────────────────────────────┘      └─────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why Multi-EC2?

| Aspect | Single EC2 | Multi-EC2 |
|--------|------------|-----------|
| **Scaling** | Scale everything | Scale app/RTMP independently |
| **Instance Type** | One size fits all | Optimize per workload |
| **Fault Isolation** | One failure = total outage | Partial failures |
| **Resource Usage** | RTMP competes with app | Dedicated resources |
| **Cost** | Single instance | Can right-size each |
| **Maintenance** | Single update | Update independently |

### Recommended Instance Types

| Server | Instance Type | Why |
|--------|---------------|-----|
| App Server | t3.small/medium | General purpose, handles chat & DB |
| RTMP Server | c5.large/xlarge | Compute optimized for video encoding |

### Security Groups

**App Server Security Group:**

| Port | Protocol | Source | Description |
|------|----------|--------|-------------|
| 22 | TCP | Your IP | SSH |
| 3000 | TCP | 0.0.0.0/0 | HTTP Chat |
| 3443 | TCP | 0.0.0.0/0 | HTTPS Chat |
| 3000 | TCP | RTMP SG | Callbacks from RTMP |

**RTMP Server Security Group:**

| Port | Protocol | Source | Description |
|------|----------|--------|-------------|
| 22 | TCP | Your IP | SSH |
| 1935 | TCP | 0.0.0.0/0 | RTMP Streaming |
| 8080 | TCP | 0.0.0.0/0 | HLS Streams |

{: .warning }
> ⚠️ Both instances should be in the same VPC for private network communication.

### Installation

#### Step 1: Launch EC2 Instances

1. Launch **App Server** EC2 (t3.small, Amazon Linux 2023 or Ubuntu)
2. Launch **RTMP Server** EC2 (c5.large, Amazon Linux 2023 or Ubuntu)
3. Note the **Private IP** of each instance

#### Step 2: Install App Server

```bash
# SSH to App Server
ssh -i your-key.pem ec2-user@APP_PUBLIC_IP

# Upload application files (from local machine)
scp -i your-key.pem -r ./* ec2-user@APP_PUBLIC_IP:~/faltubaat/

# Run installation
git clone https://github.com/faltubaatdeploy/faltubaat.git
cd faltubaat/deploy/ec2/multi-ec2
chmod +x *.sh
sudo ./install-app-server.sh

# When prompted, enter the RTMP server's PRIVATE IP
```

#### Step 3: Install RTMP Server

```bash
# SSH to RTMP Server
ssh -i your-key.pem ec2-user@RTMP_PUBLIC_IP

# Upload install script
scp -i your-key.pem ./deploy/ec2/multi-ec2/install-rtmp-server.sh ec2-user@RTMP_PUBLIC_IP:~/

# Run installation
chmod +x install-rtmp-server.sh
sudo ./install-rtmp-server.sh

# When prompted, enter the App server's PRIVATE IP
```

#### Step 4: Verify Connection

```bash
# On RTMP server, test callback to App server
curl http://APP_PRIVATE_IP:3000/

# Should return HTML or a response from the app
```

---

## Configuration

### File Locations

| Path | Description |
|------|-------------|
| `/opt/faltubaat/` | Application files |
| `/opt/faltubaat/data/` | SQLite database |
| `/opt/faltubaat/.env` | Environment configuration |
| `/opt/faltubaat/key.pem` | SSL private key |
| `/opt/faltubaat/cert.pem` | SSL certificate |
| `/var/www/html/hls/` | HLS stream files |
| `/var/log/faltubaat/` | Application logs |
| `/var/log/nginx/` | Nginx logs |
| `/etc/nginx/nginx.conf` | Nginx configuration |
| `/etc/systemd/system/faltubaat.service` | Systemd service |

### Environment Variables

Edit `/opt/faltubaat/.env`:

```bash
sudo nano /opt/faltubaat/.env
```

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_ENV` | production | Environment mode |
| `PORT` | 3000 | HTTP port |
| `HTTPS_PORT` | 3443 | HTTPS port |
| `JWT_SECRET` | (auto-generated) | Token signing secret |
| `DB_PATH` | /opt/faltubaat/data/faltubaat.db | Database path |
| `MAX_CONCURRENT_STREAMS` | 10 | Max simultaneous streams |
| `STREAM_COOLDOWN_MS` | 10000 | Cooldown between streams |

After editing, restart the service:
```bash
sudo systemctl restart faltubaat
```

---

## Service Management

### Application Service

| Action | Command |
|--------|---------|
| Check status | `sudo systemctl status faltubaat` |
| Start service | `sudo systemctl start faltubaat` |
| Stop service | `sudo systemctl stop faltubaat` |
| Restart service | `sudo systemctl restart faltubaat` |
| View logs | `sudo journalctl -u faltubaat -f` |
| View last 100 logs | `sudo journalctl -u faltubaat -n 100` |

### Nginx Service

| Action | Command |
|--------|---------|
| Check status | `sudo systemctl status nginx` |
| Restart | `sudo systemctl restart nginx` |
| Test config | `sudo nginx -t` |
| View logs | `sudo tail -f /var/log/nginx/error.log` |

---

## Security Setup

### AWS Security Group Configuration

Create inbound rules for your EC2 Security Group:

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | Your IP | SSH access |
| Custom TCP | TCP | 3000 | 0.0.0.0/0 | HTTP app |
| Custom TCP | TCP | 3443 | 0.0.0.0/0 | HTTPS app |
| Custom TCP | TCP | 1935 | 0.0.0.0/0 | RTMP streaming |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | HLS playback |

### Firewall Configuration (if enabled)

**Amazon Linux (firewalld):**
```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=3443/tcp
sudo firewall-cmd --permanent --add-port=1935/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

**Ubuntu (ufw):**
```bash
sudo ufw allow 3000/tcp
sudo ufw allow 3443/tcp
sudo ufw allow 1935/tcp
sudo ufw allow 8080/tcp
sudo ufw reload
```

---

## SSL Certificates

### Using Let's Encrypt (Recommended for Production)

```bash
# Install Certbot
sudo apt install certbot  # Ubuntu
sudo yum install certbot  # Amazon Linux

# Generate certificate (requires domain name)
sudo certbot certonly --standalone -d yourdomain.com

# Update .env with cert paths
echo "SSL_CERT=/etc/letsencrypt/live/yourdomain.com/fullchain.pem" | sudo tee -a /opt/faltubaat/.env
echo "SSL_KEY=/etc/letsencrypt/live/yourdomain.com/privkey.pem" | sudo tee -a /opt/faltubaat/.env

sudo systemctl restart faltubaat
```

### Auto-renewal
```bash
# Add to crontab
echo "0 0 1 * * certbot renew --quiet && systemctl restart faltubaat" | sudo tee -a /etc/crontab
```

---

## Streaming Guide

### How to Start a Live Stream

1. **Login** to the application at `https://YOUR_EC2_IP:3443`
2. **Navigate** to the Live section
3. **Click** "Go Live" to start streaming from your browser

### Stream with OBS Studio

1. Open OBS Studio
2. Go to **Settings** → **Stream**
3. Configure:
   - **Service:** Custom
   - **Server:** `rtmp://YOUR_EC2_IP:1935/live` (or RTMP Server IP for Multi-EC2)
   - **Stream Key:** `your-stream-key`
4. Click **Start Streaming**

### Test Stream with FFmpeg

```bash
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=1000:sample_rate=44100 \
  -c:v libx264 -preset ultrafast -tune zerolatency \
  -c:a aac -ar 44100 -ac 2 \
  -f flv rtmp://YOUR_EC2_IP:1935/live/test
```

### Watch HLS Stream

Open in browser or VLC:
```
http://YOUR_EC2_IP:8080/hls/test.m3u8
```

---

## Troubleshooting

### Application Won't Start

```bash
# Check service status
sudo systemctl status faltubaat

# Check detailed logs
sudo journalctl -u faltubaat -n 50 --no-pager

# Check if port is in use
sudo netstat -tlnp | grep -E "3000|3443"

# Test manually
cd /opt/faltubaat
sudo -u faltubaat node server-https.js
```

### RTMP Streaming Not Working

```bash
# Check Nginx status
sudo systemctl status nginx

# Check Nginx error logs
sudo tail -f /var/log/nginx/error.log

# Test Nginx configuration
sudo nginx -t

# Check if RTMP port is listening
sudo netstat -tlnp | grep 1935

# Test RTMP port
telnet localhost 1935
```

### Permission Errors

```bash
# Fix ownership
sudo chown -R faltubaat:faltubaat /opt/faltubaat
sudo chown -R faltubaat:faltubaat /var/www/html/hls

# Fix permissions
sudo chmod 755 /opt/faltubaat
sudo chmod 755 /var/www/html/hls
```

### Database Issues

```bash
# Check database file
ls -la /opt/faltubaat/data/

# Reinitialize database
cd /opt/faltubaat
sudo -u faltubaat npm run init-db

# Check permissions
sudo chown -R faltubaat:faltubaat /opt/faltubaat/data
```

---

## Updating the Application

### Quick Update

```bash
# Upload new files to EC2
scp -i your-key.pem -r ./* ec2-user@YOUR_EC2_IP:~/faltubaat/

# SSH and restart
ssh -i your-key.pem ec2-user@YOUR_EC2_IP
sudo systemctl restart faltubaat
```

### Full Update

```bash
# Stop service
sudo systemctl stop faltubaat

# Backup current version
sudo cp -r /opt/faltubaat /opt/faltubaat.backup

# Copy new files
sudo cp -r ~/faltubaat/* /opt/faltubaat/
sudo chown -R faltubaat:faltubaat /opt/faltubaat

# Update dependencies
cd /opt/faltubaat
sudo -u faltubaat npm install --production

# Start service
sudo systemctl start faltubaat
```

---

## Uninstallation

### Single EC2

```bash
sudo ./deploy/ec2/single-ec2/ec2-uninstall.sh
```

### Manual Uninstall

```bash
# Stop and disable service
sudo systemctl stop faltubaat
sudo systemctl disable faltubaat

# Remove files
sudo rm -f /etc/systemd/system/faltubaat.service
sudo rm -rf /opt/faltubaat
sudo rm -rf /var/www/html/hls
sudo rm -rf /var/log/faltubaat

# Remove user
sudo userdel faltubaat

# Reload systemd
sudo systemctl daemon-reload
```

---

## Related Documentation

- [Docker Deployment](../docker/) - Container-based deployment
- [ECS Single Container](../ecs-single/) - AWS ECS single container
- [ECS Multi Container](../ecs-multi/) - AWS ECS multi container
- [EKS Deployment](../eks/) - Kubernetes deployment

