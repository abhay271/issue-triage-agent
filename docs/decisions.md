# Decision Log

Settled design decisions and the reasons for them. Add new entries at the end. Do not rewrite old ones: if a decision is reversed, add a new entry that references the old one.

| # | Decision | Reason |
|---|---|---|
| 1 | Project: a shadow-mode triage agent that learns from maintainer corrections and earns autonomy | Existing triage bots use a fixed prompt plus rules; none learn or graduate. Maintainer labels provide free ground truth. |
| 2 | Scope v1: labels only | Duplicate detection is cut to future work to fit about 3 weeks |
| 3 | Shadow mode: read-only, never writes to GitHub | Safe to run on public repos; scored against what maintainers actually did |
| 4 | Learning = retrieval memory of past human-labelled issues used as few-shot examples | No training loop needed; the effect is visible as a learning curve |
| 5 | Repos: pandas-dev/pandas and astral-sh/ruff | Both have about 2k+ human-labelled issues. pandas has issue-form type labels plus maintainer topic labels; ruff has pure maintainer triage. fastapi moved to Discussions; pydantic and scikit-learn have sparse topic labels. |
| 6 | Ground truth = human labels net of removals within 30 days; labels tagged first-session or later | Keeps the richer target while showing performance on day-1 triage labels separately |
| 7 | Issues with no human label are excluded from scoring (count reported) | Avoids penalising the agent for issues maintainers never triaged |
| 8 | Template (issue-form) labels are inputs, not targets | They are known at creation; predicting them would inflate scores |
| 9 | Drop issues with edited bodies; add recovery only if more than 15% are dropped | Simpler than recovering edit history; prevents leakage |
| 10 | Labels with fewer than 20 human uses are excluded; vocabulary coverage reported | Removes a noisy long tail and keeps prompts small |
| 11 | Autonomy is gated on the issue-level clean rate (no wrong label applied) | Matches what maintainers care about |
| 12 | The agent may abstain; a coverage floor of 0.5 is required for promotion | Prevents earning autonomy by skipping hard issues |
| 13 | Autonomy thresholds are fixed in config before the final run | Avoids fitting thresholds to results |
| 14 | Headline curve starts from empty memory; a warm start is in the appendix | Shows learning from zero |
| 15 | Headline autonomy run: memory learns only from audited issues while autonomous | Realistic: in autonomy only about 10% of issues get human labels |
| 16 | Windows: pandas tune 2023H2 / test 2024–25; ruff tune 2024H2 / test 2025 | Separate tuning data; about 2k issues each, within LLM budget |
| 17 | Zero-shot LLM is evaluated on a fixed 500-issue sample per repo | It does not learn, so one estimate with a CI suffices; this saves about 3k LLM calls |
| 18 | Non-LLM KNNVote baseline included | The LLM must justify its cost against a strong simple method |
| 19 | If MemoryLLM does not beat KNNVote, the story is the learning loop and earned autonomy, with the predictor pluggable | Keeps the project honest and robust to results |
| 20 | LogReg and the Ollama fallback are in scope | LogReg strengthens the comparison; Ollama makes development runs free |
| 21 | Live shadow is a stretch goal after the dashboard and README | The backtest alone delivers the core result; live mode adds a validation line if time allows |
| 22 | Live shadow (if built): memory pre-filled with history, autonomy starts in suggestion mode, state on a `shadow-data` branch | Realistic deployment; autonomy must be re-earned on live data; free hosting |
