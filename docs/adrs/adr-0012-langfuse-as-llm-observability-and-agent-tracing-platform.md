# ADR-0012: Langfuse as LLM Observability and Agent Tracing Platform

| Field         | Value                                                                        |
|---------------|------------------------------------------------------------------------------|
| ADR ID        | ADR-0012                                                                     |
| Title         | Langfuse as LLM Observability and Agent Tracing Platform                     |
| File name     | adr-0012-langfuse-as-llm-observability-and-agent-tracing-platform.md         |
| Status        | Accepted                                                                     |
| Date          | 2026-04-11                                                                   |
| Deciders      | Engineering Lead                                                             |
| Supersedes    | —                                                                            |
| Superseded by | —                                                                            |
| Related ADRs  | ADR-0008, ADR-0009, ADR-0003, ADR-0006                                       |

---

## 1. Context and Problem Statement

The Corepixen pipeline executes a sequence of LLM-driven operations for every lead that enters the system. Starting from the Lead Queue, each job moves through a state machine: `RESEARCH → PROMPT_BUILD → BUILD → QA_CHECK → AWAITING_HUMAN_REVIEW → AWAITING_PUBLISH → PUBLISH`. Every state transition is driven by one or more calls to an AI model — research synthesis, prompt generation, website code generation, and QA evaluation.

When a job fails, degrades in quality, or loops repeatedly through the `BUILD → QA_CHECK` retry cycle, the root cause is almost always in the LLM layer: a poorly constructed prompt, degraded upstream context passed from a previous state, a model response that did not conform to expected structure, or a cost-inefficient call pattern. The existing observability stack — Grafana, Prometheus, and Loki — is instrumented at the infrastructure layer. It answers questions about process health, resource utilisation, and flow execution status. It cannot answer:

- What exact prompt did the orchestrator send to Claude during the `PROMPT_BUILD` state for job `#X`?
- What did the model return, and how did that output flow into the `BUILD` state?
- Did the `QA_CHECK` agent flag a genuine quality issue, or did it fail due to a malformed evaluation prompt?
- Is a specific prompt template degrading in quality across multiple jobs over time?
- What is the per-job AI token cost, and which state in the pipeline accounts for the most spend?

Without answers to these questions, debugging a failing pipeline requires manually reconstructing LLM interactions from raw Python logs — a process that does not scale and is not practical for a non-specialist operator.

A dedicated LLM observability layer is required that traces every AI interaction across the full pipeline state machine, links traces to job records in PostgreSQL, and surfaces them directly within the human review dashboard.

---

## 2. Pipeline Architecture Reference

This ADR references the following pipeline topology. All observability design is expressed in terms of this model. Agent tier references are not used.

```
┌─────────────────────────────────────────────────────┐
│                  DISCOVERY ENGINE                    │
│  Scheduled / continuous web scraper + lead scorer   │
│  Runs independently on its own cron/trigger         │
└──────────────────────┬──────────────────────────────┘
                       │ pushes lead to queue
                       ▼
              [ LEAD QUEUE ]
              (persistent job queue — PostgreSQL)
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              PIPELINE ORCHESTRATOR                   │
│              (Prefect → Temporal in Phase 2)         │
│                                                     │
│  Job picked from queue → state machine per job:     │
│                                                     │
│  RESEARCH → PROMPT_BUILD → BUILD → QA_CHECK         │
│                               ▲         │           │
│                               └─────────┘           │
│                          (max N retries)             │
│                               │ pass                │
│                               ▼                     │
│                      AWAITING_HUMAN_REVIEW          │
│                               │ approved            │
│                               ▼                     │
│                      AWAITING_PUBLISH               │
│                               │                     │
│                               ▼                     │
│                     PUBLISH + NOTIFY                │
└─────────────────────────────────────────────────────┘
                       │
                       ▼
              [ REVIEW DASHBOARD ]
         Human sees all jobs in their state.
         Reviews site, approves or requests changes.
         Trace link available per job.
```

Each named state in the orchestrator that involves an LLM call produces one or more **Langfuse spans** nested under a single **Langfuse trace** per job. The trace is opened when the job is picked from the Lead Queue and closed when the job reaches `PUBLISH` or a terminal failure state.

---

