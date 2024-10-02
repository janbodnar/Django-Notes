# Admin 

Available at: `http://127.0.0.1:8000/admin/`. 


To access the admin account, we need to create a superuser:  
`py manage.py createsuperuser - create superuser`


## Create pagination

```python
from django.contrib import admin

from . models import Message

# Register your models here.

class MessageAdmin(admin.ModelAdmin):
    
    list_display = ['text']
    list_per_page = 10


admin.site.register(Message, MessageAdmin)
```
