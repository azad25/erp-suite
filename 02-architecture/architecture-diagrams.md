# High-Level Project Diagrams

## System Architecture Overview

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web App - Next.js/React]
        MOBILE[Mobile App - React Native]
        API_CLIENT[External API Clients]
    end
    
    subgraph "API Gateway Layer"
        GATEWAY[API Gateway - Kong/Nginx]
        AUTH_PROXY[Auth Proxy]
        RATE_LIMIT[Rate Limiter]
    end
    
    subgraph "Core Services Layer - Django"
        AUTH[Auth Service]
        TENANT[Tenant Service]
        USER[User Management]
        RBAC[RBAC Service]
    end
    
    subgraph "Business Services Layer - Django"
        HRM[HRM Service]
        CRM[CRM Service]
        FINANCE[Finance Service]
        INVENTORY[Inventory Service]
        PROJECTS[Projects Service]
        NOTIFICATIONS[Notification Service]
    end
    
    subgraph "Microservices Layer - Go"
        INVOICE[Invoice Generator]
        REPORTS[Report Generator]
        EMAIL[Email Service]
        FILE[File Processing]
        SEARCH[Search Service]
    end
    
    subgraph "AI Plugin Layer (Optional)"
        AI_GATEWAY[AI Gateway]
        RAG[RAG Engine]
        EMBEDDINGS[Embedding Service]
        LLM[LLM Provider]
    end
    
    subgraph "Data Layer"
        POSTGRES[(PostgreSQL)]
        MONGO[(MongoDB)]
        REDIS[(Redis)]
        VECTOR[(Vector DB)]
        ELASTIC[(Elasticsearch)]
    end
    
    subgraph "Message Layer"
        KAFKA[Apache Kafka]
        WEBSOCKET[WebSocket Server]
    end
    
    subgraph "External Integrations"
        GMAIL[Gmail API]
        DRIVE[Google Drive]
        MAPS[Google Maps]
        PAYMENT[Payment Gateways]
    end
    
    WEB --> GATEWAY
    MOBILE --> GATEWAY
    API_CLIENT --> GATEWAY
    
    GATEWAY --> AUTH_PROXY
    AUTH_PROXY --> RATE_LIMIT
    
    RATE_LIMIT --> AUTH
    RATE_LIMIT --> TENANT
    RATE_LIMIT --> HRM
    RATE_LIMIT --> CRM
    RATE_LIMIT --> FINANCE
    RATE_LIMIT --> INVENTORY
    RATE_LIMIT --> PROJECTS
    
    HRM --> INVOICE
    FINANCE --> REPORTS
    CRM --> EMAIL
    INVENTORY --> FILE
    
    AUTH --> POSTGRES
    HRM --> POSTGRES
    CRM --> POSTGRES
    FINANCE --> POSTGRES
    INVENTORY --> POSTGRES
    PROJECTS --> POSTGRES
    
    NOTIFICATIONS --> MONGO
    EMAIL --> MONGO
    REPORTS --> MONGO
    
    AUTH --> REDIS
    TENANT --> REDIS
    
    SEARCH --> ELASTIC
    
    AI_GATEWAY --> RAG
    RAG --> EMBEDDINGS
    RAG --> VECTOR
    RAG --> LLM
    
    HRM --> KAFKA
    CRM --> KAFKA
    FINANCE --> KAFKA
    INVENTORY --> KAFKA
    
    KAFKA --> INVOICE
    KAFKA --> REPORTS
    KAFKA --> EMAIL
    KAFKA --> NOTIFICATIONS
    
    EMAIL --> GMAIL
    FILE --> DRIVE
    CRM --> MAPS
    FINANCE --> PAYMENT
