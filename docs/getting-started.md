# Getting Started: Validate Agent Interactions in 5 Minutes

Set up Prisma AI to validate contact center agent interactions against your company's response policies using OTLP log ingestion and a custom policy metric.

## What You'll Learn

- Define a **policy metric** that encodes your organization's agent response standards
- Log contact center transcript records via the OTLP logs pipeline
- Validate each interaction automatically against your policy metric
- Identify non-compliant agent behavior in the Prisma dashboard

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **Access to a running Prisma instance**

No third-party API keys are required. Prisma handles all validation internally.

## 1. Install Dependencies

```bash
pip install prisma-ai
```

The `prisma-ai` package is the unified SDK that includes both OTLP log ingestion and the platform client.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Define a Policy Metric

Create a file called `app.py` and define a `response-compliance` policy metric. This metric encodes your company's agent response policy using the **4-prompt pattern**: an evaluation system prompt, an evaluation human prompt, a re-evaluation system prompt, and a re-evaluation human prompt.

```python
import prisma_ai
from prisma_ai import Evaluators

response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],

    # --- Prompt 1: Evaluation system prompt ---
    # Sets the role and criteria for the validator.
    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires all four of the following:\n\n"
        "1. GREETING: The agent uses a professional greeting that includes their "
        "name and the company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action on the account, the "
        "agent verifies the customer's identity by requesting at least two pieces "
        "of identifying information (e.g., account number plus date of birth, or "
        "last four digits of SSN plus mailing address).\n"
        "3. ACCURATE INFORMATION: The agent provides information that is consistent "
        "with company policy. The reference field contains the expected policy-correct "
        "response for this scenario.\n"
        "4. PROFESSIONAL CLOSE: The agent ends the interaction with a summary of "
        "actions taken, clear next steps for the customer, and an offer for further "
        "assistance.\n\n"
        "If ALL four criteria are met, classify the interaction as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify it as 'non_compliant'."
    ),

    # --- Prompt 2: Evaluation human prompt ---
    # Presents the specific interaction to validate.
    evaluation_human_prompt=(
        "Validate the following contact center interaction against company policy.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain which of the four policy criteria "
        "(greeting, identity verification, accurate information, professional close) "
        "were met or violated, then state your classification."
    ),

    # --- Prompt 3: Re-evaluation system prompt ---
    # Used when the initial validation is ambiguous. Provides stricter guidance.
    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the company's agent response policy strictly. "
        "Pay special attention to identity verification — this is the most critical "
        "step in financial services interactions. An agent who skips identity "
        "verification before accessing or modifying account information is always "
        "non-compliant, regardless of how well they performed on other criteria."
    ),

    # --- Prompt 4: Re-evaluation human prompt ---
    # Includes the prior evaluation reasoning for a second pass.
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction but was uncertain. "
        "Here is the interaction and their notes.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Based on the four-point policy (greeting, identity verification, accurate "
        "information, professional close), provide your final classification as "
        "either 'compliant' or 'non_compliant'. Explain your reasoning."
    ),
)
```

The 4-prompt pattern gives you precise control over how Prisma validates each interaction. The evaluation prompts handle the initial pass, and the re-evaluation prompts handle edge cases where the first pass is uncertain.

## 4. Initialize Prisma AI

Add the following to `app.py` to initialize a session with your policy metric:

```python
session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=[response_compliance],
    enable_logging=True,
)

logger = session.get_logger()
```

This does three things:

1. **Creates the project and run** on your Prisma instance.
2. **Registers the policy metric** so every logged interaction is validated against it.
3. **Configures the OTLP logs pipeline** so each `logger.log()` call sends a structured record for validation.

## 5. Log Contact Center Transcripts

Add three transcript records to `app.py`. These represent real interactions between customers and agents at a financial services company. Two are fully compliant; one has a critical policy violation.

