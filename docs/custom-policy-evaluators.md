# Build Custom Policy Compliance Evaluators

Create domain-specific evaluators that enforce regulatory and quality standards across insurance, financial services, and healthcare LLM applications.

## What You'll Learn

- Define custom evaluators using the 4-prompt pattern (evaluation + re-evaluation)
- Write effective evaluation prompts with proper placeholders for regulated industries
- Combine multiple custom evaluators into a single evaluation configuration
- Use both real-time tracing and batch evaluation with custom evaluators

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- Access to a running Prisma instance
- Familiarity with [Getting Started](getting-started.md) concepts (projects, runs, evaluators)

## 1. Install Dependencies

```bash
pip install prisma-ai pq-prisma-client
```

The `prisma-ai` package provides the unified SDK for tracing and evaluation configuration. The `pq-prisma-client` package is the async HTTP client used for batch evaluation and direct API access.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Understand the 4-Prompt Pattern

Every custom evaluator in Prisma uses four prompts that work together:

| Prompt | Purpose | Required Placeholders |
|---|---|---|
| `evaluation_system_prompt` | Sets the evaluator's role and criteria | `{evaluation_options}` |
| `evaluation_human_prompt` | Presents the interaction to evaluate | `{query}`, `{response}`, `{evaluation_options}` |
| `reeval_system_prompt` | Sets context for re-evaluation with memory | `{evaluation_options}` |
| `reeval_human_prompt` | Re-presents the interaction with past review context | `{query}`, `{response}`, `{memories}` |

When Prisma encounters an ambiguous result during initial evaluation, it triggers re-evaluation. The re-evaluation prompts receive `{memories}` -- context drawn from past human reviews of similar cases. This is how the system learns from your compliance team's decisions over time.

## 4. Build a Tone and Professionalism Evaluator (Insurance)

Insurance claims conversations require consistent professionalism, especially when discussing sensitive topics like claim denials or coverage disputes. This evaluator classifies agent tone on every interaction.

```python
from prisma_ai import Evaluators, EvaluationConfig

tone_evaluator = Evaluators.custom(
    name="tone-professionalism",
    options=["professional", "casual", "inappropriate"],
    evaluation_system_prompt=(
        "You are an insurance industry quality assurance specialist. "
        "Your task is to evaluate the tone and professionalism of agent responses "
        "during claims-related conversations. Agents must maintain empathy and "
        "formality, especially when delivering unfavorable outcomes.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- professional: Response uses appropriate formal language, shows empathy, "
        "avoids jargon overload, and maintains a respectful tone throughout.\n"
        "- casual: Response is understandable but uses informal language, slang, "
        "or a tone that would be inappropriate in a regulated claims context.\n"
        "- inappropriate: Response contains dismissive language, sarcasm, blame, "
        "or any phrasing that could escalate a sensitive situation."
    ),
    evaluation_human_prompt=(
        "Evaluate the tone of the following insurance claims interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the agent's tone as one of: {evaluation_options}\n\n"
        "Provide your classification and a brief justification."
    ),
    reeval_system_prompt=(
        "You are an insurance industry quality assurance specialist performing "
        "a second-pass review of an agent's tone. You have access to decisions "
        "from previous human reviews of similar interactions to guide your judgment.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate the tone of this insurance claims interaction using the "
        "context from previous human reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past review decisions:\n{memories}\n\n"
        "Based on the interaction and the review history, provide your final "
        "classification and justification."
    ),
)
```

Note that `"ambiguous"` is automatically added to the options list. When the initial evaluation returns `"ambiguous"`, Prisma triggers re-evaluation with memories from past human reviews. If the result is still ambiguous after re-evaluation, the interaction is routed to a human reviewer.

## 5. Build a Disclosure Compliance Evaluator (Financial Services)

Financial advisors and robo-advisors must include specific regulatory disclosures when discussing investment performance, risk, or recommendations. This evaluator checks whether required disclosures are present.

