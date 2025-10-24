# Fixtures

A *fixture* is a collection of files that contain the serialized contents of the database.  
Fixtures are used to populate the database with predefined data, which is especially  
useful for testing, development, and initial application setup.

## Why use fixtures?

* **Testing** - Provide consistent test data across different test runs
* **Development** - Quickly populate a development database with sample data
* **Initial setup** - Seed a new database with essential data (e.g., default users, categories)
* **Sharing data** - Transfer data between different environments or team members
* **Version control** - Keep test/demo data in version control alongside code

## Fixture formats

Fixtures can be written as:

* **JSON** - Most commonly used, readable and widely supported
* **XML** - Alternative format, more verbose but structured
* **YAML** - Human-friendly format (requires PyYAML: `pip install pyyaml`)


## Fixture location

Django will search in these locations for fixtures:

1. In the `fixtures` directory of every installed application
2. In any directory listed in the `FIXTURE_DIRS` setting
3. In the literal path named by the fixture

### Examples

`python manage.py loaddata mydata.json` - looks for `mydata.json` fixture in the aforementioned locations  
`python manage.py loaddata mydata` - looks for fixture with JSON, XML, or YAML formats (tries all extensions)  
`python manage.py loaddata /path/to/mydata.json` - loads from absolute path  

## Fixture directory structure

Recommended structure for organizing fixtures in a Django app:

```
myapp/
├── fixtures/
│   ├── initial_data.json      # Loaded automatically with syncdb (Django < 1.9)
│   ├── test_data.json         # Test fixtures
│   └── sample_customers.yaml  # Sample data for development
├── models.py
└── views.py
```

## Example fixtures

### YAML format

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

### JSON format

The same data in JSON format `customers.json`:

```json
[
  {
    "model": "testapp.customer",
    "pk": 1,
    "fields": {
      "first_name": "John",
      "last_name": "Doe",
      "occupation": "gardener"
    }
  },
  {
    "model": "testapp.customer",
    "pk": 2,
    "fields": {
      "first_name": "Roger",
      "last_name": "Roe",
      "occupation": "driver"
    }
  },
  {
    "model": "testapp.customer",
    "pk": 3,
    "fields": {
      "first_name": "Paul",
      "last_name": "Smith",
      "occupation": "teacher"
    }
  }
]
```

Load with `python manage.py loaddata customers.json`

### XML format

The same data in XML format `customers.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<django-objects version="1.0">
  <object model="testapp.customer" pk="1">
    <field name="first_name" type="CharField">John</field>
    <field name="last_name" type="CharField">Doe</field>
    <field name="occupation" type="CharField">gardener</field>
  </object>
  <object model="testapp.customer" pk="2">
    <field name="first_name" type="CharField">Roger</field>
    <field name="last_name" type="CharField">Roe</field>
    <field name="occupation" type="CharField">driver</field>
  </object>
</django-objects>
```

Load with `python manage.py loaddata customers.xml`

## Model definition

In `models.py`:

```python
from django.db import models

class Customer(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)

    def __str__(self):
        return f'{self.first_name} {self.last_name}'
```


## Using fixtures in views

In `views.py`:

```python
from django.http import JsonResponse

from .models import Customer

def home(req):

    customers = Customer.objects.all().values()
    data = {'customers': list(customers)}

    return JsonResponse(data)
```

## Fixtures in testing

Fixtures are commonly used in Django tests to provide consistent test data.

### Using fixtures in test classes

```python
from django.test import TestCase
from .models import Customer

class CustomerTestCase(TestCase):
    fixtures = ['customers.json']  # Automatically loaded before each test

    def test_customer_count(self):
        # Test data from fixture is available
        self.assertEqual(Customer.objects.count(), 4)

    def test_customer_name(self):
        customer = Customer.objects.get(pk=1)
        self.assertEqual(customer.first_name, 'John')
        self.assertEqual(customer.last_name, 'Doe')
```

### Loading fixtures in setUp method

```python
from django.test import TestCase
from django.core.management import call_command

class CustomerTestCase(TestCase):
    
    def setUp(self):
        # Load fixtures programmatically
        call_command('loaddata', 'customers.json', verbosity=0)

    def test_customers_loaded(self):
        self.assertTrue(Customer.objects.exists())
```

## Dumping data 

Export database data to fixtures using the `dumpdata` command.

### Basic dumpdata commands

`python manage.py dumpdata > db.json` - dump entire database  
`python manage.py dumpdata testapp > testapp.json` - dump specific app  
`python manage.py dumpdata testapp.customer > customers.json` - dump specific model  
`python manage.py dumpdata --exclude auth.permission > db.json` - exclude specified apps/models  
`python manage.py dumpdata --exclude contenttypes --exclude auth.permission > db.json` - exclude multiple  

