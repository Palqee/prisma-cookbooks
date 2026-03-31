# Policy Compliance in CI/CD

Catch policy regressions before they reach production. This cookbook builds a quality gate that validates a small golden dataset of contact center interactions against a policy metric using Prisma's sync API, then integrates that gate into GitHub Actions and pytest.

The sync API (`POST /api/v1/evaluate`) is designed for small datasets where you need immediate results -- a golden dataset of 10-15 interactions, not a full production corpus. It returns pass/fail synchronously, which makes it ideal for CI/CD pipelines that block on a result.

## What You'll Learn

- Build a golden dataset of realistic contact center interactions with policy-specific scenarios
- Define a `response-compliance` policy metric inline using the 4-prompt pattern
- Call the Prisma sync API for immediate validation results
- Set pass/fail thresholds based on policy metric scores
- Write a validation gate script that loads the dataset, sends it to the sync API, and checks thresholds
- Wire the gate into GitHub Actions and pytest

## Prerequisites

- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)
- **Access to a running Prisma instance**
- **A GitHub repository** (for the Actions integration)
- Familiarity with [Building Policy-Based Metrics](policy-based-metrics.md) -- you should understand the 4-prompt pattern

## 1. Create a Golden Dataset

A golden dataset is a curated set of interactions where you already know the expected outcome. Each record has three fields:

- **`query`** -- The customer side of the interaction.
- **`response`** -- The agent side of the interaction.
- **`reference`** -- The policy-correct response for this scenario, used as a benchmark during validation.

The dataset should cover the range of scenarios your agents handle, with most interactions being compliant and a few deliberately non-compliant. The non-compliant cases are your regression tests -- if a change to your agent tooling causes these to start passing, something is wrong with your metric. If compliant cases start failing, your metric may have become too strict.

Save this as `tests/eval/golden_dataset.json`:

