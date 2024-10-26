# Configuration

## Environment variables

They `python-decouple` package can be used for configuration.  

Inside the `.env` file:

```
SECRET_KEY=@lywhkp7^ccmvjv03pv6kh&y8f3+gtj49i@*bace0tb0lo6nb%
ENGINE=django.db.backends.postgresql
HOST=localhost
NAME=testdb
USER=postgres
PASSWORD=andrea
APP_NAME=myapp
```

In `settings.py` file: 

```python
from decouple import config

SECRET_KEY = config('SECRET_KEY')

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

## Enable toolbar

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

In main `ulrs.py` file:  

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

