# HTTP 


## Plain text HttpResponse

In `urls.py`:  

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.home),
    path('hello', views.hello, name='hello'),
]
```

In `views.py`:  

```python
from django.http import HttpResponse

def hello(req):

    return HttpResponse('Hello there!', content_type='text/plain')
```

or 

```python
from django.http import HttpResponse

def hello(req):

    http_response = HttpResponse('', content_type='text/plain')
    http_response.write('Ahoy!')

    return http_response
```

launch request with:  `http localhost:8000/hello`  

## Limiting methods

Limiting views to a specific HTTP method.  

```python
from django.http import HttpResponse
from django.views.decorators.http import require_GET, require_http_methods
from django.views.decorators.csrf import csrf_exempt

@require_GET
def hello(req):

    return HttpResponse('Hello there!', content_type='text/plain')

@require_http_methods(["GET", "HEAD"])
@csrf_exempt
def status(req):
   
   return HttpResponse(status=200)
```

## URL parameters

In `views.py`:  

```python
def params(req):

    params = req.GET.dict()
    return JsonResponse(params)
```

In `urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path('params/', views.params),
]
```

## URL path parameters

Path convertors: 

* `int` – Matches zero or any positive integer.
* `str` – Matches any non-empty string, excluding the path separator ('/').
* `slug` – Matches any slug string, i.e. a string consisting of alphabets, digits, hyphen and underscore.
* `path` – Matches any non-empty string including the path separator ('/')
* `uuid` – Matches a UUID (universal unique identifier).

In `urls.py`:

```python
from django.urls import path

from . import views

urlpatterns = [
    path('hello/<name>/', views.hello_url),
    path('random/<int:n>/', views.random_vals),
    path('', views.home),
]
```

In `views.py`:

```python
def home(req):
    return HttpResponse('Home page.', content_type='text/plain')

def hello_url(req, name='Guest'):
    return HttpResponse(f'hello {name}')

def random_vals(req, n=1):
    rands = {e: randint(-100, 100) for e in range(1, n+1)}
    print(rands)
    data = { 'vals': rands }
    return JsonResponse(data)
```

## JsonResponse examples

### Returning JSON data

```python
from django.http import JsonResponse

def user_data(req):
    data = {
        'name': 'John Doe',
        'age': 30,
        'email': 'john@example.com'
    }
    return JsonResponse(data)
```

### Returning a list as JSON

```python
from django.http import JsonResponse

def users_list(req):
    users = [
        {'id': 1, 'name': 'John'},
        {'id': 2, 'name': 'Jane'},
        {'id': 3, 'name': 'Bob'}
    ]
    return JsonResponse(users, safe=False)
```

### Custom JSON encoder

```python
from django.http import JsonResponse
from django.core.serializers.json import DjangoJSONEncoder
from datetime import datetime

def timestamp_data(req):
    data = {
        'timestamp': datetime.now(),
        'message': 'Hello'
    }
    return JsonResponse(data, encoder=DjangoJSONEncoder)
```

## HTTP Headers

### Reading request headers

```python
from django.http import HttpResponse

def show_headers(req):
    user_agent = req.META.get('HTTP_USER_AGENT', 'Unknown')
    content_type = req.META.get('CONTENT_TYPE', 'Unknown')
    
    return HttpResponse(f'User-Agent: {user_agent}\nContent-Type: {content_type}')
```

### Setting response headers

```python
from django.http import HttpResponse

def custom_headers(req):
    response = HttpResponse('Content with custom headers')
    response['X-Custom-Header'] = 'MyValue'
    response['Cache-Control'] = 'no-cache'
    return response
```

## Cookies

### Setting cookies

```python
from django.http import HttpResponse

def set_cookie(req):
    response = HttpResponse('Cookie has been set')
    response.set_cookie('username', 'john_doe', max_age=3600)
    return response
```

### Reading cookies

```python
from django.http import HttpResponse

def read_cookie(req):
    username = req.COOKIES.get('username', 'Guest')
    return HttpResponse(f'Hello, {username}!')
```

### Deleting cookies

```python
from django.http import HttpResponse

def delete_cookie(req):
    response = HttpResponse('Cookie has been deleted')
    response.delete_cookie('username')
    return response
```

## POST data handling

### Processing form data

```python
from django.http import HttpResponse, JsonResponse

def process_form(req):
    if req.method == 'POST':
        name = req.POST.get('name', '')
        email = req.POST.get('email', '')
        
        return JsonResponse({
            'status': 'success',
            'name': name,
            'email': email
        })
    
    return HttpResponse('Send POST request')
