# GiGPO：Group-in-Group Policy Optimization for LLM Agent Training

## 元信息
- **arXiv**：https://arxiv.org/abs/2505.10978 （v3，2025-10-28；v1 2025-05-16）
- **标题**：Group-in-Group Policy Optimization for LLM Agent Training
- **作者/机构**：Lang Feng、Zhenghai Xue、Tingcong Liu、Bo An（通讯）；Nanyang Technological University（南洋理工大学），Bo An 另属 Skywork AI，Singapore
- **发表**：NeurIPS 2025
- **代码**：https://github.com/langfengQ/verl-agent
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法与实验表格均抓取到）

## 背景问题
- group-based RL（GRPO / RLOO）在单轮数学、代码任务上很成功，但迁移到多轮 LLM agent 训练时扩展性受限。
- agent 与环境的交互跨越很多步、token 很长（如 ALFWorld 一条 episode 可达 50 步、2 万+ token），奖励稀疏、常只在结尾出现，单个动作的影响往往延迟很久才显现 → 跨 step 的信用分配（credit assignment）非常困难。
- 直接套用 group-based RL 会把 step 级差异"抹平"（单一轨迹级 advantage 广播到所有 step）。核心问题：能否在保留 group-based RL 的 critic-free、低显存、稳定收敛优点的同时，引入细粒度 step 级信用分配？

## 框架设计 / 方法
GiGPO 用"两层嵌套的相对 advantage"结构：

1. **Episode 级（宏观相对 advantage）**：对同一任务、同一初始状态采一组完整轨迹，按总回报计算组相对 advantage，与 vanilla GRPO 一致，记为 A^E(τ_i)，衡量整条轨迹的完成质量。
2. **Step 级（微观相对 advantage）**：提出 **anchor state grouping（锚点状态分组）** 机制。关键洞察：在相同任务与初始条件下，组内多条轨迹常因无效动作或绕圈反复回到相同环境状态（同一网页/房间/游戏场景）。把源自同一 anchor state 的动作归为一组，在组内算相对 advantage A^S(a_t^(i))，实现局部信用分配，且无需额外 rollout 或 critic。
3. **合并**：
   - A(a_t^(i)) = A^E(τ_i) + ω · A^S(a_t^(i))，ω ∈ ℝ≥0 为加权系数。
   - 最终用 GRPO 式的 clipped surrogate 目标 + KL 惩罚优化（公式 9）。
   - 论文提供 F_norm = std 与 F_norm = 1（w/o std）两种归一化变体。

训练细节：base model 为 Qwen2.5-1.5B/3B/7B-Instruct；ALFWorld/WebShop 上 rollout 组大小 N=8，ω 默认设为 1（原文正文中无进一步调参，附注 ω 稳健区间见后）；search-augmented QA 沿用 Search-R1 设置，检索器用 E5，N=5、最大轮数 4，并引入 similarity-based 分组（最长匹配子序列相似度 > 0.9 即视为同一状态）。

## 实验结果
**基准**：ALFWorld（具身，3827 个任务实例，6 类家务）、WebShop（网购，110 万商品、12k 指令）；search-augmented QA（单跳 NQ/TriviaQA/PopQA，多跳 HotpotQA/2Wiki/MuSiQue/Bamboogle）。
**基线**：闭源 GPT-4o、Gemini-2.5-Pro；prompting ReAct、Reflexion；RL 的 PPO（带 critic）、RLOO、GRPO；QA 另比 Search-R1、ZeroSearch、StepSearch。

ALFWorld / WebShop（3 seed 平均，表 1 摘录）：
- **Qwen2.5-1.5B**：GRPO 在 ALFWorld All = 72.8、WebShop Succ. = 56.8；GiGPO(w/o std) 达 ALFWorld All = 86.1、WebShop Succ. = 67.4；GiGPO(w/ std) ALFWorld All = 86.7、WebShop Succ. = 65.0。
- **Qwen2.5-7B**：GRPO ALFWorld All = 77.6、WebShop Succ. = 66.1；GiGPO(w/ std) ALFWorld All = 90.8、WebShop Succ. = 72.8；GiGPO(w/o std) ALFWorld All = 90.2、WebShop Succ. = 75.2。
- 论文结论：GiGPO(w/o std) 相比 GRPO，在 ALFWorld 提升 13.3%（1.5B）/12.6%（7B），WebShop 提升 10.6%（1.5B）/9.1%（7B），即摘要所说 ">12% on ALFWorld、>9% on WebShop"。

Search-augmented QA（表 2，各数据集平均）：
- GiGPO 3B 平均 = 42.1，7B 平均 = 47.2，显著优于 Search-R1（3B 32.5 / 7B 38.5）、ZeroSearch（31.7 / 39.1）、StepSearch 等。
- 工具效率：7B 在至多 3 次工具调用限制下，单跳任务平均约 0.9 次、多跳约 1.6 次调用，接近 OTC（约 1.0 / 1.7）。

**开销**：与 GRPO 同等 GPU 显存、相同 LLM rollout、几乎零额外时间。另有 VLM 扩展（Qwen2.5-VL-3B，Sokoban/EZPoints 等，表 3）。

## 创新点
- 首个把"组内分组（group-in-group）"思想用于 critic-free step 级信用分配的事实起点，被后续工作（iStar、ProxMO、ReBel 等）当作"最强 step baseline"。
- anchor state grouping：利用轨迹天然的重复状态构造 step 级分组基准，无需额外 rollout、critic 或过程标注（监督档 L2：用 outcome reward + 自动过程信号，免人工步骤标注）。
- 两级 advantage 以 ω 加权融合，兼顾全局轨迹质量与局部 step 有效性，同时保持 group-based RL 的低显存、稳定收敛。

## 局限与讨论
- 锚点分组依赖**状态匹配**：若环境过于复杂或含大量随机文本噪声，不同轨迹之间难以捕捉到完全相同的状态 → 可能出现 singleton（单例）退化，step 级分组失效（后续 ReBel、ProxMO 正是针对这一点改进为软聚合/信念分组）。
- QA 任务相对短程，step 级信号带来的增益较小（原文亦指出）。
- 原文未在正文给出 ω 的完整敏感性曲线表（RLpaper.md 中"0.8 最佳、0.2–1.0 稳健"的说法为二手描述，本次全文未逐一核实到对应数字，谨慎起见以"原文未在抓取正文中明确列出"记）。
