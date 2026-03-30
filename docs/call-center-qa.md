# Automated QA for Call Center Transcripts

Evaluate call center agent responses at scale for correctness, hallucination, and policy compliance using Prisma AI's batch evaluation pipeline.

## What You'll Learn

- Build a batch evaluation pipeline for call center transcript data
- Combine built-in evaluators (correctness, hallucination) with a custom policy adherence evaluator
- Map CSV dataset columns to evaluation fields using `InputMapping`
- Interpret evaluation results to identify compliance gaps across your contact center

## Prerequisites

- Python 3.10+
- A Prisma AI API key (from your Prisma instance settings)
- Access to a running Prisma instance
- Call center transcript data in CSV format (or use the sample dataset below)

## 1. Install Dependencies

```bash
pip install prisma-ai pq-prisma-client pandas
```

The `pq-prisma-client` package provides the async `PrismaClient` for interacting with the Prisma platform API. `pandas` handles CSV loading and preparation.

## 2. Configure Environment Variables

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Prepare the Transcript Dataset

Call center transcripts typically include the customer query, the agent's response, an expected (ideal) response, and a reference to the governing policy. Create a file called `call_center_qa.py` and start with a sample dataset:

```python
import pandas as pd

# Sample call center transcript dataset
data = [
    {
        "call_id": "CALL-001",
        "customer_query": "I'd like to know the current interest rate on a 30-year fixed mortgage.",
        "agent_response": "Our current 30-year fixed mortgage rate is 6.75% APR for qualified borrowers. This rate is subject to change and your actual rate depends on your credit profile and down payment amount.",
        "expected_response": "The current 30-year fixed mortgage rate is 6.75% APR. Rates vary based on creditworthiness, down payment, and loan amount. Refer the customer to a loan officer for a personalized quote.",
        "policy_reference": "Mortgage Disclosure Policy §4.2: Agents must state the current advertised rate, clarify that rates are subject to change, and recommend a loan officer consultation.",
    },
    {
        "call_id": "CALL-002",
        "customer_query": "Why was I charged a $35 overdraft fee? I thought I had enough in my account.",
        "agent_response": "I see the overdraft occurred on March 15th when a $200 debit posted before your direct deposit cleared. I can submit a one-time courtesy reversal for you since this is your first overdraft in 12 months.",
        "expected_response": "Explain the transaction sequence that caused the overdraft, offer a courtesy fee reversal if eligible per the fee waiver matrix, and suggest overdraft protection enrollment.",
        "policy_reference": "Fee Dispute Policy §2.1: First overdraft in 12 months qualifies for automatic courtesy reversal. Agent must explain the cause and offer overdraft protection.",
    },
    {
        "call_id": "CALL-003",
        "customer_query": "I need to dispute a charge of $450 on my credit card from a merchant I don't recognize.",
        "agent_response": "I'll open a dispute for that $450 charge right away. You'll receive a provisional credit within 24 hours while we investigate. The investigation takes up to 90 days under Regulation E.",
        "expected_response": "Initiate a fraud dispute, issue provisional credit within 10 business days per Regulation Z (credit cards), confirm the merchant details, and advise the customer to monitor their account for additional unauthorized charges.",
        "policy_reference": "Fraud Dispute Policy §3.4: Credit card disputes fall under Regulation Z, not Regulation E. Provisional credit timeline is 10 business days. Agent must verify merchant details before filing.",
    },
    {
        "call_id": "CALL-004",
        "customer_query": "Can I get a copy of my account statements from the last two years for my tax preparer?",
        "agent_response": "Sure, I can mail those to the address on file. There's a $5 fee per statement for anything older than 6 months. Would you like me to proceed?",
        "expected_response": "Inform the customer that statements from the last 7 years are available free of charge through online banking. Paper copies older than 6 months incur a $5 per statement fee. Offer to walk them through the online download process first.",
        "policy_reference": "Document Request Policy §1.3: Always offer the free digital option before charging for paper copies. Statements are available online for 7 years at no cost.",
    },
    {
        "call_id": "CALL-005",
        "customer_query": "I want to close my savings account and transfer the balance to my checking account.",
        "agent_response": "I can process that for you right now. Your savings balance of $12,340 will be transferred to your checking account ending in 4521. The account will be closed immediately.",
        "expected_response": "Confirm the customer's identity with two-factor verification before processing. Disclose any early closure penalties if the account is under 6 months old. Confirm the transfer details and provide a confirmation number.",
        "policy_reference": "Account Closure Policy §5.1: Identity verification required for closure requests. Disclose early closure fees for accounts under 6 months. Provide written confirmation within 2 business days.",
    },
    {
        "call_id": "CALL-006",
        "customer_query": "My debit card was stolen. I need to cancel it and get a new one.",
        "agent_response": "I've immediately blocked your debit card ending in 7892. A replacement card will arrive in 5-7 business days. Would you like an expedited card for a $25 fee? In the meantime, you can use our mobile wallet for contactless payments.",
        "expected_response": "Block the card immediately, initiate a replacement, offer expedited shipping, document the theft for fraud monitoring, advise the customer to review recent transactions for unauthorized charges, and file a fraud alert.",
        "policy_reference": "Lost/Stolen Card Policy §6.2: Immediate block required. Agent must review last 48 hours of transactions with the customer and initiate fraud monitoring. Expedited replacement at $25 is optional.",
    },
    {
        "call_id": "CALL-007",
        "customer_query": "I'm having trouble making my loan payments. Is there any hardship program available?",
        "agent_response": "I understand this is a difficult situation. We do offer a hardship program that can reduce your payments by up to 50% for six months. I can transfer you to our loss mitigation team to discuss your options.",
        "expected_response": "Express empathy, confirm the hardship program exists, explain that options include payment deferral, reduced payments, or loan modification depending on circumstances. Transfer to loss mitigation for formal assessment. Do not promise specific terms.",
        "policy_reference": "Hardship Assistance Policy §8.1: Agents must not promise specific relief terms. Only the loss mitigation team can approve hardship modifications after reviewing the borrower's financial documentation.",
    },
    {
        "call_id": "CALL-008",
        "customer_query": "Can you tell me my account balance and recent transactions over the phone?",
        "agent_response": "Your checking account balance is $3,250.00. Your last three transactions were: $45 at Grocery Mart, $120 at Gas Station, and $89 at Online Retailer.",
        "expected_response": "Before disclosing any account information, verify the customer's identity using the standard authentication process (security questions or one-time passcode). Then provide the requested information.",
        "policy_reference": "Information Security Policy §10.1: Account information must never be disclosed without completing identity verification. Minimum two-factor authentication required for phone inquiries.",
    },
    {
        "call_id": "CALL-009",
        "customer_query": "I want to wire $50,000 to an international account in the Cayman Islands for a real estate investment.",
        "agent_response": "I can set up that wire transfer for you. The fee for international wires is $45. I'll just need the recipient's bank details and the transfer should complete in 2-3 business days.",
        "expected_response": "For international wire transfers over $10,000, complete enhanced due diligence including the purpose of the transfer, recipient relationship verification, and OFAC screening. Flag for BSA/AML review if the transaction meets suspicious activity indicators.",
        "policy_reference": "BSA/AML Policy §12.3: International transfers over $10,000 require enhanced due diligence, OFAC screening, and documentation of the purpose and recipient relationship. Agents must file a CTR and escalate if suspicious activity indicators are present.",
    },
    {
        "call_id": "CALL-010",
        "customer_query": "I noticed my CD matured last week. What are my options for reinvestment?",
        "agent_response": "Your 12-month CD matured with a balance of $25,000. You're currently in the 10-day grace period. Your options are: renew at the current 12-month rate of 4.50% APY, choose a different term, or withdraw the funds. If you don't act within the grace period, it auto-renews at the current rate.",
        "expected_response": "Confirm CD maturity details and grace period timeline. Present all reinvestment options including current rates for various terms. Mention the auto-renewal policy and any rate differences from the original term. Offer to connect with an investment advisor for larger balances.",
        "policy_reference": "CD Maturity Policy §7.1: Agents must disclose all available terms and current rates during the grace period. For balances over $10,000, offer a consultation with an investment advisor.",
    },
    {
        "call_id": "CALL-011",
        "customer_query": "I need to add my spouse as a joint account holder on my checking account.",
        "agent_response": "I can start that process. Your spouse will need to visit a branch with two forms of government-issued ID and sign the joint account agreement. There's no fee for adding a joint holder.",
        "expected_response": "Explain that adding a joint holder requires both parties to sign the joint account agreement. This can be done in-branch or via a notarized mail-in form. Only one government-issued ID is required per person. Disclose that both parties will have full access and liability.",
        "policy_reference": "Joint Account Policy §9.2: One government-issued ID per person required, not two. Both in-branch and notarized mail-in options must be offered. Agent must disclose shared liability.",
    },
    {
        "call_id": "CALL-012",
        "customer_query": "I'm calling about the promotional offer I received for a new credit card with 0% APR.",
        "agent_response": "I can see the offer on your profile. It's our Platinum Rewards card with 0% APR for the first 18 months on purchases and balance transfers. After the promotional period, the rate goes to our standard variable rate. Would you like to apply now?",
        "expected_response": "Confirm the specific offer details including the promotional period, standard APR after promotion, annual fee, balance transfer fees, and any spending requirements. Provide the Schumer Box disclosures verbally or direct the customer to the written terms.",
        "policy_reference": "Credit Card Marketing Policy §11.1: Agents must disclose the post-promotional APR, annual fee, and balance transfer fee (typically 3-5%) before processing an application. TILA Schumer Box disclosures are mandatory.",
    },
]

df = pd.DataFrame(data)
df.to_csv("call_center_transcripts.csv", index=False)
print(f"Dataset created: {len(df)} transcripts")
```