```python
transcripts = [
    # Transcript 1: Fully compliant — refund request
    {
        "query": (
            "Hi, I noticed a duplicate charge of $29.99 on my account this month "
            "for my premium subscription. Can I get that corrected?"
        ),
        "response": (
            "Thank you for calling Meridian Financial, my name is Sarah. I'd be happy "
            "to help with that duplicate charge. For security purposes, could you please "
            "verify your account number and the date of birth on file? ... "
            "Thank you, Ms. Chen, I've confirmed your identity. I can see the duplicate "
            "charge of $29.99 posted on March 15th. I've initiated a refund for that "
            "amount — you'll see it back on your card within 3-5 business days. I've also "
            "flagged this with our billing team to prevent it from recurring. Is there "
            "anything else I can help you with today?"
        ),
        "reference": (
            "Duplicate charges should be refunded immediately upon verification. Agent must "
            "verify customer identity before accessing account details. Inform the customer "
            "the refund will appear within 3-5 business days and provide the exact amount. "
            "Flag the issue with the billing team for root cause analysis."
        ),
    },
    # Transcript 2: Fully compliant — account closure with data deletion
    {
        "query": (
            "I'd like to close my investment account and have all my personal data "
            "removed from your systems."
        ),
        "response": (
            "Good afternoon, this is James with Meridian Financial. I'm sorry to hear "
            "you'd like to close your account, but I'm here to help. Before I can process "
            "that, I'll need to verify your identity — could you provide your account number "
            "and the last four digits of your Social Security number? ... "
            "Thank you, Mr. Alvarez, identity confirmed. I've submitted the account closure "
            "request effective today. Per our data retention policy, your personal data will "
            "be purged from our systems within 30 days, in compliance with applicable "
            "regulations. You'll receive a confirmation email within the hour with a summary "
            "of this request and a reference number. If you have any questions during the "
            "30-day period, you can call us back with that reference number. Is there anything "
            "else before we wrap up?"
        ),
        "reference": (
            "Account closure is effective immediately upon request. Agent must verify customer "
            "identity before processing. Inform the customer that personal data deletion "
            "occurs within 30 days per the data retention policy (GDPR Article 17 / CCPA). "
            "Send a confirmation email with a reference number."
        ),
    },
    # Transcript 3: NON-COMPLIANT — skips identity verification on a sensitive request
    {
        "query": (
            "I need to transfer my remaining balance of $4,200 from my savings account "
            "to an external account. Can you do that now?"
        ),
        "response": (
            "Sure, I can process that transfer for you right away. What's the routing "
            "number and account number for the destination account? I'll get that $4,200 "
            "moved over as soon as I have those details."
        ),
        "reference": (
            "Outbound transfers require identity verification of the account holder before "
            "proceeding. Agent must request at least two forms of identification (account "
            "number plus date of birth, or last four SSN digits plus mailing address). "
            "Do not initiate or discuss transfer details until KYC verification is complete, "
            "per Policy FIN-302. After verification, confirm the transfer amount, destination, "
            "and expected settlement time. Close with a summary and reference number."
        ),
    },
]

for transcript in transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Logged: {transcript['query'][:70]}...")

session.end()
```

Notice Transcript 3: the agent jumps straight to processing a $4,200 outbound transfer without verifying the customer's identity. This violates Policy FIN-302 and should be flagged as `non_compliant`.

## 6. Run the Application

```bash
python app.py
```

Expected output:

```
Logged: Hi, I noticed a duplicate charge of $29.99 on my account this month...
Logged: I'd like to close my investment account and have all my personal da...
Logged: I need to transfer my remaining balance of $4,200 from my savings a...
```

Behind the scenes, each `logger.log()` call sends an OTLP log record to your Prisma instance. The validation pipeline picks up each record and runs it against your `response-compliance` policy metric.

## 7. View Results in Prisma

Open Prisma and navigate to the **contact-center-qa** project. The project overview shows:

- **Total Runs**: Active and completed validation runs
- **Validations**: Total validations performed across all policy metrics
- **HITL Requests**: Ambiguous cases queued for human expert review
- **Cost**: Token usage for the validation pipeline

Click into your run to see per-record results. The `response-compliance` metric shows:

| Transcript | Result | Reason |
|---|---|---|
| Duplicate charge refund | `compliant` | Greeting, verification, accurate info, professional close |
| Account closure + data deletion | `compliant` | Greeting, verification, accurate info, professional close |
| Balance transfer request | `non_compliant` | **Skipped identity verification** before processing sensitive transfer |

The third interaction is flagged because the agent asked for destination account details without first verifying the caller's identity — a critical compliance failure in financial services.

## 8. Customize Your Setup

### Rename a run

By default, `init()` generates a random run name. Set your own for easier tracking:

```python
session = prisma_ai.init(
    project_name="contact-center-qa",
    run_name="weekly-qa-review-2026-03",
    evaluators=[response_compliance],
    enable_logging=True,
)
```

### Custom input mapping

If your transcript data uses different field names, specify a custom mapping:

```python
from prisma_ai import InputMapping

session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=[response_compliance],
    enable_logging=True,
    input_mapping=InputMapping(
        query="customer_message",
        response="agent_reply",
        reference="expected_response",
    ),
)
```

### Update config mid-session

Add or change policy metrics while a session is active:

```python
from prisma_ai import EvaluationConfig

session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=[response_compliance],
    enable_logging=True,
)

# Later, add a second policy metric
session.update_evaluation_config(
    EvaluationConfig(
        evaluators=[response_compliance, tone_professionalism],
    )
)
```

## Complete Example

A single file you can copy and run:

```python
"""Prisma AI Getting Started — Validate Contact Center Interactions."""

import prisma_ai
from prisma_ai import Evaluators

# 1. Define a policy metric using the 4-prompt pattern
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],
    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires all four of the following:\n\n"
        "1. GREETING: The agent uses a professional greeting that includes their "
        "name and the company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action on the account, the "
        "agent verifies the customer's identity by requesting at least two pieces "
        "of identifying information (e.g., account number plus date of birth, or "
        "last four digits of SSN plus mailing address).\n"
        "3. ACCURATE INFORMATION: The agent provides information that is consistent "
        "with company policy. The reference field contains the expected policy-correct "
        "response for this scenario.\n"
        "4. PROFESSIONAL CLOSE: The agent ends the interaction with a summary of "
        "actions taken, clear next steps for the customer, and an offer for further "
        "assistance.\n\n"
        "If ALL four criteria are met, classify the interaction as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify it as 'non_compliant'."
    ),
    evaluation_human_prompt=(
        "Validate the following contact center interaction against company policy.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain which of the four policy criteria "
        "(greeting, identity verification, accurate information, professional close) "
        "were met or violated, then state your classification."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the company's agent response policy strictly. "
        "Pay special attention to identity verification — this is the most critical "
        "step in financial services interactions. An agent who skips identity "
        "verification before accessing or modifying account information is always "
        "non-compliant, regardless of how well they performed on other criteria."
    ),
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction but was uncertain. "
        "Here is the interaction and their notes.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Based on the four-point policy (greeting, identity verification, accurate "
        "information, professional close), provide your final classification as "
        "either 'compliant' or 'non_compliant'. Explain your reasoning."
    ),
)

# 2. Initialize Prisma with the policy metric and OTLP logging
session = prisma_ai.init(
    project_name="contact-center-qa",
    evaluators=[response_compliance],
    enable_logging=True,
)

# 3. Get the DataLogger
logger = session.get_logger()

# 4. Log contact center transcripts for validation
transcripts = [
    {
        "query": (
            "Hi, I noticed a duplicate charge of $29.99 on my account this month "
            "for my premium subscription. Can I get that corrected?"
        ),
        "response": (
            "Thank you for calling Meridian Financial, my name is Sarah. I'd be happy "
            "to help with that duplicate charge. For security purposes, could you please "
            "verify your account number and the date of birth on file? ... "
            "Thank you, Ms. Chen, I've confirmed your identity. I can see the duplicate "
            "charge of $29.99 posted on March 15th. I've initiated a refund for that "
            "amount — you'll see it back on your card within 3-5 business days. I've also "
            "flagged this with our billing team to prevent it from recurring. Is there "
            "anything else I can help you with today?"
        ),
        "reference": (
            "Duplicate charges should be refunded immediately upon verification. Agent must "
            "verify customer identity before accessing account details. Inform the customer "
            "the refund will appear within 3-5 business days and provide the exact amount. "
            "Flag the issue with the billing team for root cause analysis."
        ),
    },
    {
        "query": (
            "I'd like to close my investment account and have all my personal data "
            "removed from your systems."
        ),
        "response": (
            "Good afternoon, this is James with Meridian Financial. I'm sorry to hear "
            "you'd like to close your account, but I'm here to help. Before I can process "
            "that, I'll need to verify your identity — could you provide your account number "
            "and the last four digits of your Social Security number? ... "
            "Thank you, Mr. Alvarez, identity confirmed. I've submitted the account closure "
            "request effective today. Per our data retention policy, your personal data will "
            "be purged from our systems within 30 days, in compliance with applicable "
            "regulations. You'll receive a confirmation email within the hour with a summary "
            "of this request and a reference number. If you have any questions during the "
            "30-day period, you can call us back with that reference number. Is there anything "
            "else before we wrap up?"
        ),
        "reference": (
            "Account closure is effective immediately upon request. Agent must verify customer "
            "identity before processing. Inform the customer that personal data deletion "
            "occurs within 30 days per the data retention policy (GDPR Article 17 / CCPA). "
            "Send a confirmation email with a reference number."
        ),
    },
    {
        "query": (
            "I need to transfer my remaining balance of $4,200 from my savings account "
            "to an external account. Can you do that now?"
        ),
        "response": (
            "Sure, I can process that transfer for you right away. What's the routing "
            "number and account number for the destination account? I'll get that $4,200 "
            "moved over as soon as I have those details."
        ),
        "reference": (
            "Outbound transfers require identity verification of the account holder before "
            "proceeding. Agent must request at least two forms of identification (account "
            "number plus date of birth, or last four SSN digits plus mailing address). "
            "Do not initiate or discuss transfer details until KYC verification is complete, "
            "per Policy FIN-302. After verification, confirm the transfer amount, destination, "
            "and expected settlement time. Close with a summary and reference number."
        ),
    },
]

for transcript in transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Logged: {transcript['query'][:70]}...")

# 5. End the session
session.end()
```

## Next Steps

You validated 3 contact center interactions against a policy metric and identified a compliance violation. Next: scale this to thousands of transcripts with batch processing and multiple policy metrics.

- [Evaluating Contact Center Agents](contact-center-agents.md) — scale to hundreds of transcripts with multiple policy metrics
- [Building Policy-Based Metrics](policy-based-metrics.md) — encode your organization's rules as metrics across any industry
- [Human Review & Evaluation Memory](human-review.md) — expert feedback that makes your policy metrics smarter over time
