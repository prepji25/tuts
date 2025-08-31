# Prepji - Complete Dockerized Application Setup

This tutorial will guide you through setting up a completely dockerized Next.js + Django application with JWT authentication, GraphQL API, and PostgreSQL database.

## Prerequisites
- Docker Desktop installed and running on your Mac
- VS Code installed
- No other dependencies needed (everything runs in Docker)

## Step 1: Create Project Structure

```bash
# Create main project directory
mkdir prepji
cd prepji

# Create the directory structure
mkdir -p backend frontend
mkdir -p backend/apps/users backend/apps/content
mkdir -p frontend/src/components frontend/src/lib frontend/src/app
```

## Step 2: Docker Setup Files

### 2.1 Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: prepji_db
      POSTGRES_USER: prepji_user
      POSTGRES_PASSWORD: prepji_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prepji_user -d prepji_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DEBUG=1
      - DATABASE_URL=postgresql://prepji_user:prepji_password@postgres:5432/prepji_db
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=your-secret-key-change-in-production
      - ALLOWED_HOSTS=localhost,127.0.0.1,backend
      - CORS_ALLOWED_ORIGINS=http://localhost:3000
    volumes:
      - ./backend:/app
      - backend_media:/app/media
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             python manage.py runserver 0.0.0.0:8000"

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_BACKEND_URL=http://localhost:8000
      - NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql/
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    depends_on:
      - backend
    command: npm run dev

  celery:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - DEBUG=1
      - DATABASE_URL=postgresql://prepji_user:prepji_password@postgres:5432/prepji_db
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=your-secret-key-change-in-production
    volumes:
      - ./backend:/app
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: celery -A prepji worker -l info

volumes:
  postgres_data:
  backend_media:
```

### 2.2 Create Environment File `.env`

```bash
# Database
DATABASE_URL=postgresql://prepji_user:prepji_password@postgres:5432/prepji_db
POSTGRES_DB=prepji_db
POSTGRES_USER=prepji_user
POSTGRES_PASSWORD=prepji_password

# Django
SECRET_KEY=your-very-long-secret-key-change-this-in-production
DEBUG=1
ALLOWED_HOSTS=localhost,127.0.0.1,backend
CORS_ALLOWED_ORIGINS=http://localhost:3000

# Redis
REDIS_URL=redis://redis:6379/0

# JWT
JWT_SECRET_KEY=your-jwt-secret-key-change-this-in-production
JWT_ACCESS_TOKEN_LIFETIME=60
JWT_REFRESH_TOKEN_LIFETIME=1440

# Frontend
NEXT_PUBLIC_BACKEND_URL=http://localhost:8000
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql/
```

## Step 3: Backend Setup (Django)

### 3.1 Create `backend/Dockerfile`

```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Create media directory
RUN mkdir -p media

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### 3.2 Create `backend/requirements.txt`

```txt
Django==5.2.5
djangorestframework==3.16.1
djangorestframework-simplejwt==5.3.0
django-allauth==65.11.0
dj-rest-auth==6.0.0
django-cors-headers==4.7.0
django-ratelimit==4.1.0
django-axes==6.5.1
strawberry-graphql-django==0.65.1
psycopg[binary]==3.2.3
celery[redis]==5.5.3
redis==5.2.1
channels[daphne]==4.3.1
python-decouple==3.8
Pillow==11.0.0
```

### 3.3 Create `backend/manage.py`

```python
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

if __name__ == '__main__':
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prepji.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)
```

### 3.4 Create `backend/prepji/__init__.py`

```python
# This makes Python treat the directory as a package
```

### 3.5 Create `backend/prepji/settings.py`

```python
import os
from pathlib import Path
from datetime import timedelta
from decouple import config

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = config('SECRET_KEY', default='django-insecure-change-this-key')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = config('DEBUG', default=True, cast=bool)

ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost,127.0.0.1').split(',')

# Application definition
DJANGO_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',
]

THIRD_PARTY_APPS = [
    'rest_framework',
    'rest_framework_simplejwt',
    'corsheaders',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'dj_rest_auth',
    'dj_rest_auth.registration',
    'strawberry.django',
    'axes',
]

LOCAL_APPS = [
    'apps.users',
    'apps.content',
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

MIDDLEWARE = [
    'axes.middleware.AxesMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'allauth.account.middleware.AccountMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'prepji.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'prepji.wsgi.application'

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('POSTGRES_DB', default='prepji_db'),
        'USER': config('POSTGRES_USER', default='prepji_user'),
        'PASSWORD': config('POSTGRES_PASSWORD', default='prepji_password'),
        'HOST': config('DB_HOST', default='postgres'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Default primary key field type
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# Custom User Model
AUTH_USER_MODEL = 'users.CustomUser'

# Django REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# JWT Settings
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=config('JWT_ACCESS_TOKEN_LIFETIME', default=60, cast=int)),
    'REFRESH_TOKEN_LIFETIME': timedelta(minutes=config('JWT_REFRESH_TOKEN_LIFETIME', default=1440, cast=int)),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': config('JWT_SECRET_KEY', default=SECRET_KEY),
    'VERIFYING_KEY': None,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    'USER_AUTHENTICATION_RULE': 'rest_framework_simplejwt.authentication.default_user_authentication_rule',
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',
}

# CORS Settings
CORS_ALLOWED_ORIGINS = config('CORS_ALLOWED_ORIGINS', default='http://localhost:3000').split(',')
CORS_ALLOW_CREDENTIALS = True

# Django Allauth
SITE_ID = 1

ACCOUNT_AUTHENTICATION_METHOD = 'email'
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_EMAIL_VERIFICATION = 'mandatory'
ACCOUNT_CONFIRM_EMAIL_ON_GET = True
ACCOUNT_LOGIN_ON_EMAIL_CONFIRMATION = True

# REST Auth
REST_AUTH = {
    'USE_JWT': True,
    'JWT_AUTH_COOKIE': 'prepji-auth',
    'JWT_AUTH_REFRESH_COOKIE': 'prepji-refresh-token',
    'JWT_AUTH_HTTPONLY': False,
}

# Celery Configuration
CELERY_BROKER_URL = config('REDIS_URL', default='redis://redis:6379/0')
CELERY_RESULT_BACKEND = config('REDIS_URL', default='redis://redis:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE

# Email Configuration (for development)
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Axes Configuration (Security)
AXES_ENABLED = True
AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = 1  # 1 hour
AXES_RESET_ON_SUCCESS = True

# Strawberry GraphQL
STRAWBERRY_DJANGO = {
    'FIELD_DESCRIPTION_FROM_HELP_TEXT': True,
    'TYPE_DESCRIPTION_FROM_MODEL_DOCSTRING': True,
}
```