In production, you would load your actual transcript export:

```python
df = pd.read_csv("path/to/your/transcripts.csv")
```

## 4. Define the Custom Policy Adherence Evaluator

Built-in evaluators handle correctness and hallucination, but call center QA also requires checking whether agents followed company policy. Create a custom evaluator with domain-specific prompts:

```python
from prisma_ai import Evaluators

policy_adherence = Evaluators.custom(
    name="policy_adherence",
    options=["compliant", "non_compliant", "partial"],
    evaluation_system_prompt=(
        "You are a compliance auditor evaluating call center agent responses "
        "for a financial services company. Your role is to determine whether "
        "the agent's response adheres to the company's internal policies and "
        "regulatory requirements.\n\n"
        "Evaluate based on these criteria:\n"
        "1. Did the agent follow the correct procedure as outlined in the policy reference?\n"
        "2. Did the agent provide all required disclosures?\n"
        "3. Did the agent avoid making unauthorized promises or commitments?\n"
        "4. Did the agent comply with applicable regulations (TILA, Reg E, Reg Z, BSA/AML)?\n\n"
        "Classify the response as:\n"
        "- compliant: The agent fully followed policy and regulatory requirements\n"
        "- partial: The agent followed some but not all policy requirements\n"
        "- non_compliant: The agent violated policy or missed critical regulatory requirements"
    ),
    evaluation_human_prompt=(
        "Customer query: {query}\n\n"
        "Agent response: {response}\n\n"
        "Evaluate the agent's response for policy adherence.\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Provide your classification and a brief explanation of which policy "
        "requirements were met or missed."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor performing a second-level review of a "
        "call center agent response evaluation. You have access to the original "
        "evaluation and additional context from the policy reference and expected "
        "response. Re-analyze the agent's response with this additional context "
        "to confirm or revise the initial classification.\n\n"
        "Pay special attention to:\n"
        "- Regulatory requirements that may have been overlooked\n"
        "- Subtle policy violations (e.g., missing disclosures, incorrect procedures)\n"
        "- Whether partial compliance should be reclassified based on the severity of gaps"
    ),
    reeval_human_prompt=(
        "Customer query: {query}\n\n"
        "Agent response: {response}\n\n"
        "Previous evaluation context: {memories}\n\n"
        "Re-evaluate the agent's response considering the additional context above. "
        "Confirm or revise the policy adherence classification."
    ),
)
```

