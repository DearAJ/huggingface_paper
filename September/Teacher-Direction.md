# The Teacher Is a Direction, Not a Destination: Extrapolating RL-Induced Representation Residuals in On-Policy Distillation

> 说明：本笔记基于 arXiv 摘要页 + HTML 全文（v2）撰写。

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.36484 （v2，HTML 全文：https://arxiv.org/html/2609.36484v2 ）
- **作者**：Hao Li, MeiJia Chen, Weijie Ren, Donghan Li, Zijun Tian, Jingchun Huang, Naibo Wang（共 7 位）
- **机构**：原文未在抓取文本中明确列出单位（全文未提供机构信息）
- **日期**：2026-09-29 提交（v1），2026-10-14 修订（v2）；19 页
- **领域**：cs.LG
- **项目页/代码**：https://github.com/xixixixixxxx/RIDE

## 背景问题
- **On-Policy Distillation (OPD，在线策略蒸馏)**：让 student 在自己的轨迹上匹配 teacher 的下一 token 分布，已有显著经验收益。其泛化变体允许 student 通过"在输出空间外推隐式奖励"来超越 teacher。
- 问题：语言模型 head 会**各向异性地（anisotropically）衰减**这种变化——teacher 隐藏态里编码的大部分变化只以很小权重到达 logits；且输出空间外推依赖的"采样 token 对数概率比"会注入噪声并被外推放大，使训练不稳定。
- 关键观察：RL 会相对 base checkpoint 移动模型的内部表征，而这种移动的**方向**可以在每一层测量。因此"teacher 是一个方向，而非终点"。
- 量化：在 Fig.1 中，head 的 512 个最弱方向承载了 teacher-to-base 残差的 **79.8% 隐藏态能量**，但仅占 **30.0% 的 centered-logit 能量**——说明输出空间目标对大部分残差监督很弱，且使早期层不受约束。

## 框架设计 / 方法
- **RIDE（RL-Induced Direction Extrapolation，RL 诱导方向外推）**：直接在**表征空间**外推 RL 诱导的变化。
- student 初始化于 teacher 的 pre-RL checkpoint，三个模型（student、teacher、pre-RL base）共享表征空间。
- 在每个 student 生成的上下文上，RIDE 在相同 token 上评估冻结 teacher 与 pre-RL checkpoint，取其**逐层隐藏态差**作为 RL 诱导残差；用单一系数把目标沿残差方向放置到"超过 teacher"的位置，并把 student 隐藏态回归（regress）到该目标。
- 理论解释：以采样轨迹为条件，该回归等价于"在以 teacher 为中心的二次惩罚下，最大化由残差定义的线性方向奖励"——显式刻画了目标如何把 student 沿 RL 诱导方向推进，同时限制其偏离 teacher。
- 与输出空间方法的关系：在共享线性 head 下，Yang et al. (2026b) 的输出空间外推目标是 RIDE 目标的 head 投影像；RIDE 保留了"奖励外推"视角，但把采样 token 对数比替换为**确定性的隐藏态梯度**。
- 系数 λ：λ=1 时恢复 OPRD（on-policy 表征蒸馏）。

## 实验结果
- **四对 base/RL-teacher**：R1-Distill-1.5B、Qwen3-4B、Llama-3.2-3B、Phi-4-mini —— 跨越不同规模、模型族与词表/预训练谱系。
- **指标/基准**：在 AIME24、AIME25、AIMO 上以 **Avg@16** 衡量。
- **主结果**：
  - RIDE 在每一对上的平均 Avg@16 都高于 OPRD 与输出空间基线；
  - RIDE 逼近或超过 RL 训练的 teacher 本身，并且是唯一"均值能做到逼近/超过 teacher"的方法；
  - 输出空间外推在每一对上都低于其 teacher，且当 teacher 接近其 base 时会退化（degrade）student。
- 另有针对 head 衰减与条件方差的分析，用以刻画 RIDE 的学习信号。

## 创新点
- 识别并提出 **RL 诱导表征残差**（teacher 与其 pre-RL checkpoint 在相同 on-policy 上下文下的逐层隐藏态差）作为"RL 如何改变模型计算"的度量。
- 把"奖励外推"从输出空间搬到表征空间（RIDE），用单一系数沿残差外推隐藏态目标（λ=1 恢复 OPRD），并给出"二次惩罚下线性方向奖励最大化"的理论诠释。
- 用确定性隐藏态梯度替代不稳定的采样 token 对数比，缓解 head 各向异性衰减与外推放大噪声的问题。

## 局限与讨论
- 方法依赖"同初始化蒸馏设定"——需要同时获得 teacher 与其 pre-RL checkpoint（两者在该设定下均可得），对无法获得 pre-RL checkpoint 的场景适用性受限。
- 效果与 RL 更新"小、结构化、跨运行一致"的前提相关；该前提不成立时方向外推的可靠性（原文未在本次抓取范围内给出进一步边界实验）。
- 其余更细的超参敏感性/更大规模验证（原文未在本次抓取范围内完整覆盖）。
