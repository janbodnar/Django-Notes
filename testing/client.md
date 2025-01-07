# Test client


Django's test client is a Python class that allows you to simulate requests to your Django application  
and test your views and their interactions with your models and templates. It's a powerful tool for  
performing integration testing on your web application.

## Key Features of Django's Test Client

1. **Simulate HTTP Requests**:
   - The test client can simulate GET, POST, and other HTTP requests to your views.
   - This allows you to test how your views handle different types of requests and user input.

2. **Test Response Status and Content**:
   - You can check the status codes (e.g., 200 OK, 404 Not Found) returned by your views.
   - You can inspect the content of the responses, including rendered templates and JSON data.

3. **Session and Cookies**:
   - The test client can simulate sessions and cookies, allowing you to test views that rely on user  
     authentication or session data.

4. **Follow Redirects**:
   - The test client can automatically follow redirects, making it easier to test views that  
     redirect to other views.

### Basic Usage

Here’s a simple example to illustrate how you might use Django's test client:

```python
from django.test import TestCase
from django.urls import reverse

class SimpleTest(TestCase):

    def test_homepage(self):
        response = self.client.get(reverse('home'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Welcome to the homepage!")

    def test_form_submission(self):
        response = self.client.post(reverse('submit_form'), {
            'name': 'Test User',
            'email': 'test@example.com'
        })
        self.assertEqual(response.status_code, 302)  # Assuming it redirects after submission
        self.assertRedirects(response, reverse('success_page'))
```

- `self.client.get(...)` and `self.client.post(...)`: These methods simulate GET and POST requests to the specified URL.
- `reverse('home')`: This function helps you get the URL from the view name.
- `self.assertEqual(response.status_code, 200)`: This assertion checks that the status code of the response is 200 (OK).
- `self.assertContains(response, "Welcome to the homepage!")`: This checks that the response content includes the specified text.
- `self.assertRedirects(response, reverse('success_page'))`: This assertion checks that the response is a redirect to the specified URL.




## Simple example

Simple example testing status code, template used, and displayed content. 

The `Product` model in `models.py`:

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField()

    def __str__(self):
        return self.name
```

The mappings in `views.py`: 

```python
from django.shortcuts import render
from .models import Product

def product_list(request):
    products = Product.objects.all()
    return render(request, 'product_list.html', {'products': products})
```

The `product_list.html` in templates:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Product List</title>
</head>
<body>
    <h1>Product List</h1>
    <ul>
        {% for product in products %}
            <li>{{ product.name }} - ${{ product.price }}</li>
        {% endfor %}
    </ul>
</body>
</html>
```

The tests in `tests.py`:

```python
from django.test import TestCase, Client
from django.urls import reverse
from .models import Product

class ProductViewTest(TestCase):

    def setUp(self):
        self.client = Client()
        self.product1 = Product.objects.create(
            name="Product 1",
            price=10.00,
            stock=100
        )
        self.product2 = Product.objects.create(
            name="Product 2",
            price=20.00,
            stock=200
        )
        self.product3 = Product.objects.create(
            name="Product 3",
            price=30.00,
            stock=300
        )

    def test_product_list_status_code(self):
        response = self.client.get(reverse('product_list'))
        self.assertEqual(response.status_code, 200)

    def test_product_list_template_used(self):
        response = self.client.get(reverse('product_list'))
        self.assertTemplateUsed(response, 'product_list.html')

    def test_product_list_content(self):
        response = self.client.get(reverse('product_list'))
        self.assertContains(response, "Product 1")
        self.assertContains(response, "Product 2")
        self.assertContains(response, "Product 3")
```
