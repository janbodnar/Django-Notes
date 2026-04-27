# ORM

Object-Relational Mapping (ORM) is a programming technique that bridges the gap  
between object-oriented programming languages and relational databases. Instead  
of writing raw SQL queries, developers interact with the database using familiar  
language constructs — classes, objects, and methods — while the ORM translates  
those operations into the appropriate SQL behind the scenes.  

Django's ORM is one of the most mature and feature-rich ORMs available in any  
web framework. It is built directly into Django and requires no additional  
installation. Every Django application that uses a database interacts with it  
through the ORM. The ORM maps each Python model class to a database table, each  
class attribute to a table column, and each model instance to a table row.  
This mapping is transparent and automatic; developers rarely need to think about  
the underlying SQL unless they choose to.  

At its core, the Django ORM operates through three primary abstractions:  
*models*, which define the data structure; *managers*, which provide the  
interface to query the database; and *QuerySets*, which represent collections  
of database records and support lazy evaluation, chaining, and caching.  

Django's ORM is database-agnostic. The same Python code runs against SQLite  
during development, PostgreSQL in production, MySQL, Oracle, or any other  
supported backend, with only a configuration change required. This portability  
is a significant architectural benefit that reduces vendor lock-in and simplifies  
environment management.  

The ORM also handles schema evolution through the *migrations* system. When a  
model changes — a field is added, renamed, or removed — Django's `makemigrations`  
command detects the difference and generates a migration file that can be applied  
to bring the database schema in sync with the models. This replaces the error-  
prone practice of hand-writing `ALTER TABLE` statements.  

Security is another core advantage. The ORM parameterises all query values by  
default, making SQL injection attacks virtually impossible when using the  
standard query API. Developers must consciously opt into raw SQL to bypass this  
protection.  

## Key Features and Benefits

- **Automatic data mapping:** ORM maps Python model classes to database tables,  
  eliminating hand-written `CREATE TABLE` statements and manual SQL queries.  
- **Query API:** A rich, chainable Python API replaces SQL for most read and  
  write operations.  
- **Data validation:** Field types and constraints (required fields, max lengths,  
  unique constraints) are validated before data reaches the database.  
- **Relationships:** One-to-one, one-to-many, and many-to-many relationships are  
  handled declaratively with automatic join generation.  
- **Migrations:** Schema changes are tracked and applied automatically through  
  the migrations system.  
- **Database portability:** The same code targets multiple database backends with  
  only a settings change.  
- **Lazy evaluation:** QuerySets are not executed until their results are actually  
  consumed, minimising unnecessary database round-trips.  
- **Security:** Automatic value parameterisation prevents SQL injection.  


Django ORM simplifies database interactions by providing a high-level,  
object-oriented interface. It automates many common database tasks, making  
development more efficient and reducing the likelihood of errors.  


## QuerySet

A `QuerySet` is the central abstraction of the Django ORM. It represents a  
lazy database query that has not yet been executed. QuerySets can be filtered,  
sliced, ordered, annotated, and combined before they are evaluated. Because they  
are lazy, chaining multiple operations does not produce multiple round-trips to  
the database — the final SQL is built up incrementally and executed only when  
the results are actually needed (e.g. when iterating, calling `list()`, or  
slicing with a step).  

```python
from myapp.models import Article

# Build a complex QuerySet without hitting the database
qs = (
    Article.objects
    .filter(status='published')
    .exclude(author__isnull=True)
    .order_by('-created_at')
    .select_related('author')
)

# Database is queried only here, when the loop starts
for article in qs:
    print(article.title)
```

QuerySets are also *re-usable*: evaluating the same QuerySet multiple times  
does not produce duplicate queries, because the results are cached in memory  
after the first evaluation. This caching behaviour means that assigning a  
QuerySet to a variable and iterating it twice only hits the database once.  
QuerySets expose a broad API — `filter`, `exclude`, `annotate`, `aggregate`,  
`order_by`, `values`, `distinct`, and many more — making it possible to express  
virtually any query without leaving Python.  


## Creating objects

New database records are created by instantiating a model class and calling  
`save`, or by using the `create` shortcut on the manager.  

```python
from myapp.models import Product

# Instantiate and save in two steps
p = Product(name='Laptop', price=999.99, stock=50)
p.save()

# Equivalent one-step shortcut
p2 = Product.objects.create(name='Mouse', price=29.99, stock=200)
```

The `save` method issues an `INSERT` when the object has no primary key yet,  
and an `UPDATE` when the primary key already exists. `create` is a convenience  
wrapper that instantiates the model and calls `save` in a single call. Both  
approaches run the model's full validation and `pre_save` / `post_save` signal  
chain, ensuring any custom logic attached to those hooks is executed.  


## Retrieving objects

