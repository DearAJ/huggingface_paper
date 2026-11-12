# CoworkOS: Scaling Agent Coworkers for Real-world Automation（CoworkOS：面向真实世界自动化的智能体协作伙伴扩展框架）

## 元信息

- **论文标题**：CoworkOS: Scaling Agent Coworkers for Real-world Automation
- **链接**：
  - arXiv 摘要页：https://arxiv.org/abs/2611.03178
  - arXiv HTML 全文：https://arxiv.org/html/2611.03178v1
  - 项目主页：https://coworkos-ai.github.io/
  - 代码仓库：https://github.com/RUC-NLPIR/CoworkOS
  - Demo 展示：https://coworkos-ai.github.io/showcase.html
- **作者**：CoworkOS Team。核心作者（并列共一）：Jiajie Jin（靳佳杰）、Guanting Dong、Chengjun Pan；通讯作者：Tao Gui（桂韬）、Yutao Zhu、Zhicheng Dou（窦志成）等；共 40 位作者。联系邮箱 jinjiajie@ruc.edu.cn、tgui@fudan.edu.cn、dou@ruc.edu.cn。
- **机构**：1 中国人民大学（Renmin University of China）、2 复旦大学（Fudan University）、3 北京大学（Peking University）、4 上海交通大学（Shanghai Jiao Tong University）、5 新加坡国立大学（National University of Singapore）、6 华东师范大学（East China Normal University）。
- **提交日期**：2026 年 11 月 2 日（v1）
- **学科分类**：cs.AI（Artificial Intelligence）、cs.CL（Computation and Language）
- **篇幅/许可**：34 页；CC BY-NC-SA 4.0
- **笔记说明**：基于 arXiv 摘要页与 HTML 全文（含第 1–5 节正文、第 4 节实验表格与作者信息）抓取撰写。以下数据均来自抓取到的正文/表格。

## 背景问题

- "协作型智能体（cowork agents）"正把人–AI 协作推向真实世界自动化，直接交付软件、设计及其他产物（artifacts）。但这类生产力工作同时要求两点，二者兼顾仍是开放难题：
  - **高质量交付复杂生产力任务**；
  - **在长时间跨度上持续监控与决策（continued monitoring and decision-making over long horizons）**。
- 作者把智能体系统的演进归纳为三个阶段：**Agent Workflow**（以推理与工具使用轨迹为中心，如 CoT、ToT、ReAct、Reflexion，以及 DSPy、AFlow、MetaGPT 等框架）→ **Agent Assistant**（引入记忆、消息、代码执行、工具接口，如 LangChain/LangGraph、AutoGen、OpenClaw）→ **Agent Cowork**（人与智能体共同贡献于演化中的产物，如 Claude Code、Codex、OpenHands、DeepSeek Harness，以及 Kimi Code、ZCode、Qoder、Hermes Agent 等）。
- 核心问题：**能否用单一的协作智能体，同时处理复杂生产力任务与长时程的持续监控决策，并保持"人在回路（human-in-the-loop）"？**

## 框架设计/方法

CoworkOS 是一个面向长时程、人在回路任务执行的"协作外壳（cowork harness）"。其核心设计理念是**将任务策略与执行管理分离**："模型自主选择任务执行策略，运行时 harness 负责维护高效、长时程的协作机制"（模型决定行动、委派与验收，运行时维护状态与反馈）。整体由三部分组成：

### 1. 分层智能体运行时架构（三层，Hierarchical Agent Runtime）
- **Main Agent（主智能体）**：面向用户的接口，兼任整体规划者、管理者与执行者；制定任务计划与执行方向、按需拆解工作、评估最终产物与返回结果；完整任务上下文保留在 Main 中。决策公式 a_t^M ∼ π_M(· | g, h_t^M, F_t)。
- **Subagents（子智能体）**：各自维护独立工作上下文，承担有界的委派任务（scoped assignment），可进行大量本地调查与工具调用而不污染 Main 上下文；返回"报告 + 产物引用 + 执行状态"三元组 r_i = (y_i, P_i, σ_i)。委派被视为"证据契约（evidence contract）"。
- **Coordinator Runtime（协调器运行时）**：两类上下文之间的控制面边界；在委派时组装任务上下文，在返回时向 Main 暴露报告、证据引用与执行状态，同时保留完整轨迹可单独访问；以持久化程序状态维护分派、执行状态与待处理反馈（ξ_{k+1} = F_C(ξ_k, e_{k+1})）。
- **同步/异步执行**：同步调用结果留在 Main 调用路径上；异步委派先返回确认与标识符，结果稍后进入 Main 的 Inbox。Main 可在等待结果时继续其他工作。
- **三条通信路径**：Main→Subagent、Subagent→Subagent（最小化的对等通道，异步非阻塞）、Subagent→Main。

