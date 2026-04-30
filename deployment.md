# Setting Up Django for Deployment: Best Practices and Production Checklist

A practical, security-focused guide for deploying Django applications to  
production. Covers environment configuration, WSGI/ASGI stacks, Nginx,  
PostgreSQL, caching, monitoring, and zero-downtime workflows.  

The guide is aimed at intermediate Django developers who are comfortable  
with local development but need a reliable, opinionated reference for  
taking an application to a live server. Each section focuses on actionable  
steps rather than theory, and includes copy-paste-ready configuration  
snippets for settings files, systemd units, Nginx virtual hosts, and  
shell commands. Follow the sections in order for a first deployment, or  
jump directly to a specific topic when hardening an existing setup.  

---

## Pre-Deployment Preparation

Prepare your project before touching a server. Every item in this section  
must be resolved before the application goes live.  

### Environment variables

Never hard-code secrets or environment-specific values in source code.  
Store them in the host environment and read them at runtime.  

Install `python-decouple`:  

```
pip install python-decouple
```

`.env` (never commit this file):  

```ini
SECRET_KEY=replace-with-a-long-random-value
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgres://dbuser:dbpass@localhost:5432/mydb
REDIS_URL=redis://localhost:6379/0
```

In `settings/base.py`:  

```python
from decouple import config, Csv

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())
```

Add `.env` to `.gitignore`:  

```
.env
*.pyc
__pycache__/
staticfiles/
media/
```

### Secret key management

- Generate a fresh key for every environment:  

  ```python
  # run once from any Python interpreter
  from django.core.management.utils import get_random_secret_key
  print(get_random_secret_key())
  ```

- Store the key in the environment, a secrets manager (AWS Secrets Manager,  
  HashiCorp Vault), or a `.env` file outside version control.  
- Rotate the key if it is ever exposed; all existing sessions will be  
  invalidated.  

### DEBUG=False implications

Setting `DEBUG=False` changes Django's behaviour in several important ways:  

- Detailed error pages are replaced with generic 500/404 responses.  
- `ALLOWED_HOSTS` is enforced; requests with an unknown `Host` header are  
  rejected with a 400 error.  
- Static files are no longer served by Django's development server.  
- Template debug information is hidden from end users.  

You must configure a custom error handler and serve static files via Nginx  
or WhiteNoise before disabling debug mode.  

### ALLOWED_HOSTS

```python
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())
# e.g. ALLOWED_HOSTS=example.com,www.example.com
```

Include every hostname and IP address through which the application will be  
reached, including the server's public IP during initial testing.  

### Static and media file strategy

Decide early how static and user-uploaded files will be served:  

| Scenario | Static files | Media files |
|---|---|---|
| Single server | Nginx or WhiteNoise | Nginx, local disk |
| Multi-server / container | CDN + WhiteNoise | Object storage (S3) |

```python
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

Run `collectstatic` as part of every deploy:  

```
python manage.py collectstatic --noinput
```

---

## Project Settings for Production

### Splitting settings (dev / staging / prod)

Organise settings into a package so each environment only loads what it  
needs:  

```
myproject/
  settings/
    __init__.py   # empty
    base.py       # shared settings
    dev.py        # imports base, overrides for local work
    staging.py    # imports base, mirrors production
    prod.py       # imports base, production-only values
```

`settings/dev.py`:  

```python
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

`settings/prod.py`:  

```python
from .base import *
import dj_database_url

DEBUG = False

DATABASES = {
    'default': dj_database_url.config(
        env='DATABASE_URL', conn_max_age=600, ssl_require=True
    )
}
```

Select the module with the `DJANGO_SETTINGS_MODULE` environment variable:  

```
DJANGO_SETTINGS_MODULE=myproject.settings.prod
```

### Secure cookies

```python
SESSION_COOKIE_SECURE = True      # only sent over HTTPS
SESSION_COOKIE_HTTPONLY = True     # not accessible via JavaScript
SESSION_COOKIE_SAMESITE = 'Lax'   # CSRF mitigation
SESSION_COOKIE_AGE = 1209600      # two weeks, in seconds

CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Lax'
```

### CSRF and HTTPS configuration

```python
CSRF_TRUSTED_ORIGINS = [
    'https://example.com',
    'https://www.example.com',
]

SECURE_SSL_REDIRECT = True              # redirect HTTP -> HTTPS
SECURE_HSTS_SECONDS = 31536000         # one year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

### Logging configuration

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/myapp/django.log',
            'maxBytes': 10 * 1024 * 1024,   # 10 MB
            'backupCount': 5,
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'WARNING',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'WARNING',
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

Create the log directory and set permissions before starting the service:  

```
sudo mkdir -p /var/log/myapp
sudo chown www-data:www-data /var/log/myapp
```

### Database configuration (PostgreSQL)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 60,      # persistent connections, seconds
        'OPTIONS': {
            'sslmode': 'require',
        },
    }
}
```

