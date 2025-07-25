# Service Breakdown and Interactions

## Service Architecture Overview

This document details the backend services for each module, their responsibilities, and how they interact within the enterprise ERP system.

## Core Infrastructure Services

### 1. Authentication Service (Django)

**Responsibilities:**
- User authentication and authorization
- JWT token management
- Multi-factor authentication
- Session management
- Password policies and security

**Key Components:**
```python
# apps/authentication/
├── models.py          # User, Token, Session models
├── services.py        # AuthService, TokenService
├── views.py          # Login, logout, refresh endpoints
├── serializers.py    # Auth request/response serializers
├── permissions.py    # Custom permission classes
└── middleware.py     # JWT authentication middleware
```

**API Endpoints:**
- `POST /auth/login` - User authentication
- `POST /auth/refresh` - Token refresh
- `POST /auth/logout` - User logout
- `GET /auth/me` - Current user info

**Database Tables:**
- `users` - User accounts
- `auth_tokens` - JWT tokens and refresh tokens
- `user_sessions` - Active user sessions
- `login_attempts` - Failed login tracking

**Interactions:**
- **→ Tenant Service**: Validate tenant context
- **→ RBAC Service**: Check user permissions
- **→ Redis**: Store session data
- **→ All Services**: Provide authentication middleware

### 2. Tenant Management Service (Django)

**Responsibilities:**
- Multi-tenant data isolation
- Tenant provisioning and configuration
- Resource allocation per tenant
- Tenant-specific settings

**Key Components:**
```python
# apps/tenants/
├── models.py          # Tenant, TenantSettings models
├── middleware.py      # Tenant resolution middleware
├── services.py        # TenantService, IsolationService
├── utils.py          # Tenant context utilities
└── managers.py       # Tenant-aware model managers
```

**Database Schema:**
```sql
-- Tenant isolation using separate schemas
CREATE SCHEMA tenant_abc;
CREATE SCHEMA tenant_xyz;
CREATE SCHEMA shared;

-- Shared tables
shared.tenants
shared.tenant_settings
shared.system_config

-- Tenant-specific tables (per schema)
tenant_abc.users
tenant_abc.employees
tenant_abc.invoices
```

**Interactions:**
- **→ All Business Services**: Provide tenant context
- **→ Database**: Route queries to correct schema
- **→ Cache**: Tenant-scoped caching
- **→ File Storage**: Tenant-specific file organization

### 3. RBAC (Role-Based Access Control) Service (Django)

**Responsibilities:**
- Role and permission management
- Access control enforcement
- Resource-level permissions
- Permission inheritance

**Key Components:**
```python
# apps/rbac/
├── models.py          # Role, Permission, UserRole models
├── permissions.py     # Permission classes and decorators
├── services.py        # RBACService, PermissionService
├── decorators.py      # Permission decorators
└── middleware.py      # Permission enforcement middleware
```

**Permission Matrix:**
```python
PERMISSIONS = {
    'hrm': {
        'employee': ['create', 'read', 'update', 'delete'],
        'payroll': ['read', 'process'],
        'leave': ['create', 'read', 'approve']
    },
    'crm': {
        'contact': ['create', 'read', 'update', 'delete'],
        'lead': ['create', 'read', 'update', 'assign'],
        'opportunity': ['create', 'read', 'update']
    },
    'finance': {
        'invoice': ['create', 'read', 'update', 'send'],
        'payment': ['create', 'read', 'process'],
        'report': ['read', 'generate']
    }
}
```

**Interactions:**
- **→ All Business Services**: Enforce permissions
- **→ Authentication Service**: Validate user context
- **→ Database**: Filter queries based on permissions

## Business Domain Services

### 4. Subscription & Billing Service (Django)

**Responsibilities:**
- Subscription lifecycle management
- Multi-provider payment processing
- Usage tracking and metering
- Feature access control and gating
- Trial management and conversion
- Customer billing portal
- Revenue analytics and reporting

**Service Structure:**
```python
# apps/subscriptions/
├── services/
│   ├── subscription_service.py  # Subscription management
│   ├── billing_service.py       # Billing operations
│   ├── payment_service.py       # Payment processing
│   ├── usage_service.py         # Usage tracking
│   ├── plan_service.py          # Plan management
│   └── feature_service.py       # Feature gating
├── models/
│   ├── subscription.py         # Subscription models
│   ├── plan.py                 # Plan models
│   ├── invoice.py              # Invoice models
│   ├── payment.py              # Payment models
│   └── usage.py                # Usage models
├── integrations/
│   ├── stripe_integration.py   # Stripe payment processor
│   ├── paypal_integration.py   # PayPal integration
│   └── webhook_handlers.py     # Payment webhooks
└── tasks.py                    # Celery billing tasks
```