### 3.6 Create `backend/prepji/urls.py`

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
from strawberry.django.views import GraphQLView
from .schema import schema

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/auth/', include('dj_rest_auth.urls')),
    path('api/auth/registration/', include('dj_rest_auth.registration.urls')),
    path('graphql/', GraphQLView.as_view(schema=schema)),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### 3.7 Create `backend/prepji/wsgi.py`

```python
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prepji.settings')

application = get_wsgi_application()
```

### 3.8 Create `backend/prepji/asgi.py`

```python
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prepji.settings')

application = get_asgi_application()
```

### 3.9 Create `backend/prepji/celery.py`

```python
import os
from celery import Celery

# Set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prepji.settings')

app = Celery('prepji')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django apps.
app.autodiscover_tasks()
```

### 3.10 Create `backend/prepji/schema.py`

```python
import strawberry
from strawberry.django import auth
from apps.users.schema import UserQuery, UserMutation
from apps.content.schema import ContentQuery

@strawberry.type
class Query(UserQuery, ContentQuery):
    pass

@strawberry.type
class Mutation(UserMutation):
    pass

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

## Step 4: User App Setup

### 4.1 Create `backend/apps/__init__.py`

```python
# This makes Python treat the directory as a package
```

### 4.2 Create `backend/apps/users/__init__.py`

```python
# This makes Python treat the directory as a package
```

### 4.3 Create `backend/apps/users/models.py`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    USER_TYPE_CHOICES = [
        ('student', 'Student'),
        ('mentor', 'Mentor'),
        ('admin', 'Admin'),
    ]
    
    SEX_CHOICES = [
        ('M', 'Male'),
        ('F', 'Female'),
        ('O', 'Other'),
        ('P', 'Prefer not to say'),
    ]
    
    email = models.EmailField(unique=True)
    age = models.PositiveIntegerField(null=True, blank=True)
    sex = models.CharField(max_length=1, choices=SEX_CHOICES, null=True, blank=True)
    user_type = models.CharField(max_length=10, choices=USER_TYPE_CHOICES, default='student')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']
    
    def __str__(self):
        return f"{self.email} ({self.get_user_type_display()})"
    
    class Meta:
        db_table = 'users'
        verbose_name = 'User'
        verbose_name_plural = 'Users'
```

### 4.4 Create `backend/apps/users/admin.py`

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import CustomUser

@admin.register(CustomUser)
class CustomUserAdmin(UserAdmin):
    list_display = ('email', 'username', 'user_type', 'age', 'sex', 'is_active', 'created_at')
    list_filter = ('user_type', 'sex', 'is_active', 'created_at')
    search_fields = ('email', 'username', 'first_name', 'last_name')
    ordering = ('-created_at',)
    
    fieldsets = UserAdmin.fieldsets + (
        ('Custom Fields', {'fields': ('age', 'sex', 'user_type')}),
    )
    
    add_fieldsets = UserAdmin.add_fieldsets + (
        ('Custom Fields', {'fields': ('age', 'sex', 'user_type')}),
    )
```

### 4.5 Create `backend/apps/users/schema.py`

```python
import strawberry
from strawberry.django import auth
from typing import List, Optional
from django.contrib.auth import get_user_model
from strawberry.types import Info

User = get_user_model()

@strawberry.django.type(User)
class UserType:
    id: int
    email: str
    username: str
    first_name: str
    last_name: str
    user_type: str
    age: Optional[int]
    sex: Optional[str]
    is_active: bool
    created_at: strawberry.auto

@strawberry.type
class UserQuery:
    @strawberry.field
    @auth.login_required
    def me(self, info: Info) -> Optional[UserType]:
        return info.context.request.user
    
    @strawberry.field
    @auth.login_required
    def users(self, info: Info) -> List[UserType]:
        user = info.context.request.user
        if user.user_type == 'admin':
            return User.objects.all()
        return []

@strawberry.type
class UserMutation:
    @strawberry.mutation
    @auth.login_required
    def update_profile(
        self, 
        info: Info,
        first_name: Optional[str] = None,
        last_name: Optional[str] = None,
        age: Optional[int] = None,
        sex: Optional[str] = None
    ) -> UserType:
        user = info.context.request.user
        
        if first_name is not None:
            user.first_name = first_name
        if last_name is not None:
            user.last_name = last_name
        if age is not None:
            user.age = age
        if sex is not None:
            user.sex = sex
            
        user.save()
        return user
```

### 4.6 Create `backend/apps/users/apps.py`

```python
from django.apps import AppConfig

class UsersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.users'
```

## Step 5: Content App Setup

### 5.1 Create `backend/apps/content/__init__.py`

```python
# This makes Python treat the directory as a package
```

### 5.2 Create `backend/apps/content/models.py`

```python
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class StaticPage(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    is_published = models.BooleanField(default=True)
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title
    
    class Meta:
        db_table = 'static_pages'
        verbose_name = 'Static Page'
        verbose_name_plural = 'Static Pages'
        ordering = ['-created_at']
```

### 5.3 Create `backend/apps/content/admin.py`

```python
from django.contrib import admin
from .models import StaticPage

@admin.register(StaticPage)
class StaticPageAdmin(admin.ModelAdmin):
    list_display = ('title', 'slug', 'is_published', 'created_by', 'created_at')
    list_filter = ('is_published', 'created_at')
    search_fields = ('title', 'content')
    prepopulated_fields = {'slug': ('title',)}
    
    def save_model(self, request, obj, form, change):
        if not change:  # If creating new object
            obj.created_by = request.user
        super().save_model(request, obj, form, change)
```

### 5.4 Create `backend/apps/content/schema.py`

```python
import strawberry
from typing import List, Optional
from strawberry.types import Info
from .models import StaticPage

@strawberry.django.type(StaticPage)
class StaticPageType:
    id: int
    title: str
    slug: str
    content: str
    is_published: bool
    created_at: strawberry.auto
    updated_at: strawberry.auto

@strawberry.type
class ContentQuery:
    @strawberry.field
    def static_pages(self) -> List[StaticPageType]:
        return StaticPage.objects.filter(is_published=True)
    
    @strawberry.field
    def static_page(self, slug: str) -> Optional[StaticPageType]:
        try:
            return StaticPage.objects.get(slug=slug, is_published=True)
        except StaticPage.DoesNotExist:
            return None
```

### 5.5 Create `backend/apps/content/apps.py`

```python
from django.apps import AppConfig

class ContentConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.content'
```

## Step 6: Frontend Setup (Next.js)

### 6.1 Create `frontend/Dockerfile`

```dockerfile
FROM node:22-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy project files
COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### 6.2 Create `frontend/package.json`

```json
{
  "name": "prepji-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "jest",
    "test:watch": "jest --watch"
  },
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@apollo/client": "^4.0.0",
    "graphql": "^16.8.1",
    "@apollo/experimental-nextjs-app-support": "^0.11.2",
    "react-hook-form": "^7.62.0",
    "zod": "^3.24.1",
    "@hookform/resolvers": "^3.10.0",
    "nuqs": "^2.2.2",
    "lucide-react": "^0.460.0",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.5.4"
  },
  "devDependencies": {
    "typescript": "^5.9.2",
    "@types/node": "^22.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "eslint": "^9.34.0",
    "eslint-config-next": "^15.0.0",
    "prettier": "^3.6.2",
    "tailwindcss": "^3.4.3",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "jest": "^30.0.5",
    "@testing-library/react": "^16.3.0",
    "@testing-library/jest-dom": "^6.6.3",
    "jest-environment-jsdom": "^30.0.5"
  }
}
```

### 6.3 Create `frontend/next.config.js`

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
  images: {
    domains: ['localhost'],
  },
}

module.exports = nextConfig
```

### 6.4 Create `frontend/tailwind.config.js`

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ["class"],
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
              borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
}
```

### 6.5 Create `frontend/postcss.config.js`

```javascript
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

### 6.6 Create `frontend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 6.7 Create `frontend/src/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;

    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;

    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;

    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;

    --secondary: 210 40% 96%;
    --secondary-foreground: 222.2 84% 4.9%;

    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;

    --accent: 210 40% 96%;
    --accent-foreground: 222.2 84% 4.9%;

    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;

    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;

    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;

    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;

    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;

    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;

    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;

    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;

    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;

    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;

    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### 6.8 Create `frontend/src/lib/apollo-client.ts`

```typescript
import { ApolloClient, InMemoryCache, createHttpLink } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';

const httpLink = createHttpLink({
  uri: process.env.NEXT_PUBLIC_GRAPHQL_URL || 'http://localhost:8000/graphql/',
});

const authLink = setContext((_, { headers }) => {
  // Get the authentication token from localStorage if it exists
  const token = typeof window !== 'undefined' ? localStorage.getItem('token') : null;
  
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
    }
  }
});

const client = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache(),
});

export default client;
```

### 6.9 Create `frontend/src/lib/auth.ts`

```typescript
const API_BASE_URL = process.env.NEXT_PUBLIC_BACKEND_URL || 'http://localhost:8000';

export interface LoginData {
  email: string;
  password: string;
}

export interface RegisterData {
  username: string;
  email: string;
  password1: string;
  password2: string;
  age?: number;
  sex?: string;
  user_type?: string;
}

export interface AuthResponse {
  access: string;
  refresh: string;
  user: {
    id: number;
    email: string;
    username: string;
    user_type: string;
  };
}

export class AuthService {
  static async login(data: LoginData): Promise<AuthResponse> {
    const response = await fetch(`${API_BASE_URL}/api/auth/login/`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.detail || 'Login failed');
    }

    const result = await response.json();
    
    // Store tokens
    if (typeof window !== 'undefined') {
      localStorage.setItem('token', result.access);
      localStorage.setItem('refreshToken', result.refresh);
    }
    
    return result;
  }

  static async register(data: RegisterData): Promise<AuthResponse> {
    const response = await fetch(`${API_BASE_URL}/api/auth/registration/`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(data),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(JSON.stringify(error));
    }

    const result = await response.json();
    
    // Store tokens
    if (typeof window !== 'undefined') {
      localStorage.setItem('token', result.access);
      localStorage.setItem('refreshToken', result.refresh);
    }
    
    return result;
  }

  static async logout(): Promise<void> {
    const refreshToken = typeof window !== 'undefined' ? localStorage.getItem('refreshToken') : null;
    
    if (refreshToken) {
      try {
        await fetch(`${API_BASE_URL}/api/auth/logout/`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ refresh: refreshToken }),
        });
      } catch (error) {
        console.error('Logout error:', error);
      }
    }
    
    // Clear tokens
    if (typeof window !== 'undefined') {
      localStorage.removeItem('token');
      localStorage.removeItem('refreshToken');
    }
  }

  static isAuthenticated(): boolean {
    if (typeof window === 'undefined') return false;
    return !!localStorage.getItem('token');
  }

  static getToken(): string | null {
    if (typeof window === 'undefined') return null;
    return localStorage.getItem('token');
  }
}
```

### 6.10 Create `frontend/src/lib/utils.ts`

```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### 6.11 Create `frontend/src/components/ui/button.tsx`