Install the driver: `pip install psycopg2-binary` (or `psycopg[binary]`  
for psycopg v3).  

---

## Choosing a Deployment Stack

### WSGI — Gunicorn + Nginx

The standard choice for traditional synchronous Django applications:  

```
Browser -> Nginx (TLS, static) -> Gunicorn (WSGI) -> Django
```

- Gunicorn forks multiple worker processes; each handles one request at a  
  time.  
- Best for CPU-bound or I/O-bound workloads without long-lived connections.  
- Works with all existing Django middleware and third-party packages.  

### ASGI — Uvicorn/Gunicorn + Nginx

Required when your application uses:  

- Django Channels (WebSockets)  
- `async` views or middleware  
- Server-Sent Events  

```
Browser -> Nginx (TLS, static) -> Gunicorn + UvicornWorker -> Django ASGI
```

Gunicorn manages worker lifecycle; each worker runs an async Uvicorn event  
loop, giving you both process supervision and async concurrency.  

### When to choose each

| Requirement | Stack |
|---|---|
| Traditional sync views only | Gunicorn (WSGI) |
| WebSockets / Channels | Gunicorn + UvicornWorker (ASGI) |
| async views, SSE | Gunicorn + UvicornWorker (ASGI) |
| Maximum simplicity | Gunicorn (WSGI) |

---

## Application Server Setup

### Gunicorn configuration

Install: `pip install gunicorn`  

`gunicorn.conf.py` (place in project root):  

```python
bind = 'unix:/run/gunicorn/myapp.sock'
workers = 3            # (2 x CPU cores) + 1 is a common starting point
worker_class = 'sync'  # or 'uvicorn.workers.UvicornWorker' for ASGI
worker_connections = 1000
timeout = 30
keepalive = 5
max_requests = 1000          # restart workers periodically to avoid leaks
max_requests_jitter = 100
accesslog = '/var/log/myapp/gunicorn-access.log'
errorlog = '/var/log/myapp/gunicorn-error.log'
loglevel = 'warning'
```

Start manually to verify configuration:  

```
gunicorn --config gunicorn.conf.py myproject.wsgi:application
```

### Uvicorn workers for ASGI

Install: `pip install uvicorn[standard]`  

Change `worker_class` in `gunicorn.conf.py`:  

```python
worker_class = 'uvicorn.workers.UvicornWorker'
```

Point Gunicorn at the ASGI application:  

```
gunicorn --config gunicorn.conf.py myproject.asgi:application
```

### systemd service examples

Create `/etc/systemd/system/gunicorn.service`:  

```ini
[Unit]
Description=Gunicorn daemon for myapp
Requires=gunicorn.socket
After=network.target

[Service]
Type=notify
User=www-data
Group=www-data
RuntimeDirectory=gunicorn
WorkingDirectory=/srv/myapp
ExecStart=/srv/myapp/.venv/bin/gunicorn \
          --config /srv/myapp/gunicorn.conf.py \
          myproject.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/gunicorn.socket`:  

```ini
[Unit]
Description=Gunicorn socket

[Socket]
ListenStream=/run/gunicorn/myapp.sock
SocketUser=www-data

[Install]
WantedBy=sockets.target
```

Enable and start:  

```
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn.socket
sudo systemctl start gunicorn
```

Store sensitive environment variables in `/etc/myapp/env`:  

```ini
DJANGO_SETTINGS_MODULE=myproject.settings.prod
SECRET_KEY=...
DATABASE_URL=postgres://...
```

Set permissions: `sudo chmod 600 /etc/myapp/env`  

### Worker tuning

| Variable | Guideline |
|---|---|
| `workers` | `(2 x CPU cores) + 1` |
| `threads` (sync worker) | 1 per worker (default) |
| `worker_connections` | 1000 for async workers |
| `timeout` | 30 s; increase only for known slow endpoints |
| `max_requests` | 1000-5000 to guard against memory leaks |

Monitor worker memory with `ps aux` or Prometheus; restart workers if RSS  
grows beyond an acceptable threshold.  

---

## Reverse Proxy Setup (Nginx)

Nginx sits in front of Gunicorn and handles TLS termination, static file  
serving, compression, and rate limiting.  

