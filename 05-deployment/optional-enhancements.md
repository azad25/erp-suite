# Optional Enhancements and Advanced Features

## Overview

This document outlines optional enhancements for the enterprise ERP system, including observability, logging, metrics, tracing, CI/CD pipelines, and environment management strategies.

## Observability and Monitoring

### 1. Comprehensive Logging Strategy

#### Structured Logging Implementation

```python
import structlog
import logging
from typing import Dict, Any
from datetime import datetime
import json

class StructuredLogger:
    """Centralized structured logging for the ERP system"""
    
    def __init__(self, service_name: str, tenant_id: str = None):
        self.service_name = service_name
        self.tenant_id = tenant_id
        
        # Configure structlog
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.stdlib.PositionalArgumentsFormatter(),
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.processors.StackInfoRenderer(),
                structlog.processors.format_exc_info,
                structlog.processors.UnicodeDecoder(),
                structlog.processors.JSONRenderer()
            ],
            context_class=dict,
            logger_factory=structlog.stdlib.LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )
        
        self.logger = structlog.get_logger(service_name)
    
    def log_business_event(self, event_type: str, entity_type: str, entity_id: str, 
                          user_id: str, action: str, details: Dict[str, Any] = None):
        """Log business events for audit and analytics"""
        
        self.logger.info(
            "business_event",
            event_type=event_type,
            entity_type=entity_type,
            entity_id=entity_id,
            user_id=user_id,
            tenant_id=self.tenant_id,
            action=action,
            details=details or {},
            timestamp=datetime.utcnow().isoformat()
        )
    
    def log_api_request(self, method: str, endpoint: str, user_id: str, 
                       response_time: float, status_code: int, request_size: int = None):
        """Log API requests for performance monitoring"""
        
        self.logger.info(
            "api_request",
            method=method,
            endpoint=endpoint,
            user_id=user_id,
            tenant_id=self.tenant_id,
            response_time_ms=response_time * 1000,
            status_code=status_code,
            request_size_bytes=request_size,
            timestamp=datetime.utcnow().isoformat()
        )
    
    def log_ai_interaction(self, user_id: str, query: str, response: str, 
                          model_used: str, tokens_used: int, response_time: float):
        """Log AI interactions for monitoring and optimization"""
        
        self.logger.info(
            "ai_interaction",
            user_id=user_id,
            tenant_id=self.tenant_id,
            query_length=len(query),
            response_length=len(response),
            model_used=model_used,
            tokens_used=tokens_used,
            response_time_ms=response_time * 1000,
            timestamp=datetime.utcnow().isoformat()
        )
    
    def log_security_event(self, event_type: str, user_id: str, ip_address: str, 
                          details: Dict[str, Any], severity: str = "medium"):
        """Log security events for threat detection"""
        
        self.logger.warning(
            "security_event",
            event_type=event_type,
            user_id=user_id,
            tenant_id=self.tenant_id,
            ip_address=ip_address,
            severity=severity,
            details=details,
            timestamp=datetime.utcnow().isoformat()
        )
```

#### Log Aggregation with ELK Stack

```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/config:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    networks:
      - elk
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elk
    depends_on:
      - logstash

volumes:
  elasticsearch_data:

networks:
  elk:
    driver: bridge
```

### 2. Metrics Collection with Prometheus

#### Custom Metrics Implementation