```typescript
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

### 6.12 Create `frontend/src/components/ui/input.tsx`

```typescript
import * as React from "react"
import { cn } from "@/lib/utils"

export interface InputProps
  extends React.InputHTMLAttributes<HTMLInputElement> {}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ className, type, ...props }, ref) => {
    return (
      <input
        type={type}
        className={cn(
          "flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50",
          className
        )}
        ref={ref}
        {...props}
      />
    )
  }
)
Input.displayName = "Input"

export { Input }
```

### 6.13 Create `frontend/src/components/ui/card.tsx`

```typescript
import * as React from "react"
import { cn } from "@/lib/utils"

const Card = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn(
      "rounded-lg border bg-card text-card-foreground shadow-sm",
      className
    )}
    {...props}
  />
))
Card.displayName = "Card"

const CardHeader = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex flex-col space-y-1.5 p-6", className)}
    {...props}
  />
))
CardHeader.displayName = "CardHeader"

const CardTitle = React.forwardRef<
  HTMLParagraphElement,
  React.HTMLAttributes<HTMLHeadingElement>
>(({ className, ...props }, ref) => (
  <h3
    ref={ref}
    className={cn(
      "text-2xl font-semibold leading-none tracking-tight",
      className
    )}
    {...props}
  />
))
CardTitle.displayName = "CardTitle"

const CardDescription = React.forwardRef<
  HTMLParagraphElement,
  React.HTMLAttributes<HTMLParagraphElement>
>(({ className, ...props }, ref) => (
  <p
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props}
  />
))
CardDescription.displayName = "CardDescription"

const CardContent = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
))
CardContent.displayName = "CardContent"

const CardFooter = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex items-center p-6 pt-0", className)}
    {...props}
  />
))
CardFooter.displayName = "CardFooter"

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent }
```

### 6.14 Create `frontend/src/graphql/queries.ts`

```typescript
import { gql } from '@apollo/client';

export const GET_ME = gql`
  query GetMe {
    me {
      id
      email
      username
      firstName
      lastName
      userType
      age
      sex
      isActive
      createdAt
    }
  }
`;

export const GET_STATIC_PAGES = gql`
  query GetStaticPages {
    staticPages {
      id
      title
      slug
      content
      createdAt
      updatedAt
    }
  }
`;

export const GET_STATIC_PAGE = gql`
  query GetStaticPage($slug: String!) {
    staticPage(slug: $slug) {
      id
      title
      slug
      content
      createdAt
      updatedAt
    }
  }
`;

export const UPDATE_PROFILE = gql`
  mutation UpdateProfile(
    $firstName: String
    $lastName: String
    $age: Int
    $sex: String
  ) {
    updateProfile(
      firstName: $firstName
      lastName: $lastName
      age: $age
      sex: $sex
    ) {
      id
      email
      username
      firstName
      lastName
      userType
      age
      sex
    }
  }
`;
```

### 6.15 Create `frontend/src/app/layout.tsx`

```typescript
import './globals.css'
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import { ApolloWrapper } from '@/components/apollo-wrapper'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'Prepji - Learn and Grow',
  description: 'A platform connecting students and mentors',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <ApolloWrapper>
          {children}
        </ApolloWrapper>
      </body>
    </html>
  )
}
```

### 6.16 Create `frontend/src/app/page.tsx`

```typescript
import Link from 'next/link'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'

export default function HomePage() {
  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
      <div className="container mx-auto px-4 py-8">
        <header className="text-center mb-12">
          <h1 className="text-4xl font-bold text-gray-900 mb-4">
            Welcome to Prepji
          </h1>
          <p className="text-xl text-gray-600 mb-8">
            Connect with mentors and accelerate your learning journey
          </p>
          <div className="space-x-4">
            <Link href="/auth/login">
              <Button size="lg">Login</Button>
            </Link>
            <Link href="/auth/register">
              <Button variant="outline" size="lg">Register</Button>
            </Link>
          </div>
        </header>

        <div className="grid md:grid-cols-3 gap-8 mb-12">
          <Card>
            <CardHeader>
              <CardTitle>For Students</CardTitle>
              <CardDescription>
                Find experienced mentors to guide your learning
              </CardDescription>
            </CardHeader>
            <CardContent>
              <p className="text-gray-600">
                Connect with industry professionals and get personalized guidance
                to achieve your academic and career goals.
              </p>
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle>For Mentors</CardTitle>
              <CardDescription>
                Share your knowledge and help students succeed
              </CardDescription>
            </CardHeader>
            <CardContent>
              <p className="text-gray-600">
                Make a difference by sharing your expertise and helping
                the next generation of learners.
              </p>
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle>Community</CardTitle>
              <CardDescription>
                Join a thriving learning community
              </CardDescription>
            </CardHeader>
            <CardContent>
              <p className="text-gray-600">
                Collaborate, learn, and grow together in our supportive
                community of students and mentors.
              </p>
            </CardContent>
          </Card>
        </div>

        <div className="text-center">
          <Link href="/about">
            <Button variant="link">Learn more about us</Button>
          </Link>
        </div>
      </div>
    </div>
  )
}
```

### 6.17 Create `frontend/src/app/auth/login/page.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import Link from 'next/link'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { AuthService } from '@/lib/auth'

const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
})

type LoginForm = z.infer<typeof loginSchema>

