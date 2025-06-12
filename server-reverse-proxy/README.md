# README: Docker Compose Configuration - Traefik Reverse Proxy

This document provides an overview of the Traefik reverse proxy service configuration with self-signed SSL certificates.

**NOTE: You need to add your SSH key into GitHub first to access all repos shared with you.**

---

## Fresh Ubuntu 24.04 LTS

It is recommended to use fresh install of Ubuntu 24.04 LTS or later.

### Docker

You will need to install Docker and Docker Compose plugin natively on Ubuntu machine and not use Docker Desktop app. Use the guide below from official docker page.

[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

After that add your user to docker group so you can run docker commands without sudo.

``` shell
sudo usermod -aG docker $USER
```

Check to see if Docker and Docker Compose is installed properly by running.

``` shell
docker --version
docker compose
```

### PHP, Composer & Node.js

You will need to install PHP 8.3.X for better handling of auto-complete in VS Code and other useful PHP tools.

Use the following command to install PHP, Composer and other PHP extensions.

``` shell
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.3)"
```

Run following to verify correct version of PHP and Composer

``` shell
php -v
composer --version
```

Now install LTS version of Node.js (recommended 20.x) on your machine using Node.js official guide with node version manager (nvm) method.

[https://nodejs.org/en/download](https://nodejs.org/en/download)

*** NOTE you will need to run Node.js LTS recommended steps one by one to properly install Node.js via node version manager (nvm).

Once Node.js is installed make sure to include recommended paths in your .bashrc or .zshrc file and reload your profile using ```source``` command.

Example of lines to add to .zshrc is as follows:

``` shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

Verify Node.js and npm by running following commands

``` shell
node -v
npm -v
```

---

## Create Directories and Setup Your Platform Paths

You need to first create main directory in your home folder such as ```/home/user``` or ```/home/yourname```

``` shell
mkdir docker-service && cd docker-service
```

---

## Getting Started

Make sure you are in ```docker-service``` directory.
Clone the ```server-reverse-proxy``` repo.

``` shell
git clone git@github.com:yourorg/server-reverse-proxy server-reverse-proxy
# Or use HTTPS if SSH is not configured
# git clone https://github.com/yourorg/server-reverse-proxy server-reverse-proxy
```

Repository needs to be in ```main``` branch.

1. Go to the proper directory:

``` shell
cd ~/docker-service/server-reverse-proxy
```

2. Ensure the `reverse-proxy` network is created:

```bash
docker network create reverse-proxy
```

3. Set appropriate permissions for container data:

```bash
sudo chown -R $USER:$USER container-data && sudo chmod -R 755 container-data
```

4. Setup .env file:

```bash
cp .sample.env .env
```

Change all values in the ```.env``` file to set it up properly. Please note that the ```.env``` file must be kept secure and should not be shared on Github or anywhere else.

5. Install mkcert for trusted local SSL certificates:

```bash
# Install mkcert
sudo apt update && sudo apt install mkcert

# Install local CA
mkcert -install

# Generate certificates for your domains
mkcert traefik.network.local.com "*.local.network.com" localhost 127.0.0.2

# Move certificates to correct location
mkdir -p container-data/certificates
mv traefik.network.local.com+3.pem container-data/certificates/selfsigned.crt
mv traefik.network.local.com+3-key.pem container-data/certificates/selfsigned.key
```

6. Start the Traefik service:

```bash
docker compose up -d --build --force-recreate
```

7. Verify Traefik is running:

```bash
# Check container status
docker ps | grep traefik

# Check logs
docker logs service-traefik

# Test connection
curl -k -I https://traefik.network.local.com/dashboard/
```

***As per latest Traefik 3.3.x version all services inside compose.yml will automatically be routed via Traefik reverse proxy on their respective ports.***

---

## Host Entries (Local)

**Important:** Since we're using `127.0.0.2` (not `127.0.0.1`) to avoid conflicts with existing nginx installations, update your hosts file accordingly.

Use a proper DNS service such as PiHole or your router to forward all traffic to Traefik reverse proxy when you hit any *.local.network.com services to redirect all traffic to 127.0.0.2 or your local Wifi or Ethernet LAN IP address.

Or you can modify your local hosts file to add the following entries:

``` bash
sudo nano /etc/hosts
```

Required entry for Traefik:

``` bash
127.0.0.2        traefik.network.local.com
```

Additional entries for other services (update domain as needed):

``` bash
127.0.0.2        web.local.network.com
127.0.0.2        api.local.network.com
127.0.0.2        chat.local.network.com
127.0.0.2        mail.local.network.com
127.0.0.2        pma.local.network.com
127.0.0.2        redis.local.network.com
127.0.0.2        s3.local.network.com
127.0.0.2        timeline.local.network.com
127.0.0.2        postgres.local.network.com
```

You can replace 127.0.0.2 with your local Wifi or Ethernet LAN IP address if you encounter any issues.

---

## SSL Configuration

This setup uses **mkcert** for locally trusted SSL certificates. The certificates are automatically trusted by your browser without security warnings.

### Accessing Traefik Dashboard

1. **URL:** `https://traefik.network.local.com/dashboard/`
2. **Username:** `treafik`
3. **Password:** `12345678`

You should see a green lock icon in your browser indicating a secure connection.

### Troubleshooting SSL

If you see SSL warnings:

1. **Verify mkcert CA is installed:**
   ```bash
   mkcert -CAROOT
   ```

2. **Reinstall mkcert CA:**
   ```bash
   mkcert -install
   ```

3. **Restart browser completely:**
   ```bash
   killall chrome firefox 2>/dev/null || true
   ```

4. **Check certificate details:**
   ```bash
   openssl x509 -in container-data/certificates/selfsigned.crt -text -noout | grep -A 5 "Subject Alternative Name"
   ```

---

## Custom Proxy Network

All containers must be attached to the `reverse-proxy` network to enable seamless communication between services.

---

## Configuration Details

### Environment Variables

The following environment variables are used in `.env`:

- `TRAEFIK_HOSTNAME`: Domain for Traefik dashboard (default: `traefik.network.local.com`)
- `TRAEFIK_DASHBOARD_CREDENTIALS`: Basic auth credentials for dashboard access
- `TRAEFIK_MAIN_DOMAIN`: Main domain for certificates (default: `local.network.com`)
- `TRAEFIK_SANS_DOMAIN`: Wildcard domain for certificates (default: `*.local.network.com`)

### Ports

- **HTTP:** `127.0.0.2:80` (redirects to HTTPS)
- **HTTPS:** `127.0.0.2:443` (main entry point)

### Security

- Self-signed SSL certificates using mkcert
- Basic authentication for dashboard access
- TLS 1.2+ encryption
- Modern cipher suites

---

## Adding New Services

To add a new service to be proxied through Traefik:

1. **Ensure the service is on the `reverse-proxy` network**
2. **Add Traefik labels to your service:**

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.your-service.entrypoints=http"
  - "traefik.http.routers.your-service.rule=Host(`your-service.local.network.com`)"
  - "traefik.http.middlewares.your-service-https-redirect.redirectscheme.scheme=https"
  - "traefik.http.routers.your-service.middlewares=your-service-https-redirect"
  - "traefik.http.routers.your-service-secure.entrypoints=https"
  - "traefik.http.routers.your-service-secure.rule=Host(`your-service.local.network.com`)"
  - "traefik.http.routers.your-service-secure.tls=true"
```

3. **Add the domain to your hosts file:**
```bash
127.0.0.2        your-service.local.network.com
```

4. **Generate certificate for the new domain (optional):**
```bash
mkcert your-service.local.network.com
```

---

## Support

For issues or questions, please check:

1. Container logs: `docker logs service-traefik`
2. Network connectivity: `docker network ls | grep reverse-proxy`
3. Certificate validity: `openssl x509 -in container-data/certificates/selfsigned.crt -text -noout`
