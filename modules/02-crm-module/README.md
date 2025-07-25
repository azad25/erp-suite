# Customer Relationship Management (CRM) Module

## 👥 Module Overview

The CRM module provides comprehensive customer relationship management capabilities including contact management, lead pipeline, opportunity tracking, and sales analytics. It's designed to help businesses manage their entire customer lifecycle from initial contact to closed deals.

**Repository**: `erp-crm-module`  
**Development Timeline**: Months 3-4  
**Team Size**: 3-4 developers  
**Dependencies**: `erp-auth-module`

## 🎯 Business Objectives

- Centralize customer and prospect information
- Streamline lead management and conversion process
- Track sales opportunities and pipeline
- Provide sales analytics and forecasting
- Improve customer relationship management
- Increase sales team productivity

## ✨ Key Features

### 📇 Contact & Company Management
- Comprehensive contact database
- Company profiles and hierarchies
- Contact categorization and tagging
- Communication history tracking
- Document and file attachments

### 🎯 Lead Management
- Lead capture and qualification
- Lead scoring and prioritization
- Pipeline stage management
- Lead assignment and routing
- Conversion tracking

### 💼 Opportunity Tracking
- Sales opportunity management
- Deal value and probability tracking
- Sales stage progression
- Forecasting and reporting
- Win/loss analysis

### 📊 Sales Analytics
- Sales performance dashboards
- Pipeline analytics
- Conversion rate tracking
- Revenue forecasting
- Activity reporting

### 📧 Communication Integration
- Email integration and tracking
- Call logging and notes
- Meeting scheduling
- Task and follow-up management
- Communication timeline

## 🛠️ Technical Specifications

### Technology Stack
- **Backend**: Django 4.2+ with Django REST Framework
- **Database**: PostgreSQL 15+ (separate CRM database)
- **Cache**: Redis 7.0+ (shared with other services)
- **API**: RESTful APIs with OpenAPI documentation
- **Email**: Django Email backend integration
- **Authentication**: JWT token validation (from auth-service)

### Architecture Pattern
```
crm-module/
├── backend/
│   ├── apps/
│   │   ├── contacts/
│   │   ├── companies/
│   │   ├── leads/
│   │   ├── opportunities/
│   │   ├── activities/
│   │   └── analytics/
│   ├── core/
│   │   ├── models/
│   │   ├── serializers/
│   │   ├── views/
│   │   └── utils/
│   ├── api/
│   │   ├── v1/
│   │   └── urls.py
│   └── tests/
├── docs/
│   ├── api-spec.yaml
│   └── README.md
└── docker/
    ├── Dockerfile
    └── docker-compose.yml
```

## 📋 Detailed Requirements

### Functional Requirements

#### FR-1: Contact Management
- **FR-1.1**: System shall store comprehensive contact information
- **FR-1.2**: System shall support contact categorization and tagging
- **FR-1.3**: System shall track communication history per contact
- **FR-1.4**: System shall support contact import/export
- **FR-1.5**: System shall prevent duplicate contacts

#### FR-2: Company Management
- **FR-2.1**: System shall manage company profiles and hierarchies
- **FR-2.2**: System shall link contacts to companies
- **FR-2.3**: System shall track company-level activities
- **FR-2.4**: System shall support company categorization

#### FR-3: Lead Management
- **FR-3.1**: System shall capture leads from multiple sources
- **FR-3.2**: System shall provide lead qualification workflow
- **FR-3.3**: System shall support lead scoring and prioritization
- **FR-3.4**: System shall enable lead assignment and routing
- **FR-3.5**: System shall track lead conversion rates

#### FR-4: Opportunity Management
- **FR-4.1**: System shall manage sales opportunities and deals
- **FR-4.2**: System shall track deal values and probabilities
- **FR-4.3**: System shall support customizable sales stages
- **FR-4.4**: System shall provide opportunity forecasting
- **FR-4.5**: System shall enable win/loss analysis

