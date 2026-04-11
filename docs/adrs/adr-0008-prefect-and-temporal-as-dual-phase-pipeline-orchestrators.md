# ADR-0008: Prefect and Temporal as Dual-Phase Pipeline Orchestration Strategy

| Field         | Value                                                                       |
|---------------|-----------------------------------------------------------------------------|
| ADR ID        | ADR-0008                                                                    |
| Title         | Prefect and Temporal as Dual-Phase Pipeline Orchestration Strategy          |
| File name     | adr-0008-prefect-and-temporal-as-dual-phase-pipeline-orchestrators.md       |
| Status        | Accepted                                                                    |
| Date          | 2026-04-11                                                                  |
| Deciders      | Engineering Lead                                                            |
| Supersedes    | —                                                                           |
| Superseded by | —                                                                           |
| Related ADRs  | ADR-0003, ADR-0004, ADR-0007, ADR-0009                                      |

---

## 1. Context and Problem Statement

The Corepixen pipeline requires an orchestration layer that manages multi-step, stateful, AI-driven workflows across six agent tiers. Each lead that enters the system must be tracked through discrete states — research, prompt generation, website build, QA evaluation, human review, and publication — with support for retry loops, escalation paths, human approval gates, and parallel job processing.

Orchestration is the most architecturally consequential layer in this system. The wrong choice produces one of two failure modes:

1. **Insufficient durability**: Workflows lose state on process restart. In-flight jobs are silently dropped. This is catastrophic in production where each job represents a real lead and potential client relationship.
2. **Excessive operational complexity**: The orchestration platform itself becomes a maintenance burden that outpaces the team's capacity to operate it, stalling product development entirely.

Two platforms — **Prefect** and **Temporal** — have been evaluated. This ADR documents the decision to adopt both in a deliberate phased strategy rather than choosing one over the other.

---

## 2. Decision Drivers

- **Operational velocity in Phase 1**: The pipeline must be buildable and observable by a single engineer with beginner-to-intermediate DevOps experience. Time-to-first-working-pipeline is a primary success metric.
- **Durability for agentic workflows in Phase 2**: Workflows that run for hours, park on human approval gates for days, or handle real client data require exactly-once execution semantics and crash-safe state persistence.
- **GitOps compatibility**: All flow definitions, deployment configurations, and workflow code must be version-controlled and deployable via the Forgejo/ArgoCD GitOps pipeline (ADR-0003, ADR-0007).
- **Self-hosted, open source**: Both platforms must be self-hostable with no mandatory SaaS dependency. ADR-0006 compliance required.
- **Resource efficiency on CX33**: Phase 1 orchestration infrastructure must fit within the shared 8 GB RAM budget of the single Hetzner CX33 node.
- **Non-overlapping scope**: The two platforms must coexist without competing for ownership of the same workflows. Scope boundaries must be explicit and maintainable.

---

## 3. Considered Options

- **Option A: Prefect (Phase 1) + Temporal (Phase 2 addition)** *(selected)*
- **Option B: Prefect only**
- **Option C: Temporal only from day one**
- **Option D: n8n**
- **Option E: Apache Airflow**

---

## 4. Decision Outcome

**Chosen option: Option A — Prefect as Phase 1 orchestrator, Temporal added in Phase 2 for durable agentic workflows**

This is a deliberate two-phase strategy, not a compromise. The platforms serve different purposes, do not overlap in practice, and will coexist cleanly in production.

### Phase 1: Prefect owns all orchestration

Prefect is deployed from day one. It manages the complete pipeline: discovery cron jobs, research flows, website build flows, QA evaluation flows, and PM notification triggers. Prefect provides:

- Python-native flow definitions that are version-controlled in Forgejo
- A built-in observability UI (Prefect UI) showing flow run history, state, logs, and retry status
- Sufficient durability for Phase 1 workloads where job loss risk is low (no real clients yet, validation phase)
- A resource footprint of approximately 400 MB RAM, compatible with CX33 co-hosting

