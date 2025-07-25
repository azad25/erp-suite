# Multi-Tenant Structure and Data Isolation

## Overview

This document describes the comprehensive multi-tenant architecture for the enterprise ERP system, covering data isolation strategies, security measures, and scalability considerations.

## Multi-Tenancy Approaches

### 1. Schema-Based Isolation (Recommended)

The system uses PostgreSQL schema-based isolation for optimal balance between security, performance, and resource utilization.

```sql
-- Database structure with schema isolation
CREATE DATABASE erp_system;

-- Shared schema for system-wide data
CREATE SCHEMA shared;

-- Tenant-specific schemas
CREATE SCHEMA tenant_company_a;
CREATE SCHEMA tenant_company_b;
CREATE SCHEMA tenant_company_c;

-- Shared tables (system-wide)
CREATE TABLE shared.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(100) UNIQUE NOT NULL,
    subdomain VARCHAR(50) UNIQUE,
    status VARCHAR(20) DEFAULT 'active',
    plan VARCHAR(50) DEFAULT 'standard',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    settings JSONB DEFAULT '{}'
);

CREATE TABLE shared.tenant_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES shared.tenants(id),
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, email)
);

CREATE TABLE shared.system_config (
    key VARCHAR(100) PRIMARY KEY,
    value JSONB NOT NULL,
    description TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Plugin configurations (shared across tenants)
CREATE TABLE shared.plugins (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    version VARCHAR(20) NOT NULL,
    config JSONB DEFAULT '{}',
    is_enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE shared.tenant_plugins (
    tenant_id UUID REFERENCES shared.tenants(id),
    plugin_id UUID REFERENCES shared.plugins(id),
    config JSONB DEFAULT '{}',
    is_enabled BOOLEAN DEFAULT true,
    PRIMARY KEY (tenant_id, plugin_id)
);
```

### 2. Tenant-Specific Schema Structure

Each tenant gets their own schema with identical table structures but isolated data.

