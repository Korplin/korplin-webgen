# ADR-0011: Phased Secret Management Strategy — Docker Compose Secrets and Infisical

| Field         | Value                                                                        |
|---------------|------------------------------------------------------------------------------|
| ADR ID        | ADR-0011                                                                     |
| Title         | Phased Secret Management Strategy — Docker Compose Secrets and Infisical     |
| File name     | adr-0011-docker-compose-secrets-phase1-infisical-phase2.md                   |
| Status        | Accepted                                                                     |
| Date          | 2026-04-11                                                                   |
| Deciders      | Engineering Lead                                                             |
| Supersedes    | —                                                                            |
| Superseded by | —                                                                            |
| Related ADRs  | ADR-0004, ADR-0006, ADR-0007                                                 |

---

## 1. Context and Problem Statement

The Corepixen pipeline handles secrets across every layer of its stack: AI model API keys (Claude, OpenAI), database credentials, webhook signing secrets, email provider API keys, internal service tokens, and ArgoCD deployment credentials. These secrets must never be committed to the Forgejo repository and must be available to services at runtime in a secure and operationally manageable way.

Secret management strategy is a function of team size, infrastructure complexity, and the operational surface area the team can realistically maintain. A single-operator Phase 1 deployment on a single CX33 node has materially different requirements than a multi-person Phase 2 deployment on a K3s cluster.

This ADR defines a two-phase strategy that applies the appropriate level of secret management complexity to each phase without introducing operational overhead that blocks pipeline development.

---

## 2. Decision Drivers

- **ADR-0004 compliance**: The deployment must be reproducible. Secret management must be part of the reproducible deployment definition, not a manual side process.
- **ADR-0006 compliance**: Phase 2 secret management must be open source and self-hosted.
- **Zero secrets in Git**: Under no circumstances are secrets committed to the Forgejo repository in any form — plaintext, base64-encoded, or otherwise.
- **Phase 1 simplicity**: Phase 1 is operated by a single engineer. Secret management must not add a service that requires ongoing maintenance before the pipeline itself is functional.
- **Phase 2 auditability**: When the team grows or the infrastructure moves to K3s, secrets must be centrally managed, auditable (who accessed what, when), and injectable into containers without modifying application code.
- **Migration path**: The transition from Phase 1 to Phase 2 secret management must not require changes to application code — only to how secrets are injected at runtime.

---

## 3. Considered Options

- **Option A: Docker Compose secrets (Phase 1) → Infisical (Phase 2)** *(selected)*
- **Option B: Infisical from day one**
- **Option C: HashiCorp Vault**
- **Option D: Environment variables in .env files**
- **Option E: Doppler (SaaS)**

---

## 4. Decision Outcome

**Chosen option: Option A — Docker Compose secrets for Phase 1, Infisical added in Phase 2**

### Phase 1: Docker Compose Secrets

Docker Compose has a native secrets mechanism. Secrets are defined as files on the host filesystem (outside of Git), referenced in the Compose definition, and mounted into containers at `/run/secrets/<secret_name>` at runtime. Application code reads secrets from these mount paths.

This provides:
- Secrets never in environment variables (reduces exposure via `docker inspect`)
- Secrets never in the Compose file itself (only references to file paths)
- Secrets never in Git (files exist only on the host)
- No additional services to deploy or maintain
- Full compatibility with the single-operator, single-node CX33 Phase 1 deployment

### Phase 2: Infisical

Infisical is a modern, open-source secret management platform. When the K3s migration occurs or a second team member is onboarded, Infisical is deployed and all secrets are migrated to it. Infisical provides:
- Centralised secret store with versioning and audit log
- Native Kubernetes secrets injection via the Infisical Operator
- Per-environment secret scoping (development, staging, production)
- RBAC: granular access control per secret, per team member
- SDK-based runtime secret injection compatible with the existing `EmailService` and agent abstractions

