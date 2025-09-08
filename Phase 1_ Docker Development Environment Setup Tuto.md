<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Phase 1: Docker Development Environment Setup Tutorial

**Document Version:** 1.0
**Date:** September 6, 2025
**Tutorial:** Phase 1 - Complete Containerized Development Environment

***

## 1. Overview

### 🎯 Learning Objectives

By completing this tutorial, you will have:

- ✅ Complete Docker-based development environment for Prepji
- ✅ All services running in isolated containers (no global installs)
- ✅ Hot-reloading for both frontend and backend development
- ✅ Database and Redis with persistent data storage
- ✅ Background task processing with Celery and monitoring via Flower
- ✅ All localhost URLs accessible for development


### 📋 Prerequisites

- ✅ MacBook Air with 24GB RAM (confirmed)
- ✅ Docker Desktop installed and running (confirmed)
- ✅ VS Code installed (confirmed)
- ✅ Terminal access
- ✅ Internet connection for downloading base images


### ⏱️ Estimated Time

**45-60 minutes** (including downloading Docker images)

### 🎚️ Difficulty Level

**Beginner-Intermediate** - Copy-paste ready with detailed explanations

***

## 2. Context \& Background

### 🏗️ What We're Building

A complete containerized development environment that replicates production infrastructure locally. This approach ensures:

- **Environment Consistency:** What works on your Mac will work in production
- **Team Collaboration:** Any developer can spin up identical environment
- **Clean System:** Zero global installations, everything contained in Docker
- **Production Parity:** Same services, same configurations


### 🎯 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                 Docker Development Environment           │
├─────────────────┬─────────────────┬─────────────────────┤
│   Frontend      │    Backend      │     Data Layer      │
│   Next.js 15    │   Django 5.2.5  │   PostgreSQL 17     │
│   Port: 3000    │   Port: 8000    │   Port: 5432        │
├─────────────────┼─────────────────┼─────────────────────┤
│   Celery Worker │   Flower Monitor│     Redis Cache     │
│   Background    │   Port: 5555    │   Port: 6379        │
│   Tasks         │                 │                     │
└─────────────────┴─────────────────┴─────────────────────┘
```


### 💼 Business Value

- **Rapid Development:** Hot-reloading for instant feedback
- **Realistic Testing:** Same environment as production
- **Easy Onboarding:** New developers productive in minutes
- **Clean Development:** No system pollution with global packages

***

## 3. Step-by-Step Implementation

### 📁 Step 3.1: Create Project Directory Structure

**What:** Set up the foundational folder structure for our monorepo.

**Commands:**

```bash
# Create main project directory
mkdir prepji
cd prepji

# Create monorepo structure
mkdir -p frontend backend scripts docs

# Create configuration directories
mkdir -p .docker/frontend .docker/backend .docker/nginx

# Create volume mount directories
mkdir -p data/postgres data/redis

# Verify structure
tree -a -L 3
```

**Expected Directory Structure:**

```
prepji/
├── .docker/
│   ├── frontend/
│   ├── backend/
│   └── nginx/
├── frontend/
├── backend/
├── scripts/
├── docs/
└── data/
    ├── postgres/
    └── redis/
```

**✅ Validation Checkpoint:**

- [ ] Directory structure matches above
- [ ] You're in the `prepji` directory
- [ ] All subdirectories created successfully

***

### 🐳 Step 3.2: Create Backend Dockerfile

**What:** Configure Python 3.13 container for Django development with all required dependencies.

**File:** `.docker/backend/Dockerfile`

```dockerfile
# Use Python 3.13 slim image for smaller size and security
FROM python:3.13-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

# Set work directory
WORKDIR /app

# Install system dependencies required for Python packages
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        libpq-dev \
        curl \
        netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Create requirements file for Python dependencies
COPY requirements.txt .

# Install Python dependencies
RUN pip install --upgrade pip \
    && pip install -r requirements.txt

# Copy project files
COPY . .

# Create media and static directories
RUN mkdir -p media staticfiles

# Expose port 8000 for Django development server
EXPOSE 8000

# Health check to ensure Django is running
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/admin/ || exit 1

# Default command for development
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**File:** `backend/requirements.txt`

