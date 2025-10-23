# Middleware



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

In `settings.py`: 

```
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "redirects.middleware.RedirectMiddleware",
]
```
