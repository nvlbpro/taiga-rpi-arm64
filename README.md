# Taiga ARM64 Images

This repository provides Docker images for running [Taiga](https://www.taiga.io/), the agile project management platform, on ARM64 architectures, such as Raspberry Pi. It builds on the official [`taigaio/taiga-docker`](https://github.com/taigaio/taiga-docker) repository but uses pre-built ARM64 images hosted on [Docker Hub](https://hub.docker.com/r/nvlbpro/taiga-rpi-arm64).

## Features

- Pre-built Docker images for `taiga-back`, `taiga-front`, `taiga-async`, `taiga-events`, and `taiga-protected`, optimized for ARM64.
- Automated CI/CD pipeline using Docker Buildx for reliable builds and tests.
- Simplified setup with `launch-taiga.sh` and `taiga-manage.sh` scripts.
- Full compatibility with Raspberry Pi and other ARM64 platforms.
- Support for WebSocket-based real-time events (`taiga-events`) via internal proxy.
- Configurable for production use with HTTPS and custom domains.

## Prerequisites

- A Raspberry Pi (or other ARM64 platform) running Debian/Ubuntu.
- Docker and Docker Compose installed:
  ```bash
  sudo apt update
  sudo apt install docker.io docker-compose
  sudo systemctl enable docker
  sudo systemctl start docker
  ```
- Open network ports: 80 (HTTP) or 443 (HTTPS, if configured). The WebSocket for `taiga-events` is handled internally by `taiga-gateway` and does not require an external port (e.g., 8888).
- A configured `.env` file (included in the repository, see below).

## Installation

1. **Clone the repository**:
   ```bash
   git clone https://github.com/nvlbpro/taiga-rpi-arm64.git
   cd taiga-rpi-arm64
   ```

2. **Edit the `.env` file** in the cloned directory to customize the following variables (ensure secure values for sensitive fields):
   ```env
   # Taiga's URLs
   TAIGA_SCHEME=http  # Use "http" or "https"
   TAIGA_DOMAIN=localhost:9000  # Taiga's base URL
   SUBPATH=""  # Use "" or "/subpath"
   WEBSOCKETS_SCHEME=ws  # Use "ws" or "wss"

   # Taiga's Secret Key
   SECRET_KEY="taiga-secret-key"  # Change to a secure, unpredictable value

   # Database settings
   POSTGRES_USER=taiga
   POSTGRES_PASSWORD=taiga  # Change to a secure password

   # SMTP settings
   EMAIL_BACKEND=console  # Use "smtp" or "console"
   EMAIL_HOST=smtp.host.example.com
   EMAIL_PORT=587
   EMAIL_HOST_USER=user
   EMAIL_HOST_PASSWORD=password
   EMAIL_DEFAULT_FROM=changeme@example.com
   EMAIL_USE_TLS=True
   EMAIL_USE_SSL=False

   # RabbitMQ settings
   RABBITMQ_USER=taiga
   RABBITMQ_PASS=taiga  # Change to a secure password
   RABBITMQ_VHOST=taiga
   RABBITMQ_ERLANG_COOKIE=secret-erlang-cookie  # Change to a secure value

   # Attachments
   ATTACHMENTS_MAX_AGE=360  # Token expiration (seconds)

   # Telemetry
   ENABLE_TELEMETRY=True
   ```
   **Security Note**: The provided `.env` file is a template. Always replace default values for `SECRET_KEY`, `POSTGRES_PASSWORD`, `RABBITMQ_PASS`, and `RABBITMQ_ERLANG_COOKIE` with secure, unique values to prevent vulnerabilities.

3. **Start Taiga**:
   ```bash
   ./launch-taiga.sh
   ```

4. **Create an admin user**:
   ```bash
   ./taiga-manage.sh createsuperuser
   ```

5. **Access Taiga**:
   - URL: `http://<your-ip>:9000` or `https://<your-domain>` (if HTTPS is configured).
   - Log in with the admin user created.

## Network Configuration (Optional)

- **DNS**: Set up a domain (e.g., `taiga.yourdomain.com`) pointing to your server’s IP.
- **HTTPS**: Use a reverse proxy (e.g., NGINX, Apache) with an SSL certificate (e.g., Let’s Encrypt) for secure access.
- **WebSocket**: Real-time events (`taiga-events`) are proxied internally by `taiga-gateway`. No external port (e.g., 8888) needs to be opened.

### Example NGINX Configuration
This configuration enables HTTPS and proxies requests to `taiga-gateway`:
```nginx
server {
    listen 443 ssl;
    server_name taiga.yourdomain.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Example Apache Configuration
For setups like AWS EC2, this Apache configuration proxies requests to `taiga-gateway` and redirects HTTP to HTTPS:
```apache
<VirtualHost *:80>
    ServerName taiga.yourdomain.com
    ProxyPass / http://localhost:9000/
    ProxyPassReverse / http://localhost:9000/
    ProxyPreserveHost On

    ErrorLog ${APACHE_LOG_DIR}/taiga-error.log
    CustomLog ${APACHE_LOG_DIR}/taiga-access.log combined

    RewriteEngine On
    RewriteCond %{SERVER_NAME} =taiga.yourdomain.com
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

<VirtualHost *:443>
    ServerName taiga.yourdomain.com
    ProxyPass / http://localhost:9000/
    ProxyPassReverse / http://localhost:9000/
    ProxyPreserveHost On

    SSLEngine On
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem

    ErrorLog ${APACHE_LOG_DIR}/taiga-error.log
    CustomLog ${APACHE_LOG_DIR}/taiga-access.log combined
</VirtualHost>
```

## Useful Commands

- Stop Taiga:
  ```bash
  docker compose -f docker-compose.yml down
  ```
- View logs:
  ```bash
  docker compose -f docker-compose.yml logs taiga-back taiga-gateway
  ```
- Backup the database:
  ```bash
  docker exec taiga-rpi-arm64-taiga-db-1 pg_dump -U taiga taiga > taiga_backup_$(date +%F).sql
  ```
- Access PostgreSQL:
  ```bash
  docker exec -it taiga-rpi-arm64-taiga-db-1 psql -U taiga -d taiga
  ```

## Docker Images

Available on [Docker Hub](https://hub.docker.com/r/nvlbpro/taiga-rpi-arm64):
- `nvlbpro/taiga-rpi-arm64:*-stable`: Production-ready, tested images.
- `nvlbpro/taiga-rpi-arm64:*-latest`: Latest builds, use with caution.

## Contributing

Contributions are welcome! Please read the [CONTRIBUTING.md](CONTRIBUTING.md) file for guidelines on submitting issues or pull requests.

## Support

For questions, issues, or feedback:
- Open an issue on [GitHub](https://github.com/nvlbpro/taiga-rpi-arm64/issues).
- Contact me via [LinkedIn](https://www.linkedin.com/in/nvlbpro) or email (contact@nvlb.fr).

## License

This project is licensed under the [MIT License](LICENSE).