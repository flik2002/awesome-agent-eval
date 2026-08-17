# Contributing

Contributions welcome — in English or 中文.

## The bar

An entry must teach something about **measuring agents**: how to build an instrument (benchmark, verifier, judge, trace pipeline), how an instrument fails, or how to run the evaluation loop. That's the whole filter.

**In scope:** agentic benchmarks and their critiques, verifier and rubric design, LLM-as-judge methods and their biases, trajectory/process evaluation, observability and trace analysis, production eval loops, reliability measurement, safety/security evaluation of agents.

**Out of scope:** static model benchmarking as an end in itself (MMLU-style leaderboard culture), training-time evaluation, classic ML metrics, product announcements without measurement content.

A useful test: if the entry would be equally at home on a model-leaderboard list, it doesn't belong here.

## House rules that are stricter than most lists

This list treats every benchmark, judge, and leaderboard as an instrument with pathologies. Accordingly:

1. **Every entry needs a "what you learn" sentence.** "A great benchmark for agents" tells a reader nothing. "Reads rewards from OS state rather than UI matching, making it contamination-resistant by construction" tells them whether to click.
2. **Numbers must be verified.** Only cite figures you confirmed in the primary source (abstract, official post). Vendor self-reported results must be labeled as such.
3. **Fast-moving claims carry dates.** Leaderboard compositions, maintenance status, and "current SOTA" statements rot in months; write "as of YYYY-MM" or leave them out.
4. **Link the critique.** If a benchmark has a documented pathology (contamination study, verifier audit, rot post-mortem), the entry or its neighbors should point to it.

## Format

One line per entry, in the section it fits:

```markdown
- [Title](https://example.com/post) — Author or org. One sentence on what you actually learn from it.
```

## Submitting

1. Fork and branch.
2. Add your entry where it fits; most sections are ordered by usefulness, so place accordingly.
3. Check the link resolves and your claims hold against the primary source.
4. Open a PR describing why the entry meets the bar.

Bilingual note: adding to only one README is fine. Say so in the PR and a maintainer will mirror it, or open a follow-up yourself.

## Longer notes

The `docs/` folder holds synthesized notes rather than link lists. Open topics: LLM-as-judge in practice, trajectory evaluation, production eval loops. Open an issue first so we can agree on scope. Follow the existing shape: a one-sentence thesis, the substance, honest open questions, and sources.

## Removals

PRs that remove dead links, superseded material, saturated benchmarks that stopped differentiating, or entries that no longer meet the bar are as welcome as additions.