```python
disclosure_evaluator = Evaluators.custom(
    name="disclosure-compliance",
    options=["compliant", "missing_disclosure", "partial"],
    evaluation_system_prompt=(
        "You are a financial regulatory compliance reviewer. Your task is to "
        "determine whether agent responses include all required regulatory "
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
        "You are a financial regulatory compliance reviewer performing a "
        "second-pass review. You have access to decisions from previous human "
        "compliance reviews to calibrate your judgment on borderline cases.\n\n"
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

## 6. Build a Medical Accuracy Evaluator (Healthcare)

Healthcare chatbots and clinical decision support tools must provide information that aligns with established clinical guidelines. This evaluator flags responses that may contain inaccurate medical information.

```python
medical_evaluator = Evaluators.custom(
    name="medical-accuracy",
    options=["accurate", "inaccurate", "needs_review"],
    evaluation_system_prompt=(
        "You are a clinical information accuracy reviewer. Your task is to "
        "evaluate whether health-related responses align with established "
        "clinical guidelines and medical consensus. You are not diagnosing "
        "patients -- you are reviewing whether the information provided by "
        "an AI assistant is medically sound.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- accurate: The medical information is consistent with current clinical "
        "guidelines, uses appropriate qualifiers, and does not overstate certainty.\n"
        "- inaccurate: The response contains factual medical errors, contradicts "
        "established guidelines, or provides dangerous recommendations.\n"
        "- needs_review: The response touches on complex or evolving medical "
        "topics where accuracy cannot be confidently determined without "
        "specialist input."
    ),
    evaluation_human_prompt=(
        "Evaluate the medical accuracy of the following healthcare interaction.\n\n"
        "Patient question:\n{query}\n\n"
        "Assistant response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "Identify any specific claims that are accurate, inaccurate, or require "
        "specialist review. Reference clinical guidelines where possible."
    ),
    reeval_system_prompt=(
        "You are a clinical information accuracy reviewer performing a "
        "second-pass review. You have access to decisions from previous "
        "medical professional reviews to help calibrate your assessment on "
        "complex or borderline cases.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate the medical accuracy of this healthcare interaction using "
        "context from previous medical professional reviews.\n\n"
        "Patient question:\n{query}\n\n"
        "Assistant response:\n{response}\n\n"
        "Relevant past medical review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification with specific clinical references where applicable."
    ),
)
```

## 7. Combine Custom Evaluators with Real-Time Tracing

Use `prisma_ai.init()` to attach all three custom evaluators (plus a built-in evaluator) to a live tracing session. Every traced interaction is automatically evaluated against all configured evaluators.

```python
import prisma_ai
from prisma_ai import Evaluators, EvaluationConfig