```python
from prometheus_client import Counter, Histogram, Gauge, CollectorRegistry, generate_latest
import time
from functools import wraps
from typing import Callable, Any

class ERPMetrics:
    """Custom metrics collector for ERP system"""
    
    def __init__(self, service_name: str):
        self.registry = CollectorRegistry()
        self.service_name = service_name
        
        # API Metrics
        self.api_requests_total = Counter(
            'erp_api_requests_total',
            'Total API requests',
            ['method', 'endpoint', 'status_code', 'tenant_id'],
            registry=self.registry
        )
        
        self.api_request_duration = Histogram(
            'erp_api_request_duration_seconds',
            'API request duration',
            ['method', 'endpoint', 'tenant_id'],
            registry=self.registry
        )
        
        # Business Metrics
        self.business_operations_total = Counter(
            'erp_business_operations_total',
            'Total business operations',
            ['operation_type', 'entity_type', 'tenant_id'],
            registry=self.registry
        )
        
        self.active_users = Gauge(
            'erp_active_users',
            'Number of active users',
            ['tenant_id'],
            registry=self.registry
        )
        
        # AI Metrics
        self.ai_requests_total = Counter(
            'erp_ai_requests_total',
            'Total AI requests',
            ['model', 'tenant_id'],
            registry=self.registry
        )
        
        self.ai_response_time = Histogram(
            'erp_ai_response_time_seconds',
            'AI response time',
            ['model', 'tenant_id'],
            registry=self.registry
        )
        
        self.ai_tokens_used = Counter(
            'erp_ai_tokens_used_total',
            'Total AI tokens used',
            ['model', 'tenant_id'],
            registry=self.registry
        )
        
        # Database Metrics
        self.db_connections_active = Gauge(
            'erp_db_connections_active',
            'Active database connections',
            ['database', 'tenant_id'],
            registry=self.registry
        )
        
        self.db_query_duration = Histogram(
            'erp_db_query_duration_seconds',
            'Database query duration',
            ['query_type', 'tenant_id'],
            registry=self.registry
        )
    
    def track_api_request(self, method: str, endpoint: str, tenant_id: str):
        """Decorator to track API requests"""
        def decorator(func: Callable) -> Callable:
            @wraps(func)
            async def wrapper(*args, **kwargs):
                start_time = time.time()
                status_code = 200
                
                try:
                    result = await func(*args, **kwargs)
                    return result
                except Exception as e:
                    status_code = getattr(e, 'status_code', 500)
                    raise
                finally:
                    duration = time.time() - start_time
                    
                    self.api_requests_total.labels(
                        method=method,
                        endpoint=endpoint,
                        status_code=status_code,
                        tenant_id=tenant_id
                    ).inc()
                    
                    self.api_request_duration.labels(
                        method=method,
                        endpoint=endpoint,
                        tenant_id=tenant_id
                    ).observe(duration)
            
            return wrapper
        return decorator
    
    def track_business_operation(self, operation_type: str, entity_type: str, tenant_id: str):
        """Track business operations"""
        self.business_operations_total.labels(
            operation_type=operation_type,
            entity_type=entity_type,
            tenant_id=tenant_id
        ).inc()
    
    def track_ai_request(self, model: str, tenant_id: str, tokens_used: int, response_time: float):
        """Track AI requests"""
        self.ai_requests_total.labels(
            model=model,
            tenant_id=tenant_id
        ).inc()
        
        self.ai_response_time.labels(
            model=model,
            tenant_id=tenant_id
        ).observe(response_time)
        
        self.ai_tokens_used.labels(
            model=model,
            tenant_id=tenant_id
        ).inc(tokens_used)
    
    def get_metrics(self) -> str:
        """Get metrics in Prometheus format"""
        return generate_latest(self.registry)

# Usage in FastAPI
from fastapi import FastAPI, Response

app = FastAPI()
metrics = ERPMetrics("erp-core")

@app.get("/metrics")
async def get_metrics():
    return Response(
        content=metrics.get_metrics(),
        media_type="text/plain"
    )
```

#### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "erp_rules.yml"

scrape_configs:
  - job_name: 'erp-core-services'
    static_configs:
      - targets: ['django-service:8000', 'fastapi-service:8001']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'erp-microservices'
    static_configs:
      - targets: ['invoice-service:8080', 'report-service:8081', 'email-service:8082']
    metrics_path: '/metrics'
    scrape_interval: 15s

  - job_name: 'erp-ai-services'
    static_configs:
      - targets: ['ai-gateway:8090', 'rag-engine:8091']
    metrics_path: '/metrics'
    scrape_interval: 30s

  - job_name: 'infrastructure'
    static_configs:
      - targets: ['postgres-exporter:9187', 'redis-exporter:9121', 'kafka-exporter:9308']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### 3. Distributed Tracing with Jaeger

#### OpenTelemetry Integration

