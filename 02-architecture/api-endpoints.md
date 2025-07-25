# REST and GraphQL API Endpoints

## API Design Overview

This document outlines the comprehensive API structure for the enterprise ERP system, covering both REST and GraphQL endpoints for core services and AI/chatbot modules.

## Base URL Structure

```
Production: https://api.erp-suite.com/v1
Staging: https://staging-api.erp-suite.com/v1
Development: http://localhost:8000/api/v1
```

## Authentication Endpoints

### REST Authentication APIs

```http
POST /auth/login
POST /auth/logout
POST /auth/refresh
POST /auth/register
POST /auth/forgot-password
POST /auth/reset-password
POST /auth/verify-email
POST /auth/change-password
GET  /auth/me
POST /auth/mfa/enable
POST /auth/mfa/verify
POST /auth/mfa/disable
```

#### Login Example
```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@company.com",
  "password": "securepassword",
  "tenant_domain": "company"
}

Response:
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
  "user": {
    "id": "uuid",
    "email": "user@company.com",
    "tenant_id": "tenant-uuid",
    "roles": ["staff", "hrm_user"]
  },
  "expires_in": 3600
}
```

## Subscription & Billing Management APIs

### REST Subscription Endpoints

```http
# Subscription Management
GET    /subscriptions
POST   /subscriptions
GET    /subscriptions/{id}
PUT    /subscriptions/{id}
DELETE /subscriptions/{id}
POST   /subscriptions/{id}/upgrade
POST   /subscriptions/{id}/downgrade
POST   /subscriptions/{id}/cancel
POST   /subscriptions/{id}/reactivate

# Plan Management
GET    /subscriptions/plans
GET    /subscriptions/plans/{id}
POST   /subscriptions/plans/compare
GET    /subscriptions/plans/features

# Billing Management
GET    /billing/invoices
GET    /billing/invoices/{id}
POST   /billing/invoices/{id}/pay
GET    /billing/invoices/{id}/pdf
GET    /billing/payment-methods
POST   /billing/payment-methods
PUT    /billing/payment-methods/{id}
DELETE /billing/payment-methods/{id}

# Usage Tracking
POST   /usage/track
GET    /usage/summary
GET    /usage/limits
GET    /usage/analytics

# Feature Access
GET    /features/check/{feature}
GET    /features/available
POST   /features/request-access
```

#### Subscription Creation Example
```http
POST /subscriptions
Content-Type: application/json
Authorization: Bearer {token}

{
  "plan_id": "plan-professional",
  "payment_method": "pm_stripe_token",
  "trial_days": 14,
  "billing_cycle": "monthly"
}

Response:
{
  "id": "sub-uuid",
  "plan": {
    "id": "plan-professional",
    "name": "Professional",
    "price": 99.00,
    "billing_cycle": "monthly"
  },
  "status": "trial",
  "trial_start": "2024-01-15T00:00:00Z",
  "trial_end": "2024-01-29T00:00:00Z",
  "current_period_start": "2024-01-15T00:00:00Z",
  "current_period_end": "2024-02-15T00:00:00Z",
  "features": [
    "crm_advanced",
    "hrm_full",
    "accounting_basic"
  ],
  "usage_limits": {
    "users": 25,
    "api_calls": 10000,
    "storage_gb": 100
  },
  "created_at": "2024-01-15T10:00:00Z"
}
```

#### Usage Tracking Example
```http
POST /usage/track
Content-Type: application/json
Authorization: Bearer {token}

{
  "feature": "api_calls",
  "quantity": 1,
  "metadata": {
    "endpoint": "/crm/contacts",
    "method": "POST",
    "user_id": "user-uuid"
  }
}

Response:
{
  "success": true,
  "current_usage": 8547,
  "limit": 10000,
  "remaining": 1453,
  "overage": 0,
  "reset_date": "2024-02-01T00:00:00Z"
}
```

## Human Resource Management (HRM) APIs

### REST HRM Endpoints

