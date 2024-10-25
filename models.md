# Models

## id

By default, Django adds an `id` field to each model, which is used as the primary key  
for that model. You can create your own primary key field by adding the keyword arg  
`primary_key=True` to a field. If you add your own primary key field, the automatic  
one will not be added.  


## Ordering

```python
from django.db import models

# Create your models here.

class Message(models.Model):

    text = models.CharField(max_length=255)
    created= models.DateField(auto_now_add=False)

    class Meta:
            ordering = ['created']
```

## Setting table name

```python
from django.db import models

class Customer(models.Model):

    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)
    birth_number = models.CharField(max_length=255)
    
    def __str__(self):
        return f'{self.first_name} {self.last_name}'
    
    class Meta:
        db_table = 'customers'
```
