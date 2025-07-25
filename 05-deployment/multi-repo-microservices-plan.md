# Multi-Repository Microservices Development Plan

## Overview

This document outlines the comprehensive plan for developing the Intelligent ERP Suite using a module-based microservices architecture with a multi-repository approach. Each business module and supporting service will have its own repository, enabling independent development, deployment, and scaling.

## Repository Architecture

### Total Repositories: 14

#### Core Infrastructure (3 Repositories)

**1. `erp-shared-libs`** - Foundation Libraries
- **Purpose**: Common utilities, types, and shared code
- **Technology**: Python, Go, TypeScript, Protocol Buffers
- **Team**: Platform Team
- **Dependencies**: None (base dependency for all services)

**2. `erp-api-gateway`** - API Gateway Service  
- **Purpose**: Request routing, authentication, rate limiting
- **Technology**: Kong/Nginx + Python/Go
- **Team**: Platform Team
- **Dependencies**: erp-shared-libs, erp-auth-service

**3. `erp-infrastructure`** - Infrastructure as Code
- **Purpose**: Deployment configs, monitoring, CI/CD templates
- **Technology**: Docker, Kubernetes, Terraform, Helm
- **Team**: DevOps/Platform Team
- **Dependencies**: None

#### Core Business Services (7 Repositories)

**4. `erp-auth-service`** - Authentication & Authorization
- **Purpose**: Multi-tenant auth, RBAC, JWT management
- **Technology**: Django/FastAPI + PostgreSQL
- **Team**: Security/Platform Team
- **Dependencies**: erp-shared-libs

**5. `erp-subscription-service`** - Subscription & Billing Management
- **Purpose**: Subscription lifecycle, billing, payment processing, usage tracking, feature gating
- **Technology**: Django + DRF + Celery + PostgreSQL + Redis
- **Team**: Subscription Team
- **Dependencies**: erp-shared-libs, erp-auth-service

**6. `erp-hrm-service`** - Human Resource Management
- **Purpose**: Employee, leave, attendance, payroll management
- **Technology**: Django + PostgreSQL
- **Team**: HRM Team
- **Dependencies**: erp-shared-libs, erp-auth-service, erp-subscription-service

**7. `erp-crm-service`** - Customer Relationship Management
- **Purpose**: Contacts, leads, opportunities, sales pipeline
- **Technology**: Django + PostgreSQL
- **Team**: CRM Team  
- **Dependencies**: erp-shared-libs, erp-auth-service, erp-subscription-service

**8. `erp-finance-service`** - Finance & Accounting
- **Purpose**: Accounting, invoicing, payments, financial reports
- **Technology**: Django + PostgreSQL
- **Team**: Finance Team
- **Dependencies**: erp-shared-libs, erp-auth-service, erp-subscription-service

**9. `erp-inventory-service`** - Inventory Management
- **Purpose**: Products, warehouses, stock, orders
- **Technology**: Django + PostgreSQL  
- **Team**: Inventory Team
- **Dependencies**: erp-shared-libs, erp-auth-service, erp-subscription-service

**10. `erp-project-service`** - Project Management
- **Purpose**: Projects, tasks, time tracking, resources
- **Technology**: Django + PostgreSQL
- **Team**: Project Team
- **Dependencies**: erp-shared-libs, erp-auth-service, erp-subscription-service#### Sup
port Services (2 Repositories)

**11. `erp-notification-service`** - Notifications & Communication
- **Purpose**: Email, SMS, push notifications, WebSocket management
- **Technology**: Go + Redis + Kafka
- **Team**: Platform Team
- **Dependencies**: erp-shared-libs, erp-auth-service

**12. `erp-file-service`** - File Management
- **Purpose**: File upload/download, document management, PDF generation
- **Technology**: Go + Object Storage (S3/MinIO)
- **Team**: Platform Team
- **Dependencies**: erp-shared-libs, erp-auth-service

#### AI and Frontend (2 Repositories)

**13. `erp-ai-platform`** - AI Services & Chatbot
- **Purpose**: AI Gateway, RAG Engine, Chatbot, Plugin system
- **Technology**: Python/FastAPI + Vector DB + LLM APIs
- **Team**: AI Team
- **Dependencies**: erp-shared-libs, erp-auth-service