#### FR-5: Activity Management
- **FR-5.1**: System shall log calls, emails, and meetings
- **FR-5.2**: System shall support task creation and assignment
- **FR-5.3**: System shall provide activity scheduling
- **FR-5.4**: System shall track activity completion
- **FR-5.5**: System shall generate activity reports

#### FR-6: Analytics & Reporting
- **FR-6.1**: System shall provide sales performance dashboards
- **FR-6.2**: System shall generate pipeline analytics
- **FR-6.3**: System shall track conversion rates
- **FR-6.4**: System shall provide revenue forecasting
- **FR-6.5**: System shall support custom report generation

### Non-Functional Requirements

#### NFR-1: Performance
- Contact search response time < 300ms
- Dashboard load time < 3 seconds
- Support for 10,000+ contacts per organization
- Efficient pagination for large datasets

#### NFR-2: Usability
- Intuitive drag-and-drop pipeline interface
- Mobile-responsive design
- Keyboard shortcuts for power users
- Bulk operations support

#### NFR-3: Integration
- Email integration capabilities
- Calendar integration support
- Import/export functionality
- API for third-party integrations

## 🗄️ Database Schema

### Core Tables

#### Contacts
```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    company_id UUID REFERENCES companies(id),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    mobile VARCHAR(50),
    title VARCHAR(100),
    department VARCHAR(100),
    address JSONB,
    social_profiles JSONB,
    tags TEXT[],
    notes TEXT,
    lead_source VARCHAR(100),
    assigned_to UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Companies
```sql
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    industry VARCHAR(100),
    size VARCHAR(50),
    website VARCHAR(255),
    phone VARCHAR(50),
    address JSONB,
    description TEXT,
    tags TEXT[],
    assigned_to UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Leads
