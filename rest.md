# Django REST Framework

Django REST Framework (DRF) is a powerful and flexible toolkit for building Web APIs  
in Django. It provides a comprehensive set of features including serialization,  
authentication, permissions, viewsets, routers, and more. DRF simplifies the creation  
of RESTful APIs by offering reusable components that handle common patterns like CRUD  
operations, content negotiation, and request/response handling. With its browsable API  
interface, extensive documentation, and active community, DRF has become the de facto  
standard for building REST APIs in the Django ecosystem. It supports various  
authentication schemes, offers flexible serialization options, and includes built-in  
support for pagination, filtering, and throttling.

## Installation and setup

To use Django REST Framework, you need to install it via pip and add it to your  
Django project's installed apps. DRF integrates seamlessly with existing Django  
projects and can be added incrementally without disrupting your current application  
structure.

```python
# Install DRF
# pip install djangorestframework

# In settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third-party apps
    'rest_framework',
    
    # Your apps
    'myapp',
]

# Optional: Configure DRF settings
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

After installation, DRF provides a browsable API interface that allows you to interact  
with your API directly from the browser. This is useful for testing and development.

## Basic serializer

Serializers in DRF convert complex data types like Django model instances into Python  
data types that can be easily rendered into JSON, XML, or other content types. They  
also provide deserialization, allowing parsed data to be converted back into complex  
types after validating the incoming data.

```python
# In serializers.py
from rest_framework import serializers

class BookSerializer(serializers.Serializer):
    """Basic serializer for book data"""
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    author = serializers.CharField(max_length=100)
    isbn = serializers.CharField(max_length=13)
    published_date = serializers.DateField()
    price = serializers.DecimalField(max_digits=6, decimal_places=2)
    
    def create(self, validated_data):
        """Create and return a new Book instance"""
        return Book.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        """Update and return an existing Book instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.isbn = validated_data.get('isbn', instance.isbn)
        instance.published_date = validated_data.get('published_date', 
                                                      instance.published_date)
        instance.price = validated_data.get('price', instance.price)
        instance.save()
        return instance
```

Basic serializers require you to explicitly define all fields and implement create  
and update methods. This gives you full control but requires more code.

## ModelSerializer

ModelSerializer is a shortcut that automatically generates serializer fields based  
on your Django model. It significantly reduces code duplication and automatically  
provides implementations of create and update methods.

```python
# In models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    isbn = models.CharField(max_length=13, unique=True)
    published_date = models.DateField()
    price = models.DecimalField(max_digits=6, decimal_places=2)
    in_stock = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title

# In serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    """ModelSerializer automatically creates fields from the model"""
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'isbn', 'published_date', 
                  'price', 'in_stock']
        # Or use '__all__' to include all fields
        # fields = '__all__'
        # Or exclude specific fields
        # exclude = ['created_at']
```

ModelSerializer inspects your model and automatically creates appropriate serializer  
fields, saving you from writing repetitive code while maintaining flexibility.

## Function-based API views

Function-based views provide a simple way to create API endpoints using the  
`@api_view` decorator. They're straightforward and easy to understand, making them  
perfect for simple endpoints or when you need fine-grained control.

```python
# In views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'POST'])
def book_list(request):
    """
    List all books or create a new book.
    GET: Returns list of all books
    POST: Creates a new book
    """
    if request.method == 'GET':
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def book_detail(request, pk):
    """
    Retrieve, update or delete a book instance.
    """
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)
    
    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    elif request.method == 'DELETE':
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

The `@api_view` decorator wraps your function to provide request parsing, content  
negotiation, and proper HTTP responses with appropriate status codes.

## APIView class-based views

Class-based views provide a more structured approach using classes and methods that  
correspond to HTTP verbs. They promote code reusability through inheritance and mixins,  
making them ideal for more complex endpoints.

```python
# In views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.http import Http404
from .models import Book
from .serializers import BookSerializer

class BookList(APIView):
    """
    List all books or create a new book.
    """
    def get(self, request, format=None):
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    def post(self, request, format=None):
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class BookDetail(APIView):
    """
    Retrieve, update or delete a book instance.
    """
    def get_object(self, pk):
        try:
            return Book.objects.get(pk=pk)
        except Book.DoesNotExist:
            raise Http404
    
    def get(self, request, pk, format=None):
        book = self.get_object(pk)
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    def put(self, request, pk, format=None):
        book = self.get_object(pk)
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk, format=None):
        book = self.get_object(pk)
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

APIView classes map HTTP methods directly to class methods, providing a clean  
separation of concerns and making the code more maintainable.

## Generic views

Generic views provide common patterns for list, create, retrieve, update, and delete  
operations. They significantly reduce boilerplate code by handling standard CRUD  
operations automatically.

```python
# In views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer

class BookListCreate(generics.ListCreateAPIView):
    """
    Concrete view for listing and creating books.
    GET: List all books
    POST: Create a new book
    """
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookRetrieveUpdateDestroy(generics.RetrieveUpdateDestroyAPIView):
    """
    Concrete view for retrieving, updating, and deleting a book.
    GET: Retrieve a book
    PUT/PATCH: Update a book
    DELETE: Delete a book
    """
    queryset = Book.objects.all()
    serializer_class = BookSerializer

# Or use separate generic views
class BookList(generics.ListAPIView):
    """List all books"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookCreate(generics.CreateAPIView):
    """Create a new book"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookDetail(generics.RetrieveAPIView):
    """Retrieve a single book"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