### Phase 2: Temporal is introduced for specific workflow classes

Temporal is introduced — without removing Prefect — when one or more of the following triggers is observed in production:

| Trigger | Description |
|---------|-------------|
| **Hours-long workflows** | An agent workflow regularly runs for more than 1 hour and process restarts mid-execution result in real data loss |
| **Human approval gate durability** | Human review states must park reliably for 24–72 hours with zero risk of loss on infrastructure events |
| **Exactly-once guarantees required** | A workflow failure results in duplicate emails sent to clients or duplicate site publications |
| **Concurrency ceiling hit** | Parallel job throughput exceeds what Prefect handles comfortably on available infrastructure |

When Temporal is introduced, **Prefect is not removed**. Scope is divided as follows:

| Workload | Owner |
|----------|-------|
| Discovery cron, lead scoring | Prefect |
| Research data pipelines | Prefect |
| QA evaluation batch runs | Prefect |
| Observability and monitoring flows | Prefect |
| Durable multi-step agentic workflows (build → QA → human gate → publish) | Temporal |

### Positive Consequences

- Phase 1 ships significantly faster. Prefect's learning curve is measured in hours, not weeks.
- All pipeline behaviour is observable from Prefect UI from day one — critical for debugging agent workflows during early development.
- Temporal is introduced to solve a concrete, observed problem rather than anticipated complexity. Investment is justified by evidence.
- Both platforms are Python-native. Flow definitions are committed to Forgejo and deployed via the same GitOps pipeline as all other infrastructure.
- Migration from Prefect to Temporal for specific workflow classes is tractable because scope boundaries are explicit from the start.

### Negative Consequences

- The team will eventually operate two orchestration systems. This adds operational surface area in Phase 2.
- Temporal's learning curve is non-trivial. When Phase 2 triggers are hit, engineering investment is required before Temporal can be put into production.
- Prefect flow definitions and Temporal workflow definitions are structurally different. There is no automatic migration path — workflows must be rewritten when transitioning a specific flow from Prefect to Temporal.

---

## 5. Pros and Cons of Options

### Option A: Prefect + Temporal (phased) *(selected)*

**Pros**
- Ships fastest in Phase 1; adds durability precisely when and where it is needed
- Both tools are best-in-class for their respective workload types
- Scope separation is clean and maintainable
- Both are self-hosted, open source, Python-native, GitOps-compatible

**Cons**
- Two systems to operate in Phase 2
- Temporal investment arrives at a later, potentially inconvenient moment

---

### Option B: Prefect Only

**Pros**
- Single system; simpler operational model indefinitely
- Fastest path to a working pipeline

**Cons**
- Prefect is not designed for durable, long-running agentic workflows with human approval gates
- At production scale with real clients, job loss on process restart becomes a business risk, not just a technical annoyance
- Does not scale to the full pipeline vision without architectural rework

---

### Option C: Temporal Only from Day One

**Pros**
- Maximum durability from the start
- One system to learn and operate long-term

**Cons**
- Temporal's operational complexity is substantial: requires a Temporal Server cluster (frontend, history, matching, worker services), a separate database, and meaningful infrastructure overhead
- Learning curve is weeks, not hours — directly conflicts with Phase 1 velocity requirement
- Solves durability problems that do not yet exist in Phase 1, while adding complexity that does exist immediately
- Premature optimisation: investing in exactly-once guarantees before knowing which workflows actually require them

---

### Option D: n8n

**Pros**
- Visual workflow editor; low initial learning curve
- Large integration library

**Cons**
- Not designed for stateful, looping, multi-agent pipelines
- Workflow state is not truly durable: in-flight executions can be lost on container restart without careful configuration
- Human approval "wait for webhook" pattern holds workflow execution open in memory indefinitely — degrades under concurrent load
- Workflows are stored as JSON blobs; Git diffs are unreadable, breaking GitOps discipline
- Version controlling n8n workflows cleanly is a known pain point
- Eliminated on durability and GitOps compatibility grounds

