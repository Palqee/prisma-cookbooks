# Evaluating Contact Center Agents at Scale

Validate a week's worth of contact center transcripts against multiple policy metrics to surface compliance gaps, missed escalations, and disclosure failures across hundreds of interactions.

## What You'll Learn

- Define **multiple policy metrics** that validate different dimensions of agent behavior
- Log dozens of transcripts through the OTLP logs pipeline in a single session
- Interpret per-metric pass rates to pinpoint which policy areas have the most violations
- Upload historical transcript files as an alternative to real-time logging

## Prerequisites

- **Completed the [Getting Started](getting-started.md) guide** — you should have a working `prisma-ai` installation and a `response-compliance` metric defined
- **Python 3.10+**
- **A Prisma AI API key** (from your Prisma instance settings)

## 1. Install Dependencies

If you followed Getting Started, you already have everything you need:

```bash
pip install prisma-ai
```

## 2. Configure Environment Variables

Same configuration as Getting Started:

```bash
export PRISMA_API_KEY="prisma_sk_your_key_here"
export PRISMA_BASE_URL="https://your-prisma-instance.example.com"
export PRISMA_WORKSPACE_ID="your-workspace-id"
```

## 3. Define Policy Metrics

Create a file called `weekly_qa.py`. You will define three policy metrics, each targeting a different dimension of agent behavior. The first — `response-compliance` — is the same metric from Getting Started. The other two are new.

### response-compliance (reference)

