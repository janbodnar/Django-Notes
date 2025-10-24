# JWT

JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact  
and self-contained way for securely transmitting information between parties  
as a JSON object. This information can be verified and trusted because it is  
digitally signed. JWTs can be signed using a secret (with the HMAC algorithm)  
or a public/private key pair using RSA or ECDSA.  

In Django applications, JWTs are commonly used for authentication and  
authorization. When a user logs in, the server generates a JWT containing  
the user's identity and claims. This token is then sent to the client, which  
includes it in subsequent requests. The server validates the token without  
needing to query the database on every request, making JWT-based  
authentication stateless and scalable.  

A JWT consists of three parts separated by dots: Header, Payload, and  
Signature. The header typically contains the type of token and the signing  
algorithm. The payload contains the claims (statements about the user and  
additional metadata). The signature is used to verify that the sender of the  
JWT is who it says it is and to ensure that the message wasn't changed along  
the way.  

Django REST Framework (DRF) provides excellent support for JWT  
authentication through packages like `djangorestframework-simplejwt` and  
`djangorestframework-jwt`. These packages handle token generation,  
validation, refresh mechanisms, and integration with Django's authentication  
system seamlessly.  

## Installation

Install Django REST Framework and Simple JWT package.  

```bash
pip install djangorestframework
pip install djangorestframework-simplejwt
```

These packages provide JWT authentication support for Django REST Framework.  
The `djangorestframework-simplejwt` is actively maintained and provides a  
robust implementation of JWT authentication with features like token refresh,  
blacklisting, and customizable token claims.  

## Basic configuration

Configure Django settings to use JWT authentication.  

In `settings.py`:  

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}
```

This configuration sets up JWT as the default authentication mechanism for  
all API views in your Django REST Framework application. Any view that  
requires authentication will now expect a valid JWT token in the request  
headers.  

## Token obtain URLs

Set up URLs for obtaining and refreshing JWT tokens.  

In main `urls.py`:  

```python
from django.contrib import admin
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

The `TokenObtainPairView` endpoint accepts username and password credentials  
and returns access and refresh tokens. The `TokenRefreshView` endpoint  
accepts a refresh token and returns a new access token. This two-token  
system allows for short-lived access tokens with the ability to obtain new  
ones without re-authenticating.  

## Simple protected view

Create a basic API view that requires JWT authentication.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class HelloView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        content = {'message': 'Hello, ' + str(request.user)}
        return Response(content)
```

This view requires a valid JWT token to access. The `IsAuthenticated`  
permission class checks if the request contains a valid token. If the token  
is valid, `request.user` will contain the authenticated user object. Without  
a valid token, the view returns a 401 Unauthorized response.  

## Testing token authentication

Test JWT authentication using curl or httpie.  

First, obtain a token:  

```bash
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username":"john", "password":"secret123"}'
```

Then use the access token to access protected views:  

```bash
curl -X GET http://localhost:8000/api/hello/ \
  -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc..."
```

The first command returns a JSON response containing `access` and `refresh`  
tokens. The access token is short-lived (usually 5-15 minutes) and must be  
included in the Authorization header of subsequent requests using the Bearer  
scheme. The refresh token has a longer lifetime and can be used to obtain  
new access tokens.  

## Configure token lifetime

Customize JWT token expiration times.  

In `settings.py`:  

```python
from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': False,
}
```

This configuration sets the access token to expire after 5 minutes and the  
refresh token after 1 day. Shorter access token lifetimes improve security  
by limiting the window of vulnerability if a token is compromised. The  
refresh token's longer lifetime balances security with user convenience by  
reducing the frequency of required logins.  

## Custom serializer for user details

Include additional user information in token response.  

In `serializers.py`:  

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        token['username'] = user.username
        token['email'] = user.email
        
        return token
```

This custom serializer adds extra claims to the JWT payload. By including  
user information directly in the token, you can avoid additional database  
queries to retrieve user details. However, be cautious about including  
sensitive information as JWT tokens can be decoded by anyone who has access  
to them (though they cannot be modified without invalidating the signature).  

## Custom token view

Use the custom serializer in a token view.  

In `views.py`:  

