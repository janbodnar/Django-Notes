# Database 


## PostgreSQL setup

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'localhost',
        'NAME': 'testdb',
        'USER': 'postgres',
        'PASSWORD': 'andrea',
    }
}
```

## Raw SQL


```python
from django.db import connection

from .models import Customer


def test_raw_sql(req):

    with connection.cursor() as cur:
        cur.execute('SELECT * FROM myapp_customer')
        rows = cur.fetchall()

        columns = [col[0] for col in cur.description]
        res = [dict(zip(columns, row)) for row in rows]

    return JsonResponse({"users": res})
```