**14. `erp-frontend`** - Frontend Applications
- **Purpose**: Next.js web app, admin dashboard, mobile app
- **Technology**: Next.js/React + TypeScript
- **Team**: Frontend Team
- **Dependencies**: erp-shared-libs (TypeScript types)

## Development Timeline by Repository

### Phase 1: Foundation (Months 1-3)

#### Month 1: Infrastructure Setup
**Week 1-2: Core Infrastructure**
- [ ] **`erp-infrastructure`**: Set up repository with Docker, K8s, Terraform
- [ ] **`erp-shared-libs`**: Create base repository with common utilities
- [ ] **CI/CD Templates**: Create reusable GitHub Actions workflows

**Week 3-4: Authentication Foundation**
- [ ] **`erp-auth-service`**: Implement multi-tenant authentication
- [ ] **`erp-api-gateway`**: Set up API gateway with routing
- [ ] **Integration**: Connect auth service with API gateway

#### Month 2: Shared Libraries and Standards
**Week 5-6: Shared Libraries Development**
- [ ] **Python Package**: Authentication, database, validation utilities
- [ ] **Go Module**: Common types, database connections, middleware
- [ ] **TypeScript Package**: API types, client utilities, validation

**Week 7-8: Development Standards**
- [ ] **Code Templates**: Service repository templates
- [ ] **Testing Framework**: Shared testing utilities and patterns
- [ ] **Documentation**: API documentation standards and tools

#### Month 3: Subscription Service (SaaS Foundation)
**Week 9-10: `erp-subscription-service` Foundation**
- [ ] **Repository Setup**: Initialize subscription service
- [ ] **Subscription Models**: Plan, subscription, invoice models
- [ ] **Payment Integration**: Stripe and PayPal integration
- [ ] **Feature Gating**: Subscription-based access control

**Week 11-12: Billing and Usage Tracking**
- [ ] **Billing Engine**: Automated billing and invoicing
- [ ] **Usage Tracking**: Real-time usage monitoring
- [ ] **Customer Portal**: Self-service subscription management
- [ ] **Integration**: Connect with auth service for feature gating

### Phase 2: Core Business Services (Months 4-13)

#### Month 4-5: CRM Service
**Week 13-14: `erp-crm-service` Foundation**
- [ ] **Repository Setup**: Initialize with service template
- [ ] **Database Models**: Contact, Lead, Opportunity models
- [ ] **Basic APIs**: CRUD operations for core entities
- [ ] **Subscription Integration**: Feature gating and usage tracking

**Week 15-16: CRM Features**
- [ ] **Contact Management**: Complete contact lifecycle
- [ ] **Sales Pipeline**: Lead to opportunity conversion
- [ ] **Activity Tracking**: Communication and task logging
- [ ] **Testing**: Unit and integration tests

#### Month 6-7: HRM Service
**Week 17-18: `erp-hrm-service` Foundation**
- [ ] **Repository Setup**: Initialize HRM service
- [ ] **Database Models**: Employee, Department, Leave models
- [ ] **Basic APIs**: CRUD operations for core entities
- [ ] **Subscription Integration**: User limits and feature gating

**Week 19-20: HRM Features**
- [ ] **Employee Management**: Complete employee lifecycle
- [ ] **Leave System**: Leave application and approval workflow
- [ ] **Attendance**: Time tracking and attendance management
- [ ] **Analytics**: Basic HRM reporting and metrics

#### Month 6: Finance Service
**Week 21-22: `erp-finance-service` Foundation**
- [ ] **Repository Setup**: Initialize finance service
- [ ] **Accounting Models**: Chart of accounts, journal entries
- [ ] **Double-Entry System**: Implement accounting principles
- [ ] **Multi-Currency**: Currency support and conversion

**Week 23-24: Finance Features**
- [ ] **Invoice Management**: Invoice generation and tracking
- [ ] **Payment Processing**: Payment recording and reconciliation
- [ ] **Financial Reports**: Income statement, balance sheet
- [ ] **Integration**: Connect with file service for PDF generation

#### Month 7: Inventory Service
**Week 25-26: `erp-inventory-service` Foundation**
- [ ] **Repository Setup**: Initialize inventory service
- [ ] **Product Models**: Product catalog and variants
- [ ] **Warehouse Models**: Multi-warehouse support
- [ ] **Stock Tracking**: Real-time inventory tracking