```txt
# Core Django Framework
Django==5.2.5
python-decouple==3.8.*

# Database
psycopg[binary]==3.2.9

# API Framework
djangorestframework==3.16.*
djangorestframework-simplejwt==5.5.*

# Authentication
dj-rest-auth==7.0.*
django-allauth==65.*

# GraphQL
strawberry-graphql-django==0.65.*

# Async and Real-time
celery[redis]==5.4.*
redis==5.2.*
channels[daphne]==4.3.*
channels-redis==4.3.*
django-redis==5.4.*

# Security and Middleware
django-cors-headers==4.7.*
django-ratelimit==4.1.*
django-axes==8.0.*

# File Handling
pillow==11.3.*

# Utilities
requests==2.32.*

# Development Tools
django-silk==5.4.*
django-filter==25.*
```

**✅ Validation Checkpoint:**

- [ ] `.docker/backend/Dockerfile` created with exact content above
- [ ] `backend/requirements.txt` created with all dependencies listed
- [ ] Files saved in correct locations

***

### ⚛️ Step 3.3: Create Frontend Dockerfile

**What:** Configure Node.js 20 container for Next.js 15 development with hot-reloading.

**File:** `.docker/frontend/Dockerfile`

```dockerfile
# Use Node 20 Alpine for smaller image size
FROM node:20-alpine

# Set environment variables
ENV NODE_ENV=development
ENV NEXT_TELEMETRY_DISABLED=1

# Install system dependencies
RUN apk add --no-cache \
    curl

# Set work directory
WORKDIR /app

# Copy package files first for better layer caching
COPY package*.json ./

# Install dependencies (FIXED: Changed from npm ci)
RUN npm install

# Copy project files
COPY . .

# Expose port 3000 for Next.js development server
EXPOSE 3000

# Health check for Next.js
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000 || exit 1

# Default command for development
CMD ["npm", "run", "dev"]

```

**File:** `frontend/package.json`

```json
{
  "name": "prepji-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "typescript": "^5.9.2",
    "@apollo/client": "^4.0.3",
    "graphql": "^16.11.0",
    "react-hook-form": "^7.47.0",
    "@hookform/resolvers": "^5.2.1",
    "zod": "^4.1.5",
    "tailwindcss": "^3.4.17",
    "@radix-ui/react-dialog": "^1.1.15",
    "@radix-ui/react-dropdown-menu": "^2.1.16",
    "@radix-ui/react-form": "^0.1.8",
    "@radix-ui/react-slot": "^1.2.3",
    "@radix-ui/react-toast": "^1.2.15",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^3.3.1",
    "tailwindcss-animate": "^1.0.0",
    "zustand": "^5.0.8",
    "js-cookie": "^3.0.5",
    "react-error-boundary": "^6.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/react": "^19.1.12",
    "@types/react-dom": "^19.1.9",
    "eslint": "^9.35.0",
    "@types/js-cookie": "^3.0.6",
    "eslint-config-next": "^15.0.0"
  }
}
```

**✅ Validation Checkpoint:**

- [ ] `.docker/frontend/Dockerfile` created
- [ ] `frontend/package.json` created with all dependencies
- [ ] Both files saved in correct locations

***

### 🐳 Step 3.4: Create Docker Compose Configuration

**What:** Orchestrate all services with proper networking, volumes, and dependencies.

**File:** `docker-compose.yml` (in root `prepji/` directory)

