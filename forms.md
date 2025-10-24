# Forms

Django forms provide a powerful framework for handling HTML forms, data validation,  
and user input processing. They serve as a bridge between HTML form elements and  
Python code, offering a clean and efficient way to collect, validate, and process  
user data. Forms in Django handle everything from rendering HTML form fields to  
validating submitted data and converting it into Python data types. The framework  
includes both regular forms for arbitrary data collection and ModelForms that are  
tightly integrated with Django models for database operations. Django's form system  
automatically handles CSRF protection, generates appropriate HTML widgets, manages  
form state, and provides detailed error messages when validation fails. This makes  
forms one of the most essential components in any Django web application, whether  
you're building simple contact forms or complex multi-step data entry interfaces.

## Plain form


In `forms.py`:

```python
from django import forms

class MessageForm(forms.Form):

    text = forms.CharField(max_length=255)
```

In `views.py`

```python
from django.shortcuts import render
from django.http import HttpRequest, HttpResponse

from . import forms


def home(req: HttpRequest):

    if req.method == 'GET':

        mform = forms.MessageForm()

        return render(req, 'index.html', {'form': mform})

    if req.method == 'POST':

        mform = forms.MessageForm(req.POST)

        if mform.is_valid():

            text = mform.cleaned_data['text']
            msg = f'you entered: {text}'
            return HttpResponse(msg)
```

The `index.html`:

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>

<body>

    <h2>Message form</h2>

    <form method="post">
        {{ form.as_p }}
        {% csrf_token %}
        <input type="submit" value="Submit">
    </form>


</body>

</html>
```


## Validate with is_valid

We have a custom widget for text attribute.  

```python
from django import forms

class MessageForm(forms.Form):

    title = forms.CharField(max_length=50, required=True)
    text = forms.CharField(widget=forms.Textarea, required=True)
```

We include `field.errors` for displaying validation errors.  

```python
from django.shortcuts import render
from django.http import HttpRequest, HttpResponse 

from . import forms


def home(req: HttpRequest):

    if req.method == 'GET':

        mform = forms.MessageForm()

        return render(req, 'index.html', {'form': mform})

    if req.method == 'POST':

        mform = forms.MessageForm(req.POST)

        if mform.is_valid():

            title = mform.cleaned_data['title']
            text = mform.cleaned_data['text']
            msg = f'you entered: {title} and {text}'

            return HttpResponse(msg)

        else:

            return render(req, 'index.html', {'form': mform})
```



```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>

<body>

    <h2>Message form</h2>


    <form method="post">
        {{ field.errors }}

        {{ form.as_p }}
        {% csrf_token %}
        <input type="submit" value="Submit">
    </form>


</body>

</html>
```

## Form field types

Django provides various field types for different data inputs. Each field type has  
built-in validation and handles data conversion automatically. Common field types  
include CharField for text, EmailField for email addresses, IntegerField for  
numbers, and BooleanField for checkboxes. Each field can be customized with  
options like required, initial values, help text, and custom validators.

```python
from django import forms

class RegistrationForm(forms.Form):

    username = forms.CharField(
        max_length=30,
        required=True,
        help_text='Enter your username'
    )
    email = forms.EmailField(
        required=True,
        help_text='Enter a valid email address'
    )
    age = forms.IntegerField(
        min_value=18,
        max_value=100,
        required=False
    )
    password = forms.CharField(
        widget=forms.PasswordInput,
        min_length=8
    )
    confirm_password = forms.CharField(
        widget=forms.PasswordInput
    )
    agree_terms = forms.BooleanField(
        required=True,
        help_text='You must agree to the terms'
    )
```

## Choice fields

ChoiceField and MultipleChoiceField allow users to select from predefined options.  
Choices are provided as tuples containing a value and display text. These fields  
render as select dropdowns by default, but can use radio buttons or checkboxes  
with appropriate widgets. The choices can be static or dynamically generated.

```python
from django import forms

