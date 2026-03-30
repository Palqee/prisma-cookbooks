# Real-Time Evaluation of Production LLM Traffic

Add automatic evaluation to a production FastAPI application so every LLM interaction is scored for correctness, hallucination, and custom criteria without adding latency to your API responses.

## What You'll Learn

- Integrate Prisma AI tracing and evaluation into a production FastAPI service using lifespan events
- Instrument multiple LLM frameworks (OpenAI, LangChain, and others) simultaneously
- Update evaluation configuration on a live application without restarting
- Handle graceful shutdown, OTEL batching, and environment-specific evaluator config

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- An OpenAI API key
- Access to a running Prisma instance
- Familiarity with the basics covered in [Getting Started](getting-started.md)

## 1. Install Dependencies

```bash
pip install prisma-ai openai openinference-instrumentation-openai fastapi uvicorn
```

The `fastapi` and `uvicorn` packages provide the web framework and ASGI server. The OpenInference instrumentor patches the OpenAI client so every API call is captured as an OpenTelemetry span and forwarded to your Prisma instance for evaluation.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
export OPENAI_API_KEY="sk-your-openai-key"
```

## 3. Define a Custom Evaluator

Before building the application, define a custom "tone" evaluator that checks whether LLM responses use an appropriate professional tone for customer support. This runs alongside the built-in `correctness` and `hallucination` evaluators.

Custom evaluators require four prompts: an evaluation system prompt, an evaluation human prompt, a re-evaluation system prompt, and a re-evaluation human prompt. The re-evaluation prompts are used when the initial evaluation is ambiguous — Prisma routes ambiguous cases through a second pass with additional context from past human reviews.

```python
from prisma_ai import Evaluators

tone_evaluator = Evaluators.custom(
    name="tone",
    options=["professional", "acceptable", "unprofessional"],
    evaluation_system_prompt=(
        "You are a customer support quality auditor evaluating the tone "
        "and professionalism of agent responses. Assess whether the response "
        "is empathetic, clear, and uses appropriate language for customer-facing "
        "communication. Classify into one of: {evaluation_options}."
    ),
    evaluation_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Classify the tone of the agent response. Choose from: {evaluation_options}."
    ),
    reeval_system_prompt=(
        "You are a senior customer support quality auditor performing a second "
        "review of a tone evaluation that was initially ambiguous. Use the "
        "additional context from prior reviews to make a more confident assessment."
    ),
    reeval_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Prior review context: {memories}\n\n"
        "Based on the response and prior review context, classify the tone "
        "as one of: professional, acceptable, or unprofessional."
    ),
)
```

The `{query}` and `{response}` placeholders are replaced with the actual user message and LLM response from the trace span. The `{evaluation_options}` placeholder is replaced with the options list. In re-evaluation, `{memories}` contains context from prior human reviews of similar cases.

## 4. Initialize Prisma in the FastAPI Lifespan

The FastAPI lifespan context manager is the right place to initialize and shut down Prisma. `prisma_ai.init()` is synchronous and safe to call here. On shutdown, `session.end()` flushes any pending spans to ensure no traces are lost.

Create a file called `app.py`:

```python
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from openinference.instrumentation.openai import OpenAIInstrumentor

import prisma_ai
from prisma_ai import EvaluationConfig, Evaluators, get_session

logger = logging.getLogger("chatbot")