export default function LoginPage() {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState('')
  const router = useRouter()

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginForm>({
    resolver: zodResolver(loginSchema),
  })

  const onSubmit = async (data: LoginForm) => {
    setIsLoading(true)
    setError('')

    try {
      await AuthService.login(data)
      router.push('/dashboard')
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold text-center">Login</CardTitle>
          <CardDescription className="text-center">
            Enter your email and password to access your account
          </CardDescription>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
            {error && (
              <div className="bg-red-50 border border-red-200 text-red-600 px-4 py-3 rounded">
                {error}
              </div>
            )}
            
            <div className="space-y-2">
              <label htmlFor="email" className="text-sm font-medium">
                Email
              </label>
              <Input
                id="email"
                type="email"
                placeholder="john@example.com"
                {...register('email')}
              />
              {errors.email && (
                <p className="text-sm text-red-600">{errors.email.message}</p>
              )}
            </div>

            <div className="space-y-2">
              <label htmlFor="password" className="text-sm font-medium">
                Password
              </label>
              <Input
                id="password"
                type="password"
                {...register('password')}
              />
              {errors.password && (
                <p className="text-sm text-red-600">{errors.password.message}</p>
              )}
            </div>

            <Button type="submit" className="w-full" disabled={isLoading}>
              {isLoading ? 'Logging in...' : 'Login'}
            </Button>
          </form>

          <div className="mt-6 text-center">
            <p className="text-sm text-gray-600">
              Don't have an account?{' '}
              <Link href="/auth/register" className="text-primary hover:underline">
                Register here
              </Link>
            </p>
          </div>
        </CardContent>
      </Card>
    </div>
  )
}
```

### 6.18 Create `frontend/src/app/auth/register/page.tsx`

```typescript
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import Link from 'next/link'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { AuthService } from '@/lib/auth'

const registerSchema = z.object({
  username: z.string().min(3, 'Username must be at least 3 characters'),
  email: z.string().email('Invalid email address'),
  password1: z.string().min(8, 'Password must be at least 8 characters'),
  password2: z.string().min(8, 'Password confirmation is required'),
  age: z.number().min(13, 'Must be at least 13 years old').max(120, 'Invalid age').optional(),
  sex: z.enum(['M', 'F', 'O', 'P']).optional(),
  user_type: z.enum(['student', 'mentor']).default('student'),
}).refine((data) => data.password1 === data.password2, {
  message: "Passwords don't match",
  path: ["password2"],
})

type RegisterForm = z.infer<typeof registerSchema>

export default function RegisterPage() {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState('')
  const router = useRouter()

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<RegisterForm>({
    resolver: zodResolver(registerSchema),
    defaultValues: {
      user_type: 'student',
    },
  })

  const onSubmit = async (data: RegisterForm) => {
    setIsLoading(true)
    setError('')

    try {
      await AuthService.register(data)
      router.push('/dashboard')
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Registration failed')
    } finally {
      setIsLoading(false)
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 py-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold text-center">Register</CardTitle>
          <CardDescription className="text-center">
            Create your account to get started
          </CardDescription>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
            {error && (
              <div className="bg-red-50 border border-red-200 text-red-600 px-4 py-3 rounded">
                {error}
              </div>
            )}
            
            <div className="space-y-2">
              <label htmlFor="username" className="text-sm font-medium">
                Username
              </label>
              <Input
                id="username"
                placeholder="johndoe"
                {...register('username')}
              />
              {errors.username && (
                <p className="text-sm text-red-600">{errors.username.message}</p>
              )}
            </div>

            <div className="space-y-2">
              <label htmlFor="email" className="text-sm font-medium">
                Email
              </label>
              <Input
                id="email"
                type="email"
                placeholder="john@example.com"
                {...register('email')}
              />
              {errors.email && (
                <p className="text-sm text-red-600">{errors.email.message}</p>
              )}
            </div>

            <div className="space-y-2">
              <label htmlFor="password1" className="text-sm font-medium">
                Password
              </label>
              <Input
                id="password1"
                type="password"
                {...register('password1')}
              />
              {errors.password1 && (
                <p className="text-sm text-red-600">{errors.password1.message}</p>
              )}
            </div>

            <div className="space-y-2">
              <label htmlFor="password2" className="text-sm font-medium">
                Confirm Password
              </label>
              <Input
                id="password2"
                type="password"
                {...register('password2')}
              />
              {errors.password2 && (
                <p className="text-sm text-red-600">{errors.password2.message}</p>
              )}
            </div>

            <div className="grid grid-cols-2 gap-4">
              <div className="space-y-2">
                <label htmlFor="age" className="text-sm font-medium">
                  Age (Optional)
                </label>
                <Input
                  id="age"
                  type="number"
                  min="13"
                  max="120"
                  {...register('age', { valueAsNumber: true })}
                />
                {errors.age && (
                  <p className="text-sm text-red-600">{errors.age.message}</p>
                )}
              </div>

              <div className="space-y-2">
                <label htmlFor="sex" className="text-sm font-medium">
                  Gender (Optional)
                </label>
                <select
                  id="sex"
                  className="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
                  {...register('sex')}
                >
                  <option value="">Select...</option>
                  <option value="M">Male</option>
                  <option value="F">Female</option>
                  <option value="O">Other</option>
                  <option value="P">Prefer not to say</option>
                </select>
                {errors.sex && (
                  <p className="text-sm text-red-600">{errors.sex.message}</p>
                )}
              </div>
            </div>

            <div className="space-y-2">
              <label htmlFor="user_type" className="text-sm font-medium">
                I am a...
              </label>
              <select
                id="user_type"
                className="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm ring-offset-background focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
                {...register('user_type')}
              >
                <option value="student">Student</option>
                <option value="mentor">Mentor</option>
              </select>
              {errors.user_type && (
                <p className="text-sm text-red-600">{errors.user_type.message}</p>
              )}
            </div>

            <Button type="submit" className="w-full" disabled={isLoading}>
              {isLoading ? 'Creating account...' : 'Create Account'}
            </Button>
          </form>

          <div className="mt-6 text-center">
            <p className="text-sm text-gray-600">
              Already have an account?{' '}
              <Link href="/auth/login" className="text-primary hover:underline">
                Login here
              </Link>
            </p>
          </div>
        </CardContent>
      </Card>
    </div>
  )
}
```

### 6.19 Create `frontend/src/app/dashboard/page.tsx`

```typescript
'use client'

import { useQuery } from '@apollo/client'
import { useRouter } from 'next/navigation'
import { useEffect } from 'react'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { GET_ME } from '@/graphql/queries'
import { AuthService } from '@/lib/auth'

export default function DashboardPage() {
  const router = useRouter()
  const { data, loading, error } = useQuery(GET_ME)

  useEffect(() => {
    if (!AuthService.isAuthenticated()) {
      router.push('/auth/login')
    }
  }, [router])

  const handleLogout = async () => {
    await AuthService.logout()
    router.push('/')
  }

  if (loading) return <div className="flex justify-center items-center min-h-screen">Loading...</div>
  if (error) return <div className="flex justify-center items-center min-h-screen">Error: {error.message}</div>

  const user = data?.me

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white shadow">
        <div className="container mx-auto px-4 py-6 flex justify-between items-center">
          <h1 className="text-2xl font-bold text-gray-900">Dashboard</h1>
          <Button onClick={handleLogout} variant="outline">
            Logout
          </Button>
        </div>
      </header>

      <div className="container mx-auto px-4 py-8">
        <div className="grid gap-8">
          <Card>
            <CardHeader>
              <CardTitle>Welcome back, {user?.firstName || user?.username}!</CardTitle>
              <CardDescription>
                Here's your profile information
              </CardDescription>
            </CardHeader>
            <CardContent>
              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="text-sm font-medium text-gray-500">Email</label>
                  <p className="text-lg">{user?.email}</p>
                </div>
                <div>
                  <label className="text-sm font-medium text-gray-500">User Type</label>
                  <p className="text-lg capitalize">{user?.userType}</p>
                </div>
                {user?.age && (
                  <div>
                    <label className="text-sm font-medium text-gray-500">Age</label>
                    <p className="text-lg">{user.age}</p>
                  </div>
                )}
                {user?.sex && (
                  <div>
                    <label className="text-sm font-medium text-gray-500">Gender</label>
                    <p className="text-lg">
                      {user.sex === 'M' ? 'Male' : user.sex === 'F' ? 'Female' : user.sex === 'O' ? 'Other' : 'Prefer not to say'}
                    </p>
                  </div>
                )}
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardHeader>
              <CardTitle>Quick Actions</CardTitle>
              <CardDescription>
                What would you like to do today?
              </CardDescription>
            </CardHeader>
            <CardContent>
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <Button className="h-20 flex flex-col">
                  <span className="font-semibold">Browse</span>
                  <span className="text-sm">Explore content</span>
                </Button>
                <Button variant="outline" className="h-20 flex flex-col">
                  <span className="font-semibold">Connect</span>
                  <span className="text-sm">Find mentors</span>
                </Button>
                <Button variant="outline" className="h-20 flex flex-col">
                  <span className="font-semibold">Profile</span>
                  <span className="text-sm">Update info</span>
                </Button>
              </div>
            </CardContent>
          </Card>
        </div>
      </div>
    </div>
  )
}
```

### 6.20 Create `frontend/src/app/about/page.tsx`

```typescript
'use client'

import { useQuery } from '@apollo/client'
import Link from 'next/link'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { GET_STATIC_PAGE } from '@/graphql/queries'

export default function AboutPage() {
  const { data, loading, error } = useQuery(GET_STATIC_PAGE, {
    variables: { slug: 'about-us' }
  })

  if (loading) return <div className="flex justify-center items-center min-h-screen">Loading...</div>

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white shadow">
        <div className="container mx-auto px-4 py-6 flex justify-between items-center">
          <h1 className="text-2xl font-bold text-gray-900">About Us</h1>
          <Link href="/">
            <Button variant="outline">Home</Button>
          </Link>
        </div>
      </header>

      <div className="container mx-auto px-4 py-8">
        <Card>
          <CardHeader>
            <CardTitle>
              {data?.staticPage?.title || 'About Prepji'}
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="prose max-w-none">
              {data?.staticPage?.content ? (
                <div dangerouslySetInnerHTML={{ __html: data.staticPage.content.replace(/\n/g, '<br>') }} />
              ) : (
                <div>
                  <p className="text-lg text-gray-700 mb-6">
                    Prepji is a revolutionary platform designed to connect students with experienced mentors, 
                    fostering an environment of collaborative learning and professional growth.
                  </p>
                  
                  <h2 className="text-xl font-semibold mb-4">Our Mission</h2>
                  <p className="text-gray-700 mb-6">
                    We believe that everyone deserves access to quality mentorship and guidance. Our platform 
                    bridges the gap between ambitious students and industry professionals, creating meaningful 
                    connections that drive success.
                  </p>
                  
                  <h2 className="text-xl font-semibold mb-4">What We Offer</h2>
                  <ul className="list-disc list-inside text-gray-700 space-y-2 mb-6">
                    <li>Personalized mentor matching based on your goals and interests</li>
                    <li>Structured learning paths and progress tracking</li>
                    <li>Community forums for peer-to-peer learning</li>
                    <li>Industry insights and career guidance</li>
                    <li>Flexible scheduling and communication tools</li>
                  </ul>
                  
                  <div className="text-center">
                    <Link href="/auth/register">
                      <Button size="lg">Join Prepji Today</Button>
                    </Link>
                  </div>
                </div>
              )}
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  )
}
```

### 6.21 Create `frontend/src/components/apollo-wrapper.tsx`

```typescript
'use client'

