# Contract Audit AI — Multi-Agent Compliance & Risk Review System

A multi-agent AI pipeline that reads vendor contracts, checks every clause against a corporate policy knowledge base, scores overall contract risk, rewrites whatever isn't compliant, and independently verifies its own fixes — with automatic retry loops when an agent isn't confident in its own answer.

I built this to explore how far a small team of specialized agents can go when each one owns exactly one job, instead of throwing one giant prompt at an LLM and hoping it does everything at once.

## Why this exists

Manually reviewing vendor contracts against internal policy is slow and inconsistent — different reviewers catch different issues, and there's no standardized way to say "how risky is this contract, really?" I wanted to see if a structured multi-agent pipeline could:

- Parse messy, unstructured contract text into clean clauses
- Retrieve the *right* policy for each clause (not just a generic one)
- Make a defensible Compliant / Non-Compliant call with a reason — and know when it's not confident in that call
- Turn a pile of clause-level verdicts into one number a manager can act on
- Actually fix the problem, not just flag it
- Check its own fix before handing it over, instead of trusting a single LLM pass
- Do all of this with full visibility into latency, tokens, and cost

<<<<<<< HEAD
## Project structure

```
contract-audit-ai/
├── README.md
├── LICENSE
├── .gitignore
├── .gitattributes
├── .env.example
├── requirements.txt
├── notebooks/
│   └── contract_audit_agent.ipynb
└── data/
    └── sample_contracts/
        ├── acmetech_mutual_nda_v3.txt
        └── dataflow_systems_msa_v1.txt
```

> **Note on `data/sample_contracts/`:** the notebook currently ingests these two contracts from an in-memory `SimulatedS3Client` (so it runs with zero setup). The `.txt` files in this folder are the same two contracts in plain-text form, included so the source documents are visible/diffable outside the notebook and so you have something to point a real ingestion step at. To actually read from these files (or from real S3 via `boto3`), swap the body of `SimulatedS3Client.get_contract()` — the rest of the pipeline doesn't need to change.

=======
>>>>>>> 6ee6bc8bc9090027bff2b49ef55790d3a6ad8968
## Architecture

Seven agents, each with a single responsibility, coordinated by an orchestrator that runs two confidence-gated retry loops:

```
ContractParserAgent
        ↓
PolicyRetrievalAgent  ←→  ChromaDB
        ↓
ComplianceJudgeAgent  ←→  Groq LLM
        ↓  ⟲ low confidence → re-judge
RiskScoringAgent
        ↓
ContractRewriterAgent ←→  Groq LLM
        ↓
ReviewerAgent         ←→  Groq LLM
        ↓  ⟲ failed review → re-rewrite
OrchestratorAgent → Final Report
```

| Agent | Role |
|---|---|
| **ContractParserAgent** | Splits raw, unstructured contract text into individual clauses using an LLM, with a regex-based fallback if parsing fails |
| **PolicyRetrievalAgent** | Wraps a ChromaDB vector store of corporate policies; retrieves the most relevant policies per clause and re-ranks them by severity (Critical → High → Medium) and semantic distance |
| **ComplianceJudgeAgent** | Evaluates each clause against its ranked policies and returns a structured verdict: Compliant / Non-Compliant, violation reason, suggested fix, and a confidence score |
| **RiskScoringAgent** | Aggregates every clause verdict into a single 0–100 risk score for the contract, weighted by violation severity and judge confidence, mapped to Low / Medium / High / Critical |
| **ContractRewriterAgent** | Rewrites only the Non-Compliant clauses into compliant versions, grounded in the judge's own violation reason and suggested fix; compliant clauses are carried over untouched |
| **ReviewerAgent** | An independent QA pass — re-checks every rewritten clause against the original policy with a skeptical eye, before it's allowed into the final contract |
| **OrchestratorAgent** | Drives execution order across all six other agents, decides when a verdict needs a second opinion or a rewrite needs another attempt, and assembles the final audit report |

### The two retry loops

This is the part I spent the most time on, because it's what actually makes this a *system* rather than a script with six functions in a row:

- **Low-confidence re-judge** — if `ComplianceJudgeAgent` isn't confident in its own verdict, the orchestrator doesn't just accept it. It sends the clause back with a wider policy search and an explicit note that the first pass was uncertain.
- **Rewrite → verify → retry** — every rewritten clause gets checked by `ReviewerAgent` before it ships. If the reviewer rejects it, the rewriter gets a second attempt with the reviewer's specific objection as feedback. If it still fails, the clause is labeled `NEEDS HUMAN REVIEW` in the output instead of being silently passed off as fixed.

## What it actually does, end to end

1. Pulls a contract from a simulated S3 bucket (drop-in replaceable with real `boto3`)
2. Splits it into clauses
3. For each clause, retrieves and ranks the relevant corporate policy rules from ChromaDB
4. Sends the clause + ranked policies to an LLM and gets back a verdict with a confidence score — low-confidence verdicts automatically get a second look
5. Rolls all verdicts up into one risk score per contract
6. Rewrites the problem clauses into a ready-to-send revised contract
7. Independently re-verifies each rewrite, retrying with feedback if it doesn't pass, and flagging anything that still doesn't pass for a human to check
8. Logs latency, token counts, and USD cost for every single LLM call — including retries
9. Renders a dashboard: compliance breakdown, latency, token usage, cumulative cost, risk score per contract, and severity distribution

