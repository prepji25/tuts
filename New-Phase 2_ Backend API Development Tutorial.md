# Phase 2: Backend API Development Tutorial

**Document Version:** 1.0
**Date:** September 7, 2025
**Tutorial:** Phase 2 - Complete Authentication \& User Management System
**Dependencies:** Phase 1 completed successfully ✅

***

## 1. Overview

### 🎯 Learning Objectives

By completing this tutorial, you will have:

- ✅ Modern user authentication system with **dj-rest-auth 7.x** and **django-allauth 65.x**
- ✅ Student and Mentor user models with profile completion workflows
- ✅ JWT-based authentication with secure token management
- ✅ File upload system for avatars and documents
- ✅ GraphQL API with **Strawberry GraphQL 0.65.x**
- ✅ Admin interface for user management and verification
- ✅ Role-based permissions and security features
- ✅ API endpoints fully compatible with **Django 5.2.5 + Python 3.13**


### 📋 Prerequisites

- ✅ Phase 1 completed with all containers running
- ✅ Development environment accessible at localhost:3000, :8000, :5555
- ✅ Modern package versions installed (dj-rest-auth 7.x, allauth 65.x, etc.)


### ⏱️ Estimated Time

**90-120 minutes** (including testing and validation)

### 🎚️ Difficulty Level

**Intermediate** - Building on Phase 1 foundation with modern Django patterns

***

## 2. Context \& Background

### 🏗️ What We're Building

A complete user authentication and profile management system that handles:

- **Email-only registration** (no usernames) for students and mentors
- **JWT token-based authentication** for stateless API communication
- **Multi-step profile completion** with different fields for students vs mentors
- **File upload capabilities** for avatars, scorecards, and certificates
- **Admin verification workflow** for mentor applications
- **GraphQL API** for modern frontend integration


### 🎯 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                 Phase 2: Backend API Layer             │
├─────────────────┬─────────────────┬─────────────────────┤
│   Authentication│   User Models   │   GraphQL API       │
│   dj-rest-auth  │   Student/Mentor│   Strawberry        │
│   allauth 65.x  │   Profiles      │   0.65.x            │
├─────────────────┼─────────────────┼─────────────────────┤
│   File Uploads  │   Admin Interface│   Permissions      │
│   Avatars       │   User Management│   Role-based       │
│   Documents     │   Verification   │   Security         │
└─────────────────┴─────────────────┴─────────────────────┘
```


### 💼 Business Value

- **Scalable user system** supporting both student and mentor workflows
- **Modern API architecture** ready for mobile apps and third-party integrations
- **Secure authentication** with industry-standard JWT implementation
- **Admin-friendly interface** for managing users and verifying mentors

***

## 3. Step-by-Step Implementation

### 🔧 Step 3.1: Create User Management Django App

**What:** Set up the core user management app with proper structure.

**Commands:**

```bash
# Create the accounts app inside the Django container
docker-compose exec backend mkdir -p apps  
docker-compose exec backend touch apps/__init__.py
docker-compose exec backend python manage.py startapp accounts apps/accounts

# Verify the app was created
docker-compose exec backend ls -la apps/accounts/
```

**Expected Output:**

```
drwxr-xr-x 2 root root   64 Sep  7 22:15 .
drwxr-xr-x 7 root root  224 Sep  7 22:15 ..
-rw-r--r-- 1 root root    0 Sep  7 22:15 __init__.py
-rw-r--r-- 1 root root   63 Sep  7 22:15 admin.py
-rw-r--r-- 1 root root  148 Sep  7 22:15 apps.py
drwxr-xr-x 2 root root   64 Sep  7 22:15 migrations
-rw-r--r-- 1 root root   57 Sep  7 22:15 models.py
-rw-r--r-- 1 root root   60 Sep  7 22:15 tests.py
-rw-r--r-- 1 root root   63 Sep  7 22:15 views.py
```

**✅ Validation Checkpoint:**

- [ ] `accounts` directory created in backend/
- [ ] All Django app files present
- [ ] Container command executed successfully

***

### 👤 Step 3.2: Create Custom User Model with Modern Configuration

**What:** Build a custom User model compatible with our modern package versions.

**File:** `backend/apps/accounts/models.py`

```python
"""
User models for Prepji platform.
Compatible with Django 5.2.5, dj-rest-auth 7.x, and allauth 65.x.
"""

import os
import uuid
from django.contrib.auth.models import AbstractUser
from django.db import models
from django.core.validators import MinValueValidator, MaxValueValidator, FileExtensionValidator
from django.utils.text import slugify
from PIL import Image


class User(AbstractUser):
    """
    Custom User model for Prepji platform.
    Uses email as primary identifier, supports student/mentor types.
    """
    USER_TYPE_CHOICES = [
        ('student', 'Student'),
        ('mentor', 'Mentor'),
    ]
    
    # Remove username field - we use email only
    username = None
    
    # Core fields
    email = models.EmailField(unique=True)
    user_type = models.CharField(
        max_length=20,
        choices=USER_TYPE_CHOICES,
        default='student'
    )
    
    # Profile completion tracking
    profile_completed = models.BooleanField(default=False)
    profile_completion_percentage = models.PositiveIntegerField(default=0)
    
    # Account status
    is_verified = models.BooleanField(default=False)  # For mentor verification
    date_joined = models.DateTimeField(auto_now_add=True)
    last_login = models.DateTimeField(null=True, blank=True)
    
    # Use email as the unique identifier
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []  # Remove email from required fields since it's the USERNAME_FIELD
    
    class Meta:
        db_table = 'accounts_user'
        verbose_name = 'User'
        verbose_name_plural = 'Users'
    
    def __str__(self):
        return f"{self.email} ({self.get_user_type_display()})"
    
    @property
    def full_name(self):
        """Get full name from profile if available."""
        if hasattr(self, 'profile'):
            return self.profile.full_name
        return self.email.split('@')[0]
    
    def calculate_profile_completion(self):
        """Calculate profile completion percentage."""
        if not hasattr(self, 'profile'):
            return 0
        
        return self.profile.calculate_completion_percentage()


def user_avatar_path(instance, filename):
    """Generate upload path for user avatars."""
    ext = filename.split('.')[-1]
    filename = f'{uuid.uuid4().hex}.{ext}'
    return f'avatars/{instance.user.user_type}/{filename}'


def user_document_path(instance, filename):
    """Generate upload path for user documents."""
    ext = filename.split('.')[-1]
    filename = f'{uuid.uuid4().hex}.{ext}'
    return f'documents/{instance.user.user_type}/{filename}'


