# ADR-0007: Forgejo as Version Control System and GitOps Platform

| Field         | Value                                                                 |
|---------------|-----------------------------------------------------------------------|
| ADR ID        | ADR-0007                                                              |
| Title         | Forgejo as Version Control System and GitOps Platform                 |
| File name     | adr-0007-forgejo-as-vcs-and-gitops-platform.md                        |
| Status        | Accepted                                                              |
| Date          | 2026-04-11                                                            |
| Deciders      | Engineering Lead                                                      |
| Supersedes    | ADR-0002 (Git-Based Single Source of Truth — platform selection left open) |
| Superseded by | —                                                                     |
| Related ADRs  | ADR-0002, ADR-0003, ADR-0004, ADR-0008                                |

---

## 1. Context and Problem Statement

The Corepixen pipeline requires a self-hosted Git platform that serves as the single authoritative source of truth for all code, infrastructure definitions, pipeline configurations, prompt templates, and deployment manifests. This is not a passive code store — it is the operational spine of the system. Infrastructure state is derived from repository contents via ArgoCD. CI/CD pipelines are executed on push events. Agent workflows are versioned and deployed from repository changes.

The platform must satisfy the following non-negotiable constraints established in ADR-0002, ADR-0003, and ADR-0006:

- Fully self-hosted with no mandatory external runtime dependencies
- Open source with credible long-term governance
- Zero per-seat or per-feature licensing cost
- GitHub Actions YAML-compatible CI/CD execution (Forgejo Actions)
- GitHub REST API-compatible surface so that tooling written against the GitHub API requires no modification
- Lightweight enough to coexist with all other services on a single Hetzner CX33 node (4 vCPU / 8 GB RAM)

---

## 2. Decision Drivers

- **ADR-0002 compliance**: Git must be the single source of truth. The platform must be reliable, durable, and under operator control at all times.
- **ADR-0003 compliance**: ArgoCD watches the repository for deployment manifest changes. The VCS must produce reliable, signed webhook events and expose a stable API.
- **ADR-0006 compliance**: Platform must be open source with no commercial feature gating. Governance must be transparent and independent.
- **GitHub Actions ecosystem compatibility**: CI/CD pipelines written in GitHub Actions YAML syntax must execute natively without runner re-configuration or syntax translation.
- **Resource efficiency**: The VCS shares the CX33 host with PostgreSQL, Redis, Prefect, LangGraph services, Crawl4AI, Playwright, MinIO, Grafana stack, and Caddy. Memory budget for the VCS is approximately 300 MB under normal load.
- **Webhook reliability**: All ArgoCD sync triggers, Prefect flow triggers, and downstream automation hooks depend on Forgejo webhook delivery. Missed or malformed webhooks directly degrade pipeline reliability.

---

## 3. Considered Options

- **Option A: Forgejo** *(selected)*
- **Option B: Gitea**
- **Option C: GitLab Community Edition**
- **Option D: GitHub (SaaS)**
- **Option E: Gogs**

---

## 4. Decision Outcome

**Chosen option: Forgejo (Option A)**

Forgejo is selected as the version control and GitOps platform for the Corepixen pipeline. It satisfies every non-negotiable constraint and outperforms all alternatives on governance integrity, GitHub Actions compatibility parity, and webhook reliability — the three criteria most operationally critical to this architecture.

### Positive Consequences

- GitHub REST API compatibility is an explicit, maintained project commitment. Agent scripts and CI/CD tooling require only a base URL change.
- Forgejo Actions executes GitHub Actions YAML natively. All CI/CD pipelines are portable to GitHub without modification if the hosting model changes.
- Memory footprint of approximately 200 MB under load leaves sufficient headroom for all co-hosted services.
- Governance under Codeberg e.V. (non-profit) provides structural independence from commercial pressure and a credible long-term maintenance trajectory.
- HMAC-SHA256 signed webhook payloads enable secure, verifiable event handling in ArgoCD and Prefect trigger endpoints.
- Active development with faster release cadence than Gitea following the 2022 fork.

### Negative Consequences

- Third-party integration marketplace does not exist. All external service wiring (notifications, PM tool hooks) requires manual webhook configuration.
- Community size is smaller than GitLab's or GitHub's. Fewer pre-built integrations and community tutorials are available.
- Operator is solely responsible for uptime, backup, and disaster recovery. No managed SLA is available.

---

## 5. Pros and Cons of Options

### Option A: Forgejo *(selected)*

**Pros**
- Explicit GitHub REST API and webhook payload compatibility by project commitment
- Native Forgejo Actions: executes GitHub Actions YAML without modification
- ~200 MB RAM — lowest footprint of all evaluated options
- AGPL v3.0 — fully open source, no enterprise tier, no feature gating
- Governed by Codeberg e.V. non-profit — structurally independent from commercial interests
- Active development; fastest release cadence of the self-hosted options post-2022
- Signed webhook payloads with HMAC-SHA256

**Cons**
- No native third-party integration marketplace
- Smaller community than GitLab or GitHub

---

### Option B: Gitea