```yaml
# version: '3.8'

services:
  # PostgreSQL Database
  db:
    image: postgres:17-alpine
    container_name: prepji_db
    environment:
      POSTGRES_DB: prepji_db
      POSTGRES_USER: prepji_user
      POSTGRES_PASSWORD: prepji_pass_2025
      POSTGRES_HOST_AUTH_METHOD: trust
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql:ro
    ports:
      - "5432:5432"
    networks:
      - prepji_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prepji_user -d prepji_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis Cache and Message Broker
  redis:
    image: redis:7-alpine
    container_name: prepji_redis
    command: redis-server --appendonly yes --requirepass prepji_redis_pass
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - prepji_network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "prepji_redis_pass", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # Backend Django Application
  backend:
    build:
      context: ./backend
      dockerfile: ../.docker/backend/Dockerfile
    container_name: prepji_backend
    environment:
      - DEBUG=1
      - DATABASE_URL=postgresql://prepji_user:prepji_pass_2025@db:5432/prepji_db
      - REDIS_URL=redis://:prepji_redis_pass@redis:6379/0
      - CELERY_BROKER_URL=redis://:prepji_redis_pass@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:prepji_redis_pass@redis:6379/0
      - CORS_ALLOWED_ORIGINS=http://localhost:3000
      - SECRET_KEY=dev-secret-key-change-in-production
      - ALLOWED_HOSTS=localhost,127.0.0.1,backend
    volumes:
      - ./backend:/app
      - django_media:/app/media
      - django_static:/app/staticfiles
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - prepji_network
    restart: unless-stopped
    command: >
      sh -c "python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             python manage.py runserver 0.0.0.0:8000"

  # Frontend Next.js Application
  frontend:
    build:
      context: ./frontend
      dockerfile: ../.docker/frontend/Dockerfile
    container_name: prepji_frontend
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:8000
      - NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql
      - NEXT_PUBLIC_REST_API_URL=http://localhost:8000/api
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    ports:
      - "3000:3000"
    depends_on:
      - backend
    networks:
      - prepji_network
    restart: unless-stopped

  # Celery Worker for Background Tasks
  celery:
    build:
      context: ./backend
      dockerfile: ../.docker/backend/Dockerfile
    container_name: prepji_celery
    environment:
      - DEBUG=1
      - DATABASE_URL=postgresql://prepji_user:prepji_pass_2025@db:5432/prepji_db
      - REDIS_URL=redis://:prepji_redis_pass@redis:6379/0
      - CELERY_BROKER_URL=redis://:prepji_redis_pass@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:prepji_redis_pass@redis:6379/0
    volumes:
      - ./backend:/app
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - prepji_network
    restart: unless-stopped
    command: celery -A prepji worker -l info -Q high,medium,low --concurrency=4

  # Flower - Celery Monitoring Dashboard (FIXED)
  flower:
    image: mher/flower
    container_name: prepji_flower
    environment:
      - CELERY_BROKER_URL=redis://:prepji_redis_pass@redis:6379/0
      - CELERY_RESULT_BACKEND=redis://:prepji_redis_pass@redis:6379/0
    ports:
      - "5555:5555"
    depends_on:
      - redis
      - celery
    networks:
      - prepji_network
    restart: unless-stopped

networks:
  prepji_network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  django_media:
  django_static:
```

**✅ Validation Checkpoint:**

- [ ] `docker-compose.yml` created in root directory
- [ ] File contains all services: db, redis, backend, frontend, celery, flower
- [ ] Proper networking and volumes configured

***

### 📄 Step 3.5: Create Environment Configuration

**What:** Set up environment variables for development, staging, and production.

**File:** `.env.development` (in root directory)

```bash
# Environment
NODE_ENV=development
DEBUG=1

# Database Configuration
POSTGRES_DB=prepji_db
POSTGRES_USER=prepji_user
POSTGRES_PASSWORD=prepji_pass_2025
DATABASE_URL=postgresql://prepji_user:prepji_pass_2025@db:5432/prepji_db

# Redis Configuration
REDIS_PASSWORD=prepji_redis_pass
REDIS_URL=redis://:prepji_redis_pass@redis:6379/0
CELERY_BROKER_URL=redis://:prepji_redis_pass@redis:6379/0
CELERY_RESULT_BACKEND=redis://:prepji_redis_pass@redis:6379/0

# Django Configuration
SECRET_KEY=dev-secret-key-change-in-production-2025
ALLOWED_HOSTS=localhost,127.0.0.1,backend,0.0.0.0
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://127.0.0.1:3000

# JWT Configuration
JWT_SECRET_KEY=jwt-secret-key-change-this-in-production-2025
JWT_ACCESS_TOKEN_LIFETIME=60
JWT_REFRESH_TOKEN_LIFETIME=1440

# Frontend Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:8000/graphql
NEXT_PUBLIC_REST_API_URL=http://localhost:8000/api

# Email Configuration (Development - Console Backend)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
DEFAULT_FROM_EMAIL=noreply@prepji.com

# File Upload Settings
MAX_FILE_SIZE=1048576  # 1MB in bytes
ALLOWED_IMAGE_TYPES=JPEG,PNG
ALLOWED_DOCUMENT_TYPES=PDF

# Feature Flags
ACCOUNT_EMAIL_VERIFICATION=optional
SOCIAL_LOGIN_ENABLED=0
```

**File:** `scripts/init-db.sql`

