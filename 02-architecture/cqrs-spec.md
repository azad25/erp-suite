CQRS Analytics Enhancement Design Document
Overview
This design document outlines the implementation of CQRS (Command Query Responsibility Segregation) pattern as an architectural enhancement to the existing 14-service ERP system. The CQRS implementation will add dedicated read models and query services while maintaining the existing command-side operations in Django business services, enabling complex business intelligence and real-time analytics without impacting transactional performance.

The design integrates with existing architectural patterns from 02-architecture/Architectural_Patterns.md and leverages the current technology stack from tech-stack.md.

Architecture
Current State Analysis
The existing system uses traditional CRUD operations in Django business services:

Command Side: Django services (HRM, CRM, Finance, Inventory, Project) handle both reads and writes
Database: PostgreSQL per service with direct query access
Performance Issue: Complex analytics queries impact transactional performance
Scalability Limitation: Read and write operations compete for the same resources
CQRS Architecture Design
API Gateway (Enhanced)

Frontend (Enhanced)

Query Side (New CQRS Implementation)

Event Bridge (Existing)

Command Side (Existing)

CRM Service
Django + DRF

HRM Service
Django + DRF

Finance Service
Django + DRF

Inventory Service
Django + DRF

Project Service
Django + DRF

CRM PostgreSQL

HRM PostgreSQL

Finance PostgreSQL

Inventory PostgreSQL

Project PostgreSQL

Kafka Event Bus

Redis Pub/Sub

Analytics Query Service
FastAPI

Reports Query Service
Go

Dashboard Query Service
FastAPI

MongoDB
Read Models

Redis
Aggregated Cache

Elasticsearch
Search Index

Next.js Frontend

Analytics Dashboard

Reports Interface

API Gateway
Go + Chi

Multi-Tenant CQRS Architecture
Read Model Storage

CQRS Event Processing

Tenant B (org_id: beta-inc)

Tenant A (org_id: acme-corp)

Command Operations

Read Models A

Events: acme-corp.*

Command Operations

Read Models B

Events: beta-inc.*

Event Processor
Tenant-Aware

Tenant Filter
organization_id

MongoDB Collection
acme_corp_analytics

MongoDB Collection
beta_inc_analytics

Redis Namespace
acme-corp:*

Redis Namespace
beta-inc:*

Components and Interfaces
1. Analytics Query Service (FastAPI)
Purpose: High-performance analytics queries with real-time data aggregation

Technology Stack:

Framework: FastAPI with async support
Database: MongoDB for flexible document storage
Cache: Redis for real-time aggregations
Authentication: JWT integration with existing auth service
Service Structure:

# erp-analytics-query-service/
├── src/
│   ├── services/
│   │   ├── analytics_service.py      # Core analytics logic
│   │   ├── aggregation_service.py    # Data aggregation
│   │   ├── cache_service.py          # Redis caching
│   │   └── event_consumer.py         # Kafka event processing
│   ├── models/
│   │   ├── read_models.py           # Pydantic read models
│   │   ├── aggregations.py          # Aggregation schemas
│   │   └── events.py                # Event schemas
│   ├── api/
│   │   ├── analytics.py             # Analytics endpoints
│   │   ├── dashboards.py            # Dashboard endpoints
│   │   └── real_time.py             # WebSocket endpoints
│   └── consumers/
│       ├── crm_consumer.py          # CRM event consumer
│       ├── hrm_consumer.py          # HRM event consumer
│       ├── finance_consumer.py      # Finance event consumer
│       ├── inventory_consumer.py    # Inventory event consumer
│       └── project_consumer.py      # Project event consumer
├── docker/
│   ├── Dockerfile.dev              # Development container
│   └── Dockerfile.prod             # Production container
├── requirements.txt
└── README.md
Key APIs:

# Analytics endpoints
GET /analytics/dashboard/{organization_id}
GET /analytics/crm/pipeline/{organization_id}
GET /analytics/hrm/headcount/{organization_id}
GET /analytics/finance/revenue/{organization_id}
GET /analytics/inventory/turnover/{organization_id}
GET /analytics/projects/utilization/{organization_id}

# Real-time endpoints
WebSocket /analytics/real-time/{organization_id}
GET /analytics/metrics/live/{organization_id}
2. Reports Query Service (Go)
Purpose: Complex report generation with optimized performance

Technology Stack:

Language: Go 1.21+ with Chi router
Database: MongoDB for report data
PDF Generation: Go PDF libraries
Export Formats: PDF, Excel, CSV
Service Structure:

