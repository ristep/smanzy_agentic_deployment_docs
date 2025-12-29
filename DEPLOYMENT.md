# Production Deployment Plan

## Overview

Zero-downtime, rollback-safe deployment architecture for Debian 12 using Docker Compose and Nginx.

---

## Server & Domain

- URL: https://smanzary.vozigo.com/
- OS: Debian 12
- Web Server: Nginx
- TLS: Let’s Encrypt via Certbot

---

## Architecture

```text
Internet
   |
Nginx (TLS)
   |
Frontend (React) --- Backend (Go)
```
