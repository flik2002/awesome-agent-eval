# How to Read an Agent Benchmark Score

*A working note. [中文版](reading-benchmark-scores.zh-CN.md) · [back to index](../README.md)*

---

## The one-sentence version

An agent benchmark score is a measurement made by an instrument, and between 2025 and 2026 the field documented — pathology by pathology — how those instruments fail; this note is the checklist for reading any score without being fooled.

## Why headline numbers mislead

Consider what the critique literature established about the most-cited numbers in the field:

| The score said | The audit found |
|---|---|
| Frontier models resolve most SWE-bench Verified issues | Models can identify the buggy file *from the issue text alone* far more often inside the benchmark than on out-of-benchmark repos — part of the score is memorization ([SWE-Bench Illusion](https://arxiv.org/abs/2506.12286)) |
| Patches "pass the tests" | Augmenting the test suites exposed hundreds of wrong patches that passed anyway, enough to reorder the leaderboard ([UTBoost](https://arxiv.org/abs/2506.09289)) |
| Web agents reached ~90% success | On live tasks the same agents performed near 2024 levels; the gap was shortcut-admissible tasks and unreliable auto-scoring ([An Illusion of Progress?](https://arxiv.org/abs/2504.01382)) |
| Computer-use agents score X% on OSWorld | A large share of tasks can be solved by bypassing the GUI via the terminal, and a nontrivial share of evaluator functions were simply broken ([Epoch's audit](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers), [OSWorld-Verified](https://xlang.ai/blog/osworld-verified)) |
| The Arena ranks models by human preference | Private variant testing and selective disclosure distort the ranking mechanics themselves ([The Leaderboard Illusion](https://arxiv.org/abs/2504.20879)) |

None of this means benchmarks are useless. It means a score is a *claim* whose validity depends on the instrument, and instruments must be audited. The rest of this note is the audit, organized as seven questions.

## 1. Is the benchmark in the training data?

Contamination is the default assumption for anything built from public GitHub/web data before the model's cutoff. Two cheap diagnostics from the [SWE-Bench Illusion](https://arxiv.org/abs/2506.12286) you can run yourself:

- **The file-path probe**: give the model only the issue text and ask which file is buggy. Success far above an out-of-benchmark baseline means it has seen the repo's issues before.
- **The reproduction probe**: ask it to write the gold function; anomalously high n-gram overlap with the reference means recall, not reasoning.

Structural resistance beats spot checks: [AndroidWorld](https://github.com/google-research/android_world) parameterizes each task into millions of variations; live-task designs like [Online-Mind2Web](https://arxiv.org/abs/2504.01382) can't be memorized at all (at the price of reproducibility — see question 5).

## 2. Does the verifier actually verify?

Grader error cuts both ways, and both directions are documented:

- **False passes**: weak test oracles credit wrong answers ([UTBoost](https://arxiv.org/abs/2506.09289)).
- **False failures**: rule-based checkers reject valid alternate solutions the benchmark authors didn't anticipate ([AgentRewardBench](https://arxiv.org/abs/2504.08942)).

A benchmark is exactly as good as its verifier. Before citing a score, find the section of the paper that validates the *grader* — against human judgment, with both error directions measured. If there is no such section, the error rate is unknown, and so is the score's meaning. The constructive side of this question has its own note: [The Verifier Design Ladder](verifier-design.md).

## 3. Does it measure what it claims to measure?

Construct validity, and the canonical case is [OSWorld's GUI bypass](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers): a large share of "computer-use" tasks are solvable entirely from the shell, so part of every score measures terminal competence, not GUI competence. Same genus: "web agent" benchmarks where the DOM contains the answer ([VisualWebArena](https://arxiv.org/abs/2401.13649) exists precisely to close that hole), and "agentic" tasks solvable in one tool call.

The systematic treatments — [Measuring What Matters](https://arxiv.org/abs/2511.04703) and [BetterBench](https://arxiv.org/abs/2411.12990) — audit benchmarks at scale and find validity problems are the norm, not the exception. [AI Agents That Matter](https://arxiv.org/abs/2407.01502) adds the cost dimension: accuracy claims without cost reporting invite Pareto-blind conclusions.

## 4. What does the aggregation hide?

The same runs produce very different headlines depending on the aggregate:

- **pass@1** — capability on an average try; hides variance completely.
- **pass@k** — did *any* of k tries succeed; a capability ceiling, flattering by construction.
- **pass^k** — did *all* k tries succeed; a reliability floor. [MCPMark](https://arxiv.org/abs/2509.24002) reports both, and the gap between its pass@1 and pass^4 is the honest measure of flakiness.

The capability-reliability gap is now its own research thread ([Beyond pass@1](https://arxiv.org/abs/2603.29231), [On the Reliability of Computer Use Agents](https://arxiv.org/abs/2604.17849)): rankings by capability and by reliability *diverge*, so a leaderboard sorted by pass@1 can put the least dependable agent on top. For anything you'll run unattended, pass^k is the number that predicts your experience.

## 5. Is the benchmark still the benchmark?

Live environments rot: [OSWorld-Verified](https://xlang.ai/blog/osworld-verified) fixed hundreds of accumulated issues — dead sites, CAPTCHAs, broken evaluators — which also means pre- and post-repair scores are not comparable. The lifecycle is now familiar: release → rot → "Verified" repair → longitudinal break → eventual retirement (a practice [Vals](https://www.vals.ai/benchmarks) actually follows once a benchmark stops differentiating). Every citation should carry a date and a version.

## 6. Are there error bars, and were enough trials run?

Agent evals are stochastic; single runs are anecdotes. The statistical toolkit exists and is short: clustered standard errors when tasks come in groups ([Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640)), Bayesian intervals at small samples ([Don't Pass@k](https://arxiv.org/abs/2510.04265)), and intraclass correlation to derive how many trials a given task type needs before a comparison means anything ([Stochasticity in Agentic Evaluations](https://arxiv.org/abs/2512.06710)). A two-point difference with no interval is noise until proven otherwise.

## 7. Who benefits from this number?

Vendor-run benchmarks of the vendor's own product are marketing until independently replicated — which does not make them worthless, just unconfirmed. The canonical calibration: [METR's RCT](https://arxiv.org/abs/2507.09089) found experienced developers *slower* with AI assistance while believing they were faster — self-perception and vendor telemetry both flatter. Leaderboard mechanics can themselves be gamed ([The Leaderboard Illusion](https://arxiv.org/abs/2504.20879)), and once a benchmark becomes a training target, its scores measure optimization pressure as much as capability.

## The checklist

Before citing any agent benchmark score:

- [ ] Contamination checked, or the design is contamination-resistant by construction
- [ ] The verifier has a validated error rate, in both directions
- [ ] The task set measures the claimed construct (no GUI-bypass-style holes)
- [ ] You know which aggregate you're reading — and whether reliability (pass^k) tells a different story than capability (pass@1)
- [ ] Version and date pinned; scores across repairs not compared
- [ ] Error bars present; enough trials for the variance of the task type
- [ ] Cost reported alongside accuracy
- [ ] Independent of the party whose product wins
- [ ] The benchmark still differentiates (not saturated, not retired)
- [ ] You've read at least a few actual transcripts — the aggregate is never the whole story

## Sources

The pathology papers: [SWE-Bench Illusion](https://arxiv.org/abs/2506.12286) · [UTBoost](https://arxiv.org/abs/2506.09289) · [An Illusion of Progress?](https://arxiv.org/abs/2504.01382) · [The Leaderboard Illusion](https://arxiv.org/abs/2504.20879) · [Epoch's OSWorld audit](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers) · [OSWorld-Verified](https://xlang.ai/blog/osworld-verified)

The methodology: [AI Agents That Matter](https://arxiv.org/abs/2407.01502) · [Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640) · [Don't Pass@k](https://arxiv.org/abs/2510.04265) · [Stochasticity in Agentic Evaluations](https://arxiv.org/abs/2512.06710) · [Measuring What Matters](https://arxiv.org/abs/2511.04703) · [BetterBench](https://arxiv.org/abs/2411.12990) · [Cost-of-Pass](https://arxiv.org/abs/2504.13359)
