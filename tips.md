# Most Important Django Development Tips

A practical reference for intermediate Django developers who want to build  
faster, write more maintainable code, and ship production-ready applications  
with confidence. Each section focuses on actionable guidance over theory.  

---

## Project Structure & Settings

Good project structure is the foundation of a maintainable Django application.  
Getting it right early prevents painful refactors later and makes onboarding  
new developers straightforward.  

### Recommended layout

Keep apps small and focused. A flat layout works well for most projects:  

```text
myproject/
├── config/               # Project package (settings, urls, wsgi/asgi)
│   ├── __init__.py
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   ├── orders/
│   └── products/
├── static/
├── templates/
├── requirements/
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env
└── manage.py
```

Place the project package (previously `myproject/`) inside a `config/`  
directory so it is never confused with an app. Group all first-party apps  
under `apps/` and import them as `apps.users`, `apps.orders`, etc.  

### Splitting settings

Never use a single `settings.py` for all environments. Split into a  
`settings/` package with a shared `base.py` and per-environment overrides:  

```python
# config/settings/base.py
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    # ...
    "apps.users",
    "apps.orders",
]
```

```python
# config/settings/dev.py
from .base import *  # noqa: F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

INSTALLED_APPS += ["debug_toolbar"]  # noqa: F405
```

```python
# config/settings/prod.py
from .base import *  # noqa: F403

DEBUG = False
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")
```

Point `DJANGO_SETTINGS_MODULE` to the correct file in each environment:  

```bash
# .env (development)
DJANGO_SETTINGS_MODULE=config.settings.dev

# shell / CI / server
export DJANGO_SETTINGS_MODULE=config.settings.prod
```

### Environment variables

Use `django-environ` or `python-decouple` to keep secrets out of source  
control. Never hard-code credentials in settings files.  

```bash
pip install django-environ
```

```python
# config/settings/base.py
import environ

env = environ.Env(
    DEBUG=(bool, False),
)
environ.Env.read_env(BASE_DIR / ".env")

SECRET_KEY = env("SECRET_KEY")
DEBUG = env("DEBUG")
DATABASES = {"default": env.db("DATABASE_URL")}
```

```ini
# .env
SECRET_KEY=your-secret-key-here
DEBUG=True
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
```

Add `.env` to `.gitignore` immediately after project creation. Provide a  
`.env.example` with placeholder values for every required variable.  

### Avoiding hard-coded paths

Always derive paths from `BASE_DIR` using `pathlib.Path`. Never write  
absolute paths like `/home/user/project/...` in settings.  

```python
# Good
TEMPLATES = [{"DIRS": [BASE_DIR / "templates"]}]
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_ROOT = BASE_DIR / "media"

# Bad – breaks on any other machine
TEMPLATES = [{"DIRS": ["/home/alice/myproject/templates"]}]
```

---

## Models & ORM Best Practices

Models are the core of every Django application. Clean, well-indexed models  
with a thoughtful query strategy are the single biggest factor in application  
performance and correctness.  

### Indexing

Add `db_index=True` or a `Meta.indexes` entry for every field that appears  
in `filter()`, `order_by()`, or `get()` calls beyond the primary key.  

```python
from django.db import models


class Order(models.Model):
    customer = models.ForeignKey(
        "users.Customer",
        on_delete=models.CASCADE,
    )
    status = models.CharField(max_length=20, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["customer", "status"]),
            models.Index(fields=["-created_at"]),
        ]
```

Use composite indexes when queries frequently filter on multiple columns  
together. Prefer `Meta.indexes` over `db_index=True` for composite or  
conditional indexes.  

### Avoiding N+1 queries

The N+1 problem is the most common Django performance trap. It occurs when  
a loop triggers one query per iteration to fetch related data.  

```python
# Bad – fires one extra query per order
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)   # SELECT on every iteration

# Good – one query with a JOIN
orders = Order.objects.select_related("customer")
for order in orders:
    print(order.customer.name)   # no extra queries
```

### Using select_related and prefetch_related

- `select_related` uses a SQL JOIN for `ForeignKey` and `OneToOneField`  
  relationships — best for single-object lookups.  
- `prefetch_related` issues a second query and joins in Python — required for  
  `ManyToManyField` and reverse `ForeignKey` relationships.  

