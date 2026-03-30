# CI/CD Quality Gates for LLM Applications

Automatically catch LLM regressions before they reach production by integrating Prisma evaluation into your CI/CD pipeline.

## What You'll Learn

- How to build a golden test dataset that captures expected LLM behavior
- How to write an evaluation gate script that calls the Prisma sync evaluation API
- How to define and enforce quality thresholds (correctness, hallucination)
- How to wire everything into GitHub Actions and pytest

## Prerequisites

- A Prisma AI account with an API key
- Python 3.10+
- An LLM endpoint you want to test (OpenAI used in examples)
- A GitHub repository (for the Actions integration)

## Setup

Install the required packages:

```bash
pip install httpx openai pytest
```

Set the following environment variables. In CI, add these as repository secrets:

```bash
export PRISMA_API_KEY="your-prisma-api-key"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export OPENAI_API_KEY="your-openai-api-key"
```

## Step 1: Create a Golden Test Dataset

A golden dataset is a curated set of queries with known-good reference answers. These are **not training data** — they are behavioral specifications for your application, similar to regression test fixtures in traditional software. Each query tests whether your application's pipeline (system prompt, retrieval, post-processing) still produces correct output.

The example below assumes a **compliance documentation assistant** — an LLM application that answers employee questions about company policies using retrieved internal documents. Save this as `tests/eval/golden_dataset.json`:

```json
[
  {
    "query": "What is the maximum reimbursement for a domestic flight under our travel policy?",
    "reference": "Domestic flights are reimbursed up to $600 per round trip. Flights exceeding this amount require VP-level pre-approval."
  },
  {
    "query": "Who do I contact to report a potential GDPR data breach?",
    "reference": "Report potential GDPR breaches to the Data Protection Officer (dpo@company.com) within 24 hours of discovery. The DPO will coordinate the 72-hour supervisory authority notification."
  },
  {
    "query": "Can a contractor access our production customer database?",
    "reference": "No. Under Policy SEC-401, contractors are restricted to sandbox and staging environments. Production access requires full-time employee status and completion of SOC 2 access training."
  },
  {
    "query": "What is the retention period for client communication records in financial services?",
    "reference": "Client communication records must be retained for a minimum of 7 years per SEC Rule 17a-4 and our internal policy FIN-203."
  },
  {
    "query": "How do I escalate a customer complaint that involves a regulatory violation?",
    "reference": "Regulatory complaints must be escalated immediately to the Compliance team via the #compliance-escalations channel and logged in the incident tracker within 4 hours. Do not attempt to resolve regulatory matters directly with the customer."
  },
  {
    "query": "What training is required before handling PII data?",
    "reference": "All employees must complete the annual Data Privacy Foundations course and the role-specific PII Handling certification before accessing any system containing personally identifiable information."
  },
  {
    "query": "Is it acceptable to use a personal device for accessing internal tools while traveling?",
    "reference": "Personal devices may access internal tools only through the approved MDM-enrolled browser profile. Downloading company data to personal device storage is prohibited under Policy SEC-112."
  },
  {
    "query": "What are the approval requirements for a vendor contract over $50,000?",
    "reference": "Vendor contracts exceeding $50,000 require sign-off from the department head, Procurement, and Legal. Contracts over $200,000 additionally require CFO approval."
  },
  {
    "query": "How frequently must access reviews be conducted for SOX-critical systems?",
    "reference": "SOX-critical systems require quarterly access reviews conducted by the system owner, with results submitted to Internal Audit within 10 business days of the quarter close."
  },
  {
    "query": "What should I do if I discover our application is logging sensitive cardholder data?",
    "reference": "Immediately notify the Security team and cease the logging. Under PCI DSS Requirement 3.4, cardholder data must never be stored in plaintext logs. File a P1 incident and reference Policy PCI-007."
  },
  {
    "query": "Summarize the company's policy on using generative AI tools with client data.",
    "reference": "Client data must not be entered into any external generative AI tool unless it has been approved by the AI Governance Board and appears on the Approved AI Tools list. Internal tools with data residency guarantees are permitted for classification and summarization tasks."
  },
  {
    "query": "What happens if an employee fails to complete mandatory compliance training by the deadline?",
    "reference": "Employees who miss the compliance training deadline have their system access suspended until completion. After 30 days, the matter is escalated to HR and the employee's manager for a formal performance note."
  }
]
```

