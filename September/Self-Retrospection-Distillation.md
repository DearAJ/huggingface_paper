# Self-Retrospection Distillation: Turning Post-hoc Experiences into Prior Foresight

> 说明：本笔记基于 arXiv 摘要页 + HTML 全文（v3）撰写。

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2610.08077 （v3，HTML 全文：https://arxiv.org/html/2610.08077v3 ）
- **作者**：Haoxiang Zhang, Qinglin Chen, Hiroaki Hayashi, Zhuofeng Li, Siming Zhang, Jiaxin Zhang, Jixuan Chen, Fang Wu, Pan Lu, Silvio Savarese, Julian McAuley, Chien-Sheng Wu（共 12 位，标注有 Core contributors）
- **机构**：Salesforce AI Research、UC San Diego、Texas A&M University、Stanford University
- **日期**：2026-10-06 提交（v1），2026-10-14 修订（v3）
- **领域**：cs.AI / cs.CL / cs.IR / cs.LG
- **代码**：https://github.com/SalesforceAIResearch/SRD

## 背景问题
- RLVR（可验证奖励强化学习）主要通过交互后的标量结果奖励把智能体经验转化为学习信号。
- 对于组相对（group-relative）目标（如 GRPO），当一组 rollout 全部获得相同奖励时，组内优势 A(τ)=(r−μ)/σ 归零，信号消失——尤其在长程推理任务里大多数尝试都失败，标准做法会直接丢弃这些 reward-uniform 组并重采样，浪费轨迹及其信息。
- 作者提出一个互补问题：事后经验（hindsight）能否教会智能体在"行动之前"就预判任务所需知识与可能失败（foresight）？

## 框架设计 / 方法
- **Prospective Learning（前瞻式学习）**：用事后经验监督"交互前视角"的预测。定义前瞻分布 p_fore = π_θ(·|x,e)（仅凭任务 x 与环境上下文 e，不看已发生的交互），和事后条件分布 p_hind = π_θ̄(·|x,e,z_hind)（额外看到特权事后信息），损失为 D(p_hind ‖ p_fore)。不直接优化动作分布。
- **Self-Retrospection Distillation (SRD)**：对每条 on-policy 轨迹，把前瞻预测与"看到完整交互的事后预测"配对，在 student 自己的前瞻 rollout 上对齐 token 级分布。
- **两类前瞻视角**：Knowledge（交互中可能需要的知识，来自成功轨迹 r=1）与 Pitfall（可能遇到的失败，来自失败轨迹 r=0）。奖励决定一条轨迹"教什么"，而非"是否能教"。
- 事后特权上下文 f^i=(τ^i, y*, ε^i)：轨迹、gold 解/标准答案、错误标注。
- Pitfall 采用两阶段构造：逐轨迹诊断 + 聚合（把同一 query 所有失败轨迹的 pitfall 合并成一条简短共享清单）。
- SRD 是"可组合的轻量辅助目标"，可叠加到 RLVR / 自蒸馏算法上；Foresight 仅作训练目标，推理时不需显式生成。
- 自蒸馏使用 EMA self-teacher（更新率 0.05/步），teacher 分布用 top-k 对数概率近似，重要性采样比裁剪为 2.0。

## 实验结果
- **训练设置**：Qwen3.5-4B 与 Qwen3.5-9B（thinking-on），单节点 8×NVIDIA H200；每步采样 16 prompts × 8 on-policy rollouts = 128 轨迹（动态采样重采样前）；温度 1.0、top_p 1.0、最多 8 轮交互。
- **数据/评测**：数学用 DAPO-Math-17K 作源池，用 DAPO 规则验证器评分；AIME 2024、AIME 2026、AMO-Bench 仅用于评测；代码训练用 LiveCodeBench stdin（medium/hard），留出 LCB-v6 functional + OJBench medium；搜索任务用 EM + LLM-as-Judge（gpt-5.6-luna）；并有 ALFWorld、WebShop 等长程智能体任务。合计 10 个工具集成推理与长程智能体任务。
- **主结果**：SRD 相比 RLVR 和自蒸馏基线最高提升 **24.2 pp**。
- 当奖励对比稀缺时优势尤为明显：跨模型规模 37%–98% 的 rollout 组是 reward-uniform。
- **极端情形**：在 2B 设置下 98% 的组全失败，纯 RLVR(GRPO) 训练最终成功率为 **0.0%**，而在相同 rollout 预算下加入 SRD 可达 **60.6%**。
- 分析（RQ1）：Knowledge 通道散度基本不变，Pitfall 散度下降 30–47%，提示 Knowledge 通道存在冗余，故将 Pitfall-only SRD 作为计算高效默认。

## 创新点
- 提出"前瞻式学习"这一新监督目标：把 RLVR/自蒸馏的"事后监督行为"转为"用事后经验监督交互前的预判"。
- SRD 把成功/失败轨迹统一为 Knowledge 与 Pitfall 两类前瞻监督，使得 reward-uniform（优势为 0）的轨迹仍能提供密集学习信号。
- 作为轻量可组合辅助目标，可叠加在已有 RL/蒸馏算法上，不改动原 RL 目标（对比干净）。

## 局限与讨论
- 作者自述理论界是一侧（one-sided）、部分量（如 L 与 ‖B‖_op）未测量，小散度仍可能携带任务相关信号，故不能据此做更强断言。
- Knowledge 通道被观测到存在冗余，默认退化为 Pitfall-only。
- 依赖特权事后信息（gold 解/错误标注）构造 teacher，对无法获得参考答案或难以自动评判的任务适用性受限（全文未就此给出系统性扩展实验）。
- 其余更细的失败模式/鲁棒性讨论（原文未在本次抓取范围内完整覆盖）。
