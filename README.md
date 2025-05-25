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

# Taiga's URLs - Variables to define where Taiga should be served
TAIGA_SCHEME=http  # serve Taiga using "http" or "https" (secured) connection
TAIGA_DOMAIN=localhost:9000  # Taiga's base URL
SUBPATH="" # it'll be appended to the TAIGA_DOMAIN (use either "" or a "/subpath")
WEBSOCKETS_SCHEME=ws  # events connection protocol (use either "ws" or "wss")

# Taiga's Secret Key - Variable to provide cryptographic signing
SECRET_KEY="taiga-secret-key"  # Please, change it to an unpredictable value!!

# Taiga's Database settings - Variables to create the Taiga database and connect to it
POSTGRES_USER=taiga  # user to connect to PostgreSQL
POSTGRES_PASSWORD=taiga  # database user's password

# Taiga's SMTP settings - Variables to send Taiga's emails to the users
EMAIL_BACKEND=console  # use an SMTP server or display the emails in the console (either "smtp" or "console")
EMAIL_HOST=smtp.host.example.com  # SMTP server address
EMAIL_PORT=587   # default SMTP port
EMAIL_HOST_USER=user  # user to connect the SMTP server
EMAIL_HOST_PASSWORD=password  # SMTP user's password
EMAIL_DEFAULT_FROM=changeme@example.com  # default email address for the automated emails
# EMAIL_USE_TLS/EMAIL_USE_SSL are mutually exclusive (only set one of those to True)
EMAIL_USE_TLS=True  # use TLS (secure) connection with the SMTP server
EMAIL_USE_SSL=False  # use implicit TLS (secure) connection with the SMTP server

# Taiga's RabbitMQ settings - Variables to leave messages for the realtime and asynchronous events
RABBITMQ_USER=taiga  # user to connect to RabbitMQ
RABBITMQ_PASS=taiga  # RabbitMQ user's password
RABBITMQ_VHOST=taiga  # RabbitMQ container name
RABBITMQ_ERLANG_COOKIE=secret-erlang-cookie  # unique value shared by any connected instance of RabbitMQ

# Taiga's Attachments - Variable to define how long the attachments will be accesible
ATTACHMENTS_MAX_AGE=360  # token expiration date (in seconds)

# Taiga's Telemetry - Variable to enable or disable the anonymous telemetry
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
