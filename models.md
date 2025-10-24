# Models

Django models are the heart of any Django application's data layer. They define the structure  
of your database tables using Python classes, allowing you to interact with your database in an  
object-oriented way. Each model class represents a database table, and each attribute of the  
class represents a database field. Django's models provide a powerful abstraction that eliminates  
the need to write raw SQL for most database operations. They handle everything from creating  
database schemas to querying data, making database management intuitive and Pythonic.

## Model Basics

Models are defined in your app's `models.py` file and inherit from `django.db.models.Model`.  
Each attribute you define on your model class becomes a field in the corresponding database table.  
Django automatically creates a primary key field for you, handles database migrations through the  
`makemigrations` and `migrate` commands, and provides a rich API for querying and manipulating data.

## id

By default, Django adds an `id` field to each model, which is used as the primary key  
for that model. You can create your own primary key field by adding the keyword arg  
`primary_key=True` to a field. If you add your own primary key field, the automatic  
one will not be added.  


## Ordering

The `ordering` attribute in the `Meta` class determines the default order in which objects  
are retrieved from the database. This is particularly useful when you want your QuerySets  
to always return results in a specific order without having to specify it in every query.  
Use a minus sign (-) before the field name for descending order.

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

By default, Django creates a table name by combining the app name and model name in lowercase  
(e.g., `myapp_customer`). You can override this default behavior using the `db_table` attribute  
in the model's `Meta` class to specify a custom table name. This is useful when integrating  
with legacy databases or following specific naming conventions.

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

## Common field types

Django provides a variety of field types to represent different kinds of data in your database.  
Each field type maps to a specific database column type and provides validation and conversion  
between Python and database values. Choosing the right field type ensures data integrity and  
optimal database performance.

```python
from django.db import models

class Product(models.Model):
    
    # Text fields
    name = models.CharField(max_length=200)  # Short text, required max_length
    description = models.TextField()  # Long text, no max_length needed
    
    # Numeric fields
    price = models.DecimalField(max_digits=10, decimal_places=2)  # Precise decimal
    quantity = models.IntegerField(default=0)  # Whole numbers
    rating = models.FloatField(null=True, blank=True)  # Floating point
    
    # Date and time fields
    created_at = models.DateTimeField(auto_now_add=True)  # Set on creation
    updated_at = models.DateTimeField(auto_now=True)  # Updated on save
    release_date = models.DateField(null=True)  # Date only
    
    # Boolean field
    is_active = models.BooleanField(default=True)
    
    # Email field with validation
    contact_email = models.EmailField(blank=True)
    
    # URL field with validation
    product_url = models.URLField(blank=True)
    
    def __str__(self):
        return self.name
```

## ForeignKey relationships

A `ForeignKey` establishes a many-to-one relationship between models. It's used when many instances  
of one model are related to a single instance of another model. For example, many books can belong  
to one author. The `on_delete` parameter specifies what happens to related objects when the  
referenced object is deleted.

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=200)
    birth_date = models.DateField()
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        Author, 
        on_delete=models.CASCADE,  # Delete books when author is deleted
        related_name='books'  # Access books via author.books.all()
    )
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    
    def __str__(self):
        return self.title
```

## ManyToMany relationships

A `ManyToManyField` creates a many-to-many relationship where multiple instances of one model  
can be related to multiple instances of another model. Django automatically creates an intermediate  
join table to manage these relationships. For example, a student can enroll in many courses,  
and each course can have many students.

```python
from django.db import models

