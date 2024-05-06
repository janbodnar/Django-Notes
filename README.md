# Django-Notes
Django framework notes

## Virtual Environment

Create Virtual Environment  

`py -m venv myenv`

Activate the virtual environment:  

On Windows: `myenv\Scripts\activate`  
On Unix or MacOS: `source ./myenv/bin/activate` 

Deactivate the virtual environment:    
`deactivate`

## Basic commands 

`pip install django` - install Django  
`py manage.py runserver` - run server  
`py manage.py createsuperuser` - create superuser  
`django-admin startapp lynx` - create new app  
`py manage.py makemigrations` - make migrations  
`py manage.py migrate` - run migrations  

## Include URLs from module (app)

In the main project's `urls.py` file, add the `path`.  

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('lynx.urls')),
]
```

## Plain text HttpResponse

In `urls.py`:  

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.home),
    path('hello', views.hello, name='hello'),
]
```

In `views.py`:  

```python
from django.http import HttpResponse

def hello(req):

    return HttpResponse('Hello there!', content_type='text/plain')
```

launch request with:  `http localhost:8000/hello`  