// erp-reports-query-service/
├── cmd/
│   └── main.go                     // Service entry point
├── internal/
│   ├── handlers/
│   │   ├── reports_handler.go      // Report endpoints
│   │   ├── export_handler.go       // Export endpoints
│   │   └── schedule_handler.go     // Scheduled reports
│   ├── services/
│   │   ├── report_service.go       // Core report logic
│   │   ├── export_service.go       // Export functionality
│   │   ├── template_service.go     // Report templates
│   │   └── scheduler_service.go    // Report scheduling
│   ├── models/
│   │   ├── report.go              // Report models
│   │   └── template.go            // Template models
│   ├── repository/
│   │   ├── mongodb.go             // MongoDB operations
│   │   └── redis.go               // Redis caching
│   └── consumers/
│       └── event_consumer.go       // Kafka event processing
├── docker/
│   ├── Dockerfile.dev
│   └── Dockerfile.prod
├── go.mod
└── README.md
Key APIs:

// Report endpoints
GET /reports/financial/{organization_id}
GET /reports/sales/{organization_id}
GET /reports/hr/{organization_id}
GET /reports/inventory/{organization_id}
GET /reports/projects/{organization_id}

// Export endpoints
POST /reports/export/pdf
POST /reports/export/excel
POST /reports/export/csv

// Scheduled reports
POST /reports/schedule
GET /reports/scheduled/{organization_id}
DELETE /reports/scheduled/{id}
3. Dashboard Query Service (FastAPI)
Purpose: Real-time dashboard data with WebSocket support

Technology Stack:

Framework: FastAPI with WebSocket support
Database: Redis for real-time data
Search: Elasticsearch for full-text search
Real-time: WebSocket connections
Service Structure:

# erp-dashboard-query-service/
├── src/
│   ├── services/
│   │   ├── dashboard_service.py     # Dashboard logic
│   │   ├── widget_service.py        # Widget management
│   │   ├── search_service.py        # Search functionality
│   │   └── websocket_service.py     # Real-time updates
│   ├── models/
│   │   ├── dashboard.py            # Dashboard models
│   │   ├── widget.py               # Widget models
│   │   └── search.py               # Search models
│   ├── api/
│   │   ├── dashboards.py           # Dashboard endpoints
│   │   ├── widgets.py              # Widget endpoints
│   │   ├── search.py               # Search endpoints
│   │   └── websocket.py            # WebSocket endpoints
│   └── consumers/
│       └── real_time_consumer.py    # Real-time event processing
Key APIs:

# Dashboard endpoints
GET /dashboards/{organization_id}
POST /dashboards/{organization_id}/widgets
PUT /dashboards/{organization_id}/layout

# Search endpoints
GET /search/{organization_id}?q={query}
GET /search/suggestions/{organization_id}

# Real-time endpoints
WebSocket /dashboards/live/{organization_id}
4. Event Processing Infrastructure
Event Schema Standardization:

{
  "event_type": "crm.contact.created",
  "organization_id": "acme-corp",
  "timestamp": "2024-01-15T10:00:00Z",
  "version": "1.0",
  "source_service": "erp-crm-service",
  "data": {
    "entity_id": "contact-uuid",
    "entity_type": "contact",
    "action": "created",
    "payload": {
      "name": "John Doe",
      "email": "john@example.com",
      "company": "ABC Corp"
    },
    "metadata": {
      "user_id": "user-uuid",
      "ip_address": "192.168.1.1",
      "user_agent": "Mozilla/5.0..."
    }
  }
}
Event Topics:

# CRM Events
crm.contact.created
crm.contact.updated
crm.contact.deleted
crm.lead.created
crm.lead.converted
crm.opportunity.created
crm.opportunity.won
crm.opportunity.lost

# HRM Events
hrm.employee.created
hrm.employee.updated
hrm.employee.terminated
hrm.leave.requested
hrm.leave.approved
hrm.attendance.checked_in
hrm.payroll.processed

# Finance Events
finance.invoice.created
finance.invoice.sent
finance.payment.received
finance.account.created
finance.transaction.posted

# Inventory Events
inventory.product.created
inventory.stock.updated
inventory.order.created
inventory.order.fulfilled
inventory.reorder.triggered

# Project Events
project.created
project.updated
project.task.created
project.task.completed
project.time.logged
Data Models
Read Model Schemas
Analytics Read Model (MongoDB):

from pydantic import BaseModel
from datetime import datetime
from typing import Dict, List, Optional

