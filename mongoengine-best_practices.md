# MongoEngine Best Practices Guide

## Contents
- [Project Structure](#project-structure)
  - [Recommended Layout](#recommended-layout)
  - [Connection Management](#connection-management)
- [Document Models & Fields](#document-models--fields)
  - [Document Definition](#document-definition)
  - [Fields & Validation](#fields--validation)
  - [Meta Options](#meta-options)
- [Indexing Strategies](#indexing-strategies)
  - [Index Types](#index-types)
  - [Index Configuration](#index-configuration)
  - [Performance Considerations](#performance-considerations)
- [Validation & Clean-up](#validation--clean-up)
  - [Custom Validation](#custom-validation)
  - [Document Cleanup](#document-cleanup)
- [Querying Best Practices](#querying-best-practices)
  - [Query Optimization](#query-optimization)
  - [Batch Processing](#batch-processing)
- [Logging & Monitoring](#logging--monitoring)
  - [Basic Setup](#basic-setup)
  - [Performance Monitoring](#performance-monitoring)
  - [Query Profiling](#query-profiling)
- [Signals & Events](#signals--events)
  - [Signal Types](#signal-types)
  - [Event Handling](#event-handling)
- [Testing & Migration](#testing--migration)
  - [Test Setup](#test-setup)
  - [Migration Strategies](#migration-strategies)
- [Performance Optimization](#performance-optimization)
  - [Caching Strategies](#caching-strategies)
  - [Query Optimization](#query-optimization)
- [Error Handling](#error-handling)
  - [Exception Types](#exception-types)
  - [Error Management](#error-management)

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
from dotenv import load_dotenv
import os

# Load environment variables from .env file
load_dotenv()

class DatabaseConfig:
    def __init__(self):
        self.MONGODB_HOST = os.getenv('MONGODB_HOST', 'mongodb://localhost:27017')
        self.MONGODB_DB = os.getenv('MONGODB_DB', 'mydb')
        self.MONGODB_USERNAME = os.getenv('MONGODB_USERNAME')
        self.MONGODB_PASSWORD = os.getenv('MONGODB_PASSWORD')
        self.MONGODB_AUTH_SOURCE = os.getenv('MONGODB_AUTH_SOURCE', 'admin')
        self.MONGODB_AUTH_MECHANISM = os.getenv('MONGODB_AUTH_MECHANISM', 'SCRAM-SHA-1')

    def get_connection_settings(self):
        settings = {
            'host': self.MONGODB_HOST,
            'db': self.MONGODB_DB,
            'username': self.MONGODB_USERNAME,
            'password': self.MONGODB_PASSWORD
        }
        
        # Add optional authentication settings if credentials are provided
        if self.MONGODB_USERNAME and self.MONGODB_PASSWORD:
            settings.update({
                'authentication_source': self.MONGODB_AUTH_SOURCE,
                'authentication_mechanism': self.MONGODB_AUTH_MECHANISM
            })
        
        return settings

@contextmanager
def mongodb_connection(config: DatabaseConfig):
    """Context manager for MongoDB connection handling with logging"""
    try:
        conn = connect(**config.get_connection_settings())
        logging.info(f"Connected to MongoDB at {config.MONGODB_HOST}")
        yield conn
    except Exception as e:
        logging.error(f"Failed to connect to MongoDB: {str(e)}")
        raise
    finally:
        conn.close()
        logging.info("Closed MongoDB connection")

# Example .env file:
"""
MONGODB_HOST=mongodb://localhost:27017
MONGODB_DB=myapp
MONGODB_USERNAME=admin
MONGODB_PASSWORD=secret
MONGODB_AUTH_SOURCE=admin
MONGODB_AUTH_MECHANISM=SCRAM-SHA-1
"""
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
        # Collection name in MongoDB (if not provided, uses class name in snake_case)
        'collection': 'blog_posts',
        
        # Index Definitions:
        'indexes': [
            # 1. Single-field index
            'title',  # Equivalent to {'fields': ['title']}
            # Creates a basic index on 'title' field
            # Good for: Queries that search by title
            # Example: Post.objects(title="Example")
            
            # 2. Compound index
            ('title', 'status'),  # Compound index on multiple fields
            # Good for: Queries that filter by both title and status
            # Example: Post.objects(title="Example", status="published")
            
            # 3. Text index with weights
            {
                'fields': ['$title', '$content'],  # $ prefix indicates text index
                'default_language': 'english',
                'weights': {'title': 10, 'content': 2}  # Title matches are 5x more important
            }
            # Good for: Full-text search across title and content
            # Example: Post.objects.search_text("mongodb")
            
            # 4. Other index types example:
            {
                'fields': ['tags'],
                'sparse': True,  # Only index documents that have 'tags'
                'unique': True   # Ensure uniqueness
            },
            
            # 5. TTL Index example:
            {
                'fields': ['created_at'],
                'expireAfterSeconds': 3600  # Documents auto-delete after 1 hour
            },
            
            # 6. Hashed Index example:
            {
                'fields': ['#user_id']  # Hash-based sharding
            }
        ],
        
        # Default ordering for queries
        'ordering': ['-created'],
        
        # Schema enforcement options
        'strict': False,  # Allow dynamic fields
        
        # Other useful meta options
        'auto_create_index': True,    # Automatically create indexes
        'index_background': True,     # Create indexes in background
        'collection_options': {       # Collection specific options
            'capped': True,
            'max': 1000,
            'size': 1000000,
        }
    }
```

### Understanding strict and dynamic Options: A Real-world Analogy

Think of a MongoDB document like a form, and the Document class as a form template:

1. **Strict Mode (strict=True)**
   - Like a rigid government form:
   - Has predefined fields only
   - Won't accept any extra fields
   - Throws error if you try to add undefined fields
   ```python
   class StrictForm(Document):
       name = StringField()
       meta = {'strict': True}
   
   # This will raise an error
   form = StrictForm(name="John", age=25)  # Error: age not defined
   ```

2. **Non-Strict Mode (strict=False)**
   - Like a flexible feedback form:
   - Has some predefined fields
   - Accepts additional fields but doesn't track them
   - Extra fields are stored but not accessible via model
   ```python
   class FlexibleForm(Document):
       name = StringField()
       meta = {'strict': False}
   
   # This works - extra field is stored but not validated
   form = FlexibleForm(name="John", age=25)
   ```

3. **Dynamic Mode (strict=False, dynamic=True)**
   - Like a completely customizable survey:
   - Has predefined fields
   - Accepts AND tracks additional fields
   - Extra fields become accessible
   ```python
   class DynamicForm(Document):
       name = StringField()
       meta = {'strict': False, 'dynamic': True}
   
   # This works and 'age' becomes accessible
   form = DynamicForm(name="John", age=25)
   print(form.age)  # Outputs: 25
   ```

Real-world Use Cases:
- **Strict Mode**: Financial records, medical records where field consistency is crucial
- **Non-Strict**: Logging systems where extra data might be stored but not processed
- **Dynamic Mode**: User preferences, where users can add custom fields
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

## Logging and Monitoring

### Basic Logging Setup
```python
import logging
from pymongo import monitoring

def setup_mongodb_logging():
    # Set up basic logging
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s [%(levelname)s] %(message)s',
        handlers=[
            logging.FileHandler('mongodb.log'),
            logging.StreamHandler()
        ]
    )

class CommandLogger(monitoring.CommandListener):
    def started(self, event):
        logging.debug("Command {0.command_name} with request id "
                     "{0.request_id} started on server "
                     "{0.connection_id}".format(event))

    def succeeded(self, event):
        logging.debug("Command {0.command_name} with request id "
                     "{0.request_id} on server {0.connection_id} "
                     "succeeded in {0.duration_micros} "
                     "microseconds".format(event))

    def failed(self, event):
        logging.error("Command {0.command_name} with request id "
                     "{0.request_id} on server {0.connection_id} "
                     "failed in {0.duration_micros} "
                     "microseconds".format(event))

# Register command logger
monitoring.register(CommandLogger())

### Performance Monitoring
class PerformanceMonitor(monitoring.CommandListener):
    def __init__(self, slow_query_threshold_ms=100):
        self.slow_query_threshold = slow_query_threshold_ms * 1000  # Convert to microseconds
        
    def succeeded(self, event):
        if event.duration_micros > self.slow_query_threshold:
            logging.warning(
                f"Slow query detected! Command {event.command_name} "
                f"took {event.duration_micros/1000:.2f}ms to execute.\n"
                f"Command: {event.command}"
            )

# Usage example
monitoring.register(PerformanceMonitor(slow_query_threshold_ms=100))
```

### Query Monitoring Example
```python
class QueryMonitor:
    @staticmethod
    def log_query_stats():
        from mongoengine.connection import get_db
        
        db = get_db()
        stats = db.command('serverStatus')
        
        metrics = {
            'operations': stats['opcounters'],
            'connections': stats['connections'],
            'network': stats['network']
        }
        
        logging.info(f"MongoDB Stats: {metrics}")

### Document Change Tracking
class AuditMiddleware:
    @staticmethod
    def log_changes(document, changes):
        logging.info(
            f"Document {document.__class__.__name__} "
            f"(id: {document.id}) changed: {changes}"
        )

    @classmethod
    def post_save(cls, sender, document, **kwargs):
        if kwargs.get('created'):
            cls.log_changes(document, "Created")
        else:
            cls.log_changes(document, "Updated")

# Connect signal
signals.post_save.connect(AuditMiddleware.post_save)
```

### Comprehensive Monitoring Setup
```python
def setup_comprehensive_monitoring():
    # 1. Command Monitoring
    monitoring.register(CommandLogger())
    monitoring.register(PerformanceMonitor())
    
    # 2. Connection Monitoring
    class ConnectionLogger(monitoring.ConnectionPoolListener):
        def pool_created(self, event):
            logging.info(f"Connection Pool created: {event.address}")
            
        def pool_closed(self, event):
            logging.info(f"Connection Pool closed: {event.address}")
    
    monitoring.register(ConnectionLogger())
    
    # 3. Server Monitoring
    class ServerLogger(monitoring.ServerListener):
        def server_opening(self, event):
            logging.info(f"Server {event.server_address} opening...")
            
        def server_closed(self, event):
            logging.info(f"Server {event.server_address} closed")
    
    monitoring.register(ServerLogger())

# Usage Example:
if __name__ == "__main__":
    setup_mongodb_logging()
    setup_comprehensive_monitoring()
    
    # Your application code here
    Post(title="Test").save()
```

### Query Profiling
```python
class QueryProfiler:
    def __init__(self):
        self.query_count = 0
        self.total_time = 0
    
    @contextmanager
    def profile(self, operation_name):
        start_time = time.time()
        self.query_count += 1
        
        try:
            yield
        finally:
            duration = time.time() - start_time
            self.total_time += duration
            
            logging.info(
                f"Operation: {operation_name}\n"
                f"Duration: {duration:.2f}s\n"
                f"Total Queries: {self.query_count}\n"
                f"Average Time: {self.total_time/self.query_count:.2f}s"
            )

# Usage:
profiler = QueryProfiler()
with profiler.profile("fetch_users"):
    users = User.objects.all()
```

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