---

### Option E: Apache Airflow

**Pros**
- Mature, battle-tested, widely deployed
- Strong scheduling and DAG management

**Cons**
- DAG model is designed for directed acyclic batch pipelines, not stateful looping agent workflows
- Cycles (QA retry loop back to builder) are awkward to model in Airflow's paradigm
- Heavier infrastructure footprint than Prefect
- Python boilerplate overhead is significant compared to Prefect's decorator model
- Less appropriate for event-driven agent orchestration than either Prefect or Temporal

---

## 6. Implementation Notes

### Prefect Deployment (Phase 1)

Prefect is deployed as a self-hosted Prefect Server instance alongside a Prefect Worker. All flow code is committed to the Forgejo repository under `/prefect/flows/`.

```yaml
services:
  prefect-server:
    image: prefecthq/prefect:3-python3.12
    container_name: prefect-server
    command: prefect server start --host 0.0.0.0
    environment:
      - PREFECT_SERVER_API_HOST=0.0.0.0
      - PREFECT_API_DATABASE_CONNECTION_URL=postgresql+asyncpg://prefect:${PREFECT_DB_PASSWORD}@postgres:5432/prefect
    ports:
      - "4200:4200"
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - corepixen-internal

  prefect-worker:
    image: prefecthq/prefect:3-python3.12
    container_name: prefect-worker
    command: prefect worker start --pool corepixen-pool
    environment:
      - PREFECT_API_URL=http://prefect-server:4200/api
    depends_on:
      - prefect-server
    restart: unless-stopped
    networks:
      - corepixen-internal
```

### Flow Structure

Each pipeline tier maps to a Prefect flow. Flows are deployed via Forgejo Actions on push to `main`:

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=30)
def research_company(lead_id: str) -> dict:
    # LangGraph agent invocation
    ...

@flow(name="corepixen-research-pipeline")
def research_pipeline(lead_id: str):
    profile = research_company(lead_id)
    # advance job state in PostgreSQL
    ...
```

### Temporal Introduction Checklist (Phase 2)

Before deploying Temporal, the following must be completed:

- [ ] Temporal Server deployed via Docker Compose, backed by a dedicated PostgreSQL database
- [ ] Temporal Web UI accessible internally via Caddy
- [ ] At least one Temporal Worker service deployed per workflow namespace
- [ ] Python Temporal SDK (`temporalio`) added to agent service dependencies
- [ ] Scope boundary documented and communicated: which flows remain in Prefect, which migrate to Temporal
- [ ] Human approval gate implemented as a Temporal Signal in the build/QA/publish workflow

---

## 7. Compliance and Security Considerations

- All Prefect flow run secrets (API keys, database credentials) are injected at runtime via Docker Compose secrets (Phase 1) or Infisical (Phase 2, ADR-0011). No secrets are hardcoded in flow definitions committed to Forgejo.
- Prefect Server API is accessible only within the Docker internal network. The Prefect UI is proxied through Caddy with TLS and restricted to internal access in production.
- Flow run logs are aggregated to Loki via Promtail for centralised observability.

---

## 8. Review and Revision Criteria

This decision should be revisited if any of the following conditions arise:

- Prefect introduces breaking changes that make self-hosted operation significantly more complex
- Temporal's operational complexity proves prohibitive when Phase 2 triggers are hit, warranting evaluation of alternatives such as Restate or Hatchet
- A single orchestration platform emerges that satisfies both Phase 1 velocity and Phase 2 durability requirements without the complexity tradeoffs of Temporal

---

## 9. References

- Prefect documentation: https://docs.prefect.io
- Temporal documentation: https://docs.temporal.io
- Temporal Python SDK: https://github.com/temporalio/sdk-python
- Related: ADR-0003, ADR-0007, ADR-0009