class CRMAnalyticsReadModel(BaseModel):
    organization_id: str
    period: str  # "2024-01", "2024-Q1", "2024"
    
    # Lead metrics
    leads_total: int
    leads_new: int
    leads_converted: int
    conversion_rate: float
    
    # Pipeline metrics
    pipeline_value: float
    pipeline_stages: Dict[str, int]
    average_deal_size: float
    
    # Activity metrics
    activities_total: int
    activities_by_type: Dict[str, int]
    
    # Performance metrics
    top_performers: List[Dict[str, any]]
    
    # Metadata
    last_updated: datetime
    data_sources: List[str]

class HRMAnalyticsReadModel(BaseModel):
    organization_id: str
    period: str
    
    # Headcount metrics
    total_employees: int
    new_hires: int
    terminations: int
    turnover_rate: float
    
    # Department breakdown
    employees_by_department: Dict[str, int]
    employees_by_role: Dict[str, int]
    
    # Leave metrics
    leave_requests: int
    leave_approved: int
    leave_pending: int
    
    # Attendance metrics
    average_attendance: float
    late_arrivals: int
    
    # Metadata
    last_updated: datetime
    data_sources: List[str]

class FinanceAnalyticsReadModel(BaseModel):
    organization_id: str
    period: str
    
    # Revenue metrics
    total_revenue: float
    recurring_revenue: float
    one_time_revenue: float
    revenue_growth: float
    
    # Invoice metrics
    invoices_sent: int
    invoices_paid: int
    invoices_overdue: int
    average_payment_time: float
    
    # Cash flow
    cash_inflow: float
    cash_outflow: float
    net_cash_flow: float
    
    # Profitability
    gross_profit: float
    net_profit: float
    profit_margin: float
    
    # Metadata
    last_updated: datetime
    data_sources: List[str]
Dashboard Widget Model (Redis):

class DashboardWidget(BaseModel):
    widget_id: str
    organization_id: str
    widget_type: str  # "chart", "metric", "table", "list"
    title: str
    
    # Data configuration
    data_source: str  # "crm", "hrm", "finance", etc.
    query_params: Dict[str, any]
    refresh_interval: int  # seconds
    
    # Display configuration
    chart_type: Optional[str]  # "line", "bar", "pie", "donut"
    position: Dict[str, int]  # x, y, width, height
    
    # Cached data
    cached_data: Optional[Dict[str, any]]
    cache_expires: Optional[datetime]
    
    # Metadata
    created_at: datetime
    updated_at: datetime
Event Sourcing Models
Event Store Schema (MongoDB):

class EventStore(BaseModel):
    event_id: str
    organization_id: str
    event_type: str
    source_service: str
    
    # Event data
    aggregate_id: str
    aggregate_type: str
    event_version: int
    event_data: Dict[str, any]
    
    # Metadata
    timestamp: datetime
    user_id: Optional[str]
    correlation_id: Optional[str]
    causation_id: Optional[str]
    
    # Processing status
    processed: bool = False
    processed_at: Optional[datetime]
    processing_errors: List[str] = []
Error Handling
Circuit Breaker Implementation
class CQRSCircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    async def call_read_service(self, service_call):
        if self.state == 'OPEN':
            if self._should_attempt_reset():
                self.state = 'HALF_OPEN'
            else:
                # Fallback to cached data or direct database query
                return await self._fallback_query()
        
        try:
            result = await service_call()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            return await self._fallback_query()
    
    async def _fallback_query(self):
        # Fallback to cached data in Redis
        # Or direct query to source database
        # Or return stale data with warning
        pass
Data Consistency Handling
class ReadModelConsistencyChecker:
    async def verify_data_consistency(self, organization_id: str):
        """Verify read model data against source databases"""
        
        inconsistencies = []
        
        # Check CRM data consistency
        crm_source = await self.get_crm_source_data(organization_id)
        crm_read_model = await self.get_crm_read_model(organization_id)
        
        if not self._compare_data(crm_source, crm_read_model):
            inconsistencies.append({
                'service': 'crm',
                'type': 'data_mismatch',
                'details': self._get_diff(crm_source, crm_read_model)
            })
        
        # Similar checks for other services...
        
        if inconsistencies:
            await self._trigger_reconciliation(inconsistencies)
            await self._alert_administrators(inconsistencies)
        
        return inconsistencies
    
    async def _trigger_reconciliation(self, inconsistencies):
        """Trigger read model rebuild for inconsistent data"""
        for issue in inconsistencies:
            await self.rebuild_read_model(
                service=issue['service'],
                organization_id=issue['organization_id']
            )
Testing Strategy
Unit Testing
Analytics Service Tests:

# tests/test_analytics_service.py
import pytest
from unittest.mock import AsyncMock, patch
from src.services.analytics_service import AnalyticsService

class TestAnalyticsService:
    @pytest.fixture
    def analytics_service(self):
        return AnalyticsService()
    
    @pytest.mark.asyncio
    async def test_get_crm_analytics(self, analytics_service):
        # Mock read model data
        mock_data = {
            'leads_total': 100,
            'conversion_rate': 0.25,
            'pipeline_value': 50000
        }
        
        with patch.object(analytics_service, 'get_read_model') as mock_get:
            mock_get.return_value = mock_data
            
            result = await analytics_service.get_crm_analytics('acme-corp')
            
            assert result['leads_total'] == 100
            assert result['conversion_rate'] == 0.25
            mock_get.assert_called_once_with('crm_analytics', 'acme-corp')
    
    @pytest.mark.asyncio
    async def test_event_processing(self, analytics_service):
        # Test event consumption and read model updates
        event = {
            'event_type': 'crm.lead.created',
            'organization_id': 'acme-corp',
            'data': {'lead_id': 'lead-123', 'value': 5000}
        }
        
        await analytics_service.process_event(event)
        
        # Verify read model was updated
        updated_model = await analytics_service.get_read_model('crm_analytics', 'acme-corp')
        assert updated_model['leads_total'] == 101
Integration Testing
End-to-End CQRS Flow Tests:

# tests/test_cqrs_integration.py
import pytest
import asyncio
from tests.fixtures import create_test_organization, create_test_user

class TestCQRSIntegration:
    @pytest.mark.asyncio
    async def test_command_to_query_flow(self):
        # Setup test data
        org = await create_test_organization('test-corp')
        user = await create_test_user(org.id)
        
        # Execute command in CRM service
        contact_data = {
            'name': 'John Doe',
            'email': 'john@test.com',
            'organization_id': org.id
        }
        
        # Create contact via command side
        response = await self.crm_client.post('/contacts', json=contact_data)
        assert response.status_code == 201
        contact_id = response.json()['id']
        
        # Wait for event processing
        await asyncio.sleep(2)
        
        # Verify read model was updated
        analytics_response = await self.analytics_client.get(f'/analytics/crm/{org.id}')
        assert analytics_response.status_code == 200
        
        analytics_data = analytics_response.json()
        assert analytics_data['contacts_total'] > 0
        
        # Verify dashboard was updated
        dashboard_response = await self.dashboard_client.get(f'/dashboards/{org.id}')
        dashboard_data = dashboard_response.json()
        
        # Find CRM widget and verify it shows updated data
        crm_widget = next(w for w in dashboard_data['widgets'] if w['data_source'] == 'crm')
        assert crm_widget['cached_data']['contacts_total'] > 0
Performance Testing
Load Testing for Read Models:

# tests/test_performance.py
import pytest
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

class TestCQRSPerformance:
    @pytest.mark.asyncio
    async def test_concurrent_analytics_queries(self):
        """Test analytics service under concurrent load"""
        
        async def make_analytics_request():
            start_time = time.time()
            response = await self.analytics_client.get('/analytics/dashboard/acme-corp')
            end_time = time.time()
            
            assert response.status_code == 200
            assert (end_time - start_time) < 2.0  # Must respond within 2 seconds
            
            return end_time - start_time
        
        # Execute 100 concurrent requests
        tasks = [make_analytics_request() for _ in range(100)]
        response_times = await asyncio.gather(*tasks)
        
        # Verify performance requirements
        avg_response_time = sum(response_times) / len(response_times)
        assert avg_response_time < 1.0  # Average under 1 second
        assert max(response_times) < 3.0  # No request over 3 seconds
    
    @pytest.mark.asyncio
    async def test_read_model_update_performance(self):
        """Test read model update speed"""
        
        # Create test event
        event = {
            'event_type': 'crm.contact.created',
            'organization_id': 'acme-corp',
            'data': {'contact_id': 'contact-123'}
        }
        
        # Measure processing time
        start_time = time.time()
        await self.analytics_service.process_event(event)
        end_time = time.time()
        
        processing_time = end_time - start_time
        assert processing_time < 0.5  # Must process within 500ms
This comprehensive design provides a robust CQRS implementation that enhances the existing 14-service ERP system with high-performance analytics and reporting capabilities while maintaining data consistency and system reliability.

# Implementation Plan

## Overview

This implementation plan converts the CQRS analytics enhancement design into a series of incremental coding tasks that build upon the existing 14-service ERP architecture. Each task is designed to be executed by a coding agent in a test-driven manner, ensuring no big jumps in complexity and proper integration with existing services.

