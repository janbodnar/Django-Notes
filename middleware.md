# Middleware

Middleware is a framework of hooks into Django's request/response processing. It's a lightweight,  
low-level "plugin" system for globally altering Django's input or output. Each middleware component  
is responsible for doing some specific function.

## How Middleware Works

Middleware processes requests and responses in Django applications. When a request comes in:

1. **Request Phase**: Django applies middleware in the order they're defined in `MIDDLEWARE` setting
2. **View Processing**: After middleware processes the request, Django calls the appropriate view
3. **Response Phase**: Middleware is applied in reverse order to process the response
4. **Exception Handling**: If an exception occurs, middleware can catch and handle it

The flow looks like this:

```
Request → Middleware 1 → Middleware 2 → Middleware 3 → View
Response ← Middleware 1 ← Middleware 2 ← Middleware 3 ← View
```

## Middleware Order Matters

The order of middleware in `MIDDLEWARE` setting is significant because middleware can depend on  
other middleware. For example:

- `SessionMiddleware` must come before `AuthenticationMiddleware` because authentication relies  
  on session data
- `SecurityMiddleware` should be first to apply security-related features early
- `CommonMiddleware` should come before any middleware that needs to work with modified URLs

## Common Django Middleware

Django includes several built-in middleware components:

- **SecurityMiddleware**: Provides security enhancements like HTTPS redirects and security headers
- **SessionMiddleware**: Enables session support across requests
- **CommonMiddleware**: Handles URL rewriting, ETags, and other common features
- **CsrfViewMiddleware**: Adds protection against Cross Site Request Forgeries
- **AuthenticationMiddleware**: Associates users with requests using sessions
- **MessageMiddleware**: Enables cookie and session-based message support
- **XFrameOptionsMiddleware**: Provides clickjacking protection

## Creating Custom Middleware

Custom middleware is a class that implements specific methods. The simplest middleware needs  
`__init__` and `__call__` methods.

### Basic Structure

```python
class SimpleMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        # One-time configuration and initialization

    def __call__(self, request):
        # Code executed for each request before the view is called
        
        response = self.get_response(request)
        
        # Code executed for each response after the view is called
        
        return response
```

### Middleware Hooks

Middleware can implement additional hooks:

- `process_view(request, view_func, view_args, view_kwargs)`: Called before Django calls the view
- `process_exception(request, exception)`: Called when a view raises an exception
- `process_template_response(request, response)`: Called after the view finishes executing

## Example: Custom Redirect Middleware

```python
class RedirectMiddleware:
    """
    Middleware to handle redirects without using Django's built-in redirects framework.
    This middleware intercepts requests and redirects them based on custom logic.
    """

    def __init__(self, get_response):
        self.get_response = get_response
        # Define redirect mappings
        self.redirects = {
            "/old-path": "/api/hello/",
            "/legacy-url": "/api/goodbye/",
            "/redirect-test": "/middleware-redirect/",
        }

    def __call__(self, request):
        # Check if the requested path is in our redirect mappings

        path = request.path.rstrip("/")
        if path in self.redirects:

            from django.http import HttpResponseRedirect

            return HttpResponseRedirect(self.redirects[path])

        response = self.get_response(request)
        return response
```

To use this middleware, add it to `MIDDLEWARE` in `settings.py`: 

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "redirects.middleware.RedirectMiddleware",  # Custom middleware
]
```

The custom middleware is placed at the end since it doesn't depend on other middleware  
and should only process requests that haven't been handled by Django's built-in middleware.
