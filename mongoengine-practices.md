# MongoEngine Best Practices Guide

## Contents
- [Project Structure](#project-structure)
- [Document Models & Fields](#document-models--fields)
- [Validation & Clean-up](#validation--clean-up)
- [Querying Best Practices](#querying-best-practices)
- [Signals & Events](#signals--events)
- [Testing & Migration](#testing--migration)
- [Performance Optimization](#performance-optimization)
- [Error Handling](#error-handling)

## Project Structure

### Recommended Layout
```
mongodb-project/
├── src/
│   ├── common/
│   │   ├── base.py        # Base document models
│   │   └── utils.py       # Shared utilities
│   ├── auth/
│   │   ├── models.py      # Document models
│   │   ├── fields.py      # Custom fields
│   │   └── services.py    # Business logic
│   ├── posts/
│   │   ├── models.py
│   │   └── services.py
│   ├── config.py          # Connection settings
│   └── database.py        # DB connection management
├── tests/
└── requirements.txt
```

### Connection Management
```python
from mongoengine import connect
from contextlib import contextmanager

class DatabaseConfig:
    def __init__(self):
        self.MONGODB_HOST = "mongodb://localhost:27017"
        self.MONGODB_DB = "mydb"
        self.MONGODB_USERNAME = None
        self.MONGODB_PASSWORD = None

    def get_connection_settings(self):
        return {
            'host': self.MONGODB_HOST,
            'db': self.MONGODB_DB,
            'username': self.MONGODB_USERNAME,
            'password': self.MONGODB_PASSWORD
        }

@contextmanager
def mongodb_connection(config: DatabaseConfig):
    conn = connect(**config.get_connection_settings())
    try:
        yield conn
    finally:
        conn.close()
```

## Document Models & Fields

### Using Meta Options Effectively
```python
class Post(Document):
    title = StringField(required=True)
    content = StringField(required=True)
    tags = ListField(StringField(max_length=50))
    status = StringField(choices=['draft', 'published'])

    meta = {
        'collection': 'blog_posts',
        'indexes': [
            'title',
            ('title', 'status'),
            {
                'fields': ['$title', '$content'],
                'default_language': 'english',
                'weights': {'title': 10, 'content': 2}
            }
        ],
        'ordering': ['-created'],
        'strict': False,  # Allow dynamic fields
    }
```

### Understanding strict and dynamic Options
1. Development Mode (Flexible)
```python
class DevelopmentModel(Document):
    name = StringField()
    meta = {
        'strict': False,  # Allows fields not in schema
        'dynamic': True   # Tracks dynamic fields
    }
```

2. Production Mode (Strict)
```python
class ProductionModel(Document):
    name = StringField()
    meta = {
        'strict': True,   # Enforces schema
        'dynamic': False  # Rejects unknown fields
    }
```

### Field Validation Patterns
```python
from mongoengine import *
from datetime import datetime

class User(Document):
    email = EmailField(required=True, unique=True)
    username = StringField(
        required=True, 
        unique=True,
        min_length=3,
        max_length=50,
        regex=r'^[a-zA-Z0-9_]+$'
    )
    age = IntField(min_value=0, max_value=120)
    created = DateTimeField(default=datetime.utcnow)

    def clean(self):
        """Custom document-level validation"""
        if self.age < 18 and not hasattr(self, 'guardian'):
            raise ValidationError('Minors must have a guardian')
```

## Validation & Clean-up

### Custom Validation Methods
```python
def validate_phone(phone):
    if not phone.startswith('+'):
        raise ValidationError('Phone must start with +')

class Contact(EmbeddedDocument):
    phone = StringField(validation=validate_phone)
    email = EmailField()

    def clean(self):
        """Document-level validation"""
        if not self.phone and not self.email:
            raise ValidationError('At least one contact method required')
```

### Using Signals for Auto-updates
```python
from mongoengine import signals

class Post(Document):
    title = StringField()
    updated_at = DateTimeField()

    def __init__(self, *args, **kwargs):
        super(Post, self).__init__(*args, **kwargs)
        self._register_signals()

    def _register_signals(self):
        signals.pre_save.connect(self.pre_save_handler, sender=self.__class__)

    def pre_save_handler(self, sender, document, **kwargs):
        document.updated_at = datetime.utcnow()
```

## Querying Best Practices

### Efficient Querying
```python
class PostService:
    @staticmethod
    def get_user_posts(user_id, page=1, per_page=20):
        return Post.objects(author=user_id)\
                  .only('title', 'snippet')\  # Select specific fields
                  .skip((page-1) * per_page)\
                  .limit(per_page)

    @staticmethod
    def get_with_references(post_id):
        return Post.objects(id=post_id)\
                  .select_related()\  # Load references efficiently
                  .first()
```

### Query Optimization
```python
class QueryOptimization:
    @staticmethod
    def efficient_batch_processing(queryset, batch_size=100):
        """Process large querysets efficiently"""
        for doc in queryset.batch_size(batch_size).no_cache():
            yield doc

    @staticmethod
    def count_with_timeout(queryset, timeout_ms=1000):
        """Count with timeout to avoid long operations"""
        return queryset.timeout(timeout_ms).count()
```

## Testing & Migration

### Test Setup with mongomock
```python
import mongomock
import unittest
from mongoengine import connect, disconnect

class TestUser(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        connect('testdb', 
                host='mongodb://localhost',
                mongo_client_class=mongomock.MongoClient)

    @classmethod
    def tearDownClass(cls):
        disconnect()

    def test_user_creation(self):
        user = User(email="test@example.com").save()
        self.assertEqual(User.objects.count(), 1)
```

### Migration Utilities
```python
class MigrationUtils:
    @staticmethod
    def migrate_field(document_class, old_field, new_field):
        """Migrate data from old field to new field"""
        collection = document_class._get_collection()
        collection.update_many(
            {old_field: {"$exists": True}},
            {"$rename": {old_field: new_field}}
        )

    @staticmethod
    def add_field_with_default(document_class, field_name, default_value):
        """Add new field with default value"""
        collection = document_class._get_collection()
        collection.update_many(
            {field_name: {"$exists": False}},
            {"$set": {field_name: default_value}}
        )
```

## Performance Optimization

### Indexing Strategies
```python
class OptimizedDocument(Document):
    title = StringField()
    category = StringField()
    tags = ListField(StringField())
    status = StringField()

    meta = {
        'indexes': [
            'title',  # Simple index
            ('category', 'status'),  # Compound index
            {
                'fields': ['$title'],  # Text index
                'default_language': 'english'
            },
            {
                'fields': ['tags'],
                'sparse': True  # Sparse index
            }
        ],
        'auto_create_index': True
    }
```

### Caching Patterns
```python
from functools import lru_cache

class CachedQueries:
    @staticmethod
    @lru_cache(maxsize=100)
    def get_active_categories():
        return list(Category.objects(active=True).scalar('name'))

    @staticmethod
    def clear_cache():
        CachedQueries.get_active_categories.cache_clear()
```

Remember that these practices should be adapted based on your specific use case and requirements. The key is to maintain consistency across your project and follow patterns that promote maintainability and performance.