tone_evaluator = Evaluators.custom(
    name="tone",
    options=["professional", "acceptable", "unprofessional"],
    evaluation_system_prompt=(
        "You are a customer support quality auditor evaluating the tone "
        "and professionalism of agent responses. Assess whether the response "
        "is empathetic, clear, and uses appropriate language for customer-facing "
        "communication. Classify into one of: {evaluation_options}."
    ),
    evaluation_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Classify the tone of the agent response. Choose from: {evaluation_options}."
    ),
    reeval_system_prompt=(
        "You are a senior customer support quality auditor performing a second "
        "review of a tone evaluation that was initially ambiguous. Use the "
        "additional context from prior reviews to make a more confident assessment."
    ),
    reeval_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Prior review context: {memories}\n\n"
        "Based on the response and prior review context, classify the tone "
        "as one of: professional, acceptable, or unprofessional."
    ),
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialize Prisma tracing and evaluation
    prisma_ai.init(
        project_name="customer-support-chatbot",
        run_name="production",
        evaluators=["correctness", "hallucination", tone_evaluator],
    )
    OpenAIInstrumentor().instrument()

    session = get_session()
    if session:
        logger.info(
            "Prisma AI initialized — project=%s run=%s",
            session.project_id,
            session.run_id,
        )

    yield

    # Shutdown: flush pending spans and close the session
    session = get_session()
    if session:
        session.end()
        logger.info("Prisma AI session ended, pending spans flushed")


app = FastAPI(title="Customer Support Chatbot", lifespan=lifespan)
```

Key points:

- `prisma_ai.init()` creates the project, the run with evaluation configuration, and configures OpenTelemetry. All three evaluators (correctness, hallucination, and the custom tone evaluator) run in parallel on every trace.
- `OpenAIInstrumentor().instrument()` patches the OpenAI client globally. Every `chat.completions.create` call now produces a trace span automatically.
- `get_session()` returns the active `PrismaSession`, which provides `project_id` and `run_id` for observability logging.
- `session.end()` on shutdown ensures the OTEL batch processor flushes all pending spans before the process exits.

## 5. Build the Chat Endpoint

Add a `/chat` endpoint that receives a customer message and returns an AI response. The OpenAI call is automatically traced and evaluated — your application code does not need to know about Prisma.

```python
from pydantic import BaseModel
from openai import AsyncOpenAI

oai_client = AsyncOpenAI()

SYSTEM_PROMPT = (
    "You are a helpful customer support assistant for a software company. "
    "Be professional, empathetic, and concise. If you don't know the answer, "
    "say so honestly rather than guessing."
)


class ChatRequest(BaseModel):
    message: str


class ChatResponse(BaseModel):
    reply: str


@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest) -> ChatResponse:
    response = await oai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": request.message},
        ],
        temperature=0.3,
    )
    reply = response.choices[0].message.content or ""
    return ChatResponse(reply=reply)
```

This is a standard FastAPI endpoint. The evaluation happens asynchronously on the Prisma server side — the `/chat` response is returned to the caller immediately, with no added latency. Traces are batched by the OTEL SDK and sent in the background.

## 6. Instrument Multiple Frameworks

If your application uses more than one LLM framework, you can instrument them all. Each instrumentor patches its respective client independently.

```python
from openinference.instrumentation.openai import OpenAIInstrumentor

# Always instrument OpenAI
OpenAIInstrumentor().instrument()

# If you also use LangChain in the same application:
# pip install openinference-instrumentation-langchain
# from openinference.instrumentation.langchain import LangChainInstrumentor
# LangChainInstrumentor().instrument()
```

Other available instrumentors follow the same pattern:

- `openinference-instrumentation-anthropic` for Anthropic
- `openinference-instrumentation-bedrock` for AWS Bedrock
- `openinference-instrumentation-llama-index` for LlamaIndex

Install the package and call `.instrument()` at startup. All traced calls from all frameworks flow to the same Prisma run and are evaluated by the same set of evaluators.

## 7. Monitor Evaluation Health

Use `get_session()` at any point to inspect the active Prisma session. This is useful for health-check endpoints and operational logging.

```python
@app.get("/health")
async def health():
    session = get_session()
    if session is None:
        return {"status": "degraded", "prisma": "not initialized"}
    return {
        "status": "healthy",
        "prisma_project_id": session.project_id,
        "prisma_run_id": session.run_id,
    }
