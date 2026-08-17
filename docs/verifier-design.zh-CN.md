# Verifier 设计阶梯

*工作笔记。[English](verifier-design.md) · [返回首页](../README.zh-CN.md)*

---

## 一句话版本

验证是 agent 评估里难的那一半 —— 打分要看环境里实际发生了什么，永远不要看 agent 说自己做了什么；只有当便宜的打分层级表达不了你的成功标准时，才往更贵的那一层爬。

## 最高指令：给真实环境打分，而不是给 transcript 打分

Agent 的叙述总是信心十足，而环境往往并不同意。[LITMUS](https://arxiv.org/abs/2605.10779) 在安全场景把这一点讲透了 —— 测量 agent *在 OS 里实际执行了什么*，而不是它声称做了什么 —— 而每一个好的 verifier 设计都共享这个直觉：去查数据库那一行、磁盘上那个文件、OS 状态、部署出去的页面。agent 对自己工作的总结，是你唯一绝对不能拿来打分的 artifact。

配套笔记[怎么读一个 agent benchmark 分数](reading-benchmark-scores.zh-CN.md)记录了 verifier 太弱时会发生什么：假通过（[UTBoost](https://arxiv.org/abs/2506.09289)）和假失败（[AgentRewardBench](https://arxiv.org/abs/2504.08942)），两者的比例都足以重排 leaderboard。从设计的第一天起，两个方向都要防。

## 阶梯

四个层级，按每次打分的成本排序。手艺在于选*最低*的那个还能表达你成功标准的层级 —— 每往上走一步，都要多花钱、多等延迟，还多引入一种新的 grader 错误来源。

### Tier 1 —— 确定性检查

对着 ground truth 的可执行验证：AST 打分（[BFCL](https://gorilla.cs.berkeley.edu/leaderboard.html) 靠它搭出了全领域被引最多的 tool-calling leaderboard）、从 OS 读出来的系统状态 reward（[AndroidWorld](https://github.com/google-research/android_world) 通过 adb 读状态，而不是匹配 UI 像素）、测试套件、对数据库行的精确匹配。

优势：可复现、规模化零成本、不需要 judge —— 那一整套 judge 偏见文献在这里统统不适用。[Harbor task format](https://harborframework.com/docs/task-format) 是打包约定：指令 + Docker 化环境 + 输出 reward 的 verifier + 一个证明任务可解的 oracle 解。*先*写 oracle；一个你自己都解不出来的任务，就是一个你验证不了的任务。

局限：只有当成功只有一种可检查的形态时才管用。一旦有效解开始偏离你的参考解 —— 另一个同样正确的 patch、不一样但没毛病的措辞 —— 确定性检查就开始少给分，这正是 AgentRewardBench 抓到的那种失败。对抗多给分，用测试增强（UTBoost 的药方）和参数化任务变体（AndroidWorld 的套路，顺带免费买到抗 contamination（数据污染）的能力）。

### Tier 2 —— 面向变化世界的动态评估器

当 ground truth 随时间变化 —— 在线 API、真实网站、实时价格 —— 冻干的标准答案会腐烂。[MCP-Universe](https://github.com/SalesforceAIResearch/MCP-Universe) 的三层评估器设计是参考模式：格式检查、能静态匹配的就静态匹配，剩下的一切交给*在打分时现取新鲜 ground truth 的动态评估器*。

代价：你的 verifier 现在有了会挂掉的依赖，benchmark 也继承了腐烂问题（[OSWorld-Verified](https://xlang.ai/blog/osworld-verified) 就是前车之鉴 —— 要按计划定期验证你的 verifier，而不是只验一次）。

### Tier 3 —— 基于 rubric 的 judge

当成功是一个质量判断、而不是状态检查时，你需要一个 judge —— 但得是*有结构的*那种。「结构胜过感觉」的证据非常一致：

- **拆解成二元判据，胜过整体打分。** 把「这个好不好？」拆成一条条可以独立检查的 yes/no 项；[Rubrics as Rewards](https://arxiv.org/abs/2507.17746) 把它形式化了，实践者的共识（binary 优于 Likert）也对得上。
- **Grading notes 是最便宜的大收益。** 给 judge 写简短的逐题笔记，在 [Databricks 的实验](https://www.databricks.com/blog/enhancing-llm-as-a-judge-with-grading-notes)里补上了大部分质量差距 —— 不需要任何微调。
- **树状 rubric 能扩展到开放式任务。** [Mind2Web 2](https://arxiv.org/abs/2506.21506) 从 rubric 树为每个任务构建一个 judge agent，同时检查正确性和 attribution；[PaperBench](https://arxiv.org/abs/2504.01848) 的 rubric 树是和论文原作者共同撰写的。

代价是整个 judge 病理学文献：position bias、self-preference、判据漂移。缓解手段都在 [README 的 judge 章节](../README.zh-CN.md#llm-as-judge)；不可妥协的那一条是：**先用人工标注验证 judge，再信任它**（[The Alternative Annotator Test](https://arxiv.org/abs/2501.10970) 给出了统计程序，[Who Validates the Validators?](https://arxiv.org/abs/2404.12272) 解释了为什么这件事必须迭代着做）。

### Tier 4 —— 盲评专家打分

当任何 verifier 都不可能存在时 —— 一份 slide deck、一份法律 memo、一项研究交付物 —— 诚实的兜底是人类专家，并把严谨性做到这个形式允许的极限：对着一份参考交付物做盲评式成对比较（[GDPval](https://arxiv.org/abs/2510.04374)），专家撰写 rubric、再由另一批没接触过的专家来执行（[Harvey's LAB](https://www.harvey.ai/blog/legal-agent-benchmark-initial-results)）。昂贵、慢、无法规模化 —— 但对一大类有真实经济价值的工作来说，它仍然是唯一测得准的仪器。

## 怎么选层级

| 你的成功标准 | Tier |
|---|---|
| 一个可检查的最终状态（测试通过、行存在、文件正确） | 1 |
| 可检查，但 ground truth 随时间变化 | 2 |
| 有效解很多；质量可以拆成一条条判据 | 3 |
| 只有专家才能做的质量判断 | 4 |

两条优先级高于表格的规则：如果更便宜的层级*能*表达你的标准，就用它 —— 如果你已经在 tier 3 及以上，把 judge 验证当一等成本来做预算，而不是事后补。

## 加固：不管在哪一层

- **增强测试，留出 holdout。** 额外的测试用例能抓多给分；一个私有 holdout 能抓「对着公开集优化」（你的 benchmark 一旦变得重要，它就会变成训练目标 —— 见[配套笔记](reading-benchmark-scores.zh-CN.md)第 7 问）。
- **每次都重置干净。** 每个 trial 都从已知状态启动；快照化的环境（Harbor/Terminal-Bench 的模式）让 trial 相互独立。trial 之间的状态泄漏是无声的分数腐蚀。
- **无论如何都要读 transcript。** 每一个打分层级都会产出两个方向的错误。想知道自己的错误率，唯一的办法是抽样已打分的 run —— 通过的*和*失败的都要 —— 然后像审计 agent 一样审计 grader。这个循环（打分 → 读 → 修 verifier → 重新打分）才是真正的手艺。
- **从生产环境里挖任务。** 最好的任务集来自真实失败（[cline-bench](https://cline.ghost.io/cline-bench-initiative/) 在系统性地收割它们）；几十个从失败里长出来、配上扎实 verifier 的任务，胜过一千个合成的。
- **当心 rubric 被 game。** 任何变成 reward 的东西都会被优化 —— 基于 rubric 的 reward 从构造上就是可 game 的（[Rubrics as Rewards](https://arxiv.org/abs/2507.17746) 对此很坦诚），这是 rubric 需要定期人工审计的又一条理由。

## 来源

- [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) —— Anthropic。这篇笔记压缩的正是这份 verifier 手艺 playbook
- [Harbor task format](https://harborframework.com/docs/task-format) · [MCP-Universe](https://github.com/SalesforceAIResearch/MCP-Universe) · [AndroidWorld](https://github.com/google-research/android_world) · [Mind2Web 2](https://arxiv.org/abs/2506.21506) · [GDPval](https://arxiv.org/abs/2510.04374) · [PaperBench](https://arxiv.org/abs/2504.01848)
- 失败证据：[UTBoost](https://arxiv.org/abs/2506.09289) · [AgentRewardBench](https://arxiv.org/abs/2504.08942) · [LITMUS](https://arxiv.org/abs/2605.10779)
- Judge 验证：[The Alternative Annotator Test](https://arxiv.org/abs/2501.10970) · [Who Validates the Validators?](https://arxiv.org/abs/2404.12272) · [Grading Notes](https://www.databricks.com/blog/enhancing-llm-as-a-judge-with-grading-notes)
