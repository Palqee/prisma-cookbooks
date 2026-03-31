# Real-Time Policy Monitoring in Production

A contact center platform validates every agent interaction as it happens, not in batch. OTLP logs from a live telephony system feed Prisma for continuous policy compliance monitoring with alerting on violations.

## What You'll Learn

- Build a FastAPI service that receives live transcript events from your CRM or telephony platform
- Log each interaction to Prisma for real-time policy validation via OTLP
- Define policy metrics that catch compliance violations and escalation procedure failures
- Update policy metrics on a running service without restarting
- Set up monitoring and alerting patterns for violation rates and compliance thresholds

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **Access to a running Prisma instance**
- Familiarity with [Getting Started](getting-started.md) and [Building Policy-Based Metrics](policy-based-metrics.md)

## 1. Install Dependencies

```bash
pip install prisma-ai fastapi uvicorn
```

The `prisma-ai` package provides the SDK for OTLP log ingestion and policy validation. `fastapi` and `uvicorn` provide the web framework and ASGI server for receiving live transcript webhooks.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Define Policy Metrics

Define two policy metrics that validate contact center interactions against your organization's compliance standards.

The first metric, `response-compliance`, validates whether agent responses follow your company's communication policies -- tone, accuracy, required disclosures, and prohibited language.

```python
from prisma_ai import Evaluators

response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant", "needs_review"],
    evaluation_system_prompt=(
        "You are a contact center compliance reviewer. Your task is to "
        "determine whether an agent's response follows company communication "
        "policies.\n\n"
        "The policies require:\n"
        "1. PROFESSIONAL TONE: The agent is courteous, empathetic, and avoids "
        "dismissive or confrontational language.\n"
        "2. ACCURATE INFORMATION: The agent provides correct information about "
        "products, services, and procedures. No guessing or speculation.\n"
        "3. REQUIRED DISCLOSURES: When discussing pricing, contracts, or policy "
        "changes, the agent includes all required disclaimers.\n"
        "4. PROHIBITED LANGUAGE: The agent does not make guarantees, promises "
        "about timelines without verification, or statements that could create "
        "legal liability.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- compliant: All policies are followed.\n"
        "- non_compliant: One or more policies are clearly violated.\n"
        "- needs_review: The response is in a gray area where compliance "
        "cannot be confidently determined without additional context."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for policy compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "For each policy (tone, accuracy, disclosures, prohibited language), "
        "state whether it was followed or violated, then provide your classification."
    ),
    reeval_system_prompt=(
        "You are a senior contact center compliance reviewer performing a "
        "second-pass review. Apply the communication policies strictly.\n\n"
        "Pay particular attention to implicit guarantees -- statements that "
        "stop short of a direct promise but still set customer expectations "
        "beyond what the agent can deliver. These are the most common "
        "borderline violations.\n\n"
        "You have access to decisions from previous compliance reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for policy compliance "
        "using context from previous compliance reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past compliance review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification with specific policy findings."
    ),
)
```

The second metric, `escalation-procedure`, validates whether the agent followed the correct escalation process when a customer requests a supervisor, threatens legal action, or reports a safety concern.

```python
escalation_procedure = Evaluators.custom(
    name="escalation-procedure",
    options=["procedure_followed", "procedure_violated", "not_applicable"],
    evaluation_system_prompt=(
        "You are an escalation process reviewer for a contact center. Your "
        "task is to determine whether an agent followed the correct escalation "
        "procedure when an escalation trigger was present.\n\n"
        "Escalation triggers include:\n"
        "- Customer explicitly requests a supervisor or manager\n"
        "- Customer threatens legal action or mentions an attorney\n"
        "- Customer reports a safety concern or physical harm\n"
        "- Customer expresses intent to contact media or regulators\n\n"
        "When a trigger is present, the required procedure is:\n"
        "1. ACKNOWLEDGE: The agent acknowledges the customer's concern "
        "without being dismissive.\n"
        "2. DOCUMENT: The agent confirms the details of the situation.\n"
        "3. TRANSFER: The agent initiates a transfer to the appropriate "
        "escalation path (supervisor, legal, safety team) without delay.\n\n"
        "If no escalation trigger is present, classify as not_applicable.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- procedure_followed: An escalation trigger was present and the "
        "agent followed all three steps.\n"
        "- procedure_violated: An escalation trigger was present but the "
        "agent skipped one or more steps, or attempted to resolve the issue "
        "without escalating.\n"
        "- not_applicable: No escalation trigger was present in the interaction."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for escalation "
        "procedure compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "First determine whether an escalation trigger is present. If so, "
        "assess each step (acknowledge, document, transfer). Then provide "
        "your classification."
    ),
    reeval_system_prompt=(
        "You are a senior escalation process reviewer performing a "
        "second-pass review. Apply the escalation procedure strictly.\n\n"
        "The most critical violation is an agent who attempts to de-escalate "
        "or resolve the issue themselves when a transfer is required. Even "
        "well-intentioned attempts to help the customer directly are procedure "
        "violations when a trigger is present.\n\n"
        "You have access to decisions from previous escalation reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for escalation procedure "
        "compliance using context from previous escalation reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past escalation review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification. Identify the specific escalation trigger (if any) "
        "and assess each procedure step."
    ),
)
```

