# TRACE：Turn-level Reward Assignment via Credit Estimation

## 元信息
- **arXiv**：https://arxiv.org/abs/2607.13988 （v1，2026-07-15）
- **标题**：TRACE: Turn-level Reward Assignment via Credit Estimation for Long-Horizon Agents
- **作者/机构**：Leitian Tao、Baolin Peng、Wenlin Yao、Tao Ge、Hao Cheng、Mike Hang Wang、Jianfeng Gao、Sharon Li；University of Wisconsin–Madison（威斯康星大学麦迪逊分校）与 Microsoft Research（微软研究院）
- **篇幅**：26 页
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法、算法与主结果数字均抓取到）

## 背景问题
- 多轮 agent 要经历几十到上百次工具调用才给出最终答案，post-training 时信用分配是根本难题。
- outcome reward 在长轨迹下稀疏、高方差，且**会误导**：失败 rollout 里可能含很多把 agent 推向目标的有用动作，但 outcome-only 训练给它们和最终错误同样的负 advantage。
- 过程监督（PRM / LLM-judge）需步骤标注、强 judge，或训练出的 PRM 评分会漂移、偏离终局正确性。
- 核心问题：如何在不依赖步骤标注、强 judge 或训练 PRM 的前提下，识别真正把长程 agent 推向答案的工具调用，并转成稠密奖励？

## 框架设计 / 方法
TRACE = critic-free 信用分配，保留终局 verifier 作 anchor，在工具调用边界加稠密信用。关键：用**冻结参考模型**当"探针"而非 judge，衡量每个轨迹前缀是否让 gold answer 更可预测。

1. **工具边界状态**（式 4）：轨迹 τ=(x, a_1,o_1,…,a_T,o_T, ŷ)，在每个工具调用边界切分出前缀状态 S_k；loss 只作用于 policy 生成的 assistant 动作与答案 token，工具观测 o_k 被 mask。
2. **参考答案分数**（式 5）：对前缀 S_k，用冻结 π_ref（policy 初始化的副本）算 gold answer 的平均对数概率
   ℓ̄_k = (1/|y⋆|) Σ_t log π_ref(y⋆_t | S_k, y⋆_{<t}) ≤ 0
   ℓ̄_k 越大（越接近 0）说明当前 transcript 已收集更多生成 y⋆ 的证据；一次批量前向即可，永不被优化。
3. **对数比例状态价值（Log-Ratio State Value）**（式 6）：原始 ℓ̄_k 尺度不适合直接做信用（同样绝对增益，在接近答案时更重要）。定义 d_k = −ℓ̄_k + ε，
   V(S_k) = log( d_0 / d_k ) = log[ (−ℓ̄_0+ε) / (−ℓ̄_k+ε) ]
   故 V(S_0)=0，V(S_k) 越大=已缩小的初始"答案不确定性 gap"的**比例**越大。
4. **消元 TD 信用（Telescoping TD）**：无中间环境奖励、γ=1 时，转移 S_k→S_{k+1} 的信用为一阶 TD 差 δ_k = V(S_{k+1}) − V(S_k) = log(d_k / d_{k+1})。一阶 TD 天然 telescoping（项相消），冗余工具调用无法虚高累积信用。另用 K 步前瞻（look-ahead）折扣 TD（K=3、γ_td=0.8）平滑。
5. **联合优化**（算法 1）：per-token advantage
   Â_{g,t} = α_out·A_g^out + α_turn·r_{g,turn(t)}^turn
   其中 A_g^out 为 GRPO 组归一化 outcome advantage（终局正确性为 anchor），turn 奖励含 TD 信用项与终局填充项。用 GRPO 式 clipped 目标更新。终局正确性仍由 outcome verifier 决定。
   超参：α_out=1.0、α_turn=0.2、ε_train=0.1、λ_term=2.0。

## 实验结果
**设置**：纯 RL（无 SFT 冷启动、无 agentic mid-training、无实时 web 数据、无强 judge、无训练 PRM）。训练于 OpenResearcher 离线语料上构造的合成多文档深搜任务，ReAct 式 harness，工具 browser.search/open/find，FAISS + Qwen3-Embedding-8B 检索；outcome reward 查归一化 EM（含小格式分）。
**模型**：Qwen3-4B-Thinking-2507、Qwen3-30B-A3B-Thinking-2507；lr 1e-6、batch 128、每 prompt 8 rollout、最多 60 工具轮。
**基准**：BrowseComp-Plus（闭卷）、BrowseComp、GAIA、xbench-DeepSearch（开卷/跨语言）。
**基线**：外部 deep-research agent（ASearcher-QwQ-32B、WebDancer-32B、CutBill-30B-A3B、TongyiDS-30B-A3B）作参照；受控 RL 变体 GRPO、GSPO、GiGRPO。

主结果（表 1，四项平均 Avg）：
| 模型与方法 | BrowseComp-Plus | BrowseComp | GAIA | xbench-DeepSearch | 四项平均 |
| --- | --- | --- | --- | --- | --- |
| Qwen3-4B Base | 7.2 | 3.3 | 24.2 | 19.0 | 13.4 |
| Qwen3-4B GRPO | 30.0 | 5.1 | 38.8 | 44.0 | 29.5 |
| Qwen3-4B TRACE | 35.6 | 6.7 | 44.6 | 49.0 | 34.0 |
| Qwen3-30B-A3B Base | 8.4 | 4.4 | 34.1 | 20.0 | 16.7 |
| Qwen3-30B-A3B GRPO | 36.4 | 10.8 | 45.6 | 37.0 | 32.5 |
| Qwen3-30B-A3B TRACE | 42.6 | 12.9 | 52.0 | 45.0 | 38.1 |

- 闭卷 BrowseComp-Plus：4B 从 7.2→35.6、30B-A3B 从 8.4→42.6（纯 RL、最小配方）。
- **零样本开卷泛化**：虽只在闭卷合成数据训练，学到的搜索策略迁移到 open-web 与跨语言（30B-A3B 达 BrowseComp 12.9、GAIA 52.0、中文 xbench-DeepSearch 45.0）。
- vs GRPO：四项平均 4B 34.0 vs 29.5、30B-A3B 38.1 vs 32.5，均大幅领先 GRPO/GSPO/GiGRPO（后者仍停在轨迹级，无法区分同轨迹内有用/冗余工具调用）。
- **训练效率**：相比 outcome-only，TRACE 曲线起飞更早、收敛更快。

## 创新点
- 用冻结参考模型作稳定探针（非 judge、非可训练 reward model），避免评分漂移与 reward hacking。
- 对数比例状态价值：把 gold answer 对数概率转成"缩小初始不确定性 gap 的比例"，解决原始似然尺度问题。
- 消元 TD 信用：一阶 TD telescoping 使冗余工具调用无法虚高累积奖励，同时累积信用与最终参考状态对齐。
- 与 GRPO outcome advantage 联合、以终局正确性为 anchor；无需 critic/PRM/步骤标注（信号来源③：参考/teacher 模型似然的 TD/势差，监督档 L1）。

## 局限与讨论
- 需训练时可得 gold answer（供 π_ref 打分），属 L1 档，无 gold 的开放场景不直接适用。
- 参考模型冻结为初始化副本，随 policy 远离初始分布，探针与当前 policy 的一致性可能下降（原文未在抓取正文中专门量化此影响）。
- 训练数据为合成闭卷深搜；虽展示迁移，但原文指出仍未追平最强外部 deep-research 系统。
- 引入 ε、K、γ_td、λ_term、α_turn 等多个超参。