This metric validates greeting, identity verification, accurate information, and professional close. See the [Getting Started guide](getting-started.md#3-define-a-policy-metric) for the full 4-prompt definition. Here we just import the same definition:

```python
import prisma_ai
from prisma_ai import Evaluators

response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],
    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires all four of the following:\n\n"
        "1. GREETING: The agent uses a professional greeting that includes their "
        "name and the company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action on the account, the "
        "agent verifies the customer's identity by requesting at least two pieces "
        "of identifying information.\n"
        "3. ACCURATE INFORMATION: The agent provides information that is consistent "
        "with company policy. The reference field contains the expected policy-correct "
        "response for this scenario.\n"
        "4. PROFESSIONAL CLOSE: The agent ends the interaction with a summary of "
        "actions taken, clear next steps, and an offer for further assistance.\n\n"
        "If ALL four criteria are met, classify as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify as 'non_compliant'."
    ),
    evaluation_human_prompt=(
        "Validate the following contact center interaction against company policy.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain which of the four policy criteria "
        "were met or violated, then state your classification."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the company's agent response policy strictly. "
        "Pay special attention to identity verification — this is the most critical "
        "step in financial services interactions."
    ),
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction but was uncertain.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Provide your final classification as 'compliant' or 'non_compliant'."
    ),
)
```

### escalation-procedure (new)

This metric checks whether the agent correctly escalated interactions that require a supervisor, specialist, or regulatory team. Financial services agents must escalate when a customer files a formal complaint, requests something beyond the agent's authority, or raises a regulatory concern.

```python
escalation_procedure = Evaluators.custom(
    name="escalation-procedure",
    options=["proper_escalation", "missed_escalation", "not_applicable"],

    # --- Prompt 1: Evaluation system prompt ---
    # Defines what constitutes a required escalation.
    evaluation_system_prompt=(
        "You are an escalation policy auditor for a financial services contact center. "
        "Your job is to determine whether the agent correctly handled escalation "
        "requirements during the interaction.\n\n"
        "Company policy (Policy ESC-200) requires agents to escalate in the following "
        "situations:\n\n"
        "1. FORMAL COMPLAINTS: Any interaction where the customer explicitly states "
        "they want to file a complaint, mentions contacting a regulator (CFPB, SEC, "
        "FINRA, state attorney general), or expresses intent to take legal action.\n"
        "2. AUTHORITY LIMITS: Requests that exceed the agent's authority, including: "
        "wire transfers over $10,000, fee waivers above $500, changes to account "
        "beneficiaries, or any modification to investment allocations.\n"
        "3. REGULATORY TRIGGERS: The customer raises concerns about discrimination, "
        "unfair lending practices, unauthorized account activity involving potential "
        "fraud rings, or elder financial abuse.\n"
        "4. UNRESOLVED DISPUTES: The customer has called more than twice about the "
        "same issue without resolution.\n\n"
        "When escalation is required, the agent must: (a) acknowledge the customer's "
        "concern, (b) explain that a specialist or supervisor will handle the matter, "
        "(c) provide a case reference number, and (d) give a specific callback timeframe.\n\n"
        "Classify as 'proper_escalation' if the agent correctly identified an "
        "escalation trigger and followed all four steps. Classify as 'missed_escalation' "
        "if an escalation trigger was present but the agent did not escalate or skipped "
        "required steps. Classify as 'not_applicable' if no escalation trigger exists "
        "in the interaction."
    ),

    # --- Prompt 2: Evaluation human prompt ---
    # Presents the interaction for escalation review.
    evaluation_human_prompt=(
        "Review the following interaction for escalation policy compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "First, identify whether any escalation trigger is present (formal complaint, "
        "authority limit, regulatory trigger, or unresolved dispute). If so, determine "
        "whether the agent followed all four escalation steps (acknowledge, explain "
        "specialist handling, provide reference number, give callback timeframe). "
        "State your classification and reasoning."
    ),

    # --- Prompt 3: Re-evaluation system prompt ---
    # Stricter guidance for ambiguous escalation cases.
    reeval_system_prompt=(
        "You are a senior escalation auditor reviewing a borderline case. Apply "
        "Policy ESC-200 strictly. When in doubt about whether an escalation trigger "
        "is present, consider the customer's intent and emotional state. A customer "
        "who says 'this is unacceptable' or 'I want to speak to someone higher up' "
        "is signaling a formal complaint even if they do not use the word 'complaint'. "
        "Implicit escalation triggers count. If the agent attempted to resolve the "
        "issue themselves when escalation was required, classify as 'missed_escalation' "
        "regardless of whether the customer seemed satisfied."
    ),

    # --- Prompt 4: Re-evaluation human prompt ---
    # Second pass with prior reasoning.
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction for escalation compliance "
        "but was uncertain.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Apply Policy ESC-200 strictly. Classify as 'proper_escalation', "
        "'missed_escalation', or 'not_applicable'. Explain your reasoning, paying "
        "close attention to implicit escalation triggers."
    ),
)
```

### disclosure-requirements (new)

This metric checks whether the agent provided required regulatory disclosures during the interaction. In financial services, certain topics — investments, fees, account terms — carry mandatory disclosure obligations.

```python
disclosure_requirements = Evaluators.custom(
    name="disclosure-requirements",
    options=["disclosures_provided", "disclosures_missing", "not_applicable"],

    # --- Prompt 1: Evaluation system prompt ---
    # Defines mandatory disclosure categories.
    evaluation_system_prompt=(
        "You are a regulatory disclosure auditor for a financial services contact "
        "center. Your job is to determine whether the agent provided all required "
        "disclosures during the interaction.\n\n"
        "Company policy (Policy DISC-100) requires the following disclosures when "
        "the corresponding topic arises:\n\n"
        "1. INVESTMENT RISK: Any discussion of investment products, portfolio "
        "performance, or asset allocation must include the statement that investments "
        "carry risk, past performance does not guarantee future results, and the "
        "customer should review the prospectus.\n"
        "2. FEE SCHEDULES: When discussing account fees, service charges, or penalty "
        "rates, the agent must disclose the specific fee amount, when it applies, "
        "and how the customer can access the full fee schedule.\n"
        "3. RATE TERMS: When quoting interest rates (mortgage, loan, savings), the "
        "agent must state whether the rate is fixed or variable, the term length, "
        "and that rates are subject to change based on market conditions and "
        "creditworthiness.\n"
        "4. ACCOUNT TERMS: When opening, closing, or modifying accounts, the agent "
        "must mention early withdrawal penalties (if applicable), minimum balance "
        "requirements, and direct the customer to the full terms and conditions.\n"
        "5. PRIVACY NOTICE: When collecting or discussing personal financial data, "
        "the agent must reference the company's privacy policy and the customer's "
        "right to opt out of data sharing with third parties.\n\n"
        "Classify as 'disclosures_provided' if all applicable disclosures were "
        "included. Classify as 'disclosures_missing' if one or more required "
        "disclosures were omitted. Classify as 'not_applicable' if the interaction "
        "did not involve any disclosure-triggering topics."
    ),

    # --- Prompt 2: Evaluation human prompt ---
    # Presents the interaction for disclosure review.
    evaluation_human_prompt=(
        "Review the following interaction for regulatory disclosure compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "First, identify which disclosure categories apply based on the topics "
        "discussed (investment risk, fee schedules, rate terms, account terms, "
        "privacy notice). Then check whether the agent provided each required "
        "disclosure. State your classification and list any missing disclosures."
    ),

    # --- Prompt 3: Re-evaluation system prompt ---
    # Stricter guidance for ambiguous disclosure cases.
    reeval_system_prompt=(
        "You are a senior regulatory auditor reviewing a borderline disclosure "
        "case. Apply Policy DISC-100 strictly. Even if the agent alluded to a "
        "topic without making a formal recommendation, the disclosure obligation "
        "is triggered by the topic being discussed, not by the agent giving advice. "
        "For example, if a customer asks about mortgage rates and the agent quotes "
        "a number, the rate terms disclosure is required even if the agent says "
        "'this is just for informational purposes'. Partial disclosures (mentioning "
        "risk but not the prospectus) count as 'disclosures_missing'."
    ),

    # --- Prompt 4: Re-evaluation human prompt ---
    # Second pass with prior reasoning.
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction for disclosure compliance "
        "but was uncertain.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Apply Policy DISC-100 strictly. Classify as 'disclosures_provided', "
        "'disclosures_missing', or 'not_applicable'. List every applicable "
        "disclosure category and whether it was satisfied."
    ),
)
```

## 4. Initialize and Log Transcripts

Now wire up the three policy metrics and log a representative sample of a week's interactions. These 12 transcripts cover the range of scenarios your QA team encounters: routine requests, escalation-worthy complaints, and disclosure-heavy financial discussions.

```python
session = prisma_ai.init(
    project_name="contact-center-qa",
    run_name="weekly-qa-2026-w13",
    evaluators=[response_compliance, escalation_procedure, disclosure_requirements],
    enable_logging=True,
)

logger = session.get_logger()

transcripts = [
    # --- 1. Mortgage rate inquiry (disclosure-heavy) ---
    {
        "query": (
            "I'm looking at refinancing my home. What's your current 30-year "
            "fixed mortgage rate?"
        ),
        "response": (
            "Thank you for calling Meridian Financial, this is David. I'd be happy "
            "to help with refinancing. Could you verify your account number and date "
            "of birth? ... Great, thank you, Mrs. Kowalski. Our current 30-year fixed "
            "rate is 6.25%. That's a really competitive rate right now. Would you like "
            "me to start an application?"
        ),
        "reference": (
            "Agent must verify identity before discussing account-specific rates. "
            "When quoting mortgage rates, agent must disclose: (1) whether the rate "
            "is fixed or variable, (2) the rate lock period, (3) that rates are "
            "subject to change based on market conditions and creditworthiness, and "
            "(4) direct the customer to the full rate sheet. Per Policy DISC-100."
        ),
    },
    # --- 2. Fraud dispute with proper escalation ---
    {
        "query": (
            "There are three charges on my account I didn't make — $800 at an "
            "electronics store, $1,200 at a jewelry shop, and $450 at a gas station, "
            "all on the same day. Someone stole my card information. I want to file "
            "a formal complaint about how long it took your fraud team to catch this."
        ),
        "response": (
            "Good morning, this is Angela with Meridian Financial. I'm sorry to "
            "hear about these unauthorized charges — I understand how concerning this "
            "is. Let me verify your identity first: could you provide your account "
            "number and the last four digits of your SSN? ... Thank you, Mr. Yamamoto. "
            "I've flagged all three charges for investigation and placed a temporary "
            "hold on your card to prevent further unauthorized use. Regarding your "
            "complaint about our fraud detection response time, I want to make sure "
            "this is handled properly. I'm escalating this to our Fraud Resolution "
            "Supervisor team. Your case reference number is FRD-2026-4471. A supervisor "
            "will call you back within 24 hours to discuss both the unauthorized charges "
            "and your concerns about our detection process. Is there anything else I can "
            "help with in the meantime?"
        ),
        "reference": (
            "Unauthorized charges: freeze card immediately, flag transactions for "
            "investigation, issue provisional credit within 10 business days per Reg E. "
            "Formal complaint about fraud detection: escalate per Policy ESC-200 — "
            "acknowledge concern, explain supervisor involvement, provide case reference, "
            "give callback timeframe."
        ),
    },
    # --- 3. Balance transfer — no verification (compliance failure) ---
    {
        "query": (
            "I'd like to transfer $3,500 from my checking to my savings. Can you "
            "do that while I'm on the phone?"
        ),
        "response": (
            "Absolutely, I can take care of that for you. Transferring $3,500 from "
            "checking to savings now. The transfer is complete — you'll see the updated "
            "balance within the hour. Is there anything else?"
        ),
        "reference": (
            "Internal transfers require identity verification before processing. "
            "Agent must verify customer identity with at least two forms of ID per "
            "Policy FIN-302. After verification, confirm transfer amount, source and "
            "destination accounts, and expected posting time. Close with summary."
        ),
    },
    # --- 4. Investment question — missing disclosures ---
    {
        "query": (
            "My 401(k) rollover is sitting in a money market fund. Should I move "
            "it into an index fund? What's your S&P 500 fund been returning?"
        ),
        "response": (
            "Hi, this is Marcus with Meridian Financial. Let me pull up your account — "
            "can I get your account number and date of birth? ... Thank you, Ms. Reeves. "
            "Our S&P 500 index fund returned 18.3% last year and has a 10-year average "
            "of 11.7%. It's been one of our best performers. Moving your rollover into "
            "that fund could significantly improve your long-term growth. I can process "
            "the reallocation for you right now if you'd like. Anything else I can help "
            "with today?"
        ),
        "reference": (
            "When discussing investment products, agent must disclose: (1) investments "
            "carry risk including possible loss of principal, (2) past performance does "
            "not guarantee future results, (3) customer should review the fund prospectus "
            "before making changes. Per Policy DISC-100. Agent must not make personalized "
            "investment recommendations without directing customer to a licensed advisor."
        ),
    },
    # --- 5. Account closure — compliant across all metrics ---
    {
        "query": (
            "I'm moving to another bank. I need to close my savings account today."
        ),
        "response": (
            "Good afternoon, this is Rachel with Meridian Financial. I'm sorry to "
            "hear you're leaving us, but I'll make this as smooth as possible. To get "
            "started, could you provide your account number and your mailing address on "
            "file? ... Thank you, Mr. Okafor, identity confirmed. Before I process the "
            "closure, I want to let you know that your savings account has a $75 early "
            "closure fee since it was opened less than 12 months ago. You can find the "
            "full fee schedule and account terms at meridianfinancial.com/terms. Your "
            "remaining balance of $2,140.25 minus the $75 fee — so $2,065.25 — will be "
            "sent as a check to your mailing address within 5-7 business days. You'll "
            "receive a closure confirmation email within the hour. Is there anything else "
            "I can help with?"
        ),
        "reference": (
            "Account closure: verify identity, disclose any early closure penalties "
            "and direct customer to full terms and conditions per Policy DISC-100. "
            "State final balance after fees, confirm disbursement method and timeline. "
            "Send confirmation email."
        ),
    },
    # --- 6. Complaint about fees — missed escalation ---
    {
        "query": (
            "This is the third time I've called about the $35 overdraft fee from "
            "January. The first agent said it would be reversed, the second agent "
            "said they couldn't find any notes, and now I'm calling again. This is "
            "completely unacceptable. I want this resolved today or I'm filing a "
            "complaint with the CFPB."
        ),
        "response": (
            "Hi, this is Kevin with Meridian Financial. I apologize for the "
            "inconvenience. Let me verify your identity — account number and date of "
            "birth, please? ... Thank you, Ms. Hernandez. I can see the $35 overdraft "
            "fee from January 12th. I'm going to go ahead and reverse that for you "
            "right now. The credit will appear in your account within 24 hours. I've "
            "also added a note to your file so this doesn't happen again. Is there "
            "anything else?"
        ),
        "reference": (
            "Third call about same issue: escalation required per Policy ESC-200 "
            "(unresolved dispute — customer has called more than twice). Customer "
            "explicitly mentions CFPB complaint: regulatory escalation trigger. "
            "Agent must: acknowledge concern, explain supervisor/specialist handling, "
            "provide case reference number, give specific callback timeframe. "
            "Fee reversal alone does not satisfy escalation requirements."
        ),
    },
    # --- 7. Wire transfer over $10k — needs escalation ---
    {
        "query": (
            "I need to wire $25,000 to my daughter's account at Chase for a "
            "down payment on her house. Can we do that today?"
        ),
        "response": (
            "Thank you for calling Meridian Financial, this is Priya. I'd love to "
            "help with that wire transfer. Let me verify your identity first — account "
            "number and last four of your SSN? ... Thank you, Mr. Chen. For wire "
            "transfers over $10,000, I'll need to connect you with our Wire Transfer "
            "Specialist team — they handle high-value transfers to ensure everything "
            "is processed securely and in compliance with federal reporting requirements. "
            "Your case reference number is WIR-2026-0892. A specialist will be on the "
            "line within 5 minutes — they'll walk you through the rest. Is there anything "
            "else you need while we wait?"
        ),
        "reference": (
            "Wire transfers over $10,000: exceeds agent authority, escalate per "
            "Policy ESC-200. Agent must verify identity, acknowledge request, explain "
            "specialist handling, provide reference number, and give timeframe. "
            "Federal CTR (Currency Transaction Report) requirements apply."
        ),
    },
    # --- 8. Savings rate inquiry — missing rate terms disclosure ---
    {
        "query": (
            "What's the interest rate on your high-yield savings account? I'm "
            "thinking about opening one."
        ),
        "response": (
            "Good morning, this is Tanya with Meridian Financial. Great question! "
            "Our high-yield savings account is currently offering 4.15% APY. It's one "
            "of the best rates in the market right now. There's no minimum balance to "
            "open, and you can set up direct deposit to start earning right away. Want "
            "me to get you started with an application?"
        ),
        "reference": (
            "When quoting savings rates, agent must disclose: (1) whether the rate "
            "is fixed or variable (high-yield savings is variable), (2) that rates are "
            "subject to change based on market conditions, (3) any minimum balance "
            "requirements for the quoted APY tier, and (4) direct the customer to full "
            "account terms. Per Policy DISC-100. Note: identity verification not required "
            "for general product inquiries where no account is accessed."
        ),
    },
    # --- 9. Compliant interaction — credit card payment ---
    {
        "query": (
            "I missed my credit card payment that was due yesterday. Can I make "
            "the payment now and avoid the late fee?"
        ),
        "response": (
            "Thank you for calling Meridian Financial, my name is Derek. I can "
            "definitely help with that. Could you verify your account number and "
            "date of birth? ... Thank you, Ms. Tanaka. I can see the payment of "
            "$412.33 was due yesterday. I've processed the payment using the bank "
            "account on file. Unfortunately, the $29 late fee has already been "
            "assessed — I'm not able to waive it, but you can find details about "
            "late fee policies and how to set up autopay to avoid this in the future "
            "at meridianfinancial.com/terms. Your next statement will reflect the "
            "payment and the fee. To summarize: payment of $412.33 posted today, "
            "late fee of $29 applied. Is there anything else I can help with?"
        ),
        "reference": (
            "Late payments: process payment immediately upon verification. Disclose "
            "the late fee amount and direct customer to fee schedule per Policy DISC-100. "
            "Offer autopay enrollment to prevent future occurrences. Agents cannot waive "
            "late fees above $25 without supervisor approval."
        ),
    },
    # --- 10. Elder financial abuse concern — missed escalation ---
    {
        "query": (
            "I'm calling about my mother's account. She's 82 and I have power of "
            "attorney. I've noticed several large withdrawals she doesn't remember "
            "making — $2,000 here, $1,500 there, over the past month. Her neighbor "
            "has been driving her to the bank. I'm worried someone is taking "
            "advantage of her."
        ),
        "response": (
            "Hi, this is Laura with Meridian Financial. I understand your concern. "
            "Let me verify your identity and the power of attorney on file — could "
            "you provide the account number and your date of birth? ... Thank you, "
            "Ms. Williams. I can see the withdrawals you're referring to. I've placed "
            "an alert on the account so you'll be notified of any future transactions "
            "over $500. I'd also recommend changing the PIN and limiting branch "
            "withdrawals to the account holder only. Would you like me to do that? "
            "Is there anything else?"
        ),
        "reference": (
            "Suspected elder financial abuse: mandatory escalation per Policy ESC-200 "
            "(regulatory trigger). Agent must immediately escalate to the Vulnerable "
            "Customer Protection team. Many states require filing a Suspicious Activity "
            "Report (SAR) within 24 hours. Agent must: acknowledge the concern, explain "
            "that the specialist team will investigate, provide a case reference number, "
            "and give a callback timeframe. Placing an alert is helpful but does NOT "
            "satisfy the escalation requirement."
        ),
    },
    # --- 11. Compliant interaction — address change ---
    {
        "query": (
            "I just moved and need to update my mailing address."
        ),
        "response": (
            "Good afternoon, this is Sam with Meridian Financial. I can update that "
            "for you right away. For security, could you provide your account number "
            "and the last four digits of your SSN? ... Thank you, Mr. Petrov. I've "
            "updated your mailing address to 742 Evergreen Terrace, Springfield. "
            "Please note that per our privacy policy, your address information is "
            "protected and will only be shared with third parties as described in "
            "our privacy notice — you can review it or opt out at "
            "meridianfinancial.com/privacy. You'll receive a confirmation letter at "
            "your new address within 3-5 business days. Is there anything else I can "
            "help with?"
        ),
        "reference": (
            "Address changes require identity verification. After updating, reference "
            "the privacy policy and customer's opt-out rights per Policy DISC-100 "
            "(privacy notice disclosure). Send confirmation to new address."
        ),
    },
    # --- 12. Beneficiary change — needs escalation, agent handles it ---
    {
        "query": (
            "I need to change the beneficiary on my life insurance policy from "
            "my ex-spouse to my sister."
        ),
        "response": (
            "Thank you for calling Meridian Financial, this is Natalie. I can "
            "help you get that process started. Let me verify your identity — could "
            "you provide your policy number and date of birth? ... Thank you, "
            "Mr. Goodwin. Beneficiary changes are handled by our Policy Services "
            "team to ensure all legal requirements are met. I've created a case for "
            "you — your reference number is BEN-2026-1134. A Policy Services specialist "
            "will call you within 48 hours to walk you through the required forms and "
            "any notarization requirements. In the meantime, your current beneficiary "
            "designation remains in effect. Is there anything else I can help with today?"
        ),
        "reference": (
            "Beneficiary changes exceed agent authority — escalate per Policy ESC-200. "
            "Agent must verify identity, acknowledge request, explain specialist handling, "
            "provide reference number, and give callback timeframe. Beneficiary changes "
            "require signed forms and may require notarization depending on state law."
        ),
    },
]

for transcript in transcripts:
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
python weekly_qa.py
```

Expected output:

```
Logged: I'm looking at refinancing my home. What's your current ...
Logged: There are three charges on my account I didn't make — $8...
Logged: I'd like to transfer $3,500 from my checking to my savi...
Logged: My 401(k) rollover is sitting in a money market fund. S...
Logged: I'm moving to another bank. I need to close my savings ...
Logged: This is the third time I've called about the $35 overdr...
Logged: I need to wire $25,000 to my daughter's account at Chas...
Logged: What's the interest rate on your high-yield savings acc...
Logged: I missed my credit card payment that was due yesterday....
Logged: I'm calling about my mother's account. She's 82 and I ...
Logged: I just moved and need to update my mailing address.    ...
Logged: I need to change the beneficiary on my life insurance p...
```

## 5. Run and Interpret Results

Open Prisma and navigate to the **contact-center-qa** project, then click into the **weekly-qa-2026-w13** run. Each transcript is now validated against all three policy metrics independently, which means a single interaction can pass one metric and fail another.

### Per-metric results

| # | Scenario | response-compliance | escalation-procedure | disclosure-requirements |
|---|---|---|---|---|
| 1 | Mortgage rate inquiry | `compliant` | `not_applicable` | `disclosures_missing` |
| 2 | Fraud dispute + complaint | `compliant` | `proper_escalation` | `not_applicable` |
| 3 | Balance transfer (no verification) | `non_compliant` | `not_applicable` | `not_applicable` |
| 4 | Investment reallocation | `compliant` | `not_applicable` | `disclosures_missing` |
| 5 | Account closure | `compliant` | `not_applicable` | `disclosures_provided` |
| 6 | Repeat fee complaint + CFPB threat | `compliant` | `missed_escalation` | `disclosures_provided` |
| 7 | Wire transfer over $10k | `compliant` | `proper_escalation` | `not_applicable` |
| 8 | Savings rate inquiry | `compliant` | `not_applicable` | `disclosures_missing` |
| 9 | Late credit card payment | `compliant` | `not_applicable` | `disclosures_provided` |
| 10 | Elder financial abuse concern | `compliant` | `missed_escalation` | `not_applicable` |
| 11 | Address change | `compliant` | `not_applicable` | `disclosures_provided` |
| 12 | Beneficiary change | `compliant` | `proper_escalation` | `not_applicable` |

### Pass rate summary

| Policy Metric | Pass | Fail | N/A | Pass Rate (excl. N/A) |
|---|---|---|---|---|
| response-compliance | 11 | 1 | 0 | **91.7%** |
| escalation-procedure | 3 | 2 | 7 | **60.0%** |
| disclosure-requirements | 4 | 3 | 5 | **57.1%** |

### What this tells you

- **Response compliance is strong** (91.7%). Agents generally follow the greeting-verification-information-close protocol. The one failure (Transcript 3) is a serious identity verification skip.
- **Escalation is the biggest risk area** (60.0%). Two of the five interactions that required escalation were missed — the repeat CFPB complaint (Transcript 6) and the elder abuse concern (Transcript 10). Both carry significant regulatory exposure.
- **Disclosures are frequently incomplete** (57.1%). Agents quote rates and discuss investment performance without the required risk, terms, or prospectus disclosures. This is a training gap — agents know the products but skip the mandated language.

A single policy metric would have missed most of these issues. The response-compliance metric alone would have flagged only one of the six total violations across all three dimensions.

## 6. Alternative: Batch File Upload

If your QA team already has transcripts stored as CSV or JSON files (exported from your CRM, telephony platform, or call recording system), you can upload them directly instead of logging one at a time.

### CSV format

Your file should have columns that map to `query`, `response`, and `reference`:

```csv
customer_message,agent_reply,expected_response
"I'm looking at refinancing my home...","Thank you for calling Meridian...","Agent must verify identity before..."
"There are three charges on my account...","Good morning, this is Angela...","Unauthorized charges: freeze card..."
```

### Upload and validate

```python
from prisma_ai import PrismaClient, InputMapping

client = PrismaClient()

# Upload the transcript file
dataset = await client.datasets.upload("weekly_transcripts.csv")

# Create a run with your policy metrics
run = await client.runs.create(
    project_name="contact-center-qa",
    run_name="weekly-qa-2026-w13-batch",
    evaluators=[response_compliance, escalation_procedure, disclosure_requirements],
    input_mapping=InputMapping(
        query="customer_message",
        response="agent_reply",
        reference="expected_response",
    ),
)

# Submit the batch job
job = await client.jobs.create(
    run_id=run.id,
    file_id=dataset.id,
)

print(f"Batch job submitted: {job.id}")
print(f"Transcripts: {dataset.row_count}")
print(f"Policy metrics: 3")
print(f"Total validations: {dataset.row_count * 3}")
```

This is useful when you have hundreds or thousands of historical transcripts to validate at once. The results appear in the same Prisma dashboard and support the same per-metric breakdown.

## 7. Production Patterns

In production, transcripts flow continuously from your telephony or CRM system rather than being logged manually. Here are common integration patterns.

### Stream from a CRM or telephony platform

```python
import prisma_ai
from prisma_ai import Evaluators

# Initialize once at application startup
session = prisma_ai.init(
    project_name="contact-center-qa",
    run_name="production-2026-q1",
    evaluators=[response_compliance, escalation_procedure, disclosure_requirements],
    enable_logging=True,
)
logger = session.get_logger()


def on_call_completed(call_record):
    """Callback triggered by your telephony platform when a call ends."""
    logger.log(
        query=call_record["customer_transcript"],
        response=call_record["agent_transcript"],
        reference=call_record["policy_reference"],
    )
```

### Scheduled weekly batch

For teams that export transcripts on a schedule (e.g., every Monday morning), combine the batch upload pattern with a cron job or workflow scheduler:

```python
# weekly_qa_job.py — triggered by cron or Airflow
import asyncio
from prisma_ai import PrismaClient
from datetime import datetime

async def run_weekly_qa():
    client = PrismaClient()
    week = datetime.now().strftime("%Y-w%W")

    dataset = await client.datasets.upload(f"/data/transcripts/week_{week}.csv")
    run = await client.runs.create(
        project_name="contact-center-qa",
        run_name=f"weekly-qa-{week}",
        evaluators=[response_compliance, escalation_procedure, disclosure_requirements],
    )
    job = await client.jobs.create(run_id=run.id, file_id=dataset.id)
    print(f"Weekly QA job submitted: {job.id} ({dataset.row_count} transcripts)")

asyncio.run(run_weekly_qa())
```

## Complete Example

A single file combining all three policy metrics and the 12 transcript scenarios:

```python
"""Weekly QA — Validate Contact Center Interactions Across Multiple Policies."""

import prisma_ai
from prisma_ai import Evaluators

# --- Policy Metric 1: Response Compliance ---
response_compliance = Evaluators.custom(
    name="response-compliance",
    options=["compliant", "non_compliant"],
    evaluation_system_prompt=(
        "You are a compliance auditor for a financial services contact center. "
        "Your job is to determine whether an agent's response to a customer "
        "follows company policy. The policy requires all four of the following:\n\n"
        "1. GREETING: The agent uses a professional greeting that includes their "
        "name and the company name.\n"
        "2. IDENTITY VERIFICATION: Before taking any action on the account, the "
        "agent verifies the customer's identity by requesting at least two pieces "
        "of identifying information.\n"
        "3. ACCURATE INFORMATION: The agent provides information that is consistent "
        "with company policy.\n"
        "4. PROFESSIONAL CLOSE: The agent ends the interaction with a summary of "
        "actions taken, clear next steps, and an offer for further assistance.\n\n"
        "If ALL four criteria are met, classify as 'compliant'. "
        "If ANY criterion is missing or incorrect, classify as 'non_compliant'."
    ),
    evaluation_human_prompt=(
        "Validate the following contact center interaction against company policy.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Which classification applies? Explain which criteria were met or violated, "
        "then state your classification."
    ),
    reeval_system_prompt=(
        "You are a senior compliance auditor reviewing a case where a previous "
        "auditor was uncertain. Apply the company's agent response policy strictly. "
        "Pay special attention to identity verification — this is the most critical "
        "step in financial services interactions."
    ),
    reeval_human_prompt=(
        "A previous auditor reviewed this interaction but was uncertain.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Expected policy-correct response:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Provide your final classification as 'compliant' or 'non_compliant'."
    ),
)

# --- Policy Metric 2: Escalation Procedure ---
escalation_procedure = Evaluators.custom(
    name="escalation-procedure",
    options=["proper_escalation", "missed_escalation", "not_applicable"],
    evaluation_system_prompt=(
        "You are an escalation policy auditor for a financial services contact center. "
        "Policy ESC-200 requires agents to escalate in these situations:\n\n"
        "1. FORMAL COMPLAINTS: Customer states they want to file a complaint, mentions "
        "a regulator (CFPB, SEC, FINRA), or threatens legal action.\n"
        "2. AUTHORITY LIMITS: Requests exceeding agent authority — wire transfers over "
        "$10,000, fee waivers above $500, beneficiary changes, investment reallocations.\n"
        "3. REGULATORY TRIGGERS: Discrimination concerns, unfair lending, unauthorized "
        "activity involving fraud rings, or elder financial abuse.\n"
        "4. UNRESOLVED DISPUTES: Customer has called more than twice about the same issue.\n\n"
        "When escalation is required, the agent must: (a) acknowledge the concern, "
        "(b) explain specialist/supervisor handling, (c) provide a case reference number, "
        "(d) give a specific callback timeframe.\n\n"
        "Classify as 'proper_escalation' if triggers present and all steps followed. "
        "'missed_escalation' if triggers present but agent did not escalate or skipped steps. "
        "'not_applicable' if no escalation triggers exist."
    ),
    evaluation_human_prompt=(
        "Review the following interaction for escalation policy compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Identify any escalation triggers, then assess whether the agent followed "
        "all four escalation steps. State your classification and reasoning."
    ),
    reeval_system_prompt=(
        "You are a senior escalation auditor reviewing a borderline case. Apply "
        "Policy ESC-200 strictly. Implicit escalation triggers count — a customer "
        "who says 'this is unacceptable' or 'I want to speak to someone higher up' "
        "is signaling a formal complaint. If the agent resolved the issue themselves "
        "when escalation was required, classify as 'missed_escalation'."
    ),
    reeval_human_prompt=(
        "A previous auditor was uncertain about this interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Classify as 'proper_escalation', 'missed_escalation', or 'not_applicable'."
    ),
)

# --- Policy Metric 3: Disclosure Requirements ---
disclosure_requirements = Evaluators.custom(
    name="disclosure-requirements",
    options=["disclosures_provided", "disclosures_missing", "not_applicable"],
    evaluation_system_prompt=(
        "You are a regulatory disclosure auditor for a financial services contact "
        "center. Policy DISC-100 requires these disclosures when the topic arises:\n\n"
        "1. INVESTMENT RISK: Investments carry risk, past performance does not guarantee "
        "future results, customer should review the prospectus.\n"
        "2. FEE SCHEDULES: Specific fee amount, when it applies, how to access full "
        "fee schedule.\n"
        "3. RATE TERMS: Fixed or variable, term length, subject to change based on "
        "market conditions and creditworthiness.\n"
        "4. ACCOUNT TERMS: Early withdrawal penalties, minimum balance requirements, "
        "direct customer to full terms.\n"
        "5. PRIVACY NOTICE: Reference privacy policy and customer's opt-out rights "
        "when collecting or discussing personal financial data.\n\n"
        "Classify as 'disclosures_provided' if all applicable disclosures included. "
        "'disclosures_missing' if any required disclosure was omitted. "
        "'not_applicable' if no disclosure-triggering topics arose."
    ),
    evaluation_human_prompt=(
        "Review the following interaction for regulatory disclosure compliance.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Classification options: {evaluation_options}\n\n"
        "Identify which disclosure categories apply, check whether each was provided, "
        "and state your classification."
    ),
    reeval_system_prompt=(
        "You are a senior regulatory auditor reviewing a borderline case. Apply "
        "Policy DISC-100 strictly. The disclosure obligation is triggered by the "
        "topic being discussed, not by the agent giving advice. Partial disclosures "
        "count as 'disclosures_missing'."
    ),
    reeval_human_prompt=(
        "A previous auditor was uncertain about this interaction.\n\n"
        "Customer message:\n{query}\n\n"
        "Agent response:\n{response}\n\n"
        "Policy context:\n{reference}\n\n"
        "Previous auditor notes:\n{memories}\n\n"
        "Classify as 'disclosures_provided', 'disclosures_missing', or 'not_applicable'. "
        "List every applicable disclosure category and whether it was satisfied."
    ),
)

# --- Initialize and log transcripts ---
session = prisma_ai.init(
    project_name="contact-center-qa",
    run_name="weekly-qa-2026-w13",
    evaluators=[response_compliance, escalation_procedure, disclosure_requirements],
    enable_logging=True,
)

logger = session.get_logger()

transcripts = [
    {
        "query": "I'm looking at refinancing my home. What's your current 30-year fixed mortgage rate?",
        "response": (
            "Thank you for calling Meridian Financial, this is David. I'd be happy to help "
            "with refinancing. Could you verify your account number and date of birth? ... "
            "Great, thank you, Mrs. Kowalski. Our current 30-year fixed rate is 6.25%. "
            "That's a really competitive rate right now. Would you like me to start an application?"
        ),
        "reference": (
            "When quoting mortgage rates, disclose: fixed or variable, rate lock period, "
            "subject to change based on market conditions and creditworthiness, direct to "
            "full rate sheet. Per Policy DISC-100."
        ),
    },
    {
        "query": (
            "There are three charges on my account I didn't make — $800 at an electronics "
            "store, $1,200 at a jewelry shop, and $450 at a gas station. I want to file a "
            "formal complaint about how long it took your fraud team to catch this."
        ),
        "response": (
            "Good morning, this is Angela with Meridian Financial. Let me verify your "
            "identity: account number and last four of your SSN? ... Thank you, Mr. Yamamoto. "
            "I've flagged all three charges and placed a temporary hold. Regarding your "
            "complaint, I'm escalating to our Fraud Resolution Supervisor team. Your case "
            "reference is FRD-2026-4471. A supervisor will call within 24 hours."
        ),
        "reference": (
            "Unauthorized charges: freeze card, flag transactions. Formal complaint: "
            "escalate per Policy ESC-200."
        ),
    },
    {
        "query": "I'd like to transfer $3,500 from my checking to my savings.",
        "response": (
            "Absolutely, I can take care of that. Transferring $3,500 now. The transfer "
            "is complete — you'll see the updated balance within the hour. Anything else?"
        ),
        "reference": "Internal transfers require identity verification per Policy FIN-302.",
    },
    {
        "query": (
            "My 401(k) rollover is in a money market fund. Should I move it into your "
            "S&P 500 index fund? What's it been returning?"
        ),
        "response": (
            "Hi, this is Marcus with Meridian Financial. Account number and date of birth? "
            "... Thank you, Ms. Reeves. Our S&P 500 fund returned 18.3% last year with a "
            "10-year average of 11.7%. Moving your rollover there could improve your growth. "
            "I can process the reallocation now."
        ),
        "reference": (
            "Investment discussion requires disclosure: risk of loss, past performance "
            "disclaimer, review prospectus. Per Policy DISC-100."
        ),
    },
    {
        "query": "I'm moving to another bank. I need to close my savings account today.",
        "response": (
            "Good afternoon, this is Rachel with Meridian Financial. Account number and "
            "mailing address? ... Thank you, Mr. Okafor. Your account has a $75 early closure "
            "fee (opened less than 12 months ago). Full terms at meridianfinancial.com/terms. "
            "Remaining $2,065.25 sent by check in 5-7 business days. Confirmation email within "
            "the hour. Anything else?"
        ),
        "reference": "Account closure: verify identity, disclose penalties and terms per Policy DISC-100.",
    },
    {
        "query": (
            "This is the third time I've called about the $35 overdraft fee. The first agent "
            "said it would be reversed, the second couldn't find notes. I'm filing a complaint "
            "with the CFPB if this isn't resolved."
        ),
        "response": (
            "Hi, this is Kevin with Meridian Financial. Account number and date of birth? ... "
            "Thank you, Ms. Hernandez. I see the $35 fee from January 12th. I'm reversing it "
            "now — credit within 24 hours. I've added a note to your file. Anything else?"
        ),
        "reference": (
            "Third call + CFPB mention: escalate per Policy ESC-200. Fee reversal alone "
            "does not satisfy escalation requirements."
        ),
    },
    {
        "query": "I need to wire $25,000 to my daughter's account at Chase for a down payment.",
        "response": (
            "Thank you for calling Meridian Financial, this is Priya. Account number and "
            "last four SSN? ... Thank you, Mr. Chen. Wire transfers over $10,000 require our "
            "Wire Transfer Specialist team. Reference number WIR-2026-0892. A specialist will "
            "be on the line within 5 minutes."
        ),
        "reference": "Wire over $10k: exceeds authority, escalate per Policy ESC-200.",
    },
    {
        "query": "What's the interest rate on your high-yield savings account?",
        "response": (
            "Good morning, this is Tanya with Meridian Financial. Our high-yield savings is "
            "currently 4.15% APY with no minimum balance to open. Want me to start an application?"
        ),
        "reference": (
            "Savings rate inquiry: disclose variable rate, subject to change, minimum balance "
            "for APY tier, direct to full terms. Per Policy DISC-100."
        ),
    },
    {
        "query": "I missed my credit card payment due yesterday. Can I pay now and avoid the late fee?",
        "response": (
            "Thank you for calling Meridian Financial, this is Derek. Account number and date "
            "of birth? ... Thank you, Ms. Tanaka. Payment of $412.33 processed. The $29 late fee "
            "was already assessed — details and autopay setup at meridianfinancial.com/terms. "
            "Summary: $412.33 posted today, $29 fee applied. Anything else?"
        ),
        "reference": "Late payment: process immediately, disclose fee, direct to fee schedule per Policy DISC-100.",
    },
    {
        "query": (
            "I'm calling about my mother's account — she's 82 and I have POA. There are "
            "several large withdrawals she doesn't remember. Her neighbor has been driving her "
            "to the bank. I'm worried someone is taking advantage of her."
        ),
        "response": (
            "Hi, this is Laura with Meridian Financial. Account number and your date of birth? "
            "... Thank you, Ms. Williams. I see the withdrawals. I've placed a $500 alert on the "
            "account and recommend changing the PIN. Would you like me to do that? Anything else?"
        ),
        "reference": (
            "Suspected elder abuse: mandatory escalation per Policy ESC-200. Alert alone "
            "does NOT satisfy escalation requirement. SAR may be required."
        ),
    },
    {
        "query": "I just moved and need to update my mailing address.",
        "response": (
            "Good afternoon, this is Sam with Meridian Financial. Account number and last four "
            "SSN? ... Thank you, Mr. Petrov. Updated to 742 Evergreen Terrace. Per our privacy "
            "policy, your info is protected — review or opt out at meridianfinancial.com/privacy. "
            "Confirmation letter in 3-5 business days. Anything else?"
        ),
        "reference": "Address change: verify identity, reference privacy policy per Policy DISC-100.",
    },
    {
        "query": "I need to change the beneficiary on my life insurance policy from my ex-spouse to my sister.",
        "response": (
            "Thank you for calling Meridian Financial, this is Natalie. Policy number and date "
            "of birth? ... Thank you, Mr. Goodwin. Beneficiary changes go through Policy Services. "
            "Reference number BEN-2026-1134. A specialist will call within 48 hours. Current "
            "designation remains in effect until then. Anything else?"
        ),
        "reference": "Beneficiary change: exceeds authority, escalate per Policy ESC-200.",
    },
]

for transcript in transcripts:
    logger.log(
        query=transcript["query"],
        response=transcript["response"],
        reference=transcript["reference"],
    )
    print(f"Logged: {transcript['query'][:60]}...")

session.end()
print("Session complete. View results in the Prisma dashboard.")
```

## Next Steps

You validated 12 contact center interactions across three policy metrics and identified compliance gaps in escalation handling and regulatory disclosures that a single metric would have missed. Next: learn how to build policy metrics for any industry.

- [Building Policy-Based Metrics](policy-based-metrics.md) — encode your organization's rules as metrics across any industry
- [Human Review & Evaluation Memory](human-review.md) — expert feedback that makes your policy metrics smarter over time
