# Human Review & Evaluation Memory

A QA lead notices that some validations come back as "ambiguous" — the policy metric can't confidently decide compliant vs non-compliant. A human expert reviews the case, makes a judgment call, and that decision becomes memory for all future validations of similar cases. Over time, ambiguous rates drop. This is the "aha moment": Prisma doesn't just validate, it **learns from your team's expertise**.

## What You'll Learn

- Why some validations produce ambiguous results and what triggers the human review queue
- How to retrieve pending reviews and submit expert feedback via the Prisma REST API
- How feedback becomes **evaluation memory** (controls) that future validations use automatically
- The flywheel effect: how a batch of expert reviews drops ambiguous rates from ~30% to ~5%
- Operational patterns for managing controls, bulk reviews, and audit exports

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **Access to a running Prisma instance**
- Familiarity with the [Getting Started](getting-started.md) guide (policy metric definition, OTLP logging)

No third-party API keys are required. Prisma handles all validation internally.

## How the Feedback Loop Works

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│ Interaction      │────▶│ Policy Metric    │────▶│ Result:             │
│ (transcript)     │     │ Validation       │     │ compliant / non_    │
└─────────────────┘     └──────────────────┘     │ compliant / AMBIG.  │
                                                  └──────────┬──────────┘
                                                             │
                                                    ambiguous cases only
                                                             │
                                                             ▼
                                                  ┌─────────────────────┐
                                                  │ HITL Review Queue    │
                                                  │ (REST API)           │
                                                  └──────────┬──────────┘
                                                             │
                                                    expert submits feedback
                                                             │
                                                             ▼
                                                  ┌─────────────────────┐
                                                  │ Control Created      │
                                                  │ (evaluation memory)  │
                                                  └──────────┬──────────┘
                                                             │
                                                    future validations
                                                             │
                                                             ▼
                                                  ┌─────────────────────┐
                                                  │ Re-validation with   │
                                                  │ {memories} context   │
                                                  │ = confident result   │
                                                  └─────────────────────┘
```

When a policy metric classifies a result as **ambiguous**, Prisma automatically queues it for human review. Once an expert submits feedback, Prisma stores it as a **control** — a piece of evaluation memory scoped to the project. When similar cases appear in future validations (in the same run or subsequent runs), the re-evaluation prompts receive this memory via the `{memories}` placeholder, allowing the metric to make more confident decisions.

Every expert review makes the system better. The validation pipeline builds institutional knowledge from your team's decisions — the same way a senior reviewer trains new auditors.

## 1. Generate Ambiguous Validations

Start by defining a `response-compliance` policy metric and logging a borderline interaction — one where the agent verified the customer's identity informally but didn't follow the exact scripted process.

Create a file called `human_review_demo.py`:

```python
import prisma_ai
from prisma_ai import Evaluators

# Define a policy metric with strict identity verification criteria
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],

    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires:\n\n"
        "1. GREETING: Professional greeting with agent name and company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action, the agent must "
        "verify the customer's identity by requesting at least two pieces of "
        "identifying information (account number + date of birth, or last four "
        "SSN digits + mailing address).\n"
        "3. ACCURATE INFORMATION: Response consistent with company policy.\n"
        "4. PROFESSIONAL CLOSE: Summary of actions, next steps, offer for "
        "further assistance.\n\n"
        "If ALL four criteria are met, classify as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify as 'non_compliant'."
    ),

    evaluation_human_prompt=(
        "Validate the following contact center interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain your reasoning."
    ),

    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the policy strictly. Pay special attention "
        "to identity verification — agents must follow the exact scripted process, "
        "not an informal or abbreviated version.\n\n"
        "Use the evaluation memory below to guide your decision. These are prior "
        "expert rulings on similar borderline cases."
    ),

    reeval_human_prompt=(
        "A previous auditor was uncertain about this interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes and evaluation memory:\n{memories}\n\n"
        "Provide your final classification as 'compliant' or 'non_compliant'. "
        "Explain your reasoning."
    ),
)

# Initialize Prisma with the policy metric
session = prisma_ai.init(
    project_name="compliance-review-demo",
    evaluators=[response_compliance],
    enable_logging=True,
)

