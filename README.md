# Docker Service Platform - Complete Setup Guide

This document provides a comprehensive step-by-step guide for setting up a complete Docker-based development platform with Traefik reverse proxy, SSL certificates, and multiple services.

**NOTE: You need to add your SSH key into GitHub first to access all repos shared with you.**

---

## 🚀 Services Included

### Core Platform

- **Traefik** - Reverse proxy with SSL termination
- **PostgreSQL** - Database server
- **PgAdmin** - PostgreSQL administration interface
- **MySQL** - Alternative database server
- **phpMyAdmin** - MySQL administration interface
- **MinIO** - S3-compatible object storage
- **Mailpit** - Email testing tool
- **Portainer** - Docker container management interface

### LiveKit Live-Streaming Stack

- **LiveKit** - WebRTC SFU + signaling server (WSS) for real-time audio/video
- **LiveKit Redis** - Internal Redis used by LiveKit & Egress to coordinate jobs
- **Egress** - Records LiveKit rooms to MP4 and produces LL-HLS segments
- **HLS Origin** - Nginx origin that serves the LL-HLS playlists/segments Egress writes
- **HLS CDN** - Public, viewer-facing Nginx edge cache in front of the HLS origin

### OpenCV / CV Pipeline

- **CV Pipeline** - OpenCV inference app with API, background worker, and LiveKit worker
- Reuses **LiveKit** and **LiveKit Redis** from the shared platform
- The app source is bind-mounted from `/var/www/html/cv-pipeline`, so code changes reflect in containers immediately

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

# Clone Portainer for container management
git clone git@github.com:yourorg/portainer portainer

# Clone LiveKit live-streaming stack
git clone git@github.com:yourorg/livekit livekit
git clone git@github.com:yourorg/livekit-redis livekit-redis
git clone git@github.com:yourorg/egress egress
git clone git@github.com:yourorg/hls-origin hls-origin
git clone git@github.com:yourorg/hls-cdn hls-cdn

# Clone CV pipeline / OpenCV app
git clone git@github.com:yourorg/cv-pipeline cv-pipeline
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

### 4. Create Shared HLS Docker Volume

The LiveKit Egress service writes LL-HLS segments into a shared Docker volume
that the HLS Origin reads from. Create it once before starting the streaming
stack:

```shell
docker volume create livekit-hls
```

> **Note:** Both `egress/compose.yml` and `hls-origin/compose.yml` reference this
> volume as `external: true`, so it MUST exist before those services start.

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
  portainer.network.local.com \
  livekit.network.local.com \
  hls.network.local.com \
  "*.network.local.com"

# Move certificates to correct location
# (the +N suffix in the filename reflects the number of extra SANs above)
mv traefik.network.local.com+8.pem container-data/certificates/selfsigned.crt
mv traefik.network.local.com+8-key.pem container-data/certificates/selfsigned.key
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

# Create backup directory and set proper permissions
sudo mkdir -p container-data/backup && sudo chown -R $USER:$USER container-data && sudo chmod -R 755 container-data

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

### 8. Configure Portainer

```shell
cd ~/docker-service/portainer

# Setup environment file
cp .sample.env .env
nano .env
```

Configure Portainer variables:

```env
TRAEFIK_HOSTNAME=portainer.network.local.com
ADMIN_PASSWORD=your_admin_password
```

---

## 🎥 LiveKit Live-Streaming Stack Configuration

The LiveKit stack provides real-time WebRTC streaming plus recording (MP4) and
low-latency HLS (LL-HLS) playback. The pieces work together like this:

```
Publisher ──WSS──> LiveKit ──> Egress ──┬──> MP4   (recordings, host folder)
                                         └──> HLS   (livekit-hls volume)
                                                       │
                              HLS Origin (nginx) ──────┘
                                     │
                              HLS CDN (nginx edge cache) ──> Viewers

LiveKit & Egress both coordinate jobs through LiveKit Redis.
```

### Project Bootpath (Important)

The Egress service writes MP4 recordings into a **host folder** that belongs to
your application project (the `Opic-Livestream` Laravel app). This path is the
"bootpath" that links the platform to the application:

```env
# egress/.env
RECORDINGS_PATH=/var/www/html/Opic-Livestream/recordings
```

Make sure this folder exists and is writable **before** starting Egress:

```shell
mkdir -p /var/www/html/Opic-Livestream/recordings
sudo chown -R $USER:$USER /var/www/html/Opic-Livestream/recordings
chmod -R 775 /var/www/html/Opic-Livestream/recordings
```

