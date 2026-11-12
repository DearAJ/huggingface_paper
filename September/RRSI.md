# RRSI: Regularized Recursive Self-Improvement of Agent Harnesses

> 说明：本笔记基于 arXiv 摘要页 + HTML 全文（v3）撰写。

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.24972 （v3，HTML 全文：https://arxiv.org/html/2609.24972v3 ）
- **作者**：Peng Xia, Rujun Han, Zifeng Wang, Yanfei Chen, Yufan Zhuang, Yoonho Lee, Chengsong Huang, Han Yu, Zhongying CuiZhu, Yifei Ming, Huaxiu Yao, Burak Gokturk, Tomas Pfister, Chen-Yu Lee（共 14 位）
- **机构**：Google Cloud AI Research、Stanford University、Washington University in St. Louis、UNC-Chapel Hill
- **日期**：2026-09-21 提交（v1），2026-10-04 修订（v3）
- **领域**：cs.LG / cs.AI / cs.CL
- **代码/项目页**：https://github.com/google-research/rrsi 、https://regularized-rsi.com/

## 背景问题
- LLM 智能体的能力很大程度由其 **harness（框架/脚手架）** 放大——即围绕冻结 backbone 模型的 prompts、控制流、工具、记忆与上下文管理。
- 近期方法通过迭代地"提出并选择"对 harness 组件的逐项编辑来自动化这一过程，实际上在"智能体系统层面"构成了一种**递归自我改进（RSI）**。
- 问题：这种递归进化可能通过**记忆训练任务而过拟合**——在分布内（in-distribution）大幅提升，但在分布外（OOD）基准上收益缩小甚至消失。作者把过拟合归因于三种耦合行为：benchmark-specific fitting（基准特定拟合）、noise chasing（追逐噪声）、complexity accumulation（复杂度堆积）。

## 框架设计 / 方法
- **RRSI**：把机器学习中的正则化原则引入 harness 自我改进，约束"进化候选的提出与选择"，但**保持 harness 编辑空间完全开放**（prompts、控制流、配置、上下文管理、工具、skills、记忆、子智能体都可改）。只正则化"搜索轨迹"，不限制假设空间。
- **提案侧（proposal-side）正则化**（三种）：
  - **L0 式退火更新稀疏**：用余弦退火预算 b_t 限制单轮候选可捆绑的"独立可归因编辑"数量，从 b_max 降到 b_min（早期多改以探索新机制，后期稀疏、可归因）。类比 L0 基数约束。
  - **证据感知的信用分配（Evidence-Aware Credit Assignment）**：记录每个候选修改的组件、假设、源 diff、分数/成本变化及是否被接受；proposer 在后续轮次以此历史为条件，被否定的机制保留为负证据，避免重复测试已被证伪的假设。
  - **结构化探索（Structured Exploration）**：当最近 w 轮进展落在经验噪声带 δ 内判为"停滞"，则预留一小部分预算给尚未尝试过的组件（类似多样性/熵正则）。
- **选择侧（selection-side）正则化**（critic + pruner）：
  - **泄漏筛查（Leakage Screening）**：critic 在完整评测前读候选 diff，拒绝显式编码任务名/实体名/任务特定值/答案或加入无效机械的编辑（在评测前拦截，避免泄漏候选获得虚高 evolve-set 分数）。
  - **稳定性/复杂度感知接受**：pruner 移除太小、太贵或不再有用的改动；候选须满足若干非补偿性（non-compensatory）准则才能替换现任。类比 L1（结构剪枝）与 L2（复杂度收缩）。
- backbone 策略全程**冻结**；proposer、写跨轮失败反馈的 analyst、泄漏 critic 均为 Claude Opus 4.8。

## 实验结果
- **三域八基准**；每域在单一 suite 上进化，然后不改动地在 held-out 基准上运行：
  - 编码：Terminal-Bench 2.1（89 个容器化终端任务，单元测试验证，Evolve）→ OOD：SWE-bench Verified（仓库级 bug 修复）。
  - 智能体工作区：Harvey LAB（法律工作，25 个实务领域；evolve 120 题 + ID held-out 40 题）→ OOD：JobBench、GDPval、APEX-Agents。
  - 工程设计：EngDesign（61 个设计任务，各由冻结模拟器评分）→ OOD：Frontier-Eng。
- **基线**：未进化基础 harness H0，以及 Meta-Harness、AHE、TTHE、HarnessX 四种近期 harness 进化方法（共享同一 H0、冻结策略、evolve set 与候选预算）。
- **主结果**：RRSI 在 evolve split 上最高 +6.0 分，在 5 个 OOD 基准上最高 +4.3 分，同时产出的 harness 比未正则化进化**少用 30% 的 policy token**。
  - evolve-set 增益：Terminal-Bench 2.1 +6.0、EngDesign +4.9、Harvey LAB +1.1。
  - 离开 evolve split 后：SWE-bench Verified +1.8；Harvey LAB ID held-out +2.3；三个 OOD 智能体基准 +3.5 ~ +4.7（7.2%–13.1%）；Frontier-Eng +4.3 Medal 分（相对 +24.3%）。无任何 held-out split 回退。
  - RRSI 跨 held-out 环境泛化，较平均基线最多高 12.5%。
- **Table 1（智能体工作区）**：RRSI 在 JobBench/GDPval/APEX-Agents 上分别 40.7/52.3/37.9，均高于 H0（36.0/48.8/34.2）与四个基线；但 evolve（Harvey LAB）本身 90.5 并非最高（Meta-Harness 93.0），体现"evolve 分低一点、transfer 更好"的取舍。
- **消融（Table 2）**：去掉接受侧约束使 evolve 分从 90.5 升到 91.5 但 OOD 均值从 43.6 降到 41.0 且 token 成本增约一半；去掉提案侧约束 evolve 仅降 0.2 但 OOD 降 1.7；两者都去掉 evolve 升到 92.8（最高）而 OOD 仅 40.3（接近未进化），token 从 2.42M 升到 3.80M/trial——印证"约束换泛化"。

## 创新点
- 识别并明确"过拟合（evolve-to-transfer gap）"是 harness 递归自我改进的关键挑战，并细分为基准特定拟合、噪声追逐、复杂度堆积三种行为。
- 提出 **RRSI**：在保持 harness 组件完全可编辑的前提下，同时正则化"提案"与"选择"两侧，把 L0/L1/L2 等经典正则化原则类比迁移到自适应 harness 搜索。
- 在迁移性与效率上双赢：既提升 OOD 表现又减少 30% policy token。

## 局限与讨论
- 实验中策略固定为单一 backbone（Claude Opus 4.8），proposer/critic 亦为同一模型；跨不同 backbone 的普适性（原文未在本次抓取范围内给出跨模型验证）。
- 正则化是"类比"经典 L0/L1/L2，而非优化带惩罚项的目标；edit pool 跨轮可变，并非对固定参数向量做稀疏化（作者明确该类比的边界）。
- 停滞判定依赖噪声带 δ、退火预算等超参设定，对阈值的敏感性（原文未在本次抓取范围内展开）。
