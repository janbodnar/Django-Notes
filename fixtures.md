# Fixtures

A *fixture* is a collection of files that contain the serialized contents of the database.  

Fixtures can be written as:

* JSON
* XML
* YAML (with PyYAML installed)


Django will search in these locations for fixtures:

* In the fixtures directory of every installed application
* In any directory listed in the `FIXTURE_DIRS` setting
* In the literal path named by the fixture

`py manage.py loaddata mydata.json` looks for `mydata.json` fixture in the aforementioned locations.  
`py manage.py loaddata mydata` looks for fixture with JSON, XML, or YAML formats.  

Test data `customers.yaml`:  

```yaml
- model: testapp.customer
  pk: 1
  fields:
    first_name: John
    last_name: Doe
    occupation: gardener
- model: testapp.customer
  pk: 2
  fields:
    first_name: Roger
    last_name: Roe
    occupation: driver
- model: testapp.customer
  pk: 3
  fields:
    first_name: Paul
    last_name: Smith
    occupation: teacher
- model: testapp.customer
  pk: 4
  fields:
    first_name: Lucia
    last_name: Novak
    occupation: teacher
```

Load data with `python manage.py loaddata customers.yaml`


In `models.py`:

```python
from django.db import models

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
