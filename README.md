# Django-Notes
Django framework notes

## Virtual Environment

Create Virtual Environment  

`py -m venv .venv`

Activate the virtual environment:  

On Windows: `.venv\Scripts\activate`  
On Unix or MacOS: `source ./.venv/bin/activate` 

Deactivate the virtual environment:    
`deactivate`

## Basic commands 

`pip install django` - install Django  
`django-admin startproject main ` - create new Django project  
`django-admin startproject main .` - create new Django project within current directory  
`py manage.py runserver` - run server  
`py manage.py createsuperuser` - create superuser  
`django-admin startapp myapp` - create new app  
`py manage.py makemigrations` - make migrations  
`py manage.py migrate` - run migrations  
`py manage.py flush` clean the database; need to create a new superuser  

## Include URLs from module (app)

In the main project's `urls.py` file, add the `path`.  

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('my.urls')),
]
```

## JSON serialize 

Use the `values` method to serialize Django models. The method returns a `QuerySet`   
that returns dictionaries, rather than model instances, when used as an iterable.   

In `models.py`:  

```python
from django.db import models

# Create your models here.

class Customer(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)
```

In `views.py`:  

```python
from django.http import JsonResponse

from .models import Customer

def home(req):

    customers = Customer.objects.all().values()
    data = {'customers': list(customers)}

    return JsonResponse(data)
```


## Custom 404 error message

In `settings.py`, set:

```python
DEBUG = False
ALLOWED_HOSTS = ['127.0.0.1', 'localhost',]
```

In main `urls.py`, set `handler404`:

```python
from django.conf.urls import handler404

...
handler404 = 'webapp.views.page_not_found'
```

In webapp's `views.py`:

```python
def page_not_found(req, exception):
    # return render(req, 'errors/error_404.html')
    return HttpResponse('404 - page not found', content_type='text/plain')
```

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

## PRG pattern

After submitting POST data, return `HttpResponseRedirect`.  

```python
return HttpResponseRedirect(reverse('home'))
```

## Generate secret key

```
py manage.py shell
```

```python
from django.core.management.utils import get_random_secret 

print(get_random_secret_key())
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

## DoesNotExist

Django's `DoesNotExist` exception is a specific exception raised when a queryset is empty.  
This typically occurs when we attempt to access an object that doesn't exist in the database.

In `urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path('<int:customer_id>/', views.detail),
]
```

In `models.py`:

```python
from django.db import models


class Customer(models.Model):

    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    email = models.EmailField()

    def __str__(self):
        
        return f'{self.first_name} {self.last_name}'
```

In `views.py`:

```python
from django.shortcuts import render
from django.http import HttpRequest, Http404


from . models import Customer


def detail(req: HttpRequest, customer_id: int):

    try:
        customer = Customer.objects.get(pk=customer_id)
    except Customer.DoesNotExist:
        raise Http404('Customer does not exist')

    ctx = {'customer': customer}

    return render(req, 'myapp/detail.html', ctx)
````

In `myapp/detail.py`: 

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Customer detail</title>
</head>
<body>

    <h2>Customer detail</h2>
    
    <ul>
        <li>{{ customer.first_name }}</li>
        <li>{{ customer.last_name }}</li>
        <li>{{ customer.email }}</li>
    </ul>


</body>
</html>
```


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

## Select tag

In `models.py`: 

```python
from django.db import models


class User(models.Model):

    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)

    def __str__(self):

        return f'{self.first_name} {self.last_name}'
```

In `views.py`:  

```python
from django.shortcuts import render
from django.http import HttpRequest

from .models import User


def show_users(req: HttpRequest):

    if req.method == 'POST':

        occupation = req.POST.get('occupation')
        if occupation == 'all':
            users = users = User.objects.all()
        else:

            users = User.objects.filter(occupation=occupation)

        occupations = User.objects.order_by().values_list(
            'occupation', flat=True).distinct()

        return render(req, 'index.html', {'users': users, 'options': occupations})

    else:

        occupations = User.objects.order_by().values_list(
            'occupation', flat=True).distinct()
        all = User.objects.all()

        return render(req, 'index.html', {'users': all, 'options': occupations})
```

Template: 

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Home page</title>
</head>

<body>

    <h2>Search users</h2>

    <form method="POST">

        {% csrf_token %}

        <select name="occupation">
            <option selected disabled="true">Select occupation</option>
            <option option="all">all</option>
            {% for option in options %}
            <option value="{{ option}}">{{ option }}</option>
            {% endfor %}
        </select>

        <input type="submit" value="Submit" />

        <table>
            <thead>
                <tr>
                    <td>First name</td>
                    <td>Last name</td>
                    <td>Occupation</td>
                </tr>

            </thead>

            {% for user in users %}
            <tr>
                <td>{{ user.first_name }}</td>
                <td>{{ user.last_name }}</td>
                <td>{{ user.occupation }}</td>
            </tr>

            {% endfor %}

        </table>

    </form>


</body>

</html>
```






