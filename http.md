# HTTP Handling in Django

This document provides a comprehensive guide to handling HTTP requests and responses in Django. It covers the fundamental concepts and provides practical examples for common use cases.

## Introduction to HttpResponse and HttpRequest

In Django, the `HttpRequest` and `HttpResponse` objects are central to the way the framework handles web requests and responses. When a user's browser sends a request to a Django application, Django creates an `HttpRequest` object containing metadata about the request, such as the method (GET, POST, etc.), headers, and any data sent.

Your Django view function takes this `HttpRequest` object as its first argument and is responsible for returning an `HttpResponse` object. This `HttpResponse` object contains the content that will be sent back to the user's browser, along with other information like the content type and status code.

---

## Plain Text `HttpResponse`

A simple `HttpResponse` can be used to send a plain text response to the client. This is useful for simple API endpoints or for debugging purposes.

In `views.py`, you can create a view that returns a plain text response:

```python
from django.http import HttpResponse

def hello(req):
    """
    This view returns a simple plain text response.
    The content_type argument is set to 'text/plain' to inform the
    browser to render the response as plain text.
    """
    return HttpResponse('Hello there!', content_type='text/plain')
```

Alternatively, you can create an `HttpResponse` object and write to it:

```python
from django.http import HttpResponse

def hello(req):
    """
    This view demonstrates creating an HttpResponse object and then
    writing content to it before returning it.
    """
    http_response = HttpResponse('', content_type='text/plain')
    http_response.write('Ahoy!')
    return http_response
```

To make this view accessible, you need to map it to a URL in `urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('hello', views.hello, name='hello'),
]
```

You can test this by running `http localhost:8000/hello`.

---

## Limiting HTTP Methods

For security and correctness, it's often desirable to restrict a view to only accept certain HTTP methods. Django provides decorators for this purpose.

The `@require_GET` decorator ensures that the view can only be accessed using the GET method. If any other method is used, it will return a 405 Method Not Allowed response.

```python
from django.http import HttpResponse
from django.views.decorators.http import require_GET

@require_GET
def hello(req):
    """
    This view is restricted to only accept GET requests.
    """
    return HttpResponse('Hello there!', content_type='text/plain')
```

The `@require_http_methods` decorator is more flexible and allows you to specify a list of allowed methods. The `@csrf_exempt` decorator is used here to bypass CSRF protection, which is often necessary for APIs that don't use cookie-based authentication.

```python
from django.http import HttpResponse
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_exempt

@require_http_methods(["GET", "HEAD"])
@csrf_exempt
def status(req):
   """
   This view only accepts GET and HEAD requests.
   It returns a 200 OK status code.
   """
   return HttpResponse(status=200)
```

---

## URL Query Parameters

URL query parameters are a common way to pass data to a view. They are appended to the URL after a `?` and are accessed in Django via `req.GET`.

Here's a view that reads query parameters and returns them as a JSON response:

In `views.py`:

```python
from django.http import JsonResponse

def params(req):
    """
    This view extracts all query parameters from the request's GET attribute,
    converts them to a dictionary, and returns them as a JSON response.
    For example, a request to /params/?a=1&b=2 will return {"a": "1", "b": "2"}.
    """
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

---

## URL Path Parameters

Path parameters are parts of the URL that are captured and passed as arguments to the view. This is useful for creating clean, readable URLs. Django uses "path converters" to specify the expected data type of the parameter.

Common path converters:
*   `int`: Matches zero or any positive integer.
*   `str`: Matches any non-empty string, excluding the path separator ('/').
*   `slug`: Matches a slug string (alphanumeric, hyphens, and underscores).
*   `path`: Matches any non-empty string, including the path separator.
*   `uuid`: Matches a formatted UUID.

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
from django.http import HttpResponse, JsonResponse
from random import randint

def home(req):
    return HttpResponse('Home page.', content_type='text/plain')

def hello_url(req, name='Guest'):
    """
    This view takes a 'name' parameter from the URL path
    and includes it in the response.
    """
    return HttpResponse(f'hello {name}')

def random_vals(req, n=1):
    """
    This view takes an integer 'n' from the URL path and generates
    'n' random numbers, returning them in a JSON response.
    """
    rands = {e: randint(-100, 100) for e in range(1, n+1)}
    data = { 'vals': rands }
    return JsonResponse(data)
```

