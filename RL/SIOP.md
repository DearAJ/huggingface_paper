# SIOP：Self-Induced Outcome Potential

## 元信息
- **arXiv**：https://arxiv.org/abs/2605.04984 （v1，2026-05-06）
- **标题**：Self-Induced Outcome Potential: Turn-Level Credit Assignment for Agents without Verifiers
- **作者/机构**：Senkang Hu、Yong Dai、Xudong Han、Zhengru Fang、Yuzhi Zhao、Sam Tak Wu Kwong、Yuguang Fang；Hong Kong JC STEM Lab of Smart City、City University of Hong Kong（香港城市大学）、University of Oxford（访问期间）、Fudan University、University of Sussex、Huazhong Univ. of Sci.&Tech.、Lingnan University 等
- **代码**：https://github.com/dl-m9/SIOP.git
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法、理论命题与主结果/消融表格均抓取到）

## 背景问题
- 长程 LLM agent 的成败取决于中间信息收集轮，但训练反馈通常只在最终答案处可得，因为过程级奖励需要大量高质量人工标注。
- 两条已有路线各有短板：
  1. **有监督 turn 级塑形**（IGPO、ΔBelief-RL、TIPS、InfoPO）：奖励"提高 gold answer 概率"的轮，但**需要 gold answer 或稳定的任务专用 verifier**——很多 agentic 任务没有唯一标准答案或稳定校验器。
  2. **无标签 RL**（EMPO、TTRL、RLPR、NOVER、DRO）：从输出分布提取自监督信号，但多停留在答案/轨迹级，**无法给中间轮分配信用**。
- 核心问题：如何在**无校验器（verifier-free）**下，把无标签的 outcome 级可靠性信号"抬升"为 turn 级信用分配。

## 框架设计 / 方法
关键抽象：把采样到的最终答案聚成**语义结果模式（semantic outcome mode）**，当作"潜在未来状态"，在其上做势能塑形。

1. **语义结果模式**：对每个 query 采 K 条 rollout，用 NLI 模型按**双向语义蕴含**（y⇒y′ 且 y′⇒y）把最终答案聚类成 𝒞(q)={c_1,…,c_M}。每个簇 c 的经验质量 m(c|q)=|S_c|/K。verifier-free 指不用 gold answer/任务校验器/手工奖励；NLI 仅作任务无关的语义原语。
2. **可靠性校准**（式 10-12）：经验频率不等于正确性。用可靠性修正 u_θ(c,q)=η·r(c,q) 重加权：
   q_θ(c|q) = m(c|q)·exp(u_θ(c,q)) / Σ_{c′} m(c′|q)·exp(u_θ(c′,q))
   对工具 agent，r(c,q) 实例化为**证据支持度**：轨迹中工具观测 obs 对该簇参考答案的平均 NLI 蕴含置信度（式 11）。效果：低频但强证据支持的正确答案被放大，高频但无证据支持的"高置信幻觉"被压低。
3. **多参考簇支持 + 势能 turn 奖励**（式 14-16）：用簇内小参考集 R(c) 近似簇支持 p̃_θ(c|τ_≤t)=Σ_{y∈R(c)} α_y^(c) p_θ(y|τ_≤t)。outcome potential
   Φ_θ(τ_≤t^(k), c(k)) = q_θ(c(k)|q)·log p̃_θ(c(k)|τ_≤t^(k))
   turn 级过程奖励 = 相邻势能差：
   r_{t,proc}^(k) = q_θ(c(k)|q)·[ log p̃_θ(c(k)|τ_≤t^(k)) − log p̃_θ(c(k)|τ_≤t-1^(k)) ]
   - **命题 1（telescoping）**：累积过程奖励只依赖初末势能差，是势能型（step-wise）。
   - **命题 2（分解）**：= 校准目标质量 × 簇级支持增益；在 q_θ(c⋆|q)=1、R(c⋆)={y⋆} 的有监督极限下，退化为标准 gold-answer 支持塑形 log p_θ(y⋆|τ_≤t) − log p_θ(y⋆|τ_≤t-1)（式 19）。
4. **turn 级 advantage + 优化**（式 20-23）：增广即时奖励 r̄_t^(k)=λ·r_proc + 1[t=T_k](1−λ)·r_final（r_final=q_θ(c(k)|q)）；按 query group 做 GRPO 式归一化、折扣回报 A_t^(k)=Σγ^{u-t} r̃_u^(k)；同一 turn 内 trainable token 共享该 turn advantage，不跨 turn 广播。用 GRPO 式 clipped 目标 + KL 优化。

## 实验结果
**基准**：搜索增强多轮 QA，训练集 merge 自 NQ+2Wiki+TriviaQA，评测 7 个：NQ、TriviaQA、PopQA（单跳）、HotpotQA、2Wiki、MuSiQue、Bamboogle（多跳）。
**backbone**：Qwen3-4B、Qwen3-8B。
**基线**：verifier-free 的 TTRL（硬多数伪标签）、EMPO（按簇频率奖励）；另列**有监督**的 GRPO、IGPO（灰色、仅作上界参照）。

主表（表 1，EM / F1 平均 Avg）：
- **4B EM**：SIOP 0.360 > EMPO 0.353 > TTRL 0.344；有监督上界 GRPO 0.371、IGPO 0.369。
- **4B F1**：SIOP 0.445 > TTRL/EMPO 0.427；接近 IGPO 0.447、GRPO 0.451。
- **8B EM**：SIOP 0.388 = GRPO 0.388 > EMPO 0.377 > TTRL 0.374；IGPO 0.401。
- **8B F1**：SIOP 0.480 > EMPO 0.459 > TTRL 0.454；GRPO 0.478、IGPO 0.493。
- 结论：在 verifier-free 基线中 SIOP 全面领先，多跳（2Wiki、HotpotQA）提升尤明显；**不用任何标注即逼近有监督 outcome 基线**，部分多跳任务（如 2Wiki F1：SIOP 0.411/0.474 vs GRPO 0.409/0.416）甚至超过基于真实答案的 GRPO。

消融（表 2，4B）：去掉 multi-ref、去掉 calibration、两者都去、去掉 process reward 后 EM/F1 平均均下降（Full SIOP EM 0.360 / F1 0.445 为最佳），验证多参考簇支持、可靠性校准、per-turn 过程奖励三者均有贡献。
行为诊断（图 2）：训练中搜索轮数、响应长度、生成熵等指标表明 SIOP 引导更精准检索、减少无意义重复搜索。

## 创新点
- 首次把"无标签 outcome 级可靠性"提升为**verifier-free turn 级信用分配**：用语义结果模式作潜在未来状态做势能塑形（信号来源①：答案分布/语义共识，监督档 L0——连 gold answer 都不用）。
- 可靠性校准把经验频率与外部证据支持结合，修正"高频错答案/低频对答案"，理论上在有监督极限退化为 gold-answer 塑形。
- turn 特定 advantage（不广播轨迹级分数），保留 GRPO 的 critic-free 优点同时实现 turn 级信用。

## 局限与讨论
- 依赖现成 NLI 模型做语义聚类与证据打分，聚类质量/蕴含判断会影响信号可靠性（虽称任务无关，但仍是外部组件）。
- 绝对 EM/F1 仍略低于有监督 IGPO 上界；在部分单跳任务上优势不明显。
- 需每 query 采 K 条 rollout 并聚类、打分，带来额外计算；reliability 强度 η、λ 等需设定。
- L0"真正无 verifier"蓝海，但正确性验证本身困难（原文亦强调靠证据校准而非正确性保证）。
