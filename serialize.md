# Serialization 

Transform data into JSON

## values method

We can utilize `values` method and/or list comprehensions.  

Model:

```python
from django.db import models

class User(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)

    def __str__(self):
        return f'{self.first_name} {self.last_name} {self.occupation}'
```

```python
def fetch_data(request):

    dop = User.objects.filter(
        Q(occupation__startswith='d') | Q(occupation__startswith='p'))

    # res = {'users': list(dop.values('first_name', 'last_name', 'occupation'))}
    res = {'users': [{'first_name': e.first_name,
                      'last_name': e.last_name, 'occupation': e.occupation} for e in dop]}

    return JsonResponse(res)
```

or 

```python
from django.http import JsonResponse

from . models import Car

def home(req):

    all = Car.objects.all()
    ctx = {'cars': list(all.values())}

    return JsonResponse(ctx)
```

or 

```python
def fetch_data(request):

    dop = User.objects.filter(
        Q(occupation__startswith='d') | Q(occupation__startswith='p'))
    res = {"users": [str(user) for user in dop.all()]}

    return JsonResponse(res)
```