```sql
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    contact_id UUID REFERENCES contacts(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    source VARCHAR(100),
    status VARCHAR(50) DEFAULT 'new',
    score INTEGER DEFAULT 0,
    value DECIMAL(12,2),
    probability INTEGER DEFAULT 0,
    expected_close_date DATE,
    assigned_to UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Opportunities
```sql
CREATE TABLE opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    lead_id UUID REFERENCES leads(id),
    contact_id UUID REFERENCES contacts(id),
    company_id UUID REFERENCES companies(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    stage VARCHAR(100) DEFAULT 'qualification',
    value DECIMAL(12,2) NOT NULL,
    probability INTEGER DEFAULT 0,
    expected_close_date DATE,
    actual_close_date DATE,
    status VARCHAR(50) DEFAULT 'open',
    assigned_to UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

#### Activities
```sql
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    contact_id UUID REFERENCES contacts(id),
    lead_id UUID REFERENCES leads(id),
    opportunity_id UUID REFERENCES opportunities(id),
    type VARCHAR(50) NOT NULL, -- call, email, meeting, task
    subject VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(50) DEFAULT 'pending',
    scheduled_at TIMESTAMP,
    completed_at TIMESTAMP,
    assigned_to UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

## 🔌 API Specifications

### Contact Endpoints

#### GET /api/crm/contacts/
```json
{
  "page": 1,
  "page_size": 20,
  "search": "john",
  "tags": ["vip", "customer"],
  "assigned_to": "user-uuid"
}
```

#### POST /api/crm/contacts/
```json
{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john.doe@example.com",
  "phone": "+1-555-0123",
  "company_id": "company-uuid",
  "title": "CEO",
  "tags": ["prospect", "enterprise"]
}
```

### Lead Endpoints

#### GET /api/crm/leads/pipeline/
```json
{
  "stage": "qualification",
  "assigned_to": "user-uuid",
  "date_range": "this_month"
}
```

#### PUT /api/crm/leads/{id}/stage/
```json
{
  "stage": "proposal",
  "notes": "Moved to proposal stage after demo"
}
```

### Opportunity Endpoints

#### GET /api/crm/opportunities/forecast/
```json
{
  "period": "quarter",
  "team": "sales-team-uuid"
}
```

#### POST /api/crm/opportunities/
```json
{
  "name": "Enterprise Software Deal",
  "contact_id": "contact-uuid",
  "value": 50000.00,
  "probability": 75,
  "expected_close_date": "2024-03-15",
  "stage": "negotiation"
}
```

## 🎨 User Interface Design

### Dashboard Layout
- **Pipeline View**: Kanban-style board with drag-and-drop
- **List View**: Tabular view with sorting and filtering
- **Analytics View**: Charts and metrics dashboard
- **Activity Feed**: Recent activities and updates

### Key Components
- **Contact Cards**: Compact contact information display
- **Lead Pipeline**: Visual pipeline with stage progression
- **Activity Timeline**: Chronological activity display
- **Analytics Charts**: Sales performance visualizations

## 🧪 Testing Strategy

### Unit Tests
- Contact CRUD operations
- Lead scoring algorithms
- Opportunity calculations
- Activity tracking logic

### Integration Tests
- API endpoint testing
- Database operations
- Authentication integration
- Email integration

### End-to-End Tests
- Contact management workflow
- Lead conversion process
- Opportunity pipeline management
- Analytics dashboard functionality

## 🚀 Deployment Guide

### Development Setup
```bash
# Ensure auth-service is running
cd ../01-auth-module && docker-compose up -d

# Setup CRM service
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver 8001
```

### Docker Deployment
```yaml
version: '3.8'
services:
  postgres-crm:
    image: postgres:15
    environment:
      POSTGRES_DB: crm_module
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    
  crm-service:
    build: ./backend
    depends_on:
      - postgres-crm
    environment:
      AUTH_SERVICE_URL: http://auth-service:8000
      DATABASE_URL: postgresql://postgres:password@postgres-crm:5432/crm_module
      REDIS_URL: redis://redis:6379
    ports:
      - "8001:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 📈 Success Metrics

### Technical Metrics
- **API Response Time**: <300ms for contact searches
- **Dashboard Load Time**: <3 seconds
- **Data Accuracy**: >99% for contact information
- **Uptime**: >99.9%

### Business Metrics
- **Lead Conversion Rate**: Track improvement over time
- **Sales Cycle Length**: Monitor reduction in sales cycle
- **Pipeline Velocity**: Measure deal progression speed
- **User Adoption**: Track active users and feature usage

## 🔄 Integration Points

### Auth Module Integration
- User authentication and authorization
- Role-based access control
- Organization-based data isolation
- User assignment and ownership

### Future Module Integrations
- **Accounting Module**: Invoice generation from opportunities
- **Email Module**: Email campaign integration
- **Calendar Module**: Meeting and appointment scheduling
- **AI Module**: Lead scoring and sales predictions

### External Integrations
- **Email Providers**: Gmail, Outlook integration
- **Calendar Systems**: Google Calendar, Outlook Calendar
- **Communication Tools**: Slack, Microsoft Teams
- **Marketing Tools**: Mailchimp, HubSpot

## 📚 Documentation Structure

```
docs/
├── README.md              # This file
├── requirements.md        # Detailed business requirements
├── api-specification.md   # Complete API documentation
├── database-schema.md     # Database design and relationships
├── user-guide.md         # End-user documentation
├── integration-guide.md   # Integration with other modules
└── deployment-guide.md    # Production deployment guide
```

## 🎯 Development Phases

### Phase 1: Core Foundation (Week 1-2)
- Contact and company management
- Basic CRUD operations
- Authentication integration

### Phase 2: Lead Management (Week 3-4)
- Lead capture and qualification
- Pipeline management
- Lead assignment

### Phase 3: Opportunities (Week 5-6)
- Opportunity tracking
- Sales stages
- Forecasting

### Phase 4: Analytics & Polish (Week 7-8)
- Sales analytics dashboard
- Reporting features
- UI/UX improvements
- Testing and deployment

This CRM module provides a solid foundation for customer relationship management while maintaining integration capabilities with other ERP modules.