## 5. Configure the Evaluation Run

Combine the built-in and custom evaluators into a single evaluation configuration, and map your CSV columns to the fields that each evaluator expects:

```python
from prisma_ai import EvaluationConfig, InputMapping

evaluation_config = EvaluationConfig(
    evaluators=[
        Evaluators.correctness(),
        Evaluators.hallucination(),
        policy_adherence,
    ],
    input_mapping=InputMapping(
        query="customer_query",
        response="agent_response",
        reference="expected_response",
    ),
)
```

The `InputMapping` tells Prisma which columns in your dataset correspond to each evaluation field:

- `query` -- the customer's question or request (maps to `{query}` in prompts)
- `response` -- the agent's actual response (maps to `{response}` in prompts)
- `reference` -- the expected or ideal response used as a ground truth comparison

## 6. Create the Project, Upload the Dataset, and Run Evaluation

Use the async `PrismaClient` to create a project, upload the transcript CSV, and submit an evaluation job:

```python
import asyncio
from pq_prisma_client import PrismaClient

async def run_evaluation():
    async with PrismaClient() as client:
        # Create or retrieve the project
        project = await client.projects.get_or_create(name="call-center-qa")
        print(f"Project: {project.name} (ID: {project.id})")

        # Create an evaluation run with the configuration
        run = await client.runs.create(
            project_id=project.id,
            name="q1-2026-transcript-review",
            run_type="dataset",
            evaluation_config=evaluation_config,
        )
        print(f"Run created: {run.name} (ID: {run.id})")

        # Upload the transcript dataset
        dataset = await client.datasets.upload("call_center_transcripts.csv")
        print(f"Dataset uploaded: {dataset.id}")

        # Submit an evaluation job — Prisma evaluates every row
        # using the evaluators and input_mapping from the run config
        job = await client.jobs.create(
            run_id=run.id,
            file_id=dataset.id,
        )
        print(f"Evaluation job submitted: {job.id}")

        return run

run = asyncio.run(run_evaluation())
```