# Define all custom evaluators
tone_evaluator = Evaluators.custom(
    name="tone-professionalism",
    options=["professional", "casual", "inappropriate"],
    evaluation_system_prompt="...",   # Full prompts as defined above
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

disclosure_evaluator = Evaluators.custom(
    name="disclosure-compliance",
    options=["compliant", "missing_disclosure", "partial"],
    evaluation_system_prompt="...",
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

medical_evaluator = Evaluators.custom(
    name="medical-accuracy",
    options=["accurate", "inaccurate", "needs_review"],
    evaluation_system_prompt="...",
    evaluation_human_prompt="...",
    reeval_system_prompt="...",
    reeval_human_prompt="...",
)

# Initialize with all evaluators — built-in and custom
prisma_ai.init(
    project_name="regulated-compliance-suite",
    run_name="prod-monitoring-v1",
    evaluators=[
        "correctness",          # Built-in evaluator
        tone_evaluator,         # Custom: insurance tone
        disclosure_evaluator,   # Custom: financial disclosures
        medical_evaluator,      # Custom: medical accuracy
    ],
)
```

Every LLM interaction traced through this session is now evaluated against all four evaluators in parallel. The Prisma project overview shows aggregate pass rates and ambiguous counts for each metric. For per-interaction breakdowns, query the Prisma REST API or connect your analytics tool via the analytics integration layer.

## 8. Use Batch Evaluation with PrismaClient

For evaluating existing datasets (such as historical transcripts or test suites), use `PrismaClient` for batch evaluation.

```python
import asyncio
from pq_prisma_client import PrismaClient
from prisma_ai import Evaluators, EvaluationConfig, InputMapping

async def run_batch_evaluation():
    async with PrismaClient() as client:

        # Define evaluators using the Evaluators factory
        tone_eval = Evaluators.custom(
        name="tone-professionalism",
        options=["professional", "casual", "inappropriate"],
        evaluation_system_prompt=(
            "You are an insurance industry quality assurance specialist. "
            "Your task is to evaluate the tone and professionalism of agent responses "
            "during claims-related conversations. Agents must maintain empathy and "
            "formality, especially when delivering unfavorable outcomes.\n\n"
            "Classify the response into one of these categories: {evaluation_options}\n\n"
            "- professional: Response uses appropriate formal language, shows empathy, "
            "avoids jargon overload, and maintains a respectful tone throughout.\n"
            "- casual: Response is understandable but uses informal language, slang, "
            "or a tone that would be inappropriate in a regulated claims context.\n"
            "- inappropriate: Response contains dismissive language, sarcasm, blame, "
            "or any phrasing that could escalate a sensitive situation."
        ),
        evaluation_human_prompt=(
            "Evaluate the tone of the following insurance claims interaction.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Classify the agent's tone as one of: {evaluation_options}\n\n"
            "Provide your classification and a brief justification."
        ),
        reeval_system_prompt=(
            "You are an insurance industry quality assurance specialist performing "
            "a second-pass review of an agent's tone. You have access to decisions "
            "from previous human reviews of similar interactions to guide your judgment.\n\n"
            "Classify the response into one of these categories: {evaluation_options}"
        ),
        reeval_human_prompt=(
            "Re-evaluate the tone of this insurance claims interaction using the "
            "context from previous human reviews.\n\n"
            "Customer message:\n{query}\n\n"
            "Agent response:\n{response}\n\n"
            "Relevant past review decisions:\n{memories}\n\n"
            "Based on the interaction and the review history, provide your final "
            "classification and justification."
        ),
    )

    # Sample dataset — historical claims interactions
    interactions = [
        {
            "query": "My car was totaled last week and I still haven't heard back about my claim. This is unacceptable.",
            "response": "I completely understand your frustration, and I sincerely apologize for the delay. Let me pull up your claim right now and get you a status update. Your claim is important to us, and I want to make sure we resolve this as quickly as possible.",
        },
        {
            "query": "Why was my claim denied? I've been paying premiums for 10 years!",
            "response": "Yeah so basically your policy doesn't cover that type of damage. You should've read the fine print before signing up lol. Anyway, you can appeal if you want.",
        },
    ]

        # Create a project and run for batch evaluation
        project = await client.projects.get_or_create(name="insurance-qa-audit")
        run = await client.runs.create(
            project_id=project.id,
            name="batch-claims-review",
            run_type="dataset",
            evaluation_config=EvaluationConfig(evaluators=[tone_eval]),
        )

        # Upload interactions as a dataset and submit for evaluation
        import json, tempfile
        with tempfile.NamedTemporaryFile(mode="w", suffix=".json", delete=False) as f:
            json.dump(interactions, f)
            temp_path = f.name

        dataset = await client.datasets.upload(temp_path)
        job = await client.jobs.create(run_id=run.id, file_id=dataset.id)

        print(f"Submitted {len(interactions)} interactions for evaluation.")
        print(f"Project: {project.id}, Run: {run.id}, Job: {job.id}")

asyncio.run(run_batch_evaluation())
```

## Expected Output

After running either the real-time or batch evaluation, the Prisma project overview shows aggregate metrics for your run. Click on the run to see per-metric summaries:

- **Correctness**: Pass rate across all evaluated interactions
- **Tone-professionalism**: Distribution of professional / casual / inappropriate classifications
- **Disclosure-compliance**: Compliant vs missing_disclosure vs partial breakdown
- **Medical-accuracy**: Accurate vs inaccurate vs needs_review counts
- **Ambiguous**: Total cases queued for human expert review

For per-interaction detail (e.g., which specific transcripts failed which evaluators), query the Prisma REST API or connect a visualization tool like Power BI via the analytics integration layer.

Interactions classified as `"ambiguous"` during initial evaluation are automatically re-evaluated with memories from past human reviews. If still ambiguous, they appear in the **HITL Requests** count in the project overview, queued for expert review.

## Complete Example

```python
"""Custom policy compliance evaluators for regulated industries.

Demonstrates building and combining three domain-specific evaluators:
tone & professionalism (insurance), disclosure compliance (financial),
and medical accuracy (healthcare).
"""

import prisma_ai
from prisma_ai import Evaluators, EvaluationConfig

# -- Evaluator 1: Tone & Professionalism (Insurance) --

tone_evaluator = Evaluators.custom(
    name="tone-professionalism",
    options=["professional", "casual", "inappropriate"],
    evaluation_system_prompt=(
        "You are an insurance industry quality assurance specialist. "
        "Your task is to evaluate the tone and professionalism of agent responses "
        "during claims-related conversations. Agents must maintain empathy and "
        "formality, especially when delivering unfavorable outcomes.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- professional: Response uses appropriate formal language, shows empathy, "
        "avoids jargon overload, and maintains a respectful tone throughout.\n"
        "- casual: Response is understandable but uses informal language, slang, "
        "or a tone that would be inappropriate in a regulated claims context.\n"
        "- inappropriate: Response contains dismissive language, sarcasm, blame, "
        "or any phrasing that could escalate a sensitive situation."
    ),
    evaluation_human_prompt=(
        "Evaluate the tone of the following insurance claims interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Classify the agent's tone as one of: {evaluation_options}\n\n"
        "Provide your classification and a brief justification."
    ),
    reeval_system_prompt=(
        "You are an insurance industry quality assurance specialist performing "
        "a second-pass review of an agent's tone. You have access to decisions "
        "from previous human reviews of similar interactions to guide your judgment.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate the tone of this insurance claims interaction using the "
        "context from previous human reviews.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Relevant past review decisions:\n{memories}\n\n"
        "Based on the interaction and the review history, provide your final "
        "classification and justification."
    ),
)

# -- Evaluator 2: Disclosure Compliance (Financial Services) --

disclosure_evaluator = Evaluators.custom(
    name="disclosure-compliance",
    options=["compliant", "missing_disclosure", "partial"],
    evaluation_system_prompt=(
        "You are a financial regulatory compliance reviewer. Your task is to "
        "determine whether agent responses include all required regulatory "
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
        "You are a financial regulatory compliance reviewer performing a "
        "second-pass review. You have access to decisions from previous human "
        "compliance reviews to calibrate your judgment on borderline cases.\n\n"
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

# -- Evaluator 3: Medical Accuracy (Healthcare) --

medical_evaluator = Evaluators.custom(
    name="medical-accuracy",
    options=["accurate", "inaccurate", "needs_review"],
    evaluation_system_prompt=(
        "You are a clinical information accuracy reviewer. Your task is to "
        "evaluate whether health-related responses align with established "
        "clinical guidelines and medical consensus. You are not diagnosing "
        "patients -- you are reviewing whether the information provided by "
        "an AI assistant is medically sound.\n\n"
        "Classify the response into one of these categories: {evaluation_options}\n\n"
        "- accurate: The medical information is consistent with current clinical "
        "guidelines, uses appropriate qualifiers, and does not overstate certainty.\n"
        "- inaccurate: The response contains factual medical errors, contradicts "
        "established guidelines, or provides dangerous recommendations.\n"
        "- needs_review: The response touches on complex or evolving medical "
        "topics where accuracy cannot be confidently determined without "
        "specialist input."
    ),
    evaluation_human_prompt=(
        "Evaluate the medical accuracy of the following healthcare interaction.\n\n"
        "Patient question:\n{query}\n\n"
        "Assistant response:\n{response}\n\n"
        "Classify the response as one of: {evaluation_options}\n\n"
        "Identify any specific claims that are accurate, inaccurate, or require "
        "specialist review. Reference clinical guidelines where possible."
    ),
    reeval_system_prompt=(
        "You are a clinical information accuracy reviewer performing a "
        "second-pass review. You have access to decisions from previous "
        "medical professional reviews to help calibrate your assessment on "
        "complex or borderline cases.\n\n"
        "Classify the response into one of these categories: {evaluation_options}"
    ),
    reeval_human_prompt=(
        "Re-evaluate the medical accuracy of this healthcare interaction using "
        "context from previous medical professional reviews.\n\n"
        "Patient question:\n{query}\n\n"
        "Assistant response:\n{response}\n\n"
        "Relevant past medical review decisions:\n{memories}\n\n"
        "Based on the interaction and review history, provide your final "
        "classification with specific clinical references where applicable."
    ),
)

# -- Combine all evaluators in a single init() call --

prisma_ai.init(
    project_name="regulated-compliance-suite",
    run_name="prod-monitoring-v1",
    evaluators=[
        "correctness",          # Built-in evaluator
        tone_evaluator,         # Custom: insurance tone
        disclosure_evaluator,   # Custom: financial disclosures
        medical_evaluator,      # Custom: medical accuracy
    ],
)

# From here, any traced LLM interaction is automatically evaluated
# against all four evaluators. See the Getting Started cookbook for
# how to instrument your LLM provider with OpenTelemetry tracing.
```

## Next Steps

- [Getting Started](getting-started.md) -- set up tracing and run your first evaluation
- [Automated QA for Call Center Transcripts](call-center-qa.md) -- evaluate batch datasets with built-in and custom metrics
- [CI/CD Quality Gates](cicd-quality-gates.md) -- enforce evaluation thresholds in your deployment pipeline