The `all` method returns a QuerySet containing every record in a table. The  
`get` method retrieves a single object matching the given criteria.  

```python
from myapp.models import Product

# All products
all_products = Product.objects.all()

# Single product by primary key
product = Product.objects.get(pk=1)

# Single product by field value
product = Product.objects.get(name='Laptop')
```

`get` raises `DoesNotExist` when no record matches, and `MultipleObjectsReturned`  
when more than one record matches. Both exceptions are attributes of the model  
class (e.g. `Product.DoesNotExist`), so they can be caught precisely without  
importing anything extra. Always prefer `get` when you expect exactly one result  
and want the application to raise an explicit error if that expectation is  
violated.  


## Filtering objects

The `filter` method returns a QuerySet containing only the records that match  
the given keyword arguments.  

```python
from myapp.models import Product

# Products under £50
cheap = Product.objects.filter(price__lt=50)

# Active products in a specific category
active = Product.objects.filter(is_active=True, category='Electronics')

# Published articles by a specific author
from myapp.models import Article
articles = Article.objects.filter(
    status='published',
    author__username='jane'
)
```

Each call to `filter` returns a new QuerySet, leaving the original unchanged.  
Multiple conditions passed to a single `filter` call are combined with SQL  
`AND`. When conditions need to be combined with `OR`, use `Q` objects (covered  
later). Filtering across relationships — such as `author__username` — causes  
Django to perform an automatic SQL `JOIN` to the related table.  


## Field lookups

Field lookups are the Django ORM's equivalent of SQL `WHERE` clause operators.  
They are appended to field names using a double-underscore separator.  

```python
from myapp.models import Product, Article

# Exact match (default)
Product.objects.filter(name__exact='Laptop')

# Case-insensitive exact match
Product.objects.filter(name__iexact='laptop')

# Contains substring (case-sensitive)
Product.objects.filter(description__contains='wireless')

# Contains substring (case-insensitive)
Product.objects.filter(description__icontains='wireless')

# Starts with / ends with
Article.objects.filter(title__startswith='Django')
Article.objects.filter(title__endswith='guide')

# Greater than / less than / range
Product.objects.filter(price__gt=100)
Product.objects.filter(price__lte=500)
Product.objects.filter(price__range=(100, 500))

# In a list of values
Product.objects.filter(status__in=['active', 'on_sale'])

# NULL checks
Article.objects.filter(published_date__isnull=True)

# Date component lookups
Article.objects.filter(created_at__year=2024)
Article.objects.filter(created_at__month=6)
```

Lookups can traverse relationships by chaining field names with double  
underscores (e.g. `author__profile__country__iexact='uk'`). Django resolves  
each step into the appropriate `JOIN` automatically. The lookup vocabulary is  
extensive — `exact`, `iexact`, `contains`, `icontains`, `startswith`,  
`endswith`, `gt`, `gte`, `lt`, `lte`, `in`, `range`, `isnull`, `regex`,  
`iregex`, and date/time decompositions — covering the vast majority of  
filtering needs without any raw SQL.  


## Excluding objects

The `exclude` method is the inverse of `filter`. It returns all records that do  
*not* match the given conditions.  

```python
from myapp.models import Product

# All products that are NOT out of stock
available = Product.objects.exclude(stock=0)

# Articles that are not drafts
published = Article.objects.exclude(status='draft')

# Products not in a specific category and not inactive
filtered = Product.objects.exclude(category='Accessories').exclude(is_active=False)
```

`exclude` follows the same double-underscore lookup syntax as `filter` and can  
traverse relationships in the same way. Multiple conditions inside a single  
`exclude` call are wrapped together in a single SQL `NOT (... AND ...)` clause.  
Chaining two `exclude` calls produces two separate `NOT` conditions, which is  
subtly different from combining both conditions inside one call — a distinction  
that matters when traversing nullable relationships.  


## Chaining QuerySets

QuerySets are immutable and return new QuerySets when methods are called on  
them, making it natural to build up complex queries incrementally.  

```python
from myapp.models import Article

base_qs = Article.objects.filter(status='published')

# Add further refinements without modifying base_qs
recent = base_qs.filter(created_at__year=2024).order_by('-created_at')
popular = base_qs.filter(views__gte=1000).order_by('-views')

# Chain across methods
result = (
    Article.objects
    .filter(status='published')
    .exclude(author__isnull=True)
    .select_related('author', 'category')
    .prefetch_related('tags')
    .order_by('-created_at')[:20]
)
```

Because QuerySets are lazy, each method call in a chain merely refines the  
internal query representation. The database is not consulted until the QuerySet  
is consumed. This design makes it straightforward to define reusable base  
QuerySets in managers and then specialise them further at the call site without  
incurring unnecessary database overhead.  