The migration from Phase 1 to Phase 2 requires no application code changes. Only the runtime injection mechanism changes (from file mounts to Infisical SDK/operator injection).

### Positive Consequences

- Phase 1 ships without the operational overhead of a secret management service
- Docker Compose secrets are significantly more secure than plain `.env` files
- Infisical is introduced precisely when its capabilities are needed: multi-person access control and K3s-native injection
- Application code is written to read from a consistent interface (environment variable or file path) from day one, making Phase 2 migration transparent

### Negative Consequences

- Phase 1 secret management is host-filesystem dependent. Secrets must be manually provisioned on the CX33 host during initial setup and after any full node reprovisioning.
- Phase 1 has no audit log for secret access. Acceptable for single-operator operation; not acceptable for multi-person teams.
- Phase 2 introduces Infisical as an additional service on the cluster. It has its own operational requirements.

---

## 5. Pros and Cons of Options

### Option A: Docker Compose Secrets → Infisical *(selected)*

**Pros**
- Right tool for each phase; no premature complexity
- Both solutions are operationally appropriate for their respective contexts
- Zero additional services in Phase 1
- Infisical is open source, self-hosted, and K3s-native — fully ADR-0006 compliant
- Application code abstraction makes the migration transparent

**Cons**
- Two different injection mechanisms to understand across phases
- Phase 1 host-filesystem secrets require manual provisioning on node reprovisioning

---

### Option B: Infisical from Day One

**Pros**
- Single secret management system across both phases
- Audit log and RBAC available from day one

**Cons**
- Infisical requires its own PostgreSQL database, Redis instance, and multiple application containers
- Adds approximately 400–600 MB RAM to the Phase 1 CX33 budget before any pipeline services are running
- Operational complexity before the pipeline itself is validated
- Solving a multi-person access problem that does not exist in Phase 1
- Premature complexity; violates "measure twice, cut once" applied to operational overhead

---

### Option C: HashiCorp Vault

**Pros**
- Industry standard for enterprise secret management
- Extremely powerful: dynamic secrets, PKI, leasing, audit logging

**Cons**
- Operational complexity is substantial even for simple use cases
- Vault requires a dedicated operational investment to set up correctly (unsealing, storage backends, policies)
- Resource footprint is higher than Infisical
- Infisical provides equivalent capabilities for this use case with significantly lower operational burden
- Overkill for a pipeline operated by one to three engineers

---

### Option D: Environment Variables in .env Files

**Pros**
- Zero setup, universally understood

**Cons**
- `.env` files are routinely and accidentally committed to version control
- Environment variables are visible in `docker inspect` output and process environment dumps
- No access control, no audit log, no versioning
- Not compatible with K3s deployment without additional tooling
- Not an acceptable security posture for a system handling AI model API keys and client communication credentials

---

### Option E: Doppler (SaaS)

**Pros**
- Excellent developer experience, simple setup
- Native Docker and Kubernetes integrations

**Cons**
- SaaS: secrets leave operator control
- Violates ADR-0006 and the self-hosting principle
- Introduces a third-party dependency on the critical path of every service startup
- Excluded by constraint

---

## 6. Implementation Notes

### Phase 1: Docker Compose Secrets

**Host setup** (performed once during initial VPS provisioning via Ansible):

```bash
# On the CX33 host, in a directory outside the Git repository
mkdir -p /opt/corepixen/secrets
chmod 700 /opt/corepixen/secrets

# One file per secret
echo "your-claude-api-key" > /opt/corepixen/secrets/claude_api_key
echo "your-db-password" > /opt/corepixen/secrets/db_password
echo "your-resend-api-key" > /opt/corepixen/secrets/resend_api_key
# ... etc.

chmod 600 /opt/corepixen/secrets/*
```

**Docker Compose definition:**

