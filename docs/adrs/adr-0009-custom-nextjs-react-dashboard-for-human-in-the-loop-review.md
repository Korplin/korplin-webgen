# ADR-0009: Custom Next.js and React Dashboard for Human-in-the-Loop Pipeline Review

| Field         | Value                                                                              |
|---------------|------------------------------------------------------------------------------------|
| ADR ID        | ADR-0009                                                                           |
| Title         | Custom Next.js and React Dashboard for Human-in-the-Loop Pipeline Review           |
| File name     | adr-0009-custom-nextjs-react-dashboard-for-human-in-the-loop-review.md             |
| Status        | Accepted                                                                           |
| Date          | 2026-04-11                                                                         |
| Deciders      | Engineering Lead                                                                   |
| Supersedes    | —                                                                                  |
| Superseded by | —                                                                                  |
| Related ADRs  | ADR-0008, ADR-0010                                                                 |

---

## 1. Context and Problem Statement

The Corepixen pipeline requires a human review gate between automated QA approval and final publication. At this gate, a human operator must: view a generated website preview, assess its quality, either approve it for publication or submit structured correction notes that re-enter the build loop, and optionally trigger re-runs with specific instructions.

Additionally, the pipeline needs a client relationship tracking layer that activates post-publication — recording that a draft website was sent to a specific prospect and enabling follow-up status management.

Two distinct tools serve these two distinct purposes:

1. **Internal review dashboard** — operational tool for the Corepixen team; used continuously during pipeline operation; must be fast, purpose-built, and tightly integrated with the pipeline's PostgreSQL job state.
2. **Client relationship tracking** — Plane is used post-publication to track outreach status per client.

This ADR governs the design and tooling decision for the internal review dashboard. The decision to use Plane post-publication is documented here as a related architectural boundary.

---

## 2. Decision Drivers

- **Pipeline integration**: The dashboard must read and write directly to the PostgreSQL jobs table that the pipeline orchestrator (Prefect/Temporal) uses as its state store. Real-time job state must be visible without polling delays or API translation layers.
- **Webhook-driven workflow resumption**: When a human approves or requests changes, the dashboard must fire a precise webhook that resumes the paused Prefect/Temporal workflow for that specific job. This integration must be reliable and deterministic.
- **Operational simplicity**: The dashboard must add zero additional infrastructure services to the CX33 node. It must not require its own database, cache, or background worker.
- **Bespoke UX**: The review interface is a single-purpose operational tool. Its entire surface area is: a job queue, a site preview iframe, an approve button, a correction notes field, and a status indicator. Generic project management tools impose significant UX overhead for this narrow use case.
- **No PM tool on the critical path**: If the PM tool (Plane) is unavailable, the review workflow must continue uninterrupted. Plane is a post-publication concern only.

---

## 3. Considered Options

- **Option A: Custom Next.js + React application** *(selected)*
- **Option B: Plane (open-source PM tool) as review dashboard**
- **Option C: Retool (self-hosted)**
- **Option D: Appsmith (self-hosted)**

---

## 4. Decision Outcome

**Chosen option: Custom Next.js + React application (Option A)**

A bespoke internal dashboard is built as a Next.js application. It connects directly to PostgreSQL for job state, serves the review interface, and fires webhook callbacks to the pipeline orchestrator on human action.

Plane is used exclusively as a **post-publication client tracking tool**. When a site is published and the outreach email is sent to the prospect, the pipeline fires a Plane API call to create an issue representing that client relationship. From that point forward, Plane tracks follow-up status — it has no involvement in the pipeline's operational review loop.

### Positive Consequences

- Zero additional infrastructure: the Next.js app is a single Docker container. No new database, no new cache, no new background worker.
- Direct PostgreSQL access gives the dashboard real-time job state with sub-second latency and no API translation overhead.
- The webhook integration between the dashboard and the orchestrator is fully controlled — no third-party API shape to accommodate.
- Plane is entirely off the critical path of pipeline operations. Plane downtime has zero impact on the review workflow.
- The dashboard is version-controlled in Forgejo alongside all other pipeline code. It deploys via the same GitOps pipeline as every other service.
- The UX is purpose-built: every pixel serves the review task. No project management noise.

### Negative Consequences

- Custom software requires development time. Initial build is estimated at one to two days for a functional MVP.
- Ongoing maintenance responsibility for the dashboard falls entirely on the operator. There is no upstream community to patch bugs or add features.
- Plane's PM features (reporting, workload views, sprint tracking) are not available for internal pipeline jobs. If internal pipeline PM becomes a requirement, a separate tool or extension of the dashboard is needed.

---

## 5. Pros and Cons of Options

### Option A: Custom Next.js + React *(selected)*

**Pros**
- Exactly the right tool surface for the job: no more, no less
- Direct PostgreSQL integration — no ORM layers, no API translation
- Zero additional infrastructure services
- Full control over webhook payload shape and orchestrator integration
- Deployed and version-controlled identically to all other pipeline services
- Next.js App Router with Server Components enables server-side data fetching with minimal client-side complexity

**Cons**
- Requires development time
- Operator owns all maintenance

---

### Option B: Plane as Review Dashboard

**Pros**
- Feature-complete PM tool with no custom development required
- Open source and self-hosted