```python
# ForeignKey / OneToOne – use select_related
articles = Article.objects.select_related("author", "category")

# ManyToMany / reverse FK – use prefetch_related
articles = Article.objects.prefetch_related("tags", "comments")

# Combine both when needed
articles = (
    Article.objects
    .select_related("author")
    .prefetch_related("tags")
)
```

Use `django-debug-toolbar` or `connection.queries` in the shell to verify  
query counts during development.  

### Migrations hygiene

- Run `makemigrations` every time you change a model.  
- Commit migration files to version control alongside the model changes.  
- Never edit a migration that has already been applied to a shared database.  
- Squash long migration chains periodically with `squashmigrations`.  
- Use `--check` in CI to fail the build if unapplied model changes exist:  

```bash
python manage.py migrate --check
```

### When to use custom managers

Use a custom manager to encapsulate commonly reused query logic and keep  
views and services clean.  

```python
from django.db import models


class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="published")


class Article(models.Model):
    title = models.CharField(max_length=255)
    status = models.CharField(max_length=20, default="draft")

    objects = models.Manager()          # default manager
    published = PublishedManager()      # custom manager


# Usage
Article.published.all()
Article.published.filter(author=user)
```

Keep the default `objects` manager intact so Django's admin and other  
built-in tools continue to work correctly.  

---

## Views & Business Logic

Views should orchestrate, not implement. A view's job is to accept a  
request, call the appropriate service, and return a response — nothing more.  

### Class-based views vs function-based views

Choose based on complexity and reuse:  

| Scenario | Preferred style |
|---|---|
| Simple endpoint, no reuse | Function-based view (FBV) |
| Standard CRUD | Class-based view (CBV) with generic mixin |
| Shared behaviour across views | CBV with mixins |
| DRF API endpoint | `APIView` / `ViewSet` |

```python
# Simple FBV – clear and explicit
from django.shortcuts import render, get_object_or_404

def article_detail(request, pk):
    article = get_object_or_404(Article, pk=pk)
    return render(request, "articles/detail.html", {"article": article})


# CBV – less boilerplate for standard patterns
from django.views.generic import DetailView

class ArticleDetailView(DetailView):
    model = Article
    template_name = "articles/detail.html"
    context_object_name = "article"
```

### Keeping logic out of views

A view that queries the database, applies business rules, sends emails, and  
formats a response is a fat view. Fat views are hard to test and impossible  
to reuse.  

```python
# Bad – business logic inside the view
def place_order(request):
    form = OrderForm(request.POST)
    if form.is_valid():
        order = form.save(commit=False)
        order.user = request.user
        order.total = sum(item.price for item in order.items.all())
        order.save()
        send_confirmation_email(order)
        inventory_service.decrement(order)
    return redirect("order-success")
```

### Using services/use-case modules

Move business logic into a dedicated `services.py` (or `services/` package)  
inside each app. Services are plain Python functions or classes — no Django  
request/response coupling.  

```python
# apps/orders/services.py
from .models import Order
from apps.notifications.email import send_confirmation_email
from apps.inventory.services import decrement_stock


def place_order(user, validated_data: dict) -> Order:
    order = Order.objects.create(user=user, **validated_data)
    order.total = _calculate_total(order)
    order.save(update_fields=["total"])
    send_confirmation_email(order)
    decrement_stock(order)
    return order


def _calculate_total(order: Order) -> int:
    return sum(item.price for item in order.items.all())
```

```python
# apps/orders/views.py
from django.shortcuts import redirect

from .forms import OrderForm
from .services import place_order


def order_create(request):
    form = OrderForm(request.POST or None)
    if form.is_valid():
        place_order(request.user, form.cleaned_data)
        return redirect("order-success")
    return render(request, "orders/create.html", {"form": form})
```

Services are easy to unit-test without a request context and can be called  
from management commands, Celery tasks, or other services.  

---

## Forms, Validation & Serialization

Validation belongs in forms and serializers, not in views or models.  
A clean validation layer makes your codebase predictable and your errors  
actionable.  

### DRF serializers

Use `ModelSerializer` for standard CRUD. Override `validate_<field>` for  
field-level rules and `validate` for cross-field rules.  

```python
from rest_framework import serializers
from .models import Article


class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["id", "title", "body", "status", "created_at"]
        read_only_fields = ["id", "created_at"]

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError(
                "Title must be at least 5 characters."
            )
        return value

    def validate(self, attrs):
        if attrs.get("status") == "published" and not attrs.get("body"):
            raise serializers.ValidationError(
                "A published article must have a body."
            )
        return attrs
```

