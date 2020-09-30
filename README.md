# Introduction

JSON Web Token & CSRF Token Authentication between a Django REST Framework API and React

<!-- TOC -->

- [Introduction](#introduction)
  - [Setup](#setup)
  - [Overview](#overview)
  - [Environment Variables](#environment-variables)
    - [env](#env)
- [Backend](#backend)
    - [Backend Dependencies](#backend-dependencies)
  - [Create Django Project](#create-django-project)
    - [main/settings.py](#mainsettingspy)
    - [main/urls.py](#mainurlspy)
  - [Users App](#users-app)
    - [Custom User Model](#custom-user-model)
      - [users/models.py](#usersmodelspy)
      - [users/admin.py](#usersadminpy)
      - [Create Superuser](#create-superuser)
    - [users/serializers.py](#usersserializerspy)
    - [users/utils.py](#usersutilspy)
    - [users/authentication.py](#usersauthenticationpy)
    - [users/urls.py](#usersurlspy)
    - [users/views.py](#usersviewspy)
      - [Imports](#imports)
      - [Register](#register)
      - [Login](#login)
      - [Auth](#auth)
      - [Extend Token](#extend-token)
      - [User Detail](#user-detail)
      - [Logout](#logout)
  - [Conclusion](#conclusion)
    - [Final backend structure](#final-backend-structure)
- [Frontend](#frontend)
  - [Frontend Dependencies](#frontend-dependencies)
  - [Auth Context](#auth-context)
    - [authContext.js](#authcontextjs)
    - [context/types.js](#contexttypesjs)
    - [context/AuthState.js](#contextauthstatejs)
    - [context/authReducer.js](#contextauthreducerjs)
  - [Auth Components](#auth-components)
    - [Register.js](#registerjs)
    - [Login.js](#loginjs)
    - [components/UserDetail.js](#componentsuserdetailjs)

<!-- /TOC -->

## Setup

[Top &#8593;](#introduction)

This project will be using **Django 3.1**, **Django REST Framework 3.1.1** and React via **create-react-app 3.4**.

Create a folder `jwt_drf_react` for the project. Inside create a folder for `backend` and `frontend`.

```bash
~/ $ mkdir jwt_def_react && cd mkdir jwt_def_react
jwt_def_react/ $ mkdir backend frontend
```

## Overview

[Top &#8593;](#introduction)

Authentication will require three items:

1. Django's Cross-Site Request Forgery (CSRF) Cookie

   - Standard Django CSRF cookie
   - Sent with each request

2. JSON Web Token (JWT) Access Token

   - Short exipiration
   - Stored in app's state
   - Used to access protected routes

3. JWT Refresh Token

   - Longer expiration
   - Used to request access tokens
   - Stored in an HTTPOnly cookie
   - Associated with foreign key to a user in the database
   - Deleted from database on logout or exipration

When a user registers or logs in, they will be assigned a refresh token and and access token. When request is made for protected data, the access token will be passed to the server via the `Authorization` HTTP Header.

If the access token is expired when sent, the refresh token is checked for validity. If the refresh token hasn't expired, a new access token is returned and the original request is repeated.

If the token isn't expired and the user with the id of the token's `user_id` is authorized to access the data, the data is returned.

The CSRF cookie will also be attached to each request and its existence and validity will be checked.

## Environment Variables

[Top &#8593;](#introduction)

Create a file called `.env` in the `jwt_drf_react` project folder. This file will be used to define environment variables. You will want to add this file the your `.gitignore` file to keep your secrets secret.

### env

```python
DJANGO_DEBUG = 'True'

# Django secret keys
DJANGO_SECRET_KEY_DEVELOPMENT='xxxxxxxxxx'
DJANGO_SECRET_KEY_PRODUCTION='xxxxxxxxxx'

# Key for encoding user refresh tokens
DJANGO_REFRESH_TOKEN_SECRET='xxxxxxxxxx'
```

Current file structure:

```bash
jwt_def_react/
│   .env
│   .gitignore
├───backend
└───frontend
```

# Backend

[Top &#8593;](#introduction)

Initialize Pipenv and install Django and other required packages.

```bash
backend/ $ pipenv shell
...creating virtualenv...

$ pipenv install django==3.1.1 djangorestframework==3.11.1 django-cors-headers==3.5.0 python-decouple==3.3 pyjwt==1.5.3
```

### Backend Dependencies

[Top &#8593;](#introduction)

- Django - Python framework for building web apps
- Django REST Framework - A powerful and flexible toolkit for building Web APIs.
- Django CORS Headers - A Django App that adds Cross-Origin Resource Sharing (CORS) headers to responses. This allows in-browser requests to your Django application from other origins.
- Python Decouple - Package for managing environment variables
- PyJWT - Python package for using JSON Web Tokens

## Create Django Project

[Top &#8593;](#introduction)

Create the Django project called `main`

```bash
jwt_drf_react/ $ django-admin startproject main .
```

The file tree should should look something like this:

```bash
jwt_drf_react/
│   .env
│   .gitignore
│
├───backend
│   │   manage.py
│   │   Pipfile
│   │   Pipfile.lock
│   │
│   ├───main
│   │       asgi.py
│   │       settings.py
│   │       urls.py
│   │       wsgi.py
│   │       __init__.py
│
└───frontend
```

---

### main/settings.py

[Top &#8593;](#introduction)

With the exception of the `DEBUG` and `SECRET_KEY` settings, all of the following code is meant to be added to the existing Django settings. This is **not** a complete `settings.py` file on its own.

```python
import decouple  # add

DEBUG = decouple.config('DJANGO_DEBUG') # pull DEBUG from .env

# set SECRET_KEY based on value of DEBUG
if DEBUG:
    SECRET_KEY = decouple.config('DJANGO_SECRET_KEY_DEVELOPMENT')
else:
    SECRET_KEY = decouple.config('DJANGO_SECRET_KEY_PRODUCTION')


INSTALLED_APPS = [
    ...

    'rest_framework', # add
    'corsheaders', # add

    'users', # add
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware', # add at the top
    ...
]

# add everything below to the bottom of settings.py

# REST Framework Defaults
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'users.authentication.SafeJWTAuthentication' # custom authentication class
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated'
    ]
}

# Secret for encoding User refresh tokens
REFRESH_TOKEN_SECRET = decouple.config('DJANGO_REFRESH_TOKEN_SECRET')

# Use custom user model for authentication
AUTH_USER_MODEL = 'users.User'

CORS_ALLOW_CREDENTIALS = True  # to accept cookies via axios

CORS_ORIGIN_WHITELIST = [
    'http://localhost:3000',
    'http://localhost:8000',
    # other whitelisted origins
]

CORS_ALLOWED_ORIGINS = [
    'http://localhost:3000',
    'http://localhost:8000',
    # other allowed origins...
]

CSRF_TRUSTED_ORIGINS = [
    'http://localhost:3000',
    'http://localhost:8000',
    # other allowed origins...
]

CORS_ALLOW_HEADERS = [
    'authorization',
    'content-type',
    'refresh_token'
    'x-csrftoken',
    # 'x-xsrf-token' # might not be needed
]

ALLOWED_HOSTS = [
    '127.0.0.1',
    'localhost',
    # other allowed hosts...
]
```

---

### main/urls.py

[Top &#8593;](#introduction)

We need to include the urls for the user app in our main urls. You can create any path you'd like for these urls, just make sure they're consistent when making calls to your API endpoints.

```python
# main/urls.py
from django.contrib import admin
from django.urls import path, include # add

urlpatterns = [
    path('admin/', admin.site.urls),

    # include user app urls
    path('users/', include('users.urls')), # add

    # add login capability to browsable REST framework api, if desired
    path('api-auth/', include('rest_framework.urls')), # add (optional)
]
```

## Users App

[Top &#8593;](#introduction)

Create a Django app called `users`.

```bash
backend/ $ python manage.py startapp users
```

### Custom User Model

[Top &#8593;](#introduction)

We'll be using a custom user model by extending the `AbstractUser` class. This might not be strictly necessary, but it will be required if you want any additional fields to the User model.

We'll also be defining the `RefreshToken` model, which will be used to request new API access tokens later on.

#### users/models.py

```python
from django.db import models
from django.contrib.auth.models import AbstractUser # add
from django.contrib.auth import get_user_model # add

class User(AbstractUser):
    email = models.EmailField(
        'email address',
        unique=True,
        error_messages={ # overwrite default error message for unique constraint
            'unique':"This email has already been registered."
        }
    )

    def __str__(self):
        return self.username

class RefreshToken(models.Model):
    user = models.ForeignKey(get_user_model(), on_delete=models.CASCADE)
    token = models.CharField(max_length=200)

    def __str__(self):
        return f"{self.user}'s refresh token"
```

Make migrations / migrate new models

```bash
backend/ $ python manage.py makemigrations
Migrations for 'users':
  users\migrations\0001_initial.py
    - Create model User
    - Create model RefreshToken

backend/ $ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, users
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0001_initial... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length...
OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying users.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_
  _add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying sessions.0001_initial... OK
```

#### users/admin.py

[Top &#8593;](#introduction)

```python
from django.contrib import admin

from .models import User, RefreshToken

# register models in admin
admin.site.register([User, RefreshToken])
```

#### Create Superuser

[Top &#8593;](#introduction)

```bash
backend/ $ python manage.py createsuperuser
```

Login to the admin panel to ensure your models are showing up.

---

### users/serializers.py

[Top &#8593;](#introduction)

Create a file called `serializers.py` which will contain the Django REST serializers for our custom User model. We'll have to overwrite the `create()` and `update()` methods for the `UserCreateSerializer` class in order to encrypt the user's password string.

```python
# users/serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model

class UserCreateSerializer(serializers.ModelSerializer):

    def create(self, validated_data):
        password = validated_data.get('password', None)
        instance = self.Meta.model(**validated_data)

        # call set_password() to hash the user's password
        if password is not None:
            instance.set_password(password)
        instance.save()
        return instance

    def update(self, instance, validated_data):
        for attr, value in validated_data.items():
            if attr == 'password':
                # call set_password() to hash the user's password
                instance.set_password(value)
            else:
                setattr(instance, attr, value)

        instance.save()
        return instance

    class Meta:
        model = get_user_model()
        fields = ['username', 'email', 'password']


class UserDetailSerializer(serializers.ModelSerializer):

    class Meta:
        model = get_user_model()
        fields = [
            'username',
            'email',
            'id',
            'last_login',
            'date_joined',
        ]
```

---

### users/utils.py

[Top &#8593;](#introduction)

We're going to need a few helper functions for generating JWT access and refresh tokens. These will be defined in `users/utils.py`

```python
# users/utils.py

# users/utils.py
import jwt
import datetime
from django.conf import settings

from users.models import User, RefreshToken

def generate_access_token(user):
    '''Generate JWT access token for the user'''

    access_token_payload = {
        # id from User instance
        'user_id': user.id,
         # expiration date
        'exp': datetime.datetime.utcnow() + datetime.timedelta(days=0, minutes=5),
        # initiated at date
        'iat': datetime.datetime.utcnow(),
        # additional items if desired
        # ...
    }

    access_token = jwt.encode(
        access_token_payload,
        settings.SECRET_KEY,
        algorithm='HS256'
    )

    return access_token


def generate_refresh_token(user):
    '''Generate JWT refresh token for the user'''
    refresh_token_payload = {
        # id from User instance
        'user_id': user.id,
         # expiration date
        'exp': datetime.datetime.utcnow() + datetime.timedelta(days=7),
        # initiated at date
        'iat': datetime.datetime.utcnow(),
        # additional items if desired
        # ...
    }

    # encode payload and decode jwt into a string
    refresh_token = jwt.encode(
        refresh_token_payload,
        settings.REFRESH_TOKEN_SECRET,
        algorithm='HS256'
    ).decode(encoding='utf-8')

    # convert refresh_token bytes object into utf-8 string
    return refresh_token


# Thanks to Ahmed Atalla for this code
# https://dev.to/a_atalla/django-rest-framework-custom-jwt-authentication-5n5
```

---

### users/authentication.py

[Top &#8593;](#introduction)

Now it's time to implement our custom authentication class. This will check the HTTP request for a CSRF Cookie and `Authorization` HTTP Header. Execptions will be raised if the CSRF Cookie is missing or invalid or if the the `Authorization` header is missing or the access token is expired.

```python
import jwt
from rest_framework.authentication import BaseAuthentication
from django.middleware.csrf import CsrfViewMiddleware
from rest_framework import exceptions
from django.conf import settings
from django.contrib.auth import get_user_model
from rest_framework.response import Response
from rest_framework import status


class CSRFCheck(CsrfViewMiddleware):
    def _reject(self, request, reason):
        # Return the failure reason instead of an HttpResponse
        return reason


class SafeJWTAuthentication(BaseAuthentication):
    '''
        custom authentication class for DRF and JWT
        https://github.com/encode/django-rest-framework/blob/master/rest_framework/authentication.py
    '''

    def authenticate(self, request):
        User = get_user_model()
        authorization_heaader = request.headers.get('Authorization')

        if not authorization_heaader:
            return None
        try:
            # header = 'Token xxxxxxxxxxxxxxxxxxxxxxxx'
            access_token = authorization_heaader.split(' ')[1]
            payload = jwt.decode(
                access_token, settings.SECRET_KEY, algorithms=['HS256'])

        # if token is expired
        except jwt.ExpiredSignatureError:
            raise exceptions.AuthenticationFailed(
                detail={
                    'msg': 'Access token expired',
                }
            )
        # if token doesn't exist
        except IndexError:
            raise exceptions.AuthenticationFailed('Token prefix missing')

        # get the user associated with the token
        user = User.objects.filter(id=payload['user_id']).first()
        if user is None:
            raise exceptions.AuthenticationFailed('User not found')

        if not user.is_active:
            raise exceptions.AuthenticationFailed('user is inactive')

        # check CSRF Cookie
        self.enforce_csrf(request)

        # return the authenticated user object
        return (user, None)

    def enforce_csrf(self, request):
        """
        Enforce CSRF validation
        """
        check = CSRFCheck()
        # populates request.META['CSRF_COOKIE'], which is used in process_view()
        check.process_request(request)
        reason = check.process_view(request, None, (), {})
        if reason:
            # CSRF failed, bail with explicit error message
            raise exceptions.PermissionDenied('CSRF Failed: %s' % reason)

# Thanks to Ahmed Atalla for this code
# https://dev.to/a_atalla/django-rest-framework-custom-jwt-authentication-5n5
```

### users/urls.py

[Top &#8593;](#introduction)

Let's create our URLs for our API endpoints. The 'users' prefix was added in `main/urls.py` so our paths in `users/urls.py` will contain everything after `users/`.

- users/
  - POST: Create new users and generate tokens
- users/login/
  - POST: Submit credentials to login user. Generate tokens.
- users/auth/ - protected
  - GET: Get the user object associated with the token
- users/:id/ - protected
  - GET: View details of user with given id
  - POST: Update user details
- users/token/
  - GET: Generate new access tokens if refresh token cookie is valid
- users/logout/
  - GET: Delete the refresh token from database and client cookie

Create `users/urls.py`

```python
# users/urls.py

from django.urls import path

from . import views

urlpatterns = [
    path('', views.register), # create
    path('auth/', views.auth), # login/get logged in user
    path('token/', views.extend_token), # request new access tokens
    path('<int:pk>/', views.user_detail), # read/update
    path('logout/', views.logout), # delete tokens
]
```

### users/views.py

[Top &#8593;](#introduction)

Now we're ready to create our API views. We will use function-based views with Django REST Framework decorators to define our allowed API methods, permission and authentication classes for each endpoint. Since we're not using class-based views, this file is kind of a doozie!

We'll break this file down into sections, by view function.

#### Imports

```python
import jwt

from django.conf import settings
from django.contrib.auth import get_user_model
from django.views.decorators.csrf import ensure_csrf_cookie, csrf_protect

from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
from rest_framework import status
from rest_framework import exceptions
from rest_framework.decorators import (
    api_view, permission_classes,
    authentication_classes
)

from .authentication import SafeJWTAuthentication
from .models import User, RefreshToken
from .serializers import UserCreateSerializer, UserDetailSerializer
from .utils import generate_access_token, generate_refresh_token
```

#### Register

[Top &#8593;](#introduction)

```python
@api_view(['POST'])
@permission_classes([AllowAny])
def register(request):
    '''Validate request POST data and create new User objects in database
    Set refresh cookie and return access token on successful registration'''
    # create response object
    response = Response()

    # serialize request JSON data
    new_user_serializer = UserCreateSerializer(data=request.data)

    if request.data.get('password') != request.data.get('password2'):
        # if password and password2 don't match return status 400
        response.data = {'msg': "Passwords don't match"}
        response.status_code = status.HTTP_400_BAD_REQUEST
        return response

    if new_user_serializer.is_valid():
        # If the data is valid, create the item in the database
        new_user = new_user_serializer.save()

        # generate access and refresh tokens for the new user
        access_token = generate_access_token(new_user)
        refresh_token = generate_refresh_token(new_user)

        # attach the access token to the response data
        # and set the response status code to 201
        response.data = {'accessToken': access_token}
        response.status_code = status.HTTP_201_CREATED

        # create refreshtoken cookie
        response.set_cookie(
            key='refreshtoken',
            value=refresh_token,
            httponly=True,  # to help prevent XSS
            samesite='strict',  # to help prevent XSS
            domain='localhost',  # change in production
            # secure=True # for https connections only
        )

        # return successful response
        return response

    # if serializer is invalid
    response.data = {'msg': 'Incorrect username or password'}
    response.status_code = status.HTTP_400_BAD_REQUEST
    # if the serialized data is NOT valid
    # send a response with error messages and status code 400
    response.data = {
        'error': [msg for msg in new_user_serializer.errors.values()]}
    response.status_code = status.HTTP_400_BAD_REQUEST
    # return failed response
    return response
```

#### Login

[Top &#8593;](#introduction)

```python
@api_view(['POST'])
@permission_classes([AllowAny])
def login(request):
    '''
    POST: Validate User credentials and generate refresh and access tokens
    '''
    # create response object
    response = Response()

    username = request.data.get('username')
    password = request.data.get('password')

    if username is None or password is None:
        response.data = {'msg': 'Username and password required.'}
        response.status_code = status.HTTP_400_BAD_REQUEST
        return response

    user = User.objects.filter(username=username).first()

    if user is None or not user.check_password(password):
        response.data = {
            'msg': 'Incorrect username or password'
        }
        response.status_code = status.HTTP_400_BAD_REQUEST
        return response

    # generate access and refresh tokens for the current user
    access_token = generate_access_token(user)
    refresh_token = generate_refresh_token(user)

    try:
        # if the user has a refresh token in the db,
        # get the old token
        old_refresh_token = RefreshToken.objects.get(user=user.id)
        # delete the old token
        old_refresh_token.delete()
        # generate new token
        RefreshToken.objects.create(user=user, token=refresh_token)

    except RefreshToken.DoesNotExist:
        # assign a new refresh token to the current user
        RefreshToken.objects.create(user=user, token=refresh_token)

    # create refreshtoken cookie
    response.set_cookie(
        key='refreshtoken',  # cookie name
        value=refresh_token,  # cookie value
        httponly=True,  # to help prevent XSS
        samesite='strict',  # to help prevent XSS
        domain='localhost',  # change in production
        # secure=True # for https connections only
    )

    # return the access token in the reponse
    response.data = {
        'accessToken': access_token
    }
    response.status_code = status.HTTP_200_OK
    return response
```

#### Auth

[Top &#8593;](#introduction)

```python
@api_view(['GET'])
@permission_classes([IsAuthenticated])
@authentication_classes([SafeJWTAuthentication])
def auth(request):
    '''Return the user data for the user id contained in a valid access token'''
    # create response object
    response = Response()

    # Get the access token from headers
    access_token = request.headers.get('Authorization').split(' ')[1]

    # decode access token payload
    payload = jwt.decode(
        access_token,
        settings.SECRET_KEY,
        algorithms=['HS256']
    )

    # get the user with the same id as the token's user_id
    user = User.objects.filter(id=payload.get('user_id')).first()

    if user is None:
        response.data = {'msg': 'User not found'}
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    if not user.is_active:
        response.data = {'msg': 'User not active'}
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    # serialize the User object and attach to response data
    serialized_user = UserDetailSerializer(instance=user)
    response.data = {'user': serialized_user.data}

    return response
```

#### Extend Token

[Top &#8593;](#introduction)

```python
@api_view(['GET'])
@permission_classes([])
def extend_token(request):
    '''Return new access token if request's refresh token cookie is valid'''
    # create response object
    response = Response()

    # get the refresh token cookie
    refresh_token = request.COOKIES.get('refreshtoken')

    # if the refresh token doesn't exist
    # return 401 - Unauthorized
    if refresh_token is None:
        response.data = {
            'msg': 'Authentication credentials were not provided'
        }

        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    # if a token is found,
    # try to decode it
    try:
        payload = jwt.decode(
            refresh_token,
            settings.REFRESH_TOKEN_SECRET,
            algorithms=['HS256']
        )

    # if the token is expired, delete it from the database
    # return 401 Unauthorized
    except jwt.ExpiredSignatureError:
        # find the expired token in the database
        expired_token = RefreshToken.objects.filter(
            token=refresh_token).first()

        # delete the old token
        expired_token.delete()

        response.data = {
            'error': 'Expired refresh token, please log in again.'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED

        # remove exipred refresh token cookie
        response.delete_cookie('refreshtoken')
        return response

    # if the token is valid,
    # get the user asscoiated with token
    user = User.objects.filter(id=payload.get('user_id')).first()
    if user is None:
        response.data = {
            'msg': 'User not found'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    if not user.is_active:
        response.data = {
            'msg': 'User is inactive'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    # generate new refresh token for the user
    new_refresh_token = generate_refresh_token(user)

    # Delete old refresh token
    # if the user has a refresh token in the db,
    # get the old token
    old_refresh_token = RefreshToken.objects.filter(user=user.id).first()
    if old_refresh_token:
        # delete the old token
        old_refresh_token.delete()

    # assign a new refresh token to the current user
    RefreshToken.objects.create(user=user, token=new_refresh_token)

    # change refreshtoken cookie
    response.set_cookie(
        key='refreshtoken',  # cookie name
        value=new_refresh_token,  # cookie value
        httponly=True,  # to help prevent XSS attacks
        samesite='strict',  # to help prevent XSS attacks
        domain='localhost',  # change in production
        # secure=True # for https connections only
    )

    # generate new access token for the user
    new_access_token = generate_access_token(user)

    response.data = {'accessToken': new_access_token}
    return response
```

#### User Detail

Methods:

- GET - Get the user object associated with the access token
- PUT - Update the user info for the user associateed with the access token

[Top &#8593;](#introduction)

```python
@api_view(['GET','PUT'])
@permission_classes([IsAuthenticated])
@authentication_classes([SafeJWTAuthentication])
@ensure_csrf_cookie
def user_detail(request, pk):
    '''
    GET: Get the user data associated with the pk
    POST: Update the user data associated with the pk
    '''
    # Create response object
    response = Response()

    # find the user associated with
    # the pk passed in the url
    user = User.objects.filter(pk=pk).first()

    if user is None:
        response.data = {
            'msg': 'User not found'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED

        return response

    if not user.is_active:
        response.data = {
            'msg': 'User is inactive'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    # Get the access token from request headers
    access_token = request.headers.get('Authorization').split(' ')[1]

    # decode token payload
    payload = jwt.decode(
        access_token,
        settings.SECRET_KEY,
        algorithms=['HS256']
    )

    # reject the request if the requested
    # pk is not the owner of the token
    if pk != payload.get('user_id'):
        response.data = {
            'msg':'Not authorized'
        }
        response.status_code = status.HTTP_401_UNAUTHORIZED
        return response

    # GET = view user details
    if request.method == 'GET':
        serialized_user = UserDetailSerializer(instance=user)

        response.data = {'user': serialized_user.data}
        response.status_code = status.HTTP_200_OK

        return response

    # PUT = update user info
    if request.method == 'PUT':

        # partial = True will allow for User fields to be missing
        serialized_user = UserCreateSerializer(data=request.data, partial=True)

        if serialized_user.is_valid():
            # combine updated with the current user instance and serialize
            serialized_user.update(instance=user, validated_data=serialized_user.validated_data)
            response.data = {'msg': 'Account info updated successffully'}
            response.status_code = status.HTTP_202_ACCEPTED
            return response

        response.data = {'error':serialized_user.errors}
        response.status_code = status.HTTP_400_BAD_REQUEST
        return response
```

#### Logout

[Top &#8593;](#introduction)

```python
@api_view(['GET'])
def logout(request):
    '''Delete refresh token from the database
    and delete the refreshtoken cookie'''
    # Create response object
    response = Response()

    # find the logged in user's refresh token
    refresh_token = RefreshToken.objects.filter(user=request.user.id).first()

    if refresh_token is None:
        response.data = {'msg':'Unauthorized'}
        response.status_code = status.HTTP_400_BAD_REQUEST
        return response

    # if the token is found, delete it
    refresh_token.delete()

    # remove the refreshtoken and csrftoken cookies
    response.delete_cookie('refreshtoken')
    response.delete_cookie('csrftoken')

    response.data = {
        'msg': 'Logout successful. See you next time!'
    }

    return response
```

## Conclusion

[Top &#8593;](#introduction)

That should do it for the backend. These views can now be tested using Python's requests module, Curl, or Postman. We could also write tests to ping each endpoint with valid and invalid tokens. Token expiration times can be shortened to test responses with expired tokens.

### Final backend structure

```bash
backend/
│   db.sqlite3
│   manage.py
│   Pipfile
│   Pipfile.lock
│
├───main
│       asgi.py
│       settings.py
│       urls.py
│       wsgi.py
│       __init__.py
│
└───users
    │   admin.py
    │   apps.py
    │   authentication.py
    │   models.py
    │   serializers.py
    │   urls.py
    │   utils.py
    │   views.py
    │   __init__.py
    │
    └───migrations
            0001_initial.py
            __init__.py
```

# Frontend

[Top &#8593;](#introduction)

Navigate into the `frontend` directory inside the main `jwt_drf_react` project folder. From here we'll use create-react-app to start our front end React project.

**Disclaimer**: Admittedly, as of 9/27/2020 as this is written, I am still learning React. Therefore, my knowledge of the inner workings of the React app may have some sizeable holes as to the "why" of certain elements of this project and there may very well be far better ways to accomplish things. I'll do my best to update the guide as I gain more understanding.

```bash
jwt_drf_react/ $ cd frontend && npx create-react-app .
```

Notice the . in place of an app name.

File structure after deleting some of the React boilerplate files:

```bash
frontend/
│    node_modules/
│    package.json
│    package-lock.json
├─── public
│        index.html
│        robots.txt
├─── src
│        App.css
│        App.js
│        index.js
```

Most of the changes we'll be making will be within the `src/` folder.

## Frontend Dependencies

- Axios
  - for HTTP requests
- react-router-dom
  - for routing between React components
- bootstrap
  - layout/styling

Install frontend dependencies

```bash
frontend/ $ npm install axios react-router-dom bootstrap
```

## Auth Context

[Top &#8593;](#introduction)

This app will utilize function-based components, React's Context API and React Hooks to manage state.

First we'll create a directory called `context` to store our auth context files.

Inside we'll create a file called `types.js` and another directory called `auth`.

Inside the `context/auth/` we'll create three files

- `AuthState.js`
- `authContext.js`
- `authReducer.js`

The `src/` directory should now look like this:

```bash
src/
│   App.css
│   App.js
│   index.css
│   index.js
│
└───context
    │   types.js
    │
    └───auth
            authContext.js
            authReducer.js
            AuthState.js
```

### authContext.js

[Top &#8593;](#introduction)

```javascript
import { createContext } from 'react';

const AuthContext = createContext();

export default AuthContext;
```

### context/types.js

[Top &#8593;](#introduction)

This file contains the different ways we'll be changing the auth state. They will be used by the Auth Reducer to return updated data to state.

```javascript
// context/types.js
```

### context/AuthState.js

[Top &#8593;](#introduction)

State and methods for authenitcating users

```javascript
// context/AuthState.js
```

### context/authReducer.js

[Top &#8593;](#introduction)

The Reducer will handle changes to state. Actions from `types.js` will be dispatched to the reducer along with a payload for each action. The payload state will be updated with the data in the payload.

```javascript
// context/authReducer.js
```

## Auth Components

We'll need a folder for components inside the `src` folder.

Inside we'll be create the following components:

- `Home.js`
  - Splash page for redirect after login
- `Navbar.js`
  - Navigating between components
- `Register.js`
  - Form for creating users
- `Login.js`
  - Form for logging in users
- `UserDetail.js`
  - View / edit user details

The file structure of these files is completely up to the needs of your project. The user detail component would probably be looped in with other user CRUD components.

### Register.js

[Top &#8593;](#introduction)

```javascript
// components/Register.js
```

### Login.js

[Top &#8593;](#introduction)

```javascript
// components/Login.js
```

### components/UserDetail.js

[Top &#8593;](#introduction)

```javascript
// components/UserDetail.js
```