class PreferencesForm(forms.Form):

    COLORS = [
        ('red', 'Red'),
        ('blue', 'Blue'),
        ('green', 'Green'),
    ]

    LANGUAGES = [
        ('python', 'Python'),
        ('javascript', 'JavaScript'),
        ('java', 'Java'),
        ('go', 'Go'),
    ]

    favorite_color = forms.ChoiceField(
        choices=COLORS,
        widget=forms.RadioSelect
    )
    programming_languages = forms.MultipleChoiceField(
        choices=LANGUAGES,
        widget=forms.CheckboxSelectMultiple,
        required=False
    )
```

## Date and time fields

Django provides specialized fields for handling dates and times with built-in  
validation. DateField, TimeField, and DateTimeField parse various date formats  
and can use custom widgets like DateInput or specialized date pickers. These  
fields automatically convert strings to Python datetime objects.

```python
from django import forms

class EventForm(forms.Form):

    event_name = forms.CharField(max_length=100)
    event_date = forms.DateField(
        widget=forms.DateInput(attrs={'type': 'date'}),
        help_text='Select event date'
    )
    event_time = forms.TimeField(
        widget=forms.TimeInput(attrs={'type': 'time'}),
        help_text='Select event time'
    )
    registration_deadline = forms.DateTimeField(
        widget=forms.DateTimeInput(attrs={'type': 'datetime-local'}),
        required=False
    )
```

## Custom validation

Forms support custom validation through clean methods. The clean method for a  
specific field is named clean_fieldname and validates that single field. The  
general clean method validates multiple fields together and can check cross-field  
constraints. Both methods should raise ValidationError with appropriate messages.

```python
from django import forms
from django.core.exceptions import ValidationError

class SignupForm(forms.Form):

    username = forms.CharField(max_length=30)
    email = forms.EmailField()
    password = forms.CharField(widget=forms.PasswordInput)
    confirm_password = forms.CharField(widget=forms.PasswordInput)

    def clean_username(self):
        username = self.cleaned_data.get('username')
        
        if len(username) < 3:
            raise ValidationError('Username must be at least 3 characters')
        
        if not username.isalnum():
            raise ValidationError('Username must contain only letters and numbers')
        
        return username

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        confirm_password = cleaned_data.get('confirm_password')

        if password and confirm_password and password != confirm_password:
            raise ValidationError('Passwords do not match')

        return cleaned_data
```

## ModelForm basics

ModelForm automatically creates a form from a Django model, generating fields  
based on the model's field definitions. This eliminates duplicate code and ensures  
consistency between your database schema and forms. The Meta class specifies which  
model to use and which fields to include or exclude from the form.

```python
from django.db import models

class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    is_published = models.BooleanField(default=False)
```

```python
from django import forms
from .models import Article

class ArticleForm(forms.ModelForm):

    class Meta:
        model = Article
        fields = ['title', 'content', 'author', 'published_date', 'is_published']
        # Or use: fields = '__all__'
        # Or exclude: exclude = ['is_published']
```

## ModelForm with custom widgets

ModelForm allows customization of widgets for each field to control how they  
render in HTML. The widgets dictionary in the Meta class maps field names to  
widget instances. You can also add custom attributes like CSS classes and  
placeholders to enhance the form's appearance and usability.

```python
from django import forms
from .models import Article

class ArticleForm(forms.ModelForm):

    class Meta:
        model = Article
        fields = '__all__'
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter article title'
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 10,
                'placeholder': 'Write your article content here'
            }),
            'author': forms.TextInput(attrs={
                'class': 'form-control'
            }),
            'published_date': forms.DateInput(attrs={
                'type': 'date',
                'class': 'form-control'
            }),
        }
        labels = {
            'is_published': 'Publish immediately',
        }
        help_texts = {
            'title': 'Maximum 200 characters',
        }
