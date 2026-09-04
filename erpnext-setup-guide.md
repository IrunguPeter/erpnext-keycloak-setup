# ERPNext v16 Installation Guide for Linux Ubuntu 24.04

## Overview
This guide walks you through installing ERPNext v16 on Ubuntu 24.04 LTS. ERPNext is a comprehensive ERP solution built on the Frappe framework.

## Prerequisites
- **OS**: Ubuntu 24.04 LTS
- **RAM**: 4GB minimum (8GB recommended)
- **Storage**: 40GB+ SSD
- **Access**: SSH root access or sudo privileges

---

## Step 1: System Preparation

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Set correct timezone
sudo timedatectl set-timezone UTC
```

## Step 2: Create Frappe User

```bash
# Create dedicated user (never run ERPNext as root)
sudo adduser frappe
sudo usermod -aG sudo frappe
sudo su - frappe
```

## Step 3: Install System Dependencies

```bash
# As frappe user, install required packages
sudo apt install -y \
  git curl wget \
  python3-dev python3-pip python3-venv \
  build-essential \
  libffi-dev libssl-dev \
  libjpeg-dev zlib1g-dev \
  libmysqlclient-dev pkg-config \
  xvfb libfontconfig fontconfig \
  redis-server \
  software-properties-common
```

## Step 4: Install Node.js 24

```bash
# Install NVM
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

# Install Node.js 24
nvm install 24
nvm use 24

# Install npm and yarn
sudo apt-get install npm -y
sudo npm install -g yarn
```

## Step 5: Install MariaDB

```bash
# Install MariaDB server
sudo apt install mariadb-server mariadb-client -y

# Secure installation
sudo mysql_secure_installation
```

### Configure MariaDB for ERPNext

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under `[mysqld]`:
```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
innodb-file-format = barracuda
innodb-file-per-table = 1
innodb-large-prefix = 1
innodb_buffer_pool_size = 2G
wait_timeout = 28800
interactive_timeout = 28800

[mysql]
default-character-set = utf8mb4
```

Restart MariaDB:
```bash
sudo systemctl restart mariadb
```

## Step 6: Install wkhtmltopdf

```bash
# Download wkhtmltopdf with patched Qt
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb

# Install
sudo apt install -y ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb

# Verify version
wkhtmltopdf --version
```

## Step 7: Install Frappe Bench

```bash
# Install bench CLI
pip3 install frappe-bench

# Add to PATH
export PATH=$PATH:/home/frappe/.local/bin
echo 'export PATH=$PATH:/home/frappe/.local/bin' >> ~/.bashrc
```

## Step 8: Initialize Frappe Bench

```bash
# Initialize bench with Frappe v16
bench init --frappe-branch version-16 frappe-bench

# Navigate to bench directory
cd frappe-bench
```

## Step 9: Create ERPNext Site

```bash
# Replace with your domain or IP
bench new-site erp.yourdomain.com \
  --mariadb-root-password YOUR_DB_ROOT_PASSWORD \
  --admin-password YOUR_ADMIN_PASSWORD \
  --no-mariadb-socket
```

## Step 10: Install ERPNext

```bash
# Download ERPNext
bench get-app --branch version-16 erpnext

# Install on site
bench --site erp.yourdomain.com install-app erpnext

# Optionally install HRMS
bench get-app --branch version-16 hrms
bench --site erp.yourdomain.com install-app hrms
```

## Step 11: Build Frontend Assets

```bash
bench build
```

## Step 12: Production Setup

```bash
# Exit frappe user first
exit

# As root, setup production
sudo apt install -y ansible nginx supervisor
sudo env "PATH=$PATH" bench setup production frappe

# Setup nginx
bench setup nginx
sudo nginx -t
sudo systemctl reload nginx

# Restart all services
sudo supervisorctl restart all
sudo supervisorctl status
```

## Step 13: SSL Certificate (Optional)

```bash
# Install certbot
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Get SSL certificate
sudo certbot --nginx -d erp.yourdomain.com
```

## Step 14: Firewall Configuration

```bash
sudo ufw allow 22,25,143,80,443,3306,3022,8000/tcp
sudo ufw enable
```

## Step 15: Access ERPNext

Open browser and navigate to:
```
http://erp.yourdomain.com
```

Login with:
- **Username**: Administrator
- **Password**: (your admin password)

---

## Useful Commands

```bash
# Start development server
bench start

# Clear cache
bench --site erp.yourdomain.com clear-cache

# Backup site
bench --site erp.yourdomain.com backup

# Update ERPNext
bench update

# Check bench status
bench doctor
```

## Troubleshooting

### Common Issues

1. **Port 80 already in use**: Stop nginx/apache if running
2. **MariaDB connection error**: Check if service is running
3. **Build fails with OOM**: Increase swap space
4. **PDF generation issues**: Verify wkhtmltopdf version

### Logs Location
- Nginx: `/var/log/nginx/`
- Supervisor: `/var/log/supervisor/`
- MariaDB: `/var/log/mysql/`

---

## Next Steps

After installation, you can:
1. Configure Keycloak for SSO (see Keycloak setup guide)
2. Install custom HR and payroll modules
3. Configure users and permissions
4. Set up email integration
5. Configure backup automation