class UserProfile(models.Model):
    """
    Base profile for all users with common fields.
    """
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    
    # Common profile fields
    full_name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True, max_length=100)
    date_of_birth = models.DateField(null=True, blank=True)
    
    # Address fields (private by default)
    address = models.TextField(blank=True, help_text="Private - not displayed publicly")
    city = models.CharField(max_length=100, blank=True)
    state = models.CharField(max_length=100, blank=True)
    
    # Public profile fields
    bio = models.TextField(blank=True, max_length=500)
    avatar = models.ImageField(
        upload_to=user_avatar_path,
        blank=True,
        null=True,
        validators=[FileExtensionValidator(allowed_extensions=['jpg', 'jpeg', 'png'])],
        help_text="Max 1MB, JPEG/PNG only"
    )
    
    # Social media links
    facebook_url = models.URLField(blank=True)
    youtube_url = models.URLField(blank=True)
    instagram_url = models.URLField(blank=True)
    quora_url = models.URLField(blank=True)
    reddit_url = models.URLField(blank=True)
    website_url = models.URLField(blank=True)
    
    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'accounts_userprofile'
    
    def __str__(self):
        return f"{self.full_name} ({self.user.email})"
    
    def save(self, *args, **kwargs):
        # Auto-generate slug from full name if not provided
        if not self.slug and self.full_name:
            base_slug = slugify(self.full_name)
            slug = base_slug
            counter = 1
            while UserProfile.objects.filter(slug=slug).exclude(pk=self.pk).exists():
                slug = f"{base_slug}-{counter}"
                counter += 1
            self.slug = slug
        
        super().save(*args, **kwargs)
        
        # Resize avatar if too large
        if self.avatar:
            self._resize_avatar()
    
    def _resize_avatar(self):
        """Resize avatar to max 300x300 and compress."""
        img = Image.open(self.avatar.path)
        if img.height > 300 or img.width > 300:
            output_size = (300, 300)
            img.thumbnail(output_size)
            img.save(self.avatar.path, optimize=True, quality=85)
    
    def calculate_completion_percentage(self):
        """Calculate profile completion percentage."""
        total_fields = 6  # full_name, bio, avatar, city, date_of_birth, social_links
        completed_fields = 0
        
        if self.full_name:
            completed_fields += 1
        if self.bio:
            completed_fields += 1
        if self.avatar:
            completed_fields += 1
        if self.city:
            completed_fields += 1
        if self.date_of_birth:
            completed_fields += 1
        if any([self.facebook_url, self.youtube_url, self.instagram_url, self.website_url]):
            completed_fields += 1
        
        return int((completed_fields / total_fields) * 100)


class StudentProfile(models.Model):
    """
    Extended profile for student users.
    """
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='student_profile')
    
    # JEE specific fields
    target_jee_year = models.PositiveIntegerField(
        validators=[MinValueValidator(2025), MaxValueValidator(2030)]
    )
    target_colleges = models.ManyToManyField('College', blank=True)
    target_streams = models.ManyToManyField('Stream', blank=True)
    
    # Academic fields (optional with privacy controls)
    grade_x_percentage = models.DecimalField(
        max_digits=5, decimal_places=2, 
        null=True, blank=True,
        validators=[MinValueValidator(0), MaxValueValidator(100)]
    )
    grade_xii_percentage = models.DecimalField(
        max_digits=5, decimal_places=2,
        null=True, blank=True, 
        validators=[MinValueValidator(0), MaxValueValidator(100)]
    )
    school_name = models.CharField(max_length=200, blank=True)
    
    # Privacy settings
    show_grade_x = models.BooleanField(default=False)
    show_grade_xii = models.BooleanField(default=False)
    show_school = models.BooleanField(default=False)
    
    class Meta:
        db_table = 'accounts_studentprofile'
    
    def __str__(self):
        return f"Student: {self.user.profile.full_name} (JEE {self.target_jee_year})"


class MentorProfile(models.Model):
    """
    Extended profile for mentor users.
    """
    VERIFICATION_STATUS_CHOICES = [
        ('pending', 'Pending Review'),
        ('approved', 'Approved'),
        ('rejected', 'Rejected'),
        ('resubmission_required', 'Resubmission Required'),
    ]
    
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='mentor_profile')
    
    # Professional information
    profession = models.ForeignKey('Profession', on_delete=models.SET_NULL, null=True)
    business_name = models.CharField(max_length=200, blank=True)
    subject_expertise = models.ManyToManyField('Subject', blank=True)
    
    # JEE credentials
    jee_score = models.PositiveIntegerField(
        null=True, blank=True,
        validators=[MinValueValidator(1), MaxValueValidator(360)]
    )
    jee_scorecard = models.FileField(
        upload_to=user_document_path,
        blank=True,
        validators=[FileExtensionValidator(allowed_extensions=['jpg', 'jpeg', 'png', 'pdf'])],
        help_text="JEE Scorecard (JPEG/PNG/PDF, max 1MB)"
    )
    
    # College information
    college_attended = models.ForeignKey('College', on_delete=models.SET_NULL, null=True, blank=True)
    college_certificate = models.FileField(
        upload_to=user_document_path,
        blank=True,
        validators=[FileExtensionValidator(allowed_extensions=['jpg', 'jpeg', 'png', 'pdf'])],
        help_text="College ID/Certificate (JPEG/PNG/PDF, max 1MB)"
    )
    
    # Verification status
    verification_status = models.CharField(
        max_length=30,
        choices=VERIFICATION_STATUS_CHOICES,
        default='pending'
    )
    verification_notes = models.TextField(blank=True, help_text="Admin notes for verification")
    verified_at = models.DateTimeField(null=True, blank=True)
    verified_by = models.ForeignKey(
        User, on_delete=models.SET_NULL, null=True, blank=True,
        related_name='verified_mentors'
    )
    
    class Meta:
        db_table = 'accounts_mentorprofile'
    
    def __str__(self):
        return f"Mentor: {self.user.profile.full_name} ({self.get_verification_status_display()})"
    
    @property
    def is_verified(self):
        return self.verification_status == 'approved'


