# Docker Compose Development Configuration

This is the `docker-compose.dev.yml` file that should be placed in the `erp-infrastructure` repository to orchestrate all 14 services for development.

```yaml
version: '3.8'

services:
  # =============================================================================
  # DATABASES
  # =============================================================================
  
  postgres-auth:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: auth_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_auth_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-subscription:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: subscription_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_subscription_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"

  postgres-crm:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: crm_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_crm_data:/var/lib/postgresql/data
    ports:
      - "5434:5432"

  postgres-hrm:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: hrm_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_hrm_data:/var/lib/postgresql/data
    ports:
      - "5435:5432"

  postgres-accounting:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: accounting_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_accounting_data:/var/lib/postgresql/data
    ports:
      - "5436:5432"

  postgres-inventory:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: inventory_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_inventory_data:/var/lib/postgresql/data
    ports:
      - "5437:5432"

  postgres-project:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: project_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_project_data:/var/lib/postgresql/data
    ports:
      - "5438:5432"

  # =============================================================================
  # CACHE & MESSAGE QUEUE
  # =============================================================================

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  # =============================================================================
  # AI & VECTOR DATABASE
  # =============================================================================

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
    command: >
      sh -c "ollama serve & 
             sleep 10 && 
             ollama pull llama2:7b && 
             ollama pull codellama:7b && 
             ollama pull mistral:7b && 
             wait"

  # =============================================================================
  # CORE SERVICES
  # =============================================================================

  erp-auth-service:
    build:
      context: ../erp-auth-service
      dockerfile: Dockerfile.dev
    ports:
      - "8001:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-auth:5432/auth_db
      - REDIS_URL=redis://redis:6379/0
      - JWT_SECRET=dev_jwt_secret_key_change_in_production
      - DEBUG=true
    volumes:
      - ../erp-auth-service:/app
      - /app/node_modules
    depends_on:
      postgres-auth:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  erp-subscription-service:
    build:
      context: ../erp-subscription-service
      dockerfile: Dockerfile.dev
    ports:
      - "8002:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-subscription:5432/subscription_db
      - REDIS_URL=redis://redis:6379/1
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - STRIPE_SECRET_KEY=sk_test_dev_key
      - STRIPE_WEBHOOK_SECRET=whsec_dev_secret
      - DEBUG=true
    volumes:
      - ../erp-subscription-service:/app
    depends_on:
      - postgres-subscription
      - redis
      - erp-auth-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # =============================================================================
  # BUSINESS SERVICES
  # =============================================================================

  erp-crm-service:
    build:
      context: ../erp-crm-service
      dockerfile: Dockerfile.dev
    ports:
      - "8003:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-crm:5432/crm_db
      - REDIS_URL=redis://redis:6379/2
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - DEBUG=true
    volumes:
      - ../erp-crm-service:/app
    depends_on:
      - postgres-crm
      - redis
      - erp-auth-service
      - erp-subscription-service

  erp-hrm-service:
    build:
      context: ../erp-hrm-service
      dockerfile: Dockerfile.dev
    ports:
      - "8004:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-hrm:5432/hrm_db
      - REDIS_URL=redis://redis:6379/3
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - DEBUG=true
    volumes:
      - ../erp-hrm-service:/app
    depends_on:
      - postgres-hrm
      - redis
      - erp-auth-service
      - erp-subscription-service

  erp-accounting-service:
    build:
      context: ../erp-accounting-service
      dockerfile: Dockerfile.dev
    ports:
      - "8005:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-accounting:5432/accounting_db
      - REDIS_URL=redis://redis:6379/4
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - DEBUG=true
    volumes:
      - ../erp-accounting-service:/app
    depends_on:
      - postgres-accounting
      - redis
      - erp-auth-service
      - erp-subscription-service

  erp-inventory-service:
    build:
      context: ../erp-inventory-service
      dockerfile: Dockerfile.dev
    ports:
      - "8006:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-inventory:5432/inventory_db
      - REDIS_URL=redis://redis:6379/5
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - DEBUG=true
    volumes:
      - ../erp-inventory-service:/app
    depends_on:
      - postgres-inventory
      - redis
      - erp-auth-service
      - erp-subscription-service

  erp-project-service:
    build:
      context: ../erp-project-service
      dockerfile: Dockerfile.dev
    ports:
      - "8007:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:dev_password@postgres-project:5432/project_db
      - REDIS_URL=redis://redis:6379/6
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - HRM_SERVICE_URL=http://erp-hrm-service:8000
      - DEBUG=true
    volumes:
      - ../erp-project-service:/app
    depends_on:
      - postgres-project
      - redis
      - erp-auth-service
      - erp-subscription-service
      - erp-hrm-service

  # =============================================================================
  # SUPPORT SERVICES
  # =============================================================================

  erp-notification-service:
    build:
      context: ../erp-notification-service
      dockerfile: Dockerfile.dev
    ports:
      - "8009:8000"
    environment:
      - REDIS_URL=redis://redis:6379/7
      - KAFKA_BROKERS=kafka:9092
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SMTP_HOST=mailhog
      - SMTP_PORT=1025
      - DEBUG=true
    volumes:
      - ../erp-notification-service:/app
    depends_on:
      - redis
      - kafka
      - erp-auth-service

  erp-file-service:
    build:
      context: ../erp-file-service
      dockerfile: Dockerfile.dev
    ports:
      - "8010:8000"
    environment:
      - REDIS_URL=redis://redis:6379/8
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - STORAGE_TYPE=local
      - STORAGE_PATH=/app/storage
      - DEBUG=true
    volumes:
      - ../erp-file-service:/app
      - file_storage:/app/storage
    depends_on:
      - redis
      - erp-auth-service

  # =============================================================================
  # AI PLATFORM
  # =============================================================================

  erp-ai-platform:
    build:
      context: ../erp-ai-platform
      dockerfile: Dockerfile.dev
    ports:
      - "8008:8000"
    environment:
      - REDIS_URL=redis://redis:6379/9
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - QDRANT_URL=http://qdrant:6333
      - OLLAMA_URL=http://ollama:11434
      - OPENAI_API_KEY=sk-dev-key-set-real-key-for-openai
      - DEBUG=true
    volumes:
      - ../erp-ai-platform:/app
    depends_on:
      - redis
      - qdrant
      - ollama
      - erp-auth-service

  # =============================================================================
  # API GATEWAY
  # =============================================================================

  erp-api-gateway:
    build:
      context: ../erp-api-gateway
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8000"
    environment:
      - AUTH_SERVICE_URL=http://erp-auth-service:8000
      - SUBSCRIPTION_SERVICE_URL=http://erp-subscription-service:8000
      - CRM_SERVICE_URL=http://erp-crm-service:8000
      - HRM_SERVICE_URL=http://erp-hrm-service:8000
      - ACCOUNTING_SERVICE_URL=http://erp-accounting-service:8000
      - INVENTORY_SERVICE_URL=http://erp-inventory-service:8000
      - PROJECT_SERVICE_URL=http://erp-project-service:8000
      - NOTIFICATION_SERVICE_URL=http://erp-notification-service:8000
      - FILE_SERVICE_URL=http://erp-file-service:8000
      - AI_SERVICE_URL=http://erp-ai-platform:8000
      - DEBUG=true
    volumes:
      - ../erp-api-gateway:/app
    depends_on:
      - erp-auth-service
      - erp-subscription-service
      - erp-crm-service
      - erp-hrm-service
      - erp-accounting-service
      - erp-inventory-service
      - erp-project-service

  # =============================================================================
  # FRONTEND
  # =============================================================================

  erp-frontend:
    build:
      context: ../erp-frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:${DJANGO_PORT:-8000}
      - NEXT_PUBLIC_GRAPHQL_URL=http://localhost:${GRAPHQL_GATEWAY_PORT:-4000}/graphql
      - NEXT_PUBLIC_AUTH_URL=http://localhost:${AUTH_SERVICE_HTTP_PORT:-8080}
      - NODE_ENV=development
    volumes:
      - ../erp-frontend:/app
      - /app/node_modules
      - /app/.next
    depends_on:
      - erp-api-gateway

  # =============================================================================
  # DEVELOPMENT TOOLS
  # =============================================================================

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI

  # =============================================================================
  # MONITORING
  # =============================================================================

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

volumes:
  postgres_auth_data:
  postgres_subscription_data:
  postgres_crm_data:
  postgres_hrm_data:
  postgres_accounting_data:
  postgres_inventory_data:
  postgres_project_data:
  redis_data:
  qdrant_data:
  ollama_data:
  file_storage:
  prometheus_data:
  grafana_data:

networks:
  default:
    name: erp-dev-network
```