## 4. Build the Monitoring Service

Create a file called `monitor.py`. This FastAPI application receives live transcript events from your CRM or telephony platform via a POST webhook, logs each interaction to Prisma for real-time validation, and manages the Prisma session lifecycle.

```python
"""Real-time policy monitoring service for contact center interactions."""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

import prisma_ai
from prisma_ai import EvaluationConfig, Evaluators

logger = logging.getLogger("policy-monitor")

# ---------------------------------------------------------------------------
# 1. Policy metric definitions
# ---------------------------------------------------------------------------
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant", "needs_review"],
    evaluation_system_prompt=(
        "You are a contact center compliance reviewer. Your task is to "
        "determine whether an agent's response follows company communication "
        "policies.\n\n"
        "The policies require:\n"
        "1. PROFESSIONAL TONE: The agent is courteous, empathetic, and avoids "
        "dismissive or confrontational language.\n"
        "2. ACCURATE INFORMATION: The agent provides correct information about "
        "products, services, and procedures. No guessing or speculation.\n"
        "3. REQUIRED DISCLOSURES: When discussing pricing, contracts, or policy "
        "changes, the agent includes all required disclaimers.\n"
        "4. PROHIBITED LANGUAGE: The agent does not make guarantees, promises "
        "about timelines without verification, or statements that could create "
        "legal liability.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- compliant: All policies are followed.\n"
        "- non_compliant: One or more policies are clearly violated.\n"
        "- needs_review: The response is in a gray area where compliance "
        "cannot be confidently determined without additional context."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for policy compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "For each policy (tone, accuracy, disclosures, prohibited language), "
        "state whether it was followed or violated, then provide your classification."
    ),
    reeval_system_prompt=(
        "You are a senior contact center compliance reviewer performing a "
        "second-pass review. Apply the communication policies strictly.\n\n"
        "Pay particular attention to implicit guarantees -- statements that "
        "stop short of a direct promise but still set customer expectations "
        "beyond what the agent can deliver. These are the most common "
        "borderline violations.\n\n"
        "You have access to decisions from previous compliance reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for policy compliance "
        "using context from previous compliance reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past compliance review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification with specific policy findings."
    ),
)

escalation_procedure = Evaluators.custom(
    name="escalation-procedure",
    options=["procedure_followed", "procedure_violated", "not_applicable"],
    evaluation_system_prompt=(
        "You are an escalation process reviewer for a contact center. Your "
        "task is to determine whether an agent followed the correct escalation "
        "procedure when an escalation trigger was present.\n\n"
        "Escalation triggers include:\n"
        "- Customer explicitly requests a supervisor or manager\n"
        "- Customer threatens legal action or mentions an attorney\n"
        "- Customer reports a safety concern or physical harm\n"
        "- Customer expresses intent to contact media or regulators\n\n"
        "When a trigger is present, the required procedure is:\n"
        "1. ACKNOWLEDGE: The agent acknowledges the customer's concern "
        "without being dismissive.\n"
        "2. DOCUMENT: The agent confirms the details of the situation.\n"
        "3. TRANSFER: The agent initiates a transfer to the appropriate "
        "escalation path (supervisor, legal, safety team) without delay.\n\n"
        "If no escalation trigger is present, classify as not_applicable.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- procedure_followed: An escalation trigger was present and the "
        "agent followed all three steps.\n"
        "- procedure_violated: An escalation trigger was present but the "
        "agent skipped one or more steps, or attempted to resolve the issue "
        "without escalating.\n"
        "- not_applicable: No escalation trigger was present in the interaction."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for escalation "
        "procedure compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "First determine whether an escalation trigger is present. If so, "
        "assess each step (acknowledge, document, transfer). Then provide "
        "your classification."
    ),
    reeval_system_prompt=(
        "You are a senior escalation process reviewer performing a "
        "second-pass review. Apply the escalation procedure strictly.\n\n"
        "The most critical violation is an agent who attempts to de-escalate "
        "or resolve the issue themselves when a transfer is required. Even "
        "well-intentioned attempts to help the customer directly are procedure "
        "violations when a trigger is present.\n\n"
        "You have access to decisions from previous escalation reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for escalation procedure "
        "compliance using context from previous escalation reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past escalation review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification. Identify the specific escalation trigger (if any) "
        "and assess each procedure step."
    ),
)


# ---------------------------------------------------------------------------
# 2. FastAPI lifespan — Prisma init and shutdown
# ---------------------------------------------------------------------------
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialize Prisma with policy metrics and OTLP logging
    session = prisma_ai.init(
        project_name="contact-center-monitoring",
        evaluators=[response_compliance, escalation_procedure],
        enable_logging=True,
    )
    data_logger = session.get_logger()
    app.state.session = session
    app.state.data_logger = data_logger

    logger.info(
        "Prisma AI initialized — project=%s, policy metrics active",
        session.project_id,
    )

    yield

    # Shutdown: flush pending OTLP logs and close the session
    app.state.session.end()
    logger.info("Prisma AI session ended, pending logs flushed")


app = FastAPI(title="Policy Monitoring Service", lifespan=lifespan)


# ---------------------------------------------------------------------------
# 3. Request/response models
# ---------------------------------------------------------------------------
class TranscriptEvent(BaseModel):
    """A transcript event from the CRM or telephony platform."""
    query: str          # Customer side of the interaction
    response: str       # Agent side of the interaction
    reference: str = "" # Optional policy-correct response for comparison


class TranscriptResponse(BaseModel):
    status: str


# ---------------------------------------------------------------------------
# 4. Transcript ingestion endpoint
# ---------------------------------------------------------------------------
@app.post("/transcripts", response_model=TranscriptResponse)
async def receive_transcript(event: TranscriptEvent):
    """Receive a live transcript event and log it to Prisma for validation."""
    data_logger = app.state.data_logger
    if data_logger is None:
        raise HTTPException(status_code=503, detail="Prisma session not active")

    data_logger.log(
        query=event.query,
        response=event.response,
        reference=event.reference,
    )

    return TranscriptResponse(status="logged")


# ---------------------------------------------------------------------------
# 5. Health check
# ---------------------------------------------------------------------------
@app.get("/health")
async def health():
    session = app.state.session
    if session is None:
        return {"status": "degraded", "prisma": "not initialized"}
    return {
        "status": "healthy",
        "prisma_project_id": session.project_id,
    }
```