## Ordering QuerySets

The `order_by` method controls the sort order of a QuerySet. Pass field names  
for ascending order or prefix them with `-` for descending order.  

```python
from myapp.models import Product, Article

# Ascending by price
Product.objects.all().order_by('price')

# Descending by created date
Article.objects.all().order_by('-created_at')

# Multiple sort keys
Product.objects.all().order_by('category', '-price')

# Order by a related field
Article.objects.all().order_by('author__last_name', 'title')

# Clear any existing ordering (useful when overriding Meta.ordering)
Article.objects.all().order_by()
```

The default ordering for a model can be set in the `Meta` class using the  
`ordering` attribute, so that every QuerySet returned by that model's manager  
is sorted consistently without specifying `order_by` each time. Calling  
`order_by()` with no arguments strips the default ordering from a QuerySet,  
which is important when passing QuerySets to `UNION` operations or when the  
ordering would interfere with `DISTINCT` queries.  


## Limiting QuerySets

QuerySets support Python's slice syntax to apply SQL `LIMIT` and `OFFSET`  
clauses.  

```python
from myapp.models import Product

# First 10 products (LIMIT 10)
first_ten = Product.objects.all()[:10]

# Products 11 to 20 (LIMIT 10 OFFSET 10)
second_page = Product.objects.all()[10:20]

# Single object using index (executes immediately)
latest = Article.objects.order_by('-created_at')[0]
```

Negative indexing is not supported on QuerySets because it would require the  
database to know the total count before it could calculate the offset, which  
is inefficient. Slicing a QuerySet with a start and stop value (e.g. `[5:10]`)  
does not produce a list immediately — the result is still a QuerySet that is  
evaluated lazily. Indexing with a single integer (e.g. `[0]`) evaluates the  
QuerySet immediately and returns a single model instance.  


## Aggregation

Aggregation functions compute a single summary value from a set of records.  
Django provides `Count`, `Sum`, `Avg`, `Max`, and `Min` in  
`django.db.models`.  

```python
from django.db.models import Avg, Count, Max, Min, Sum
from myapp.models import Product, Order

# Total number of products
total = Product.objects.count()

# Average product price
avg_price = Product.objects.aggregate(average=Avg('price'))
# Returns: {'average': Decimal('149.99')}

# Multiple aggregates in one query
stats = Product.objects.aggregate(
    total_products=Count('id'),
    lowest_price=Min('price'),
    highest_price=Max('price'),
    total_stock_value=Sum('price'),
)

# Aggregate filtered QuerySet
avg_active = Product.objects.filter(is_active=True).aggregate(
    average=Avg('price')
)
```

`aggregate` returns a plain Python dictionary, not a QuerySet. The keys are  
either auto-generated (e.g. `price__avg`) or the custom aliases supplied as  
keyword arguments. Aggregation is performed in a single SQL query, making it  
far more efficient than fetching all rows and summing in Python. When you need  
per-object aggregated values rather than a single summary, use `annotate`  
instead.  


## The annotate method

The `annotate` method adds a calculated field to each object in a QuerySet,  
computing aggregate or derived values on a per-row basis rather than across  
the entire QuerySet.  

```python
from django.db.models import Avg, Count, F, Sum
from myapp.models import Author, Category

# Count the number of books per author
authors = Author.objects.annotate(book_count=Count('books'))
for author in authors:
    print(f'{author.name}: {author.book_count} books')

# Annotate with average rating of related reviews
authors = Author.objects.annotate(avg_rating=Avg('books__reviews__rating'))

# Annotate with a computed arithmetic expression
from django.db.models import ExpressionWrapper, FloatField
products = Product.objects.annotate(
    discounted_price=ExpressionWrapper(
        F('price') * 0.9, output_field=FloatField()
    )
)
```

Adding the `calculated_age` attribute via the `annotate` method:  

```python
from datetime import datetime
from django.db import models

def get_users_by_age(age):

    today = datetime.now()
    cutoff_date = today.replace(year=today.year - age)

    customers = Customer.objects.annotate(
        dob_as_date=Cast('dob', output_field=models.DateField()),
        calculated_age=models.ExpressionWrapper(
            today - models.F('dob'), output_field=models.IntegerField()
        )
    ).filter(dob_as_date__lte=cutoff_date)

    return customers
```

Similarly, a custom `age` property on the model avoids the annotation entirely  
for display-only scenarios:  

```python
from django.db import models
from datetime import date
from dateutil.relativedelta import relativedelta


class Customer(models.Model):

    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    occupation = models.CharField(max_length=255)
    dob = models.CharField(max_length=15)

    @property
    def age(self):
        birthdate = date.fromisoformat(self.dob)
        today = date.today()
        return relativedelta(today, birthdate).years

    def __str__(self):
        return f'{self.first_name} {self.last_name}'
```