### Custom validation

Keep custom validators as standalone functions or classes so they can be  
shared across forms and serializers without duplication.  

```python
# apps/core/validators.py
from django.core.exceptions import ValidationError


def validate_no_profanity(value: str) -> None:
    blocked = {"spam", "badword"}
    if any(word in value.lower() for word in blocked):
        raise ValidationError("Content contains prohibited language.")
```

```python
# Reuse in a model field
from apps.core.validators import validate_no_profanity

class Comment(models.Model):
    body = models.TextField(validators=[validate_no_profanity])
```

```python
# Reuse in a DRF serializer field
class CommentSerializer(serializers.ModelSerializer):
    body = serializers.CharField(validators=[validate_no_profanity])
```

### Clean separation of concerns

- **Forms / serializers** — validate raw input, transform data types.  
- **Services** — apply business rules, orchestrate operations.  
- **Models** — represent data structure and database-level constraints.  
- **Views** — pass validated data to services, return responses.  

Never call `request.POST` directly in a model method or a service function.  
Pass only clean, validated data through the service boundary.  

---

## Templates & Frontend Integration

Templates that contain business logic are a common source of bugs and  
maintenance pain. Keep templates thin and the template context rich.  

### Template organization

Organise templates to mirror the app structure:  

```text
templates/
├── base.html              # Site-wide base layout
├── partials/
│   ├── _navbar.html
│   └── _footer.html
├── articles/
│   ├── list.html
│   └── detail.html
└── users/
    ├── login.html
    └── profile.html
```

Use a project-level `templates/` directory and set it in `settings.py` so  
that project-level templates take precedence over app-level templates.  

```python
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],
        "APP_DIRS": True,
        ...
    }
]
```

### Avoiding heavy logic in templates

The Django template language deliberately limits what you can do. If you  
find yourself writing complex conditions or loops in a template, move that  
logic to the view or to a custom template tag.  

```html
{# Bad – complex filtering inside template #}
{% for item in items %}
  {% if item.status == "active" and item.user == request.user %}
    ...
  {% endif %}
{% endfor %}
```

```python
# Good – filter in the view, keep template dumb
def dashboard(request):
    items = Item.objects.filter(
        status="active", user=request.user
    )
    return render(request, "dashboard.html", {"items": items})
```

For reusable UI logic (e.g. truncating text, formatting dates), write  
custom template filters or tags in `templatetags/`:  

```python
# apps/core/templatetags/text_filters.py
from django import template

register = template.Library()


@register.filter
def truncate_words(value, num):
    """Return the first *num* words of *value*, appending '…' if truncated."""
    words = value.split()
    return " ".join(words[:num]) + ("…" if len(words) > num else "")
```

### Static files best practices

- Use `{% load static %}` and `{% static 'path/to/file' %}` — never  
  hard-code URLs like `/static/css/style.css`.  
- Run `collectstatic` as part of every deployment, not manually.  
- In production, serve static files via a CDN or a dedicated web server  
  (nginx, WhiteNoise) — never through Django's `runserver`.  

```python
# settings/prod.py — WhiteNoise for zero-config static serving
MIDDLEWARE = [
    "whitenoise.middleware.WhiteNoiseMiddleware",
    ...
]
STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
```

---

## Security Essentials

Django provides strong security defaults, but misconfiguration is common.  
Review each setting before going to production.  

### CSRF

CSRF protection is enabled by default via  
`django.middleware.csrf.CsrfViewMiddleware`. Never disable it globally.  
For API views that use token authentication instead of cookies, use  
DRF's `@csrf_exempt`-aware `APIView` — it disables CSRF only for  
session-cookie-less clients.  

```html
{# All HTML forms must include the CSRF token #}
<form method="POST">
  {% csrf_token %}
  ...
</form>
```

### XSS

Django auto-escapes all template variables. Risks arise when you use  
`{{ value|safe }}` or `mark_safe()` carelessly. Only mark content as safe  
when you fully control the source.  

```python
# Dangerous – user input rendered as raw HTML
from django.utils.safestring import mark_safe
description = mark_safe(user_supplied_html)   # XSS risk

# Safe – let Django escape it
description = user_supplied_html              # escaped automatically
```

