# Human-in-the-Loop: Expert Feedback That Makes Evaluations Smarter

Close the feedback loop between automated evaluation and human expertise. When Prisma encounters ambiguous cases, expert reviewers provide feedback that becomes evaluation memory — giving future evaluations context they did not have before.

## What You'll Learn

- Understand how ambiguous evaluations are routed to the human review queue
- Retrieve pending HITL review requests via the Prisma REST API
- Submit expert feedback that becomes evaluation memory (controls)
- See how future evaluations use that feedback as additional context via the `{memories}` placeholder
- Plan for integrating HITL reviews into tools like Jira, Slack, ServiceNow, or Zendesk

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- An OpenAI API key
- Access to a running Prisma instance
- A project with at least one run that has produced ambiguous evaluations (see [Getting Started](getting-started.md))

## How the Feedback Loop Works

```
┌─────────────┐     ┌──────────────┐     ┌───────────────────┐
│ LLM         │────▶│ Prisma       │────▶│ Result:           │
│ Interaction │     │ Evaluators   │     │ correct / fail /  │
└─────────────┘     └──────────────┘     │ AMBIGUOUS         │
                                          └────────┬──────────┘
                                                   │
                                          ambiguous cases only
                                                   │
                                                   ▼
                                          ┌───────────────────┐
                                          │ HITL Review Queue  │
                                          │ (API / Jira /      │
                                          │  Slack / Zendesk)  │
                                          └────────┬──────────┘
                                                   │
                                          expert submits feedback
                                                   │
                                                   ▼
                                          ┌───────────────────┐
                                          │ Control Created    │
                                          │ (evaluation memory)│
                                          └────────┬──────────┘
                                                   │
                                          future evaluations
                                                   │
                                                   ▼
                                          ┌───────────────────┐
                                          │ Re-evaluation with │
                                          │ {memories} context │
                                          │ = more confident   │
                                          └───────────────────┘
```

When an evaluator classifies a result as **ambiguous**, Prisma automatically queues it for human review. This appears as a pending HITL request in the project overview. Once an expert submits feedback, Prisma stores it as a **control** — a piece of evaluation memory scoped to the project. When similar cases appear in the future (in the same run or subsequent runs), the re-evaluation prompts receive this memory via the `{memories}` placeholder, allowing the evaluator to make more confident decisions.

Over time, your evaluation pipeline builds institutional knowledge from expert decisions — the same way a senior reviewer trains new auditors.

## 1. Install Dependencies

```bash
pip install prisma-ai httpx openai openinference-instrumentation-openai
```

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
export OPENAI_API_KEY="sk-your-openai-key"
```

## 3. Generate Ambiguous Evaluations

First, let's create a scenario that produces ambiguous evaluations. We'll set up a custom evaluator for "policy adherence" in a financial services context, then send interactions that are genuinely borderline — not clearly compliant or non-compliant.

```python
import prisma_ai
from prisma_ai import Evaluators
from openai import OpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor

policy_evaluator = Evaluators.custom(
    name="policy-adherence",
    options=["compliant", "non_compliant"],
    evaluation_system_prompt=(
        "You are a financial services compliance auditor evaluating whether "
        "agent responses adhere to company policy. Evaluate based on:\n"
        "1. Were required disclosures provided?\n"
        "2. Were unauthorized promises avoided?\n"
        "3. Was the customer directed to the appropriate channel?\n\n"
        "If the response is clearly compliant or clearly non-compliant, classify it. "
        "If it is borderline or you are uncertain, classify as ambiguous.\n\n"
        "Options: {evaluation_options}"
    ),
    evaluation_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Classify the agent's policy adherence. Choose from: {evaluation_options}"
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor performing a second review of an "
        "ambiguous policy adherence evaluation. You have access to feedback from "
        "prior expert reviews of similar cases. Use this context to make a more "
        "confident classification.\n\n"
        "Options: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Context from prior expert reviews:\n{memories}\n\n"
        "Based on the interaction and expert review context, classify the agent's "
        "policy adherence. Choose from: compliant or non_compliant."
    ),
)