**Week 27-28: Inventory Features**
- [ ] **Order Management**: Purchase and sales orders
- [ ] **Stock Movements**: Track all inventory changes
- [ ] **Reorder Management**: Automated reorder points
- [ ] **Integration**: Connect with finance for costing

#### Month 8: Project Service
**Week 29-30: `erp-project-service` Foundation**
- [ ] **Repository Setup**: Initialize project service
- [ ] **Project Models**: Project, task, and resource models
- [ ] **Time Tracking**: Timesheet and hour logging
- [ ] **Resource Management**: Team and resource allocation

**Week 31-32: Project Features**
- [ ] **Task Management**: Advanced task features and dependencies
- [ ] **Collaboration**: Team communication and file sharing
- [ ] **Project Analytics**: Progress tracking and reporting
- [ ] **Integration**: Connect with HRM for resource data

### Phase 3: Support Services (Months 9-11)

#### Month 9: Notification Service
**Week 33-34: `erp-notification-service`**
- [ ] **Repository Setup**: Initialize notification service in Go
- [ ] **Email System**: SMTP and email template management
- [ ] **Real-time Notifications**: WebSocket implementation
- [ ] **Event Processing**: Kafka event consumption

**Week 35-36: Advanced Notifications**
- [ ] **SMS Integration**: Twilio or similar SMS service
- [ ] **Push Notifications**: Mobile and web push notifications
- [ ] **Notification Preferences**: User preference management
- [ ] **Analytics**: Notification delivery tracking

#### Month 10: File Service
**Week 37-38: `erp-file-service`**
- [ ] **Repository Setup**: Initialize file service in Go
- [ ] **File Storage**: Object storage integration (S3/MinIO)
- [ ] **Upload/Download**: Secure file transfer APIs
- [ ] **Document Management**: File organization and metadata

**Week 39-40: Advanced File Features**
- [ ] **PDF Generation**: Invoice and report PDF creation
- [ ] **Image Processing**: Thumbnail generation and optimization
- [ ] **Google Drive**: Integration with Google Drive API
- [ ] **Security**: File access control and virus scanning

#### Month 11: Integration and Optimization
**Week 41-42: Service Integration**
- [ ] **Cross-Service Testing**: End-to-end integration tests
- [ ] **Performance Testing**: Load testing for all services
- [ ] **Security Audit**: Security review of all services
- [ ] **Documentation**: Complete API documentation

**Week 43-44: Optimization**
- [ ] **Performance Tuning**: Database and API optimization
- [ ] **Caching Strategy**: Implement Redis caching
- [ ] **Monitoring**: Enhanced monitoring and alerting
- [ ] **Deployment**: Production deployment preparation

### Phase 4: AI Platform (Months 12-15)

#### Month 12: AI Foundation
**Week 45-46: `erp-ai-platform` Setup**
- [ ] **Repository Setup**: Initialize AI platform
- [ ] **Plugin System**: Core plugin architecture
- [ ] **AI Gateway**: Request routing and model management
- [ ] **Security**: AI-specific security measures

**Week 47-48: LLM Integration**
- [ ] **OpenAI Plugin**: GPT model integration
- [ ] **Local LLM Plugin**: Ollama/local model support
- [ ] **Model Management**: Model switching and configuration
- [ ] **Usage Tracking**: Token usage and cost monitoring

#### Month 13: RAG Engine
**Week 49-50: RAG Implementation**
- [ ] **Document Processing**: Text extraction and chunking
- [ ] **Vector Database**: Qdrant integration and indexing
- [ ] **Embedding Generation**: Document embedding pipeline
- [ ] **Semantic Search**: Context retrieval system

**Week 51-52: RAG Optimization**
- [ ] **Permission Filtering**: RBAC-aware document retrieval
- [ ] **Real-time Sync**: Live data synchronization
- [ ] **Performance**: Optimize retrieval speed and accuracy
- [ ] **Testing**: RAG accuracy and performance testing

#### Month 14: AI Chatbot
**Week 53-54: Chatbot Core**
- [ ] **Chat Interface**: Basic chat functionality
- [ ] **Intent Recognition**: Natural language understanding
- [ ] **Context Management**: Conversation state handling
- [ ] **Response Generation**: AI response formatting