The `datasets.upload()` method handles the file upload via presigned URLs. The `jobs.create()` call triggers the evaluation pipeline, which processes each row in the CSV using the evaluators and field mapping configured on the run. You don't need to iterate over rows yourself — the platform handles parallelization and batching.

## 7. Review Results

Open Prisma and navigate to the **call-center-qa** project. The project overview shows total validations, HITL requests pending expert review, and LLM cost. Click on the **q1-2026-transcript-review** run to see per-metric summaries:

- **Correctness pass rate**: How often agent responses matched expected answers
- **Hallucination pass rate**: How often agents stayed grounded in policy and fact
- **Policy adherence pass rate**: Percentage of transcripts classified as `compliant` vs `non_compliant` or `partial`
- **Ambiguous cases**: Transcripts routed to the human review queue for expert judgment

For detailed per-transcript results — which specific calls failed which evaluators — use the Prisma REST API to query evaluation results programmatically, or connect a visualization tool like Power BI via Prisma's analytics integration layer to build compliance dashboards.

The sample dataset includes several deliberate policy gaps to look for in the results:

| Call ID  | Expected Issue                                              |
|----------|-------------------------------------------------------------|
| CALL-003 | Agent cited Regulation E instead of Regulation Z            |
| CALL-005 | Agent skipped identity verification before account closure  |
| CALL-008 | Agent disclosed account info without identity verification  |
| CALL-009 | Agent skipped BSA/AML due diligence for a large international wire |
| CALL-011 | Agent required two IDs instead of one per policy            |

## Expected Output

When you run the complete script, you will see:

```
Dataset created: 12 transcripts
Project: call-center-qa (ID: proj_abc123)
Run created: q1-2026-transcript-review (ID: run_def456)
```

Evaluation results will be available in the Prisma project overview as the pipeline processes each transcript. Use the REST API or analytics integrations for detailed per-transcript breakdowns. Typical processing time depends on the number of transcripts and evaluators configured.

## Complete Example

