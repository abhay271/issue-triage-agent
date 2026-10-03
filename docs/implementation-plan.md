# Implementation Plan

This file is the project's todo list. Work top to bottom. Check a task off (`- [x]`) only when all of its required tests pass. Record anything that changed from the plan under **Deviations** at the end of the relevant phase.

For design context see [architecture.md](architecture.md). For the reasoning behind each choice see [decisions.md](decisions.md).

**Goal:** a backtest on pandas-dev/pandas and astral-sh/ruff that shows two things:
1. label accuracy rising as the agent sees more maintainer decisions;
2. the point at which the agent would have earned autonomy.

---

## Phase 0 — Project setup

- [ ] Initialise git repo, `.gitignore` (data/, .env, caches, `*.db`, `.venv/`)
- [ ] `pyproject.toml` managed by `uv`, Python 3.13, `src/triage/` package layout
- [ ] Dependencies:
  - runtime: httpx, pydantic, pydantic-settings, typer, chromadb, sentence-transformers, torch (CUDA), groq, ollama, jinja2, scikit-learn, pandas, streamlit, plotly
  - dev: pytest, respx, ruff
- [ ] `config.py`: settings from `.env` (`GITHUB_TOKEN`, `GROQ_API_KEY`) plus per-repo settings from `repos.yaml`
- [ ] `repos.yaml` with pandas and ruff entries:
  - tuning and backtest windows
  - label denylist and aliases
  - autonomy thresholds
- [ ] `.env.example`
- [ ] Typer CLI entry point `triage` with a `--help` that runs
- [ ] Verify CUDA is visible to torch (`torch.cuda.is_available()`)

**Done when:**
- [ ] `uv run pytest` runs (an empty suite passes)
- [ ] `uv run triage --help` prints the command list

---

## Phase 1 — Data collection and reconstruction

### 1.1 GitHub client (`github/client.py`)
- [ ] Authenticated httpx client with Link-header pagination
- [ ] Rate-limit handling: sleep until `X-RateLimit-Reset` when remaining is low. Back off on 403 secondary limits and on 5xx.
- [ ] GraphQL helper for batched `lastEditedAt` queries

**Tests required** (respx mocks, no network):
- [ ] Pagination follows `Link: rel="next"` until exhausted
- [ ] Client waits and retries when the rate limit is hit
- [ ] 5xx is retried with backoff and gives up after N attempts

### 1.2 Storage (`store/db.py`)
- [ ] SQLite schema: `issues`, `label_events`, `ground_truth`, `predictions`, `autonomy_log`, `runs`, `llm_cache`
- [ ] Typed insert and query helpers. Collection is idempotent (upserts).

**Tests required:**
- [ ] Schema creates on an empty file; re-running collection does not duplicate rows

### 1.3 Collector (`github/collect.py`)
- [ ] Spike: can the repo-wide `/issues/events` endpoint cover the full window? Use it if yes. Otherwise use per-issue `/timeline`.
- [ ] Fetch issues (excluding PRs) for tuning and backtest windows
- [ ] Fetch labeled, unlabeled and renamed events with actor type
- [ ] Fetch `lastEditedAt` for every issue (GraphQL, batched)
- [ ] Resumable per-repo cursor
- [ ] `triage collect --repo <owner/name>` command
- [ ] Run collection for pandas and ruff

### 1.4 Reconstruction and ground truth (`github/reconstruct.py`)
This is the leakage boundary.
- [ ] `title_open`: the `from` of the earliest `renamed` event, else the current title
- [ ] Body edits: drop issues whose body was edited more than 10 minutes after creation, and report the count. If more than 15% are dropped, add original-body recovery via `userContentEdits`.
- [ ] Template labels: labels added by the issue author within 60 s of creation. They are stored as input, never as targets.
- [ ] Ground truth:
  - human (`actor_type == "User"`) non-template labels
  - net of `unlabeled` events
  - applied within 30 days of opening
  - denylisted labels removed
- [ ] `revealed_at` = time of the last counted label event
- [ ] Tag each true label as `first_session` (within 1 h of the first human label) or `later`
- [ ] Issues with no human label: excluded from scoring and counted
- [ ] Label vocabulary: labels with at least 20 human applications. Report the share of human label applications it covers.

**Tests required** (fixture timelines):
- [ ] Renamed title reverts to the original
- [ ] Bot-applied labels are excluded from ground truth
- [ ] Template labels appear as input and never in ground truth
- [ ] Label added then removed is not in ground truth
- [ ] Labels after the 30-day window are ignored
- [ ] Edited-body issues are excluded
- [ ] The reconstructed issue contains no text or label from after `created_at`
- [ ] Rare labels are dropped from both vocabulary and ground truth