# Initialize Prisma with the custom evaluator
session = prisma_ai.init(
    project_name="hitl-demo",
    run_name="feedback-loop-demo",
    evaluators=["correctness", policy_evaluator],
)

OpenAIInstrumentor().instrument()
client = OpenAI()
```

Now send borderline interactions that are likely to produce ambiguous evaluations:

```python
borderline_interactions = [
    {
        "system": "You are a bank customer service agent. Follow company policy strictly.",
        "user": "Can I get a higher credit limit? I've been a customer for 15 years.",
    },
    {
        "system": "You are a bank customer service agent. Follow company policy strictly.",
        "user": "I want to transfer $50,000 internationally. Can you process that now?",
    },
    {
        "system": "You are a bank customer service agent. Follow company policy strictly.",
        "user": "My financial advisor told me this account has guaranteed returns. Is that right?",
    },
]

for interaction in borderline_interactions:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": interaction["system"]},
            {"role": "user", "content": interaction["user"]},
        ],
        temperature=0.7,
    )
    answer = response.choices[0].message.content or ""
    print(f"Q: {interaction['user']}")
    print(f"A: {answer[:120]}...\n")
```

These interactions are designed to be borderline: the LLM may give partially correct advice, miss a required disclosure, or handle the request in a way that isn't clearly compliant or non-compliant. The evaluator will classify some as **ambiguous**, which routes them to the HITL review queue.

## 4. Retrieve Pending HITL Reviews

After the evaluation pipeline processes the traces, ambiguous cases appear as pending HITL requests. Retrieve them via the REST API:

```python
import httpx

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]

headers = {
    "x-api-key": PRISMA_API_KEY,
    "Content-Type": "application/json",
}

# List pending HITL reviews for this project
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews",
    headers=headers,
    params={
        "project_id": session.project_id,
        "status": "pending",
    },
)
reviews = response.json()["reviews"]

print(f"Pending reviews: {len(reviews)}\n")

for review in reviews:
    print(f"Review ID: {review['id']}")
    print(f"  Evaluator: {review['evaluator_type']}")
    print(f"  Query: {review['query'][:80]}...")
    print(f"  Response: {review['response'][:80]}...")
    print(f"  Initial classification: {review['classification']} (ambiguous)")
    print(f"  Confidence: {review['confidence']:.2f}")
    print()
```

Expected output:

```
Pending reviews: 2

Review ID: rev_abc123
  Evaluator: policy-adherence
  Query: Can I get a higher credit limit? I've been a customer for 15 years....
  Response: I appreciate your loyalty! I can certainly look into that for you. Based on...
  Initial classification: ambiguous
  Confidence: 0.42

Review ID: rev_def456
  Evaluator: policy-adherence
  Query: My financial advisor told me this account has guaranteed returns. Is that ...
  Response: Thank you for reaching out. While I can't comment on what your advisor may...
  Initial classification: ambiguous
  Confidence: 0.38
```

Each review contains the original interaction, the evaluator that flagged it, and the confidence score that led to the ambiguous classification.

## 5. Submit Expert Feedback

An expert reviewer examines each case and submits a classification with reasoning. This feedback becomes a **control** — evaluation memory that Prisma uses for future re-evaluations.

```python
# Expert reviews the first case and determines it's actually compliant
# but missing a required disclosure about credit check impact
feedback_1 = httpx.post(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/{reviews[0]['id']}/feedback",
    headers=headers,
    json={
        "classification": "compliant",
        "reasoning": (
            "The agent correctly offered to review the credit limit request and "
            "referenced the customer's account history. However, the agent should "
            "have disclosed that a credit limit increase triggers a hard credit "
            "inquiry. Mark as compliant with a note about the missing disclosure — "
            "this is a coaching opportunity, not a policy violation."
        ),
        "reviewer": "jane.doe@acmecorp.com",
        "tags": ["credit-limit", "disclosure-gap", "coaching"],
    },
)
print(f"Feedback submitted: {feedback_1.json()['status']}")