```sql
-- Initial database setup for Prepji
-- This script runs when PostgreSQL container starts for the first time

-- Create additional databases if needed
-- CREATE DATABASE prepji_test;

-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Set timezone
SET timezone = 'UTC';

-- Initial admin user will be created via Django management commands
-- in Phase 2 of development

-- Log initialization
DO $$
BEGIN
    RAISE NOTICE 'Prepji database initialized successfully at %', NOW();
END $$;
```

**File:** `.gitignore`

```gitignore
# Environment variables
.env
.env.local
.env.production
.env.staging

# Docker
.docker/*/node_modules/
.docker/*/.next/

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# IDE
.vscode/settings.json
.idea/

# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Dependencies
node_modules/
__pycache__/
*.pyc

# Build outputs
.next/
dist/
build/
staticfiles/

# Database
*.sqlite3
*.db

# Media files
media/

# Local data
data/postgres/*
data/redis/*
!data/postgres/.gitkeep
!data/redis/.gitkeep
```

**✅ Validation Checkpoint:**

- [ ] `.env.development` created with all configuration
- [ ] `scripts/init-db.sql` created
- [ ] `.gitignore` created with proper exclusions

***

### 🚀 Step 3.6: Create Development Scripts

**What:** Convenience scripts for common development tasks.

**File:** `scripts/dev-start.sh`

```bash
#!/bin/bash
# Development startup script for Prepji

echo "🚀 Starting Prepji Development Environment..."

# Check if Docker is running
if ! docker info > /dev/null 2>&1; then
    echo "❌ Docker is not running. Please start Docker Desktop."
    exit 1
fi

# Load environment variables
if [ -f .env.development ]; then
    export $(cat .env.development | grep -v '^#' | xargs)
fi

# Build and start all services
echo "📦 Building and starting all services..."
docker-compose up --build -d

# Wait for services to be healthy
echo "⏳ Waiting for services to be ready..."
sleep 10

# Check service health
echo "🔍 Checking service health..."
docker-compose ps

# Display URLs
echo ""
echo "✅ Prepji Development Environment Ready!"
echo ""
echo "📊 Available Services:"
echo "   Frontend:     http://localhost:3000"
echo "   Backend:      http://localhost:8000"
echo "   API Docs:     http://localhost:8000/api/"
echo "   GraphQL:      http://localhost:8000/graphql/"
echo "   Admin:        http://localhost:8000/admin/"
echo "   Flower:       http://localhost:5555"
echo ""
echo "📝 Useful Commands:"
echo "   View logs:    docker-compose logs -f"
echo "   Stop all:     docker-compose down"
echo "   Restart:      docker-compose restart [service]"
echo ""
```

**File:** `scripts/dev-stop.sh`

```bash
#!/bin/bash
# Stop development environment

echo "🛑 Stopping Prepji Development Environment..."

docker-compose down

echo "✅ All services stopped."
echo "💡 Data is preserved in Docker volumes."
echo "💡 To remove all data: docker-compose down -v"
```

**File:** `scripts/dev-reset.sh`

```bash
#!/bin/bash
# Reset development environment (removes all data)

echo "⚠️  DANGER: This will delete all data!"
read -p "Are you sure? (y/N): " -n 1 -r
echo

if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "🗑️  Stopping services and removing data..."
    docker-compose down -v
    docker system prune -f
    
    # Recreate empty data directories
    mkdir -p data/postgres data/redis
    touch data/postgres/.gitkeep data/redis/.gitkeep
    
    echo "✅ Development environment reset complete."
    echo "💡 Run './scripts/dev-start.sh' to restart with fresh data."
else
    echo "❌ Reset cancelled."
fi
```

**Make scripts executable:**

```bash
chmod +x scripts/*.sh
```

**✅ Validation Checkpoint:**

- [ ] All three scripts created in `scripts/` directory
- [ ] Scripts are executable (`chmod +x` applied)
- [ ] Scripts contain proper error handling and user feedback

***

### 🏗️ Step 3.7: Create Basic Django Project Structure

**What:** Initialize minimal Django project to test container connectivity.

**File:** `backend/manage.py`

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

**File:** `backend/prepji/celery.py`

```python
"""
Celery configuration for Prepji project.
"""

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

@app.task(bind=True)
def debug_task(self):
    print(f'Request: {self.request!r}')
```

**File:** `backend/prepji/__init__.py`

```python
# This makes Python treat the directory as a package

# Import Celery app to ensure it's loaded when Django starts
from .celery import app as celery_app

__all__ = ('celery_app',)
```