The plan assumes access to the existing architecture documents (requirements.md, design.md) and follows the established patterns from `02-architecture/Architectural_Patterns.md`.

## Implementation Tasks

- [ ] 1. Set up CQRS infrastructure foundation
  - Create shared event schemas and utilities for CQRS implementation
  - Set up MongoDB and Redis infrastructure for read models
  - Implement tenant-aware event processing utilities
  - _Requirements: 6.1, 6.3, 8.1_

- [ ] 1.1 Create shared CQRS utilities and event schemas
  - Write Python package for common CQRS patterns and event schemas
  - Implement tenant-aware event processing base classes
  - Create event validation and serialization utilities
  - Write unit tests for shared utilities
  - _Requirements: 6.1, 6.3_

- [ ] 1.2 Set up read model database infrastructure
  - Configure MongoDB for read model storage with tenant isolation
  - Set up Redis for real-time aggregations and caching
  - Implement database connection utilities with proper error handling
  - Create database migration and seeding scripts for development
  - _Requirements: 6.3, 8.1_

- [ ] 1.3 Implement event processing base infrastructure
  - Create Kafka consumer base classes for read model updates
  - Implement event routing and filtering by organization_id
  - Add error handling and retry logic for event processing
  - Write integration tests for event processing pipeline
  - _Requirements: 6.1, 6.2, 8.4_

- [ ] 2. Enhance existing Django services with CQRS event publishing
  - Add event publishing to all 5 Django business services
  - Implement consistent event schemas across services
  - Ensure proper tenant isolation in published events
  - _Requirements: 6.2, 7.4, 8.1_

- [ ] 2.1 Add event publishing to CRM Service
  - Implement event publishers for contact, lead, and opportunity operations
  - Add Django signals to trigger event publishing on model changes
  - Ensure events include proper organization_id and user context
  - Write unit tests for event publishing functionality
  - _Requirements: 3.2, 6.2, 7.1_

- [ ] 2.2 Add event publishing to HRM Service
  - Implement event publishers for employee, leave, and attendance operations
  - Add event publishing for payroll and department changes
  - Ensure RBAC permissions are respected in event data
  - Write unit tests for HRM event publishing
  - _Requirements: 6.2, 7.1, 8.1_

- [ ] 2.3 Add event publishing to Finance Service
  - Implement event publishers for invoice, payment, and account operations
  - Add event publishing for financial transactions and reports
  - Ensure sensitive financial data is properly handled in events
  - Write unit tests for Finance event publishing
  - _Requirements: 2.2, 6.2, 7.1_

- [ ] 2.4 Add event publishing to Inventory Service
  - Implement event publishers for product, stock, and order operations
  - Add event publishing for warehouse and procurement changes
  - Ensure inventory events include proper stock level data
  - Write unit tests for Inventory event publishing
  - _Requirements: 4.2, 6.2, 7.1_

- [ ] 2.5 Add event publishing to Project Service
  - Implement event publishers for project, task, and time tracking operations
  - Add event publishing for resource allocation changes
  - Ensure project events include proper team and resource context
  - Write unit tests for Project event publishing
  - _Requirements: 6.2, 7.1, 8.1_

- [ ] 3. Implement Analytics Query Service (FastAPI)
  - Create high-performance analytics service with real-time aggregations
  - Implement read model consumers for all business services
  - Add WebSocket support for real-time dashboard updates
  - _Requirements: 1.1, 1.2, 1.3, 3.1, 3.2_

- [ ] 3.1 Create Analytics Service foundation
  - Set up FastAPI service structure with async support
  - Implement MongoDB read model repositories
  - Add Redis caching layer for real-time aggregations
  - Create service configuration and dependency injection
  - _Requirements: 1.1, 1.2, 6.3_

- [ ] 3.2 Implement CRM analytics read models and consumers
  - Create CRM analytics read model schemas and MongoDB collections
  - Implement Kafka consumers for CRM events (contacts, leads, opportunities)
  - Add real-time aggregation logic for CRM metrics
  - Write unit tests for CRM analytics processing
  - _Requirements: 3.1, 3.2, 3.3, 7.1_

- [ ] 3.3 Implement HRM analytics read models and consumers
  - Create HRM analytics read model schemas for employee metrics
  - Implement Kafka consumers for HRM events (employees, leave, attendance)
  - Add headcount and performance aggregation logic
  - Write unit tests for HRM analytics processing
  - _Requirements: 7.1, 8.1_