---

## `JsonResponse` Examples

`JsonResponse` is a subclass of `HttpResponse` that is specifically designed for returning JSON-encoded data. It sets the `Content-Type` header to `application/json` and serializes the given data to JSON.

### Returning JSON Data

```python
from django.http import JsonResponse

def user_data(req):
    """
    This view returns a dictionary as a JSON response.
    """
    data = {
        'name': 'John Doe',
        'age': 30,
        'email': 'john@example.com'
    }
    return JsonResponse(data)
```

### Returning a List as JSON

When returning a list or other non-dictionary object, you must set the `safe` parameter to `False`. This is a security measure to prevent a potential JSON hijacking vulnerability.

```python
from django.http import JsonResponse

def users_list(req):
    """
    This view returns a list of dictionaries as a JSON response.
    The `safe=False` parameter is required for non-dictionary objects.
    """
    users = [
        {'id': 1, 'name': 'John'},
        {'id': 2, 'name': 'Jane'},
        {'id': 3, 'name': 'Bob'}
    ]
    return JsonResponse(users, safe=False)
```

### Custom JSON Encoder

If your data contains types that the default JSON encoder cannot handle (like `datetime` objects), you can provide a custom encoder. Django provides `DjangoJSONEncoder` which can handle more data types.

```python
from django.http import JsonResponse
from django.core.serializers.json import DjangoJSONEncoder
from datetime import datetime

def timestamp_data(req):
    """
    This view demonstrates using a custom JSON encoder to serialize
    a dictionary containing a datetime object.
    """
    data = {
        'timestamp': datetime.now(),
        'message': 'Hello'
    }
    return JsonResponse(data, encoder=DjangoJSONEncoder)
```

---

## HTTP Headers

HTTP headers are a crucial part of the request-response cycle, carrying information about the client, server, and the data being transferred.

### Reading Request Headers

You can access request headers through the `req.META` dictionary. Header names are capitalized and prefixed with `HTTP_`.

```python
from django.http import HttpResponse

def show_headers(req):
    """
    This view reads the User-Agent and Content-Type headers from the
    request and displays them in the response.
    """
    user_agent = req.META.get('HTTP_USER_AGENT', 'Unknown')
    content_type = req.META.get('CONTENT_TYPE', 'Unknown')
    
    return HttpResponse(f'User-Agent: {user_agent}\nContent-Type: {content_type}')
```

### Setting Response Headers

You can set custom headers on the `HttpResponse` object just like a dictionary.

```python
from django.http import HttpResponse

def custom_headers(req):
    """
    This view sets custom headers on the HttpResponse object before
    returning it.
    """
    response = HttpResponse('Content with custom headers')
    response['X-Custom-Header'] = 'MyValue'
    response['Cache-Control'] = 'no-cache'
    return response
```

---

## Cookies

Cookies are small pieces of data stored on the client's browser. They are often used for session management and tracking.

### Setting Cookies

The `set_cookie` method on the `HttpResponse` object is used to set a cookie on the client.

```python
from django.http import HttpResponse

def set_cookie(req):
    """
    This view sets a cookie named 'username' with a max age of 3600 seconds.
    """
    response = HttpResponse('Cookie has been set')
    response.set_cookie('username', 'john_doe', max_age=3600)
    return response
```

### Reading Cookies

Incoming cookies are available in the `req.COOKIES` dictionary.

```python
from django.http import HttpResponse

def read_cookie(req):
    """
    This view reads the 'username' cookie from the request.
    If the cookie is not present, it defaults to 'Guest'.
    """
    username = req.COOKIES.get('username', 'Guest')
    return HttpResponse(f'Hello, {username}!')
```