**Cons**
- Plane is designed for human sprint management, not high-frequency programmatic state updates from an automated pipeline
- Plane requires its own PostgreSQL instance, Redis, and several microservices — adds approximately 600–800 MB RAM to the CX33 budget
- Plane's webhook support for outbound events (triggering pipeline resumption on state change) is limited and not well documented for custom automation consumers
- Plane's API is optimised for human interaction patterns, not sub-second pipeline state reads
- Putting Plane on the critical review path means Plane downtime blocks human review of all pipeline jobs
- **Decision: Plane is used post-publication only, never as a review dashboard**

---

### Option C: Retool (self-hosted)

**Pros**
- Visual internal tool builder; fast to create basic dashboards
- Direct database connector support

**Cons**
- Self-hosted Retool requires a licence key; violates ADR-0006 spirit
- Resource-heavy for a single-purpose dashboard
- Vendor-controlled product roadmap and pricing trajectory

---

### Option D: Appsmith (self-hosted)

**Pros**
- Open source, self-hosted internal tool builder
- Direct PostgreSQL connector

**Cons**
- Additional infrastructure service with its own resource footprint
- Webhook integration with the pipeline orchestrator requires configuration work that is comparable in effort to building the custom dashboard
- Generalist tooling with significant UX overhead for a narrow, single-purpose review task
- Version controlling Appsmith app definitions is non-trivial

---

## 6. Implementation Notes

### Application Architecture

The review dashboard is a Next.js 14+ application using the App Router. It is structured as a single Docker container deployed via Docker Compose alongside all other pipeline services.

**Core views:**

| Route | Purpose |
|-------|---------|
| `/` | Job queue: list of all jobs in `AWAITING_REVIEW` state, sorted by creation time |
| `/review/[job_id]` | Review view: site preview iframe, company profile summary, QA report, approve/reject controls |
| `/jobs` | Full job history: all jobs across all states with filtering |

**Review action flow:**

1. Human opens `/review/[job_id]`
2. Dashboard fetches job record from PostgreSQL via server-side data fetch
3. Site preview loads in iframe served from MinIO via Caddy subdomain
4. Human clicks **Approve** or submits correction notes via **Request Changes**
5. Dashboard calls internal API route `/api/review/[job_id]/decision`
6. API route updates PostgreSQL job state and fires webhook to Prefect/Temporal orchestrator
7. Orchestrator resumes the paused workflow for that job

### Technology Stack

| Component | Choice | Reason |
|-----------|--------|--------|
| Framework | Next.js 14+ (App Router) | Server-side rendering, API routes, single deployable unit |
| UI | React + Tailwind CSS | Familiar, fast to build, minimal bundle |
| Database client | `pg` (node-postgres) | Direct PostgreSQL access from Next.js API routes |
| Containerisation | Docker | Consistent with all other pipeline services |

### Docker Compose Entry

```yaml
services:
  dashboard:
    build:
      context: ./dashboard
      dockerfile: Dockerfile
    container_name: corepixen-dashboard
    environment:
      - DATABASE_URL=postgresql://corepixen:${DB_PASSWORD}@postgres:5432/corepixen
      - ORCHESTRATOR_WEBHOOK_URL=http://prefect-server:4200/api/webhooks/${WEBHOOK_ID}
      - NEXTAUTH_SECRET=${DASHBOARD_SECRET}
    ports:
      - "3001:3000"
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - corepixen-internal
```

### Plane Integration (Post-Publication Only)

When a job transitions to `PUBLISHED` state, the pipeline orchestrator fires a single Plane API call to create a client tracking issue:

```python
import httpx

def create_plane_client_issue(company_name: str, preview_url: str, email_sent_at: str):
    httpx.post(
        f"{PLANE_API_URL}/api/v1/workspaces/{WORKSPACE}/projects/{PROJECT}/issues/",
        headers={"X-Api-Key": PLANE_API_KEY},
        json={
            "name": f"Draft delivered: {company_name}",
            "description": f"Preview: {preview_url}\nEmail sent: {email_sent_at}",
            "state": "to-follow-up",
        }
    )
```

Plane is never called before publication. Plane downtime after publication has no effect on pipeline operation.

---

## 7. Compliance and Security Considerations

- The dashboard is accessible only within the Docker internal network. External access is proxied through Caddy with TLS.
- Authentication is enforced via NextAuth.js. Only authorised operators can access the review interface.
- All database credentials and webhook secrets are injected via Docker Compose secrets. No secrets are committed to the Forgejo repository.
- The dashboard performs no direct write operations to job records beyond state transitions explicitly triggered by human action. It cannot modify pipeline configuration or agent code.

---

## 8. Review and Revision Criteria

This decision should be revisited if any of the following conditions arise:

- Team grows to more than three operators and the bespoke dashboard's PM capabilities become insufficient for workload coordination
- Pipeline job volume grows to a point where the dashboard's direct PostgreSQL queries require query optimisation or a read replica
- A requirement for mobile review access emerges, warranting a more sophisticated frontend architecture

---

## 9. References

- Next.js App Router documentation: https://nextjs.org/docs/app
- node-postgres documentation: https://node-postgres.com
- Plane API documentation: https://developers.plane.so
- Related: ADR-0008, ADR-0010
