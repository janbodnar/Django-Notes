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


## seed command 

```python
from django.db import models

# Create your models here.

class Message(models.Model):

    text = models.CharField(max_length=255)
    created= models.DateField(auto_now_add=False)
```

```python
from django.core.management.base import BaseCommand

from faker import Faker

# from myapp.models import Message
from ... models import Message


class Command(BaseCommand):
    help = 'seed data'

    def add_arguments(self, parser):

        group = parser.add_mutually_exclusive_group()

        group.add_argument('total', type=int, default=10,
                            help='# of messages to create', nargs='?')

        group.add_argument('-c', '--clear', action='store_true', help='clear messages')


    def handle(self, *args, **kwargs):

        clear_flag = kwargs['clear']

        if clear_flag:
            Message.objects.all().delete()

            return

        total = kwargs['total']

        faker = Faker()

        for _ in range(total):

            msg = faker.text(max_nb_chars=30)
            created = faker.date_between('-10y', 'today')
            message = Message(text=msg, created=created)
            message.save()

        self.stdout.write(self.style.SUCCESS(f'{total} messages generated'))
```
