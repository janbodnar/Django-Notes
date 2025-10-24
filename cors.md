# CORS

Cross-Origin Resource Sharing (CORS) is a security mechanism implemented by web  
browsers that controls how web pages from one domain can request and access  
resources from a different domain. It's a critical security feature that prevents  
unauthorized cross-origin requests while allowing legitimate ones through a system  
of HTTP headers. CORS works by having the browser send an HTTP request with an  
`Origin` header, and the server responds with specific CORS headers that tell the  
browser whether the cross-origin request should be allowed. For simple requests,  
this happens transparently, but for complex requests (like those with custom  
headers or methods other than GET/POST), the browser first sends a preflight  
OPTIONS request to check if the actual request is safe to send.  

Django doesn't include built-in CORS support because it's primarily a server-side  
framework, and CORS is enforced by browsers on the client side. However, Django  
applications often need to handle CORS when building APIs that will be consumed by  
frontend applications running on different domains, such as React, Vue, or Angular  
SPAs. The most common approach is to use the `django-cors-headers` package, which  
provides middleware that automatically adds the appropriate CORS headers to  
responses. Alternatively, developers can implement custom middleware or use  
decorators to handle CORS for specific views. Understanding CORS is essential for  
modern web development, especially when building RESTful APIs, as misconfigured  
CORS policies can either expose applications to security vulnerabilities or  
prevent legitimate requests from functioning properly.  


## Why CORS Exists

CORS exists to enforce the Same-Origin Policy, a fundamental security concept in  
web browsers. Without CORS, any website could make requests to any other website  
using the visitor's credentials, potentially accessing sensitive data or performing  
unauthorized actions. For example, a malicious site could make requests to a  
banking website and steal user data if the browser didn't enforce origin  
restrictions. CORS provides a controlled way to relax these restrictions when  
necessary.  

The Same-Origin Policy considers two URLs to have the same origin only if they  
match in protocol (http/https), domain, and port. This means that `http://api.example.com`  
and `https://api.example.com` are different origins, as are `http://example.com:8000`  
and `http://example.com:8080`. CORS headers allow servers to specify which origins  
are permitted to access their resources, what HTTP methods are allowed, and  
whether credentials like cookies can be included in cross-origin requests.  


## Installing django-cors-headers

```bash
pip install django-cors-headers
```

The `django-cors-headers` package is the most popular and recommended solution  
for handling CORS in Django applications. It provides a middleware component that  
automatically adds appropriate CORS headers to HTTP responses based on  
configuration settings in your Django settings file. This package supports all  
CORS features including preflight requests, credentials, custom headers, and  
fine-grained origin control.  


## Basic CORS Setup

Add the package to installed apps and middleware in `settings.py`:  

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'corsheaders',  # Add django-cors-headers
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',  # Should be before CommonMiddleware
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

The middleware must be placed before `CommonMiddleware` to ensure that CORS  
headers are added before Django processes the response. This basic setup installs  
the package but doesn't enable CORS yet - you need to configure which origins are  
allowed through additional settings. The middleware intercepts every HTTP response  
and adds the appropriate CORS headers based on the request's `Origin` header and  
your configuration settings.  


## Allow All Origins

```python
CORS_ALLOW_ALL_ORIGINS = True
```

This setting allows requests from any origin, which is useful during development  
but should never be used in production. When enabled, the middleware will add  
`Access-Control-Allow-Origin: *` to all responses, allowing any website to make  
requests to your API. This completely disables CORS protection and can expose your  
application to security vulnerabilities including data theft and CSRF attacks.  


## Allow Specific Origins

```python
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://www.example.com",
    "http://localhost:3000",
    "http://127.0.0.1:3000",
]
```

This is the recommended approach for production environments. It explicitly lists  
which origins are permitted to make cross-origin requests to your API. Each origin  
must include the full protocol and domain, and optionally the port if it's not the  
default for that protocol. The middleware will check the incoming request's  
`Origin` header against this list and only add the `Access-Control-Allow-Origin`  
header if there's a match. This provides fine-grained control over which domains  
can access your resources while maintaining security.  


## Allow Origins with Regex

```python
import re

CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.example\.com$",
    r"^http://localhost:\d+$",
]
```