### Deleting Cookies

The `delete_cookie` method is used to remove a cookie from the client's browser.

```python
from django.http import HttpResponse

def delete_cookie(req):
    """
    This view deletes the 'username' cookie.
    """
    response = HttpResponse('Cookie has been deleted')
    response.delete_cookie('username')
    return response
```

---

## POST Data Handling

Handling POST requests is essential for processing form submissions and API requests that send data.

### Processing Form Data

When a form is submitted with `Content-Type: application/x-www-form-urlencoded` or `multipart/form-data`, the data is available in `req.POST`.

```python
from django.http import HttpResponse, JsonResponse
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt # For demonstration purposes, disable CSRF protection
def process_form(req):
    """
    This view processes form data from a POST request.
    It's important to handle both GET and POST requests gracefully.
    """
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

### Processing JSON Body

For APIs, it's common to receive data in the request body as a JSON object. This data can be accessed from `req.body` and needs to be decoded.

```python
import json
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def process_json(req):
    """
    This view processes a JSON payload from a POST request.
    It includes error handling for invalid JSON.
    """
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

---

## File Uploads

Django provides a robust system for handling file uploads. Uploaded files are available in `req.FILES`.

### Handling a Single File Upload

```python
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def upload_file(req):
    """
    This view handles a single file upload. The uploaded file is saved
    to a temporary directory. In a real application, you would typically
    use Django's file storage system.
    """
    if req.method == 'POST' and req.FILES.get('file'):
        uploaded_file = req.FILES['file']
        
        # Save file to a temporary location
        with open(f'/tmp/{uploaded_file.name}', 'wb+') as destination:
            for chunk in uploaded_file.chunks():
                destination.write(chunk)
        
        return HttpResponse(f'File {uploaded_file.name} uploaded successfully')
    
    return HttpResponse('Upload a file via POST named "file"')
```

---

## File Downloads

Django can serve files as attachments, prompting the user to download them.

### `FileResponse` for Serving Files

`FileResponse` is a specialized `HttpResponse` subclass for streaming file content. It's memory-efficient for large files.

```python
from django.http import FileResponse, HttpResponse
import os

def download_file(req, filename):
    """
    This view serves a file for download using FileResponse.
    It sets the Content-Disposition header to trigger a download prompt.
    """
    # In a real application, this path should be carefully controlled
    # to avoid security vulnerabilities.
    file_path = f'/path/to/files/{filename}'
    
    if os.path.exists(file_path):
        response = FileResponse(open(file_path, 'rb'))
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        return response
    
    return HttpResponse('File not found', status=404)
```

### Streaming CSV Download

For dynamically generated content like CSV files, `StreamingHttpResponse` is ideal as it can send data in chunks without buffering the entire content in memory.

```python
from django.http import StreamingHttpResponse
import csv
import io

def download_csv(req):
    """
    This view streams a dynamically generated CSV file.
    """
    def csv_generator():
        # A simple generator for CSV rows
        yield ['Name', 'Age', 'Email']
        yield ['John', '30', 'john@example.com']
        yield ['Jane', '25', 'jane@example.com']

    def stream_csv():
        # A generator function that streams the CSV data
        output = io.StringIO()
        writer = csv.writer(output)
        for row in csv_generator():
            writer.writerow(row)
            output.seek(0)
            data = output.read()
            # Reset the buffer for the next row
            output.seek(0)
            output.truncate()
            yield data

    response = StreamingHttpResponse(stream_csv(), content_type='text/csv')
    response['Content-Disposition'] = 'attachment; filename="data.csv"'
    return response
```

---

## Redirects

Redirects are used to send the user from one URL to another.

### Simple Redirect

`HttpResponseRedirect` performs a temporary (302) redirect.

```python
from django.http import HttpResponseRedirect

def old_view(req):
    """
    This view performs a simple temporary redirect to a new URL.
    """
    return HttpResponseRedirect('/new-url/')
```

