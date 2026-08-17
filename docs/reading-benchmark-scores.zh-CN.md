# 怎么读一个 Agent Benchmark 分数

*工作笔记。[English](reading-benchmark-scores.md) · [返回首页](../README.zh-CN.md)*

---

## 一句话版本

Agent benchmark 分数是一台仪器做出来的测量，而 2025 到 2026 这两年，这个领域把这些仪器怎么失效 —— 一种病理一种病理地 —— 记录了下来。这篇笔记就是那份 checklist：读任何分数，不被糊弄。

## 为什么头条数字会骗人

看看批评文献对这个领域被引用最多的那批数字都查出了什么：

| 分数说 | 审计发现 |
|---|---|
| 前沿模型能解决 SWE-bench Verified 上的大多数 issue | 模型*只凭 issue 文本*就能指出哪个文件有 bug，这件事在 benchmark 内部的成功率远高于 benchmark 外的仓库 —— 分数里有一部分是记忆（[SWE-Bench Illusion](https://arxiv.org/abs/2506.12286)） |
| Patch「通过了测试」 | 增强测试套件后，暴露出数百个照样通过的错误 patch，多到足以让 leaderboard 重新排序（[UTBoost](https://arxiv.org/abs/2506.09289)） |
| Web agent 成功率到了 ~90% | 换到 live 任务上，同一批 agent 的表现回到了 2024 年的水平；差距来自允许走捷径的任务和不可靠的自动打分（[An Illusion of Progress?](https://arxiv.org/abs/2504.01382)） |
| Computer-use agent 在 OSWorld 上拿了 X% | 一大块任务可以经由终端绕过 GUI 解决，还有不小一部分 evaluator 函数干脆就是坏的（[Epoch 的审计](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers)、[OSWorld-Verified](https://xlang.ai/blog/osworld-verified)） |
| Arena 按人类偏好给模型排名 | 私下测试变体和选择性披露，扭曲的是排名机制本身（[The Leaderboard Illusion](https://arxiv.org/abs/2504.20879)） |

这些都不代表 benchmark 没用。它代表：分数是一个*主张*，主张的有效性取决于仪器，而仪器必须被审计。这篇笔记剩下的部分就是那场审计，组织成七个问题。

## 1. Benchmark 在训练数据里吗？

对任何用模型 cutoff 之前的公开 GitHub / 网页数据构建的 benchmark，contamination（数据污染）都是默认假设。[SWE-Bench Illusion](https://arxiv.org/abs/2506.12286) 给了两个便宜的诊断，你自己就能跑：

- **文件路径探针**：只给模型 issue 文本，问哪个文件有 bug。成功率远高于 benchmark 外的基线，就说明它见过这个仓库的 issue。
- **复现探针**：让它把 gold function 写出来；和参考实现的 n-gram 重叠高得反常，说明是背诵，不是推理。

结构性的抵抗好过抽查：[AndroidWorld](https://github.com/google-research/android_world) 把每个任务参数化成数百万种变体；[Online-Mind2Web](https://arxiv.org/abs/2504.01382) 这类 live 任务设计根本没法被背下来（代价是可复现性 —— 见问题 5）。

## 2. Verifier 真的在验证吗？

Grader 的错误是双向的，而且两个方向都有文献记录：

- **假通过（false pass）**：弱的 test oracle 会给错误答案发通过（[UTBoost](https://arxiv.org/abs/2506.09289)）。
- **假失败（false failure）**：基于规则的 checker 会拒掉 benchmark 作者没预料到的合法替代解（[AgentRewardBench](https://arxiv.org/abs/2504.08942)）。

一个 benchmark 的成色，恰好等于它 verifier 的成色。引用分数之前，先去论文里找验证 *grader* 的那一节 —— 要对照人类判断，两个错误方向都要测。如果没有这一节，错误率就是未知数，分数的含义也是。这个问题的建设性那一面有自己的笔记：[The Verifier Design Ladder](verifier-design.zh-CN.md)。

## 3. 它测的真是它声称要测的东西吗？

这问的是 construct validity（构念效度），经典案例是 [OSWorld 的 GUI 绕过](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers)：一大块「computer-use」任务完全可以只靠 shell 解决，所以每个分数里都有一部分测的是终端能力，不是 GUI 能力。同属一类的还有：答案直接写在 DOM 里的「web agent」benchmark（[VisualWebArena](https://arxiv.org/abs/2401.13649) 就是为了堵这个洞才存在的），以及一次 tool call 就能做完的「agentic」任务。

系统性的处理是 [Measuring What Matters](https://arxiv.org/abs/2511.04703) 和 [BetterBench](https://arxiv.org/abs/2411.12990) —— 规模化地审计 benchmark，结论是：效度问题是常态，不是例外。[AI Agents That Matter](https://arxiv.org/abs/2407.01502) 补上了成本这个维度：只声称 accuracy 而不报成本，就是在邀请无视 Pareto 的结论。

## 4. 聚合方式藏住了什么？

同一批 run，换个聚合方式，就是完全不同的头条：

- **pass@1** —— 平均一次尝试的能力；把方差完全藏起来。
- **pass@k** —— k 次尝试里有*任何*一次成功吗；这是能力天花板，构造上就偏好看。
- **pass^k** —— k 次尝试是否*全部*成功；这是可靠性地板。[MCPMark](https://arxiv.org/abs/2509.24002) 两个都报，它的 pass@1 和 pass^4 之间的差距，才是对 flakiness 的诚实度量。

能力-可靠性差距现在已经是一条独立的研究线（[Beyond pass@1](https://arxiv.org/abs/2603.29231)、[On the Reliability of Computer Use Agents](https://arxiv.org/abs/2604.17849)）：按能力排名和按可靠性排名会*分叉*，所以一个按 pass@1 排序的 leaderboard，完全可能把最不可靠的 agent 放在榜首。任何你打算无人值守运行的东西，pass^k 才是那个能预测你真实体验的数字。

## 5. 这个 benchmark 还是原来那个 benchmark 吗？

Live 环境会腐烂：[OSWorld-Verified](https://xlang.ai/blog/osworld-verified) 一口气修掉了几百个累积的问题 —— 死掉的网站、CAPTCHA、坏掉的 evaluator —— 这也意味着修复前和修复后的分数不可比。这个生命周期现在已经很眼熟了：发布 → 腐烂 →「Verified」修复 → 纵向对比断裂 → 最终退役（benchmark 失去区分度之后就该退役，[Vals](https://www.vals.ai/benchmarks) 真的在这么做）。每一次引用都应该带上日期和版本。

## 6. 有误差条（error bars）吗？trial 跑够了吗？

Agent eval 是随机的；单次 run 只是轶事。统计工具箱是现成的，而且很短：任务成组出现时用 clustered standard errors（[Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640)），小样本时用贝叶斯区间（[Don't Pass@k](https://arxiv.org/abs/2510.04265)），再用 intraclass correlation 推导某类任务要跑多少次 trial、比较才开始有意义（[Stochasticity in Agentic Evaluations](https://arxiv.org/abs/2512.06710)）。两个点的差距，只要没带区间，在证明之前就是噪音。

## 7. 这个数字对谁有利？

Vendor 给自家产品跑的 benchmark，在被独立复现之前就是 marketing —— 这不等于一文不值，只是未经证实。经典的校准案例：[METR's RCT](https://arxiv.org/abs/2507.09089) 发现，有经验的开发者用 AI 辅助后实际*更慢*，但自己坚信更快了 —— 自我感知和 vendor 遥测都会往好里报。Leaderboard 机制本身也可以被操纵（[The Leaderboard Illusion](https://arxiv.org/abs/2504.20879)），而一旦一个 benchmark 成了训练目标，它的分数测到的优化压力，就跟能力一样多。

## 那份 checklist

引用任何 agent benchmark 分数之前：

- [ ] Contamination 查过了，或者设计本身在构造上就抗污染
- [ ] Verifier 有经过验证的错误率，而且两个方向都有
- [ ] 任务集测的确实是它声称的那个构念（没有 GUI 绕过式的洞）
- [ ] 你知道自己读的是哪种聚合 —— 以及可靠性（pass^k）讲的故事是否和能力（pass@1）不一样
- [ ] 版本和日期钉死了；跨修复版本的分数不拿来比
- [ ] 有误差条；trial 次数配得上这类任务的方差
- [ ] 成本和 accuracy 一起报了
- [ ] 出数字的人，和产品赢了的那一方相互独立
- [ ] 这个 benchmark 还有区分度（没饱和，也没退役）
- [ ] 你至少读过几份真实的 transcript —— 聚合数字从来不是故事的全部

## 来源

病理类论文：[SWE-Bench Illusion](https://arxiv.org/abs/2506.12286) · [UTBoost](https://arxiv.org/abs/2506.09289) · [An Illusion of Progress?](https://arxiv.org/abs/2504.01382) · [The Leaderboard Illusion](https://arxiv.org/abs/2504.20879) · [Epoch's OSWorld audit](https://epoch.ai/blog/what-does-osworld-tell-us-about-ais-ability-to-use-computers) · [OSWorld-Verified](https://xlang.ai/blog/osworld-verified)

方法论：[AI Agents That Matter](https://arxiv.org/abs/2407.01502) · [Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640) · [Don't Pass@k](https://arxiv.org/abs/2510.04265) · [Stochasticity in Agentic Evaluations](https://arxiv.org/abs/2512.06710) · [Measuring What Matters](https://arxiv.org/abs/2511.04703) · [BetterBench](https://arxiv.org/abs/2411.12990) · [Cost-of-Pass](https://arxiv.org/abs/2504.13359)