**Key Business Logic:**
```python
class SubscriptionService:
    def create_subscription(self, organization_id, plan_id, payment_method):
        # Create subscription with trial period
        # Set up payment method
        # Activate plan features
        # Send welcome email
        # Publish subscription_created event
        
    def upgrade_subscription(self, subscription_id, new_plan_id):
        # Calculate proration
        # Process payment difference
        # Update feature access
        # Send upgrade confirmation
        # Publish subscription_upgraded event
        
    def track_usage(self, organization_id, feature, quantity=1):
        # Record usage event
        # Check usage limits
        # Calculate overages
        # Trigger alerts if needed
        # Update usage analytics
```

**Event Publishing:**
```python
# Events published to Kafka
EVENTS = [
    'subscription.created',
    'subscription.upgraded',
    'subscription.downgraded',
    'subscription.cancelled',
    'payment.succeeded',
    'payment.failed',
    'usage.limit_exceeded',
    'trial.expiring',
    'trial.converted'
]
```

**Interactions:**
- **→ Payment Gateways**: Process payments via Stripe, PayPal
- **→ Email Service**: Send billing notifications
- **→ All Business Services**: Enforce feature access and usage limits
- **→ Analytics Service**: Provide billing and usage data
- **← All Business Services**: Receive usage tracking events

### 5. HRM Service (Django)

**Responsibilities:**
- Employee lifecycle management
- Leave and attendance tracking
- Payroll processing
- Performance management

**Service Structure:**
```python
# apps/hrm/
├── services/
│   ├── employee_service.py    # Employee CRUD operations
│   ├── leave_service.py       # Leave management logic
│   ├── attendance_service.py  # Attendance tracking
│   ├── payroll_service.py     # Payroll calculations
│   └── performance_service.py # Performance reviews
├── models/
│   ├── employee.py           # Employee model
│   ├── department.py         # Department model
│   ├── leave.py             # Leave models
│   ├── attendance.py        # Attendance models
│   └── payroll.py           # Payroll models
├── tasks.py                 # Celery background tasks
└── signals.py              # Django signals for events
```

**Key Business Logic:**
```python
class EmployeeService:
    def create_employee(self, data, user_context):
        # Validate permissions
        # Create employee record
        # Send welcome email
        # Create default leave balances
        # Publish employee_created event
        
    def process_leave_request(self, leave_id, action, approver):
        # Validate approver permissions
        # Update leave status
        # Calculate leave balance
        # Send notifications
        # Publish leave_processed event
```

**Event Publishing:**
```python
# Events published to Kafka
EVENTS = [
    'employee.created',
    'employee.updated',
    'employee.terminated',
    'leave.requested',
    'leave.approved',
    'leave.rejected',
    'payroll.processed',
    'attendance.marked'
]
```

**Interactions:**
- **→ Email Service**: Send notifications
- **→ File Service**: Handle document uploads
- **→ Notification Service**: Real-time alerts
- **→ Report Service**: Generate HR reports
- **← AI Service**: Receive AI-generated insights

### 6. CRM Service (Django)

**Responsibilities:**
- Contact and lead management
- Sales pipeline tracking
- Opportunity management
- Customer service ticketing

**Service Structure:**
```python
# apps/crm/
├── services/
│   ├── contact_service.py     # Contact management
│   ├── lead_service.py        # Lead processing
│   ├── opportunity_service.py # Sales pipeline
│   ├── activity_service.py    # Activity tracking
│   └── ticket_service.py      # Customer service
├── models/
│   ├── contact.py            # Contact models
│   ├── lead.py              # Lead models
│   ├── opportunity.py       # Opportunity models
│   └── activity.py          # Activity models
└── integrations/
    ├── email_integration.py  # Email sync
    └── social_integration.py # Social media leads
```

**Lead Scoring Algorithm:**
```python
class LeadScoringService:
    def calculate_score(self, lead):
        score = 0
        
        # Company size scoring
        if lead.company_size > 1000:
            score += 30
        elif lead.company_size > 100:
            score += 20
            
        # Industry scoring
        if lead.industry in ['technology', 'finance']:
            score += 25
            
        # Engagement scoring
        score += lead.email_opens * 2
        score += lead.website_visits * 3
        score += lead.demo_requests * 40
        
        return min(score, 100)
```

**Interactions:**
- **→ Email Service**: Send marketing emails
- **→ Google Plugin**: Gmail integration
- **→ Notification Service**: Lead alerts
- **→ Invoice Service**: Convert opportunities to invoices
- **← AI Service**: Lead scoring and recommendations

