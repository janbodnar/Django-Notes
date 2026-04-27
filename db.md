# Database

Django provides a full-featured database layer covering the ORM, raw SQL,  
migrations, and transactions.  

Django's database layer is one of the framework's most powerful components.  
It provides a complete abstraction over relational databases through its  
Object-Relational Mapping (ORM) system, while still giving developers the  
ability to drop down to raw SQL whenever precision or performance demands it.  

At the heart of the database layer is the concept of a *model* — a Python  
class that maps directly to a database table. Django automatically handles  
the creation and modification of tables through its migration system, meaning  
you rarely need to write DDL (Data Definition Language) by hand. Queries are  
expressed as chained Python method calls on `QuerySet` objects, which are  
translated to SQL at execution time and remain lazy until data is actually  
needed.  

Django supports multiple database backends out of the box: SQLite (the default  
for development), PostgreSQL, MySQL and MariaDB, and Oracle. Each backend is  
accessed through a thin adapter layer, so switching databases requires only a  
change in settings and the installation of the appropriate Python driver.  

The database layer also encompasses:  

- **Migrations** — version-controlled schema changes tracked in Python files  
- **Transactions** — atomic blocks that guarantee data consistency  
- **Raw SQL** — escape hatches that let you execute hand-written queries while  
  still integrating cleanly with the ORM  
- **Query optimisation tools** — `select_related`, `prefetch_related`, and  
  database indexes defined directly on model classes  

Understanding the database layer end-to-end — from the initial settings  
configuration all the way to advanced ORM techniques and raw SQL — is  
essential for writing efficient, reliable Django applications.  


## Supported databases

Django ships with four built-in backend drivers.  

| Database   | ENGINE value                          | Python driver    |
|------------|---------------------------------------|------------------|
| SQLite     | `django.db.backends.sqlite3`          | built-in         |
| PostgreSQL | `django.db.backends.postgresql`       | `psycopg2` / `psycopg` |
| MySQL      | `django.db.backends.mysql`            | `mysqlclient`    |
| MariaDB    | `django.db.backends.mysql`            | `mysqlclient`    |
| Oracle     | `django.db.backends.oracle`           | `cx_Oracle`      |

SQLite requires no additional installation and is the default when you create  
a new project with `django-admin startproject`. It writes the entire database  
to a single file on disk, making it ideal for local development and small  
embedded applications. For production workloads, PostgreSQL is generally the  
recommended choice due to its robustness, concurrency model, and rich feature  
set.  


## Installing database adapters

Install the appropriate adapter before changing your `DATABASES` setting.  

**PostgreSQL**: use `psycopg2` (the classic C-extension adapter):  

```bash
uv add psycopg2-binary
```

Or use the pure-Python `psycopg` (version 3):  

```bash
uv add psycopg[binary]
```

**MySQL / MariaDB**:  

```bash
uv add mysqlclient
```

**Oracle**:  

```bash
uv add cx_Oracle
```

The `psycopg2-binary` package bundles the native C library and is the  
easiest way to get started. In production, prefer building `psycopg2`  
from source (without the `-binary` suffix) so that it links against the  
system's `libpq` and picks up security updates automatically.  


## The DATABASES setting

All database configuration lives in the `DATABASES` dictionary in  
`settings.py`. The dictionary maps alias names to connection parameters.  
The key `'default'` is the database used when no alias is specified  
explicitly.  

### SQLite (default)

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

Django creates the SQLite file automatically on the first `migrate` run.  
The `NAME` value is a `Path` object pointing to the database file.  

### PostgreSQL setup

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'postgres',
        'PASSWORD': 's$cret',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

`HOST` defaults to an empty string, which tells `psycopg2` to connect via a  
Unix-domain socket. Set it to `'localhost'` to force a TCP connection.  
`PORT` defaults to `''`, which uses the database engine's standard port.  

### MySQL setup

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mydb',
        'USER': 'root',
        'PASSWORD': 's$cret',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'OPTIONS': {
            'charset': 'utf8mb4',
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
        },
    }
}
```

The `OPTIONS` key passes extra parameters directly to the underlying  
database driver. For MySQL, setting `charset` to `utf8mb4` ensures full  
Unicode support including emoji storage. `STRICT_TRANS_TABLES` makes MySQL  
behave more like PostgreSQL by rejecting invalid data instead of silently  
truncating values.  

### Connection options

All backends accept a common set of keys in `OPTIONS`:  

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'postgres',
        'PASSWORD': 's$cret',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 60,        # seconds; 0 = close after each request
        'CONN_HEALTH_CHECKS': True, # verify connection before reuse
        'OPTIONS': {
            'connect_timeout': 10,
            'sslmode': 'require',
        },
        'TEST': {
            'NAME': 'test_mydb',   # name of the test database
        },
    }
}
```