```json
[
  {
    "query": "Hi, I noticed a duplicate charge of $29.99 on my account this month for my premium subscription. Can I get that corrected?",
    "response": "Thank you for calling Meridian Financial, my name is Sarah. I'd be happy to help with that duplicate charge. For security purposes, could you please verify your account number and the date of birth on file? ... Thank you, Ms. Chen, I've confirmed your identity. I can see the duplicate charge of $29.99 posted on March 15th. I've initiated a refund for that amount -- you'll see it back on your card within 3-5 business days. I've also flagged this with our billing team to prevent it from recurring. Is there anything else I can help you with today?",
    "reference": "Duplicate charges should be refunded immediately upon verification. Agent must verify customer identity before accessing account details. Inform the customer the refund will appear within 3-5 business days and provide the exact amount. Flag the issue with the billing team for root cause analysis."
  },
  {
    "query": "I'd like to close my investment account and have all my personal data removed from your systems.",
    "response": "Good afternoon, this is James with Meridian Financial. I'm sorry to hear you'd like to close your account, but I'm here to help. Before I can process that, I'll need to verify your identity -- could you provide your account number and the last four digits of your Social Security number? ... Thank you, Mr. Alvarez, identity confirmed. I've submitted the account closure request effective today. Per our data retention policy, your personal data will be purged from our systems within 30 days, in compliance with applicable regulations. You'll receive a confirmation email within the hour with a summary of this request and a reference number. Is there anything else before we wrap up?",
    "reference": "Account closure is effective immediately upon request. Agent must verify customer identity before processing. Inform the customer that personal data deletion occurs within 30 days per the data retention policy (GDPR Article 17 / CCPA). Send a confirmation email with a reference number."
  },
  {
    "query": "I'm calling because I was charged a $35 overdraft fee but I had set up overdraft protection. Can you explain why I was still charged?",
    "response": "Thank you for reaching out to Meridian Financial, this is Priya speaking. I understand the frustration -- let me look into this for you. Could you please confirm your account number and the last four digits of your Social Security number? ... Thank you, Mr. Park. I can see your overdraft protection is linked to your savings account, but the savings balance was $0 at the time of the transaction, which is why the fee was applied. I've submitted a one-time courtesy waiver for the $35 fee -- that will be reflected on your account within 1-2 business days. Going forward, I'd recommend keeping a small buffer in your linked savings account. Would you like me to set up a low-balance alert for that account?",
    "reference": "If overdraft protection is linked but the backup account had insufficient funds, explain why the fee was applied. Agent may offer a one-time courtesy waiver per Policy FIN-118. Suggest preventive measures such as low-balance alerts. Verify identity before accessing account information."
  },
  {
    "query": "I want to update my mailing address. I just moved to 742 Evergreen Terrace, Springfield.",
    "response": "Welcome to Meridian Financial, my name is David. I'd be happy to help you update your address. For security, could you please verify your account number and date of birth? ... Perfect, thank you, Ms. Rivera. I've updated your mailing address to 742 Evergreen Terrace, Springfield. This will apply to all future statements and correspondence. Please allow one billing cycle for the change to take full effect. Is there anything else I can assist you with today?",
    "reference": "Address changes require identity verification before processing. Confirm the new address back to the customer. Inform them the change applies to all correspondence and may take one billing cycle to reflect on statements."
  },
  {
    "query": "I lost my debit card and I need it replaced immediately. I'm traveling and have no other way to access my funds.",
    "response": "Thank you for calling Meridian Financial, this is Angela. I'm sorry to hear about your lost card -- I understand how stressful that can be, especially while traveling. Let me help you right away. Could you verify your account number and the last four digits of your Social Security number? ... Thank you, Mr. Okafor. I've deactivated your lost card to prevent unauthorized use and ordered a rush replacement. The new card will be shipped overnight to the address on file. In the meantime, I've enabled temporary mobile wallet access so you can make purchases using your phone. You'll receive a text with activation instructions shortly. Is there anything else I can help with?",
    "reference": "Lost card reports require immediate deactivation of the compromised card after identity verification. Rush replacement should be offered to traveling customers per Policy FIN-205. If available, enable temporary mobile wallet access as an interim solution. Confirm the shipping address for the replacement."
  },
  {
    "query": "I'm disputing a charge of $189.50 from a merchant I don't recognize on my latest statement.",
    "response": "Good morning, this is Marcus with Meridian Financial. I'm sorry to hear about the unfamiliar charge. I'll help you get this resolved. Could you verify your account number and your date of birth? ... Thank you, Ms. Tanaka. I can see the $189.50 charge from 'GreenLeaf Services' posted on March 22nd. I've opened a formal dispute case -- reference number DP-2026-4471. You'll receive provisional credit for $189.50 within 2 business days while we investigate. The investigation typically takes 30-60 days. We'll contact you if we need additional information. In the meantime, I've flagged your card for enhanced monitoring. Is there anything else I can help with?",
    "reference": "Charge disputes require identity verification before proceeding. Open a formal dispute case and provide the customer with a reference number. Issue provisional credit within the timeframe specified by Regulation E (10 business days, though Meridian policy is 2 business days). Inform the customer of the investigation timeline (30-60 days). Offer enhanced card monitoring."
  },
  {
    "query": "Can you tell me about your current CD rates? I'm thinking about putting $50,000 into a certificate of deposit.",
    "response": "Thank you for calling Meridian Financial, this is Laura. Great question about our CD rates -- I'd love to help you explore your options. Our current rates for a $50,000 deposit are: 12-month CD at 4.75% APY, 24-month at 5.10% APY, and 36-month at 5.25% APY. These rates are guaranteed for the term of the CD. Keep in mind that early withdrawal penalties apply -- for a 12-month CD, that's 90 days of interest. I'd recommend speaking with one of our investment advisors who can help you decide which term best fits your financial goals. Would you like me to schedule a consultation?",
    "reference": "Provide current CD rates with APY for the relevant deposit amount. Disclose early withdrawal penalties per Truth in Savings Act requirements. Recommend consultation with an investment advisor for deposits over $25,000 per Policy FIN-310. Do not guarantee future rates beyond the stated term."
  },
  {
    "query": "I'm having trouble logging into my online banking. I've tried resetting my password but the link never arrives.",
    "response": "Welcome to Meridian Financial, this is Raj. I understand how frustrating login issues can be -- let me help you get back in. For security, I'll need to verify your identity first. Could you provide your account number and date of birth? ... Thank you, Ms. Kim. I can see that your password reset emails are being sent to an outdated email address ending in @oldprovider.com. I've updated it to the email address you confirmed and triggered a new password reset. You should receive it within a few minutes. If it doesn't arrive, check your spam folder. Is there anything else I can help you with?",
    "reference": "Password reset issues require identity verification before accessing account settings. Check the email address on file for accuracy. If outdated, update it with the customer's confirmed new address. Trigger a new password reset and advise the customer to check spam. Do not share the full email address on file -- reference only a partial version."
  },
  {
    "query": "I want to set up automatic bill payments from my checking account. Can you walk me through that?",
    "response": "Thank you for calling Meridian Financial, my name is Chen. I'd be happy to help you set up automatic bill payments. Before we get started, could you please verify your account number and the last four digits of your Social Security number? ... Perfect, thank you, Mr. Brooks. I can set up recurring payments from your checking account. I'll need the payee name, their account number or payment address, the amount, and the frequency -- weekly, biweekly, or monthly. Once it's set up, you'll receive a confirmation email with the details and you can modify or cancel any time through online banking. Shall we go ahead and set up the first one?",
    "reference": "Automatic payment setup requires identity verification. Collect payee details, payment amount, and frequency. Confirm setup via email. Inform the customer they can manage auto-payments through online banking. Ensure sufficient funds warnings are communicated per Policy FIN-112."
  },
  {
    "query": "I was told I'd receive a callback from a manager about a complaint I filed last week, but nobody has called me. This is unacceptable.",
    "response": "Thank you for calling Meridian Financial, this is Diane. I sincerely apologize for the missed callback -- I understand how frustrating that must be, and you're right to expect better follow-through from us. Let me verify your identity so I can pull up your complaint. Could you provide your account number and date of birth? ... Thank you, Mr. Yamamoto. I can see your complaint filed on March 23rd -- reference CR-2026-0891. The assigned manager is Lisa Torres. I've escalated this internally and you will receive a callback from Ms. Torres or her designee within 4 business hours. I've also added a note to your case documenting the missed callback. If you haven't heard back by end of day, please call us and reference your case number. Is there anything else I can do for you right now?",
    "reference": "Missed callback complaints should be acknowledged with a sincere apology. Verify identity before accessing complaint records. Provide the complaint reference number and assigned manager name. Re-escalate with a guaranteed callback window per Policy SVC-201 (4 business hours). Document the service failure in the case notes."
  },
  {
    "query": "I need to transfer my remaining balance of $4,200 from my savings account to an external account. Can you do that now?",
    "response": "Sure, I can process that transfer for you right away. What's the routing number and account number for the destination account? I'll get that $4,200 moved over as soon as I have those details.",
    "reference": "Outbound transfers require identity verification of the account holder before proceeding. Agent must request at least two forms of identification (account number plus date of birth, or last four SSN digits plus mailing address). Do not initiate or discuss transfer details until KYC verification is complete, per Policy FIN-302. After verification, confirm the transfer amount, destination, and expected settlement time. Close with a summary and reference number."
  },
  {
    "query": "My business partner and I want to open a joint business checking account. What do we need?",
    "response": "Hi there! So for a joint business account you just need to bring in your IDs and we'll get it set up. It's pretty straightforward. You can come into any branch.",
    "reference": "Joint business checking accounts require: valid government-issued photo ID for all signatories, EIN or tax identification number, articles of incorporation or partnership agreement, and a completed beneficial ownership form per FinCEN requirements. Explain the documentation requirements clearly and offer to email a checklist. Schedule a branch appointment with a business banking specialist per Policy BUS-101."
  }
]
```