Annotated fields are attached directly to each object in the QuerySet,  
accessible as regular attributes. They can be used in subsequent `filter`,  
`order_by`, or `values` calls within the same QuerySet chain. Unlike  
`aggregate`, which collapses the entire QuerySet into a single dictionary,  
`annotate` preserves individual records and enriches each one with the computed  
value.  


## F expressions

`F` expressions represent the value of a model field at the database level,  
allowing field-to-field comparisons and in-place updates without fetching  
data into Python.  

```python
from django.db.models import F
from myapp.models import Product

# Increment stock by 10 for all products — no Python loop needed
Product.objects.update(stock=F('stock') + 10)

# Filter products where discount price is less than half the original
Product.objects.filter(sale_price__lt=F('price') / 2)

# Compare two fields on the same model
from myapp.models import Employee
Employee.objects.filter(salary__gt=F('manager__salary'))

# Use F in annotations
from django.db.models import ExpressionWrapper, DecimalField
Product.objects.annotate(
    profit=ExpressionWrapper(
        F('price') - F('cost'), output_field=DecimalField()
    )
)
```

Because `F` objects are resolved at the SQL level, they avoid race conditions  
that would occur if you fetched a value into Python, modified it, and saved it  
back. Two concurrent requests calling `product.stock += 1; product.save()` can  
both read the same original value and lose one of the increments. Using  
`Product.objects.filter(pk=product.pk).update(stock=F('stock') + 1)` delegates  
the arithmetic to the database, where it is executed atomically.  


## Q objects

`Q` objects encapsulate filter conditions and can be combined with the `|` (OR),  
`&` (AND), and `~` (NOT) operators to build complex queries.  

```python
from django.db.models import Q
from myapp.models import Article, Product

# OR condition: articles that are published OR featured
Article.objects.filter(Q(status='published') | Q(is_featured=True))

# AND with OR: published AND (featured OR high-traffic)
Article.objects.filter(
    Q(status='published') & (Q(is_featured=True) | Q(views__gte=10000))
)

# NOT: products that are not out of stock
Product.objects.filter(~Q(stock=0))

# Complex query with both Q and keyword arguments
Product.objects.filter(
    Q(category='Electronics') | Q(category='Computers'),
    is_active=True,
    price__lte=1000
)
```

When keyword arguments and `Q` objects are mixed, all `Q` objects must come  
before the keyword arguments. Combining `Q` objects with operators is entirely  
equivalent to the corresponding SQL boolean logic. The `~Q(...)` construct  
produces a SQL `NOT (...)` clause. `Q` objects are especially useful when the  
exact set of filter conditions is not known until runtime — for example, when  
building a search feature that combines user-supplied criteria.  


## select_related

`select_related` performs a SQL `JOIN` and includes the related object data  
in a single database query, eliminating the N+1 query problem for  
`ForeignKey` and `OneToOneField` relationships.  

```python
from myapp.models import Article

# Without select_related: 1 query for articles + 1 per article for author
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # extra query each time

# With select_related: 1 query with JOIN
articles = Article.objects.select_related('author').all()
for article in articles:
    print(article.author.name)  # no extra query

# Traverse multiple levels
articles = Article.objects.select_related('author__profile', 'category')
```

`select_related` is most effective when the related objects are always needed  
alongside the main objects, because the `JOIN` fetches all data upfront. It  
is limited to relationships that traverse a single database row — `ForeignKey`  
and `OneToOneField`. For `ManyToManyField` or reverse `ForeignKey` lookups  
(one-to-many), use `prefetch_related` instead. Calling `select_related()` with  
no arguments follows all non-null foreign keys automatically, which can  
generate very large `JOIN` queries — prefer listing specific relations  
explicitly.  


## prefetch_related

`prefetch_related` performs a separate query for each relationship and joins  
the results in Python. It handles `ManyToManyField`, reverse `ForeignKey`  
lookups, and any relationship that cannot be expressed as a simple `JOIN`.  

```python
from django.db.models import Prefetch
from myapp.models import Author, Article

# Prefetch all books for each author in a separate query
authors = Author.objects.prefetch_related('books')
for author in authors:
    for book in author.books.all():  # no extra query
        print(book.title)

# Prefetch with a filtered sub-QuerySet using Prefetch()
from myapp.models import Review
authors = Author.objects.prefetch_related(
    Prefetch(
        'books__reviews',
        queryset=Review.objects.filter(rating__gte=4),
        to_attr='good_reviews'
    )
)
for author in authors:
    for book in author.books.all():
        print(book.good_reviews)  # already filtered, no extra query
```

