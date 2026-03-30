# Detect Hallucinations in a RAG Pipeline

Build a Retrieval-Augmented Generation pipeline with automatic hallucination detection, so every answer your LLM produces is evaluated against the retrieved context.

## What You'll Learn

- Build a minimal RAG pipeline using OpenAI embeddings and ChromaDB
- Configure `InputMapping` to connect RAG fields (query, answer, retrieved context) to Prisma evaluators
- Detect when the LLM fabricates information that is not present in the retrieved documents
- Review hallucination and correctness pass rates in the Prisma project overview

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- An OpenAI API key
- Access to a running Prisma instance
- Familiarity with the basics covered in [Getting Started](getting-started.md)

## 1. Install Dependencies

```bash
pip install prisma-ai openai openinference-instrumentation-openai chromadb
```

`chromadb` provides a local vector store for document retrieval. In production you would swap this for a managed vector database, but the Prisma instrumentation and evaluation setup stays identical.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
export OPENAI_API_KEY="sk-your-openai-key"
```

## 3. Create a Knowledge Base

Start a file called `rag_hallucination.py`. First, define a small set of documents that the RAG pipeline will use as its source of truth. In a real system these might come from a regulatory corpus, internal wiki, or product documentation.

```python
"""RAG hallucination detection with Prisma AI evaluation."""

# -- Knowledge base ----------------------------------------------------------
# Each document represents a fact the retriever can surface.
DOCUMENTS = [
    {
        "id": "doc-1",
        "text": (
            "Prisma AI is an LLM evaluation platform founded in 2024. "
            "It provides automatic evaluation of LLM outputs using configurable "
            "evaluators for correctness, hallucination, and custom policies."
        ),
    },
    {
        "id": "doc-2",
        "text": (
            "The Prisma AI SDK supports Python 3.10 and above. "
            "Users install it via pip with the package name prisma-ai. "
            "The SDK integrates with OpenTelemetry for distributed tracing."
        ),
    },
    {
        "id": "doc-3",
        "text": (
            "Prisma AI evaluators run server-side. When traces arrive at the "
            "Prisma instance, the evaluation pipeline scores each interaction "
            "based on the evaluators configured during prisma_ai.init()."
        ),
    },
    {
        "id": "doc-4",
        "text": (
            "Acme Corp's employee handbook states that the annual leave policy "
            "grants 20 days of paid time off per calendar year. Unused days "
            "do not carry over to the next year."
        ),
    },
    {
        "id": "doc-5",
        "text": (
            "Acme Corp's data retention policy requires that all customer data "
            "be deleted within 90 days of account closure. Audit logs must be "
            "retained for a minimum of 7 years."
        ),
    },
]
```

## 4. Build the Vector Store

Use ChromaDB to embed and index the documents. OpenAI's embedding model converts each document into a vector so the retriever can find the most relevant passages for a given query.

```python
import chromadb
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction

# ChromaDB uses OpenAI embeddings to index and search documents
embedding_fn = OpenAIEmbeddingFunction(
    model_name="text-embedding-3-small",
)

chroma_client = chromadb.Client()
collection = chroma_client.create_collection(
    name="knowledge_base",
    embedding_function=embedding_fn,
)

# Index documents
collection.add(
    ids=[doc["id"] for doc in DOCUMENTS],
    documents=[doc["text"] for doc in DOCUMENTS],
)
```

## 5. Implement the RAG Pipeline

The pipeline has two stages: retrieve relevant context, then generate an answer conditioned on that context. The system prompt instructs the LLM to answer only from the provided context.

```python
from openai import OpenAI

oai_client = OpenAI()

SYSTEM_PROMPT = (
    "You are a helpful assistant. Answer the user's question using ONLY the "
    "context provided below. If the context does not contain enough information "
    "to answer the question, say 'I don't have enough information to answer that.'\n\n"
    "Context:\n{context}"
)


def retrieve(query: str, n_results: int = 2) -> str:
    """Retrieve the top-n most relevant documents for a query."""
    results = collection.query(query_texts=[query], n_results=n_results)
    documents = results["documents"][0]  # first (and only) query
    return "\n\n".join(documents)


def generate(query: str, context: str) -> str:
    """Generate an answer grounded in the retrieved context."""
    response = oai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
            {"role": "user", "content": query},
        ],
        temperature=0.7,
    )
    return response.choices[0].message.content


def rag_query(query: str) -> dict:
    """Run a full RAG cycle: retrieve, then generate."""
    context = retrieve(query)
    answer = generate(query, context)
    return {"query": query, "context": context, "answer": answer}
