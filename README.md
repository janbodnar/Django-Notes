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
`django-admin startproject myapp ` - create new Django project  
`django-admin startproject myapp .` - create new Django project within current directory  
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

or 

```python
from django.http import HttpResponse

def hello(req):

    http_response = HttpResponse('', content_type='text/plain')
    http_response.write('Ahoy!')

    return http_response
```

launch request with:  `http localhost:8000/hello`  

## Determine user agent 

In `urls.py`, add the `user-agent/` endpoint.  

```python
from django.urls import path
from . import views

urlpatterns = [
    path('user-agent/', views.user_agent, name='user-agent' ),
]
```

From the `HttpRequest` object get the user agent and return it with `HttpResponse`.   

```python
from django.http import HttpRequest, HttpResponse

def user_agent(req: HttpRequest):

    ua = req.META['HTTP_USER_AGENT']
    return HttpResponse(ua, content_type='text/plain')
```

## Send file 

In `views.py`:  

```python
from django.http import HttpResponse, HttpResponseNotFound

def get_image(req):

    sid_path = 'lynx/images/sid.png'

    try:
        with open(sid_path, 'rb') as f:
           data = f.read()

        resp = HttpResponse(data, content_type='image/png')
        # resp['Content-Disposition'] = 'attachment; filename="sid.png"'
        resp['Content-Disposition'] = 'inline; filename="sid.png"'


    except IOError:
        resp = HttpResponseNotFound('<h2>file does not exist</h2>')

    return resp
```

or 

```python
from django.http import FileResponse, HttpResponseNotFound


def get_image(req):

    sid_path = 'lynx/images/sid.png'
    try:
        resp = FileResponse(open(sid_path, 'rb'), as_attachment=False,
                            filename='sid.png')

    except IOError:
        resp = HttpResponseNotFound('<h2>file does not exist</h2>')

    return resp
```

The `FileResponse` closes the file automatically.  