Key points:

- `prisma_ai.init()` creates the project, registers both policy metrics, and configures the OTLP logs pipeline. The `enable_logging=True` flag activates the DataLogger for manual log ingestion.
- `session.get_logger()` returns the DataLogger. Each `data_logger.log()` call sends a structured OTLP log record to your Prisma instance for validation against both policy metrics.
- The `/transcripts` endpoint is a webhook target. Configure your CRM or telephony platform to POST transcript events here as interactions complete.
- `session.end()` on shutdown flushes any pending OTLP log records before the process exits.

## 5. Test the Service

Start the server:

```bash
uvicorn monitor:app --host 0.0.0.0 --port 8000
```

Send a compliant interaction:

```bash
curl -X POST http://localhost:8000/transcripts \
  -H "Content-Type: application/json" \
  -d '{
    "query": "I have been waiting three weeks for my refund. What is going on?",
    "response": "I understand your frustration, and I apologize for the delay. Let me pull up your account to check the status of your refund right now. I can see the refund was processed on the 15th — it typically takes 5-10 business days to appear in your account. If it has not arrived by Friday, please call us back and we will escalate this to our finance team."
  }'
```

```json
{"status": "logged"}
```

Send an interaction with an escalation trigger:

```bash
curl -X POST http://localhost:8000/transcripts \
  -H "Content-Type: application/json" \
  -d '{
    "query": "This is unacceptable. I want to speak to your supervisor immediately.",
    "response": "I completely understand your frustration. Let me make sure I have all the details of your concern documented, and I will transfer you to my supervisor right away. Please hold for just a moment."
  }'
```

