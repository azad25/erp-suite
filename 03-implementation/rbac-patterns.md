# RBAC Patterns and Shared Utilities

This document shows the dev-first RBAC patterns that enable rapid development with automatic permission checking, tenant isolation, and audit logging.

## Shared Libraries Structure

### Python Package (erp-shared-libs/python/erp_shared/)

#### auth/decorators.py
```python
from functools import wraps
from django.http import JsonResponse
from django.core.exceptions import PermissionDenied
from .services import AuthService, SubscriptionService

def require_permission(permission):
    """Decorator to require specific permission"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            auth_service = AuthService()
            
            # Get user from JWT token
            user = auth_service.get_user_from_request(request)
            if not user:
                return JsonResponse({'error': 'Authentication required'}, status=401)
            
            # Check permission
            if not auth_service.user_has_permission(user, permission):
                return JsonResponse({'error': f'Permission {permission} required'}, status=403)
            
            # Add user to request
            request.user = user
            request.tenant_id = user.tenant_id
            
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

def require_role(role):
    """Decorator to require specific role"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            auth_service = AuthService()
            
            user = auth_service.get_user_from_request(request)
            if not user:
                return JsonResponse({'error': 'Authentication required'}, status=401)
            
            if not auth_service.user_has_role(user, role):
                return JsonResponse({'error': f'Role {role} required'}, status=403)
            
            request.user = user
            request.tenant_id = user.tenant_id
            
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

def require_subscription(feature=None):
    """Decorator to check subscription status and feature access"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            subscription_service = SubscriptionService()
            
            # Check if subscription is active
            if not subscription_service.is_subscription_active(request.tenant_id):
                return JsonResponse({
                    'error': 'Subscription required',
                    'upgrade_url': '/billing/upgrade'
                }, status=402)
            
            # Check feature access if specified
            if feature and not subscription_service.has_feature_access(request.tenant_id, feature):
                return JsonResponse({
                    'error': f'Feature {feature} not available in current plan',
                    'upgrade_url': '/billing/upgrade'
                }, status=402)
            
            return view_func(request, *args, **kwargs)
        return wrapper
    return decorator

def track_usage(usage_type, quantity=1):
    """Decorator to track usage for billing"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(request, *args, **kwargs):
            subscription_service = SubscriptionService()
            
            # Check usage limits
            if not subscription_service.check_usage_limit(request.tenant_id, usage_type, quantity):
                return JsonResponse({
                    'error': f'Usage limit exceeded for {usage_type}',
                    'upgrade_url': '/billing/upgrade'
                }, status=429)
            
            # Execute the view
            response = view_func(request, *args, **kwargs)
            
            # Track usage if successful
            if response.status_code < 400:
                subscription_service.record_usage(request.tenant_id, usage_type, quantity)
            
            return response
        return wrapper
    return decorator
```

#### auth/viewsets.py
```python
from rest_framework import viewsets, status
from rest_framework.response import Response
from django.db import models
from .services import AuthService, AuditService

class RBACViewSet(viewsets.ModelViewSet):
    """Base ViewSet with automatic RBAC, tenant isolation, and audit logging"""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.auth_service = AuthService()
        self.audit_service = AuditService()
    
    def get_queryset(self):
        """Automatically filter by tenant"""
        queryset = super().get_queryset()
        
        # Apply tenant filtering if model has tenant_id field
        if hasattr(queryset.model, 'tenant_id'):
            queryset = queryset.filter(tenant_id=self.request.tenant_id)
        
        return queryset
    
    def perform_create(self, serializer):
        """Add tenant_id and audit logging on create"""
        # Add tenant_id if model supports it
        if hasattr(serializer.Meta.model, 'tenant_id'):
            serializer.save(tenant_id=self.request.tenant_id)
        else:
            serializer.save()
        
        # Log the action
        self.audit_service.log_action(
            user=self.request.user,
            action='create',
            model=serializer.Meta.model.__name__,
            object_id=serializer.instance.id,
            data=serializer.validated_data
        )
    
    def perform_update(self, serializer):
        """Add audit logging on update"""
        old_data = serializer.instance.__dict__.copy()
        serializer.save()
        
        self.audit_service.log_action(
            user=self.request.user,
            action='update',
            model=serializer.Meta.model.__name__,
            object_id=serializer.instance.id,
            old_data=old_data,
            new_data=serializer.validated_data
        )
    
    def perform_destroy(self, instance):
        """Add audit logging on delete"""
        self.audit_service.log_action(
            user=self.request.user,
            action='delete',
            model=instance.__class__.__name__,
            object_id=instance.id,
            data=instance.__dict__
        )
        instance.delete()

class RBACPermissionMixin:
    """Mixin to add permission checking to ViewSets"""
    
    permission_map = {
        'list': 'read',
        'retrieve': 'read',
        'create': 'write',
        'update': 'write',
        'partial_update': 'write',
        'destroy': 'delete'
    }
    
    def get_required_permission(self):
        """Get required permission for current action"""
        action = self.action
        permission_type = self.permission_map.get(action, 'read')
        model_name = self.get_queryset().model._meta.model_name
        return f"{model_name}.{permission_type}"
    
    def check_permissions(self, request):
        """Check permissions for current action"""
        super().check_permissions(request)
        
        required_permission = self.get_required_permission()
        auth_service = AuthService()
        
        if not auth_service.user_has_permission(request.user, required_permission):
            raise PermissionDenied(f"Permission {required_permission} required")
```