Regular expressions provide a flexible way to allow multiple related origins  
without listing each one individually. This is particularly useful when you have  
multiple subdomains or development environments with different ports. The first  
regex allows any subdomain of `example.com` over HTTPS, while the second allows  
localhost with any port number. The middleware compiles these patterns and tests  
each incoming `Origin` header against them. Be careful with regex patterns - make  
them as specific as possible to avoid accidentally allowing unintended origins.  


## Allow Credentials

```python
CORS_ALLOW_CREDENTIALS = True

CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "http://localhost:3000",
]
```

When working with authenticated requests that include cookies, session data, or  
authorization headers, you must explicitly enable credentials support. This adds  
the `Access-Control-Allow-Credentials: true` header to responses, telling the  
browser it's safe to include credentials in the cross-origin request. Note that  
when credentials are enabled, you cannot use `CORS_ALLOW_ALL_ORIGINS = True` -  
you must specify explicit origins. This is a security measure to prevent any  
website from making authenticated requests to your API.  


## Allow Specific HTTP Methods

```python
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]
```

This setting controls which HTTP methods are allowed in cross-origin requests.  
The middleware adds these methods to the `Access-Control-Allow-Methods` header in  
preflight responses. By default, django-cors-headers allows the methods shown  
above. You can restrict this list if your API only supports certain operations.  
For example, if your API is read-only, you might only allow `GET` and `OPTIONS`.  
The `OPTIONS` method should always be included as it's used for preflight requests.  


## Allow Custom Headers

```python
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
    'x-custom-header',
]
```

Many modern APIs use custom headers for various purposes like authentication  
tokens, API versioning, or request tracking. The `CORS_ALLOW_HEADERS` setting  
specifies which request headers are allowed in cross-origin requests. This list is  
sent to the browser in the `Access-Control-Allow-Headers` response header during  
preflight requests. The default list includes common headers, but you can add  
custom ones as needed. If a client attempts to send a header that's not in this  
list, the browser will block the request.  


## Expose Headers to Frontend

```python
CORS_EXPOSE_HEADERS = [
    'Content-Type',
    'X-Total-Count',
    'X-Page-Number',
    'X-Rate-Limit',
]
```

By default, browsers only expose a limited set of response headers to JavaScript  
code making cross-origin requests. If your API returns custom headers that the  
frontend needs to access (like pagination info or rate limit data), you must list  
them in `CORS_EXPOSE_HEADERS`. This adds the `Access-Control-Expose-Headers`  
header to responses, telling the browser which headers are safe to expose to the  
requesting JavaScript code. Without this setting, custom response headers will be  
invisible to your frontend application even though they're sent by the server.  


## Preflight Request Caching

```python
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours in seconds
```

Preflight requests add latency to cross-origin API calls because the browser must  
make an OPTIONS request before the actual request. The `CORS_PREFLIGHT_MAX_AGE`  
setting tells browsers how long they can cache the results of a preflight request,  
reducing the number of OPTIONS requests. The value is specified in seconds, and  
the default is typically around 24 hours. Longer cache times improve performance  
but mean that changes to your CORS configuration will take longer to take effect  
on client browsers. Balance performance needs with configuration flexibility when  
setting this value.  


## Custom CORS Middleware

```python
class CustomCorsMiddleware:
    """
    Custom middleware for handling CORS with additional business logic.
    Useful when you need more control than django-cors-headers provides.
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.allowed_origins = [
            'https://example.com',
            'http://localhost:3000',
        ]
    
    def __call__(self, request):
        origin = request.META.get('HTTP_ORIGIN')
        
        # Process request
        response = self.get_response(request)
        
        # Add CORS headers if origin is allowed
        if origin in self.allowed_origins:
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        
        return response
```

Sometimes you need more control over CORS handling than the standard package  
provides, such as dynamically determining allowed origins based on database  
configuration or implementing custom logging. This middleware class demonstrates  
the basic pattern for handling CORS manually. It checks the incoming request's  
origin against an allowed list and adds the appropriate headers to the response.  
This approach gives you complete control but requires careful implementation to  
handle all CORS edge cases correctly, including preflight requests.  


## Handling Preflight Requests