```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.django import DjangoInstrumentor
from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

class TracingManager:
    """Manage distributed tracing for the ERP system"""
    
    def __init__(self, service_name: str, jaeger_endpoint: str):
        self.service_name = service_name
        
        # Configure tracer provider
        trace.set_tracer_provider(TracerProvider())
        tracer = trace.get_tracer_provider()
        
        # Configure Jaeger exporter
        jaeger_exporter = JaegerExporter(
            agent_host_name="jaeger",
            agent_port=6831,
        )
        
        # Add span processor
        span_processor = BatchSpanProcessor(jaeger_exporter)
        tracer.add_span_processor(span_processor)
        
        # Auto-instrument common libraries
        DjangoInstrumentor().instrument()
        Psycopg2Instrumentor().instrument()
        RedisInstrumentor().instrument()
        RequestsInstrumentor().instrument()
        
        self.tracer = trace.get_tracer(service_name)
    
    def trace_business_operation(self, operation_name: str, entity_type: str, entity_id: str):
        """Decorator for tracing business operations"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    span.set_attribute("entity.type", entity_type)
                    span.set_attribute("entity.id", entity_id)
                    span.set_attribute("service.name", self.service_name)
                    
                    try:
                        result = await func(*args, **kwargs)
                        span.set_attribute("operation.status", "success")
                        return result
                    except Exception as e:
                        span.set_attribute("operation.status", "error")
                        span.set_attribute("error.message", str(e))
                        span.record_exception(e)
                        raise
            
            return wrapper
        return decorator
    
    def trace_ai_operation(self, model: str, operation_type: str):
        """Decorator for tracing AI operations"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(f"ai.{operation_type}") as span:
                    span.set_attribute("ai.model", model)
                    span.set_attribute("ai.operation", operation_type)
                    
                    start_time = time.time()
                    
                    try:
                        result = await func(*args, **kwargs)
                        
                        # Add AI-specific attributes
                        if hasattr(result, 'usage'):
                            span.set_attribute("ai.tokens.input", result.usage.prompt_tokens)
                            span.set_attribute("ai.tokens.output", result.usage.completion_tokens)
                        
                        span.set_attribute("ai.response_time", time.time() - start_time)
                        return result
                        
                    except Exception as e:
                        span.set_attribute("ai.error", str(e))
                        span.record_exception(e)
                        raise
            
            return wrapper
        return decorator

# Usage example
tracing = TracingManager("erp-core", "http://jaeger:14268")

@tracing.trace_business_operation("create_employee", "employee", "emp_123")
async def create_employee(employee_data):
    # Business logic here
    pass

@tracing.trace_ai_operation("gpt-4", "generate_response")
async def generate_ai_response(prompt):
    # AI logic here
    pass
```

### 4. Grafana Dashboards

#### ERP System Dashboard Configuration

```json
{
  "dashboard": {
    "id": null,
    "title": "ERP System Overview",
    "tags": ["erp", "overview"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "API Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(erp_api_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec"
          }
        ]
      },
      {
        "id": 2,
        "title": "API Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(erp_api_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(erp_api_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "id": 3,
        "title": "Active Users by Tenant",
        "type": "stat",
        "targets": [
          {
            "expr": "erp_active_users",
            "legendFormat": "{{tenant_id}}"
          }
        ]
      },
      {
        "id": 4,
        "title": "AI Token Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(erp_ai_tokens_used_total[1h])",
            "legendFormat": "{{model}} - {{tenant_id}}"
          }
        ]
      },
      {
        "id": 5,
        "title": "Database Connection Pool",
        "type": "graph",
        "targets": [
          {
            "expr": "erp_db_connections_active",
            "legendFormat": "{{database}}"
          }
        ]
      },
      {
        "id": 6,
        "title": "Business Operations",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(erp_business_operations_total[5m])",
            "legendFormat": "{{operation_type}} - {{entity_type}}"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## CI/CD Pipeline Implementation

### 1. GitHub Actions Workflows

#### Main CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

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
          POSTGRES_DB: test_erp
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
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
    
    - name: Install Python dependencies
      run: |
        cd services/core
        pip install -r requirements.txt
        pip install -r requirements-dev.txt
    
    - name: Install Go dependencies
      run: |
        cd services/microservices
        go mod download
    
    - name: Install Node.js dependencies
      run: |
        cd frontend/web
        npm ci
    
    - name: Run Python tests
      run: |
        cd services/core
        python -m pytest tests/ --cov=apps --cov-report=xml
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_erp
        REDIS_URL: redis://localhost:6379/0
    
    - name: Run Go tests
      run: |
        cd services/microservices
        go test -v -race -coverprofile=coverage.out ./...
    
    - name: Run Frontend tests
      run: |
        cd frontend/web
        npm test -- --coverage --watchAll=false
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        files: ./services/core/coverage.xml,./services/microservices/coverage.out,./frontend/web/coverage/lcov.info

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Run Bandit security linter
      run: |
        pip install bandit
        bandit -r services/core/apps -f json -o bandit-report.json
    
    - name: Run npm audit
      run: |
        cd frontend/web
        npm audit --audit-level moderate

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    strategy:
      matrix:
        service:
          - name: core
            context: ./services/core
            dockerfile: ./services/core/Dockerfile
          - name: ai-gateway
            context: ./services/ai/ai-gateway
            dockerfile: ./services/ai/ai-gateway/Dockerfile
          - name: invoice-service
            context: ./services/microservices/invoice-service
            dockerfile: ./services/microservices/invoice-service/Dockerfile
          - name: frontend
            context: ./frontend/web
            dockerfile: ./frontend/web/Dockerfile
    
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
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-${{ matrix.service.name }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: ${{ matrix.service.context }}
        file: ${{ matrix.service.dockerfile }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to staging
      run: |
        # Deploy to staging environment
        echo "Deploying to staging..."
        # Add deployment commands here
```