## 3. Decision Drivers

- **LLM-layer debuggability**: The Grafana/Loki stack covers infrastructure and flow execution. It does not cover LLM prompt/response content, token usage, or inter-state context propagation. A dedicated LLM observability tool is required to close this gap.
- **Human review integration**: The review dashboard (ADR-0009) is the operator's primary operational interface. LLM traces must be surfaced within the dashboard per job — not in a separate, disconnected tool that requires context-switching. A `trace_id` stored in PostgreSQL alongside the job record enables this without architectural complexity.
- **Prompt iteration and quality improvement**: The pipeline's output quality depends on prompt quality. Langfuse's evaluation and comparison features allow prompts to be iterated against historical runs with measurable quality signals. This is not available in any other tool in the current stack.
- **Cost visibility**: Every LLM call has a token cost. Langfuse tracks token usage and cost per call, per state, and per job. Without this, cost per lead is invisible, making business model validation impossible.
- **ADR-0006 compliance**: The platform must be open source and self-hosted.
- **Resource efficiency on CX33**: The service must integrate with the existing PostgreSQL instance rather than requiring a dedicated database container.

---

## 4. Considered Options

- **Option A: Langfuse (self-hosted)** *(selected)*
- **Option B: Custom logging to PostgreSQL / Loki**
- **Option C: LangSmith (LangChain SaaS)**
- **Option D: OpenTelemetry + Grafana Tempo**

---

## 5. Decision Outcome

**Chosen option: Langfuse, self-hosted (Option A)**

Langfuse is deployed as part of the core pipeline infrastructure from day one. Every LLM call executed by LangGraph agents across all pipeline states is instrumented via the Langfuse Python SDK. Each job in the Lead Queue is assigned a Langfuse `trace_id` at the point it is picked up by the orchestrator. This `trace_id` is stored in the `jobs` table in PostgreSQL alongside all other job state.

The review dashboard (ADR-0009) reads the `trace_id` from the job record and renders a direct deep link to the Langfuse trace view for that job. The human reviewer can inspect the full LLM interaction history for any job without leaving their operational context.

### Trace Structure Per Job

Each job produces a single Langfuse trace with the following span hierarchy:

```
trace: job_{job_id}
│
├── span: discovery_engine
│   └── span: lead_scoring_llm_call
│
├── span: research
│   ├── span: web_crawl (no LLM)
│   └── span: research_synthesis_llm_call
│
├── span: prompt_build
│   └── span: website_brief_generation_llm_call
│
├── span: build (iteration N)
│   └── span: website_code_generation_llm_call
│
├── span: qa_check (iteration N)
│   ├── span: playwright_evaluation (no LLM)
│   └── span: qa_judgement_llm_call
│
└── span: publish
    └── span: email_generation_llm_call
```

If the `BUILD → QA_CHECK` retry loop executes multiple times, each iteration is a new nested span under the same trace. The retry count and QA failure reason are recorded as span metadata. This makes it immediately visible how many iterations a job required and what the QA agent flagged on each attempt.

### Positive Consequences

- Every LLM prompt and response across the full pipeline state machine is inspectable per job, in chronological order, from a single Langfuse trace view.
- Root cause analysis for QA failures is reduced from manual log reconstruction to opening a trace and reading the QA judgement span.
- Token cost per job is automatically calculated and visible. Cost per pipeline state is visible. This enables direct business model validation.
- Prompt templates can be versioned and compared in Langfuse against historical performance. Quality regressions are detectable before they compound across many jobs.
- The `trace_id` → PostgreSQL → dashboard integration adds zero architectural complexity. It is one additional column in the `jobs` table and one additional link in the review UI.
- Langfuse uses the existing PostgreSQL instance. No additional database container is required. Memory overhead on CX33 is the Langfuse application container only (~300–400 MB).

### Negative Consequences

- Langfuse is an additional service on the already-constrained CX33 node. Peak RAM usage is estimated to push the Phase 1 total to 7.0–7.5 GB, within the 8 GB limit but with reduced headroom.
- LangGraph agent code must be instrumented with the Langfuse SDK. This is a one-time integration task of approximately half a day, but it is not zero.
- Langfuse stores LLM prompt and response content in PostgreSQL. At high job volume, prompt/response storage can become a non-trivial fraction of the database size. A retention policy should be defined at deployment.

