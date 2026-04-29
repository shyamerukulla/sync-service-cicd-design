# Part 2 – Infrastructure Design
## `sync-service` on GCP — Architecture & Key Decisions

---

## Overview

This document describes the proposed GCP infrastructure for the `sync-service` Spring Boot application. The design prioritises **zero-downtime deployments**, **secure private networking**, **cost efficiency for a startup**, and **minimal operational overhead**.

---

## 1. Compute Choice — Cloud Run (over GKE / Compute Engine)

### Decision: **Cloud Run (fully managed)**

| Option | Verdict | Reason |
|---|---|---|
| Compute Engine VMs | ❌ Rejected | Already in use; requires manual scaling, patching, provisioning. Highest ops overhead. |
| GKE (Kubernetes) | ❌ Rejected for now | Powerful, but expensive cluster management overhead at startup scale. Adds complexity without payoff until 10+ services. |
| **Cloud Run** | ✅ Chosen | Serverless containers. Auto-scales to zero (QA) or a configured minimum (prod). Pay per request. Zero cluster management. Spring Boot JARs containerise trivially. |

### Cloud Run configuration per environment

| Environment | Min Instances | Max Instances | CPU | Memory |
|---|---|---|---|---|
| `qa` | 0 | 3 | 1 vCPU | 512 Mi |
| `staging` | 1 | 5 | 1 vCPU | 1 Gi |
| `prod` | 2 | 20 | 2 vCPU | 2 Gi |

- **`min-instances > 0` for prod/staging** avoids cold-start latency.
- **Traffic splitting** on Cloud Run enables blue/green and canary rollouts natively without extra infrastructure.
- Each environment is a **separate Cloud Run service** in the same GCP project but tagged to a separate VPC Connector and Secret Manager path.

---

## 2. MongoDB Hosting — MongoDB Atlas (GCP-peered)

### Decision: **MongoDB Atlas on GCP (same region as Cloud Run)**

**Why not self-hosted MongoDB on GCE?**

Managing MongoDB yourself means handling replica-set configuration, backups, patching, monitoring, and failover — significant toil for a startup. Atlas provides all of this managed.

**Why not Cloud Firestore or Bigtable?**

`sync-service` already uses MongoDB; switching datastores is out of scope and carries migration risk.

### Atlas cluster sizing per environment

| Environment | Atlas Tier | Notes |
|---|---|---|
| `qa` | M10 (shared) | Lowest cost; acceptable for functional testing |
| `staging` | M20 (dedicated) | Closer to prod behaviour for perf testing |
| `prod` | M30 — 3-node replica set | HA, automatic failover, daily backups, point-in-time restore |

### Network path: Private Service Connect

Cloud Run services connect to Atlas **via VPC Connector + Private Service Connect**, so no traffic crosses the public internet. Atlas is configured to accept connections only from the GCP project's private IP range via a **GCP network peering** (Atlas native feature). MongoDB credentials are never in the container image — they are injected at runtime from Secret Manager (see §5).

---

## 3. Networking — VPC, Ingress, Egress

### VPC Layout

```
GCP Project
└── VPC: 10.0.0.0/16
    ├── Subnet: 10.0.1.0/24  — Cloud Run VPC Connector (us-central1)
    └── Subnet: 10.0.2.0/24  — Reserved for future GKE migration
```

### Ingress path

```
Client → Cloud Armor (WAF, DDoS, IP allowlist) 
       → HTTPS Global Load Balancer (SSL termination, cert managed by GCP)
       → Cloud Run service (env routing via URL mask or custom domain)
```

- **Cloud Armor** enforces rate limiting and geo-blocking before traffic hits application code.
- The Load Balancer handles TLS; Cloud Run receives plain HTTP internally.
- **Custom domains** per environment: `api-qa.example.com`, `api-staging.example.com`, `api.example.com` — mapped via Cloud Run domain mappings.

### Egress path (Cloud Run → MongoDB Atlas)

Cloud Run is **serverless** and does not run inside the VPC by default. A **Serverless VPC Access Connector** bridges Cloud Run to the VPC, enabling private RFC-1918 routing to Atlas via Private Service Connect.