Send a non-compliant interaction:

```bash
curl -X POST http://localhost:8000/transcripts \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Will my claim be approved?",
    "response": "Yes, absolutely. You will definitely get approved and should see the payout within a week."
  }'
```

Behind the scenes, each `data_logger.log()` call sends an OTLP log record to Prisma. The validation pipeline runs both policy metrics on every interaction:

- **response-compliance** catches the guarantee and timeline promise in the third example ("Yes, absolutely" and "definitely get approved" violate the prohibited language policy).
- **escalation-procedure** validates that the second example follows the acknowledge-document-transfer sequence correctly.

Check the health endpoint:

```bash
curl http://localhost:8000/health
```

```json
{"status": "healthy", "prisma_project_id": "contact-center-monitoring"}
```

## 6. Update Metrics Without Restart

You can add or change policy metrics on a running service without restarting the process. This is useful for responding to new compliance requirements, adding metrics for a new product line, or tightening criteria after an audit finding.

Add an admin endpoint to the service:

> **Note:** In production, protect administrative endpoints with authentication middleware. This example omits auth for brevity.

```python
@app.post("/admin/update-metrics")
async def update_metrics():
    """Add a data-handling metric without restarting the service."""
    session = app.state.session
    if session is None:
        return {"error": "Prisma session not active"}

    data_handling = Evaluators.custom(
        name="data-handling",
        options=["compliant", "violation", "needs_review"],
        evaluation_system_prompt=(
            "You are a data handling compliance reviewer for a contact center. "
            "Your task is to determine whether an agent's response properly "
            "handles sensitive customer data.\n\n"
            "The policies require:\n"
            "1. Never read back full account numbers, SSNs, or credit card numbers.\n"
            "2. Verify identity through approved methods only (last 4 digits, "
            "security questions).\n"
            "3. Never share one customer's information with another caller.\n"
            "4. Do not store or repeat sensitive data unnecessarily.\n\n"
            "Classify the response into one of: {evaluation_options}\n\n"
            "- compliant: All data handling policies are followed.\n"
            "- violation: The agent exposed, repeated, or mishandled "
            "sensitive data.\n"
            "- needs_review: The interaction involves sensitive data but "
            "compliance cannot be confidently determined."
        ),
        evaluation_human_prompt=(
            "Review the following interaction for data handling compliance.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Classify the response as one of: {evaluation_options}\n\n"
            "Identify any sensitive data present and assess whether it was "
            "handled correctly."
        ),
        reeval_system_prompt=(
            "You are a senior data handling compliance reviewer performing "
            "a second-pass review. Apply data handling policies strictly.\n\n"
            "Even partial exposure of sensitive data (e.g., reading back 6 "
            "of 16 credit card digits) is a violation.\n\n"
            "You have access to decisions from previous reviews to calibrate "
            "your judgment.\n\n"
            "Classify the response into one of: {evaluation_options}"
        ),
        reeval_human_prompt=(
            "Re-evaluate this interaction for data handling compliance using "
            "context from previous data handling reviews.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Relevant past data handling review decisions:\n{memories}\n\n"
            "Based on the interaction and review history, provide your final "
            "classification."
        ),
    )

    session.update_evaluation_config(
        EvaluationConfig(
            evaluators=[
                response_compliance,
                escalation_procedure,
                data_handling,
            ],
        )
    )
    return {"status": "policy metrics updated, data-handling metric added"}
```

After calling this endpoint, all new interactions are validated against three policy metrics instead of two. Interactions that were already logged continue to use the configuration that was active when they arrived.

```bash
curl -X POST http://localhost:8000/admin/update-metrics
```

```json
{"status": "policy metrics updated, data-handling metric added"}
```

## 7. Monitoring and Alerting Patterns

Once the service is running and interactions are flowing through Prisma, set up monitoring to detect compliance trends and alert on violations.

### Viewing results in Prisma

Open your Prisma instance and navigate to the `contact-center-monitoring` project. The project overview shows:

- **Total validations** -- how many interactions have been validated
- **Per-metric pass rates** -- the percentage of interactions classified as compliant for each policy metric
- **HITL requests** -- interactions routed to human review after ambiguous validation results
- **Violation breakdown** -- which policy metrics are producing the most non-compliant classifications

Click on a specific policy metric to see per-interaction detail: the customer message, the agent response, the classification, and the validator's reasoning.

### Alerting on violation rates

Use the Prisma REST API or analytics integration layer to build alerting on top of your validation data. A typical pattern is a scheduled check that queries violation rates and triggers alerts when compliance drops below a threshold.

```python
"""Compliance alerting — run on a schedule (e.g., every 15 minutes)."""

import httpx
import os

PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]
PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
ALERT_THRESHOLD = 0.90  # Alert if compliance drops below 90%


def check_compliance_rates():
    """Query Prisma for recent validation results and alert on violations."""
    url = f"{PRISMA_BASE_URL}/api/v1/projects/contact-center-monitoring/metrics"

    with httpx.Client(timeout=30) as client:
        resp = client.get(
            url,
            headers={"x-api-key": PRISMA_API_KEY},
            params={"window": "1h"},  # Last hour of validations
        )
        resp.raise_for_status()
        metrics = resp.json()

    for metric_name, metric_data in metrics.items():
        compliance_rate = metric_data.get("compliance_rate", 1.0)
        violation_count = metric_data.get("violation_count", 0)

        if compliance_rate < ALERT_THRESHOLD:
            send_alert(
                metric=metric_name,
                compliance_rate=compliance_rate,
                violation_count=violation_count,
            )


def send_alert(metric: str, compliance_rate: float, violation_count: int):
    """Send an alert to your monitoring system."""
    message = (
        f"COMPLIANCE ALERT: {metric} compliance rate dropped to "
        f"{compliance_rate:.1%} ({violation_count} violations in last hour)"
    )
    # Route to your alerting system: Slack, PagerDuty, email, etc.
    print(message)


if __name__ == "__main__":
    check_compliance_rates()
```

### Key metrics to track

| Metric | What it tells you | Alert threshold (example) |
|---|---|---|
| `response-compliance` compliance rate | Percentage of interactions that pass communication policy validation | Alert if < 90% over a 1-hour window |
| `escalation-procedure` violation rate | How often agents fail to follow the escalation process | Alert if > 5% of escalation-trigger interactions are violations |
| Ambiguous / needs_review rate | How often the validator cannot make a confident classification | Investigate if > 15% -- your metric prompts may need tightening |
| Human review queue depth | How many interactions are waiting for human review | Alert if queue exceeds a time-based SLA (e.g., > 50 pending for > 2 hours) |

## Complete Example

