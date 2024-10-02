# Management commands

First, we need to manually create `management/commands` directories inside an app.  

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Displays current time'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.NOTICE('Hello!'))
```

This command shows a hello message.  