### SQL injection

The ORM parameterises all values by default. SQL injection can only occur  
when you use `.raw()` or `connection.cursor()` with string interpolation.  

```python
# Dangerous
User.objects.raw(f"SELECT * FROM auth_user WHERE name='{name}'")

# Safe – always use parameterised queries
User.objects.raw("SELECT * FROM auth_user WHERE name=%s", [name])
```

Avoid `.extra()` with interpolated SQL. Prefer the ORM or, when raw SQL is  
unavoidable, always use parameter placeholders.  

### Secure password handling

Django uses PBKDF2 with a SHA-256 hash by default. Use Argon2 for  
stronger hashing in production (requires `argon2-cffi`):  

```python
# settings/prod.py
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
]
```

Never store plain-text or reversibly-encrypted passwords. Always call  
`user.set_password(raw_password)` — never assign to `user.password`  
directly.  

### ALLOWED_HOSTS

Set `ALLOWED_HOSTS` to an explicit list in production. An empty list or  
`["*"]` allows HTTP host header injection attacks.  

```python
# settings/prod.py
ALLOWED_HOSTS = ["api.example.com", "www.example.com"]
```

### Avoiding common misconfigurations

```python
# settings/prod.py
DEBUG = False                         # NEVER True in production
SECRET_KEY = env("SECRET_KEY")        # Never commit to source control
SECURE_SSL_REDIRECT = True            # Redirect HTTP → HTTPS
SESSION_COOKIE_SECURE = True          # Cookies over HTTPS only
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000        # Enable HSTS (1 year)
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
X_FRAME_OPTIONS = "DENY"             # Prevent clickjacking
```

---

## Performance Tips

Performance problems in Django usually fall into three categories: too many  
queries, too much data loaded per query, and no caching. Fix in that order.  

### Caching

Django supports per-view, per-template-fragment, and low-level caching.  
Configure a cache backend before using any cache decorator:  

```python
# settings/prod.py — Redis cache backend
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL"),
    }
}
```

**Per-view cache** — wraps an entire view response:  

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)    # cache for 15 minutes
def article_list(request):
    ...
```

**Per-template-fragment cache** — caches expensive template sections:  

```html
{% load cache %}
{% cache 900 sidebar user.id %}
  {# expensive sidebar rendering #}
{% endcache %}
```

**Low-level cache** — fine-grained control over what is stored:  

```python
from django.core.cache import cache

def get_trending_articles():
    key = "trending_articles"
    articles = cache.get(key)
    if articles is None:
        articles = list(
            Article.published.order_by("-views")[:10]
        )
        cache.set(key, articles, timeout=300)
    return articles
```

### Query optimisation

- Use `only()` or `defer()` to load a subset of fields when you do not  
  need the entire model.  
- Use `values()` or `values_list()` when you only need raw data, not  
  model instances.  
- Use `count()` and `exists()` instead of `len(queryset)` and  
  `bool(queryset)`.  
- Use `bulk_create()` and `bulk_update()` instead of loops.  

```python
# Load only the fields you need
Article.objects.only("title", "slug").filter(status="published")

# Check existence without loading data
if Order.objects.filter(user=user, status="pending").exists():
    ...

# Bulk insert – one query instead of N
articles = [Article(title=t) for t in titles]
Article.objects.bulk_create(articles, batch_size=500)
```

### Using Redis

Redis is the standard choice for caching, sessions, and message brokering  
in Django deployments. Use `django-redis` for the cache backend and  
Celery + Redis for async task queues:  

```bash
pip install django-redis celery
```

```python
# settings/prod.py
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": env("REDIS_URL"),
        "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

### Pagination

Always paginate list endpoints. Returning unbounded querysets is a  
performance and security risk.  

```python
# DRF – global default pagination
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS":
        "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 25,
}
```

```python
# Django views – use the built-in Paginator
from django.core.paginator import Paginator

def article_list(request):
    qs = Article.published.order_by("-created_at")
    paginator = Paginator(qs, 25)
    page = paginator.get_page(request.GET.get("page"))
    return render(request, "articles/list.html", {"page": page})
```

---

## Testing & Debugging

Untested Django code accumulates hidden regressions. A fast, isolated test  
suite is the best safety net for refactoring and deployment confidence.  

### pytest + pytest-django

Prefer `pytest` with `pytest-django` over Django's built-in test runner  
for cleaner syntax, better fixtures, and superior plugin support.  

```bash
pip install pytest pytest-django pytest-cov
```

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
python_files = tests.py test_*.py *_test.py
```

```python
# Minimal test using pytest-django
import pytest
from apps.orders.services import place_order


@pytest.mark.django_db
def test_place_order_creates_record(user, order_data):
    order = place_order(user, order_data)
    assert order.pk is not None
    assert order.user == user
```

### Factories instead of fixtures

Use `factory_boy` to create test objects. Factories produce minimal,  
realistic instances without requiring a full fixture file.  

```bash
pip install factory-boy
```

```python
# apps/users/factories.py
import factory
from django.contrib.auth import get_user_model

User = get_user_model()


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    username = factory.Sequence(lambda n: f"user{n}")
    email = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    password = factory.PostGenerationMethodCall("set_password", "password")
```

```python
# In a test
def test_profile_view(client):
    user = UserFactory()
    client.force_login(user)
    response = client.get(f"/users/{user.pk}/")
    assert response.status_code == 200
```

### Testing views, models, APIs

```python
# Test a DRF endpoint
import pytest
from rest_framework.test import APIClient
from apps.users.factories import UserFactory


@pytest.fixture
def api_client():
    return APIClient()


@pytest.mark.django_db
def test_article_list_returns_200(api_client):
    user = UserFactory()
    api_client.force_authenticate(user=user)
    response = api_client.get("/api/articles/")
    assert response.status_code == 200
    assert "results" in response.data
```

### Using Django Debug Toolbar

Install and configure `django-debug-toolbar` in the development  
environment only. It surfaces SQL queries, cache hits, template render  
times, and signal calls in the browser.  

```bash
pip install django-debug-toolbar
```

```python
# config/settings/dev.py
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE = ["debug_toolbar.middleware.DebugToolbarMiddleware"] + MIDDLEWARE
INTERNAL_IPS = ["127.0.0.1"]
```

```python
# config/urls.py (development only)
from django.conf import settings

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path("__debug__/", include(debug_toolbar.urls)),
    ] + urlpatterns
