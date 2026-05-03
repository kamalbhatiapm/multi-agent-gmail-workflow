# Multi-Agent Gmail Workflow — Contract Analysis & Key Term Extraction

An **n8n** automation workflow that monitors Gmail for incoming contract emails, extracts PDF attachments, and uses a multi-agent AI system to analyze contracts — performing key term extraction and/or risk analysis depending on the sender's intent.

## How It Works

### High-Level Flow

```
Gmail Trigger → Fetch Attachments → Extract PDF Text → Orchestrator Agent → Format HTML Email → Reply via Gmail
```

1. **Gmail Trigger** — Polls for new emails every minute from a specified sender.
2. **Fetch Attachments** — Retrieves the full email along with any PDF attachments.
3. **Extract PDF Text** — Extracts raw text content from the first PDF attachment.
4. **Orchestrator Agent** — An AI agent that detects intent and contract type, then delegates work to specialized sub-agents in the correct order.
5. **Format HTML Email** — Transforms the agent's output into a styled HTML report.
6. **Send Results** — Replies to the original sender with the formatted analysis.

---

## Multi-Agent Architecture

The workflow uses a **hierarchical multi-agent design** where a central Orchestrator delegates tasks to three specialized agents, all sharing context through a buffer window memory.

### Orchestrator Agent

The brain of the system. It performs two detection steps before delegating:

**Step 1 — Intent Detection** (from the email body):
| Keywords | Detected Intent |
|---|---|
| "key terms", "extract terms", "list terms", "summarize terms" | `key_term_extraction` |
| "risk", "review", "analyze", "red flags", "should I sign" | `risk_analysis` |

**Step 2 — Contract Type Detection** (from the contract text):
| Keywords | Detected Type |
|---|---|
| "NDA", "Non-Disclosure Agreement", "confidential information" | `NDA` |
| "MSA", "Master Services Agreement", "statement of work" | `MSA` |

**Step 3 — Agent Delegation:**

- **Key Term Extraction** intent:
  `Playbook Agent` → `Key Term Extraction Agent`

- **Risk Analysis** intent:
  `Playbook Agent` → `Key Term Extraction Agent` → `Risk Agent`

---

### Specialized Agents

#### 1. Contract Playbook Agent
- **Role:** Fetches the correct playbook rules from a **Supabase** database based on contract type.
- **Always runs first** — loads the rule definitions that guide downstream agents.
- **Output:** A structured summary of rules stored in shared memory (term names, labels, descriptions, risk weights).

#### 2. Key Term Extraction Agent
- **Role:** Extracts key terms from the contract text using the playbook rules as a guide.
- **Reads from shared memory** — contract text + playbook rules.
- **Output format:**
  ```
  Contract Type: NDA
  Parties: Party A & Party B
  Effective Date: January 1, 2025

  KEY TERMS:
  - Governing Law: State of Delaware
  - Term Duration: 2 years
  - Confidentiality Scope: All written materials marked "Confidential"
  ```

#### 3. Contract Risk Agent
- **Role:** Performs comprehensive risk analysis using detailed scoring criteria.
- **Only runs for `risk_analysis` intent** — called last, after key terms are extracted.
- **Scores risks** on a 1–10 scale across contract-type-specific criteria.
- **Saves results** to a `contract_risk_results` table in Supabase.

**NDA Risk Criteria:** Governing Law, Term Duration, Confidentiality Scope, Permitted Disclosures, Return/Destruction, Remedy Clause

**MSA Risk Criteria:** Payment Terms, Liability Cap, IP Ownership, Termination Rights, Indemnification, SLA Terms

Each criterion has four severity levels (LOW → MEDIUM → HIGH → CRITICAL) with specific conditions.

**Output format:**
```
--- CONTRACT RISK ANALYSIS REPORT ---
Contract Type: NDA
Overall Risk Score: 6/10
Overall Risk Level: HIGH

EXECUTIVE SUMMARY: ...

RISK FINDINGS:
- Risk #1
  Term: Governing Law
  Risk Level: MEDIUM
  Finding: ...
  Contract Language: "..."
  Recommendation: ...

MISSING CLAUSES: ...
FAVORABLE TERMS: ...

OVERALL RECOMMENDATION: SIGN WITH MODIFICATIONS
--- END RISK ANALYSIS REPORT ---
```

