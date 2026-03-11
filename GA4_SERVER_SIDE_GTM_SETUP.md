# Beast Insights — GA4 + Server-Side GTM Full Setup & Event Tracking Plan

> **Date**: 2026-03-10
> **Platform**: Beast Insights (Multi-tenant Payment Analytics)
> **Stack**: Next.js 16 + React 19 frontend, Express.js backend
> **Current State**: GTM container `GTM-T3C33TL5` (client-side) + Microsoft Clarity active
> **Goal**: Full GA4 behavioral tracking via server-side GTM on self-hosted Linux

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [GA4 Property Setup](#2-ga4-property-setup)
3. [Linux Server Setup for sGTM](#3-linux-server-setup-for-sgtm)
4. [Server-Side GTM Container Setup](#4-server-side-gtm-container-setup)
5. [Web GTM Container Configuration](#5-web-gtm-container-configuration)
6. [Frontend Implementation (Next.js)](#6-frontend-implementation-nextjs)
7. [Custom Dimensions & Metrics in GA4](#7-custom-dimensions--metrics-in-ga4)
8. [Event Tracking Sheets](#8-event-tracking-sheets)
9. [Testing & Validation](#9-testing--validation)
10. [Maintenance & Monitoring](#10-maintenance--monitoring)

---

## 1. Architecture Overview

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────┐
│   User Browser   │────▶│  Web GTM (client) │────▶│  sGTM Server │
│  (Next.js App)   │     │  GTM-T3C33TL5    │     │  (your Linux)│
│                  │     │                  │     │              │
│  gtag / dataLayer│     │  GA4 Config tag   │     │  GA4 tag     │
│  push events     │     │  (transport_url   │     │  (forwards   │
│                  │     │   → sGTM server)  │     │   to Google)  │
└─────────────────┘     └──────────────────┘     └──────┬──────┘
                                                         │
                                                         ▼
                                                  ┌─────────────┐
                                                  │  Google GA4   │
                                                  │  (analytics   │
                                                  │   .google.com)│
                                                  └─────────────┘
```

**Why server-side GTM?**
- First-party data collection (bypass ad blockers)
- Full control over data before it reaches Google
- Better data accuracy (server-managed cookies)
- Privacy compliance (PII scrubbing on your server)
- Lower client-side JS payload

---

## 2. GA4 Property Setup

### 2.1 Create GA4 Property

1. Go to [Google Analytics](https://analytics.google.com/) → **Admin** (gear icon)
2. Click **+ Create Property**
3. Fill in:
   - **Property name**: `Beast Insights - Production`
   - **Reporting time zone**: Your primary timezone
   - **Currency**: USD
4. Click **Next** → Select **Business** industry, **Medium** size
5. Select objectives: **Examine user behavior**, **Generate leads**
6. Click **Create**

### 2.2 Create Data Stream

1. In your new property → **Admin** → **Data Streams**
2. Click **Add stream** → **Web**
3. Fill in:
   - **Website URL**: `app.beastinsights.com` (your production domain)
   - **Stream name**: `Beast Insights Web App`
4. **Disable** Enhanced Measurement (we'll track everything custom)
5. Click **Create stream**
6. Copy the **Measurement ID** (format: `G-XXXXXXXXXX`) — you'll need this

### 2.3 Create API Secret (for sGTM)

1. In the data stream settings → **Measurement Protocol API secrets**
2. Click **Create** → Nickname: `sGTM Server`
3. Copy the **API secret** — save it securely

### 2.4 Data Retention Settings

1. **Admin** → **Data Settings** → **Data Retention**
2. Set **Event data retention** to **14 months** (max)
3. Toggle ON **Reset user data on new activity**

### 2.5 Disable Default Enhanced Measurement

1. **Admin** → **Data Streams** → Click your stream
2. Click the **gear icon** next to Enhanced measurement
3. **Turn OFF all** toggles (page views, scrolls, outbound clicks, site search, video, file downloads, form interactions)
4. We'll track all of these with custom events for better control

---

## 3. Linux Server Setup for sGTM

> **IMPORTANT — Shared Server Safety**
> This server already runs other services (n8n, Jenkins, etc.). Every step below is
> **additive only** — no existing configs are overwritten, no packages are force-upgraded,
> and no firewall rules are reset. Each step includes a pre-check so you can skip
> what's already in place.

### 3.1 Additional Resource Requirements (sGTM only)

sGTM is lightweight. On top of what your server already uses:

| Resource | sGTM Addition |
|----------|---------------|
| CPU | ~0.1-0.3 vCPU (idle), bursts to ~0.5 under load |
| RAM | ~150-300 MB |
| Disk | ~500 MB (Docker image + logs) |
| Port | 8079 (localhost only, proxied via Nginx) |

### 3.2 DNS Setup

Add a subdomain pointing to your **existing server IP** (no new server needed):

```
Type: A
Name: analytics.beastinsights.com
Value: <YOUR_EXISTING_SERVER_IP>
TTL: 300
```

### 3.3 Pre-Flight Checks

Before doing anything, audit what's already running so nothing conflicts:

```bash
# 1. Check existing Docker containers and their ports
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"

# 2. Check existing Nginx server blocks
ls -la /etc/nginx/sites-enabled/

# 3. Check which ports are in use (avoid picking a conflicting port)
#    Jenkins runs on host :8080 — sGTM will use host :8079 (no conflict)
sudo ss -tlnp | grep -E ':(80|443|8079|8080)\b'

# 4. Check existing firewall rules (DO NOT change them)
sudo ufw status verbose

# 5. Check Docker and Nginx are already installed
docker --version
nginx -v

# 6. Check disk space
df -h /
```

> **If port 8079 is already in use**, pick a different one (e.g., 8089) and substitute
> it everywhere below. Same for 8080 inside the container — but 8080 is the container's
> internal port and won't conflict with host ports.

### 3.4 Install Docker (Skip if Already Installed)

```bash
# Check first — if this outputs a version, skip this entire step
docker --version

# Only run if Docker is NOT installed:
sudo apt update   # safe — only updates package index, does NOT upgrade anything
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
docker --version
```

> **DO NOT run `sudo apt upgrade -y`** — this can upgrade packages that n8n,
> Jenkins, or other services depend on and break them.

### 3.5 Install Nginx (Skip if Already Installed)

```bash
# Check first
nginx -v

# Only run if Nginx is NOT installed:
sudo apt install -y nginx
```

### 3.6 SSL Certificate for the sGTM Subdomain Only

```bash
# Check if certbot is installed
certbot --version

# Only install if missing:
sudo apt install -y certbot python3-certbot-nginx

# Get certificate ONLY for the sGTM subdomain
# The --certonly flag gets the cert WITHOUT modifying any existing Nginx config
sudo certbot certonly --nginx -d analytics.beastinsights.com

# Verify the cert was created
sudo ls /etc/letsencrypt/live/analytics.beastinsights.com/

# Auto-renewal should already be set up. Verify:
sudo certbot renew --dry-run
```

> **Key difference**: We use `certonly` instead of just `--nginx`. This obtains the
> certificate **without** auto-editing your Nginx configs. Your existing n8n/Jenkins
> server blocks remain untouched.

### 3.7 Add Nginx Server Block (Additive — Does NOT Touch Existing Configs)

Create a **new, separate** server block file for sGTM. This does NOT modify any
existing files in `sites-available/` or `sites-enabled/`:

```bash
# Verify this file doesn't already exist
ls /etc/nginx/sites-available/sgtm 2>/dev/null && echo "EXISTS — edit instead of creating" || echo "OK to create"

sudo nano /etc/nginx/sites-available/sgtm
```

Paste this config — it **only** matches the `analytics.beastinsights.com` subdomain
and will never intercept traffic for your other services:

```nginx
# ─── sGTM Reverse Proxy ───────────────────────────────────────
# This block ONLY handles analytics.beastinsights.com
# It does NOT affect any other server_name blocks (n8n, Jenkins, etc.)

server {
    listen 443 ssl http2;
    server_name analytics.beastinsights.com;   # <-- ONLY this subdomain

    ssl_certificate /etc/letsencrypt/live/analytics.beastinsights.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/analytics.beastinsights.com/privkey.pem;

    # Security headers (scoped to this server block only)
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:8079;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_http_version 1.1;
    }
}

server {
    listen 80;
    server_name analytics.beastinsights.com;   # <-- ONLY this subdomain
    return 301 https://$host$request_uri;
}
```

Enable and validate **without restarting** (reload is graceful and non-disruptive):

```bash
# Symlink to enable
sudo ln -s /etc/nginx/sites-available/sgtm /etc/nginx/sites-enabled/

# TEST config BEFORE applying — this catches syntax errors without affecting running Nginx
sudo nginx -t

# Only if nginx -t says "syntax is ok" and "test is successful":
sudo systemctl reload nginx    # <-- reload, NOT restart (zero downtime for existing services)
```

> **Why `reload` not `restart`?** `reload` gracefully applies new config without
> dropping existing connections. Your n8n webhooks, Jenkins builds, etc. continue
> uninterrupted.

### 3.8 Deploy sGTM Docker Container

The sGTM container binds to **localhost:8079** only (proxied by Nginx). It does
not interfere with other Docker containers or host ports.

```bash
# Double-check port 8079 is free on the host
sudo ss -tlnp | grep :8079 && echo "PORT IN USE — pick another" || echo "Port 8079 is free"

# Verify Jenkins is on 8080 (we must NOT use 8080 on the host)
sudo ss -tlnp | grep :8080
# Expected: Jenkins listening on :8080 — this is fine.
# sGTM uses 8080 INSIDE its container only, mapped to host port 8079.
# Docker container ports are fully isolated — no conflict with Jenkins.
```

#### Option A: Docker Compose (Recommended)

Create an isolated directory for sGTM:

```bash
mkdir -p ~/sgtm
```

Create `~/sgtm/docker-compose.yml`:

```yaml
version: '3.8'

services:
  sgtm:
    image: gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable
    container_name: sgtm
    restart: unless-stopped
    ports:
      - "127.0.0.1:8079:8080"    # Maps container's internal 8080 → host 8079
                                  # Jenkins runs on host 8080 — NO conflict because
                                  # Docker container ports are isolated from the host.
                                  # External traffic: Nginx (443) → localhost:8079 → container:8080
    environment:
      - CONTAINER_CONFIG=${CONTAINER_CONFIG}
      - PREVIEW_SERVER_URL=https://analytics.beastinsights.com
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
    # Resource limits to protect other services on this shared server
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 128M
```

Create `~/sgtm/.env`:

```bash
CONTAINER_CONFIG=aWQ9R1RNLVQzNjhCMkdSJmVudj0xJmF1dGg9bDdjMEJVdWJlcUo2ZzhjTEcxRmpQUQ==
```

> **Note**: `CONTAINER_CONFIG` value comes from step 4 below. You'll get this from
> the GTM server container provisioning screen.

Run:

```bash
cd ~/sgtm
docker compose up -d

# Verify it's running and healthy
docker compose ps
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8079/healthz
# Should return: 200
```

#### Option B: Standalone Docker Run (if not using Compose)

```bash
docker pull gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable

docker run -d \
  --name sgtm \
  --restart unless-stopped \
  -p 127.0.0.1:8079:8080 \    # host 8079 ← container 8080 (Jenkins on host 8080 is unaffected)
  --memory=512m \
  --cpus=1.0 \
  -e CONTAINER_CONFIG="aWQ9R1RNLVQzNjhCMkdSJmVudj0xJmF1dGg9bDdjMEJVdWJlcUo2ZzhjTEcxRmpQUQ==" \
  -e PREVIEW_SERVER_URL="https://analytics.beastinsights.com" \
  gcr.io/cloud-tagging-10302018/gtm-cloud-image:stable
```

### 3.9 Firewall — Safe Setup From Inactive State

> **SITUATION**: UFW is currently **inactive** (all ports open). We want to enable it
> without locking out Jenkins, n8n, or any other running service.
>
> **RULE**: Whitelist every port in use FIRST, then enable UFW. If you enable UFW
> before whitelisting, you'll be locked out of SSH and all services instantly.

#### Step 1: Discover ALL ports currently in use

```bash
# List every service listening on a port (run this and save the output!)
sudo ss -tlnp | awk 'NR>1 {print $4}' | sed 's/.*://' | sort -un

# More detailed view — shows service name + port
sudo ss -tlnp
```

Example output you might see:

```
22      ← SSH
80      ← Nginx HTTP
443     ← Nginx HTTPS
3000    ← n8n (or other app)
5678    ← n8n webhook
8080    ← Jenkins
8079    ← sGTM (after you deploy it)
9090    ← Prometheus / other monitoring
...
```

**Write down every port number from the output. You need ALL of them.**

#### Step 2: Allow SSH FIRST (critical — do this before anything else)

```bash
# ALWAYS allow SSH first — if you skip this and enable UFW, you're locked out
sudo ufw allow 22/tcp comment "SSH"
```

#### Step 3: Allow all your existing service ports

```bash
# Core web ports (Nginx handles sGTM + other web services)
sudo ufw allow 80/tcp comment "HTTP - Nginx"
sudo ufw allow 443/tcp comment "HTTPS - Nginx"

# Jenkins
sudo ufw allow 8080/tcp comment "Jenkins"

# n8n — adjust port numbers to match YOUR actual n8n setup
# Common n8n ports: 5678 (main UI), sometimes 443 if behind Nginx
sudo ufw allow 5678/tcp comment "n8n"

# ──────────────────────────────────────────────────────
# ADD EVERY OTHER PORT from Step 1 output here!
# Template:
#   sudo ufw allow <PORT>/tcp comment "<SERVICE_NAME>"
#
# Examples (uncomment/adjust as needed):
#   sudo ufw allow 3000/tcp comment "n8n or other app"
#   sudo ufw allow 9090/tcp comment "Prometheus"
#   sudo ufw allow 5432/tcp comment "PostgreSQL"
#   sudo ufw allow 6379/tcp comment "Redis"
#   sudo ufw allow 9000/tcp comment "Portainer"
#   sudo ufw allow 50000/tcp comment "Jenkins agent"
# ──────────────────────────────────────────────────────
```

> **Note**: sGTM on port 8079 does NOT need a firewall rule — it's bound to
> `127.0.0.1` (localhost only). External traffic reaches it through Nginx on 443.

#### Step 4: Review rules BEFORE enabling

```bash
# This shows what WILL be applied — review carefully
sudo ufw show added

# Cross-check against your running services
sudo ss -tlnp | awk 'NR>1 {print $4}' | sed 's/.*://' | sort -un
```

**Compare both lists. Every port from `ss` must have a matching `ufw allow` rule.**

#### Step 5: Set default policy and enable

```bash
# Default: deny incoming, allow outgoing (standard secure config)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable UFW — type 'y' when prompted
sudo ufw enable

# Verify it's active and rules are correct
sudo ufw status verbose
```

#### Step 6: Immediately verify nothing is broken

```bash
# Test from ANOTHER terminal/SSH session BEFORE closing this one!
# If you can't SSH in from the new session, you still have this session to fix it.

# Check all services are reachable:
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080    # Jenkins → 200
curl -s -o /dev/null -w "%{http_code}" http://localhost:5678    # n8n → 200 (adjust port)
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8079/healthz  # sGTM → 200
curl -s -o /dev/null -w "%{http_code}" https://analytics.beastinsights.com/healthz  # sGTM via Nginx → 200
```

#### Emergency: If you get locked out

If you accidentally enable UFW without allowing SSH:

- **Cloud provider console**: Use the VPS provider's web console (not SSH) to log in
- **Run**: `sudo ufw disable` to immediately turn off the firewall
- **Then**: Go back to Step 2 and add the missing rules

#### Adding new services later

Whenever you add a new service in the future:

```bash
# Add the rule BEFORE starting the service (or while UFW is active):
sudo ufw allow <PORT>/tcp comment "<SERVICE_NAME>"
sudo ufw status numbered
```

### 3.10 Auto-Update Script (sGTM Only)

Create `~/sgtm/update.sh`:

```bash
#!/bin/bash
# Updates ONLY the sGTM container — does not touch any other Docker containers
set -e

cd ~/sgtm
echo "[$(date)] Starting sGTM update..."

# Pull new image
docker compose pull sgtm

# Recreate only the sgtm service (other containers unaffected)
docker compose up -d --no-deps sgtm

# Clean up only dangling images (safe — won't remove images used by other containers)
docker image prune -f --filter "dangling=true"

echo "[$(date)] sGTM update complete"
```

Add to cron for weekly updates:

```bash
chmod +x ~/sgtm/update.sh

# Add to existing crontab (appends, does not overwrite)
(crontab -l 2>/dev/null; echo "0 3 * * 0 /home/$(whoami)/sgtm/update.sh >> /home/$(whoami)/sgtm/update.log 2>&1") | crontab -
```

### 3.11 Verify Nothing Else Was Affected

After completing all steps, run this checklist:

```bash
# 1. All existing Docker containers still running?
docker ps --format "table {{.Names}}\t{{.Status}}"

# 2. Nginx serving all existing sites?
sudo nginx -t
curl -s -o /dev/null -w "%{http_code}" http://localhost     # or your n8n/Jenkins URLs

# 3. sGTM healthy?
curl -s -o /dev/null -w "%{http_code}" https://analytics.beastinsights.com/healthz

# 4. Firewall unchanged (same rules as before + any additions)?
sudo ufw status numbered

# 5. No unexpected port conflicts?
sudo ss -tlnp | grep -E ':(80|443|8079)\b'
```

---

## 4. Server-Side GTM Container Setup

### 4.1 Create Server Container in GTM

1. Go to [Google Tag Manager](https://tagmanager.google.com/)
2. Click **Create Container**
3. Fill in:
   - **Container name**: `Beast Insights - Server`
   - **Target platform**: **Server**
4. Click **Create**
5. In the setup modal, select **Manually provision tagging server**
6. Copy the **Container Config** string — this goes in your Docker `CONTAINER_CONFIG` env var
7. Update your Docker container/compose with this value and restart

### 4.2 Create GA4 Client (in Server Container)

1. In your **Server** container → **Clients** (left menu)
2. The default **GA4** client should already exist
3. Verify it's enabled — it parses incoming GA4 requests automatically
4. Settings:
   - **Default GA4 paths**: ON
   - **Compress server response**: ON

### 4.3 Create GA4 Tag (in Server Container)

1. **Tags** → **New** → **GA4**
2. Configuration:
   - **Tag Type**: Google Analytics: GA4
   - **Measurement ID**: `G-XXXXXXXXXX` (your GA4 measurement ID)
   - **Send to**: Google Analytics server endpoint
3. **Triggering**: All Pages + All Custom Events (create 2 triggers)
4. **Advanced Settings** → Tag firing priority: 1

### 4.4 Create Event Data Variables (Server Container)

Create these built-in variables if not already enabled:

| Variable Name | Type | Purpose |
|--------------|------|---------|
| `Event Name` | Event Data | GA4 event name |
| `Client ID` | Event Data | GA4 client_id |
| `Page Location` | Event Data | Full URL |
| `Page Title` | Event Data | Document title |
| `Page Referrer` | Event Data | Referrer URL |

Create custom event data variables for your custom params:

| Variable Name | Event Data Key | Purpose |
|--------------|----------------|---------|
| `x-bi-client_id` | `client_id` | Beast Insights client/tenant ID |
| `x-bi-user_role` | `user_role` | User role |
| `x-bi-report_key` | `report_key` | Current report |
| `x-bi-visual_type` | `visual_type` | Visual type |

### 4.5 (Optional) PII Scrubbing Transformation

1. **Transformations** → **New**
2. **Name**: `Strip PII`
3. **Type**: Custom transformation
4. **Rule**: If event parameter contains email/password patterns → remove
5. This runs before data leaves your server to Google

### 4.6 Publish Server Container

1. Click **Submit** (top right)
2. **Version Name**: `v1.0 - GA4 forwarding`
3. Click **Publish**

---

## 5. Web GTM Container Configuration

> You already have `GTM-T3C33TL5`. We'll reconfigure it to route through sGTM.

### 5.1 Update GA4 Configuration Tag

1. Open your **Web** container `GTM-T3C33TL5` in GTM
2. Find your existing GA4 Configuration tag (or create one):
   - **Tag Type**: Google tag (gtag.js)
   - **Tag ID**: `G-XXXXXXXXXX`
3. **Configuration Settings** → Add these parameters:

| Parameter | Value |
|-----------|-------|
| `server_container_url` | `https://analytics.beastinsights.com` |
| `first_party_collection` | `true` |

4. **Triggering**: All Pages (Initialization trigger preferred)

### 5.2 Create Custom Event Tags

For each event category, create a GA4 Event tag. Below is the pattern:

#### Example: `report_viewed` Event Tag

1. **Tags** → **New**
2. **Tag Type**: Google Analytics: GA4 Event
3. **Configuration Tag**: Select your GA4 Config tag
4. **Event Name**: `{{Event Name from DL}}` (or specific name)
5. **Event Parameters**: Map from dataLayer variables
6. **Triggering**: Custom Event trigger where `Event` equals `report_viewed`

### 5.3 Create DataLayer Variables

Create these variables in Web GTM (type: Data Layer Variable):

| Variable Name | Data Layer Variable Name | Purpose |
|--------------|------------------------|---------|
| `DL - client_id` | `client_id` | Tenant ID |
| `DL - user_role` | `user_role` | User role |
| `DL - user_id` | `user_id` | User identifier |
| `DL - report_key` | `report_key` | Report template key |
| `DL - report_name` | `report_name` | Report display name |
| `DL - report_type` | `report_type` | stock / custom / templatized |
| `DL - visual_type` | `visual_type` | kpi / table / chart / matrix |
| `DL - visual_id` | `visual_id` | Visual identifier |
| `DL - filter_type` | `filter_type` | date / dimension / toggle |
| `DL - filter_key` | `filter_key` | Filter dimension key |
| `DL - filter_value` | `filter_value` | Filter value(s) applied |
| `DL - action` | `action` | Sub-action (sort, resize, etc.) |
| `DL - element_text` | `element_text` | Button/link text clicked |
| `DL - section` | `section` | Page section |
| `DL - duration_ms` | `duration_ms` | Time spent (ms) |

### 5.4 Create Triggers (Web GTM)

#### Custom Event Triggers

| Trigger Name | Type | Fires When |
|-------------|------|------------|
| `CE - report_viewed` | Custom Event | Event = `report_viewed` |
| `CE - filter_changed` | Custom Event | Event = `filter_changed` |
| `CE - visual_interacted` | Custom Event | Event = `visual_interacted` |
| `CE - builder_action` | Custom Event | Event = `builder_action` |
| `CE - navigation` | Custom Event | Event = `navigation` |
| `CE - auth_action` | Custom Event | Event = `auth_action` |
| `CE - table_interacted` | Custom Event | Event = `table_interacted` |
| `CE - chart_interacted` | Custom Event | Event = `chart_interacted` |
| `CE - kpi_interacted` | Custom Event | Event = `kpi_interacted` |
| `CE - bookmark_action` | Custom Event | Event = `bookmark_action` |
| `CE - export_action` | Custom Event | Event = `export_action` |
| `CE - error_occurred` | Custom Event | Event = `error_occurred` |
| `CE - engagement` | Custom Event | Event = `engagement` |
| `CE - admin_action` | Custom Event | Event = `admin_action` |

### 5.5 Create GA4 Event Tags (Web GTM)

Create one GA4 Event tag per trigger. Example for `report_viewed`:

**Tag Configuration:**
- **Tag Type**: Google Analytics: GA4 Event
- **Measurement ID**: `G-XXXXXXXXXX`
- **Event Name**: `report_viewed`
- **Event Parameters**:
  - `client_id` → `{{DL - client_id}}`
  - `user_role` → `{{DL - user_role}}`
  - `report_key` → `{{DL - report_key}}`
  - `report_name` → `{{DL - report_name}}`
  - `report_type` → `{{DL - report_type}}`
- **Trigger**: `CE - report_viewed`

Repeat this pattern for all events. Use the event tables in Section 8 for parameter mappings.

### 5.6 Publish Web Container

1. Click **Submit** → **Publish**
2. Version: `v2.0 - sGTM routing + custom events`

---

## 6. Frontend Implementation (Next.js)

### 6.1 Analytics Utility Module

Create `beastinsights/lib/analytics.ts`:

```typescript
/**
 * GA4 Analytics utility for Beast Insights.
 * Pushes structured events to GTM dataLayer.
 */

// ─── Types ───────────────────────────────────────────────────────

export interface AnalyticsEvent {
  event: string;
  [key: string]: string | number | boolean | undefined | null;
}

interface UserProperties {
  user_id: string;
  user_role: string;
  client_id: string;
  client_name?: string;
}

// ─── DataLayer Push ──────────────────────────────────────────────

declare global {
  interface Window {
    dataLayer: Record<string, unknown>[];
  }
}

function push(payload: AnalyticsEvent): void {
  if (typeof window === 'undefined') return;
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push(payload);
}

// ─── User Identity ──────────────────────────────────────────────

export function identifyUser(props: UserProperties): void {
  push({
    event: 'user_properties_set',
    user_id: props.user_id,
    user_role: props.user_role,
    client_id: props.client_id,
    client_name: props.client_name,
  });
}

// ─── Page / Navigation ──────────────────────────────────────────

export function trackPageView(params: {
  page_path: string;
  page_title: string;
  report_key?: string;
  report_name?: string;
  report_type?: string;
}): void {
  push({ event: 'page_view', ...params });
}

export function trackNavigation(params: {
  action: string;                        // sidebar_click, tab_switch, breadcrumb
  destination: string;                   // target route or report key
  section?: string;                      // sidebar section (analytics, routing, admin)
  element_text?: string;                 // link/button text
}): void {
  push({ event: 'navigation', ...params });
}

// ─── Report Events ──────────────────────────────────────────────

export function trackReportViewed(params: {
  report_key: string;
  report_name: string;
  report_type: string;                   // stock | custom | templatized
  visual_count: number;
  client_id: string;
}): void {
  push({ event: 'report_viewed', ...params });
}

export function trackReportTabSwitch(params: {
  report_key: string;
  from_tab: string;
  to_tab: string;
}): void {
  push({ event: 'report_tab_switch', ...params });
}

// ─── Filter Events ──────────────────────────────────────────────

export function trackFilterChanged(params: {
  report_key: string;
  filter_type: string;                   // date_range | dimension | toggle | parameter
  filter_key: string;                    // dimension key or toggle name
  filter_value?: string;                 // applied value(s) — avoid PII
  filter_count?: number;                 // number of selected values
}): void {
  push({ event: 'filter_changed', ...params });
}

export function trackFilterCleared(params: {
  report_key: string;
  clear_type: string;                    // single | all
  filter_key?: string;
}): void {
  push({ event: 'filter_cleared', ...params });
}

export function trackBookmarkAction(params: {
  action: string;                        // create | apply | delete | set_default
  report_key: string;
  bookmark_name?: string;
}): void {
  push({ event: 'bookmark_action', ...params });
}

// ─── Visual Interaction Events ──────────────────────────────────

export function trackVisualInteracted(params: {
  report_key: string;
  visual_id: string;
  visual_type: string;                   // kpi | table | chart | matrix
  action: string;                        // hover, click, expand, collapse, etc.
  detail?: string;                       // additional context
}): void {
  push({ event: 'visual_interacted', ...params });
}

// ─── Table Events ───────────────────────────────────────────────

export function trackTableInteraction(params: {
  report_key: string;
  visual_id: string;
  action: string;                        // sort | resize | paginate | scroll | group_by_change | column_toggle | column_reorder | row_expand | export
  column_key?: string;
  sort_direction?: string;
  page_number?: number;
  group_by?: string;
}): void {
  push({ event: 'table_interacted', ...params });
}

// ─── Chart Events ───────────────────────────────────────────────

export function trackChartInteraction(params: {
  report_key: string;
  visual_id: string;
  chart_type: string;                    // bar | line | area | combo | donut | trend
  action: string;                        // hover | legend_toggle | metric_switch | granularity_change | zoom | comparison_toggle | anomaly_toggle | insights_view
  metric_key?: string;
  granularity?: string;                  // day | week | month
}): void {
  push({ event: 'chart_interacted', ...params });
}

// ─── KPI Events ─────────────────────────────────────────────────

export function trackKPIInteraction(params: {
  report_key: string;
  visual_id: string;
  kpi_variant: string;                   // simple | sparkline | gauge | waterfall | comparison | distribution | breakdown | insights
  action: string;                        // hover | click | insight_navigate | insight_auto_rotate
  metric_key?: string;
}): void {
  push({ event: 'kpi_interacted', ...params });
}

// ─── Builder Events ─────────────────────────────────────────────

export function trackBuilderAction(params: {
  builder_type: string;                  // v1 | v2 | creator
  action: string;                        // open | add_visual | edit_visual | remove_visual | reorder | resize | configure_filters | save | cancel
  report_id?: string;
  visual_type?: string;
  visual_preset?: string;
  visual_count?: number;
}): void {
  push({ event: 'builder_action', ...params });
}

// ─── Auth Events ────────────────────────────────────────────────

export function trackAuthAction(params: {
  action: string;                        // login | logout | forgot_password | reset_password | onboard
  method?: string;
  success: boolean;
  error_message?: string;
}): void {
  push({ event: 'auth_action', ...params });
}

// ─── Client Switch ──────────────────────────────────────────────

export function trackClientSwitch(params: {
  from_client_id: string;
  to_client_id: string;
}): void {
  push({ event: 'client_switch', ...params });
}

// ─── Engagement / Time Tracking ─────────────────────────────────

export function trackEngagement(params: {
  type: string;                          // report_dwell | visual_dwell | session_duration | scroll_depth
  report_key?: string;
  visual_id?: string;
  duration_ms?: number;
  scroll_percent?: number;
}): void {
  push({ event: 'engagement', ...params });
}

// ─── Error Tracking ─────────────────────────────────────────────

export function trackError(params: {
  error_type: string;                    // api_error | render_error | auth_error | network_error
  error_message: string;
  page_path: string;
  report_key?: string;
  status_code?: number;
}): void {
  push({ event: 'error_occurred', ...params });
}

// ─── Export / Download ──────────────────────────────────────────

export function trackExport(params: {
  report_key: string;
  visual_id?: string;
  format: string;                        // csv | pdf | xlsx
  row_count?: number;
}): void {
  push({ event: 'export_action', ...params });
}

// ─── Admin Events ───────────────────────────────────────────────

export function trackAdminAction(params: {
  action: string;                        // user_create | user_edit | user_delete | maintenance_toggle | etl_view | filter_config_change
  target_entity?: string;
  target_id?: string;
}): void {
  push({ event: 'admin_action', ...params });
}

// ─── Search Events ──────────────────────────────────────────────

export function trackSearch(params: {
  search_location: string;               // filter_options | client_selector | column_selector | report_list | builder_presets
  search_term: string;                   // Do NOT log PII — truncate/hash if needed
  results_count?: number;
}): void {
  push({ event: 'search', ...params });
}

// ─── Sidebar Events ─────────────────────────────────────────────

export function trackSidebarAction(params: {
  action: string;                        // expand | collapse | category_expand | category_collapse
  category?: string;
}): void {
  push({ event: 'sidebar_action', ...params });
}

// ─── Theme Events ───────────────────────────────────────────────

export function trackThemeSwitch(params: {
  from_theme: string;
  to_theme: string;
}): void {
  push({ event: 'theme_switch', ...params });
}

// ─── Integration / Schedule Events ──────────────────────────────

export function trackIntegrationAction(params: {
  action: string;                        // connect | disconnect | oauth_start | oauth_complete | oauth_fail
  provider: string;
}): void {
  push({ event: 'integration_action', ...params });
}

export function trackScheduleAction(params: {
  action: string;                        // create | update | delete | toggle_active
  schedule_type: string;                 // email | webhook
  frequency?: string;
}): void {
  push({ event: 'schedule_action', ...params });
}
```

### 6.2 Report Dwell Time Hook

Create `beastinsights/hooks/useReportDwellTime.ts`:

```typescript
import { useEffect, useRef } from 'react';
import { trackEngagement } from '@/lib/analytics';

/**
 * Tracks time spent on a report page.
 * Fires engagement event when user leaves or after intervals.
 */
export function useReportDwellTime(reportKey: string | undefined) {
  const startRef = useRef<number>(Date.now());
  const reportKeyRef = useRef(reportKey);

  useEffect(() => {
    if (!reportKey) return;

    reportKeyRef.current = reportKey;
    startRef.current = Date.now();

    // Fire heartbeat every 30s for active tracking
    const heartbeat = setInterval(() => {
      trackEngagement({
        type: 'report_dwell',
        report_key: reportKeyRef.current,
        duration_ms: Date.now() - startRef.current,
      });
    }, 30_000);

    return () => {
      clearInterval(heartbeat);

      // Fire final dwell time on unmount
      const duration = Date.now() - startRef.current;
      if (duration > 1000) {
        trackEngagement({
          type: 'report_dwell',
          report_key: reportKeyRef.current,
          duration_ms: duration,
        });
      }
    };
  }, [reportKey]);
}
```

### 6.3 Scroll Depth Hook

Create `beastinsights/hooks/useScrollDepth.ts`:

```typescript
import { useEffect, useRef, useCallback } from 'react';
import { trackEngagement } from '@/lib/analytics';

/**
 * Tracks scroll depth milestones (25%, 50%, 75%, 100%).
 */
export function useScrollDepth(reportKey: string | undefined) {
  const milestonesRef = useRef<Set<number>>(new Set());

  const handleScroll = useCallback(() => {
    if (!reportKey) return;

    const scrollTop = window.scrollY;
    const docHeight = document.documentElement.scrollHeight - window.innerHeight;
    if (docHeight <= 0) return;

    const percent = Math.round((scrollTop / docHeight) * 100);

    for (const milestone of [25, 50, 75, 100]) {
      if (percent >= milestone && !milestonesRef.current.has(milestone)) {
        milestonesRef.current.add(milestone);
        trackEngagement({
          type: 'scroll_depth',
          report_key: reportKey,
          scroll_percent: milestone,
        });
      }
    }
  }, [reportKey]);

  useEffect(() => {
    milestonesRef.current.clear();
    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);
}
```

### 6.4 Visibility Tracking Hook

Create `beastinsights/hooks/useVisualVisibility.ts`:

```typescript
import { useEffect, useRef } from 'react';
import { trackVisualInteracted } from '@/lib/analytics';

/**
 * Tracks when a visual card enters the viewport (impression tracking).
 */
export function useVisualVisibility(
  ref: React.RefObject<HTMLElement | null>,
  params: { report_key: string; visual_id: string; visual_type: string }
) {
  const trackedRef = useRef(false);

  useEffect(() => {
    const el = ref.current;
    if (!el || trackedRef.current) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && !trackedRef.current) {
          trackedRef.current = true;
          trackVisualInteracted({
            ...params,
            action: 'impression',
          });
          observer.disconnect();
        }
      },
      { threshold: 0.5 }
    );

    observer.observe(el);
    return () => observer.disconnect();
  }, [ref, params]);
}
```

### 6.5 Integration Points in Existing Code

Below are the key files where you add tracking calls. Each is a **minimal** addition — typically 1-3 lines.

#### A. Report Page — `reports/[templateKey]/page.tsx`

```typescript
import { trackReportViewed, trackReportTabSwitch } from '@/lib/analytics';
import { useReportDwellTime } from '@/hooks/useReportDwellTime';
import { useScrollDepth } from '@/hooks/useScrollDepth';

// Inside component:
useReportDwellTime(templateKey);
useScrollDepth(templateKey);

// On report data load success:
trackReportViewed({
  report_key: templateKey,
  report_name: template.name,
  report_type: template.report_type,
  visual_count: template.sections.flatMap(s => s.visuals).length,
  client_id: clientId,
});

// On tab switch:
trackReportTabSwitch({
  report_key: templateKey,
  from_tab: prevTab,
  to_tab: newTab,
});
```

#### B. Slicer Slice — `store/slices/slicerSlice.ts`

Add a Redux listener middleware or track in component-level dispatches:

```typescript
// In the component dispatching the filter change:
import { trackFilterChanged, trackFilterCleared } from '@/lib/analytics';

// After dispatch(setDateRange(...)):
trackFilterChanged({
  report_key: activeReportKey,
  filter_type: 'date_range',
  filter_key: 'date_range',
  filter_value: `${startDate}_${endDate}`,
});

// After dispatch(setFilter(...)):
trackFilterChanged({
  report_key: activeReportKey,
  filter_type: 'dimension',
  filter_key: dimensionKey,
  filter_count: selectedValues.length,
});

// After dispatch(setToggle(...)):
trackFilterChanged({
  report_key: activeReportKey,
  filter_type: 'toggle',
  filter_key: toggleName,
  filter_value: toggleValue,
});
```

#### C. DataTable — `components/ui-tremor/tables/data-table.tsx`

```typescript
import { trackTableInteraction } from '@/lib/analytics';

// On column sort:
trackTableInteraction({ report_key, visual_id, action: 'sort', column_key, sort_direction });

// On column resize:
trackTableInteraction({ report_key, visual_id, action: 'resize', column_key });

// On page change:
trackTableInteraction({ report_key, visual_id, action: 'paginate', page_number });

// On group-by change:
trackTableInteraction({ report_key, visual_id, action: 'group_by_change', group_by });

// On column toggle:
trackTableInteraction({ report_key, visual_id, action: 'column_toggle', column_key });

// On column reorder:
trackTableInteraction({ report_key, visual_id, action: 'column_reorder' });

// On infinite scroll load:
trackTableInteraction({ report_key, visual_id, action: 'scroll_load', page_number });
```

#### D. Builder Components

```typescript
import { trackBuilderAction } from '@/lib/analytics';

// On builder open:
trackBuilderAction({ builder_type: 'v1', action: 'open', report_id });

// On add visual:
trackBuilderAction({ builder_type: 'v1', action: 'add_visual', visual_type, visual_preset });

// On remove visual:
trackBuilderAction({ builder_type: 'v1', action: 'remove_visual', visual_type });

// On save:
trackBuilderAction({ builder_type: 'v1', action: 'save', report_id, visual_count });
```

#### E. Auth — Login/Logout handlers

```typescript
import { trackAuthAction, identifyUser } from '@/lib/analytics';

// On login success:
trackAuthAction({ action: 'login', success: true });
identifyUser({ user_id, user_role, client_id, client_name });

// On login failure:
trackAuthAction({ action: 'login', success: false, error_message: err.message });

// On logout:
trackAuthAction({ action: 'logout', success: true });
```

#### F. Layout — `app/(user-type)/layout.js`

```typescript
import { trackClientSwitch } from '@/lib/analytics';

// In client selector onChange:
trackClientSwitch({ from_client_id: prevId, to_client_id: newId });
```

### 6.6 Update Root Layout GTM Script

Update `beastinsights/app/layout.tsx` to point to your sGTM:

```html
<!-- Replace the existing GTM snippet's gtm.js source with sGTM URL -->
<!-- Change: https://www.googletagmanager.com/gtm.js → https://analytics.beastinsights.com/gtm.js -->
```

The existing GTM container ID (`GTM-T3C33TL5`) stays the same. The sGTM acts as a proxy.

---

## 7. Custom Dimensions & Metrics in GA4

### 7.1 Register Custom Dimensions

Go to **GA4 Admin** → **Custom definitions** → **Custom dimensions** → **Create custom dimension**

| Display Name | Event Parameter | Scope | Description |
|-------------|----------------|-------|-------------|
| Client ID | `client_id` | Event | Tenant identifier |
| User Role | `user_role` | User | User permission level |
| Report Key | `report_key` | Event | Report template key |
| Report Name | `report_name` | Event | Report display name |
| Report Type | `report_type` | Event | stock / custom / templatized |
| Visual Type | `visual_type` | Event | kpi / table / chart / matrix |
| Visual ID | `visual_id` | Event | Individual visual identifier |
| Visual Preset | `visual_preset` | Event | Preset key used in builder |
| Filter Type | `filter_type` | Event | date_range / dimension / toggle / parameter |
| Filter Key | `filter_key` | Event | Dimension or toggle key |
| Filter Value | `filter_value` | Event | Applied filter value |
| Action | `action` | Event | Sub-action detail |
| Chart Type | `chart_type` | Event | bar / line / area / combo / donut / trend |
| Builder Type | `builder_type` | Event | v1 / v2 / creator |
| Error Type | `error_type` | Event | api / render / auth / network |
| Error Message | `error_message` | Event | Error description |
| Search Location | `search_location` | Event | Where search was performed |
| Search Term | `search_term` | Event | What was searched |
| Format | `format` | Event | Export format (csv/pdf/xlsx) |
| Section | `section` | Event | Page section context |
| Element Text | `element_text` | Event | Clicked element text |
| Provider | `provider` | Event | Integration provider |
| Granularity | `granularity` | Event | day / week / month |
| From Tab | `from_tab` | Event | Previous tab |
| To Tab | `to_tab` | Event | Destination tab |
| Destination | `destination` | Event | Navigation destination |
| KPI Variant | `kpi_variant` | Event | KPI card style |
| Category | `category` | Event | Sidebar/report category |
| Bookmark Name | `bookmark_name` | Event | Saved view name |

### 7.2 Register Custom Metrics

| Display Name | Event Parameter | Scope | Unit |
|-------------|----------------|-------|------|
| Duration (ms) | `duration_ms` | Event | Milliseconds |
| Scroll Percent | `scroll_percent` | Event | Standard |
| Filter Count | `filter_count` | Event | Standard |
| Visual Count | `visual_count` | Event | Standard |
| Results Count | `results_count` | Event | Standard |
| Row Count | `row_count` | Event | Standard |
| Page Number | `page_number` | Event | Standard |
| Status Code | `status_code` | Event | Standard |

---

## 8. Event Tracking Sheets

### 8.1 Page & Navigation Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 1 | `page_view` | auto | Page load / route change | `page_path`, `page_title`, `report_key`, `report_name`, `report_type` |
| 2 | `navigation` | sidebar_click | User clicks sidebar link | `destination`, `section`, `element_text` |
| 3 | `navigation` | tab_switch | Switch report tab | `report_key`, `from_tab`, `to_tab` |
| 4 | `navigation` | breadcrumb_click | Breadcrumb navigation | `destination`, `element_text` |
| 5 | `sidebar_action` | expand | Sidebar expanded | — |
| 6 | `sidebar_action` | collapse | Sidebar collapsed | — |
| 7 | `sidebar_action` | category_expand | Sidebar category opened | `category` |
| 8 | `sidebar_action` | category_collapse | Sidebar category closed | `category` |
| 9 | `theme_switch` | toggle | Theme changed | `from_theme`, `to_theme` |
| 10 | `client_switch` | select | Tenant changed | `from_client_id`, `to_client_id` |

### 8.2 Authentication Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 11 | `auth_action` | login | User logs in | `success`, `method`, `error_message` |
| 12 | `auth_action` | logout | User logs out | `success` |
| 13 | `auth_action` | forgot_password | Password reset requested | `success` |
| 14 | `auth_action` | reset_password | Password reset completed | `success` |
| 15 | `auth_action` | onboard | User onboarding completed | `success` |
| 16 | `user_properties_set` | identify | User identity set for session | `user_id`, `user_role`, `client_id`, `client_name` |

### 8.3 Report View Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 17 | `report_viewed` | load | Report page fully loaded | `report_key`, `report_name`, `report_type`, `visual_count`, `client_id` |
| 18 | `report_tab_switch` | switch | Switched between tabbed reports | `report_key`, `from_tab`, `to_tab` |

### 8.4 Filter & Slicer Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 19 | `filter_changed` | date_range | Date range changed | `report_key`, `filter_type: date_range`, `filter_key`, `filter_value` |
| 20 | `filter_changed` | date_preset | Date preset selected | `report_key`, `filter_type: date_range`, `filter_key`, `filter_value` (preset name) |
| 21 | `filter_changed` | dimension | Dimension filter applied | `report_key`, `filter_type: dimension`, `filter_key`, `filter_count` |
| 22 | `filter_changed` | toggle | Toggle value changed | `report_key`, `filter_type: toggle`, `filter_key`, `filter_value` |
| 23 | `filter_changed` | parameter | Parameter input adjusted | `report_key`, `filter_type: parameter`, `filter_key`, `filter_value` |
| 24 | `filter_changed` | more_filters | Advanced filter applied | `report_key`, `filter_type: dimension`, `filter_key`, `filter_count` |
| 25 | `filter_changed` | comparison | Comparison mode toggled | `report_key`, `filter_type: comparison`, `filter_value` (on/off) |
| 26 | `filter_cleared` | single | One filter removed | `report_key`, `clear_type: single`, `filter_key` |
| 27 | `filter_cleared` | all | All filters cleared | `report_key`, `clear_type: all` |
| 28 | `bookmark_action` | create | Saved filter view created | `report_key`, `bookmark_name` |
| 29 | `bookmark_action` | apply | Saved view applied | `report_key`, `bookmark_name` |
| 30 | `bookmark_action` | delete | Saved view deleted | `report_key`, `bookmark_name` |
| 31 | `bookmark_action` | set_default | Default bookmark set | `report_key`, `bookmark_name` |

### 8.5 Table Interaction Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 32 | `table_interacted` | sort | Column sorted | `report_key`, `visual_id`, `column_key`, `sort_direction` |
| 33 | `table_interacted` | resize | Column resized | `report_key`, `visual_id`, `column_key` |
| 34 | `table_interacted` | paginate | Page changed | `report_key`, `visual_id`, `page_number` |
| 35 | `table_interacted` | scroll_load | Infinite scroll triggered | `report_key`, `visual_id`, `page_number` |
| 36 | `table_interacted` | group_by_change | Group-by dimension changed | `report_key`, `visual_id`, `group_by` |
| 37 | `table_interacted` | column_toggle | Column shown/hidden | `report_key`, `visual_id`, `column_key` |
| 38 | `table_interacted` | column_reorder | Columns reordered (drag) | `report_key`, `visual_id` |
| 39 | `table_interacted` | row_expand | Row expanded for detail | `report_key`, `visual_id` |
| 40 | `table_interacted` | composite_toggle | Composite sub-value toggled | `report_key`, `visual_id`, `column_key` |
| 41 | `table_interacted` | export | Table data exported | `report_key`, `visual_id`, `format`, `row_count` |

### 8.6 Chart Interaction Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 42 | `chart_interacted` | hover | Data point hovered | `report_key`, `visual_id`, `chart_type` |
| 43 | `chart_interacted` | legend_toggle | Legend item toggled | `report_key`, `visual_id`, `chart_type`, `metric_key` |
| 44 | `chart_interacted` | metric_switch | Active metric changed (trend tabs) | `report_key`, `visual_id`, `chart_type`, `metric_key` |
| 45 | `chart_interacted` | granularity_change | Time granularity changed | `report_key`, `visual_id`, `chart_type`, `granularity` |
| 46 | `chart_interacted` | comparison_toggle | Comparison overlay toggled | `report_key`, `visual_id`, `chart_type` |
| 47 | `chart_interacted` | anomaly_toggle | Anomaly detection toggled | `report_key`, `visual_id`, `chart_type` |
| 48 | `chart_interacted` | insights_view | Insights strip expanded/viewed | `report_key`, `visual_id`, `chart_type` |
| 49 | `chart_interacted` | zoom | Chart zoom interaction | `report_key`, `visual_id`, `chart_type` |
| 50 | `chart_interacted` | table_toggle | Data table below chart toggled | `report_key`, `visual_id`, `chart_type` |

### 8.7 KPI Card Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 51 | `kpi_interacted` | hover | KPI card hovered (tooltip) | `report_key`, `visual_id`, `kpi_variant`, `metric_key` |
| 52 | `kpi_interacted` | gauge_hover | Gauge ring hovered | `report_key`, `visual_id`, `kpi_variant: gauge` |
| 53 | `kpi_interacted` | waterfall_hover | Waterfall item hovered | `report_key`, `visual_id`, `kpi_variant: waterfall`, `metric_key` |
| 54 | `kpi_interacted` | insight_navigate | Manual insight navigation (arrow) | `report_key`, `visual_id`, `kpi_variant: insights` |
| 55 | `kpi_interacted` | insight_auto_rotate | Insight auto-rotated | `report_key`, `visual_id`, `kpi_variant: insights` |
| 56 | `kpi_interacted` | sparkline_hover | Sparkline hover on KPI | `report_key`, `visual_id`, `kpi_variant: sparkline` |

### 8.8 Visual Impression & Dwell Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 57 | `visual_interacted` | impression | Visual enters viewport (50%+) | `report_key`, `visual_id`, `visual_type` |
| 58 | `engagement` | report_dwell | Time on report (heartbeat 30s) | `report_key`, `duration_ms` |
| 59 | `engagement` | report_dwell | Final time on unmount | `report_key`, `duration_ms` |
| 60 | `engagement` | scroll_depth | Scroll milestone reached | `report_key`, `scroll_percent` (25/50/75/100) |

### 8.9 Report Builder Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 61 | `builder_action` | open | Builder opened (new report) | `builder_type` |
| 62 | `builder_action` | edit | Builder opened (edit existing) | `builder_type`, `report_id` |
| 63 | `builder_action` | add_visual | Visual added to canvas | `builder_type`, `visual_type`, `visual_preset`, `visual_count` |
| 64 | `builder_action` | edit_visual | Visual config edited | `builder_type`, `visual_type`, `report_id` |
| 65 | `builder_action` | remove_visual | Visual deleted | `builder_type`, `visual_type`, `report_id` |
| 66 | `builder_action` | reorder | Visuals reordered | `builder_type`, `report_id` |
| 67 | `builder_action` | resize | Visual size changed | `builder_type`, `visual_type`, `report_id` |
| 68 | `builder_action` | configure_filters | Filter config sheet opened | `builder_type`, `report_id` |
| 69 | `builder_action` | apply_filters | Filter config applied | `builder_type`, `report_id` |
| 70 | `builder_action` | save | Report saved | `builder_type`, `report_id`, `visual_count` |
| 71 | `builder_action` | cancel | Builder closed without save | `builder_type`, `report_id` |
| 72 | `builder_action` | wizard_step1 | Visual type catalog opened | `builder_type` |
| 73 | `builder_action` | wizard_step2 | Config panel opened | `builder_type`, `visual_type` |
| 74 | `builder_action` | preview | Live preview triggered | `builder_type`, `report_id` |
| 75 | `builder_action` | preset_select | Preset visual selected | `builder_type`, `visual_preset` |
| 76 | `builder_action` | dnd_reorder | Drag-and-drop reorder (V2) | `builder_type: v2`, `report_id` |

### 8.10 Search Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 77 | `search` | filter_options | Search within filter dropdown | `search_location`, `search_term`, `results_count` |
| 78 | `search` | client_selector | Search in client switcher | `search_location`, `search_term`, `results_count` |
| 79 | `search` | column_selector | Search in column manager | `search_location`, `search_term` |
| 80 | `search` | report_list | Search on reports page | `search_location`, `search_term`, `results_count` |
| 81 | `search` | builder_presets | Search preset catalog | `search_location`, `search_term` |
| 82 | `search` | group_by_selector | Search group-by dimensions | `search_location`, `search_term` |

### 8.11 Export & Download Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 83 | `export_action` | table_export | Table data exported | `report_key`, `visual_id`, `format`, `row_count` |
| 84 | `export_action` | report_export | Full report exported | `report_key`, `format` |

### 8.12 Error Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 85 | `error_occurred` | api_error | Backend API call failed | `error_type`, `error_message`, `page_path`, `report_key`, `status_code` |
| 86 | `error_occurred` | render_error | Component render error | `error_type`, `error_message`, `page_path` |
| 87 | `error_occurred` | auth_error | Authentication failed | `error_type`, `error_message`, `page_path` |
| 88 | `error_occurred` | network_error | Network request failed | `error_type`, `error_message`, `page_path` |

### 8.13 Admin Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 89 | `admin_action` | user_create | Admin creates user | `target_entity: user` |
| 90 | `admin_action` | user_edit | Admin edits user | `target_entity: user`, `target_id` |
| 91 | `admin_action` | user_delete | Admin deletes user | `target_entity: user`, `target_id` |
| 92 | `admin_action` | maintenance_toggle | Maintenance mode toggled | `target_entity: maintenance` |
| 93 | `admin_action` | etl_view | ETL logs viewed | `target_entity: etl` |
| 94 | `admin_action` | filter_config_change | Filter config modified | `target_entity: filter_config` |

### 8.14 Integration & Schedule Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 95 | `integration_action` | connect | Integration connected | `provider` |
| 96 | `integration_action` | disconnect | Integration disconnected | `provider` |
| 97 | `integration_action` | oauth_start | OAuth flow started | `provider` |
| 98 | `integration_action` | oauth_complete | OAuth flow completed | `provider` |
| 99 | `integration_action` | oauth_fail | OAuth flow failed | `provider`, `error_message` |
| 100 | `schedule_action` | create | Schedule created | `schedule_type`, `frequency` |
| 101 | `schedule_action` | update | Schedule updated | `schedule_type`, `frequency` |
| 102 | `schedule_action` | delete | Schedule deleted | `schedule_type` |
| 103 | `schedule_action` | toggle_active | Schedule enabled/disabled | `schedule_type` |

### 8.15 Billing Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 104 | `billing_action` | portal_open | Stripe billing portal opened | — |
| 105 | `billing_action` | past_due_banner_shown | Past due banner displayed | — |
| 106 | `billing_action` | past_due_banner_click | Past due banner clicked | — |

### 8.16 Inline Edit Events

| # | Event Name | Action | Description | Key Parameters |
|---|-----------|--------|-------------|----------------|
| 107 | `inline_edit` | enter | Entered inline edit mode on report | `report_key`, `report_id` |
| 108 | `inline_edit` | save | Saved inline edit changes | `report_key`, `report_id` |
| 109 | `inline_edit` | cancel | Cancelled inline edit | `report_key`, `report_id` |

---

## 8.17 Event Summary by GA4 Category

| Category | Event Count | Purpose |
|----------|-------------|---------|
| Navigation & Pages | 10 | Where users go |
| Authentication | 6 | Login/signup funnel |
| Report Viewing | 2 | Report engagement |
| Filters & Slicers | 13 | How users slice data |
| Table Interactions | 10 | Table usage patterns |
| Chart Interactions | 9 | Chart engagement |
| KPI Interactions | 6 | KPI card engagement |
| Impressions & Dwell | 4 | Visibility & time spent |
| Builder | 16 | Report creation funnel |
| Search | 6 | Search behavior |
| Export | 2 | Data export patterns |
| Errors | 4 | Error monitoring |
| Admin | 6 | Admin activity |
| Integration & Schedule | 9 | Feature adoption |
| Billing | 3 | Revenue-related |
| Inline Edit | 3 | Edit mode usage |
| **TOTAL** | **109** | |

---

## 9. Testing & Validation

### 9.1 GTM Preview Mode

1. In your **Web** GTM container → Click **Preview**
2. Enter your app URL → GTM debugger opens in new tab
3. Navigate your app — verify each event fires in the debugger
4. Check that all parameters are populated correctly

### 9.2 sGTM Preview Mode

1. In your **Server** GTM container → Click **Preview**
2. This opens a debugging panel for your server container
3. Navigate your app — verify events arrive at the server
4. Check the GA4 tag fires and forwards to Google

### 9.3 GA4 DebugView

1. **GA4** → **Admin** → **DebugView**
2. Use the [GA4 Debug Chrome Extension](https://chrome.google.com/webstore/detail/google-analytics-debugger/jnkmfdileelhofjcijamephohjechhna)
3. Navigate your app — events appear in real-time in DebugView
4. Click each event to verify parameters

### 9.4 Validation Checklist

| Check | Tool | Expected |
|-------|------|----------|
| dataLayer populates | Browser console: `dataLayer.filter(e => e.event)` | All custom events listed |
| Web GTM fires tags | GTM Preview mode | Tags fire on correct triggers |
| sGTM receives events | sGTM Preview mode | All events arrive |
| GA4 receives events | GA4 DebugView | Events with correct params |
| Custom dimensions populated | GA4 DebugView → click event | All params shown |
| No PII leakage | sGTM Preview → inspect payload | No emails/passwords in params |
| Server health | `curl https://analytics.beastinsights.com/healthz` | 200 OK |

### 9.5 Testing Specific Flows

```bash
# Test sGTM is receiving events
curl -X POST https://analytics.beastinsights.com/g/collect \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "v=2&tid=G-XXXXXXXXXX&cid=test123&en=test_event&ep.test_param=hello"
```

### 9.6 Browser Console Debugging

```javascript
// In browser console — monitor all dataLayer pushes
window.dataLayer = new Proxy(window.dataLayer || [], {
  get(target, prop) {
    if (prop === 'push') {
      return function(...args) {
        console.log('[dataLayer]', ...args);
        return target.push(...args);
      };
    }
    return target[prop];
  }
});
```

---

## 10. Maintenance & Monitoring

### 10.1 Server Monitoring

```bash
# Check container status
docker ps | grep sgtm

# Check container logs
docker logs sgtm --tail 100 -f

# Check health endpoint
curl -s -o /dev/null -w "%{http_code}" https://analytics.beastinsights.com/healthz

# Monitor disk usage
df -h

# Monitor Docker resource usage
docker stats sgtm --no-stream
```

### 10.2 Uptime Monitoring Setup

Set up a simple health check cron (restarts **only the sgtm container**, nothing else):

```bash
# Create monitor script
cat > ~/sgtm/monitor.sh << 'SCRIPT'
#!/bin/bash
STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://analytics.beastinsights.com/healthz)
if [ "$STATUS" != "200" ]; then
  echo "[$(date)] sGTM is DOWN (status: $STATUS). Restarting sgtm container only..." >> ~/sgtm/monitor.log
  cd ~/sgtm && docker compose restart sgtm    # <-- targets ONLY the sgtm service
else
  echo "[$(date)] sGTM healthy" >> ~/sgtm/monitor.log
fi
SCRIPT

chmod +x ~/sgtm/monitor.sh

# Append to existing crontab (does NOT overwrite other cron jobs)
(crontab -l 2>/dev/null; echo "*/5 * * * * /home/$(whoami)/sgtm/monitor.sh") | crontab -
```

### 10.3 Log Rotation

Docker logs are configured in the compose file with `max-size: 10m` and `max-file: 3`. For Nginx:

```bash
# Nginx log rotation (usually auto-configured)
sudo cat /etc/logrotate.d/nginx
```

### 10.4 SSL Certificate Renewal

Certbot auto-renews. Verify:

```bash
sudo certbot renew --dry-run
# Check next renewal date
sudo certbot certificates
```

### 10.5 GA4 Data Quality Checks (Weekly)

| Check | Where | What to Look For |
|-------|-------|-----------------|
| Event count trend | GA4 → Reports → Realtime | Consistent event volume |
| Missing parameters | GA4 → Explore → Free form | Events with `(not set)` dimensions |
| Top events | GA4 → Reports → Events | Verify expected events appear |
| Error rate | GA4 → Explore | `error_occurred` event frequency |
| User count | GA4 → Reports → User attributes | Consistent with expected MAU |

### 10.6 Updating GTM Tags

When adding new events:

1. Add the tracking call in frontend code using `analytics.ts` functions
2. Create corresponding **DataLayer Variable** in Web GTM
3. Create **Custom Event Trigger** in Web GTM
4. Create **GA4 Event Tag** in Web GTM
5. (Optional) Create corresponding **Event Data Variable** in Server GTM
6. Register new **Custom Dimensions/Metrics** in GA4 if new params
7. Test in Preview mode → Publish both containers

---

## Appendix A: Full Event Parameter Reference

| Parameter | Type | Used In Events | Example Values |
|-----------|------|---------------|----------------|
| `page_path` | string | page_view | `/reports/sales-report` |
| `page_title` | string | page_view | `Sales Report` |
| `client_id` | string | report_viewed, user_properties_set | `10009` |
| `client_name` | string | user_properties_set | `Acme Corp` |
| `user_id` | string | user_properties_set | `usr_abc123` |
| `user_role` | string | user_properties_set | `admin`, `user`, `consultant` |
| `report_key` | string | 40+ events | `sales-report`, `approval-rate` |
| `report_name` | string | report_viewed | `Sales Report` |
| `report_type` | string | report_viewed | `stock`, `custom`, `templatized` |
| `report_id` | string | builder events | `rpt_123` |
| `visual_id` | string | visual/table/chart/kpi events | `vis_sales_table_1` |
| `visual_type` | string | visual events | `kpi`, `table`, `chart`, `matrix` |
| `visual_preset` | string | builder events | `sales_table_default` |
| `visual_count` | number | builder_action, report_viewed | `5` |
| `filter_type` | string | filter_changed | `date_range`, `dimension`, `toggle`, `parameter` |
| `filter_key` | string | filter_changed | `campaign_id`, `date_basis` |
| `filter_value` | string | filter_changed | `last_30_days`, `cb_date` |
| `filter_count` | number | filter_changed | `3` |
| `clear_type` | string | filter_cleared | `single`, `all` |
| `action` | string | many events | `sort`, `hover`, `save`, `expand` |
| `column_key` | string | table events | `revenue`, `campaign_name` |
| `sort_direction` | string | table_interacted | `asc`, `desc` |
| `page_number` | number | table_interacted | `2` |
| `group_by` | string | table_interacted | `campaign_id` |
| `chart_type` | string | chart_interacted | `bar`, `line`, `combo`, `trend` |
| `metric_key` | string | chart/kpi events | `gross_revenue`, `approval_rate` |
| `granularity` | string | chart_interacted | `day`, `week`, `month` |
| `kpi_variant` | string | kpi_interacted | `gauge`, `waterfall`, `sparkline` |
| `builder_type` | string | builder_action | `v1`, `v2`, `creator` |
| `duration_ms` | number | engagement | `45000` |
| `scroll_percent` | number | engagement | `75` |
| `search_location` | string | search | `filter_options`, `client_selector` |
| `search_term` | string | search | `visa` |
| `results_count` | number | search | `12` |
| `format` | string | export_action | `csv`, `pdf`, `xlsx` |
| `row_count` | number | export_action | `500` |
| `error_type` | string | error_occurred | `api_error`, `render_error` |
| `error_message` | string | error_occurred | `500 Internal Server Error` |
| `status_code` | number | error_occurred | `500` |
| `destination` | string | navigation | `/reports/sales-report` |
| `section` | string | navigation, sidebar_action | `analytics`, `routing` |
| `element_text` | string | navigation | `Sales Report` |
| `from_tab` | string | report_tab_switch | `sales-report` |
| `to_tab` | string | report_tab_switch | `approval-rate` |
| `from_theme` | string | theme_switch | `light` |
| `to_theme` | string | theme_switch | `dark` |
| `from_client_id` | string | client_switch | `10005` |
| `to_client_id` | string | client_switch | `10009` |
| `bookmark_name` | string | bookmark_action | `My Default View` |
| `provider` | string | integration_action | `stripe`, `slack` |
| `schedule_type` | string | schedule_action | `email`, `webhook` |
| `frequency` | string | schedule_action | `daily`, `weekly` |
| `target_entity` | string | admin_action | `user`, `filter_config` |
| `target_id` | string | admin_action | `usr_456` |
| `success` | boolean | auth_action | `true`, `false` |
| `method` | string | auth_action | `email_password` |

---

## Appendix B: GA4 Exploration Templates

### B.1 Report Engagement Funnel

- **Technique**: Funnel exploration
- **Steps**: `page_view` (report page) → `filter_changed` → `table_interacted` → `export_action`
- **Breakdown**: `report_key`, `user_role`

### B.2 Builder Conversion Funnel

- **Technique**: Funnel exploration
- **Steps**: `builder_action(open)` → `builder_action(add_visual)` → `builder_action(save)`
- **Breakdown**: `builder_type`

### B.3 Feature Adoption Heatmap

- **Technique**: Free form
- **Rows**: Event name
- **Columns**: `user_role`
- **Values**: Event count
- **Filter**: Last 30 days

### B.4 Time Spent by Report

- **Technique**: Free form
- **Rows**: `report_key`
- **Values**: Average `duration_ms` (from `engagement` events)
- **Sort**: Descending by duration

### B.5 Error Dashboard

- **Technique**: Free form
- **Rows**: `error_type`, `error_message`
- **Values**: Event count
- **Filter**: `event_name` = `error_occurred`

---

## Appendix C: Quick Reference — Implementation Checklist

- [ ] **GA4**: Create property + data stream + API secret
- [ ] **GA4**: Register all 28 custom dimensions
- [ ] **GA4**: Register all 8 custom metrics
- [ ] **GA4**: Set data retention to 14 months
- [ ] **GA4**: Disable enhanced measurement
- [ ] **Server**: Provision Linux VPS with Docker
- [ ] **Server**: Configure DNS subdomain (`analytics.beastinsights.com`)
- [ ] **Server**: Install Docker + Nginx + Certbot
- [ ] **Server**: Deploy sGTM Docker container
- [ ] **Server**: Configure Nginx reverse proxy with SSL
- [ ] **Server**: Set up firewall (UFW)
- [ ] **Server**: Set up health monitoring cron
- [ ] **GTM Server**: Create server container + get config string
- [ ] **GTM Server**: Configure GA4 client + GA4 tag
- [ ] **GTM Server**: Create event data variables
- [ ] **GTM Server**: (Optional) Add PII scrubbing transformation
- [ ] **GTM Server**: Publish container
- [ ] **GTM Web**: Update GA4 config tag with `server_container_url`
- [ ] **GTM Web**: Create all DataLayer variables (15+)
- [ ] **GTM Web**: Create all Custom Event triggers (14)
- [ ] **GTM Web**: Create all GA4 Event tags (14+)
- [ ] **GTM Web**: Publish container
- [ ] **Frontend**: Create `lib/analytics.ts` utility module
- [ ] **Frontend**: Create `hooks/useReportDwellTime.ts`
- [ ] **Frontend**: Create `hooks/useScrollDepth.ts`
- [ ] **Frontend**: Create `hooks/useVisualVisibility.ts`
- [ ] **Frontend**: Add tracking to report page (`page.tsx`)
- [ ] **Frontend**: Add tracking to filter components
- [ ] **Frontend**: Add tracking to DataTable
- [ ] **Frontend**: Add tracking to chart components
- [ ] **Frontend**: Add tracking to KPI cards
- [ ] **Frontend**: Add tracking to builder components
- [ ] **Frontend**: Add tracking to auth flows
- [ ] **Frontend**: Add tracking to sidebar/navigation
- [ ] **Frontend**: Add tracking to search inputs
- [ ] **Frontend**: Add tracking to admin pages
- [ ] **Frontend**: Update root layout GTM script (sGTM URL)
- [ ] **Testing**: Validate with GTM Preview (web + server)
- [ ] **Testing**: Validate with GA4 DebugView
- [ ] **Testing**: Check all 109 events fire correctly
- [ ] **Testing**: Verify no PII leakage
- [ ] **Testing**: Verify sGTM health endpoint
- [ ] **Production**: Deploy frontend changes
- [ ] **Production**: Publish both GTM containers (non-preview)

---

*Document generated for Beast Insights platform. 109 events covering all user interactions across 20+ pages, 15+ component categories, and 3 builder variants.*