```

### Processing JSON body

```python
import json
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def process_json(req):
    if req.method == 'POST':
        try:
            data = json.loads(req.body)
            name = data.get('name', '')
            
            return JsonResponse({
                'status': 'success',
                'received': data
            })
        except json.JSONDecodeError:
            return JsonResponse({'error': 'Invalid JSON'}, status=400)
    
    return JsonResponse({'error': 'POST required'}, status=405)
```

## File uploads

### Handling file upload

```python
from django.http import HttpResponse

def upload_file(req):
    if req.method == 'POST' and req.FILES:
        uploaded_file = req.FILES['file']
        
        # Save file
        with open(f'/tmp/{uploaded_file.name}', 'wb+') as destination:
            for chunk in uploaded_file.chunks():
                destination.write(chunk)
        
        return HttpResponse(f'File {uploaded_file.name} uploaded successfully')
    
    return HttpResponse('Upload a file via POST')
```

## File downloads

### FileResponse for serving files

```python
from django.http import FileResponse
import os

def download_file(req, filename):
    file_path = f'/path/to/files/{filename}'
    
    if os.path.exists(file_path):
        response = FileResponse(open(file_path, 'rb'))
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        return response
    
    return HttpResponse('File not found', status=404)
```

### Streaming CSV download

```python
from django.http import StreamingHttpResponse
import csv

def download_csv(req):
    def csv_generator():
        yield ['Name', 'Age', 'Email']
        yield ['John', '30', 'john@example.com']
        yield ['Jane', '25', 'jane@example.com']
    
    def generate():
        import io
        output = io.StringIO()
        writer = csv.writer(output)
        for row in csv_generator():
            writer.writerow(row)
            output.seek(0)
            data = output.read()
            output.seek(0)
            output.truncate()
            yield data
    
    response = StreamingHttpResponse(generate(), content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="data.csv"'
    return response
```

## Redirects

### Simple redirect

```python
from django.http import HttpResponseRedirect
from django.urls import reverse

def old_view(req):
    return HttpResponseRedirect('/new-url/')
```

### Using redirect shortcut

```python
from django.shortcuts import redirect

def redirect_view(req):
    return redirect('view-name')  # Redirect to named URL
```

### Permanent redirect

```python
from django.http import HttpResponsePermanentRedirect

def old_page(req):
    return HttpResponsePermanentRedirect('/new-page/')
```

## Custom status codes

### Returning custom status codes

```python
from django.http import HttpResponse

def not_found(req):
    return HttpResponse('Resource not found', status=404)

def server_error(req):
    return HttpResponse('Internal server error', status=500)

def created(req):
    return HttpResponse('Resource created', status=201)
```

## Request metadata

### Getting client IP address

```python
from django.http import HttpResponse

def get_client_ip(req):
    x_forwarded_for = req.META.get('HTTP_X_FORWARDED_FOR')
    if x_forwarded_for:
        ip = x_forwarded_for.split(',')[0]
    else:
        ip = req.META.get('REMOTE_ADDR')
    
    return HttpResponse(f'Your IP address is: {ip}')
```

### Request information

```python
from django.http import JsonResponse

def request_info(req):
    info = {
        'method': req.method,
        'path': req.path,
        'scheme': req.scheme,
        'is_secure': req.is_secure(),
        'is_ajax': req.headers.get('X-Requested-With') == 'XMLHttpRequest',
        'content_type': req.content_type,
    }
    return JsonResponse(info)
```

## Session handling

### Setting session data

```python
from django.http import HttpResponse

def set_session(req):
    req.session['username'] = 'john_doe'
    req.session['visit_count'] = req.session.get('visit_count', 0) + 1
    
    return HttpResponse('Session data set')
```

### Reading session data

```python
from django.http import HttpResponse

def get_session(req):
    username = req.session.get('username', 'Guest')
    visit_count = req.session.get('visit_count', 0)
    
    return HttpResponse(f'Hello {username}! Visits: {visit_count}')
```

## Content negotiation

### Returning different content types

```python
from django.http import HttpResponse, JsonResponse

def content_negotiation(req):
    accept = req.META.get('HTTP_ACCEPT', '')
    
    data = {'message': 'Hello', 'status': 'ok'}
    
    if 'application/json' in accept:
        return JsonResponse(data)
    else:
        return HttpResponse(f"Message: {data['message']}", content_type='text/plain')
```