### Recommended Nginx configuration

`/etc/nginx/sites-available/myapp`:  

```nginx
upstream django {
    server unix:/run/gunicorn/myapp.sock fail_timeout=0;
}

server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    client_max_body_size 20M;

    location /static/ {
        alias /srv/myapp/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /srv/myapp/media/;
        expires 7d;
        add_header Cache-Control "public";
    }

    location / {
        proxy_pass         http://django;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_redirect     off;
        proxy_read_timeout 30s;
        proxy_connect_timeout 10s;
    }
}
```

Enable the site:  

```
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Gzip and Brotli

Add to the `http` block in `/etc/nginx/nginx.conf`:  

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript
           text/xml application/xml image/svg+xml;
gzip_min_length 256;
```

For Brotli (requires `nginx-module-brotli`):  

```nginx
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript
             text/xml application/xml image/svg+xml;
```

### Caching headers

Static assets built with a content hash in their filename can be cached  
indefinitely:  

```nginx
location /static/ {
    alias /srv/myapp/staticfiles/;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

For resources without versioned filenames, use a shorter TTL:  

```nginx
expires 7d;
add_header Cache-Control "public, max-age=604800";
```

### HTTPS termination

Use Certbot with the Nginx plugin to obtain and auto-renew Let's Encrypt  
certificates:  

```
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot creates a cron job or systemd timer that renews certificates  
before they expire.  

---

## Database and Caching

### PostgreSQL configuration basics

Create a dedicated database and role:  

```sql
CREATE ROLE myapp_user WITH LOGIN PASSWORD 'strong-password';
CREATE DATABASE myapp_db OWNER myapp_user;
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO myapp_user;
```

Harden `pg_hba.conf` to allow only local connections from the app user:  

```
# TYPE  DATABASE    USER        ADDRESS       METHOD
local   myapp_db    myapp_user                scram-sha-256
host    myapp_db    myapp_user  127.0.0.1/32  scram-sha-256
```

### Connection pooling

Under high load, opening a new PostgreSQL connection for every request is  
expensive. Use PgBouncer as a lightweight connection pooler:  

```
sudo apt install pgbouncer
```

`/etc/pgbouncer/pgbouncer.ini`:  

```ini
[databases]
myapp_db = host=127.0.0.1 port=5432 dbname=myapp_db

[pgbouncer]
listen_port = 6432
listen_addr = 127.0.0.1
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
```

Point Django at PgBouncer's port (6432) instead of PostgreSQL's port  
(5432). Set `CONN_MAX_AGE = 0` when using transaction pooling.  

### Redis for caching and sessions

Install: `pip install django-redis`  

```python
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': config('REDIS_URL', default='redis://127.0.0.1:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
        },
        'KEY_PREFIX': 'myapp',
    }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

### Migrations workflow

Run migrations before restarting application workers to avoid serving  
requests against an out-of-date schema:  

```
python manage.py migrate --run-syncdb
```

For zero-downtime migrations, make backward-compatible schema changes  
first, deploy the application, then remove old columns in a subsequent  
migration.  

---

## Static and Media Files

### collectstatic

Run as part of every deploy:  

```
python manage.py collectstatic --noinput --clear
```

`--clear` removes stale files from `STATIC_ROOT` before copying. Omit it  
if you want a simpler incremental copy.  

### WhiteNoise vs Nginx

**WhiteNoise** serves static files directly from the Django/Gunicorn  
process - no separate Nginx location block required. Suitable for  
single-server deployments and PaaS platforms (Heroku, Render).  

Install: `pip install whitenoise`  

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',   # second, after Security
    ...
]

STATICFILES_STORAGE = (
    'whitenoise.storage.CompressedManifestStaticFilesStorage'
)
```

**Nginx** is more efficient for high-traffic sites because it serves files  
from the OS kernel without involving Python. Use Nginx when you control  
the server and performance matters.  

### Media file storage options

| Option | Package | Use case |
|---|---|---|
| Local disk + Nginx | — | Single-server, low traffic |
| AWS S3 | `django-storages[s3]` | Multi-server, scalable |
| Google Cloud Storage | `django-storages[gcs]` | GCP-hosted apps |
| Azure Blob Storage | `django-storages[azure]` | Azure-hosted apps |

S3 example (`settings/prod.py`):  

```python
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = config('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = config('AWS_S3_REGION_NAME', default='us-east-1')
AWS_S3_FILE_OVERWRITE = False
AWS_DEFAULT_ACL = None   # use bucket policy, not object ACL
```

---

## Security Hardening