### Format options

`python manage.py dumpdata --format yaml testapp.customer > customers.yaml` - export as YAML  
`python manage.py dumpdata --format xml testapp.customer > customers.xml` - export as XML  
`python manage.py dumpdata --format json --indent 2 testapp.customer > customers.json` - pretty print JSON  
`python manage.py dumpdata --format xml --indent 4 testapp.customer > customers.xml` - pretty print XML  

### Advanced dumpdata options

`python manage.py dumpdata --natural-foreign` - use natural foreign keys  
`python manage.py dumpdata --natural-primary` - use natural primary keys  
`python manage.py dumpdata --all` - include models that would otherwise be excluded  
`python manage.py dumpdata -e contenttypes -e auth.permission --indent 2 > db.json` - short form exclude  

### Filtering by primary key

```bash
# Export specific records by filtering in the database first
python manage.py dumpdata testapp.customer --pks=1,2,3 > specific_customers.json
```

## Best practices

1. **Version control** - Keep fixtures in version control to track changes over time
2. **Small fixtures** - Create small, focused fixtures for specific test scenarios
3. **Avoid sensitive data** - Never include passwords or sensitive information in fixtures
4. **Use natural keys** - Consider using natural keys for better readability and maintainability
5. **Exclude unnecessary data** - Exclude contenttypes, permissions, and sessions when not needed
6. **Document fixtures** - Add comments or README files explaining what each fixture contains
7. **Separate concerns** - Keep test fixtures separate from initial/sample data fixtures

## Common use cases

### Initial application data

Create fixtures for default data that should be present in every installation:

```bash
python manage.py dumpdata myapp.category myapp.settings --indent 2 > myapp/fixtures/initial_data.json
```

### Test data for continuous integration

Create comprehensive test data for CI/CD pipelines:

```bash
python manage.py dumpdata --exclude contenttypes --exclude auth.permission \
  --exclude sessions --indent 2 > fixtures/test_data.json
```

### Sample data for development

Provide sample data to help developers set up their environment:

```bash
python manage.py dumpdata --natural-foreign --natural-primary \
  --indent 2 myapp > fixtures/sample_data.json
```

## Troubleshooting

### Fixture loading errors

**Problem**: `DeserializationError: Invalid model identifier`  
**Solution**: Ensure the model name in the fixture matches the app label and model name exactly.

**Problem**: `IntegrityError: duplicate key value violates unique constraint`  
**Solution**: Clear existing data before loading: `python manage.py flush` or delete specific records.

**Problem**: `DoesNotExist: Matching query does not exist`  
**Solution**: Check foreign key references in fixtures - ensure referenced objects exist or are defined earlier.

### Circular dependencies

When models have circular foreign key relationships, load fixtures in multiple passes:

```bash
python manage.py loaddata authors.json
python manage.py loaddata books.json
python manage.py loaddata author_books.json
```

Or use natural keys to avoid primary key conflicts.

### Natural keys

Natural keys allow you to reference objects by unique fields instead of primary keys:

In `models.py`:

```python
from django.db import models

class CountryManager(models.Manager):
    def get_by_natural_key(self, code):
        return self.get(code=code)

class Country(models.Model):
    objects = CountryManager()
    
    code = models.CharField(max_length=2, unique=True)
    name = models.CharField(max_length=100)
    
    def natural_key(self):
        return (self.code,)
```

Dump with natural keys:

```bash
python manage.py dumpdata --natural-foreign --natural-primary myapp.country
```

This creates more readable and maintainable fixtures that reference objects by meaningful identifiers.

## Loading multiple fixtures

Load multiple fixtures at once:

```bash
python manage.py loaddata fixture1.json fixture2.json fixture3.json
```

Or load all fixtures in a directory:

```bash
python manage.py loaddata myapp/fixtures/*.json
```

## Automating fixture loading

### Using Django signals

Automatically load fixtures after migrations:

```python
from django.db.models.signals import post_migrate
from django.core.management import call_command
from django.dispatch import receiver

@receiver(post_migrate)
def load_initial_data(sender, **kwargs):
    if sender.name == 'myapp':
        call_command('loaddata', 'initial_data.json', verbosity=0)
```

### In management commands

Create a custom management command to load fixtures:

```python
from django.core.management.base import BaseCommand
from django.core.management import call_command

class Command(BaseCommand):
    help = 'Load all test fixtures'

    def handle(self, *args, **kwargs):
        fixtures = ['users.json', 'customers.json', 'products.json']
        for fixture in fixtures:
            self.stdout.write(f'Loading {fixture}...')
            call_command('loaddata', fixture)
        self.stdout.write(self.style.SUCCESS('All fixtures loaded successfully'))
```