```sql
-- Template for tenant-specific tables
-- This structure is replicated for each tenant schema

-- User management within tenant
CREATE TABLE tenant_{{tenant_id}}.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_user_id UUID REFERENCES shared.tenant_users(id),
    employee_id VARCHAR(50),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    department_id UUID,
    position VARCHAR(100),
    hire_date DATE,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Role-based access control
CREATE TABLE tenant_{{tenant_id}}.roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    permissions JSONB DEFAULT '[]',
    is_system_role BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tenant_{{tenant_id}}.user_roles (
    user_id UUID REFERENCES tenant_{{tenant_id}}.users(id),
    role_id UUID REFERENCES tenant_{{tenant_id}}.roles(id),
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by UUID REFERENCES tenant_{{tenant_id}}.users(id),
    PRIMARY KEY (user_id, role_id)
);

-- Business entities (HRM)
CREATE TABLE tenant_{{tenant_id}}.departments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    manager_id UUID REFERENCES tenant_{{tenant_id}}.users(id),
    parent_id UUID REFERENCES tenant_{{tenant_id}}.departments(id),
    budget DECIMAL(15,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tenant_{{tenant_id}}.employees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES tenant_{{tenant_id}}.users(id),
    employee_id VARCHAR(50) UNIQUE NOT NULL,
    department_id UUID REFERENCES tenant_{{tenant_id}}.departments(id),
    manager_id UUID REFERENCES tenant_{{tenant_id}}.employees(id),
    salary DECIMAL(12,2),
    salary_currency VARCHAR(3) DEFAULT 'USD',
    employment_type VARCHAR(20) DEFAULT 'full_time',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Business entities (CRM)
CREATE TABLE tenant_{{tenant_id}}.contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(20) DEFAULT 'person', -- person, company
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(20),
    address JSONB,
    assigned_to UUID REFERENCES tenant_{{tenant_id}}.users(id),
    tags TEXT[],
    custom_fields JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tenant_{{tenant_id}}.leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id UUID REFERENCES tenant_{{tenant_id}}.contacts(id),
    title VARCHAR(200) NOT NULL,
    description TEXT,
    source VARCHAR(50),
    status VARCHAR(50) DEFAULT 'new',
    score INTEGER DEFAULT 0,
    estimated_value DECIMAL(15,2),
    expected_close_date DATE,
    assigned_to UUID REFERENCES tenant_{{tenant_id}}.users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Business entities (Finance)
CREATE TABLE tenant_{{tenant_id}}.accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    type VARCHAR(50) NOT NULL, -- asset, liability, equity, revenue, expense
    parent_id UUID REFERENCES tenant_{{tenant_id}}.accounts(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tenant_{{tenant_id}}.invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id UUID REFERENCES tenant_{{tenant_id}}.contacts(id),
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    subtotal DECIMAL(15,2) NOT NULL,
    tax_amount DECIMAL(15,2) DEFAULT 0,
    total_amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) DEFAULT 'draft',
    created_by UUID REFERENCES tenant_{{tenant_id}}.users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Audit and activity logs
CREATE TABLE tenant_{{tenant_id}}.audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES tenant_{{tenant_id}}.users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Tenant Resolution and Context

### 1. Tenant Resolution Middleware

```python
class TenantResolutionMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
        self.tenant_service = TenantService()
    
    def __call__(self, request):
        # Resolve tenant from various sources
        tenant = self.resolve_tenant(request)
        
        if not tenant:
            return JsonResponse(
                {'error': 'Tenant not found'}, 
                status=404
            )
        
        # Set tenant context for the request
        request.tenant = tenant
        
        # Set database schema for this request
        self.set_database_schema(tenant.schema_name)
        
        response = self.get_response(request)
        
        # Clean up database connection
        self.reset_database_schema()
        
        return response
    
    def resolve_tenant(self, request):
        """Resolve tenant from subdomain, domain, or header"""
        
        # 1. Try subdomain resolution
        host = request.get_host()
        if '.' in host:
            subdomain = host.split('.')[0]
            tenant = self.tenant_service.get_by_subdomain(subdomain)
            if tenant:
                return tenant
        
        # 2. Try custom domain resolution
        tenant = self.tenant_service.get_by_domain(host)
        if tenant:
            return tenant
        
        # 3. Try header-based resolution (for API clients)
        tenant_id = request.headers.get('X-Tenant-ID')
        if tenant_id:
            return self.tenant_service.get_by_id(tenant_id)
        
        # 4. Try JWT token tenant claim
        auth_header = request.headers.get('Authorization')
        if auth_header and auth_header.startswith('Bearer '):
            token = auth_header[7:]
            try:
                payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])
                tenant_id = payload.get('tenant_id')
                if tenant_id:
                    return self.tenant_service.get_by_id(tenant_id)
            except jwt.InvalidTokenError:
                pass
        
        return None
    
    def set_database_schema(self, schema_name):
        """Set PostgreSQL search path to tenant schema"""
        from django.db import connection
        
        with connection.cursor() as cursor:
            cursor.execute(f"SET search_path TO {schema_name}, shared, public")
    
    def reset_database_schema(self):
        """Reset database schema to default"""
        from django.db import connection
        
        with connection.cursor() as cursor:
            cursor.execute("SET search_path TO public")
```

### 2. Tenant-Aware Database Manager

```python
class TenantAwareManager(models.Manager):
    """Custom manager that automatically filters by tenant"""
    
    def get_queryset(self):
        # Get current tenant from thread-local storage
        tenant = get_current_tenant()
        if tenant:
            # Schema-based isolation handles this automatically
            return super().get_queryset()
        else:
            # Fallback to empty queryset if no tenant context
            return super().get_queryset().none()

class TenantAwareModel(models.Model):
    """Base model for tenant-aware entities"""
    
    objects = TenantAwareManager()
    
    class Meta:
        abstract = True
    
    def save(self, *args, **kwargs):
        # Ensure we're in the correct tenant context
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context available")
        
        super().save(*args, **kwargs)
    
    def delete(self, *args, **kwargs):
        # Ensure we're in the correct tenant context
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context available")
        
        super().delete(*args, **kwargs)

# Example usage
class Employee(TenantAwareModel):
    employee_id = models.CharField(max_length=50, unique=True)
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    email = models.EmailField()
    department = models.ForeignKey('Department', on_delete=models.SET_NULL, null=True)
    
    class Meta:
        db_table = 'employees'  # Will be prefixed with tenant schema
