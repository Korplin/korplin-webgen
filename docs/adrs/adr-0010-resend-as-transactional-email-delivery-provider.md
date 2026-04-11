# ADR-0010: Resend as Transactional Email Delivery Provider

| Field         | Value                                                                  |
|---------------|------------------------------------------------------------------------|
| ADR ID        | ADR-0010                                                               |
| Title         | Resend as Transactional Email Delivery Provider                        |
| File name     | adr-0010-resend-as-transactional-email-delivery-provider.md            |
| Status        | Accepted                                                               |
| Date          | 2026-04-11                                                             |
| Deciders      | Engineering Lead                                                       |
| Supersedes    | —                                                                      |
| Superseded by | —                                                                      |
| Related ADRs  | ADR-0006 (amended), ADR-0009                                           |

---

## 1. Context and Problem Statement

The Corepixen pipeline sends transactional emails to prospective clients when a draft website has been generated, reviewed, and approved for outreach. This is a direct, automated, business-critical communication: the email contains the prospect's preview URL and represents the first point of contact in the sales relationship.

Email delivery reliability is therefore a business outcome, not an infrastructure preference. A failure to deliver, or delivery to a spam folder, directly degrades lead conversion. The operator cannot recover from a missed first impression.

This ADR also documents a deliberate, bounded amendment to ADR-0006. ADR-0006 establishes a preference for open-source, self-hosted infrastructure. This decision supersedes that preference for the email delivery layer based on the reasoning documented below.

---

## 2. Decision Drivers

- **Deliverability as a primary constraint**: Email deliverability is governed by sender reputation, IP warming history, DNS configuration (SPF, DKIM, DMARC), and blacklist status. These are operational concerns that take weeks to establish correctly on a self-hosted mail server and months to fully mature. Failing this during active pipeline operation risks real business damage.
- **Phase 1 is a validation phase**: The pipeline's early operation is for validation and iteration, not production client acquisition. Infrastructure investment must not block product learning. Email infrastructure that requires weeks of DNS reputation warming before it is safe to use is incompatible with this phase.
- **Developer-first API**: Email is sent programmatically from the pipeline orchestrator. The email provider must expose a clean, modern REST API and a Python SDK that integrates with minimal code.
- **ADR-0006 amendment**: ADR-0006 permits paid and proprietary AI tooling as an explicit exception to the open-source preference. This decision extends that exception to email delivery infrastructure, for the reasons documented in this ADR. This is a bounded, justified deviation, not a general relaxation of the open-source principle.
- **Path to self-hosted**: The long-term goal remains a self-hosted mail infrastructure (Postal) when the pipeline is operating at production volume and the operator has the capacity to manage it. This decision explicitly defers that complexity to Phase 2.

---

## 3. Considered Options

- **Option A: Resend** *(selected)*
- **Option B: Brevo (formerly Sendinblue)**
- **Option C: SendGrid (Twilio)**
- **Option D: Postal (self-hosted mail server)**
- **Option E: AWS SES**

---

## 4. Decision Outcome

**Chosen option: Resend (Option A)**

Resend is selected as the transactional email provider for Phase 1 and Phase 2 operation. Migration to Postal (self-hosted) is explicitly deferred to Phase 3, gated on the conditions defined in Section 8.

### Positive Consequences

- Deliverability is managed professionally from day one. No IP warming, no DNS debugging, no blacklist monitoring required from the operator.
- Integration is a single Python SDK call. No SMTP configuration, no mail server infrastructure, no Docker services consuming RAM on the CX33 node.
- Resend's free tier (3,000 emails per month) covers Phase 1 and early Phase 2 volume at zero cost.
- Developer experience is modern and well-documented. Integration time is measured in minutes.
- MailHog continues to be used in local development and staging environments, intercepting all outbound email and preventing accidental client contact from non-production pipelines.
- Future migration to Postal is straightforward: Resend's SDK call is abstracted behind an `EmailService` class. Swapping the implementation requires changes in one file.

### Negative Consequences

- This is a managed SaaS service. The operator does not control the infrastructure.
- Resend is a third-party dependency with its own pricing trajectory. If pricing changes materially at scale, migration planning is required.
- Partial deviation from ADR-0006. Documented and justified here; does not set a precedent for other infrastructure layers.

---

## 5. Pros and Cons of Options

### Option A: Resend *(selected)*

**Pros**
- Purpose-built for developers: clean REST API, first-class Python SDK, excellent documentation
- Generous free tier: 3,000 emails/month, no credit card required to start
- Deliverability handled by Resend's managed infrastructure
- React Email support for building email templates in JSX — consistent with the Next.js dashboard stack
- Modern product with active development and strong community adoption in 2025–2026
- Single HTTP call from the pipeline; no infrastructure footprint on CX33

**Cons**
- SaaS dependency
- Free tier limited to one custom domain

---

### Option B: Brevo (formerly Sendinblue)

**Pros**
- Very generous free tier: 300 emails/day with no credit card required
- Solid deliverability record
- Good REST API

**Cons**
- API and SDK are less developer-ergonomic than Resend
- Dashboard and tooling feel dated compared to Resend
- Less active development momentum in the current market

