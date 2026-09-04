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

---

## PostgreSQL Support (Experimental)

### Current Status (2026)

**Important**: PostgreSQL support for ERPNext is **NOT production-ready**.

| Component | PostgreSQL Support |
|-----------|-------------------|
| Frappe Framework | ✅ Supported |
| ERPNext | ❌ Not officially supported |
| HRMS | ❌ Not officially supported |
| Custom Apps | ⚠️ With caveats |

**What works:**
- Frappe Framework core functionality
- Basic CRUD operations
- Custom doctypes (simple)

**What doesn't work:**
- ERPNext modules (accounts, stock, buying, etc.)
- Transaction isolation issues
- Complex queries with JOINs
- Some MariaDB-specific SQL syntax

**Expected timeline:**
- PostgreSQL support may come in ERPNext v17
- Not a priority for Frappe team currently
- Community efforts ongoing but not production-ready

### Recommendation

**Use MariaDB for production ERPNext deployments.**

PostgreSQL is recommended only for:
- Development/testing environments
- Custom Frappe apps (without ERPNext)
- Future-proofing (when support arrives)

---

## PostgreSQL Configuration for Custom Apps

If you're building **custom Frappe apps** (without ERPNext modules), you can use PostgreSQL:

### 1. Install PostgreSQL

```bash
# Install PostgreSQL 15
sudo apt install postgresql postgresql-contrib -y

# Start and enable
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database user
sudo -u postgres psql
CREATE USER frappe WITH PASSWORD 'your_password';
CREATE DATABASE frappe OWNER frappe;
\q
```

### 2. Configure Frappe for PostgreSQL

When initializing bench with PostgreSQL:

```bash
# Initialize bench with PostgreSQL
bench init --frappe-branch version-16 --db-type postgres frappe-bench

# Or for existing bench, create site with PostgreSQL
bench new-site pg-site.localhost \
  --db-type postgres \
  --db-host localhost \
  --db-port 5432 \
  --db-name frappe \
  --db-password your_password \
  --admin-password admin
```

### 3. Site Configuration

Edit `site_config.json`:

```json
{
  "db_host": "localhost",
  "db_port": 5432,
  "db_name": "frappe",
  "db_password": "your_password",
  "db_type": "postgres"
}
```

### 4. Custom App with PostgreSQL

When developing custom apps:

```python
# hooks.py - Use multisql for database-specific queries

import frappe

def get_data():
    # Use frappe.db.multisql for database-specific queries
    return frappe.db.multisql({
        "mariadb": "SELECT * FROM tabUser WHERE name = %s",
        "postgres": "SELECT * FROM tabUser WHERE name = %s"
    }, (frappe.session.user,), as_dict=True)
```

### 5. Query Compatibility

**Avoid MariaDB-specific syntax:**

```python
# ❌ Bad - MariaDB specific
frappe.db.sql("""
    UPDATE tabVehicle 
    SET status = 'Active'
    JOIN tabDepartment ON tabVehicle.department = tabDepartment.name
""")

# ✅ Good - Database agnostic
frappe.db.set_value("Vehicle", vehicle_name, "status", "Active")

# ✅ Good - Use multisql for complex queries
frappe.db.multisql({
    "mariadb": """
        UPDATE tabVehicle v
        JOIN tabDepartment d ON v.department = d.name
        SET v.status = 'Active'
    """,
    "postgres": """
        UPDATE tabVehicle v
        SET status = 'Active'
        FROM tabDepartment d
        WHERE v.department = d.name
    """
})
```

### 6. Custom App Development Setup

For custom apps without ERPNext:

```bash
# Create app
bench new-app custom_fleet_mgmt

# Initialize with PostgreSQL
bench init --db-type postgres frappe-bench-pg
cd frappe-bench-pg

# Install custom app only (no ERPNext)
bench get-app /path/to/custom_fleet_mgmt
bench new-site dev.localhost --db-type postgres
bench --site dev.localhost install-app custom_fleet_mgmt
```

### 7. Testing Custom Apps on PostgreSQL

```bash
# Run tests with PostgreSQL
bench --site pg-test.localhost run-tests --app custom_fleet_mgmt

# Check for SQL compatibility issues
bench --site pg-test.localhost mariadb
```

### 8. Docker Development Environment

Use community PostgreSQL images for development:

```bash
# Pull PostgreSQL development image
docker pull vyogo/erpnext:sne-postgres-develop

# Run container
docker run -d \
  --name erpnext-pg-dev \
  -p 8000:8000 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=ChangeMe \
  vyogo/erpnext:sne-postgres-develop

# Access: http://localhost:8000
# User: Administrator
# Password: admin
```

### 9. Migration from MariaDB to PostgreSQL

If migrating existing site:

```bash
# Export from MariaDB
bench --site mariadb-site backup --with-files

# Create new PostgreSQL site
bench new-site pg-site.localhost --db-type postgres

# Import data (requires custom migration script)
bench --site pg-site.localhost mariadb
```

**Note**: Full migration requires rewriting MariaDB-specific queries.

### 10. Best Practices for PostgreSQL Compatibility

1. **Use ORM methods** instead of raw SQL:
   ```python
   # Instead of raw SQL
   frappe.get_all("Vehicle", filters={"status": "Active"})
   
   # Instead of INSERT
   frappe.get_doc({"doctype": "Vehicle", ...}).insert()
   ```

2. **Use `frappe.db.multisql`** for database-specific queries

3. **Avoid MariaDB-specific functions**:
   - `GROUP_CONCAT` → Use Python aggregation
   - `IFNULL` → Use `COALESCE`
   - `NOW()` → Use `frappe.utils.now_datetime()`

4. **Test on both databases** during development

---

## Summary: MariaDB vs PostgreSQL

| Feature | MariaDB | PostgreSQL |
|---------|---------|------------|
| ERPNext Support | ✅ Full | ❌ Not supported |
| Production Ready | ✅ Yes | ❌ No |
| Custom Apps | ✅ Yes | ⚠️ Limited |
| Performance | Good | Better (complex queries) |
| Community Support | ✅ Official | ⚠️ Community |
| Recommended For | Production ERPNext | Custom apps only |

**Final Recommendation**: Use MariaDB for this project. PostgreSQL support is coming but not ready for production ERPNext deployments.