### 7. Finance Service (Django)

**Responsibilities:**
- Accounting and bookkeeping
- Invoice and payment processing
- Financial reporting
- Budget management

**Service Structure:**
```python
# apps/finance/
├── services/
│   ├── account_service.py     # Chart of accounts
│   ├── invoice_service.py     # Invoice management
│   ├── payment_service.py     # Payment processing
│   ├── journal_service.py     # Journal entries
│   └── report_service.py      # Financial reports
├── models/
│   ├── account.py            # Account models
│   ├── invoice.py           # Invoice models
│   ├── payment.py           # Payment models
│   └── journal.py           # Journal entry models
└── integrations/
    ├── payment_gateways.py   # Payment gateway APIs
    └── banking_apis.py       # Bank integrations
```

**Double-Entry Bookkeeping:**
```python
class JournalService:
    def create_journal_entry(self, description, entries):
        total_debits = sum(e['debit'] for e in entries)
        total_credits = sum(e['credit'] for e in entries)
        
        if total_debits != total_credits:
            raise ValidationError("Debits must equal credits")
            
        journal_entry = JournalEntry.objects.create(
            description=description,
            date=timezone.now().date()
        )
        
        for entry in entries:
            JournalEntryLine.objects.create(
                journal_entry=journal_entry,
                account_id=entry['account_id'],
                debit=entry.get('debit', 0),
                credit=entry.get('credit', 0)
            )
```

**Interactions:**
- **→ Invoice Service**: Generate invoice PDFs
- **→ Payment Gateway**: Process payments
- **→ Email Service**: Send invoices and receipts
- **→ Report Service**: Generate financial reports
- **← CRM Service**: Receive opportunity data

### 8. Inventory Service (Django)

**Responsibilities:**
- Product catalog management
- Stock level tracking
- Purchase and sales order processing
- Warehouse management

**Service Structure:**
```python
# apps/inventory/
├── services/
│   ├── product_service.py     # Product management
│   ├── stock_service.py       # Stock tracking
│   ├── order_service.py       # Order processing
│   ├── warehouse_service.py   # Warehouse operations
│   └── procurement_service.py # Purchase orders
├── models/
│   ├── product.py            # Product models
│   ├── stock.py             # Stock models
│   ├── order.py             # Order models
│   └── warehouse.py         # Warehouse models
└── algorithms/
    ├── reorder_point.py      # Reorder calculations
    └── demand_forecast.py    # Demand forecasting
```

**Stock Movement Tracking:**
```python
class StockService:
    def update_stock(self, product_id, warehouse_id, quantity, movement_type):
        with transaction.atomic():
            stock = Stock.objects.select_for_update().get(
                product_id=product_id,
                warehouse_id=warehouse_id
            )
            
            old_quantity = stock.quantity
            stock.quantity += quantity
            stock.save()
            
            StockMovement.objects.create(
                product_id=product_id,
                warehouse_id=warehouse_id,
                movement_type=movement_type,
                quantity=quantity,
                old_quantity=old_quantity,
                new_quantity=stock.quantity
            )
            
            # Check reorder point
            if stock.quantity <= stock.reorder_point:
                self.trigger_reorder_alert(stock)
```

**Interactions:**
- **→ Finance Service**: Create purchase invoices
- **→ Email Service**: Send order confirmations
- **→ Report Service**: Generate inventory reports
- **← AI Service**: Demand forecasting insights

### 9. Project Management Service (Django)

**Responsibilities:**
- Project and task management
- Resource allocation
- Time tracking
- Project reporting

**Service Structure:**
```python
# apps/projects/
├── services/
│   ├── project_service.py     # Project management
│   ├── task_service.py        # Task operations
│   ├── time_service.py        # Time tracking
│   ├── resource_service.py    # Resource allocation
│   └── collaboration_service.py # Team collaboration
├── models/
│   ├── project.py            # Project models
│   ├── task.py              # Task models
│   ├── timesheet.py         # Time tracking models
│   └── resource.py          # Resource models
└── algorithms/
    ├── scheduling.py         # Task scheduling
    └── capacity_planning.py  # Resource planning
```

**Resource Allocation Algorithm:**
```python
class ResourceService:
    def allocate_resources(self, project_id, requirements):
        available_resources = self.get_available_resources()
        
        allocation = {}
        for requirement in requirements:
            best_match = self.find_best_resource_match(
                requirement, available_resources
            )
            
            if best_match:
                allocation[requirement.id] = best_match
                available_resources.remove(best_match)
                
        return allocation
```

