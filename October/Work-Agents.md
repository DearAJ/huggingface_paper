# Work Agents: Systems, Capabilities, and Optimization

## 元信息
- **arXiv 链接**: https://arxiv.org/abs/2610.35845 （v1；本文仅有 PDF，无 arXiv HTML 版本）
- **提交日期**: 2026 年 10 月 26 日提交（v1）
- **作者/机构**: Zishan Xu、Enjun Du、Mingqi Gao、Cheng Yang、Junhao Shen、Zirong Chen、Zhiyuan Yao、Yifu Guo、Yuquan Lu、Jinyang Huang、Zhengxi Lu、Siyi Liu、Ziyu Zheng、Yongqi Zhang、Difan Zou、Weiwen Liu（共 16 位作者；机构在已抓取文本中未明确列出，原文未显式给出机构名）
- **学科分类**: cs.LG（机器学习）
- **本笔记依据**: **仅基于 arXiv 摘要页**（该论文无 HTML 全文，PDF 正文未抓取；以下内容均来自摘要）

## 背景问题
- 随着基础模型从"信息生成"演进到"数字工作执行（digital work execution）"，智能体越来越能够跨工具、应用、工作流完成复杂任务。
- **现有研究缺乏统一视角**来刻画智能体如何可靠地完成被委派的工作（delegated work）。
- 本文旨在填补这一空白，提出一个以"工作"为中心、系统级（work-centric, system-level）的视角。

## 框架设计/方法
（以下均来自摘要，细节原文未在摘要中展开）
- **核心定义**：将工作智能体（work agents）视为**结合基础模型与运行时框架（runtime harnesses）**的系统，在有状态工作空间（stateful workspaces）中执行任务并交付可验证的结果（verifiable outcomes）。
- **两个互补维度**刻画工作条件：
  1. **工作持续性（work persistence）**
  2. **工作开放性（work openness）**
- **数字工作的五种形态**：根据任务范围（task scope）与组织方式（organization）将数字工作分为五种形式（具体五类名称原文摘要未逐一列出）。
- **五大核心能力族（five core capability families）**：识别可靠工作执行所需的五类核心能力（具体名称原文摘要未逐一列出）。
- **优化路径**：考察三类优化方法——
  1. 模型优化（model optimization）
  2. 框架优化（harness optimization）
  3. 二者的联合优化（joint optimization）
- **评估分析**：从"工作结果（work outcomes）"视角分析现有评估方法与指标。

## 实验结果
- 本文是一篇**框架/综述型（unified framework / perspective）**工作，摘要中**未报告具体实验数据、数据集或基准指标**。
- （原文未提供具体实验数值——该论文定位为统一框架与分析，而非实证评测。）

## 创新点
1. 提出以"工作"为中心、系统级的统一视角，把工作智能体定义为"基础模型 + 运行时框架"在有状态工作空间中交付可验证结果的系统。
2. 提出 **work persistence** 与 **work openness** 两个互补维度刻画工作条件。
3. 将数字工作分类为五种形态，并识别出五大核心能力族。
4. 系统梳理模型优化、框架优化及其联合优化三条路径，并从工作结果视角审视评估方法与指标。
5. 讨论未来方向：持续性工作委派（sustained work delegation）、对开放环境的适应（adaptation to open environments）、以及工作执行与能力提升之间的交互。

## 局限与讨论
- 作为综述/框架型论文，缺乏实证验证——摘要未给出实验证据来支撑所提框架的有效性（原文未提供实验部分）。
- 五种工作形态、五大能力族的**具体名称与细节未在摘要中给出**，需查阅 PDF 全文（本笔记仅基于摘要，PDF 正文未抓取到）。
- 作者自述的未来研究方向（持续委派、开放环境适应、执行与能力提升的交互）也表明当前框架仍属于理论梳理阶段。