```

### 3. Tenant Context Management

```python
import threading
from contextvars import ContextVar

# Thread-local storage for tenant context
_tenant_context: ContextVar[Optional['Tenant']] = ContextVar('tenant_context', default=None)

class TenantContext:
    """Manage tenant context throughout request lifecycle"""
    
    @staticmethod
    def set_current_tenant(tenant):
        """Set current tenant in context"""
        _tenant_context.set(tenant)
    
    @staticmethod
    def get_current_tenant():
        """Get current tenant from context"""
        return _tenant_context.get()
    
    @staticmethod
    def clear_tenant():
        """Clear tenant context"""
        _tenant_context.set(None)

def get_current_tenant():
    """Convenience function to get current tenant"""
    return TenantContext.get_current_tenant()

def require_tenant(func):
    """Decorator to ensure tenant context is available"""
    def wrapper(*args, **kwargs):
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("Tenant context required")
        return func(*args, **kwargs)
    return wrapper
```

## Cache Isolation

### 1. Tenant-Scoped Caching

```python
class TenantAwareCache:
    """Cache implementation with tenant isolation"""
    
    def __init__(self, cache_backend):
        self.cache = cache_backend
    
    def _get_tenant_key(self, key):
        """Prefix cache key with tenant ID"""
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context for cache operation")
        
        return f"tenant:{tenant.id}:{key}"
    
    def get(self, key, default=None):
        """Get value from tenant-scoped cache"""
        tenant_key = self._get_tenant_key(key)
        return self.cache.get(tenant_key, default)
    
    def set(self, key, value, timeout=None):
        """Set value in tenant-scoped cache"""
        tenant_key = self._get_tenant_key(key)
        return self.cache.set(tenant_key, value, timeout)
    
    def delete(self, key):
        """Delete value from tenant-scoped cache"""
        tenant_key = self._get_tenant_key(key)
        return self.cache.delete(tenant_key)
    
    def clear_tenant_cache(self, tenant_id=None):
        """Clear all cache entries for a tenant"""
        if not tenant_id:
            tenant = get_current_tenant()
            tenant_id = tenant.id if tenant else None
        
        if tenant_id:
            pattern = f"tenant:{tenant_id}:*"
            keys = self.cache.keys(pattern)
            if keys:
                self.cache.delete_many(keys)