```

## Saving ModelForm

ModelForm instances can be saved directly to the database using the save method.  
The save method creates or updates a model instance based on the form data. Using  
commit=False returns an unsaved model instance, allowing you to modify it before  
saving. This is useful for setting additional fields or performing extra logic.

```python
from django.shortcuts import render, redirect
from .forms import ArticleForm

def create_article(request):

    if request.method == 'POST':
        form = ArticleForm(request.POST)
        
        if form.is_valid():
            # Save directly
            article = form.save()
            return redirect('article_detail', pk=article.pk)
    
    else:
        form = ArticleForm()

    return render(request, 'create_article.html', {'form': form})

def create_article_with_modifications(request):

    if request.method == 'POST':
        form = ArticleForm(request.POST)
        
        if form.is_valid():
            # Save with commit=False to modify before saving
            article = form.save(commit=False)
            article.author = request.user.username
            article.save()
            return redirect('article_detail', pk=article.pk)
    
    else:
        form = ArticleForm()

    return render(request, 'create_article.html', {'form': form})
```

## Updating with ModelForm

ModelForm can be used for updating existing model instances by passing the  
instance parameter. The form will be pre-populated with the current values from  
the database. When saved, it updates the existing record instead of creating a  
new one. This pattern is essential for edit views in CRUD applications.

```python
from django.shortcuts import render, redirect, get_object_or_404
from .models import Article
from .forms import ArticleForm

def edit_article(request, pk):

    article = get_object_or_404(Article, pk=pk)

    if request.method == 'POST':
        form = ArticleForm(request.POST, instance=article)
        
        if form.is_valid():
            form.save()
            return redirect('article_detail', pk=article.pk)
    
    else:
        form = ArticleForm(instance=article)

    context = {
        'form': form,
        'article': article
    }
    
    return render(request, 'edit_article.html', context)
```

## Form rendering options

Django provides multiple ways to render forms in templates. The as_p method wraps  
each field in paragraph tags, as_table renders in table rows, and as_ul creates  
an unordered list. For complete control, iterate over form fields manually and  
render each with its label, widget, errors, and help text individually.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Form Rendering</title>
</head>
<body>

    <h2>Render as paragraphs</h2>
    <form method="post">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Submit</button>
    </form>

    <h2>Render as table</h2>
    <form method="post">
        {% csrf_token %}
        <table>
            {{ form.as_table }}
        </table>
        <button type="submit">Submit</button>
    </form>

    <h2>Manual field rendering</h2>
    <form method="post">
        {% csrf_token %}
        {% for field in form %}
            <div class="field-wrapper">
                {{ field.label_tag }}
                {{ field }}
                {% if field.help_text %}
                    <small>{{ field.help_text }}</small>
                {% endif %}
                {% if field.errors %}
                    <div class="errors">{{ field.errors }}</div>
                {% endif %}
            </div>
        {% endfor %}
        <button type="submit">Submit</button>
    </form>

</body>
</html>
```

## File upload forms

FileField and ImageField handle file uploads in Django forms. The form must use  
enctype multipart in the HTML form tag. In the view, pass both request.POST and  
request.FILES to the form constructor. The uploaded file is accessible through  
cleaned_data and can be saved to the model or processed as needed.

```python
from django.db import models

class Document(models.Model):
    title = models.CharField(max_length=100)
    file = models.FileField(upload_to='documents/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

```python
from django import forms
from .models import Document

class DocumentForm(forms.ModelForm):

    class Meta:
        model = Document
        fields = ['title', 'file']
```

```python
from django.shortcuts import render, redirect
from .forms import DocumentForm

def upload_document(request):

    if request.method == 'POST':
        form = DocumentForm(request.POST, request.FILES)
        
        if form.is_valid():
            form.save()
            return redirect('upload_success')
    
    else:
        form = DocumentForm()

    return render(request, 'upload.html', {'form': form})
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Upload Document</title>
</head>
<body>

    <h2>Upload Document</h2>

    <form method="post" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Upload</button>
    </form>