### Using the `redirect` Shortcut

The `redirect` shortcut is more flexible and can take a URL, a model, or a view name.

```python
from django.shortcuts import redirect

def redirect_view(req):
    """
    This view uses the redirect shortcut to redirect to a named URL.
    """
    return redirect('view-name')  # 'view-name' should be the name of a URL pattern
```

### Permanent Redirect

`HttpResponsePermanentRedirect` performs a permanent (301) redirect, which is useful for SEO when content has moved permanently.

```python
from django.http import HttpResponsePermanentRedirect

def old_page(req):
    """
    This view performs a permanent redirect.
    """
    return HttpResponsePermanentRedirect('/new-page/')
```

---

## Custom Status Codes

You can return any HTTP status code by setting the `status` argument on the `HttpResponse`.

### Returning Custom Status Codes

```python
from django.http import HttpResponse

def not_found(req):
    """Returns a 404 Not Found response."""
    return HttpResponse('Resource not found', status=404)

def server_error(req):
    """Returns a 500 Internal Server Error response."""
    return HttpResponse('Internal server error', status=500)

def created(req):
    """Returns a 201 Created response."""
    return HttpResponse('Resource created', status=201)
```

---

## Request Metadata

The `HttpRequest` object contains a wealth of information about the incoming request.

### Getting the Client's IP Address

The client's IP address can be found in `req.META`. It's important to check `HTTP_X_FORWARDED_FOR` if your application is behind a proxy.

```python
from django.http import HttpResponse

def get_client_ip(req):
    """
    This view determines the client's IP address, accounting for proxies.
    """
    x_forwarded_for = req.META.get('HTTP_X_FORWARDED_FOR')
    if x_forwarded_for:
        ip = x_forwarded_for.split(',')[0]
    else:
        ip = req.META.get('REMOTE_ADDR')
    
    return HttpResponse(f'Your IP address is: {ip}')
```

### General Request Information

You can inspect various attributes of the `HttpRequest` object to get more information about the request.

```python
from django.http import JsonResponse

def request_info(req):
    """
    This view returns a JSON response with various details about the request.
    """
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

---

## Session Handling

Django provides a session framework for storing data on the server for individual users.

### Setting Session Data

Session data is accessed via the `req.session` dictionary.

```python
from django.http import HttpResponse

def set_session(req):
    """
    This view sets or updates data in the user's session.
    """
    req.session['username'] = 'john_doe'
    req.session['visit_count'] = req.session.get('visit_count', 0) + 1
    
    return HttpResponse('Session data set')
```

### Reading Session Data

You can read data from the session in any subsequent request from the same user.

```python
from django.http import HttpResponse

def get_session(req):
    """
    This view reads data from the user's session.
    """
    username = req.session.get('username', 'Guest')
    visit_count = req.session.get('visit_count', 0)
    
    return HttpResponse(f'Hello {username}! Visits: {visit_count}')
```

---

## Content Negotiation

Content negotiation allows a client to request the same resource in different formats (e.g., JSON, XML, HTML).

### Returning Different Content Types

You can inspect the `Accept` header of the request to determine the client's preferred content type.

```python
from django.http import HttpResponse, JsonResponse

def content_negotiation(req):
    """
    This view returns either JSON or plain text based on the
    Accept header of the request.
    """
    accept = req.META.get('HTTP_ACCEPT', '')
    
    data = {'message': 'Hello', 'status': 'ok'}
    
    if 'application/json' in accept:
        return JsonResponse(data)
    else:
        return HttpResponse(f"Message: {data['message']}", content_type='text/plain')
```

---

## ETags for Cache Validation

ETags (entity tags) are identifiers for a specific version of a resource. They allow caches to be more efficient and save bandwidth, as a web server does not need to send a full response if the content has not changed.

```python
import hashlib
from django.http import HttpResponse, HttpRequest