**Interactions:**
- **→ HRM Service**: Get employee availability
- **→ Finance Service**: Project billing
- **→ Email Service**: Project notifications
- **→ File Service**: Project document management

## Microservices (Go)

### 10. Invoice Generation Service (Go)

**Responsibilities:**
- PDF invoice generation
- Invoice template management
- Batch invoice processing
- Invoice delivery

**Service Structure:**
```go
// services/microservices/invoice-service/
├── cmd/main.go
├── internal/
│   ├── handlers/
│   │   ├── grpc_handler.go    # gRPC endpoints
│   │   └── http_handler.go    # HTTP endpoints
│   ├── services/
│   │   ├── invoice_service.go # Core business logic
│   │   ├── pdf_service.go     # PDF generation
│   │   └── template_service.go # Template management
│   ├── models/
│   │   └── invoice.go         # Invoice data structures
│   └── repository/
│       ├── postgres.go        # Database operations
│       └── redis.go          # Caching operations
```

**Key Functions:**
```go
type InvoiceService struct {
    db     *sql.DB
    cache  *redis.Client
    logger *zap.Logger
}

func (s *InvoiceService) GenerateInvoice(ctx context.Context, req *pb.GenerateInvoiceRequest) (*pb.GenerateInvoiceResponse, error) {
    // Fetch invoice data
    invoice, err := s.getInvoiceData(req.InvoiceId)
    if err != nil {
        return nil, err
    }
    
    // Generate PDF
    pdfBytes, err := s.generatePDF(invoice)
    if err != nil {
        return nil, err
    }
    
    // Store PDF
    fileURL, err := s.storePDF(pdfBytes, invoice.ID)
    if err != nil {
        return nil, err
    }
    
    // Send email if requested
    if req.SendEmail {
        err = s.sendInvoiceEmail(invoice, fileURL)
        if err != nil {
            s.logger.Error("Failed to send invoice email", zap.Error(err))
        }
    }
    
    return &pb.GenerateInvoiceResponse{
        Success: true,
        FileUrl: fileURL,
    }, nil
}
```

**Event Consumption:**
```go
// Kafka event handlers
func (s *InvoiceService) handleOrderCreated(event OrderCreatedEvent) {
    // Generate invoice from order
    // Send to customer
    // Update order status
}

func (s *InvoiceService) handlePaymentReceived(event PaymentReceivedEvent) {
    // Mark invoice as paid
    // Send receipt
    // Update accounting records
}
```

### 11. Report Generation Service (Go)

**Responsibilities:**
- Complex report generation
- Data aggregation and analysis
- Scheduled report delivery
- Report caching and optimization

**Service Structure:**
```go
// services/microservices/report-service/
├── internal/
│   ├── services/
│   │   ├── report_service.go     # Core reporting logic
│   │   ├── aggregation_service.go # Data aggregation
│   │   ├── export_service.go     # Export formats
│   │   └── scheduler_service.go  # Scheduled reports
│   ├── models/
│   │   ├── report.go            # Report definitions
│   │   └── data.go              # Data structures
│   └── repository/
│       ├── postgres.go          # Data queries
│       └── mongodb.go           # Analytics data
```

**Report Generation:**
```go
func (s *ReportService) GenerateFinancialReport(ctx context.Context, req *pb.FinancialReportRequest) (*pb.ReportResponse, error) {
    // Query financial data
    data, err := s.getFinancialData(req.StartDate, req.EndDate, req.TenantId)
    if err != nil {
        return nil, err
    }
    
    // Aggregate data
    aggregated := s.aggregateFinancialData(data)
    
    // Generate report based on format
    var reportBytes []byte
    switch req.Format {
    case "PDF":
        reportBytes, err = s.generatePDFReport(aggregated)
    case "EXCEL":
        reportBytes, err = s.generateExcelReport(aggregated)
    case "CSV":
        reportBytes, err = s.generateCSVReport(aggregated)
    }
    
    if err != nil {
        return nil, err
    }
    
    // Store report
    fileURL, err := s.storeReport(reportBytes, req.Format)
    if err != nil {
        return nil, err
    }
    
    return &pb.ReportResponse{
        Success: true,
        FileUrl: fileURL,
        Format:  req.Format,
    }, nil
}
```

### 12. Email Service (Go)

**Responsibilities:**
- Email delivery and tracking
- Template management
- Bulk email processing
- Email analytics

**Service Structure:**
```go
// services/microservices/email-service/
├── internal/
│   ├── services/
│   │   ├── email_service.go     # Core email logic
│   │   ├── template_service.go  # Template processing
│   │   ├── delivery_service.go  # Email delivery
│   │   └── tracking_service.go  # Email tracking
│   ├── providers/
│   │   ├── smtp.go             # SMTP provider
│   │   ├── sendgrid.go         # SendGrid provider
│   │   └── ses.go              # AWS SES provider
```