Twelve records covering the key scenarios:

| Records 1-10 | Compliant | Standard scenarios: refunds, account closures, disputes, address changes, lost cards, CD inquiries, login issues, bill pay, complaint escalation |
|---|---|---|
| Record 11 | **Non-compliant** | Skips identity verification before processing a $4,200 transfer |
| Record 12 | **Non-compliant** | Vague, incomplete response that omits required documentation details and regulatory requirements |

The mix matters. If your gate expects a 90% pass rate, 10 out of 12 passing gives you ~83% -- just below the threshold. This is intentional for testing. In production, you would adjust the threshold to match your team's actual compliance baseline, or adjust the dataset to include the ratio of compliant-to-non-compliant interactions that matches your expected pass rate.

## 2. Define the Policy Metric

Define the `response-compliance` policy metric inline as part of the validation payload. This is the same 4-prompt pattern used throughout these cookbooks, but instead of registering it through the SDK, you pass it directly to the sync API.

```python
# eval_config.py

RESPONSE_COMPLIANCE = {
    "name": "response-compliance",
    "type": "custom",
    "options": ["compliant", "non_compliant"],
    "evaluation_system_prompt": (
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
    "evaluation_human_prompt": (
        "Validate the following contact center interaction against company policy.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain which of the four policy criteria "
        "(greeting, identity verification, accurate information, professional close) "
        "were met or violated, then state your classification."
    ),
    "reeval_system_prompt": (
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the company's agent response policy strictly. "
        "Pay special attention to identity verification -- this is the most critical "
        "step in financial services interactions. An agent who skips identity "
        "verification before accessing or modifying account information is always "
        "non_compliant, regardless of how well they performed on other criteria."
    ),
    "reeval_human_prompt": (
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
}
```

