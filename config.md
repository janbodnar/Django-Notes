# Configuration

They `python-decouple` package can be used for configuration.  

Inside the `.env` file:

```
SECRET_KEY=@lywhkp7^ccmvjv03pv6kh&y8f3+gtj49i@*bace0tb0lo6nb%
ENGINE=django.db.backends.postgresql
HOST=localhost
NAME=testdb
USER=postgres
PASSWORD=andrea
APP_NAME=myapp
```

In `settings.py` file: 

```python
from decouple import config

SECRET_KEY = config('SECRET_KEY')

...

DATABASES = {
    'default': {
        'ENGINE': config('ENGINE'),
        'HOST': config('HOST'),
        'NAME': config('NAME'),
        'USER': config('USER'),
        'PASSWORD': config('PASSWORD'),
    }
}
```
