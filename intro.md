# Introduction

Django is a high-level Python web framework that enables rapid development  
of secure and maintainable websites. Built by experienced developers, Django  
takes care of much of the hassle of web development, so you can focus on  
writing your application without needing to reinvent the wheel. It's free,  
open source, and follows the model-template-view (MTV) architectural pattern.  
Django emphasizes the DRY (Don't Repeat Yourself) principle and focuses on  
automating as much as possible while adhering to best practices in web  
development.

## History

Django was created in 2003 by Adrian Holovaty and Simon Willison while working  
at the Lawrence Journal-World newspaper in Kansas. The framework was developed  
to meet the fast-paced demands of a newsroom environment, where new features  
needed to be implemented quickly and reliably. In 2005, Django was released to  
the public under a BSD license, making it free and open-source software.

The name "Django" was chosen as a tribute to jazz guitarist Django  
Reinhardt, reflecting the framework's emphasis on elegance and efficiency.  
Since its release, Django has grown into one of the most popular web  
frameworks, with a large and active community contributing to its  
development. The Django Software Foundation, established in 2008, oversees  
the framework's development and promotes its use worldwide.

Major milestones include the release of version 1.0 in 2008, which marked the  
framework's maturity and production-readiness, and subsequent versions that  
have added support for modern web technologies, asynchronous views, improved  
security features, and better performance. Django follows a time-based release  
schedule with new major versions released approximately every eight months.

## Goals and Philosophy

Django was designed with several core goals and principles that guide its  
development and make it distinctive among web frameworks:

**Rapid Development**: Django enables developers to build applications quickly  
by providing built-in features for common web development tasks. The framework  
includes an ORM, authentication system, admin interface, and form handling out  
of the box.

**Security**: Security is a top priority in Django. The framework provides  
protection against many common security threats including SQL injection,  
cross-site scripting (XSS), cross-site request forgery (CSRF), and  
clickjacking. It encourages secure defaults and makes it easy to implement  
proper security practices.

**Scalability**: Django is designed to help developers build applications that  
can scale from small personal projects to large enterprise systems. It uses a  
"shared-nothing" architecture, where components can be easily separated and  
scaled independently.

**Versatility**: While originally designed for content-heavy websites, Django  
has proven to be versatile enough to build any type of web application, from  
APIs to social networks, scientific computing platforms to financial systems.

**Don't Repeat Yourself (DRY)**: Django promotes code reusability and the  
elimination of redundancy. The framework encourages developers to write code  
once and reuse it throughout the application.

**Explicit is Better Than Implicit**: Following Python's philosophy, Django  
values clarity and readability. The framework strives to make its behavior  
predictable and easy to understand.

## Use Cases

Django excels in a wide variety of web development scenarios:

**Content Management Systems**: Django's robust admin interface and flexible  
content modeling make it ideal for CMS applications. Major news organizations  
and content publishers use Django for their platforms.

**E-commerce Platforms**: With its secure handling of transactions, user  
authentication, and data validation, Django is well-suited for building online  
stores and marketplace applications.

**Social Networks and Community Platforms**: Django's user management, media  
handling, and scalability features make it a strong choice for social  
networking sites and online communities.

**Scientific and Research Platforms**: Many scientific institutions use Django  
to build data analysis tools, research databases, and scientific computing  
platforms due to its integration with Python's scientific libraries.

**REST APIs and Web Services**: With packages like Django REST Framework,  
Django is excellent for building robust RESTful APIs that power mobile apps  
and single-page applications.

**Financial Applications**: Django's emphasis on security, data integrity, and  
transaction handling makes it suitable for banking, fintech, and payment  
processing applications.

**Educational Platforms**: Learning management systems and online education  
platforms often choose Django for its flexibility and comprehensive feature set.

Notable companies and organizations using Django include Instagram, Mozilla,  
NASA, National Geographic, Pinterest, Spotify, The Washington Post, and many  
more.

## Advantages Over Other Frameworks

Django offers several distinct advantages compared to other web frameworks:

**Batteries Included**: Unlike minimalist frameworks like Flask or FastAPI,  
Django comes with everything you need out of the box - ORM, admin interface,  
authentication, forms, and more. This reduces the time spent choosing and  
integrating third-party packages.

**Powerful ORM**: Django's Object-Relational Mapping system is one of the most  
mature and feature-rich in any web framework. It supports complex queries,  
migrations, and works with multiple database backends without code changes.

**Admin Interface**: Django automatically generates a professional,  
production-ready admin interface from your models. This feature alone can  
save weeks of development time and is unique among major web frameworks.

**Strong Security Defaults**: While frameworks like Express.js or Ruby on  
Rails require additional configuration for security, Django provides secure  
defaults out of the box and actively maintains security updates.

**Comprehensive Documentation**: Django's documentation is widely considered  
the gold standard for framework documentation - clear, comprehensive, and  
well-maintained with examples for both beginners and advanced users.

**Mature Ecosystem**: With over 15 years of active development, Django has a  
vast ecosystem of reusable packages, a large community, and proven scalability  
in production environments.

**Python Integration**: Built on Python, Django leverages one of the most  
popular and versatile programming languages, with access to extensive  
libraries for data science, machine learning, automation, and more.

**Convention Over Configuration**: While maintaining flexibility, Django  
provides sensible defaults and conventions that reduce boilerplate code and  
decision fatigue, unlike frameworks that require extensive configuration.

Compared to frameworks like Laravel (PHP), Django offers better performance  
and access to Python's scientific computing stack. Compared to Express.js  
(Node.js), Django provides more built-in features and a synchronous  
programming model that many developers find easier to reason about. Compared  
to Flask, Django offers more structure and built-in features, making it  
better for larger projects.

## Setting Up a Classic Django Web Application

This example demonstrates how to create a traditional server-rendered Django  
web application using `uv` for package management.

First, install `uv` if you haven't already:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Create a new project directory and initialize a Python project with `uv`:

```bash
mkdir mywebapp
cd mywebapp
uv init
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

Install Django (latest version) using `uv`:

```bash
uv add django
```

Create a new Django project:

```bash
uv run django-admin startproject mysite .
```

The project structure will look like this:

```
mywebapp/
├── .venv/
├── mysite/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
├── manage.py
└── pyproject.toml
```

Create a Django app for your application logic:

```bash
uv run python manage.py startapp blog
```

Register the app in `mysite/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
]
```

Create a model in `blog/models.py`:

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title
```

