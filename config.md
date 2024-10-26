# Configuration

They `python-decouple` package can be used for configuration.  

Inside the `.env` file:

```
SECRET_KEY=django-insecure-zl&sv%+yvc_0=618^sr)87j$*l#gy_3mv8eu_&(!_idocyphk8
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
