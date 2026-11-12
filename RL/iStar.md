# iStar：Agentic Reinforcement Learning with Implicit Step Rewards

## 元信息
- **arXiv**：https://arxiv.org/abs/2509.19199 （v3，2025-09-28；v1 2025-09-23）
- **标题**：Agentic Reinforcement Learning with Implicit Step Rewards（iStar）
- **作者/机构**：Xiaoqian Liu、Ke Wang、Yuchuan Wu、Fei Huang、Yongbin Li、Junge Zhang、Jianbin Jiao；University of Chinese Academy of Sciences（中国科学院大学）、Tongyi Lab（阿里通义实验室）、Institute of Automation, CAS（中科院自动化所）
- **篇幅**：18 页，8 图
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法、理论与实验表格均抓取到）

## 背景问题
- agentic RL 中奖励稀疏、延迟，有时**根本不可验证**（如开放对话），训练作为 policy 的 LLM agent 时信用分配极难。
- 已有过程监督的缺陷：手工步骤标注昂贵/有偏/易 reward hacking；生成式 reward model（LLM-as-judge）跨域噪声大；token 级隐式 PRM 在 agent 训练里过细 → 高方差、破坏稳定性；基于相同状态分组（GiGPO）在状态几乎不重叠的开放语言环境中失效。
- 核心问题：如何设计一个标签高效、稳定、可扩展到多轮、且对（不）可验证奖励都鲁棒的信用分配策略？

## 框架设计 / 方法
iStar 将一个隐式 PRM 与 policy（LLM agent）**交替优化**，用在线采集的轨迹生成隐式 step reward，形成自强化训练闭环。

1. **隐式 step reward 定义**（式 1）：对轨迹中第 t 步动作 a_t，
   r_φ(o_{1:t}, a_t) = β log [ π_φ(a_t | o_{1:t}, x) / π_{θ_old}(a_t | o_{1:t}, x) ]
   即隐式 PRM π_φ 相对旧策略快照 π_{θ_old} 对当前动作的相对偏好；正值=该动作被认为对近期改进有贡献，负值=应抑制。β ∈ [0,1] 为温度。
2. **轨迹级 DPO 训练 PRM**（式 2）：policy 采样 rollout，由 outcome verifier（或 model）排序成正负轨迹对 (τ⁺, τ⁻)，以 π_{θ_old}（随训练变化，而非冻结初始模型）为 reference，用轨迹级 DPO 目标优化 π_φ。
   - WebShop/VisualSokoban：成功率>0 的为正轨迹；SOTOPIA：goal completion score>6 为正。
   - **理论分析**（3.2 节）：该目标等价于一个带 step-wise reward function 的 Bradley-Terry 模型，即 Eq.2 产出的确是逐步奖励函数。
3. **两级 advantage 融合**（式 3-5）：
   - episode 级 A^E(τ_i) = (r_o(τ_i) − mean) / std（outcome 奖励组内归一化）；
   - step 级 A^S(a_t^i) = (r_φ(a_t^i) − mean) / std（隐式 step reward 组内归一化）；
   - 合并 A(a_t^i) = A^E(τ_i) + α·A^S(a_t^i)，α 平衡两级。
4. **策略优化**：GRPO/PPO 式 clipped surrogate（式 6，不含 KL 惩罚），采用 step 级重要性采样比，兼容 GRPO、RLOO、REINFORCE++、DAPO 等；无需步骤标注、无需额外 rollout。

## 实验结果
**基准**：WebShop（text web，base Qwen2.5-7B-Instruct）、VisualSokoban（6×6，视觉+文本空间规划，base Qwen2.5-VL-7B-Instruct）、SOTOPIA（开放社交博弈，不可验证奖励）。
**基线**：prompting 的 GPT-5、Gemini-2.5-Pro、DeepSeek-R1、Claude-Sonnet-4-Thinking；vanilla RL（PPO、GRPO、RLOO、REINFORCE++）；单轮 RL 的 PRIME（token 级过程奖励）；agentic RL 的 GiGPO。

表 1（WebShop / VisualSokoban，3 seed 平均）：
- RLOO w/ iStar：**WebShop Success 86.5 / Score 93.6，VisualSokoban Success 91.7**，均为最佳。
- 对比：GiGPO（WebShop 84.1 / 91.2，Sokoban 85.9）、PRIME（81.5 / 91.3，Sokoban 不适用）、GRPO（80.1 / 89.3，85.6）；prompting 最强 GPT-5 仅 37.5 / 66.1 / 16.6。

表 2（SOTOPIA，goal completion 0–10）：
- Llama3.1-8B + GRPO w/ iStar：Self-Chat Hard 8.06、GPT-4o-as-Partner Hard 7.16，均为最佳。
- 论文称：相比 vanilla RL，困难社交场景 self-chat 提升 14%（7.92→8.06），与 GPT-4o 对抗提升高达 48%（6.68→7.16）。

其他：
- **采样/探索效率**：WebShop 上仅 105 步即达 vanilla RLOO 最终性能（约 2× 加速），165 步达最高 94.7%；step reward 先涨、episode reward 后跟，episode 长度下降而成功率不降。
- **消融（表 3，1 seed）**：w/ iStar（WebShop 89.1 / 94.7，Sokoban 93.0）优于 w/ token-level process rewards（82.0 / 90.0 / 89.1）、w/ merged rewards（81.3 / 90.7 / 88.3）、w/ ground-truth process rewards（Sokoban 87.5）与纯 RLOO（76.6 / 84.2 / 85.9），验证 step 级（非 token 级）隐式奖励 + 两级 advantage 的有效性。
- iStar 对 RLOO/REINFORCE++/GRPO 均带来提升（RLOO 上两环境 success 各 +6.3%）。

## 创新点
- 首次把"隐式 PRM"引入 agentic RL，是"无显式过程标注"思路的关键源头（信号来源④：轨迹 DPO → step reward，监督档 L1）。
- 用轨迹级 DPO + 旧策略快照为 reference，理论证明可得 step-wise reward function；step 级（非 token 级）粒度在稠密反馈与低方差间取得平衡。
- 仅依赖轨迹偏好，可由（不）可验证 outcome reward 产生，统一覆盖可验证与开放不可验证环境；PRM 与 policy 交替优化形成自强化闭环。

## 局限与讨论
- 需要能对轨迹排序的 outcome verifier 或 model 来构造正负对；完全无任何终局信号时不适用。
- 隐式 PRM 与 policy 交替优化增加了训练管线复杂度；β、α 等需调。
- SOTOPIA 的"goal completion"由 GPT-4o 作代理评判（temperature=0），指标依赖 LLM judge。
- 代码在论文发布时标注"coming soon"。