Each of these queries tests something your **application pipeline** must get right — the correct answer depends on retrieved company documents and your system prompt, not on the LLM's general knowledge. If a code change breaks your retrieval logic or alters your system prompt, these queries will catch it.

Keep this dataset small (10–15 examples) so evaluation runs fast in CI. Focus on high-value queries that cover your application's critical behavioral categories: policy lookup, escalation procedures, access control rules, and regulatory requirements.

## Step 2: Write the Evaluation Gate Script

This script loads the golden dataset, runs each query through your LLM, sends the results to Prisma's sync evaluation API, and checks whether scores meet your thresholds.

```python
# tests/eval/eval_gate.py
"""CI/CD quality gate for LLM evaluation using Prisma AI."""

import json
import sys
import os
from pathlib import Path

import httpx
from openai import OpenAI


# --- Configuration ---

PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]
PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]

# Quality thresholds — adjust these to match your requirements
THRESHOLDS = {
    "correctness": 0.9,     # At least 90% of responses must be correct
    "hallucination": 0.1,   # No more than 10% hallucination rate
}

EVALUATORS = ["correctness", "hallucination"]

DATASET_PATH = Path(__file__).parent / "golden_dataset.json"

# Your LLM configuration — match your application's actual settings
LLM_MODEL = "gpt-4o"
LLM_SYSTEM_PROMPT = (
    "You are a compliance documentation assistant for a regulated financial services company. "
    "Answer employee questions using only the retrieved policy documents provided as context. "
    "Always cite the specific policy number. If the answer is not in the provided documents, "
    "say so — do not guess or use general knowledge."
)


def load_golden_dataset() -> list[dict]:
    """Load the golden Q&A dataset from disk."""
    with open(DATASET_PATH) as f:
        return json.load(f)


def generate_responses(dataset: list[dict]) -> list[dict]:
    """Run each query through the LLM and build evaluation records."""
    client = OpenAI(api_key=OPENAI_API_KEY)
    records = []

    for item in dataset:
        completion = client.chat.completions.create(
            model=LLM_MODEL,
            messages=[
                {"role": "system", "content": LLM_SYSTEM_PROMPT},
                {"role": "user", "content": item["query"]},
            ],
            max_tokens=256,
        )
        response = completion.choices[0].message.content

        records.append({
            "query": item["query"],
            "response": response,
            "reference": item["reference"],
        })

    return records


def evaluate_with_prisma(records: list[dict]) -> dict:
    """Send records to the Prisma sync evaluation API and return results."""
    url = f"{PRISMA_BASE_URL}/api/v1/evaluate"

    payload = {
        "records": records,
        "evaluators": EVALUATORS,
        "timeout_seconds": 120,
    }

    with httpx.Client(timeout=180) as client:
        response = client.post(
            url,
            json=payload,
            headers={
                "x-api-key": PRISMA_API_KEY,
                "Content-Type": "application/json",
            },
        )
        response.raise_for_status()
        return response.json()


def check_thresholds(summary: dict[str, float]) -> tuple[bool, list[str]]:
    """Check evaluation summary against quality thresholds.

    Returns a tuple of (passed, list of failure messages).
    """
    failures = []

    for evaluator, threshold in THRESHOLDS.items():
        score = summary.get(evaluator)
        if score is None:
            failures.append(f"  {evaluator}: score missing from results")
            continue

        if evaluator == "hallucination":
            # Hallucination score should be BELOW threshold
            if score > threshold:
                failures.append(
                    f"  {evaluator}: {score:.2f} > {threshold:.2f} (too high)"
                )
        else:
            # Other scores should be ABOVE threshold
            if score < threshold:
                failures.append(
                    f"  {evaluator}: {score:.2f} < {threshold:.2f} (too low)"
                )

    return len(failures) == 0, failures


def main() -> int:
    """Run the full evaluation gate. Returns 0 on pass, 1 on failure."""
    print("Loading golden dataset...")
    dataset = load_golden_dataset()
    print(f"  {len(dataset)} test cases loaded")

    print("Generating LLM responses...")
    records = generate_responses(dataset)
    print(f"  {len(records)} responses generated")

    print("Running Prisma evaluation...")
    result = evaluate_with_prisma(records)

    summary = result["summary"]
    processing_time = result.get("processing_time_ms", 0)
    print(f"  Evaluation completed in {processing_time}ms")
    print(f"  Scores: {json.dumps(summary, indent=2)}")

    passed, failures = check_thresholds(summary)

    if passed:
        print("\nQUALITY GATE PASSED")
        return 0
    else:
        print("\nQUALITY GATE FAILED")
        for msg in failures:
            print(msg)
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

## Step 3: Define Quality Thresholds

The thresholds in the script above are a starting point. Here is how to think about tuning them:

| Evaluator | Threshold | Direction | Rationale |
|---|---|---|---|
| `correctness` | >= 0.90 | Higher is better | 90% of responses must be factually correct against reference answers |
| `hallucination` | <= 0.10 | Lower is better | No more than 10% of responses should contain fabricated information |

**Tips for threshold tuning:**

- **Start permissive, tighten over time.** Begin with `correctness >= 0.8` and raise it as your LLM pipeline matures.
- **Use separate thresholds per environment.** Staging might tolerate `correctness >= 0.85` while production requires `>= 0.95`.
- **Track trends, not just pass/fail.** A score dropping from 0.98 to 0.91 still passes but signals a regression worth investigating.

## Step 4: GitHub Actions Integration

Add this workflow to your repository. It runs the evaluation gate on every pull request and blocks merging if quality drops below thresholds.

```yaml
# .github/workflows/llm-quality-gate.yml
name: LLM Quality Gate

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  llm-eval:
    name: Evaluate LLM Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install httpx openai pytest

      - name: Run evaluation gate
        env:
          PRISMA_API_KEY: ${{ secrets.PRISMA_API_KEY }}
          PRISMA_BASE_URL: ${{ secrets.PRISMA_BASE_URL }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: python tests/eval/eval_gate.py
```

To make this a required check, go to your repository's **Settings > Branches > Branch protection rules** and add `llm-eval` as a required status check for the `main` branch.

## Step 5: Pytest Integration

If you prefer running evaluation as part of your existing test suite, wrap the gate logic in a pytest test:

```python
# tests/eval/test_llm_quality.py
"""Pytest integration for LLM quality gate."""

import json
import os
from pathlib import Path

import httpx
import pytest
from openai import OpenAI


PRISMA_API_KEY = os.environ.get("PRISMA_API_KEY", "")
PRISMA_BASE_URL = os.environ.get("PRISMA_BASE_URL", "")
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")

DATASET_PATH = Path(__file__).parent / "golden_dataset.json"
EVALUATORS = ["correctness", "hallucination"]

LLM_MODEL = "gpt-4o"
LLM_SYSTEM_PROMPT = (
    "You are a compliance documentation assistant for a regulated financial services company. "
    "Answer employee questions using only the retrieved policy documents provided as context. "
    "Always cite the specific policy number. If the answer is not in the provided documents, "
    "say so — do not guess or use general knowledge."
)


@pytest.fixture(scope="module")
def golden_dataset() -> list[dict]:
    """Load the golden test dataset."""
    with open(DATASET_PATH) as f:
        return json.load(f)


@pytest.fixture(scope="module")
def evaluation_results(golden_dataset: list[dict]) -> dict:
    """Generate LLM responses and evaluate them via Prisma."""
    client = OpenAI(api_key=OPENAI_API_KEY)
    records = []

    for item in golden_dataset:
        completion = client.chat.completions.create(
            model=LLM_MODEL,
            messages=[
                {"role": "system", "content": LLM_SYSTEM_PROMPT},
                {"role": "user", "content": item["query"]},
            ],
            max_tokens=256,
        )
        records.append({
            "query": item["query"],
            "response": completion.choices[0].message.content,
            "reference": item["reference"],
        })

    url = f"{PRISMA_BASE_URL}/api/v1/evaluate"
    with httpx.Client(timeout=180) as http:
        response = http.post(
            url,
            json={"records": records, "evaluators": EVALUATORS, "timeout_seconds": 120},
            headers={"x-api-key": PRISMA_API_KEY, "Content-Type": "application/json"},
        )
        response.raise_for_status()
        return response.json()


class TestLLMQuality:
    """Quality gate tests for LLM output."""

    def test_correctness_above_threshold(self, evaluation_results: dict):
        """LLM responses must be at least 90% correct."""
        score = evaluation_results["summary"]["correctness"]
        assert score >= 0.9, (
            f"Correctness score {score:.2f} is below threshold 0.90"
        )

    def test_hallucination_below_threshold(self, evaluation_results: dict):
        """LLM hallucination rate must stay below 10%."""
        score = evaluation_results["summary"]["hallucination"]
        assert score <= 0.1, (
            f"Hallucination score {score:.2f} is above threshold 0.10"
        )
```

Run the tests:

```bash
pytest tests/eval/test_llm_quality.py -v
```

To skip evaluation tests when Prisma credentials are not available (e.g., in local development), add this to your `conftest.py`:

```python
# tests/eval/conftest.py
import os
import pytest


def pytest_collection_modifyitems(config, items):
    """Skip evaluation tests when PRISMA_API_KEY is not set."""
    if not os.environ.get("PRISMA_API_KEY"):
        skip_marker = pytest.mark.skip(reason="PRISMA_API_KEY not set")
        for item in items:
            if "eval" in str(item.fspath):
                item.add_marker(skip_marker)
```

## Expected Output

A passing run looks like this:

```
Loading golden dataset...
  12 test cases loaded
Generating LLM responses...
  12 responses generated
Running Prisma evaluation...
  Evaluation completed in 3421ms
  Scores: {
    "correctness": 0.95,
    "hallucination": 0.03
  }

QUALITY GATE PASSED
```

A failing run exits with code 1 and prints the violations:

```
Loading golden dataset...
  12 test cases loaded
Generating LLM responses...
  12 responses generated
Running Prisma evaluation...
  Evaluation completed in 2876ms
  Scores: {
    "correctness": 0.78,
    "hallucination": 0.22
  }

QUALITY GATE FAILED
  correctness: 0.78 < 0.90 (too low)
  hallucination: 0.22 > 0.10 (too high)
```

## Complete Example

Here is the full file structure for integrating Prisma evaluation into your CI pipeline:

```
your-repo/
├── .github/
│   └── workflows/
│       └── llm-quality-gate.yml     # GitHub Actions workflow
├── tests/
│   └── eval/
│       ├── conftest.py              # Skip logic for local dev
│       ├── eval_gate.py             # Standalone evaluation script
│       ├── golden_dataset.json      # Curated test cases
│       └── test_llm_quality.py      # Pytest integration
└── ...
```

The standalone script (`eval_gate.py`) and the pytest tests (`test_llm_quality.py`) are two ways to achieve the same result. Use whichever fits your CI setup:

- **`eval_gate.py`** is best for simple pipelines where you want a single pass/fail step.
- **`test_llm_quality.py`** is best when you already run pytest in CI and want evaluation results alongside your other tests.

## Next Steps

- [Getting Started](getting-started.md) -- Set up your Prisma AI account and run your first evaluation.
- [Custom Policy Evaluators](custom-policy-evaluators.md) -- Build domain-specific evaluators for your quality gates.
- [Real-Time Production Evaluation](realtime-production-evaluation.md) -- Extend evaluation beyond CI into live production traffic.