#### Database Migration Workflow

```yaml
# .github/workflows/migrations.yml
name: Database Migrations

on:
  push:
    paths:
      - 'services/core/apps/*/migrations/**'
    branches: [main, develop]

jobs:
  check-migrations:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_erp
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        cd services/core
        pip install -r requirements.txt
    
    - name: Check for migration conflicts
      run: |
        cd services/core
        python manage.py makemigrations --check --dry-run
    
    - name: Run migrations
      run: |
        cd services/core
        python manage.py migrate
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_erp
    
    - name: Check migration reversibility
      run: |
        cd services/core
        python manage.py migrate --fake-initial
        python manage.py showmigrations
```

### 2. Environment Management

#### Development Environment Setup

```bash
#!/bin/bash
# scripts/setup-dev.sh

set -e

echo "Setting up development environment..."

# Check prerequisites
command -v docker >/dev/null 2>&1 || { echo "Docker is required but not installed. Aborting." >&2; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "Docker Compose is required but not installed. Aborting." >&2; exit 1; }

# Create environment files
if [ ! -f .env ]; then
    cp .env.example .env
    echo "Created .env file. Please update with your configuration."
fi

# Start infrastructure services
echo "Starting infrastructure services..."
docker-compose -f docker-compose.infrastructure.yml up -d

# Wait for services to be ready
echo "Waiting for services to be ready..."
sleep 30

# Set up Python environment
echo "Setting up Python environment..."
cd services/core
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Run migrations
echo "Running database migrations..."
python manage.py migrate

# Create superuser
echo "Creating superuser..."
python manage.py createsuperuser --noinput --email admin@example.com || true

# Set up Go environment
echo "Setting up Go environment..."
cd ../../services/microservices
go mod download

# Set up Node.js environment
echo "Setting up Node.js environment..."
cd ../../frontend/web
npm install

# Generate API documentation
echo "Generating API documentation..."
cd ../../services/core
python manage.py spectacular --file schema.yml

echo "Development environment setup complete!"
echo "Run 'docker-compose up' to start all services."
```

#### Production Deployment Script

```bash
#!/bin/bash
# scripts/deploy-production.sh

set -e

ENVIRONMENT=${1:-production}
VERSION=${2:-latest}

echo "Deploying ERP system to $ENVIRONMENT environment..."

# Backup database
echo "Creating database backup..."
kubectl exec -n $ENVIRONMENT deployment/postgres -- pg_dump -U postgres erp_production > backup-$(date +%Y%m%d-%H%M%S).sql

# Update Kubernetes manifests
echo "Updating Kubernetes manifests..."
sed -i "s/image: .*/image: ghcr.io\/company\/erp-core:$VERSION/" k8s/$ENVIRONMENT/core-deployment.yaml
sed -i "s/image: .*/image: ghcr.io\/company\/erp-ai-gateway:$VERSION/" k8s/$ENVIRONMENT/ai-gateway-deployment.yaml

# Apply Kubernetes manifests
echo "Applying Kubernetes manifests..."
kubectl apply -f k8s/$ENVIRONMENT/

# Wait for rollout
echo "Waiting for deployment rollout..."
kubectl rollout status deployment/core -n $ENVIRONMENT
kubectl rollout status deployment/ai-gateway -n $ENVIRONMENT

# Run database migrations
echo "Running database migrations..."
kubectl exec -n $ENVIRONMENT deployment/core -- python manage.py migrate

# Health check
echo "Performing health check..."
HEALTH_URL="https://api-$ENVIRONMENT.erp-system.com/health"
for i in {1..30}; do
    if curl -f $HEALTH_URL; then
        echo "Health check passed!"
        break
    fi
    echo "Health check failed, retrying in 10 seconds..."
    sleep 10
done

echo "Deployment to $ENVIRONMENT completed successfully!"
```