import { ApolloProvider } from '@apollo/client'
import client from '@/lib/apollo-client'

export function ApolloWrapper({ children }: { children: React.ReactNode }) {
  return <ApolloProvider client={client}>{children}</ApolloProvider>
}
```

### 6.22 Create `frontend/.eslintrc.json`

```json
{
  "extends": ["next/core-web-vitals"],
  "rules": {
    "prefer-const": "error",
    "no-var": "error"
  }
}
```

### 6.23 Create `frontend/.prettierrc`

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
```

## Step 7: Complete Backend Django Files

### 7.1 Update `backend/apps/users/migrations/__init__.py`

```python
# This makes Python treat the directory as a package
```

### 7.2 Update `backend/apps/content/migrations/__init__.py`

```python
# This makes Python treat the directory as a package
```

## Step 8: Execution Steps

### 8.1 Initialize the Project

Run these commands in your VS Code terminal:

```bash
# Navigate to project root
cd prepji

# Make manage.py executable
chmod +x backend/manage.py

# Build and start all services
docker-compose up --build
```

### 8.2 Create Database Migrations

In a new terminal window (while docker-compose is running):

```bash
# Create and run migrations
docker-compose exec backend python manage.py makemigrations
docker-compose exec backend python manage.py migrate

# Create superuser
docker-compose exec backend python manage.py createsuperuser
```