#### auth/middleware.py
```python
from django.utils.deprecation import MiddlewareMixin
from django.http import JsonResponse
from .services import AuthService, SubscriptionService

class AuthMiddleware(MiddlewareMixin):
    """Middleware to handle authentication and tenant context"""
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.auth_service = AuthService()
        super().__init__(get_response)
    
    def process_request(self, request):
        # Skip auth for health checks and public endpoints
        if request.path in ['/health', '/docs', '/openapi.json']:
            return None
        
        # Get user from JWT token
        user = self.auth_service.get_user_from_request(request)
        if user:
            request.user = user
            request.tenant_id = user.tenant_id
        else:
            request.user = None
            request.tenant_id = None
        
        return None

class SubscriptionMiddleware(MiddlewareMixin):
    """Middleware to check subscription status"""
    
    def __init__(self, get_response):
        self.get_response = get_response
        self.subscription_service = SubscriptionService()
        super().__init__(get_response)
    
    def process_request(self, request):
        # Skip for non-authenticated requests
        if not hasattr(request, 'tenant_id') or not request.tenant_id:
            return None
        
        # Check subscription status
        if not self.subscription_service.is_subscription_active(request.tenant_id):
            return JsonResponse({
                'error': 'Subscription expired or inactive',
                'upgrade_url': '/billing/upgrade'
            }, status=402)
        
        return None
```

#### models/base.py
```python
from django.db import models
import uuid

class TenantAwareModel(models.Model):
    """Base model with tenant isolation"""
    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    tenant_id = models.CharField(max_length=100, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True
        indexes = [
            models.Index(fields=['tenant_id', 'created_at']),
        ]

class AuditableModel(TenantAwareModel):
    """Base model with audit fields"""
    
    created_by = models.CharField(max_length=100, null=True, blank=True)
    updated_by = models.CharField(max_length=100, null=True, blank=True)
    
    class Meta:
        abstract = True
```

### TypeScript Package (erp-shared-libs/typescript/src/)

#### auth/rbac-hooks.ts
```typescript
import { useAuth } from './auth-context';
import { useSubscription } from './subscription-context';

export const usePermission = (permission: string) => {
  const { user } = useAuth();
  
  return {
    hasPermission: user?.permissions?.includes(permission) || false,
    loading: !user
  };
};

export const useRole = (role: string) => {
  const { user } = useAuth();
  
  return {
    hasRole: user?.roles?.includes(role) || false,
    loading: !user
  };
};

export const useFeatureAccess = (feature: string) => {
  const { subscription } = useSubscription();
  
  return {
    hasAccess: subscription?.features?.includes(feature) || false,
    loading: !subscription
  };
};
```

#### auth/rbac-components.tsx
```typescript
import React from 'react';
import { usePermission, useRole, useFeatureAccess } from './rbac-hooks';

interface RBACProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

interface PermissionProps extends RBACProps {
  permission: string;
}

interface RoleProps extends RBACProps {
  role: string;
}

interface FeatureProps extends RBACProps {
  feature: string;
}

export const WithPermission: React.FC<PermissionProps> = ({ 
  permission, 
  children, 
  fallback = null 
}) => {
  const { hasPermission, loading } = usePermission(permission);
  
  if (loading) return <div>Loading...</div>;
  if (!hasPermission) return <>{fallback}</>;
  
  return <>{children}</>;
};

export const WithRole: React.FC<RoleProps> = ({ 
  role, 
  children, 
  fallback = null 
}) => {
  const { hasRole, loading } = useRole(role);
  
  if (loading) return <div>Loading...</div>;
  if (!hasRole) return <>{fallback}</>;
  
  return <>{children}</>;
};

export const WithFeature: React.FC<FeatureProps> = ({ 
  feature, 
  children, 
  fallback = null 
}) => {
  const { hasAccess, loading } = useFeatureAccess(feature);
  
  if (loading) return <div>Loading...</div>;
  if (!hasAccess) return <>{fallback}</>;
  
  return <>{children}</>;
};

// HOC versions
export const withPermission = (permission: string) => 
  <P extends object>(Component: React.ComponentType<P>) => 
    (props: P) => (
      <WithPermission permission={permission}>
        <Component {...props} />
      </WithPermission>
    );

export const withRole = (role: string) => 
  <P extends object>(Component: React.ComponentType<P>) => 
    (props: P) => (
      <WithRole role={role}>
        <Component {...props} />
      </WithRole>
    );

export const withFeature = (feature: string) => 
  <P extends object>(Component: React.ComponentType<P>) => 
    (props: P) => (
      <WithFeature feature={feature}>
        <Component {...props} />
      </WithFeature>
    );
```