### 3. Kubernetes Deployment Manifests

#### Core Service Deployment

```yaml
# k8s/production/core-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: core
  namespace: production
  labels:
    app: erp-core
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: erp-core
  template:
    metadata:
      labels:
        app: erp-core
        version: v1
    spec:
      containers:
      - name: core
        image: ghcr.io/company/erp-core:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: secret-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: core-service
  namespace: production
spec:
  selector:
    app: erp-core
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: ClusterIP
```

#### Ingress Configuration

```yaml
# k8s/production/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: erp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.erp-system.com
    - app.erp-system.com
    secretName: erp-tls
  rules:
  - host: api.erp-system.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: core-service
            port:
              number: 80
  - host: app.erp-system.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 4. Performance Testing

#### Load Testing with K6

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 10 }, // Ramp up
    { duration: '5m', target: 50 }, // Stay at 50 users
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 0 }, // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must complete below 500ms
    errors: ['rate<0.1'], // Error rate must be below 10%
  },
};

const BASE_URL = 'https://api.erp-system.com';

export function setup() {
  // Login and get auth token
  const loginResponse = http.post(`${BASE_URL}/auth/login`, {
    email: 'test@example.com',
    password: 'testpassword',
  });
  
  return {
    authToken: loginResponse.json('access_token'),
  };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.authToken}`,
    'Content-Type': 'application/json',
  };
  
  // Test API endpoints
  const endpoints = [
    '/api/hrm/employees',
    '/api/crm/contacts',
    '/api/finance/invoices',
    '/api/inventory/products',
  ];
  
  const endpoint = endpoints[Math.floor(Math.random() * endpoints.length)];
  const response = http.get(`${BASE_URL}${endpoint}`, { headers });
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  errorRate.add(response.status !== 200);
  
  sleep(1);
}

export function teardown(data) {
  // Cleanup if needed
}
```

### 5. Backup and Disaster Recovery

#### Automated Backup Script

```bash
#!/bin/bash
# scripts/backup.sh

set -e

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

echo "Starting backup process..."

# Database backup
echo "Backing up PostgreSQL database..."
kubectl exec -n production deployment/postgres -- pg_dump -U postgres erp_production | gzip > "$BACKUP_DIR/db_backup_$DATE.sql.gz"

# Redis backup
echo "Backing up Redis data..."
kubectl exec -n production deployment/redis -- redis-cli BGSAVE
kubectl cp production/redis-pod:/data/dump.rdb "$BACKUP_DIR/redis_backup_$DATE.rdb"

# File storage backup
echo "Backing up file storage..."
kubectl exec -n production deployment/core -- tar -czf - /app/media | cat > "$BACKUP_DIR/files_backup_$DATE.tar.gz"

# Vector database backup
echo "Backing up vector database..."
kubectl exec -n production deployment/qdrant -- tar -czf - /qdrant/storage | cat > "$BACKUP_DIR/vector_backup_$DATE.tar.gz"

# Upload to cloud storage
echo "Uploading backups to cloud storage..."
aws s3 sync "$BACKUP_DIR" s3://erp-backups/$(date +%Y/%m/%d)/

# Cleanup old backups
echo "Cleaning up old backups..."
find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.rdb" -mtime +$RETENTION_DAYS -delete

echo "Backup process completed successfully!"
```

This comprehensive set of optional enhancements provides robust observability, automated deployment, and operational excellence for the enterprise ERP system.