</body>
</html>
```

## Form with initial data

Forms can be initialized with default values using the initial parameter. This is  
useful for pre-filling forms with data from the user's profile, previous  
submissions, or application state. Initial data can be set per field or for the  
entire form. This data appears in the form but can be modified by the user.

```python
from django.shortcuts import render
from django import forms

class ProfileForm(forms.Form):

    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    bio = forms.CharField(widget=forms.Textarea)
    receive_newsletter = forms.BooleanField(required=False)

def edit_profile(request):

    if request.method == 'POST':
        form = ProfileForm(request.POST)
        
        if form.is_valid():
            # Process form data
            pass
    
    else:
        # Pre-fill form with initial data
        initial_data = {
            'name': request.user.get_full_name(),
            'email': request.user.email,
            'bio': 'Tell us about yourself',
            'receive_newsletter': True
        }
        form = ProfileForm(initial=initial_data)

    return render(request, 'edit_profile.html', {'form': form})
```

## Form with custom field validation

Individual field validators can be added using the validators parameter. Django  
provides built-in validators like MinLengthValidator, MaxLengthValidator, and  
EmailValidator. You can also create custom validators as functions that raise  
ValidationError. Multiple validators can be chained for comprehensive validation.

```python
from django import forms
from django.core.validators import MinLengthValidator, EmailValidator
from django.core.exceptions import ValidationError

def validate_no_special_chars(value):
    if not value.isalnum():
        raise ValidationError('Only letters and numbers are allowed')

def validate_not_blacklisted(value):
    blacklist = ['admin', 'root', 'superuser']
    if value.lower() in blacklist:
        raise ValidationError(f'{value} is not allowed as username')

class AccountForm(forms.Form):

    username = forms.CharField(
        max_length=30,
        validators=[
            MinLengthValidator(3, 'Username must be at least 3 characters'),
            validate_no_special_chars,
            validate_not_blacklisted
        ]
    )
    email = forms.EmailField(
        validators=[EmailValidator('Enter a valid email address')]
    )
    website = forms.URLField(required=False)
```

## Hidden fields and disabled fields

Forms can include hidden fields that are not displayed to users but are submitted  
with the form. The HiddenInput widget renders an input with type hidden. Fields  
can also be disabled to prevent user modification while still displaying the  
value. Disabled fields are not submitted, so use readonly or hidden alternatives.

```python
from django import forms

class OrderForm(forms.Form):

    # Hidden field for order ID
    order_id = forms.CharField(
        widget=forms.HiddenInput()
    )
    
    # Display order number but prevent editing
    order_number = forms.CharField(
        max_length=20,
        disabled=True,
        help_text='Order number cannot be changed'
    )
    
    # Regular fields
    shipping_address = forms.CharField(
        widget=forms.Textarea,
        max_length=500
    )
    
    # Hidden timestamp
    submitted_at = forms.DateTimeField(
        widget=forms.HiddenInput()
    )
```

```python
from django.shortcuts import render
from django.utils import timezone
from .forms import OrderForm

def process_order(request):

    if request.method == 'POST':
        form = OrderForm(request.POST)
        
        if form.is_valid():
            order_id = form.cleaned_data['order_id']
            address = form.cleaned_data['shipping_address']
            # Process order
    
    else:
        form = OrderForm(initial={
            'order_id': '12345',
            'order_number': 'ORD-2024-001',
            'submitted_at': timezone.now()
        })

    return render(request, 'order.html', {'form': form})
```

## Formsets for multiple forms

Formsets manage multiple instances of the same form on a single page. They're  
useful for handling related records like order items or tasks. Django provides  
formset_factory to create formset classes from regular forms, and  
modelformset_factory for model-based formsets. Formsets handle validation and  
can limit the number of forms with extra and max_num parameters.

```python
from django import forms

