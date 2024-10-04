# CSRF

CSRF is a malicious attack where an attacker tricks a user into performing an unwanted  
action on a trusted site. This happens when a user is logged into a website and then  
visits a malicious website that contains a specially crafted link or form. This link  
or form, known as a CSRF token, can exploit vulnerabilities in the trusted website to  
execute unauthorized actions on the user's behalf.  

CSRF protection is automatically enabled for all POST requests in Django.  
GET requests are not protected by default. If we need to protect GET requests, we can  
use the `@csrf_protect` decorator.


## How CSRF Works in Django

Django implements robust CSRF protection mechanisms to mitigate this risk. 

1. CSRF Token Generation
   - Django automatically generates a unique, random CSRF token for each  
     user session.
   - This token is stored in the user's session data and is included in every  
     HTML form that requires user interaction.

2. **CSRF Token Validation:**
   - When a user submits a form, Django verifies that the CSRF token included in the form  
     matches the one stored in the user's session.  
   - If the tokens match, the request is considered legitimate and proceeds.  
   - If the tokens don't match, Django raises a `CsrfViewError` exception, indicating a  
     potential CSRF attack.  

## Key Points:

- CSRF protection is enabled by default in Django projects.
- The CSRF token is typically included in a hidden input field within HTML forms.  
- Django provides mechanisms to customize the CSRF token name and storage location.  
- It's essential to use Django's built-in CSRF protection features to safeguard your web  
  applications from CSRF attacks.

## Example

```html
<form action="/submit_form/" method="POST">
    {% csrf_token %}
    <input type="submit" value="Submit">
</form>
```

In this example, `{% csrf_token %}` automatically inserts the CSRF token into the form.  

## Removing protection for views

If we need to disable CSRF protection for specific views, we can use the `@csrf_exempt` decorator.  
However, exercise caution when doing so, as it can introduce vulnerabilities.  

Disabling CSRF for a view:  

```python
from django.views.decorators.csrf import csrf_exempt

path("test/", csrf_exempt(views.TestView.as_view()))
```

or  

```python
from django.views.generic import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator


@method_decorator(csrf_exempt, name='dispatch')
class TestView(View):

    def get(self, req):
        return HttpResponse('get request')

    def post(self, req):
        return HttpResponse('post request')
```

Sending POST request in the terminal using httpie tool:

`http POST localhost:8000/test/`
