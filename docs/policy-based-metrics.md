# Building Policy-Based Metrics

Encode your organization's specific rules as policy metrics that Prisma validates against. Policy metrics are the standard way to use Prisma -- the 4-prompt pattern is the core abstraction that turns any compliance rule, quality standard, or process guideline into an automated validator.

This cookbook walks through the 4-prompt pattern in depth, then builds complete metrics for three regulated industries: financial services, healthcare, and insurance.

## What You'll Learn

- How the 4-prompt pattern works -- what each prompt does, when it fires, and what placeholders are available
- Build a `disclosure-compliance` metric step by step for financial services
- Build a `clinical-communication` metric for healthcare
- Build a `claims-handling` metric for insurance
- Combine multiple policy metrics into a single validation configuration
- Write effective options and prompts that produce consistent results

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **Access to a running Prisma instance**
- Familiarity with [Getting Started](getting-started.md) concepts (projects, runs, policy metrics)

## 1. The 4-Prompt Pattern

Every policy metric in Prisma is defined by four prompts. Together, they form a two-pass validation pipeline: an initial pass that classifies the interaction, and a re-evaluation pass that fires only when the initial result is uncertain.

### How the prompts work together

```
Interaction arrives
        │
        ▼
┌──────────────────────────┐
│  evaluation_system_prompt │  ← Sets the validator's role and criteria
│  evaluation_human_prompt  │  ← Presents the interaction to classify
└────────────┬─────────────┘
             │
             ▼
      ┌─────────────┐
      │   Result?    │
      └──┬──────┬────┘
         │      │
    definitive  ambiguous
         │      │
         ▼      ▼
      Done   ┌──────────────────────────┐
             │  reeval_system_prompt     │  ← Stricter guidance for second pass
             │  reeval_human_prompt      │  ← Includes memory from past reviews
             └────────────┬─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │   Result?    │
                   └──┬──────┬───┘
                      │      │
                 definitive  still ambiguous
                      │      │
                      ▼      ▼
                   Done    Routed to
                           human review
```

### Prompt reference

| Prompt | When it fires | What it does | Available placeholders |
|---|---|---|---|
| `evaluation_system_prompt` | Every interaction | Sets the validator's role, domain expertise, and classification criteria. Think of it as the "job description" for the validator. | `{evaluation_options}` |
| `evaluation_human_prompt` | Every interaction | Presents the specific interaction to classify. This is where you structure what the validator sees. | `{query}`, `{response}`, `{evaluation_options}`, `{reference}` (optional) |
| `reeval_system_prompt` | Only when initial result is ambiguous | Provides stricter, more specific guidance for the second pass. Often emphasizes the most critical criteria. | `{evaluation_options}` |
| `reeval_human_prompt` | Only when initial result is ambiguous | Re-presents the interaction along with memory from past human reviews. This is how the system learns from your team's decisions. | `{query}`, `{response}`, `{memories}` |

### Placeholder details

- **`{query}`** -- The input side of the interaction (customer message, patient question, user request).
- **`{response}`** -- The output side (agent reply, advisor response, system answer).
- **`{reference}`** -- Optional ground truth or policy-correct response. Include it in `evaluation_human_prompt` when you have a known-good answer to compare against.
- **`{evaluation_options}`** -- The list of classification options you defined, formatted for the validator. Prisma populates this automatically from your `options` list.
- **`{memories}`** -- Context from past human reviews of similar interactions. Only available in `reeval_human_prompt`. This is the feedback loop -- expert decisions on past ambiguous cases become guidance for future validations.

### The "ambiguous" option

You do not need to add `"ambiguous"` to your options list. Prisma adds it automatically. When the initial validation returns `ambiguous`, re-evaluation fires with `{memories}`. If the result is still ambiguous after re-evaluation, the interaction is routed to a human reviewer.

## 2. Build a Metric Step by Step

Let's build a `disclosure-compliance` metric for financial services. Financial advisors must include specific regulatory disclosures when discussing investments, performance data, or recommendations. This metric validates whether those disclosures are present.

### Step 1: Choose your options

```python
options=["compliant", "missing_disclosure", "partial"]
```

Three options is a good starting point for compliance metrics:

- **`compliant`** -- All required disclosures are present and clearly stated.
- **`missing_disclosure`** -- The response discusses investments or performance but omits one or more required disclosures entirely.
- **`partial`** -- Some disclosures are present but others are missing, or disclosures are buried in a way that reduces their visibility.

