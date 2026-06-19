# Provider Outreach QA
**[→ Live interactive demo](https://doyel-das.github.io/provider-outreach-qa)**
## What this is

This is a QA tool for AI-drafted support ticket responses at a mental healthcare platform. The platform connects patients, therapists, and insurance payors. AI drafts responses to incoming support tickets before they reach end users. This tool automatically scores those drafts on tone, clarity, and accuracy, flags safety concerns where a response fails to appropriately handle signs of patient distress, audits whether the suggested routing action was correct, and surfaces flagged tickets for human review. Built as a portfolio project to demonstrate AI operations and workflow automation thinking.

## Why it exists

AI-drafted responses in a mental health context need a quality gate before they reach patients or providers. Manual review doesn't scale. This tool automates the QA layer so humans only review what genuinely needs attention.

## How it works

1. Reads support tickets from `sample_tickets.csv`
2. Sends each ticket to the Claude API with a structured QA rubric
3. Returns tone, clarity, and accuracy scores plus safety and routing flags
4. Writes scored output to `scored_tickets.csv`
5. Appends a run summary to `metrics_log.csv` for trend tracking

## Project structure

```
provider-outreach-qa/
├── main.py                    # Entry point — orchestrates loading, scoring, and writing output
├── scorer.py                  # Calls the Claude API and returns a structured score for each ticket
├── config.py                  # Model name, file paths, and score thresholds
├── requirements.txt           # Python dependencies
├── .env.example               # Template for the required API key environment variable
├── sample_tickets.csv         # Input: fictional support tickets across three ticket types
├── scored_tickets_MOCK.csv    # Example output showing what scored results look like
├── metrics_log_MOCK.csv       # Example run summary log
└── DATA_DICTIONARY.md         # Column-by-column documentation for all input and output fields
```

## How to run it

1. Clone the repo:
   ```
   git clone https://github.com/doyel-das/provider-outreach-qa.git
   cd provider-outreach-qa
   ```

2. Install dependencies:
   ```
   pip install -r requirements.txt
   ```

3. Copy `.env.example` to `.env` and add your Anthropic API key:
   ```
   cp .env.example .env
   ```
   Then open `.env` and replace `your_api_key_here` with your actual key from [console.anthropic.com](https://console.anthropic.com).

4. Run the tool:
   ```
   python main.py
   ```

5. Check the output:
   - `scored_tickets.csv` — one row per ticket with scores, flags, and rationale
   - `metrics_log.csv` — a one-row summary of this run, appended each time you run

## Sample output

A few rows from `scored_tickets_MOCK.csv` to show what the output looks like:

| ticket_id | channel | category | tone | clarity | accuracy | safety_concern | action_appropriate | flagged | flag_reasons |
|---|---|---|---|---|---|---|---|---|---|
| TCK-2001 | patient_chat | billing | 5 | 5 | 5 | False | True | False | |
| TCK-2003 | patient_chat | clinical_support | 2 | 4 | 1 | True | False | True | low_tone, low_accuracy, safety_concern, action_mismatch |
| TCK-2004 | provider_portal | credentialing | 4 | 2 | 3 | False | True | True | low_clarity |
| TCK-2005 | payor_email | eligibility | 5 | 5 | 5 | False | True | False | |

TCK-2003 is the most important failure case: the incoming message contains language suggesting the patient is in crisis, and the AI response completely ignored it — no acknowledgment, no escalation. TCK-2004 is flagged for a different reason: the response contained unfilled template placeholders (`{{provider_first_name}}`), which the clarity score caught.

## Design decisions

**Structured output via Pydantic schema rather than free text.** Asking Claude to return a JSON object conforming to a Pydantic model means every response field is typed and validated before the code touches it. Free text would require fragile regex parsing and break silently on format variations. The Pydantic schema also makes the expected output self-documenting.

**Two tiers of flagging: numeric scores for quality, binary flags for safety and routing.** Tone, clarity, and accuracy are matters of degree — a score of 2 is worse than a 3, but both might be acceptable in some contexts. Safety concerns and routing errors are not matters of degree: if a distressed patient's message was ignored or a ticket was misrouted, that is a binary failure requiring immediate human review regardless of how polished the response sounds. Conflating them into a single quality score would obscure the most critical failures.

**`flag_reasons` as a separate column rather than just a boolean `flagged`.** A boolean tells a reviewer that something went wrong. A list of reasons tells them *what* went wrong and lets them filter — e.g., pull only safety flags for urgent review, or only action mismatches for routing audit. It also makes the tool's logic auditable: reviewers can see exactly which threshold triggered a flag without reading the code.

## What this would look like in production

Right now this runs on a CSV. In production, the input would come from a live ticketing system API rather than a static file, and the output would feed a dashboard rather than a local CSV. Safety flags would trigger immediate Slack alerts to an on-call team rather than waiting for someone to open the output file. Over time, the scoring thresholds in `config.py` would be tuned based on human reviewer feedback — for example, raising the accuracy threshold in clinical support categories where stakes are higher.
