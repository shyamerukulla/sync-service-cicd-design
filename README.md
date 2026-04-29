# sync-service — CI/CD Pipeline

This repository contains the pipeline definition and deployment infrastructure for `sync-service`, a Spring Boot + MongoDB backend deployed to GCP VMs.

## Repository Structure

```
.
├── Jenkinsfile              # Full CI/CD pipeline
├── docs/
│   └── CICD_DESIGN.md       # Full design document
├── ansible/
│   ├── deploy.yml           # Blue/Green deploy playbook
│   ├── inventory/
│   │   ├── qa.ini
│   │   ├── staging.ini
│   │   └── prod.ini
│   └── templates/
│       └── nginx_upstream.conf.j2
└── scripts/
    ├── health_check.sh
    └── rollback.sh
```

## Quick Reference

| Branch | Environment | Deploy Trigger |
|---|---|---|
| `feature/*` | — | PR: build + test only |
| `develop` | QA | Auto on merge |
| `release/*` | Staging | Auto on merge |
| `main` | Production | Manual approval required |

## Jenkins Credentials Required

| Credential ID | Type | Description |
|---|---|---|
| `gcp-project-id` | Secret text | GCP project ID |
| `gcp-sa-qa` | Secret file | GCP service account JSON (QA) |
| `gcp-sa-staging` | Secret file | GCP service account JSON (Staging) |
| `gcp-sa-prod` | Secret file | GCP service account JSON (Prod) |
| `mongo-uri-qa` | Secret text | MongoDB URI for QA |
| `mongo-uri-staging` | Secret text | MongoDB URI for Staging |
| `mongo-uri-prod` | Secret text | MongoDB URI for Prod |

## Design Document

See [`docs/CICD_DESIGN.md`](docs/CICD_DESIGN.md) for the full design covering:
- Branching strategy
- Pipeline stages
- Configuration & secrets management
- Blue/Green deployment rationale

## Part 2 — Infrastructure Design

See [part2-infrastructure/INFRASTRUCTURE_DESIGN.md](part2-infrastructure/INFRASTRUCTURE_DESIGN.md) for the full design covering:

- Compute choice (Cloud Run over GKE/GCE)
- MongoDB Atlas hosting approach
- VPC networking and private DB access
- Secrets & IAM setup
- Logging & monitoring stack

### Architecture Diagram

![Architecture Diagram](part2-infrastructure/architecture-diagram.png)