**File:** `backend/prepji/settings.py`

```python
"""
Django settings for Prepji project.
Development configuration with Docker support.
"""

import os
from pathlib import Path
from decouple import config

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# Security settings
SECRET_KEY = config('SECRET_KEY', default='dev-secret-key-change-in-production')
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
]

THIRD_PARTY_APPS = [
    'rest_framework',
    'rest_framework.authtoken',  # ← ADD THIS - Required for dj-rest-auth 7.x
    'rest_framework_simplejwt',
    'dj_rest_auth',
    'dj_rest_auth.registration',
    'allauth',
    'allauth.account',
    'corsheaders',
    'strawberry.django',
]

LOCAL_APPS = [
    # Will be added in Phase 2
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'allauth.account.middleware.AccountMiddleware',
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

# Database configuration
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('POSTGRES_DB', default='prepji_db'),
        'USER': config('POSTGRES_USER', default='prepji_user'),
        'PASSWORD': config('POSTGRES_PASSWORD', default='prepji_pass_2025'),
        'HOST': config('DB_HOST', default='db'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Default primary key field type
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# CORS settings
CORS_ALLOWED_ORIGINS = config(
    'CORS_ALLOWED_ORIGINS', 
    default='http://localhost:3000,http://127.0.0.1:3000'
).split(',')

CORS_ALLOW_CREDENTIALS = True

# REST Framework configuration
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
}

# Django Allauth configuration for latest version (65.x)
ACCOUNT_LOGIN_METHODS = {"email"}  # New syntax for allauth 65.x
ACCOUNT_EMAIL_VERIFICATION = "optional"
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_EMAIL_REQUIRED = True

# dj-rest-auth configuration for 7.x
REST_AUTH = {
    'USE_JWT': True,
    'JWT_AUTH_COOKIE': 'prepji-auth',
    'JWT_AUTH_REFRESH_COOKIE': 'prepji-refresh',
    'JWT_AUTH_SECURE': False,  # Set to True in production with HTTPS
}

# Celery Configuration
CELERY_BROKER_URL = config('CELERY_BROKER_URL', default='redis://redis:6379/0')
CELERY_RESULT_BACKEND = config('CELERY_RESULT_BACKEND', default='redis://redis:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE

# Logging configuration
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'root': {
        'handlers': ['console'],
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': 'INFO',
        },
    },
}
```

**File:** `backend/prepji/urls.py`

```python
"""
URL configuration for Prepji project.
"""
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
from django.http import JsonResponse

def api_root(request):
    """API root endpoint showing available endpoints."""
    return JsonResponse({
        'message': 'Welcome to Prepji API',
        'version': '1.0.0',
        'endpoints': {
            'admin': '/admin/',
            'api': '/api/',
            'graphql': '/graphql/',
        },
        'status': 'Development'
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', api_root, name='api_root'),
    # API endpoints will be added in Phase 2
    # path('api/auth/', include('dj_rest_auth.urls')),
    # path('graphql/', include('strawberry.django.urls')),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

**File:** `backend/prepji/wsgi.py`

```python
"""
WSGI config for Prepji project.
"""

import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'prepji.settings')
application = get_wsgi_application()
```

**✅ Validation Checkpoint:**

- [ ] Django project structure created
- [ ] All files saved in correct locations
- [ ] No syntax errors in Python files

***

### ⚛️ Step 3.8: Create Basic Next.js Project Structure

**What:** Initialize minimal Next.js application to test container connectivity.

**File:** `frontend/next.config.js`

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://backend:8000/api/:path*',
      },
    ];
  },
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NEXT_PUBLIC_GRAPHQL_URL: process.env.NEXT_PUBLIC_GRAPHQL_URL,
  },
};

module.exports = nextConfig;
```

**File:** `frontend/app/layout.tsx`

```tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Prepji - JEE Mentorship Platform',
  description: 'Connect JEE aspirants with experienced mentors',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100">
          <header className="bg-white shadow-sm border-b">
            <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
              <div className="flex items-center justify-between h-16">
                <div className="flex items-center">
                  <h1 className="text-2xl font-bold text-indigo-600">
                    Prepji
                  </h1>
                  <span className="ml-2 text-sm text-gray-500">
                    Development Environment
                  </span>
                </div>
              </div>
            </div>
          </header>
          <main className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
            {children}
          </main>
        </div>
      </body>
    </html>
  );
}
```