> Adjust the path if your application lives somewhere else — it must match
> `RECORDINGS_PATH` in `egress/.env`.

### 9. Configure LiveKit Server

```shell
cd ~/docker-service/livekit

# Setup environment file
cp .sample.env .env
nano .env
```

Configure the LiveKit hostname:

```env
TRAEFIK_HOSTNAME=livekit.network.local.com
```

The API key/secret and Redis address live in the config file at
[livekit/application/livekit.yaml](livekit/application/livekit.yaml). The keys
**must match** the application `.env` (`LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET`):

```yaml
keys:
  devkey: secret

redis:
  address: service-livestream-redis:6379
```

> **Ports:** LiveKit exposes `7880` (signaling/API), `7881` (RTC TCP fallback)
> and the UDP range `51000-51100` for WebRTC media. The UDP range is published
> on all interfaces because ICE candidates must be directly reachable — WebRTC
> media cannot pass through the Traefik HTTP proxy.

### 10. Configure LiveKit Redis

LiveKit Redis has **no environment file** and is **not routed through Traefik**.
It is an internal TCP backend reachable by container name
(`service-livestream-redis:6379`). No configuration is required — just start it.

### 11. Configure Egress (Recorder)

```shell
cd ~/docker-service/egress

# Setup environment file
cp .sample.env .env
nano .env
```

Set the host recordings folder (the project bootpath described above):

```env
RECORDINGS_PATH=/var/www/html/Opic-Livestream/recordings
```

The API key/secret and WebSocket URL live in
[egress/application/egress.yaml](egress/application/egress.yaml) and must match
the LiveKit config:

```yaml
api_key: devkey
api_secret: secret
ws_url: ws://service-livekit:7880
redis:
  address: service-livestream-redis:6379
```

> **File permissions:** The Egress process runs as a **non-root user (uid 1001,
> gid 0)**. Because of this it cannot write to a root-owned HLS volume. The
> `service-egress-hls-init` one-shot container in `egress/compose.yml`
> automatically fixes the `livekit-hls` volume permissions
> (`chgrp -R 0 /hls && chmod -R 2775 /hls`) on every `docker compose up`, so
> Egress can create per-room HLS subdirectories. You normally don't need to
> touch this — just be aware that Egress depends on that init step completing
> successfully. Egress also needs the `SYS_ADMIN` capability (already set in the
> compose file) for the headless Chrome recorder.

### 12. Configure HLS Origin

HLS Origin has **no environment file** and is **not routed through Traefik** —
it is an internal origin that the CDN reads from by container name
(`service-hls-origin:80`). It mounts the shared `livekit-hls` volume
**read-only** and serves the playlists/segments via
[hls-origin/application/nginx.conf](hls-origin/application/nginx.conf).

No configuration is required — just ensure the `livekit-hls` volume exists
(created in step 4).

### 13. Configure HLS CDN (Viewer Endpoint)

```shell
cd ~/docker-service/hls-cdn

# Setup environment file
cp .sample.env .env
nano .env
```

Configure the public HLS hostname that viewers connect to:

```env
TRAEFIK_HOSTNAME=hls.network.local.com
```

This is the **public LL-HLS endpoint**. It proxies and caches the HLS Origin
responses (long cache for immutable segments, near-live cache for playlists) via
[hls-cdn/application/nginx.conf](hls-cdn/application/nginx.conf). In production,
point a real CDN (Cloudflare / CloudFront) at the same origin.

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
127.0.0.2        portainer.network.local.com

# Database Admin Interfaces
127.0.0.2        pga.network.local.com
127.0.0.2        pma.network.local.com

# Storage Services
127.0.0.2        s3.network.local.com
127.0.0.2        s3api.network.local.com

# Communication Services
127.0.0.2        mail.network.local.com

# LiveKit Live-Streaming Stack
127.0.0.2        livekit.network.local.com
127.0.0.2        hls.network.local.com

# CV Pipeline / OpenCV App
127.0.0.2        cv-pipeline.network.local.com

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
docker compose up -d --build --force-recreate

cd ~/docker-service/mysql
docker compose up -d --build --force-recreate
```

3. **Start admin interfaces:**

```shell
cd ~/docker-service/pgadmin
docker compose up -d --build --force-recreate

cd ~/docker-service/phpmyadmin
docker compose up -d --build --force-recreate
```

4. **Start other services:**

```shell
cd ~/docker-service/minio
docker compose up -d --build --force-recreate