```python
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods

@csrf_exempt
@require_http_methods(["OPTIONS"])
def cors_preflight(request):
    """
    Handle preflight OPTIONS requests explicitly.
    This view responds to preflight checks for complex CORS requests.
    """
    response = HttpResponse()
    response['Access-Control-Allow-Origin'] = 'https://example.com'
    response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
    response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization, X-Custom-Header'
    response['Access-Control-Max-Age'] = '86400'
    return response
```

When browsers make "complex" requests (those with custom headers, methods other  
than GET/POST, or non-simple content types), they first send an OPTIONS preflight  
request to check if the actual request is allowed. This view handles such  
preflight requests manually, returning the appropriate CORS headers without  
processing any business logic. The `@csrf_exempt` decorator is necessary because  
OPTIONS requests typically don't include CSRF tokens. The `Access-Control-Max-Age`  
header tells the browser how long to cache this preflight response.  


## View-Level CORS with Decorator

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

def cors_headers_decorator(allowed_origins=None):
    """
    Decorator to add CORS headers to specific views.
    Provides fine-grained control over CORS on a per-view basis.
    """
    if allowed_origins is None:
        allowed_origins = ['*']
    
    def decorator(view_func):
        def wrapped_view(request, *args, **kwargs):
            response = view_func(request, *args, **kwargs)
            
            origin = request.META.get('HTTP_ORIGIN')
            if '*' in allowed_origins or origin in allowed_origins:
                response['Access-Control-Allow-Origin'] = origin if origin else '*'
                response['Access-Control-Allow-Methods'] = 'GET, POST, OPTIONS'
                response['Access-Control-Allow-Headers'] = 'Content-Type'
            
            return response
        return wrapped_view
    return decorator

@csrf_exempt
@cors_headers_decorator(allowed_origins=['https://example.com', 'http://localhost:3000'])
def api_endpoint(request):
    """API endpoint with CORS enabled only for specific origins."""
    data = {'message': 'Hello from CORS-enabled endpoint'}
    return JsonResponse(data)
```

This decorator pattern allows you to enable CORS on specific views rather than  
globally through middleware. It's useful when only certain endpoints need to be  
accessible cross-origin, or when different endpoints need different CORS policies.  
The decorator checks the request origin against the allowed list and adds CORS  
headers accordingly. This approach provides more granular control but requires  
applying the decorator to each view that needs CORS support, which can be more  
maintenance overhead than using global middleware.  


## CORS with Django REST Framework

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    # ... other middleware
]

CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://example.com",
]

# views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def api_data(request):
    """
    DRF view that works seamlessly with django-cors-headers.
    The middleware handles CORS automatically for all DRF responses.
    """
    if request.method == 'GET':
        data = {'users': ['Alice', 'Bob', 'Charlie']}
        return Response(data)
    elif request.method == 'POST':
        return Response({'status': 'created'}, status=201)
```

Django REST Framework works perfectly with django-cors-headers middleware. The  
middleware intercepts all responses, including those from DRF views, and adds the  
appropriate CORS headers automatically. You don't need any special configuration  
or decorators on your DRF views - the middleware handles everything. This makes it  
easy to build CORS-enabled APIs with DRF. The `@api_view` decorator handles  
different HTTP methods, and the middleware ensures that all responses include the  
correct CORS headers based on your global settings.  


## CORS with Cookies and Sessions

```python
# settings.py
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
]

# For Django 4.0+, you may also need:
CSRF_TRUSTED_ORIGINS = [
    "http://localhost:3000",
]

SESSION_COOKIE_SAMESITE = 'None'
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SAMESITE = 'None'
CSRF_COOKIE_SECURE = True

# views.py
from django.http import JsonResponse
from django.views.decorators.csrf import ensure_csrf_cookie

@ensure_csrf_cookie
def get_csrf_token(request):
    """
    View that sets the CSRF cookie for cross-origin requests.
    Frontend can call this to get a CSRF token before making POST requests.
    """
    return JsonResponse({'detail': 'CSRF cookie set'})

def protected_endpoint(request):
    """
    Protected endpoint that requires CSRF token and session cookie.
    Works with cross-origin requests when properly configured.
    """
    if request.method == 'POST':
        # Process authenticated request
        return JsonResponse({'status': 'success'})
    return JsonResponse({'detail': 'Use POST method'})
```

