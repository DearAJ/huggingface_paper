# Scaling Agentic Data for Long-Horizon Terminal Intelligence

## 元信息
- **arXiv 链接**: https://arxiv.org/abs/2610.13166 （v2，HTML 全文：https://arxiv.org/html/2610.13166v2）
- **提交日期**: 2026 年 10 月 9 日提交（v1），2026 年 10 月 20 日修订（v2）
- **作者/机构**: Shun Zou、Ziyu Ma、Yi Zou（共同一作）、Yong Wang（项目负责人）、Lin Chen、Zehui Chen、Guanhua Chen、Xiangxiang Chu、Feng Zhao 等 9 人。机构：中国科学技术大学（USTC）、阿里巴巴高德（AMAP, Alibaba Group）、南方科技大学。
- **学科分类**: cs.CL（计算与语言）
- **项目主页/数据模型**: https://zs1314.github.io/TerminalHorizon/ ；数据与模型 https://huggingface.co/collections/shunzou05/terminalhorizon
- **本笔记依据**: arXiv 摘要页 + HTML 全文（引言与方法第 1、2 节；后续实验与附录章节未完全抓取到）

## 背景问题
- 终端智能体（terminal agents）正从本地操作走向完成真实世界的专业任务。终端为代码、数据、工具、执行反馈提供了统一的共享工作空间接口。
- **短轨迹（short trajectories）**能演示命令和工具调用，但无法充分展现智能体如何：通过探索累积证据、用证据做判断、基于反馈持续调整策略、把中间结果转化为最终交付物。
- **高质量长程轨迹（long-horizon trajectories）**能够在单一问题求解过程中捕捉这些"智能体行为模式（agentic behavioral patterns）"及其相互连接，为持续自主工作提供更丰富的经验。
- **核心难题**：环境合成（environment synthesis）可能在构造时提前完成或简化了关键工作，从而移除了本应由智能体执行的探索、实现与验证环节。具体两大挑战：
  1. 复杂任务不一定能引出预期的智能体行为模式（素材可能已解决部分问题，让智能体跳过探索/实现）。
  2. 即使模式出现，也需要让其跨越多个阶段展开并相互连接。

## 框架设计/方法
提出 **TerminalHorizon**——一个全自动数据合成引擎，含两个阶段：

- **阶段一：挑战引导的长程合成（Challenge-Guided Long-Horizon Synthesis）**
  - 任务被形式化为可执行终端任务包 **T = (I, E, S, V, M)**：公开指令 I、可执行环境 E、参考解 S、私有验证器 V、执行配置 M；执行时智能体只能访问公开部分 𝒫(T)=(I, E)。
  - **自主批次规划（Autonomous Batch Planning）**：Planner Agent 依据已有任务与研究/评审记录选择下一步研究方向，产出研究简报。
  - **证据驱动的深度研究（Evidence-Grounded Deep Research）**：Deep Research Agent 调查文档、代码仓库、issue 报告、专业材料，形成任务种子 z（含草拟公开指令与支撑证据）。
  - **联合环境构造（Joint Environment Construction）**：Construction Agent 合成具体实例，联合构造指令、环境、参考解、验证器、执行配置；要求验证条件只能基于可发现证据（约束式 (3)）。
  - **独立挑战（Independent Challenge）**：由未参与构造的 Independent Challenger 仅凭 𝒫(T) 尝试任务，生成轨迹并由验证器评分。
  - **Horizon Gap 分析与重构**：对比种子中设定的工作与实际轨迹，定位因构造而被简化/提前完成的工作，重构任务（Refine）以恢复必须执行的工作，然后再次独立执行，形成"挑战—重构"闭环。
- **阶段二：渐进式地平线扩展（Progressive Horizon Expansion）**
  - 在相同目标下扩展原任务，引入需要进一步分析、实现、验证的新需求；并让新增工作依赖前序结果与执行反馈，使智能体行为模式跨阶段延展。
- **行为模式分类体系**：用 **27 个智能体行为模式、分为 5 个族（families）**来描述探索、诊断、构造、验证、策略协调等过程（如 relationship tracing）。

## 实验结果
- **数据集 TerminalHorizon-3K**：构建 **1,535 个任务环境**，收集 **2,852 条训练轨迹**；轨迹平均 **52.8 个交互轮次**，显著长于对比方法。
- **模型 TerminalHorizon-27B**：在约 3K 条轨迹上对 **Qwen3.5-27B** 做监督微调（SFT）得到。
- **主要增益**（相对基座模型）：
  - Terminal-Bench 2.0：**+14.2%**；Terminal-Bench 2.1：**+13.85%**。
  - AutomationBench（真实生产力任务）：Pass Rate **+11.7%**，Partial score **+23.3%**。
  - VitaBench：**+11.3%**。
  - 摘要称在覆盖编码智能体任务、真实生产场景智能体任务与通用推理的 **全部 12 个基准**上均有显著提升。
- **RSI 案例研究**：多个长程递归自我改进（recursive self-improvement）案例，每个有 12 小时研究预算，展示模型能在初始方案之外持续实验、基于反馈改进方法，并把有效改进保留进最终交付物。

## 创新点
1. 提出以"智能体行为模式及其跨阶段连接"为中心看待高质量长程终端数据的视角，强调环境必须保留能引出这些模式的实质性工作。
2. 提出 **TerminalHorizon** 全自动数据合成引擎，将"挑战引导构造"与"渐进式扩展"结合，用独立执行诊断并重构被简化/提前完成的工作。
3. 仅用约 3K 条轨迹的 SFT 即在 12 个基准上取得全面提升，并通过 12 小时 RSI 案例验证持续实验与交付能力。

## 局限与讨论
- 摘要/引言未设独立"局限"小节，本小节内容**原文未显式提供**（全文实验与讨论章节未完全抓取到）。
- 可归纳的讨论点：
  - 方法依赖多智能体（Planner / Deep Research / Construction / Independent Challenger）协同与独立执行，流水线较复杂，计算成本未在已抓取文本中给出。
  - 训练仅采用 SFT，未提及强化学习等进一步优化（基于已抓取内容）。
  - 结论基于单一基座（Qwen3.5-27B）与特定基准集，泛化到其他模型规模/领域的程度未在摘要中说明。
