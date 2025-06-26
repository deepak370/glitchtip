
# GlitchTip Setup Documentation

This guide explains how to set up **GlitchTip** for self-hosted error monitoring and integrate it into **Laravel (PHP)** and **Django (Python)** applications.

---

## 1. System Requirements

### Core Services Required:
- PostgreSQL (v13+)
- Redis or Valkey
- Web service
- Worker service (Celery)

### Recommended System Specs:
- 1 GB RAM
- x86 or arm64 CPU

### Minimum System Specs:
- 512 MB RAM + Swap

💡 Disk usage depends on event volume. For example, 1 million events/month may require ~30 GB of disk.

---

## 2. Create Your Project Directory

```bash
mkdir glitchtip && cd glitchtip
```

---

## 3. Docker Compose Setup

Docker Compose is the simplest way to deploy GlitchTip on a single server.

### 3.1 Install Docker and Docker Compose (Debian/Ubuntu)
```bash
sudo apt install docker-compose docker.io
```

### 3.2 Create `docker-compose.yml`
Navigate to the `glitchtip/` directory and create the `docker-compose.yml` file.

📘 Official reference: [GlitchTip Docs](https://glitchtip.com/documentation/install)

Example `docker-compose.yml`:

```yaml
x-environment: &default-environment
  DATABASE_URL: postgres://postgres:postgres@postgres:5432/postgres
  SECRET_KEY: change_me_to_something_random
  PORT: 8000
  EMAIL_URL: consolemail://
  GLITCHTIP_DOMAIN: https://glitchtip.example.com
  DEFAULT_FROM_EMAIL: email@example.com
  CELERY_WORKER_AUTOSCALE: "1,3"

x-depends_on: &default-depends_on
  - postgres
  - redis

services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_HOST_AUTH_METHOD: "trust"
    restart: unless-stopped
    volumes:
      - pg-data:/var/lib/postgresql/data

  redis:
    image: valkey/valkey
    restart: unless-stopped

  web:
    image: glitchtip/glitchtip
    depends_on: *default-depends_on
    ports:
      - "8000:8000"
    environment: *default-environment
    restart: unless-stopped
    volumes:
      - uploads:/code/uploads

  worker:
    image: glitchtip/glitchtip
    command: ./bin/run-celery-with-beat.sh
    depends_on: *default-depends_on
    environment: *default-environment
    restart: unless-stopped
    volumes:
      - uploads:/code/uploads

  migrate:
    image: glitchtip/glitchtip
    depends_on: *default-depends_on
    command: ./bin/run-migrate.sh
    environment: *default-environment

volumes:
  pg-data:
  uploads:
```

### 3.3 Edit the Environment Values
Update these keys in your `docker-compose.yml`:
- `SECRET_KEY` (use: `openssl rand -hex 32`)
- `PORT`
- `EMAIL_URL`
- `GLITCHTIP_DOMAIN`
- `DEFAULT_FROM_EMAIL`

### 3.4 Deploy
```bash
docker-compose up -d
```
This command sets up:
- PostgreSQL
- Redis
- Web and Worker services
- Migration on startup

---

## 4. Configure Nginx Reverse Proxy

Create `/etc/nginx/conf.d/glitchtip.conf`:

```nginx
server {
  listen 80;
  listen 443 ssl;
  server_name glitchtip.example.in;

  ssl_certificate /etc/ssl/certs/pay1ssl25.crt;
  ssl_certificate_key /etc/ssl/certs/pay1ssl25.key;

  client_max_body_size 40M;

  if ($host != "glitchtip.example.in") {
    return 404;
  }

  location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

🔒 **Security Tip:** Use `127.0.0.1` instead of `0.0.0.0` for local binding.

---

## 5. Create an Admin User

After containers are running, create an admin:
```bash
docker-compose exec web ./manage.py createsuperuser
```
Follow prompts for:
- Email
- Password

🔐 Visit: `https://glitchtip.example.in` and log in.


![GlitchTip Login Page](login.png)

---

## Next Steps
- Configure GlitchTip projects for your Laravel and Django applications.
- Use the DSN from GlitchTip UI to add to your app’s config.