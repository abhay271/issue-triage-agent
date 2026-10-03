# Architecture

## Overview
The agent predicts labels for GitHub issues in **shadow mode**: it never writes to GitHub. It learns from maintainers through a **correction memory**. Every issue a human labels is stored. For each new issue, the most similar past issues and their human labels are shown to an LLM as examples.

An **autonomy controller** tracks how often the agent gets every label right on an issue. It promotes the agent from suggestion mode to autonomous mode once that is reliably high, and demotes it if spot checks fail.

The core deliverable is a **backtest**. It replays two years of past issues from pandas-dev/pandas and astral-sh/ruff in date order and produces two outputs:
- a learning curve (accuracy vs human decisions seen);
- the point at which the agent would have earned autonomy.

## Data flow
```
GitHub REST + GraphQL (read-only)
        │  collect
        ▼
SQLite: issues, label_events ──reconstruct──▶ issue-as-opened + ground truth + revealed_at
        │                                              │
        │                                        embed (bge-small, GPU)
        ▼                                              ▼
Replay engine: event queue ordered by time
   OPEN(issue)    → Predictor.predict(issue, now) → log prediction
   REVEAL(issue)  → score → autonomy controller → Predictor.observe() → memory
        │
        ▼
results/ (curves, per-label, tables)  ──▶  Streamlit dashboard
```

## Components
| Module | Responsibility |
|---|---|
| `config.py` | Secrets from `.env`, per-repo settings from `repos.yaml` |
| `github/client.py` | Read-only HTTP client: pagination, rate limits, retries, GraphQL |
| `github/collect.py` | Downloads issues, label/rename events and edit timestamps into SQLite |
| `github/reconstruct.py` | Rebuilds each issue as it was when opened, and derives ground truth. This is the leakage boundary. |
| `store/db.py` | SQLite schema and helpers |
| `store/memory.py` | ChromaDB correction memory, queried only for items revealed before a given time |
| `embed.py` | Local `bge-small-en-v1.5` embeddings, cached to disk |
| `llm/` | Groq (final runs) and Ollama (development) clients, response cache, prompt template |
| `predict/` | Predictors: MajorityLabel, KNNVote, ZeroShotLLM, MemoryLLM, LogReg |
| `eval/scoring.py` | Per-issue, rolling and per-label metrics |
| `eval/autonomy.py` | Suggestion/autonomous state machine with audits and demotion |
| `eval/replay.py` | Event-driven backtest |
| `eval/report.py` | Writes result tables and curves to `results/` |
| `dashboard/app.py` | Streamlit UI over `results/` and SQLite |
| `live/` | Stretch: live polling and resolution via GitHub Actions |

## Data model (SQLite)
| Table | Contents |
|---|---|
| `issues` | Repo, number, `title_open`, `body_open`, author, `created_at`, `last_edited_at`, template labels |
| `label_events` | Label, action (labeled/unlabeled), actor, actor type, timestamp |
| `ground_truth` | Final human label set per issue, `revealed_at`, and whether each label was `first_session` or `later` |
| `predictions` | Run, predictor, applied labels, confidences, cited issues, abstain flag, mode, audited flag |
| `autonomy_log` | State transitions and the rolling and audited metrics at each step |
| `runs` | Run config and git SHA |
| `llm_cache` | Prompt hash → response |

## Key rules

### Issue as opened
- The title is restored from rename history.
- Issues whose body was edited more than 10 minutes after creation are dropped.
- Comments are never used.

### Ground truth
- Human (`User`) labels applied within 30 days of opening, net of removals.
- Excluded:
  - template labels
  - denylisted status labels
  - labels with fewer than 20 human applications
- Issues with no human label are not scored.

### Template labels
Labels the author selects on the issue form (e.g. pandas `Bug`) are given to the agent as input, never used as targets.

### No lookahead
- Memory is filled only on `REVEAL` events, in time order.
- Every memory query also filters on `revealed_at < now`.

### Decisions and metrics
- A prediction either applies labels (those with confidence ≥ `tau_apply`) or abstains.
- An issue is **clean** when at least one label was applied and none were wrong.
- Coverage is the share of issues not abstained.
- Learning curves use micro precision, recall and F1 over a rolling window of 100.

### Autonomy
- **Promote** at a clean rate ≥ 0.90 over the last 100 decisions, with coverage ≥ 0.5.
- **When autonomous:**
  - 10% of decisions are audited, and only audits feed the controller;
  - the headline run lets memory learn only from audited issues.
- **Demote** when the audited clean rate over the last 30 audits falls below 0.80, with a 50-decision cooldown.
- Thresholds are fixed before the final run.

### Tuning
`tau_apply` and KNNVote τ are tuned only on a period before each backtest window.

## Repos and windows
| Repo | Tuning window | Backtest window | Approx. issues |
|---|---|---|---|
| pandas-dev/pandas | 2023-07 → 2023-12 | 2024-01 → 2025-12 | ~2,200 |
| astral-sh/ruff | 2024-07 → 2024-12 | 2025-01 → 2025-12 | ~2,000 |

## Tech stack
| Area | Choice |
|---|---|
| Language | Python 3.13, `uv` |
| Storage | SQLite, ChromaDB |
| Embeddings | `sentence-transformers` + `BAAI/bge-small-en-v1.5` on an RTX 4050 (6 GB) |
| LLM | Groq `gpt-oss-120b` (final runs), Ollama 7–8B (development) |
| Data source | GitHub REST + GraphQL with a fine-grained read-only token |
| CLI | Typer |
| UI | Streamlit, deployed to Hugging Face Spaces |
| Tests | pytest, respx (no network in tests) |

## LLM budget
- Groq's free tier caps daily requests, so every call is cached and runs resume where they stopped.
- ZeroShotLLM runs on a fixed 500-issue sample per repo, since it has no learning curve.
- MemoryLLM runs on the full stream.
- Development iterations use Ollama or a 300-issue subsample.