- [ ] 3.4 Implement Finance analytics read models and consumers
  - Create Finance analytics read model schemas for revenue metrics
  - Implement Kafka consumers for Finance events (invoices, payments)
  - Add financial aggregation logic with proper accounting rules
  - Write unit tests for Finance analytics processing
  - _Requirements: 2.1, 2.2, 2.3, 7.1_

- [ ] 3.5 Implement Inventory analytics read models and consumers
  - Create Inventory analytics read model schemas for stock metrics
  - Implement Kafka consumers for Inventory events (products, stock, orders)
  - Add inventory turnover and stock level aggregation logic
  - Write unit tests for Inventory analytics processing
  - _Requirements: 4.1, 4.2, 4.3, 7.1_

- [ ] 3.6 Implement Project analytics read models and consumers
  - Create Project analytics read model schemas for utilization metrics
  - Implement Kafka consumers for Project events (projects, tasks, time)
  - Add resource utilization and project performance aggregation logic
  - Write unit tests for Project analytics processing
  - _Requirements: 7.1, 8.1_

- [ ] 3.7 Add cross-service analytics aggregation
  - Implement cross-service analytics that combine data from multiple services
  - Create unified business metrics that span CRM, Finance, and Project data
  - Add data consistency validation across service boundaries
  - Write integration tests for cross-service analytics
  - _Requirements: 7.1, 7.2, 7.3, 7.4_

- [ ] 3.8 Implement Analytics API endpoints
  - Create REST API endpoints for all analytics queries
  - Add proper RBAC integration with existing auth patterns
  - Implement query parameter validation and filtering
  - Add API documentation and response schemas
  - _Requirements: 1.1, 1.2, 5.1, 6.4_

- [ ] 3.9 Add WebSocket support for real-time analytics
  - Implement WebSocket endpoints for real-time dashboard updates
  - Add real-time event broadcasting to connected clients
  - Ensure proper tenant isolation in WebSocket connections
  - Write integration tests for real-time functionality
  - _Requirements: 1.3, 3.2, 6.4_

- [ ] 4. Implement Reports Query Service (Go)
  - Create high-performance report generation service
  - Implement complex report templates and export functionality
  - Add scheduled report processing capabilities
  - _Requirements: 2.1, 2.3, 5.1, 5.2_

- [ ] 4.1 Create Reports Service foundation
  - Set up Go service structure with Chi router
  - Implement MongoDB read model repositories in Go
  - Add Redis caching for report data
  - Create service configuration and middleware
  - _Requirements: 2.1, 5.1, 6.3_

- [ ] 4.2 Implement report template engine
  - Create flexible report template system for different report types
  - Implement template rendering with dynamic data binding
  - Add support for charts, tables, and formatted layouts
  - Write unit tests for template engine functionality
  - _Requirements: 2.1, 2.3, 5.1_

- [ ] 4.3 Add financial report generation
  - Implement financial report templates (P&L, Balance Sheet, Cash Flow)
  - Add financial data aggregation logic with proper accounting rules
  - Ensure reports respect RBAC permissions and tenant isolation
  - Write unit tests for financial report generation
  - _Requirements: 2.1, 2.2, 2.3, 8.3_

- [ ] 4.4 Add CRM and sales report generation
  - Implement CRM report templates (pipeline, performance, activity)
  - Add sales data aggregation and trend analysis
  - Include lead conversion and opportunity tracking reports
  - Write unit tests for CRM report generation
  - _Requirements: 3.4, 8.3_

- [ ] 4.5 Add HR and operational report generation
  - Implement HR report templates (headcount, attendance, performance)
  - Add inventory and project reporting capabilities
  - Include cross-functional operational reports
  - Write unit tests for operational report generation
  - _Requirements: 7.1, 8.3_

- [ ] 4.6 Implement export functionality
  - Add PDF export using Go PDF libraries
  - Implement Excel export with proper formatting
  - Add CSV export for data analysis
  - Write unit tests for all export formats
  - _Requirements: 2.3, 3.4, 5.1_

- [ ] 4.7 Add scheduled report processing
  - Implement report scheduling system with cron-like functionality
  - Add email delivery for scheduled reports
  - Include report history and management features
  - Write integration tests for scheduled reporting
  - _Requirements: 5.1, 5.2_

- [ ] 4.8 Implement Reports API endpoints
  - Create REST API endpoints for report generation and management
  - Add proper RBAC integration and tenant isolation
  - Implement request validation and error handling
  - Add API documentation and response schemas
  - _Requirements: 5.1, 5.2, 6.4_