### HTTPS everywhere

Redirect all HTTP traffic to HTTPS in Nginx (see Reverse Proxy section).  
Also instruct Django to enforce it:  

```python
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

### HSTS

HTTP Strict Transport Security tells browsers to always use HTTPS for your  
domain. Start with a short `max-age` and increase it after verifying  
everything works:  

```python
SECURE_HSTS_SECONDS = 31536000         # one year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True             # submit to browser preload lists
```

### Secure cookies

```python
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
```

### CSRF and XSS protections

- `CsrfViewMiddleware` is enabled by default; never disable it.  
- Set `CSRF_TRUSTED_ORIGINS` to your actual domain(s).  
- Use Django's template auto-escaping to prevent XSS; never mark  
  user-supplied data as `safe`.  
- Add security headers in Nginx:  

```nginx
add_header X-Content-Type-Options  "nosniff" always;
add_header X-Frame-Options         "DENY" always;
add_header Referrer-Policy         "strict-origin-when-cross-origin" always;
add_header Permissions-Policy      "geolocation=(), microphone=()" always;
add_header Content-Security-Policy
    "default-src 'self'; script-src 'self'; object-src 'none';" always;
```

### Avoiding common misconfigurations

- Run `python manage.py check --deploy` — Django audits production settings  
  and reports every detected problem.  
- Set `DJANGO_SETTINGS_MODULE` explicitly in the service environment file  
  so the wrong settings module is never loaded by accident.  
- Remove the `django.contrib.admin` URL if the admin interface is not  
  needed in production, or move it to a non-obvious path.  
- Disable the browsable API in Django REST Framework for production:  

  ```python
  REST_FRAMEWORK = {
      'DEFAULT_RENDERER_CLASSES': ['rest_framework.renderers.JSONRenderer'],
  }
  ```

---

## Performance Optimization

### Caching layers

Apply caching from the innermost to the outermost layer:  

1. **Per-view cache** — cache an entire view response:  

   ```python
   from django.views.decorators.cache import cache_page

   @cache_page(60 * 15)   # 15 minutes
   def article_list(request):
       ...
   ```

2. **Template fragment cache**:  

   ```html
   {% load cache %}
   {% cache 900 sidebar %}
       ...expensive sidebar...
   {% endcache %}
   ```

3. **Low-level cache API** — cache arbitrary Python objects:  

   ```python
   from django.core.cache import cache

   result = cache.get('heavy_computation')
   if result is None:
       result = run_heavy_computation()
       cache.set('heavy_computation', result, timeout=300)
   ```

### Query optimization

- Use `select_related` for forward FK and O2O relationships.  
- Use `prefetch_related` for reverse and M2M relationships.  
- Add database indexes on frequently filtered or sorted columns:  

  ```python
  class Article(models.Model):
      published_at = models.DateTimeField(db_index=True)
      category = models.ForeignKey(
          Category, on_delete=models.CASCADE, db_index=True
      )
  ```

- Use `only()` or `defer()` to fetch a subset of columns for wide tables.  
- Audit slow queries with `EXPLAIN ANALYZE` in PostgreSQL or Django  
  Debug Toolbar in development.  

### Compression

Enable Nginx gzip (see Reverse Proxy section). WhiteNoise adds  
pre-compressed `.gz` and `.br` files alongside each static asset  
automatically when using `CompressedManifestStaticFilesStorage`.  

### Using a CDN for static and media files

Point your CDN (Cloudflare, AWS CloudFront, Fastly) at:  

- `/static/` — cache aggressively with long TTLs and versioned filenames.  
- `/media/` — cache with shorter TTLs or use signed URLs for private  
  content.  

Update Django settings to use the CDN origin:  

```python
STATIC_URL = 'https://cdn.example.com/static/'
MEDIA_URL = 'https://cdn.example.com/media/'
```

---

## Monitoring and Observability

### Logs

Structured logging makes log aggregation and alerting much easier. Use  
`python-json-logger` to emit JSON lines:  

```
pip install python-json-logger
```

```python
'formatters': {
    'json': {
        '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
        'format': '%(asctime)s %(levelname)s %(name)s %(message)s',
    },
},
```

Ship logs to a centralised system (Loki + Grafana, Datadog, Papertrail,  
AWS CloudWatch) rather than reading files directly on the server.  

### Health checks

Expose a lightweight health endpoint that load balancers and uptime  
monitors can poll:  

```python
# urls.py
from django.http import JsonResponse

def health(request):
    return JsonResponse({'status': 'ok'})