```http
# Employee Management
GET    /hrm/employees
POST   /hrm/employees
GET    /hrm/employees/{id}
PUT    /hrm/employees/{id}
DELETE /hrm/employees/{id}
GET    /hrm/employees/{id}/profile
PUT    /hrm/employees/{id}/profile
GET    /hrm/employees/{id}/documents
POST   /hrm/employees/{id}/documents

# Department Management
GET    /hrm/departments
POST   /hrm/departments
GET    /hrm/departments/{id}
PUT    /hrm/departments/{id}
DELETE /hrm/departments/{id}
GET    /hrm/departments/{id}/employees

# Leave Management
GET    /hrm/leaves
POST   /hrm/leaves
GET    /hrm/leaves/{id}
PUT    /hrm/leaves/{id}
DELETE /hrm/leaves/{id}
POST   /hrm/leaves/{id}/approve
POST   /hrm/leaves/{id}/reject
GET    /hrm/leaves/balance/{employee_id}

# Attendance Management
GET    /hrm/attendance
POST   /hrm/attendance/checkin
POST   /hrm/attendance/checkout
GET    /hrm/attendance/{employee_id}
GET    /hrm/attendance/reports
POST   /hrm/attendance/bulk-import

# Payroll Management
GET    /hrm/payroll
POST   /hrm/payroll/generate
GET    /hrm/payroll/{id}
GET    /hrm/payroll/{id}/payslip
POST   /hrm/payroll/{id}/process
GET    /hrm/payroll/reports
```

#### Employee Creation Example
```http
POST /hrm/employees
Content-Type: application/json
Authorization: Bearer {token}

{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john.doe@company.com",
  "department_id": "dept-uuid",
  "position": "Software Engineer",
  "hire_date": "2024-01-15",
  "salary": 75000,
  "employee_id": "EMP001"
}

Response:
{
  "id": "employee-uuid",
  "employee_id": "EMP001",
  "first_name": "John",
  "last_name": "Doe",
  "email": "john.doe@company.com",
  "department": {
    "id": "dept-uuid",
    "name": "Engineering"
  },
  "position": "Software Engineer",
  "hire_date": "2024-01-15",
  "status": "active",
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Customer Relationship Management (CRM) APIs

### REST CRM Endpoints

```http
# Contact Management
GET    /crm/contacts
POST   /crm/contacts
GET    /crm/contacts/{id}
PUT    /crm/contacts/{id}
DELETE /crm/contacts/{id}
GET    /crm/contacts/{id}/activities
POST   /crm/contacts/{id}/activities
GET    /crm/contacts/search?q={query}

# Lead Management
GET    /crm/leads
POST   /crm/leads
GET    /crm/leads/{id}
PUT    /crm/leads/{id}
DELETE /crm/leads/{id}
POST   /crm/leads/{id}/convert
GET    /crm/leads/pipeline
POST   /crm/leads/{id}/assign

# Opportunity Management
GET    /crm/opportunities
POST   /crm/opportunities
GET    /crm/opportunities/{id}
PUT    /crm/opportunities/{id}
DELETE /crm/opportunities/{id}
POST   /crm/opportunities/{id}/stage
GET    /crm/opportunities/forecast

# Customer Service
GET    /crm/tickets
POST   /crm/tickets
GET    /crm/tickets/{id}
PUT    /crm/tickets/{id}
POST   /crm/tickets/{id}/close
POST   /crm/tickets/{id}/escalate
GET    /crm/tickets/{id}/history
```

#### Lead Creation Example
```http
POST /crm/leads
Content-Type: application/json
Authorization: Bearer {token}

{
  "contact_id": "contact-uuid",
  "title": "Website Redesign Project",
  "source": "website",
  "status": "new",
  "estimated_value": 25000,
  "expected_close_date": "2024-03-15",
  "assigned_to": "user-uuid",
  "notes": "Interested in complete website overhaul"
}