**Pros**
- GitHub API compatible (Forgejo is a Gitea fork; both share this baseline)
- Very lightweight
- Large body of community documentation

**Cons**
- In 2022, project governance was transferred to Gitea Ltd (for-profit) without a community vote. This directly conflicts with ADR-0006.
- Development velocity has declined relative to Forgejo post-fork
- Selecting Gitea over Forgejo provides no technical advantage while accepting materially worse governance posture and supply chain risk

---

### Option C: GitLab Community Edition

**Pros**
- Most complete built-in feature set: PM, CI/CD, security scanning, container registry
- Comprehensive REST and GraphQL APIs
- Strong native integration ecosystem

**Cons**
- Requires 4–8 GB RAM minimum — incompatible with co-hosting on CX33
- Would require a dedicated server, adding infrastructure cost and operational complexity
- Feature set significantly exceeds requirements; PM layer is covered by the custom review dashboard (ADR-0009) and Plane
- Not GitHub REST API compatible; tooling rewrite required

---

### Option D: GitHub (SaaS)

**Pros**
- Industry-standard API, Actions ecosystem, and integration library
- Zero operational overhead

**Cons**
- SaaS: violates self-hosting requirement and ADR-0006
- Source code, pipeline definitions, and prompt templates leave operator control
- Excluded by constraint

---

### Option E: Gogs

**Pros**
- Extremely lightweight

**Cons**
- Development is largely stagnant
- Webhook reliability is poor
- No native CI/CD runner
- API surface is significantly narrower than GitHub's; compatibility is not actively maintained
- Not a credible long-term choice for a production pipeline

---

## 6. Implementation Notes

### Deployment

Forgejo is deployed as a Docker container via Docker Compose. The Compose definition is committed to the infrastructure repository and is the authoritative deployment definition.

```yaml
services:
  forgejo:
    image: codeberg.org/forgejo/forgejo:9
    container_name: forgejo
    environment:
      - USER_UID=1000
      - USER_GID=1000
    volumes:
      - forgejo-data:/data
    ports:
      - "3000:3000"
      - "2222:22"
    restart: unless-stopped
    networks:
      - corepixen-internal

volumes:
  forgejo-data:
```

Caddy terminates TLS externally and reverse-proxies to port 3000. SSH access for Git operations is exposed on port 2222 to avoid conflict with the host SSH daemon.

### Repository Structure

All pipeline components follow a monorepo layout within Forgejo:

```
/
├── .forgejo/
│   └── workflows/          # Forgejo Actions CI/CD definitions
├── docs/
│   └── adrs/               # Architecture Decision Records
├── agents/
│   ├── tier-1-scout/
│   ├── tier-2-research/
│   ├── tier-3-builder/
│   ├── tier-4-qa/
│   └── tier-5-notifier/
├── dashboard/              # Next.js human-in-the-loop review app
├── infrastructure/
│   ├── docker-compose.yml
│   ├── ansible/
│   └── argocd/
├── prefect/                # Prefect flow definitions
└── prompts/                # Versioned prompt templates
```

### ArgoCD Integration

ArgoCD is configured to watch the `infrastructure/argocd/` path in the Forgejo repository. On any commit to `main` that modifies manifests under this path, ArgoCD automatically reconciles the live stack to match Git state, enforcing ADR-0003.

### Webhook Security

All webhook consumers (ArgoCD, Prefect trigger endpoints) validate HMAC-SHA256 signatures on every Forgejo webhook payload before processing:

```python
import hmac
import hashlib

def verify_forgejo_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

### Backup

Forgejo data volume is included in the Hetzner automated snapshot schedule. A secondary daily backup job exports a Forgejo dump to Hetzner Object Storage (S3-compatible) via a Forgejo Actions cron workflow committed to the repository.

---

## 7. Compliance and Security Considerations

- All secrets (API keys, database credentials) are managed via Docker Compose secrets (Phase 1) and Infisical (Phase 2 per ADR-0011). No secrets are committed to the Forgejo repository.
- Agent service accounts are granted minimum required repository scopes: read-only for all tiers except Tier 5 (write access to update deployment manifests).
- The Forgejo instance is accessible only within the Docker internal network. External access is exclusively via Caddy reverse proxy with TLS termination.
- The Forgejo container image is pinned to a major version tag and updated via a controlled process committed to the infrastructure repository.

---

## 8. Review and Revision Criteria

This decision should be revisited if any of the following conditions arise:

- Forgejo diverges from GitHub REST API compatibility in a way that breaks existing agent tooling or ArgoCD integration
- Team growth makes GitLab CE's richer native security scanning and PM features worth the cost of a dedicated server
- A managed SLA-backed uptime requirement emerges that cannot be met by self-managed infrastructure

---

## 9. References

- Forgejo project: https://forgejo.org
- Forgejo API documentation: https://forgejo.org/docs/latest/api/
- Codeberg e.V. governance: https://codeberg.org/Codeberg/org
- Forgejo Actions documentation: https://forgejo.org/docs/latest/user/actions/
- Related: ADR-0002, ADR-0003, ADR-0004, ADR-0008