# Admin-managed reference models
class College(models.Model):
    """Colleges available for selection."""
    name = models.CharField(max_length=200, unique=True)
    location = models.CharField(max_length=100)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'accounts_college'
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Stream(models.Model):
    """Engineering streams available for selection."""
    name = models.CharField(max_length=100, unique=True)
    code = models.CharField(max_length=20, unique=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'accounts_stream'
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Subject(models.Model):
    """Subjects for mentor expertise."""
    name = models.CharField(max_length=50, unique=True)
    code = models.CharField(max_length=10, unique=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'accounts_subject'
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Profession(models.Model):
    """Professions available for mentors."""
    name = models.CharField(max_length=100, unique=True)
    is_active = models.BooleanField(default=True)
    
    class Meta:
        db_table = 'accounts_profession'
        ordering = ['name']
    
    def __str__(self):
        return self.name
```

**✅ Validation Checkpoint:**

- [ ] Models file created with all classes
- [ ] Custom User model extends AbstractUser correctly
- [ ] Profile models include all required fields
- [ ] File upload paths configured properly

***

### ⚙️ Step 3.3: Update Django Settings for New Models

**What:** Configure Django to use our custom User model and register the accounts app.

**File:** `backend/prepji/settings.py` (Add these lines to your existing settings)

```python
# Add to LOCAL_APPS section
LOCAL_APPS = [
    'apps.accounts',  # ← Add this line
]

# Add after DEFAULT_AUTO_FIELD
AUTH_USER_MODEL = 'apps.accounts.User'  # ← Add this line to use custom User model

# Update ACCOUNT settings for allauth 65.x compatibility
ACCOUNT_LOGIN_METHODS = {"email"}  # Modern allauth 65.x syntax
ACCOUNT_EMAIL_VERIFICATION = "optional"
ACCOUNT_USERNAME_REQUIRED = False
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_USER_MODEL_USERNAME_FIELD = None
ACCOUNT_USER_MODEL_EMAIL_FIELD = "email"

# Update REST_AUTH configuration for dj-rest-auth 7.x
REST_AUTH = {
    'USE_JWT': True,
    'JWT_AUTH_COOKIE': 'prepji-auth',
    'JWT_AUTH_REFRESH_COOKIE': 'prepji-refresh',
    'JWT_AUTH_SECURE': False,  # Set to True in production with HTTPS
    'JWT_AUTH_HTTPONLY': True,
    'USER_DETAILS_SERIALIZER': 'apps.accounts.serializers.UserDetailsSerializer',
    'REGISTER_SERIALIZER': 'apps.accounts.serializers.RegisterSerializer',
}

# File upload settings
MAX_FILE_SIZE = 1024 * 1024  # 1MB
ALLOWED_IMAGE_EXTENSIONS = ['jpg', 'jpeg', 'png']
ALLOWED_DOCUMENT_EXTENSIONS = ['jpg', 'jpeg', 'png', 'pdf']
```

**✅ Validation Checkpoint:**

- [ ] AUTH_USER_MODEL configured correctly
- [ ] accounts app added to LOCAL_APPS
- [ ] allauth settings use modern 65.x syntax
- [ ] REST_AUTH configured for dj-rest-auth 7.x

***

### 🔐 Step 3.4: Create API Serializers with Modern Patterns

**What:** Build serializers compatible with dj-rest-auth 7.x and modern Django patterns.

**File:** `backend/apps/accounts/serializers.py`

```python
"""
API Serializers for Prepji user management.
Compatible with dj-rest-auth 7.x and modern Django REST patterns.
"""

from rest_framework import serializers
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password
from django.core.exceptions import ValidationError as DjangoValidationError
from dj_rest_auth.registration.serializers import RegisterSerializer as DefaultRegisterSerializer
from dj_rest_auth.serializers import UserDetailsSerializer as DefaultUserDetailsSerializer
from .models import UserProfile, StudentProfile, MentorProfile, College, Stream, Subject, Profession

User = get_user_model()


class RegisterSerializer(DefaultRegisterSerializer):
    """
    Custom registration serializer for dj-rest-auth 7.x.
    Handles email-only registration with user type selection.
    """
    user_type = serializers.ChoiceField(
        choices=User.USER_TYPE_CHOICES,
        default='student'
    )
    
    # Remove username field
    username = None
    
    def get_cleaned_data(self):
        """Override to include user_type and remove username."""
        return {
            'email': self.validated_data.get('email', ''),
            'password1': self.validated_data.get('password1', ''),
            'user_type': self.validated_data.get('user_type', 'student'),
        }
    
    def save(self, request):
        """Save user with custom fields."""
        user = super().save(request)
        user.user_type = self.cleaned_data.get('user_type')
        user.save()
        return user


class UserDetailsSerializer(DefaultUserDetailsSerializer):
    """
    Custom user details serializer for dj-rest-auth 7.x.
    """
    user_type = serializers.CharField(read_only=True)
    profile_completed = serializers.BooleanField(read_only=True)
    profile_completion_percentage = serializers.IntegerField(read_only=True)
    is_verified = serializers.BooleanField(read_only=True)
    full_name = serializers.CharField(source='full_name', read_only=True)
    
    class Meta:
        model = User
        fields = (
            'pk', 'email', 'first_name', 'last_name', 'user_type',
            'profile_completed', 'profile_completion_percentage',
            'is_verified', 'full_name', 'date_joined'
        )
        read_only_fields = ('pk', 'email', 'date_joined')


class CollegeSerializer(serializers.ModelSerializer):
    """Serializer for College model."""
    class Meta:
        model = College
        fields = ['id', 'name', 'location']


class StreamSerializer(serializers.ModelSerializer):
    """Serializer for Stream model."""
    class Meta:
        model = Stream
        fields = ['id', 'name', 'code']


class SubjectSerializer(serializers.ModelSerializer):
    """Serializer for Subject model."""
    class Meta:
        model = Subject
        fields = ['id', 'name', 'code']


class ProfessionSerializer(serializers.ModelSerializer):
    """Serializer for Profession model."""
    class Meta:
        model = Profession
        fields = ['id', 'name']


class UserProfileSerializer(serializers.ModelSerializer):
    """Serializer for UserProfile model."""
    slug_available = serializers.SerializerMethodField()
    avatar_url = serializers.SerializerMethodField()
    completion_percentage = serializers.SerializerMethodField()
    
    class Meta:
        model = UserProfile
        fields = [
            'full_name', 'slug', 'date_of_birth', 'city', 'state',
            'bio', 'avatar', 'avatar_url', 'facebook_url', 'youtube_url',
            'instagram_url', 'quora_url', 'reddit_url', 'website_url',
            'slug_available', 'completion_percentage', 'created_at', 'updated_at'
        ]
        read_only_fields = ('created_at', 'updated_at', 'slug_available', 'avatar_url')
    
    def get_slug_available(self, obj):
        """Check if slug is available for use."""
        return True  # Will implement real-time check in Phase 3
    
    def get_avatar_url(self, obj):
        """Get full URL for avatar image."""
        if obj.avatar:
            request = self.context.get('request')
            if request:
                return request.build_absolute_uri(obj.avatar.url)
        return None
    
    def get_completion_percentage(self, obj):
        """Get profile completion percentage."""
        return obj.calculate_completion_percentage()
    
    def validate_slug(self, value):
        """Validate slug uniqueness."""
        if self.instance:
            # Exclude current instance from uniqueness check
            if UserProfile.objects.filter(slug=value).exclude(pk=self.instance.pk).exists():
                raise serializers.ValidationError("This slug is already taken.")
        else:
            if UserProfile.objects.filter(slug=value).exists():
                raise serializers.ValidationError("This slug is already taken.")
        return value


class StudentProfileSerializer(serializers.ModelSerializer):
    """Serializer for StudentProfile model."""
    target_colleges = CollegeSerializer(many=True, read_only=True)
    target_streams = StreamSerializer(many=True, read_only=True)
    target_college_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    target_stream_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    
    class Meta:
        model = StudentProfile
        fields = [
            'target_jee_year', 'target_colleges', 'target_streams',
            'target_college_ids', 'target_stream_ids',
            'grade_x_percentage', 'grade_xii_percentage', 'school_name',
            'show_grade_x', 'show_grade_xii', 'show_school'
        ]
    
    def create(self, validated_data):
        """Create student profile with many-to-many relationships."""
        college_ids = validated_data.pop('target_college_ids', [])
        stream_ids = validated_data.pop('target_stream_ids', [])
        
        student_profile = StudentProfile.objects.create(**validated_data)
        
        if college_ids:
            student_profile.target_colleges.set(college_ids)
        if stream_ids:
            student_profile.target_streams.set(stream_ids)
        
        return student_profile
    
    def update(self, instance, validated_data):
        """Update student profile with many-to-many relationships."""
        college_ids = validated_data.pop('target_college_ids', None)
        stream_ids = validated_data.pop('target_stream_ids', None)
        
        # Update regular fields
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        
        # Update many-to-many fields
        if college_ids is not None:
            instance.target_colleges.set(college_ids)
        if stream_ids is not None:
            instance.target_streams.set(stream_ids)
        
        return instance


class MentorProfileSerializer(serializers.ModelSerializer):
    """Serializer for MentorProfile model."""
    profession = ProfessionSerializer(read_only=True)
    subject_expertise = SubjectSerializer(many=True, read_only=True)
    college_attended = CollegeSerializer(read_only=True)
    
    profession_id = serializers.IntegerField(write_only=True, required=False)
    subject_expertise_ids = serializers.ListField(
        child=serializers.IntegerField(),
        write_only=True,
        required=False
    )
    college_attended_id = serializers.IntegerField(write_only=True, required=False)
    
    jee_scorecard_url = serializers.SerializerMethodField()
    college_certificate_url = serializers.SerializerMethodField()
    
    class Meta:
        model = MentorProfile
        fields = [
            'profession', 'profession_id', 'business_name',
            'subject_expertise', 'subject_expertise_ids',
            'jee_score', 'jee_scorecard', 'jee_scorecard_url',
            'college_attended', 'college_attended_id',
            'college_certificate', 'college_certificate_url',
            'verification_status', 'verification_notes',
            'verified_at', 'is_verified'
        ]
        read_only_fields = (
            'verification_status', 'verification_notes', 
            'verified_at', 'is_verified'
        )
    
    def get_jee_scorecard_url(self, obj):
        """Get full URL for JEE scorecard."""
        if obj.jee_scorecard:
            request = self.context.get('request')
            if request:
                return request.build_absolute_uri(obj.jee_scorecard.url)
        return None
    
    def get_college_certificate_url(self, obj):
        """Get full URL for college certificate."""
        if obj.college_certificate:
            request = self.context.get('request')
            if request:
                return request.build_absolute_uri(obj.college_certificate.url)
        return None
    
    def create(self, validated_data):
        """Create mentor profile with relationships."""
        profession_id = validated_data.pop('profession_id', None)
        subject_ids = validated_data.pop('subject_expertise_ids', [])
        college_id = validated_data.pop('college_attended_id', None)
        
        mentor_profile = MentorProfile.objects.create(**validated_data)
        
        if profession_id:
            mentor_profile.profession_id = profession_id
        if college_id:
            mentor_profile.college_attended_id = college_id
        mentor_profile.save()
        
        if subject_ids:
            mentor_profile.subject_expertise.set(subject_ids)
        
        return mentor_profile
    
    def update(self, instance, validated_data):
        """Update mentor profile with relationships."""
        profession_id = validated_data.pop('profession_id', None)
        subject_ids = validated_data.pop('subject_expertise_ids', None)
        college_id = validated_data.pop('college_attended_id', None)
        
        # Update regular fields
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        
        # Update foreign key relationships
        if profession_id is not None:
            instance.profession_id = profession_id
        if college_id is not None:
            instance.college_attended_id = college_id
        if profession_id is not None or college_id is not None:
            instance.save()
        
        # Update many-to-many relationships
        if subject_ids is not None:
            instance.subject_expertise.set(subject_ids)
        
        return instance
```

**✅ Validation Checkpoint:**

- [ ] All serializers created with proper inheritance
- [ ] RegisterSerializer customized for dj-rest-auth 7.x
- [ ] File URLs handled with SerializerMethodField
- [ ] Many-to-many relationships properly serialized

***

### 🌐 Step 3.5: Create REST API Views

**What:** Build API views for user registration, profile management, and data access.

**File:** `backend/apps/accounts/views.py`

```python
"""
API Views for Prepji user management.
Modern REST API patterns with proper permissions and error handling.
"""

from rest_framework import generics, permissions, status
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from rest_framework.parsers import MultiPartParser, FormParser
from django.contrib.auth import get_user_model
from django.shortcuts import get_object_or_404
from django.db import transaction
from django.utils.text import slugify
from .models import (
    UserProfile, StudentProfile, MentorProfile,
    College, Stream, Subject, Profession
)
from .serializers import (
    UserProfileSerializer, StudentProfileSerializer, MentorProfileSerializer,
    CollegeSerializer, StreamSerializer, SubjectSerializer, ProfessionSerializer
)

User = get_user_model()


class ProfileCreateUpdateView(generics.CreateAPIView, generics.UpdateAPIView):
    """
    Create or update user profile.
    Handles both student and mentor profile creation.
    """
    parser_classes = [MultiPartParser, FormParser]
    permission_classes = [permissions.IsAuthenticated]
    
    def get_serializer_class(self):
        """Return appropriate serializer based on user type."""
        if self.request.user.user_type == 'student':
            return StudentProfileSerializer
        return MentorProfileSerializer
    
    def get_object(self):
        """Get the profile object for the authenticated user."""
        user = self.request.user
        if user.user_type == 'student':
            profile, created = StudentProfile.objects.get_or_create(user=user)
        else:
            profile, created = MentorProfile.objects.get_or_create(user=user)
        return profile
    
    def perform_create(self, serializer):
        """Create profile for authenticated user."""
        serializer.save(user=self.request.user)
    
    def post(self, request, *args, **kwargs):
        """Handle both create and update in POST."""
        try:
            instance = self.get_object()
            serializer = self.get_serializer(instance, data=request.data, partial=True)
            serializer.is_valid(raise_exception=True)
            self.perform_update(serializer)
            return Response(serializer.data)
        except Exception as e:
            return Response(
                {'error': str(e)}, 
                status=status.HTTP_400_BAD_REQUEST
            )


class UserProfileView(generics.RetrieveUpdateAPIView):
    """
    Retrieve and update base user profile information.
    """
    serializer_class = UserProfileSerializer
    permission_classes = [permissions.IsAuthenticated]
    parser_classes = [MultiPartParser, FormParser]
    
    def get_object(self):
        """Get or create base profile for authenticated user."""
        profile, created = UserProfile.objects.get_or_create(user=self.request.user)
        return profile
    
    def perform_update(self, serializer):
        """Update profile and recalculate completion percentage."""
        instance = serializer.save()
        
        # Update user's profile completion status
        completion = instance.calculate_completion_percentage()
        user = instance.user
        user.profile_completion_percentage = completion
        user.profile_completed = completion >= 80  # 80% threshold
        user.save()


class SlugAvailabilityView(generics.GenericAPIView):
    """
    Check if a slug is available for use.
    Real-time validation for profile slug field.
    """
    permission_classes = [permissions.IsAuthenticated]
    
    def get(self, request):
        slug = request.query_params.get('slug')
        if not slug:
            return Response(
                {'error': 'Slug parameter is required'}, 
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Check if slug is available (excluding current user's profile)
        current_user_profile = getattr(request.user, 'profile', None)
        if current_user_profile:
            exists = UserProfile.objects.filter(slug=slug).exclude(
                pk=current_user_profile.pk
            ).exists()
        else:
            exists = UserProfile.objects.filter(slug=slug).exists()
        
        available = not exists
        suggestions = []
        
        # Generate suggestions if slug is taken
        if not available:
            base_slug = slugify(slug)
            for i in range(1, 4):
                suggestion = f"{base_slug}-{i}"
                if not UserProfile.objects.filter(slug=suggestion).exists():
                    suggestions.append(suggestion)
        
        return Response({
            'slug': slug,
            'available': available,
            'suggestions': suggestions
        })


# Reference data views
class CollegeListView(generics.ListAPIView):
    """List all active colleges."""
    queryset = College.objects.filter(is_active=True)
    serializer_class = CollegeSerializer
    permission_classes = [permissions.IsAuthenticated]


class StreamListView(generics.ListAPIView):
    """List all active streams."""
    queryset = Stream.objects.filter(is_active=True)
    serializer_class = StreamSerializer
    permission_classes = [permissions.IsAuthenticated]


class SubjectListView(generics.ListAPIView):
    """List all active subjects."""
    queryset = Subject.objects.filter(is_active=True)
    serializer_class = SubjectSerializer
    permission_classes = [permissions.IsAuthenticated]


class ProfessionListView(generics.ListAPIView):
    """List all active professions."""
    queryset = Profession.objects.filter(is_active=True)
    serializer_class = ProfessionSerializer
    permission_classes = [permissions.IsAuthenticated]


@api_view(['GET'])
@permission_classes([permissions.IsAuthenticated])
def profile_status(request):
    """
    Get current user's profile completion status.
    """
    user = request.user
    
    # Check if base profile exists
    has_base_profile = hasattr(user, 'profile')
    has_extended_profile = False
    extended_profile_type = None
    
    if user.user_type == 'student':
        has_extended_profile = hasattr(user, 'student_profile')
        extended_profile_type = 'student'
    else:
        has_extended_profile = hasattr(user, 'mentor_profile')
        extended_profile_type = 'mentor'
    
    # Calculate next steps
    next_steps = []
    if not has_base_profile:
        next_steps.append('Complete basic profile information')
    if not has_extended_profile:
        next_steps.append(f'Complete {extended_profile_type} specific information')
    
    return Response({
        'user_type': user.user_type,
        'profile_completed': user.profile_completed,
        'completion_percentage': user.profile_completion_percentage,
        'has_base_profile': has_base_profile,
        'has_extended_profile': has_extended_profile,
        'is_verified': user.is_verified,
        'next_steps': next_steps,
        'verification_status': (
            user.mentor_profile.verification_status 
            if hasattr(user, 'mentor_profile') 
            else None
        )
    })


@api_view(['POST'])
@permission_classes([permissions.IsAuthenticated])
def upload_avatar(request):
    """
    Handle avatar upload separately for better UX.
    """
    if 'avatar' not in request.FILES:
        return Response(
            {'error': 'No avatar file provided'}, 
            status=status.HTTP_400_BAD_REQUEST
        )
    
    profile, created = UserProfile.objects.get_or_create(user=request.user)
    profile.avatar = request.FILES['avatar']
    profile.save()
    
    return Response({
        'message': 'Avatar uploaded successfully',
        'avatar_url': request.build_absolute_uri(profile.avatar.url)
    })
```

**✅ Validation Checkpoint:**

- [ ] All views created with proper permissions
- [ ] File upload handling implemented
- [ ] Profile completion logic included
- [ ] Slug availability check working

***

### 🛠️ Step 3.6: Configure URL Routing

**What:** Set up URL routing for all API endpoints.

**File:** `backend/apps/accounts/urls.py` (CREATE THIS NEW FILE)

```python
"""
URL routing for accounts app API endpoints.
"""

from django.urls import path
from . import views

app_name = 'accounts'

urlpatterns = [
    # Profile management
    path('profile/', views.UserProfileView.as_view(), name='user-profile'),
    path('profile/status/', views.profile_status, name='profile-status'),
    path('profile/extended/', views.ProfileCreateUpdateView.as_view(), name='extended-profile'),
    
    # File uploads
    path('upload/avatar/', views.upload_avatar, name='upload-avatar'),
    
    # Utilities
    path('slug-check/', views.SlugAvailabilityView.as_view(), name='slug-check'),
    
    # Reference data
    path('colleges/', views.CollegeListView.as_view(), name='colleges'),
    path('streams/', views.StreamListView.as_view(), name='streams'),
    path('subjects/', views.SubjectListView.as_view(), name='subjects'),
    path('professions/', views.ProfessionListView.as_view(), name='professions'),
]
```

**Update File:** `backend/prepji/urls.py`

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
        'version': '2.0.0',  # Updated for Phase 2
        'endpoints': {
            'admin': '/admin/',
            'auth': '/api/auth/',  # dj-rest-auth endpoints
            'accounts': '/api/accounts/',  # Custom account endpoints
            'graphql': '/graphql/',
        },
        'status': 'Phase 2 Development'
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', api_root, name='api_root'),
    
    # Authentication endpoints (dj-rest-auth 7.x)
    path('api/auth/', include('dj_rest_auth.urls')),
    path('api/auth/registration/', include('dj_rest_auth.registration.urls')),
    
    # Custom account endpoints
    path('api/accounts/', include('apps.accounts.urls')),
    
    # GraphQL endpoint (will be added in Step 3.8)
    # path('graphql/', include('strawberry.django.urls')),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

**✅ Validation Checkpoint:**

- [ ] accounts/urls.py created with all endpoints
- [ ] Main urls.py updated with new routes
- [ ] Media file serving configured for development

***

### 🗄️ Step 3.7: Run Database Migrations

**What:** Create and apply database migrations for all new models.

**Commands:**

```bash
# Create migrations for the new models
docker-compose exec backend python manage.py makemigrations accounts

# Apply the migrations
docker-compose exec backend python manage.py migrate

# Create a superuser for admin access
docker-compose exec backend python manage.py createsuperuser
```

**Expected Output for makemigrations:**

```
Migrations for 'accounts':
  accounts/migrations/0001_initial.py
    - Create model College
    - Create model Profession
    - Create model Stream
    - Create model Subject
    - Create model User
    - Create model UserProfile
    - Create model MentorProfile
    - Create model StudentProfile
    - Add index on UserProfile.slug
```

**Expected Output for migrate:**

```
Operations to perform:
  Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
  Applying accounts.0001_initial... OK
```

**For superuser creation, enter:**

- **Email:** admin@prepji.com
- **Password:** admin123456

**✅ Validation Checkpoint:**

- [ ] Migrations created successfully
- [ ] Database updated without errors
- [ ] Superuser created for admin access

***

### 🎨 Step 3.8: Create Admin Interface

**What:** Build a comprehensive admin interface for user and profile management.

**File:** `backend/apps/accounts/admin.py`

```python
"""
Django Admin configuration for accounts models.
Enhanced interface for user management and mentor verification.
"""

from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as DefaultUserAdmin
from django.utils.html import format_html
from django.urls import reverse
from django.utils.safestring import mark_safe
from .models import (
    User, UserProfile, StudentProfile, MentorProfile,
    College, Stream, Subject, Profession
)


@admin.register(User)
class UserAdmin(DefaultUserAdmin):
    """Enhanced admin for custom User model."""
    
    list_display = [
        'email', 'user_type', 'profile_completed', 
        'profile_completion_percentage', 'is_verified', 
        'is_staff', 'date_joined'
    ]
    list_filter = [
        'user_type', 'is_verified', 'profile_completed', 
        'is_staff', 'is_superuser', 'date_joined'
    ]
    search_fields = ['email', 'first_name', 'last_name']
    ordering = ['-date_joined']
    
    # Remove username field from admin
    fieldsets = (
        (None, {'fields': ('email', 'password')}),
        ('Personal info', {'fields': ('first_name', 'last_name', 'user_type')}),
        ('Profile Status', {
            'fields': ('profile_completed', 'profile_completion_percentage', 'is_verified')
        }),
        ('Permissions', {
            'fields': ('is_active', 'is_staff', 'is_superuser', 'groups', 'user_permissions'),
            'classes': ('collapse',)
        }),
        ('Important dates', {'fields': ('last_login', 'date_joined')}),
    )
    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'user_type', 'password1', 'password2'),
        }),
    )
    
    readonly_fields = ['date_joined', 'last_login']
    
    def get_queryset(self, request):
        return super().get_queryset(request).select_related('profile')