Response:
{
  "id": "lead-uuid",
  "title": "Website Redesign Project",
  "contact": {
    "id": "contact-uuid",
    "name": "ABC Company",
    "email": "contact@abc.com"
  },
  "source": "website",
  "status": "new",
  "estimated_value": 25000,
  "score": 75,
  "assigned_to": {
    "id": "user-uuid",
    "name": "Sales Rep"
  },
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Finance Management APIs

### REST Finance Endpoints

```http
# Account Management
GET    /finance/accounts
POST   /finance/accounts
GET    /finance/accounts/{id}
PUT    /finance/accounts/{id}
DELETE /finance/accounts/{id}
GET    /finance/accounts/chart

# Invoice Management
GET    /finance/invoices
POST   /finance/invoices
GET    /finance/invoices/{id}
PUT    /finance/invoices/{id}
DELETE /finance/invoices/{id}
POST   /finance/invoices/{id}/send
POST   /finance/invoices/{id}/pay
GET    /finance/invoices/{id}/pdf

# Payment Management
GET    /finance/payments
POST   /finance/payments
GET    /finance/payments/{id}
PUT    /finance/payments/{id}
POST   /finance/payments/{id}/process
GET    /finance/payments/methods

# Financial Reports
GET    /finance/reports/income-statement
GET    /finance/reports/balance-sheet
GET    /finance/reports/cash-flow
GET    /finance/reports/trial-balance
GET    /finance/reports/aged-receivables
GET    /finance/reports/aged-payables

# Budget Management
GET    /finance/budgets
POST   /finance/budgets
GET    /finance/budgets/{id}
PUT    /finance/budgets/{id}
GET    /finance/budgets/{id}/variance
POST   /finance/budgets/{id}/approve
```

#### Invoice Creation Example
```http
POST /finance/invoices
Content-Type: application/json
Authorization: Bearer {token}

{
  "customer_id": "customer-uuid",
  "invoice_number": "INV-2024-001",
  "invoice_date": "2024-01-15",
  "due_date": "2024-02-15",
  "currency": "USD",
  "line_items": [
    {
      "description": "Web Development Services",
      "quantity": 40,
      "unit_price": 100,
      "tax_rate": 0.08
    }
  ],
  "notes": "Payment terms: Net 30 days"
}

Response:
{
  "id": "invoice-uuid",
  "invoice_number": "INV-2024-001",
  "customer": {
    "id": "customer-uuid",
    "name": "ABC Company"
  },
  "invoice_date": "2024-01-15",
  "due_date": "2024-02-15",
  "subtotal": 4000.00,
  "tax_amount": 320.00,
  "total_amount": 4320.00,
  "status": "draft",
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Inventory Management APIs

### REST Inventory Endpoints

```http
# Product Management
GET    /inventory/products
POST   /inventory/products
GET    /inventory/products/{id}
PUT    /inventory/products/{id}
DELETE /inventory/products/{id}
GET    /inventory/products/{id}/stock
PUT    /inventory/products/{id}/stock

# Warehouse Management
GET    /inventory/warehouses
POST   /inventory/warehouses
GET    /inventory/warehouses/{id}
PUT    /inventory/warehouses/{id}
GET    /inventory/warehouses/{id}/stock
POST   /inventory/warehouses/{id}/transfer

# Purchase Orders
GET    /inventory/purchase-orders
POST   /inventory/purchase-orders
GET    /inventory/purchase-orders/{id}
PUT    /inventory/purchase-orders/{id}
POST   /inventory/purchase-orders/{id}/approve
POST   /inventory/purchase-orders/{id}/receive

# Sales Orders
GET    /inventory/sales-orders
POST   /inventory/sales-orders
GET    /inventory/sales-orders/{id}
PUT    /inventory/sales-orders/{id}
POST   /inventory/sales-orders/{id}/fulfill
POST   /inventory/sales-orders/{id}/ship

# Stock Management
GET    /inventory/stock
POST   /inventory/stock/adjustment
GET    /inventory/stock/movements
GET    /inventory/stock/valuation
POST   /inventory/stock/cycle-count
```

## Project Management APIs

### REST Project Endpoints

```http
# Project Management
GET    /projects
POST   /projects
GET    /projects/{id}
PUT    /projects/{id}
DELETE /projects/{id}
GET    /projects/{id}/tasks
POST   /projects/{id}/tasks

# Task Management
GET    /projects/tasks
POST   /projects/tasks
GET    /projects/tasks/{id}
PUT    /projects/tasks/{id}
DELETE /projects/tasks/{id}
POST   /projects/tasks/{id}/assign
POST   /projects/tasks/{id}/complete

# Time Tracking
GET    /projects/timesheets
POST   /projects/timesheets
GET    /projects/timesheets/{id}
PUT    /projects/timesheets/{id}
POST   /projects/timesheets/{id}/submit
POST   /projects/timesheets/{id}/approve

# Resource Management
GET    /projects/resources
POST   /projects/resources/allocate
GET    /projects/resources/availability
GET    /projects/resources/utilization
```

## AI and Chatbot APIs

### REST AI Endpoints

```http
# Chat Interface
POST   /ai/chat
GET    /ai/chat/history
DELETE /ai/chat/history/{conversation_id}
POST   /ai/chat/feedback

# AI Actions
POST   /ai/actions/execute
GET    /ai/actions/history
POST   /ai/actions/schedule
GET    /ai/actions/scheduled
DELETE /ai/actions/scheduled/{id}

# Knowledge Base
POST   /ai/knowledge/index
GET    /ai/knowledge/search
PUT    /ai/knowledge/update
DELETE /ai/knowledge/{document_id}

# AI Configuration
GET    /ai/config
PUT    /ai/config
POST   /ai/config/test-connection
GET    /ai/config/models
```

#### Chat Request Example
```http
POST /ai/chat
Content-Type: application/json
Authorization: Bearer {token}

{
  "message": "Show me all overdue invoices for this month",
  "conversation_id": "conv-uuid",
  "context": {
    "current_module": "finance",
    "user_role": "finance_manager"
  }
}

Response:
{
  "response": "I found 5 overdue invoices for January 2024 totaling $12,450. Here are the details:",
  "data": {
    "invoices": [
      {
        "id": "inv-1",
        "customer": "ABC Corp",
        "amount": 2500,
        "due_date": "2024-01-10",
        "days_overdue": 5
      }
    ]
  },
  "actions": [
    {
      "type": "navigate",
      "target": "/finance/invoices?status=overdue"
    }
  ],
  "conversation_id": "conv-uuid"
}
```

## GraphQL Schema and Queries

### Core GraphQL Schema

```graphql
type Query {
  # User and Authentication
  me: User
  users(filter: UserFilter, pagination: Pagination): UserConnection
  
  # HRM Queries
  employees(filter: EmployeeFilter, pagination: Pagination): EmployeeConnection
  employee(id: ID!): Employee
  departments: [Department!]!
  leaves(filter: LeaveFilter): [Leave!]!
  
  # CRM Queries
  contacts(filter: ContactFilter, pagination: Pagination): ContactConnection
  contact(id: ID!): Contact
  leads(filter: LeadFilter): [Lead!]!
  opportunities(filter: OpportunityFilter): [Opportunity!]!
  
  # Finance Queries
  invoices(filter: InvoiceFilter, pagination: Pagination): InvoiceConnection
  invoice(id: ID!): Invoice
  accounts: [Account!]!
  financialReports(type: ReportType!, period: DateRange!): FinancialReport
  
  # Inventory Queries
  products(filter: ProductFilter, pagination: Pagination): ProductConnection
  product(id: ID!): Product
  warehouses: [Warehouse!]!
  stockLevels(warehouseId: ID): [StockLevel!]!
  
  # Project Queries
  projects(filter: ProjectFilter): [Project!]!
  project(id: ID!): Project
  tasks(projectId: ID, assignedTo: ID): [Task!]!
  
  # AI Queries
  chatHistory(conversationId: ID): [ChatMessage!]!
  aiActions(status: ActionStatus): [AIAction!]!
}

type Mutation {
  # Authentication
  login(input: LoginInput!): AuthPayload!
  logout: Boolean!
  refreshToken(token: String!): AuthPayload!
  
  # HRM Mutations
  createEmployee(input: CreateEmployeeInput!): Employee!
  updateEmployee(id: ID!, input: UpdateEmployeeInput!): Employee!
  deleteEmployee(id: ID!): Boolean!
  submitLeave(input: LeaveInput!): Leave!
  approveLeave(id: ID!): Leave!
  
  # CRM Mutations
  createContact(input: CreateContactInput!): Contact!
  updateContact(id: ID!, input: UpdateContactInput!): Contact!
  createLead(input: CreateLeadInput!): Lead!
  convertLead(id: ID!, input: ConvertLeadInput!): Opportunity!
  
  # Finance Mutations
  createInvoice(input: CreateInvoiceInput!): Invoice!
  updateInvoice(id: ID!, input: UpdateInvoiceInput!): Invoice!
  processPayment(input: PaymentInput!): Payment!
  
  # Inventory Mutations
  createProduct(input: CreateProductInput!): Product!
  updateStock(input: StockUpdateInput!): StockLevel!
  createPurchaseOrder(input: PurchaseOrderInput!): PurchaseOrder!
  
  # Project Mutations
  createProject(input: CreateProjectInput!): Project!
  createTask(input: CreateTaskInput!): Task!
  updateTaskStatus(id: ID!, status: TaskStatus!): Task!
  logTime(input: TimeLogInput!): TimeEntry!
  
  # AI Mutations
  sendChatMessage(input: ChatInput!): ChatResponse!
  executeAIAction(input: AIActionInput!): AIActionResult!
  scheduleAIAction(input: ScheduleActionInput!): ScheduledAction!
}

type Subscription {
  # Real-time notifications
  notifications(userId: ID!): Notification!
  chatMessages(conversationId: ID!): ChatMessage!
  taskUpdates(projectId: ID!): Task!
  invoiceUpdates(customerId: ID): Invoice!
}
```

### GraphQL Query Examples

#### Complex Employee Query
```graphql
query GetEmployeeDetails($id: ID!) {
  employee(id: $id) {
    id
    firstName
    lastName
    email
    department {
      id
      name
      manager {
        firstName
        lastName
      }
    }
    position
    hireDate
    salary
    leaves(status: PENDING) {
      id
      type
      startDate
      endDate
      status
    }
    timesheets(month: "2024-01") {
      id
      date
      hoursWorked
      project {
        name
      }
    }
  }
}
```

#### CRM Dashboard Query
```graphql
query CRMDashboard {
  leads(filter: { status: OPEN }) {
    totalCount
    edges {
      node {
        id
        title
        estimatedValue
        contact {
          name
          company
        }
        assignedTo {
          firstName
          lastName
        }
      }
    }
  }
  
  opportunities(filter: { stage: NEGOTIATION }) {
    totalCount
    totalValue
    edges {
      node {
        id
        title
        value
        probability
        expectedCloseDate
      }
    }
  }
  
  recentActivities(limit: 10) {
    id
    type
    description
    createdAt
    user {
      firstName
      lastName
    }
  }
}
```

## API Response Standards

### Success Response Format
```json
{
  "success": true,
  "data": {
    // Response data
  },
  "meta": {
    "pagination": {
      "page": 1,
      "per_page": 20,
      "total": 100,
      "total_pages": 5
    },
    "timestamp": "2024-01-15T10:00:00Z"
  }
}
```

### Error Response Format
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      }
    ]
  },
  "meta": {
    "timestamp": "2024-01-15T10:00:00Z",
    "request_id": "req-uuid"
  }
}
```

## API Security and Rate Limiting

### Authentication Headers
```http
Authorization: Bearer {jwt_token}
X-Tenant-ID: {tenant_id}
X-API-Version: v1
```

### Rate Limiting
- **Authenticated Users**: 1000 requests/hour
- **AI Endpoints**: 100 requests/hour
- **File Upload**: 50 requests/hour
- **Reports**: 20 requests/hour

### API Versioning
- URL versioning: `/api/v1/`, `/api/v2/`
- Header versioning: `X-API-Version: v1`
- Backward compatibility for 2 major versions

This comprehensive API documentation provides a solid foundation for building and integrating with the enterprise ERP system.