Working with cookies and sessions in cross-origin contexts requires careful  
configuration. The `CORS_ALLOW_CREDENTIALS` setting allows cookies to be sent  
with cross-origin requests. The `SESSION_COOKIE_SAMESITE = 'None'` and  
`SESSION_COOKIE_SECURE = True` settings are required for cookies to work in  
cross-origin contexts in modern browsers. The CSRF cookie needs similar  
configuration. The first view provides a way for frontends to obtain a CSRF token,  
which they can then include in subsequent POST requests. This pattern is common  
when building SPAs that need to authenticate with a Django backend.  


## Dynamic CORS Origins from Database

```python
# models.py
from django.db import models

class AllowedOrigin(models.Model):
    """Model to store allowed CORS origins dynamically."""
    origin = models.URLField(unique=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.origin

# middleware.py
from .models import AllowedOrigin

class DynamicCorsMiddleware:
    """
    Middleware that loads allowed origins from database.
    Useful for SaaS applications where origins are configured per tenant.
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
        self._cache_timeout = 300  # Cache for 5 minutes
        self._last_update = 0
        self._allowed_origins = set()
    
    def _update_origins(self):
        """Load allowed origins from database with caching."""
        import time
        current_time = time.time()
        
        if current_time - self._last_update > self._cache_timeout:
            self._allowed_origins = set(
                AllowedOrigin.objects.filter(is_active=True)
                .values_list('origin', flat=True)
            )
            self._last_update = current_time
    
    def __call__(self, request):
        self._update_origins()
        origin = request.META.get('HTTP_ORIGIN')
        
        response = self.get_response(request)
        
        if origin in self._allowed_origins:
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        
        return response
```

For multi-tenant applications or platforms where users can register their own  
domains, storing allowed origins in the database provides flexibility. This  
middleware loads allowed origins from the database and caches them to avoid  
hitting the database on every request. The cache is refreshed every 5 minutes,  
balancing performance with the ability to update origins dynamically. This pattern  
is particularly useful for SaaS platforms where customers need to whitelist their  
own domains without requiring code deployment or server restart.  


## CORS with File Uploads

```python
# settings.py
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'content-disposition',  # Important for file downloads
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]

# views.py
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.core.files.storage import default_storage
from django.conf import settings

@csrf_exempt
def upload_file(request):
    """
    Handle file uploads from cross-origin requests.
    Supports multipart/form-data content type.
    """
    if request.method == 'POST' and request.FILES.get('file'):
        uploaded_file = request.FILES['file']
        
        # Save file
        filename = default_storage.save(
            f'uploads/{uploaded_file.name}',
            uploaded_file
        )
        
        return JsonResponse({
            'success': True,
            'filename': uploaded_file.name,
            'size': uploaded_file.size,
            'url': default_storage.url(filename)
        })
    
    return JsonResponse({'error': 'No file provided'}, status=400)

def download_file(request, filename):
    """
    Serve files for cross-origin download.
    Includes proper headers for file downloads.
    """
    from django.http import FileResponse
    import os
    
    file_path = os.path.join(settings.MEDIA_ROOT, 'uploads', filename)
    
    if os.path.exists(file_path):
        response = FileResponse(open(file_path, 'rb'))
        response['Content-Disposition'] = f'attachment; filename="{filename}"'
        response['Access-Control-Expose-Headers'] = 'Content-Disposition'
        return response
    
    return JsonResponse({'error': 'File not found'}, status=404)
```

File uploads and downloads in cross-origin contexts require special attention to  
headers and content types. For uploads, the `content-type` header must be allowed  
to support `multipart/form-data` requests. The upload view handles file storage  
and returns information about the uploaded file. For downloads, the  
`Content-Disposition` header is used to specify the filename, and this header must  
be exposed through `Access-Control-Expose-Headers` so that JavaScript can read it.  
The `@csrf_exempt` decorator is used here but in production, you should implement  
proper authentication and CSRF protection.  


## CORS Error Handling

