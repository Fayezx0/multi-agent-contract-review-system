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

## Architecture

Seven agents, each with a single responsibility, coordinated by an orchestrator that sits above the pipeline and runs two confidence-gated retry loops:

```
                    OrchestratorAgent
                  (coordinates everything below)
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ContractParser → PolicyRetrieval → ComplianceJudge → RiskScoring → ContractRewriter
   (Agent 1)          (Agent 2)          (Agent 3)        (Agent 4)      (Agent 5)
                                              │                              │
                                    low confidence?                         ▼
                                       │       │                      ReviewerAgent
                                      Yes      No                       (Agent 7)
                                       │       │                              │
                              Orchestrator     │                    ┌────────┴────────┐
                              re-judges    ─────┘                Passed             Failed
                              (back to                              │                  │
                               Agent 3)                              ▼                  │
                                                              Final Report     Orchestrator re-rewrites
                                                           (assembled by         (back to Agent 5)
                                                              Agent 6)
```

The orchestrator isn't just the last step in a chain — it's the controller sitting above all six other agents. It decides the execution order, inspects the confidence/pass-fail signals each agent reports back, and routes work back to Agent 3 or Agent 5 when a retry is warranted, before finally assembling everything into the report.

| Agent | Role |
|---|---|
| **ContractParserAgent** | Splits raw, unstructured contract text into individual clauses using an LLM, with a regex-based fallback if parsing fails |
| **PolicyRetrievalAgent** | Wraps a ChromaDB vector store of corporate policies; retrieves the most relevant policies per clause and re-ranks them by severity (Critical → High → Medium) and semantic distance |
| **ComplianceJudgeAgent** | Evaluates each clause against its ranked policies and returns a structured verdict: Compliant / Non-Compliant, violation reason, suggested fix, and a confidence score |
| **RiskScoringAgent** | Aggregates every clause verdict into a single 0–100 risk score for the contract, weighted by violation severity and judge confidence, mapped to Low / Medium / High / Critical |
| **ContractRewriterAgent** | Rewrites only the Non-Compliant clauses into compliant versions, grounded in the judge's own violation reason and suggested fix; compliant clauses are carried over untouched |
| **ReviewerAgent** | An independent QA pass — re-checks every rewritten clause against the original policy with a skeptical eye, before it's allowed into the final contract |
| **OrchestratorAgent** | Sits above all six other agents as the controller — decides execution order, decides when a verdict needs a second opinion or a rewrite needs another attempt, and assembles the final audit report |

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
- **Runtime:** Python 3.10+, designed to run in Google Colab

## Observability, not just outputs

Every LLM call — including re-judge and re-rewrite attempts — is wrapped in a `time.perf_counter()` timer and counted with `tiktoken` *before* the response is even parsed, so metrics are captured even when the model returns malformed JSON. Every agent that depends on JSON output has a layered fallback: direct parse → regex extraction → a safe sentinel result. Nothing in the pipeline throws and stops the whole run — a bad clause degrades gracefully into a flagged "Parse Error" row, and a fix that can't be verified gets labeled for human review, instead of crashing the notebook or shipping silently.

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

## Possible next steps

- Swap the simulated S3 client for a real one and point ChromaDB at a hosted cluster instead of in-memory
- Surface clauses flagged `NEEDS HUMAN REVIEW` in an actual review queue/UI instead of just a console label
- Track risk scores over time per vendor to flag drift in contract quality
- Let `ReviewerAgent` choose between multiple tools (e.g. a live regulatory lookup for GDPR-flagged clauses) instead of relying on the same static policy store the judge used

## License

MIT — use it, fork it, break it, learn from it.
