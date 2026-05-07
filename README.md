# Three-Agent Research System

A multi-agent research system that fans out a single question to three heterogeneous LLMs in parallel, aggregates their independent answers into a consensus analysis, optionally runs a verification pass on disputed claims, and synthesizes a final confidence-weighted response.

All models are accessed through **Amazon Bedrock**. The orchestration layer is built with the **[Strands Agents SDK](https://strandsagents.com)** Graph pattern.

---

## Design Background

### Multi-Agent Orchestration Patterns (primary reference)

The architecture in this project is based on the **Orchestrator-Worker pattern with Fan-out/Fan-in execution and a verification loop**, as described in `Multi-Agent Orchestration Patterns.pdf` (included in this repository).

Key concepts from the document applied here:

| Concept | Implementation |
|---------|---------------|
| **Orchestrator-Worker pattern** | Central graph routes queries to three independent worker agents, collects results, and drives the verification pipeline |
| **Fan-out / Fan-in (scatter-gather)** | All three workers receive the same query simultaneously; results are collected only after all three complete (AND semantics) |
| **Council Mode consensus** | Three heterogeneous models are queried in parallel; their outputs are cross-validated through structured agreement analysis |
| **Tiered verification loop** | Fast consensus check → targeted source validation on disputed claims → full synthesis |
| **Confidence-weighted synthesis** | High-consensus facts stated directly; medium-confidence claims qualified; disputed claims flagged |
| **Model diversity as a feature** | Disagreements between agents are treated as investigation signals, not noise |

The PDF cites research showing Council Mode reduces hallucinations by approximately **35.9%** compared to single-model responses — the core motivation for using three different model families rather than one model called three times.

### Mixture of Agents (MoA) — Together AI

The concept in this project is closely related to **Mixture of Agents (MoA)**, introduced by Together AI: [https://www.together.ai/blog/together-moa](https://www.together.ai/blog/together-moa)

MoA makes the same foundational observation: *LLMs produce better outputs when they can reference responses from other models*, even weaker ones. The architecture maps almost directly:

| MoA Term | This Project |
|----------|-------------|
| Proposers | `claude_worker`, `nova_worker`, `mistral_worker` |
| Aggregator | `aggregator` agent (consensus analysis) |
| Layered refinement | Verification loop (`verifier` → `synthesizer`) |

The difference is emphasis: MoA focuses on **iterative layering** (aggregator outputs feed back as inputs for progressive refinement), while this project focuses on **structured disagreement analysis** — the aggregator explicitly maps which claims are agreed upon vs. disputed, and routes disputed claims to a dedicated verifier before synthesis. Both approaches exploit model diversity through collaborative synthesis rather than simple voting.

If you are evaluating these approaches, Together AI's MoA paper is worth reading alongside the orchestration patterns document — they arrive at complementary conclusions from different starting points.

---

## Architecture

```
              [User Query]
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
 [Claude]       [Nova]       [Mistral]     ← Fan-out: parallel workers
       └────────────┼────────────┘
                    │                      ← Fan-in: waits for ALL three
             [Aggregator]                  ← Consensus analysis
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
   (CONSENSUS)           (PARTIAL/DISPUTED)
         │                     │
         │                [Verifier]       ← Scrutinises disputed claims
         │                     │
         └──────────┬──────────┘
                    ▼
             [Synthesizer]                 ← Confidence-weighted final answer
```

---

## Models (Amazon Bedrock, us-east-1)

| Role | Model ID | Provider |
|------|----------|----------|
| Worker 1 | `us.anthropic.claude-opus-4-7` | Anthropic |
| Worker 2 | `us.amazon.nova-pro-v1:0` | Amazon |
| Worker 3 | `mistral.mistral-large-3-675b-instruct` | Mistral AI |
| Aggregator / Verifier / Synthesizer | `us.anthropic.claude-opus-4-7` | Anthropic |

Worker models were chosen to maximise architectural and training diversity — the system's hallucination-reduction benefit scales with how different the models are from each other.

---

## Tools

Each worker agent has access to two tools for grounding answers in retrieved evidence:

- **`serpapi_search`** — Google search via [SerpApi](https://serpapi.com); returns titles, URLs, and snippets
- **`web_fetch`** — Fetches and strips any URL to plain text (scripts/styles removed, capped at 8 000 chars)

---

## Getting Started

### Prerequisites

- Python 3.10+
- An AWS account with Amazon Bedrock access enabled for all three worker model IDs above
- A [SerpApi](https://serpapi.com) API key

### Installation

```bash
git clone <repo-url>
cd three-research-agents
pip install -r requirements.txt
```

### Configuration

```bash
cp env.example .env
# Edit .env and fill in your credentials
```

`.env` keys:

```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
SERPAPI_API_KEY=...
```

### Running

Open `three_research_agents.ipynb` in JupyterLab or VS Code and run cells top to bottom.

Change the `QUERY` variable in **Section 8** to ask any research question:

```python
QUERY = "What are the implications of quantum computing for current cryptography standards?"
```

Or use the convenience function defined in **Section 12**:

```python
answer = research("Your question here", verbose=True)
print(answer)
```

---

## Project Structure

```
three-research-agents/
├── three_research_agents.ipynb   # Main notebook — all code and explanations
├── requirements.txt
├── env.example
├── Multi-Agent Orchestration Patterns.pdf
└── README.md
```

---

## Known Limitations

| Model | Limitation |
|-------|-----------|
| Claude Opus 4.7 | `temperature` and other sampling parameters removed |
| Mistral Large 3 | No cross-region inference profile; uses direct regional endpoint |
| Any Bedrock Llama model | Tool use with streaming not supported — requires `streaming=False` |
| gpt-oss (OpenAI on Bedrock) | After multi-turn tool loops, leaks reasoning chain into text output |

---

## Acknowledgements

- [Strands Agents SDK](https://strandsagents.com) — multi-agent graph orchestration
- [Together AI — Mixture of Agents](https://www.together.ai/blog/together-moa) — foundational research on collaborative LLM synthesis
- `Multi-Agent Orchestration Patterns.pdf` — primary architectural reference for this implementation
