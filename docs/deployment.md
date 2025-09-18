# Deployment Guide - AI Cryptocurrency Trading System

## Overview

This guide provides comprehensive instructions for deploying the AI Cryptocurrency Trading System in a local/on-premise environment. The system is designed for secure, high-performance trading operations with minimal external dependencies.

## Prerequisites

### Hardware Requirements
- **CPU**: 8+ cores (Intel i7/i9 or AMD Ryzen 7/9)
- **RAM**: 32GB minimum (64GB recommended for ML operations)
- **Storage**: 2TB NVMe SSD (for high-speed data access)
- **Network**: Gigabit Ethernet with low-latency internet connection
- **GPU**: Optional NVIDIA GPU for ML model training acceleration

### Software Requirements
- **Operating System**: Ubuntu 20.04 LTS or Windows 10/11 Pro
- **Docker**: Version 20.10+
- **Docker Compose**: Version 2.0+
- **Git**: Version 2.30+
- **Node.js**: Version 18+ (for frontend development)
- **Python**: Version 3.11+ (for backend development)

---

## Quick Start Deployment

### 1. Clone Repository
```bash
git clone https://github.com/your-org/ai-trading-system.git
cd ai-trading-system
```

### 2. Environment Configuration
```bash
# Copy environment template
cp .env.example .env

# Edit configuration
nano .env
```

**Environment Variables**:
```bash
# Database Configuration
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=ai_trading
POSTGRES_USER=trading_user
POSTGRES_PASSWORD=secure_password_123

# Redis Configuration
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis_password_123

# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
SECRET_KEY=your_super_secret_key_here
JWT_SECRET=jwt_secret_key_here

# Exchange API Keys (Binance)
BINANCE_API_KEY=your_binance_api_key
BINANCE_SECRET_KEY=your_binance_secret_key
BINANCE_TESTNET=true

# ML Configuration
ML_MODEL_PATH=/app/data/models
ML_TRAINING_DATA_PATH=/app/data/training

# Security
ENCRYPTION_KEY=your_encryption_key_here
SSL_CERT_PATH=/app/certs/cert.pem
SSL_KEY_PATH=/app/certs/key.pem
```

### 3. Quick Deploy with Docker
```bash
# Build and start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f
```

---

## Detailed Deployment Steps

### Step 1: System Preparation

#### Ubuntu/Linux Setup
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Install additional tools
sudo apt install -y git curl wget htop iotop
```

#### Windows Setup
```powershell
# Install Docker Desktop for Windows
# Download from: https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe

# Install Git for Windows
# Download from: https://git-scm.com/download/win

# Install Windows Subsystem for Linux (WSL2)
wsl --install
```

### Step 2: SSL Certificate Setup

#### Generate Self-Signed Certificates
```bash
# Create certificates directory
mkdir -p certs

# Generate private key
openssl genrsa -out certs/key.pem 2048

# Generate certificate
openssl req -new -x509 -key certs/key.pem -out certs/cert.pem -days 365 \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"

# Set permissions
chmod 600 certs/key.pem
chmod 644 certs/cert.pem
```

#### Production Certificates (Let's Encrypt)
```bash
# Install certbot
sudo apt install certbot

# Generate certificate for domain
sudo certbot certonly --standalone -d your-domain.com

# Copy certificates
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem certs/cert.pem
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem certs/key.pem
```

### Step 3: Database Setup

#### PostgreSQL with TimescaleDB
```bash
# Create data directory
mkdir -p data/postgres

# Start PostgreSQL container
docker run -d \
  --name postgres-trading \
  -e POSTGRES_DB=ai_trading \
  -e POSTGRES_USER=trading_user \
  -e POSTGRES_PASSWORD=secure_password_123 \
  -p 5432:5432 \
  -v $(pwd)/data/postgres:/var/lib/postgresql/data \
  timescale/timescaledb:latest-pg15

# Wait for database to start
sleep 10

# Initialize database schema
docker exec -i postgres-trading psql -U trading_user -d ai_trading < scripts/init_database.sql
```

#### Redis Setup
```bash
# Create Redis data directory
mkdir -p data/redis

# Start Redis container
docker run -d \
  --name redis-trading \
  -p 6379:6379 \
  -v $(pwd)/data/redis:/data \
  redis:7-alpine redis-server --requirepass redis_password_123
```

### Step 4: Backend Deployment

#### Build Backend Image
```bash
# Build Docker image
docker build -f backend/Dockerfile -t ai-trading-backend:latest backend/

# Or use docker-compose
docker-compose build backend
```

#### Backend Configuration
```yaml
# docker-compose.yml - Backend service
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - POSTGRES_HOST=postgres
      - REDIS_HOST=redis
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET_KEY=${BINANCE_SECRET_KEY}
    volumes:
      - ./data/models:/app/data/models
      - ./data/logs:/app/logs
      - ./certs:/app/certs
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
```

### Step 5: Frontend Deployment

#### Build Frontend
```bash
# Install dependencies
cd frontend
npm install

# Build production version
npm run build

# Return to root directory
cd ..
```

#### Frontend Configuration
```yaml
# docker-compose.yml - Frontend service
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=https://localhost:8000/api/v1
      - NEXT_PUBLIC_WS_URL=wss://localhost:8000/ws
    volumes:
      - ./certs:/app/certs
    depends_on:
      - backend
    restart: unless-stopped
```

### Step 6: Nginx Reverse Proxy

#### Nginx Configuration
```nginx
# nginx/nginx.conf
upstream backend {
    server backend:8000;
}

upstream frontend {
    server frontend:3000;
}