`CONN_MAX_AGE` controls persistent connections: a value of `0` (the default)  
closes the database connection after every request, while any positive  
integer reuses it for that many seconds. `CONN_HEALTH_CHECKS`, introduced in  
Django 4.1, automatically discards stale persistent connections before they  
are used in a new request.  

The `TEST` sub-dictionary lets you customise the temporary database that  
Django creates when running the test suite. By default, Django prefixes the  
production database name with `test_`.  


## Environment-based settings

Hardcoding credentials in `settings.py` is a security risk. The recommended  
approach is to read sensitive values from environment variables, either via  
the `python-decouple` package or Django's built-in `os.environ`.  

### Using python-decouple

```bash
uv add python-decouple
```

Create a `.env` file in the project root (never commit this file):  

```
DB_ENGINE=django.db.backends.postgresql
DB_NAME=mydb
DB_USER=postgres
DB_PASSWORD=s$cret
DB_HOST=localhost
DB_PORT=5432
```

Reference the variables in `settings.py`:  

```python
from decouple import config

DATABASES = {
    'default': {
        'ENGINE': config('DB_ENGINE',
                         default='django.db.backends.sqlite3'),
        'NAME': config('DB_NAME', default='db.sqlite3'),
        'USER': config('DB_USER', default=''),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default=''),
        'PORT': config('DB_PORT', default=''),
    }
}
```

The `config` function reads from `.env` first, then from actual environment  
variables, and falls back to the `default` when neither source provides  
a value. This makes it trivial to run the same `settings.py` in development  
(with SQLite) and in production (with PostgreSQL) simply by swapping the  
environment file.  


## Migrations

Migrations are Django's mechanism for propagating model changes — adding  
a field, renaming a table, creating an index — into the database schema.  
Each migration is a Python file stored in an app's `migrations/` directory.  
The files form a directed acyclic graph (DAG) that Django traverses in order  
to bring the schema up to date.  

### Creating migrations

After adding or modifying a model, generate migration files with:  

```bash
python manage.py makemigrations
```

To limit generation to a specific app:  

```bash
python manage.py makemigrations myapp
```

`makemigrations` inspects your models, compares them with the current  
migration state, and writes one or more new migration files that describe  
the required schema changes. No database is touched at this stage.  

### Applying migrations

```bash
python manage.py migrate
```

The `migrate` command applies all outstanding migrations in dependency  
order. On the very first run it creates the `django_migrations` table,  
which tracks which migrations have already been applied.  

Apply migrations for a single app only:  

```bash
python manage.py migrate myapp
```

### Inspecting migration status

```bash
python manage.py showmigrations
```

This prints all migrations and marks each one as applied (`[X]`) or  
unapplied (`[ ]`).  

To preview the SQL that a migration would execute without actually running it:  

```bash
python manage.py sqlmigrate myapp 0002
```

### Rolling back migrations

To roll back to a specific migration, pass the migration name to `migrate`:  

```bash
python manage.py migrate myapp 0001
```

Django runs the reverse operations of every migration applied  
after `0001` in reverse order. If a migration has no reverse defined,  
Django will refuse to roll it back.  

To unapply **all** migrations for an app:  

```bash
python manage.py migrate myapp zero
```

### Squashing migrations

Over time an app accumulates many small migration files. Squashing collapses  
a range of them into a single optimised file:  

```bash
python manage.py squashmigrations myapp 0001 0010
```

This creates a new migration that replaces `0001` through `0010`. Django  
keeps the old files in place until all deployments have applied the  
squashed migration, at which point the originals can be deleted.  

### Data migrations

A data migration runs arbitrary Python code against the database — for  
example, back-filling a new column. Create an empty migration file and  
fill in the `operations` list:  

