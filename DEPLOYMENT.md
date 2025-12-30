# Production Deployment Plan

## Overview

This document defines a **zero-downtime, rollback-safe deployment architecture**
for a full-stack web application running on **Debian 12** using **Docker Compose**
and **Nginx**.

---

## Server & Domain

- URL: https://smanzary.vozigo.com/
- OS: Debian 12
- Web Server: Nginx (host) + Nginx (frontend container)
- TLS: Let's Encrypt via Certbot (Nginx plugin)
- Access: SSH

---

## Project Structure

```text
smanzari_site/
├── docker-compose.prod.yml    # Production orchestration
├── .env                       # Environment variables (not in git)
├── scripts/
│   ├── deploy.sh              # Main deployment script
│   ├── rollback.sh            # Rollback to previous version
│   └── health-check.sh        # Health verification
├── nginx_conf/
│   └── smanzary.vozigo.com.conf  # Host Nginx config
├── smanzy_backend/
│   ├── Dockerfile             # Go multi-stage build
│   └── ...
└── smanzy_react_spa/
    ├── Dockerfile             # React multi-stage build
    ├── nginx.conf             # Container Nginx config (SPA)
    └── ...
```

---

## High-Level Architecture

```text
                   ┌─────────────────────┐
                   │   Internet / DNS    │
                   └─────────┬───────────┘
                             │
                    https://smanzary.vozigo.com
                             │
                   ┌─────────▼───────────┐
                   │   Host Nginx        │
                   │   (TLS + Routing)   │
                   │   Port 80/443       │
                   └─────────┬───────────┘
             ┌───────────────┴───────────────┐
             │                               │
   ┌─────────▼─────────┐           ┌─────────▼─────────┐
   │  smanzy_frontend  │           │  smanzy_backend   │
   │  (Nginx + React)  │           │  (Go API)         │
   │  Container :80    │           │  Container :8080  │
   └─────────┬─────────┘           └─────────┬─────────┘
             │                               │
        /health                    /health endpoint
                                             │
                               ┌─────────────▼─────────────┐
                               │     smanzy_postgres       │
                               │     (PostgreSQL 16)       │
                               │     Container :5432       │
                               └───────────────────────────┘
```

---

## Prerequisites

### Server Requirements

- Debian 12 (Bookworm)
- Docker Engine 24.0+
- Docker Compose v2.20+
- Nginx 1.22+
- Certbot with Nginx plugin
- Git

### Install Dependencies (Debian 12)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt install docker-compose-plugin -y

# Install Nginx
sudo apt install nginx -y

# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Verify installations
docker --version
docker compose version
nginx -v
certbot --version
```

---

## Environment Configuration

### Create `.env` file

Create a `.env` file in the project root (`smanzari_site/.env`):

```bash
# Database
POSTGRES_USER=smanzy_user
POSTGRES_PASSWORD=<strong-random-password>
POSTGRES_DB=smanzy_db

# Backend
JWT_SECRET=<strong-random-secret-min-32-chars>
SERVER_PORT=8080

# Optional: PgAdmin (remove in production if not needed)
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=<strong-password>
```

**Security Notes:**
- Never commit `.env` to version control
- Use strong, randomly generated passwords
- Rotate secrets periodically

---

## Initial Server Setup

### 1. Clone Repository

```bash
cd /srv
sudo git clone <repository-url> smanzy
sudo chown -R $USER:$USER /srv/smanzy
cd /srv/smanzy
```

### 2. Create Required Directories

```bash
# Uploads directory (for media files)
sudo mkdir -p /srv/smanzy/uploads
sudo chown -R 1000:1000 /srv/smanzy/uploads

# Certbot webroot
sudo mkdir -p /var/www/certbot
```

### 3. Configure Host Nginx

```bash
# Copy nginx config
sudo cp /srv/smanzy/nginx_conf/smanzary.vozigo.com.conf /etc/nginx/sites-available/

# Enable site
sudo ln -sf /etc/nginx/sites-available/smanzary.vozigo.com.conf /etc/nginx/sites-enabled/

# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Test config (will fail until TLS is set up)
sudo nginx -t
```

### 4. Obtain TLS Certificate

```bash
# First, temporarily modify nginx config to allow HTTP for ACME challenge
# Then run certbot:
sudo certbot --nginx -d smanzary.vozigo.com