The 4-prompt pattern:

1. **`evaluation_system_prompt`** -- Sets the validator's role and the specific policy criteria.
2. **`evaluation_human_prompt`** -- Presents the interaction with `{query}`, `{response}`, `{reference}`, and `{evaluation_options}` placeholders.
3. **`reeval_system_prompt`** -- Stricter guidance for ambiguous cases on the second pass.
4. **`reeval_human_prompt`** -- Includes `{memories}` from past human reviews for calibration.

For a deeper explanation of each prompt and how to write effective metrics, see [Building Policy-Based Metrics](policy-based-metrics.md).

## 3. Write the Validation Gate Script

This script loads the golden dataset, sends it to the Prisma sync API, and checks the results against your policy metric thresholds.

Save this as `tests/eval/gate.py`:

```python
"""Policy compliance validation gate for CI/CD pipelines."""

import json
import os
import sys

import httpx

from eval_config import RESPONSE_COMPLIANCE

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]

# --- Thresholds ---
# response-compliance pass rate must be at least 90%.
# "Pass" means the metric returned "compliant" for the interaction.
PASS_RATE_THRESHOLD = 0.90


def load_golden_dataset(path: str) -> list[dict]:
    """Load the golden dataset from a JSON file."""
    with open(path) as f:
        return json.load(f)


def build_evaluator_config() -> dict:
    """Build the evaluator configuration for the sync API."""
    return {
        "name": RESPONSE_COMPLIANCE["name"],
        "type": RESPONSE_COMPLIANCE["type"],
        "options": RESPONSE_COMPLIANCE["options"],
        "evaluation_system_prompt": RESPONSE_COMPLIANCE["evaluation_system_prompt"],
        "evaluation_human_prompt": RESPONSE_COMPLIANCE["evaluation_human_prompt"],
        "reeval_system_prompt": RESPONSE_COMPLIANCE["reeval_system_prompt"],
        "reeval_human_prompt": RESPONSE_COMPLIANCE["reeval_human_prompt"],
    }


def run_validation(records: list[dict], evaluator_config: dict) -> dict:
    """Send records to the Prisma sync API and return the result."""
    url = f"{PRISMA_BASE_URL}/api/v1/evaluate"
    payload = {
        "records": records,
        "evaluators": [evaluator_config],
        "timeout_seconds": 120,
    }
    response = httpx.post(
        url,
        json=payload,
        headers={
            "x-api-key": PRISMA_API_KEY,
            "Content-Type": "application/json",
        },
        timeout=180.0,
    )
    response.raise_for_status()
    return response.json()


def check_thresholds(result: dict) -> bool:
    """Check whether policy metric results meet the defined thresholds.

    Returns True if all thresholds pass, False otherwise.
    """
    summary = result["summary"]
    metric_name = "response-compliance"

    if metric_name not in summary:
        print(f"FAIL: metric '{metric_name}' not found in results")
        return False

    metric_summary = summary[metric_name]
    pass_rate = metric_summary.get("compliant", 0)
    total = sum(metric_summary.values())

    if total == 0:
        print(f"FAIL: no results for metric '{metric_name}'")
        return False

    actual_pass_rate = pass_rate / total

    print(f"\n{'='*60}")
    print(f"Policy Metric: {metric_name}")
    print(f"  Compliant:     {pass_rate}/{total}")
    print(f"  Pass rate:     {actual_pass_rate:.1%}")
    print(f"  Threshold:     {PASS_RATE_THRESHOLD:.1%}")
    print(f"  Result:        {'PASS' if actual_pass_rate >= PASS_RATE_THRESHOLD else 'FAIL'}")
    print(f"{'='*60}")

    if actual_pass_rate < PASS_RATE_THRESHOLD:
        print(f"\nFAIL: {metric_name} pass rate {actual_pass_rate:.1%} "
              f"is below threshold {PASS_RATE_THRESHOLD:.1%}")
        return False

    return True


def print_per_record_detail(result: dict) -> None:
    """Print per-record validation results for debugging."""
    records = result.get("results", [])
    print(f"\nPer-record detail ({len(records)} records):")
    print(f"{'-'*60}")
    for i, record in enumerate(records):
        query_preview = record.get("query", "")[:60]
        metrics = record.get("metrics", {})
        compliance = metrics.get("response-compliance", "N/A")
        status = "PASS" if compliance == "compliant" else "FAIL"
        print(f"  [{status}] Record {i+1}: {query_preview}...")
        print(f"         response-compliance: {compliance}")
    print(f"{'-'*60}")


def main() -> int:
    """Run the validation gate. Returns 0 on success, 1 on failure."""
    dataset_path = os.environ.get(
        "GOLDEN_DATASET_PATH",
        "tests/eval/golden_dataset.json",
    )

    print(f"Loading golden dataset from {dataset_path}")
    records = load_golden_dataset(dataset_path)
    print(f"Loaded {len(records)} records")

    evaluator_config = build_evaluator_config()
    print(f"Running validation with metric: {evaluator_config['name']}")

    result = run_validation(records, evaluator_config)

    print_per_record_detail(result)
    passed = check_thresholds(result)

    return 0 if passed else 1


if __name__ == "__main__":
    sys.exit(main())
```