# Expert reviews the second case and determines it's non-compliant
feedback_2 = httpx.post(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/{reviews[1]['id']}/feedback",
    headers=headers,
    json={
        "classification": "non_compliant",
        "reasoning": (
            "The customer referenced 'guaranteed returns' from their financial "
            "advisor. The agent should have immediately corrected this — no "
            "investment product has guaranteed returns, and failing to address "
            "this creates regulatory risk under SEC/FINRA suitability rules. "
            "The agent's response was evasive rather than corrective."
        ),
        "reviewer": "jane.doe@acmecorp.com",
        "tags": ["guaranteed-returns", "suitability", "regulatory-risk"],
    },
)
print(f"Feedback submitted: {feedback_2.json()['status']}")
```

The `reasoning` field is critical — this is the text that appears in `{memories}` during future re-evaluations. Write it as you would coach a junior auditor: explain the decision, cite the relevant policy, and note what the agent should have done differently.

The `tags` field helps organize controls by topic, making it easier to find and manage evaluation memory as it grows.

## 6. How Feedback Becomes Evaluation Memory

Once feedback is submitted, Prisma creates a **control** — a stored piece of evaluation memory scoped to the project. Here's what happens when a similar interaction appears in the future:

1. The evaluator runs its initial classification
2. If the result is ambiguous (or even as part of routine re-evaluation), the re-evaluation prompt fires
3. Prisma retrieves relevant controls based on the evaluator type and interaction similarity
4. These controls are injected into the `{memories}` placeholder in the `reeval_human_prompt`

For example, the next time a customer asks about guaranteed returns, the re-evaluation prompt will include:

```
Context from prior expert reviews:
- [2026-03-27] Reviewer: jane.doe@acmecorp.com | Classification: non_compliant
  "The customer referenced 'guaranteed returns' from their financial advisor.
  The agent should have immediately corrected this — no investment product
  has guaranteed returns, and failing to address this creates regulatory risk
  under SEC/FINRA suitability rules. The agent's response was evasive rather
  than corrective."
```

This gives the evaluator the context to classify confidently rather than marking it ambiguous again. Over time, the number of ambiguous cases decreases as the evaluation pipeline accumulates expert knowledge.

## 7. Verify the Feedback Loop

Send a similar interaction and observe that it no longer produces an ambiguous result:

```python
# Send a similar "guaranteed returns" interaction
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a bank customer service agent."},
        {"role": "user", "content": "My broker promised me guaranteed returns on this product. Can you confirm?"},
    ],
    temperature=0.7,
)
print(f"Response: {response.choices[0].message.content}")
```

This interaction will be traced and evaluated as before. But during re-evaluation, the evaluator now has the expert's reasoning about guaranteed returns claims. Instead of classifying it as ambiguous, it can confidently determine whether the agent's response appropriately corrected the customer's misconception.

Check the HITL review count to confirm fewer ambiguous cases:

```python
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews",
    headers=headers,
    params={
        "project_id": session.project_id,
        "status": "pending",
    },
)
pending = response.json()["reviews"]
print(f"Pending reviews after feedback: {len(pending)}")
```

## 8. List Controls (Evaluation Memory)

You can retrieve all controls for a project to audit the accumulated evaluation memory:

```python
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/controls",
    headers=headers,
    params={"project_id": session.project_id},
)
controls = response.json()["controls"]

print(f"Total controls: {len(controls)}\n")

for control in controls:
    print(f"Control ID: {control['id']}")
    print(f"  Evaluator: {control['evaluator_type']}")
    print(f"  Classification: {control['classification']}")
    print(f"  Reasoning: {control['reasoning'][:100]}...")
    print(f"  Tags: {control['tags']}")
    print(f"  Created: {control['created_at']}")
    print(f"  Reviewer: {control['reviewer']}")
    print()
```

Controls are the institutional knowledge your evaluation pipeline builds over time. They are project-scoped, so different projects can have different evaluation standards reflecting their domain requirements.

## 9. Bulk Operations

For teams processing high volumes of HITL reviews, you can export pending reviews and import feedback in bulk:

```python
# Export all pending reviews as JSON
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/export",
    headers=headers,
    params={
        "project_id": session.project_id,
        "status": "pending",
        "format": "json",
    },
)

