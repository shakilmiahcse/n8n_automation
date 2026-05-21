# Production-Ready n8n Development Environment with Docker

This repository contains a containerized, production-ready configuration for deploying **n8n** with a **PostgreSQL** database and an **Nginx** reverse proxy. The architecture is designed to run seamlessly on a local Windows 10 development machine (via Docker Desktop) and easily migrate to a **Red Hat Enterprise Linux (RHEL)** production server.

---

## 🛠️ Repository Directory Structure

```text
n8n_automation/
├── docker-compose.yml         # Multi-container service definitions
├── .env                       # Active environment configurations (git-ignored)
├── .env.example               # Template for setting up environment variables
├── nginx/
│   ├── default.conf           # Reverse proxy, WebSocket, and SSL-ready config
│   └── certs/                 # Mapped folder for Let's Encrypt certificates (SSL-ready)
├── n8n_data/                  # Persistent n8n configuration and workflow database files
└── postgres_data/             # Persistent PostgreSQL database cluster files
```

---

## ⚙️ Architecture & Service Overview

The environment isolates components to guarantee high availability, performance, and security.

```mermaid
graph TD
    User([User / Webhook Trigger]) -->|Port 80/443| Nginx[Nginx Reverse Proxy]
    subgraph Private Docker Network (n8n-network)
        Nginx -->|Proxy Pass / WebSocket| n8n[n8n App Engine]
        n8n -->|Internal Connection| Postgres[(PostgreSQL DB)]
    end
    
    subgraph Host Storage
        n8n -.->|Bind Mount| n8n_data[./n8n_data]
        Postgres -.->|Bind Mount| postgres_data[./postgres_data]
        Nginx -.->|Read-Only Certs| certs[./nginx/certs]
    end
```

### 1. How Each Service Works
*   **PostgreSQL (`postgres:16-alpine`):** Handles all persistence. n8n stores users, credentials, workflow JSON structures, execution histories, variables, and audit logs here. The alpine base ensures a minimal footprint.
*   **n8n (`docker.n8n.io/n8nio/n8n:latest`):** The workflow orchestration engine. It runs the node processes, handles execution queues, evaluates logic, and provides the web-based visual editor.
*   **Nginx (`nginx:1.25-alpine`):** A high-performance reverse proxy that sits in front of the application. It acts as the gatekeeper, terminates incoming HTTP/HTTPS requests, handles WebSocket upgrades required by the n8n UI, buffers request payloads, and prevents direct internet access to your backend services.

### 2. How Containers Communicate
*   All three services are attached to a private user-defined bridge network called `n8n-network`.
*   Docker provides an internal DNS resolver. Containers communicate using their service names defined in `docker-compose.yml` (e.g., n8n resolves the database host as `postgres` instead of using IP addresses).
*   **Port Isolation:** Only Nginx exposes ports to the host machine (`80` and `443`). The database (`5432`) and n8n (`5678`) do not publish ports to the host, protecting them from unauthorized external access.

### 3. How Webhook URLs Work
*   When trigger nodes (e.g., Webhook, Stripe, GitHub, Slack) are active, n8n registers a listening path (e.g., `/webhook/some-uuid`).
*   n8n prefixes this path with the `WEBHOOK_URL` environment variable defined in your `.env` file to generate the callback URL.
*   When an external service sends an HTTP request to this generated URL, the request hits Nginx (acting on port 80/443). Nginx proxy-passes it to n8n at `http://n8n:5678/webhook/...`.
*   **Crucial Rule:** The `WEBHOOK_URL` environment variable must always match the external domain name and protocol (HTTP/HTTPS) and include a trailing slash (e.g., `https://n8n.yourdomain.com/`).

---

## 💻 Windows Local Development Setup

### Prerequisites
1. Install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/).
2. Ensure Docker Desktop is configured to use the **WSL 2 backend** (recommended).

### Step-by-Step Launch
1. Clone this directory to your Windows machine.
2. In the project root, copy `.env.example` to `.env`:
   ```powershell
   copy .env.example .env
   ```
3. Set the required variables in `.env`. The defaults are pre-configured to work locally out of the box using `http://localhost/` via Nginx.
4. Launch the services in detached mode:
   ```powershell
   docker compose up -d
   ```
5. Verify container health:
   ```powershell
   docker compose ps
   ```
6. Access the n8n dashboard by opening your browser and navigating to **`http://localhost`**.

---

## 🐧 Red Hat Enterprise Linux (RHEL) Deployment

Moving from local Windows development to RHEL requires specific adjustments for production security, firewalls, and SELinux.

### Prerequisites
Install Docker Engine and Docker Compose on RHEL:
```bash
sudo dnf install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
```

### 1. Folder Permissions & Owner Mapping
RHEL enforces strict POSIX permissions. Containers run under low-privilege system users (`node` with UID `1000` for n8n, and `postgres` with UID `999` for PostgreSQL).
On your host, run:
```bash
sudo chown -R 1000:1000 ./n8n_data
sudo chown -R 999:999 ./postgres_data
```

### 2. SELinux Policy Adjustment
By default, SELinux will block the containers from reading or writing files in host bind-mounted directories. You must apply the container-shared security context (`z` or `Z` label, or apply it recursively on the host):
```bash
sudo chcon -Rt container_file_t ./n8n_data ./postgres_data ./nginx
```