```python
"""Automated QA for Call Center Transcripts — full example."""

import asyncio

import pandas as pd
from pq_prisma_client import PrismaClient
from prisma_ai import EvaluationConfig, InputMapping
from prisma_ai import Evaluators

# -- 1. Create sample dataset -------------------------------------------------

data = [
    {
        "call_id": "CALL-001",
        "customer_query": "I'd like to know the current interest rate on a 30-year fixed mortgage.",
        "agent_response": "Our current 30-year fixed mortgage rate is 6.75% APR for qualified borrowers. This rate is subject to change and your actual rate depends on your credit profile and down payment amount.",
        "expected_response": "The current 30-year fixed mortgage rate is 6.75% APR. Rates vary based on creditworthiness, down payment, and loan amount. Refer the customer to a loan officer for a personalized quote.",
        "policy_reference": "Mortgage Disclosure Policy §4.2: Agents must state the current advertised rate, clarify that rates are subject to change, and recommend a loan officer consultation.",
    },
    {
        "call_id": "CALL-002",
        "customer_query": "Why was I charged a $35 overdraft fee? I thought I had enough in my account.",
        "agent_response": "I see the overdraft occurred on March 15th when a $200 debit posted before your direct deposit cleared. I can submit a one-time courtesy reversal for you since this is your first overdraft in 12 months.",
        "expected_response": "Explain the transaction sequence that caused the overdraft, offer a courtesy fee reversal if eligible per the fee waiver matrix, and suggest overdraft protection enrollment.",
        "policy_reference": "Fee Dispute Policy §2.1: First overdraft in 12 months qualifies for automatic courtesy reversal. Agent must explain the cause and offer overdraft protection.",
    },
    {
        "call_id": "CALL-003",
        "customer_query": "I need to dispute a charge of $450 on my credit card from a merchant I don't recognize.",
        "agent_response": "I'll open a dispute for that $450 charge right away. You'll receive a provisional credit within 24 hours while we investigate. The investigation takes up to 90 days under Regulation E.",
        "expected_response": "Initiate a fraud dispute, issue provisional credit within 10 business days per Regulation Z (credit cards), confirm the merchant details, and advise the customer to monitor their account for additional unauthorized charges.",
        "policy_reference": "Fraud Dispute Policy §3.4: Credit card disputes fall under Regulation Z, not Regulation E. Provisional credit timeline is 10 business days. Agent must verify merchant details before filing.",
    },
    {
        "call_id": "CALL-004",
        "customer_query": "Can I get a copy of my account statements from the last two years for my tax preparer?",
        "agent_response": "Sure, I can mail those to the address on file. There's a $5 fee per statement for anything older than 6 months. Would you like me to proceed?",
        "expected_response": "Inform the customer that statements from the last 7 years are available free of charge through online banking. Paper copies older than 6 months incur a $5 per statement fee. Offer to walk them through the online download process first.",
        "policy_reference": "Document Request Policy §1.3: Always offer the free digital option before charging for paper copies. Statements are available online for 7 years at no cost.",
    },
    {
        "call_id": "CALL-005",
        "customer_query": "I want to close my savings account and transfer the balance to my checking account.",
        "agent_response": "I can process that for you right now. Your savings balance of $12,340 will be transferred to your checking account ending in 4521. The account will be closed immediately.",
        "expected_response": "Confirm the customer's identity with two-factor verification before processing. Disclose any early closure penalties if the account is under 6 months old. Confirm the transfer details and provide a confirmation number.",
        "policy_reference": "Account Closure Policy §5.1: Identity verification required for closure requests. Disclose early closure fees for accounts under 6 months. Provide written confirmation within 2 business days.",
    },
    {
        "call_id": "CALL-006",
        "customer_query": "My debit card was stolen. I need to cancel it and get a new one.",
        "agent_response": "I've immediately blocked your debit card ending in 7892. A replacement card will arrive in 5-7 business days. Would you like an expedited card for a $25 fee? In the meantime, you can use our mobile wallet for contactless payments.",
        "expected_response": "Block the card immediately, initiate a replacement, offer expedited shipping, document the theft for fraud monitoring, advise the customer to review recent transactions for unauthorized charges, and file a fraud alert.",
        "policy_reference": "Lost/Stolen Card Policy §6.2: Immediate block required. Agent must review last 48 hours of transactions with the customer and initiate fraud monitoring. Expedited replacement at $25 is optional.",
    },
    {
        "call_id": "CALL-007",
        "customer_query": "I'm having trouble making my loan payments. Is there any hardship program available?",
        "agent_response": "I understand this is a difficult situation. We do offer a hardship program that can reduce your payments by up to 50% for six months. I can transfer you to our loss mitigation team to discuss your options.",
        "expected_response": "Express empathy, confirm the hardship program exists, explain that options include payment deferral, reduced payments, or loan modification depending on circumstances. Transfer to loss mitigation for formal assessment. Do not promise specific terms.",
        "policy_reference": "Hardship Assistance Policy §8.1: Agents must not promise specific relief terms. Only the loss mitigation team can approve hardship modifications after reviewing the borrower's financial documentation.",
    },
    {
        "call_id": "CALL-008",
        "customer_query": "Can you tell me my account balance and recent transactions over the phone?",
        "agent_response": "Your checking account balance is $3,250.00. Your last three transactions were: $45 at Grocery Mart, $120 at Gas Station, and $89 at Online Retailer.",
        "expected_response": "Before disclosing any account information, verify the customer's identity using the standard authentication process (security questions or one-time passcode). Then provide the requested information.",
        "policy_reference": "Information Security Policy §10.1: Account information must never be disclosed without completing identity verification. Minimum two-factor authentication required for phone inquiries.",
    },
    {
        "call_id": "CALL-009",
        "customer_query": "I want to wire $50,000 to an international account in the Cayman Islands for a real estate investment.",
        "agent_response": "I can set up that wire transfer for you. The fee for international wires is $45. I'll just need the recipient's bank details and the transfer should complete in 2-3 business days.",
        "expected_response": "For international wire transfers over $10,000, complete enhanced due diligence including the purpose of the transfer, recipient relationship verification, and OFAC screening. Flag for BSA/AML review if the transaction meets suspicious activity indicators.",
        "policy_reference": "BSA/AML Policy §12.3: International transfers over $10,000 require enhanced due diligence, OFAC screening, and documentation of the purpose and recipient relationship. Agents must file a CTR and escalate if suspicious activity indicators are present.",
    },
    {
        "call_id": "CALL-010",
        "customer_query": "I noticed my CD matured last week. What are my options for reinvestment?",
        "agent_response": "Your 12-month CD matured with a balance of $25,000. You're currently in the 10-day grace period. Your options are: renew at the current 12-month rate of 4.50% APY, choose a different term, or withdraw the funds. If you don't act within the grace period, it auto-renews at the current rate.",
        "expected_response": "Confirm CD maturity details and grace period timeline. Present all reinvestment options including current rates for various terms. Mention the auto-renewal policy and any rate differences from the original term. Offer to connect with an investment advisor for larger balances.",
        "policy_reference": "CD Maturity Policy §7.1: Agents must disclose all available terms and current rates during the grace period. For balances over $10,000, offer a consultation with an investment advisor.",
    },
    {
        "call_id": "CALL-011",
        "customer_query": "I need to add my spouse as a joint account holder on my checking account.",
        "agent_response": "I can start that process. Your spouse will need to visit a branch with two forms of government-issued ID and sign the joint account agreement. There's no fee for adding a joint holder.",
        "expected_response": "Explain that adding a joint holder requires both parties to sign the joint account agreement. This can be done in-branch or via a notarized mail-in form. Only one government-issued ID is required per person. Disclose that both parties will have full access and liability.",
        "policy_reference": "Joint Account Policy §9.2: One government-issued ID per person required, not two. Both in-branch and notarized mail-in options must be offered. Agent must disclose shared liability.",
    },
    {
        "call_id": "CALL-012",
        "customer_query": "I'm calling about the promotional offer I received for a new credit card with 0% APR.",
        "agent_response": "I can see the offer on your profile. It's our Platinum Rewards card with 0% APR for the first 18 months on purchases and balance transfers. After the promotional period, the rate goes to our standard variable rate. Would you like to apply now?",
        "expected_response": "Confirm the specific offer details including the promotional period, standard APR after promotion, annual fee, balance transfer fees, and any spending requirements. Provide the Schumer Box disclosures verbally or direct the customer to the written terms.",
        "policy_reference": "Credit Card Marketing Policy §11.1: Agents must disclose the post-promotional APR, annual fee, and balance transfer fee (typically 3-5%) before processing an application. TILA Schumer Box disclosures are mandatory.",
    },
]

df = pd.DataFrame(data)
df.to_csv("call_center_transcripts.csv", index=False)
print(f"Dataset created: {len(df)} transcripts")

# -- 2. Define evaluators -----------------------------------------------------

policy_adherence = Evaluators.custom(
    name="policy_adherence",
    options=["compliant", "non_compliant", "partial"],
    evaluation_system_prompt=(
        "You are a compliance auditor evaluating call center agent responses "
        "for a financial services company. Your role is to determine whether "
        "the agent's response adheres to the company's internal policies and "
        "regulatory requirements.\n\n"
        "Evaluate based on these criteria:\n"
        "1. Did the agent follow the correct procedure as outlined in the policy reference?\n"
        "2. Did the agent provide all required disclosures?\n"
        "3. Did the agent avoid making unauthorized promises or commitments?\n"
        "4. Did the agent comply with applicable regulations (TILA, Reg E, Reg Z, BSA/AML)?\n\n"
        "Classify the response as:\n"
        "- compliant: The agent fully followed policy and regulatory requirements\n"
        "- partial: The agent followed some but not all policy requirements\n"
        "- non_compliant: The agent violated policy or missed critical regulatory requirements"
    ),
    evaluation_human_prompt=(
        "Customer query: {query}\n\n"
        "Agent response: {response}\n\n"
        "Evaluate the agent's response for policy adherence.\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Provide your classification and a brief explanation of which policy "
        "requirements were met or missed."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor performing a second-level review of a "
        "call center agent response evaluation. You have access to the original "
        "evaluation and additional context from the policy reference and expected "
        "response. Re-analyze the agent's response with this additional context "
        "to confirm or revise the initial classification.\n\n"
        "Pay special attention to:\n"
        "- Regulatory requirements that may have been overlooked\n"
        "- Subtle policy violations (e.g., missing disclosures, incorrect procedures)\n"
        "- Whether partial compliance should be reclassified based on the severity of gaps"
    ),
    reeval_human_prompt=(
        "Customer query: {query}\n\n"
        "Agent response: {response}\n\n"
        "Previous evaluation context: {memories}\n\n"
        "Re-evaluate the agent's response considering the additional context above. "
        "Confirm or revise the policy adherence classification."
    ),
)

# -- 3. Configure evaluation ---------------------------------------------------

evaluation_config = EvaluationConfig(
    evaluators=[
        Evaluators.correctness(),
        Evaluators.hallucination(),
        policy_adherence,
    ],
    input_mapping=InputMapping(
        query="customer_query",
        response="agent_response",
        reference="expected_response",
    ),
)

# -- 4. Create project, upload dataset, and run evaluation -------------------

async def run_evaluation():
    async with PrismaClient() as client:
        project = await client.projects.get_or_create(name="call-center-qa")
        print(f"Project: {project.name} (ID: {project.id})")

        run = await client.runs.create(
            project_id=project.id,
            name="q1-2026-transcript-review",
            run_type="dataset",
            evaluation_config=evaluation_config,
        )
        print(f"Run created: {run.name} (ID: {run.id})")

        dataset = await client.datasets.upload("call_center_transcripts.csv")
        print(f"Dataset uploaded: {dataset.id}")

        job = await client.jobs.create(run_id=run.id, file_id=dataset.id)
        print(f"Evaluation job submitted: {job.id}")

        return run

run = asyncio.run(run_evaluation())
```

## Next Steps

- [Getting Started](getting-started.md) -- set up Prisma AI tracing and evaluation from scratch
- [Detect Hallucinations in a RAG Pipeline](rag-hallucination-detection.md) -- evaluate retrieval-augmented generation quality
- [Build Custom Policy Compliance Evaluators](custom-policy-evaluators.md) -- go deeper on domain-specific evaluation criteria for regulated industries
- [CI/CD Quality Gates](cicd-quality-gates.md) -- automate evaluation as part of your deployment pipeline