```python
"""Real-time policy monitoring service for contact center interactions."""

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

import prisma_ai
from prisma_ai import EvaluationConfig, Evaluators

logger = logging.getLogger("policy-monitor")

# ---------------------------------------------------------------------------
# 1. Policy metric definitions
# ---------------------------------------------------------------------------
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant", "needs_review"],
    evaluation_system_prompt=(
        "You are a contact center compliance reviewer. Your task is to "
        "determine whether an agent's response follows company communication "
        "policies.\n\n"
        "The policies require:\n"
        "1. PROFESSIONAL TONE: The agent is courteous, empathetic, and avoids "
        "dismissive or confrontational language.\n"
        "2. ACCURATE INFORMATION: The agent provides correct information about "
        "products, services, and procedures. No guessing or speculation.\n"
        "3. REQUIRED DISCLOSURES: When discussing pricing, contracts, or policy "
        "changes, the agent includes all required disclaimers.\n"
        "4. PROHIBITED LANGUAGE: The agent does not make guarantees, promises "
        "about timelines without verification, or statements that could create "
        "legal liability.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- compliant: All policies are followed.\n"
        "- non_compliant: One or more policies are clearly violated.\n"
        "- needs_review: The response is in a gray area where compliance "
        "cannot be confidently determined without additional context."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for policy compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "For each policy (tone, accuracy, disclosures, prohibited language), "
        "state whether it was followed or violated, then provide your classification."
    ),
    reeval_system_prompt=(
        "You are a senior contact center compliance reviewer performing a "
        "second-pass review. Apply the communication policies strictly.\n\n"
        "Pay particular attention to implicit guarantees -- statements that "
        "stop short of a direct promise but still set customer expectations "
        "beyond what the agent can deliver. These are the most common "
        "borderline violations.\n\n"
        "You have access to decisions from previous compliance reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for policy compliance "
        "using context from previous compliance reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past compliance review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification with specific policy findings."
    ),
)

escalation_procedure = Evaluators.custom(
    name="escalation-procedure",
    options=["procedure_followed", "procedure_violated", "not_applicable"],
    evaluation_system_prompt=(
        "You are an escalation process reviewer for a contact center. Your "
        "task is to determine whether an agent followed the correct escalation "
        "procedure when an escalation trigger was present.\n\n"
        "Escalation triggers include:\n"
        "- Customer explicitly requests a supervisor or manager\n"
        "- Customer threatens legal action or mentions an attorney\n"
        "- Customer reports a safety concern or physical harm\n"
        "- Customer expresses intent to contact media or regulators\n\n"
        "When a trigger is present, the required procedure is:\n"
        "1. ACKNOWLEDGE: The agent acknowledges the customer's concern "
        "without being dismissive.\n"
        "2. DOCUMENT: The agent confirms the details of the situation.\n"
        "3. TRANSFER: The agent initiates a transfer to the appropriate "
        "escalation path (supervisor, legal, safety team) without delay.\n\n"
        "If no escalation trigger is present, classify as not_applicable.\n\n"
        "Classify the response into one of: {evaluation_options}\n\n"
        "- procedure_followed: An escalation trigger was present and the "
        "agent followed all three steps.\n"
        "- procedure_violated: An escalation trigger was present but the "
        "agent skipped one or more steps, or attempted to resolve the issue "
        "without escalating.\n"
        "- not_applicable: No escalation trigger was present in the interaction."
    ),
    evaluation_human_prompt=(
        "Review the following contact center interaction for escalation "
        "procedure compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "First determine whether an escalation trigger is present. If so, "
        "assess each step (acknowledge, document, transfer). Then provide "
        "your classification."
    ),
    reeval_system_prompt=(
        "You are a senior escalation process reviewer performing a "
        "second-pass review. Apply the escalation procedure strictly.\n\n"
        "The most critical violation is an agent who attempts to de-escalate "
        "or resolve the issue themselves when a transfer is required. Even "
        "well-intentioned attempts to help the customer directly are procedure "
        "violations when a trigger is present.\n\n"
        "You have access to decisions from previous escalation reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this contact center interaction for escalation procedure "
        "compliance using context from previous escalation reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past escalation review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification. Identify the specific escalation trigger (if any) "
        "and assess each procedure step."
    ),
)


# ---------------------------------------------------------------------------
# 2. FastAPI lifespan — Prisma init and shutdown
# ---------------------------------------------------------------------------
@asynccontextmanager
async def lifespan(app: FastAPI):
    session = prisma_ai.init(
        project_name="contact-center-monitoring",
        evaluators=[response_compliance, escalation_procedure],
        enable_logging=True,
    )
    data_logger = session.get_logger()
    app.state.session = session
    app.state.data_logger = data_logger

    logger.info(
        "Prisma AI initialized — project=%s, policy metrics active",
        session.project_id,
    )

    yield

    app.state.session.end()
    logger.info("Prisma AI session ended, pending logs flushed")


app = FastAPI(title="Policy Monitoring Service", lifespan=lifespan)


# ---------------------------------------------------------------------------
# 3. Request/response models
# ---------------------------------------------------------------------------
class TranscriptEvent(BaseModel):
    query: str
    response: str
    reference: str = ""


class TranscriptResponse(BaseModel):
    status: str


# ---------------------------------------------------------------------------
# 4. Transcript ingestion endpoint
# ---------------------------------------------------------------------------
@app.post("/transcripts", response_model=TranscriptResponse)
async def receive_transcript(event: TranscriptEvent):
    data_logger = app.state.data_logger
    if data_logger is None:
        raise HTTPException(status_code=503, detail="Prisma session not active")

    data_logger.log(
        query=event.query,
        response=event.response,
        reference=event.reference,
    )
    return TranscriptResponse(status="logged")


# ---------------------------------------------------------------------------
# 5. Health check
# ---------------------------------------------------------------------------
@app.get("/health")
async def health():
    session = app.state.session
    if session is None:
        return {"status": "degraded", "prisma": "not initialized"}
    return {
        "status": "healthy",
        "prisma_project_id": session.project_id,
    }


# ---------------------------------------------------------------------------
# 6. Admin: update policy metrics without restart
# ---------------------------------------------------------------------------
@app.post("/admin/update-metrics")
async def update_metrics():
    session = app.state.session
    if session is None:
        return {"error": "Prisma session not active"}

    data_handling = Evaluators.custom(
        name="data-handling",
        options=["compliant", "violation", "needs_review"],
        evaluation_system_prompt=(
            "You are a data handling compliance reviewer for a contact center. "
            "Your task is to determine whether an agent's response properly "
            "handles sensitive customer data.\n\n"
            "The policies require:\n"
            "1. Never read back full account numbers, SSNs, or credit card numbers.\n"
            "2. Verify identity through approved methods only (last 4 digits, "
            "security questions).\n"
            "3. Never share one customer's information with another caller.\n"
            "4. Do not store or repeat sensitive data unnecessarily.\n\n"
            "Classify the response into one of: {evaluation_options}\n\n"
            "- compliant: All data handling policies are followed.\n"
            "- violation: The agent exposed, repeated, or mishandled "
            "sensitive data.\n"
            "- needs_review: The interaction involves sensitive data but "
            "compliance cannot be confidently determined."
        ),
        evaluation_human_prompt=(
            "Review the following interaction for data handling compliance.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Classify the response as one of: {evaluation_options}\n\n"
            "Identify any sensitive data present and assess whether it was "
            "handled correctly."
        ),
        reeval_system_prompt=(
            "You are a senior data handling compliance reviewer performing "
            "a second-pass review. Apply data handling policies strictly.\n\n"
            "Even partial exposure of sensitive data (e.g., reading back 6 "
            "of 16 credit card digits) is a violation.\n\n"
            "You have access to decisions from previous reviews to calibrate "
            "your judgment.\n\n"
            "Classify the response into one of: {evaluation_options}"
        ),
        reeval_human_prompt=(
            "Re-evaluate this interaction for data handling compliance using "
            "context from previous data handling reviews.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Relevant past data handling review decisions:\n{memories}\n\n"
            "Based on the interaction and review history, provide your final "
            "classification."
        ),
    )

    session.update_evaluation_config(
        EvaluationConfig(
            evaluators=[
                response_compliance,
                escalation_procedure,
                data_handling,
            ],
        )
    )
    return {"status": "policy metrics updated, data-handling metric added"}
```

