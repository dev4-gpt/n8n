# 🚀 Self-Hosted n8n Automation Engine

## Core Infrastructure for GTM, Lead Gen & Marketing Workflows

A fully self-hosted, containerized n8n instance deployed on an independent cloud environment to orchestrate data pipelines, lead enrichment sequences, and marketing automation systems.

* **Live Instance Portal:** http://67.207.89.85:5678

---

## 🏗️ Architecture Overview

```text
┌────────────────────────────────────────────────────────────-┐
│                    Your Mac (local)                         │
│  ssh -i ~/.ssh/id_new_droplet root@67.207.89.85  ──► Droplet│
└────────────────────────────────────────────────────────────-┘
                                        │
                          ┌─────────────▼─────────────┐
                          │  Ubuntu 24.04 LTS (NYC1)  │
                          │  Docker & Docker Compose  │
                          │  ┌─────────────────────┐  │
                          │  │      n8n Core       │  │
                          │  │  Listener: :5678    │  │
                          │  └─────────────────────┘  │
                          └───────────────────────────┘

```

| Component | Specification |
| --- | --- |
| **Host Machine** | DigitalOcean Droplet (Ubuntu 24.04 LTS, 1 vCPU / 2 GB RAM) |
| **Network IP** | `67.207.89.85` |
| **Runtime Engine** | Standalone Docker Engine + Compose Plugin |
| **Data Persistence** | Volume mapped isolated storage (`n8n_data`) |
| **Config Location** | `~/n8n/docker-compose.yml` |

---

## ⚙️ Host Infrastructure Setup

Follow this sequential block sequence to configure a fresh server deployment from scratch:

### 1. OS & Runtime Preparation

Log into your clean cloud environment via SSH and run core package updates alongside the official automated Docker engine setup utilities:

```bash
# Update local package indexes and upgrade system core
apt update && apt upgrade -y

# Fetch and execute the official Docker engine utility deployment script
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh
sh get-docker.sh

```

### 2. Environment Configuration

Isolate your configuration matrix inside the root application space:

```bash
mkdir ~/n8n
cd ~/n8n
nano docker-compose.yml

```

Paste the following container engine layout declaration into the file:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=67.207.89.85
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - N8N_SECURE_COOKIE=false
      - WEBHOOK_URL=[http://67.207.89.85:5678/](http://67.207.89.85:5678/)
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.local/share/n8n

volumes:
  n8n_data:

```

Save the changes (`Ctrl + O`, `Enter`) and close down the text editor (`Ctrl + X`).

### 3. Stack Initialization

Bring your automation network online decoupled in background execution mode:

```bash
docker compose up -d

```

Access the fresh orchestration board interface at: `http://67.207.89.85:5678/setup`

---

## 🛠️ Infrastructure Operations & Lifecycle

All system settings run isolated within `~/n8n`.

### Quick Diagnostics Commands

```bash
cd ~/n8n

# View container operational parameters
docker ps

# Stream active logs in real-time
docker compose logs -f

# Perform a safe configuration restart
docker compose down && docker compose up -d

```

### Upgrading the Runtime Layout Engine

```bash
cd ~/n8n
docker compose pull
docker compose up -d --force-recreate

```

> 💾 **Data Preservation Note:** Because database engines and runtime workflows are mounted independently inside the named volume `n8n_data`, executing full container image builds or tearing down environments will never result in data loss.

---

## 📁 Repository Directory Matrix

```text
.
├── docker-compose.yml       # Active service stack orchestration configuration
├── README.md                # System documentation asset
└── workflows/               # Managed JSON structural pipeline tracking exports
    ├── lead-generation/     # Scheduled lead ingestion and cleaning schemas
    └── gtm-marketing/       # Webhook targets and cross-platform notification trees

```
