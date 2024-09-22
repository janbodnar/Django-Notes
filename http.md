# HTTP 


## URL parameters

In `views.py`:  

```python
def params(req):

    params = req.GET.dict()
    return JsonResponse(params)
```

In `urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path('params/', views.params),
]
```

## URL path parameters

Path convertors: 

* `int` – Matches zero or any positive integer.
* `str` – Matches any non-empty string, excluding the path separator ('/').
* `slug` – Matches any slug string, i.e. a string consisting of alphabets, digits, hyphen and underscore.
* `path` – Matches any non-empty string including the path separator ('/')
* `uuid` – Matches a UUID (universal unique identifier).

In `urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path('hello/<name>/', views.hello_url),
    path('random/<int:n>/', views.random_vals),
    path('', views.home),
]
```

In `views.py`:

```python
def home(req):
    return HttpResponse('Home page.', content_type='text/plain')

def hello_url(req, name='Guest'):
    return HttpResponse(f'hello {name}')

def random_vals(req, n=1):
    rands = {e: randint(-100, 100) for e in range(1, n+1)}
    print(rands)
    data = { 'vals': rands }
    return JsonResponse(data)
```
