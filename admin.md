# Admin

Django's built-in admin interface is one of the framework's most powerful features.  
It provides a ready-to-use web-based interface for managing application data without  
writing a single line of HTML or JavaScript. The admin is automatically generated  
from your model definitions and is fully customisable through Python code. It handles  
common data-management tasks such as creating, reading, updating, and deleting records,  
filtering lists, bulk actions, and search. Because it is driven entirely by your  
models, any change you make to a model is reflected in the admin immediately after  
running migrations. The admin is intended for trusted staff users — developers, content  
editors, and site managers — rather than end users of your public site.

The admin interface is available at: `http://127.0.0.1:8000/admin/`.

To access the admin you must first create a superuser account:

```bash
py manage.py createsuperuser
```

Model registration and customisation live in each app's `admin.py` file.

## Plain admin model

Registering a model with `admin.site.register` is the quickest way to make it  
manageable through the admin interface.

```python
from django.contrib import admin

from .models import Task

admin.site.register(Task)
```

Calling `admin.site.register` with only the model class produces a default admin  
view that displays each object using its `__str__` representation. Django generates  
all standard CRUD pages automatically — a list view, an add form, and a change form.  
This one-liner is useful during early development or when no custom behaviour is  
needed. For production use it is common to accompany the model with a  
`ModelAdmin` subclass to control exactly which fields and features are exposed.

## Create pagination

Large tables become unwieldy without pagination. The `list_per_page` attribute  
controls how many records appear on a single page of the change list.

```python
from django.contrib import admin

from .models import Message


class MessageAdmin(admin.ModelAdmin):

    list_display = ['text']
    list_per_page = 10


admin.site.register(Message, MessageAdmin)
```

`list_per_page` defaults to `100` in Django's base `ModelAdmin`. Setting it to a  
smaller value such as `10` keeps the page responsive and reduces the amount of data  
fetched from the database on every request. The `list_display` attribute determines  
which columns appear in the change list; at least one column must always be present.  
Django renders pagination controls at the bottom of the list automatically once the  
number of objects exceeds `list_per_page`.

## Ordering

The `ordering` attribute specifies the default sort order of records displayed in  
the change list. A leading minus sign reverses the direction to descending.

```python
from django.contrib import admin

from .models import Message


class MessageAdmin(admin.ModelAdmin):

    list_display = ['text', 'created']
    list_per_page = 10
    ordering = ['-created']


admin.site.register(Message, MessageAdmin)
```

`ordering` accepts a list of field names, mirroring the syntax used in ORM  
`QuerySet.order_by` calls. Prefixing a field name with `-` sorts that column in  
descending order, so `-created` shows the newest messages first. This admin-level  
ordering overrides any default ordering defined in the model's `Meta` class, but  
users can still click column headers in the browser to change the sort direction  
interactively.

## list_display

`list_display` controls which model fields (or callable columns) appear as columns  
in the change list table.

```python
from django.contrib import admin

from .models import Article


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'author', 'published', 'created_at']


admin.site.register(Article, ArticleAdmin)
```

Each entry in `list_display` can be a model field name, a callable that accepts a  
model instance, or the name of a method defined on the `ModelAdmin` class. When a  
field name is used, Django renders the column header using the field's `verbose_name`.  
Boolean fields are displayed with tick/cross icons. Fields that are not database  
columns (such as methods) are not sortable by default unless you set the  
`admin_order_field` attribute on the callable.

## search_fields

`search_fields` adds a search box to the change list that filters records using  
`LIKE` (or `icontains`) queries across the specified fields.

```python
from django.contrib import admin

from .models import Customer


class CustomerAdmin(admin.ModelAdmin):

    list_display = ['first_name', 'last_name', 'email']
    search_fields = ['first_name', 'last_name', 'email']


admin.site.register(Customer, CustomerAdmin)
```

Each entry in `search_fields` is a field lookup path. You can traverse relations  
using the double-underscore notation (e.g. `'author__username'`) and apply lookup  
types with a prefix: `^` for `startswith`, `=` for exact match, `@` for full-text  
search (on databases that support it), and no prefix for `icontains`. When a user  
enters multiple words, Django splits them and requires that every word match at least  
one of the listed fields, narrowing the results progressively.