class Student(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    student_id = models.CharField(max_length=20, unique=True)
    
    def __str__(self):
        return f'{self.first_name} {self.last_name}'

class Course(models.Model):
    name = models.CharField(max_length=200)
    code = models.CharField(max_length=10, unique=True)
    students = models.ManyToManyField(
        Student,
        related_name='courses',  # Access courses via student.courses.all()
        blank=True  # Course can exist without students
    )
    credits = models.IntegerField()
    
    def __str__(self):
        return self.name
```

## Model methods

Model methods are custom functions defined on your model class that encapsulate business logic  
or provide convenient ways to work with model instances. Common methods include `__str__` for  
string representation, custom `save` methods for pre-save logic, and `get_absolute_url` for  
generating URLs to specific model instances.

```python
from django.db import models
from django.urls import reverse
from django.utils.text import slugify

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    slug = models.SlugField(max_length=200, unique=True, blank=True)
    author = models.CharField(max_length=100)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        # String representation shown in admin and shell
        return self.title
    
    def save(self, *args, **kwargs):
        # Custom save logic: auto-generate slug from title
        if not self.slug:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
    
    def get_absolute_url(self):
        # Generate URL for this article
        return reverse('article-detail', kwargs={'slug': self.slug})
    
    def word_count(self):
        # Custom method to count words in content
        return len(self.content.split())
```

## Model properties

Properties are computed attributes that behave like regular model fields but are calculated  
on-the-fly rather than stored in the database. They're useful for derived data that doesn't  
need to be persisted. Use the `@property` decorator to define them.

```python
from django.db import models
from datetime import date

class Employee(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    date_of_birth = models.DateField()
    salary = models.DecimalField(max_digits=10, decimal_places=2)
    hire_date = models.DateField()
    
    def __str__(self):
        return self.full_name
    
    @property
    def full_name(self):
        # Computed property for full name
        return f'{self.first_name} {self.last_name}'
    
    @property
    def age(self):
        # Calculate age from date of birth
        today = date.today()
        return today.year - self.date_of_birth.year - (
            (today.month, today.day) < 
            (self.date_of_birth.month, self.date_of_birth.day)
        )
    
    @property
    def years_of_service(self):
        # Calculate years working at company
        today = date.today()
        return today.year - self.hire_date.year
```

## Default values and null/blank options

Understanding `null`, `blank`, and `default` is crucial for proper model design. `null=True`  
allows NULL values in the database, `blank=True` allows empty values in forms, and `default`  
provides a default value when a new object is created. Different field types have different  
best practices for these options.

```python
from django.db import models
from django.utils import timezone

class BlogPost(models.Model):
    # Required field (null=False, blank=False by default)
    title = models.CharField(max_length=200)
    
    # Optional text field (use blank=True, not null=True for text fields)
    subtitle = models.CharField(max_length=200, blank=True, default='')
    
    # Optional with null allowed (for non-text fields)
    published_date = models.DateTimeField(null=True, blank=True)
    
    # Field with default value
    status = models.CharField(
        max_length=20,
        default='draft',
        choices=[
            ('draft', 'Draft'),
            ('published', 'Published'),
            ('archived', 'Archived'),
        ]
    )
    
    # Default using callable (evaluates at object creation)
    created_at = models.DateTimeField(default=timezone.now)
    
    # Required foreign key
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    
    # Optional foreign key
    editor = models.ForeignKey(
        'auth.User',
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='edited_posts'
    )
    
    def __str__(self):
        return self.title
```

## Unique constraints

Unique constraints ensure that specific fields or combinations of fields contain unique values  
across all records in the table. This is essential for maintaining data integrity. Use `unique=True`  
for single fields or `unique_together` in Meta for multiple fields.

```python
from django.db import models

class Company(models.Model):
    name = models.CharField(max_length=200)
    # Single field unique constraint
    tax_id = models.CharField(max_length=20, unique=True)
    registration_number = models.CharField(max_length=50, unique=True)
    email = models.EmailField(unique=True)
    
    def __str__(self):
        return self.name

class Project(models.Model):
    company = models.ForeignKey(Company, on_delete=models.CASCADE)
    name = models.CharField(max_length=200)
    code = models.CharField(max_length=10)
    year = models.IntegerField()
    
    class Meta:
        # Combination of fields must be unique
        unique_together = [['company', 'code'], ['company', 'name', 'year']]
        
    def __str__(self):
        return f'{self.company.name} - {self.name}'

class Membership(models.Model):
    user = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    organization = models.ForeignKey(Company, on_delete=models.CASCADE)
    role = models.CharField(max_length=50)
    joined_date = models.DateField(auto_now_add=True)
    
    class Meta:
        # Using constraints (Django 2.2+) - more flexible than unique_together
        constraints = [
            models.UniqueConstraint(
                fields=['user', 'organization'],
                name='unique_user_organization'
            )
        ]
```

## Database indexes

Indexes improve query performance by creating optimized data structures for fast lookups.  
Use `db_index=True` on frequently queried fields or define composite indexes in the Meta class.  
While indexes speed up reads, they can slow down writes, so use them judiciously.

```python
from django.db import models

class Customer(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    
    # Single field index - useful for frequent lookups
    email = models.EmailField(unique=True, db_index=True)
    
    # Indexed for filtering and sorting
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    country = models.CharField(max_length=50, db_index=True)
    
    status = models.CharField(
        max_length=20,
        choices=[('active', 'Active'), ('inactive', 'Inactive')],
        db_index=True
    )
    
    class Meta:
        # Composite index for queries that filter by multiple fields
        indexes = [
            models.Index(fields=['last_name', 'first_name']),
            models.Index(fields=['country', 'status']),
            models.Index(fields=['-created_at']),  # Descending order index
        ]
        
    def __str__(self):
        return f'{self.first_name} {self.last_name}'
```

## Custom managers

Custom managers allow you to define reusable query methods and modify the default QuerySet  
returned by `Model.objects`. They encapsulate common queries and business logic, making your  
code more maintainable and DRY (Don't Repeat Yourself).

```python
from django.db import models

class PublishedManager(models.Manager):
    """Custom manager that returns only published posts"""
    def get_queryset(self):
        return super().get_queryset().filter(status='published')

class FeaturedManager(models.Manager):
    """Custom manager that returns featured posts"""
    def get_queryset(self):
        return super().get_queryset().filter(is_featured=True, status='published')

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    status = models.CharField(
        max_length=20,
        choices=[
            ('draft', 'Draft'),
            ('published', 'Published'),
            ('archived', 'Archived'),
        ],
        default='draft'
    )
    is_featured = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    # Default manager (must be first)
    objects = models.Manager()
    
    # Custom managers
    published = PublishedManager()
    featured = FeaturedManager()
    
    def __str__(self):
        return self.title

# Usage examples:
# Post.objects.all()  # All posts
# Post.published.all()  # Only published posts
# Post.featured.all()  # Only featured, published posts
```

## Model inheritance

Django supports three types of model inheritance: abstract base classes, multi-table inheritance,  
and proxy models. Abstract base classes are the most common and allow you to define common fields  
and methods that multiple models can share without creating separate database tables for the  
base class.

```python
from django.db import models

# Abstract base class - no database table created
class TimeStampedModel(models.Model):
    """Abstract model providing created and modified timestamps"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True  # This makes it an abstract base class

class CommonInfo(models.Model):
    """Another abstract base class with common fields"""
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        abstract = True
        ordering = ['name']
    
    def __str__(self):
        return self.name

# Concrete models inheriting from abstract base classes
class Category(TimeStampedModel, CommonInfo):
    """Inherits fields from both TimeStampedModel and CommonInfo"""
    slug = models.SlugField(unique=True)
    
    class Meta(CommonInfo.Meta):
        verbose_name_plural = 'Categories'

class Tag(TimeStampedModel, CommonInfo):
    """Another model using the same base classes"""
    color = models.CharField(max_length=7, default='#000000')  # Hex color
    
# Multi-table inheritance - creates separate tables with automatic OneToOne
class Place(models.Model):
    name = models.CharField(max_length=200)
    address = models.CharField(max_length=200)

class Restaurant(Place):
    """Inherits from Place and adds restaurant-specific fields"""
    cuisine = models.CharField(max_length=100)
    serves_alcohol = models.BooleanField(default=False)
```
