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


## Static files 

By default, static files are served from the `static` subdirectory of each module.  
Additional static directories can be specified in `STATICFILES_DIRS ` in `settings.py`.  

```html
<!DOCTYPE html>
{% load static %}

<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ article.title }}</title>
    <link rel="stylesheet" href="{% static 'css/style.css' %}">
</head>
<body>
    
    {{ article.title }}
    <br>
    {{ article.description }}

</body>
</html>
```

Static files are loaded with `{% load static %}` and later referenced with command such as `{% static 'css/style.css' %}`.   




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

## Upload images

Install pillow: `pip install pillow`.  

In `settings.py`: 

```python
MEDIA_URL = 'media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

In main `urls.py` add:  

```python
from django.conf import settings

urlpatterns += static(settings.MEDIA_URL,
                      document_root=settings.MEDIA_ROOT)
```

Model:  

```python
class Article(models.Model):
    title = models.CharField(max_length=255)
    description = models.TextField()
    banner = models.ImageField(default='flag.png', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
...
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

## Pagination

Message object.  

```python
from django.db import models

class Message(models.Model):
    text = models.CharField(max_length=255)

    def __str__(self):
        return self.text
```


Fetch messages and send them in paginator object.  

```python
from django.shortcuts import render
from django.core.paginator import Paginator, EmptyPage


from .models import Message


def index(request):

    messages = Message.objects.all()
    paginator = Paginator(messages, 20)

    try:
        page_num = request.GET.get('page', 1)
        page = paginator.page(page_num)
    except EmptyPage as e:
        page = paginator.page(1)

    ctx = {'messages': page}

    return render(request, 'index.html', ctx)
```

Display messages in template.  

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>

    {% for message in messages %}
    {{ message.text }}
    <br>
    {% endfor %}

    {% if messages.has_previous %}
    <a href="{% url 'index' %}?page={{messages.previous_page_number}}">Previous page</a>
    {% endif %}

    {% if messages.has_next %}
    <a href="{% url 'index' %}?page={{messages.next_page_number}}">Next page</a>
    {% endif %}


</body>
</html>
```

## Slugs

A slug is a user and seo friendly way of creating meaningful URLs to refer to our documents.  

```python
from django.db import models


class Article(models.Model):
    title = models.CharField(max_length=255)
    body = models.TextField()
    slug = models.SlugField(max_length=255)

    def __str__(self):
        return self.title
```

The `Article` model contains the `models.SlugField` attribute.  

```python
from django.contrib import admin
from . models import Article


@admin.register(Article)
class (admin.ModelAdmin):
    list_display = ['title', 'body', 'slug']
    list_filter = ['title', 'body', 'slug']
    search_fields = ['title', 'body']
    prepopulated_fields = {'slug': ('title',)}
```

The `ArticleAdmin` model causes the slug to be automatically generated when writing a  
title to our article using `prepopulated_fields`.  

```python
from django.shortcuts import render
from . models import Article


def index(request):

    articles = Article.objects.all()
    ctx = {'articles': articles}

    return render(request, 'index.html', ctx)


def show_article(request, slug):

    content = Article.objects.get(slug=slug)
    ctx = {'content': content}

    return render(request, 'show_article.html', context=ctx)
```

In the home page, we show all articles and in the `show_article.html` we show the contents of the  
currently selected article. We retrieve the articles by their slug via `Article.objects.get(slug=slug)`.  

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index),
    path('<slug:slug>', views.show_article, name='show-article'),
]
```

We use `<slug:slug>` to map the slug to the show-article view.  

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    
    {% for article in articles %}

       <a href="{{ article.slug }}">{{ article.title }}</a> <br>

    {% endfor %}

</body>

</html>
```

In the home page (`index.html`), we list all artiles. The links are the slugs of the articles.  

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>

    <h2>{{ content.title  }}</h2>
    

    {{ content.body }}

</body>
</html>
```

We display the title an the body of the article in `show_article.html`.    

## get_object_or_404

The `get_object_or_404` is a shortcut to send 404 page if an object is not found.  

```python
from django.http import Http404
from django.shortcuts import render, get_object_or_404


from .models import Product


def product_detail(request, pid):

    # try:
    #     product = Product.objects.get(id=pid)
    #     ctx = {'product': product}
    # except Product.DoesNotExist:
    #     raise Http404

    product = get_object_or_404(Product, id=pid)
    ctx = {'product': product}

    return render(request, 'product_detail.html', ctx)
```

We get a `Product` by its id.  

```python
from django.urls import path

from . import views

urlpatterns = [
    path("products/<pid>/", views.product_detail, name ="product-detail"),
]
```

The `products/<pid>/` path automaticall adds `pid` parameter to the `views.product_detail` function.  

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=8, decimal_places=2)


    def __str__(self):
        return self.name
```

A `Product` model.  

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>

    <p>
        {{ product.name}} - {{ product.price }}
    </p>
</body>

</html>
```

Display product name and price.  