- [ ] 5. Implement Dashboard Query Service (FastAPI)
  - Create real-time dashboard service with WebSocket support
  - Implement customizable dashboard widgets and layouts
  - Add search functionality with Elasticsearch integration
  - _Requirements: 1.3, 5.1, 6.4_

- [ ] 5.1 Create Dashboard Service foundation
  - Set up FastAPI service structure with WebSocket support
  - Implement Redis-based dashboard data storage
  - Add Elasticsearch integration for search functionality
  - Create service configuration and authentication middleware
  - _Requirements: 1.3, 5.1, 6.3_

- [ ] 5.2 Implement dashboard widget system
  - Create flexible widget system supporting charts, metrics, and tables
  - Implement widget configuration and layout management
  - Add widget data binding to analytics and report services
  - Write unit tests for widget functionality
  - _Requirements: 1.1, 1.3, 5.1_

- [ ] 5.3 Add real-time dashboard updates
  - Implement WebSocket connections for live dashboard updates
  - Add real-time data streaming from analytics service
  - Ensure proper tenant isolation in WebSocket connections
  - Write integration tests for real-time functionality
  - _Requirements: 1.3, 3.2, 6.4_

- [ ] 5.4 Implement search functionality
  - Create Elasticsearch integration for cross-service search
  - Implement search indexing for all business entities
  - Add search suggestions and autocomplete features
  - Write unit tests for search functionality
  - _Requirements: 6.4, 8.1_

- [ ] 5.5 Add dashboard customization features
  - Implement user-specific dashboard layouts and preferences
  - Add drag-and-drop widget arrangement functionality
  - Include dashboard sharing and template features
  - Write unit tests for customization features
  - _Requirements: 1.3, 5.1, 6.4_

- [ ] 5.6 Implement Dashboard API endpoints
  - Create REST API endpoints for dashboard management
  - Add WebSocket endpoints for real-time updates
  - Implement proper RBAC integration and tenant isolation
  - Add API documentation and response schemas
  - _Requirements: 5.1, 6.4_

- [ ] 6. Integrate CQRS services with API Gateway
  - Update API Gateway to route requests to new CQRS services
  - Implement proper load balancing and circuit breaker patterns
  - Add monitoring and observability for CQRS endpoints
  - _Requirements: 5.1, 5.4, 6.4_

- [ ] 6.1 Update API Gateway routing
  - Add routing rules for Analytics, Reports, and Dashboard services
  - Implement proper request forwarding and response handling
  - Add service discovery integration for CQRS services
  - Write integration tests for gateway routing
  - _Requirements: 5.1, 6.4_

- [ ] 6.2 Implement circuit breaker patterns
  - Add circuit breaker middleware for CQRS service calls
  - Implement fallback mechanisms for service failures
  - Add health check endpoints for all CQRS services
  - Write unit tests for circuit breaker functionality
  - _Requirements: 5.4, 6.4_

- [ ] 6.3 Add CQRS-specific middleware
  - Implement caching middleware for read-heavy operations
  - Add rate limiting specific to analytics and reporting endpoints
  - Include request/response logging for CQRS operations
  - Write unit tests for middleware functionality
  - _Requirements: 1.2, 5.4, 6.4_

- [ ] 7. Enhance Frontend with CQRS integration
  - Update Next.js frontend to consume CQRS services
  - Implement real-time dashboard components
  - Add analytics and reporting interfaces
  - _Requirements: 1.1, 1.3, 3.1, 3.4_

- [ ] 7.1 Create analytics dashboard components
  - Implement React components for analytics visualization
  - Add chart libraries and data visualization components
  - Create responsive dashboard layouts for different screen sizes
  - Write unit tests for dashboard components
  - _Requirements: 1.1, 1.2, 3.1_

- [ ] 7.2 Add real-time WebSocket integration
  - Implement WebSocket client for real-time dashboard updates
  - Add connection management and reconnection logic
  - Create real-time data binding for dashboard widgets
  - Write integration tests for WebSocket functionality
  - _Requirements: 1.3, 3.2, 5.1_

- [ ] 7.3 Implement report generation interface
  - Create React components for report configuration and generation
  - Add report preview and export functionality
  - Implement scheduled report management interface
  - Write unit tests for report components
  - _Requirements: 2.3, 3.4, 5.1_

- [ ] 7.4 Add search and filtering interfaces
  - Implement advanced search components with Elasticsearch integration
  - Add filtering and sorting capabilities for analytics data
  - Create search suggestions and autocomplete components
  - Write unit tests for search functionality
  - _Requirements: 6.4, 8.1_