```

## Service Boundaries and Interactions

```mermaid
graph LR
    subgraph "Frontend Boundary"
        FE[React/Next.js App]
    end
    
    subgraph "API Boundary"
        GW[API Gateway]
        REST[REST APIs]
        GQL[GraphQL APIs]
        WS[WebSocket APIs]
    end
    
    subgraph "Core Business Boundary"
        AUTH[Authentication]
        RBAC[Authorization]
        TENANT[Multi-Tenancy]
        
        subgraph "Business Logic"
            HRM_BL[HRM Logic]
            CRM_BL[CRM Logic]
            FIN_BL[Finance Logic]
            INV_BL[Inventory Logic]
            PROJ_BL[Projects Logic]
        end
    end
    
    subgraph "Processing Boundary"
        subgraph "Sync Processing"
            CRUD[CRUD Operations]
            QUERY[Query Processing]
            VALID[Validation]
        end
        
        subgraph "Async Processing"
            EVENTS[Event Processing]
            JOBS[Background Jobs]
            NOTIF[Notifications]
        end
    end
    
    subgraph "Data Boundary"
        RDBMS[(Relational Data)]
        NOSQL[(Document Data)]
        CACHE[(Cache Layer)]
        SEARCH_IDX[(Search Index)]
        VECTOR_DB[(Vector Store)]
    end
    
    subgraph "Integration Boundary"
        GOOGLE[Google Services]
        PAYMENT_GW[Payment Gateways]
        AI_PLUGINS[AI Plugins]
    end
    
    FE --> GW
    GW --> REST
    GW --> GQL
    GW --> WS
    
    REST --> AUTH
    GQL --> AUTH
    WS --> AUTH
    
    AUTH --> RBAC
    RBAC --> TENANT
    
    TENANT --> HRM_BL
    TENANT --> CRM_BL
    TENANT --> FIN_BL
    TENANT --> INV_BL
    TENANT --> PROJ_BL
    
    HRM_BL --> CRUD
    CRM_BL --> CRUD
    FIN_BL --> CRUD
    INV_BL --> CRUD
    PROJ_BL --> CRUD
    
    CRUD --> RDBMS
    QUERY --> RDBMS
    VALID --> CACHE
    
    HRM_BL --> EVENTS
    CRM_BL --> EVENTS
    FIN_BL --> EVENTS
    
    EVENTS --> JOBS
    JOBS --> NOTIF
    
    NOTIF --> NOSQL
    JOBS --> NOSQL
    
    QUERY --> SEARCH_IDX
    
    HRM_BL --> GOOGLE
    CRM_BL --> GOOGLE
    FIN_BL --> PAYMENT_GW
    
    QUERY --> AI_PLUGINS
    AI_PLUGINS --> VECTOR_DB
```

## Data Flow Architecture

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant Auth
    participant BusinessService
    participant Database
    participant EventBus
    participant Microservice
    participant AI
    
    Client->>Gateway: HTTP Request
    Gateway->>Auth: Validate Token
    Auth->>Gateway: User Context + Permissions
    Gateway->>BusinessService: Authorized Request
    
    BusinessService->>Database: Query/Update Data
    Database->>BusinessService: Data Response
    
    BusinessService->>EventBus: Publish Domain Event
    EventBus->>Microservice: Event Notification
    Microservice->>Database: Process Event
    
    alt AI Query
        BusinessService->>AI: RAG Query with Context
        AI->>Database: Retrieve Relevant Data
        AI->>AI: Generate Response
        AI->>BusinessService: AI Response
    end
    
    BusinessService->>Gateway: Business Response
    Gateway->>Client: HTTP Response
    
    EventBus->>Client: WebSocket Notification (if subscribed)
```

## RAG Integration Architecture