`prefetch_related` always issues at least two queries — one for the main  
objects and one per prefetched relationship — but this is far cheaper than the  
N+1 pattern it replaces. The `Prefetch` helper class gives fine-grained control  
over the sub-QuerySet and allows the result to be stored under a custom  
attribute name via `to_attr`. Combining `select_related` and `prefetch_related`  
on the same QuerySet is perfectly valid and often necessary for models with both  
`ForeignKey` and `ManyToManyField` relationships.  


## values and values_list

`values` returns QuerySet items as dictionaries instead of model instances.  
`values_list` returns them as tuples. Both methods are useful when only a  
subset of fields is needed and full model instantiation is unnecessary.  

```python
from myapp.models import Product

# Dictionaries with selected fields
products = Product.objects.values('name', 'price')
# [{'name': 'Laptop', 'price': Decimal('999.99')}, ...]

# Tuples with selected fields
products = Product.objects.values_list('name', 'price')
# [('Laptop', Decimal('999.99')), ...]

# Flat list of a single field (flat=True)
names = Product.objects.values_list('name', flat=True)
# ['Laptop', 'Mouse', ...]

# Useful for collecting IDs
ids = Product.objects.filter(is_active=True).values_list('id', flat=True)

# values() supports traversal of related fields
articles = Article.objects.values('title', 'author__username', 'category__name')
```

`values` and `values_list` reduce memory usage significantly when working with  
large QuerySets because model instances are not constructed. They are ideal  
for producing data for serialisation, CSV export, or analytics pipelines where  
only raw column values are required. Both methods return a QuerySet, so all  
the usual chaining operations (`filter`, `order_by`, `annotate`, etc.) remain  
available before evaluation.  


## distinct

The `distinct` method adds a `SELECT DISTINCT` clause, removing duplicate rows  
from the result.  

```python
from myapp.models import Article, Tag

# Unique categories from published articles
categories = (
    Article.objects
    .filter(status='published')
    .values_list('category__name', flat=True)
    .distinct()
)

# Authors who have at least one published article
authors = (
    Article.objects
    .filter(status='published')
    .values('author__id', 'author__username')
    .distinct()
)

# Distinct full model instances (PostgreSQL supports DISTINCT ON)
from django.db.models.expressions import RawSQL
articles = Article.objects.order_by('author_id', '-created_at').distinct('author_id')
```

On most databases, `distinct()` with no arguments deduplicates rows based on  
all selected columns. On PostgreSQL, `distinct(field_name)` produces a  
`DISTINCT ON` clause that selects the first row for each unique value of the  
specified field — this is a PostgreSQL extension not available on other  
backends. Duplicates most commonly arise when a QuerySet traverses  
one-to-many relationships (e.g. filtering through a `ManyToManyField`), which  
causes the same parent row to appear once per matching child row.  


## exists and count

`exists` returns `True` if the QuerySet contains at least one record.  
`count` returns the number of records that match the QuerySet.  

```python
from myapp.models import Product, Article

# Check existence without fetching objects
if Product.objects.filter(stock=0).exists():
    print('Some products are out of stock')

# Prefer exists() over len() or bool(qs) for performance
qs = Article.objects.filter(status='published')
if qs.exists():   # executes SELECT 1 LIMIT 1
    render_list(qs)

# Count matching records
total_published = Article.objects.filter(status='published').count()

# count() on the whole table
total = Product.objects.count()
```

`exists` translates to a `SELECT 1 ... LIMIT 1` query, which is the most  
efficient possible check for non-emptiness. Never use `len(queryset) > 0` or  
evaluate the QuerySet as a boolean (`if queryset:`) when existence is all that  
matters — doing so fetches all matching rows into memory. Likewise, `count`  
issues a `SELECT COUNT(*)` query without instantiating any model objects,  
making it far faster than `len(queryset)` for large tables.  


## Bulk operations

`bulk_create` inserts many objects in a single SQL `INSERT` statement.  
`bulk_update` updates specific fields for a list of objects in one query.  

```python
from myapp.models import Product

# Insert 1000 objects in one query instead of 1000 queries
products = [
    Product(name=f'Product {i}', price=i * 10, stock=100)
    for i in range(1000)
]
Product.objects.bulk_create(products, batch_size=500)

# bulk_create returns the created objects (with PKs on most databases)
created = Product.objects.bulk_create([
    Product(name='Keyboard', price=79.99, stock=30),
    Product(name='Monitor', price=349.99, stock=15),
])

# Update the price and stock fields for a list of existing objects
products = list(Product.objects.filter(category='Electronics'))
for product in products:
    product.price = product.price * 0.95  # 5% discount
    product.stock += 10

Product.objects.bulk_update(products, fields=['price', 'stock'], batch_size=500)
```

