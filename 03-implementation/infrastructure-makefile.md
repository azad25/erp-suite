# Infrastructure Makefile

This is the main Makefile that should be placed in the `erp-infrastructure` repository to provide the one-command development experience.

```makefile
# Intelligent ERP Suite - Development Environment
# Usage: make up (starts everything)

.PHONY: help up down restart logs logs-follow reset health test docs clean

# Default target
.DEFAULT_GOAL := help

# Colors for output
RED := \033[31m
GREEN := \033[32m
YELLOW := \033[33m
BLUE := \033[34m
RESET := \033[0m

# Configuration
COMPOSE_FILE := docker-compose.dev.yml
SERVICES := erp-auth-service erp-subscription-service erp-crm-service erp-hrm-service erp-accounting-service erp-inventory-service erp-project-service erp-notification-service erp-file-service erp-ai-platform erp-frontend
REPOS_DIR := ../

help: ## Show this help message
	@echo "$(BLUE)Intelligent ERP Suite - Development Commands$(RESET)"
	@echo ""
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "$(GREEN)%-20s$(RESET) %s\n", $$1, $$2}'

up: clone-repos ## Start all services with hot reloading
	@echo "$(GREEN)🚀 Starting Intelligent ERP Suite...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) up -d
	@echo "$(GREEN)⏳ Waiting for services to start...$(RESET)"
	@sleep 30
	@$(MAKE) seed-data
	@echo "$(GREEN)✅ All services started!$(RESET)"
	@echo "$(BLUE)Frontend: http://localhost:3000$(RESET)"
	@echo "$(BLUE)API Gateway: http://localhost:8080$(RESET)"
	@echo "$(BLUE)Dev Dashboard: http://localhost:3000/dev$(RESET)"

down: ## Stop all services
	@echo "$(YELLOW)🛑 Stopping all services...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) down
	@echo "$(GREEN)✅ All services stopped$(RESET)"

restart: ## Restart specific service (usage: make restart service=erp-crm-service)
	@if [ -z "$(service)" ]; then \
		echo "$(RED)❌ Please specify service: make restart service=erp-crm-service$(RESET)"; \
		exit 1; \
	fi
	@echo "$(YELLOW)🔄 Restarting $(service)...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) restart $(service)
	@echo "$(GREEN)✅ $(service) restarted$(RESET)"

logs: ## View logs for all services or specific service (usage: make logs service=erp-crm-service)
	@if [ -z "$(service)" ]; then \
		docker-compose -f $(COMPOSE_FILE) logs --tail=100; \
	else \
		docker-compose -f $(COMPOSE_FILE) logs --tail=100 $(service); \
	fi

logs-follow: ## Follow logs in real-time (usage: make logs-follow service=erp-crm-service)
	@if [ -z "$(service)" ]; then \
		docker-compose -f $(COMPOSE_FILE) logs -f; \
	else \
		docker-compose -f $(COMPOSE_FILE) logs -f $(service); \
	fi

reset: down ## Reset entire environment (stops, removes containers, volumes)
	@echo "$(YELLOW)🗑️  Resetting environment...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) down -v --remove-orphans
	@docker system prune -f
	@echo "$(GREEN)✅ Environment reset complete$(RESET)"

reset-db: ## Reset databases only (usage: make reset-db service=erp-crm-service)
	@if [ -z "$(service)" ]; then \
		echo "$(RED)❌ Please specify service: make reset-db service=erp-crm-service$(RESET)"; \
		exit 1; \
	fi
	@echo "$(YELLOW)🗑️  Resetting database for $(service)...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) exec $(service) python manage.py flush --noinput
	@docker-compose -f $(COMPOSE_FILE) exec $(service) python manage.py migrate
	@docker-compose -f $(COMPOSE_FILE) exec $(service) python manage.py seed_data
	@echo "$(GREEN)✅ Database reset for $(service)$(RESET)"

health: ## Check health of all services
	@echo "$(BLUE)🏥 Checking service health...$(RESET)"
	@for service in $(SERVICES); do \
		echo -n "$$service: "; \
		if curl -s -f http://localhost:$$(docker-compose -f $(COMPOSE_FILE) port $$service 8000 | cut -d: -f2)/health > /dev/null 2>&1; then \
			echo "$(GREEN)✅ Healthy$(RESET)"; \
		else \
			echo "$(RED)❌ Unhealthy$(RESET)"; \
		fi; \
	done

test: ## Run tests for all services or specific service
	@if [ -z "$(service)" ]; then \
		echo "$(BLUE)🧪 Running tests for all services...$(RESET)"; \
		for svc in $(SERVICES); do \
			echo "Testing $$svc..."; \
			docker-compose -f $(COMPOSE_FILE) exec $$svc make test || true; \
		done; \
	else \
		echo "$(BLUE)🧪 Running tests for $(service)...$(RESET)"; \
		docker-compose -f $(COMPOSE_FILE) exec $(service) make test; \
	fi

test-api: ## Test specific API endpoint (usage: make test-api endpoint=/api/crm/contacts)
	@if [ -z "$(endpoint)" ]; then \
		echo "$(RED)❌ Please specify endpoint: make test-api endpoint=/api/crm/contacts$(RESET)"; \
		exit 1; \
	fi
	@echo "$(BLUE)🧪 Testing API endpoint $(endpoint)...$(RESET)"
	@curl -s -H "Authorization: Bearer $$(make get-test-token)" http://localhost:8080$(endpoint) | jq .

docs: ## Generate and open API documentation
	@echo "$(BLUE)📚 Opening API documentation...$(RESET)"
	@open http://localhost:8080/docs

db-connect: ## Connect to database (usage: make db-connect service=erp-crm-service)
	@if [ -z "$(service)" ]; then \
		echo "$(RED)❌ Please specify service: make db-connect service=erp-crm-service$(RESET)"; \
		exit 1; \
	fi
	@docker-compose -f $(COMPOSE_FILE) exec $(service)-db psql -U postgres -d $(service)_db

db-query: ## Run database query (usage: make db-query service=erp-crm-service query="SELECT * FROM contacts LIMIT 5")
	@if [ -z "$(service)" ] || [ -z "$(query)" ]; then \
		echo "$(RED)❌ Usage: make db-query service=erp-crm-service query=\"SELECT * FROM contacts LIMIT 5\"$(RESET)"; \
		exit 1; \
	fi
	@docker-compose -f $(COMPOSE_FILE) exec $(service)-db psql -U postgres -d $(service)_db -c "$(query)"

db-status: ## Check database connection status
	@echo "$(BLUE)🗄️  Checking database connections...$(RESET)"
	@for service in $(SERVICES); do \
		if [[ "$$service" == *"-service" ]]; then \
			echo -n "$$service DB: "; \
			if docker-compose -f $(COMPOSE_FILE) exec $$service-db pg_isready -U postgres > /dev/null 2>&1; then \
				echo "$(GREEN)✅ Connected$(RESET)"; \
			else \
				echo "$(RED)❌ Disconnected$(RESET)"; \
			fi; \
		fi; \
	done

seed-data: ## Seed all databases with test data
	@echo "$(BLUE)🌱 Seeding databases with test data...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) exec erp-auth-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-subscription-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-crm-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-hrm-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-accounting-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-inventory-service python manage.py seed_data
	@docker-compose -f $(COMPOSE_FILE) exec erp-project-service python manage.py seed_data
	@echo "$(GREEN)✅ All databases seeded$(RESET)"

get-test-token: ## Get test JWT token for API testing
	@curl -s -X POST http://localhost:8001/auth/login \
		-H "Content-Type: application/json" \
		-d '{"email":"admin@acme.com","password":"password123"}' | jq -r .access_token

clone-repos: ## Clone all repositories if they don't exist
	@echo "$(BLUE)📦 Checking repositories...$(RESET)"
	@for repo in erp-shared-libs $(SERVICES); do \
		if [ ! -d "$(REPOS_DIR)$$repo" ]; then \
			echo "Cloning $$repo..."; \
			git clone https://github.com/intelligent-erp-suite/$$repo.git $(REPOS_DIR)$$repo; \
		fi; \
	done
	@echo "$(GREEN)✅ All repositories ready$(RESET)"

build: ## Build all Docker images
	@echo "$(BLUE)🏗️  Building all images...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) build
	@echo "$(GREEN)✅ All images built$(RESET)"

build-prod: ## Build production images
	@echo "$(BLUE)🏗️  Building production images...$(RESET)"
	@docker-compose -f docker-compose.prod.yml build
	@echo "$(GREEN)✅ Production images built$(RESET)"

deploy-staging: ## Deploy to staging environment
	@echo "$(BLUE)🚀 Deploying to staging...$(RESET)"
	@kubectl apply -f k8s/staging/
	@echo "$(GREEN)✅ Deployed to staging$(RESET)"

deploy-prod: ## Deploy to production environment
	@echo "$(YELLOW)⚠️  Deploying to production...$(RESET)"
	@kubectl apply -f k8s/production/
	@echo "$(GREEN)✅ Deployed to production$(RESET)"

clean: ## Clean up Docker resources
	@echo "$(YELLOW)🧹 Cleaning up Docker resources...$(RESET)"
	@docker system prune -f
	@docker volume prune -f
	@echo "$(GREEN)✅ Cleanup complete$(RESET)"

debug-reload: ## Debug hot reloading issues
	@echo "$(BLUE)🔍 Debugging hot reload...$(RESET)"
	@docker-compose -f $(COMPOSE_FILE) exec erp-frontend npm run dev -- --verbose
	@docker-compose -f $(COMPOSE_FILE) logs --tail=50 erp-crm-service

monitoring: ## Open monitoring dashboards
	@echo "$(BLUE)📊 Opening monitoring dashboards...$(RESET)"
	@open http://localhost:9090  # Prometheus
	@open http://localhost:3001  # Grafana
	@open http://localhost:16686 # Jaeger

# Development shortcuts
dev: up ## Alias for 'make up'
stop: down ## Alias for 'make down'
status: health ## Alias for 'make health'
```

## Service-Specific Makefiles

Each service should also have its own Makefile:

### Example: erp-crm-service/Makefile
```makefile
.PHONY: test migrate seed-data lint format

test:
	python manage.py test

migrate:
	python manage.py makemigrations
	python manage.py migrate

seed-data:
	python manage.py seed_data

lint:
	flake8 .
	black --check .

format:
	black .
	isort .

shell:
	python manage.py shell

runserver:
	python manage.py runserver 0.0.0.0:8000
```

This Makefile provides a comprehensive development experience with one-command setup, testing, debugging, and deployment capabilities.