**File:** `frontend/app/page.tsx`

```tsx
'use client';

import { useEffect, useState } from 'react';

interface ApiStatus {
  message: string;
  version: string;
  status: string;
  endpoints: {
    admin: string;
    api: string;
    graphql: string;
  };
}

export default function HomePage() {
  const [apiStatus, setApiStatus] = useState<ApiStatus | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const checkApiConnection = async () => {
      try {
        const response = await fetch('http://localhost:8000/api/');
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        setApiStatus(data);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    };

    checkApiConnection();
  }, []);

  return (
    <div className="space-y-8">
      {/* Welcome Section */}
      <div className="bg-white rounded-lg shadow-md p-8">
        <h2 className="text-3xl font-bold text-gray-900 mb-4">
          🎉 Welcome to Prepji Development Environment!
        </h2>
        <p className="text-gray-600 mb-6">
          Your containerized development environment is now running. 
          All services are isolated in Docker containers with hot-reloading enabled.
        </p>
        
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          <div className="bg-blue-50 border border-blue-200 rounded-lg p-4">
            <h3 className="font-semibold text-blue-900">Frontend</h3>
            <p className="text-sm text-blue-700">Next.js 15 + React 19</p>
            <p className="text-xs text-blue-600">Port: 3000</p>
          </div>
          <div className="bg-green-50 border border-green-200 rounded-lg p-4">
            <h3 className="font-semibold text-green-900">Backend</h3>
            <p className="text-sm text-green-700">Django 5.2.5 + Python 3.13</p>
            <p className="text-xs text-green-600">Port: 8000</p>
          </div>
          <div className="bg-purple-50 border border-purple-200 rounded-lg p-4">
            <h3 className="font-semibold text-purple-900">Database</h3>
            <p className="text-sm text-purple-700">PostgreSQL 17</p>
            <p className="text-xs text-purple-600">Port: 5432</p>
          </div>
        </div>
      </div>

      {/* API Connection Status */}
      <div className="bg-white rounded-lg shadow-md p-8">
        <h3 className="text-xl font-semibold text-gray-900 mb-4">
          🔌 Backend API Connection Status
        </h3>
        
        {loading && (
          <div className="text-center py-4">
            <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-indigo-600 mx-auto"></div>
            <p className="mt-2 text-gray-600">Connecting to backend...</p>
          </div>
        )}

        {error && (
          <div className="bg-red-50 border border-red-200 rounded-lg p-4">
            <h4 className="font-semibold text-red-900">❌ Connection Failed</h4>
            <p className="text-red-700 text-sm mt-1">{error}</p>
            <p className="text-red-600 text-xs mt-2">
              Make sure the backend container is running and healthy.
            </p>
          </div>
        )}

        {apiStatus && (
          <div className="bg-green-50 border border-green-200 rounded-lg p-4">
            <h4 className="font-semibold text-green-900">✅ Connected Successfully</h4>
            <div className="mt-3 space-y-2">
              <p className="text-sm">
                <span className="font-medium">Message:</span> {apiStatus.message}
              </p>
              <p className="text-sm">
                <span className="font-medium">Version:</span> {apiStatus.version}
              </p>
              <p className="text-sm">
                <span className="font-medium">Status:</span> {apiStatus.status}
              </p>
            </div>
          </div>
        )}
      </div>

      {/* Available Services */}
      <div className="bg-white rounded-lg shadow-md p-8">
        <h3 className="text-xl font-semibold text-gray-900 mb-4">
          🌐 Available Development URLs
        </h3>
        
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <a 
            href="http://localhost:3000" 
            target="_blank" 
            rel="noopener noreferrer"
            className="block p-4 border border-gray-200 rounded-lg hover:bg-gray-50 transition-colors"
          >
            <h4 className="font-medium text-indigo-600">Frontend (This Page)</h4>
            <p className="text-sm text-gray-600">localhost:3000</p>
          </a>
          
          <a 
            href="http://localhost:8000/api/" 
            target="_blank" 
            rel="noopener noreferrer"
            className="block p-4 border border-gray-200 rounded-lg hover:bg-gray-50 transition-colors"
          >
            <h4 className="font-medium text-green-600">Backend API</h4>
            <p className="text-sm text-gray-600">localhost:8000/api/</p>
          </a>
          
          <a 
            href="http://localhost:8000/admin/" 
            target="_blank" 
            rel="noopener noreferrer"
            className="block p-4 border border-gray-200 rounded-lg hover:bg-gray-50 transition-colors"
          >
            <h4 className="font-medium text-red-600">Django Admin</h4>
            <p className="text-sm text-gray-600">localhost:8000/admin/</p>
          </a>
          
          <a 
            href="http://localhost:5555" 
            target="_blank" 
            rel="noopener noreferrer"
            className="block p-4 border border-gray-200 rounded-lg hover:bg-gray-50 transition-colors"
          >
            <h4 className="font-medium text-yellow-600">Celery Monitor</h4>
            <p className="text-sm text-gray-600">localhost:5555</p>
          </a>
        </div>
      </div>
    </div>
  );
}
```