`bulk_create` bypasses the `save` method and model-level signals, so any  
custom logic in `save` or `pre_save`/`post_save` signal handlers will not run.  
The same is true for `bulk_update`. Both methods are intended for high-  
throughput data loading scenarios where maximum insertion speed matters and the  
normal single-object lifecycle is acceptable to skip. The `batch_size` parameter  
controls how many rows are included in each underlying SQL statement, preventing  
excessively large queries when inserting thousands of rows.  


## Updating objects

The `update` method issues a single SQL `UPDATE` statement for all records  
matched by a QuerySet, without fetching them into Python.  

```python
from django.db.models import F
from myapp.models import Product, Article

# Deactivate all out-of-stock products
Product.objects.filter(stock=0).update(is_active=False)

# Increase all prices by 5% using an F expression
Product.objects.update(price=F('price') * 1.05)

# Update a single object's field without fetching the full object
Article.objects.filter(pk=42).update(status='published', views=F('views') + 1)

# update() returns the number of rows affected
count = Product.objects.filter(category='Legacy').update(is_active=False)
print(f'{count} products deactivated')
```

Like `bulk_create` and `bulk_update`, the QuerySet `update` method bypasses  
the model's `save` method and does not trigger `pre_save`/`post_save` signals.  
It also does not call full model validation. Use it when you need to modify  
many rows efficiently and those side effects are not required. For single  
objects where signals or custom `save` logic must run, fetch the instance and  
call `save` explicitly.  


## Deleting objects

The `delete` method removes records from the database, either for a specific  
model instance or for an entire QuerySet.  

```python
from myapp.models import Product, Article

# Delete a single instance
product = Product.objects.get(pk=1)
product.delete()

# Delete all records matching a filter (no Python loop needed)
deleted_count, detail = Article.objects.filter(status='archived').delete()
print(f'Deleted {deleted_count} records: {detail}')

# Delete all records in a table
Product.objects.all().delete()

# Cascade behaviour is determined by on_delete on ForeignKey fields
# CASCADE: related records are also deleted
# SET_NULL: related FK is set to NULL
# PROTECT: raises ProtectedError to prevent deletion
```

`delete` returns a tuple of `(total_count, {model_label: count})`, describing  
exactly how many rows were removed from each table (including cascades). Django  
collects all objects to delete in Python before issuing the DELETE statements,  
so that `pre_delete` and `post_delete` signals fire for each object. For very  
large deletions this collection can be memory-intensive; splitting large deletes  
into batches mitigates the problem.  


## get_or_create and update_or_create

`get_or_create` retrieves an object matching the given criteria, or creates it  
if none exists. `update_or_create` retrieves and updates it, or creates it.  

```python
from myapp.models import Tag, Article

# Returns (instance, created) tuple
tag, created = Tag.objects.get_or_create(
    name='django',
    defaults={'slug': 'django', 'color': '#092E20'}
)
if created:
    print('Tag was just created')

# update_or_create: update existing or create new
article, created = Article.objects.update_or_create(
    slug='orm-guide',
    defaults={
        'title': 'Complete ORM Guide',
        'status': 'published',
    }
)

# Useful for idempotent upserts in data import pipelines
from myapp.models import Product
product, created = Product.objects.update_or_create(
    sku='LAPTOP-001',
    defaults={'name': 'Pro Laptop', 'price': 1299.00, 'stock': 20}
)
```

Both methods are *atomic at the application level* only: they wrap the lookup  
and creation in a transaction, but a race condition can still occur on  
databases that do not support `INSERT ... ON CONFLICT`. Wrapping the call in  
`select_for_update` or using PostgreSQL's native upsert via raw SQL is  
necessary in high-concurrency scenarios. The `defaults` dictionary contains  
fields that are only applied during creation for `get_or_create`, or applied  
during both creation and update for `update_or_create`.  


## Transactions

Django wraps each HTTP request in a transaction by default  
(`ATOMIC_REQUESTS = True` in settings). For manual control, use  
`transaction.atomic`.  

```python
from django.db import transaction
from myapp.models import Account

# As a context manager
def transfer_funds(from_id, to_id, amount):
    with transaction.atomic():
        from_account = Account.objects.select_for_update().get(pk=from_id)
        to_account = Account.objects.select_for_update().get(pk=to_id)

        if from_account.balance < amount:
            raise ValueError('Insufficient funds')

        from_account.balance -= amount
        to_account.balance += amount
        from_account.save()
        to_account.save()

# As a decorator
@transaction.atomic
def create_order_with_items(order_data, items_data):
    order = Order.objects.create(**order_data)
    for item_data in items_data:
        OrderItem.objects.create(order=order, **item_data)
    return order

# Savepoints: nested atomic blocks create savepoints
def nested_example():
    with transaction.atomic():          # outer transaction
        do_something()
        with transaction.atomic():      # inner savepoint
            do_something_risky()        # can be rolled back independently
```