def etag_example(request: HttpRequest) -> HttpResponse:
    """
    This view uses an ETag to avoid sending the same content repeatedly.
    """
    content = "some content that might change"
    etag = hashlib.md5(content.encode()).hexdigest()

    if request.headers.get('If-None-Match') == etag:
        return HttpResponse(status=304)

    response = HttpResponse(content)
    response['ETag'] = etag
    return response
```

---

## Last-Modified Headers

Similar to ETags, the `Last-Modified` header can be used for caching. It indicates the last time the resource was modified.

```python
from django.http import HttpResponse
from django.views.decorators.http import etag
import datetime

def last_modified_example(request):
    """
    This view uses the Last-Modified header for caching.
    """
    last_modified = datetime.datetime(2023, 10, 26, 10, 0, 0)

    if 'If-Modified-Since' in request.headers:
        if_modified_since = datetime.datetime.strptime(
            request.headers['If-Modified-Since'],
            "%a, %d %b %Y %H:%M:%S %Z"
        )
        if last_modified <= if_modified_since:
            return HttpResponse(status=304)

    response = HttpResponse("Content last modified at " + last_modified.isoformat())
    response['Last-Modified'] = last_modified.strftime("%a, %d %b %Y %H:%M:%S GMT")
    return response
```

---

## Conditional GET Requests

Django provides decorators to simplify handling of conditional GET requests.

```python
from django.views.decorators.http import condition
from django.http import HttpResponse
import datetime
import hashlib

def latest_entry(request):
    return datetime.datetime.now()

def entry_etag(request):
    return hashlib.md5(str(latest_entry(request)).encode()).hexdigest()

@condition(etag_func=entry_etag, last_modified_func=latest_entry)
def conditional_view(request):
    """
    This view uses the @condition decorator to handle conditional GETs.
    """
    return HttpResponse("Latest content")
```

---

## Content-Security-Policy (CSP) Header

CSP is a security header that helps prevent cross-site scripting (XSS) and other code injection attacks.

```python
from django.http import HttpResponse

def csp_view(request):
    """
    This view sets a restrictive Content-Security-Policy header.
    """
    response = HttpResponse("Content with CSP")
    response['Content-Security-Policy'] = "default-src 'self'"
    return response
```

---

## HTTP Strict Transport Security (HSTS)

HSTS is a security header that tells browsers to only communicate with the server over HTTPS. In `settings.py`:

```python
# settings.py
SECURE_HSTS_SECONDS = 3600 # 1 hour
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
SECURE_SSL_REDIRECT = True
```

---

## Cross-Origin Resource Sharing (CORS)

CORS headers are necessary to allow web pages from one domain to make requests to another domain. A popular package for this is `django-cors-headers`.

After installing `django-cors-headers`, configure it in `settings.py`:
```python
# settings.py
INSTALLED_APPS = [
    ...,
    'corsheaders',
    ...,
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    ...,
]

CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://sub.example.com",
    "http://localhost:8080",
    "http://127.0.0.1:9000",
]
```

---

## HTTP Basic Authentication

A simple way to protect a view is with HTTP Basic Authentication.

```python
import base64
from django.http import HttpResponse

def basic_auth_required(view_func):
    def _wrapped_view(request, *args, **kwargs):
        if 'HTTP_AUTHORIZATION' in request.META:
            auth = request.META['HTTP_AUTHORIZATION'].split()
            if len(auth) == 2 and auth[0].lower() == "basic":
                uname, passwd = base64.b64decode(auth[1]).decode('utf-8').split(':')
                if uname == 'user' and passwd == 'password':
                    return view_func(request, *args, **kwargs)

        response = HttpResponse(status=401)
        response['WWW-Authenticate'] = 'Basic realm="Restricted"'
        return response
    return _wrapped_view

@basic_auth_required
def protected_view(request):
    return HttpResponse("This is a protected view.")
```

---

## Custom Error Handlers (404, 500)

You can define your own views to handle errors like 404 Not Found and 500 Server Error.

In `urls.py`:
```python
# urls.py
handler404 = 'myapp.views.custom_404'
handler500 = 'myapp.views.custom_500'
```

In `views.py`:
```python
# myapp/views.py
from django.shortcuts import render

