# Dev-First Setup Guide

## 🚀 One-Command Development Environment

This guide shows how to get the entire Intelligent ERP Suite running locally with a single command, complete with seeded data, hot-reloading, and local LLM support.

## Prerequisites

- Docker and Docker Compose
- Make
- Git
- Node.js 18+ (for frontend development)
- Python 3.11+ (for AI/backend development)
- Go 1.21+ (for auth/notification services)

## Quick Start

```bash
# Clone the infrastructure repository
git clone https://github.com/intelligent-erp-suite/erp-infrastructure
cd erp-infrastructure

# Start everything with one command
make up

# Wait 2-3 minutes for all services to start and seed data
# Then open http://localhost:3000 for the frontend
# API Gateway available at http://localhost:8080
```

## What `make up` Does

1. **Clones all repositories** (if not already present)
2. **Starts all 14 services** with Docker Compose
3. **Seeds databases** with realistic test data
4. **Sets up local LLM** (Ollama) for AI development
5. **Configures hot-reloading** for all services
6. **Sets up service discovery** and routing
7. **Initializes monitoring** and logging

## Development Commands

```bash
# Start all services
make up

# Stop all services
make down

# Restart specific service
make restart service=erp-crm-service

# View logs for all services
make logs

# View logs for specific service
make logs service=erp-auth-service

# Reset all data (fresh start)
make reset

# Run tests across all services
make test

# Generate API documentation
make docs

# Check service health
make health
```

## Service URLs

| Service | URL | Description |
|---------|-----|-------------|
| Frontend | http://localhost:3000 | Main ERP interface |
| API Gateway | http://localhost:8080 | API entry point |
| Auth Service | http://localhost:8001 | Authentication |
| Subscription Service | http://localhost:8002 | Billing & subscriptions |
| CRM Service | http://localhost:8003 | Customer management |
| HRM Service | http://localhost:8004 | Human resources |
| Accounting Service | http://localhost:8005 | Finance & accounting |
| Inventory Service | http://localhost:8006 | Inventory management |
| Project Service | http://localhost:8007 | Project management |
| AI Platform | http://localhost:8008 | AI services |
| Notification Service | http://localhost:8009 | Notifications |
| File Service | http://localhost:8010 | File management |

## Development Dashboard

Access the development dashboard at http://localhost:3000/dev which provides:

- Service health status
- Database connection status
- Real-time logs
- API documentation links
- Test data management
- Performance metrics

## Seeded Test Data

The system comes with pre-loaded test data:

### Organizations & Users
- **Organization**: "Acme Corp" (tenant_id: acme-corp)
- **Admin User**: admin@acme.com / password123
- **HR Manager**: hr@acme.com / password123
- **Sales Rep**: sales@acme.com / password123

### Sample Data
- **Employees**: 50 employees across 5 departments
- **Contacts**: 100 contacts and 20 companies
- **Products**: 200 products across 10 categories
- **Projects**: 10 active projects with tasks
- **Invoices**: 50 invoices with various statuses

## Hot Reloading Setup

### Backend Services (Django/FastAPI)
- Code changes trigger automatic reloads
- Database migrations run automatically
- API documentation updates in real-time

### Frontend (Next.js)
- React components hot-reload instantly
- TypeScript compilation happens automatically
- Tailwind CSS changes apply immediately

### Go Services (Auth/Notifications)
- Uses Air for hot reloading
- Binary rebuilds on code changes
- Configuration reloads automatically

## Local LLM Setup

The AI platform comes with local LLM support:

```bash
# Check if Ollama is running
curl http://localhost:11434/api/tags

# Available models (pre-downloaded):
# - llama2:7b (general purpose)
# - codellama:7b (code generation)
# - mistral:7b (fast responses)

# Test AI integration
curl -X POST http://localhost:8008/ai/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Show me all employees in the sales department"}'
```

## Development Workflow

### 1. Feature Development
```bash
# Create feature branch
git checkout -b feature/new-crm-feature

# Make changes to any service
# Hot reloading handles the rest

# Run tests
make test service=erp-crm-service

# Commit and push
git add .
git commit -m "Add new CRM feature"
git push origin feature/new-crm-feature
```

### 2. Database Changes
```bash
# Make model changes in any Django service
# Migrations are auto-generated and applied

# Or manually create migration
make migrate service=erp-crm-service

# Reset database if needed
make reset-db service=erp-crm-service
```

### 3. API Development
```bash
# Add new endpoint to any service
# API documentation updates automatically

# View updated docs
open http://localhost:8080/docs

# Test new endpoint
make test-api endpoint=/api/crm/contacts
```

## RBAC Development Pattern

### Backend (Django)
```python
# apps/crm/views.py
from erp_shared.auth import RBACViewSet, require_permission

class ContactViewSet(RBACViewSet):
    """Contacts with automatic RBAC"""
    
    @require_permission('crm.contact.read')
    def list(self, request):
        # Automatic tenant filtering applied
        return super().list(request)
    
    @require_permission('crm.contact.write')
    def create(self, request):
        # Automatic audit logging
        return super().create(request)
```

### Frontend (React)
```typescript
// components/ContactList.tsx
import { withPermission } from '@erp-shared/react-rbac';

const ContactList = withPermission('crm.contact.read')(
  () => {
    // Component only renders if user has permission
    return <div>Contact List</div>;
  }
);

export default ContactList;
```

## Debugging

### Service Logs
```bash
# View all logs
make logs

# View specific service logs
make logs service=erp-crm-service

# Follow logs in real-time
make logs-follow service=erp-crm-service
```

### Database Access
```bash
# Connect to any service database
make db-connect service=erp-crm-service

# Run database queries
make db-query service=erp-crm-service query="SELECT * FROM contacts LIMIT 5"
```

### Performance Monitoring
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3001
- Jaeger: http://localhost:16686

## Troubleshooting

### Common Issues

**Services won't start**
```bash
# Check Docker resources
docker system df

# Clean up if needed
make clean
docker system prune -f
```

**Database connection errors**
```bash
# Reset databases
make reset-db

# Check database status
make db-status
```

**Hot reloading not working**
```bash
# Restart specific service
make restart service=erp-crm-service

# Check file watchers
make debug-reload
```

### Getting Help

1. Check service health: `make health`
2. View logs: `make logs service=<service-name>`
3. Reset environment: `make reset`
4. Check documentation: http://localhost:3000/dev

## Production Deployment

When ready for production:

```bash
# Build production images
make build-prod

# Deploy to staging
make deploy-staging

# Deploy to production
make deploy-prod
```

This dev-first setup ensures you can start coding immediately with a fully functional, realistic development environment that mirrors production behavior.