The script does four things:

1. **Loads the golden dataset** from a JSON file.
2. **Sends all records to the sync API** in a single request. The sync API validates every record against the policy metric and returns results synchronously -- no polling, no callbacks.
3. **Prints per-record detail** so you can see exactly which interactions passed and which failed.
4. **Checks the pass rate** against your threshold and exits with code 0 (pass) or 1 (fail).

## 4. Define Quality Thresholds

The threshold in the gate script is based on the policy metric, not on generic scores:

```python
# response-compliance pass rate must be at least 90%.
PASS_RATE_THRESHOLD = 0.90
```

This means: at least 90% of the golden dataset interactions must be classified as `compliant` by the `response-compliance` policy metric.

### Choosing a threshold

Start with your golden dataset composition. If you have 12 interactions and 2 are deliberately non-compliant, then 10/12 = ~83% is the theoretical maximum if the non-compliant cases are correctly caught. In that case, setting the threshold to 0.80 means "all compliant interactions must pass." Setting it higher (e.g., 0.90) means you expect even the borderline cases to pass.

In practice:

| Threshold | What it means |
|---|---|
| 0.80 | All interactions you expect to pass do pass. Non-compliant cases are correctly caught. |
| 0.90 | Tight tolerance -- leaves room for only 1 unexpected failure in a 12-record dataset. |
| 1.00 | Zero tolerance -- every single interaction must be classified as compliant. Only use this if your golden dataset contains no deliberately non-compliant records. |

Adjust the threshold as your golden dataset evolves. When you add new scenarios, re-calibrate.

## 5. GitHub Actions Integration

Add the validation gate as a step in your CI/CD pipeline. This runs on every pull request that touches agent-facing configuration, prompts, or tools.

Save this as `.github/workflows/policy-gate.yml`:

```yaml
name: Policy Compliance Gate

on:
  pull_request:
    paths:
      - "prompts/**"
      - "tools/**"
      - "config/**"
      - "tests/eval/**"

jobs:
  policy-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install httpx

      - name: Run policy compliance gate
        env:
          PRISMA_API_KEY: ${{ secrets.PRISMA_API_KEY }}
          PRISMA_BASE_URL: ${{ secrets.PRISMA_BASE_URL }}
          GOLDEN_DATASET_PATH: tests/eval/golden_dataset.json
        run: python tests/eval/gate.py
```

Key design decisions:

- **`paths` filter** -- The gate only runs when files that affect agent behavior change. No point re-validating if someone updates the README.
- **`GOLDEN_DATASET_PATH`** -- Set via environment variable so you can override it for different test suites.
- **Secrets** -- `PRISMA_API_KEY` and `PRISMA_BASE_URL` are stored as GitHub repository secrets. Never commit API keys.
- **Exit code** -- The gate script exits with code 1 on failure, which causes the GitHub Actions step to fail and blocks the PR.

## 6. Pytest Integration

If your team uses pytest, wrap the validation gate as a test. This lets you run policy validation alongside your unit tests and see failures in the same test report.

Save this as `tests/eval/test_policy_compliance.py`:

```python
"""Pytest integration for policy compliance validation."""

import json
import os

import httpx
import pytest

from eval_config import RESPONSE_COMPLIANCE

PRISMA_BASE_URL = os.environ["PRISMA_BASE_URL"]
PRISMA_API_KEY = os.environ["PRISMA_API_KEY"]
PASS_RATE_THRESHOLD = 0.90


@pytest.fixture
def golden_dataset():
    """Load the golden dataset."""
    dataset_path = os.environ.get(
        "GOLDEN_DATASET_PATH",
        "tests/eval/golden_dataset.json",
    )
    with open(dataset_path) as f:
        return json.load(f)


@pytest.fixture
def evaluator_config():
    """Build the evaluator configuration."""
    return {
        "name": RESPONSE_COMPLIANCE["name"],
        "type": RESPONSE_COMPLIANCE["type"],
        "options": RESPONSE_COMPLIANCE["options"],
        "evaluation_system_prompt": RESPONSE_COMPLIANCE["evaluation_system_prompt"],
        "evaluation_human_prompt": RESPONSE_COMPLIANCE["evaluation_human_prompt"],
        "reeval_system_prompt": RESPONSE_COMPLIANCE["reeval_system_prompt"],
        "reeval_human_prompt": RESPONSE_COMPLIANCE["reeval_human_prompt"],
    }


def test_policy_compliance_gate(golden_dataset, evaluator_config):
    """Validate that golden dataset interactions meet policy compliance thresholds."""
    url = f"{PRISMA_BASE_URL}/api/v1/evaluate"
    payload = {
        "records": golden_dataset,
        "evaluators": [evaluator_config],
        "timeout_seconds": 120,
    }
    response = httpx.post(
        url,
        json=payload,
        headers={
            "x-api-key": PRISMA_API_KEY,
            "Content-Type": "application/json",
        },
        timeout=180.0,
    )
    response.raise_for_status()
    result = response.json()

    # Check per-record results
    records = result.get("results", [])
    for i, record in enumerate(records):
        metrics = record.get("metrics", {})
        compliance = metrics.get("response-compliance", "N/A")
        query_preview = record.get("query", "")[:60]
        print(f"  Record {i+1} [{compliance}]: {query_preview}...")

    # Check threshold
    summary = result["summary"]
    metric_summary = summary["response-compliance"]
    pass_count = metric_summary.get("compliant", 0)
    total = sum(metric_summary.values())
    actual_pass_rate = pass_count / total if total > 0 else 0

    assert actual_pass_rate >= PASS_RATE_THRESHOLD, (
        f"response-compliance pass rate {actual_pass_rate:.1%} "
        f"is below threshold {PASS_RATE_THRESHOLD:.1%} "
        f"({pass_count}/{total} compliant)"
    )
```

Run it:

```bash
pytest tests/eval/test_policy_compliance.py -v
```

The pytest version has the same logic as the standalone gate script but integrates with pytest's assertion and reporting system. Failed thresholds produce clear assertion errors with the actual vs. expected pass rate.

## Expected Output

### Passing run

When all compliant interactions pass and non-compliant ones are correctly caught (and you have set your threshold to match):