---

## 6. Pros and Cons of Options

### Option A: Langfuse (self-hosted) *(selected)*

**Pros**
- Purpose-built for LLM pipeline observability — traces, spans, prompt versioning, evaluation, cost tracking all in one tool
- Native LangChain/LangGraph integration: SDK auto-instruments LangChain calls with minimal configuration
- MIT licensed, fully open source, no feature gating
- Self-hosted Docker Compose deployment; integrates with existing PostgreSQL instance
- Active development with strong adoption in production AI engineering teams
- Web UI is purpose-built for reading LLM conversations — far more useful for this use case than Loki log queries
- `trace_id` integration with the review dashboard is clean and requires no additional infrastructure

**Cons**
- Additional RAM on CX33 (~350 MB)
- Initial instrumentation investment in LangGraph agent code
- Prompt/response data volume in PostgreSQL requires retention policy management

---

### Option B: Custom Logging to PostgreSQL / Loki

**Pros**
- Zero additional services
- Full control over data structure

**Cons**
- Building a purpose-built LLM observability tool from scratch is weeks of work that does not advance the pipeline
- Loki is optimised for log line queries, not LLM conversation inspection
- No prompt versioning, no evaluation features, no cost tracking
- This is solving a solved problem with custom engineering — waste of limited resources

---

### Option C: LangSmith (LangChain SaaS)

**Pros**
- Seamless LangChain/LangGraph integration — zero instrumentation code required
- Excellent UI for LLM trace inspection

**Cons**
- SaaS: all LLM prompt and response content — including client company research data — leaves operator control and is sent to LangChain's servers
- Violates self-hosting principle and introduces a data privacy risk for client information
- Paid above free tier limits
- Excluded on data sovereignty and ADR-0006 grounds

---

### Option D: OpenTelemetry + Grafana Tempo

**Pros**
- Industry standard distributed tracing
- Integrates with the existing Grafana stack

**Cons**
- OpenTelemetry traces generic spans — it has no concept of LLM prompts, responses, token counts, or model parameters
- Building LLM-aware tracing on top of OpenTelemetry requires significant custom instrumentation work
- The resulting tool is inferior to Langfuse for LLM debugging while requiring more implementation effort
- Appropriate for infrastructure tracing (already covered by Prometheus/Loki); not appropriate as an LLM observability layer

---

## 7. Implementation Notes

### Deployment

Langfuse is deployed as a Docker container connected to the existing PostgreSQL instance via a dedicated `langfuse` database. No additional database container is required.

```yaml
services:
  langfuse:
    image: langfuse/langfuse:latest
    container_name: langfuse
    environment:
      - DATABASE_URL=postgresql://langfuse:${LANGFUSE_DB_PASSWORD}@postgres:5432/langfuse
      - NEXTAUTH_SECRET=${LANGFUSE_SECRET}
      - NEXTAUTH_URL=https://langfuse.internal.corepixen.com
      - SALT=${LANGFUSE_SALT}
    ports:
      - "3002:3000"
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - corepixen-internal
```

Caddy reverse-proxies `langfuse.internal.corepixen.com` to port 3002. Access is restricted to the internal network — Langfuse is not publicly exposed.

### LangGraph Instrumentation

The Langfuse Python SDK integrates with LangChain/LangGraph via a callback handler. Instrumentation is applied once at the agent service level and automatically captures all downstream LLM calls:

```python
from langfuse.callback import CallbackHandler
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph

def create_instrumented_agent(job_id: str, state_name: str) -> CallbackHandler:
    return CallbackHandler(
        trace_name=f"job_{job_id}",
        session_id=job_id,
        metadata={
            "pipeline_state": state_name,
            "job_id": job_id,
        }
    )

# Usage in any pipeline state handler:
def run_research_state(job_id: str, lead_data: dict) -> dict:
    langfuse_handler = create_instrumented_agent(job_id, "RESEARCH")
    llm = ChatAnthropic(model="claude-opus-4-5").with_config(
        {"callbacks": [langfuse_handler]}
    )
    # All LLM calls made via this llm instance are automatically traced
    ...
    trace_id = langfuse_handler.get_trace_id()
    return {"result": ..., "trace_id": trace_id}
```

