# Getting Started: Evaluate Your First LLM App in 5 Minutes

Set up Prisma AI tracing and automatic evaluation on an OpenAI-powered application with just a few lines of code.

## What You'll Learn

- Install and configure the Prisma AI SDK
- Instrument an OpenAI chat application with automatic tracing
- Enable correctness and hallucination evaluation on every LLM call
- Access evaluation results via the Prisma project overview and REST API

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- An OpenAI API key
- Access to a running Prisma instance

## 1. Install Dependencies

```bash
pip install prisma-ai openai openinference-instrumentation-openai
```

The `prisma-ai` package is the unified SDK that includes both tracing (`prisma-otel`) and the platform client (`pq-prisma-client`). The `openinference-instrumentation-openai` package automatically captures OpenAI calls as OpenTelemetry spans.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
export OPENAI_API_KEY="sk-your-openai-key"
```

## 3. Initialize Prisma AI

Create a file called `app.py`:

```python
import prisma_ai
from openai import OpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor

# Initialize Prisma: creates a project, a run, and configures tracing
prisma_ai.init(
    project_name="my-first-app",
    evaluators=["correctness", "hallucination"],
)

# Instrument OpenAI — all calls are now traced automatically
OpenAIInstrumentor().instrument()
```

That's it. Three lines to set up tracing and evaluation:

1. `prisma_ai.init()` creates your project and run on the Prisma platform, stores the evaluation configuration, and configures OpenTelemetry tracing.
2. `OpenAIInstrumentor().instrument()` patches the OpenAI client so every API call produces a trace span.
3. Traces flow to your Prisma instance, where the evaluation pipeline automatically scores each interaction for correctness and hallucination.

## 4. Build a Simple Q&A Application

Add the following to `app.py`:

```python
client = OpenAI()

questions = [
    "What is the capital of France?",
    "How many planets are in our solar system?",
    "Who wrote Romeo and Juliet?",
]

for question in questions:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer questions accurately and concisely."},
            {"role": "user", "content": question},
        ],
    )
    answer = response.choices[0].message.content
    print(f"Q: {question}")
    print(f"A: {answer}\n")
```

## 5. Run the Application

```bash
python app.py
```

Expected output:

```
Q: What is the capital of France?
A: The capital of France is Paris.

Q: How many planets are in our solar system?
A: There are 8 planets in our solar system.

Q: Who wrote Romeo and Juliet?
A: Romeo and Juliet was written by William Shakespeare.
```

Behind the scenes, each OpenAI call generates a trace span that is sent to your Prisma instance. The evaluation pipeline picks up these traces and runs correctness and hallucination checks automatically.

## 6. View Results in Prisma

Open Prisma and navigate to the **my-first-app** project. The project overview shows:

- **Total Runs**: How many evaluation runs are active or completed
- **Validations**: Total validations performed across all metrics
- **HITL Requests**: Ambiguous cases queued for human expert review
- **LLM Cost**: Token usage and cost for the evaluation pipeline

Click on your run to see per-metric summaries — pass rates, failure counts, and ambiguous classifications for correctness and hallucination.

For granular access to individual evaluation results, use the Prisma REST API or connect your preferred analytics tool (Power BI, Tableau, Grafana) via Prisma's analytics integration layer to build custom views over your evaluation data.

## 7. Customize Your Setup

### Change the run name

By default, `init()` generates a random run name. You can set your own:

```python
prisma_ai.init(
    project_name="my-first-app",
    run_name="experiment-v1",
    evaluators=["correctness", "hallucination"],
)
```

### Update evaluation config mid-session

```python
from prisma_ai import EvaluationConfig

session = prisma_ai.init(
    project_name="my-first-app",
    evaluators=["correctness"],
)

# Later, add hallucination detection
session.update_evaluation_config(
    EvaluationConfig(
        evaluators=["correctness", "hallucination"],
    )
)
```

### Use the async client

The pattern works identically with the async OpenAI client. The same `prisma_ai.init()` and `OpenAIInstrumentor().instrument()` setup from above applies — the instrumentor patches both sync and async clients:

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def main():
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "What is 2 + 2?"}],
    )
    print(response.choices[0].message.content)

asyncio.run(main())
```

## Complete Example

```python
"""Prisma AI Getting Started — full example."""

import prisma_ai
from openai import OpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor

# 1. Initialize Prisma with evaluation
prisma_ai.init(
    project_name="my-first-app",
    evaluators=["correctness", "hallucination"],
)

# 2. Instrument OpenAI
OpenAIInstrumentor().instrument()

# 3. Use OpenAI as normal
client = OpenAI()

questions = [
    "What is the capital of France?",
    "How many planets are in our solar system?",
    "Who wrote Romeo and Juliet?",
]

for question in questions:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Answer questions accurately and concisely."},
            {"role": "user", "content": question},
        ],
    )
    print(f"Q: {question}")
    print(f"A: {response.choices[0].message.content}\n")

# 4. Clean up (optional — also runs automatically on exit)
session = prisma_ai.get_session()
if session:
    session.end()
```

## Next Steps

- [Detect Hallucinations in a RAG Pipeline](rag-hallucination-detection.md) — add retrieval context and evaluate RAG quality
- [Automated QA for Call Center Transcripts](call-center-qa.md) — evaluate batch datasets with custom metrics
- [Build Custom Policy Compliance Evaluators](custom-policy-evaluators.md) — create domain-specific evaluation criteria