**Week 55-56: AI Actions**
- [ ] **Action Framework**: Execute business operations via AI
- [ ] **ERP Integration**: Connect AI with all business services
- [ ] **Permission Validation**: Ensure AI respects user permissions
- [ ] **Audit Logging**: Track all AI actions and decisions

#### Month 15: AI Enhancement
**Week 57-58: Advanced AI Features**
- [ ] **Predictive Analytics**: Sales and inventory forecasting
- [ ] **Smart Suggestions**: Context-aware recommendations
- [ ] **Workflow Automation**: AI-driven process automation
- [ ] **Personalization**: User-specific AI behavior

**Week 59-60: AI Testing and Optimization**
- [ ] **AI Testing**: Comprehensive AI functionality testing
- [ ] **Performance**: Optimize AI response times
- [ ] **Accuracy**: Improve AI response quality
- [ ] **Documentation**: AI user guides and API docs

### Phase 5: Frontend and Production (Months 16-18)

#### Month 16: Frontend Development
**Week 61-62: `erp-frontend` Foundation**
- [ ] **Repository Setup**: Next.js application setup
- [ ] **Authentication**: Frontend auth integration
- [ ] **API Integration**: Connect with all backend services
- [ ] **UI Framework**: Design system and component library

**Week 63-64: Frontend Features**
- [ ] **Module UIs**: Interfaces for all ERP modules
- [ ] **Dashboard**: Real-time analytics dashboard
- [ ] **AI Chat**: Chatbot interface integration
- [ ] **Mobile Responsive**: Mobile-friendly design

#### Month 17: Production Preparation
**Week 65-66: Production Setup**
- [ ] **Infrastructure**: Production environment setup
- [ ] **Security**: Security hardening and penetration testing
- [ ] **Performance**: Load testing and optimization
- [ ] **Monitoring**: Production monitoring and alerting

**Week 67-68: Deployment**
- [ ] **Staging Deployment**: Deploy to staging environment
- [ ] **User Testing**: User acceptance testing
- [ ] **Bug Fixes**: Address testing feedback
- [ ] **Production Deployment**: Go-live preparation

#### Month 18: Launch and Optimization
**Week 69-70: Production Launch**
- [ ] **Go-Live**: Production system launch
- [ ] **User Training**: End-user training and support
- [ ] **Monitoring**: Intensive production monitoring
- [ ] **Issue Resolution**: Address any production issues

**Week 71-72: Post-Launch Optimization**
- [ ] **Performance Tuning**: Optimize based on real usage
- [ ] **User Feedback**: Collect and implement user feedback
- [ ] **Documentation**: Finalize all documentation
- [ ] **Handover**: Complete project handover

## Repository Management Strategy

### Shared Libraries Versioning
```
erp-shared-libs versions:
├── v1.0.0 - Initial release
├── v1.1.0 - Added validation utilities
├── v1.2.0 - Enhanced auth helpers
└── v2.0.0 - Breaking changes (major refactor)
```

### Service Dependencies
```
Service Version Matrix:
├── erp-auth-service v1.0.0 → erp-shared-libs v1.0.0
├── erp-hrm-service v1.0.0 → erp-shared-libs v1.1.0
├── erp-crm-service v1.0.0 → erp-shared-libs v1.1.0
└── erp-finance-service v1.0.0 → erp-shared-libs v1.2.0
```

### Cross-Repository CI/CD
```yaml
# Shared library update triggers
erp-shared-libs (updated) →
├── Trigger erp-auth-service CI
├── Trigger erp-hrm-service CI
├── Trigger erp-crm-service CI
└── Trigger all dependent services
```

## Team Structure and Ownership

### Development Teams (9 Teams)
1. **Platform Team** (4 people): Infrastructure, API Gateway, Shared Libraries, Notifications, Files
2. **Auth Team** (2 people): Authentication service
3. **Subscription Team** (3 people): Subscription & billing service (SaaS core)
4. **CRM Team** (3 people): CRM service
5. **HRM Team** (3 people): HRM service
6. **Finance Team** (3 people): Finance service
7. **Inventory Team** (3 people): Inventory service
8. **Project Team** (2 people): Project management service
9. **AI Team** (4 people): AI platform and chatbot
10. **Frontend Team** (4 people): Web and mobile applications