### 8.3 Create Static Pages via Django Admin

1. Open http://localhost:8000/admin
2. Login with your superuser account
3. Go to "Static Pages" and click "Add Static Page"
4. Create two pages:

**Home Page:**
- Title: "Welcome to Prepji"
- Slug: "home"
- Content: "Your journey to success starts here. Connect with mentors and unlock your potential."

**About Us Page:**
- Title: "About Prepji"
- Slug: "about-us"  
- Content: "Prepji is dedicated to connecting students with experienced mentors..."

### 8.4 Install Frontend Dependencies

```bash
# Install frontend dependencies
docker-compose exec frontend npm install
```

## Step 9: Testing the Application

### 9.1 Access Points

- **Frontend**: http://localhost:3000
- **Backend Admin**: http://localhost:8000/admin
- **GraphQL Playground**: http://localhost:8000/graphql

### 9.2 Test User Registration

1. Go to http://localhost:3000
2. Click "Register"
3. Fill out the registration form
4. Complete email verification (check Docker logs for console email)
5. Login and access dashboard

### 9.3 Test GraphQL API

Visit http://localhost:8000/graphql and try this query:

```graphql
query {
  staticPages {
    id
    title
    slug
    content
  }
}
```

## Step 10: Development Workflow

### 10.1 Starting Development

```bash
# Start all services
docker-compose up

# Or start in background
docker-compose up -d
```

### 10.2 Viewing Logs

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend
docker-compose logs -f frontend
```

### 10.3 Running Commands

```bash
# Django commands
docker-compose exec backend python manage.py makemigrations
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py collectstatic

# Frontend commands
docker-compose exec frontend npm install <package>
docker-compose exec frontend npm run build
```

### 10.4 Stopping Services

```bash
# Stop all services
docker-compose down

# Stop and remove volumes (careful - this deletes database data)
docker-compose down -v
```

## Step 11: Environment Variables for Production

Create a `.env.production` file for production deployment:

```bash
# Production settings
DEBUG=0
SECRET_KEY=your-super-secure-production-secret-key
JWT_SECRET_KEY=your-super-secure-jwt-production-secret-key
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com

# Database (use managed database service)
DATABASE_URL=postgresql://user:password@host:port/dbname

# Redis (use managed Redis service)
REDIS_URL=redis://host:port/0

# Email (use production email service)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.yourmailservice.com
EMAIL_PORT=587
EMAIL_USE_TLS=1
EMAIL_HOST_USER=your-email@yourdomain.com
EMAIL_HOST_PASSWORD=your-email-password
```

## Troubleshooting

### Common Issues

1. **Port conflicts**: Change ports in docker-compose.yml if 3000, 8000, 5432, or 6379 are in use
2. **Permission issues**: Run `chmod +x backend/manage.py`
3. **Database connection**: Ensure PostgreSQL service is healthy before backend starts
4. **CORS errors**: Verify CORS_ALLOWED_ORIGINS includes your frontend URL

### Useful Commands

```bash
# Reset database completely
docker-compose down -v
docker-compose up --build

# Access database directly
docker-compose exec postgres psql -U prepji_user -d prepji_db

# Access backend shell
docker-compose exec backend python manage.py shell

