# Docker Setup & Best Practices for Django Projects (with uv)

A practical guide to containerizing Django with Docker, docker-compose,  
and uv for reproducible, secure, production-ready deployments.  

Docker gives Django projects reproducible, portable environments that work  
the same on every machine and in every stage of the delivery pipeline.  
Combined with **uv** — a fast, deterministic Python package manager — you  
get reliable builds, lean images, and consistent dependency resolution  
without the overhead of traditional pip/venv workflows. This guide covers  
everything from a first `docker compose up` to zero-downtime production  
deployments, with code examples for every step.

---

## Why Use Docker for Django

### Reproducible environments

Every developer and CI runner starts from the same base image and the  
same locked dependency set. "It works on my machine" problems disappear  
because the container *is* the machine.

### Consistent dev/prod parity

The same Dockerfile (or a thin variation of it) is used for local  
development and production. Configuration differences are injected via  
environment variables, not baked into the image.

### Simplified onboarding

A new contributor can clone the repo and run `docker compose up` without  
installing Python, Postgres, or Redis locally.

### Isolation of dependencies

Each project runs in its own container with its own Python environment.  
System packages, Python versions, and library versions never collide  
between projects.

### Why uv instead of pip/venv inside containers

- `uv pip install` is 10–100× faster than `pip install` due to its  
  Rust-based resolver and parallel downloads.  
- `uv.lock` provides fully pinned, cross-platform lockfiles (like  
  `poetry.lock` but faster and simpler).  
- A virtualenv is unnecessary inside a container — `uv` can install  
  directly into the system Python with `--system`, which keeps the  
  image smaller and the `PATH` simpler.  
- `uv` honours `pyproject.toml` natively, so there is one source of  
  truth for project metadata and dependencies.

---

## Using uv for Dependency Management

### `uv pip install` vs `pip install`

`uv pip install` is a drop-in replacement for `pip install` that resolves  
and installs packages in parallel. It produces identical results but  
significantly faster, which matters most in CI and Docker layer caching.

```bash
# Traditional
pip install -r requirements.txt

# With uv (same semantics, much faster)
uv pip install -r requirements.txt --system
```

### Using `pyproject.toml` + `uv.lock`

Declare runtime and dev dependencies in `pyproject.toml`:

```toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "django>=5.0",
    "psycopg[binary]>=3.1",
    "gunicorn>=22.0",
    "redis>=5.0",
]

[dependency-groups]
dev = [
    "pytest-django>=4.8",
    "coverage>=7.4",
]
```

The `[project]` table defines runtime dependencies that are required in  
every environment. `psycopg[binary]` is the modern PostgreSQL adapter that  
ships with pre-compiled C extensions, avoiding the need for `libpq-dev` at  
runtime. The `[dependency-groups]` table holds extras like test runners  
that are only installed in development and CI, keeping the production  
image free of unnecessary packages.

Generate and commit the lockfile:

```bash
uv lock          # create/update uv.lock
uv sync          # install exactly what is in uv.lock
uv sync --group dev  # include dev dependencies
```

### Deterministic builds

The `uv.lock` file pins every direct and transitive dependency to an  
exact version and hash. Commit it to source control. Docker builds that  
run `uv sync` against a committed lockfile always produce the same  
installed environment.

### Caching layers in Docker

Place dependency installation before copying application code so that the  
layer is rebuilt only when `pyproject.toml` or `uv.lock` changes:

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --system
COPY . .
```

Docker executes each `RUN`, `COPY`, and `ADD` instruction as a separate  
immutable layer, and reuses cached layers when the inputs have not changed.  
By copying the lockfile and installing dependencies before the application  
source, only a change to `pyproject.toml` or `uv.lock` triggers a fresh  
`uv sync`. Changing any Python source file — which happens far more  
frequently — reuses the cached dependency layer, cutting build times from  
minutes to seconds on a warm cache.

### Recommended directory layout

```
myproject/
├── myproject/          # Django project package
├── myapp/              # Django app
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── entrypoint.sh
└── nginx/
    └── nginx.conf
```

### Avoiding virtualenvs inside containers

Pass `--system` to `uv pip install` or `uv sync` so packages are  
installed into the container's system Python. There is no need for  
`python -m venv .venv` or `source .venv/bin/activate` inside a  
Dockerfile. This keeps paths simple and images smaller.

---

## Dockerfile Best Practices (with uv)

### Multi-stage builds

Multi-stage builds separate the build environment from the runtime  
environment. The final image contains only what is needed to run the app.

```dockerfile
# ── builder stage ────────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Cache dependency layer separately from application code
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --system

