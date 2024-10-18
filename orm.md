# ORM

Django ORM stands for **Object-Relational Mapper**. It's a powerful tool that bridges  
the gap between your Python objects and the underlying relational database  
(like SQLite, PostgreSQL, MySQL, etc.) in Django web applications.

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
