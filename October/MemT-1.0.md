# MemT-1.0: Towards Memory-Native Evolution

## 元信息
- **arXiv 链接**: https://arxiv.org/abs/2610.26825 （v2，HTML 全文：https://arxiv.org/html/2610.26825v2）
- **提交日期**: 2026 年 10 月 19 日提交（v1），2026 年 10 月 20 日修订（v2）
- **作者/机构**: MemTensor Team（Zhiyu Li、Zeyu Zhang、Linyun Liu、Yiwei Bu 等共 74 位作者；通讯/资深作者包括 Zhi-Qin John Xu、Ying-Cong Chen、Ningyu Zhang、Haofen Wang、Junchi Yan、Feiyu Xiong 等）
- **学科分类**: cs.CL（计算与语言）、cs.LG（机器学习）
- **篇幅**: 66 页，15 张图，16 张表
- **开源**: 代码 https://github.com/MemTensor/MemT-1.0 ；模型 https://huggingface.co/collections/MemTensor/memt-1
- **本笔记依据**: arXiv 摘要页 + HTML 全文（引言与贡献部分，部分章节未抓取到）

## 背景问题
- "记忆原生演化"（memory-native evolution）要求把记忆变成基础模型的**内在能力**，使模型能够持续地积累并利用信息，而不是依赖外部上下文。
- 作者认为，许多能力（如推理、多模态）正逐步从外部系统"内化"到模型自身；记忆是另一项基础能力，使模型能够在长期交互中积累经验并自适应。
- 团队的前作 **Metis** 首次提出"记忆基础模型"（memory foundation models），将记忆作为基础模型的原生能力。MemT-1.0 是在 Metis 基础上的重大升级，目标是突破外部上下文的限制。

## 框架设计/方法
- **MemT-1.0 架构**：一种新的记忆基础模型架构，将记忆内化为模型计算的一部分；保留了 Metis 的"动态局部记忆（dynamic local memory）"与"可训练超记忆（trainable hyper memory）"结构。
- **Gated DeltaNet-2 (GDN-2)**：用稳定的 token 级记忆更新机制替代原来的 token 选择机制，在每一轮（turn）内进行稳定的 token 级更新。
- **Mixture-of-Storage (MoS)**：一种新引入的机制，用于扩展原生记忆容量（scale native memory capacity）。
- **模型家族**：
  - **MemT-1.0-Nano**：36B 参数（激活 3B，动态记忆 10M）
  - **MemT-1.0-Flash**：288B 参数（激活 17B，动态记忆 336M）
- **两阶段训练流水线**（在"刻意的 backbone 准备"之后）：
  1. **持续预训练（CPT）**：建立具备原生记忆的语言建模能力。
  2. **后训练（post-training）**：适配下游记忆任务，同时保留通用能力。
  - 训练数据规模：**34B tokens**。
- **基础设施**：开发了分布式基础设施，保证独立轨迹（trajectory）之间的记忆隔离；将有状态推理组织为分离的"记忆存储"与"记忆利用"过程，共享模型参数、按轨迹维护记忆状态。对固定记忆配置，持久状态大小随交互累积保持恒定。

## 实验结果
- **记忆基准**：在 11 个记忆基准（benchmark）上，模型平均优于参数化记忆（parametric memory）基线，在推理时不访问原始上下文即可有效利用历史信息；在噪声设置下也在很大程度上保留了通用能力。
- **与检索增强方法对比**：保持与检索增强（retrieval-augmented）方法相当的竞争力。
- **推理效率与存储**（在 128K tokens 历史长度下，MemT-1.0-Nano）：
  - 相对于全上下文 KV cache，端到端推理**约 1.86× 加速**。
  - 持久化存储**约 66× 缩减**。
  - 随着历史变长，推理延迟与存储开销的优势进一步增大。
- **智能体任务**：将检索到的"程序性经验（procedural experience）"内化进原生记忆以指导后续动作。在 **SWE-bench Verified** 与 **SWE-bench Multilingual** 上，相比无经验的智能体，提升了平均任务成功率与多次运行间的一致性，并达到与显式文本经验相当的表现。
- **缩放研究**：与通用基础模型不同，记忆基础模型引入"记忆容量"作为关键缩放维度；作者就记忆容量与训练数据进行了缩放研究（scaling studies）。

## 创新点
1. 提出 **MemT-1.0** 记忆基础模型架构，改进了记忆的存储与利用机制，并引入 **Mixture-of-Storage (MoS)** 实现可扩展的记忆容量。
2. 采用基于 **Gated DeltaNet-2** 的稳定 token 级记忆更新机制，替代了 Metis 的 token 选择机制。
3. 设计了"backbone 准备 + CPT + 后训练"的两阶段训练流水线及配套基础设施，支持记忆隔离的有状态推理。
4. 将原生记忆的角色从"信息回忆（information recall）"扩展到多步执行中的"程序性指导（procedural guidance）"，在编码智能体任务上得到验证。
5. 探讨了记忆基础模型特有的"记忆容量"缩放维度。

## 局限与讨论
- 摘要/引言未显式给出一节专门讨论局限，本小节大部分**原文未明确提供**（全文未完全抓取到）。
- 从已抓取内容可归纳的讨论点：
  - 相对检索增强方法为"保持竞争力"而非全面超越，说明在部分场景下外部检索仍有优势。
  - 效率/存储优势主要在长历史（如 128K tokens）下体现，短历史下优势相对有限。
  - 模型参数规模较大（Nano 36B / Flash 288B），对训练与部署资源有一定要求。
- 作者将该工作定位为"迈向记忆原生演化的务实一步（a practical step）"，暗示仍是阶段性成果。