# Save for offline review or import into another system
import json
with open("pending_reviews.json", "w") as f:
    json.dump(response.json(), f, indent=2)
print(f"Exported {len(response.json()['reviews'])} reviews to pending_reviews.json")

# After expert review, submit feedback in bulk
reviewed_feedback = [
    {
        "review_id": "rev_abc123",
        "classification": "compliant",
        "reasoning": "Agent followed correct procedure...",
        "reviewer": "jane.doe@acmecorp.com",
    },
    {
        "review_id": "rev_ghi789",
        "classification": "non_compliant",
        "reasoning": "Agent failed to verify identity...",
        "reviewer": "jane.doe@acmecorp.com",
    },
]

response = httpx.post(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/bulk-feedback",
    headers=headers,
    json={"feedback": reviewed_feedback},
)
result = response.json()
print(f"Bulk feedback submitted: {result['accepted']} accepted, {result['rejected']} rejected")
```

## 10. Integrating with External Tools

The HITL review workflow is designed to integrate with your existing operations tools. While direct integrations are on the roadmap, you can connect Prisma's HITL queue to external systems today using the REST API patterns shown above.

**Planned integrations:**

| Tool | Use Case |
|------|----------|
| **Jira / Azure DevOps** | Create review tickets automatically when ambiguous cases are detected. Experts resolve tickets; a webhook submits feedback back to Prisma. |
| **Slack / Microsoft Teams** | Push pending reviews to a dedicated channel. Experts react or reply with their classification. |
| **ServiceNow** | Route reviews through existing compliance workflows with SLA tracking. |
| **Zendesk** | Attach reviews to customer interaction records for end-to-end QA traceability. |
| **Power BI / Grafana** | Visualize HITL trends: ambiguity rate over time, reviewer agreement, feedback volume by evaluator type. |

**Webhook pattern (available now):**

You can poll for new HITL requests and push them to external systems:

```python
import time

def poll_and_dispatch(project_id: str, interval_seconds: int = 60):
    """Poll for new HITL reviews and dispatch to external systems."""
    seen_ids = set()

    while True:
        response = httpx.get(
            f"{PRISMA_BASE_URL}/api/v1/hitl/reviews",
            headers=headers,
            params={"project_id": project_id, "status": "pending"},
        )
        reviews = response.json()["reviews"]

        for review in reviews:
            if review["id"] not in seen_ids:
                seen_ids.add(review["id"])
                # Dispatch to your tool of choice:
                # send_to_jira(review)
                # send_to_slack(review)
                # send_to_servicenow(review)
                print(f"New review: {review['id']} — {review['evaluator_type']}")

        time.sleep(interval_seconds)
```

In production, replace the polling pattern with a webhook subscription once available, or use a scheduled job to sync pending reviews to your external tool on a regular cadence.

## Production Considerations

### Review SLAs

Track how long reviews sit in the pending queue. Stale reviews mean your evaluation pipeline is not learning. Set up alerts when pending review count exceeds a threshold or when reviews are older than your target SLA.

### Reviewer Calibration

Multiple reviewers may classify the same case differently. Periodically review controls for consistency — the tags field helps you find clusters of related decisions. If reviewers disagree on similar cases, that signals a policy ambiguity that should be resolved upstream.

### Control Lifecycle

Controls accumulate over time. Periodically audit them to ensure they still reflect current policy. If company policy changes, outdated controls may cause the evaluation pipeline to apply stale reasoning. Use the tags and creation dates to identify controls that may need updating.

## Next Steps

- [Getting Started](getting-started.md) — set up Prisma AI tracing and evaluation from scratch
- [Build Custom Policy Compliance Evaluators](custom-policy-evaluators.md) — design the evaluators that produce the ambiguous cases this workflow resolves
- [Real-Time Evaluation of Production LLM Traffic](realtime-production-evaluation.md) — see how real-time tracing feeds the HITL queue automatically