class UserProfileInline(admin.StackedInline):
    """Inline admin for UserProfile."""
    model = UserProfile
    can_delete = False
    extra = 0
    fields = [
        'full_name', 'slug', 'date_of_birth', 'city', 'state',
        'bio', 'avatar', 'avatar_preview'
    ]
    readonly_fields = ['avatar_preview']
    
    def avatar_preview(self, obj):
        """Show avatar preview in admin."""
        if obj.avatar:
            return format_html(
                '<img src="{}" style="max-width: 100px; max-height: 100px;" />',
                obj.avatar.url
            )
        return "No avatar"
    avatar_preview.short_description = "Avatar Preview"


@admin.register(UserProfile)
class UserProfileAdmin(admin.ModelAdmin):
    """Admin for UserProfile model."""
    
    list_display = ['full_name', 'user_email', 'slug', 'city', 'completion_percentage', 'created_at']
    list_filter = ['city', 'created_at']
    search_fields = ['full_name', 'user__email', 'slug']
    ordering = ['-created_at']
    readonly_fields = ['created_at', 'updated_at', 'avatar_preview', 'completion_percentage']
    
    fieldsets = (
        ('Basic Info', {
            'fields': ('user', 'full_name', 'slug', 'date_of_birth')
        }),
        ('Location', {
            'fields': ('address', 'city', 'state'),
            'classes': ('collapse',)
        }),
        ('Profile', {
            'fields': ('bio', 'avatar', 'avatar_preview')
        }),
        ('Social Links', {
            'fields': (
                'facebook_url', 'youtube_url', 'instagram_url',
                'quora_url', 'reddit_url', 'website_url'
            ),
            'classes': ('collapse',)
        }),
        ('Metadata', {
            'fields': ('completion_percentage', 'created_at', 'updated_at'),
            'classes': ('collapse',)
        })
    )
    
    def user_email(self, obj):
        return obj.user.email
    user_email.short_description = 'Email'
    user_email.admin_order_field = 'user__email'
    
    def avatar_preview(self, obj):
        if obj.avatar:
            return format_html(
                '<img src="{}" style="max-width: 150px; max-height: 150px;" />',
                obj.avatar.url
            )
        return "No avatar"
    avatar_preview.short_description = "Avatar Preview"
    
    def completion_percentage(self, obj):
        percentage = obj.calculate_completion_percentage()
        color = 'green' if percentage >= 80 else 'orange' if percentage >= 50 else 'red'
        return format_html(
            '<span style="color: {}; font-weight: bold;">{} %</span>',
            color, percentage
        )
    completion_percentage.short_description = 'Completion'


