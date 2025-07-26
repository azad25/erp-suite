# Intelligent ERP Suite - Multi-Repository SaaS Platform

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Architecture](https://img.shields.io/badge/architecture-14--repositories-green.svg)](./05-deployment/multi-repo-microservices-plan.md)
[![Dev-First](https://img.shields.io/badge/dev--first-make%20up-blue.svg)](./03-implementation/dev-first-setup.md)
[![AI-Powered](https://img.shields.io/badge/AI-RAG%20Enabled-purple.svg)](./ai-chatbot-design.md)

## 🛠️ Technology Stack

### Backend & APIs
![Python](https://img.shields.io/badge/Python_3.11+-3776AB?style=flat&logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django_4.2+-092E20?style=flat&logo=django&logoColor=white)
![Django REST](https://img.shields.io/badge/Django_REST_Framework-ff1709?style=flat&logo=django&logoColor=white)
![Go](https://img.shields.io/badge/Go_1.21+-00ADD8?style=flat&logo=go&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![Chi Router](https://img.shields.io/badge/Chi_Router-00ADD8?style=flat&logo=go&logoColor=white)

### Frontend & UI
![Next.js](https://img.shields.io/badge/Next.js_14+-000000?style=flat&logo=next.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=flat&logo=tailwind-css&logoColor=white)
![Zustand](https://img.shields.io/badge/Zustand-FF6B35?style=flat&logo=react&logoColor=white)
![React Query](https://img.shields.io/badge/React_Query-FF4154?style=flat&logo=react-query&logoColor=white)

### Databases & Storage
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![Qdrant](https://img.shields.io/badge/Qdrant-FF6B6B?style=flat&logo=qdrant&logoColor=white)
![AWS S3](https://img.shields.io/badge/AWS_S3-569A31?style=flat&logo=amazon-s3&logoColor=white)

### AI & Machine Learning
![Ollama](https://img.shields.io/badge/Ollama-000000?style=flat&logo=ollama&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-1C3C3C?style=flat&logo=langchain&logoColor=white)
![LLaMA 2](https://img.shields.io/badge/LLaMA_2-FF6B35?style=flat&logo=meta&logoColor=white)
![CodeLlama](https://img.shields.io/badge/CodeLlama-FF6B35?style=flat&logo=meta&logoColor=white)
![Mistral](https://img.shields.io/badge/Mistral-FF6B35?style=flat&logo=mistral&logoColor=white)

### DevOps & Infrastructure
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-2496ED?style=flat&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-623CE4?style=flat&logo=terraform&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![Air](https://img.shields.io/badge/Air-00ADD8?style=flat&logo=go&logoColor=white)

### Monitoring & Testing
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=flat&logo=pytest&logoColor=white)
![Jest](https://img.shields.io/badge/Jest-C21325?style=flat&logo=jest&logoColor=white)

### Payment & SaaS
![Stripe](https://img.shields.io/badge/Stripe-008CDD?style=flat&logo=stripe&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-000000?style=flat&logo=json-web-tokens&logoColor=white)

> **A dev-first, subscription-based SaaS ERP platform with 14 independent microservices and embedded AI capabilities**

## 🚀 Project Overview

The Intelligent ERP Suite is a **dev-first, multi-repository SaaS platform** built with 14 independent microservices. Each service can be developed, tested, and deployed independently while sharing common RBAC patterns and subscription integration. The platform offers `make up` one-command setup, hot reloading across all services, and local LLM support for rapid development.

### ✨ Key Features

- � **Dev-cFirst Experience**: `make up` starts all 14 services with hot reloading and seeded data
- �️ **14-iRepository Architecture**: Independent development and deployment per service
- 💳 **SaaS Foundation**: Subscription service with feature gating and usage tracking
- � **Sohared RBAC Patterns**: Common authentication, authorization, and tenant isolation
- 🤖 **AI Platform**: RAG-enabled chatbot with local LLM support (Ollama)
- 📊 **Subscription Analytics**: Usage tracking, billing automation, and customer portals
- ⚡ **Hot Reloading**: Instant code changes across Django, Go, and Next.js services
- 🧪 **Comprehensive Testing**: Shared testing utilities and patterns across all services

### 🏗️ 14-Service Architecture

| Service | Repository | Tech Stack | Purpose |
|---------|------------|------------|---------|
| **Auth Service** | `erp-auth-service` | Go + JWT | Multi-tenant authentication & RBAC |
| **Subscription Service** | `erp-subscription-service` | Django + Stripe | SaaS billing & feature gating |
| **CRM Service** | `erp-crm-service` | Django + DRF | Contact & lead management |
| **HRM Service** | `erp-hrm-service` | Django + DRF | Employee & leave management |
| **Accounting Service** | `erp-accounting-service` | Django + DRF | Financial management & invoicing |
| **Inventory Service** | `erp-inventory-service` | Django + DRF | Product & stock management |
| **Project Service** | `erp-project-service` | Django + DRF | Task & resource management |
| **AI Platform** | `erp-ai-platform` | FastAPI + Ollama | RAG chatbot & ML services |
| **Notification Service** | `erp-notification-service` | Go + WebSocket | Email, SMS, real-time notifications |
| **File Service** | `erp-file-service` | Go + S3 | Document management & PDF generation |
| **API Gateway** | `erp-api-gateway` | Go + Chi | Request routing & rate limiting |
| **Frontend** | `erp-frontend` | Next.js + TypeScript | Subscription-aware UI |
| **Shared Libraries** | `erp-shared-libs` | Python + Go | Common RBAC & utilities |
| **Infrastructure** | `erp-infrastructure` | Terraform + K8s | IaC & deployment configs |

## 🛠️ Dev-First Technology Stack

### Development Experience
- **Setup**: `make up` - One command starts all 14 services
- **Hot Reloading**: Django dev server, Air (Go), Next.js fast refresh
- **Local AI**: Ollama with pre-downloaded models (llama2, codellama, mistral)
- **Seeded Data**: Realistic test data across all services
- **Debugging**: Integrated logging and monitoring

### Business Services (Django + DRF)
- **Language**: Python 3.11+ with Django 4.2+
- **API**: Django REST Framework with shared RBAC decorators
- **Database**: PostgreSQL per service with tenant isolation
- **Testing**: pytest with shared fixtures and utilities
- **Hot Reload**: Django development server with auto-reload

### Infrastructure Services (Go)
- **Language**: Go 1.21+ with Chi router
- **Services**: Auth, Notifications, File Management, API Gateway
- **Hot Reload**: Air for automatic binary rebuilds
- **Performance**: High-throughput request handling
- **Deployment**: Lightweight Docker containers

### AI Platform (FastAPI + Local LLM)
- **Framework**: FastAPI with async support
- **Local LLM**: Ollama (llama2:7b, codellama:7b, mistral:7b)
- **Vector DB**: Qdrant for embeddings and RAG
- **RAG Engine**: LangChain with permission-aware retrieval
- **Development**: Hot reloading with FastAPI dev server

### Frontend (Next.js + Subscription-Aware UI)
- **Framework**: Next.js 14+ with App Router
- **Language**: TypeScript with strict mode
- **Styling**: Tailwind CSS with design system
- **State**: Zustand + React Query for server state
- **Components**: Subscription-aware HOCs and RBAC components

### Development Infrastructure
- **Orchestration**: Docker Compose for local development
- **Database**: PostgreSQL per service with shared connection pooling
- **Cache**: Redis for sessions, caching, and pub/sub
- **Message Queue**: Redis for background jobs and events
- **Monitoring**: Prometheus + Grafana for metrics

## 📚 Documentation Structure

```
📁 intelligent-erp-suite-specs/
├── 📂 .kiro/                   # Kiro IDE specifications
├── 📂 02-architecture/         # System architecture & API design
│   ├── �  api-endpoints.md     # REST API specifications
│   └── � SAAS_AsRCHITECTURE_UPDATES.md # SaaS transformation details
├── 📂 03-implementation/       # Development guides & patterns
│   ├── � d4ev-first-setup.md   # One-command development setup
│   ├── � rb-ac-patterns.md     # Shared RBAC utilities
│   ├── � 0docker-compose-dev.md # 14-service orchestration
│   └── � infrastrcucture-makefile.md # Development commands
├── 📂 05-deployment/           # Deployment strategies
│   └── � multni-repo-microservices-plan.md # 14-repository deployment
├── 📂 modules/                 # Service specifications
│   ├── 📄 MODULE_SPECIFICATIONS.md # All 14 services overview
│   └── � M[service-folders]/   # Individual service specs
├── 📄 MODULAR_DEVELOPMENT_PLAN.md # Multi-repository development strategy
├── 📄 ai-chatbot-design.md     # AI platform & RAG implementation
├── 📄 application-flow.md      # Request flow across services
└── 📄 README.md                # This file
```

### 📋 Quick Navigation

| Section | Description | Key Documents |
|---------|-------------|---------------|
| **[Dev-First Setup](./03-implementation/dev-first-setup.md)** | `make up` - Start all 14 services instantly | Hot reloading & seeded data |
| **[Multi-Repository Plan](./MODULAR_DEVELOPMENT_PLAN.md)** | 14-service development strategy | Independent development & deployment |
| **[Service Specifications](./modules/MODULE_SPECIFICATIONS.md)** | All 14 services overview | Tech stacks & dependencies |
| **[RBAC Patterns](./03-implementation/rbac-patterns.md)** | Shared authentication utilities | Common auth & tenant isolation |
| **[SaaS Architecture](./02-architecture/SAAS_ARCHITECTURE_UPDATES.md)** | Subscription-driven design | Feature gating & billing integration |
| **[AI Platform](./ai-chatbot-design.md)** | RAG chatbot & local LLM | Permission-aware AI assistance |
| **[Deployment](./05-deployment/multi-repo-microservices-plan.md)** | 14-repository deployment | Independent service deployment |

## 🎯 Getting Started

### For Developers (5-Minute Setup)
```bash
# Clone the infrastructure repository
git clone https://github.com/your-org/erp-infrastructure
cd erp-infrastructure

# Start all 14 services with one command
make up

# What you get:
# ✅ All services running with hot reloading
# ✅ Seeded databases with realistic test data  
# ✅ Local LLM (Ollama) ready for AI development
# ✅ API documentation at http://localhost:8080/docs
# ✅ Frontend at http://localhost:3000
```

### For Project Managers
1. 📖 Read [Multi-Repository Development Plan](./MODULAR_DEVELOPMENT_PLAN.md)
2. � RReview [14-Service Architecture](./modules/MODULE_SPECIFICATIONS.md)
3. � Ctheck [SaaS Foundation Timeline](./02-architecture/SAAS_ARCHITECTURE_UPDATES.md)
4. �  Monitor independent service development progress

### For Architects & Tech Leads
1. 🏗️ Study [14-Repository Architecture](./05-deployment/multi-repo-microservices-plan.md)
2. 🔧 Review [Shared RBAC Patterns](./03-implementation/rbac-patterns.md)
3. 🚀 Examine [Dev-First Setup](./03-implementation/dev-first-setup.md)
4. 🛠️ Plan [Service Integration Strategies](./application-flow.md)

### For DevOps Engineers
1. 🛠️ Review [Docker Compose Development](./03-implementation/docker-compose-dev.md)
2. 🚀 Study [Multi-Repository Deployment](./05-deployment/multi-repo-microservices-plan.md)
3. 📁 Check [Infrastructure Makefile](./03-implementation/infrastructure-makefile.md)
4. ⚙️ Set up [14-Service Orchestration](./03-implementation/dev-first-setup.md)

## 📊 Project Metrics

- **🏗️ Architecture**: 14 independent repositories and microservices
- **� Dev Expetrience**: 5-minute onboarding with `make up`
- **🔨 Implementation**: Service-by-service development with shared patterns
- **👥 Team Structure**: Independent teams per service with shared utilities
- **⏱️ Timeline**: 19-month development cycle (SaaS-first approach)
- **🧪 Testing**: >90% coverage with shared testing utilities
- **📈 SaaS Metrics**: Subscription conversion, usage tracking, feature adoption

## 🤝 Contributing

This is a specification repository. For implementation contributions:

1. **Requirements Changes**: Submit issues with business justification
2. **Architecture Updates**: Propose changes through architecture review board
3. **Implementation Feedback**: Update task specifications based on development learnings
4. **Documentation**: Keep specifications synchronized with implementation

## 📄 License

This project specification is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🔗 14-Repository Architecture

### 🏗️ Infrastructure & Shared
- **`erp-infrastructure`** - Terraform, K8s, Docker Compose, `make up` setup
- **`erp-shared-libs`** - Common RBAC utilities, auth decorators, testing patterns
- **`erp-api-gateway`** - Go-based request routing, rate limiting, auth validation

### 🔐 SaaS Foundation
- **`erp-auth-service`** - Go + JWT multi-tenant authentication & RBAC
- **`erp-subscription-service`** - Django + Stripe subscription & billing management

### 💼 Business Services (Django + DRF)
- **`erp-crm-service`** - Contact & lead management with subscription gating
- **`erp-hrm-service`** - Employee & leave management with RBAC
- **`erp-accounting-service`** - Financial management & invoicing
- **`erp-inventory-service`** - Product & stock management
- **`erp-project-service`** - Task & resource management

### 🤖 AI & Support Services
- **`erp-ai-platform`** - FastAPI + Ollama RAG chatbot & ML services
- **`erp-notification-service`** - Go-based email, SMS, WebSocket notifications
- **`erp-file-service`** - Go-based document management & PDF generation

### 🎨 Frontend
- **`erp-frontend`** - Next.js with subscription-aware UI & RBAC components

## 📞 Support & Contact

- **Documentation Issues**: Create an issue in this repository
- **Architecture Questions**: Contact the Architecture Review Board
- **Implementation Support**: Reach out to respective development teams
- **Business Requirements**: Contact Product Management team

---

**Built with ❤️ for modern enterprises seeking intelligent automation**