```python
from django.http import JsonResponse
from django.core.exceptions import PermissionDenied

class CorsErrorMiddleware:
    """
    Middleware to handle CORS-related errors and provide clear responses.
    Helps debug CORS issues by adding informative error messages.
    """
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.allowed_origins = [
            'https://example.com',
            'http://localhost:3000',
        ]
    
    def __call__(self, request):
        origin = request.META.get('HTTP_ORIGIN')
        
        # Check if origin is allowed
        if origin and origin not in self.allowed_origins:
            if request.method == 'OPTIONS':
                # Preflight request from disallowed origin
                return JsonResponse({
                    'error': 'CORS preflight failed',
                    'detail': f'Origin {origin} is not allowed',
                    'allowed_origins': self.allowed_origins
                }, status=403)
        
        response = self.get_response(request)
        
        # Add CORS headers for allowed origins
        if origin in self.allowed_origins:
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
            response['Access-Control-Max-Age'] = '3600'
        
        return response
    
    def process_exception(self, request, exception):
        """Handle exceptions and ensure CORS headers are still added."""
        origin = request.META.get('HTTP_ORIGIN')
        
        if isinstance(exception, PermissionDenied):
            response = JsonResponse({
                'error': 'Permission denied',
                'detail': str(exception)
            }, status=403)
        else:
            response = JsonResponse({
                'error': 'Internal server error',
                'detail': 'An error occurred processing your request'
            }, status=500)
        
        # Add CORS headers even for error responses
        if origin in self.allowed_origins:
            response['Access-Control-Allow-Origin'] = origin
        
        return response
```

Proper error handling for CORS is crucial for debugging and security. This  
middleware demonstrates how to provide clear error messages when CORS requests  
fail, which helps frontend developers understand why their requests are being  
blocked. It also ensures that CORS headers are added even to error responses,  
which is important because browsers need these headers to display error messages  
to the JavaScript code. The `process_exception` method handles exceptions and  
ensures that error responses still include CORS headers so that the frontend can  
read the error details.  


## Testing CORS Configuration

```python
from django.test import TestCase, Client
from django.urls import reverse

class CorsTestCase(TestCase):
    """
    Test cases for CORS configuration.
    Verifies that CORS headers are correctly added to responses.
    """
    
    def setUp(self):
        self.client = Client()
        self.allowed_origin = 'https://example.com'
        self.disallowed_origin = 'https://malicious.com'
    
    def test_cors_allowed_origin(self):
        """Test that allowed origins receive proper CORS headers."""
        response = self.client.get(
            '/api/data/',
            HTTP_ORIGIN=self.allowed_origin
        )
        
        self.assertEqual(
            response['Access-Control-Allow-Origin'],
            self.allowed_origin
        )
    
    def test_cors_disallowed_origin(self):
        """Test that disallowed origins don't receive CORS headers."""
        response = self.client.get(
            '/api/data/',
            HTTP_ORIGIN=self.disallowed_origin
        )
        
        self.assertNotIn('Access-Control-Allow-Origin', response)
    
    def test_cors_preflight_request(self):
        """Test preflight OPTIONS request handling."""
        response = self.client.options(
            '/api/data/',
            HTTP_ORIGIN=self.allowed_origin,
            HTTP_ACCESS_CONTROL_REQUEST_METHOD='POST',
            HTTP_ACCESS_CONTROL_REQUEST_HEADERS='Content-Type'
        )
        
        self.assertEqual(response.status_code, 200)
        self.assertIn('Access-Control-Allow-Methods', response)
        self.assertIn('Access-Control-Allow-Headers', response)
    
    def test_cors_with_credentials(self):
        """Test that credentials header is present when configured."""
        response = self.client.get(
            '/api/data/',
            HTTP_ORIGIN=self.allowed_origin
        )
        
        self.assertEqual(
            response.get('Access-Control-Allow-Credentials'),
            'true'
        )
```

Testing CORS configuration is essential to ensure your API works correctly with  
cross-origin requests. This test class demonstrates common CORS test scenarios:  
verifying that allowed origins receive the correct headers, disallowed origins  
don't get CORS headers, preflight requests are handled properly, and credentials  
support works as expected. The `HTTP_ORIGIN` header is set in test requests to  
simulate cross-origin requests. These tests help catch configuration issues before  
deployment and ensure that changes to CORS settings don't break existing  
functionality.  


## CORS Security Best Practices