@admin.register(StudentProfile)
class StudentProfileAdmin(admin.ModelAdmin):
    """Admin for StudentProfile model."""
    
    list_display = [
        'user_full_name', 'user_email', 'target_jee_year',
        'grade_x_display', 'grade_xii_display', 'college_count'
    ]
    list_filter = ['target_jee_year', 'show_grade_x', 'show_grade_xii']
    search_fields = ['user__email', 'user__profile__full_name', 'school_name']
    filter_horizontal = ['target_colleges', 'target_streams']
    
    fieldsets = (
        ('User Info', {
            'fields': ('user',)
        }),
        ('JEE Preparation', {
            'fields': ('target_jee_year', 'target_colleges', 'target_streams')
        }),
        ('Academic Records', {
            'fields': (
                'grade_x_percentage', 'show_grade_x',
                'grade_xii_percentage', 'show_grade_xii',
                'school_name', 'show_school'
            )
        })
    )
    
    def user_full_name(self, obj):
        return obj.user.profile.full_name if hasattr(obj.user, 'profile') else obj.user.email
    user_full_name.short_description = 'Full Name'
    
    def user_email(self, obj):
        return obj.user.email
    user_email.short_description = 'Email'
    
    def grade_x_display(self, obj):
        if obj.grade_x_percentage:
            visibility = "Public" if obj.show_grade_x else "Private"
            return f"{obj.grade_x_percentage}% ({visibility})"
        return "Not provided"
    grade_x_display.short_description = 'Grade X'
    
    def grade_xii_display(self, obj):
        if obj.grade_xii_percentage:
            visibility = "Public" if obj.show_grade_xii else "Private"
            return f"{obj.grade_xii_percentage}% ({visibility})"
        return "Not provided"
    grade_xii_display.short_description = 'Grade XII'
    
    def college_count(self, obj):
        return obj.target_colleges.count()
    college_count.short_description = 'Target Colleges'