```yaml
secrets:
  claude_api_key:
    file: /opt/corepixen/secrets/claude_api_key
  db_password:
    file: /opt/corepixen/secrets/db_password
  resend_api_key:
    file: /opt/corepixen/secrets/resend_api_key

services:
  prefect-worker:
    image: corepixen/prefect-worker:latest
    secrets:
      - claude_api_key
      - db_password
    # Secret available at /run/secrets/claude_api_key inside the container
```

**Application code** reads secrets from a consistent helper:

```python
import os
from pathlib import Path

def get_secret(name: str) -> str:
    """
    Read a secret from Docker Compose secrets mount (Phase 1)
    or from environment variable (local development).
    """
    secret_path = Path(f"/run/secrets/{name}")
    if secret_path.exists():
        return secret_path.read_text().strip()
    env_value = os.getenv(name.upper())
    if env_value:
        return env_value
    raise RuntimeError(f"Secret '{name}' not found in /run/secrets or environment")
```

This abstraction means application code requires no changes when Phase 2 injects secrets via Infisical.

### Phase 2: Infisical

Infisical is deployed to the K3s cluster when Phase 2 migration occurs. The Infisical Kubernetes Operator is installed and configured to inject secrets as Kubernetes Secrets or environment variables into pods.

The `get_secret()` helper function above continues to work — Infisical injection populates either environment variables or file mounts depending on operator configuration.

**Phase 2 migration checklist:**
- [ ] Deploy Infisical Server to K3s cluster
- [ ] Import all Phase 1 secrets from host filesystem into Infisical
- [ ] Install Infisical Kubernetes Operator
- [ ] Configure `InfisicalSecret` CRDs for each service namespace
- [ ] Validate secret injection in staging environment
- [ ] Remove host filesystem secret files from CX33 after migration validation

### Secrets Inventory

The following secrets are managed under this ADR:

| Secret | Used By | Rotation Frequency |
|--------|---------|-------------------|
| `claude_api_key` | All LangGraph agent services | On compromise |
| `openai_api_key` | LangGraph agent services (fallback) | On compromise |
| `db_password` | All services with PostgreSQL access | Quarterly |
| `resend_api_key` | Email service | On compromise |
| `prefect_api_key` | Prefect worker | On compromise |
| `dashboard_secret` | Next.js dashboard (NextAuth) | Quarterly |
| `orchestrator_webhook_secret` | Dashboard → orchestrator webhook | On compromise |
| `argocd_webhook_secret` | Forgejo → ArgoCD webhook | On compromise |
| `minio_root_password` | MinIO and services with object storage access | Quarterly |

---

## 7. Compliance and Security Considerations

- No secrets are committed to the Forgejo repository. The Ansible playbook that provisions the host creates the secrets directory and files from a separate, operator-managed source (1Password, local encrypted vault, or similar) that is never part of the Git repository.
- The secrets directory on the host (`/opt/corepixen/secrets/`) has `chmod 700` permissions. Individual secret files have `chmod 600` permissions.
- Docker Compose secrets are mounted read-only inside containers.
- The `.gitignore` at repository root explicitly excludes `*.env`, `secrets/`, and `**/.env*` patterns.
- In Phase 2, Infisical's audit log provides a complete record of secret access by service identity and team member.

---

## 8. Review and Revision Criteria

Phase 2 migration to Infisical is triggered by **either** of the following conditions:

1. A second team member requires access to production secrets
2. The infrastructure migrates from Docker Compose on CX33 to a K3s cluster

The decision to use Infisical over alternatives in Phase 2 should be revisited if Infisical's operational requirements prove incompatible with the K3s cluster's resource budget at that time.

---

## 9. References

- Docker Compose secrets documentation: https://docs.docker.com/compose/how-tos/use-secrets/
- Infisical documentation: https://infisical.com/docs
- Infisical Kubernetes Operator: https://infisical.com/docs/documentation/getting-started/kubernetes
- Related: ADR-0004, ADR-0006, ADR-0007