logger = session.get_logger()
```

Now log several borderline interactions. These are deliberately ambiguous — the agent does *something* to verify identity, but it doesn't match the scripted two-factor process exactly:

```python
borderline_transcripts = [
    {
        "query": (
            "Hi, I need to update the mailing address on my account."
        ),
        "response": (
            "Good morning, this is Rebecca with Meridian Financial. I'd be happy to "
            "help with that address change. I can see your account is already pulled "
            "up from your phone number on file — can you just confirm the last four "
            "digits of your account number? Great, thank you. I've updated your "
            "mailing address to 742 Evergreen Terrace. You'll receive a confirmation "
            "letter at the new address within 5 business days. Anything else I can "
            "help with today?"
        ),
        "reference": (
            "Address changes require two-factor identity verification: account number "
            "plus date of birth, or last four SSN digits plus current mailing address. "
            "Phone number caller ID does not count as a verification factor. Confirm "
            "the change and send written confirmation."
        ),
    },
    {
        "query": (
            "I'd like to increase my credit card limit from $5,000 to $10,000."
        ),
        "response": (
            "Hello, this is Marcus at Meridian Financial. I can look into a credit "
            "limit increase for you. Before I proceed, could you verify your date of "
            "birth? Perfect, thank you. I've submitted a request to increase your "
            "limit to $10,000. You'll receive a decision by email within 2 business "
            "days. Is there anything else I can assist with?"
        ),
        "reference": (
            "Credit limit increases require full identity verification with two "
            "factors before processing. Agent must request account number plus date "
            "of birth, or last four SSN digits plus mailing address. A single "
            "verification factor is insufficient per Policy FIN-201."
        ),
    },
    {
        "query": (
            "Can you tell me the interest rate on my savings account?"
        ),
        "response": (
            "Thanks for calling Meridian Financial, I'm Dana. Sure, let me look "
            "that up — I just need to confirm who I'm speaking with. Can you give "
            "me your account number and the last four of your Social? Perfect. Your "
            "savings account is currently earning 3.25% APY. That rate is variable "
            "and reviewed quarterly. Would you like to hear about our fixed-rate CD "
            "options? Great, is there anything else I can help with?"
        ),
        "reference": (
            "For account inquiries, verify identity with two factors before "
            "disclosing any account details. Provide the current rate and note "
            "whether it is fixed or variable. Offer related products if appropriate."
        ),
    },
    {
        "query": (
            "I need to dispute a charge of $89.99 from an online retailer."
        ),
        "response": (
            "Hi there, Marcus here at Meridian Financial. I'm sorry to hear about "
            "that charge. Let me pull up your account — could you confirm your name "
            "and the last four digits of the card? OK, I can see the $89.99 charge. "
            "I've opened a dispute and you'll receive provisional credit within 10 "
            "business days. We'll notify you of the outcome. Anything else?"
        ),
        "reference": (
            "Disputes require identity verification with two factors: account number "
            "plus date of birth, or last four SSN plus mailing address. Name and "
            "partial card number alone do not meet the two-factor requirement per "
            "Policy SEC-105. After verification, open the dispute and confirm the "
            "provisional credit timeline."
        ),
    },
]

for transcript in borderline_transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Logged: {transcript['query'][:60]}...")

session.end()
```

Run the script:

```bash
python human_review_demo.py
```

Expected output:

```
Logged: Hi, I need to update the mailing address on my account....
Logged: I'd like to increase my credit card limit from $5,000 t...
Logged: Can you tell me the interest rate on my savings account?...
Logged: I need to dispute a charge of $89.99 from an online ret...
```

Behind the scenes, Prisma validates each interaction against the `response-compliance` metric. The third transcript (savings account inquiry) should pass — the agent used two proper verification factors. The others are borderline: the agents verified identity, but used only one factor, or used factors that don't meet the scripted requirements (caller ID, name + partial card number). These are the cases that land in the ambiguous queue.

## 2. Retrieve Pending Reviews

Once validation completes, check the Prisma dashboard. Under the **compliance-review-demo** project, you'll see HITL requests queued for expert review. You can also retrieve them programmatically via the REST API.

Create a file called `review_pending.py`:

```python
import httpx
import os
import json

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]

headers = {
    "Authorization": f"Bearer {PRISMA_API_KEY}",
    "Content-Type": "application/json",
}

# List pending HITL reviews for the project
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews",
    params={"project_id": "your-project-id", "status": "pending"},
    headers=headers,
)

reviews = response.json()
print(f"Pending reviews: {len(reviews['items'])}")

for review in reviews["items"]:
    print(f"\n--- Review {review['id']} ---")
    print(f"  Metric: {review['evaluator_name']}")
    print(f"  Result: {review['classification']} (ambiguous)")
    print(f"  Query:  {review['input']['query'][:80]}...")
    print(f"  Response: {review['input']['response'][:80]}...")