```

The SQL panel is the most valuable — look for duplicate queries (N+1)  
and unexpectedly slow queries.  

---

## Deployment & Production

A production Django deployment is not just `runserver` in a different  
environment. Each layer — process server, static files, database  
connections, and logging — needs deliberate configuration.  

### WSGI vs ASGI

| | WSGI | ASGI |
|---|---|---|
| Protocol | Synchronous only | Sync + async |
| Server | gunicorn, uWSGI | uvicorn, daphne |
| Use case | Traditional Django apps | WebSockets, async views, Django Channels |

For most projects without WebSockets or long-lived connections, WSGI with  
gunicorn is the simpler, more battle-tested choice.  

### gunicorn / uvicorn

```bash
# WSGI – gunicorn
pip install gunicorn
gunicorn config.wsgi:application \
    --workers 4 \
    --bind 0.0.0.0:8000 \
    --access-logfile -

# ASGI – uvicorn
pip install uvicorn[standard]
uvicorn config.asgi:application \
    --workers 4 \
    --host 0.0.0.0 \
    --port 8000
```

A good starting point for worker count is `2 * CPU_cores + 1`. Tune  
upward under load testing, not speculatively.  

### Static and media handling

- **Static files**: run `collectstatic` during the build step, then serve  
  with nginx or WhiteNoise. Never serve static files through Django.  
- **Media files** (user uploads): serve via nginx in production or upload  
  directly to an object store (S3, GCS) using `django-storages`.  

```python
# settings/prod.py – django-storages with S3
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
STATICFILES_STORAGE = "storages.backends.s3boto3.S3StaticStorage"
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")
AWS_S3_REGION_NAME = env("AWS_S3_REGION_NAME")
```

### Database connections

Use connection pooling with `pgBouncer` at the infrastructure level, or  
configure Django's persistent connections:  

```python
# settings/prod.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": env("DB_NAME"),
        "USER": env("DB_USER"),
        "PASSWORD": env("DB_PASSWORD"),
        "HOST": env("DB_HOST"),
        "PORT": env("DB_PORT", default="5432"),
        "CONN_MAX_AGE": 60,   # reuse connections for 60 seconds
    }
}
```

### Logging

Send logs to stdout in containerised deployments and ship them to an  
aggregator (Datadog, Sentry, CloudWatch). Avoid writing log files inside  
the container filesystem.  

```python
# settings/prod.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
        },
    },
    "root": {
        "handlers": ["console"],
        "level": "WARNING",
    },
    "loggers": {
        "django": {
            "handlers": ["console"],
            "level": "INFO",
            "propagate": False,
        },
    },
}
```

Integrate Sentry for automatic error tracking:  

```bash
pip install sentry-sdk
```

```python
# settings/prod.py
import sentry_sdk

sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    traces_sample_rate=0.2,
)
```

### Health checks

Expose a lightweight health-check endpoint for load balancers and  
orchestrators (Kubernetes, ECS). A minimal implementation:  

```python
# apps/core/views.py
from django.db import connection
from django.http import JsonResponse


def health_check(request):
    """Return 200 OK when the database is reachable, 503 otherwise.

    Used by load balancers and orchestrators (e.g. Kubernetes liveness
    probes) to determine whether the instance should receive traffic.
    """
    try:
        connection.ensure_connection()
        db_ok = True
    except Exception:
        db_ok = False
    status = 200 if db_ok else 503
    return JsonResponse({"db": db_ok}, status=status)
```

```python
# config/urls.py
path("health/", health_check, name="health-check"),
```

---

## Common Pitfalls to Avoid

Even experienced Django developers fall into these traps. Knowing them in  
advance saves significant debugging time.  

### Circular imports

Circular imports arise when two apps import from each other at the module  
level. The most common cause is importing models in `__init__.py` or at  
the top of a service module.  

```python
# Bad – importing at module level creates cycles
from apps.orders.models import Order   # may trigger circular import

# Good – defer the import inside the function
def some_task():
    from apps.orders.models import Order
    Order.objects.filter(...)
```

For cross-app model references in `ForeignKey`, use string references to  
avoid importing the model class at all:  

```python
class Order(models.Model):
    customer = models.ForeignKey(
        "users.Customer",        # string reference – no import needed
        on_delete=models.CASCADE,
    )
```

### Fat views

A view function or class that exceeds ~30 lines is usually doing too much.  
Move anything that is not request/response handling into a service.  
The symptom: a view that is difficult to test without spinning up an  
HTTP client.  

### Unoptimised queries

Key anti-patterns to avoid:  

| Anti-pattern | Fix |
|---|---|
| `len(queryset)` | `queryset.count()` |
| `queryset[0]` existence check | `.exists()` |
| Loop with `model.related_set.all()` | `prefetch_related` |
| `Model.objects.all()` in a template | Filter in the view, pass context |
| Counting in Python after fetching all rows | Use `aggregate(Count(...))` |

### Mixing business logic into models

Fat models are a tempting middle ground between fat views and services, but  
they become untestable and difficult to reuse. Limit model methods to  
simple property calculations or data transformations that are tightly  
coupled to the model's fields.  

```python
# Acceptable model method – pure data calculation
class Order(models.Model):
    @property
    def is_overdue(self) -> bool:
        from django.utils import timezone
        return self.due_date < timezone.now().date()

# Belongs in a service – has side effects and external dependencies
class Order(models.Model):
    def complete(self):             # Bad
        self.status = "complete"
        self.save()
        send_receipt_email(self)    # side effect in model
        inventory.decrement(self)   # cross-app dependency
```

### Misusing signals

Signals are powerful but make code hard to follow, test, and debug because  
the relationship between sender and receiver is implicit. Common misuse  
patterns include:  

- Using `post_save` to trigger business logic that should be in a service.  
- Sending emails or making HTTP requests inside a signal handler.  
- Creating circular signal chains (signal A triggers model save, which  
  fires signal B, which saves another model...).  

**Prefer explicit service calls over implicit signals** for business logic.  
Reserve signals for genuinely cross-cutting concerns (audit logging,  
cache invalidation) where you do not want the calling code to know about  
the side effect.  

```python
# Prefer this – explicit and testable
def register_user(email, password):
    user = User.objects.create_user(email=email, password=password)
    send_welcome_email(user)          # explicit call in service
    return user

# Avoid this for business logic
@receiver(post_save, sender=User)
def on_user_created(sender, instance, created, **kwargs):
    if created:
        send_welcome_email(instance)  # hidden side effect
```
