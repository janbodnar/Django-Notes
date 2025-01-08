# Form submission


The `Contact` model in `models.py`:

```python
from django.db import models

class Contact(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
    message = models.TextField()

    def __str__(self):
        return self.name
```

The `ContactForm` in `forms.py`:

```python
from django import forms
from .models import Contact

class ContactForm(forms.ModelForm):
    class Meta:
        model = Contact
        fields = ['name', 'email', 'message']
```

The views in `views.py`:

```python
from django.shortcuts import render, redirect
from .forms import ContactForm

def contact_view(request):
    if request.method == 'POST':
        form = ContactForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('success')
    else:
        form = ContactForm()
    return render(request, 'contact.html', {'form': form})

def success_view(request):
    return render(request, 'success.html')
```

The URLs in `urls.py`:

```python
from django.urls import path
from .views import contact_view, success_view

urlpatterns = [
    path('contact/', contact_view, name='contact'),
    path('success/', success_view, name='success'),
]
```

The main URLs:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),
]
```

The `contact.html` in templates: 

```html
<!DOCTYPE html>
<html>
<head>
    <title>Contact Us</title>
</head>
<body>
    <h1>Contact Us</h1>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Submit</button>
    </form>
</body>
</html>
```

The `success.html` in templates:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Success</title>
</head>
<body>
    <h1>Thank You!</h1>
    <p>Your message has been sent successfully.</p>
</body>
</html>
```

The `tests.py`:

```python
from django.test import TestCase, Client
from django.urls import reverse
from .models import Contact

class ContactFormTest(TestCase):

    def setUp(self):
        self.client = Client()

    def test_contact_form_get(self):
        response = self.client.get(reverse('contact'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'contact.html')

    def test_contact_form_post_valid(self):
        response = self.client.post(reverse('contact'), {
            'name': 'John Doe',
            'email': 'john.doe@example.com',
            'message': 'This is a test message.'
        })
        self.assertEqual(response.status_code, 302)
        self.assertRedirects(response, reverse('success'))
        self.assertEqual(Contact.objects.count(), 1)
        self.assertEqual(Contact.objects.first().name, 'John Doe')

    def test_contact_form_post_invalid(self):
        response = self.client.post(reverse('contact'), {
            'name': '',
            'email': 'invalid-email',
            'message': ''
        })
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'contact.html')
        self.assertEqual(Contact.objects.count(), 0)
```

## Expanded example 

Example tests error messages that are shown when the form receives invalid  
data. 

**Note**:

Form validation and model validation in Django essentially provide a double check to ensure data  
integrity and proper user input handling.

### Form Validation
- **Purpose**: Ensures that user input meets the expected criteria before processing or saving it.
- **When Triggered**: When `form.is_valid()` is called.
- **Error Messages**: Can be customized in the form's `Meta` class using `error_messages`.

### Model Validation
- **Purpose**: Enforces data integrity rules directly at the database level.
- **When Triggered**: When `model.full_clean()` is called, which happens implicitly during form saving  
  or can be called explicitly in code.
- **Error Messages**: Defined within the model field validators.

### Why Both Are Important
- **User Experience**: Form validation provides immediate feedback to the user about incorrect input,  
  allowing them to correct it before submission.
- **Data Integrity**: Model validation ensures that even if data bypasses the form  
  (e.g., through an API or direct database access), it still adheres to the defined constraints.  

Using both levels of validation helps catch potential issues early in the process and maintains    
a robust and reliable application. 




The `Contact` model in `models.py`:

```python
from django.db import models
from django.core.validators import EmailValidator, MinLengthValidator

class Contact(models.Model):
    name = models.CharField(max_length=100, validators=[MinLengthValidator(1, "Please enter your name.")])
    email = models.EmailField(validators=[EmailValidator("Please enter a valid email address.")])
    message = models.TextField(validators=[MinLengthValidator(1, "Please enter your message.")])

    def __str__(self):
        return self.name
```

The `ContactForm` in `forms.py`:

```python
from django import forms
from .models import Contact

class ContactForm(forms.ModelForm):
    class Meta:
        model = Contact
        fields = ['name', 'email', 'message']
        error_messages = {
            'name': {
                'required': 'Please enter your name.',
            },
            'email': {
                'required': 'Please enter your email address.',
                'invalid': 'Please enter a valid email address.',
            },
            'message': {
                'required': 'Please enter your message.',
            },
        }
```

The `contact.html` in templates:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Contact Us</title>
</head>
<body>
    <h1>Contact Us</h1>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        {% if form.errors %}
            <ul>
                {% for field in form %}
                    {% for error in field.errors %}
                        <li>{{ error }}</li>
                    {% endfor %}
                {% endfor %}
            </ul>
        {% endif %}
        <button type="submit">Submit</button>
    </form>
</body>
</html>
```

The `tests.py`:

```python
from django.test import TestCase, Client
from django.urls import reverse
from .models import Contact

class ContactFormTest(TestCase):

    def setUp(self):
        self.client = Client()

    def test_contact_form_get(self):
        response = self.client.get(reverse('contact'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'contact.html')

    def test_contact_form_post_valid(self):
        response = self.client.post(reverse('contact'), {
            'name': 'John Doe',
            'email': 'john.doe@example.com',
            'message': 'This is a test message.'
        })
        self.assertEqual(response.status_code, 302)
        self.assertRedirects(response, reverse('success'))
        self.assertEqual(Contact.objects.count(), 1)
        self.assertEqual(Contact.objects.first().name, 'John Doe')

    def test_contact_form_post_invalid(self):
        response = self.client.post(reverse('contact'), {
            'name': '',
            'email': 'invalid-email',
            'message': ''
        })
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'contact.html')
        self.assertContains(response, "Please enter your name.")
        self.assertContains(response, "Please enter a valid email address.")
        self.assertContains(response, "Please enter your message.")
        self.assertEqual(Contact.objects.count(), 0)
```