```mermaid
graph TB
    subgraph "User Interface"
        CHAT[Chat Interface]
        CONTEXT[Context Panel]
    end
    
    subgraph "AI Gateway"
        ROUTER[Query Router]
        VALIDATOR[Permission Validator]
        FORMATTER[Response Formatter]
    end
    
    subgraph "RAG Pipeline"
        PARSER[Query Parser]
        RETRIEVER[Document Retriever]
        RANKER[Relevance Ranker]
        GENERATOR[Response Generator]
    end
    
    subgraph "Knowledge Base"
        EMBEDDER[Document Embedder]
        INDEXER[Vector Indexer]
        UPDATER[Real-time Updater]
    end
    
    subgraph "Data Sources"
        ERP_DATA[(ERP Database)]
        DOCS[(Document Store)]
        LOGS[(Activity Logs)]
    end
    
    subgraph "Vector Storage"
        VECTORS[(Vector Database)]
        METADATA[(Metadata Store)]
    end
    
    subgraph "LLM Layer"
        LOCAL[Local LLM]
        CLOUD[Cloud LLM]
        FALLBACK[Fallback Model]
    end
    
    CHAT --> ROUTER
    ROUTER --> VALIDATOR
    VALIDATOR --> PARSER
    
    PARSER --> RETRIEVER
    RETRIEVER --> VECTORS
    VECTORS --> RANKER
    
    RANKER --> GENERATOR
    GENERATOR --> LOCAL
    GENERATOR --> CLOUD
    GENERATOR --> FALLBACK
    
    LOCAL --> FORMATTER
    CLOUD --> FORMATTER
    FALLBACK --> FORMATTER
    
    FORMATTER --> CONTEXT
    
    ERP_DATA --> EMBEDDER
    DOCS --> EMBEDDER
    LOGS --> EMBEDDER
    
    EMBEDDER --> INDEXER
    INDEXER --> VECTORS
    INDEXER --> METADATA
    
    ERP_DATA --> UPDATER
    UPDATER --> VECTORS
```

## Plugin System Architecture

```mermaid
graph TB
    subgraph "Core ERP System"
        CORE[Core Services]
        HOOKS[Plugin Hooks]
        REGISTRY[Plugin Registry]
        LOADER[Plugin Loader]
    end
    
    subgraph "Plugin Interface Layer"
        API[Plugin API]
        AUTH_PLUGIN[Auth Interface]
        DATA_PLUGIN[Data Interface]
        EVENT_PLUGIN[Event Interface]
    end
    
    subgraph "AI Plugins"
        AI_MAIN[AI Main Plugin]
        RAG_PLUGIN[RAG Plugin]
        LLM_PLUGIN[LLM Plugin]
        EMBEDDING_PLUGIN[Embedding Plugin]
    end
    
    subgraph "Integration Plugins"
        GOOGLE_PLUGIN[Google Services Plugin]
        PAYMENT_PLUGIN[Payment Plugin]
        EMAIL_PLUGIN[Email Plugin]
        SMS_PLUGIN[SMS Plugin]
    end
    
    subgraph "Custom Plugins"
        CUSTOM1[Custom Plugin 1]
        CUSTOM2[Custom Plugin 2]
        THIRD_PARTY[Third Party Plugin]
    end
    
    subgraph "Plugin Isolation"
        SANDBOX[Sandbox Environment]
        RESOURCE_LIMIT[Resource Limits]
        SECURITY[Security Boundary]
    end
    
    CORE --> HOOKS
    HOOKS --> REGISTRY
    REGISTRY --> LOADER
    
    LOADER --> API
    API --> AUTH_PLUGIN
    API --> DATA_PLUGIN
    API --> EVENT_PLUGIN
    
    AUTH_PLUGIN --> AI_MAIN
    DATA_PLUGIN --> RAG_PLUGIN
    EVENT_PLUGIN --> LLM_PLUGIN
    
    AUTH_PLUGIN --> GOOGLE_PLUGIN
    DATA_PLUGIN --> PAYMENT_PLUGIN
    EVENT_PLUGIN --> EMAIL_PLUGIN
    
    AUTH_PLUGIN --> CUSTOM1
    DATA_PLUGIN --> CUSTOM2
    EVENT_PLUGIN --> THIRD_PARTY
    
    AI_MAIN --> SANDBOX
    GOOGLE_PLUGIN --> SANDBOX
    CUSTOM1 --> SANDBOX
    
    SANDBOX --> RESOURCE_LIMIT
    RESOURCE_LIMIT --> SECURITY
```