```
Loading golden dataset from tests/eval/golden_dataset.json
Loaded 12 records
Running validation with metric: response-compliance

Per-record detail (12 records):
------------------------------------------------------------
  [PASS] Record 1:  Hi, I noticed a duplicate charge of $29.99 on my accoun...
         response-compliance: compliant
  [PASS] Record 2:  I'd like to close my investment account and have all my...
         response-compliance: compliant
  [PASS] Record 3:  I'm calling because I was charged a $35 overdraft fee b...
         response-compliance: compliant
  [PASS] Record 4:  I want to update my mailing address. I just moved to 74...
         response-compliance: compliant
  [PASS] Record 5:  I lost my debit card and I need it replaced immediately....
         response-compliance: compliant
  [PASS] Record 6:  I'm disputing a charge of $189.50 from a merchant I don...
         response-compliance: compliant
  [PASS] Record 7:  Can you tell me about your current CD rates? I'm thinki...
         response-compliance: compliant
  [PASS] Record 8:  I'm having trouble logging into my online banking. I've...
         response-compliance: compliant
  [PASS] Record 9:  I want to set up automatic bill payments from my checki...
         response-compliance: compliant
  [PASS] Record 10: I was told I'd receive a callback from a manager about a...
         response-compliance: compliant
  [FAIL] Record 11: I need to transfer my remaining balance of $4,200 from m...
         response-compliance: non_compliant
  [FAIL] Record 12: My business partner and I want to open a joint business ...
         response-compliance: non_compliant
------------------------------------------------------------

============================================================
Policy Metric: response-compliance
  Compliant:     10/12
  Pass rate:     83.3%
  Threshold:     80.0%
  Result:        PASS
============================================================
```

### Failing run

When a code change causes a previously compliant interaction to fail -- for example, a prompt update that makes the agent skip the professional close:

```
Per-record detail (12 records):
------------------------------------------------------------
  [PASS] Record 1:  Hi, I noticed a duplicate charge of $29.99 on my accoun...
         response-compliance: compliant
  [PASS] Record 2:  I'd like to close my investment account and have all my...
         response-compliance: compliant
  [FAIL] Record 3:  I'm calling because I was charged a $35 overdraft fee b...
         response-compliance: non_compliant
  [PASS] Record 4:  I want to update my mailing address. I just moved to 74...
         response-compliance: compliant
  [FAIL] Record 5:  I lost my debit card and I need it replaced immediately....
         response-compliance: non_compliant
  [PASS] Record 6:  I'm disputing a charge of $189.50 from a merchant I don...
         response-compliance: compliant
  [PASS] Record 7:  Can you tell me about your current CD rates? I'm thinki...
         response-compliance: compliant
  [PASS] Record 8:  I'm having trouble logging into my online banking. I've...
         response-compliance: compliant
  [PASS] Record 9:  I want to set up automatic bill payments from my checki...
         response-compliance: compliant
  [PASS] Record 10: I was told I'd receive a callback from a manager about a...
         response-compliance: compliant
  [FAIL] Record 11: I need to transfer my remaining balance of $4,200 from m...
         response-compliance: non_compliant
  [FAIL] Record 12: My business partner and I want to open a joint business ...
         response-compliance: non_compliant
------------------------------------------------------------

============================================================
Policy Metric: response-compliance
  Compliant:     8/12
  Pass rate:     66.7%
  Threshold:     80.0%
  Result:        FAIL
============================================================

FAIL: response-compliance pass rate 66.7% is below threshold 80.0%
```

Records 3 and 5 regressed -- they were compliant before the code change but are now flagged as `non_compliant`. The per-record detail tells you exactly which interactions broke, so you can trace the regression to the specific change.

## Complete Example

File structure for a project with the policy compliance gate:

```
your-project/
├── .github/
│   └── workflows/
│       └── policy-gate.yml          # GitHub Actions workflow
├── prompts/
│   └── agent_system_prompt.txt      # Agent-facing prompts (changes trigger the gate)
├── tools/
│   └── ...                          # Agent tools (changes trigger the gate)
├── tests/
│   └── eval/
│       ├── eval_config.py           # Policy metric definition
│       ├── gate.py                  # Standalone validation gate script
│       ├── golden_dataset.json      # Golden dataset (~12 records)
│       └── test_policy_compliance.py  # Pytest integration
└── config/
    └── ...                          # Agent configuration (changes trigger the gate)
```

The gate runs whenever someone modifies prompts, tools, or configuration -- the files that directly affect how your agents respond. The golden dataset acts as a regression test suite: if a change causes previously compliant interactions to fail the policy metric, the pipeline blocks the merge.

## Next Steps

This cookbook used a single policy metric (`response-compliance`) as the quality gate. Prisma also supports traditional AI performance metrics -- pre-built validators for common quality dimensions that you can add alongside your policy metrics.

Next: [AI Performance Metrics](ai-performance-metrics.md)
