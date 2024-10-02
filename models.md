# Models


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