```

## 6. Initialize Prisma with InputMapping

This is where Prisma ties into your RAG pipeline. The `InputMapping` tells Prisma which span attributes correspond to the user question, the LLM answer, and the retrieved reference context. This mapping is essential for the hallucination evaluator to compare the answer against the retrieved documents.

```python
import prisma_ai
from prisma_ai import InputMapping
from openinference.instrumentation.openai import OpenAIInstrumentor

# Map RAG fields to evaluator inputs:
#   input.value  -> the user's question (default)
#   output.value -> the LLM's answer (default)
#   input.reference -> the retrieved context used for hallucination checking
prisma_ai.init(
    project_name="rag-hallucination-detection",
    run_name="cookbook-run",
    evaluators=["hallucination", "correctness"],
    input_mapping=InputMapping(
        query="input.value",
        response="output.value",
        reference="input.reference",
    ),
)

# Instrument OpenAI so every call is traced
OpenAIInstrumentor().instrument()
```

The `InputMapping` maps evaluation fields to span attributes. The defaults (`input.value`, `output.value`, `input.reference`) match the OpenInference semantic conventions used by the instrumentors. Here they are specified explicitly for clarity — if your spans use different attribute names, adjust the mapping accordingly.

## 7. Run Queries -- Including a Hallucination Scenario

Now run a set of queries. The first two have answers that exist in the knowledge base. The third asks a question the documents do not cover, which is likely to trigger a hallucination if the LLM ignores its instructions and fabricates an answer.

```python
queries = [
    # Answerable from the knowledge base
    "What programming languages does the Prisma AI SDK support?",
    "What is Acme Corp's data retention policy for customer data?",
    # Not in the knowledge base — likely to trigger hallucination
    "What is Prisma AI's pricing model?",
]

for query in queries:
    result = rag_query(query)
    print(f"Q: {result['query']}")
    print(f"Context: {result['context'][:120]}...")
    print(f"A: {result['answer']}")
    print("-" * 60)
```

The third question is the key scenario. The knowledge base contains no pricing information, so a well-behaved RAG pipeline should respond with "I don't have enough information." If the LLM instead invents pricing tiers or dollar amounts, the Prisma hallucination evaluator flags the response because the answer contains claims that are absent from the retrieved context.

## 8. Expected Output

```
Q: What programming languages does the Prisma AI SDK support?
Context: The Prisma AI SDK supports Python 3.10 and above. Users install it via pip with the pa...
A: The Prisma AI SDK supports Python 3.10 and above.
------------------------------------------------------------
Q: What is Acme Corp's data retention policy for customer data?
Context: Acme Corp's data retention policy requires that all customer data be deleted within 90 ...
A: Acme Corp's data retention policy requires that all customer data be deleted within 90 days of account closure. Audit logs must be retained for a minimum of 7 years.
------------------------------------------------------------
Q: What is Prisma AI's pricing model?
Context: Prisma AI is an LLM evaluation platform founded in 2024. It provides automatic evaluat...
A: Prisma AI offers a tiered pricing model starting at $99/month for small teams...
------------------------------------------------------------
```

With `temperature=0.7`, the model is likely to fabricate a pricing answer for the third query even though no pricing information exists in the knowledge base. This is exactly the kind of hallucination that is difficult to catch manually but that Prisma flags automatically.

> **Note:** LLM outputs are non-deterministic. You may see different results on each run. If the model correctly declines to answer, try increasing the temperature or removing the "say so" instruction from the system prompt. In production, hallucinations occur unpredictably — the value of automated evaluation is catching them at scale.

## 9. Review Results in Prisma

Open Prisma and navigate to the **rag-hallucination-detection** project. The project overview shows aggregate metrics across all runs — total validations, HITL requests, and LLM cost. Click on your run to see per-metric summaries:

- **Hallucination pass rate**: The percentage of responses that are grounded in the retrieved context. A response that invents pricing details would be classified as hallucinated; one that correctly declines would pass.
- **Correctness pass rate**: The percentage of responses that are factually accurate given the reference context.
- **Ambiguous count**: Cases where the evaluator was uncertain, which are queued for human expert review.

For granular access to individual evaluation results (e.g., which specific queries triggered hallucination flags), use the Prisma REST API or connect an analytics tool like Power BI or Grafana via Prisma's analytics integration layer.

For regulated industries this gives you an auditable record: every answer your RAG system produces is automatically checked against the source material, with results stored in your Prisma instance.

## Complete Example

```python
"""RAG hallucination detection with Prisma AI — full copy-paste example."""

