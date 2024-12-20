# ORM

Django ORM, or Object-Relational Mapping, is a powerful tool in the Django web  
framework that allows developers to interact with their database in an intuitive,  
Pythonic way. Instead of writing raw SQL queries, you can use Django's ORM to  
define your data models as Python classes and interact with the database using  
Python code. This abstraction layer simplifies database operations, improves code  
readability, and helps keep your database interactions consistent and secure.  
It's like having a translator between your Python code and the SQL database, making  
database operations smoother and more integrated with your application logic. 

## Key Features and Benefits

- **Automatic Data Mapping:** ORM automatically maps your Python classes (models) to  
  corresponding database tables, eliminating the need for manual SQL queries.
- **Query API:** Provides a high-level API for querying and manipulating data in the  
  database using Python syntax.
- **Data Validation:** Enforces data integrity by automatically validating data against  
   defined constraints (e.g., required fields, data types, unique constraints).
- **Relationships:** Handles complex relationships between different data models  
  (e.g., one-to-one, one-to-many, many-to-many) seamlessly.
- **Migrations:** Automatically generates database schema changes based on model  
  modifications, making database updates efficient and less error-prone.
- **Abstraction:** Provides a layer of abstraction over the database, making your code  
  more portable and less dependent on specific database technologies.


Django ORM simplifies database interactions in Django web applications by providing a high-level,  
object-oriented interface. It automates many common database tasks, making development more efficient  
and reducing the likelihood of errors.


## QuerySet

A Django `QuerySet` is a collection of database queries to retrieve objects from your database.  
It's like a list of objects but with database superpowers: you can filter, sort, and manipulate  
the data without writing raw SQL queries. QuerySets are lazy, meaning they're not executed until  
you actually need the data. This efficiency lets you chain multiple filters and operations without  
hitting the database multiple times. It's a powerful feature that makes interacting with your data  
both efficient and Pythonic.


## The annotate method

The annotate method in Django ORM is used to add a calculated field to each object in  
a `QuerySet`. Essentially, it allows you to generate summary values on a per-object basis.  
These annotations are defined as expressions that use the aggregate functions provided  
by Django's ORM, like `Count`, `Sum`, `Avg`, `Max`, and `Min`. For example, if you have a model  
for books and you want to add the number of authors for each book, you could use annotate  
with `Count` to achieve this. It's a way to enrich your `QuerySet` with additional computed data.


Adding the `calculated_age` attribute via the `annotate` method.

```python
def get_users_by_age(age):

    today = datetime.now()
    cutoff_date = today.replace(year=today.year - age)

    # Querying customers whose dob is less than or equal to cutoff_date
    customers = Customer.objects.annotate(
        dob_as_date=Cast('dob', output_field=models.DateField()),
        calculated_age=models.ExpressionWrapper(
            today - models.F('dob'), output_field=models.IntegerField())
    ).filter(dob_as_date__lte=cutoff_date)

    return customers
```

Similarly, we can create a custom `age` property on the model:

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

Attributes are later referred to in the template: 

```html
<ul>
    {% for customer in customers %}

    <li>{{ customer.first_name }} {{ customer.last_name }} {{customer.occupation}} {{ customer.age }}</li>

    {% endfor%}
</ul>
```


## Raw SQL

 Calculate age in raw SQL (SQLite specific syntax):  

```python
def get_users_by_age(age):

    adult_customers = Customer.objects.raw(
        '''
        SELECT *,
        CAST(strftime("%%Y", "now") - strftime("%%Y", dob) AS INTEGER) AS age 
        FROM users_customer WHERE age >= %s''', [age]
    )

    return adult_customers
```

Note that the `%` sign is double in the `strftime` function. 
