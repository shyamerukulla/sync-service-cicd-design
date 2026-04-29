# CI/CD Pipeline Design — `sync-service`

> **Service:** Spring Boot backend (`sync-service`) connected to MongoDB  
> **Infrastructure:** GCP VMs  
> **Environments:** `qa` → `staging` → `prod`  
> **Build Tool:** Jenkins

---

## 1. Branching Strategy

### Branch → Environment Mapping

```
feature/*  ──► (no deploy) PR build + unit tests only
               │
               ▼
develop    ──► QA environment        (auto-deploy on merge)
               │
               ▼
release/*  ──► Staging environment   (auto-deploy on merge to release branch)
               │
               ▼
main       ──► Production            (manual approval gate required)
```

| Branch | Environment | Trigger | Gate |
|---|---|---|---|
| `feature/*` | — | PR opened/updated | Lint + Test must pass |
| `develop` | QA | Merge to `develop` | Automated |
| `release/*` | Staging | Merge to `release/*` | Automated |
| `main` | Production | Merge to `main` | **Manual approval** |

### Preventing Accidental Prod Deployments

1. **Branch protection on `main`:** No direct pushes. All changes via PR from `release/*` only.
2. **Required reviewers:** Minimum 2 approvals for any PR targeting `main`.
3. **Jenkins input step:** Even after a merge to `main`, the pipeline pauses at the `Deploy to Prod` stage and requires explicit sign-off from a designated approver in the Jenkins UI.
4. **Environment variable guard:** Pipeline reads `DEPLOY_ENV` and validates it matches the branch. A mismatch aborts the run immediately.
5. **Separate credentials:** Prod GCP service account is stored in a distinct Jenkins credential ID (`gcp-sa-prod`) that only the `main` branch pipeline is authorized to access.

---

## 2. Jenkins Pipeline

### High-Level Stages

```
Checkout → Build → Unit Test → Code Quality → Docker Build
    → Push to Registry → Deploy → Smoke Test → [Notify]
```

### What Happens on PR vs Merge

#### On Pull Request (feature/* → develop, or release/* → main)
- Checkout code
- Compile with Maven (`mvn package -DskipTests`)
- Run unit + integration tests (`mvn verify`)
- SonarQube code quality scan
- **No Docker build. No deployment.**
- Report status back to GitHub — PR is blocked if any stage fails.

#### On Merge to `develop` / `release/*` / `main`
- All PR stages, plus:
- Docker image build and tag with Git SHA + env label
- Push image to Google Artifact Registry (GAR)
- Deploy to the mapped environment
- Run smoke tests (HTTP health check on `/actuator/health`)
- Notify Slack/email on success or failure

### Rollback Strategy

**Automated rollback on deploy failure:**

If the `Deploy` stage fails (health check does not return `200 OK` within the timeout window), Jenkins automatically:

1. Identifies the previously running image tag stored in `deploy-manifest.txt` on the VM.
2. Re-runs the deploy script with the previous image tag.
3. Verifies health check passes on the rolled-back version.
4. Marks the build as `UNSTABLE` and sends an alert.

```groovy
// Pseudocode captured in Jenkinsfile
def rollback(env, previousTag) {
    sh "ansible-playbook deploy.yml -e env=${env} -e image_tag=${previousTag}"
}
```

**Manual rollback:**
Any previous successful build in Jenkins can be replayed. The image tag is derived from the Git commit SHA, so it is always traceable and re-deployable.

---

## 3. Configuration Management

### Environment-Specific Configs

The service uses **Spring profiles** (`application-qa.yml`, `application-staging.yml`, `application-prod.yml`) stored in a **separate private Git repo** (`sync-service-config`). This follows the Config Server pattern without necessarily running a full Spring Cloud Config Server.

At deploy time, the pipeline:
1. Checks out the config repo.
2. Copies the correct profile file to the VM.
3. Passes `--spring.profiles.active=<env>` as a JVM argument on startup.

This keeps config out of the application Docker image and out of the main source repo.