server {
    listen 80;
    server_name localhost;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name localhost;

    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Frontend
    location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Backend API
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket
    location /ws {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Nginx Docker Service
```yaml
# docker-compose.yml - Nginx service
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - frontend
      - backend
    restart: unless-stopped
```

---

## Complete Docker Compose Configuration

### docker-compose.yml
```yaml
version: '3.8'

services:
  postgres:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init_database.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - SECRET_KEY=${SECRET_KEY}
      - JWT_SECRET=${JWT_SECRET}
      - BINANCE_API_KEY=${BINANCE_API_KEY}
      - BINANCE_SECRET_KEY=${BINANCE_SECRET_KEY}
      - BINANCE_TESTNET=${BINANCE_TESTNET}
    volumes:
      - ./data/models:/app/data/models
      - ./data/logs:/app/logs
      - ./certs:/app/certs
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=https://localhost/api/v1
      - NEXT_PUBLIC_WS_URL=wss://localhost/ws
    volumes:
      - ./certs:/app/certs
    depends_on:
      backend:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - frontend
      - backend
    restart: unless-stopped

  # ML Training Service (Optional)
  ml-trainer:
    build:
      context: ./backend
      dockerfile: Dockerfile.ml
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - ./data/models:/app/data/models
      - ./data/training:/app/data/training
    depends_on:
      postgres:
        condition: service_healthy
    restart: "no"
    profiles:
      - ml-training

volumes:
  postgres_data:
  redis_data:

networks:
  default:
    driver: bridge
```

---

## Production Deployment

### System Service Setup (systemd)
```bash
# Create systemd service file
sudo tee /etc/systemd/system/ai-trading.service > /dev/null <<EOF
[Unit]
Description=AI Trading System
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/ai-trading
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl enable ai-trading.service
sudo systemctl start ai-trading.service
```

### Monitoring Setup

#### Prometheus Configuration
```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ai-trading-backend'
    static_configs:
      - targets: ['backend:8000']
    metrics_path: '/metrics'

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
```

#### Grafana Dashboard
```yaml
# monitoring/docker-compose.monitoring.yml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/dashboards:/etc/grafana/provisioning/dashboards

volumes:
  grafana_data:
```

### Backup and Recovery

#### Database Backup Script
```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="ai_trading_backup_${DATE}.sql"

# Create backup directory
mkdir -p $BACKUP_DIR

# Perform backup
docker exec postgres-trading pg_dump -U trading_user -d ai_trading > $BACKUP_DIR/$BACKUP_FILE

# Compress backup
gzip $BACKUP_DIR/$BACKUP_FILE

# Remove backups older than 30 days
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_DIR/${BACKUP_FILE}.gz"
```

#### Automated Backup with Cron
```bash
# Add to crontab
crontab -e

# Daily backup at 2 AM
0 2 * * * /opt/ai-trading/scripts/backup.sh
```

---

## Security Hardening

### Firewall Configuration
```bash
# Ubuntu UFW
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.1.0/24 to any port 5432  # Database access from local network only
```

### Docker Security
```bash
# Run containers as non-root user
# Add to Dockerfile
RUN addgroup --system --gid 1001 trading
RUN adduser --system --uid 1001 --gid 1001 trading
USER trading

# Limit container resources
docker run --memory=2g --cpus=2 ai-trading-backend
```

### API Security
```python
# backend/app/middleware/security.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://localhost", "https://your-domain.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)

# Trusted hosts
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "your-domain.com"]
)
```

---

## Troubleshooting

### Common Issues

#### Database Connection Issues
```bash
# Check database status
docker logs postgres-trading

# Test connection
docker exec -it postgres-trading psql -U trading_user -d ai_trading -c "SELECT 1;"

# Reset database
docker-compose down
docker volume rm ai-trading_postgres_data
docker-compose up -d postgres
```

#### Memory Issues
```bash
# Check memory usage
docker stats

# Increase swap space
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

#### SSL Certificate Issues
```bash
# Regenerate certificates
rm -rf certs/*
openssl genrsa -out certs/key.pem 2048
openssl req -new -x509 -key certs/key.pem -out certs/cert.pem -days 365

# Restart services
docker-compose restart nginx
```

### Log Analysis
```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend

# Search logs for errors
docker-compose logs backend | grep ERROR

# Export logs
docker-compose logs --no-color > system.log
```

### Performance Monitoring
```bash
# System resources
htop
iotop
df -h

# Docker resources
docker system df
docker system prune

# Database performance
docker exec -it postgres-trading psql -U trading_user -d ai_trading -c "
SELECT query, calls, total_time, mean_time 
FROM pg_stat_statements 
ORDER BY total_time DESC 
LIMIT 10;"
```

---

## Maintenance

### Regular Maintenance Tasks

#### Daily
- Monitor system logs for errors
- Check disk space usage
- Verify backup completion
- Monitor trading performance

#### Weekly
- Update system packages
- Review security logs
- Analyze performance metrics
- Test backup restoration

#### Monthly
- Update Docker images
- Review and rotate API keys
- Perform security audit
- Update SSL certificates (if needed)

### Update Procedure
```bash
# 1. Backup current system
./scripts/backup.sh

# 2. Pull latest code
git pull origin main

# 3. Update Docker images
docker-compose pull

# 4. Rebuild and restart services
docker-compose down
docker-compose build
docker-compose up -d

# 5. Verify system health
docker-compose ps
curl -k https://localhost/api/v1/health
```

This deployment guide provides a comprehensive approach to setting up and maintaining the AI Cryptocurrency Trading System in a production environment with proper security, monitoring, and backup procedures.