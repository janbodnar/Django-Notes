# Smoke tests


Smoke tests are a type of preliminary test to check the basic functionality of an application. They're  
like a quick check-up to ensure that the essential features of your application are working correctly  
before you dive into more extensive testing. Think of it as making sure the car starts and runs before  
you head out on a road trip.

The `models.py` file:  

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField()

    def __str__(self):
        return self.name
```

The `tests.py` file:

```python
from django.test import TestCase, tag
from .models import Product

class ProductSmokeTest(TestCase):

    @tag('smoke')
    def test_product_creation(self):
        product = Product.objects.create(
            name="Test Product",
            price=10.00,
            stock=100
        )
        self.assertIsNotNone(product.id)

    @tag('smoke')
    def test_product_str(self):
        product = Product.objects.create(
            name="Test Product",
            price=10.00,
            stock=100
        )
        self.assertEqual(str(product), "Test Product")

    @tag('smoke')
    def test_product_price(self):
        product = Product.objects.create(
            name="Test Product",
            price=10.00,
            stock=100
        )
        self.assertEqual(product.price, 10.00)
```

Running the smoke tests: `python manage.py test --tag=smoke`. 
