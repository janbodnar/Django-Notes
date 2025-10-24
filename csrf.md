# CSRF

Cross-Site Request Forgery (CSRF) is a malicious attack where an attacker tricks a user  
into performing an unwanted action on a trusted site where they are authenticated. This  
happens when a user is logged into a website and then visits a malicious website that  
contains a specially crafted link or form that submits requests to the trusted site  
without the user's knowledge or consent.

For example, if a user is logged into their bank account and visits a malicious site, that  
site could submit a request to transfer money from the user's account without their permission.

## Why CSRF Protection is Important

Without CSRF protection, attackers can:
- Transfer funds or make purchases on behalf of authenticated users
- Change user account settings (email, password)
- Post content or messages as the user
- Delete user data
- Perform any action the authenticated user is authorized to do

CSRF protection is automatically enabled for all POST, PUT, PATCH, and DELETE requests in Django.  
GET requests are not protected by default because they should be idempotent (safe and not cause  
state changes). If you need to protect GET requests, you can use the `@csrf_protect` decorator.


## How CSRF Works in Django

Django implements robust CSRF protection mechanisms to mitigate this risk. 

**1. CSRF Token Generation**
   - Django automatically generates a unique, random CSRF token for each  
     user session.
   - This token is stored in the user's session data and is included in every  
     HTML form that requires user interaction.
   - The token can also be stored in a cookie for stateless authentication.

**2. CSRF Token Validation**
   - When a user submits a form, Django verifies that the CSRF token included in the form  
     matches the one stored in the user's session or cookie.  
   - If the tokens match, the request is considered legitimate and proceeds.  
   - If the tokens don't match or are missing, Django raises a `CsrfViewError` exception,  
     indicating a potential CSRF attack.  

**3. CSRF Middleware**
   - The `CsrfViewMiddleware` is responsible for CSRF protection
   - It must be included in the `MIDDLEWARE` setting in `settings.py`
   - It processes both incoming requests and outgoing responses

## Key Points

- CSRF protection is enabled by default in Django projects.
- The CSRF token is typically included in a hidden input field within HTML forms.  
- Django provides mechanisms to customize the CSRF token name and storage location.  
- It's essential to use Django's built-in CSRF protection features to safeguard your web  
  applications from CSRF attacks.
- Never disable CSRF protection globally - only exempt specific views when absolutely necessary.

## CSRF Middleware Configuration

The `CsrfViewMiddleware` must be included in the `MIDDLEWARE` setting in `settings.py`:

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',  # CSRF middleware
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

Important settings related to CSRF:

```python
# The name of the cookie to use for the CSRF authentication token
CSRF_COOKIE_NAME = 'csrftoken'

# Whether to use a secure cookie for the CSRF cookie
CSRF_COOKIE_SECURE = True  # Use True in production with HTTPS

# Whether to use HttpOnly flag on the CSRF cookie
CSRF_COOKIE_HTTPONLY = False  # Must be False for JavaScript access

# The age of CSRF cookies, in seconds
CSRF_COOKIE_AGE = 31449600  # One year

# A list of trusted origins for unsafe requests (POST, PUT, DELETE)
CSRF_TRUSTED_ORIGINS = ['https://example.com']
```

## Basic Form Example

```html
<form action="/submit_form/" method="POST">
    {% csrf_token %}
    <input type="submit" value="Submit">
</form>
```

In this example, `{% csrf_token %}` automatically inserts the CSRF token into the form.  
The template tag renders as a hidden input field:

```html
<input type="hidden" name="csrfmiddlewaretoken" value="long-random-token-value">
```

## Example with Form Fields

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Contact Form</title>
</head>
<body>
    <h2>Contact Us</h2>
    
    <form action="/contact/" method="POST">
        {% csrf_token %}
        
        <label for="name">Name:</label>
        <input type="text" id="name" name="name" required>
        <br>
        
        <label for="email">Email:</label>
        <input type="email" id="email" name="email" required>
        <br>
        
        <label for="message">Message:</label>
        <textarea id="message" name="message" required></textarea>
        <br>
        
        <input type="submit" value="Send Message">
    </form>
</body>
</html>
```

In the corresponding view:

```python
from django.shortcuts import render
from django.http import HttpResponse

def contact(request):
    if request.method == 'POST':
        name = request.POST.get('name')
        email = request.POST.get('email')
        message = request.POST.get('message')
        
        # Process the form data
        return HttpResponse('Thank you for your message!')
    
    return render(request, 'contact.html')
