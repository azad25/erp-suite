# Helm Charts Structure for Intelligent ERP Suite

## 📦 Helm Chart Architecture

This document outlines the comprehensive Helm chart structure for deploying the Intelligent ERP Suite on Kubernetes.

## 🏗️ Chart Structure Overview

```
helm-charts/
├── erp-suite/                          # Main umbrella chart
│   ├── Chart.yaml
│   ├── values.yaml                     # Default values
│   ├── values-dev.yaml                 # Development overrides
│   ├── values-staging.yaml             # Staging overrides
│   ├── values-prod.yaml                # Production overrides
│   ├── templates/
│   │   ├── _helpers.tpl                # Template helpers
│   │   ├── configmap.yaml              # Global configuration
│   │   ├── secret.yaml                 # Global secrets
│   │   ├── ingress.yaml                # Ingress configuration
│   │   ├── networkpolicy.yaml          # Network policies
│   │   └── tests/                      # Helm tests
│   │       ├── test-connection.yaml
│   │       └── test-api-health.yaml
│   └── charts/                         # Sub-charts
│       ├── auth-service/
│       ├── subscription-service/
│       ├── crm-service/
│       ├── hrm-service/
│       ├── accounting-service/
│       ├── inventory-service/
│       ├── project-service/
│       ├── ai-platform/
│       ├── api-gateway/
│       ├── frontend/
│       ├── databases/
│       └── monitoring/
└── shared/                             # Shared chart library
    ├── Chart.yaml
    └── templates/
        ├── _deployment.tpl
        ├── _service.tpl
        ├── _configmap.tpl
        ├── _secret.tpl
        ├── _ingress.tpl
        └── _helpers.tpl
```

This comprehensive Helm chart structure provides a scalable, maintainable deployment strategy for the Intelligent ERP Suite on Kubernetes.