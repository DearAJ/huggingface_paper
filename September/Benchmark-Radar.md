# Benchmark Radar: A Living Database and Search Engine for AI Benchmarks and Evaluation（面向 AI 基准与评测的活数据库与搜索引擎）

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.11115 （v4，最新修订）
- **作者**：Koutian Wu, Wenjun Zhang, Junjie Zhou, Pengqian Han, Qiming Bao, Ergan Shang, Jiayu Wang, Junkai Wang, Wanghan Xu, Songyuanyi Lu, Yixuan Li, Rudy Mizrahi Celekli, Lazizbek Ravshanov, Yaoliang Bian, Xudong Han, Lin Shi（16 位作者；Koutian Wu 与 Wenjun Zhang 为共同第一作者，Koutian Wu 为通讯作者）
- **机构**：Earth-Space-AI / Tacite AI、The University of Queensland（昆士兰大学）、Hangzhou Dianzi University（杭州电子科技大学）、The University of Auckland（奥克兰大学）、Xtracta、Carnegie Mellon University（卡内基梅隆大学）、Xi'an Jiaotong University（西安交通大学）、Tsinghua University（清华大学）、Shanghai Jiao Tong University（上海交通大学）、The University of Hong Kong（香港大学）、Independent Researcher、Gradia、University of Cincinnati（辛辛那提大学）、Westlake University（西湖大学）/ Shenzhen Loop Area Institute、University of Sussex（萨塞克斯大学）、Cornell Tech（康奈尔理工）
- **提交日期**：2026 年 9 月 10 日（v1）；最新修订 2026 年 10 月 27 日（v4）
- **领域**：cs.AI（人工智能）；cs.IR（信息检索）
- **代码**：https://github.com/ktwu01/benchmark-radar
- **项目站点 / 仪表盘**：https://benchmark-radar.org/
- **笔记依据**：基于 arXiv 摘要页 + HTML 全文（已抓取到摘要、引言、相关工作等正文，但部分章节正文被截断）

## 背景问题
- LLM 及其他 AI 系统的基准研究者与开发者需要：找到相关评测、定位其基准数据集与代码、理解所报告分数背后的设置。
- 要查找一个基准、它的数据集或仓库、以及它被使用的报告，需要跨越论文服务器、代码托管站点、数据集中心、厂商发布、博客、基准目录等多个分散资源。
- 现有资源（LLM Stats、OpenCompass、Artificial Analysis）提供排行榜、评测平台和模型分析视图，但把新发布的基准与其任务材料、以及后续在模型报告中的使用连接起来，仍需查阅多个独立资源。
- 基准种类繁多：广义知识推理（MMLU、GPQA、Humanity's Last Exam）、具体能力（SWE-bench 解决真实 GitHub issue、LiveCodeBench 基于日期切分减少污染、Terminal-Bench 测命令行智能体）、领域专用（FinanceBench、MATH、SciBench、LAB-Bench、ChemBench、ESM-Bench 等）。

## 框架设计/方法
- **Benchmark Radar**：一个用于检索与发现 AI 基准的**活数据库与搜索引擎（living database and search engine）**，覆盖 LLM 评测、智能体与工具使用基准、编程、推理、安全以及领域专用评测。
- 系统将三部分结合：
  1. **每日发现（daily discovery）**：持续抓取基准论文、仓库、数据集和发布。
  2. **可搜索的基准目录（searchable benchmark catalog）**：含在模型卡（model cards）与技术报告中的提及、以及分数历史。
  3. **保留来源身份与引用**：读者可检查候选基准及其评测证据。
- **数据来源**：每日发现来自 **37 个来源**——13 个直接连接器（direct connectors）+ 24 个第一方研究与工程信息流（first-party feeds）。
- **目录规模**：目录含 **1,283 条来源记录（source records）**，来自 **4 个基准目录**；在 **790 条记录**上有 **12,916 个数值观测（numeric observations）**。
- 系统保留"每个来源基准一条记录"，包括没有分数、日期或引用的条目；经审查的身份链接（identity links）连接相关记录，同时保留各自独立的测量值。
- **发布内容**：Web 仪表盘（基准排行榜、分数对测量使用的 Pareto 前沿视图、饱和度与趋势视图、每日信息流、可下载证据）、用于离线查询的命令行工具（CLI）、以及可复现的分析。
- 论文还给出一个**完整的先验技术检索（prior-art search）示例**，展示在设计新评测时如何查询目录并检查基准证据。

## 实验结果
- 本文为系统/数据库类工作，核心"结果"是对全目录的普查（census）与审计：
  - 目录含 1,283 条来源记录（来自 4 个基准目录），790 条记录上有 12,916 个数值观测。
  - 每日发现覆盖 37 个来源（13 个直接连接器 + 24 个第一方信息流）。
- 论文审计了全目录，并考察了**基准饱和度（saturation）**、**采纳趋势（adoption trends）** 以及**分数比较的局限（limits of score comparisons）**。
- （具体的饱和度/趋势量化数值、普查明细表格：全文相应章节正文在抓取中被截断，**原文未完整提供到本笔记**，详见论文正文第 2 节及后续与仪表盘。）

## 创新点
1. **把目录检索与每日发现、模型报告中的提及、以及带文档化评测设置和来源的分数整合在一起**，填补了"把新基准连接到其任务材料和后续使用"的空白。
2. 一个**活的（living）** 搜索系统：持续更新，保留来源身份与引用，可检查评测证据。
3. 提供 **Web 与离线（CLI）两种访问方式**及可下载证据，并给出**可复现的全目录普查**。
4. 独特的 **Pareto 前沿视图**：将基准分数与"测量到的实际使用量"对照展示。

## 局限与讨论
- 论文明确讨论了**分数比较的局限**：不同评测设置（提示、数据切分、评测流程）会使跨来源分数不可直接比较，这也是系统强调保留来源身份与评测设置文档的原因。
- 存在**来源集中（source concentration）** 与**评测设置缺口（gaps in evaluation settings）** 问题（论文对此做了审计）。
- 作为"活数据库"，其内容随时间变化，特定快照（如 v4 的 1,283/12,916/790 数字）会随后续更新而改变。
- 本笔记对饱和度、趋势、普查等**具体量化结论未能完整获取**（全文对应正文段落在抓取时被截断）。
