# BMI Health Tracker — Manual Deployment Guide

> Complete step-by-step guide to deploy the BMI Health Tracker on AWS EC2 Ubuntu **without any automation scripts**.
> Every step mirrors exactly what `IMPLEMENTATION_AUTO.sh` does internally.
> For post-deploy updates use `AppUpdate_AUTO.sh`.

---

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [Prerequisites](#2-prerequisites)
3. [Part A — AWS Console Setup](#part-a--aws-console-setup)
4. [Part B — Server Bootstrap](#part-b--server-bootstrap)
5. [Step 1 — Install System Dependencies](#step-1--install-system-dependencies)
6. [Step 2 — Setup Database](#step-2--setup-database)
7. [Step 3 — Create Backend Runtime Directory and .env](#step-3--create-backend-runtime-directory-and-env)
8. [Step 4 — Backup Existing Deployment](#step-4--backup-existing-deployment)
9. [Step 5 — Deploy Backend](#step-5--deploy-backend)
10. [Step 6 — Deploy Frontend](#step-6--deploy-frontend)
11. [Step 7 — Configure PM2](#step-7--configure-pm2)
12. [Step 8 — Configure Nginx](#step-8--configure-nginx)
13. [Step 9 — SSL / HTTPS Setup (Optional)](#step-9--ssl--https-setup-optional)
14. [Step 10 — Health Checks](#step-10--health-checks)
15. [Useful Day-2 Commands](#useful-day-2-commands)
16. [Troubleshooting](#troubleshooting)

---

## 1. Application Overview

**BMI Health Tracker** — a 3-tier full-stack web application.

| Layer    | Technology                     | Runs on        |
|----------|--------------------------------|----------------|
| Frontend | React 18 + Vite 5 + Chart.js 4 | Nginx (static) |
| Backend  | Node.js 18 LTS + Express 4     | PM2 port 3000  |
| Database | PostgreSQL 14                  | localhost:5432 |

Traffic flow: `Browser → Nginx :80/:443 → /api/* proxy → Node :3000 → PostgreSQL`

### Key paths used throughout this guide

| Path | Purpose |
|------|---------|
| `~/single-server-3tier-webapp/` | Git repository (source of truth) |
| `/opt/bmi-app/backend/` | Backend runtime directory |
| `/var/www/bmi-health-tracker/` | Frontend static files served by Nginx |
| `~/bmi_deployments_backup/` | Timestamped deployment backups |
| `/etc/nginx/sites-available/bmi-health-tracker` | Nginx virtual host config |
| `/opt/bmi-app/backend/.env` | DB credentials and server config |

---

## 2. Prerequisites

### What you need before starting

| Item | Detail |
|------|--------|
| AWS account | EC2 launch permissions |
| EC2 key pair | `.pem` file on your local machine |
| Ubuntu 22.04 LTS instance | `t2.micro` or larger |
| Security group rules | Ports 22, 80, 443 open inbound |
| Domain (optional) | Required only for SSL — A record pointing to EC2 IP |

---

## Part A — AWS Console Setup

*Do this once in the AWS Console before SSHing to the server.*

### A.1 Launch EC2 Instance

1. Go to **EC2 → Launch Instance**
2. Name: `bmi-health-tracker`
3. AMI: **Ubuntu Server 22.04 LTS (HVM), SSD Volume Type**
4. Instance type: `t2.micro` (free tier) or `t3.small`
5. Key pair: select or create a `.pem` key pair — save it securely
6. Storage: 20 GB gp3
7. Click **Launch Instance**

### A.2 Configure Security Group

| Type  | Protocol | Port | Source    | Purpose         |
|-------|----------|------|-----------|-----------------|
| SSH   | TCP      | 22   | Your IP   | Admin access    |
| HTTP  | TCP      | 80   | 0.0.0.0/0 | Web traffic     |
| HTTPS | TCP      | 443  | 0.0.0.0/0 | SSL web traffic |

### A.3 (Optional) Elastic IP

Assign a static IP so it does not change after reboots:
**EC2 → Elastic IPs → Allocate → Associate → select your instance**

### A.4 Connect via SSH

```bash
# Set key permissions (first time only)
chmod 400 your-key.pem

# Connect
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

---

## Part B — Server Bootstrap

*Run once on the server immediately after first SSH login.*

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget unzip build-essential

# Clone the repository
cd ~
git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp
```

---

## Step 1 — Install System Dependencies

### 1.1 Install NVM and Node.js LTS

```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load NVM into the current session
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# Install Node.js LTS
nvm install --lts
nvm use --lts
nvm alias default lts/*

# Verify
node -v   # e.g. v20.x.x
npm -v    # e.g. 10.x.x
```

> NVM is automatically added to `~/.bashrc` — it loads on every new SSH session.

### 1.2 Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify
sudo systemctl status postgresql
```

### 1.3 Install Nginx

```bash
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify
sudo systemctl status nginx
```

### 1.4 Install PM2

```bash
npm install -g pm2

# Verify
pm2 -v
```

---

## Step 2 — Setup Database

### 2.1 Create the database user and database

```bash
# Create application user with password
sudo -u postgres psql -c "CREATE USER bmi_user WITH PASSWORD 'your_password';"

# Create the database
sudo -u postgres psql -c "CREATE DATABASE bmidb;"

# Grant full privileges on the database
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE bmidb TO bmi_user;"

# Grant schema privileges (needed for PostgreSQL 15+)
sudo -u postgres psql -d bmidb -c "GRANT ALL ON SCHEMA public TO bmi_user;"
sudo -u postgres psql -d bmidb -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO bmi_user;"
```

> Replace `your_password` with your chosen password. Use the same value everywhere in this guide.

### 2.2 Configure PostgreSQL password authentication

```bash
# Find the pg_hba.conf file path
PG_HBA=$(sudo -u postgres psql -t -P format=unaligned -c 'SHOW hba_file')
echo $PG_HBA
# Typically: /etc/postgresql/14/main/pg_hba.conf

# Backup before editing
sudo cp "$PG_HBA" "${PG_HBA}.backup"

# Add md5 auth rule for bmi_user on bmidb
sudo sed -i "/^# IPv4 local connections:/a host    bmidb    bmi_user    127.0.0.1/32    md5" "$PG_HBA"

# Reload PostgreSQL to apply
sudo systemctl reload postgresql
```

### 2.3 Test the connection

```bash
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost -c "SELECT 1;"
# Expected: (1 row)
# If this fails, check Step 2.2 and that the password matches
```

---

## Step 3 — Create Backend Runtime Directory and .env

### 3.1 Create the runtime directory

```bash
# Create /opt/bmi-app/backend and give the current user ownership
sudo mkdir -p /opt/bmi-app/backend
sudo chown -R $USER:$USER /opt/bmi-app
```

### 3.2 Create the .env file

```bash
cat > /opt/bmi-app/backend/.env << EOF
# Database Configuration
DATABASE_URL=postgresql://bmi_user:your_password@localhost:5432/bmidb

# Alternative individual settings
DB_USER=bmi_user
DB_PASSWORD=your_password
DB_NAME=bmidb
DB_HOST=localhost
DB_PORT=5432

# Server Configuration
PORT=3000
NODE_ENV=production

# CORS Configuration (update with your domain if needed)
CORS_ORIGIN=*
EOF

# Restrict permissions — readable only by the current user
chmod 600 /opt/bmi-app/backend/.env

# Verify content
cat /opt/bmi-app/backend/.env
```

> Replace `your_password` with the same password used in Step 2.

---

## Step 4 — Backup Existing Deployment

*Skip on a fresh server where nothing is deployed yet.*

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$HOME/bmi_deployments_backup/deployment_$TIMESTAMP"
mkdir -p "$BACKUP_PATH"

# Backup backend .env (credentials must survive redeploys)
cp /opt/bmi-app/backend/.env "$BACKUP_PATH/" 2>/dev/null || echo "No .env to backup"

# Backup deployed frontend
sudo cp -r /var/www/bmi-health-tracker "$BACKUP_PATH/frontend_deployed" 2>/dev/null || echo "No frontend to backup"

# Backup Nginx config
sudo cp /etc/nginx/sites-available/bmi-health-tracker "$BACKUP_PATH/" 2>/dev/null || echo "No Nginx config to backup"

# Backup database
PGPASSWORD=your_password pg_dump -U bmi_user -h localhost bmidb \
  > "$BACKUP_PATH/database_backup.sql" 2>/dev/null || echo "Database backup skipped"

echo "Backup saved to: $BACKUP_PATH"

# Keep only the last 5 backups — remove older ones
cd "$HOME/bmi_deployments_backup"
ls -t | tail -n +6 | xargs -r rm -rf
```

---

## Step 5 — Deploy Backend

### 5.1 Sync backend source from git repo to runtime directory

```bash
# rsync copies backend/ into /opt/bmi-app/backend/
# --exclude ensures .env, node_modules, and logs are never overwritten
rsync -a \
  --exclude 'node_modules' \
  --exclude '.env' \
  --exclude 'logs' \
  ~/single-server-3tier-webapp/backend/ \
  /opt/bmi-app/backend/

# Confirm .env is still present (rsync must NOT have removed it)
ls -la /opt/bmi-app/backend/.env
```

### 5.2 Install backend dependencies

```bash
cd /opt/bmi-app/backend
npm install --production
```

### 5.3 Run database migrations in order

```bash
cd /opt/bmi-app/backend

# Migration 001 — creates the measurements table
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -f migrations/001_create_measurements.sql

# Migration 002 — adds measurement_date column
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -f migrations/002_add_measurement_date.sql

# Verify the final table structure
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -c "\d measurements"
```

### 5.4 Confirm database connection from the runtime directory

```bash
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost -c "SELECT 1;"
# Must return: (1 row) — if not, stop and fix Step 2 before continuing
```

---

## Step 6 — Deploy Frontend

### 6.1 Build the React app

```bash
cd ~/single-server-3tier-webapp/frontend
npm install
npm run build

# Verify the build output exists
ls -la dist/
# Must show index.html — if missing, the build failed
```

### 6.2 Copy the build to the Nginx directory

```bash
sudo mkdir -p /var/www/bmi-health-tracker

# Remove stale files and deploy the fresh build
sudo rm -rf /var/www/bmi-health-tracker/*
sudo cp -r dist/* /var/www/bmi-health-tracker/

# Set correct ownership and permissions for Nginx
sudo chown -R www-data:www-data /var/www/bmi-health-tracker
sudo chmod -R 755 /var/www/bmi-health-tracker

# Critical check — missing index.html causes a 500 redirect loop in Nginx
ls -la /var/www/bmi-health-tracker/
# Must show index.html
```

---

## Step 7 — Configure PM2

### 7.1 Stop any existing bmi-backend process

```bash
pm2 stop bmi-backend 2>/dev/null || true
pm2 delete bmi-backend 2>/dev/null || true
```

### 7.2 Start the backend with PM2

```bash
cd /opt/bmi-app/backend
pm2 start src/server.js --name bmi-backend --env production

# Verify — status must show: online
pm2 status
```

### 7.3 Save the PM2 process list

```bash
pm2 save
```

### 7.4 Configure PM2 to auto-start on server reboot

```bash
# Generate the systemd startup command
pm2 startup systemd -u ubuntu --hp /home/ubuntu

# PM2 prints a command like:
#   sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/vXX.X.X/bin ...
# Copy that exact command and run it, then save again:
pm2 save

# Verify
sudo systemctl status pm2-ubuntu
```

---

## Step 8 — Configure Nginx

### 8.1 Get your EC2 public IP

```bash
# IMDSv2 (AWS recommended method)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
EC2_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/public-ipv4)
echo $EC2_IP
```

### 8.2 Write the Nginx configuration

Replace `YOUR_SERVER_NAME` with the IP printed above, or your domain name.

```bash
sudo tee /etc/nginx/sites-available/bmi-health-tracker > /dev/null << 'EOF'
server {
    listen 80;
    listen [::]:80;

    server_name YOUR_SERVER_NAME;

    # Frontend static files
    root /var/www/bmi-health-tracker;
    index index.html;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript
               application/x-javascript application/xml+rss
               application/javascript application/json;

    # Frontend routing — serves index.html for all React Router paths
    location / {
        try_files $uri $uri/ /index.html;

        # Cache static assets for 1 year
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # API reverse proxy — forwards /api/* to Node.js on port 3000
    location /api/ {
        proxy_pass http://127.0.0.1:3000/api/;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        proxy_cache_bypass $http_upgrade;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    server_tokens off;

    access_log /var/log/nginx/bmi-access.log;
    error_log /var/log/nginx/bmi-error.log;
}
EOF
```

### 8.3 Enable the site and restart Nginx

```bash
# Enable site via symlink
sudo ln -sf /etc/nginx/sites-available/bmi-health-tracker \
            /etc/nginx/sites-enabled/bmi-health-tracker

# Remove the default Nginx placeholder site
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration syntax — must pass before restarting
sudo nginx -t

# Restart Nginx and enable on boot
sudo systemctl restart nginx
sudo systemctl enable nginx

# Verify
sudo systemctl status nginx
```

---

## Step 9 — SSL / HTTPS Setup (Optional)

> **Requirement**: domain A record must resolve to the EC2 public IP **before** running Certbot.
> Let's Encrypt validates ownership via HTTP on port 80.

### 9.1 Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 9.2 Update the Nginx server_name with your domain

```bash
sudo sed -i "s/server_name .*/server_name yourdomain.com;/" \
  /etc/nginx/sites-available/bmi-health-tracker
sudo nginx -t && sudo systemctl reload nginx
```

### 9.3 Request the certificate

```bash
# --redirect automatically configures HTTP → HTTPS redirect
sudo certbot --nginx -d yourdomain.com \
  --non-interactive --agree-tos \
  --email your@email.com \
  --redirect
```

### 9.4 Verify auto-renewal

```bash
sudo certbot renew --dry-run
sudo systemctl status certbot.timer
```

---

## Step 10 — Health Checks

```bash
# Give services 3 seconds to stabilise
sleep 3

# 1. Backend API responds directly on port 3000
curl -f http://localhost:3000/api/measurements
# Expected: JSON array (empty [] on fresh install is correct)

# 2. Nginx serves the frontend (expect: 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost/

# 3. API proxy through Nginx works (expect: 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost/api/measurements

# 4. PM2 process is online
pm2 status
# bmi-backend row must show: online

# 5. Database row count
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -tAc "SELECT COUNT(*) FROM measurements;"

# 6. Open in browser
# http://<EC2-PUBLIC-IP>
```

---

## Useful Day-2 Commands

```bash
# --- PM2 / Backend ---
pm2 status                           # Show all processes
pm2 logs bmi-backend                 # Stream logs live
pm2 logs bmi-backend --lines 50      # Last 50 lines
pm2 restart bmi-backend              # Restart
pm2 reload bmi-backend               # Zero-downtime reload
pm2 stop bmi-backend                 # Stop

# --- Nginx ---
sudo nginx -t                        # Test config syntax
sudo systemctl reload nginx          # Reload config (no downtime)
sudo systemctl restart nginx         # Full restart
sudo tail -f /var/log/nginx/bmi-error.log
sudo tail -f /var/log/nginx/bmi-access.log

# --- PostgreSQL ---
sudo systemctl status postgresql
psql -U bmi_user -d bmidb -h localhost   # App user shell (prompts password)
sudo -u postgres psql                    # Admin shell

# --- SSL ---
sudo certbot certificates            # Show cert status and expiry
sudo certbot renew                   # Force renewal
sudo certbot renew --dry-run         # Test renewal without changes

# --- Firewall ---
sudo ufw status
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH

# --- System ---
df -h                                # Disk usage
free -h                              # Memory usage
```

---

## Troubleshooting

### `SASL: client password must be a string`

`DB_PASSWORD` in `/opt/bmi-app/backend/.env` is empty or missing.

```bash
nano /opt/bmi-app/backend/.env
# Set: DB_PASSWORD=yourpassword
pm2 restart bmi-backend
```

---

### `ECONNREFUSED 127.0.0.1:5432`

PostgreSQL is not running.

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

---

### Nginx 502 Bad Gateway

Backend is down.

```bash
pm2 status
pm2 logs bmi-backend --lines 20
# If stopped:
cd /opt/bmi-app/backend
pm2 start src/server.js --name bmi-backend --env production
```

---

### Nginx 404 on React frontend routes

The `try_files` directive is missing from the Nginx config.

```bash
sudo grep "try_files" /etc/nginx/sites-available/bmi-health-tracker
```

If missing, re-do Step 8.

---

### Nginx rewrite/redirection cycle (500 error)

`index.html` is missing from `/var/www/bmi-health-tracker/` — Nginx loops trying to serve it.

```bash
ls -la /var/www/bmi-health-tracker/
# If empty or missing index.html — re-run Step 6
```

---

### Re-run migrations (idempotent — safe to re-apply)

```bash
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -f /opt/bmi-app/backend/migrations/001_create_measurements.sql
PGPASSWORD=your_password psql -U bmi_user -d bmidb -h localhost \
  -f /opt/bmi-app/backend/migrations/002_add_measurement_date.sql
```

---

### PM2 not auto-starting after reboot

```bash
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# Copy and run the exact command it prints, then:
pm2 save
```

---

### SSL certificate request fails

- Verify DNS: `nslookup yourdomain.com` — must return EC2 public IP
- Verify ports 80 and 443 are open in the EC2 Security Group
- Check Certbot logs: `sudo journalctl -u certbot`

---

**Last Updated**: April 18, 2026
**Version**: 4.1
**Replaces**: IMPLEMENTATION_GUIDE.md (v2.1) + IMPLEMENTATION_GUIDE_ORDER.md (v3.0)

---

*MD Sarowar Alam*
Lead DevOps Engineer, WPP Production
📧 Email: sarowar@hotmail.com
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/