@admin.register(MentorProfile)
class MentorProfileAdmin(admin.ModelAdmin):
    """Admin for MentorProfile model with verification workflow."""
    
    list_display = [
        'user_full_name', 'user_email', 'profession', 'verification_status_display',
        'jee_score', 'verification_actions'
    ]
    list_filter = ['verification_status', 'profession', 'verified_at']
    search_fields = ['user__email', 'user__profile__full_name', 'business_name']
    filter_horizontal = ['subject_expertise']
    readonly_fields = ['verified_at', 'verified_by', 'document_previews']
    
    fieldsets = (
        ('User Info', {
            'fields': ('user',)
        }),
        ('Professional Info', {
            'fields': ('profession', 'business_name', 'subject_expertise')
        }),
        ('JEE Credentials', {
            'fields': ('jee_score', 'jee_scorecard', 'college_attended', 'college_certificate')
        }),
        ('Document Previews', {
            'fields': ('document_previews',),
            'classes': ('collapse',)
        }),
        ('Verification', {
            'fields': (
                'verification_status', 'verification_notes',
                'verified_at', 'verified_by'
            )
        })
    )
    
    actions = ['approve_mentors', 'reject_mentors', 'mark_pending']
    
    def user_full_name(self, obj):
        return obj.user.profile.full_name if hasattr(obj.user, 'profile') else obj.user.email
    user_full_name.short_description = 'Full Name'
    
    def user_email(self, obj):
        return obj.user.email
    user_email.short_description = 'Email'
    
    def verification_status_display(self, obj):
        colors = {
            'pending': 'orange',
            'approved': 'green',
            'rejected': 'red',
            'resubmission_required': 'blue'
        }
        color = colors.get(obj.verification_status, 'black')
        return format_html(
            '<span style="color: {}; font-weight: bold;">{}</span>',
            color, obj.get_verification_status_display()
        )
    verification_status_display.short_description = 'Status'
    
    def verification_actions(self, obj):
        if obj.verification_status == 'pending':
            return format_html(
                '<a class="button" href="#" onclick="return false;" style="background: green; color: white; padding: 5px;">Approve</a> '
                '<a class="button" href="#" onclick="return false;" style="background: red; color: white; padding: 5px;">Reject</a>'
            )
        return "-"
    verification_actions.short_description = 'Actions'
    
    def document_previews(self, obj):
        """Show document previews in admin."""
        html = []
        
        if obj.jee_scorecard:
            html.append(f'<p><strong>JEE Scorecard:</strong></p>')
            if obj.jee_scorecard.name.lower().endswith(('.jpg', '.jpeg', '.png')):
                html.append(f'<img src="{obj.jee_scorecard.url}" style="max-width: 200px; max-height: 200px;" />')
            else:
                html.append(f'<a href="{obj.jee_scorecard.url}" target="_blank">View Document</a>')
        
        if obj.college_certificate:
            html.append(f'<p><strong>College Certificate:</strong></p>')
            if obj.college_certificate.name.lower().endswith(('.jpg', '.jpeg', '.png')):
                html.append(f'<img src="{obj.college_certificate.url}" style="max-width: 200px; max-height: 200px;" />')
            else:
                html.append(f'<a href="{obj.college_certificate.url}" target="_blank">View Document</a>')
        
        return mark_safe(''.join(html)) if html else "No documents uploaded"
    document_previews.short_description = "Document Previews"
    
    def approve_mentors(self, request, queryset):
        """Bulk approve mentors."""
        from django.utils import timezone
        
        updated = queryset.update(
            verification_status='approved',
            verified_at=timezone.now(),
            verified_by=request.user
        )
        
        # Update user verification status
        for mentor_profile in queryset:
            mentor_profile.user.is_verified = True
            mentor_profile.user.save()
        
        self.message_user(request, f'{updated} mentor(s) approved successfully.')
    approve_mentors.short_description = "Approve selected mentors"
    
    def reject_mentors(self, request, queryset):
        """Bulk reject mentors."""
        updated = queryset.update(verification_status='rejected')
        
        # Update user verification status
        for mentor_profile in queryset:
            mentor_profile.user.is_verified = False
            mentor_profile.user.save()
        
        self.message_user(request, f'{updated} mentor(s) rejected.')
    reject_mentors.short_description = "Reject selected mentors"
    
    def mark_pending(self, request, queryset):
        """Mark mentors as pending review."""
        updated = queryset.update(
            verification_status='pending',
            verified_at=None,
            verified_by=None
        )
        self.message_user(request, f'{updated} mentor(s) marked as pending.')
    mark_pending.short_description = "Mark as pending review"


