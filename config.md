# Configuration

## Environment variables

The `python-decouple` package can be used for configuration management. It helps to  
separate configuration from code, making it easier to manage different environments  
(development, testing, production).

Install the package: `pip install python-decouple`

Inside the `.env` file:

```
SECRET_KEY=@lywhkp7^ccmvjv03pv6kh&y8f3+gtj49i@*bace0tb0lo6nb%
DEBUG=True
ENGINE=django.db.backends.postgresql
HOST=localhost
NAME=testdb
USER=postgres
PASSWORD=andrea
APP_NAME=myapp
ALLOWED_HOSTS=localhost,127.0.0.1
```

In `settings.py` file: 

```python
from decouple import config, Csv

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=Csv())

...

DATABASES = {
    'default': {
        'ENGINE': config('ENGINE'),
        'HOST': config('HOST'),
        'NAME': config('NAME'),
        'USER': config('USER'),
        'PASSWORD': config('PASSWORD'),
    }
}
```

The `Csv()` cast converts comma-separated values into a list. The `cast=bool` parameter  
ensures that string values like 'True' or 'False' are converted to boolean values.

## Debug toolbar

The Django Debug Toolbar is a configurable set of panels that display various debug  
information about the current request/response and when clicked, display more details  
about the panel's content.

Install module: `pip install django-debug-toolbar`  

In `settings.py`:  

```python
INSTALLED_APPS = [
    ...
    'django.contrib.staticfiles',

    'debug_toolbar',
    'lynx',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    ...
]
```

In main `urls.py` file:  

```python
urlpatterns = [
    ...
    path('__debug__/', include('debug_toolbar.urls'))
]
```

or 

```python
from debug_toolbar.toolbar import debug_toolbar_urls

urlpatterns = [
    ...
] + debug_toolbar_urls()
```

Resolve `JavaScript files are resolving to the wrong content type` error on  
Windows:  

```python
import mimetypes

mimetypes.add_type("application/javascript", ".js", True)

DEBUG_TOOLBAR_CONFIG = {
    "INTERCEPT_REDIRECTS": False,
}

INTERNAL_IPS = [
    '127.0.0.1',
]
```

## Static files configuration

Static files (CSS, JavaScript, Images) need proper configuration for both development  
and production environments.

In `settings.py`:

```python
import os

# URL to use when referring to static files
STATIC_URL = '/static/'

# Additional locations of static files
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static'),
]

# Directory where collectstatic will collect static files for deployment
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# Static files finders
STATICFILES_FINDERS = [
    'django.contrib.staticfiles.finders.FileSystemFinder',
    'django.contrib.staticfiles.finders.AppDirectoriesFinder',
]
```

For production, run `py manage.py collectstatic` to gather all static files into  
`STATIC_ROOT` directory.

## Media files configuration

Media files are user-uploaded content like images, videos, and documents. They should  
be configured separately from static files.

Install Pillow for image handling: `pip install pillow`

In `settings.py`:

```python
import os

# URL that handles the media served from MEDIA_ROOT
MEDIA_URL = '/media/'

# Absolute filesystem path to the directory for user-uploaded files
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

In main `urls.py` (for development only):

```python
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... your url patterns
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

## Multiple databases

Django can work with multiple databases simultaneously. This is useful for read replicas,  
legacy databases, or separating different types of data.

In `settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'main_db',
        'USER': 'postgres',
        'PASSWORD': 's$cret',
        'HOST': 'localhost',
        'PORT': '5432',
    },
    'users_db': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'users_database',
        'USER': 'mysql_user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '3306',
    },
    'legacy_db': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'legacy.sqlite3',
    }
}
```

Using specific database in models:

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    
    class Meta:
        app_label = 'myapp'
        # This model will use 'users_db'
```

Querying specific database:

```python
# Read from users_db
User.objects.using('users_db').all()

# Write to users_db
user = User(name='John')
user.save(using='users_db')
```

## CORS configuration

Cross-Origin Resource Sharing (CORS) configuration is necessary when your Django  
backend serves API requests from a different domain (e.g., a React or Vue frontend).

Install the package: `pip install django-cors-headers`

In `settings.py`:

```python
INSTALLED_APPS = [
    ...
    'corsheaders',
    ...
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]

# Allow all origins (development only)
CORS_ALLOW_ALL_ORIGINS = True

# Or specify allowed origins (production)
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://127.0.0.1:3000",
    "https://example.com",
]

# Allow credentials (cookies, authorization headers)
CORS_ALLOW_CREDENTIALS = True

# Allowed HTTP methods
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

# Allowed headers
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]
```

## Logging configuration

Logging helps track application behavior, errors, and performance issues. Django's  
logging configuration uses Python's logging module.

In `settings.py`:

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'filters': {
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'level': 'INFO',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        },
        'file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': BASE_DIR / 'logs/django.log',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}
```

Create the logs directory: `mkdir logs`

Using logger in views:

```python
import logging

logger = logging.getLogger('myapp')

def my_view(request):
    logger.debug('Debug message')
    logger.info('Info message')
    logger.warning('Warning message')
    logger.error('Error message')
    logger.critical('Critical message')
    
    return HttpResponse('Check logs')
```