import chromadb
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction
from openai import OpenAI
from openinference.instrumentation.openai import OpenAIInstrumentor

import prisma_ai
from prisma_ai import InputMapping

# ---------------------------------------------------------------------------
# 1. Knowledge base
# ---------------------------------------------------------------------------
DOCUMENTS = [
    {
        "id": "doc-1",
        "text": (
            "Prisma AI is an LLM evaluation platform founded in 2024. "
            "It provides automatic evaluation of LLM outputs using configurable "
            "evaluators for correctness, hallucination, and custom policies."
        ),
    },
    {
        "id": "doc-2",
        "text": (
            "The Prisma AI SDK supports Python 3.10 and above. "
            "Users install it via pip with the package name prisma-ai. "
            "The SDK integrates with OpenTelemetry for distributed tracing."
        ),
    },
    {
        "id": "doc-3",
        "text": (
            "Prisma AI evaluators run server-side. When traces arrive at the "
            "Prisma instance, the evaluation pipeline scores each interaction "
            "based on the evaluators configured during prisma_ai.init()."
        ),
    },
    {
        "id": "doc-4",
        "text": (
            "Acme Corp's employee handbook states that the annual leave policy "
            "grants 20 days of paid time off per calendar year. Unused days "
            "do not carry over to the next year."
        ),
    },
    {
        "id": "doc-5",
        "text": (
            "Acme Corp's data retention policy requires that all customer data "
            "be deleted within 90 days of account closure. Audit logs must be "
            "retained for a minimum of 7 years."
        ),
    },
]

# ---------------------------------------------------------------------------
# 2. Vector store
# ---------------------------------------------------------------------------
embedding_fn = OpenAIEmbeddingFunction(model_name="text-embedding-3-small")

chroma_client = chromadb.Client()
collection = chroma_client.create_collection(
    name="knowledge_base",
    embedding_function=embedding_fn,
)
collection.add(
    ids=[doc["id"] for doc in DOCUMENTS],
    documents=[doc["text"] for doc in DOCUMENTS],
)

# ---------------------------------------------------------------------------
# 3. RAG pipeline
# ---------------------------------------------------------------------------
oai_client = OpenAI()

SYSTEM_PROMPT = (
    "You are a helpful assistant. Answer the user's question using ONLY the "
    "context provided below. If the context does not contain enough information "
    "to answer the question, say 'I don't have enough information to answer that.'\n\n"
    "Context:\n{context}"
)


def retrieve(query: str, n_results: int = 2) -> str:
    """Retrieve the top-n most relevant documents for a query."""
    results = collection.query(query_texts=[query], n_results=n_results)
    documents = results["documents"][0]
    return "\n\n".join(documents)


def generate(query: str, context: str) -> str:
    """Generate an answer grounded in the retrieved context."""
    response = oai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
            {"role": "user", "content": query},
        ],
        temperature=0.7,
    )
    return response.choices[0].message.content


def rag_query(query: str) -> dict:
    """Run a full RAG cycle: retrieve, then generate."""
    context = retrieve(query)
    answer = generate(query, context)
    return {"query": query, "context": context, "answer": answer}


# ---------------------------------------------------------------------------
# 4. Prisma AI initialization
# ---------------------------------------------------------------------------
prisma_ai.init(
    project_name="rag-hallucination-detection",
    run_name="cookbook-run",
    evaluators=["hallucination", "correctness"],
    input_mapping=InputMapping(
        query="input.value",
        response="output.value",
        reference="input.reference",
    ),
)
OpenAIInstrumentor().instrument()

# ---------------------------------------------------------------------------
# 5. Run queries
# ---------------------------------------------------------------------------
queries = [
    "What programming languages does the Prisma AI SDK support?",
    "What is Acme Corp's data retention policy for customer data?",
    "What is Prisma AI's pricing model?",  # not in knowledge base
]

for query in queries:
    result = rag_query(query)
    print(f"Q: {result['query']}")
    print(f"Context: {result['context'][:120]}...")
    print(f"A: {result['answer']}")
    print("-" * 60)
```

## Next Steps

- [Getting Started](getting-started.md) -- set up Prisma AI on a simple Q&A app if you haven't already
- [Automated QA for Call Center Transcripts](call-center-qa.md) -- evaluate batch datasets with domain-specific metrics
- [Build Custom Policy Compliance Evaluators](custom-policy-evaluators.md) -- create evaluators tailored to your organization's compliance requirements
