# Dream-RSI: Recursive Self-Improvement through Evolving Worlds（通过演化世界实现递归自我改进）

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.14858 （v2，最新修订）
- **作者**：Tong Zheng, Xidong Wu, Zheng Zhang, Zhankui He, Chaoyi Zhang, Benjamin Coleman, Ruoqiao Wei, Di Bai, Haolin Liu, Rui Liu, Xue Wang, Yue Zhuan, Wang-Cheng Kang, Renkai Xiang, Heng Huang, Xinwu Cheng, Yunsong Guo（17 位作者）
- **机构**：University of Maryland, College Park（马里兰大学帕克分校）；Google DeepMind；University of Virginia（弗吉尼亚大学）等（部分作者机构在原文中以占位符显示）
- **提交日期**：2026 年 9 月 14 日（v1）；最新修订 2026 年 10 月 6 日（v2）
- **篇幅/领域**：11 页；cs.CL（计算与语言）
- **代码 / 站点**：https://github.com/zhengkid/Dream-RSI ；https://dream-rsi.com/
- **笔记依据**：基于 arXiv 摘要页 + HTML 全文（已抓取到摘要、引言、动机等正文，部分实验章节正文被截断）

## 背景问题
- **递归自我改进（Recursive Self-Improvement, RSI）** 对自主 AI 智能体日益重要，其进展依赖在复杂领域中发现高价值解。有效的**探索（exploration）** 驱动这一过程，但管理和改进探索策略仍是主要瓶颈。
- 现有系统面临一个根本两难：
  - **固定策略**：随搜索空间扩大无法自适应，可能反复把算力浪费在无效的搜索方向上。
  - **在线策略优化（online policy optimization）**：必须在庞大的元搜索空间（meta-search space）中导航，且反馈来自长时程 rollout，延迟且昂贵。
- 两个根本瓶颈：
  1. **元层反馈延迟且昂贵**：评估一个探索策略需要观察它如何在多个"提议—评估"循环中塑造发现过程，而不像评估单个候选那样简单。
  2. **元策略空间庞大**：需要尝试很多备选才能找到好的，使元层改进代价高昂。

## 框架设计/方法
- **核心洞见**：累积的发现历史（discovery history）本身可以充当"重放模拟器（replay simulator）"——完成的发现过程本就记录了探索决策及其真实代码执行结果的结构化树。类比基于模型的强化学习与 World Models。
- **Dream-RSI 框架**：一个可扩展、递归自我改进的元探索（meta-exploration）框架，用于智能体驱动的发现。
- **轻量编排层（orchestration layer）**：使探索变得显式且可编程（控制分支 branching、并行 parallelism、停止 stopping），同时保持底层基础智能体不变。
- **自我改进循环**（见论文 Figure 1）：
  1. **在线探索（Online Exploration）**：当前策略驱动基础智能体扩展一棵发现树，记录其执行轨迹。
  2. **重放模拟器构建（Replay Simulator Construction）**：把记录的发现树蒸馏成可复用的模拟器池（"世界 world"）。
  3. **基于"做梦"的策略改进（Dreaming-based Policy Improvement）**：同一个基础智能体在模拟器中"做梦（dreaming）"，内部生成大量候选策略并在模拟器中重放，获得快速、低成本的 off-policy 反馈。
  - 改进后的策略重新部署到在线，产生新经验以扩充模拟器池，从而在元探索层闭合一个 RSI 循环。
- 关键机制：由于发现树每个节点的执行结果都已本地存储，评估一个新策略只需读取过往记录（可改变搜索哪些分支、以何顺序、多大并发、何时停止），把元策略改进从昂贵的在线试错转化为快速的基于模拟的"做梦"过程。

## 实验结果
- 在 **4 个领域、9 个发现任务**上评估，Dream-RSI 取得有竞争力的质量，并在若干设置下改进发现效率：
  - **算法工程（Algorithm engineering，Lasso path 求解器）**：优于 sklearn 和强基线，同时相比 SimpleTES 使用至多 **162×** 更少的智能体调用（agent calls）。
  - **数学优化（mathematical optimization）**：在三个任务中的两个上，达到或超过所比较基线中的最佳结果。
  - **GPU 内核工程（KernelBench）**：以 **1.79×–2.43×** 更少的生成次数达到相近速度；或在可比预算下将内核性能提升至多 **2.09×**。
  - **Harness 优化**：在 Terminal-Bench 2.1 上将种子 harness 提升**超过 8 分**。
- （各任务的完整数值表格：全文实验章节正文在抓取中被部分截断，**本笔记以引言中给出的汇总数字为准**。）

## 创新点
1. **历史即重放模拟器（History as Replay Simulator）**：把完成的发现历史重新诠释为重放模拟器，使延迟的探索反馈可复用于廉价的元探索策略评估。
2. **元层 RSI 循环（Meta-Layer RSI Loop）**：提出 Dream-RSI，据作者所知是**首个利用发现历史作为重放模拟器来闭合元层 RSI 循环**的框架。
3. **实证验证**：在 4 个领域 9 个任务上验证了有竞争力的发现质量并在若干设置下改进"发现—智能体调用"效率。
4. 轻量编排层使探索可编程，同时**保持底层基础智能体不变**。

## 局限与讨论
- 摘要表述为"在若干设置下（in several settings）改进发现效率"，而非全面提升——说明效率收益并非在所有任务上一致。
- 重放模拟器本质上只能在**已实现的搜索空间（realized search space）** 上重放：它对历史已经探索过的分支给出 off-policy 反馈，对完全未探索的新区域的外推能力受限（这是 off-policy 重放的固有限制，论文将其定位为"在预先记录的树上导航"）。
- 效果依赖于发现历史的质量与覆盖面；若初始在线探索覆盖不足，模拟器池的价值也受限。
- 本笔记对各任务的**具体基线对比数值表格未能完整获取**（全文对应正文在抓取时被截断）。
