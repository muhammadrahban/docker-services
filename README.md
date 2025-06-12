# Docker Service Platform - Complete Setup Guide

This document provides a comprehensive step-by-step guide for setting up a complete Docker-based development platform with Traefik reverse proxy, SSL certificates, and multiple services.

**NOTE: You need to add your SSH key into GitHub first to access all repos shared with you.**

---

## 🚀 Services Included

- **Traefik** - Reverse proxy with SSL termination
- **PostgreSQL** - Database server
- **PgAdmin** - PostgreSQL administration interface
- **MySQL** - Alternative database server
- **phpMyAdmin** - MySQL administration interface
- **MinIO** - S3-compatible object storage
- **Mailpit** - Email testing tool

---

## 📋 Prerequisites

### Fresh Ubuntu 24.04 LTS

It is recommended to use fresh install of Ubuntu 24.04 LTS or later.

### Docker Installation

You will need to install Docker and Docker Compose plugin natively on Ubuntu machine and not use Docker Desktop app. Use the guide below from official docker page.

[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

After that add your user to docker group so you can run docker commands without sudo.

```shell
sudo usermod -aG docker $USER
```

**Logout and login again** to apply group changes.

Check to see if Docker and Docker Compose is installed properly by running:

```shell
docker --version
docker compose version
```

### PHP, Composer & Node.js

You will need to install PHP 8.3.X for better handling of auto-complete in VS Code and other useful PHP tools.

Use the following command to install PHP, Composer and other PHP extensions:

```shell
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.3)"
```

Run following to verify correct version of PHP and Composer:

```shell
php -v
composer --version
```

Now install LTS version of Node.js (recommended 20.x) on your machine using Node.js official guide with node version manager (nvm) method.

[https://nodejs.org/en/download](https://nodejs.org/en/download)

**NOTE:** you will need to run Node.js LTS recommended steps one by one to properly install Node.js via node version manager (nvm).

Once Node.js is installed make sure to include recommended paths in your .bashrc or .zshrc file and reload your profile using `source` command.

Example of lines to add to .zshrc:

```shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

Verify Node.js and npm by running following commands:

```shell
node -v
npm -v
```

---

## 🏗️ Platform Setup

### 1. Create Main Directory Structure

Create main directory in your home folder:

```shell
mkdir docker-service && cd docker-service
```

### 2. Clone All Service Repositories

Clone each service repository into the docker-service directory:

```shell
# Clone Traefik reverse proxy
git clone git@github.com:yourorg/server-reverse-proxy server-reverse-proxy

# Clone database services
git clone git@github.com:yourorg/postgresql postgresql
git clone git@github.com:yourorg/mysql mysql

# Clone admin interfaces
git clone git@github.com:yourorg/pgadmin pgadmin
git clone git@github.com:yourorg/phpmyadmin phpmyadmin

# Clone storage and email services
git clone git@github.com:yourorg/minio minio
git clone git@github.com:yourorg/mailpit mailpit
```

**Alternative:** If SSH is not configured, use HTTPS:

```shell
git clone https://github.com/yourorg/server-reverse-proxy server-reverse-proxy
# ... repeat for other repositories
```

### 3. Create Docker Network

Create the shared reverse-proxy network:

```shell
docker network create reverse-proxy
```

---

## 🔐 SSL Certificate Setup

### Install mkcert for Trusted Local SSL Certificates

```shell
# Install mkcert
sudo apt update && sudo apt install mkcert

# Install local CA
mkcert -install
```

### Generate Certificates for All Services

```shell
# Navigate to Traefik directory
cd ~/docker-service/server-reverse-proxy

# Create certificates directory
mkdir -p container-data/certificates

# Generate certificates for all services
mkcert \
  traefik.network.local.com \
  pga.network.local.com \
  pma.network.local.com \
  s3.network.local.com \
  s3api.network.local.com \
  mail.network.local.com \
  "*.network.local.com"

# Move certificates to correct location
mv traefik.network.local.com+6.pem container-data/certificates/selfsigned.crt
mv traefik.network.local.com+6-key.pem container-data/certificates/selfsigned.key
```

---

## ⚙️ Service Configuration

### 1. Configure Traefik (Reverse Proxy)

```shell
cd ~/docker-service/server-reverse-proxy

# Set appropriate permissions
sudo chown -R $USER:$USER container-data && sudo chmod -R 755 container-data

# Setup environment file
cp .sample.env .env
```

Edit the `.env` file with your configuration:

```shell
nano .env
```

Required variables:
```env
TRAEFIK_HOSTNAME=traefik.network.local.com
TRAEFIK_DASHBOARD_CREDENTIALS=treafik:$2y$10$... # Generated password hash
TRAEFIK_MAIN_DOMAIN=network.local.com
TRAEFIK_SANS_DOMAIN=*.network.local.com
```

### 2. Configure PostgreSQL

```shell
cd ~/docker-service/postgresql

# Setup environment file
cp .sample.env .env
nano .env
```

Configure PostgreSQL variables:
```env
POSTGRES_DB=development
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
```

### 3. Configure PgAdmin

```shell
cd ~/docker-service/pgadmin

# Setup environment file
cp .sample.env .env
nano .env
```

Configure PgAdmin variables:
```env
TRAEFIK_HOSTNAME=pga.network.local.com
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=your_admin_password
```

### 4. Configure MySQL

```shell
cd ~/docker-service/mysql

# Setup environment file
cp .sample.env .env
nano .env
```

Configure MySQL variables:
```env
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_DATABASE=development
MYSQL_USER=developer
MYSQL_PASSWORD=your_user_password
```

### 5. Configure phpMyAdmin

```shell
cd ~/docker-service/phpmyadmin

# Setup environment file
cp .sample.env .env
nano .env
```

Configure phpMyAdmin variables:
```env
TRAEFIK_HOSTNAME=pma.network.local.com
PMA_HOST=service-mysql
PMA_PORT=3306
MAX_EXECUTION_TIME=600
MEMORY_LIMIT=512M
UPLOAD_LIMIT=128M
```

### 6. Configure MinIO

```shell
cd ~/docker-service/minio

# Setup environment file
cp .sample.env .env
nano .env

# Create data directory
mkdir -p container-data/data
```

Configure MinIO variables:
```env
TRAEFIK_HOSTNAME=s3.network.local.com
TRAEFIK_HOSTNAME_API=s3api.network.local.com
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your_minio_password
```

### 7. Configure Mailpit

```shell
cd ~/docker-service/mailpit

# Setup environment file
cp .sample.env .env
nano .env
```

Configure Mailpit variables:
```env
TRAEFIK_HOSTNAME=mail.network.local.com
MP_MAX_MESSAGES=500
MP_SMTP_AUTH_ALLOW_INSECURE=true
MP_SMTP_AUTH_ACCEPT_ANY=true
```

---

## 🌐 Host Entries Configuration

Update your local hosts file to route traffic to the correct IP:

```shell
sudo nano /etc/hosts
```

Add the following entries:

```bash
# Traefik Services (using 127.0.0.2 to avoid conflicts with nginx on 127.0.0.1)
127.0.0.2        traefik.network.local.com

# Database Admin Interfaces
127.0.0.2        pga.network.local.com
127.0.0.2        pma.network.local.com

# Storage Services
127.0.0.2        s3.network.local.com
127.0.0.2        s3api.network.local.com

# Communication Services
127.0.0.2        mail.network.local.com

# Additional services (add as needed)
127.0.0.2        api.network.local.com
127.0.0.2        web.network.local.com
```

**Note:** Replace `127.0.0.2` with your local WiFi or Ethernet LAN IP address if you encounter any issues.

---

## 🚀 Starting Services

### Start Services in Order

1. **Start Traefik first:**
```shell
cd ~/docker-service/server-reverse-proxy
docker compose up -d --build --force-recreate
```

2. **Start database services:**
```shell
cd ~/docker-service/postgresql
docker compose up -d

cd ~/docker-service/mysql
docker compose up -d
```

3. **Start admin interfaces:**
```shell
cd ~/docker-service/pgadmin
docker compose up -d

cd ~/docker-service/phpmyadmin
docker compose up -d
```

4. **Start other services:**
```shell
cd ~/docker-service/minio
docker compose up -d

cd ~/docker-service/mailpit
docker compose up -d
```

### Verify All Services

Check that all services are running:

```shell
docker ps
```

You should see all containers running without any exit codes.

---

## 🔍 Service Access URLs

Once all services are running, you can access them via HTTPS:

| Service | URL | Credentials |
|---------|-----|-------------|
| **Traefik Dashboard** | https://traefik.network.local.com/dashboard/ | treafik / 12345678 |
| **PgAdmin** | https://pga.network.local.com | admin@example.com / your_admin_password |
| **phpMyAdmin** | https://pma.network.local.com | root / your_root_password |
| **MinIO Console** | https://s3.network.local.com | minioadmin / your_minio_password |
| **MinIO API** | https://s3api.network.local.com | - |
| **Mailpit** | https://mail.network.local.com | No authentication |

### SSL Certificate Status

All services should show a **green lock icon** in your browser, indicating trusted SSL certificates thanks to mkcert.

---

## 🧪 Testing the Platform

### Test Database Connectivity

**PostgreSQL:**
```shell
# Test from host
psql -h 127.0.0.2 -p 5432 -U postgres -d development

# Test via PgAdmin at https://pga.network.local.com
# Add server: service-postgresql, port 5432
```

**MySQL:**
```shell
# Test from host
mysql -h 127.0.0.2 -P 3306 -u root -p

# Test via phpMyAdmin at https://pma.network.local.com
```

### Test Object Storage

**MinIO:**
```shell
# Install MinIO client
curl https://dl.min.io/client/mc/release/linux-amd64/mc -o mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure client
mc alias set local https://s3api.network.local.com minioadmin your_minio_password

# Test operations
mc mb local/test-bucket
mc ls local/
```

### Test Email

**Mailpit:**
```shell
# Send test email via SMTP (port 1025)
echo "Test email body" | mail -s "Test Subject" -S smtp=127.0.0.2:1025 test@example.com

# View emails at https://mail.network.local.com
```

---

## 🔧 Troubleshooting

### Common Issues

1. **Services not accessible via HTTPS:**
   ```shell
   # Check Traefik logs
   docker logs service-traefik
   
   # Verify network connectivity
   docker network inspect reverse-proxy
   ```

2. **SSL certificate warnings:**
   ```shell
   # Reinstall mkcert CA
   mkcert -install
   
   # Restart browser completely
   killall chrome firefox 2>/dev/null || true
   ```

3. **Port conflicts:**
   ```shell
   # Check what's using ports
   sudo netstat -tlnp | grep :80
   sudo netstat -tlnp | grep :443
   ```

4. **Container startup issues:**
   ```shell
   # Check individual service logs
   docker logs service-postgresql
   docker logs service-pgladmin
   docker logs service-phpmyadmin
   docker logs service-minio
   docker logs service-mailpit
   ```

### Service Health Checks

```shell
# Check all services status
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Test HTTP responses
curl -k -I https://traefik.network.local.com/dashboard/
curl -k -I https://pga.network.local.com
curl -k -I https://pma.network.local.com
curl -k -I https://s3.network.local.com
curl -k -I https://mail.network.local.com
```

### Reset Everything

If you need to start fresh:

```shell
# Stop all services
cd ~/docker-service
for dir in */; do
  if [ -f "$dir/compose.yml" ]; then
    cd "$dir"
    docker compose down
    cd ..
  fi
done

# Remove all containers and volumes
docker system prune -a --volumes

# Recreate network
docker network rm reverse-proxy
docker network create reverse-proxy

# Start services again following the startup order
```

---

## 📚 Adding New Services

To add a new service to the platform:

1. **Create service directory:**
```shell
cd ~/docker-service
git clone git@github.com:yourorg/new-service new-service
```

2. **Configure Traefik labels in compose.yml:**
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=reverse-proxy"
  # HTTP Router (redirects to HTTPS)
  - "traefik.http.routers.new-service.entrypoints=http"
  - "traefik.http.routers.new-service.rule=Host(`new-service.network.local.com`)"
  - "traefik.http.middlewares.new-service-https-redirect.redirectscheme.scheme=https"
  - "traefik.http.routers.new-service.middlewares=new-service-https-redirect"
  # HTTPS Router
  - "traefik.http.routers.new-service-secure.entrypoints=https"
  - "traefik.http.routers.new-service-secure.rule=Host(`new-service.network.local.com`)"
  - "traefik.http.routers.new-service-secure.tls=true"
  - "traefik.http.services.new-service.loadbalancer.server.port=8080"
```

3. **Add to hosts file:**
```shell
echo "127.0.0.2        new-service.network.local.com" | sudo tee -a /etc/hosts
```

4. **Generate SSL certificate:**
```shell
cd ~/docker-service/server-reverse-proxy
mkcert new-service.network.local.com
# Update Traefik certificates if needed
```

5. **Ensure network connectivity:**
```yaml
networks:
  - reverse-proxy
```

---

## 🔒 Security Considerations

### Development Environment

This setup is designed for **local development only**. For production use:

1. **Change all default passwords**
2. **Use proper SSL certificates** (Let's Encrypt)
3. **Enable authentication** on all services
4. **Use environment-specific configurations**
5. **Implement proper backup strategies**
6. **Use secure networking** (VPN, firewall rules)

### Credential Management

- Store all passwords in a secure password manager
- Use `.env` files for environment variables
- Never commit `.env` files to version control
- Rotate passwords regularly

---

## 📞 Support

For issues or questions:

1. **Check service logs:** `docker logs <container-name>`
2. **Verify network connectivity:** `docker network inspect reverse-proxy`
3. **Test SSL certificates:** `openssl s_client -connect service.network.local.com:443`
4. **Review Traefik dashboard:** https://traefik.network.local.com/dashboard/

---

## 📄 License

This platform setup is for development purposes. Please ensure all individual services comply with their respective licenses.

---

**🎉 Congratulations! Your complete Docker service platform is now ready for development.**