def custom_404(request, exception):
    return render(request, '404.html', status=404)

def custom_500(request):
    return render(request, '500.html', status=500)
```

---

## GZip Compression

You can use Django's `GZipMiddleware` to compress responses, which can significantly improve performance.

In `settings.py`:
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.gzip.GZipMiddleware',
    ...,
]
```

---

## Signing Cookies

For security, you can sign cookies to make them tamper-proof.

```python
from django.core.signing import Signer, BadSignature
from django.http import HttpResponse

def set_signed_cookie(request):
    signer = Signer()
    response = HttpResponse("Signed cookie set.")
    response.set_cookie('signed_cookie', signer.sign('my_value'))
    return response

def read_signed_cookie(request):
    signer = Signer()
    try:
        value = signer.unsign(request.COOKIES['signed_cookie'])
        return HttpResponse(f"The value is {value}")
    except (KeyError, BadSignature):
        return HttpResponse("Invalid or missing signed cookie.", status=400)
```

---

## HTTP/2 and HTTP/3 Awareness

Django itself is protocol-agnostic. Support for HTTP/2 and HTTP/3 depends on your deployment server (e.g., Nginx, Gunicorn). When running behind a capable server, Django will transparently benefit from the features of these newer protocols, like multiplexing and header compression.

---

## Serving Static Media in Development

During development, you can configure Django to serve user-uploaded media files.

In `urls.py`:
```python
# urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... your other urls
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## Class-Based Views for HTTP Dispatching

Class-based views provide an alternative way to structure your views. They are particularly well-suited for handling different HTTP methods.

```python
from django.http import HttpResponse
from django.views import View

class MyView(View):
    def get(self, request):
        return HttpResponse("This is a GET request.")

    def post(self, request):
        return HttpResponse("This is a POST request.")
```

---

## `QueryDict` Objects

`request.GET` and `request.POST` are instances of `django.http.QueryDict`. A key difference from a standard dictionary is that `QueryDict` can handle multiple values for the same key.

```python
from django.http import JsonResponse

def querydict_example(request):
    """
    Demonstrates handling a QueryDict with multiple values for a key.
    A request to /?a=1&a=2 will return {'a': ['1', '2']}.
    """
    return JsonResponse(request.GET.lists())
```

---

## The `HttpRequest.headers` attribute

As of Django 3.0, you can access request headers via the `request.headers` attribute, which is a case-insensitive, dict-like object. This is the modern way to access headers.

```python
from django.http import HttpResponse

def modern_header_access(request):
    user_agent = request.headers.get('User-Agent', 'Unknown')
    return HttpResponse(f"User-Agent: {user_agent}")
```

---

## `HttpResponseNotModified`

This is a shortcut for returning a 304 Not Modified response. It should not have any content.

```python
from django.http import HttpResponseNotModified

def not_modified_view(request):
    return HttpResponseNotModified()
```

---

## `HttpResponseForbidden`

A shortcut for returning a 403 Forbidden response.

```python
from django.http import HttpResponseForbidden

def forbidden_view(request):
    return HttpResponseForbidden("You are not allowed here.")
```

---

## `HttpResponseBadRequest`

A shortcut for a 400 Bad Request response.

```python
from django.http import HttpResponseBadRequest

def bad_request_view(request):
    return HttpResponseBadRequest("This was a bad request.")
```

---

## `HttpResponseGone`

A shortcut for a 410 Gone response, indicating that the resource is no longer available.

```python
from django.http import HttpResponseGone

def gone_view(request):
    return HttpResponseGone("This resource is gone.")
```

---

## Asynchronous Views

Django supports asynchronous ("async") views, which can be beneficial for long-running I/O-bound tasks.

```python
import asyncio
from django.http import HttpResponse

async def async_view(request):
    await asyncio.sleep(1)
    return HttpResponse("This is an async view.")
```