### Cross-Team Coordination
- **Architecture Review Board**: Weekly architecture decisions
- **API Design Committee**: API contract reviews
- **Release Coordination**: Bi-weekly release planning
- **Security Review**: Monthly security assessments

This multi-repository approach enables independent development while maintaining system coherence through shared libraries, standardized processes, and coordinated releases.## Det
ailed Repository Configurations

### 1. erp-shared-libs Repository Structure
```
erp-shared-libs/
├── python/
│   ├── erp_shared/
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── jwt_utils.py
│   │   │   ├── rbac.py
│   │   │   └── middleware.py
│   │   ├── database/
│   │   │   ├── __init__.py
│   │   │   ├── connection.py
│   │   │   ├── models.py
│   │   │   └── migrations.py
│   │   ├── validation/
│   │   │   ├── __init__.py
│   │   │   ├── schemas.py
│   │   │   └── validators.py
│   │   └── utils/
│   │       ├── __init__.py
│   │       ├── logging.py
│   │       ├── encryption.py
│   │       └── helpers.py
│   ├── setup.py
│   ├── requirements.txt
│   └── tests/
├── go/
│   ├── auth/
│   │   ├── jwt.go
│   │   ├── middleware.go
│   │   └── rbac.go
│   ├── database/
│   │   ├── connection.go
│   │   ├── models.go
│   │   └── migrations.go
│   ├── utils/
│   │   ├── logging.go
│   │   ├── validation.go
│   │   └── helpers.go
│   ├── go.mod
│   ├── go.sum
│   └── tests/
├── typescript/
│   ├── src/
│   │   ├── types/
│   │   │   ├── auth.ts
│   │   │   ├── common.ts
│   │   │   └── api.ts
│   │   ├── utils/
│   │   │   ├── validation.ts
│   │   │   ├── formatting.ts
│   │   │   └── api-client.ts
│   │   └── index.ts
│   ├── package.json
│   ├── tsconfig.json
│   └── tests/
├── proto/
│   ├── auth.proto
│   ├── common.proto
│   ├── hrm.proto
│   ├── crm.proto
│   ├── finance.proto
│   ├── inventory.proto
│   └── projects.proto
├── .github/
│   └── workflows/
│       ├── publish-python.yml
│       ├── publish-go.yml
│       ├── publish-npm.yml
│       └── trigger-dependents.yml
└── README.md
```

### 2. Service Repository Template Structure
```
erp-[module]-service/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd-staging.yml
│       ├── cd-production.yml
│       └── security-scan.yml
├── src/
│   ├── models/
│   ├── services/
│   ├── controllers/
│   ├── middleware/
│   ├── utils/
│   └── main.py (or main.go)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── docs/
│   ├── api/
│   ├── deployment/
│   └── development.md
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── docker-compose.test.yml
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── secrets.yaml
├── scripts/
│   ├── setup.sh
│   ├── test.sh
│   ├── build.sh
│   └── deploy.sh
├── requirements.txt (or go.mod)
├── .env.example
├── .gitignore
└── README.md
```

### 3. CI/CD Pipeline Templates

#### Shared Library CI/CD
```yaml
# erp-shared-libs/.github/workflows/publish.yml
name: Publish Shared Libraries
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  publish-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Build and publish Python package
        run: |
          cd python
          python setup.py sdist bdist_wheel
          pip install twine
          twine upload dist/*
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_TOKEN }}

  publish-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      - name: Tag Go module
        run: |
          cd go
          git tag go/v${{ github.ref_name }}
          git push origin go/v${{ github.ref_name }}

  publish-npm:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          registry-url: 'https://registry.npmjs.org'
      - name: Build and publish NPM package
        run: |
          cd typescript
          npm ci
          npm run build
          npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

  trigger-dependents:
    needs: [publish-python, publish-go, publish-npm]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        repo: [
          'erp-auth-service',
          'erp-hrm-service',
          'erp-crm-service',
          'erp-finance-service',
          'erp-inventory-service',
          'erp-project-service',
          'erp-notification-service',
          'erp-file-service',
          'erp-ai-platform',
          'erp-frontend'
        ]
    steps:
      - name: Trigger dependent service
        uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.REPO_ACCESS_TOKEN }}
          repository: company/${{ matrix.repo }}
          event-type: shared-lib-updated
          client-payload: '{"version": "${{ github.ref_name }}"}'
```