### Production considerations

**Scaling.** The `/transcripts` endpoint is stateless -- the DataLogger handles batching and delivery of OTLP log records internally. You can run multiple instances of this service behind a load balancer. Each instance initializes its own Prisma session and DataLogger, and all instances write to the same project.

**Error handling.** If `data_logger.log()` fails (network issue, Prisma instance unavailable), the OTLP SDK retries with exponential backoff. The `/transcripts` endpoint returns `200` immediately because logging is asynchronous -- the interaction is queued for delivery, not sent synchronously. If Prisma is down for an extended period, the OTLP batch queue will eventually fill and drop records. Monitor your service logs for OTLP delivery warnings.

**Backpressure.** For high-volume contact centers (thousands of interactions per minute), the OTLP batch processor handles backpressure by batching records and flushing at intervals. If your volume consistently exceeds the batch processor's capacity, increase the batch size and flush interval in the OTLP SDK configuration, or partition interactions across multiple Prisma projects by team or product line.

**Graceful shutdown.** Calling `session.end()` in the lifespan shutdown handler is important. It signals the OTLP batch processor to flush any pending log records before the process exits. Without this, interactions logged in the final seconds before shutdown could be lost.

## Next Steps

- [Getting Started](getting-started.md) -- set up Prisma AI and run your first policy validation
- [Building Policy-Based Metrics](policy-based-metrics.md) -- learn the 4-prompt pattern for writing effective policy metrics
- [Human Review & Evaluation Memory](human-review.md) -- make your policy metrics smarter over time with human feedback
- [Policy Compliance in CI/CD](cicd-policy-compliance.md) -- add policy checks to your deployment pipeline

Next: add policy checks to your deployment pipeline.