---

## Shared Memory

All agents share a **Buffer Window Memory** (context window of 90 messages) that allows them to pass data without explicit parameter passing. The orchestrator writes the contract text, the playbook agent writes the rules, and each subsequent agent reads everything accumulated so far.

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow Engine | [n8n](https://n8n.io) |
| LLM | OpenAI (GPT) |
| Database | [Supabase](https://supabase.com) |
| Email | Gmail (OAuth2) |
| PDF Parsing | n8n Extract from File node |

### Database Tables (Supabase)

- **`contract_playbook`** — Stores playbook rules per contract type (key term names, labels, descriptions, risk weights).
- **`contract_risk_results`** — Stores completed risk analysis results (score, level, summary, recommendation, status).

---

## Setup

### Prerequisites

- An **n8n** instance (self-hosted or cloud)
- **Gmail** account with OAuth2 credentials configured
- **OpenAI** API key
- **Supabase** project with the required tables

### Installation

1. **Import the workflow** — In n8n, go to *Workflows → Import from File* and select `multiAgent-gmailWorkflow-keyTermExtraction.json`.

2. **Configure credentials:**
   - **Gmail OAuth2** — Connect your Gmail account.
   - **OpenAI API** — Add your API key.
   - **Supabase** — Add your project URL and API key.

3. **Set up Supabase tables:**

   **`contract_playbook`** table:
   | Column | Type | Description |
   |---|---|---|
   | `contract_type` | text | NDA, MSA, etc. |
   | `key_term_name` | text | Internal term identifier |
   | `key_term_label` | text | Display label |
   | `key_term_description` | text | What the term means |
   | `risk_weight` | numeric | Weight for risk scoring |

   **`contract_risk_results`** table:
   | Column | Type | Description |
   |---|---|---|
   | `session_id` | text | Execution tracking ID |
   | `sender_email` | text | Who sent the contract |
   | `risk_score` | numeric | Overall score (1–10) |
   | `risk_level` | text | LOW / MEDIUM / HIGH / CRITICAL |
   | `risk_summary` | text | Full analysis report |
   | `recommendation` | text | SIGN / SIGN WITH MODIFICATIONS / DO NOT SIGN |
   | `status` | text | Processing status |

4. **Update the Gmail filter** — Edit the Gmail Trigger node to filter emails from your desired sender address.

5. **Activate the workflow** — Toggle the workflow to active. It will begin polling for new emails every minute.

---

## Usage

1. Send an email **with a PDF contract attachment** to the monitored Gmail account.
2. Include your intent in the email body:
   - *"Please extract the key terms from the attached contract"* → Key term extraction only
   - *"Can you review this contract for risks?"* → Full risk analysis
3. The workflow will process the contract and **reply with a formatted HTML email** containing the results.

---

## Architecture Diagram

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Gmail       │────▶│ Get Email with   │────▶│ Extract PDF      │
│ Trigger     │     │ Attachments      │     │ Text             │
└─────────────┘     └──────────────────┘     └────────┬─────────┘
                                                      │
                                                      ▼
                                             ┌──────────────────┐
                                             │  Orchestrator    │
                                             │  Agent           │
                                             └──┬───┬───┬───────┘
                                                │   │   │
                              ┌─────────────────┘   │   └─────────────────┐
                              ▼                     ▼                     ▼
                    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
                    │ Playbook Agent   │  │ Key Term Agent   │  │ Risk Agent       │
                    │ (runs 1st)       │  │ (runs 2nd)       │  │ (runs 3rd*)      │
                    └────────┬─────────┘  └──────────────────┘  └────────┬─────────┘
                             │                                           │
                             ▼                                           ▼
                    ┌──────────────────┐                        ┌──────────────────┐
                    │ Supabase         │                        │ Supabase         │
                    │ (playbook rules) │                        │ (risk results)   │
                    └──────────────────┘                        └──────────────────┘

                        * Risk Agent only runs for risk_analysis intent

                              ┌──────────────────┐     ┌──────────────────┐
                              │ Format HTML      │────▶│ Send Email       │
                              │ Email            │     │ Reply            │
                              └──────────────────┘     └──────────────────┘
```

---

## License

This project is open source. Feel free to use and modify it for your own contract analysis needs.