Why three instead of two? A binary `compliant` / `non_compliant` split forces the validator to make a hard call on borderline cases. The `partial` option gives it a place to put responses that are technically incomplete but not egregiously wrong -- and those cases often end up in re-evaluation where `{memories}` from past human reviews provide the nuance to make a final call.

### Step 2: Write the evaluation system prompt

This is the validator's "job description." Be specific about the domain, the rules, and what each option means.

```python
evaluation_system_prompt=(
    "You are a financial regulatory compliance reviewer. Your task is to "
    "determine whether advisor responses include all required regulatory "
    "disclosures when discussing investments, performance data, or financial "
    "recommendations.\n\n"
    "Required disclosures include (when applicable):\n"
    "- Past performance is not indicative of future results\n"
    "- Investment involves risk, including possible loss of principal\n"
    "- Consult a qualified financial advisor before making decisions\n"
    "- Securities are not FDIC insured\n\n"
    "Classify the response into one of these categories: {evaluation_options}\n\n"
    "- compliant: All applicable disclosures are present and clearly stated.\n"
    "- missing_disclosure: The response discusses investments or performance "
    "but omits one or more required disclosures entirely.\n"
    "- partial: Some disclosures are present but others are missing, or "
    "disclosures are buried in a way that reduces their visibility."
),
```

Design decisions:

- **List the specific rules.** Don't say "include required disclosures." Say exactly which disclosures are required. The validator needs the same rulebook your compliance team uses.
- **Define every option.** The validator needs to know what each classification means in this domain. Vague options produce inconsistent results.
- **Use `{evaluation_options}` in context.** Embedding it in the criteria section reinforces the connection between rules and classifications.

### Step 3: Write the evaluation human prompt

This structures the interaction for the validator. Keep it clean and consistent.

```python
evaluation_human_prompt=(
    "Review the following financial services interaction for disclosure "
    "compliance.\n\n"
    "Customer inquiry:\n{query}\n\n"
    "Advisor response:\n{response}\n\n"
    "Classify the response as one of: {evaluation_options}\n\n"
    "List which disclosures are present, which are missing, and provide "
    "your classification."
),
```

Design decisions:

- **Label sections clearly.** "Customer inquiry:" and "Advisor response:" make it obvious which side is which.
- **Ask for reasoning before the classification.** "List which disclosures are present, which are missing" forces the validator to do the analytical work before committing to a label.
- **Keep the template consistent.** Every interaction gets the same structure, so the validator builds reliable patterns.

### Step 4: Write the re-evaluation system prompt

This fires only when the first pass returned `ambiguous`. Make it stricter and more specific.

```python
reeval_system_prompt=(
    "You are a senior financial regulatory compliance reviewer performing "
    "a second-pass review. Apply disclosure requirements strictly. If a "
    "response discusses any aspect of investment performance, risk, or "
    "recommendations without ALL applicable disclosures clearly stated, "
    "classify it as missing_disclosure or partial -- not compliant.\n\n"
    "You have access to decisions from previous human compliance reviews "
    "to calibrate your judgment on borderline cases.\n\n"
    "Classify the response into one of these categories: {evaluation_options}"
),
```

Design decisions:

- **Escalate the seniority.** "Senior reviewer" signals this is a higher-stakes pass.
- **Tighten the criteria.** The initial prompt says "when applicable." The re-evaluation prompt says "if a response discusses ANY aspect." This narrows the gray area.
- **Reference the human review context.** Mentioning that past decisions are available primes the validator to use `{memories}` effectively.

### Step 5: Write the re-evaluation human prompt

This includes `{memories}` -- context from past human reviews.

```python
reeval_human_prompt=(
    "Re-evaluate this financial services interaction for disclosure compliance "
    "using context from previous human reviews.\n\n"
    "Customer inquiry:\n{query}\n\n"
    "Advisor response:\n{response}\n\n"
    "Relevant past compliance review decisions:\n{memories}\n\n"
    "Based on the interaction and past review history, provide your final "
    "classification with specific disclosure findings."
),
```

### The complete metric