cd ~/docker-service/mailpit
docker compose up -d --build --force-recreate
```

5. **Start Portainer:**

```shell
cd ~/docker-service/portainer
docker compose up -d --build --force-recreate
```

6. **Start the LiveKit live-streaming stack (in this exact order):**

The streaming services have startup dependencies, so order matters:

```shell
# a) Redis first — LiveKit & Egress need it to coordinate jobs
cd ~/docker-service/livekit-redis
docker compose up -d --build --force-recreate

# b) LiveKit server (connects to Redis)
cd ~/docker-service/livekit
docker compose up -d --build --force-recreate

# c) Egress (depends on the hls-init step; needs the livekit-hls volume)
cd ~/docker-service/egress
docker compose up -d --build --force-recreate

# d) HLS Origin (reads the livekit-hls volume that Egress writes to)
cd ~/docker-service/hls-origin
docker compose up -d --build --force-recreate

# e) HLS CDN (public viewer endpoint; proxies the origin)
cd ~/docker-service/hls-cdn
docker compose up -d --build --force-recreate
```

> **Reminder:** Make sure both the `reverse-proxy` network and the `livekit-hls`
> volume exist (steps 3 and 4) and that `RECORDINGS_PATH` in `egress/.env` points
> to an existing, writable host folder before starting Egress.

7. **Start the CV Pipeline stack:**

The CV Pipeline stack reuses the shared LiveKit server and LiveKit Redis, so
start it after the LiveKit services are already up.

```shell
cd ~/docker-service/cv-pipeline
docker compose up -d --build --force-recreate
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

| Service               | URL                                          | Credentials                             |
| --------------------- | -------------------------------------------- | --------------------------------------- |
| **Traefik Dashboard** | https://traefik.network.local.com/dashboard/ | treafik / 12345678                      |
| **PgAdmin**           | https://pga.network.local.com                | admin@example.com / your_admin_password |
| **phpMyAdmin**        | https://pma.network.local.com                | root / your_root_password               |
| **MinIO Console**     | https://s3.network.local.com                 | minioadmin / your_minio_password        |
| **MinIO API**         | https://s3api.network.local.com              | -                                       |
| **Mailpit**           | https://mail.network.local.com               | No authentication                       |
| **Portainer**         | https://portainer.network.local.com          | Create admin account on first visit     |
| **LiveKit (WSS)**     | wss://livekit.network.local.com              | API key/secret: devkey / secret         |
| **HLS Playback**      | https://hls.network.local.com                | No authentication (viewer endpoint)     |
| **CV Pipeline**       | https://cv-pipeline.network.local.com        | App-specific auth / API config          |

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

### Test the LiveKit Streaming Stack

```shell
# Confirm the shared HLS volume exists
docker volume inspect livekit-hls

# Verify LiveKit signaling is reachable (should return HTTP 200/426)
curl -k -I https://livekit.network.local.com

# Verify the HLS CDN edge is up (health endpoint)
curl -k https://hls.network.local.com/healthz

# Watch Egress pick up a recording job (start a recording from the app first)
docker logs -f service-egress

# List recorded MP4 files on the host (the project bootpath)
ls -lah /var/www/html/Opic-Livestream/recordings
```

During a live session, viewers load the playlist from
`https://hls.network.local.com/<roomName>/index.m3u8`. The CDN caches segments
from the origin, which in turn serves the files Egress wrote to the
`livekit-hls` volume.

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

5. **LiveKit / streaming issues:**

   ```shell
   # "volume livekit-hls not found" — create it, then restart egress/hls-origin
   docker volume create livekit-hls

   # Egress can't write HLS / permission denied — the init container fixes perms;
   # confirm it completed successfully
   docker logs service-egress-hls-init

   # No MP4 files appearing — check RECORDINGS_PATH exists and is writable, and
   # that it matches egress/.env
   docker logs service-egress
   ls -lah /var/www/html/Opic-Livestream/recordings

   # Egress/LiveKit can't reach Redis — ensure redis is up and on the network
   docker logs service-livestream-redis
   docker exec service-livestream-redis redis-cli ping   # -> PONG

   # WebRTC media not connecting — verify the UDP range 51000-51100 is open
   sudo ufw status | grep 51000
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
curl -k -I https://livekit.network.local.com
curl -k https://hls.network.local.com/healthz
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

# Recreate the shared HLS volume (removed by the prune above)
docker volume create livekit-hls

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