# ── runtime stage ─────────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /usr/local/lib/python3.12/site-packages \
                    /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Copy application source
COPY . .

# Non-root user
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

The builder stage installs all Python dependencies (including compilation  
toolchains if needed) into the system Python. The runtime stage starts  
from a fresh `python:3.12-slim` image and copies only the installed  
`site-packages` and binaries — not gcc, libpq-dev, or any other build  
tool. This means the final image is typically 50–80 MB smaller than a  
single-stage build, and the attack surface is reduced because build tools  
are absent at runtime.

### Using Python slim images

`python:3.12-slim` is based on Debian slim and is the best general  
default. It omits documentation, man pages, and many system utilities  
while still supporting `pip`, `gcc`, and standard C libraries via  
`apt-get`. Avoid `:alpine` for Django — building `psycopg`, `Pillow`,  
and similar packages against musl libc is error-prone.

### Installing system dependencies

Install system packages in the builder stage so they do not bloat the  
final image:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq-dev \
        gcc \
    && rm -rf /var/lib/apt/lists/*
```

`libpq-dev` provides the PostgreSQL client headers required to compile  
`psycopg` (the C extension). `gcc` is the C compiler needed by several  
Python packages that ship with native extensions. Both are build-time  
dependencies only — in the runtime stage, you only need `libpq5` (the  
shared library), not the dev headers. Deleting `/var/lib/apt/lists/*` in  
the same `RUN` layer removes the package index, which can save 30–40 MB  
from the image.

If the package is needed at runtime (e.g. `libpq5`), install it in the  
runtime stage as well.

### Caching uv install layers

Docker rebuilds every layer after a changed layer. Order matters:

1. Copy only `pyproject.toml` and `uv.lock` first.  
2. Run `uv sync`.  
3. Copy application source last.

This way, changing a Python file does not invalidate the dependency layer.

### Copying only required files

Use `.dockerignore` to exclude everything that should not be in the  
image. Copy explicit paths rather than relying solely on `.dockerignore`:

```dockerfile
COPY myproject/ myproject/
COPY myapp/     myapp/
COPY manage.py  manage.py
COPY entrypoint.sh entrypoint.sh
```

### `.dockerignore` essentials

```
.git
.venv
__pycache__
*.pyc
*.pyo
.env
.env.*
*.sqlite3
media/
staticfiles/
.mypy_cache
.pytest_cache
node_modules
```

### Creating a non-root user

Running as root inside a container is a security risk. Create a dedicated  
system user in the final stage:

```dockerfile
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup --no-create-home appuser
USER appuser
```

---

## Development vs Production Images

### Dev image with hot-reload

```dockerfile
FROM python:3.12-slim AS dev

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --system  # includes dev dependencies

COPY . .

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

For async projects, replace the CMD with:

```dockerfile
CMD ["uvicorn", "myproject.asgi:application", \
     "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Prod image with Gunicorn/Uvicorn

```dockerfile
CMD ["gunicorn", "myproject.wsgi:application", \
     "--workers", "4", \
     "--bind", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### Why dev and prod images should be separate

- Dev images include test runners, debuggers, and the Django dev server —  
  none of which belong in production.  
- Mixing them bloats the production image and increases the attack surface.  
- Use Docker Compose overrides or separate `Dockerfile.dev` /  
  `Dockerfile.prod` files to keep them distinct.

### Environment-specific settings

```python
# settings/base.py   — shared settings
# settings/dev.py    — imports base, sets DEBUG=True
# settings/prod.py   — imports base, sets DEBUG=False
```

Select via an environment variable:

```bash
DJANGO_SETTINGS_MODULE=myproject.settings.prod
```

---

## docker-compose for Local Development

```yaml
# docker-compose.yml
services:
  web:
    build:
      context: .
      target: dev
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app              # live code reload
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    entrypoint: /app/entrypoint.sh

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

### entrypoint.sh — running migrations automatically

```bash
#!/bin/sh
set -e

echo "Waiting for database..."
python manage.py migrate --noinput
exec "$@"
```

`set -e` causes the script to exit immediately if any command returns a  
non-zero status, preventing the server from starting in a broken state.  
Running migrations here (rather than in the Dockerfile) ensures they  
execute against the live database at startup, which is only available at  
runtime. `exec "$@"` replaces the shell process with the container's  
main command (e.g. `gunicorn`), so signals like `SIGTERM` are delivered  
directly to the application and Docker can perform a clean shutdown.

Make it executable: `chmod +x entrypoint.sh`

### Volume mounts

The `.:/app` bind mount makes host source code changes visible inside  
the container immediately. Do **not** use bind mounts in production.

### Environment variables

Store all secrets and config in `.env` (never committed to git):

```bash
# .env
DEBUG=True
SECRET_KEY=dev-secret-change-in-prod
DATABASE_URL=postgres://myuser:mypassword@db:5432/mydb
REDIS_URL=redis://redis:6379/0
ALLOWED_HOSTS=localhost,127.0.0.1
```

### Using `depends_on` correctly

`depends_on` alone only waits for the container to *start*, not until the  
service inside it is *ready*. Use `condition: service_healthy` combined  
with a `healthcheck` to ensure Postgres and Redis are accepting  
connections before Django starts.

### Updating dependencies inside the container

```bash
# Add a new package and update the lockfile
docker compose run --rm web uv add celery
docker compose run --rm web uv sync --system
```

---

## Production Deployment with Docker

### Gunicorn or Uvicorn workers

```dockerfile
CMD ["gunicorn", "myproject.wsgi:application", \
     "--workers", "4", \
     "--worker-class", "gthread", \
     "--threads", "2", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--access-logfile", "-"]
```

For ASGI (Django Channels, async views):

```dockerfile
CMD ["uvicorn", "myproject.asgi:application", \
     "--host", "0.0.0.0", "--port", "8000", \
     "--workers", "4"]
```

Rule of thumb: `workers = 2 × CPU cores + 1`.

### Nginx reverse proxy

```yaml
# docker-compose.prod.yml (excerpt)
  nginx:
    image: nginx:1.26-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_volume:/app/staticfiles:ro
      - media_volume:/app/media:ro
    depends_on:
      - web
```

```nginx
# nginx/nginx.conf
server {
    listen 80;

    location /static/ {
        alias /app/staticfiles/;
    }

    location /media/ {
        alias /app/media/;
    }

    location / {
        proxy_pass         http://web:8000;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

The `/static/` and `/media/` blocks let Nginx serve files directly from  
named volumes, completely bypassing Python — this is orders of magnitude  
faster than routing those requests through Gunicorn. The proxy headers  
are essential for Django: `X-Forwarded-For` allows `request.META['REMOTE_ADDR']`  
to reflect the real client IP rather than the container network address,  
and `X-Forwarded-Proto` ensures `request.is_secure()` returns `True` for  
HTTPS requests terminated at the proxy.

### Static/media file handling

Run `collectstatic` during the Docker build or in the entrypoint:

```dockerfile
RUN python manage.py collectstatic --noinput
```

Mount a named volume for `MEDIA_ROOT` so uploaded files persist across  
container restarts.

### Healthchecks

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1
```

Add a lightweight `/health/` view that returns HTTP 200:

```python
# urls.py
from django.http import HttpResponse
from django.urls import path

urlpatterns += [path("health/", lambda r: HttpResponse("ok"))]
```

### Logging

Log to stdout/stderr so the Docker daemon captures output via  
`docker logs` and forwards it to your logging infrastructure:

```python
LOGGING = {
    "version": 1,
    "handlers": {
        "console": {"class": "logging.StreamHandler"},
    },
    "root": {"handlers": ["console"], "level": "INFO"},
}
```

### Container restarts

```yaml
services:
  web:
    restart: unless-stopped
```

Use `restart: unless-stopped` for long-running services and  
`restart: on-failure` for one-shot tasks like migrations.

### Deterministic production images with uv

```dockerfile
RUN uv sync --frozen --no-dev --system
```

`--frozen` prevents uv from updating `uv.lock`, ensuring the exact  
versions recorded in the lockfile are installed every time.

---

## Database & Caching Containers

### PostgreSQL configuration

```yaml
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --locale=en_US.UTF-8"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Redis for caching and sessions

```yaml
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
```

Enable AOF persistence (`--appendonly yes`) so Redis survives restarts  
without losing session data.

### Persistent volumes

Always use named volumes (not bind mounts) for database data in  
production. Named volumes are managed by Docker and survive container  
removal:

```yaml
volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

### Backup strategy

```bash
# Dump Postgres
docker exec db pg_dump -U myuser mydb > backup.sql

# Restore
docker exec -i db psql -U myuser mydb < backup.sql
```

Schedule backups with a cron container or an external tool such as  
`pgbackups` or a cloud snapshot.

---

## Environment Variables & Secrets

### `.env` files

```bash
# .env  (never commit to git — add to .gitignore)
SECRET_KEY=replace-with-a-real-random-key
DEBUG=False
DATABASE_URL=postgres://myuser:mypassword@db:5432/mydb
REDIS_URL=redis://redis:6379/0
ALLOWED_HOSTS=example.com
```

Load in `settings.py` via `django-environ`:

```python
import environ

env = environ.Env()
environ.Env.read_env()

SECRET_KEY = env("SECRET_KEY")
DEBUG = env.bool("DEBUG", default=False)
DATABASES = {"default": env.db("DATABASE_URL")}
```

`django-environ` wraps Python's `os.environ` and adds type coercion,  
default values, and URL parsing. `env.bool("DEBUG", default=False)` ensures  
the value is a Python `bool` regardless of whether the shell variable is  
`"True"`, `"true"`, or `"1"`. `env.db("DATABASE_URL")` parses a standard  
connection URL such as `postgres://user:pass@host:5432/db` into the  
`DATABASES` dictionary format Django expects, removing the need to  
manually split host, port, name, and credentials into separate settings.

### Avoiding secrets in images

- Never `COPY .env` into a Dockerfile.  
- Never use `ENV SECRET_KEY=...` in a Dockerfile.  
- Pass secrets at runtime via `env_file`, `--env-file`, or an external  
  secrets manager (AWS Secrets Manager, Vault, Docker Secrets).

### Secure configuration patterns

```yaml
services:
  web:
    env_file:
      - .env          # local dev
    environment:
      - DJANGO_SETTINGS_MODULE=myproject.settings.prod
```

In production, inject secrets via the orchestrator (Kubernetes Secrets,  
ECS task definitions, Docker Swarm secrets) rather than `.env` files.

---

## Static & Media Files in Docker

### `collectstatic` workflow

Run `collectstatic` in the Dockerfile so static files are baked into the  
image (preferred for production):

```dockerfile
RUN python manage.py collectstatic --noinput --clear
```

Or run it in the entrypoint before starting the server if settings are  
only available at runtime.

### Serving static files in production

Never serve static files with `runserver` or Gunicorn in production.  
Use Nginx (or a CDN) to serve the `STATIC_ROOT` directory:

```python
STATIC_URL  = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

MEDIA_URL   = "/media/"
MEDIA_ROOT  = BASE_DIR / "media"
```

### Media file storage options

| Option | Best for |
|--------|----------|
| Named Docker volume | Single-server setups |
| AWS S3 / GCS / Azure Blob | Multi-server, cloud deployments |
| NFS / shared volume | On-premise multi-node setups |

For S3, use `django-storages` with `S3Boto3Storage`:

```python
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
```

---

## Security Best Practices

### Non-root user

```dockerfile
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup --no-create-home appuser
USER appuser
```

### Read-only filesystem

```yaml
services:
  web:
    read_only: true
    tmpfs:
      - /tmp
      - /app/staticfiles   # if collectstatic runs at startup
```

Mark the container filesystem read-only and allow writes only to  
explicitly declared `tmpfs` paths.

### Limiting container capabilities

```yaml
services:
  web:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE   # only if binding to port <1024
    security_opt:
      - no-new-privileges:true
```

### Avoiding SSH inside containers

Do not install or run `sshd` inside containers. Use  
`docker exec -it <container> sh` for interactive access. In production,  
use your cloud provider's access mechanisms or a jump host.

### Image scanning

Scan images for known CVEs before pushing to production:

```bash
# Using Docker Scout
docker scout cves myimage:latest

# Using Trivy
trivy image myimage:latest
```

Integrate scanning into the CI pipeline and fail the build on  
critical/high severity findings.

---

## Performance Optimization

### Worker tuning

Gunicorn: `workers = 2 × vCPU + 1`, `threads = 2–4` per worker.  
Uvicorn: match `workers` to vCPU count; it is async so fewer are needed.  
Avoid over-provisioning — too many workers cause excessive memory usage  
and context switching.

### Caching layers

- Pin base image tags (e.g. `python:3.12.4-slim`) to avoid unexpected  
  rebuilds caused by tag updates.  
- Split `COPY` into dependency files first, then source code.  
- Use `--mount=type=cache` (BuildKit) to persist uv and pip caches  
  between builds:

```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --system
```

`--mount=type=cache` is a BuildKit feature that mounts a persistent cache  
directory for the duration of the `RUN` instruction. uv stores downloaded  
wheel and sdist files in `/root/.cache/uv`, so on the next build, packages  
that have not changed are read from the local cache instead of being  
re-downloaded from PyPI. In a CI environment with a warm cache this can  
reduce a full `uv sync` from 60+ seconds to under 5 seconds, with no  
impact on the final image size because the cache directory is not  
committed to any layer.

### Using slim base images

| Image | Compressed size |
|-------|----------------|
| `python:3.12` | ~350 MB |
| `python:3.12-slim` | ~50 MB |
| `python:3.12-alpine` | ~25 MB |

Prefer `slim` for Django. Alpine requires musl and causes build issues  
with compiled Python extensions.

### Minimizing layers

Combine related `RUN` commands with `&&` and clean up caches in the  
same layer:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*
```

### uv's speed advantages in CI/CD

- uv resolves and downloads packages in parallel.  
- Combined with `--mount=type=cache`, a full `uv sync` on a warm CI  
  cache typically finishes in under 5 seconds.  
- The `--frozen` flag ensures no network calls are made beyond downloading  
  pinned packages, making builds fast and reproducible.

---

## CI/CD Integration

### Building images in CI

```yaml
# .github/workflows/build.yml (GitHub Actions excerpt)
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: runtime
          cache-from: type=gha
          cache-to: type=gha,mode=max
          tags: myimage:${{ github.sha }}
          push: false
```

### Tagging strategy

| Tag | Purpose |
|-----|---------|
| `myimage:latest` | Latest stable build (CD only) |
| `myimage:<git-sha>` | Immutable reference to a specific commit |
| `myimage:v1.2.3` | Semantic version tag for releases |

Never use `latest` as the only tag in production — it is not immutable.

### Pushing to registries

```bash
docker tag myimage:abc123 registry.example.com/myimage:abc123
docker push registry.example.com/myimage:abc123
```

Use OIDC-based authentication (e.g. GitHub Actions + ECR / GAR) rather  
than storing registry credentials in CI secrets.

### Zero-downtime deploys

- Use a load balancer (Nginx, Traefik, or a cloud ALB) with multiple  
  replicas.  
- Perform rolling restarts: bring up new containers before tearing down  
  old ones.  
- Run database migrations in a separate pre-deploy step, not inside the  
  main entrypoint, to avoid downtime from long migrations.

### Using uv in CI for reproducible builds

```yaml
      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Run tests
        run: uv run pytest
```

`uv run` installs dependencies from `uv.lock` and executes the command  
in a single step, with no manual virtualenv activation needed.

---

## Common Pitfalls to Avoid

- **`DEBUG=True` in production** — Django exposes tracebacks and internal  
  settings. Always set `DEBUG=False` and configure `ALLOWED_HOSTS`.

- **Mixing dev and prod settings** — Use separate settings modules  
  (`settings/dev.py`, `settings/prod.py`) and select with  
  `DJANGO_SETTINGS_MODULE`.

- **Storing data inside containers** — Container filesystems are  
  ephemeral. Always use named volumes for Postgres, Redis, and media  
  files.

- **Bloated images** — Multi-stage builds, `.dockerignore`, and  
  `apt-get clean` keep images lean. Audit with `docker image history`.

- **Forgetting healthchecks** — Without healthchecks, orchestrators  
  (Compose, Kubernetes, ECS) cannot know when a container is actually  
  ready to serve traffic.

- **Using pip/venv inside Docker instead of uv** — `pip` is slower,  
  lacks a lock file mechanism, and leaves virtualenv overhead inside  
  containers. Use `uv sync --frozen --system` for fast, deterministic,  
  no-virtualenv installs.

- **Hardcoding secrets in Dockerfiles or images** — Use runtime  
  environment injection. Anything in a Dockerfile layer can be extracted  
  from the image.

- **Not pinning base image tags** — `python:3.12-slim` can change  
  between builds. Pin to a specific digest or patch version for  
  reproducibility in production pipelines.