```python
# myapp/migrations/0003_populate_slug.py
from django.db import migrations
from django.utils.text import slugify


def populate_slug(apps, schema_editor):
    Article = apps.get_model('myapp', 'Article')
    for article in Article.objects.all():
        article.slug = slugify(article.title)
        article.save(update_fields=['slug'])


def reverse_populate_slug(apps, schema_editor):
    Article = apps.get_model('myapp', 'Article')
    Article.objects.all().update(slug='')


class Migration(migrations.Migration):

    dependencies = [
        ('myapp', '0002_article_slug'),
    ]

    operations = [
        migrations.RunPython(populate_slug, reverse_populate_slug),
    ]
```

Inside a data migration you must always access models through  
`apps.get_model` rather than importing them directly. This gives you the  
historical version of the model that matches the migration's point in time,  
preventing errors when the model's definition changes later.  


## ORM queries

The Django ORM exposes a fluent `QuerySet` API. `QuerySets` are lazy: no  
SQL is sent to the database until the `QuerySet` is evaluated (for example,  
by iterating over it, calling `list()`, or accessing `len()`).  

### Retrieving all records

```python
from myapp.models import Product

products = Product.objects.all()
```

`all()` returns a `QuerySet` containing every row in the `product` table.  
The SQL issued is `SELECT * FROM myapp_product`.  

### Filtering records

```python
active = Product.objects.filter(is_active=True)
cheap = Product.objects.filter(price__lt=10.00)
recent_active = Product.objects.filter(
    is_active=True,
    created_at__year=2024,
)
```

`filter` accepts field lookups such as `__lt` (less than), `__gt` (greater  
than), `__contains`, `__startswith`, `__iexact` (case-insensitive equality),  
and many others. Multiple keyword arguments are combined with `AND`.  

### Excluding records

```python
available = Product.objects.exclude(stock=0)
```

`exclude` is the complement of `filter`; it returns rows that do *not* match  
the given conditions.  

### Ordering

```python
by_price = Product.objects.order_by('price')        # ascending
by_newest = Product.objects.order_by('-created_at') # descending
```

A leading `-` on the field name reverses the sort direction. You can pass  
multiple fields to break ties.  

### Retrieving a single object

```python
product = Product.objects.get(pk=1)
```

`get` raises `Product.DoesNotExist` if no row matches and  
`Product.MultipleObjectsReturned` if more than one matches. Use `filter`  
followed by `.first()` when you want a safe nullable result.  

### Creating records

```python
product = Product.objects.create(
    name='Laptop',
    price=999.99,
    stock=10,
)
```

`create` instantiates the model and calls `save()` in a single step,  
returning the new instance with its auto-assigned primary key.  

### Updating records

```python
# Update a single instance
product = Product.objects.get(pk=1)
product.price = 899.99
product.save()

# Bulk update — one UPDATE query, no Python loops
Product.objects.filter(stock=0).update(is_active=False)
```

Calling `save()` on an existing instance issues an `UPDATE` for all fields  
by default. Pass `update_fields=['price']` to limit the update to specific  
columns and avoid accidental overwrites.  

### Deleting records

```python
# Delete a single instance
product = Product.objects.get(pk=1)
product.delete()

# Bulk delete
Product.objects.filter(is_active=False).delete()
```

Bulk deletion issues a single `DELETE` query. Django still calls  
`pre_delete` and `post_delete` signals for each object, but it bypasses  
the model's `delete()` method for the bulk case.  

### Chaining and slicing

```python
top_five = (
    Product.objects
    .filter(is_active=True)
    .order_by('-rating')[:5]
)
```

`QuerySets` are chainable and sliceable. A Python slice translates to a  
`LIMIT` / `OFFSET` clause in SQL.  

### Aggregation

```python
from django.db.models import Avg, Count, Max, Min, Sum

stats = Product.objects.aggregate(
    total=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    min_price=Min('price'),
)
# stats = {'total': 42, 'avg_price': 149.5, ...}
```

`aggregate` returns a plain dictionary of computed values across the entire  
`QuerySet`. Each aggregate function maps to an SQL aggregate like `COUNT`,  
`AVG`, `MAX`, `MIN`, or `SUM`.  

### Annotation

```python
from django.db.models import Count

from myapp.models import Author

authors = Author.objects.annotate(book_count=Count('books'))

for author in authors:
    print(author.name, author.book_count)
```