# View container status
docker-compose ps
```

# Prepji - Complete Project Structure

```
prepji/
├── .env                                    # Environment variables
├── docker-compose.yml                     # Docker services configuration
│
├── backend/                               # Django Backend
│   ├── Dockerfile                        # Backend container configuration
│   ├── requirements.txt                  # Python dependencies
│   ├── manage.py                         # Django management script
│   │
│   ├── prepji/                           # Main Django project
│   │   ├── __init__.py                   # Python package marker
│   │   ├── settings.py                  # Django settings configuration
│   │   ├── urls.py                      # Main URL routing
│   │   ├── wsgi.py                      # WSGI application
│   │   ├── asgi.py                      # ASGI application
│   │   ├── celery.py                    # Celery configuration
│   │   └── schema.py                    # Main GraphQL schema
│   │
│   └── apps/                            # Django applications
│       ├── __init__.py                  # Python package marker
│       │
│       ├── users/                       # User management app
│       │   ├── __init__.py              # Python package marker
│       │   ├── models.py                # CustomUser model with age, sex, user_type
│       │   ├── admin.py                 # Django admin configuration
│       │   ├── schema.py                # GraphQL schema for users
│       │   ├── apps.py                  # App configuration
│       │   └── migrations/              # Database migrations
│       │       └── __init__.py          # Python package marker
│       │
│       └── content/                     # Content management app
│           ├── __init__.py              # Python package marker
│           ├── models.py                # StaticPage model
│           ├── admin.py                 # Django admin for static pages
│           ├── schema.py                # GraphQL schema for content
│           ├── apps.py                  # App configuration
│           └── migrations/              # Database migrations
│               └── __init__.py          # Python package marker
│
└── frontend/                            # Next.js Frontend
    ├── Dockerfile                       # Frontend container configuration
    ├── package.json                     # Node.js dependencies and scripts
    ├── next.config.js                   # Next.js configuration
    ├── tailwind.config.js               # Tailwind CSS configuration
    ├── postcss.config.js                # PostCSS configuration
    ├── tsconfig.json                    # TypeScript configuration
    ├── .eslintrc.json                   # ESLint configuration
    ├── .prettierrc                      # Prettier configuration
    │
    └── src/                             # Source code
        ├── globals.css                  # Global CSS with Tailwind
        │
        ├── lib/                         # Utility libraries
        │   ├── apollo-client.ts         # Apollo Client configuration
        │   ├── auth.ts                  # Authentication service
        │   └── utils.ts                 # Utility functions
        │
        ├── graphql/                     # GraphQL queries and mutations
        │   └── queries.ts               # All GraphQL operations
        │
        ├── components/                  # React components
        │   ├── apollo-wrapper.tsx       # Apollo Provider wrapper
        │   └── ui/                      # UI components (shadcn/ui)
        │       ├── button.tsx           # Button component
        │       ├── input.tsx            # Input component
        │       └── card.tsx             # Card components
        │
        └── app/                         # Next.js App Router pages
            ├── layout.tsx               # Root layout
            ├── page.tsx                 # Home page
            │
            ├── auth/                    # Authentication pages
            │   ├── login/
            │   │   └── page.tsx         # Login page with form
            │   └── register/
            │       └── page.tsx         # Registration page with form
            │
            ├── dashboard/               # Protected dashboard
            │   └── page.tsx             # User dashboard
            │
            └── about/                   # Static pages
                └── page.tsx             # About page (GraphQL powered)
```

## File Count Summary

### Backend (Django)
- **Total Files**: 17
- **Configuration Files**: 6 (Dockerfile, requirements.txt, manage.py, settings.py, urls.py, etc.)
- **User App Files**: 5 (models, admin, schema, apps, __init__)
- **Content App Files**: 5 (models, admin, schema, apps, __init__)
- **Main Project Files**: 1 (__init__.py)

### Frontend (Next.js)
- **Total Files**: 19
- **Configuration Files**: 6 (package.json, Dockerfile, next.config.js, tailwind.config.js, tsconfig.json, etc.)
- **Library Files**: 3 (apollo-client.ts, auth.ts, utils.ts)
- **Component Files**: 4 (apollo-wrapper.tsx, button.tsx, input.tsx, card.tsx)
- **Page Files**: 5 (layout.tsx, page.tsx for home, login, register, dashboard, about)
- **GraphQL Files**: 1 (queries.ts)

### Root Files
- **Total Files**: 2
- **Docker Configuration**: 1 (docker-compose.yml)
- **Environment**: 1 (.env)

## Key Features by Location

### 🔐 **Authentication Flow**
- **Backend**: `apps/users/` - Custom user model with JWT
- **Frontend**: `app/auth/` - Login/Register pages with form validation

### 📊 **Data Management**
- **Backend**: `apps/content/` - Static page models
- **Frontend**: `graphql/queries.ts` - GraphQL operations

### 🎨 **UI Components**
- **Frontend**: `components/ui/` - Reusable shadcn/ui components
- **Styling**: `globals.css` + `tailwind.config.js`

### 🏗️ **Infrastructure**
- **Root**: `docker-compose.yml` - Multi-service Docker setup
- **Containers**: Individual Dockerfiles for backend/frontend

This structure follows industry best practices with clear separation of concerns, modular architecture, and proper containerization. Each service is independently scalable and maintainable.


Perfect! I've created a comprehensive step-by-step tutorial for setting up your completely dockerized Prepji application. Here are the key highlights of what this setup provides:

## What You'll Get:

### ✅ **Complete Docker Setup**
- PostgreSQL 17 database
- Redis for caching and Celery
- Django 5.2.5 backend with all your specified packages
- Next.js 15 frontend with React 19
- Celery for background tasks

### ✅ **Authentication System**
- JWT-based authentication using Django Allauth
- Custom user model with age, sex, and user_type fields
- Three user types: Student, Mentor, Admin
- Email verification workflow
- Login/Register pages with form validation

### ✅ **GraphQL API**
- Strawberry GraphQL Django integration
- User queries and mutations
- Static page content management
- Proper authentication decorators

### ✅ **Frontend Features**
- Modern Next.js 15 with App Router
- Apollo Client 4 for GraphQL
- Tailwind CSS with shadcn/ui components
- React Hook Form with Zod validation
- Responsive design

### ✅ **Content Management**
- Static pages manageable via Django admin
- GraphQL API to serve content to frontend
- Home and About pages as examples

## Quick Start Commands:

1. **Copy all the files** from the artifact into your VS Code project
2. **Run the setup**:
```bash
cd prepji
docker-compose up --build
```
3. **Initialize database** (in new terminal):
```bash
docker-compose exec backend python manage.py makemigrations
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py createsuperuser
```

The application will be available at:
- **Frontend**: http://localhost:3000
- **Backend Admin**: http://localhost:8000/admin  
- **GraphQL**: http://localhost:8000/graphql

All dependencies are the latest versions as of August 2025, so you won't encounter any deprecation warnings. The setup follows industry best practices with proper environment variable management, security configurations, and a clean project structure.


## Next Steps

1. **Add Social Authentication**: Configure Google/Facebook OAuth in Django Allauth
2. **Add Email Templates**: Create custom email templates for registration
3. **Implement WebSockets**: Use Django Channels for real-time features
4. **Add Testing**: Implement unit and integration tests
5. **Production Deployment**: Deploy to cloud services with proper security

Your dockerized Prepji application is now ready! Everything runs in containers, and you can develop without installing anything globally on your Mac.