### 2. 面向人–智能体协作的可扩展应用生态（Scalable Application Ecosystem）
- 提供基础工具能力：CLI 执行、搜索、文件操作、MCP 接口；并可扩展接入专业应用：文档编辑器、PPTist（演示文稿）、CAD、Blender；覆盖代码、办公产物、三维内容与游戏。
- **共享应用对象（shared application object）**：人与智能体可检视并修改同一份不断演化的文档/模型/场景；人既能通过对话、也能通过直接修改工作本身来参与，从而把"人在回路"嵌入自主执行之中。
- **资源协议（Resource Protocol）**：资源表示为 r = (ι, κ, x, v)（身份、种类、内容、已提交修订版本）；统一的提交准入规则 Commit(r, u, v̂, α)：基准版本一致则版本号 +1，不一致则返回冲突（conflict）而非静默覆盖，使"人机混合编辑"变成可审计的接受变更序列。
- 两条集成路径：CoworkOS 管理的**原生应用**（文本文档、演示文稿、参数化 CAD、Blender 工程）与通过 **MCP 连接的既有软件**（如 FreeCAD、Blender 外部窗口，CAD 与 OfficeHarness 的内嵌视图）。浏览器/桌面交互作为正交的交互模式。

### 3. 面向持久化工作区的状态管理（State Management）
- 保留的任务状态 z_k = (W_k, P_k, R_k)：工作对象与产物引用、进度/决策/证据、进行中的执行与待处理反馈。
- **持久化执行与监控**：长计算作为后台 Jobs 运行，带持久身份、过程记录与日志，可跨对话轮次、上下文重置与 Main 重启继续；确定性脚本通过与基线对比来监控外部状态，未变化的观测无需调用模型。
- **检查点与恢复（Checkpointing and Recovery）**：在完成的轮次检查点化消息轨迹、压缩摘要与分界位置，支持可恢复中断后续作。
- **分层工作区与持久语义记忆（hierarchical workspaces & persistent semantic memory）**：工作文件、关键决策、长运行 Job 日志可共享、复用、可在中断后恢复。
- **记忆与上下文管理**：c_k = Φ(g, h_k, z_k)，tokens(c_k) ≤ L；Main 维护记忆文档（目标、已完成工作、后续行动、重要约束）；在阶段边界进行上下文重建与压缩（compaction）。
- **实验历史台账（Experimental History）**：记录实验目标、委派的 Run、候选变体、产物与指标及其来源引用，区分"从运行产出的指标文件读取的测量值"与"仅在叙述性报告中陈述的值"。

## 实验结果

**评测设置**：以 **GLM-5.3** 为骨干模型（High thinking effort，256K 上下文窗口，单次最多 64K 输出 token）；与三个主流开源 harness 对比：**OpenHands、Codex、DeepSeek Harness（DSH）**。覆盖 **12 个基准、4 个能力域**。

### 基准评测（Table 1，各域为等权平均）
- **Tool Use（工具使用）**：SkillsBench、τ³-Bench Banking、AutomationBench。CoworkOS 域平均 **55.22**，与 DSH 的 55.31 几乎持平，超过 OpenHands（51.33）与 Codex（43.40）；其中在 **SkillsBench 得 64.34（第一）**（OpenHands 63.30 / Codex 53.48 / DSH 56.75）。τ³-Bench Banking：CoworkOS 49.48（DSH 55.67 最高）；AutomationBench：CoworkOS 51.83（DSH 53.50 最高）。
- **Productivity（生产力）**：JobBench、Harvey Legal Agent Bench、GDPVal(Elo)、OfficeQA Pro、WorkspaceBench-Lite。CoworkOS 域平均 **57.54（最高）**，高于 DSH 的 56.13。其中 **JobBench 59.88（第一）**；GDPVal Elo **1164（第一）**（OpenHands 1122 / Codex 925 / DSH 1157）；Harvey 22.74、OfficeQA Pro 65.90、WorkspaceBench-Lite 67.14。
- **Search（搜索）**：WideSearch、BrowseComp。**WideSearch 82.47（第一）**；BrowseComp 81.80；域平均 82.14，与 OpenHands 的 82.22 几乎持平。
- **ML Coding（机器学习编程）**：MLE-Bench-Lite、MLS-Bench。**两项均第一**：MLE-Bench-Lite **76.90**、MLS-Bench **35.62**；域平均 **56.26（最高）**。
- 结论：CoworkOS 在每个域平均都位列前二，并在 Productivity 与 ML Coding 取得最高域平均，在四个 harness 中综合表现最强。

