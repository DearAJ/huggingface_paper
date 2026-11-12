# IGPO：Information Gain-based Policy Optimization

## 元信息
- **arXiv**：https://arxiv.org/abs/2510.14967 （v2，2026-03-24；v1 2025-10-16）
- **标题**：Information Gain-based Policy Optimization: A Simple and Effective Approach for Multi-Turn Search Agents
- **作者/机构**：Guoqing Wang、Sunhao Dai（通讯）、Guangze Ye、Zeyu Gan、Wei Yao、Yong Deng、Xiaofeng Wu、Zhenzhe Ying；主要来自 Venus Team, Ant Group（蚂蚁集团），部分作者来自 Renmin University of China（中国人民大学）
- **发表**：ICLR 2026 接收
- **代码**：https://github.com/GuoqingWang1/IGPO
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法与分析抓取到；主结果主表数字因 HTML 渲染被截断，相关结论以抓取到的分析文字为准，详见下方说明）

## 背景问题
outcome-only RL（只在最终答案给奖励）在多轮长程 agent 中有三大病：
1. **advantage collapse（优势塌陷）**：组内 rollout 答案全对或全错时，奖励相同，组相对 advantage 归零、无梯度信号——小模型因更难解决复杂 query 而尤为严重。
2. **缺乏细粒度 credit**：多轮中后轮依赖前轮，当前轮正确可能被前面错误废掉，outcome 信号糊掉了中间轮的对错。
3. **样本效率低**：整条几千 token 的长轨迹仅得一个终局标量信号，丰富的中间信息被浪费。

已有过程奖励方案要么靠外部 oracle / reward model（昂贵且有偏），要么靠 Monte Carlo 估计 step 值（样本不足时高方差）——都缺乏简单稳定的内在监督。

## 框架设计 / 方法
核心直觉：把 agent 与环境的每一轮交互视为"向 ground truth 递进获取信息"的过程。

1. **信息增益奖励（Turn-level IG Reward）**：第 t 轮结束时，计算当前策略生成正确答案 a 的对数概率相比上一轮的边际增量：
   r_{i,t}^{IG} = log π_θ(a | q, o_{i,≤t}) − log π_θ(a | q, o_{i,≤t-1})
   内在信号，直接来自模型自身的 belief update，不依赖外部 reward model，天然具备 ground-truth awareness（提升正确答案置信度即得正奖励，反之为负）。
2. **分组归一化 + 折扣回报**：组内对 IG 奖励与 outcome 奖励**分别**做 z-score 归一化，再引入折扣因子 γ 计算 turn 级折扣回报 R̃_{i,t} = Σ_{k=t}^{T} γ^{k-t} r̃_{i,k}，捕捉长程依赖。
3. **GRPO 式优化**：用 GRPO surrogate 目标，把 rollout 级 advantage 替换为 turn 级折扣回报，按 turn 做 credit assignment（而非像 GRPO 把 outcome advantage 广播到所有 token）。
4. **向量化高效实现（Custom Attention Mask）**：朴素实现需 T 次前向；IGPO 在轨迹末尾追加 T 份 ground-truth 答案拷贝，用自定义注意力掩码限制每份拷贝只看对应 turn，单次前向即可算出所有 T 轮的 log 概率，渐进加速约 T/3（T=10 时约 3×）。理论分析（附录 F）表明额外 FLOPs 相对 GRPO 不足约 2g/s（量级 10^-3），每训练步额外耗时约万分之一，可忽略。

base model：Qwen2.5-3B/7B-Instruct；prompt 风格沿用 DeepResearcher。

## 实验结果
**基准**：4 个域内（NQ、TriviaQA/TQ、HotpotQA、2Wiki）+ 3 个域外（MuSiQue、Bamboogle、PopQA）搜索增强 QA。
**基线**：GRPO、PPO、DeepResearcher、StepSearch、GiGPO 等。

- 论文摘要与分析部分给出：IGPO 在多轮场景中**一致优于强基线**，在答案准确率与数据效率上均有明显提升，尤其对小模型（缓解 advantage collapse）。
- 文中提供的"IGPO vs GRPO 逐数据集胜负比例"表显示：全体样本上 IGPO>GRPO 占 48.3%、持平 48.1%、IGPO<GRPO 仅 3.6%；MuSiQue 上 IGPO>GRPO 高达 71.2%。
- **抑制虚假相关性**：用外部强模型仅凭推理轨迹反推答案的测试表明，IGPO 导出的中间思维轨迹信息密度/逻辑质量更高，并非"蒙对答案"。
- 失败案例分析（图 9）：对 HotpotQA 中"Who is the author of Childhood?"这类天然歧义题，正确但非 gold 答案会被误判为错，产生误导性负奖励，削弱 IGPO 效果。

> 说明：RLpaper.md 中记录的"IGPO 平均 60.2，优于 GRPO 51.9 / PPO 51.5 / DeepResearcher 53.9 / StepSearch 46.0 / GiGPO 47.2"以及"3B 上平均 F1 从 GRPO 32.3 提升到 48.9（+16.6）"等具体数字，本次抓取的 HTML 正文主结果表被渲染截断，**未能在原文抓取文本中逐项核实**，故此处不作为已核实数字引用，仅如实标注来源于二手描述、待核。

## 创新点
- 用模型自身 belief update 的"信息增益"作为 turn 级内在过程奖励，无需外部 reward model、无需 Monte Carlo 采样，开销近乎为零。
- 组内 IG / outcome 奖励分别归一化 + 折扣回报，稳定训练并直接化解 advantage collapse。
- 向量化 attention mask 实现，使稠密 turn 级奖励几乎零额外成本（监督档 L1：仍用 gold answer 做终局锚，但无额外价值模型）。

## 局限与讨论
- 依赖 gold answer（计算正确答案似然），属"免 critic/PRM 但仍需标准答案"的 L1 档，无法直接用于无 gold answer 的开放场景（SIOP 正是针对此扩展）。
- 对天然歧义题，gold answer 唯一性假设会产生误导性负奖励，惩罚了合理但非标准的推理（原文图 9 案例）。
- 本次全文抓取中主结果数值表被截断，具体逐数据集分数需读者回原文 PDF 核对。