### 1.5 Label audit report
- [ ] `triage audit-labels --repo <owner/name>`. It prints:
  - issue count
  - share of issues human-labelled
  - share of labels bot-applied
  - labels per issue
  - vocabulary size and coverage
  - share of issues dropped for edits
  - median time to first label
- [ ] Review the audit for both repos. Adjust denylists and aliases in `repos.yaml`.

### 1.6 Embeddings (`embed.py`)
- [ ] `bge-small-en-v1.5` wrapper that uses CUDA when available and normalises vectors
- [ ] Embed `title_open + body_open[:2000]` for every issue. Cache to `data/embeddings/{repo}.npy` with an id index.

**Tests required:**
- [ ] Output shape is (n, 384) and vectors are unit-norm
- [ ] The cache is reused on a second run (the model is not called)

**Deviations:** _none yet_

---

## Phase 2 — Memory, baselines, scoring, replay (no LLM)

### 2.1 Correction memory (`store/memory.py`)
- [ ] One Chroma collection per repo. Metadata: number, labels, `revealed_at_ts`.
- [ ] `add(issue, labels, revealed_at)`
- [ ] `query(embedding, k, before_ts)`. The time filter is always applied.

**Tests required:**
- [ ] `query` never returns an item with `revealed_at_ts >= before_ts`
- [ ] Returns at most k items, ordered by similarity

### 2.2 Predictor interface and baselines (`predict/`)
- [ ] `Prediction` model: applied labels, per-label confidence, cited issues, abstain flag
- [ ] `Predictor` protocol: `predict(issue, now_ts)` and `observe(issue, labels, ts)`
- [ ] `MajorityLabel`: the most frequent label seen so far
- [ ] `KNNVote`: similarity-weighted vote over the top-k neighbours' labels. Apply a label when its score is at least τ. Abstain when nothing passes.

**Tests required:**
- [ ] MajorityLabel updates as labels are observed
- [ ] KNNVote on a hand-built memory gives the expected labels and abstains when nothing passes τ

### 2.3 Scoring (`eval/scoring.py`)
- [ ] Per issue: precision, recall, exact match, and **clean** (nothing wrong applied, at least one label applied)
- [ ] Rolling window of 100: micro precision, recall and F1, clean rate, coverage, with 95% bootstrap CI
- [ ] Per-label P/R/F1 and support, split by `first_session` and `later`

**Tests required:**
- [ ] Hand-computed P/R/F1 match on small cases
- [ ] Abstained issues count against coverage and are excluded from precision
- [ ] Rolling window drops the oldest entries correctly

### 2.4 Replay engine (`eval/replay.py`)
- [ ] Event queue ordered by timestamp:
  - `OPEN` triggers a prediction
  - `REVEAL` triggers scoring, `observe()` and an autonomy update
- [ ] Cold start (empty memory) by default. `--warm N` optionally seeds N pre-window issues.
- [ ] Writes predictions to SQLite and `results/{repo}/{predictor}/curve.csv` and `per_label.csv`
- [ ] `triage backtest --repo <r> --predictor <p>`

**Tests required:**
- [ ] On a synthetic stream, no prediction can see an issue revealed after its `created_at`
- [ ] An issue opened before another is revealed does not retrieve it, even when the other was opened earlier
- [ ] A full replay of the baselines on pandas completes and writes the CSVs

**Deviations:** _none yet_

---

## Phase 3 — LLM predictors

### 3.1 LLM clients (`llm/`)
- [ ] `LLMClient` protocol: `complete_json(prompt) -> dict`
- [ ] Groq client (`gpt-oss-120b`, JSON mode). Waits on rate limits. Check the current free-tier limits before the final runs.
- [ ] Ollama client (local 7–8B, e.g. `qwen2.5:7b`) for development runs
- [ ] SQLite response cache keyed by `sha256(model + prompt)`. Runs resume from the cache after hitting a daily limit.

**Tests required:**
- [ ] A cache hit makes no client call
- [ ] Invalid JSON triggers exactly one repair retry, then the prediction is recorded as an abstain

### 3.2 Output parsing
- [ ] Pydantic schema: `labels[{name, confidence, evidence}]`, `abstain`, `rationale`
- [ ] Labels outside the vocabulary are dropped. A label is applied only if its confidence is at least `tau_apply`.

**Tests required:**
- [ ] Out-of-vocabulary labels are removed
- [ ] Confidences below `tau_apply` are not applied; if none pass, the prediction is an abstain

### 3.3 Predictors
- [ ] `ZeroShotLLM`: label vocabulary with one-line descriptions, template labels, and the issue
- [ ] `MemoryLLM`: the same, plus the top-k (k=8) retrieved past issues with their human labels
- [ ] Prompt template `llm/prompts/predict.j2`, shared by both