```

For real-time evaluation metrics, open Prisma and navigate to your project. The project overview shows total validations, HITL requests pending expert review, and LLM cost. Click on the active run to see per-metric pass rates, failure counts, and ambiguous classifications as evaluations arrive. For granular per-request data, use the Prisma REST API or connect an analytics tool like Power BI or Grafana via the analytics integration layer.

## 8. Update Evaluation Config Without Restart

You can add or change evaluators on a live application without restarting the process. This is useful for A/B testing new evaluators or responding to incidents.

> **Note:** In production, protect administrative endpoints with authentication middleware. This example omits auth for brevity.

```python
@app.post("/admin/update-evaluators")
async def update_evaluators():
    session = get_session()
    if session is None:
        return {"error": "Prisma session not active"}

    pii_evaluator = Evaluators.custom(
        name="pii-detection",
        options=["clean", "contains_pii"],
        evaluation_system_prompt=(
            "You are a data privacy auditor checking whether AI responses "
            "inadvertently expose personally identifiable information (PII). "
            "Look for names, email addresses, phone numbers, physical addresses, "
            "account numbers, or any other identifying data. "
            "Classify into one of: {evaluation_options}."
        ),
        evaluation_human_prompt=(
            "Customer message: {query}\n"
            "Agent response: {response}\n\n"
            "Does the agent response contain any PII? "
            "Choose from: {evaluation_options}."
        ),
        reeval_system_prompt=(
            "You are a senior data privacy auditor reviewing a PII detection "
            "evaluation that was initially ambiguous. Use context from prior "
            "reviews to determine if the flagged content is truly PII."
        ),
        reeval_human_prompt=(
            "Customer message: {query}\n"
            "Agent response: {response}\n\n"
            "Prior review context: {memories}\n\n"
            "Determine whether the response contains PII. "
            "Classify as: clean or contains_pii."
        ),
    )

    session.update_evaluation_config(
        EvaluationConfig(
            evaluators=[
                "correctness",
                "hallucination",
                tone_evaluator,
                pii_evaluator,
            ],
        )
    )
    return {"status": "evaluation config updated, PII detection added"}
```

After this call, all new traces will be evaluated with the updated set of evaluators. Traces that were already submitted continue to use the configuration that was active when they arrived.

## 9. Production Considerations

### OTEL Span Batching

The OpenTelemetry SDK batches spans before sending them to your Prisma instance. This means spans are not sent one-by-one with each request — they are collected and flushed in batches. This is efficient and avoids adding per-request overhead, but it means there is a short delay (typically a few seconds) before evaluations appear in the project overview.

### Graceful Shutdown

Calling `session.end()` in the lifespan shutdown handler is important. It signals the OTEL batch processor to flush any pending spans before the process exits. Without this, spans generated in the final seconds before shutdown could be lost.

### Environment-Specific Configuration

Use different evaluators for different environments. In staging, you might run more expensive evaluators to catch issues before production. In production, you might focus on the most critical checks.

```python
import os

environment = os.getenv("APP_ENV", "production")

if environment == "staging":
    evaluators = ["correctness", "hallucination", tone_evaluator]
else:
    evaluators = ["correctness", "hallucination"]

prisma_ai.init(
    project_name="customer-support-chatbot",
    run_name=environment,
    evaluators=evaluators,
)
```

## Expected Output

Start the server:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

Send a request:

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "I can not log in to my account. Can you help?"}'
```

Response:

```json
{
  "reply": "I'm sorry to hear you're having trouble logging in. I'd be happy to help. Could you let me know if you're seeing a specific error message? In the meantime, you can try resetting your password using the 'Forgot Password' link on the login page."
}
```

Behind the scenes, the OpenAI call generates a trace span that is batched and sent to your Prisma instance. The evaluation pipeline then scores the interaction:

- **Correctness**: Is the response factually sound?
- **Hallucination**: Does the response contain fabricated information?
- **Tone**: Does the response meet professional customer support standards?

All three evaluators run in parallel on the server side. Aggregate results appear in the Prisma project overview within seconds. Use the REST API or analytics integrations for per-request detail.