## list_filter

`list_filter` adds a sidebar panel to the change list with clickable filters that  
narrow the displayed records by field value.

```python
from django.contrib import admin

from .models import Article


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'author', 'published', 'created_at']
    list_filter = ['published', 'author', 'created_at']


admin.site.register(Article, ArticleAdmin)
```

Field names listed in `list_filter` generate filter widgets automatically. Boolean  
fields produce "Yes / No / All" links, `DateField` and `DateTimeField` produce a  
date-drill-down panel with shortcuts such as *Any date*, *Today*, *Past 7 days*, and  
*This month*. You can also supply a custom `SimpleListFilter` subclass when you need  
logic that cannot be expressed with a plain field name, for example filtering by a  
computed property or an annotation.

## date_hierarchy

`date_hierarchy` adds a date drill-down navigation bar at the top of the change list,  
allowing quick filtering by year, month, and day.

```python
from django.contrib import admin

from .models import Order


class OrderAdmin(admin.ModelAdmin):

    list_display = ['id', 'customer', 'total', 'created_at']
    date_hierarchy = 'created_at'


admin.site.register(Order, OrderAdmin)
```

Set `date_hierarchy` to the name of a `DateField` or `DateTimeField` on the model.  
Django renders a breadcrumb-style navigation row that starts at the year level. Users  
can click through to month and then day views to find records within a specific time  
period. Only one field can be used as the hierarchy at a time, and the field must be  
present on the model (or accessible via a relation using the double-underscore  
notation).

## readonly_fields

`readonly_fields` lists fields that are displayed on the add/change form but cannot  
be edited by the admin user.

```python
from django.contrib import admin

from .models import Article


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'slug', 'created_at']
    readonly_fields = ['slug', 'created_at', 'updated_at']


admin.site.register(Article, ArticleAdmin)
```

Fields in `readonly_fields` are rendered as plain text rather than form inputs,  
preventing accidental modification of auto-generated or auditing data. This is  
especially useful for timestamps set by `auto_now_add` or `auto_now`, which Django  
excludes from forms by default anyway, but which you may still want to display.  
You can also include callables in `readonly_fields` to show computed values that have  
no corresponding database column at all.

## fieldsets

`fieldsets` groups the fields on the add/change form into named sections, improving  
readability for models with many fields.

```python
from django.contrib import admin

from .models import Customer


class CustomerAdmin(admin.ModelAdmin):

    fieldsets = [
        ('Personal info', {
            'fields': ['first_name', 'last_name', 'birth_date'],
        }),
        ('Contact details', {
            'fields': ['email', 'phone'],
            'classes': ['collapse'],
        }),
        ('Account', {
            'fields': ['is_active', 'created_at'],
        }),
    ]
    readonly_fields = ['created_at']


admin.site.register(Customer, CustomerAdmin)
```

Each element of `fieldsets` is a two-tuple containing a section title and a  
dictionary of options. The `fields` key lists the fields to include in that section.  
Adding `'collapse'` to `classes` makes the section collapsible in the browser,  
keeping the form compact for optional or rarely edited data. Setting the title to  
`None` renders the section without a heading. When `fieldsets` is defined, the  
`fields` option on `ModelAdmin` is ignored.

## StackedInline

`StackedInline` embeds a related model's form inside the parent model's change page,  
with each related object's fields stacked vertically.

```python
from django.contrib import admin

from .models import Author, Book


class BookInline(admin.StackedInline):

    model = Book
    extra = 1


class AuthorAdmin(admin.ModelAdmin):

    list_display = ['name', 'birth_date']
    inlines = [BookInline]


admin.site.register(Author, AuthorAdmin)
```

`StackedInline` is suitable when each related object has several fields and you want  
them clearly separated. The `extra` attribute controls how many blank forms are  
rendered for adding new related objects. Setting `extra = 0` shows only existing  
records and a link to add more. You can also set `max_num` to cap the total number  
of related objects, and `min_num` to require a minimum. The inline model must have a  
`ForeignKey` pointing to the parent model.

