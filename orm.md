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