```

```bash
python review_pending.py
```

Expected output:

```
Pending reviews: 3

--- Review rev_a1b2c3 ---
  Metric: response-compliance
  Result: ambiguous (ambiguous)
  Query:  Hi, I need to update the mailing address on my account....
  Response: Good morning, this is Rebecca with Meridian Financial. I'd be happy to help...

--- Review rev_d4e5f6 ---
  Metric: response-compliance
  Result: ambiguous (ambiguous)
  Query:  I'd like to increase my credit card limit from $5,000 to $10,000....
  Response: Hello, this is Marcus at Meridian Financial. I can look into a credit limit...

--- Review rev_g7h8i9 ---
  Metric: response-compliance
  Result: ambiguous (ambiguous)
  Query:  I need to dispute a charge of $89.99 from an online retailer....
  Response: Hi there, Marcus here at Meridian Financial. I'm sorry to hear about that c...
```

Three of the four interactions are ambiguous. The savings account inquiry passed because the agent used two proper verification factors (account number + last four SSN). The other three used informal or incomplete verification — exactly the kind of edge case that needs a human expert.

## 3. Submit Expert Feedback

Now the QA lead examines each case and submits a judgment. This is the critical step: the expert's reasoning becomes part of Prisma's evaluation memory.

Create a file called `submit_feedback.py`:

```python
import httpx
import os

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]

headers = {
    "Authorization": f"Bearer {PRISMA_API_KEY}",
    "Content-Type": "application/json",
}

# Expert reviews each ambiguous case and submits feedback
feedback_decisions = [
    {
        "review_id": "rev_a1b2c3",  # Address change — caller ID + partial account
        "classification": "non_compliant",
        "reasoning": (
            "The agent used caller ID (phone number on file) as one verification "
            "factor and only requested the last four digits of the account number "
            "as the second. Caller ID does not count as a verification factor per "
            "Policy SEC-105. The agent needed to request a second proper factor "
            "(date of birth or mailing address) in addition to the account number. "
            "This is non-compliant despite the agent's good intent."
        ),
        "reviewer": "jane.doe@company.com",
        "tags": ["identity-verification", "single-factor", "address-change"],
    },
    {
        "review_id": "rev_d4e5f6",  # Credit limit increase — single factor only
        "classification": "non_compliant",
        "reasoning": (
            "The agent requested only date of birth as a verification factor. "
            "Policy FIN-201 requires two distinct factors for credit limit changes. "
            "The agent should have also requested the account number or last four "
            "SSN digits. A single factor is insufficient regardless of the factor's "
            "strength. Non-compliant."
        ),
        "reviewer": "jane.doe@company.com",
        "tags": ["identity-verification", "single-factor", "credit-limit"],
    },
    {
        "review_id": "rev_g7h8i9",  # Dispute — name + partial card
        "classification": "non_compliant",
        "reasoning": (
            "The agent verified the customer's name and last four digits of the "
            "card number. Neither of these qualifies as a verification factor under "
            "Policy SEC-105 — name is not a secret, and the card number is printed "
            "on the physical card. The agent needed account number + date of birth "
            "or last four SSN + mailing address. Non-compliant."
        ),
        "reviewer": "jane.doe@company.com",
        "tags": ["identity-verification", "weak-factors", "dispute"],
    },
]

for decision in feedback_decisions:
    review_id = decision.pop("review_id")
    response = httpx.post(
        f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/{review_id}/feedback",
        json=decision,
        headers=headers,
    )
    print(f"Feedback submitted for {review_id}: {response.status_code}")
```

```bash
python submit_feedback.py
```

Expected output:

```
Feedback submitted for rev_a1b2c3: 200
Feedback submitted for rev_d4e5f6: 200
Feedback submitted for rev_g7h8i9: 200
```

Each feedback submission does two things: it resolves the pending review, and it creates a **control** — a piece of evaluation memory that Prisma stores for future use.

## 4. How Feedback Becomes Evaluation Memory

When you submit feedback on an ambiguous case, Prisma creates a control scoped to the project and metric. A control contains the expert's classification and reasoning. When future validations encounter similar borderline cases, the re-evaluation prompt receives these controls via the `{memories}` placeholder.

Here's what `{memories}` looks like when it's populated with controls:

```
Prior expert rulings on similar cases:

1. [non_compliant] Address change request — "The agent used caller ID (phone
   number on file) as one verification factor and only requested the last four
   digits of the account number as the second. Caller ID does not count as a
   verification factor per Policy SEC-105. The agent needed to request a second
   proper factor (date of birth or mailing address) in addition to the account
   number. This is non-compliant despite the agent's good intent."

2. [non_compliant] Credit limit increase — "The agent requested only date of
   birth as a verification factor. Policy FIN-201 requires two distinct factors
   for credit limit changes. A single factor is insufficient regardless of the
   factor's strength."

3. [non_compliant] Dispute with weak factors — "The agent verified the
   customer's name and last four digits of the card number. Neither qualifies
   as a verification factor under Policy SEC-105 — name is not a secret, and
   the card number is printed on the physical card."
```

This is the key insight. The re-evaluation prompt from Section 1 includes:

```
Previous auditor notes and evaluation memory:\n{memories}
```

So when the metric encounters a new borderline case — say, an agent who verified identity using only a mother's maiden name — it now has three expert-reviewed precedents that establish a clear pattern: **informal or single-factor verification is always non-compliant**. The metric doesn't need to guess anymore. It has institutional knowledge.

## 5. Verify the Flywheel

Now log a new borderline interaction — similar to the ones the expert already reviewed — and see how the metric handles it with evaluation memory in place.

Add to `human_review_demo.py` or create a new file called `verify_flywheel.py`:

```python
import prisma_ai
from prisma_ai import Evaluators

# Same policy metric definition as before (from Section 1)
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],
    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires:\n\n"
        "1. GREETING: Professional greeting with agent name and company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action, the agent must "
        "verify the customer's identity by requesting at least two pieces of "
        "identifying information (account number + date of birth, or last four "
        "SSN digits + mailing address).\n"
        "3. ACCURATE INFORMATION: Response consistent with company policy.\n"
        "4. PROFESSIONAL CLOSE: Summary of actions, next steps, offer for "
        "further assistance.\n\n"
        "If ALL four criteria are met, classify as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify as 'non_compliant'."
    ),
    evaluation_human_prompt=(
        "Validate the following contact center interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain your reasoning."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the policy strictly. Pay special attention "
        "to identity verification — agents must follow the exact scripted process, "
        "not an informal or abbreviated version.\n\n"
        "Use the evaluation memory below to guide your decision. These are prior "
        "expert rulings on similar borderline cases."
    ),
    reeval_human_prompt=(
        "A previous auditor was uncertain about this interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes and evaluation memory:\n{memories}\n\n"
        "Provide your final classification as 'compliant' or 'non_compliant'. "
        "Explain your reasoning."
    ),
)

# Start a new run in the same project — controls carry over
session = prisma_ai.init(
    project_name="compliance-review-demo",
    run_name="post-feedback-verification",
    evaluators=[response_compliance],
    enable_logging=True,
)

logger = session.get_logger()

# A NEW borderline interaction — similar pattern to the ones reviewed
logger.log(
    query=(
        "I'd like to set up automatic bill pay from my checking account."
    ),
    response=(
        "Hi, this is Taylor with Meridian Financial. I can set that up for you. "
        "Let me just verify — can you confirm your mother's maiden name? Great, "
        "thank you. I've enabled automatic bill pay on your checking account "
        "ending in 3847. Your first automatic payment will process on the 1st "
        "of next month. You'll get an email confirmation shortly. Is there "
        "anything else I can help with today?"
    ),
    reference=(
        "Setting up automatic bill pay requires two-factor identity verification: "
        "account number plus date of birth, or last four SSN digits plus mailing "
        "address. Mother's maiden name is not an approved verification factor "
        "per Policy SEC-105. After verification, confirm the bill pay details "
        "and send written confirmation."
    ),
)
print("Logged new borderline interaction for verification.")

session.end()
```

```bash
python verify_flywheel.py
```

**Before evaluation memory:** This interaction would have been marked ambiguous — the agent *did* attempt verification (mother's maiden name), but it's unclear whether that meets policy.

**After evaluation memory:** The metric now has three expert-reviewed controls establishing that informal or non-standard verification factors are always non-compliant. The re-evaluation prompt receives these controls via `{memories}`, and the metric confidently classifies the interaction as `non_compliant` without needing another human review.

This is the flywheel in action. Each expert review teaches the metric how to handle similar edge cases.

### Before vs. After: Ambiguous Rates

After submitting a batch of expert reviews across a real project with hundreds of interactions, the impact is measurable:

| Metric | Before HITL Reviews | After HITL Reviews |
|---|---|---|
| Ambiguous rate | ~30% | ~5% |
| Cases needing human review per run | ~90 of 300 | ~15 of 300 |
| Average time to resolve per case | Manual review required | Resolved automatically |

The remaining ~5% ambiguous cases represent genuinely novel edge cases that the metric hasn't seen before — which is exactly when you want a human expert involved.

## 6. Managing Evaluation Memory

As your team submits more feedback, you'll accumulate controls that shape how your policy metrics behave. Prisma provides REST API endpoints for managing this evaluation memory at scale.

### List Controls

View all controls (evaluation memory) for a project:

```python
import httpx
import os

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]