## Service Dockerfile.dev Examples

### Django Service (e.g., erp-crm-service/Dockerfile.dev)
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt requirements-dev.txt ./
RUN pip install --no-cache-dir -r requirements-dev.txt

# Copy application code
COPY . .

# Install development tools
RUN pip install watchdog[watchmedo]

# Create entrypoint script
RUN echo '#!/bin/bash\n\
set -e\n\
echo "Waiting for database..."\n\
while ! pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do\n\
  sleep 1\n\
done\n\
echo "Database is ready!"\n\
python manage.py migrate\n\
python manage.py collectstatic --noinput\n\
exec "$@"' > /entrypoint.sh && chmod +x /entrypoint.sh

EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### Go Service (e.g., erp-auth-service/Dockerfile.dev)
```dockerfile
FROM golang:1.21-alpine

WORKDIR /app

# Install Air for hot reloading
RUN go install github.com/cosmtrek/air@latest

# Install dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Create Air config
RUN echo 'root = "."\n\
testdata_dir = "testdata"\n\
tmp_dir = "tmp"\n\
\n\
[build]\n\
  args_bin = []\n\
  bin = "./tmp/main"\n\
  cmd = "go build -o ./tmp/main ."\n\
  delay = 1000\n\
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]\n\
  exclude_file = []\n\
  exclude_regex = ["_test.go"]\n\
  exclude_unchanged = false\n\
  follow_symlink = false\n\
  full_bin = ""\n\
  include_dir = []\n\
  include_ext = ["go", "tpl", "tmpl", "html"]\n\
  kill_delay = "0s"\n\
  log = "build-errors.log"\n\
  send_interrupt = false\n\
  stop_on_root = false\n\
\n\
[color]\n\
  app = ""\n\
  build = "yellow"\n\
  main = "magenta"\n\
  runner = "green"\n\
  watcher = "cyan"\n\
\n\
[log]\n\
  time = false\n\
\n\
[misc]\n\
  clean_on_exit = false' > .air.toml

EXPOSE 8000

CMD ["air", "-c", ".air.toml"]
```

### Next.js Frontend (erp-frontend/Dockerfile.dev)
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package.json package-lock.json ./
RUN npm ci

# Copy source code
COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

This Docker Compose setup provides a complete development environment with hot reloading, database seeding, monitoring, and all the tools needed for rapid development.