**Email Processing:**
```go
func (s *EmailService) SendEmail(ctx context.Context, req *pb.SendEmailRequest) (*pb.SendEmailResponse, error) {
    // Process template
    content, err := s.processTemplate(req.TemplateId, req.Data)
    if err != nil {
        return nil, err
    }
    
    // Create email message
    message := &EmailMessage{
        To:      req.Recipients,
        Subject: content.Subject,
        Body:    content.Body,
        IsHTML:  content.IsHTML,
    }
    
    // Send email
    messageId, err := s.deliveryService.Send(message)
    if err != nil {
        return nil, err
    }
    
    // Track email
    s.trackingService.TrackSent(messageId, req.Recipients)
    
    return &pb.SendEmailResponse{
        Success:   true,
        MessageId: messageId,
    }, nil
}
```

### 13. File Processing Service (Go)

**Responsibilities:**
- File upload and storage
- Image processing and thumbnails
- Document parsing and indexing
- Virus scanning

**Service Structure:**
```go
// services/microservices/file-service/
├── internal/
│   ├── services/
│   │   ├── upload_service.go    # File upload handling
│   │   ├── storage_service.go   # Storage operations
│   │   ├── processing_service.go # File processing
│   │   └── security_service.go  # Security scanning
│   ├── processors/
│   │   ├── image_processor.go   # Image processing
│   │   ├── document_processor.go # Document parsing
│   │   └── video_processor.go   # Video processing
```

### 14. Search Service (Go)

**Responsibilities:**
- Elasticsearch integration
- Search indexing and updates
- Search query processing
- Search analytics

**Service Structure:**
```go
// services/microservices/search-service/
├── internal/
│   ├── services/
│   │   ├── search_service.go    # Search operations
│   │   ├── indexing_service.go  # Index management
│   │   └── analytics_service.go # Search analytics
│   ├── indexers/
│   │   ├── contact_indexer.go   # Contact indexing
│   │   ├── product_indexer.go   # Product indexing
│   │   └── document_indexer.go  # Document indexing
```

## AI Services

### 15. AI Gateway Service (Python/FastAPI)

**Responsibilities:**
- AI request routing and management
- Conversation context handling
- AI action execution
- Model provider abstraction

**Service Structure:**
```python
# services/ai/ai-gateway/
├── services/
│   ├── chat_service.py          # Chat processing
│   ├── action_service.py        # AI action execution
│   ├── context_service.py       # Context management
│   └── provider_service.py      # LLM provider management
├── models/
│   ├── conversation.py          # Conversation models
│   ├── action.py               # Action models
│   └── context.py              # Context models
```

### 16. RAG Engine Service (Python)

**Responsibilities:**
- Document embedding and indexing
- Semantic search and retrieval
- Context augmentation
- Knowledge base management

**Service Structure:**
```python
# services/ai/rag-engine/
├── services/
│   ├── embedding_service.py     # Document embedding
│   ├── retrieval_service.py     # Information retrieval
│   ├── generation_service.py    # Response generation
│   └── indexing_service.py      # Vector indexing
├── pipelines/
│   ├── ingestion_pipeline.py    # Data ingestion
│   ├── processing_pipeline.py   # Data processing
│   └── update_pipeline.py       # Real-time updates
```

## Service Communication Patterns

### Synchronous Communication (gRPC)
```protobuf
// Proto definition for service communication
service InvoiceService {
  rpc GenerateInvoice(GenerateInvoiceRequest) returns (GenerateInvoiceResponse);
  rpc GetInvoiceStatus(GetInvoiceStatusRequest) returns (InvoiceStatusResponse);
}

service ReportService {
  rpc GenerateReport(GenerateReportRequest) returns (GenerateReportResponse);
  rpc ScheduleReport(ScheduleReportRequest) returns (ScheduleReportResponse);
}
```

### Asynchronous Communication (Kafka Events)
```json
{
  "event_type": "invoice.created",
  "tenant_id": "tenant-123",
  "data": {
    "invoice_id": "inv-456",
    "customer_id": "cust-789",
    "amount": 1500.00,
    "due_date": "2024-02-15"
  },
  "timestamp": "2024-01-15T10:00:00Z",
  "version": "1.0"
}
```

### Circuit Breaker Pattern
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if self._should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                raise CircuitBreakerOpenException()
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
```

This comprehensive service breakdown provides a clear understanding of how each service operates and interacts within the enterprise ERP ecosystem.