headers = {
    "Authorization": f"Bearer {PRISMA_API_KEY}",
    "Content-Type": "application/json",
}

response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/controls",
    params={"project_id": "your-project-id"},
    headers=headers,
)

controls = response.json()
print(f"Total controls: {len(controls['items'])}")

for control in controls["items"]:
    print(f"\n--- Control {control['id']} ---")
    print(f"  Classification: {control['classification']}")
    print(f"  Reviewer: {control['reviewer']}")
    print(f"  Tags: {control['tags']}")
    print(f"  Created: {control['created_at']}")
```

### Bulk Feedback

When your QA team has a batch of reviews ready — say, from a weekly review session — submit them all at once:

```python
bulk_payload = {
    "feedback": [
        {
            "review_id": "rev_x1y2z3",
            "classification": "compliant",
            "reasoning": (
                "The agent verified identity with account number and date of birth "
                "before proceeding. Both factors meet Policy SEC-105 requirements."
            ),
            "reviewer": "jane.doe@company.com",
            "tags": ["identity-verification", "two-factor", "compliant-example"],
        },
        {
            "review_id": "rev_a4b5c6",
            "classification": "non_compliant",
            "reasoning": (
                "The agent asked for the customer's email address and zip code. "
                "Neither is an approved verification factor. Non-compliant."
            ),
            "reviewer": "jane.doe@company.com",
            "tags": ["identity-verification", "unapproved-factors"],
        },
        # ... more feedback items
    ]
}

response = httpx.post(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/bulk-feedback",
    json=bulk_payload,
    headers=headers,
)

result = response.json()
print(f"Submitted: {result['submitted']} | Errors: {result['errors']}")
```

### Export Pending Reviews

Export the current review queue for offline analysis or integration with external tools:

```python
response = httpx.get(
    f"{PRISMA_BASE_URL}/api/v1/hitl/reviews/export",
    params={
        "project_id": "your-project-id",
        "status": "pending",
        "format": "json",
    },
    headers=headers,
)

# Save to file for review in a spreadsheet or external tool
with open("pending_reviews.json", "w") as f:
    f.write(response.text)

print(f"Exported {len(response.json()['items'])} pending reviews")
```

## Production Considerations

### Review Cadence

Establish a regular rhythm for expert reviews. Weekly works well for most teams:

- **Monday**: Export the previous week's ambiguous cases
- **Tuesday-Thursday**: QA leads review and submit feedback
- **Friday**: Verify that ambiguous rates dropped for the current week's runs

The first few weeks will have the most reviews. As controls accumulate, the queue shrinks naturally.

### Control Lifecycle

Controls are scoped to a project and metric. As your policies change, old controls may become outdated. Review your controls periodically:

- After a policy update, audit existing controls to ensure they still reflect current standards
- Tag controls with the policy version they reference (e.g., `sec-105-v2`) so you can filter and retire them when the policy changes
- Prisma applies all active controls during re-evaluation — removing outdated ones keeps the `{memories}` context clean and relevant

### Reviewer Consistency

When multiple reviewers submit feedback on the same type of case, inconsistent reasoning creates conflicting evaluation memory. Mitigate this by:

- Establishing written guidelines for common edge cases before reviewers start
- Assigning borderline categories to specific domain experts (e.g., identity verification cases go to the security team lead)
- Reviewing controls periodically for conflicting reasoning and resolving disagreements

## Next Steps

Your policy metrics now learn from your team's expertise. Every human review makes future validations more confident and consistent. Next: run policy validation on live production traffic.

- [Real-Time Policy Monitoring](realtime-policy-monitoring.md) — validate interactions as they happen in production
- [Getting Started](getting-started.md) — set up Prisma AI from scratch
- [Building Policy-Based Metrics](policy-based-metrics.md) — encode your organization's rules as metrics