## TabularInline

`TabularInline` also embeds related objects inside the parent change page, but  
arranges each object's fields in a compact table row instead of a vertical stack.

```python
from django.contrib import admin

from .models import Order, OrderItem


class OrderItemInline(admin.TabularInline):

    model = OrderItem
    extra = 0
    fields = ['product', 'quantity', 'unit_price']
    readonly_fields = ['unit_price']


class OrderAdmin(admin.ModelAdmin):

    list_display = ['id', 'customer', 'created_at']
    inlines = [OrderItemInline]


admin.site.register(Order, OrderAdmin)
```

`TabularInline` is the preferred style when related objects have only a few fields,  
because the table layout is much more space-efficient than the stacked layout.  
Combining `extra = 0` with `readonly_fields` is a common pattern for order-line  
items where prices should not be editable after the order is placed. The `fields`  
option restricts which columns appear in the table, hiding sensitive or irrelevant  
database columns from the inline view.

## Custom admin actions

Admin actions are functions that operate on a queryset of selected objects. They  
appear in the *Action* dropdown at the top of the change list.

```python
from django.contrib import admin
from django.utils import timezone

from .models import Article


@admin.action(description='Mark selected articles as published')
def make_published(modeladmin, request, queryset):
    queryset.update(published=True, published_at=timezone.now())


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'published', 'published_at']
    actions = [make_published]


admin.site.register(Article, ArticleAdmin)
```

An action is any callable that accepts three arguments: the `ModelAdmin` instance,  
the current `HttpRequest`, and a `QuerySet` of the selected objects. Using  
`queryset.update` is efficient because it issues a single SQL `UPDATE` statement  
rather than loading each object into Python. The `@admin.action` decorator sets the  
label that appears in the dropdown. Actions can also return an `HttpResponse` if they  
need to redirect the user or offer a file download after processing.

## prepopulated_fields

`prepopulated_fields` automatically fills a field (typically a `SlugField`) from  
another field using JavaScript as the user types.

```python
from django.contrib import admin

from .models import Article


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'slug']
    prepopulated_fields = {'slug': ('title',)}


admin.site.register(Article, ArticleAdmin)
```

The dictionary maps the field to be pre-populated to a tuple of source fields. As  
the user types in the `title` field, Django's admin JavaScript converts the text to  
a URL-friendly slug and fills the `slug` field in real time, applying the same  
transformations as Django's `slugify` utility (lowercasing, replacing spaces with  
hyphens, and removing special characters). Pre-population stops once the user  
manually edits the target field, preserving intentional overrides. This feature only  
works on the add form; on the change form the field is rendered as a regular  
read-write input without the live update behaviour.

## raw_id_fields

`raw_id_fields` replaces the default `<select>` widget for `ForeignKey` and  
`ManyToManyField` fields with a plain text input that accepts a primary-key value,  
paired with a small lookup popup.

```python
from django.contrib import admin

from .models import Review


class ReviewAdmin(admin.ModelAdmin):

    list_display = ['id', 'product', 'rating', 'created_at']
    raw_id_fields = ['product', 'reviewer']


admin.site.register(Review, ReviewAdmin)
```

The default `<select>` widget loads every row of the related table into the HTML  
page, which becomes very slow when the related model has thousands or millions of  
records. Switching to `raw_id_fields` avoids that query — the admin renders a text  
box for the primary key plus a magnifying-glass icon that opens a pop-up search  
window. The pop-up lets the user search and select the related object without  
overwhelming the browser with a large dropdown.

## Custom list display method

A method defined on `ModelAdmin` can be added to `list_display` to show a computed  
value as an extra column in the change list.

```python
from django.contrib import admin
from django.utils.html import format_html

from .models import Product


class ProductAdmin(admin.ModelAdmin):

    list_display = ['name', 'price', 'stock_status']

    @admin.display(description='Stock', ordering='quantity')
    def stock_status(self, obj):
        if obj.quantity == 0:
            return format_html('<span style="color:red;">Out of stock</span>')
        if obj.quantity < 10:
            return format_html('<span style="color:orange;">Low ({})</span>', obj.quantity)
        return format_html('<span style="color:green;">OK ({})</span>', obj.quantity)


admin.site.register(Product, ProductAdmin)
```

