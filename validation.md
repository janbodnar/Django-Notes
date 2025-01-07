# Validation

## Simple regex validation

```python
from django.db import models
from django.core.validators import RegexValidator


class Customer(models.Model):

    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)
    birth_number = models.CharField(max_length=255, validators=[
                                    RegexValidator(f'^\d\d\d\d\d\d/\d\d\d\d$', message="wrong birth number")])
```


## Clean a specific field 

```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=255)
```

Simple model having a single `name` field.  

```python
from django.core.exceptions import ValidationError
from django import forms
from . models import Product


class ProductForm(forms.ModelForm):

    class Meta:
        model = Product
        fields = ('name', )

    def clean_name(self):
        name: str = self.cleaned_data.get('name')

        if not name.isascii():
            raise ValidationError('Product name must be in ASCII letters')

        return name
```

The `client_name` is automatically called for the `name` field.  

```python
from django.shortcuts import render
from django.http import HttpResponseRedirect
from .forms import ProductForm


def index(request):

    if request.method == 'POST':
        form = ProductForm(request.POST)
        if form.is_valid():
            form.save()
            return HttpResponseRedirect('product-added')

    else:
        form = ProductForm()

    ctx = {'form': form}

    return render(request, 'index.html', ctx)


def product_added(request):

    return render(request, 'product_added.html')
```

Generating the `ProductForm`.   


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Form</title>
</head>
<body>

    <h2>Customer form</h2>

    <form action="" method="post">
        {% csrf_token %}
        {{form}}
        <input type="submit" value="Submit">

    </form>
    
</body>
</html>
```

Displaying the form.  


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
        Product added successfully
    </p>
    
</body>
</html>
```

