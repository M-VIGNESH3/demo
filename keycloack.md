# Keycloak Installation & Configuration for Kubernetes OIDC Authentication

## Complete Production Setup Guide

**Author:** DevOps Engineering Team
**Version:** 2.0
**Last Updated:** July 2025
**Classification:** Internal Documentation

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [SSL Certificate Generation with Certbot](#3-ssl-certificate-generation-with-certbot)
4. [HAProxy Configuration](#4-haproxy-configuration)
5. [Keycloak Installation](#5-keycloak-installation)
6. [Keycloak OIDC Configuration](#6-keycloak-oidc-configuration)
7. [Kubernetes API Server OIDC Integration](#7-kubernetes-api-server-oidc-integration)
8. [Kubeconfig & Client Configuration](#8-kubeconfig--client-configuration)
9. [RBAC Policies](#9-rbac-policies)
10. [Testing & Validation](#10-testing--validation)
11. [Troubleshooting](#11-troubleshooting)
12. [Maintenance & Backup](#12-maintenance--backup)

---

## 1. Architecture Overview

```
                          ┌─────────────────────────────────────────────┐
                          │              ARCHITECTURE FLOW              │
                          └─────────────────────────────────────────────┘

  ┌──────────┐       ┌──────────────────┐       ┌──────────────────────┐
  │          │       │                  │       │                      │
  │  Users / │──────▶│   HAProxy        │──────▶│   Keycloak Server    │
  │  DevOps  │ HTTPS │   (TLS Term.)    │ HTTP  │   (OIDC Provider)    │
  │  Teams   │       │   Port 443       │:8080  │   auth.example.com   │
  │          │       │                  │       │                      │
  └──────────┘       └──────────────────┘       └──────────────────────┘
       │                     │                           │
       │                     │                           │
       │              ┌──────┴───────┐            ┌──────┴───────┐
       │              │  Certbot     │            │  PostgreSQL  │
       │              │  (Let's      │            │  Database    │
       │              │   Encrypt)   │            │              │
       │              └──────────────┘            └──────────────┘
       │
       │         ┌──────────────────────────────────────────┐
       │         │         Kubernetes Cluster               │
       └────────▶│  ┌──────────────────────────────────┐   │
     kubectl     │  │  API Server (--oidc-* flags)     │   │
     + OIDC      │  │  validates JWT tokens from       │   │
     token       │  │  Keycloak                        │   │
                 │  └──────────────────────────────────┘   │
                 │  ┌──────────────────────────────────┐   │
                 │  │  RBAC (ClusterRoles/Bindings)    │   │
                 │  │  mapped to Keycloak groups       │   │
                 │  └──────────────────────────────────┘   │
                 └──────────────────────────────────────────┘
```

### Component Summary

| Component | Purpose | Version |
|-----------|---------|---------|
| Keycloak | OIDC Identity Provider | 25.x+ |
| HAProxy | Reverse Proxy + TLS Termination | 2.8+ |
| Certbot | SSL Certificate (Let's Encrypt) | Latest |
| PostgreSQL | Keycloak Persistent Database | 15+ |
| Kubernetes | Target Cluster | 1.28+ |

---

## 2. Prerequisites

### 2.1 Infrastructure Requirements

```
HAProxy Server:
  - OS: Ubuntu 22.04 / RHEL 9
  - RAM: 2 GB minimum
  - CPU: 2 vCPU
  - Disk: 20 GB
  - Public IP: Required (for Certbot DNS validation / HTTP challenge)

Keycloak Server:
  - OS: Ubuntu 22.04 / RHEL 9
  - RAM: 4 GB minimum (8 GB recommended)
  - CPU: 2 vCPU (4 recommended)
  - Disk: 50 GB
  - Private/Public IP: Accessible from HAProxy
```

### 2.2 DNS Configuration

Create the following DNS A record **before** proceeding:

```
auth.example.com  →  <HAProxy_Public_IP>
```

> **⚠️ IMPORTANT:** Replace `example.com` with your actual domain throughout this document. The DNS record must be propagated before SSL certificate generation.

Verify DNS propagation:

```bash
dig +short auth.example.com
nslookup auth.example.com
```

### 2.3 Network/Firewall Requirements

```
HAProxy Server:
  - Inbound: 80/tcp (Certbot HTTP challenge), 443/tcp (HTTPS)
  - Outbound: 8080/tcp to Keycloak server

Keycloak Server:
  - Inbound: 8080/tcp from HAProxy
  - Outbound: 5432/tcp to PostgreSQL (if external DB)

Kubernetes API Server:
  - Outbound: 443/tcp to auth.example.com (to fetch OIDC discovery)
```

### 2.4 Software Installation (HAProxy Server)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y
sudo apt install -y haproxy certbot curl wget jq

# RHEL/CentOS
sudo dnf install -y haproxy certbot curl wget jq
```

### 2.5 Software Installation (Keycloak Server)

```bash
# Install Java 17 (Required for Keycloak)
# Ubuntu/Debian
sudo apt update
sudo apt install -y openjdk-17-jdk-headless wget curl unzip

# RHEL/CentOS
sudo dnf install -y java-17-openjdk-headless wget curl unzip

# Verify Java
java -version
```

---

## 3. SSL Certificate Generation with Certbot

### 3.1 Generate SSL Certificate (Standalone Mode)

> **Note:** Stop HAProxy temporarily if it's already running on port 80.

```bash
# On the HAProxy Server

# Stop HAProxy if running (port 80 must be free)
sudo systemctl stop haproxy 2>/dev/null || true

# Generate certificate using standalone HTTP challenge
sudo certbot certonly \
  --standalone \
  --preferred-challenges http \
  -d auth.example.com \
  --non-interactive \
  --agree-tos \
  --email devops@example.com
```

### 3.2 Verify Certificate Files

```bash
# Certificate files are stored at:
ls -la /etc/letsencrypt/live/auth.example.com/

# You should see:
#   cert.pem       → Server certificate
#   chain.pem      → Intermediate certificate
#   fullchain.pem  → Full certificate chain
#   privkey.pem    → Private key
```

### 3.3 Prepare Combined PEM for HAProxy

HAProxy requires the certificate and private key in a single PEM file:

```bash
# Create HAProxy SSL directory
sudo mkdir -p /etc/haproxy/certs

# Combine fullchain + private key into a single PEM file
sudo cat /etc/letsencrypt/live/auth.example.com/fullchain.pem \
         /etc/letsencrypt/live/auth.example.com/privkey.pem \
  | sudo tee /etc/haproxy/certs/auth.example.com.pem > /dev/null

# Secure the file
sudo chmod 600 /etc/haproxy/certs/auth.example.com.pem
sudo chown root:root /etc/haproxy/certs/auth.example.com.pem
```

### 3.4 Setup Auto-Renewal with HAProxy Reload

Create the renewal hook script:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/haproxy-deploy.sh << 'SCRIPT'
#!/bin/bash
# Certbot post-renewal hook for HAProxy
# This script runs after certificate renewal

DOMAIN="auth.example.com"
HAPROXY_CERT_DIR="/etc/haproxy/certs"

# Combine renewed certificate files
cat /etc/letsencrypt/live/${DOMAIN}/fullchain.pem \
    /etc/letsencrypt/live/${DOMAIN}/privkey.pem \
  > ${HAPROXY_CERT_DIR}/${DOMAIN}.pem

# Set permissions
chmod 600 ${HAPROXY_CERT_DIR}/${DOMAIN}.pem

# Reload HAProxy (graceful - no downtime)
systemctl reload haproxy

echo "$(date): Certificate renewed and HAProxy reloaded for ${DOMAIN}" \
  >> /var/log/certbot-renewal.log
SCRIPT

sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/haproxy-deploy.sh
```

Test auto-renewal (dry run):

```bash
sudo certbot renew --dry-run
```

Verify the cron/timer is active:

```bash
# Certbot auto-renewal timer
sudo systemctl status certbot.timer
sudo systemctl enable certbot.timer
```

---

## 4. HAProxy Configuration

### 4.1 Backup Default Configuration

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup.$(date +%F)
```

### 4.2 Configure HAProxy

```bash
sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         /dev/log local0
    log         /dev/log local1 notice
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # SSL tuning
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    tune.ssl.default-dh-param 2048

#---------------------------------------------------------------------
# Default settings
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  forwardfor except 127.0.0.0/8
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# Frontend: HTTP → HTTPS Redirect
#---------------------------------------------------------------------
frontend http_front
    bind *:80
    mode http

    # Allow Certbot ACME challenge
    acl is_acme path_beg /.well-known/acme-challenge/
    use_backend certbot_backend if is_acme

    # Redirect all other HTTP traffic to HTTPS
    redirect scheme https code 301 if !is_acme

#---------------------------------------------------------------------
# Frontend: HTTPS (TLS Termination)
#---------------------------------------------------------------------
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/auth.example.com.pem alpn h2,http/1.1
    mode http

    # Security headers
    http-response set-header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    http-response set-header X-Frame-Options "SAMEORIGIN"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-XSS-Protection "1; mode=block"

    # ACL: Route based on subdomain
    acl is_keycloak hdr(host) -i auth.example.com

    # Route to Keycloak backend
    use_backend keycloak_backend if is_keycloak

    # Default backend (optional: return 403 for unmatched hosts)
    default_backend no_match

#---------------------------------------------------------------------
# Backend: Keycloak
#---------------------------------------------------------------------
backend keycloak_backend
    mode http
    balance roundrobin
    option httpchk GET /health/ready
    http-check expect status 200

    # Forward original client information
    http-request set-header X-Forwarded-For %[src]
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Host %[req.hdr(host)]
    http-request set-header X-Forwarded-Port 443

    # Keycloak server(s)
    # Replace <KEYCLOAK_SERVER_IP> with actual IP
    server keycloak1 <KEYCLOAK_SERVER_IP>:8080 check inter 5s fall 3 rise 2

#---------------------------------------------------------------------
# Backend: Certbot ACME Challenge
#---------------------------------------------------------------------
backend certbot_backend
    mode http
    server certbot 127.0.0.1:8888

#---------------------------------------------------------------------
# Backend: No Match (Default deny)
#---------------------------------------------------------------------
backend no_match
    mode http
    http-request deny deny_status 403

#---------------------------------------------------------------------
# Stats Page (Optional - restrict access in production)
#---------------------------------------------------------------------
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
    stats auth admin:StrongPassword123!
EOF
```

### 4.3 Validate and Start HAProxy

```bash
# Validate configuration
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# Expected output: "Configuration file is valid"

# Enable and start HAProxy
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```

### 4.4 Verify SSL Endpoint

```bash
# Test HTTPS access (should return 503 until Keycloak is running)
curl -I https://auth.example.com

# Check SSL certificate details
openssl s_client -connect auth.example.com:443 -servername auth.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

---

## 5. Keycloak Installation

### 5.1 Install and Configure PostgreSQL

```bash
# On the Keycloak Server (or a dedicated DB server)

# Install PostgreSQL
# Ubuntu/Debian
sudo apt install -y postgresql postgresql-contrib

# RHEL/CentOS
sudo dnf install -y postgresql-server postgresql-contrib
sudo postgresql-setup --initdb

# Start and enable PostgreSQL
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

Create the Keycloak database and user:

```bash
sudo -u postgres psql << 'DBSQL'
-- Create Keycloak database user
CREATE USER keycloak WITH PASSWORD 'K3ycl0ak_Str0ng_P@ssw0rd!';

-- Create Keycloak database
CREATE DATABASE keycloak OWNER keycloak;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;

-- Verify
\l keycloak
\du keycloak

\q
DBSQL
```

Configure PostgreSQL to allow connections (if Keycloak is on a different server):

```bash
# Edit pg_hba.conf to allow Keycloak server IP
# Find the file location
sudo -u postgres psql -c "SHOW hba_file;"

# Add this line (replace with actual Keycloak IP)
echo "host keycloak keycloak <KEYCLOAK_IP>/32 scram-sha-256" | \
  sudo tee -a /etc/postgresql/15/main/pg_hba.conf

# Edit postgresql.conf to listen on all interfaces (if needed)
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" \
  /etc/postgresql/15/main/postgresql.conf

# Restart PostgreSQL
sudo systemctl restart postgresql
```

### 5.2 Create Keycloak System User

```bash
# Create a dedicated system user for Keycloak
sudo groupadd -r keycloak
sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
```

### 5.3 Download and Install Keycloak

```bash
# Set version (check https://github.com/keycloak/keycloak/releases for latest)
KEYCLOAK_VERSION="25.0.2"

# Download Keycloak
cd /tmp
wget https://github.com/keycloak/keycloak/releases/download/${KEYCLOAK_VERSION}/keycloak-${KEYCLOAK_VERSION}.tar.gz

# Extract to /opt
sudo tar -xzf keycloak-${KEYCLOAK_VERSION}.tar.gz -C /opt/
sudo mv /opt/keycloak-${KEYCLOAK_VERSION} /opt/keycloak

# Set ownership
sudo chown -R keycloak:keycloak /opt/keycloak
```

### 5.4 Configure Keycloak

```bash
sudo tee /opt/keycloak/conf/keycloak.conf << 'EOF'
# =============================================================
# Keycloak Configuration
# =============================================================

# --- Database ---
db=postgres
db-url-host=localhost
db-url-port=5432
db-url-database=keycloak
db-username=keycloak
db-password=K3ycl0ak_Str0ng_P@ssw0rd!
db-schema=public

# --- HTTP ---
# Keycloak listens on HTTP (TLS is terminated at HAProxy)
http-enabled=true
http-host=0.0.0.0
http-port=8080

# --- Proxy ---
# HAProxy handles TLS; Keycloak runs behind a reverse proxy
proxy-headers=xforwarded
http-relative-path=/

# --- Hostname ---
# The public hostname users access (via HAProxy)
hostname=auth.example.com
hostname-strict=true
hostname-strict-https=true

# --- Health ---
health-enabled=true

# --- Metrics ---
metrics-enabled=true

# --- Logging ---
log=console,file
log-level=INFO
log-file=/opt/keycloak/data/log/keycloak.log
EOF

# Create log directory
sudo mkdir -p /opt/keycloak/data/log
sudo chown -R keycloak:keycloak /opt/keycloak/data
```

### 5.5 Build Keycloak (Optimized Production Mode)

```bash
sudo -u keycloak /opt/keycloak/bin/kc.sh build
```

Expected output:
```
Server configuration updated and target map file generated.
```

### 5.6 Create Systemd Service

```bash
sudo tee /etc/systemd/system/keycloak.service << 'EOF'
[Unit]
Description=Keycloak Identity and Access Management
After=network.target postgresql.service
Requires=network.target

[Service]
Type=exec
User=keycloak
Group=keycloak
WorkingDirectory=/opt/keycloak

# Initial admin credentials (only used on first boot)
Environment="KEYCLOAK_ADMIN=admin"
Environment="KEYCLOAK_ADMIN_PASSWORD=Admin@Str0ng_P@ssw0rd!"

# Java options
Environment="JAVA_OPTS=-Xms512m -Xmx2048m -XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m"

ExecStart=/opt/keycloak/bin/kc.sh start --optimized

# Restart policy
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/keycloak/data

[Install]
WantedBy=multi-user.target
EOF
```

### 5.7 Start Keycloak

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable and start Keycloak
sudo systemctl enable keycloak
sudo systemctl start keycloak

# Check status
sudo systemctl status keycloak

# View logs
sudo journalctl -u keycloak -f --no-pager
```

### 5.8 Verify Keycloak is Running

```bash
# Local health check
curl -s http://localhost:8080/health/ready | jq .

# Expected output:
# {
#   "status": "UP",
#   "checks": [...]
# }

# Test via HAProxy (from any machine)
curl -I https://auth.example.com
# Should return 200 OK
```

Access the Keycloak Admin Console:

```
URL:      https://auth.example.com/admin
Username: admin
Password: Admin@Str0ng_P@ssw0rd!
```

> **⚠️ SECURITY:** Change the admin password immediately after first login. Remove the `KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD` environment variables from the systemd service file after initial setup.

---

## 6. Keycloak OIDC Configuration

### 6.1 Create a New Realm

> A realm is a space where you manage users, credentials, roles, and groups. Do NOT use the `master` realm for applications.

**Via Admin Console:**

1. Login to `https://auth.example.com/admin`
2. Click the dropdown in the top-left (shows "master")
3. Click **"Create Realm"**
4. Configure:

| Field | Value |
|-------|-------|
| Realm Name | `kubernetes` |
| Enabled | ON |

5. Click **"Create"**

**Via CLI (using kcadm):**

```bash
# Authenticate to the admin CLI
/opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password 'Admin@Str0ng_P@ssw0rd!'

# Create the kubernetes realm
/opt/keycloak/bin/kcadm.sh create realms \
  -s realm=kubernetes \
  -s enabled=true \
  -s displayName="Kubernetes Cluster Authentication"
```

### 6.2 Create an OIDC Client

**Via Admin Console:**

1. Navigate to: **Kubernetes realm → Clients → Create client**
2. Configure:

**General Settings:**

| Field | Value |
|-------|-------|
| Client type | OpenID Connect |
| Client ID | `kubernetes-cluster` |
| Name | Kubernetes Cluster OIDC |
| Always display in UI | ON |

3. Click **"Next"**

**Capability Config:**

| Field | Value |
|-------|-------|
| Client authentication | ON |
| Authorization | OFF |
| Standard flow | ✅ Enabled |
| Direct access grants | ✅ Enabled |
| Implicit flow | ❌ Disabled |
| Service account roles | ❌ Disabled |

4. Click **"Next"**

**Login Settings:**

| Field | Value |
|-------|-------|
| Root URL | `https://auth.example.com` |
| Home URL | `https://auth.example.com` |
| Valid Redirect URIs | `http://localhost:8000/*` |
| | `http://localhost:18000/*` |
| | `urn:ietf:wg:oauth:2.0:oob` |
| Web Origins | `+` |

> **Note:** The redirect URIs are for kubectl OIDC plugins (like `kubelogin`) that run a local callback server.

5. Click **"Save"**

**Retrieve Client Secret:**

1. Go to **Clients → kubernetes-cluster → Credentials**
2. Copy the **Client Secret** — you will need this later

```bash
# Save the client secret for later use
CLIENT_SECRET="<copied-client-secret>"
```

**Via CLI:**

```bash
# Create the client
/opt/keycloak/bin/kcadm.sh create clients \
  -r kubernetes \
  -s clientId=kubernetes-cluster \
  -s name="Kubernetes Cluster OIDC" \
  -s enabled=true \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s standardFlowEnabled=true \
  -s directAccessGrantsEnabled=true \
  -s 'redirectUris=["http://localhost:8000/*","http://localhost:18000/*","urn:ietf:wg:oauth:2.0:oob"]' \
  -s 'webOrigins=["+"]'

# Get the client UUID
CLIENT_UUID=$(/opt/keycloak/bin/kcadm.sh get clients -r kubernetes \
  -q clientId=kubernetes-cluster --fields id --format csv --noquotes)

# Get the client secret
/opt/keycloak/bin/kcadm.sh get clients/${CLIENT_UUID}/client-secret -r kubernetes
```

### 6.3 Create Client Scopes and Mappers

Kubernetes needs `groups` information in the JWT token to map users to RBAC roles.

**Create a `groups` Client Scope:**

1. Navigate to: **Client scopes → Create client scope**
2. Configure:

| Field | Value |
|-------|-------|
| Name | `groups` |
| Description | Maps user groups to token claims |
| Type | Default |
| Display on consent screen | OFF |
| Protocol | OpenID Connect |

3. Click **"Save"**

**Add a Group Membership Mapper:**

1. Inside the `groups` scope → **Mappers → Configure a new mapper**
2. Select: **"Group Membership"**
3. Configure:

| Field | Value |
|-------|-------|
| Name | `groups` |
| Token Claim Name | `groups` |
| Full group path | OFF |
| Add to ID token | ✅ ON |
| Add to access token | ✅ ON |
| Add to userinfo | ✅ ON |
| Add to token introspection | ✅ ON |

4. Click **"Save"**

**Assign the `groups` Scope to the Client:**

1. Navigate to: **Clients → kubernetes-cluster → Client scopes**
2. Click **"Add client scope"**
3. Select `groups` → Click **"Add" → "Default"**

**Add Username Mapper (Optional but Recommended):**

1. Navigate to: **Clients → kubernetes-cluster → Client scopes → dedicated scope → Mappers → Configure a new mapper**
2. Select: **"User Property"**
3. Configure:

| Field | Value |
|-------|-------|
| Name | `username` |
| Property | `username` |
| Token Claim Name | `preferred_username` |
| Claim JSON Type | String |
| Add to ID token | ✅ ON |
| Add to access token | ✅ ON |

4. Click **"Save"**

### 6.4 Create Groups

Create groups that will map to Kubernetes RBAC roles:

1. Navigate to: **Groups → Create group**

Create the following groups:

| Group Name | Purpose |
|------------|---------|
| `cluster-admins` | Full cluster admin access |
| `developers` | Namespace-scoped developer access |
| `viewers` | Read-only access across cluster |
| `sre-team` | SRE team with elevated access |

**Via CLI:**

```bash
/opt/keycloak/bin/kcadm.sh create groups -r kubernetes -s name="cluster-admins"
/opt/keycloak/bin/kcadm.sh create groups -r kubernetes -s name="developers"
/opt/keycloak/bin/kcadm.sh create groups -r kubernetes -s name="viewers"
/opt/keycloak/bin/kcadm.sh create groups -r kubernetes -s name="sre-team"
```

### 6.5 Create Users

1. Navigate to: **Users → Add user**

**Example: Create an admin user:**

| Field | Value |
|-------|-------|
| Username | `john.admin` |
| Email | `john@example.com` |
| First Name | John |
| Last Name | Admin |
| Enabled | ON |
| Email Verified | ON |

2. Click **"Create"**
3. Go to **Credentials** tab → **Set password**
   - Password: `<strong-password>`
   - Temporary: OFF (or ON to force change on first login)
4. Go to **Groups** tab → **Join Group** → Select `cluster-admins`

**Via CLI:**

```bash
# Create user
/opt/keycloak/bin/kcadm.sh create users -r kubernetes \
  -s username=john.admin \
  -s email=john@example.com \
  -s firstName=John \
  -s lastName=Admin \
  -s enabled=true \
  -s emailVerified=true

# Set password
/opt/keycloak/bin/kcadm.sh set-password -r kubernetes \
  --username john.admin \
  --new-password 'UserStr0ngP@ss!'

# Get group ID
GROUP_ID=$(/opt/keycloak/bin/kcadm.sh get groups -r kubernetes \
  -q name=cluster-admins --fields id --format csv --noquotes)

# Get user ID
USER_ID=$(/opt/keycloak/bin/kcadm.sh get users -r kubernetes \
  -q username=john.admin --fields id --format csv --noquotes)

# Add user to group
/opt/keycloak/bin/kcadm.sh update users/${USER_ID}/groups/${GROUP_ID} \
  -r kubernetes -s realm=kubernetes -s userId=${USER_ID} -s groupId=${GROUP_ID} -n
```

### 6.6 Verify OIDC Discovery Endpoint

```bash
# Test OIDC discovery endpoint
curl -s https://auth.example.com/realms/kubernetes/.well-known/openid-configuration | jq .
```

Expected key fields in the response:
```json
{
  "issuer": "https://auth.example.com/realms/kubernetes",
  "authorization_endpoint": "https://auth.example.com/realms/kubernetes/protocol/openid-connect/auth",
  "token_endpoint": "https://auth.example.com/realms/kubernetes/protocol/openid-connect/token",
  "userinfo_endpoint": "https://auth.example.com/realms/kubernetes/protocol/openid-connect/userinfo",
  "jwks_uri": "https://auth.example.com/realms/kubernetes/protocol/openid-connect/certs",
  ...
}
```

### 6.7 Test Token Generation

```bash
# Test direct access grant (Resource Owner Password Credentials)
TOKEN_RESPONSE=$(curl -s -X POST \
  "https://auth.example.com/realms/kubernetes/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=kubernetes-cluster" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "username=john.admin" \
  -d "password=UserStr0ngP@ss!" \
  -d "scope=openid profile email groups")

# Extract tokens
ACCESS_TOKEN=$(echo $TOKEN_RESPONSE | jq -r .access_token)
ID_TOKEN=$(echo $TOKEN_RESPONSE | jq -r .id_token)

# Decode and inspect the ID token (JWT)
echo $ID_TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .
```

Expected decoded ID token:
```json
{
  "exp": 1720000000,
  "sub": "a1b2c3d4-...",
  "iss": "https://auth.example.com/realms/kubernetes",
  "aud": "kubernetes-cluster",
  "preferred_username": "john.admin",
  "email": "john@example.com",
  "groups": [
    "cluster-admins"
  ]
}
```

> **✅ Checkpoint:** If you see `groups` in the decoded token, the OIDC configuration is correct.

---

## 7. Kubernetes API Server OIDC Integration

### 7.1 Ensure API Server Can Reach Keycloak

From each **control plane node**:

```bash
# Test connectivity to Keycloak OIDC discovery
curl -s https://auth.example.com/realms/kubernetes/.well-known/openid-configuration | jq .issuer

# If the API server cannot resolve the domain, add to /etc/hosts:
echo "<HAPROXY_IP> auth.example.com" | sudo tee -a /etc/hosts
```

### 7.2 Configure API Server OIDC Flags

#### For kubeadm-based clusters:

Edit the API server manifest on **each control plane node**:

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following flags under `spec.containers[0].command`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    # ... existing flags ...

    # ========== OIDC Configuration ==========
    - --oidc-issuer-url=https://auth.example.com/realms/kubernetes
    - --oidc-client-id=kubernetes-cluster
    - --oidc-username-claim=preferred_username
    - --oidc-username-prefix=oidc:
    - --oidc-groups-claim=groups
    - --oidc-groups-prefix=oidc:
    # Optional: If using a self-signed CA for Keycloak
    # - --oidc-ca-file=/etc/kubernetes/pki/keycloak-ca.crt
    # ==========================================

    # ... remaining configuration ...
```

> **Note:** The `oidc-username-prefix` and `oidc-groups-prefix` prevent collisions with system users/groups. A user `john.admin` in Keycloak becomes `oidc:john.admin` in Kubernetes.

#### For k3s clusters:

```bash
# Edit k3s service configuration
sudo vi /etc/systemd/system/k3s.service

# Or add to /etc/rancher/k3s/config.yaml
sudo tee -a /etc/rancher/k3s/config.yaml << 'EOF'
kube-apiserver-arg:
  - "oidc-issuer-url=https://auth.example.com/realms/kubernetes"
  - "oidc-client-id=kubernetes-cluster"
  - "oidc-username-claim=preferred_username"
  - "oidc-username-prefix=oidc:"
  - "oidc-groups-claim=groups"
  - "oidc-groups-prefix=oidc:"
EOF

sudo systemctl restart k3s
```

#### For RKE2 clusters:

```bash
sudo tee -a /etc/rancher/rke2/config.yaml << 'EOF'
kube-apiserver-arg:
  - "oidc-issuer-url=https://auth.example.com/realms/kubernetes"
  - "oidc-client-id=kubernetes-cluster"
  - "oidc-username-claim=preferred_username"
  - "oidc-username-prefix=oidc:"
  - "oidc-groups-claim=groups"
  - "oidc-groups-prefix=oidc:"
EOF

sudo systemctl restart rke2-server
```

### 7.3 Verify API Server Restart

```bash
# For kubeadm: API server pod restarts automatically after manifest change
# Wait for it to come back
kubectl get pods -n kube-system -l component=kube-apiserver -w

# Check API server logs for OIDC initialization
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i oidc

# Verify API server is running with OIDC flags
kubectl get pods -n kube-system kube-apiserver-<node-name> -o yaml | grep oidc
```

### 7.4 OIDC Flag Reference

| Flag | Description | Value |
|------|-------------|-------|
| `--oidc-issuer-url` | URL of the OIDC provider | `https://auth.example.com/realms/kubernetes` |
| `--oidc-client-id` | Client ID for the OIDC client | `kubernetes-cluster` |
| `--oidc-username-claim` | JWT claim to use as the username | `preferred_username` |
| `--oidc-username-prefix` | Prefix for OIDC usernames | `oidc:` |
| `--oidc-groups-claim` | JWT claim to use for groups | `groups` |
| `--oidc-groups-prefix` | Prefix for OIDC groups | `oidc:` |
| `--oidc-ca-file` | CA certificate for OIDC provider | Path to CA cert (if self-signed) |
| `--oidc-required-claim` | Required claim key=value pair | Optional |

---

## 8. Kubeconfig & Client Configuration

### 8.1 Install kubelogin (OIDC kubectl Plugin)

`kubelogin` (also known as `kubectl oidc-login`) handles the OIDC authentication flow.

```bash
# Install via krew (kubectl plugin manager)
kubectl krew install oidc-login

# Or install directly
# Linux AMD64
curl -LO https://github.com/int128/kubelogin/releases/latest/download/kubelogin_linux_amd64.zip
unzip kubelogin_linux_amd64.zip
sudo mv kubelogin /usr/local/bin/kubectl-oidc_login
chmod +x /usr/local/bin/kubectl-oidc_login

# macOS
brew install int128/kubelogin/kubelogin

# Verify installation
kubectl oidc-login --help
```

### 8.2 Configure Kubeconfig for OIDC Users

Create or update the kubeconfig file for users:

```bash
# Set variables
CLUSTER_NAME="my-production-cluster"
CLUSTER_API_SERVER="https://<KUBERNETES_API_SERVER>:6443"
CLUSTER_CA_DATA="<base64-encoded-cluster-CA>"  # from existing kubeconfig
CLIENT_ID="kubernetes-cluster"
CLIENT_SECRET="<keycloak-client-secret>"
ISSUER_URL="https://auth.example.com/realms/kubernetes"

# --- Option A: Using kubelogin (Browser-based Login) ---

# Set cluster
kubectl config set-cluster ${CLUSTER_NAME} \
  --server=${CLUSTER_API_SERVER} \
  --certificate-authority=/path/to/cluster-ca.crt \
  --embed-certs=true

# Set OIDC credentials
kubectl config set-credentials oidc-user \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=${ISSUER_URL} \
  --exec-arg=--oidc-client-id=${CLIENT_ID} \
  --exec-arg=--oidc-client-secret=${CLIENT_SECRET} \
  --exec-arg=--oidc-extra-scope=groups \
  --exec-arg=--oidc-extra-scope=profile \
  --exec-arg=--oidc-extra-scope=email

# Set context
kubectl config set-context ${CLUSTER_NAME}-oidc \
  --cluster=${CLUSTER_NAME} \
  --user=oidc-user

# Use the OIDC context
kubectl config use-context ${CLUSTER_NAME}-oidc
```

### 8.3 Complete Kubeconfig File Example

```yaml
# ~/.kube/config
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
    certificate-authority-data: <BASE64_CA_DATA>
    server: https://<KUBERNETES_API_SERVER>:6443
  name: my-production-cluster

contexts:
- context:
    cluster: my-production-cluster
    user: oidc-user
  name: my-production-cluster-oidc

current-context: my-production-cluster-oidc

users:
- name: oidc-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://auth.example.com/realms/kubernetes
      - --oidc-client-id=kubernetes-cluster
      - --oidc-client-secret=<CLIENT_SECRET>
      - --oidc-extra-scope=groups
      - --oidc-extra-scope=profile
      - --oidc-extra-scope=email
      env: null
      provideClusterInfo: false
```

### 8.4 Distribute Kubeconfig to Team Members

```bash
# Generate a distributable kubeconfig file
cat > kubeconfig-oidc-template.yaml << 'KUBEEOF'
apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority-data: ${CLUSTER_CA_DATA}
    server: ${CLUSTER_API_SERVER}
  name: ${CLUSTER_NAME}
contexts:
- context:
    cluster: ${CLUSTER_NAME}
    user: oidc-user
  name: ${CLUSTER_NAME}-oidc
current-context: ${CLUSTER_NAME}-oidc
users:
- name: oidc-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://auth.example.com/realms/kubernetes
      - --oidc-client-id=kubernetes-cluster
      - --oidc-client-secret=${CLIENT_SECRET}
      - --oidc-extra-scope=groups
      - --oidc-extra-scope=profile
      - --oidc-extra-scope=email
KUBEEOF

# Replace variables with envsubst or sed
envsubst < kubeconfig-oidc-template.yaml > kubeconfig-oidc.yaml
```

---

## 9. RBAC Policies

### 9.1 Cluster Admin RBAC (cluster-admins group)

```yaml
# cluster-admin-rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-cluster-admins
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
    rbac.keycloak.io/group: cluster-admins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin  # Built-in Kubernetes cluster-admin role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "oidc:cluster-admins"  # Note the prefix matching --oidc-groups-prefix
```

### 9.2 Developer RBAC (developers group)

```yaml
# developer-rbac.yaml
---
# Custom ClusterRole for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-role
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
rules:
# Workload resources
- apiGroups: ["", "apps", "batch"]
  resources:
  - pods
  - pods/log
  - pods/exec
  - deployments
  - replicasets
  - statefulsets
  - daemonsets
  - jobs
  - cronjobs
  - services
  - configmaps
  - secrets
  - persistentvolumeclaims
  - events
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Networking
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# Bind to specific namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-developers
  namespace: development  # Restrict to this namespace
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: developer-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "oidc:developers"

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-developers
  namespace: staging
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: developer-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "oidc:developers"
```

### 9.3 Viewer RBAC (viewers group)

```yaml
# viewer-rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-viewers
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view  # Built-in Kubernetes view role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "oidc:viewers"
```

### 9.4 SRE Team RBAC

```yaml
# sre-rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre-role
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["", "apps"]
  resources:
  - pods
  - pods/exec
  - pods/log
  - deployments
  - replicasets
  - statefulsets
  - services
  - configmaps
  - nodes
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch", "cordon", "uncordon", "drain"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oidc-sre-team
  labels:
    app.kubernetes.io/managed-by: keycloak-oidc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sre-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "oidc:sre-team"
```

### 9.5 Apply All RBAC Policies

```bash
# Apply RBAC configurations
kubectl apply -f cluster-admin-rbac.yaml
kubectl apply -f developer-rbac.yaml
kubectl apply -f viewer-rbac.yaml
kubectl apply -f sre-rbac.yaml

# Verify
kubectl get clusterrolebindings -l app.kubernetes.io/managed-by=keycloak-oidc
kubectl get rolebindings -A -l app.kubernetes.io/managed-by=keycloak-oidc
```

---

## 10. Testing & Validation

### 10.1 End-to-End Login Test

```bash
# Switch to OIDC context
kubectl config use-context my-production-cluster-oidc

# This should open a browser for Keycloak login
kubectl get pods -A

# If browser doesn't open, use grant-type=password for testing:
kubectl oidc-login get-token \
  --oidc-issuer-url=https://auth.example.com/realms/kubernetes \
  --oidc-client-id=kubernetes-cluster \
  --oidc-client-secret=<CLIENT_SECRET> \
  --oidc-extra-scope=groups \
  --grant-type=password \
  --username=john.admin \
  --password='UserStr0ngP@ss!'
```

### 10.2 Verify Token and Identity

```bash
# Check who you are authenticated as
kubectl auth whoami

# Expected output for john.admin in cluster-admins group:
# Username: oidc:john.admin
# Groups:   [oidc:cluster-admins system:authenticated]
```

### 10.3 Test RBAC Permissions

```bash
# As cluster-admin: should succeed
kubectl get nodes
kubectl get pods -A

# Test as a developer (create a user in developers group and login)
kubectl auth can-i create deployments --namespace=development
# Expected: yes

kubectl auth can-i delete nodes
# Expected: no

# Test as a viewer
kubectl auth can-i get pods --all-namespaces
# Expected: yes

kubectl auth can-i create pods
# Expected: no
```

### 10.4 Comprehensive Validation Checklist

```bash
#!/bin/bash
# validation-checklist.sh

echo "============================================"
echo " Keycloak OIDC Integration Validation"
echo "============================================"

echo ""
echo "[1] Checking HAProxy..."
curl -sI https://auth.example.com | head -5
echo ""

echo "[2] Checking Keycloak Health..."
curl -s https://auth.example.com/health/ready | jq .status
echo ""

echo "[3] Checking OIDC Discovery..."
ISSUER=$(curl -s https://auth.example.com/realms/kubernetes/.well-known/openid-configuration | jq -r .issuer)
echo "Issuer: $ISSUER"
echo ""

echo "[4] Testing Token Generation..."
TOKEN=$(curl -s -X POST \
  "https://auth.example.com/realms/kubernetes/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=kubernetes-cluster" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "username=john.admin" \
  -d "password=UserStr0ngP@ss!" \
  -d "scope=openid groups" | jq -r .access_token)

if [ "$TOKEN" != "null" ] && [ -n "$TOKEN" ]; then
  echo "✅ Token generation: SUCCESS"
  echo "Token claims:"
  echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '{preferred_username, groups, iss, aud}'
else
  echo "❌ Token generation: FAILED"
fi
echo ""

echo "[5] Checking API Server OIDC Configuration..."
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml 2>/dev/null | grep -c "oidc-issuer-url"
if [ $? -eq 0 ]; then
  echo "✅ API Server OIDC flags: CONFIGURED"
else
  echo "❌ API Server OIDC flags: NOT FOUND"
fi
echo ""

echo "[6] Checking RBAC Bindings..."
kubectl get clusterrolebindings -l app.kubernetes.io/managed-by=keycloak-oidc -o custom-columns=NAME:.metadata.name,ROLE:.roleRef.name,SUBJECT:.subjects[0].name
echo ""

echo "[7] Checking SSL Certificate Expiry..."
echo | openssl s_client -connect auth.example.com:443 -servername auth.example.com 2>/dev/null | openssl x509 -noout -dates
echo ""

echo "============================================"
echo " Validation Complete"
echo "============================================"
```

---

## 11. Troubleshooting

### 11.1 Common Issues and Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| API Server can't reach Keycloak | `error: You must be logged in` | Verify DNS/connectivity from control plane nodes to `auth.example.com:443` |
| Groups not in token | RBAC denies access even with correct group | Verify `groups` client scope is assigned to client as "Default" |
| Certificate errors | `x509: certificate signed by unknown authority` | Ensure Let's Encrypt CA is trusted, or use `--oidc-ca-file` |
| Token expired | `Unauthorized` after some time | Configure token refresh in kubelogin; check Keycloak token lifespan |
| HAProxy 503 | `503 Service Unavailable` | Check Keycloak backend health: `curl http://<keycloak-ip>:8080/health/ready` |
| Redirect URI mismatch | `Invalid redirect_uri` error in browser | Ensure `http://localhost:8000/*` is in the client's Valid Redirect URIs |
| Username prefix mismatch | RBAC doesn't match users/groups | Ensure `oidc:` prefix is used in all RBAC subjects |

### 11.2 Debug Commands

```bash
# --- HAProxy ---
# Check HAProxy status
sudo systemctl status haproxy
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# View HAProxy logs
sudo journalctl -u haproxy -f
sudo tail -f /var/log/haproxy.log

# Check HAProxy stats
curl http://localhost:8404/stats

# --- Keycloak ---
# Check Keycloak status and logs
sudo systemctl status keycloak
sudo journalctl -u keycloak -f --since "5 minutes ago"
sudo tail -f /opt/keycloak/data/log/keycloak.log

# Check Keycloak health endpoints
curl -s http://localhost:8080/health/ready | jq .
curl -s http://localhost:8080/health/live | jq .
curl -s http://localhost:8080/metrics

# --- Kubernetes API Server ---
# Check API server logs for OIDC errors
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i "oidc"
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i "error"

# Verify OIDC flags are loaded
ps aux | grep kube-apiserver | grep oidc

# --- SSL/TLS ---
# Check certificate chain
openssl s_client -connect auth.example.com:443 -servername auth.example.com -showcerts </dev/null

# Check certificate expiry
echo | openssl s_client -connect auth.example.com:443 2>/dev/null | openssl x509 -noout -enddate

# --- Token Debugging ---
# Decode a JWT token manually
echo "<JWT_TOKEN>" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .

# Test token against Kubernetes API directly
curl -k -H "Authorization: Bearer ${ID_TOKEN}" \
  https://<K8S_API_SERVER>:6443/api/v1/namespaces
```

### 11.3 Keycloak Session and Token Tuning

Navigate to: **Realm Settings → Tokens**

| Setting | Recommended Value | Description |
|---------|-------------------|-------------|
| Access Token Lifespan | 5 minutes | Short-lived for security |
| SSO Session Idle | 30 minutes | Session timeout |
| SSO Session Max | 10 hours | Maximum session length |
| Client Session Idle | 15 minutes | Client-specific idle |
| Offline Session Idle | 30 days | For refresh tokens |

---

## 12. Maintenance & Backup

### 12.1 Keycloak Realm Export/Backup

```bash
# Export realm configuration (run on Keycloak server)
/opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/export \
  --realm kubernetes \
  --users realm_file

# Or via Admin REST API
ACCESS_TOKEN=$(curl -s -X POST \
  "https://auth.example.com/realms/master/protocol/openid-connect/token" \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=Admin@Str0ng_P@ssw0rd!" \
  | jq -r .access_token)

curl -s -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://auth.example.com/admin/realms/kubernetes" | jq . > kubernetes-realm-backup.json
```

### 12.2 Database Backup

```bash
# PostgreSQL backup
sudo -u postgres pg_dump keycloak > /backup/keycloak_db_$(date +%F_%H%M).sql

# Automated backup via cron
sudo tee /etc/cron.d/keycloak-backup << 'EOF'
# Daily Keycloak database backup at 2:00 AM
0 2 * * * postgres pg_dump keycloak | gzip > /backup/keycloak_db_$(date +\%F).sql.gz
# Keep only last 30 days
0 3 * * * root find /backup/ -name "keycloak_db_*.sql.gz" -mtime +30 -delete
EOF
```

### 12.3 SSL Certificate Monitoring

```bash
# Create a monitoring script
sudo tee /usr/local/bin/check-ssl-expiry.sh << 'SCRIPT'
#!/bin/bash
DOMAIN="auth.example.com"
EXPIRY_DATE=$(echo | openssl s_client -connect ${DOMAIN}:443 -servername ${DOMAIN} 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "${EXPIRY_DATE}" +%s)
CURRENT_EPOCH=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_EPOCH - CURRENT_EPOCH) / 86400 ))

echo "SSL Certificate for ${DOMAIN}: ${DAYS_LEFT} days remaining (expires: ${EXPIRY_DATE})"

if [ ${DAYS_LEFT} -lt 14 ]; then
  echo "⚠️  WARNING: Certificate expires in less than 14 days!"
  # Add alerting here (email, Slack, PagerDuty, etc.)
fi
SCRIPT

sudo chmod +x /usr/local/bin/check-ssl-expiry.sh

# Add to cron (daily check)
echo "0 9 * * * root /usr/local/bin/check-ssl-expiry.sh >> /var/log/ssl-check.log 2>&1" | \
  sudo tee /etc/cron.d/ssl-expiry-check
```

### 12.4 Keycloak Upgrade Procedure

```bash
# 1. Backup everything first
sudo -u postgres pg_dump keycloak > /backup/keycloak_pre_upgrade_$(date +%F).sql
cp -r /opt/keycloak/conf /backup/keycloak_conf_$(date +%F)

# 2. Stop Keycloak
sudo systemctl stop keycloak

# 3. Download new version
NEW_VERSION="26.0.0"
cd /tmp
wget https://github.com/keycloak/keycloak/releases/download/${NEW_VERSION}/keycloak-${NEW_VERSION}.tar.gz

# 4. Extract alongside old installation
sudo tar -xzf keycloak-${NEW_VERSION}.tar.gz -C /opt/
sudo mv /opt/keycloak /opt/keycloak-old
sudo mv /opt/keycloak-${NEW_VERSION} /opt/keycloak

# 5. Copy configuration
sudo cp /opt/keycloak-old/conf/keycloak.conf /opt/keycloak/conf/
sudo cp -r /opt/keycloak-old/data /opt/keycloak/

# 6. Set ownership
sudo chown -R keycloak:keycloak /opt/keycloak

# 7. Build optimized
sudo -u keycloak /opt/keycloak/bin/kc.sh build

# 8. Start and verify
sudo systemctl start keycloak
sudo systemctl status keycloak
curl -s https://auth.example.com/health/ready | jq .
```

---

## Quick Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║                    QUICK REFERENCE CARD                        ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                ║
║  Keycloak Admin Console:                                       ║
║    URL:   https://auth.example.com/admin                       ║
║    Realm: kubernetes                                           ║
║                                                                ║
║  OIDC Discovery:                                               ║
║    https://auth.example.com/realms/kubernetes/                  ║
║    .well-known/openid-configuration                            ║
║                                                                ║
║  Client ID:     kubernetes-cluster                             ║
║  Issuer URL:    https://auth.example.com/realms/kubernetes     ║
║                                                                ║
║  RBAC Group Mapping:                                           ║
║    Keycloak Group    → K8s Group (with prefix)                 ║
║    cluster-admins    → oidc:cluster-admins                     ║
║    developers        → oidc:developers                         ║
║    viewers           → oidc:viewers                            ║
║    sre-team          → oidc:sre-team                           ║
║                                                                ║
║  Key Paths:                                                    ║
║    Keycloak:  /opt/keycloak/                                   ║
║    Config:    /opt/keycloak/conf/keycloak.conf                 ║
║    HAProxy:   /etc/haproxy/haproxy.cfg                         ║
║    SSL Cert:  /etc/haproxy/certs/auth.example.com.pem          ║
║    LE Certs:  /etc/letsencrypt/live/auth.example.com/          ║
║                                                                ║
║  Service Commands:                                             ║
║    sudo systemctl {start|stop|restart|status} keycloak         ║
║    sudo systemctl {start|stop|restart|status} haproxy          ║
║    sudo journalctl -u keycloak -f                              ║
║    sudo certbot renew --dry-run                                ║
║                                                                ║
╚══════════════════════════════════════════════════════════════════╝
```

---

**Document Review:**

| Reviewer | Role | Date | Status |
|----------|------|------|--------|
| | Sr. DevOps Engineer | | Pending |
| | Security Team | | Pending |
| | Platform Lead | | Pending |

**Change Log:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | - | DevOps Team | Initial draft |
| 2.0 | July 2025 | DevOps Team | Production-ready with RBAC, backup, monitoring |