`annotate` adds a computed attribute to *each* object in the `QuerySet`  
rather than collapsing the result to a single row. The query above joins  
the `Author` and `Book` tables and groups by author, producing one row per  
author with a `book_count` column attached.  

### Selecting related objects

```python
# Avoid N+1 queries for ForeignKey / OneToOne
books = Book.objects.select_related('author')

# Avoid N+1 queries for ManyToMany / reverse ForeignKey
authors = Author.objects.prefetch_related('books')
```

Without `select_related`, accessing `book.author` inside a loop would  
trigger a separate SQL query for every book — the classic N+1 problem.  
`select_related` performs a SQL `JOIN`, fetching all related objects in a  
single query. `prefetch_related` issues a second query and performs the  
joining in Python, which is more efficient for `ManyToMany` relationships.  

### values and values_list

```python
# Returns QuerySet of dicts
names = Product.objects.filter(is_active=True).values('id', 'name')

# Returns QuerySet of tuples
ids = Product.objects.values_list('id', flat=True)
```

`values` and `values_list` tell Django to return dictionaries or tuples  
instead of full model instances. They are especially useful when you only  
need a subset of columns or when serialising results directly to JSON.  


## Raw SQL with Model.objects.raw

`Model.objects.raw` executes a raw SQL `SELECT` and maps each row to an  
instance of the model. You get the convenience of model instances (attributes,  
methods, `__str__`) without writing any ORM.  

```python
from myapp.models import Customer


def get_adult_customers():
    customers = Customer.objects.raw(
        'SELECT * FROM myapp_customer WHERE age >= %s',
        [18],
    )
    return list(customers)
```

Parameters are passed as a list in the second argument and escaped by the  
database driver, preventing SQL injection. Never use Python string  
formatting to interpolate values into a raw query.  

### Mapping to non-model columns

`raw` can select computed columns and expose them as attributes on the  
returned model instances:  

```python
from myapp.models import Customer


def get_customers_with_name_length():
    customers = Customer.objects.raw(
        '''
        SELECT id,
               first_name,
               last_name,
               LENGTH(first_name) AS name_len
        FROM myapp_customer
        ORDER BY name_len DESC
        '''
    )
    for c in customers:
        print(c.first_name, c.name_len)
```

Any column in the `SELECT` list that does not correspond to a model field  
is attached to the instance as a plain Python attribute. The primary key  
column must always be included in the `SELECT`; Django raises an  
`InvalidQuery` error otherwise.  

### Deferred fields

Columns not included in the `SELECT` are deferred — accessing them triggers  
an additional query per object. To explicitly defer a field, simply omit it  
from the `SELECT`:  

```python
customers = Customer.objects.raw(
    'SELECT id, first_name FROM myapp_customer'
)
# customer.last_name would trigger an extra query
```

### SQLite age calculation with raw

Calculate age in raw SQL using SQLite's `strftime`:  

```python
def get_users_by_age(age):

    adult_customers = Customer.objects.raw(
        '''
        SELECT *,
        CAST(strftime("%%Y", "now") - strftime("%%Y", dob)
             AS INTEGER) AS age
        FROM users_customer
        WHERE age >= %s
        ''',
        [age],
    )

    return adult_customers
```

Note the double `%%` inside `strftime`: `%s` is a placeholder consumed  
by Django's parameter substitution, so literal `%` signs must be escaped  
as `%%` to survive that substitution step.  


## Raw SQL with connection.cursor

For queries that do not map neatly to a single model — `INSERT … RETURNING`,  
`UPDATE … RETURNING`, multi-table joins, DDL, or stored procedure calls —  
use the low-level `connection.cursor` interface.  

### Basic cursor usage

```python
from django.db import connection
from django.http import JsonResponse

from myapp.models import Customer


def list_customers(request):

    with connection.cursor() as cur:
        cur.execute('SELECT * FROM myapp_customer')
        rows = cur.fetchall()
        columns = [col[0] for col in cur.description]
        result = [dict(zip(columns, row)) for row in rows]

    return JsonResponse({'customers': result})
```

The `with` statement opens a cursor and guarantees it is closed when the  
block exits, even if an exception is raised. `cur.description` is a  
sequence of 7-item tuples whose first element is the column name; zipping  
column names with row values produces a list of dictionaries ready for  
serialisation.  

### Parameterised queries

Always use parameterised queries — never interpolate values with string  
formatting.  