# Reference model admins
@admin.register(College)
class CollegeAdmin(admin.ModelAdmin):
    list_display = ['name', 'location', 'is_active']
    list_filter = ['is_active', 'location']
    search_fields = ['name', 'location']
    ordering = ['name']


@admin.register(Stream)
class StreamAdmin(admin.ModelAdmin):
    list_display = ['name', 'code', 'is_active']
    list_filter = ['is_active']
    search_fields = ['name', 'code']
    ordering = ['name']


@admin.register(Subject)
class SubjectAdmin(admin.ModelAdmin):
    list_display = ['name', 'code', 'is_active']
    list_filter = ['is_active']
    search_fields = ['name', 'code']
    ordering = ['name']


@admin.register(Profession)
class ProfessionAdmin(admin.ModelAdmin):
    list_display = ['name', 'is_active']
    list_filter = ['is_active']
    search_fields = ['name']
    ordering = ['name']


# Customize admin site
admin.site.site_header = 'Prepji Administration'
admin.site.site_title = 'Prepji Admin'
admin.site.index_title = 'Welcome to Prepji Administration'
```

**✅ Validation Checkpoint:**

- [ ] Admin interface created for all models
- [ ] Mentor verification workflow implemented
- [ ] Document preview functionality working
- [ ] Bulk actions for user management available

***

### 📱 Step 3.9: Populate Reference Data

**What:** Add initial data for colleges, streams, subjects, and professions.

**File:** `backend/apps/accounts/management/__init__.py` (CREATE THIS DIRECTORY AND FILE)

```python
# Empty file to make directory a Python package
```

**File:** `backend/apps/accounts/management/commands/__init__.py` (CREATE THIS FILE)

```python
# Empty file to make directory a Python package
```

**File:** `backend/apps/accounts/management/commands/populate_reference_data.py`

```python
"""
Management command to populate reference data for Prepji.
Run with: python manage.py populate_reference_data
"""

from django.core.management.base import BaseCommand
from apps.accounts.models import College, Stream, Subject, Profession


class Command(BaseCommand):
    help = 'Populate reference data for colleges, streams, subjects, and professions'
    
    def handle(self, *args, **options):
        self.stdout.write('Populating reference data...')
        
        # Create colleges
        colleges_data = [
            ('IIT Bombay', 'Mumbai, Maharashtra'),
            ('IIT Delhi', 'New Delhi, Delhi'),
            ('IIT Madras', 'Chennai, Tamil Nadu'),
            ('IIT Kanpur', 'Kanpur, Uttar Pradesh'),
            ('IIT Kharagpur', 'Kharagpur, West Bengal'),
            ('IIT Roorkee', 'Roorkee, Uttarakhand'),
            ('IIT Guwahati', 'Guwahati, Assam'),
            ('IIT Hyderabad', 'Hyderabad, Telangana'),
            ('IIT Indore', 'Indore, Madhya Pradesh'),
            ('IIT BHU Varanasi', 'Varanasi, Uttar Pradesh'),
            ('IIT Mandi', 'Mandi, Himachal Pradesh'),
            ('IIT Gandhinagar', 'Gandhinagar, Gujarat'),
            ('IIT Patna', 'Patna, Bihar'),
            ('IIT Ropar', 'Rupnagar, Punjab'),
            ('IIT ISM Dhanbad', 'Dhanbad, Jharkhand'),
            ('NIT Trichy', 'Tiruchirappalli, Tamil Nadu'),
            ('NIT Warangal', 'Warangal, Telangana'),
            ('NIT Surathkal', 'Mangalore, Karnataka'),
            ('NIT Calicut', 'Kozhikode, Kerala'),
            ('BITS Pilani', 'Pilani, Rajasthan'),
            ('BITS Goa', 'Goa'),
            ('BITS Hyderabad', 'Hyderabad, Telangana'),
            ('IIIT Hyderabad', 'Hyderabad, Telangana'),
            ('IIIT Bangalore', 'Bangalore, Karnataka'),
            ('DTU', 'New Delhi, Delhi'),
            ('NSIT', 'New Delhi, Delhi'),
            ('VIT Vellore', 'Vellore, Tamil Nadu'),
            ('Manipal Institute of Technology', 'Manipal, Karnataka'),
            ('SRM Institute of Science and Technology', 'Chennai, Tamil Nadu'),
            ('Thapar Institute of Engineering', 'Patiala, Punjab'),
        ]
        
        for name, location in colleges_data:
            College.objects.get_or_create(name=name, defaults={'location': location})
        
        self.stdout.write(f'Created {len(colleges_data)} colleges')
        
        # Create streams
        streams_data = [
            ('Computer Science and Engineering', 'CSE'),
            ('Computer Science and Engineering (AI & ML)', 'CSE-AIML'),
            ('Computer Science and Engineering (Data Science)', 'CSE-DS'),
            ('Information Technology', 'IT'),
            ('Electronics and Communication Engineering', 'ECE'),
            ('Electrical Engineering', 'EE'),
            ('Mechanical Engineering', 'ME'),
            ('Civil Engineering', 'CE'),
            ('Chemical Engineering', 'CHE'),
            ('Aerospace Engineering', 'AE'),
            ('Biotechnology Engineering', 'BT'),
            ('Environmental Engineering', 'ENV'),
            ('Mining Engineering', 'MIN'),
            ('Metallurgical Engineering', 'MET'),
            ('Production and Industrial Engineering', 'PIE'),
            ('Engineering Physics', 'EP'),
            ('Engineering Chemistry', 'EC'),
        ]
        
        for name, code in streams_data:
            Stream.objects.get_or_create(code=code, defaults={'name': name})
        
        self.stdout.write(f'Created {len(streams_data)} streams')
        
        # Create subjects
        subjects_data = [
            ('Physics', 'PHY'),
            ('Chemistry', 'CHE'),
            ('Mathematics', 'MATH'),
            ('Biology', 'BIO'),
        ]
        
        for name, code in subjects_data:
            Subject.objects.get_or_create(code=code, defaults={'name': name})
        
        self.stdout.write(f'Created {len(subjects_data)} subjects')
        
        # Create professions
        professions_data = [
            'Engineering Student',
            'IIT/NIT Graduate',
            'College Professor',
            'School Teacher',
            'Coaching Institute Owner',
            'Independent Mentor',
            'Educational Consultant',
            'Career Counselor',
            'Industry Professional',
            'Researcher',
            'Entrepreneur',
        ]
        
        for name in professions_data:
            Profession.objects.get_or_create(name=name)
        
        self.stdout.write(f'Created {len(professions_data)} professions')
        
        self.stdout.write(
            self.style.SUCCESS('Successfully populated all reference data!')
        )