If an exception propagates out of an `atomic` block, the entire transaction  
(or the savepoint, if nested) is rolled back. Code that should run only after  
the transaction commits — such as sending an email or publishing to a message  
queue — should be placed inside `transaction.on_commit` callbacks to avoid  
sending notifications for operations that were ultimately rolled back.  


## Conditional expressions

`Case`/`When` implements conditional SQL logic equivalent to the SQL  
`CASE WHEN ... THEN ... ELSE ... END` expression.  

```python
from django.db.models import Case, CharField, IntegerField, Value, When
from myapp.models import Product, Employee

# Annotate with a computed category label
products = Product.objects.annotate(
    price_band=Case(
        When(price__lt=50, then=Value('budget')),
        When(price__lt=200, then=Value('mid-range')),
        When(price__gte=200, then=Value('premium')),
        output_field=CharField(),
    )
)

# Conditional aggregation: count products in each price band
from django.db.models import Count, Q
stats = Product.objects.aggregate(
    budget=Count('id', filter=Q(price__lt=50)),
    mid=Count('id', filter=Q(price__range=(50, 199))),
    premium=Count('id', filter=Q(price__gte=200)),
)

# Bulk conditional update
Employee.objects.update(
    bonus=Case(
        When(years_of_service__gte=10, then=Value(5000)),
        When(years_of_service__gte=5, then=Value(2500)),
        default=Value(1000),
        output_field=IntegerField(),
    )
)
```

`Case`/`When` expressions are evaluated entirely within the database, avoiding  
the need to fetch rows into Python just to apply conditional logic. They can be  
used anywhere an expression is accepted — `annotate`, `aggregate`, `update`,  
`order_by`, and `filter`. The `filter` keyword argument on aggregate functions  
(available since Django 2.0) is a concise shorthand for `Case`/`When` when  
performing conditional aggregation.  


## Subqueries

`Subquery` and `OuterRef` allow embedding one QuerySet as a subquery inside  
another query.  

```python
from django.db.models import OuterRef, Subquery
from myapp.models import Article, Comment

# Annotate each article with its most recent comment date
latest_comment = (
    Comment.objects
    .filter(article=OuterRef('pk'))
    .order_by('-created_at')
    .values('created_at')[:1]
)

articles = Article.objects.annotate(latest_comment_date=Subquery(latest_comment))

# Annotate with the text of the most recent comment
latest_comment_text = (
    Comment.objects
    .filter(article=OuterRef('pk'))
    .order_by('-created_at')
    .values('body')[:1]
)

articles = Article.objects.annotate(
    latest_comment=Subquery(latest_comment_text)
)

# Use Exists subquery for existence checks without a COUNT
from django.db.models import Exists
articles_with_comments = Article.objects.annotate(
    has_comments=Exists(Comment.objects.filter(article=OuterRef('pk')))
)
```

`OuterRef` references a field on the outer QuerySet, functioning like a  
correlated subquery in SQL. The subquery must be sliced to a single row  
(`[:1]`) when used with `Subquery`, because SQL scalar subqueries must return  
at most one value. `Exists` is a specialised subquery that returns a boolean  
and is more efficient than `Count` for existence checks because the database  
can short-circuit after finding the first match.  


## Database functions

Django exposes many database functions through `django.db.models.functions`,  
enabling string manipulation, date arithmetic, type casting, and more — all  
executed inside the database.  

```python
from django.db.models.functions import (
    Coalesce, Concat, ExtractYear, Lower, Now, TruncMonth, Upper,
)
from django.db.models import Value
from myapp.models import Article, Product

# Coalesce: return the first non-NULL value
articles = Article.objects.annotate(
    display_name=Coalesce('subtitle', 'title')
)

# Concatenate strings in the database
products = Product.objects.annotate(
    full_label=Concat('brand', Value(' - '), 'name')
)

# Convert to lowercase for case-insensitive sorting
articles = Article.objects.annotate(
    lower_title=Lower('title')
).order_by('lower_title')

# Truncate datetime to month for grouping
from django.db.models import Count
monthly = (
    Article.objects
    .annotate(month=TruncMonth('created_at'))
    .values('month')
    .annotate(count=Count('id'))
    .order_by('month')
)

# Extract date part
articles = Article.objects.annotate(year=ExtractYear('created_at'))

# Current database timestamp
articles = Article.objects.filter(published_at__lte=Now())
```

Using database functions keeps computations in SQL rather than Python, which  
is especially beneficial when sorting or grouping by derived values — doing  
that in Python would require fetching the entire result set. Django's function  
layer is extensible: you can subclass `Func` to wrap any database function not  
already provided in the standard library, including database-specific functions  
unique to PostgreSQL, MySQL, or Oracle.  