### 真实世界任务定性分析（第 4.2 节）
- 覆盖 **24 个案例**：4 个前端网站、4 个游戏、4 个 Office 演示任务、8 个交互式可视化、2 个 CAD 装配、2 个 Blender 场景；与 Codex、DSH 逐案例对比（如 Courtyard House、Ridge Pack、Paper Glider、Fruit Slicer、Mogao Caves、Aral Sea、CREMA Espresso、Turbofan Engine、Qianqing Palace 等），展示其在信息整合、反馈可读性、组件区分度与空间深度上的优势。

### RSI 自动化后训练案例研究（第 4.3 节，核心亮点）
- 任务：以 CoworkOS（骨干 **GLM-5.3** 作为控制器与数据教师）对 **Qwen3.6-35B-A3B** 进行"递归自我改进（RSI）"驱动的自动化后训练。
- 流程：先在来自多来源的 **1,345 条工具交互轨迹**上做种子 SFT 建立 R0 基线，再进行 **5 轮**反馈驱动的后训练（评估→能力缺口诊断与数据管线修订→数据生成/验证/过滤→SFT→再评估）；用 BM25 任务相似度过滤控制训练/评测污染。
- 评估基准（8 个）：AutomationBench、BFCL v4、OfficeQA Pro、τ³-Bench Banking、Harvey Legal Agent Bench、SkillsBench、GDPVal（归一化累积 rubric 分）、JobBench；聚合为 8 项百分比分的算术平均（每项最多贡献 12.5 分）。
- **结果：5 轮后 8 基准平均分从 33.19（初始 SFT 模型）升至 42.95，提升 +9.76**；最终模型在 8 个基准中 7 个超过初始基线，并在 6 个（含并列）达到六轮中最高分。
- 任务级：Automation 与 SkillsBench 增幅最大，分别较 R0 提升 **33.15** 与 **14.91** 分；BFCL v4 在 R4→R5 提升 9.07 分（对应该轮加强函数调用监督，R5 新轨迹中 61.1% 为函数调用）；Automation 在 R3 占训练轨迹的 78.3%，从 R3 的 39.49 升至 R5 的 44.15。R2 从 R1 检查点继续训练后平均分一度从 36.94 降到 35.90，R3 扩展到 558 条独立轨迹、训练 4 个 epoch 后恢复到 39.22。

## 创新点

1. **一种协作 harness 架构**：把"复杂生产力任务"与"长时程持续监控决策"统一到同一框架；通过"模型选策略 / 运行时管执行"的职责分离，以及 Main–Coordinator–Subagents 三层结构，用集体智能支撑复杂交付、用 Coordinator 支撑长时程监控。
2. **人在回路的可扩展应用生态**：基础工具 + 可扩展专业应用（文档、演示、CAD、Blender）；**共享、可演化的应用对象**让人既能通过对话、也能通过直接修改产物来参与，把人在回路嵌入自主执行；统一的资源协议与修订/冲突检测使人机混合编辑可审计。
3. **持久工作状态与执行连续性**：分层工作区 + 持久语义记忆 + Jobs（保留长运行操作的状态与日志），使工作文件、产物、决策与未完成工作可跨智能体共享复用、中断后准确恢复。
4. **跨域泛化 + RSI 原生能力**：12 基准 4 域 + 6 类真实应用场景上与主流 harness 相当；并在 5 轮 RSI 原生后训练中把 Qwen3.6-35B-A3B 的 8 基准平均分从 33.19 提升到 42.95，验证其作为"RSI 原生协作者"的潜力。

## 局限与讨论

- 作者在结论中指出：RSI 后训练的**增益在不同任务间并不均衡**（如 GDPVal 与 JobBench 整体波动、BFCL 在 R1–R4 一度低于初始基线），联合后训练中存在任务间权衡的变化；图注也说明各轮的变化是"并发干预（concurrent interventions）"而非孤立的因果效应，轮次编号不代表经过的时间。
- 基准结果为"实际所用配置下"的分数，各基准具体任务范围、评分流程与执行上限见附录 A.1（本笔记未逐项抓取附录细节）。
- 本笔记基于全文第 1–5 节与第 4 节主要表格；附录 A.2（案例证据与检视协议）、A.3（自动化后训练运行记录）等的详细内容未逐一抓取到。（原文附录细节未完整抓取）