**Tests required:**
- [ ] The rendered MemoryLLM prompt contains only memory items revealed before the issue opened
- [ ] The rendered prompt with k=0 matches the zero-shot prompt

### 3.4 Tuning and full runs
- [ ] Tune `tau_apply` (and the KNNVote τ) on each repo's tuning window only
- [ ] ZeroShotLLM on a fixed random 500 issues per repo (seeded)
- [ ] MemoryLLM on the full backtest window for both repos, first on Ollama or a 300-issue subsample, then on Groq
- [ ] If the learning curve is flat: sweep k ∈ {0, 4, 8, 16} and check how often retrieved neighbours share a label with the issue

### 3.5 LogReg (`predict/logreg.py`)
- [ ] One-vs-rest logistic regression on embedding plus template-label one-hot, retrained every 100 revealed issues

**Tests required:**
- [ ] It trains only on issues revealed before the current one

**Deviations:** _none yet_

---

## Phase 4 — Autonomy controller

### 4.1 State machine (`eval/autonomy.py`)
Thresholds are fixed in `repos.yaml` and committed before the final run.
- [ ] Promote SUGGESTION → AUTONOMOUS when there are at least 100 non-abstained decisions, the clean rate over the last 100 is at least 0.90, and coverage is at least 0.5
- [ ] In AUTONOMOUS, sample 10% of decisions for audit (seeded). Only audited outcomes feed the controller.
- [ ] Demote when the audited clean rate over the last 30 audits falls below 0.80. A cooldown of 50 decisions applies before re-promotion.
- [ ] `memory_feed`:
  - `audited` (headline): memory only learns from audited issues while autonomous
  - `all` (upper bound)
- [ ] Log every transition to `autonomy_log`

**Tests required** (scripted outcome sequences):
- [ ] Promotes exactly when all promotion conditions hold, and not before
- [ ] Low coverage blocks promotion even at 100% precision
- [ ] Audit rate is about 10% over a long run and reproducible with the same seed
- [ ] Demotes below the audited threshold; the cooldown prevents immediate re-promotion
- [ ] With `memory_feed=audited`, unaudited issues do not reach memory while autonomous

### 4.2 Final backtests
- [ ] Commit thresholds. Run all predictors on both repos with the autonomy controller and both memory feeds.
- [ ] `eval/report.py` produces:
  - results table: MajorityLabel, KNNVote, ZeroShotLLM, MemoryLLM, LogReg
  - learning curves as PNG + CSV with the autonomy point marked
  - per-label table
- [ ] Commit `results/`

**Done when:**
- [ ] MemoryLLM's micro-F1 is compared against every baseline at equal coverage, with CIs on the last 500 issues. The result is reported whichever way it goes.
- [ ] The learning curve and autonomy timeline exist for both repos

**Deviations:** _none yet_

---

## Phase 5 — Dashboard and write-up

### 5.1 Dashboard (`dashboard/app.py`)
Streamlit, reading committed `results/` CSVs and SQLite. Pages:
- [ ] Learning curve: predictors overlaid, CI bands, zero-shot as a flat band, autonomy point marked
- [ ] Autonomy timeline: state bands plus rolling and audited clean rate
- [ ] Results table
- [ ] Disagreement explorer: the issue, agent vs maintainer labels, the cited past issues with their labels, and the rationale
- [ ] Per-label P/R
- [ ] Deploy to Hugging Face Spaces

**Tests required:**
- [ ] The dashboard loads against the committed `results/` without errors (smoke test)

### 5.2 README
- [ ] Problem, approach, architecture diagram
- [ ] Results table and learning curve image
- [ ] Leakage safeguards, with the counts of dropped and excluded issues
- [ ] Honest failure cases taken from the disagreement explorer
- [ ] Known shift: ty-related issues in ruff during 2025
- [ ] Demo GIF and how to reproduce

**Deviations:** _none yet_

---

## Phase 6 — Stretch: live shadow

Start only after Phase 5 is complete.
- [ ] `live/poll.py`: fetch issues since the cursor and predict with MemoryLLM (memory pre-filled with all history, autonomy starting in SUGGESTION)
- [ ] `live/resolve.py`:
  - score a prediction once its first human label arrives, then add the issue to memory
  - mark it `no_ground_truth` after 14 days
- [ ] GitHub Action on a 6-hour cron. It commits `shadow.db` to a `shadow-data` branch and rebuilds memory from SQLite each run.
- [ ] Dashboard page: live vs backtest clean rate and precision at equal decision counts
- [ ] After at least 3 weeks, report live results with CIs in the README

**Tests required:**
- [ ] Resolve scores only predictions whose issue now has a human label
- [ ] Issues unlabelled for 14 days are marked `no_ground_truth` and never scored

**Deviations:** _none yet_

---

## Out of scope
- Duplicate detection (future work)
- Writing anything to GitHub
