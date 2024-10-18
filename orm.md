# ORM



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