```python
from django.db import connection


def get_customer(customer_id):

    with connection.cursor() as cur:
        cur.execute(
            'SELECT id, first_name, last_name '
            'FROM myapp_customer '
            'WHERE id = %s',
            [customer_id],
        )
        row = cur.fetchone()

    if row is None:
        return None

    return {'id': row[0], 'first_name': row[1], 'last_name': row[2]}
```

Parameters are passed as a list (or tuple) in the second argument.  
The database driver substitutes them safely, making it impossible for  
user-supplied input to alter the query structure.  

### fetchmany and fetchall

```python
from django.db import connection


def paginate_products(offset, limit):

    with connection.cursor() as cur:
        cur.execute(
            'SELECT id, name, price FROM myapp_product '
            'ORDER BY name LIMIT %s OFFSET %s',
            [limit, offset],
        )
        rows = cur.fetchmany(limit)
        columns = [col[0] for col in cur.description]
        return [dict(zip(columns, row)) for row in rows]
```

`fetchone` returns a single row or `None`. `fetchmany(n)` returns up to `n`  
rows as a list of tuples. `fetchall` returns all remaining rows.  

### Write operations

```python
from django.db import connection


def transfer_stock(source_id, dest_id, quantity):

    with connection.cursor() as cur:
        cur.execute(
            'UPDATE myapp_product SET stock = stock - %s WHERE id = %s',
            [quantity, source_id],
        )
        cur.execute(
            'UPDATE myapp_product SET stock = stock + %s WHERE id = %s',
            [quantity, dest_id],
        )
```

`INSERT`, `UPDATE`, and `DELETE` statements are executed just like `SELECT`  
queries. The changes are committed when the surrounding transaction closes,  
which by default happens at the end of each Django view via the  
`ATOMIC_REQUESTS` setting or an explicit `transaction.atomic` block.  

### Using a named database

To target a non-default database, import its connection by alias:  

```python
from django.db import connections


def fetch_from_replica(query, params=None):

    with connections['replica'].cursor() as cur:
        cur.execute(query, params or [])
        return cur.fetchall()
```

`django.db.connections` is a dictionary-like object that returns a  
`DatabaseWrapper` for the given alias. All the same cursor methods are  
available on each connection.  


## Database transactions

By default Django wraps each HTTP request in a transaction when  
`ATOMIC_REQUESTS = True` is set in `DATABASES`. For finer control, use  
the `transaction.atomic` context manager or decorator.  

### atomic context manager

```python
from django.db import transaction

from myapp.models import Order, Product


def place_order(product_id, quantity):

    with transaction.atomic():
        product = Product.objects.select_for_update().get(pk=product_id)
        if product.stock < quantity:
            raise ValueError('Insufficient stock')
        product.stock -= quantity
        product.save(update_fields=['stock'])
        Order.objects.create(product=product, quantity=quantity)
```

Everything inside `transaction.atomic()` runs in a single database  
transaction. If an exception propagates out of the block, all changes are  
rolled back automatically. `select_for_update` locks the selected rows for  
the duration of the transaction, preventing concurrent requests from reading  
a stale stock value.  

### atomic decorator

```python
from django.db import transaction

from myapp.models import Invoice, Payment


@transaction.atomic
def record_payment(invoice_id, amount):
    invoice = Invoice.objects.get(pk=invoice_id)
    invoice.paid += amount
    invoice.save(update_fields=['paid'])
    Payment.objects.create(invoice=invoice, amount=amount)
```

The `@transaction.atomic` decorator wraps the entire function body in a  
transaction, providing the same rollback guarantee as the context manager.  

### Savepoints

`atomic` blocks can be nested. Each nested block creates a database  
savepoint rather than a new transaction, enabling partial rollbacks.  

```python
from django.db import transaction

from myapp.models import Tag, Article


def create_article_with_tags(title, content, tag_names):

    with transaction.atomic():
        article = Article.objects.create(title=title, content=content)

        for name in tag_names:
            try:
                with transaction.atomic():   # savepoint per tag
                    tag, _ = Tag.objects.get_or_create(name=name)
                    article.tags.add(tag)
            except Exception:
                # Only this tag's savepoint is rolled back;
                # the article and previously added tags are preserved.
                pass

    return article
```

If an inner `atomic` block raises an exception, Django rolls back to the  
savepoint that was created when that block was entered, leaving the outer  
transaction intact.  

