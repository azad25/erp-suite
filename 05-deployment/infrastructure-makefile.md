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
	@grep -E '^[a-zA-Z_-]+:.*?## .*$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "$(GREEN)%-20s$(RESET) %s\n", $1, $2}'

up: prepare-environment ## Start all services with hot reloading
	@echo "$(GREEN)🚀 Starting Intelligent ERP Suite...$(RESET)"
	@docker compose --profile infrastructure --profile api-layer --profile monitoring up -d
	@echo "$(GREEN)⏳ Waiting for services to start...$(RESET)"
	@sleep 30
	@$(MAKE) seed-data
	@echo "$(GREEN)✅ All services started!$(RESET)"
	@echo "$(BLUE)GraphQL Gateway: http://localhost:4000/graphql$(RESET)"
	@echo "$(BLUE)GraphQL Playground: http://localhost:4000/playground$(RESET)"
	@echo "$(BLUE)Service Registry: http://localhost:8500$(RESET)"
	@echo "$(BLUE)Frontend: http://localhost:3000$(RESET)" # (if using full-stack profile)
	@echo "$(BLUE)Django Core: http://localhost:8000$(RESET)" # (if using full-stack profile)
```

This comprehensive Makefile provides all the automation needed for the development and deployment workflow of the Intelligent ERP Suite.