### Secrets Handling

**Tool: Google Secret Manager** (native GCP, no extra infrastructure needed on GCP VMs)

| Secret | Key Name in GSM | Who Reads It |
|---|---|---|
| MongoDB URI | `sync-service-mongo-uri-<env>` | Jenkins pipeline (injects as env var) |
| MongoDB Password | `sync-service-mongo-password-<env>` | Jenkins pipeline |
| API Keys | `sync-service-api-key-<env>` | Jenkins pipeline |

**Pipeline behavior:**

```groovy
withCredentials([
    string(credentialsId: "mongo-uri-${ENV}", variable: 'MONGO_URI')
]) {
    sh 'ansible-playbook deploy.yml -e mongo_uri=$MONGO_URI'
}
```

The secret is passed as a runtime environment variable to the JVM — it is **never written to disk**, never printed in logs (Jenkins masks it), and never baked into the Docker image.

Jenkins itself stores its own service account key for GSM access in the Jenkins credential store (type: `Secret File`), not in any source file.

---

## 4. Deployment Strategy

### Chosen Strategy: **Blue/Green**

#### Why Blue/Green over the alternatives?

| Strategy | Downtime | Rollback Speed | Resource Cost | Chosen? |
|---|---|---|---|---|
| Recreate | Yes (brief) | Slow | Low | ✗ |
| Rolling | Near-zero | Medium | Low | ✗ |
| Blue/Green | **Zero** | **Instant** | Medium | **✓** |

**Reasoning:**

- `sync-service` connects to MongoDB. Rolling updates risk having two different versions of the service writing to the same DB simultaneously, which can cause schema/migration conflicts. Blue/Green avoids this: traffic only shifts after the new version is fully healthy.
- Rollback is instant — just point the load balancer back to the old (blue) environment.
- GCP VMs make Blue/Green straightforward: maintain two VM instance groups (blue and green) per environment, and swap the GCP Load Balancer backend.

#### Zero-Downtime Approach

```
                          ┌──────────────────┐
                          │  GCP Load Balancer│
                          └─────────┬────────┘
                                    │
              ┌─────────────────────┴──────────────────────┐
              │                                            │
     ┌────────▼─────────┐                       ┌─────────▼────────┐
     │   BLUE (live)    │                       │  GREEN (standby) │
     │  sync-service    │                       │  sync-service    │
     │  v1.4.2          │                       │  v1.5.0 (new)    │
     └──────────────────┘                       └──────────────────┘
```

**Deploy sequence:**

1. Deploy new image to the **green** (standby) VM group.
2. Run health checks against green directly (bypassing the LB).
3. Run smoke tests and DB migration checks.
4. If healthy → shift 100% of LB traffic to green. Green becomes the new "live."
5. Keep blue running for 15 minutes as instant rollback window.
6. After window: tear down or keep blue for next deployment cycle.

If step 3 fails → traffic never shifts, blue stays live, green is recycled. Zero user impact.

---

## 5. Folder Structure (Repository)

```
sync-service/
├── src/                         # Spring Boot source
├── Dockerfile
├── Jenkinsfile                  # Pipeline definition
├── ansible/
│   ├── deploy.yml               # Ansible playbook for VM deploy
│   └── inventory/
│       ├── qa.ini
│       ├── staging.ini
│       └── prod.ini
├── scripts/
│   ├── health_check.sh
│   └── rollback.sh
└── docs/
    └── CICD_DESIGN.md           # This document
```

---

## 6. Key Tools Summary

| Concern | Tool |
|---|---|
| Source control | GitHub |
| CI/CD orchestration | Jenkins |
| Container registry | Google Artifact Registry (GAR) |
| VM configuration | Ansible |
| Secrets | Google Secret Manager |
| Load balancing / traffic shift | GCP HTTP(S) Load Balancer |
| App config | Spring Profiles (separate config repo) |
| Code quality | SonarQube |
| Notifications | Slack (Jenkins Slack plugin) |