urlpatterns = [
    path('health/', health),
    ...
]
```

For a deeper check (database connectivity, cache availability), use  
`django-health-check`:  

```
pip install django-health-check
```

### Uptime monitoring

- Use an external service (Better Uptime, Pingdom, UptimeRobot) to poll  
  `/health/` every minute from outside your network.  
- Set up alerts for downtime and slow response times.  

### Error tracking (Sentry)

Install: `pip install sentry-sdk`  

```python
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=config('SENTRY_DSN'),
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.2,   # 20% of requests for performance data
    send_default_pii=False,
    environment=config('DJANGO_ENV', default='production'),
)
```

Sentry captures unhandled exceptions, slow transactions, and can be wired  
to Slack or PagerDuty for on-call alerting.  

---

## Deployment Workflow

### Zero-downtime deploys

Use systemd socket activation (the socket is already set up in the  
*Application Server Setup* section). When you reload Gunicorn with  
`SIGHUP`, new workers start before old workers stop, so in-flight requests  
finish normally:  

```bash
# deploy script
git pull origin main
source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate --noinput
python manage.py collectstatic --noinput
sudo systemctl reload gunicorn
```

### Migrations order

Always run migrations **before** restarting the application. This ensures  
that new code never runs against an old schema:  

1. Deploy new code to the server (do not restart yet).  
2. Run `python manage.py migrate`.  
3. Reload Gunicorn workers.  

For large tables, prefer additive changes (new nullable columns, new  
tables) over destructive changes (column renames, drops) to keep  
migrations backward-compatible during the rollover window.  

### Rollback strategy

- Keep the previous release in a dated directory (e.g., `/srv/releases/`).  
- Maintain a `current` symlink pointing at the active release.  
- To roll back:  

  ```
  sudo ln -sfn /srv/releases/2024-01-14 /srv/myapp/current
  sudo systemctl reload gunicorn
  ```

- Only roll back the database if absolutely necessary, and only if the  
  migration is reversible: `python manage.py migrate myapp 0023`.  

### Using containers (optional)

For containerised deployments, the patterns are the same; the delivery  
mechanism changes.  

`Dockerfile`:  

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN python manage.py collectstatic --noinput

CMD ["gunicorn", "--config", "gunicorn.conf.py", \
     "myproject.wsgi:application"]
```

`docker-compose.yml` (production-oriented snippet):  

```yaml
services:
  web:
    image: myapp:latest
    env_file: .env
    depends_on:
      - db
      - redis
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

Use a container orchestrator (Kubernetes, AWS ECS, Fly.io) and rolling  
deployments for production container workloads.  

---

## Common Pitfalls to Avoid

### DEBUG=True in production

`DEBUG=True` exposes full stack traces with local variable values to  
anyone who triggers a 500 error. Set `DEBUG=False` and configure  
`ALLOWED_HOSTS` before going live. Verify with:  

```
python manage.py check --deploy
```

### Missing static files

Forgetting to run `collectstatic` or misconfiguring `STATIC_ROOT` results  
in CSS/JS 404 errors. Always include `collectstatic` in the deploy  
pipeline and confirm `STATIC_ROOT` is writable by the deploy user.  

### Incorrect file permissions

Gunicorn must be able to write to its socket directory, and Nginx must be  
able to read static/media directories:  

```
sudo chown -R www-data:www-data /srv/myapp/staticfiles /srv/myapp/media
sudo chmod -R 755 /srv/myapp/staticfiles /srv/myapp/media
sudo mkdir -p /run/gunicorn
sudo chown www-data:www-data /run/gunicorn
```

### Unoptimized Gunicorn workers

Too few workers starve legitimate traffic; too many exhaust server memory.  
The formula `(2 x CPU cores) + 1` is a reliable starting point. Monitor  
RSS per worker and adjust `max_requests` to recycle workers that grow too  
large.  

### Forgetting to configure HTTPS

Serving Django over plain HTTP exposes session cookies and CSRF tokens to  
eavesdroppers. Obtain a TLS certificate (Let's Encrypt is free) before  
the first production request is served, and enforce redirection from HTTP  
to HTTPS in Nginx.  

### Committing secrets to version control

Even a single committed secret in git history is a security incident.  
Use `git-secrets` or `gitleaks` as a pre-commit hook to block accidental  
commits of API keys and passwords.  

### Not setting CONN_MAX_AGE

Without persistent connections, Django opens a new database connection  
on every request. Set `CONN_MAX_AGE = 60` (or higher) in production to  
reuse connections across requests in the same worker process. If using  
PgBouncer in transaction mode, set it to `0` instead.  