```python
# settings.py - Production Configuration
from django.core.exceptions import ImproperlyConfigured
import os

# Never use CORS_ALLOW_ALL_ORIGINS in production
if os.environ.get('ENVIRONMENT') == 'production':
    if getattr(settings, 'CORS_ALLOW_ALL_ORIGINS', False):
        raise ImproperlyConfigured(
            "CORS_ALLOW_ALL_ORIGINS must not be True in production"
        )

# Use explicit origin whitelist
CORS_ALLOWED_ORIGINS = [
    "https://www.example.com",
    "https://app.example.com",
]

# Only allow credentials with explicit origins
CORS_ALLOW_CREDENTIALS = True

# Restrict allowed methods to what your API actually supports
CORS_ALLOW_METHODS = [
    'GET',
    'POST',
    'PUT',
    'DELETE',
    'OPTIONS',
]

# Limit exposed headers to necessary ones
CORS_EXPOSE_HEADERS = [
    'Content-Type',
    'X-Total-Count',
]

# Allow only necessary request headers
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]

# Set reasonable preflight cache time
CORS_PREFLIGHT_MAX_AGE = 3600  # 1 hour

# Additional security settings
SECURE_CROSS_ORIGIN_OPENER_POLICY = 'same-origin'
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
```

Security is paramount when configuring CORS. This configuration demonstrates  
security best practices: never allowing all origins in production, using explicit  
origin whitelists, restricting HTTP methods to only what's needed, limiting  
headers to the minimum required, and setting appropriate preflight cache times.  
The configuration also includes a safety check that raises an error if someone  
tries to use `CORS_ALLOW_ALL_ORIGINS` in production. Additional security headers  
like `SECURE_CROSS_ORIGIN_OPENER_POLICY` provide defense-in-depth. Remember that  
CORS is not a substitute for proper authentication and authorization - it only  
controls which origins can make requests, not what those requests can do.  


## Troubleshooting Common CORS Issues

Common CORS problems and their solutions:  

### Issue: "No 'Access-Control-Allow-Origin' header is present"

This occurs when the server doesn't add CORS headers to the response. Check that:  
- `corsheaders` middleware is installed and properly positioned in `MIDDLEWARE`  
- The origin is in `CORS_ALLOWED_ORIGINS` or `CORS_ALLOW_ALL_ORIGINS = True`  
- The middleware is actually running (check for typos in app names)  

### Issue: "Credential is not supported if the CORS header is '*'"

This happens when you try to use `CORS_ALLOW_ALL_ORIGINS = True` with  
`CORS_ALLOW_CREDENTIALS = True`. You must use explicit origins when allowing  
credentials. Solution: Replace `CORS_ALLOW_ALL_ORIGINS` with  
`CORS_ALLOWED_ORIGINS` list.  

### Issue: Preflight request returns 403 or 401

Preflight OPTIONS requests shouldn't require authentication. Ensure your  
authentication middleware doesn't block OPTIONS requests, or use `@csrf_exempt`  
on views that handle OPTIONS.  

### Issue: Custom headers not allowed

If your frontend sends custom headers like `X-Custom-Token`, add them to  
`CORS_ALLOW_HEADERS`. The browser checks this during preflight and will block  
the request if headers aren't explicitly allowed.  

### Issue: Response headers not accessible in JavaScript

If your API returns custom headers that JavaScript needs to read (like pagination  
info), add them to `CORS_EXPOSE_HEADERS`. Otherwise, they're hidden from  
JavaScript even though they're sent by the server.  

### Issue: Cookies not being sent

For cross-origin requests with cookies:  
- Set `CORS_ALLOW_CREDENTIALS = True`  
- Use explicit origins, not wildcard  
- Set `SESSION_COOKIE_SAMESITE = 'None'` and `SESSION_COOKIE_SECURE = True`  
- Frontend must use `credentials: 'include'` in fetch options  

### Issue: CORS works in development but not production

Development often uses `localhost` which has relaxed security. In production:  
- Ensure HTTPS is used for both frontend and backend  
- Update `CORS_ALLOWED_ORIGINS` with production domains  
- Check that `CSRF_TRUSTED_ORIGINS` includes your frontend domain  
- Verify SSL certificates are valid