**File:** `frontend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "es6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "node",
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
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

**✅ Validation Checkpoint:**

- [ ] Next.js project structure created
- [ ] TypeScript configuration set up
- [ ] All files saved in correct locations

***

### 🎯 Step 3.9: Fix Django Settings Import

**What:** Add missing import to Django settings file.

**Update File:** `backend/prepji/settings.py` (add missing import at the top)

```python
"""
Django settings for Prepji project.
Development configuration with Docker support.
"""

import os
from pathlib import Path  # Add this line
from decouple import config

# Continue with rest of the file (no other changes needed)
```


***

### 🚀 Step 3.10: Start the Development Environment

**What:** Launch all services and verify everything is working.

**Commands:**

```bash
# Ensure you're in the prepji root directory
cd prepji

# Start the development environment
./scripts/dev-start.sh
```

**Expected Output:**

```
🚀 Starting Prepji Development Environment...
📦 Building and starting all services...
⏳ Waiting for services to be ready...
🔍 Checking service health...

    Name                 Command               State                 Ports
-------------------------------------------------------------------------------
prepji_backend      sh -c python manage.py ...   Up      0.0.0.0:8000->8000/tcp
prepji_celery       celery -A prepji worker ...   Up
prepji_db           docker-entrypoint.sh postgres Up      0.0.0.0:5432->5432/tcp
prepji_flower       celery -A prepji flower ...   Up      0.0.0.0:5555->5555/tcp
prepji_frontend     npm run dev                   Up      0.0.0.0:3000->3000/tcp
prepji_redis        redis-server --appendonly ... Up      0.0.0.0:6379->6379/tcp

✅ Prepji Development Environment Ready!

📊 Available Services:
   Frontend:     http://localhost:3000
   Backend:      http://localhost:8000
   API Docs:     http://localhost:8000/api/
   GraphQL:      http://localhost:8000/graphql/
   Admin:        http://localhost:8000/admin/
   Flower:       http://localhost:5555
```


***

## 4. Testing \& Verification

### ✅ Step 4.1: Verify All Services Are Running

**Check Docker containers:**

```bash
docker-compose ps
```

**All containers should show "Up" status.**

### ✅ Step 4.2: Test Frontend Application

1. **Open in browser:** http://localhost:3000
2. **You should see:**
    - Welcome message with Prepji branding
    - Service status cards showing Frontend, Backend, Database
    - API connection status (should show "Connected Successfully")
    - Grid of available development URLs

### ✅ Step 4.3: Test Backend API

1. **Open in browser:** http://localhost:8000/api/
2. **You should see JSON response:**
```json
{
  "message": "Welcome to Prepji API",
  "version": "1.0.0",
  "endpoints": {
    "admin": "/admin/",
    "api": "/api/",
    "graphql": "/graphql/"
  },
  "status": "Development"
}
```


### ✅ Step 4.4: Test Django Admin

1. **Open in browser:** http://localhost:8000/admin/
2. **You should see:** Django administration login page

### ✅ Step 4.5: Test Celery Monitoring

1. **Open in browser:** http://localhost:5555
2. **You should see:** Flower dashboard showing Celery workers

### ✅ Step 4.6: Test Hot Reloading

**Frontend Hot Reload Test:**

1. Open `frontend/app/page.tsx`
2. Change the welcome message
3. Save the file
4. Browser should automatically refresh with changes

**Backend Hot Reload Test:**

1. Open `backend/prepji/urls.py`
2. Modify the JSON response message
3. Save the file
4. Refresh http://localhost:8000/api/ to see changes

***

## 5. Troubleshooting Common Issues

### 🔧 Issue: Containers Won't Start

**Symptoms:** Docker compose fails to start services

**Solutions:**

```bash
# Check Docker Desktop is running
docker info