```

## AJAX Requests with CSRF Token

When making AJAX requests, you need to include the CSRF token in the request headers.

### Method 1: Reading from Cookie

```javascript
// Function to get CSRF token from cookie
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Using fetch API
fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': csrftoken
    },
    body: JSON.stringify({
        data: 'example'
    })
})
.then(response => response.json())
.then(data => console.log(data));

// Using XMLHttpRequest
const xhr = new XMLHttpRequest();
xhr.open('POST', '/api/endpoint/');
xhr.setRequestHeader('X-CSRFToken', csrftoken);
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.send(JSON.stringify({ data: 'example' }));
```

### Method 2: Using jQuery

```javascript
// Set up jQuery to always send CSRF token
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

$.ajaxSetup({
    beforeSend: function(xhr, settings) {
        if (!/^(GET|HEAD|OPTIONS|TRACE)$/i.test(settings.type) && !this.crossDomain) {
            xhr.setRequestHeader("X-CSRFToken", csrftoken);
        }
    }
});

// Now all AJAX requests will include the CSRF token
$.post('/api/endpoint/', {data: 'example'}, function(response) {
    console.log(response);
});
```

### Method 3: Reading from DOM

You can also read the token from a DOM element:

```html
<script>
    const csrftoken = document.querySelector('[name=csrfmiddlewaretoken]').value;
    
    fetch('/api/endpoint/', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': csrftoken
        },
        body: JSON.stringify({
            data: 'example'
        })
    });
</script>
```

## Protecting Specific Views

If CSRF middleware is disabled globally, you can protect specific views with the `@csrf_protect` decorator:

```python
from django.views.decorators.csrf import csrf_protect
from django.shortcuts import render

@csrf_protect
def my_view(request):
    return render(request, 'template.html')
```

For class-based views:

```python
from django.utils.decorators import method_decorator
from django.views.decorators.csrf import csrf_protect
from django.views.generic import View

@method_decorator(csrf_protect, name='dispatch')
class MyView(View):
    def post(self, request):
        # Handle POST request
        pass
```

## Removing Protection for Specific Views

If you need to disable CSRF protection for specific views, you can use the `@csrf_exempt` decorator.  
However, exercise caution when doing so, as it can introduce vulnerabilities. Only exempt views that  
handle requests from trusted sources or implement their own security measures.

### Disabling CSRF for Function-Based Views

```python
from django.views.decorators.csrf import csrf_exempt
from django.http import HttpResponse

@csrf_exempt
def my_api_view(request):
    if request.method == 'POST':
        # Process the request
        return HttpResponse('Success')
    return HttpResponse('Send a POST request')
```

### Disabling CSRF in URLconf

```python
from django.views.decorators.csrf import csrf_exempt
from . import views

urlpatterns = [
    path("test/", csrf_exempt(views.TestView.as_view())),
]
```

### Disabling CSRF for Class-Based Views

Method 1: Using `@method_decorator` on the class:

```python
from django.views.generic import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from django.http import HttpResponse

@method_decorator(csrf_exempt, name='dispatch')
class TestView(View):

    def get(self, request):
        return HttpResponse('GET request')

    def post(self, request):
        return HttpResponse('POST request')
```

Method 2: Overriding the `dispatch` method:

```python
from django.views.generic import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from django.http import HttpResponse

class TestView(View):
    
    @method_decorator(csrf_exempt)
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)
    
    def get(self, request):
        return HttpResponse('GET request')
    
    def post(self, request):
        return HttpResponse('POST request')
```

### Testing with HTTPie

Sending POST request in the terminal using httpie tool:

`http POST localhost:8000/test/`

Or with cURL:

`curl -X POST http://localhost:8000/test/`

## CSRF with Django REST Framework

Django REST Framework (DRF) handles CSRF differently for API views:

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class MyAPIView(APIView):
    """
    For session authentication, CSRF protection is enforced.
    For token authentication, CSRF is not required.
    """
    
    def post(self, request):
        data = request.data
        # Process data
        return Response({'message': 'Success'}, status=status.HTTP_200_OK)
```

When using session authentication with DRF, include CSRF token:

```javascript
fetch('/api/endpoint/', {
    method: 'POST',
    credentials: 'include',  // Include cookies
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCookie('csrftoken')
    },
    body: JSON.stringify({ data: 'example' })
});
```

## CSRF Token Rotation

Django can rotate CSRF tokens to enhance security:

```python
from django.views.decorators.csrf import csrf_protect, rotate_token
from django.shortcuts import render

