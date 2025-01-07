# Test client


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

```python
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