## Complete Example

```python
"""Real-time evaluation of production LLM traffic with Prisma AI."""

import logging
import os
from contextlib import asynccontextmanager

from fastapi import FastAPI
from openai import AsyncOpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor
from pydantic import BaseModel

import prisma_ai
from prisma_ai import EvaluationConfig, Evaluators, get_session

logger = logging.getLogger("chatbot")

# ---------------------------------------------------------------------------
# 1. Custom evaluator definition
# ---------------------------------------------------------------------------
tone_evaluator = Evaluators.custom(
    name="tone",
    options=["professional", "acceptable", "unprofessional"],
    evaluation_system_prompt=(
        "You are a customer support quality auditor evaluating the tone "
        "and professionalism of agent responses. Assess whether the response "
        "is empathetic, clear, and uses appropriate language for customer-facing "
        "communication. Classify into one of: {evaluation_options}."
    ),
    evaluation_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Classify the tone of the agent response. Choose from: {evaluation_options}."
    ),
    reeval_system_prompt=(
        "You are a senior customer support quality auditor performing a second "
        "review of a tone evaluation that was initially ambiguous. Use the "
        "additional context from prior reviews to make a more confident assessment."
    ),
    reeval_human_prompt=(
        "Customer message: {query}\n"
        "Agent response: {response}\n\n"
        "Prior review context: {memories}\n\n"
        "Based on the response and prior review context, classify the tone "
        "as one of: professional, acceptable, or unprofessional."
    ),
)

# ---------------------------------------------------------------------------
# 2. FastAPI lifespan — Prisma init and shutdown
# ---------------------------------------------------------------------------
environment = os.getenv("APP_ENV", "production")

if environment == "staging":
    evaluators = ["correctness", "hallucination", tone_evaluator]
else:
    evaluators = ["correctness", "hallucination"]


@asynccontextmanager
async def lifespan(app: FastAPI):
    prisma_ai.init(
        project_name="customer-support-chatbot",
        run_name=environment,
        evaluators=evaluators,
    )
    OpenAIInstrumentor().instrument()

    session = get_session()
    if session:
        logger.info(
            "Prisma AI initialized — project=%s run=%s",
            session.project_id,
            session.run_id,
        )

    yield

    session = get_session()
    if session:
        session.end()
        logger.info("Prisma AI session ended, pending spans flushed")


app = FastAPI(title="Customer Support Chatbot", lifespan=lifespan)

# ---------------------------------------------------------------------------
# 3. Chat endpoint
# ---------------------------------------------------------------------------
oai_client = AsyncOpenAI()

SYSTEM_PROMPT = (
    "You are a helpful customer support assistant for a software company. "
    "Be professional, empathetic, and concise. If you don't know the answer, "
    "say so honestly rather than guessing."
)


class ChatRequest(BaseModel):
    message: str


class ChatResponse(BaseModel):
    reply: str


@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest) -> ChatResponse:
    response = await oai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": request.message},
        ],
        temperature=0.3,
    )
    reply = response.choices[0].message.content or ""
    return ChatResponse(reply=reply)


# ---------------------------------------------------------------------------
# 4. Health check
# ---------------------------------------------------------------------------
@app.get("/health")
async def health():
    session = get_session()
    if session is None:
        return {"status": "degraded", "prisma": "not initialized"}
    return {
        "status": "healthy",
        "prisma_project_id": session.project_id,
        "prisma_run_id": session.run_id,
    }
```

## Next Steps

- [Getting Started](getting-started.md) — set up Prisma AI on a simple Q&A app if you haven't already
- [Detect Hallucinations in a RAG Pipeline](rag-hallucination-detection.md) — add retrieval context and evaluate RAG quality
- [Build Custom Policy Compliance Evaluators](custom-policy-evaluators.md) — create domain-specific evaluation criteria for regulated industries
- [CI/CD Quality Gates](cicd-quality-gates.md) — integrate evaluation scores into your deployment pipeline