- [ ] 7.5 Integrate with existing RBAC patterns
  - Ensure all CQRS components respect existing permission patterns
  - Add permission-based component rendering
  - Implement tenant-aware data filtering in frontend
  - Write integration tests for RBAC integration
  - _Requirements: 5.1, 6.4, 8.3_

- [ ] 8. Implement monitoring and observability
  - Add comprehensive monitoring for CQRS services
  - Implement performance metrics and alerting
  - Create operational dashboards for system health
  - _Requirements: 5.4, 6.4, 8.2_

- [ ] 8.1 Add service monitoring and metrics
  - Implement Prometheus metrics collection for all CQRS services
  - Add custom metrics for read model update latency and query performance
  - Create Grafana dashboards for CQRS service monitoring
  - Write monitoring configuration and alerting rules
  - _Requirements: 5.4, 6.4_

- [ ] 8.2 Implement distributed tracing
  - Add Jaeger tracing integration for CQRS request flows
  - Implement trace correlation across command and query sides
  - Add performance profiling for complex analytics queries
  - Write tracing configuration and analysis tools
  - _Requirements: 5.4, 8.2_

- [ ] 8.3 Add data consistency monitoring
  - Implement automated data consistency checks between command and query sides
  - Add alerting for read model synchronization failures
  - Create reconciliation tools for data inconsistencies
  - Write monitoring scripts and automated recovery procedures
  - _Requirements: 8.1, 8.2, 8.4_

- [ ] 9. Performance optimization and testing
  - Implement comprehensive performance testing for CQRS services
  - Add load testing for analytics and reporting endpoints
  - Optimize read model queries and caching strategies
  - _Requirements: 1.1, 1.2, 1.3, 2.1_

- [ ] 9.1 Implement performance testing suite
  - Create load testing scripts for analytics service endpoints
  - Add stress testing for real-time WebSocket connections
  - Implement performance benchmarking for report generation
  - Write automated performance regression testing
  - _Requirements: 1.1, 1.2, 2.1_

- [ ] 9.2 Optimize read model queries and indexing
  - Analyze and optimize MongoDB queries for read models
  - Add proper indexing strategies for analytics aggregations
  - Implement query result caching with Redis
  - Write performance monitoring for query optimization
  - _Requirements: 1.1, 1.2, 4.3_

- [ ] 9.3 Add caching optimization
  - Implement multi-level caching strategy for analytics data
  - Add cache invalidation logic for real-time updates
  - Optimize Redis usage for dashboard and widget data
  - Write cache performance monitoring and tuning
  - _Requirements: 1.2, 1.3, 4.3_

- [ ] 10. Documentation and deployment
  - Create comprehensive documentation for CQRS implementation
  - Add deployment configurations for all new services
  - Implement CI/CD pipelines for CQRS services
  - _Requirements: 5.1, 6.4_

- [ ] 10.1 Create CQRS documentation
  - Write developer documentation for CQRS architecture and patterns
  - Add API documentation for all CQRS service endpoints
  - Create operational runbooks for CQRS service management
  - Write user documentation for analytics and reporting features
  - _Requirements: 5.1, 6.4_

- [ ] 10.2 Implement deployment configurations
  - Create Docker configurations for all CQRS services
  - Add Kubernetes deployment manifests and Helm charts
  - Implement environment-specific configuration management
  - Write deployment automation scripts
  - _Requirements: 6.4_

- [ ] 10.3 Set up CI/CD pipelines
  - Create GitHub Actions workflows for CQRS service testing and deployment
  - Add automated testing integration for all CQRS components
  - Implement deployment pipelines with proper staging and production promotion
  - Write pipeline monitoring and failure recovery procedures
  - _Requirements: 6.4_

## Implementation Notes

### Development Sequence
1. **Foundation (Tasks 1-2)**: Set up infrastructure and enhance existing services
2. **Core Services (Tasks 3-5)**: Implement the three main CQRS query services
3. **Integration (Tasks 6-7)**: Connect services with API Gateway and Frontend
4. **Operations (Tasks 8-10)**: Add monitoring, optimization, and deployment

### Testing Strategy
- Each task includes unit tests for individual components
- Integration tests verify service-to-service communication
- Performance tests ensure sub-2-second response times
- End-to-end tests validate complete CQRS workflows

### Rollback Strategy
- CQRS services are additive and don't modify existing command operations
- Read models can be rebuilt from event store if needed
- Circuit breakers provide fallback to direct database queries
- Feature flags allow gradual rollout and quick rollback

This implementation plan provides a systematic approach to adding CQRS capabilities to the existing ERP system while maintaining system stability and performance.