```python
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import MyTokenObtainPairSerializer

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

In `urls.py`:  

```python
from django.urls import path
from .views import MyTokenObtainPairView

urlpatterns = [
    path('api/token/', MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
]
```

This replaces the default token obtain view with your custom version that  
includes additional user information in the token. Users will receive the  
enhanced tokens when they authenticate through this endpoint.  

## User registration endpoint

Create an endpoint for user registration with JWT response.  

In `views.py`:  

```python
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView
from django.contrib.auth.models import User
from rest_framework_simplejwt.tokens import RefreshToken

class RegisterView(APIView):
    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        email = request.data.get('email')
        
        if User.objects.filter(username=username).exists():
            return Response(
                {'error': 'Username already exists'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        user = User.objects.create_user(
            username=username,
            password=password,
            email=email
        )
        
        refresh = RefreshToken.for_user(user)
        
        return Response({
            'refresh': str(refresh),
            'access': str(refresh.access_token),
        }, status=status.HTTP_201_CREATED)
```

This view handles user registration and immediately returns JWT tokens,  
allowing newly registered users to authenticate without a separate login  
step. The view validates that the username is unique, creates the user with  
a hashed password, and generates tokens using the `RefreshToken.for_user`  
method.  

## User profile endpoint

Create an endpoint to retrieve authenticated user's profile.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class ProfileView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        user = request.user
        return Response({
            'username': user.username,
            'email': user.email,
            'first_name': user.first_name,
            'last_name': user.last_name,
            'date_joined': user.date_joined,
        })
```

This endpoint returns the authenticated user's profile information. The  
`request.user` object is automatically populated by the JWT authentication  
backend when a valid token is provided. This eliminates the need to pass  
user IDs in URLs or query parameters, as the user's identity is derived  
directly from the token.  

## Custom user model with JWT

Use JWT with a custom user model.  

In `models.py`:  

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    phone_number = models.CharField(max_length=15, blank=True)
    date_of_birth = models.DateField(null=True, blank=True)
    
    def __str__(self):
        return self.email
```

In `settings.py`:  

```python
AUTH_USER_MODEL = 'myapp.CustomUser'
```

Custom user models allow you to extend Django's default user with additional  
fields specific to your application. When using JWT with a custom user  
model, ensure you set `AUTH_USER_MODEL` before running initial migrations.  
The JWT authentication will automatically work with your custom user model.  

## Serializer for custom user

Create serializers for the custom user model.  

In `serializers.py`:  

```python
from rest_framework import serializers
from .models import CustomUser

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = CustomUser
        fields = ['id', 'username', 'email', 'phone_number', 'date_of_birth']
        read_only_fields = ['id']

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)
    
    class Meta:
        model = CustomUser
        fields = ['username', 'email', 'password', 'phone_number', 'date_of_birth']
    
    def create(self, validated_data):
        user = CustomUser.objects.create_user(**validated_data)
        return user
```

These serializers handle serialization and validation for the custom user  
model. The `UserRegistrationSerializer` includes write-only password field  
to ensure passwords are never returned in API responses. The `create` method  
uses `create_user` to properly hash passwords before storing them.  

## Token blacklisting

Enable token blacklisting to invalidate tokens on logout.  

In `settings.py`:  

```python
INSTALLED_APPS = [
    # ...
    'rest_framework_simplejwt.token_blacklist',
]

SIMPLE_JWT = {
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}
```

Run migrations:  

```bash
python manage.py migrate
```

Token blacklisting allows you to invalidate tokens before they expire,  
essential for logout functionality. When `ROTATE_REFRESH_TOKENS` is enabled,  
using a refresh token generates both a new access token and a new refresh  
token. The old refresh token is then blacklisted, preventing its reuse.  

## Logout endpoint

Implement logout functionality with token blacklisting.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.tokens import RefreshToken

class LogoutView(APIView):
    permission_classes = [IsAuthenticated]
    
    def post(self, request):
        try:
            refresh_token = request.data["refresh"]
            token = RefreshToken(refresh_token)
            token.blacklist()
            
            return Response(status=status.HTTP_205_RESET_CONTENT)
        except Exception as e:
            return Response(status=status.HTTP_400_BAD_REQUEST)
```

This logout endpoint accepts a refresh token and adds it to the blacklist,  
effectively invalidating both the refresh token and its associated access  
tokens. Once blacklisted, the tokens cannot be used to authenticate or  
refresh. This provides true logout functionality in JWT-based systems.  

## Permission classes

Use permission classes with JWT authentication.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class AdminOnlyView(APIView):
    permission_classes = [IsAuthenticated, IsAdminUser]
    
    def get(self, request):
        return Response({'message': 'Admin access granted'})

class ReadOnlyView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        return Response({'data': 'Read-only data'})
```

Permission classes provide fine-grained access control beyond simple  
authentication. `IsAdminUser` restricts access to users with `is_staff=True`.  
You can chain multiple permission classes, and all must pass for the request  
to be authorized. This allows you to implement role-based access control.  

## Custom permission class

Create custom permission for specific user roles.  

In `permissions.py`:  

```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        
        return obj.owner == request.user

class IsPremiumUser(permissions.BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and \
               hasattr(request.user, 'is_premium') and \
               request.user.is_premium
```

Custom permissions allow you to implement complex authorization logic  
specific to your application's needs. The `has_permission` method checks  
view-level permissions, while `has_object_permission` checks permissions for  
specific objects. These can be combined with JWT authentication to create  
sophisticated access control systems.  

## ViewSet with JWT

Use JWT authentication with ViewSets.  

In `views.py`:  

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Article
from .serializers import ArticleSerializer

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

In `urls.py`:  

```python
from rest_framework.routers import DefaultRouter
from .views import ArticleViewSet

router = DefaultRouter()
router.register(r'articles', ArticleViewSet)

urlpatterns = router.urls
```

ViewSets provide a complete CRUD interface with minimal code. The  
`perform_create` method automatically associates created articles with the  
authenticated user from the JWT token. This pattern is common for resources  
that belong to specific users.  

## Filtering by authenticated user

Filter querysets based on the authenticated user.  

In `views.py`:  

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        return Task.objects.filter(user=self.request.user)
    
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

This pattern ensures users can only access their own tasks. The  
`get_queryset` method filters results based on the authenticated user from  
the JWT token. Combined with `perform_create`, this creates a secure,  
user-scoped API where each user's data remains isolated.  

## Token refresh endpoint usage

Implement token refresh in client applications.  

Client-side example (JavaScript):  

```javascript
async function refreshAccessToken(refreshToken) {
    const response = await fetch('http://localhost:8000/api/token/refresh/', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({
            refresh: refreshToken
        })
    });
    
    if (response.ok) {
        const data = await response.json();
        localStorage.setItem('accessToken', data.access);
        return data.access;
    }
    
    throw new Error('Token refresh failed');
}
```

Token refresh is essential for maintaining long-lived sessions without  
requiring users to re-authenticate frequently. Client applications should  
store the refresh token securely and use it to obtain new access tokens when  
the current one expires. This example shows a basic implementation that could  
be integrated into an authentication service.  

## Token verification

Manually verify JWT tokens in views.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework_simplejwt.tokens import AccessToken
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError

class VerifyTokenView(APIView):
    def post(self, request):
        token = request.data.get('token')
        
        try:
            access_token = AccessToken(token)
            user_id = access_token['user_id']
            
            return Response({
                'valid': True,
                'user_id': user_id
            })
        except (InvalidToken, TokenError) as e:
            return Response({
                'valid': False,
                'error': str(e)
            }, status=status.HTTP_400_BAD_REQUEST)
```

Token verification is useful for validating tokens without making  
authenticated requests. This can be used in microservice architectures where  
different services need to verify tokens independently, or in scenarios where  
you need to check token validity before performing expensive operations.  

## Decode token payload

Extract information from JWT token payload.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from rest_framework_simplejwt.tokens import AccessToken

class TokenInfoView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')
        token_string = auth_header.split(' ')[1] if auth_header else None
        
        if token_string:
            token = AccessToken(token_string)
            return Response({
                'token_type': token['token_type'],
                'exp': token['exp'],
                'iat': token['iat'],
                'jti': token['jti'],
                'user_id': token['user_id'],
            })
        
        return Response({'error': 'No token provided'})
```

This view demonstrates how to extract and examine the claims stored in a JWT  
token. The token contains standard claims like `exp` (expiration time), `iat`  
(issued at), and `jti` (JWT ID), as well as custom claims you've added. This  
is useful for debugging or displaying token information to users.  

## API key and JWT combination

Combine API key and JWT authentication.  

In `authentication.py`:  

```python
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed
from django.contrib.auth.models import User

class APIKeyAuthentication(BaseAuthentication):
    def authenticate(self, request):
        api_key = request.META.get('HTTP_X_API_KEY')
        
        if not api_key:
            return None
        
        try:
            user = User.objects.get(profile__api_key=api_key)
            return (user, None)
        except User.DoesNotExist:
            raise AuthenticationFailed('Invalid API key')
```

In `settings.py`:  

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'myapp.authentication.APIKeyAuthentication',
    ),
}
```

Multiple authentication schemes allow flexibility in how clients authenticate.  
DRF tries each authentication class in order, using the first that succeeds.  
This pattern is useful when supporting both user-based JWT authentication and  
service-to-service API key authentication.  

## Throttling with JWT

Implement rate limiting for authenticated users.  

In `settings.py`:  

```python
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
```

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from rest_framework.throttling import UserRateThrottle

class DataView(APIView):
    permission_classes = [IsAuthenticated]
    throttle_classes = [UserRateThrottle]
    
    def get(self, request):
        return Response({'data': 'Rate-limited data'})
```

Rate limiting prevents abuse by restricting the number of requests a user can  
make within a time period. JWT authentication integrates seamlessly with  
DRF's throttling system. The user's identity from the JWT token is used to  
track request counts, allowing for user-specific rate limits.  

## Testing JWT authentication

Write tests for JWT-authenticated endpoints.  

In `tests.py`:  

```python
from django.test import TestCase
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from rest_framework_simplejwt.tokens import RefreshToken

class JWTAuthenticationTest(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        
    def test_obtain_token(self):
        response = self.client.post('/api/token/', {
            'username': 'testuser',
            'password': 'testpass123'
        })
        
        self.assertEqual(response.status_code, 200)
        self.assertIn('access', response.data)
        self.assertIn('refresh', response.data)
    
    def test_access_protected_view(self):
        refresh = RefreshToken.for_user(self.user)
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Bearer {refresh.access_token}'
        )
        
        response = self.client.get('/api/protected/')
        self.assertEqual(response.status_code, 200)
```

Testing JWT authentication ensures your endpoints are properly secured. The  
`APIClient` from DRF provides methods to include JWT tokens in test requests.  
Create tokens programmatically using `RefreshToken.for_user()` and set them  
using `credentials()` method for authenticated test requests.  

## Custom token claims

Add custom claims to JWT tokens.  

In `serializers.py`:  

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        token['username'] = user.username
        token['is_staff'] = user.is_staff
        token['user_role'] = user.profile.role if hasattr(user, 'profile') else 'user'
        
        return token
    
    def validate(self, attrs):
        data = super().validate(attrs)
        
        data['user_id'] = self.user.id
        data['username'] = self.user.username
        
        return data
```

Custom claims allow you to embed application-specific data in tokens. This  
reduces database queries by including frequently accessed user attributes  
directly in the token. However, be mindful of token size and avoid including  
sensitive information. The `validate` method adds extra data to the response  
beyond the token itself.  

## Multiple token types

Implement different token types for different purposes.  

In `tokens.py`:  

```python
from rest_framework_simplejwt.tokens import Token

class EmailVerificationToken(Token):
    token_type = 'email_verification'
    lifetime = timedelta(hours=24)

class PasswordResetToken(Token):
    token_type = 'password_reset'
    lifetime = timedelta(hours=1)
```

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from django.contrib.auth.models import User
from .tokens import EmailVerificationToken

class SendVerificationEmailView(APIView):
    def post(self, request):
        user = User.objects.get(email=request.data['email'])
        token = EmailVerificationToken.for_user(user)
        
        # Send email with token
        verification_url = f"http://example.com/verify/{token}"
        
        return Response({'message': 'Verification email sent'})
```

Custom token types allow you to create specialized tokens with different  
lifetimes and purposes. Email verification tokens might be valid for 24  
hours, while password reset tokens might expire after 1 hour. Each token type  
can have its own validation logic and security requirements.  

## Token sliding

Implement sliding token authentication.  

In `settings.py`:  

```python
SIMPLE_JWT = {
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.SlidingToken',),
    'SLIDING_TOKEN_LIFETIME': timedelta(minutes=5),
    'SLIDING_TOKEN_REFRESH_LIFETIME': timedelta(days=1),
}
```

In `urls.py`:  

```python
from rest_framework_simplejwt.views import (
    TokenObtainSlidingView,
    TokenRefreshSlidingView,
)

urlpatterns = [
    path('api/token/', TokenObtainSlidingView.as_view()),
    path('api/token/refresh/', TokenRefreshSlidingView.as_view()),
]
```

Sliding tokens combine access and refresh functionality into a single token.  
When refreshed, the token's expiration time slides forward rather than  
generating a new token. This approach simplifies token management but offers  
less granular control compared to separate access and refresh tokens.  

## CORS configuration for JWT

Configure CORS for JWT authentication in frontend applications.  

In `settings.py`:  

```python
INSTALLED_APPS = [
    # ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    # ...
]

CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://localhost:8080",
]

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
```

CORS configuration is essential when your frontend application runs on a  
different domain than your Django API. The Authorization header must be  
explicitly allowed for JWT tokens to work. Install `django-cors-headers`  
package and configure it to accept requests from your frontend domains.  

## Custom authentication backend

Create a custom authentication backend for JWT.  

In `authentication.py`:  

```python
from rest_framework_simplejwt.authentication import JWTAuthentication
from rest_framework.exceptions import AuthenticationFailed

class CustomJWTAuthentication(JWTAuthentication):
    def authenticate(self, request):
        result = super().authenticate(request)
        
        if result is not None:
            user, token = result
            
            if not user.is_active:
                raise AuthenticationFailed('User account is disabled.')
            
            if hasattr(user, 'profile') and not user.profile.email_verified:
                raise AuthenticationFailed('Email not verified.')
        
        return result
```

In `settings.py`:  

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'myapp.authentication.CustomJWTAuthentication',
    ),
}
```

Custom authentication backends allow you to add additional validation logic  
beyond standard JWT verification. This example checks if the user's account  
is active and if their email is verified. You can implement any custom logic  
needed for your application's security requirements.  

## Refresh token rotation

Implement refresh token rotation for enhanced security.  

In `settings.py`:  

```python
SIMPLE_JWT = {
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
}
```

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework_simplejwt.exceptions import TokenError

class RotatingRefreshView(APIView):
    def post(self, request):
        try:
            refresh_token = request.data['refresh']
            token = RefreshToken(refresh_token)
            
            new_refresh = token.rotate()
            
            return Response({
                'access': str(token.access_token),
                'refresh': str(new_refresh),
            })
        except TokenError as e:
            return Response(
                {'error': str(e)},
                status=status.HTTP_401_UNAUTHORIZED
            )
```

Token rotation improves security by ensuring each refresh token can only be  
used once. When a refresh token is used, it's blacklisted and a new one is  
issued. This prevents token replay attacks where stolen refresh tokens could  
be used indefinitely. Combined with blacklisting, this provides strong  
protection against token theft.  

## JWT with social authentication

Integrate JWT with social authentication providers.  

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.contrib.auth import authenticate
from rest_framework_simplejwt.tokens import RefreshToken
import requests

class GoogleAuthView(APIView):
    def post(self, request):
        token = request.data.get('token')
        
        # Verify Google token
        google_response = requests.get(
            'https://www.googleapis.com/oauth2/v3/userinfo',
            headers={'Authorization': f'Bearer {token}'}
        )
        
        if google_response.status_code != 200:
            return Response(
                {'error': 'Invalid Google token'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        user_data = google_response.json()
        email = user_data['email']
        
        # Get or create user
        user, created = User.objects.get_or_create(
            email=email,
            defaults={'username': email}
        )
        
        # Generate JWT tokens
        refresh = RefreshToken.for_user(user)
        
        return Response({
            'refresh': str(refresh),
            'access': str(refresh.access_token),
            'created': created
        })
```

Social authentication allows users to sign in using their existing accounts  
from providers like Google, Facebook, or GitHub. After verifying the social  
provider's token, create or retrieve the user and generate JWT tokens for  
your application. This provides a seamless authentication experience while  
maintaining JWT-based session management.  

## WebSocket authentication with JWT

Use JWT tokens for WebSocket authentication.  

In `middleware.py`:  

```python
from channels.middleware import BaseMiddleware
from channels.db import database_sync_to_async
from django.contrib.auth.models import AnonymousUser
from rest_framework_simplejwt.tokens import AccessToken
from django.contrib.auth import get_user_model

User = get_user_model()

class JWTAuthMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        headers = dict(scope['headers'])
        
        if b'authorization' in headers:
            token_name, token_key = headers[b'authorization'].decode().split()
            if token_name == 'Bearer':
                try:
                    access_token = AccessToken(token_key)
                    user = await self.get_user(access_token['user_id'])
                    scope['user'] = user
                except Exception:
                    scope['user'] = AnonymousUser()
        else:
            scope['user'] = AnonymousUser()
        
        return await super().__call__(scope, receive, send)
    
    @database_sync_to_async
    def get_user(self, user_id):
        try:
            return User.objects.get(id=user_id)
        except User.DoesNotExist:
            return AnonymousUser()
```

WebSocket connections require special handling for JWT authentication since  
they're long-lived connections. This middleware extracts the JWT token from  
WebSocket connection headers and authenticates the user. The authenticated  
user is then available throughout the WebSocket connection lifetime.  

## API versioning with JWT

Implement API versioning with JWT authentication.  

In `urls.py`:  

```python
from django.urls import path, include
from rest_framework_simplejwt.views import TokenObtainPairView

urlpatterns = [
    path('api/v1/token/', TokenObtainPairView.as_view()),
    path('api/v1/', include('myapp.urls.v1')),
    path('api/v2/', include('myapp.urls.v2')),
]
```

In `settings.py`:  

```python
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}
```

In `views.py`:  

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class DataViewV1(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        return Response({'version': 'v1', 'data': 'old format'})

class DataViewV2(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        return Response({'version': 'v2', 'data': 'new format'})
```

API versioning allows you to evolve your API while maintaining backward  
compatibility. JWT authentication works seamlessly across different API  
versions, as tokens remain valid regardless of which version endpoint is  
accessed. This enables smooth transitions when updating API functionality.  

## Monitoring and logging JWT usage

Implement logging and monitoring for JWT authentication.  

In `middleware.py`:  

```python
import logging
from django.utils.deprecation import MiddlewareMixin
from rest_framework_simplejwt.tokens import AccessToken

logger = logging.getLogger(__name__)

class JWTLoggingMiddleware(MiddlewareMixin):
    def process_request(self, request):
        auth_header = request.META.get('HTTP_AUTHORIZATION', '')
        
        if auth_header.startswith('Bearer '):
            token_string = auth_header.split(' ')[1]
            try:
                token = AccessToken(token_string)
                user_id = token.get('user_id')
                
                logger.info(f"JWT Auth: User {user_id} accessed {request.path}")
            except Exception as e:
                logger.warning(f"Invalid JWT token attempt: {str(e)}")
        
        return None
```

In `settings.py`:  

```python
MIDDLEWARE = [
    # ...
    'myapp.middleware.JWTLoggingMiddleware',
]

LOGGING = {
    'version': 1,
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': 'jwt_auth.log',
        },
    },
    'loggers': {
        'myapp.middleware': {
            'handlers': ['file'],
            'level': 'INFO',
        },
    },
}
```

Logging JWT authentication events helps track usage patterns, identify  
security issues, and debug authentication problems. This middleware logs  
successful authentications and invalid token attempts. In production,  
consider using structured logging and sending logs to centralized monitoring  
systems for analysis and alerting.