The `Post` model represents a blog post with a title, content, and creation  
timestamp. Django will automatically generate the corresponding database table.

Create and apply migrations:

```bash
uv run python manage.py makemigrations
uv run python manage.py migrate
```

The `makemigrations` command creates migration files based on model changes,  
while `migrate` applies those migrations to the database.

Create a view in `blog/views.py`:

```python
from django.shortcuts import render
from .models import Post

def post_list(request):
    posts = Post.objects.all().order_by('-created_at')
    return render(request, 'blog/post_list.html', {'posts': posts})
```

This view retrieves all blog posts ordered by creation date and passes them to  
a template for rendering.

Create a template directory and file `blog/templates/blog/post_list.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Blog Posts</title>
    <style>
        body { 
            font-family: Arial, sans-serif; 
            max-width: 800px; 
            margin: 0 auto; 
            padding: 20px; 
        }
        .post { 
            border-bottom: 1px solid #ccc; 
            padding: 20px 0; 
        }
        .post h2 { 
            margin: 0 0 10px 0; 
        }
        .post-date { 
            color: #666; 
            font-size: 0.9em; 
        }
    </style>
</head>
<body>
    <h1>My Blog</h1>
    {% for post in posts %}
    <div class="post">
        <h2>{{ post.title }}</h2>
        <p class="post-date">{{ post.created_at|date:"F d, Y" }}</p>
        <p>{{ post.content }}</p>
    </div>
    {% empty %}
    <p>No posts yet.</p>
    {% endfor %}
</body>
</html>
```

The template uses Django's template language to iterate over posts and display  
them with proper formatting. The `date` filter formats the timestamp.

Configure URLs in `blog/urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.post_list, name='post-list'),
]
```

Include the app URLs in `mysite/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('blog.urls')),
]
```

Register the model with the admin interface in `blog/admin.py`:

```python
from django.contrib import admin
from .models import Post

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'created_at']
    search_fields = ['title', 'content']
```

The admin registration allows you to manage blog posts through Django's built-  
in admin interface without writing any additional CRUD views.

Create a superuser to access the admin:

```bash
uv run python manage.py createsuperuser
```

Run the development server:

```bash
uv run python manage.py runserver
```

Access the application at `http://127.0.0.1:8000/` and the admin interface  
at `http://127.0.0.1:8000/admin/`. You can now create blog posts through the  
admin and view them on the main page.