The `@admin.display` decorator sets the column header (`description`) and the  
database field used when the column header is clicked to sort (`ordering`). Using  
`format_html` instead of a plain f-string is essential — it marks the HTML as safe  
and automatically escapes any dynamic values, preventing cross-site scripting (XSS)  
vulnerabilities. The method receives the model instance as its second argument, so  
it has full access to any field or property on the object.

## list_editable

`list_editable` allows certain fields to be edited directly in the change list  
without opening the individual object's change form.

```python
from django.contrib import admin

from .models import Product


class ProductAdmin(admin.ModelAdmin):

    list_display = ['name', 'price', 'quantity', 'is_active']
    list_editable = ['price', 'quantity', 'is_active']


admin.site.register(Product, ProductAdmin)
```

Fields in `list_editable` are rendered as form inputs inside the change list table.  
A single *Save* button at the bottom of the page commits all edits in one request,  
making it efficient to update many records at once. A field must appear in  
`list_display` before it can be added to `list_editable`, and the first column of  
the list (which acts as a link to the change form) cannot be editable. This feature  
is best suited for simple scalar fields like prices, counts, or boolean flags.

## list_select_related

`list_select_related` tells Django to follow `ForeignKey` relations when fetching  
the change list queryset, reducing the number of database queries through SQL `JOIN`s  
or `SELECT_RELATED` prefetching.

```python
from django.contrib import admin

from .models import Book


class BookAdmin(admin.ModelAdmin):

    list_display = ['title', 'author_name', 'publisher', 'published_date']
    list_select_related = ['author', 'publisher']

    @admin.display(description='Author', ordering='author__name')
    def author_name(self, obj):
        return obj.author.name


admin.site.register(Book, BookAdmin)
```

Without `list_select_related`, accessing `obj.author.name` in a list display method  
triggers a separate database query for every row — the classic N+1 query problem.  
Setting `list_select_related` to a list of relation names instructs Django to fetch  
those related objects in the initial query using `select_related`, so the entire list  
page is served with a constant number of queries regardless of how many rows it  
contains. Set it to `True` to follow all `ForeignKey` relations automatically.

## Admin site title and header

The admin site's browser tab title and the banner at the top of every admin page can  
be customised by setting attributes directly on `admin.site`.

```python
# In your main urls.py or in an AppConfig.ready() method

from django.contrib import admin

admin.site.site_header = 'Bookshop Management'
admin.site.site_title = 'Bookshop Admin'
admin.site.index_title = 'Dashboard'
```

`site_header` changes the `<h1>` banner displayed at the top of every admin page,  
replacing the default "Django administration" text. `site_title` changes the  
`<title>` element that appears in browser tabs and bookmarks. `index_title` replaces  
the "Site administration" heading shown on the admin home page. These three  
attributes are the simplest way to brand the admin for a specific project without  
creating a custom `AdminSite` subclass or overriding templates.

## Custom AdminSite

A custom `AdminSite` subclass lets you create a completely independent admin  
interface with its own URL, branding, and set of registered models.

```python
# myapp/admin_site.py
from django.contrib.admin import AdminSite


class StaffAdminSite(AdminSite):

    site_header = 'Staff Portal'
    site_title = 'Staff Admin'
    index_title = 'Staff Dashboard'


staff_admin = StaffAdminSite(name='staff_admin')
```

```python
# myapp/admin.py
from .admin_site import staff_admin
from .models import Report, StaffNote

staff_admin.register(Report)
staff_admin.register(StaffNote)
```

```python
# urls.py
from django.urls import path
from myapp.admin_site import staff_admin

urlpatterns = [
    path('staff/', staff_admin.urls),
]
```