class TaskForm(forms.Form):

    task_name = forms.CharField(max_length=100)
    priority = forms.ChoiceField(choices=[
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High')
    ])
    completed = forms.BooleanField(required=False)
```

```python
from django.shortcuts import render
from django.forms import formset_factory
from .forms import TaskForm

def manage_tasks(request):

    TaskFormSet = formset_factory(TaskForm, extra=3, max_num=10)

    if request.method == 'POST':
        formset = TaskFormSet(request.POST)
        
        if formset.is_valid():
            for form in formset:
                if form.cleaned_data:
                    task_name = form.cleaned_data['task_name']
                    priority = form.cleaned_data['priority']
                    # Save task to database
    
    else:
        formset = TaskFormSet()

    return render(request, 'tasks.html', {'formset': formset})
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Manage Tasks</title>
</head>
<body>

    <h2>Task List</h2>

    <form method="post">
        {% csrf_token %}
        {{ formset.management_form }}
        
        {% for form in formset %}
            <div class="task-form">
                {{ form.as_p }}
            </div>
        {% endfor %}
        
        <button type="submit">Save Tasks</button>
    </form>

</body>
</html>
```

## Inline formsets with models

Inline formsets manage related objects through foreign key relationships. The  
inlineformset_factory creates formsets for child models related to a parent  
model. This is perfect for one-to-many relationships like an order with multiple  
items. The formset handles creating, updating, and deleting related objects.

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()

class Book(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    isbn = models.CharField(max_length=13)
    publication_year = models.IntegerField()
```

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.forms import inlineformset_factory
from .models import Author, Book

def edit_author_books(request, author_id):

    author = get_object_or_404(Author, pk=author_id)
    
    BookFormSet = inlineformset_factory(
        Author, 
        Book, 
        fields=['title', 'isbn', 'publication_year'],
        extra=2,
        can_delete=True
    )

    if request.method == 'POST':
        formset = BookFormSet(request.POST, instance=author)
        
        if formset.is_valid():
            formset.save()
            return redirect('author_detail', author_id=author.id)
    
    else:
        formset = BookFormSet(instance=author)

    context = {
        'author': author,
        'formset': formset
    }
    
    return render(request, 'edit_books.html', context)
```

## Dynamic choice fields

ChoiceField choices can be dynamically populated in the form's __init__ method.  
This allows loading choices from the database or filtering based on the current  
user or request context. Override __init__ to accept additional parameters and  
modify field properties before the form is rendered. This is essential for  
creating context-aware forms.

```python
from django import forms
from .models import Category, Tag