#### Service CI/CD Template
```yaml
# Service repository CI/CD template
name: Service CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  repository_dispatch:
    types: [shared-lib-updated]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Update shared libraries
      if: github.event_name == 'repository_dispatch'
      run: |
        pip install --upgrade erp-shared==${{ github.event.client_payload.version }}
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install -r requirements-test.txt
    
    - name: Run tests
      run: |
        python -m pytest tests/ --cov=src --cov-report=xml
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379/0
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Bandit security scan
      run: |
        pip install bandit
        bandit -r src/ -f json -o bandit-report.json
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - name: Deploy to staging
      run: |
        kubectl set image deployment/${{ github.event.repository.name }} \
          ${{ github.event.repository.name }}=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
          -n staging
        kubectl rollout status deployment/${{ github.event.repository.name }} -n staging
```

## Inter-Service Communication Patterns

### 1. Synchronous Communication (gRPC)
```protobuf
// proto/hrm.proto
syntax = "proto3";

package erp.hrm.v1;

service HRMService {
  rpc GetEmployee(GetEmployeeRequest) returns (GetEmployeeResponse);
  rpc CreateEmployee(CreateEmployeeRequest) returns (CreateEmployeeResponse);
  rpc UpdateEmployee(UpdateEmployeeRequest) returns (UpdateEmployeeResponse);
  rpc DeleteEmployee(DeleteEmployeeRequest) returns (DeleteEmployeeResponse);
  rpc GetEmployeesByDepartment(GetEmployeesByDepartmentRequest) returns (GetEmployeesByDepartmentResponse);
}

message Employee {
  string id = 1;
  string tenant_id = 2;
  string employee_id = 3;
  string first_name = 4;
  string last_name = 5;
  string email = 6;
  string department_id = 7;
  string position = 8;
  string status = 9;
  int64 created_at = 10;
  int64 updated_at = 11;
}

message GetEmployeeRequest {
  string id = 1;
  string tenant_id = 2;
}

message GetEmployeeResponse {
  Employee employee = 1;
}
```

### 2. Asynchronous Communication (Kafka Events)
```json
{
  "event_type": "employee.created",
  "version": "1.0",
  "timestamp": "2024-01-15T10:00:00Z",
  "tenant_id": "tenant-123",
  "source_service": "erp-hrm-service",
  "data": {
    "employee_id": "emp-456",
    "first_name": "John",
    "last_name": "Doe",
    "email": "john.doe@company.com",
    "department_id": "dept-789",
    "position": "Software Engineer",
    "hire_date": "2024-01-15"
  },
  "metadata": {
    "correlation_id": "corr-123",
    "user_id": "user-456",
    "trace_id": "trace-789"
  }
}
```

### 3. Event Handlers Example
```python
# erp-notification-service event handler
from kafka import KafkaConsumer
import json

class EmployeeEventHandler:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'employee.events',
            bootstrap_servers=['kafka:9092'],
            value_deserializer=lambda x: json.loads(x.decode('utf-8'))
        )
    
    def handle_employee_created(self, event):
        """Send welcome email to new employee"""
        employee_data = event['data']
        
        # Send welcome email
        self.email_service.send_template_email(
            to=employee_data['email'],
            template='employee_welcome',
            data={
                'first_name': employee_data['first_name'],
                'position': employee_data['position']
            }
        )
        
        # Notify HR team
        self.notification_service.send_notification(
            recipients=['hr-team'],
            message=f"New employee {employee_data['first_name']} {employee_data['last_name']} has been added",
            type='info'
        )
    
    def process_events(self):
        for message in self.consumer:
            event = message.value
            
            if event['event_type'] == 'employee.created':
                self.handle_employee_created(event)
            elif event['event_type'] == 'employee.updated':
                self.handle_employee_updated(event)
```

## Release Coordination Strategy