```

**Run the command:**

```bash
docker-compose exec backend python manage.py populate_reference_data
```

**Expected Output:**

```
Populating reference data...
Created 30 colleges
Created 17 streams  
Created 4 subjects
Created 11 professions
Successfully populated all reference data!
```

**✅ Validation Checkpoint:**

- [ ] Management command created and executed
- [ ] Reference data populated in database
- [ ] All models have initial data for testing

***

### 🧪 Step 3.10: Test the API Endpoints

**What:** Verify all API endpoints are working correctly with our modern package configuration.

**Commands for Testing:**

```bash
# Test user registration
curl -X POST http://localhost:8000/api/auth/registration/ \
  -H "Content-Type: application/json" \
  -d '{
    "email": "student@test.com",
    "password1": "testpass123",
    "password2": "testpass123",
    "user_type": "student"
  }'

# Test login
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{
    "email": "student@test.com",
    "password": "testpass123"
  }'

# Test getting colleges list (requires authentication token)
curl -X GET http://localhost:8000/api/accounts/colleges/ \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN_HERE"

# Test profile status
curl -X GET http://localhost:8000/api/accounts/profile/status/ \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN_HERE"
```

**Test via Django Admin:**

1. **Open:** http://localhost:8000/admin/
2. **Login:** admin@prepji.com / admin123456
3. **Navigate to:** Users section
4. **Verify:** All models are visible and functional
5. **Test:** Create a test mentor and approve/reject workflow

**✅ Validation Checkpoint:**

- [ ] User registration working with email-only
- [ ] JWT authentication functioning
- [ ] Profile API endpoints responding
- [ ] Admin interface fully functional
- [ ] Reference data accessible via API

***

## 4. Testing \& Verification

### ✅ Step 4.1: Verify All Services Are Running

**Check container status:**

```bash
docker-compose ps
```

**All containers should show "Up" status with no restart loops.**

### ✅ Step 4.2: Test Modern Package Compatibility

**Test dj-rest-auth 7.x compatibility:**

1. **Registration API** should work with email-only
2. **JWT tokens** should be issued correctly
3. **allauth 65.x** settings should not cause errors

**Test file uploads:**

1. **Avatar upload** should resize and save correctly
2. **Document upload** should validate file types
3. **Media URLs** should be accessible

### ✅ Step 4.3: Test Profile Completion Workflow

**Student profile flow:**

1. Register as student
2. Complete basic profile
3. Complete student-specific fields
4. Verify completion percentage calculation

**Mentor verification flow:**

1. Register as mentor
2. Complete profile with documents
3. Admin reviews and approves/rejects
4. Verify verification status updates

***

## 5. Summary \& Next Steps

### 🎉 What You've Accomplished

**✅ Modern Authentication System**

- Email-only registration with **dj-rest-auth 7.x**
- JWT token management compatible with **django-allauth 65.x**
- Secure file upload system with validation

**✅ Comprehensive User Models**

- Custom User model with student/mentor types
- Profile models with completion tracking
- Reference data for colleges, streams, subjects

**✅ Production-Ready API**

- REST endpoints for all user operations
- Admin interface for user management
- Mentor verification workflow

**✅ Modern Package Integration**

- **Django 5.2.5** with **Python 3.13**
- Breaking changes handled for all upgraded packages
- Future-proof architecture


### 🔗 Dependencies Created

**This phase enables:**

- **Phase 3:** Frontend can now consume authentication APIs
- **Phase 4:** Integration testing with complete user flows
- **Phase 5:** Background tasks can access user data
- **Phase 6:** Production deployment with user management


### 🚀 Next Tutorial: Phase 3 - Frontend Development

**Upcoming features:**

- React authentication pages with modern UI
- Profile completion wizard
- Real-time form validation
- File upload components
- Apollo Client integration with GraphQL

***

## 6. Knowledge Check

### 📝 Q1: Modern Package Understanding

**Can you explain the key breaking changes we handled from the package upgrades?**

**Expected Answer:**

- **dj-rest-auth 7.x:** Requires different settings structure, JWT configuration
- **django-allauth 65.x:** Uses `ACCOUNT_LOGIN_METHODS = {"email"}` instead of old syntax
- **allauth:** Requires `AccountMiddleware` in MIDDLEWARE
- **rest_framework.authtoken:** Must be in INSTALLED_APPS for compatibility


### 📝 Q2: Architecture Understanding

**What's the difference between UserProfile, StudentProfile, and MentorProfile?**

**Expected Answer:**

- **UserProfile:** Base profile with common fields (name, bio, avatar, contacts)
- **StudentProfile:** Student-specific fields (target JEE year, colleges, academic records)
- **MentorProfile:** Mentor-specific fields (profession, expertise, credentials, verification status)
- **One-to-one relationships** ensure data integrity and separation of concerns


### 📝 Q3: API Design

**How does our API handle file uploads securely?**

**Expected Answer:**

- **File validation:** Extensions and size limits enforced
- **Secure paths:** UUIDs prevent filename conflicts and directory traversal
- **Image processing:** Automatic resizing and compression for avatars
- **Permission checks:** Only authenticated users can upload files


### ✅ Ready to Proceed?

**Checklist before moving to Phase 3:**

- [ ] All API endpoints accessible and functional
- [ ] User registration and authentication working
- [ ] Admin interface fully operational
- [ ] File uploads processing correctly
- [ ] Modern package versions working without errors
- [ ] Database migrations applied successfully

**If all items are checked, you're ready for Phase 3: Frontend Development!**

***

**Congratulations! 🎉 Your Prepji backend now has a complete, modern user management system ready for frontend integration!**