class ArticleForm(forms.Form):

    title = forms.CharField(max_length=200)
    content = forms.TextField()
    category = forms.ChoiceField()
    tags = forms.MultipleChoiceField(required=False)

    def __init__(self, *args, user=None, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Load categories from database
        categories = Category.objects.filter(active=True)
        self.fields['category'].choices = [
            (cat.id, cat.name) for cat in categories
        ]
        
        # Load tags based on user access
        if user and user.is_staff:
            tags = Tag.objects.all()
        else:
            tags = Tag.objects.filter(public=True)
        
        self.fields['tags'].choices = [
            (tag.id, tag.name) for tag in tags
        ]
        
        # Add custom widget attributes
        self.fields['category'].widget.attrs.update({
            'class': 'form-control'
        })
```

```python
from django.shortcuts import render
from .forms import ArticleForm

def create_article(request):

    if request.method == 'POST':
        form = ArticleForm(request.POST, user=request.user)
        
        if form.is_valid():
            # Process form
            pass
    
    else:
        form = ArticleForm(user=request.user)

    return render(request, 'create_article.html', {'form': form})
```

## Custom form widgets

Custom widgets control how form fields are rendered in HTML. By subclassing  
existing widgets or creating new ones, you can generate custom HTML, add  
JavaScript behavior, or integrate third-party UI components. Widgets define a  
render method that returns HTML and can include custom CSS and JavaScript.

```python
from django import forms
from django.forms.widgets import Widget
from django.utils.safestring import mark_safe

class ColorPickerWidget(Widget):
    
    template_name = 'widgets/color_picker.html'
    
    def render(self, name, value, attrs=None, renderer=None):
        if value is None:
            value = '#000000'
        
        html = f'''
        <input type="color" name="{name}" value="{value}" 
               id="id_{name}" class="color-picker">
        <input type="text" value="{value}" 
               id="id_{name}_text" class="color-text">
        '''
        return mark_safe(html)

class RatingWidget(Widget):
    
    def render(self, name, value, attrs=None, renderer=None):
        if value is None:
            value = 0
        
        stars = ''
        for i in range(1, 6):
            checked = 'checked' if i == int(value) else ''
            stars += f'''
            <input type="radio" name="{name}" value="{i}" 
                   id="star{i}" {checked}>
            <label for="star{i}">★</label>
            '''
        
        html = f'<div class="rating-widget">{stars}</div>'
        return mark_safe(html)

class ProductReviewForm(forms.Form):

    title = forms.CharField(max_length=100)
    review = forms.CharField(widget=forms.Textarea)
    rating = forms.IntegerField(
        widget=RatingWidget,
        min_value=1,
        max_value=5
    )
    favorite_color = forms.CharField(
        widget=ColorPickerWidget,
        required=False
    )
```

## Form error handling

Django forms provide comprehensive error handling with field-specific and  
non-field errors. The errors attribute contains validation errors, accessible  
in templates as form.errors or field.errors. The add_error method allows adding  
custom errors during validation. Error messages can be customized per field  
using error_messages parameter for better user experience.

```python
from django import forms
from django.core.exceptions import ValidationError

class ContactForm(forms.Form):

    name = forms.CharField(
        max_length=100,
        error_messages={
            'required': 'Please enter your name',
            'max_length': 'Name must be less than 100 characters'
        }
    )
    email = forms.EmailField(
        error_messages={
            'required': 'Email address is required',
            'invalid': 'Please enter a valid email address'
        }
    )
    phone = forms.CharField(max_length=20, required=False)
    message = forms.CharField(widget=forms.Textarea)
    captcha = forms.CharField(max_length=10)

    def clean_phone(self):
        phone = self.cleaned_data.get('phone')
        
        if phone and not phone.replace('-', '').replace(' ', '').isdigit():
            raise ValidationError('Phone number must contain only digits')
        
        return phone

    def clean_captcha(self):
        captcha = self.cleaned_data.get('captcha')
        expected_captcha = self.initial.get('captcha_answer', '')
        
        if captcha != expected_captcha:
            raise ValidationError('Incorrect captcha answer')
        
        return captcha

    def clean(self):
        cleaned_data = super().clean()
        email = cleaned_data.get('email')
        phone = cleaned_data.get('phone')

        if not email and not phone:
            self.add_error(None, 'Please provide either email or phone number')

        return cleaned_data
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Contact Form</title>
    <style>
        .error { color: red; }
        .field-error { border: 1px solid red; }
    </style>
</head>
<body>

    <h2>Contact Us</h2>

    {% if form.non_field_errors %}
        <div class="error">
            {{ form.non_field_errors }}
        </div>
    {% endif %}

    <form method="post">
        {% csrf_token %}
        
        {% for field in form %}
            <div class="form-field">
                {{ field.label_tag }}
                {{ field }}
                {% if field.errors %}
                    <div class="error">
                        {% for error in field.errors %}
                            <p>{{ error }}</p>
                        {% endfor %}
                    </div>
                {% endif %}
            </div>
        {% endfor %}
        
        <button type="submit">Send Message</button>
    </form>

</body>
</html>
```