### 1. Release Orchestration
```yaml
# erp-release-orchestrator/release-configs/v1.0.0.yml
release:
  version: "1.0.0"
  name: "Initial Release"
  date: "2024-06-01"
  
services:
  - name: erp-shared-libs
    version: "1.0.0"
    repository: "company/erp-shared-libs"
    
  - name: erp-auth-service
    version: "1.0.0"
    repository: "company/erp-auth-service"
    dependencies:
      - erp-shared-libs: "1.0.0"
    
  - name: erp-hrm-service
    version: "1.0.0"
    repository: "company/erp-hrm-service"
    dependencies:
      - erp-shared-libs: "1.0.0"
      - erp-auth-service: "1.0.0"
    
  - name: erp-crm-service
    version: "1.0.0"
    repository: "company/erp-crm-service"
    dependencies:
      - erp-shared-libs: "1.0.0"
      - erp-auth-service: "1.0.0"

deployment_order:
  1. erp-shared-libs
  2. erp-auth-service
  3. [erp-hrm-service, erp-crm-service, erp-finance-service]
  4. [erp-notification-service, erp-file-service]
  5. erp-api-gateway
  6. erp-frontend

testing:
  integration_tests:
    - repository: "company/erp-integration-tests"
    - environment: "staging"
    - timeout: "30m"
  
  smoke_tests:
    - repository: "company/erp-smoke-tests"
    - environment: "production"
    - timeout: "10m"
```

### 2. Release Automation Script
```bash
#!/bin/bash
# erp-release-orchestrator/scripts/coordinate-release.sh

RELEASE_CONFIG=$1
ENVIRONMENT=${2:-staging}

echo "Starting coordinated release from config: $RELEASE_CONFIG"

# Parse release configuration
SERVICES=$(yq eval '.services[].name' $RELEASE_CONFIG)
DEPLOYMENT_ORDER=$(yq eval '.deployment_order[]' $RELEASE_CONFIG)

# Deploy services in order
for STAGE in $DEPLOYMENT_ORDER; do
    echo "Deploying stage: $STAGE"
    
    # Parse services in this stage
    if [[ $STAGE == *"["* ]]; then
        # Parallel deployment
        STAGE_SERVICES=$(echo $STAGE | tr -d '[]' | tr ',' ' ')
        for SERVICE in $STAGE_SERVICES; do
            deploy_service $SERVICE $ENVIRONMENT &
        done
        wait # Wait for all parallel deployments
    else
        # Sequential deployment
        deploy_service $STAGE $ENVIRONMENT
    fi
    
    # Health check after each stage
    health_check_stage $STAGE $ENVIRONMENT
done

# Run integration tests
echo "Running integration tests..."
run_integration_tests $ENVIRONMENT

# Run smoke tests if production
if [[ $ENVIRONMENT == "production" ]]; then
    echo "Running smoke tests..."
    run_smoke_tests $ENVIRONMENT
fi

echo "Release completed successfully!"
```

## Monitoring and Observability

### 1. Service-Specific Metrics
```python
# Each service implements standard metrics
from prometheus_client import Counter, Histogram, Gauge

class ServiceMetrics:
    def __init__(self, service_name):
        self.service_name = service_name
        
        # Request metrics
        self.request_count = Counter(
            'http_requests_total',
            'Total HTTP requests',
            ['method', 'endpoint', 'status_code', 'tenant_id']
        )
        
        self.request_duration = Histogram(
            'http_request_duration_seconds',
            'HTTP request duration',
            ['method', 'endpoint', 'tenant_id']
        )
        
        # Business metrics
        self.business_operations = Counter(
            f'{service_name}_operations_total',
            f'Total {service_name} operations',
            ['operation_type', 'tenant_id']
        )
        
        # Health metrics
        self.health_status = Gauge(
            f'{service_name}_health_status',
            f'{service_name} health status'
        )
```

### 2. Distributed Tracing
```python
# Shared tracing configuration
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

def setup_tracing(service_name):
    trace.set_tracer_provider(TracerProvider())
    tracer = trace.get_tracer_provider()
    
    jaeger_exporter = JaegerExporter(
        agent_host_name="jaeger",
        agent_port=6831,
    )
    
    span_processor = BatchSpanProcessor(jaeger_exporter)
    tracer.add_span_processor(span_processor)
    
    return trace.get_tracer(service_name)

# Usage in service
tracer = setup_tracing("erp-hrm-service")

@tracer.start_as_current_span("create_employee")
def create_employee(employee_data):
    # Business logic here
    pass
```

This comprehensive multi-repository microservices plan provides a structured approach to developing the ERP system with clear service boundaries, independent development capabilities, and coordinated release management.