## Multi-Tenant Data Isolation

```mermaid
graph TB
    subgraph "Application Layer"
        APP[Application Request]
        TENANT_RESOLVER[Tenant Resolver]
        CONTEXT[Tenant Context]
    end
    
    subgraph "Database Schema Isolation"
        SHARED_DB[(Shared Database)]
        
        subgraph "Tenant A Schema"
            SCHEMA_A[tenant_a]
            USERS_A[users]
            ORDERS_A[orders]
            INVOICES_A[invoices]
        end
        
        subgraph "Tenant B Schema"
            SCHEMA_B[tenant_b]
            USERS_B[users]
            ORDERS_B[orders]
            INVOICES_B[invoices]
        end
        
        subgraph "Shared Schema"
            SHARED[shared]
            TENANTS[tenants]
            PLUGINS[plugins]
            SYSTEM[system_config]
        end
    end
    
    subgraph "Cache Isolation"
        REDIS_CACHE[(Redis)]
        TENANT_A_CACHE[tenant_a:*]
        TENANT_B_CACHE[tenant_b:*]
        SHARED_CACHE[shared:*]
    end
    
    subgraph "Search Isolation"
        ELASTIC_SEARCH[(Elasticsearch)]
        INDEX_A[tenant_a_index]
        INDEX_B[tenant_b_index]
    end
    
    subgraph "Vector DB Isolation"
        VECTOR_DB[(Vector Database)]
        COLLECTION_A[tenant_a_vectors]
        COLLECTION_B[tenant_b_vectors]
    end
    
    APP --> TENANT_RESOLVER
    TENANT_RESOLVER --> CONTEXT
    
    CONTEXT --> SCHEMA_A
    CONTEXT --> SCHEMA_B
    CONTEXT --> SHARED
    
    CONTEXT --> TENANT_A_CACHE
    CONTEXT --> TENANT_B_CACHE
    CONTEXT --> SHARED_CACHE
    
    CONTEXT --> INDEX_A
    CONTEXT --> INDEX_B
    
    CONTEXT --> COLLECTION_A
    CONTEXT --> COLLECTION_B
```

## Security and RBAC Flow

```mermaid
graph TB
    subgraph "Authentication Layer"
        LOGIN[User Login]
        JWT[JWT Token]
        REFRESH[Token Refresh]
        MFA[Multi-Factor Auth]
    end
    
    subgraph "Authorization Layer"
        RBAC_ENGINE[RBAC Engine]
        ROLES[Role Management]
        PERMISSIONS[Permission Management]
        POLICIES[Policy Engine]
    end
    
    subgraph "Resource Access"
        RESOURCE[Resource Request]
        GUARD[Permission Guard]
        FILTER[Data Filter]
        AUDIT[Audit Logger]
    end
    
    subgraph "Tenant Context"
        TENANT_ID[Tenant Identification]
        TENANT_SCOPE[Tenant Scoping]
        ISOLATION[Data Isolation]
    end
    
    LOGIN --> JWT
    JWT --> MFA
    MFA --> RBAC_ENGINE
    
    RBAC_ENGINE --> ROLES
    ROLES --> PERMISSIONS
    PERMISSIONS --> POLICIES
    
    RESOURCE --> GUARD
    GUARD --> RBAC_ENGINE
    RBAC_ENGINE --> FILTER
    FILTER --> AUDIT
    
    JWT --> TENANT_ID
    TENANT_ID --> TENANT_SCOPE
    TENANT_SCOPE --> ISOLATION
    
    ISOLATION --> FILTER
```

This architecture provides a comprehensive view of the enterprise ERP system with clear separation of concerns, scalable microservices architecture, optional AI integration, and robust security measures.