Creating a custom `AdminSite` is useful when an application needs multiple distinct  
admin areas — for example a public content editor and a restricted staff portal.  
Each `AdminSite` instance has its own URL namespace (set by the `name` parameter),  
its own login page, and its own registry of models, so the two admin areas remain  
fully independent. You can also override methods such as `has_permission` to apply  
stricter access control than the default superuser check.

## Overriding get_queryset

Overriding `get_queryset` on `ModelAdmin` restricts which objects appear in the  
change list, for example to show only objects belonging to the currently logged-in  
user.

```python
from django.contrib import admin

from .models import Report


class ReportAdmin(admin.ModelAdmin):

    list_display = ['title', 'owner', 'created_at']

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        if request.user.is_superuser:
            return qs
        return qs.filter(owner=request.user)


admin.site.register(Report, ReportAdmin)
```

`get_queryset` is called every time the change list page is loaded. Calling  
`super().get_queryset(request)` first ensures that any ordering, annotations, or  
`select_related` calls that Django or other mixins add are preserved. The method  
receives the current `HttpRequest`, giving access to `request.user` for  
permission-based filtering. This pattern is also commonly combined with  
`save_model` to automatically set the owner field when a new object is created.

## Custom save_model

Overriding `save_model` on `ModelAdmin` lets you inject custom logic — such as  
automatically setting a field value — whenever an object is saved through the admin.

```python
from django.contrib import admin

from .models import Article


class ArticleAdmin(admin.ModelAdmin):

    list_display = ['title', 'author', 'created_at']
    readonly_fields = ['author', 'created_at']

    def save_model(self, request, obj, form, change):
        if not change:
            # Set author only when creating a new object
            obj.author = request.user
        super().save_model(request, obj, form, change)


admin.site.register(Article, ArticleAdmin)
```

`save_model` receives four arguments: the current `HttpRequest`, the model instance  
about to be saved, the bound `ModelForm`, and a boolean `change` that is `True` when  
editing an existing object and `False` when creating a new one. Calling  
`super().save_model` performs the actual database write, so you must always include  
it. This hook is the cleanest place to set audit fields, trigger notifications, or  
run any side-effect that should happen every time an object is saved from the admin.

## list_display_links

`list_display_links` controls which columns in the change list are rendered as  
hyperlinks to the object's change form.

```python
from django.contrib import admin

from .models import Customer


class CustomerAdmin(admin.ModelAdmin):

    list_display = ['id', 'first_name', 'last_name', 'email']
    list_display_links = ['id', 'first_name']


admin.site.register(Customer, CustomerAdmin)
```

By default Django makes only the first column a link. Setting `list_display_links`  
to a list of field names turns each of those columns into a clickable link. You can  
set it to `None` to remove all links from the list, which is sometimes done when  
`list_editable` is used instead, keeping the list view as a bulk-edit table with no  
drill-down. At least one field in `list_display_links` must also appear in  
`list_display`, and you cannot have a field in both `list_display_links` and  
`list_editable` at the same time.

## Custom admin form

The `form` attribute on `ModelAdmin` replaces the auto-generated `ModelForm` with a  
custom one, enabling custom validation, widgets, and field overrides on the  
add/change page.

```python
from django import forms
from django.contrib import admin

from .models import Product


class ProductAdminForm(forms.ModelForm):

    class Meta:
        model = Product
        fields = '__all__'
        widgets = {
            'description': forms.Textarea(attrs={'rows': 3, 'cols': 60}),
        }

    def clean_price(self):
        price = self.cleaned_data['price']
        if price <= 0:
            raise forms.ValidationError('Price must be greater than zero.')
        return price


class ProductAdmin(admin.ModelAdmin):

    form = ProductAdminForm
    list_display = ['name', 'price', 'quantity']


admin.site.register(Product, ProductAdmin)
```

Assigning a custom `ModelForm` to `form` gives you the full power of Django's form  
validation system inside the admin. The `clean_<field>` methods you define run  
whenever the form is submitted, and any `ValidationError` they raise is shown as a  
field-level error message in the admin UI. Custom widgets specified in the `Meta`  
inner class change how individual fields are rendered without affecting the rest of  
the form. This approach is cleaner than overriding the admin template and keeps  
validation logic co-located with the model layer.