- Cloud Run → VPC Connector → Private Service Connect → Atlas private endpoint
- **Cloud NAT** provides a **static egress IP** for any external API calls the service makes, enabling IP-allowlisting on third-party APIs.

### No public MongoDB ports

Atlas cluster is configured with:
- **Network peering** to GCP project VPC only
- **IP Access List** restricted to VPC Connector's private CIDR
- No public endpoint exposed

---

## 4. Secrets & IAM

### Secrets — Google Secret Manager

All sensitive values are stored in Secret Manager, **not in environment variables baked into the image** or source code.

| Secret name | Contents | Access |
|---|---|---|
| `sync-service-qa-mongo-uri` | MongoDB Atlas connection string (QA) | QA service account only |
| `sync-service-staging-mongo-uri` | MongoDB Atlas connection string (staging) | Staging SA only |
| `sync-service-prod-mongo-uri` | MongoDB Atlas connection string (prod) | Prod SA only |
| `sync-service-prod-api-key` | Third-party API key | Prod SA only |

Secrets are mounted as environment variables at Cloud Run startup via the `--set-secrets` flag in the deploy command. They are **never logged** (Spring Boot's `spring.config.import=sm://` or explicit env injection both work).

### IAM — Workload Identity / per-environment service accounts

```
sync-service-qa@PROJECT.iam.gserviceaccount.com
  roles/secretmanager.secretAccessor  (qa secrets only, via condition)
  roles/run.invoker

sync-service-staging@PROJECT.iam.gserviceaccount.com
  roles/secretmanager.secretAccessor  (staging secrets only)
  roles/run.invoker

sync-service-prod@PROJECT.iam.gserviceaccount.com
  roles/secretmanager.secretAccessor  (prod secrets only)
  roles/run.invoker

ci-deploy@PROJECT.iam.gserviceaccount.com  (Jenkins deploy SA)
  roles/run.developer
  roles/artifactregistry.writer
  roles/iam.serviceAccountUser        (to deploy as env SAs)
```

**No human users have production deploy permissions** — only the Jenkins CI service account does, and that SA is short-lived-token based (Workload Identity Federation with the Jenkins GCP plugin).

---

## 5. Container Image Management — Artifact Registry

All Docker images are pushed to **Google Artifact Registry** (preferred over Container Registry, which is legacy).

```
us-central1-docker.pkg.dev/PROJECT/sync-service/sync-service:GIT_SHA
```

- Images are **tagged by Git SHA**, not `latest`, to guarantee reproducibility.
- The Jenkins pipeline pushes to the registry and the deploy step references the exact SHA.
- **Vulnerability scanning** is enabled on the registry (Container Analysis API); critical CVEs block promotion to prod via a pipeline gate.

---

## 6. Auto-scaling

Cloud Run auto-scaling is **request-based** out of the box:

- Scales out when concurrent requests per instance exceed the **concurrency limit** (set to 80 for this Spring Boot service).
- Scales in after a configurable cooldown window.
- **`--min-instances 2`** for prod ensures no cold starts under load.
- **`--max-instances 20`** caps spend; set a billing alert at 70% of expected max.

For the Spring Boot app specifically:
- Set `server.tomcat.threads.max=200` to handle concurrency within each instance.
- Enable Cloud Run's **CPU is always allocated** option for prod (prevents throttling between requests on warm instances).

---

## 7. Deployment Strategy — Blue/Green via Cloud Run Traffic Splitting

Cloud Run has **built-in traffic splitting**, making blue/green essentially free:

```bash
# Deploy new revision (receives 0% traffic)
gcloud run deploy sync-service-prod \
  --image us-central1-docker.pkg.dev/PROJECT/sync-service/sync-service:NEW_SHA \
  --no-traffic

# Smoke test the new revision directly via its URL
curl https://NEW_REVISION_URL/actuator/health

# Shift traffic (canary: 10% first, then 100%)
gcloud run services update-traffic sync-service-prod \
  --to-revisions NEW_SHA=10,OLD_SHA=90

# After validation, full cutover
gcloud run services update-traffic sync-service-prod \
  --to-revisions NEW_SHA=100
```

**Rollback** is instant — one command re-routes 100% traffic to the previous revision, which is still running.

---

## 8. Logging & Monitoring

### Logging — Cloud Logging (structured JSON)

Configure Spring Boot to emit **structured JSON logs** using Logback + `logstash-logback-encoder`:

```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

Cloud Run captures `stdout` and forwards to Cloud Logging automatically. JSON logs are parsed natively — fields like `severity`, `traceId`, and `spanId` are indexed for querying.

**Log-based metrics**: Create log-based metrics for `ERROR` severity count to drive alerting policies.

### Metrics — Cloud Monitoring

- Spring Boot Actuator exposes `/actuator/prometheus` — scraped by a **Cloud Monitoring Prometheus sidecar** (or the managed `google-cloud-monitoring` metrics export).
- Key metrics to alert on:
  - `run.googleapis.com/request_latencies` p99 > 2 s → page on-call
  - `run.googleapis.com/container/instance_count` near max → notify
  - `custom/sync_service/error_rate` > 1% → page on-call
  - MongoDB Atlas connection pool exhaustion (Atlas metric forwarded via Atlas Data Federation → Cloud Monitoring)

### Tracing — Cloud Trace

Add `spring-cloud-gcp-starter-trace` to `pom.xml`. Traces are auto-exported to Cloud Trace and correlated with Cloud Logging entries via the `traceId` field.

### Uptime checks

Cloud Monitoring **uptime checks** on `https://api.example.com/actuator/health` every 60 seconds from multiple GCP regions. An alerting policy notifies the on-call channel (PagerDuty or Slack webhook) if the check fails from 2+ regions.

---

## 9. Cost Estimate (Startup)

| Component | Est. Monthly Cost |
|---|---|
| Cloud Run (prod, ~10M req/mo, 2 min instances) | ~$60–120 |
| Cloud Run (qa + staging, low traffic) | ~$10–20 |
| MongoDB Atlas M30 (prod) | ~$200 |
| MongoDB Atlas M10+M20 (qa/staging) | ~$60 |
| Cloud Armor | ~$20 |
| Secret Manager | ~$2 |
| Artifact Registry + logging storage | ~$15 |
| **Total estimate** | **~$370–440/mo** |

Cost is controlled by:
- Cloud Run scaling to zero on QA overnight.
- Atlas M10 shared tier for QA.
- Log retention policy: 30 days hot, 90 days cold (Cloud Storage archive tier).

---

## 10. Future Migration Path

If the service grows and Kubernetes becomes necessary:

1. Cloud Run → **GKE Autopilot** (minimal cluster management, same container images, same CI pipeline).
2. Atlas → retain as-is (Atlas works equally well with GKE via private endpoint).
3. Networking — same VPC, replace VPC Connector with direct GKE pod CIDR peering to Atlas.

The Cloud Run architecture is explicitly designed so that the container image and deployment pipeline require **no changes** for this migration — only the `gcloud run deploy` commands are replaced with `kubectl apply`.

---

## Summary of Key Choices

| Concern | Choice | Why |
|---|---|---|
| Compute | Cloud Run | Serverless, auto-scaling, zero cluster ops, pay-per-request |
| Database | MongoDB Atlas (GCP peer) | Managed, HA in prod, startup-friendly pricing on lower tiers |
| Networking | VPC + Private Service Connect | No public DB exposure, static egress IP for third-party allowlisting |
| Secrets | Secret Manager | Audit trail, per-env isolation, no secrets in code or image |
| IAM | Per-env service accounts + Workload Identity | Least-privilege; no long-lived keys |
| Deployment | Cloud Run traffic splitting (blue/green) | Instant rollback, no extra infra, canary support built-in |
| Observability | Cloud Logging + Monitoring + Trace | Fully managed, native GCP integration, no self-hosted stack |
| Images | Artifact Registry (SHA-tagged) | Reproducible deploys, built-in vulnerability scanning |

