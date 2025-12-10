# Aldea Infrastructure Documentation

Infrastructure architecture diagrams and documentation for Aldea's cloud infrastructure.

## Overview

This repository contains Mermaid.js diagrams documenting Aldea's infrastructure architecture, including:

- **Current ECS Architecture** - AWS ECS Fargate-based infrastructure
- **Previous EKS Architecture** - AWS EKS Kubernetes-based infrastructure (historical reference)
- **Migration Comparison** - Side-by-side comparison of EKS to ECS migration

## Diagrams

### Current Production Architecture

| Document | Description |
|----------|-------------|
| [ECS Architecture](diagrams/ecs-architecture.md) | Current ECS Fargate infrastructure for ASR services |

### Historical Reference

| Document | Description |
|----------|-------------|
| [EKS Architecture](diagrams/eks-architecture.md) | Previous EKS/Kubernetes infrastructure |

### Migration Documentation

| Document | Description |
|----------|-------------|
| [EKS to ECS Comparison](diagrams/eks-to-ecs-comparison.md) | Detailed comparison of migration changes |

## Viewing Diagrams

The diagrams use [Mermaid.js](https://mermaid.js.org/) syntax. You can view them:

1. **GitHub** - GitHub natively renders Mermaid diagrams in markdown files
2. **VS Code** - Install the "Markdown Preview Mermaid Support" extension
3. **Mermaid Live Editor** - Paste diagram code at https://mermaid.live

## Infrastructure Summary

### Current Architecture (ECS)

```
Users → Route53 → Global Accelerator → WAF → ALB → ECS Fargate Services
                                                  ├── ws-proxy (WebSocket)
                                                  ├── http-proxy (HTTP)
                                                  ├── backend (API)
                                                  └── frontend (UI)
```

**Key Components:**
- AWS ECS Fargate (serverless containers)
- ElastiCache Redis (API key validation, rate limiting)
- RDS PostgreSQL (application data)
- Global Accelerator (anycast IPs)
- AWS WAF (security)

### Environments

| Environment | Cluster | Domain |
|-------------|---------|--------|
| Production | prod-ws-proxy | api.aldea.ai |
| Dev | aldea-ecs-dev | dev-api.aldea.ai |

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [aldea-proxy-services](https://github.com/aldea-ai/aldea-proxy-services) | Proxy service source code |
| [aldea-eks-platform](https://github.com/aldea-ai/aldea-eks-platform) | Terraform infrastructure code |
| [imp-backend](https://github.com/aldea-ai/imp-backend) | Platform API backend |
| [imp-frontend](https://github.com/aldea-ai/imp-frontend) | Platform web frontend |

## Contributing

When updating diagrams:

1. Use Mermaid.js syntax
2. Test diagrams render correctly before committing
3. Keep diagrams focused on specific aspects (networking, services, CI/CD, etc.)
4. Update this README if adding new diagram files