# Certbot will:
# - Obtain certificate
# - Configure auto-renewal
# - Update nginx config if needed
```

### 5. Make Scripts Executable

```bash
chmod +x /srv/smanzy/scripts/*.sh
```

---

## Deployment

### First Deployment

```bash
cd /srv/smanzy

# Create .env file (see Environment Configuration)
cp .env.example .env
nano .env

# Run deployment
./scripts/deploy.sh
```

### Subsequent Deployments

```bash
cd /srv/smanzy

# Pull latest changes
git pull origin main

# Deploy
./scripts/deploy.sh
```

### What `deploy.sh` Does

1. **Validates** `.env` file exists
2. **Backs up** current Docker images with timestamp tag
3. **Builds** new images (no cache)
4. **Stops** old containers gracefully
5. **Starts** new containers
6. **Verifies** health of all services
7. **Reports** backup tag for rollback

---

## Rollback

If a deployment fails or causes issues:

```bash
# List available backups
./scripts/rollback.sh

# Rollback to specific backup
./scripts/rollback.sh backup-20250101-120000
```

### Manual Rollback

```bash
# Stop current containers
docker compose -f docker-compose.prod.yml stop backend frontend

# Restore backup images
docker tag smanzy_backend:backup-YYYYMMDD-HHMMSS smanzy_backend:latest
docker tag smanzy_frontend:backup-YYYYMMDD-HHMMSS smanzy_frontend:latest

# Start containers
docker compose -f docker-compose.prod.yml up -d backend frontend
```

---

## Health Checks

### Using the Script

```bash
./scripts/health-check.sh
```

### Manual Checks

```bash
# Container status
docker ps | grep smanzy

# Container health
docker inspect --format='{{.State.Health.Status}}' smanzy_backend
docker inspect --format='{{.State.Health.Status}}' smanzy_frontend
docker inspect --format='{{.State.Health.Status}}' smanzy_postgres

# API health endpoint
curl -f http://localhost:8080/health

# Frontend
curl -f http://localhost:80/health

# External (through Nginx)
curl -f https://smanzary.vozigo.com/api/health
curl -f https://smanzary.vozigo.com/
```

---

## Database Management

### Run Migrations

```bash
# Enter backend container and run with migrate flag
docker exec -it smanzy_backend ./server -migrate
```

### Database Backup

```bash
# Create backup
docker exec smanzy_postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > backup_$(date +%Y%m%d).sql

# Restore backup
cat backup_20250101.sql | docker exec -i smanzy_postgres psql -U $POSTGRES_USER $POSTGRES_DB
```

---

## Logs & Debugging

### View Logs

```bash
# All services
docker compose -f docker-compose.prod.yml logs -f

# Specific service
docker compose -f docker-compose.prod.yml logs -f backend
docker compose -f docker-compose.prod.yml logs -f frontend
docker compose -f docker-compose.prod.yml logs -f postgres

# Host Nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### Enter Container

```bash
docker exec -it smanzy_backend sh
docker exec -it smanzy_frontend sh
docker exec -it smanzy_postgres psql -U $POSTGRES_USER $POSTGRES_DB
```

---

## SSL Certificate Renewal

Certbot auto-renewal is configured via systemd timer. To verify:

```bash
# Check timer status
sudo systemctl status certbot.timer

# Test renewal (dry run)
sudo certbot renew --dry-run

# Manual renewal if needed
sudo certbot renew
sudo systemctl reload nginx
```

---

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker compose -f docker-compose.prod.yml logs backend

# Common issues:
# - Missing .env file
# - Database not ready (check postgres health)
# - Port already in use
```

### 502 Bad Gateway

```bash
# Check if containers are running
docker ps | grep smanzy

# Check container health
docker inspect --format='{{.State.Health.Status}}' smanzy_backend

# Check Nginx can reach containers
docker network inspect smanzari_site_smanzy_network
```

### Database Connection Failed

```bash
# Check postgres is healthy
docker inspect --format='{{.State.Health.Status}}' smanzy_postgres

# Check DB_DSN in backend logs
docker logs smanzy_backend | head -20

# Verify credentials in .env match
```

### TLS/Certificate Issues

```bash
# Check certificate status
sudo certbot certificates

# Check nginx config
sudo nginx -t

# Verify certificate files exist
ls -la /etc/letsencrypt/live/smanzary.vozigo.com/
```

---

## Security Checklist

- [ ] `.env` file has correct permissions (`chmod 600 .env`)
- [ ] `.env` is in `.gitignore`
- [ ] Strong passwords for database and JWT secret
- [ ] Firewall allows only ports 80, 443, 22
- [ ] SSH key authentication (disable password auth)
- [ ] Regular security updates (`apt update && apt upgrade`)
- [ ] TLS certificate auto-renewal working
- [ ] Database backups scheduled

---

## Useful Commands Reference

```bash
# Start all services
docker compose -f docker-compose.prod.yml up -d

# Stop all services
docker compose -f docker-compose.prod.yml down

# Rebuild single service
docker compose -f docker-compose.prod.yml build --no-cache backend
docker compose -f docker-compose.prod.yml up -d backend

# View resource usage
docker stats

# Clean up unused images
docker image prune -a

# Restart host Nginx
sudo systemctl restart nginx
```