## Usage Examples

### Django Service Example (erp-crm-service)

#### apps/crm/views.py
```python
from rest_framework import viewsets
from erp_shared.auth import RBACViewSet, require_permission, require_subscription, track_usage
from erp_shared.auth.decorators import require_role
from .models import Contact, Lead
from .serializers import ContactSerializer, LeadSerializer

class ContactViewSet(RBACViewSet):
    """Contact management with automatic RBAC"""
    queryset = Contact.objects.all()
    serializer_class = ContactSerializer
    
    @require_permission('crm.contact.read')
    @require_subscription('crm_module')
    @track_usage('api_calls')
    def list(self, request):
        return super().list(request)
    
    @require_permission('crm.contact.write')
    @require_subscription('crm_module')
    @track_usage('contacts_created')
    def create(self, request):
        return super().create(request)

class LeadViewSet(RBACViewSet):
    """Lead management with role-based access"""
    queryset = Lead.objects.all()
    serializer_class = LeadSerializer
    
    @require_role('sales_rep')
    @require_subscription('crm_advanced')
    def create(self, request):
        return super().create(request)
    
    @require_role('sales_manager')
    def destroy(self, request, pk=None):
        return super().destroy(request, pk)
```

#### apps/crm/models.py
```python
from django.db import models
from erp_shared.models import TenantAwareModel, AuditableModel

class Contact(AuditableModel):
    """Contact model with automatic tenant isolation"""
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    email = models.EmailField()
    company = models.CharField(max_length=200, blank=True)
    phone = models.CharField(max_length=20, blank=True)
    
    class Meta:
        db_table = 'crm_contacts'
        indexes = [
            models.Index(fields=['tenant_id', 'email']),
            models.Index(fields=['tenant_id', 'company']),
        ]

class Lead(AuditableModel):
    """Lead model with automatic tenant isolation"""
    title = models.CharField(max_length=200)
    contact = models.ForeignKey(Contact, on_delete=models.CASCADE)
    value = models.DecimalField(max_digits=10, decimal_places=2)
    stage = models.CharField(max_length=50, default='new')
    probability = models.IntegerField(default=0)
    
    class Meta:
        db_table = 'crm_leads'
```

### React Frontend Example

#### components/ContactList.tsx
```typescript
import React from 'react';
import { WithPermission, WithFeature } from '@erp-shared/react-rbac';

const ContactList: React.FC = () => {
  return (
    <WithFeature 
      feature="crm_module"
      fallback={<div>CRM module not available in your plan</div>}
    >
      <WithPermission 
        permission="crm.contact.read"
        fallback={<div>You don't have permission to view contacts</div>}
      >
        <div>
          <h1>Contacts</h1>
          {/* Contact list implementation */}
        </div>
      </WithPermission>
    </WithFeature>
  );
};

export default ContactList;
```

#### components/CreateContact.tsx
```typescript
import React from 'react';
import { withPermission, withFeature } from '@erp-shared/react-rbac';

const CreateContactForm: React.FC = () => {
  return (
    <form>
      <h2>Create New Contact</h2>
      {/* Form implementation */}
    </form>
  );
};

// Apply RBAC with HOCs
export default withFeature('crm_module')(
  withPermission('crm.contact.write')(CreateContactForm)
);
```

This RBAC pattern provides:
- **Automatic tenant isolation** in all database queries
- **Permission checking** at the API and UI level
- **Subscription-based feature gating** 
- **Usage tracking** for billing
- **Audit logging** for all actions
- **Graceful degradation** when features aren't available

The dev-first approach means developers can focus on business logic while RBAC, tenant isolation, and subscription checking happen automatically.