```python
from prisma_ai import Evaluators

disclosure_compliance = Evaluators.custom(
    name="disclosure-compliance",
    options=["compliant", "missing_disclosure", "partial"],
    evaluation_system_prompt=(
        "You are a financial regulatory compliance reviewer. Your task is to "
        "determine whether advisor responses include all required regulatory "
        "disclosures when discussing investments, performance data, or financial "
        "recommendations.\n\n"
        "Required disclosures include (when applicable):\n"
        "- Past performance is not indicative of future results\n"
        "- Investment involves risk, including possible loss of principal\n"
        "- Consult a qualified financial advisor before making decisions\n"
        "- Securities are not FDIC insured\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- compliant: All applicable disclosures are present and clearly stated.\n"
        "- missing_disclosure: The response discusses investments or performance "
        "but omits one or more required disclosures entirely.\n"
        "- partial: Some disclosures are present but others are missing, or "
        "disclosures are buried in a way that reduces their visibility."
    ),
    evaluation_human_prompt=(
        "Review the following financial services interaction for disclosure "
        "compliance.\n\n"
        "Customer inquiry:\n{query}\n\n"
        "Advisor response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "List which disclosures are present, which are missing, and provide "
        "your classification."
    ),
    reeval_system_prompt=(
        "You are a senior financial regulatory compliance reviewer performing "
        "a second-pass review. Apply disclosure requirements strictly. If a "
        "response discusses any aspect of investment performance, risk, or "
        "recommendations without ALL applicable disclosures clearly stated, "
        "classify it as missing_disclosure or partial -- not compliant.\n\n"
        "You have access to decisions from previous human compliance reviews "
        "to calibrate your judgment on borderline cases.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this financial services interaction for disclosure compliance "
        "using context from previous human reviews.\n\n"
        "Customer inquiry:\n{query}\n\n"
        "Advisor response:\n{response}\n\n"
        "Relevant past compliance review decisions:\n{memories}\n\n"
        "Based on the interaction and past review history, provide your final "
        "classification with specific disclosure findings."
    ),
)
```

## 3. Healthcare: Clinical Communication Guidelines

Healthcare advisors and support agents must follow strict communication guidelines: use appropriate qualifiers ("may," "could," "consult your doctor"), refer to specialists when the question exceeds their scope, and never provide a diagnosis via chat. This metric validates adherence to those guidelines.

```python
from prisma_ai import Evaluators

clinical_communication = Evaluators.custom(
    name="clinical-communication",
    options=["compliant", "guideline_violation", "needs_specialist_review"],
    evaluation_system_prompt=(
        "You are a healthcare communication compliance reviewer. Your task is "
        "to determine whether a health advisor's response follows clinical "
        "communication guidelines.\n\n"
        "The guidelines require:\n"
        "1. APPROPRIATE QUALIFIERS: The advisor uses hedging language such as "
        "'may,' 'could,' 'it is possible,' or 'consult your doctor' when "
        "discussing symptoms, conditions, or treatments. Absolute statements "
        "about medical outcomes are prohibited.\n"
        "2. SCOPE BOUNDARIES: When a question involves diagnosis, prescription "
        "recommendations, or specialist-level clinical judgment, the advisor "
        "must refer the patient to a qualified healthcare professional rather "
        "than attempting to answer directly.\n"
        "3. NO DIAGNOSIS VIA CHAT: The advisor must never state or imply a "
        "diagnosis. Phrases like 'you have,' 'this is likely,' or 'it sounds "
        "like you are experiencing [condition]' are violations.\n"
        "4. EMPATHY AND CLARITY: The advisor responds with empathy, avoids "
        "medical jargon without explanation, and provides clear next steps.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- compliant: All four guidelines are followed. Qualifiers are present, "
        "scope boundaries are respected, no diagnosis is offered, and the tone "
        "is empathetic and clear.\n"
        "- guideline_violation: One or more guidelines are clearly violated -- "
        "the advisor made absolute medical claims, offered a diagnosis, or "
        "failed to refer when appropriate.\n"
        "- needs_specialist_review: The response is in a gray area where "
        "compliance cannot be confidently determined without clinical expertise "
        "(e.g., nuanced qualifier usage, evolving treatment protocols)."
    ),
    evaluation_human_prompt=(
        "Review the following healthcare interaction for compliance with "
        "clinical communication guidelines.\n\n"
        "Patient message:\n{query}\n\n"
        "Advisor response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "For each guideline (qualifiers, scope boundaries, no diagnosis, "
        "empathy/clarity), state whether it was followed or violated, then "
        "provide your classification."
    ),
    reeval_system_prompt=(
        "You are a senior clinical communication compliance reviewer performing "
        "a second-pass review. Apply the guidelines strictly.\n\n"
        "Pay particular attention to implicit diagnoses -- statements that stop "
        "short of saying 'you have X' but still lead the patient to a specific "
        "conclusion. These are the most common borderline violations.\n\n"
        "You have access to decisions from previous clinical reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this healthcare interaction for compliance with clinical "
        "communication guidelines, using context from previous clinical reviews.\n\n"
        "Patient message:\n{query}\n\n"
        "Advisor response:\n{response}\n\n"
        "Relevant past clinical review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification. Cite specific phrases that comply with or violate "
        "each guideline."
    ),
)
```

