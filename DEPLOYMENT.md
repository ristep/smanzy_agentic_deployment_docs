# Production Deployment Plan

## Overview

This document defines a **zero-downtime, rollback-safe deployment architecture**
for a full-stack web application running on **Debian 12** using **Docker Compose**
and **Nginx**.

---

## Server & Domain

- URL: https://smanzary.vozigo.com/
- OS: Debian 12
- Web Server: Nginx
- TLS: Let’s Encrypt via Certbot (Nginx plugin)
- Access: SSH

---

## Project Structure

- `smanzy_backend` — Go backend
- `smanzy_react_spa` — React (Vite) frontend
- `nginx_conf` — Nginx configs only

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
                   │        Nginx         │
                   │  (Reverse Proxy)    │
                   └─────────┬───────────┘
             ┌───────────────┴───────────────┐
             │                               │
   ┌─────────▼─────────┐           ┌─────────▼─────────┐
   │  Frontend (React) │           │   Backend (Go)    │
   │  Vite Build       │           │  REST / API       │
   │  Docker Container │           │  Docker Container │
   └─────────┬─────────┘           └─────────┬─────────┘
             │                               │
     Health Check                     /health + /metrics
