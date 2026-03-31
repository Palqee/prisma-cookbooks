# AI Performance Metrics

The previous cookbooks covered policy-based metrics -- Prisma's core capability for validating human-led review workflows. Prisma also includes built-in AI performance metrics for teams that need to measure traditional AI quality signals like correctness and hallucination.

These metrics are useful when you need a quick, automated check on whether an LLM's responses are factually accurate or contain fabricated information -- without writing custom policy prompts.

## What You'll Learn

- Use Prisma's built-in `correctness` and `hallucination` evaluators on Q&A data
- Detect hallucinated content in a RAG pipeline using OpenInference tracing
- Upload evaluation data via batch file upload as an alternative to real-time logging

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **An OpenAI API key** (for the RAG pipeline example)
- **Access to a running Prisma instance**
- Familiarity with the [Getting Started](getting-started.md) guide

## 1. Built-In Metrics Overview

Prisma ships with two built-in AI performance metrics. Unlike policy metrics (which you define with the 4-prompt pattern), these work out of the box -- just pass their names as strings to the `evaluators` list.

| Metric | What it measures | Inputs used |
|---|---|---|
| `correctness` | Whether the response matches a known-correct reference answer in meaning and factual content | `query`, `response`, `reference` |
| `hallucination` | Whether the response contains information that was fabricated -- not present in the reference or retrieved context | `query`, `response`, `reference` |

Both metrics produce a pass/fail result with an explanation.

## 2. Correctness and Hallucination Validation

This example logs a few question/answer pairs with reference answers and validates them against both built-in metrics using the OTLP logs pipeline.

### Install dependencies

```bash
pip install prisma-ai
```

### Configure environment variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

### Log Q&A pairs for evaluation

```python
import prisma_ai

# Initialize with built-in metrics -- pass names as strings, not Evaluators.custom()
session = prisma_ai.init(
    project_name="qa-accuracy-check",
    evaluators=["correctness", "hallucination"],
    enable_logging=True,
)

logger = session.get_logger()

qa_pairs = [
    {
        "query": "What is the capital of France?",
        "response": "The capital of France is Paris.",
        "reference": "Paris is the capital of France.",
    },
    {
        "query": "When was the Python programming language first released?",
        "response": "Python was first released in 1991 by Guido van Rossum.",
        "reference": "Python was first released on February 20, 1991.",
    },
    {
        "query": "What is the speed of light?",
        "response": (
            "The speed of light is approximately 300,000 km/s. It was first "
            "measured accurately by Albert Einstein during his 1905 experiments "
            "at the Swiss Federal Observatory."
        ),
        "reference": (
            "The speed of light in a vacuum is approximately 299,792 km/s. "
            "Ole Roemer made the first quantitative estimate in 1676 based on "
            "observations of Jupiter's moon Io."
        ),
    },
]

for pair in qa_pairs:
    logger.log(
        query=pair["query"],
        response=pair["response"],
        reference=pair["reference"],
    )

session.end()
```

### What the metrics detect

| Q&A Pair | Correctness | Hallucination | Why |
|---|---|---|---|
| Capital of France | Pass | Pass | Response matches reference |
| Python release date | Pass | Pass | Core fact (1991) is correct; additional detail (Guido van Rossum) is accurate |
| Speed of light | Fail | Fail | Einstein did not measure the speed of light at a "Swiss Federal Observatory" -- this is fabricated information |

The third pair demonstrates both metrics working together: `correctness` flags the inaccurate attribution, and `hallucination` flags the fabricated detail about Einstein's experiments.

## 3. Hallucination Detection in RAG Pipelines

When your LLM generates answers from retrieved documents, hallucination detection becomes critical -- the model may fabricate information that is not present in the retrieved context. This example uses OpenInference tracing instead of the DataLogger, which captures the full trace of the OpenAI call alongside Prisma's evaluation.

### Install dependencies

```bash
pip install prisma-ai openai openinference-instrumentation-openai chromadb
```

### Configure environment variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
export OPENAI_API_KEY="sk-your-openai-key"
```

### Build a minimal RAG pipeline with tracing

```python
import prisma_ai
from openai import OpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor
import chromadb

# 1. Initialize Prisma with hallucination detection
prisma_ai.init(
    project_name="rag-hallucination-check",
    evaluators=["hallucination"],
)

# 2. Instrument OpenAI -- all completions are now traced automatically
OpenAIInstrumentor().instrument()

client = OpenAI()

# 3. Set up a local vector store with some documents
chroma = chromadb.Client()
collection = chroma.create_collection("company_docs")

documents = [
    "Acme Corp was founded in 2019 in Austin, Texas.",
    "Acme Corp has 150 employees as of January 2026.",
    "Acme Corp's annual revenue for 2025 was $12 million.",
]

collection.add(
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))],
)

# 4. Query the RAG pipeline
question = "How many employees does Acme Corp have and who is the CEO?"

results = collection.query(query_texts=[question], n_results=3)
context = "\n".join(results["documents"][0])

# 5. Generate a response using retrieved context
completion = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "system",
            "content": (
                "Answer the question using only the provided context. "
                "If the context does not contain the answer, say so.\n\n"
                f"Context:\n{context}"
            ),
        },
        {"role": "user", "content": question},
    ],
)

print(completion.choices[0].message.content)
```

### What happens

The retrieved context contains Acme Corp's employee count (150) but says nothing about a CEO. A well-behaved model would answer the employee question and note that CEO information is not available. If the model instead fabricates a CEO name -- for example, "The CEO is John Smith" -- Prisma's `hallucination` evaluator flags it.

Because OpenInference tracing is active, Prisma receives the full trace: the prompt, the retrieved context, and the completion. The hallucination evaluator checks whether every claim in the response is grounded in the context that was actually retrieved.

## 4. Alternative: Batch File Upload

For teams that already have evaluation datasets in CSV format, Prisma supports batch file upload as an alternative to real-time logging.

```python
from pq_prisma_client import PrismaClient

client = PrismaClient()
dataset = await client.datasets.upload("qa_data.csv")
```

The CSV should include columns for `query`, `response`, and `reference`. Once uploaded, you can run any combination of built-in metrics or policy metrics against the dataset from the Prisma dashboard.

## Next Steps

This cookbook covered Prisma's built-in AI performance metrics. For Prisma's core capabilities -- encoding your organization's rules as automated validators with human-in-the-loop feedback -- see:

- [Getting Started](getting-started.md) -- set up policy-based validation in 5 minutes
- [Building Policy-Based Metrics](policy-based-metrics.md) -- the 4-prompt pattern for turning any compliance rule into an automated validator
- [Human Review & Evaluation Memory](human-review.md) -- expert feedback that makes your validators smarter over time