### What this metric catches

| Interaction | Expected result | Why |
|---|---|---|
| Patient asks about headache causes; advisor says "there are several possible causes, and I'd recommend discussing this with your doctor" | `compliant` | Appropriate qualifier, refers to doctor, no diagnosis |
| Patient describes chest pain; advisor says "this sounds like it could be angina -- you should take aspirin" | `guideline_violation` | Implies diagnosis ("sounds like angina"), provides treatment recommendation without referral |
| Patient asks about a new medication interaction; advisor says "interactions between these medications may vary -- I'd suggest confirming with your pharmacist" | `compliant` | Uses qualifier ("may vary"), refers to specialist (pharmacist), no diagnosis |
| Patient describes symptoms; advisor says "these symptoms are sometimes associated with several conditions -- a specialist would be best positioned to help" | `compliant` | Hedged language, scope boundary respected, no diagnosis |

## 4. Insurance: Claims Handling Process

Insurance claims agents must follow a structured triage process: assess damage, verify coverage, and communicate timelines. Skipping steps or providing timeline commitments without verifying coverage creates regulatory and legal exposure. This metric validates whether the agent followed the correct sequence.

```python
from prisma_ai import Evaluators

claims_handling = Evaluators.custom(
    name="claims-handling",
    options=["process_followed", "step_skipped", "incomplete"],
    evaluation_system_prompt=(
        "You are an insurance claims process compliance reviewer. Your task is "
        "to determine whether a claims agent followed the correct triage process "
        "during a customer interaction.\n\n"
        "The required claims triage process has three steps, in order:\n"
        "1. DAMAGE ASSESSMENT: The agent gathers details about the damage or "
        "loss -- what happened, when, extent of damage, and any documentation "
        "the customer can provide (photos, police reports, receipts).\n"
        "2. COVERAGE VERIFICATION: Before making any commitments, the agent "
        "verifies the customer's policy coverage for the type of claim. The "
        "agent must NOT discuss payout amounts, approval likelihood, or "
        "timelines until coverage is confirmed.\n"
        "3. TIMELINE COMMUNICATION: After verifying coverage, the agent "
        "communicates the expected timeline for claim processing, next steps, "
        "and any additional documentation required. Timelines must include a "
        "range (e.g., '5-10 business days'), not a specific date.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- process_followed: All three steps are present in the correct order. "
        "The agent gathered damage details, verified coverage before making "
        "commitments, and communicated an appropriate timeline range.\n"
        "- step_skipped: One or more required steps are entirely missing. The "
        "most critical violation is discussing payouts or timelines before "
        "verifying coverage.\n"
        "- incomplete: The agent started the process correctly but the "
        "interaction ended before all steps were completed, or a step was "
        "partially addressed (e.g., damage details gathered but no follow-up "
        "on documentation)."
    ),
    evaluation_human_prompt=(
        "Review the following insurance claims interaction for compliance with "
        "the claims triage process.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "For each step (damage assessment, coverage verification, timeline "
        "communication), state whether it was completed, skipped, or partially "
        "addressed. Then provide your classification."
    ),
    reeval_system_prompt=(
        "You are a senior insurance claims process reviewer performing a "
        "second-pass review. Apply the triage process requirements strictly.\n\n"
        "The most critical compliance point is the ordering constraint: coverage "
        "must be verified BEFORE any timeline or payout discussion. An agent who "
        "says 'you should have your check within two weeks' before confirming "
        "the policy covers the claim type has committed a process violation, "
        "even if they eventually verify coverage later in the conversation.\n\n"
        "You have access to decisions from previous claims process reviews to "
        "calibrate your judgment.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate this insurance claims interaction for compliance with the "
        "claims triage process, using context from previous process reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past claims process review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification. Pay specific attention to the ordering of steps."
    ),
)
```