**Assessment**: Strong alternative if Resend's free tier limits are hit early. Upgrade path, not first choice.

---

### Option C: SendGrid (Twilio)

**Pros**
- Battle-tested deliverability
- Mature API with broad language support

**Cons**
- Owned by Twilio. Twilio has undergone significant cost-cutting and product deprioritisation since 2022. Long-term product direction for SendGrid is uncertain.
- Free tier requires a credit card and is limited to 100 emails/day — inadequate for pipeline testing
- API design and dashboard are showing their age
- No material technical advantage over Resend; inferior developer experience
- Eliminated in favour of Resend

---

### Option D: Postal (self-hosted)

**Pros**
- Fully self-hosted and open source
- Complete control over infrastructure and data
- No per-email cost at scale
- Aligns with ADR-0006

**Cons**
- Running a self-hosted mail server correctly requires proper configuration of SPF, DKIM, and DMARC DNS records, IP reputation warming (minimum 4–6 weeks), active blacklist monitoring, and bounce/complaint handling
- This operational overhead is disproportionate for Phase 1 where pipeline validation is the primary goal
- A misconfigured mail server sending cold outreach emails can result in IP blacklisting that takes months to recover from — a direct business risk
- **Decision**: Postal is the Phase 3 target, not the Phase 1 implementation. This is deferred, not abandoned.

---

### Option E: AWS SES

**Pros**
- Extremely low per-email cost at scale
- Excellent deliverability backed by Amazon's IP reputation infrastructure

**Cons**
- Requires an AWS account and introduces an AWS dependency into an otherwise AWS-free infrastructure
- New accounts start in sandbox mode: outbound email is restricted to verified addresses only, requiring a manual production access request to AWS support before any real client emails can be sent
- Sandboxed start significantly delays pipeline go-live
- Operational complexity higher than Resend for marginal deliverability benefit at Phase 1 volume

---

## 6. Implementation Notes

### Integration

Email sending is abstracted behind a single `EmailService` class committed to the agents codebase. This ensures that migrating from Resend to Postal in Phase 3 requires changes in exactly one file:

```python
import resend
from dataclasses import dataclass

@dataclass
class DraftWebsiteEmail:
    to: str
    company_name: str
    preview_url: str
    from_name: str = "Corepixen"
    from_address: str = "hello@corepixen.com"

class EmailService:
    def __init__(self, api_key: str):
        resend.api_key = api_key

    def send_draft_notification(self, email: DraftWebsiteEmail) -> str:
        response = resend.Emails.send({
            "from": f"{email.from_name} <{email.from_address}>",
            "to": email.to,
            "subject": f"We built a draft website for {email.company_name}",
            "html": self._render_template(email),
        })
        return response["id"]

    def _render_template(self, email: DraftWebsiteEmail) -> str:
        # Template rendering logic
        ...
```

### Development and Staging

MailHog is deployed in all non-production environments. All outbound email from the pipeline is intercepted by MailHog and visible in its web UI at `http://localhost:8025`. No real email is ever sent from development or staging environments regardless of email address configuration.

```yaml
services:
  mailhog:
    image: mailhog/mailhog:latest
    container_name: mailhog
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI
    networks:
      - corepixen-internal
    profiles:
      - development
```

The `profiles: [development]` flag ensures MailHog is never started in production.

### DNS Requirements

When the Resend account is set up, the following DNS records must be added to the sending domain:

| Type | Purpose |
|------|---------|
| TXT (SPF) | Authorises Resend to send on behalf of the domain |
| CNAME (DKIM) | Cryptographically signs outbound messages |
| TXT (DMARC) | Instructs receiving servers on handling policy |

Resend's onboarding UI provides the exact record values. This is a one-time setup with no ongoing maintenance.

---

## 7. ADR-0006 Amendment — Justification

ADR-0006 establishes that all infrastructure must be open source and self-hosted, with AI model API services as the sole permitted exception.

This decision extends that exception to transactional email delivery, bounded by the following conditions:

1. The exception applies exclusively to the email delivery layer
2. The exception is time-bounded: it is the intended implementation for Phase 1 and Phase 2 only
3. The long-term target remains self-hosted Postal infrastructure
4. The migration path to Postal is explicitly designed into the implementation via the `EmailService` abstraction class
5. This exception does not set a precedent for any other infrastructure layer

---

## 8. Review and Revision Criteria — Migration to Postal

Migration from Resend to Postal should be evaluated when **all three** of the following conditions are met:

1. The pipeline is operating in full production with real client outreach (not validation phase)
2. Monthly outbound email volume justifies the operational cost of managing a self-hosted mail server
3. Engineering capacity is available to correctly configure and maintain SPF, DKIM, DMARC, IP warming, and bounce handling without impacting pipeline development velocity

Migration at that point is straightforward: swap the `EmailService` implementation. No other pipeline code changes.

---

## 9. References

- Resend documentation: https://resend.com/docs
- Resend Python SDK: https://github.com/resend/resend-python
- React Email: https://react.email
- Postal project: https://postalserver.io
- Related: ADR-0006, ADR-0009
