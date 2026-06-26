# Contract Audit AI — Multi-Agent Compliance & Risk Review System

A multi-agent AI pipeline that reads vendor contracts, checks every clause against a corporate policy knowledge base, scores overall contract risk, rewrites non-compliant clauses, and independently verifies its own fixes — with automatic retry loops when an agent is not confident in its answer.

## Project structure

```
contract-audit-ai/
├── README.md
├── LICENSE
├── .gitignore
├── requirements.txt
├── notebooks/
│   └── contract_audit_agent.ipynb
└── data/
    └── sample_contracts/
        ├── acmetech_mutual_nda_v3.txt
        └── dataflow_systems_msa_v1.txt
```

## Why this exists

Manually reviewing vendor contracts against internal policy is slow and inconsistent. This project explores a structured multi-agent pipeline that can:

- Parse messy, unstructured contract text into clean clauses
- Retrieve the right policy for each clause
- Make defensible Compliant / Non-Compliant calls with confidence scores
- Aggregate clause verdicts into one actionable risk score
- Rewrite and verify non-compliant clauses before human review
- Track latency, tokens, and cost for every LLM call

## Architecture

Seven agents, coordinated by an orchestrator with two confidence-gated retry loops:

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
| **ContractParserAgent** | Splits raw contract text into clauses (LLM + regex fallback) |
| **PolicyRetrievalAgent** | Retrieves and severity-ranks policies from ChromaDB |
| **ComplianceJudgeAgent** | Returns Compliant / Non-Compliant verdicts with confidence |
| **RiskScoringAgent** | Aggregates verdicts into a 0–100 risk score |
| **ContractRewriterAgent** | Rewrites non-compliant clauses into compliant versions |
| **ReviewerAgent** | Independently verifies every rewrite before acceptance |
| **OrchestratorAgent** | Drives execution, retry loops, and final report assembly |

## Tech stack

- **LLM:** [Groq](https://groq.com/) — `llama-3.3-70b-versatile`
- **Vector retrieval:** [ChromaDB](https://www.trychroma.com/)
- **Token accounting:** `tiktoken`
- **Reporting:** `pandas`, `matplotlib`
- **Runtime:** Python 3.10+

## Setup

### 1. Clone and install dependencies

```bash
cd contract-audit-ai
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Set your Groq API key

```bash
export GROQ_API_KEY="your-key-here"   # Windows PowerShell: $env:GROQ_API_KEY="your-key-here"
```

Get a free key at [console.groq.com](https://console.groq.com/).

### 3. Run the notebook

```bash
jupyter notebook notebooks/contract_audit_agent.ipynb
```

Run all cells top to bottom. The notebook also works in Google Colab — add `GROQ_API_KEY` as a Colab secret with notebook access enabled.

## Sample contracts

Two vendor contracts in `data/sample_contracts/` contain deliberate policy violations for demo purposes:

| File | Type | Planted issues |
|---|---|---|
| `acmetech_mutual_nda_v3.txt` | Mutual NDA | Short confidentiality term, oversized liability cap, slow breach notification |
| `dataflow_systems_msa_v1.txt` | MSA | Short termination notice, GDPR gap, unlimited liability |

The notebook ships with a simulated S3 client pre-loaded with these contracts. To use the local files directly, point ingestion at `data/sample_contracts/` or swap in real `boto3` S3 calls.

## License

MIT — see [LICENSE](LICENSE).