# Check for port conflicts
lsof -i :3000  # Should be empty
lsof -i :8000  # Should be empty
lsof -i :5432  # Should be empty

# Reset and try again
./scripts/dev-reset.sh
./scripts/dev-start.sh
```


### 🔧 Issue: Frontend Can't Connect to Backend

**Symptoms:** API connection shows failed on frontend

**Solutions:**

```bash
# Check backend logs
docker-compose logs backend

# Restart backend service
docker-compose restart backend

# Verify network connectivity
docker-compose exec frontend curl http://backend:8000/api/
```


### 🔧 Issue: Database Connection Failed

**Symptoms:** Django shows database connection errors

**Solutions:**

```bash
# Check database logs
docker-compose logs db

# Wait for database to fully initialize
sleep 30

# Run migrations manually
docker-compose exec backend python manage.py migrate
```


### 🔧 Issue: Permission Denied on Scripts

**Symptoms:** `./scripts/dev-start.sh: Permission denied`

**Solution:**

```bash
chmod +x scripts/*.sh
```


***

## 6. Summary \& Next Steps

### 🎉 What You've Accomplished

**✅ Complete Containerized Environment**

- All services running in isolated Docker containers
- No global installations - clean macOS system
- Hot-reloading enabled for rapid development

**✅ Service Architecture**

- **Frontend:** Next.js 15 + React 19 (Port 3000)
- **Backend:** Django 5.2.5 + Python 3.13 (Port 8000)
- **Database:** PostgreSQL 17 with persistent storage
- **Cache:** Redis 7 for caching and task queuing
- **Workers:** Celery for background tasks
- **Monitoring:** Flower dashboard (Port 5555)

**✅ Development Workflow**

- One-command startup: `./scripts/dev-start.sh`
- Real-time code changes with hot-reload
- Comprehensive logging and monitoring
- Easy reset and cleanup scripts


### 🔗 Dependencies Created

**This phase enables:**

- **Phase 2:** Backend API development (Django ready)
- **Phase 3:** Frontend development (Next.js ready)
- **Phase 4:** Integration testing (both services communicating)
- **Phase 5:** Background task development (Celery ready)


### 🚀 Next Tutorial: Phase 2 - Backend API Development

**Upcoming features:**

- User registration and authentication APIs
- JWT token-based authentication
- Profile management with student/mentor types
- File upload capabilities
- GraphQL schema and resolvers
- Admin interface customization

***

## 7. Knowledge Check

### 📝 Q1: Understanding Check

**Can you explain what each Docker container does in our setup?**

**Expected Answer:**

- **db:** PostgreSQL database for persistent data storage
- **redis:** Cache and message broker for Celery tasks
- **backend:** Django API server with REST and GraphQL endpoints
- **frontend:** Next.js React application for user interface
- **celery:** Background task worker for processing async jobs
- **flower:** Monitoring dashboard for Celery workers


### 📝 Q2: Practical Application

**If you wanted to add a new Python package to the backend, what files would you need to update?**

**Expected Answer:**

- Add package to `backend/requirements.txt`
- Rebuild the backend container: `docker-compose build backend`
- Restart the service: `docker-compose restart backend`


### 📝 Q3: Development Workflow

**What's the correct sequence to make changes and see them reflected?**

**Expected Answer:**

- **Frontend:** Edit files → Save → Browser auto-refreshes
- **Backend:** Edit Python files → Save → Django auto-reloads
- **Dependencies:** Update requirements/package files → Rebuild containers


### ✅ Ready to Proceed?

**Checklist before moving to Phase 2:**

- [ ] All services running and healthy
- [ ] All localhost URLs accessible
- [ ] Hot-reloading working for both frontend and backend
- [ ] No errors in container logs
- [ ] Understanding of Docker container roles
- [ ] Comfortable with development scripts

**If all items are checked, you're ready for Phase 2: Backend API Development!**

***

**Congratulations! 🎉 Your Prepji development environment is now fully operational and ready for building the complete platform.**
<span style="display:none">[^1][^2][^3][^4]</span>

<div style="text-align: center">⁂</div>

[^1]: User-Types-and-Registration.md

[^2]: designdocument.md

[^3]: design_document_template.md

[^4]: image.jpg

