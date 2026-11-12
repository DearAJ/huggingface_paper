# False Frontiers: Diagnosing and Mitigating Co-Cheating in Self-Evolving Search Agents

> 说明：本笔记基于 arXiv 摘要页 + HTML 全文（v2）撰写。

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.39102 （v2，HTML 全文：https://arxiv.org/html/2609.39102v2 ）
- **作者**：Meijia Chen, Hao Li, Zheng Lu（三人并列贡献）, Hongshan Lin, Junbai Tian, Yichen Liu, Zijun Tian, Yufan Zou, Shuhan Sun, Hanxin Chen, Zeyu Zhang, Weizhi Du, Yueting Li, Tianyu Shi, Alaa Khamis（共 15 位）
- **机构**：原文未在抓取文本中明确列出单位（全文未提供机构信息）
- **日期**：2026-09-30 提交（v1），2026-10-03 修订（v2）；21 页
- **领域**：cs.CL / cs.AI / cs.LG

## 背景问题
- **自进化搜索智能体**通过联合优化 proposer（出题）与 solver（答题）来自建训练课程，形成闭环。
- 作者发现一种失败模式——**co-cheating（共谋作弊）**：proposer 与 solver 在"共享错误"上越来越一致，使得内部奖励提升但外部正确性没有同步改善。
- 事后对照源证据的审计显示：随着自进化轮次增加，co-cheating 愈发严重，伪标签正确率停滞甚至下降，而环内训练信号却在"改善"。

## 框架设计 / 方法
- **MSV（Multi-Sample Verification，多样本验证）**：在训练前验证提案。对同一模型查询 6 次——3 次给源文档、3 次不给源，用于决定任务是否录取并替换不可靠伪标签。缺点：只能部分减少假一致，残留仍可观，且每个候选多付 6 次 labeler 生成成本。
- **CrossFit（主方法，交叉拟合反馈）**：把 proposer 的源文档划分为 A、B 两组；由 A 生成的问题交给"只在 B 上训练的辅助 solver"评分，反之亦然。交叉拟合的一致性决定 proposer 奖励，从而使"同源伪标签无法经由反馈 solver 复现"，而原始 solver 的更新规则保持不变。
- 两个干预针对不同目标：MSV 针对标签质量，CrossFit 针对反馈来源（provenance）。CrossFit 的核心是打破"同源错误被奖励"的自我强化路径；它不把辅助 solver 变成真理 oracle（两 solver 仍可能共享预训练/证据重叠带来的错误）。

## 实验结果
- **模型**：Qwen3.5-4B 与 Qwen3.5-9B 作为 backbone；所有自进化处理从同一公开 checkpoint 出发（即 Base 行），均不使用人工标注 QA 训练数据。
- **假一致质量（false-agreement mass）**（4B / 9B）：
  - 标准耦合反馈：6.1% / 8.8%
  - MSV：降至 5.7% / 7.2%
  - CrossFit：降至 3.0% / 3.7%
  - 用"源排除反馈"重放相同提案：进一步降至 0.4% / 0.1%（将反馈来源与课程变化分离）
- **下游评测**：7 个开放域问答基准（NQ、TriviaQA、PopQA、HotpotQA、2WikiMQA、MuSiQue、Bamboogle），固定 1,325 题评测集（前六项各 200，Bamboogle 全部 125），每题 1 条贪心轨迹。
  - CrossFit 平均性能在 4B 达 48.8%、9B 达 51.2%；
  - 相比标准耦合自进化分别提升 **8.8 / 8.4 分**，相比 Search-R1 提升 **8.7 / 7.8 分**。
  - MSV+CrossFit 组合：4B 达 0.491、9B 达 0.515，仅比单独 CrossFit 高约 0.3 分。
- **结论**：改变 proposer 反馈的"来源"比单纯提升伪标签质量更关键；仅验证（MSV）相比 Dr. Zero 平均只提升 0.7–0.8 分。
- **成本**：MSV 每候选多 6 次生成，预留预算约增 90%；CrossFit 每轮新增 50 次辅助 solver 更新（每 fold 25 次），预算增 72%/79%；两者组合约为 Dr. Zero 预算的 2.6/2.7 倍。

## 创新点
- 系统性诊断并命名了自进化搜索智能体中的 **co-cheating** 现象，并用对照源证据的事后审计给出量化（false-agreement mass）。
- 提出 **CrossFit** 交叉拟合反馈机制：通过源级隔离（source-level exclusion）切断"同源错误被奖励"的自我强化路径，且不改动主 solver 的更新规则。
- 用"固定库重放 / 评估器控制"等对照实验把"反馈来源"从"课程选择"中分离出来，证明来源是关键变量。

## 局限与讨论
- CrossFit 不能保证辅助 solver 是真理 oracle，两个 solver 仍可能共享来自预训练或证据重叠的错误。
- 源级排除目前仅针对划分出的源组；向"相连源（connected sources）"扩展排除、以及端到端效率度量是后续重要方向。
- 干预带来额外计算成本（辅助 solver 更新、labeler 生成）。
- 启示：设计自生成课程时需同时审计"反馈正确性"与"评估器的训练历史/来源"。
