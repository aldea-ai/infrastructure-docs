# Aldea Infrastructure Documentation

Visual diagrams explaining Aldea's cloud infrastructure architecture.

## Quick Links

| Diagram | What It Shows |
|---------|---------------|
| [ECS Architecture](diagrams/ecs-architecture.md) | **Current** production setup using AWS ECS |
| [Monitoring & Observability](diagrams/monitoring-observability.md) | Prometheus, Grafana, alerting, and metrics |
| [EKS Architecture](diagrams/eks-architecture.md) | **Previous** Kubernetes-based setup (historical) |
| [Migration Comparison](diagrams/eks-to-ecs-comparison.md) | Side-by-side comparison of EKS vs ECS |

## What Does Aldea Do?

Aldea provides speech-to-text (ASR) services. Users connect to:

- **api.aldea.ai** - Real-time audio transcription
- **backend.aldea.ai** - Account management API
- **platform.aldea.ai** - Web dashboard

## Architecture at a Glance

```
Your App → api.aldea.ai → Load Balancer → ECS Services → Speech Engine
                                              ↓
                                          Redis Cache
                                          (API keys)
```

### Monitoring

```
Prometheus → Blackbox → STT Servers
     ↓
  Grafana → Alerts → Slack
     ↓
CloudWatch ← EMF Metrics ← ECS Services
```

## Viewing the Diagrams

The diagrams use [Mermaid.js](https://mermaid.js.org/) and render automatically on GitHub.

You can also:
- Use VS Code with "Markdown Preview Mermaid Support" extension
- Paste diagram code at https://mermaid.live

## Key Takeaways

1. **Current Architecture (ECS)**: Simple, serverless, AWS-native
2. **Previous Architecture (EKS)**: Complex, Kubernetes-based
3. **Why We Migrated**: Reduced operational complexity

## Related Repositories

| Repo | Purpose |
|------|---------|
| [aldea-proxy-services](https://github.com/aldea-ai/aldea-proxy-services) | Proxy service code |
| [aldea-iac-platform](https://github.com/aldea-ai/aldea-iac-platform) | Terraform infrastructure |
| [aldea-observability-platform](https://github.com/aldea-ai/aldea-observability-platform) | Grafana dashboards and alerts |
| [imp-backend](https://github.com/aldea-ai/imp-backend) | Platform API |
| [imp-frontend](https://github.com/aldea-ai/imp-frontend) | Web dashboard |