## Setting Up a Django REST API Application

This example demonstrates how to create a RESTful API using Django REST  
Framework with `uv` for package management.

Create a new project directory and initialize with `uv`:

```bash
mkdir myrestapi
cd myrestapi
uv init
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
```

Install Django and Django REST Framework:

```bash
uv add django
uv add djangorestframework
```

Django REST Framework (DRF) is a powerful toolkit for building Web APIs with  
Django. It provides serialization, authentication, viewsets, and browsable API  
interface out of the box.

Create a Django project:

```bash
uv run django-admin startproject api_project .
```

Create an app for the API:

```bash
uv run python manage.py startapp products
```

Configure installed apps in `api_project/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'products',
]

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 
        'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}
```

The `REST_FRAMEWORK` settings configure pagination, authentication, and  
permissions for the API. These defaults allow read access to anyone but  
require authentication for write operations.

Create a model in `products/models.py`:

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.name
```

The `Product` model includes fields for name, description, price, and stock  
quantity. The `Meta` class specifies default ordering by creation date.

Create migrations and apply them:

```bash
uv run python manage.py makemigrations
uv run python manage.py migrate
```

Create a serializer in `products/serializers.py`:

```python
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'stock', 
                  'created_at', 'updated_at']
        read_only_fields = ['created_at', 'updated_at']
    
    def validate_price(self, value):
        if value < 0:
            raise serializers.ValidationError(
                "Price must be a positive number"
            )
        return value
    
    def validate_stock(self, value):
        if value < 0:
            raise serializers.ValidationError(
                "Stock must be a non-negative number"
            )
        return value
```

Serializers convert complex Django models to JSON and vice versa. The  
`ModelSerializer` automatically generates fields based on the model. Custom  
validation ensures data integrity by checking that prices and stock values are  
valid.

Create API views in `products/views.py`:

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    @action(detail=False, methods=['get'])
    def low_stock(self, request):
        low_stock_products = Product.objects.filter(stock__lt=10)
        serializer = self.get_serializer(low_stock_products, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def restock(self, request, pk=None):
        product = self.get_object()
        quantity = request.data.get('quantity', 0)
        product.stock += int(quantity)
        product.save()
        serializer = self.get_serializer(product)
        return Response(serializer.data)
```

The `ModelViewSet` provides complete CRUD operations automatically. The  
`low_stock` action returns products with stock below 10 units, while `restock`  
increases a product's stock quantity. These custom actions extend the API  
beyond basic CRUD operations.

Configure URLs in `products/urls.py`:

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = [
    path('', include(router.urls)),
]
```

The `DefaultRouter` automatically generates URL patterns for the viewset,  
creating endpoints for list, create, retrieve, update, and delete operations.

Include API URLs in `api_project/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('products.urls')),
    path('api-auth/', include('rest_framework.urls')),
]
```

The `api-auth/` endpoint provides login/logout functionality for the browsable  
API interface.

Register the model with admin in `products/admin.py`:

```python
from django.contrib import admin
from .models import Product

@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ['name', 'price', 'stock', 'created_at']
    list_filter = ['created_at']
    search_fields = ['name', 'description']
```

Create a superuser:

```bash
uv run python manage.py createsuperuser
```

Run the development server:

```bash
uv run python manage.py runserver
```

The API will be available at `http://127.0.0.1:8000/api/`. The browsable API  
interface allows you to interact with endpoints directly from your browser.

Available endpoints include:
- `GET /api/products/` - List all products
- `POST /api/products/` - Create a new product
- `GET /api/products/{id}/` - Retrieve a specific product
- `PUT /api/products/{id}/` - Update a product
- `PATCH /api/products/{id}/` - Partial update
- `DELETE /api/products/{id}/` - Delete a product
- `GET /api/products/low_stock/` - List low stock products
- `POST /api/products/{id}/restock/` - Restock a product

You can test the API using curl, httpie, or any HTTP client:

```bash
# List all products
curl http://127.0.0.1:8000/api/products/

# Create a new product
curl -X POST http://127.0.0.1:8000/api/products/ \
  -H "Content-Type: application/json" \
  -d '{"name":"Laptop","description":"Gaming laptop",\
"price":"999.99","stock":50}'

# Get low stock products
curl http://127.0.0.1:8000/api/products/low_stock/
```

This REST API provides a solid foundation that can be extended with additional  
features like authentication tokens, filtering, searching, custom permissions,  
and more complex business logic as needed.