### 3. Open Firewall Ports
Allow traffic through the system firewall (`firewalld`):
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### 4. SSL Integration using Let's Encrypt (Certbot)
To run in production, secure your instance with an SSL certificate.

1. Install Certbot on the host:
   ```bash
   sudo dnf install -y epel-release
   sudo dnf install -y certbot
   ```
2. Request a certificate (ensure port 80 is temporarily free, or configure a webroot challenge):
   ```bash
   sudo certbot certonly --standalone -d n8n.yourdomain.com
   ```
3. Link or copy the certificates to your `./nginx/certs` directory:
   ```bash
   # Create folders if not present
   mkdir -p ./nginx/certs
   # Bind mount links certificates. Let's Encrypt certificates are located in:
   # /etc/letsencrypt/live/n8n.yourdomain.com/
   ```
4. Modify your `.env` configuration:
   ```env
   DOMAIN_NAME=n8n.yourdomain.com
   N8N_PROTOCOL=https
   WEBHOOK_URL=https://n8n.yourdomain.com/
   ```
5. Modify `nginx/default.conf`:
   * Uncomment the HTTPS server block (listening on port 443).
   * Update `ssl_certificate` and `ssl_certificate_key` to point to `/etc/letsencrypt/live/n8n.yourdomain.com/...` (which maps to your host certs directory).
   * Uncomment the redirect block in the HTTP server block (port 80) to force HTTPS.
6. Re-launch containers:
   ```bash
   docker compose down
   docker compose up -d
   ```

---

## 💾 Backup and Restore Procedures

Protect your workflows, execution data, and secrets using these standard database and file commands.

### 1. Database Backups (PostgreSQL)
Run these commands from the project root directory.

*   **Backup database (Compressed Format):**
    ```bash
    # Windows (PowerShell)
    docker exec -t n8n-postgres pg_dump -U n8n_db_user -d n8n_database -F c -b -v -f /var/lib/postgresql/data/db_backup.dump
    
    # RHEL (Linux)
    docker exec -t n8n-postgres pg_dump -U n8n_db_user -d n8n_database -F c -b -v -f /var/lib/postgresql/data/db_backup.dump
    ```
    This creates a binary file `/var/lib/postgresql/data/db_backup.dump` which translates on the host to `./postgres_data/db_backup.dump`.

*   **Restore database:**
    ```bash
    # Drop and recreate schema inside container, then restore:
    docker exec -it n8n-postgres pg_restore -U n8n_db_user -d n8n_database --clean --verbose /var/lib/postgresql/data/db_backup.dump
    ```

### 2. Application Files Backup (n8n Encryption Keys & Files)
n8n stores its encryption keys and workflow definitions (if configured via local JSON) inside `./n8n_data`.
*   **Create backup archive:**
    ```bash
    # Windows (PowerShell)
    Compress-Archive -Path ./n8n_data -DestinationPath ./n8n_data_backup.zip
    
    # Linux (RHEL)
    tar -czvf n8n_data_backup.tar.gz ./n8n_data
    ```
    *Keep `N8N_ENCRYPTION_KEY` from `.env` safe. Without this key, you cannot decrypt credentials imported into a restored n8n instance.*

---

## 🔄 Docker Update & Maintenance

Keep your services up to date with the latest security patches.

```bash
# 1. Pull the latest images defined in docker-compose.yml
docker compose pull

# 2. Re-create and restart the containers with updated images
docker compose up -d --remove-orphans

# 3. Clean up older, unused dangling Docker images
docker image prune -f
```

---

## 🔍 Troubleshooting Guide

### 1. Nginx Port Conflict
*   **Symptom:** `Bind for 0.0.0.0:80 failed: port is already allocated` or container restarts repeatedly.
*   **Solution:** Another web server (e.g. XAMPP Apache, IIS) is using port 80. Open your `.env` file and change `HOST_HTTP_PORT` to an alternative port (e.g., `HOST_HTTP_PORT=8080`). Access n8n at `http://localhost:8080`.

### 2. Permission Denied on n8n_data/postgres_data (Linux/RHEL)
*   **Symptom:** `n8n-app` logs show `Error: EACCES: permission denied, open '/home/node/.n8n/config'` or PostgreSQL logs show `chmod /var/lib/postgresql/data failed: Permission denied`.
*   **Solution:** Ensure correct host-level permissions and SELinux contexts (see *RHEL Deployment* section). Run:
    ```bash
    sudo chown -R 1000:1000 ./n8n_data
    sudo chown -R 999:999 ./postgres_data
    sudo chcon -Rt container_file_t ./n8n_data ./postgres_data
    ```

### 3. Webhook Nodes Fail / Timeout
*   **Symptom:** External services fail to trigger workflows, or webhook executions show "Connection Refused".
*   **Solution:** Check if `WEBHOOK_URL` in `.env` matches your external URL exactly. When testing locally, external services cannot trigger `http://localhost/` webhooks unless you use a tunneling service (e.g., ngrok or Localtunnel) and set `WEBHOOK_URL` to the tunnel URL.

### 4. Database Connection Refused
*   **Symptom:** `n8n-app` container logs show `Postgres connection failed: Dial tcp 172.20.0.X:5432 connect: connection refused`.
*   **Solution:** Verify that the Postgres service is healthy. Check health status using `docker compose ps`. Check postgres logs: `docker compose logs postgres`. Ensure that `POSTGRES_HOST` in `.env` is set to `postgres` (matching the service name in `docker-compose.yml`).