Generic views handle all the common logic internally, requiring you to only specify  
the queryset and serializer class. This makes your code more concise and maintainable.

## ViewSets

ViewSets combine the logic for multiple related views into a single class. Instead  
of defining separate views for list, create, retrieve, update, and delete operations,  
you define them once in a ViewSet.

```python
# In views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    """
    ViewSet for handling all CRUD operations on Book model.
    Automatically provides list, create, retrieve, update, and destroy actions.
    """
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=False, methods=['get'])
    def in_stock(self, request):
        """Custom action to get only books that are in stock"""
        books = Book.objects.filter(in_stock=True)
        serializer = self.get_serializer(books, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def mark_out_of_stock(self, request, pk=None):
        """Custom action to mark a book as out of stock"""
        book = self.get_object()
        book.in_stock = False
        book.save()
        serializer = self.get_serializer(book)
        return Response(serializer.data)

# Read-only ViewSet (only provides list and retrieve)
class BookReadOnlyViewSet(viewsets.ReadOnlyModelViewSet):
    """ViewSet that only allows read operations"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

ViewSets are especially powerful when combined with routers, which automatically  
generate URL patterns for all the standard actions.

## Routers

Routers automatically generate URL patterns for your ViewSets, eliminating the need  
to manually define URL patterns for each action. They follow REST conventions and  
create consistent, predictable URLs.

```python
# In urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

# Create a router and register our viewsets
router = DefaultRouter()
router.register(r'books', views.BookViewSet, basename='book')

# The API URLs are now determined automatically by the router
urlpatterns = [
    path('api/', include(router.urls)),
]

# This generates the following URL patterns:
# GET    /api/books/              -> list
# POST   /api/books/              -> create
# GET    /api/books/{pk}/         -> retrieve
# PUT    /api/books/{pk}/         -> update
# PATCH  /api/books/{pk}/         -> partial_update
# DELETE /api/books/{pk}/         -> destroy
# GET    /api/books/in_stock/     -> custom action
# POST   /api/books/{pk}/mark_out_of_stock/ -> custom action

# Alternative: Using SimpleRouter (no API root view)
from rest_framework.routers import SimpleRouter