## defer and only

`defer` postpones loading of specified fields until they are accessed. `only`  
is the inverse — it loads *only* the listed fields and defers everything else.  

```python
from myapp.models import Article

# Defer the large 'content' and 'metadata' fields
articles = Article.objects.defer('content', 'metadata')
for article in articles:
    print(article.title)       # already loaded
    print(article.content)     # triggers an extra query here

# Load only the fields needed for a list view
articles = Article.objects.only('id', 'title', 'created_at', 'author_id')

# Combine with select_related
articles = (
    Article.objects
    .only('id', 'title', 'author__username')
    .select_related('author')
)
```

`defer` and `only` are optimisation tools for situations where a model has  
heavy columns — large text blobs, binary data, or JSON fields — that are  
rarely needed in certain code paths. Deferred fields are loaded transparently  
on first access via an additional `SELECT`, so they do not require any change  
to the consuming code. However, this deferred loading can accidentally trigger  
many extra queries if deferred attributes are accessed inside a loop; always  
profile before and after to confirm that deferral actually improves performance.  


## Model signals

Django's signal dispatcher allows decoupled components to react to events that  
occur on models, such as object creation, update, or deletion.  

```python
from django.db.models.signals import post_save, post_delete, pre_save
from django.dispatch import receiver
from myapp.models import Article, UserProfile
from django.contrib.auth.models import User


@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    """Create a UserProfile whenever a new User is saved."""
    if created:
        UserProfile.objects.create(user=instance)


@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    """Keep UserProfile in sync when the User is updated."""
    instance.userprofile.save()


@receiver(pre_save, sender=Article)
def set_article_slug(sender, instance, **kwargs):
    """Auto-generate a slug before the Article is saved."""
    from django.utils.text import slugify
    if not instance.slug:
        instance.slug = slugify(instance.title)


@receiver(post_delete, sender=Article)
def clean_up_article_files(sender, instance, **kwargs):
    """Remove uploaded files when an Article is deleted."""
    if instance.cover_image:
        instance.cover_image.delete(save=False)
```

Signals must be connected before they can fire. The recommended approach is  
to place signal handlers in a dedicated `signals.py` file and import that  
module inside the app's `AppConfig.ready` method to ensure connection at  
startup. The `@receiver` decorator is a convenient shorthand for  
`Signal.connect`. Signal handlers receive keyword arguments including `sender`  
(the model class), `instance` (the object), and `created` (for `post_save`  
only). Signals are synchronous and run within the same database transaction as  
the triggering operation, so exceptions raised inside a handler will roll back  
the transaction if `ATOMIC_REQUESTS` is enabled.  


## Raw SQL

Sometimes a query is too complex or requires database-specific features that  
the ORM cannot express. Django provides two mechanisms for running raw SQL  
while still working with model instances.  

```python
from django.db import connection
from myapp.models import Customer, Product

# Manager.raw() — returns model instances from a raw SELECT
customers = Customer.objects.raw(
    'SELECT * FROM myapp_customer WHERE birth_year > %s',
    [1990]
)
for customer in customers:
    print(customer.first_name)

# Raw query with an annotation that maps to a model attribute
products = Product.objects.raw(
    '''
    SELECT id, name, price,
           (price * 0.9) AS discounted_price
    FROM myapp_product
    WHERE is_active = %s
    ''',
    [True]
)

# connection.cursor() — full access, returns raw tuples
def get_product_stats():
    with connection.cursor() as cursor:
        cursor.execute(
            '''
            SELECT category, COUNT(*) AS cnt, AVG(price) AS avg_price
            FROM myapp_product
            GROUP BY category
            ORDER BY cnt DESC
            '''
        )
        columns = [col[0] for col in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]
```

Calculate age in raw SQL (SQLite-specific syntax):  

```python
def get_users_by_age(age):

    adult_customers = Customer.objects.raw(
        '''
        SELECT *,
        CAST(strftime("%%Y", "now") - strftime("%%Y", dob) AS INTEGER) AS age
        FROM users_customer WHERE age >= %s
        ''',
        [age]
    )

    return adult_customers
```

Note that the `%` sign is doubled in `strftime` because Django's raw query  
interface uses `%s` as a placeholder and a literal `%` must be escaped as `%%`.  
`Manager.raw` supports most QuerySet operations (slicing, iteration) and  
returns proper model instances with deferred fields for any columns not  
included in the `SELECT`. `connection.cursor` provides unrestricted access to  
the underlying database driver and is the right choice for `INSERT`, `UPDATE`,  
`DELETE`, or DDL statements that the ORM does not need to know about. Both  
approaches automatically parameterise values to prevent SQL injection, but  
never interpolate user-supplied data directly into the SQL string.  