## Tech stack

- **LLM inference:** [Groq](https://groq.com/) running `llama-3.3-70b-versatile`
- **Vector retrieval:** [ChromaDB](https://www.trychroma.com/) (in-memory, cosine similarity)
- **Token accounting:** `tiktoken`
- **Data & reporting:** `pandas`, `matplotlib`
<<<<<<< HEAD
- **Runtime:** Python 3.10+

## Setup

### Option A — Google Colab (fastest, zero local setup)

1. Open `notebooks/contract_audit_agent.ipynb` in Google Colab
2. Add a Colab secret named `GROQ_API_KEY` with your [Groq API key](https://console.groq.com/) and enable notebook access
3. `Runtime → Run all` — dependencies install automatically in the first cell

### Option B — Run locally

```bash
git clone <repo-url>
cd contract-audit-ai
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Set your Groq API key (either works — the notebook checks `GROQ_API_KEY` as an environment variable when it's not running in Colab):

```bash
cp .env.example .env        # then edit .env and paste your real key
export GROQ_API_KEY="your-key-here"   # Windows PowerShell: $env:GROQ_API_KEY="your-key-here"
```

> The notebook reads `GROQ_API_KEY` directly from the environment — it does not auto-load `.env` files. `cp .env.example .env` is there so you have a private place to keep the key, but you still need to `export` it (or add `python-dotenv` + `load_dotenv()` yourself if you'd rather not export it manually each session).

Then launch the notebook and run all cells top to bottom:

```bash
jupyter notebook notebooks/contract_audit_agent.ipynb
```
=======
- **Runtime:** Python 3.10+, designed to run in Google Colab
>>>>>>> 6ee6bc8bc9090027bff2b49ef55790d3a6ad8968

## Observability, not just outputs

Every LLM call — including re-judge and re-rewrite attempts — is wrapped in a `time.perf_counter()` timer and counted with `tiktoken` *before* the response is even parsed, so metrics are captured even when the model returns malformed JSON. Every agent that depends on JSON output has a layered fallback: direct parse → regex extraction → a safe sentinel result. Nothing in the pipeline throws and stops the whole run — a bad clause degrades gracefully into a flagged "Parse Error" row, and a fix that can't be verified gets labeled for human review, instead of crashing the notebook or shipping silently.

<<<<<<< HEAD
## Sample contracts

Two sample vendor contracts ship with this repo (see [`data/sample_contracts/`](data/sample_contracts/)), each containing deliberately planted policy violations so the pipeline has something real to catch:

| File | Type | Planted issues |
|---|---|---|
| `acmetech_mutual_nda_v3.txt` | Mutual NDA | Short confidentiality term (1 year vs. 3-year policy minimum), oversized liability cap, slow breach-notification window (30 days vs. GDPR's 72-hour requirement) |
| `dataflow_systems_msa_v1.txt` | Master Services Agreement | Too-short termination notice (7 days vs. 90-day policy minimum), GDPR notification gap (96 hours vs. 72-hour requirement), unlimited liability clause |

These same two contracts are pre-loaded inside the notebook's `SimulatedS3Client` so the pipeline runs end-to-end with zero external setup. Swap that class for real `boto3` S3 calls and the rest of the pipeline doesn't need to change.
=======
## Running it

1. Open the notebook in Google Colab
2. Add a Colab secret named `GROQ_API_KEY` with your [Groq API key](https://console.groq.com/) and enable notebook access
3. `Runtime → Run all`

That's it — no other setup. Dependencies install automatically in the first cell.

## Sample contracts included

Two sample vendor contracts ship with the notebook, each containing deliberately planted policy violations so the pipeline has something real to catch:

- A **Mutual NDA** (weak confidentiality term, an oversized liability cap, a slow breach-notification window)
- A **Master Services Agreement** (a too-short termination notice period, a GDPR notification gap, an unlimited liability clause)

Swap these out for `boto3` calls to a real S3 bucket and the rest of the pipeline doesn't need to change.
>>>>>>> 6ee6bc8bc9090027bff2b49ef55790d3a6ad8968

## Possible next steps

- Swap the simulated S3 client for a real one and point ChromaDB at a hosted cluster instead of in-memory
- Surface clauses flagged `NEEDS HUMAN REVIEW` in an actual review queue/UI instead of just a console label
- Track risk scores over time per vendor to flag drift in contract quality
- Let `ReviewerAgent` choose between multiple tools (e.g. a live regulatory lookup for GDPR-flagged clauses) instead of relying on the same static policy store the judge used

## License

<<<<<<< HEAD
MIT — see [LICENSE](LICENSE). Use it, fork it, break it, learn from it.
=======
MIT — use it, fork it, break it, learn from it.
>>>>>>> 6ee6bc8bc9090027bff2b49ef55790d3a6ad8968