router = SimpleRouter()
router.register(r'books', views.BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

Routers reduce boilerplate code and ensure your URLs follow REST conventions  
consistently across your API.

## Serializer fields and validation

Custom fields and validators allow you to control how data is serialized and validated.  
You can add custom validation logic to ensure data integrity beyond what Django models  
provide.

```python
# In serializers.py
from rest_framework import serializers
from .models import Book
import datetime

class BookSerializer(serializers.ModelSerializer):
    # Read-only field
    days_since_published = serializers.SerializerMethodField()
    
    # Write-only field (e.g., for sensitive data)
    internal_code = serializers.CharField(write_only=True, required=False)
    
    # Custom field with source parameter
    book_author = serializers.CharField(source='author')
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'book_author', 'isbn', 'published_date', 
                  'price', 'days_since_published', 'internal_code']
        read_only_fields = ['id']
    
    def get_days_since_published(self, obj):
        """Calculate days since publication"""
        delta = datetime.date.today() - obj.published_date
        return delta.days
    
    def validate_isbn(self, value):
        """Field-level validation for ISBN"""
        if len(value) != 13:
            raise serializers.ValidationError("ISBN must be 13 characters")
        if not value.isdigit():
            raise serializers.ValidationError("ISBN must contain only digits")
        return value
    
    def validate_price(self, value):
        """Ensure price is positive"""
        if value <= 0:
            raise serializers.ValidationError("Price must be greater than 0")
        return value
    
    def validate(self, data):
        """Object-level validation across multiple fields"""
        if data.get('published_date'):
            if data['published_date'] > datetime.date.today():
                raise serializers.ValidationError(
                    "Published date cannot be in the future"
                )
        return data
```

Custom validators help enforce business rules and ensure data consistency before  
it reaches your database.

## Nested serializers

Nested serializers allow you to represent relationships between models in your API  
responses. This is useful for including related data without requiring additional  
API calls.

```python
# In models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    birth_date = models.DateField()
    email = models.EmailField()
    
    def __str__(self):
        return self.name

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, 
                               related_name='books')
    isbn = models.CharField(max_length=13)
    published_date = models.DateField()
    
    def __str__(self):
        return self.title

# In serializers.py
from rest_framework import serializers
from .models import Author, Book

class BookSerializer(serializers.ModelSerializer):
    """Serializer for books without nested author"""
    class Meta:
        model = Book
        fields = ['id', 'title', 'isbn', 'published_date']

class AuthorSerializer(serializers.ModelSerializer):
    """Serializer with nested books"""
    books = BookSerializer(many=True, read_only=True)
    book_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'birth_date', 'email', 'books', 'book_count']
    
    def get_book_count(self, obj):
        return obj.books.count()

# Alternative: Using depth parameter for automatic nesting
class AuthorDetailSerializer(serializers.ModelSerializer):
    class Meta:
        model = Author
        fields = ['id', 'name', 'birth_date', 'email', 'books']
        depth = 1  # Automatically nest related objects one level deep
```

Nested serializers provide a convenient way to return related data in a single  
response, reducing the number of API calls clients need to make.

## Writable nested serializers

By default, nested serializers are read-only. To create or update nested objects,  
you need to explicitly handle the nested data in your serializer's create and update  
methods.

```python
# In serializers.py
from rest_framework import serializers
from .models import Author, Book

class BookCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['title', 'isbn', 'published_date']

class AuthorWithBooksSerializer(serializers.ModelSerializer):
    books = BookCreateSerializer(many=True)
    
    class Meta:
        model = Author
        fields = ['id', 'name', 'birth_date', 'email', 'books']
    
    def create(self, validated_data):
        """Create author with nested books"""
        books_data = validated_data.pop('books')
        author = Author.objects.create(**validated_data)
        
        for book_data in books_data:
            Book.objects.create(author=author, **book_data)
        
        return author
    
    def update(self, instance, validated_data):
        """Update author and nested books"""
        books_data = validated_data.pop('books', None)
        
        # Update author fields
        instance.name = validated_data.get('name', instance.name)
        instance.birth_date = validated_data.get('birth_date', 
                                                  instance.birth_date)
        instance.email = validated_data.get('email', instance.email)
        instance.save()
        
        # Update or create books
        if books_data is not None:
            # Simple approach: delete old books and create new ones
            instance.books.all().delete()
            for book_data in books_data:
                Book.objects.create(author=instance, **book_data)
        
        return instance
```

Implementing writable nested serializers gives you fine-grained control over how  
related objects are created and updated through your API.

## HyperlinkedModelSerializer

HyperlinkedModelSerializer uses hyperlinks to represent relationships instead of  
primary keys. This makes your API more browsable and follows REST best practices  
by using URIs to identify resources.

```python
# In serializers.py
from rest_framework import serializers
from .models import Author, Book

class BookHyperlinkedSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Book
        fields = ['url', 'title', 'author', 'isbn', 'published_date']
        extra_kwargs = {
            'url': {'view_name': 'book-detail'},
            'author': {'view_name': 'author-detail'}
        }

class AuthorHyperlinkedSerializer(serializers.HyperlinkedModelSerializer):
    books = BookHyperlinkedSerializer(many=True, read_only=True)
    
    class Meta:
        model = Author
        fields = ['url', 'name', 'birth_date', 'email', 'books']
        extra_kwargs = {
            'url': {'view_name': 'author-detail'}
        }

# In views.py
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookHyperlinkedSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorHyperlinkedSerializer

# In urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'books', BookViewSet, basename='book')
router.register(r'authors', AuthorViewSet, basename='author')
```

Hyperlinked serializers make your API self-documenting by including URLs to related  
resources, making it easier for clients to navigate your API.

## Authentication

DRF provides multiple authentication schemes including token authentication, session  
authentication, and OAuth. You can configure authentication globally or per-view.

```python
# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}

# For TokenAuthentication, add to INSTALLED_APPS
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework.authtoken',
]

# Run migrations to create token table
# python manage.py migrate

# In views.py
from rest_framework.authentication import TokenAuthentication, SessionAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    authentication_classes = [TokenAuthentication, SessionAuthentication]
    permission_classes = [IsAuthenticated]

# Create tokens for users
from rest_framework.authtoken.models import Token
from django.contrib.auth.models import User

# For existing users
for user in User.objects.all():
    Token.objects.get_or_create(user=user)

# For new users (in signals or views)
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)

# Login view to obtain token
from rest_framework.authtoken.views import obtain_auth_token
from django.urls import path

urlpatterns = [
    path('api-token-auth/', obtain_auth_token, name='api_token_auth'),
]
```

Authentication identifies who is making the request, which is essential for  
implementing permissions and tracking user actions.

## Permissions

Permissions determine whether a request should be granted or denied access to a view.  
DRF provides several built-in permission classes and supports custom permissions.

```python
# In views.py
from rest_framework import viewsets, permissions
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_permissions(self):
        """Set different permissions for different actions"""
        if self.action in ['list', 'retrieve']:
            permission_classes = [permissions.AllowAny]
        else:
            permission_classes = [permissions.IsAuthenticated]
        return [permission() for permission in permission_classes]

# Custom permission
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission: only owners can edit objects
    """
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed for any request
        if request.method in permissions.SAFE_METHODS:
            return True
        
        # Write permissions only for the owner
        return obj.owner == request.user

class IsAdminOrReadOnly(permissions.BasePermission):
    """
    Custom permission: only admin users can modify
    """
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff

# Using custom permissions
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
```

Permissions control what authenticated users can do, allowing you to implement  
fine-grained access control in your API.

## Filtering

Filtering allows clients to narrow down query results based on specific criteria.  
DRF supports filtering through query parameters using django-filter integration.

```python
# Install django-filter: pip install django-filter

# In settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
}

# In views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['author', 'published_date', 'in_stock']

# Custom filter
from django_filters import rest_framework as filters

class BookFilter(filters.FilterSet):
    title = filters.CharFilter(lookup_expr='icontains')
    author = filters.CharFilter(field_name='author__name', 
                                lookup_expr='icontains')
    min_price = filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = filters.NumberFilter(field_name='price', lookup_expr='lte')
    published_after = filters.DateFilter(field_name='published_date', 
                                        lookup_expr='gte')
    
    class Meta:
        model = Book
        fields = ['title', 'author', 'min_price', 'max_price', 
                  'published_after']

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = BookFilter

# Usage:
# GET /api/books/?author=Smith
# GET /api/books/?min_price=10&max_price=50
# GET /api/books/?title=django&in_stock=true
```

Filtering enables clients to query specific subsets of data, making your API more  
flexible and efficient.

## Searching

Search functionality allows clients to perform text-based searches across multiple  
fields. DRF's SearchFilter provides a simple way to add search capabilities.

```python
# In views.py
from rest_framework import viewsets, filters

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['title', 'author', 'isbn']
    # Or use different lookup types
    # search_fields = ['=title', '@author', 'isbn']
    # = : Exact match
    # ^ : Starts-with search
    # @ : Full-text search (PostgreSQL only)
    # $ : Regex search

# Combining search with other filters
from django_filters.rest_framework import DjangoFilterBackend

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, 
                      filters.OrderingFilter]
    filterset_fields = ['in_stock', 'published_date']
    search_fields = ['title', 'author']
    ordering_fields = ['published_date', 'price', 'title']
    ordering = ['-published_date']  # Default ordering

# Usage:
# GET /api/books/?search=django
# GET /api/books/?search=python&in_stock=true
# GET /api/books/?search=programming&ordering=-price
```

Search functionality makes it easy for users to find specific resources without  
needing to know exact field values.

## Pagination

Pagination divides large result sets into smaller pages, improving API performance  
and user experience. DRF provides several pagination styles out of the box.

```python
# In settings.py - Global pagination
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 
        'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}

# Custom pagination classes
from rest_framework.pagination import (
    PageNumberPagination, 
    LimitOffsetPagination,
    CursorPagination
)

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 100

class LargeResultsSetPagination(PageNumberPagination):
    page_size = 100
    page_size_query_param = 'page_size'
    max_page_size = 1000

class CustomLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 10
    max_limit = 100

class CustomCursorPagination(CursorPagination):
    page_size = 10
    ordering = '-created_at'  # Required for cursor pagination
    cursor_query_param = 'cursor'

# In views.py
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    pagination_class = StandardResultsSetPagination

# Different pagination for different views
class RecentBooksView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    pagination_class = CustomCursorPagination

# Usage:
# PageNumberPagination: /api/books/?page=2
# LimitOffsetPagination: /api/books/?limit=10&offset=20
# CursorPagination: /api/books/?cursor=cD0yMDIx...
```

Proper pagination is essential for APIs that return large datasets, ensuring good  
performance and preventing server overload.

## Ordering

Ordering allows clients to sort results by specific fields. The OrderingFilter  
provides a flexible way to enable sorting on your API endpoints.

```python
# In views.py
from rest_framework import viewsets, filters

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['title', 'published_date', 'price']
    ordering = ['title']  # Default ordering

# Allow ordering by all fields
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = '__all__'

# Custom ordering with related fields
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.select_related('author')
    serializer_class = BookSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['title', 'published_date', 'price', 'author__name']

# Usage:
# GET /api/books/?ordering=price
# GET /api/books/?ordering=-published_date  # Descending order
# GET /api/books/?ordering=author__name,title  # Multiple fields
```

Ordering gives clients control over how results are sorted, making data easier  
to browse and analyze.

## Throttling

Throttling controls the rate at which clients can make requests to your API.  
It helps prevent abuse and ensures fair usage of API resources.

```python
# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}

# Custom throttle classes
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'

# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day'
    }
}

# In views.py
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [AnonRateThrottle, UserRateThrottle]

# Different throttles for different actions
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_throttles(self):
        if self.action == 'create':
            throttle_classes = [BurstRateThrottle]
        else:
            throttle_classes = [SustainedRateThrottle]
        return [throttle() for throttle in throttle_classes]
```

Throttling protects your API from abuse and helps maintain service quality for  
all users by preventing any single client from overwhelming your servers.

## File uploads

DRF supports file uploads through serializers and provides various ways to handle  
file fields. You can upload images, documents, and other file types.

```python
# In models.py
from django.db import models

class BookCover(models.Model):
    book = models.OneToOneField(Book, on_delete=models.CASCADE, 
                               related_name='cover')
    image = models.ImageField(upload_to='book_covers/')
    uploaded_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f"Cover for {self.book.title}"

# In serializers.py
from rest_framework import serializers

class BookCoverSerializer(serializers.ModelSerializer):
    class Meta:
        model = BookCover
        fields = ['id', 'book', 'image', 'uploaded_at']
        read_only_fields = ['uploaded_at']

class BookWithCoverSerializer(serializers.ModelSerializer):
    cover = BookCoverSerializer(read_only=True)
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'cover']

# In views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.parsers import MultiPartParser, FormParser

class BookCoverViewSet(viewsets.ModelViewSet):
    queryset = BookCover.objects.all()
    serializer_class = BookCoverSerializer
    parser_classes = [MultiPartParser, FormParser]

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookWithCoverSerializer
    
    @action(detail=True, methods=['post'], 
            parser_classes=[MultiPartParser, FormParser])
    def upload_cover(self, request, pk=None):
        book = self.get_object()
        serializer = BookCoverSerializer(data=request.data)
        
        if serializer.is_valid():
            serializer.save(book=book)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

File uploads are handled through multipart form data, and DRF's parsers handle  
the complexity of processing uploaded files.

## Custom actions

Custom actions allow you to add endpoints that don't fit the standard CRUD pattern.  
Use the `@action` decorator to create custom endpoints on ViewSets.

```python
# In views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from django.db.models import Avg, Count
from datetime import date, timedelta

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    @action(detail=False, methods=['get'])
    def recent(self, request):
        """Get books published in the last 30 days"""
        thirty_days_ago = date.today() - timedelta(days=30)
        recent_books = Book.objects.filter(published_date__gte=thirty_days_ago)
        serializer = self.get_serializer(recent_books, many=True)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'])
    def statistics(self, request):
        """Get statistics about books"""
        stats = Book.objects.aggregate(
            total_books=Count('id'),
            average_price=Avg('price'),
            in_stock_count=Count('id', filter=models.Q(in_stock=True))
        )
        return Response(stats)
    
    @action(detail=True, methods=['post'])
    def toggle_stock(self, request, pk=None):
        """Toggle book's stock status"""
        book = self.get_object()
        book.in_stock = not book.in_stock
        book.save()
        serializer = self.get_serializer(book)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def by_author(self, request, pk=None):
        """Get all books by the same author"""
        book = self.get_object()
        same_author_books = Book.objects.filter(
            author=book.author
        ).exclude(pk=book.pk)
        serializer = self.get_serializer(same_author_books, many=True)
        return Response(serializer.data)

# Generated URLs:
# GET /api/books/recent/
# GET /api/books/statistics/
# POST /api/books/{pk}/toggle_stock/
# GET /api/books/{pk}/by_author/
```

Custom actions extend your API beyond basic CRUD operations, allowing you to  
implement business-specific endpoints.

## Serializer context

Serializer context allows you to pass additional data to serializers, such as  
the current request, user, or any other information needed during serialization.

```python
# In serializers.py
from rest_framework import serializers

class BookSerializer(serializers.ModelSerializer):
    is_favorited = serializers.SerializerMethodField()
    can_edit = serializers.SerializerMethodField()
    absolute_url = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'price', 
                  'is_favorited', 'can_edit', 'absolute_url']
    
    def get_is_favorited(self, obj):
        """Check if current user has favorited this book"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return obj.favorites.filter(user=request.user).exists()
        return False
    
    def get_can_edit(self, obj):
        """Check if current user can edit this book"""
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            return request.user.is_staff or obj.owner == request.user
        return False
    
    def get_absolute_url(self, obj):
        """Generate absolute URL for the book"""
        request = self.context.get('request')
        if request:
            return request.build_absolute_uri(f'/api/books/{obj.pk}/')
        return None

# In views.py
class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_serializer_context(self):
        """Add custom context data"""
        context = super().get_serializer_context()
        context['show_sensitive_data'] = self.request.user.is_staff
        context['api_version'] = 'v1'
        return context

# Manual serializer usage with context
serializer = BookSerializer(
    book, 
    context={'request': request, 'custom_data': 'value'}
)
```

Context provides a way to pass request-specific or view-specific data to  
serializers, enabling dynamic serialization behavior.

## Versioning

API versioning allows you to make changes to your API while maintaining backward  
compatibility. DRF supports multiple versioning schemes.

```python
# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 
        'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}

# URL path versioning
# In urls.py
from django.urls import path, include

urlpatterns = [
    path('api/v1/', include(('myapp.urls', 'v1'), namespace='v1')),
    path('api/v2/', include(('myapp.urls', 'v2'), namespace='v2')),
]

# In views.py
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    
    def get_serializer_class(self):
        """Return different serializers for different versions"""
        if self.request.version == 'v1':
            return BookSerializerV1
        elif self.request.version == 'v2':
            return BookSerializerV2
        return BookSerializer
    
    def list(self, request, *args, **kwargs):
        """Different behavior for different versions"""
        if request.version == 'v1':
            # Old behavior
            queryset = self.get_queryset()
        else:
            # New behavior with additional filtering
            queryset = self.get_queryset().filter(in_stock=True)
        
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)

# Alternative versioning schemes:
# 1. AcceptHeaderVersioning: Accept: application/json; version=1.0
# 2. NamespaceVersioning: Using URL namespaces
# 3. HostNameVersioning: v1.example.com
# 4. QueryParameterVersioning: /api/books/?version=v1
```

Versioning enables you to evolve your API over time while maintaining support  
for existing clients.

## Testing DRF APIs

DRF provides testing utilities that make it easy to write comprehensive tests  
for your API endpoints. Use APITestCase and APIClient for testing.

```python
# In tests.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth.models import User
from .models import Book
from django.urls import reverse

class BookAPITestCase(APITestCase):
    def setUp(self):
        """Set up test data"""
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.book = Book.objects.create(
            title='Test Book',
            author='Test Author',
            isbn='1234567890123',
            published_date='2023-01-01',
            price=19.99,
            in_stock=True
        )
    
    def test_list_books(self):
        """Test listing all books"""
        url = reverse('book-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
    
    def test_create_book_authenticated(self):
        """Test creating a book while authenticated"""
        self.client.force_authenticate(user=self.user)
        url = reverse('book-list')
        data = {
            'title': 'New Book',
            'author': 'New Author',
            'isbn': '9876543210987',
            'published_date': '2023-06-01',
            'price': 24.99
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Book.objects.count(), 2)
    
    def test_create_book_unauthenticated(self):
        """Test that unauthenticated users cannot create books"""
        url = reverse('book-list')
        data = {'title': 'Should Fail'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_retrieve_book(self):
        """Test retrieving a single book"""
        url = reverse('book-detail', kwargs={'pk': self.book.pk})
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Book')
    
    def test_update_book(self):
        """Test updating a book"""
        self.client.force_authenticate(user=self.user)
        url = reverse('book-detail', kwargs={'pk': self.book.pk})
        data = {'title': 'Updated Title', 'price': 29.99}
        response = self.client.patch(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.book.refresh_from_db()
        self.assertEqual(self.book.title, 'Updated Title')
    
    def test_delete_book(self):
        """Test deleting a book"""
        self.client.force_authenticate(user=self.user)
        url = reverse('book-detail', kwargs={'pk': self.book.pk})
        response = self.client.delete(url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Book.objects.count(), 0)
    
    def test_custom_action(self):
        """Test custom action endpoint"""
        url = reverse('book-recent')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

# Run tests: python manage.py test
```

Comprehensive testing ensures your API behaves correctly and helps catch bugs  
before they reach production.

## Exception handling

DRF provides built-in exception handling and allows you to customize error responses  
to match your API's needs. You can create custom exception handlers for consistent  
error formatting.

```python
# In exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import APIException
from rest_framework import status

class ServiceUnavailable(APIException):
    """Custom exception for service unavailable"""
    status_code = status.HTTP_503_SERVICE_UNAVAILABLE
    default_detail = 'Service temporarily unavailable, try again later.'
    default_code = 'service_unavailable'

class InvalidInput(APIException):
    """Custom exception for invalid input"""
    status_code = status.HTTP_400_BAD_REQUEST
    default_detail = 'Invalid input provided.'
    default_code = 'invalid_input'

def custom_exception_handler(exc, context):
    """
    Custom exception handler that adds additional context to errors
    """
    # Call DRF's default exception handler first
    response = exception_handler(exc, context)
    
    if response is not None:
        # Customize the response data
        custom_response_data = {
            'error': {
                'status_code': response.status_code,
                'message': response.data,
                'path': context['request'].path,
                'method': context['request'].method,
            }
        }
        response.data = custom_response_data
    
    return response

# In settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'myapp.exceptions.custom_exception_handler'
}

# Using custom exceptions in views
from rest_framework import viewsets
from rest_framework.response import Response
from .exceptions import ServiceUnavailable

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def list(self, request, *args, **kwargs):
        # Check some condition
        if not self.check_service_availability():
            raise ServiceUnavailable()
        
        return super().list(request, *args, **kwargs)
    
    def check_service_availability(self):
        # Implement your logic
        return True
```

Custom exception handling provides a consistent error response format across your  
API and helps clients handle errors more effectively.

## Content negotiation

Content negotiation allows your API to return different representations of the same  
resource based on the client's Accept header. DRF supports JSON, XML, and custom  
renderers.

```python
# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
}

# Custom renderer
from rest_framework.renderers import BaseRenderer
import csv
from io import StringIO

class CSVRenderer(BaseRenderer):
    media_type = 'text/csv'
    format = 'csv'
    
    def render(self, data, accepted_media_type=None, renderer_context=None):
        """Render data as CSV"""
        if not isinstance(data, list):
            data = [data]
        
        if not data:
            return ''
        
        output = StringIO()
        writer = csv.DictWriter(output, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)
        
        return output.getvalue()

# In views.py
from rest_framework import viewsets

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    renderer_classes = [JSONRenderer, BrowsableAPIRenderer, CSVRenderer]

# Clients can request different formats:
# Accept: application/json
# Accept: text/csv
# Or via URL: /api/books/?format=csv
```

Content negotiation makes your API more flexible by supporting multiple response  
formats based on client preferences.

## JWT authentication

JSON Web Token (JWT) authentication is a popular stateless authentication method  
for APIs. Unlike token authentication, JWTs contain encoded user information and  
don't require database lookups on every request.

```python
# Install: pip install djangorestframework-simplejwt

# In settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
}

# In urls.py
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]

# Custom JWT claims
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # Add custom claims
        token['username'] = user.username
        token['email'] = user.email
        token['is_staff'] = user.is_staff
        
        return token

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer

# Usage:
# POST /api/token/ with {"username": "user", "password": "pass"}
# Returns: {"access": "...", "refresh": "..."}
# Use: Authorization: Bearer <access_token>
```

JWT authentication is ideal for stateless APIs and single-page applications,  
providing secure authentication without server-side session storage.

## CORS configuration

Cross-Origin Resource Sharing (CORS) allows your API to be accessed from different  
domains. This is essential when your frontend and backend are hosted on different  
origins.

```python
# Install: pip install django-cors-headers

# In settings.py
INSTALLED_APPS = [
    # ...
    'corsheaders',
    'rest_framework',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Should be as high as possible
    'django.middleware.common.CommonMiddleware',
    # ...
]

# Allow all origins (development only)
CORS_ALLOW_ALL_ORIGINS = True

# Or specify allowed origins (production)
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "https://sub.example.com",
    "http://localhost:3000",
    "http://127.0.0.1:8000",
]

# Allow credentials (cookies, authorization headers)
CORS_ALLOW_CREDENTIALS = True

# Allowed HTTP methods
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

# Allowed headers
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
]

# Regex for allowed origins
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://\w+\.example\.com$",
]

# Expose specific headers to the browser
CORS_EXPOSE_HEADERS = [
    'Content-Type',
    'X-Total-Count',
]

# Cache preflight requests
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours
```

Proper CORS configuration is crucial for allowing your API to be consumed by  
web applications hosted on different domains while maintaining security.

## Response customization

Customizing API responses allows you to standardize your response format and add  
metadata. You can wrap responses with custom data structures or add pagination info.

```python
# Custom response wrapper
from rest_framework.response import Response
from rest_framework import status

class APIResponse:
    """Custom response wrapper for consistent API responses"""
    
    @staticmethod
    def success(data=None, message="Success", status_code=status.HTTP_200_OK):
        return Response({
            'success': True,
            'message': message,
            'data': data,
            'error': None
        }, status=status_code)
    
    @staticmethod
    def error(message="Error occurred", errors=None, 
              status_code=status.HTTP_400_BAD_REQUEST):
        return Response({
            'success': False,
            'message': message,
            'data': None,
            'error': errors
        }, status=status_code)

# In views.py
from rest_framework import viewsets
from rest_framework.decorators import action

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def list(self, request, *args, **kwargs):
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)
        return APIResponse.success(
            data=serializer.data,
            message="Books retrieved successfully"
        )
    
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        if serializer.is_valid():
            self.perform_create(serializer)
            return APIResponse.success(
                data=serializer.data,
                message="Book created successfully",
                status_code=status.HTTP_201_CREATED
            )
        return APIResponse.error(
            message="Validation failed",
            errors=serializer.errors,
            status_code=status.HTTP_400_BAD_REQUEST
        )

# Custom renderer for consistent formatting
from rest_framework.renderers import JSONRenderer

class CustomJSONRenderer(JSONRenderer):
    def render(self, data, accepted_media_type=None, renderer_context=None):
        response_data = {
            'status': 'success',
            'data': data
        }
        
        # Add error status for error responses
        if renderer_context and renderer_context.get('response'):
            status_code = renderer_context['response'].status_code
            if status_code >= 400:
                response_data['status'] = 'error'
        
        return super().render(response_data, accepted_media_type, 
                            renderer_context)

# In settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'myapp.renderers.CustomJSONRenderer',
    ],
}
```

Custom response formats provide a consistent structure across all API endpoints,  
making it easier for clients to parse and handle responses.

## Caching

Caching improves API performance by storing frequently accessed data. DRF supports  
various caching strategies including view-level caching and queryset caching.

```python
# In settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}

# Or use Memcached
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

# View-level caching
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from rest_framework import viewsets

class BookViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    # Cache list view for 5 minutes
    @method_decorator(cache_page(60 * 5))
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
    
    # Cache detail view for 15 minutes
    @method_decorator(cache_page(60 * 15))
    def retrieve(self, request, *args, **kwargs):
        return super().retrieve(request, *args, **kwargs)

# Low-level cache API
from django.core.cache import cache
from rest_framework.views import APIView
from rest_framework.response import Response

class BookStatisticsView(APIView):
    def get(self, request):
        # Try to get from cache
        cache_key = 'book_statistics'
        statistics = cache.get(cache_key)
        
        if statistics is None:
            # Calculate statistics (expensive operation)
            statistics = {
                'total_books': Book.objects.count(),
                'total_authors': Author.objects.count(),
                'average_price': Book.objects.aggregate(
                    avg_price=Avg('price')
                )['avg_price']
            }
            # Cache for 1 hour
            cache.set(cache_key, statistics, 60 * 60)
        
        return Response(statistics)

# Cache invalidation
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver([post_save, post_delete], sender=Book)
def invalidate_book_cache(sender, **kwargs):
    """Invalidate cache when books are modified"""
    cache.delete('book_statistics')
    cache.delete_pattern('book_list_*')

# Using cache with pagination
from rest_framework.pagination import PageNumberPagination

class CachedPagination(PageNumberPagination):
    page_size = 10
    
    def paginate_queryset(self, queryset, request, view=None):
        page_number = request.query_params.get(self.page_query_param, 1)
        cache_key = f'book_list_page_{page_number}'
        
        cached_page = cache.get(cache_key)
        if cached_page:
            return cached_page
        
        page = super().paginate_queryset(queryset, request, view)
        cache.set(cache_key, page, 60 * 10)  # Cache for 10 minutes
        return page
```

Effective caching strategies can dramatically improve API performance and reduce  
database load, especially for read-heavy applications.