@csrf_protect
def login_view(request):
    if request.method == 'POST':
        # Authenticate user
        # ...
        # Rotate CSRF token after login
        rotate_token(request)
        return redirect('dashboard')
    return render(request, 'login.html')
```

## Ensuring CSRF Token in Context

To make the CSRF token available in templates, ensure the context processor is enabled:

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.csrf',  # CSRF context processor
                # ... other context processors
            ],
        },
    },
]
```

In templates, you can access the token directly:

```html
<script>
    const csrfToken = '{{ csrf_token }}';
</script>
```

## Troubleshooting Common Issues

### CSRF Verification Failed

If you encounter "CSRF verification failed" errors:

1. **Check middleware order**: Ensure `CsrfViewMiddleware` is in `MIDDLEWARE` setting
2. **Verify cookie settings**: Check `CSRF_COOKIE_SECURE`, `CSRF_COOKIE_HTTPONLY`
3. **Check trusted origins**: Add your domain to `CSRF_TRUSTED_ORIGINS` if needed
4. **Ensure token is sent**: Verify the token is included in POST requests
5. **Check HTTPS**: If `CSRF_COOKIE_SECURE = True`, ensure you're using HTTPS

### CSRF Token Missing or Incorrect

```python
# In settings.py, enable CSRF failure logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'WARNING',
            'class': 'logging.FileHandler',
            'filename': 'csrf_debug.log',
        },
    },
    'loggers': {
        'django.security.csrf': {
            'handlers': ['file'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}
```

### AJAX Requests Failing

Make sure to:
- Include the CSRF token in the `X-CSRFToken` header
- Set `CSRF_COOKIE_HTTPONLY = False` to allow JavaScript access
- Use `credentials: 'include'` in fetch requests if using cookies

### Subdomain Issues

If your application uses subdomains:

```python
# settings.py
CSRF_COOKIE_DOMAIN = '.example.com'  # Note the leading dot
CSRF_TRUSTED_ORIGINS = [
    'https://*.example.com',
]
```

## Best Practices

1. **Always use CSRF protection**: Never disable it globally
2. **Use HTTPS in production**: Set `CSRF_COOKIE_SECURE = True`
3. **Exempt only when necessary**: Only use `@csrf_exempt` for trusted endpoints
4. **Validate origin**: Use `CSRF_TRUSTED_ORIGINS` for external requests
5. **Rotate tokens on auth changes**: Use `rotate_token()` after login/logout
6. **Log CSRF failures**: Monitor and investigate CSRF failures
7. **Use proper headers**: Always include `X-CSRFToken` in AJAX requests
8. **Test thoroughly**: Test CSRF protection in your test suite

## Testing CSRF Protection

In your tests, Django provides ways to handle CSRF:

```python
from django.test import TestCase, Client

class MyViewTest(TestCase):
    
    def test_post_with_csrf(self):
        # Django test client handles CSRF automatically
        client = Client(enforce_csrf_checks=True)
        response = client.post('/my-view/', {'data': 'test'})
        self.assertEqual(response.status_code, 200)
    
    def test_post_without_csrf_fails(self):
        client = Client(enforce_csrf_checks=True)
        # This will fail with CSRF error
        response = client.post('/my-view/', {'data': 'test'}, 
                              HTTP_X_CSRFTOKEN='invalid-token')
        self.assertEqual(response.status_code, 403)
```

Using `RequestFactory`:

```python
from django.test import RequestFactory
from django.middleware.csrf import get_token

class MyViewTest(TestCase):
    
    def setUp(self):
        self.factory = RequestFactory()
    
    def test_my_view(self):
        request = self.factory.post('/my-view/', {'data': 'test'})
        # Add CSRF token to request
        get_token(request)
        response = my_view(request)
        self.assertEqual(response.status_code, 200)
```

## Custom CSRF Failure View

You can customize the view shown when CSRF validation fails:

```python
# views.py
from django.shortcuts import render

def csrf_failure(request, reason=""):
    return render(request, 'csrf_failure.html', {'reason': reason}, status=403)
```

```python
# settings.py
CSRF_FAILURE_VIEW = 'myapp.views.csrf_failure'
```

Template `csrf_failure.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>CSRF Verification Failed</title>
</head>
<body>
    <h1>CSRF Verification Failed</h1>
    <p>The CSRF token was missing or incorrect.</p>
    <p>Reason: {{ reason }}</p>
    <p><a href="/">Go back to home page</a></p>
</body>
</html>
```
