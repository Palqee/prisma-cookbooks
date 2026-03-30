# Getting Started: Validate Contact Center Interactions in 5 Minutes

Set up Prisma AI to automatically evaluate contact center agent responses for correctness and policy compliance using OTLP log ingestion.

## What You'll Learn

- Install and configure the Prisma AI SDK
- Log contact center transcript records via the OTLP logs pipeline
- Enable automatic correctness and hallucination evaluation on every logged interaction
- Access evaluation results via the Prisma project overview and REST API

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- Access to a running Prisma instance

## 1. Install Dependencies

```bash
pip install prisma-ai
```

The `prisma-ai` package is the unified SDK that includes both OTLP log ingestion (`prisma-otel`) and the platform client (`pq-prisma-client`).

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Initialize Prisma AI

Create a file called `app.py`:

```python
import prisma_ai

# Initialize Prisma: creates a project, a run, configures OTLP logging,
# and sets up automatic evaluation on every logged record
session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=["correctness", "hallucination"],
    enable_logging=True,
)

# Get the DataLogger — each .log() call sends an OTLP log record
# to Prisma for automatic evaluation
logger = session.get_logger()
```

That's it. Three lines to set up logging and evaluation:

1. `prisma_ai.init()` creates your project and run on the Prisma platform, stores the evaluation configuration, and configures the OTLP logs pipeline.
2. `enable_logging=True` activates the DataLogger, which sends structured records as OTLP log signals.
3. Each logged record flows to your Prisma instance, where the evaluation pipeline automatically scores it for correctness and hallucination.

## 4. Log Contact Center Transcripts

Add the following to `app.py`. Each record represents a real interaction between a customer and a contact center agent, along with the expected (policy-correct) response:

```python
# Sample contact center transcripts — in production, these come from
# your CRM, telephony system, or transcript pipeline
transcripts = [
    {
        "query": "I was charged twice for my subscription this month. Can I get a refund?",
        "response": "I can see the duplicate charge on your account. I've initiated a refund for the second charge of $29.99. You'll see it back on your card within 3-5 business days.",
        "reference": "Duplicate charges should be refunded immediately. Inform the customer the refund will appear within 3-5 business days and provide the exact amount.",
    },
    {
        "query": "I want to cancel my account and delete all my data.",
        "response": "I've cancelled your account effective immediately. Your data will be deleted within 30 days per our retention policy. You'll receive a confirmation email shortly.",
        "reference": "Account cancellation is immediate. Inform the customer that data deletion occurs within 30 days per the data retention policy (GDPR Article 17). Send a confirmation email.",
    },
    {
        "query": "Can you transfer my remaining balance to a different account?",
        "response": "Sure, I can do that right away. What's the account number you'd like to transfer to?",
        "reference": "Balance transfers between accounts require identity verification of both account holders before proceeding. Do not initiate the transfer without completing KYC verification per Policy FIN-302.",
    },
]

for transcript in transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Customer: {transcript['query']}")
    print(f"Agent:    {transcript['response']}")
    print()

session.end()
```

Notice the third transcript: the agent skipped identity verification before a balance transfer. Prisma's evaluation pipeline will flag this as a correctness failure against the reference policy.

## 5. Run the Application

```bash
python app.py
```

Expected output:

```
Customer: I was charged twice for my subscription this month. Can I get a refund?
Agent:    I can see the duplicate charge on your account. I've initiated a refund for the second charge of $29.99. You'll see it back on your card within 3-5 business days.

Customer: I want to cancel my account and delete all my data.
Agent:    I've cancelled your account effective immediately. Your data will be deleted within 30 days per our retention policy. You'll receive a confirmation email shortly.

Customer: Can you transfer my remaining balance to a different account?
Agent:    Sure, I can do that right away. What's the account number you'd like to transfer to?
```

Behind the scenes, each `logger.log()` call sends an OTLP log record to your Prisma instance. The evaluation pipeline picks up these records and runs correctness and hallucination checks automatically against the reference answers.

## 6. View Results in Prisma

Open Prisma and navigate to the **contact-center-qa** project. The project overview shows:

- **Total Runs**: How many evaluation runs are active or completed
- **Validations**: Total validations performed across all metrics
- **HITL Requests**: Ambiguous cases queued for human expert review
- **LLM Cost**: Token usage and cost for the evaluation pipeline

Click on your run to see per-metric summaries — pass rates, failure counts, and ambiguous classifications for correctness and hallucination. The third transcript (balance transfer without KYC) should show as a correctness failure, demonstrating how Prisma catches policy violations in agent responses.

For granular access to individual evaluation results, use the Prisma REST API or connect your preferred analytics tool (Power BI, Tableau, Grafana) via Prisma's analytics integration layer to build custom views over your evaluation data.

## 7. Customize Your Setup

### Change the run name

By default, `init()` generates a random run name. You can set your own:

```python
session = prisma_ai.init(
    project_name="contact-center-qa",
    run_name="weekly-qa-review-2026-03",
    evaluators=["correctness", "hallucination"],
    enable_logging=True,
)
```

### Update evaluation config mid-session

```python
from prisma_ai import EvaluationConfig

session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=["correctness"],
    enable_logging=True,
)

# Later, add hallucination detection
session.update_evaluation_config(
    EvaluationConfig(
        evaluators=["correctness", "hallucination"],
    )
)
```

### Custom input mapping

If your transcript data uses different field names, specify a custom mapping:

```python
from prisma_ai import InputMapping

session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=["correctness", "hallucination"],
    enable_logging=True,
    input_mapping=InputMapping(
        query="customer_message",
        response="agent_reply",
        reference="expected_response",
    ),
)
```

## Complete Example

```python
"""Prisma AI Getting Started — Contact Center QA."""

import prisma_ai

# 1. Initialize Prisma with evaluation and OTLP logging
session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=["correctness", "hallucination"],
    enable_logging=True,
)

# 2. Get the DataLogger
logger = session.get_logger()

# 3. Log contact center transcripts for evaluation
transcripts = [
    {
        "query": "I was charged twice for my subscription this month. Can I get a refund?",
        "response": "I can see the duplicate charge on your account. I've initiated a refund for the second charge of $29.99. You'll see it back on your card within 3-5 business days.",
        "reference": "Duplicate charges should be refunded immediately. Inform the customer the refund will appear within 3-5 business days and provide the exact amount.",
    },
    {
        "query": "I want to cancel my account and delete all my data.",
        "response": "I've cancelled your account effective immediately. Your data will be deleted within 30 days per our retention policy. You'll receive a confirmation email shortly.",
        "reference": "Account cancellation is immediate. Inform the customer that data deletion occurs within 30 days per the data retention policy (GDPR Article 17). Send a confirmation email.",
    },
    {
        "query": "Can you transfer my remaining balance to a different account?",
        "response": "Sure, I can do that right away. What's the account number you'd like to transfer to?",
        "reference": "Balance transfers between accounts require identity verification of both account holders before proceeding. Do not initiate the transfer without completing KYC verification per Policy FIN-302.",
    },
]

for transcript in transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Customer: {transcript['query']}")
    print(f"Agent:    {transcript['response']}\n")

# 4. Clean up (optional — also runs automatically on exit)
session.end()
```

## Next Steps

- [Automated QA for Call Center Transcripts](call-center-qa.md) — scale to batch evaluation of thousands of transcripts with custom metrics
- [Custom Policy Evaluators](custom-policy-evaluators.md) — build domain-specific evaluators for regulated industries
- [Human-in-the-Loop Feedback](hitl-feedback-loop.md) — route ambiguous cases to expert reviewers