# Usage in Django settings
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Wrap with tenant-aware cache
cache = TenantAwareCache(caches['default'])
```

### 2. Session Management

```python
class TenantAwareSessionMiddleware:
    """Session middleware with tenant isolation"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Get tenant context
        tenant = getattr(request, 'tenant', None)
        
        if tenant:
            # Modify session key to include tenant
            original_session_key = request.session.session_key
            if original_session_key:
                request.session.session_key = f"{tenant.id}:{original_session_key}"
        
        response = self.get_response(request)
        return response
```

## File Storage Isolation

### 1. Tenant-Specific File Organization

```python
class TenantAwareFileStorage:
    """File storage with tenant isolation"""
    
    def __init__(self, base_storage):
        self.base_storage = base_storage
    
    def _get_tenant_path(self, name):
        """Get tenant-specific file path"""
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context for file operation")
        
        return f"tenants/{tenant.id}/{name}"
    
    def save(self, name, content, max_length=None):
        """Save file in tenant-specific directory"""
        tenant_path = self._get_tenant_path(name)
        return self.base_storage.save(tenant_path, content, max_length)
    
    def delete(self, name):
        """Delete file from tenant-specific directory"""
        tenant_path = self._get_tenant_path(name)
        return self.base_storage.delete(tenant_path)
    
    def exists(self, name):
        """Check if file exists in tenant-specific directory"""
        tenant_path = self._get_tenant_path(name)
        return self.base_storage.exists(tenant_path)
    
    def url(self, name):
        """Get URL for file in tenant-specific directory"""
        tenant_path = self._get_tenant_path(name)
        return self.base_storage.url(tenant_path)

# File field with tenant isolation
class TenantAwareFileField(models.FileField):
    def __init__(self, *args, **kwargs):
        kwargs['storage'] = TenantAwareFileStorage(default_storage)
        super().__init__(*args, **kwargs)
```

## Search Index Isolation

### 1. Elasticsearch Tenant Isolation

```python
class TenantAwareElasticsearch:
    """Elasticsearch client with tenant isolation"""
    
    def __init__(self, es_client):
        self.es = es_client
    
    def _get_tenant_index(self, index_name):
        """Get tenant-specific index name"""
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context for search operation")
        
        return f"tenant_{tenant.id}_{index_name}"
    
    def index(self, index, doc_type, body, id=None):
        """Index document in tenant-specific index"""
        tenant_index = self._get_tenant_index(index)
        return self.es.index(
            index=tenant_index,
            doc_type=doc_type,
            body=body,
            id=id
        )
    
    def search(self, index, body):
        """Search in tenant-specific index"""
        tenant_index = self._get_tenant_index(index)
        return self.es.search(index=tenant_index, body=body)
    
    def delete(self, index, doc_type, id):
        """Delete document from tenant-specific index"""
        tenant_index = self._get_tenant_index(index)
        return self.es.delete(
            index=tenant_index,
            doc_type=doc_type,
            id=id
        )
```

## Vector Database Isolation

### 1. Qdrant Tenant Collections

```python
class TenantAwareVectorDB:
    """Vector database client with tenant isolation"""
    
    def __init__(self, qdrant_client):
        self.client = qdrant_client
    
    def _get_tenant_collection(self, collection_name):
        """Get tenant-specific collection name"""
        tenant = get_current_tenant()
        if not tenant:
            raise ValueError("No tenant context for vector operation")
        
        return f"tenant_{tenant.id}_{collection_name}"
    
    async def create_collection(self, collection_name, vector_size):
        """Create tenant-specific collection"""
        tenant_collection = self._get_tenant_collection(collection_name)
        
        return await self.client.recreate_collection(
            collection_name=tenant_collection,
            vectors_config=VectorParams(size=vector_size, distance=Distance.COSINE)
        )
    
    async def upsert(self, collection_name, points):
        """Upsert points to tenant-specific collection"""
        tenant_collection = self._get_tenant_collection(collection_name)
        
        return await self.client.upsert(
            collection_name=tenant_collection,
            points=points
        )
    
    async def search(self, collection_name, query_vector, limit=10, score_threshold=None):
        """Search in tenant-specific collection"""
        tenant_collection = self._get_tenant_collection(collection_name)
        
        return await self.client.search(
            collection_name=tenant_collection,
            query_vector=query_vector,
            limit=limit,
            score_threshold=score_threshold
        )
```

## Tenant Provisioning and Management

### 1. Tenant Provisioning Service

```python
class TenantProvisioningService:
    """Service for creating and managing tenants"""
    
    def __init__(self):
        self.db = get_database_connection()
        self.cache = get_cache_client()
        self.search = get_search_client()
        self.vector_db = get_vector_db_client()
    
    async def create_tenant(self, tenant_data):
        """Create new tenant with full isolation setup"""
        
        # 1. Create tenant record
        tenant = await self._create_tenant_record(tenant_data)
        
        # 2. Create database schema
        await self._create_database_schema(tenant)
        
        # 3. Create search indexes
        await self._create_search_indexes(tenant)
        
        # 4. Create vector collections
        await self._create_vector_collections(tenant)
        
        # 5. Set up default roles and permissions
        await self._setup_default_rbac(tenant)
        
        # 6. Create admin user
        admin_user = await self._create_admin_user(tenant, tenant_data['admin'])
        
        # 7. Initialize default data
        await self._initialize_default_data(tenant)
        
        return tenant
    
    async def _create_database_schema(self, tenant):
        """Create tenant-specific database schema"""
        schema_name = f"tenant_{tenant.id.replace('-', '_')}"
        
        # Create schema
        await self.db.execute(f"CREATE SCHEMA {schema_name}")
        
        # Create all tables in the new schema
        table_definitions = self._get_table_definitions()
        
        for table_sql in table_definitions:
            # Replace placeholder with actual schema name
            tenant_sql = table_sql.replace('{{tenant_schema}}', schema_name)
            await self.db.execute(tenant_sql)
        
        # Create indexes
        index_definitions = self._get_index_definitions()
        for index_sql in index_definitions:
            tenant_sql = index_sql.replace('{{tenant_schema}}', schema_name)
            await self.db.execute(tenant_sql)
    
    async def _create_search_indexes(self, tenant):
        """Create tenant-specific search indexes"""
        indexes = ['contacts', 'products', 'documents', 'employees']
        
        for index_name in indexes:
            tenant_index = f"tenant_{tenant.id}_{index_name}"
            
            # Create index with appropriate mapping
            mapping = self._get_index_mapping(index_name)
            await self.search.indices.create(
                index=tenant_index,
                body={'mappings': mapping}
            )
    
    async def _create_vector_collections(self, tenant):
        """Create tenant-specific vector collections"""
        collections = [
            ('documents', 384),  # sentence-transformers embedding size
            ('products', 384),
            ('conversations', 384)
        ]
        
        for collection_name, vector_size in collections:
            tenant_collection = f"tenant_{tenant.id}_{collection_name}"
            
            await self.vector_db.recreate_collection(
                collection_name=tenant_collection,
                vectors_config=VectorParams(
                    size=vector_size,
                    distance=Distance.COSINE
                )
            )
    
    async def delete_tenant(self, tenant_id):
        """Completely remove tenant and all associated data"""
        
        tenant = await self._get_tenant(tenant_id)
        if not tenant:
            raise ValueError(f"Tenant {tenant_id} not found")
        
        # 1. Delete vector collections
        await self._delete_vector_collections(tenant)
        
        # 2. Delete search indexes
        await self._delete_search_indexes(tenant)
        
        # 3. Clear cache
        await self._clear_tenant_cache(tenant)
        
        # 4. Delete database schema
        await self._delete_database_schema(tenant)
        
        # 5. Delete file storage
        await self._delete_file_storage(tenant)
        
        # 6. Delete tenant record
        await self._delete_tenant_record(tenant)
```

## Security Considerations

### 1. Cross-Tenant Data Leakage Prevention

```python
class TenantSecurityMiddleware:
    """Middleware to prevent cross-tenant data access"""
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.audit_logger = AuditLogger()
    
    def __call__(self, request):
        # Store original tenant context
        original_tenant = get_current_tenant()
        
        try:
            response = self.get_response(request)
            
            # Verify tenant context wasn't changed during request
            current_tenant = get_current_tenant()
            if original_tenant and current_tenant and original_tenant.id != current_tenant.id:
                self.audit_logger.log_security_violation(
                    event_type="TENANT_CONTEXT_VIOLATION",
                    original_tenant=original_tenant.id,
                    current_tenant=current_tenant.id,
                    request=request
                )
                
                return JsonResponse(
                    {'error': 'Security violation detected'},
                    status=403
                )
            
            return response
            
        except Exception as e:
            # Log any exceptions that might indicate security issues
            self.audit_logger.log_error(
                event_type="TENANT_SECURITY_ERROR",
                tenant_id=original_tenant.id if original_tenant else None,
                error=str(e),
                request=request
            )
            raise
```

### 2. Database Query Validation

```python
class TenantQueryValidator:
    """Validate database queries for tenant isolation"""
    
    @staticmethod
    def validate_query(query, tenant_id):
        """Ensure query only accesses tenant-specific data"""
        
        # Check if query contains tenant schema
        expected_schema = f"tenant_{tenant_id.replace('-', '_')}"
        
        if expected_schema not in query and 'shared.' not in query:
            raise SecurityError(
                f"Query does not specify tenant schema: {query}"
            )
        
        # Check for potential cross-tenant access
        forbidden_patterns = [
            r'tenant_[a-f0-9]{8}_[a-f0-9]{4}_[a-f0-9]{4}_[a-f0-9]{4}_[a-f0-9]{12}',
            r'SET search_path',
            r'DROP SCHEMA',
            r'CREATE SCHEMA'
        ]
        
        for pattern in forbidden_patterns:
            if re.search(pattern, query, re.IGNORECASE):
                if expected_schema not in re.findall(pattern, query, re.IGNORECASE):
                    raise SecurityError(
                        f"Potentially unsafe query detected: {query}"
                    )
```

This comprehensive multi-tenant structure ensures complete data isolation while maintaining performance and scalability for the enterprise ERP system.