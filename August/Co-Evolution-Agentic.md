# Co-Evolution in Agentic Systems: Toward Self-Directed Evolution Beyond Human Design

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2608.10299 （HTML 全文：https://arxiv.org/html/2608.10299v1）
- **作者/机构**：Qing Zong、Jiayu Liu、Junhao Shen、Zecong Tang、Linsi Wu、Yuxuan Liu、Rui Wang、Zhaowei Wang、Weiqi Wang、Cheng Qian、Xiusi Chen、Yangqiu Song（共 12 人）。所列单位包括香港科技大学（HKUST）、伊利诺伊大学厄巴纳-香槟分校（UIUC）、香港中文大学、香港大学、北京大学（通讯 Yangqiu Song, yqsong@cse.ust.hk）
- **提交日期**：2026 年 8 月 10 日（v1）
- **学科分类**：cs.CL
- **论文类型**：综述（survey）
- **全文获取情况**：已获取 HTML 全文（分类体系与形式化定义均来自正文）

## 背景问题（要解决什么痛点）
- Agentic 系统日益被期望"部署后仍能持续改进"，但**单实体自进化（single-entity self-evolution）**（在单个 agent 内更新模型骨干、记忆或技能）常被固定的外部条件（固定任务、固定反馈）所限。
- 这呼应生物学中的 **Red Queen 效应**：持续进步需要相互适应，而非只有一方适应。
- 已有综述覆盖 LLM agent/多 agent 系统、agent 组件与 harness 工程、交互式 agent 的环境扩展；最接近的"自进化 agent"综述也只把 co-evolution 当作子主题。**尚无以"agentic 系统中的 co-evolution（协同进化）"为中心组织轴的综述**。本文填补该空白，追问：哪些组件协同进化、进化自由度的边界如何跨组件扩展、这一过程的终极形态可能是什么。

## 框架设计 / 方法（分类体系与形式化定义）
- **Agentic 系统形式化**：系统 S=(A, E)，A 为 agent 集合、E 为环境。单个 agent a_i=(m_i, h_i)，含模型骨干 m_i 与 harness h_i（记忆、工具、技能、提示、工作流等）。agent 进化条件：Δa_i≠0 ⇔ (Δm_i≠0) ∨ (Δh_i≠0)。agent 集合 A=({a_1,…,a_n}, Π)，Π 编码角色、通信拓扑与分工；A 在任一 a_i 或 Π 变化时进化。
- **Co-Evolution 定义**：设 Ω 为驱动状态转移的进化机制 S^{t+1}=Ω(S^t, τ^t)（它规定进化什么、何时触发、如何生成/更新变体、在何处进化、如何评估进化质量）。co-evolution 是一种耦合的自进化形式：至少两个进化单元共同适应并持续重塑彼此的后续进化（而非仅交换信息或交互）——要求存在 x≠y∈S^t 使 x^{t+1}≠x^t、y^{t+1}≠y^t，且 x 与 y 之间存在相互"进化压力"。
- **渐进式三阶段分类体系**（按"系统被允许进化的范围"逐步扩大、逐步剥离人工设计约束）：
  - **Stage 1 — Agent–Agent Co-Evolution**：固定环境内 agent 集合内部的耦合进化，A^{t+1}=({a_i^{t+1}}, Π^{t+1})=Ω(A^t, E, τ^t)。细分为：
    - **对抗型（Adversarial）**：目标对立、一方进步即抬高对手难度。按压力规模分为"成对对抗压力"（源自 GAN；RARL、竞争式自对弈、SPAG；LLM 安全领域的 ACE-Safety、AdvGRPO、MAGIC、CHASE、Self-RedTeam、ARLAS 等攻防共进化）与"多源对抗压力"（AlphaStar 联赛训练、TriPlay-RL 三方攻击者/防御者/评估者、AdvEvo-MARL 等）。
    - **协作型（Collaborative）**：共享目标，一方改进迫使伙伴调整协作内容。分为"平行协作"（等价 agent）与"角色分化协作"（固定不同角色）。
    - **演化型 agent 组织**：进一步自适应 agent 的角色与交互结构 Π。
  - **Stage 2 — Agent–Environment Co-Evolution**：把适应扩展到环境组件——任务、反馈、交互空间随 agent 变化，(A^{t+1}, E^{t+1})=Ω(A^t, E^t, τ^t)。涵盖自适应任务（工具/数据库状态、多跳可验证问题、GUI 任务、仓库修复等）、演化的反馈/奖励模型（PEBBLE、DAPPER、DUO、ROSKA、CURE 等）、以及与 agent 共进化的世界模型（WebEvolver、DreamGym、EvolvingWorld 等）。
  - **Stage 3 — Meta Co-Evolution**：进化机制本身变为可进化——低层协同进化系统经由自生成的修订过程 Γ^t 修订其进化机制：Ω^{t+1}=Γ^t(S^t, Ω^t, τ^t)，再 S^{t+1}=Ω^{t+1}(S^t, τ^t)。此递归提供走向开放式（open-endedness）的路径，以持续新颖性（Ω^{t+1}≠Ω^t）和无界发散（adaptive capability ℋ 满足 lim_{t→∞} ℋ(S^t, Ω^t)=∞）为特征。

## 实验结果
- 本文为**综述论文，不含自有实验或 benchmark 数值**。全文通过三阶段分类体系组织并综述现有文献（如 GAN、RARL、SPAG、AlphaStar、AdvEvo-MARL、PEBBLE、DUO、CURE、WebEvolver、DreamGym、EvolvingWorld、SEAL 等代表性工作），并给出形式化定义与"论文地图（paper landscape）"。
- （原文未提供定量实验指标，无法获取该部分数字细节。）

## 创新点
- 据作者所述，这是**首个专门聚焦 agentic 系统 co-evolution 的综述**，明确区分"相互适应（mutual adaptation）"与单纯的交互/信息交换。
- 提出**渐进式三阶段分类体系**（Agent–Agent → Agent–Environment → Meta），追踪进化自由度如何从"仅 agent"扩展到"agent 与环境"，最终到"可进化的进化机制"。
- 给出统一的形式化语言（agent=(模型骨干, harness)、集合=(agents, Π)、进化机制 Ω、meta 修订 Γ）来刻画 co-evolution 与通往开放式的路径。
- 讨论评估、跨多组件扩展、以及对日益自主的进化过程保持安全可控等开放挑战与未来方向。

## 局限与讨论
- 作者列出的开放挑战即为其讨论重点：**如何评估此类共进化系统**、**如何在多个组件间扩展（scaling）**、以及**如何让日益自主的进化过程保持安全与可控（safe and controllable）**。
- Meta Co-Evolution 被描述为"新兴（emerging）"阶段，表明该方向证据与实践仍较初步。
- 作为综述，论文本身不提供实证验证；其价值在于提供统一基础与分类框架，而非给出可复现的性能结论。
