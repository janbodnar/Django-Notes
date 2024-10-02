# Admin 

Available at: `http://127.0.0.1:8000/admin/`. 


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