### trace_id Storage in PostgreSQL

The `jobs` table includes a `langfuse_trace_id` column. This is populated when the first LLM call for a job completes:

```sql
ALTER TABLE jobs ADD COLUMN langfuse_trace_id TEXT;
ALTER TABLE jobs ADD COLUMN langfuse_trace_url TEXT;
```

The trace URL is constructed as:
```
https://langfuse.internal.corepixen.com/trace/{langfuse_trace_id}
```

### Review Dashboard Integration (ADR-0009)

The Next.js review dashboard fetches the `langfuse_trace_url` alongside all other job data. The review view renders it as a direct link:

```tsx
// app/review/[job_id]/page.tsx
export default async function ReviewPage({ params }: { params: { job_id: string } }) {
  const job = await getJob(params.job_id); // reads from PostgreSQL

  return (
    <div>
      <SitePreview url={job.preview_url} />
      <JobDetails job={job} />
      {job.langfuse_trace_url && (
        <a
          href={job.langfuse_trace_url}
          target="_blank"
          rel="noopener noreferrer"
          className="text-sm text-blue-600 underline"
        >
          View AI trace →
        </a>
      )}
      <ReviewActions jobId={job.id} />
    </div>
  );
}
```

No iframe, no authentication sharing, no embedded Langfuse UI. One link per job. The reviewer opens the trace in a new tab when they need to understand why a site looks the way it does.

### Data Retention Policy

LLM prompt and response content is stored in the `langfuse` PostgreSQL database. The following retention policy is applied at deployment:

| Data type | Retention |
|-----------|-----------|
| Traces and spans for jobs in `PUBLISHED` state | 90 days |
| Traces for jobs in terminal failure state | 30 days |
| Traces for jobs currently in pipeline | Indefinite (until job completes) |

A scheduled PostgreSQL cleanup job (run as a Prefect flow) enforces this policy. Storage growth is estimated at 2–5 MB per job depending on prompt/response verbosity.

### Observability Stack Responsibility Matrix

To avoid overlap and confusion between tools, each observability concern has a single owner:

| Question | Tool |
|----------|------|
| Is the pipeline running? Are flows completing? | Prefect UI |
| Is a service down? Is RAM / CPU spiking? | Grafana + Prometheus |
| What did a Python exception say? | Loki (via Grafana) |
| What prompt did we send to Claude for job X? | Langfuse |
| Why did QA fail three times for this lead? | Langfuse |
| How many tokens did this job cost? | Langfuse |
| Is prompt quality degrading across jobs over time? | Langfuse |
| What did the human reviewer decide, and when? | PostgreSQL / Review Dashboard |

---

## 8. Compliance and Security Considerations

- The Langfuse instance is accessible only within the Docker internal network. It is not publicly reachable.
- LLM prompt and response content — which may include scraped company data and generated website copy — is stored exclusively on operator-controlled infrastructure. No data is sent to external services.
- Langfuse credentials (`LANGFUSE_SECRET`, `LANGFUSE_SALT`, `LANGFUSE_DB_PASSWORD`) are managed under ADR-0011 (Docker Compose secrets in Phase 1, Infisical in Phase 2).
- The `langfuse` PostgreSQL database is isolated from the main `corepixen` database. A dedicated database user with access scoped to the `langfuse` database only is used.

---

## 9. Review and Revision Criteria

This decision should be revisited if any of the following conditions arise:

- Langfuse's PostgreSQL storage growth from prompt/response data exceeds the CX33 disk budget and migration to a dedicated database server is not feasible
- A future version of Grafana Tempo introduces native LLM observability features that render Langfuse redundant within the existing stack
- Pipeline job volume grows to a level where Langfuse's RAM footprint must be moved to a dedicated node

---

## 10. References

- Langfuse documentation: https://langfuse.com/docs
- Langfuse self-hosting guide: https://langfuse.com/docs/deployment/self-host
- Langfuse LangChain integration: https://langfuse.com/docs/integrations/langchain
- Langfuse Python SDK: https://github.com/langfuse/langfuse-python
- Related: ADR-0008, ADR-0009, ADR-0003, ADR-0006