### What this metric catches

| Interaction | Expected result | Why |
|---|---|---|
| Customer reports water damage; agent asks for photos, checks policy, then says "processing takes 5-10 business days" | `process_followed` | All three steps in order, timeline is a range |
| Customer reports a fender bender; agent immediately says "we'll have a check to you within a week" | `step_skipped` | Jumped to timeline without damage assessment or coverage verification |
| Customer reports theft; agent gathers details and asks for a police report but does not verify coverage or communicate a timeline | `incomplete` | Damage assessment done, but interaction ended before coverage check and timeline |
| Customer reports storm damage; agent gathers details, verifies coverage, says "your claim will be processed by March 15th" | `step_skipped` | Specific date instead of a range violates the timeline communication requirement |

## 5. Combine Multiple Metrics

Run all three metrics together on every interaction. Each metric validates independently -- an interaction can be `compliant` on disclosures but `step_skipped` on claims handling.

```python
import prisma_ai
from prisma_ai import Evaluators

# Define all three metrics (using the full definitions above)
disclosure_compliance = Evaluators.custom(
    name="disclosure-compliance",
    options=["compliant", "missing_disclosure", "partial"],
    evaluation_system_prompt="...",   # Full prompts as defined in Section 2
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

clinical_communication = Evaluators.custom(
    name="clinical-communication",
    options=["compliant", "guideline_violation", "needs_specialist_review"],
    evaluation_system_prompt="...",   # Full prompts as defined in Section 3
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

claims_handling = Evaluators.custom(
    name="claims-handling",
    options=["process_followed", "step_skipped", "incomplete"],
    evaluation_system_prompt="...",   # Full prompts as defined in Section 4
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

# Initialize with all metrics
session = prisma_ai.init(
    project_name="multi-industry-compliance",
    evaluators=[
        disclosure_compliance,
        clinical_communication,
        claims_handling,
    ],
    enable_logging=True,
)

logger = session.get_logger()
```

In practice, you would typically apply industry-specific metrics to the relevant project. A financial services project uses `disclosure-compliance`; a healthcare project uses `clinical-communication`. Combining them in a single project is useful for shared services teams that handle interactions across multiple regulated domains.

## 6. Tips for Writing Effective Metrics

### Choosing options

**Use 3-5 options.** Two options (pass/fail) force the validator into hard calls on borderline cases. More than five options create overlapping categories that produce inconsistent results.

**Name options clearly.** Use lowercase with underscores. Each option name should be self-explanatory:

| Good | Bad |
|---|---|
| `compliant` | `good` |
| `missing_disclosure` | `fail_type_1` |
| `process_followed` | `yes` |
| `needs_specialist_review` | `maybe` |
| `step_skipped` | `error` |

**Make options mutually exclusive.** If a validator could reasonably assign two different options to the same interaction, your options overlap. Tighten the definitions in the system prompt.

**Include a "gray area" option.** Options like `partial`, `incomplete`, or `needs_specialist_review` give the validator a place to put borderline cases instead of forcing them into a definitive bucket. These cases often end up in re-evaluation, where `{memories}` provides the context to resolve them.

### Writing prompts

**Be specific about the rules.** Don't say "follow company policy." List the exact criteria, numbered and named. The validator needs the same rulebook your compliance team uses.

**Define every option in the system prompt.** The validator needs to know what each classification means. A one-line definition per option prevents drift.

**Ask for reasoning before the classification.** In the human prompt, ask the validator to explain its analysis before stating its label. This produces more consistent results because the reasoning constrains the final answer.

**Make re-evaluation prompts stricter.** The initial pass gets reasonable latitude. The re-evaluation pass should narrow the gray area -- tighten criteria, emphasize the most critical rules, and resolve common ambiguities.

### Testing your metrics

Start with 10-20 interactions where you already know the expected result. Log them through your metric and compare. If the metric disagrees with your compliance team on more than a few, the prompts need refinement -- usually the option definitions are too vague or the rules aren't specific enough.

## Next Steps

Your policy metrics validate interactions against your organization's rules. But what happens when the validator isn't sure? The re-evaluation prompts receive `{memories}` -- context from past human reviews that calibrates future decisions. This feedback loop is how your metrics get smarter over time.

Next: make your metrics smarter over time with human review. See [Human Review & Evaluation Memory](human-review.md) for how this feedback loop works.
