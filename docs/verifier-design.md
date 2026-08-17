# The Verifier Design Ladder

*A working note. [中文版](verifier-design.zh-CN.md) · [back to index](../README.md)*

---

## The one-sentence version

Verification is the hard half of agent evaluation — grade what actually happened in the environment, never what the agent says happened, and climb to a more expensive grading tier only when the cheaper one can't express your success criterion.

## The prime directive: grade the substrate, not the transcript

Agents narrate confidently while the environment disagrees. [LITMUS](https://arxiv.org/abs/2605.10779) makes the point for safety — measuring what agents *execute in the OS* rather than what they claim — and every good verifier design shares that instinct: check the database row, the file on disk, the OS state, the deployed page. An agent's own summary of its work is the one artifact you must never grade.

The companion note, [How to Read an Agent Benchmark Score](reading-benchmark-scores.md), documents what happens when verifiers are weak: false passes ([UTBoost](https://arxiv.org/abs/2506.09289)) and false failures ([AgentRewardBench](https://arxiv.org/abs/2504.08942)) both at rates that reorder leaderboards. Design against both from the start.

## The ladder

Four tiers, ordered by cost per graded run. The craft is choosing the *lowest* tier that can express your success criterion — every step up costs money, latency, and a new source of grader error.

### Tier 1 — Deterministic checks

Executable verification against ground truth: AST-based scoring ([BFCL](https://gorilla.cs.berkeley.edu/leaderboard.html) built the field's most-cited tool-calling leaderboard on it), system-state rewards read from the OS ([AndroidWorld](https://github.com/google-research/android_world) reads state via adb rather than matching UI pixels), test suites, exact-match checks on database rows.

Strengths: reproducible, free at scale, judge-free — no bias literature applies. The [Harbor task format](https://harborframework.com/docs/task-format) is the packaging convention: instruction + Dockerized environment + verifier emitting a reward + an oracle solution proving solvability. Write the oracle *first*; a task you can't solve yourself is a task you can't verify.

Limits: only works when success has one checkable shape. The moment valid solutions diverge from your reference — alternate correct patches, different-but-fine phrasings — deterministic checks start under-crediting, which is exactly the AgentRewardBench failure. Fight over-crediting with test augmentation (the UTBoost remedy) and parameterized task variants (the AndroidWorld pattern, which buys contamination resistance for free).

### Tier 2 — Dynamic evaluators for a moving world

When ground truth is time-varying — live APIs, real websites, current prices — freeze-dried answers rot. [MCP-Universe](https://github.com/SalesforceAIResearch/MCP-Universe)'s three-tier evaluator design is the reference pattern: format checks, static matches where possible, and *dynamic evaluators that fetch fresh ground truth at grading time* for everything else.

The tax: your verifier now has dependencies that fail, and the benchmark inherits the rot problem ([OSWorld-Verified](https://xlang.ai/blog/osworld-verified) is the cautionary tale — verify your verifiers on a schedule, not once).

### Tier 3 — Rubric-based judges

When success is a quality judgment, not a state check, you need a judge — but a *structured* one. The evidence for structure over vibes is consistent:

- **Decomposed binary criteria beat holistic scores.** Break "is this good?" into independently checkable yes/no items; [Rubrics as Rewards](https://arxiv.org/abs/2507.17746) formalizes it, and the practitioner canon (binary over Likert) matches.
- **Grading notes are the cheapest big win.** Short per-question notes for the judge closed most of the quality gap in [Databricks' experiments](https://www.databricks.com/blog/enhancing-llm-as-a-judge-with-grading-notes) — no fine-tuning required.
- **Tree-structured rubrics scale to open-ended tasks.** [Mind2Web 2](https://arxiv.org/abs/2506.21506) builds a judge agent per task from a rubric tree checking both correctness and attribution; [PaperBench](https://arxiv.org/abs/2504.01848) co-authors its rubric trees with the original paper authors.

The tax is the entire judge-pathology literature: position bias, self-preference, criteria drift. The mitigations live in the [README's judge sections](../README.md#llm-as-judge); the non-negotiable one is **validate the judge against human labels before trusting it** ([The Alternative Annotator Test](https://arxiv.org/abs/2501.10970) is the statistical procedure, [Who Validates the Validators?](https://arxiv.org/abs/2404.12272) the reason it's iterative).

### Tier 4 — Blinded expert grading

When no verifier can exist — a slide deck, a legal memo, a research deliverable — the honest fallback is human experts, made as rigorous as the format allows: blinded pairwise comparison against a reference deliverable ([GDPval](https://arxiv.org/abs/2510.04374)), expert-authored rubrics applied by fresh experts ([Harvey's LAB](https://www.harvey.ai/blog/legal-agent-benchmark-initial-results)). Expensive, slow, unscalable — and for whole classes of economically real work, still the only instrument that measures the right thing.

## Choosing a tier

| Your success criterion | Tier |
|---|---|
| One checkable end-state (test passes, row exists, file correct) | 1 |
| Checkable, but ground truth changes over time | 2 |
| Many valid solutions; quality is decomposable into criteria | 3 |
| Quality judgment only an expert can make | 4 |

Two rules that override the table: if a cheaper tier *can* express the criterion, use it — and if you're at tier 3+, budget for judge validation as a first-class cost, not an afterthought.

## Hardening, whatever the tier

- **Augment and hold out.** Extra test cases catch over-crediting; a private holdout catches optimization against the public set (the moment your benchmark matters, it becomes a training target — see [the companion note](reading-benchmark-scores.md), question 7).
- **Reset clean.** Every trial starts from a known state; snapshotted environments (the Harbor/Terminal-Bench pattern) make trials independent. State leakage between trials is silent score corruption.
- **Read transcripts anyway.** Every grading tier produces both error directions. The only way to measure yours is to sample graded runs — passes *and* failures — and audit the grader like you'd audit the agent. This loop (grade → read → fix the verifier → regrade) is the actual craft.
- **Mine production for tasks.** The best task sets come from real failures ([cline-bench](https://cline.ghost.io/cline-bench-initiative/) harvests them systematically); a few dozen failure-derived tasks with solid verifiers beat a thousand synthetic ones.
- **Beware rubric gaming.** Anything that becomes a reward gets optimized against — rubric-based rewards are gameable by construction ([Rubrics as Rewards](https://arxiv.org/abs/2507.17746) is candid about this), which is one more reason rubrics need periodic human audit.

## Sources

- [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic; the verifier-craft playbook this note compresses
- [Harbor task format](https://harborframework.com/docs/task-format) · [MCP-Universe](https://github.com/SalesforceAIResearch/MCP-Universe) · [AndroidWorld](https://github.com/google-research/android_world) · [Mind2Web 2](https://arxiv.org/abs/2506.21506) · [GDPval](https://arxiv.org/abs/2510.04374) · [PaperBench](https://arxiv.org/abs/2504.01848)
- The failure evidence: [UTBoost](https://arxiv.org/abs/2506.09289) · [AgentRewardBench](https://arxiv.org/abs/2504.08942) · [LITMUS](https://arxiv.org/abs/2605.10779)
- Judge validation: [The Alternative Annotator Test](https://arxiv.org/abs/2501.10970) · [Who Validates the Validators?](https://arxiv.org/abs/2404.12272) · [Grading Notes](https://www.databricks.com/blog/enhancing-llm-as-a-judge-with-grading-notes)
