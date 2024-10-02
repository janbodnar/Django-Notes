# Management commands

First, we need to manually create `management/commands` directories inside an app.  

The filename is `hello.py`:

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Displays hello message'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.NOTICE('Hello!'))
```

The command is then launched with `py manage.py hello`. 

Run `py manage.py` to list all available commands.  
