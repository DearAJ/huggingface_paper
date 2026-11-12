# LongHorizon-Harness: Advancing Long-Horizon Agents for Real-World Tasks

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2608.01964 （HTML 全文：https://arxiv.org/html/2608.01964v1）
- **作者/机构**：Ziyu Ma、Hailang Huang、Shun Zou、Yong Wang、Shidong Yang、Yiming Hu、Fei Wei、XiangXiang Chu（共 8 人）。代码仓库 https://github.com/AMAP-ML/LongHorizon-Harness （AMAP-ML 即高德/阿里巴巴地图团队），项目主页 https://lh-harness.pages.dev
- **提交日期**：2026 年 8 月 3 日（v1），篇幅 29 页
- **学科分类**：cs.CV
- **全文获取情况**：已获取 HTML 全文（方法与实验细节均来自正文）

## 背景问题（要解决什么痛点）
长时程任务难点不在单步，而在跨大量相互依赖步骤维持连贯进展。论文总结三类共性挑战：
- **(i) 误差累积与目标漂移**：早期动作/决策的错误沿轨迹累积，扭曲后续选择，逐渐偏离初始目标。
- **(ii) Context rot（上下文腐坏）**：交互历史越长，相关信息越难检索使用；上下文利用越过临界阈值后性能骤降。
- **(iii) 任务状态丢失**：长任务需要准确、及时的任务状态（待满足的需求、已完成的动作、已产出的产物、从环境发现的事实），但 agent 常无法恢复、保留并更新该状态。

现有 harness（Claude Code、Codex CLI、OpenClaw 等）存在两个结构性缺陷：**任务执行与任务状态管理共用同一不断增长的上下文**；**任务执行与完成判定耦合**——agent 自己执行又自己判定是否完成，错误判定会被记入任务状态并成为后续决策前提。

## 框架设计 / 方法
核心思想：把长时程执行从"单条不断增长的轨迹"**重构为任务状态管理问题**——把任务状态作为显式记录维护在执行之外，**只用从环境独立验证到的事实更新它**，并据当前记录与原始目标推导下一个子任务。

- **Manage-Execute-Audit（MEA）循环**：每一轮由三个角色组成。
  - **Manager（管理者）**：拥有持久任务状态，读取当前状态并定义一个带依赖、约束、验收标准的子任务契约 c_i；可访问原始任务 𝒯、当前任务状态与全部累积审计报告，但**无任何环境接口**（不能观察应用状态、不能调用 GUI/CLI 工具、不能修改环境），其决策完全基于任务状态与审计记录的环境证据。
  - **Executor（执行者）**：在**全新上下文**中只执行这一个子任务，其交互历史在每轮结束后被丢弃。
  - **Auditor（审计者）**：**只读**地独立检查环境状态，判断发生了什么变化、完成了什么、还有什么未满足，产出审计报告 v_i。
  - Manager 用 v_i 更新任务状态 S_{i+1} 并决定是否需要下一轮。跨轮只保留紧凑、已验证的任务状态及其支撑审计报告；执行者原始轨迹被丢弃。
- **任务状态**：结构化记录集合（requirement 表示派生出的目标或约束等）。
- **AgentAdapter**：轻量统一适配接口，在不改动既有系统原生 agent loop 的前提下支持三个角色可互换的后端——跨模型（Claude Opus、GPT、Qwen）与跨 harness（Codex CLI、Claude Code、OpenClaw、Hermes Agent）。
- **预算**：各角色独立预算——executor 每轮限 1800 秒，manager 与 auditor 各限 300 秒；MEA 最大轮数 N_max=25。默认 manager/executor/auditor 用同一骨干模型，以隔离"任务状态管理"本身的贡献。

## 实验结果
在三个长时程 benchmark 上评测（覆盖跨界面协调、桌面工作流、命令行任务），匹配模型后端与评测协议：

**WeaveBench（表 1，114 个需 GUI+CLI 协同的任务；报告 PassRate 与 Overall 均分）**
- Qwen 3.7-Plus + Claude Code 基线：PassRate 51.8%，均分 0.702。
- 同模型、仍以 Claude Code 为执行后端的 LongHorizon-Harness：**PassRate 80.7%，均分 0.835**，且八个任务域全部提升；几乎翻倍于 Claude Opus 4.7 + Claude Code 的最强官方报告值 41.2%。

**OSWorld 2.0（表 2/表 3，Binary=最终分=1 的任务占比，Partial=平均分）**
- Qwen 3.7-Plus：single action 基线 Binary 2.8% / Partial 21.5% → LongHorizon-Harness（hybrid）**Binary 8.3% / Partial 35.2%**（Binary 约 3 倍提升）。
- Claude Opus 4.7（34 任务子集）：single action 20.6% / 55.8% → LongHorizon-Harness **35.3% / 66.9%**（摘要记为从 20.0% 升至 34.3%）。

**Terminal-Bench 2.1（图 3，纯命令行）**
- Qwen 3.7-Plus + Claude Code：**从 69.7% 提升到 77.2%**。
- 以 Codex 为执行后端 + GPT-5.6 Luna：进一步达 **83.1%**。

**成本分析（图 4/5）**
- token 开销随任务而变：WeaveBench 约为基线 2.3×、OSWorld 2.0 约为基线输出 token 的 3.6×，而 Terminal-Bench 2.1 反而少用 24% token 却取得更高成功率——框架不施加固定 token 倍率。
- 角色占比：Manager 仅占 2.8% / 2.0% / 8.1%（三 benchmark），Auditor 占 19.4% / 24.8% / 38.1%（独立验证是主要额外投入）。
- Qwen 3.7-Plus 平均输出 token 从 28.9K 升到 104K/任务。

**WeaveBench Games 子集（表 4，17 任务）**
- Opus 均分 0.680→0.809，Qwen 均分 0.524→0.733；Qwen 基线≤0.04 的全部 6 个任务恢复到 0.30–0.92；Qwen+LH（0.733）甚至超过 Opus+原生 Claude Code（0.680）。
- token：Qwen 从 10.7M 升到 34.3M/任务，而 Opus 从 16.5M **降到** 11.1M——更强模型用更少审计-重规划轮即可满足子任务契约。

## 创新点
- 把长时程执行**重构为任务状态管理问题**：任务状态显式维护在执行之外，只用独立验证的事实更新，并据此 + 原始目标推导下一子任务。
- 提出 **Manage-Execute-Audit（MEA）循环**：Manager 定义子任务、fresh-context Executor 执行、只读 Auditor 独立核验环境状态后再进入下一轮，解耦"执行"与"完成判定"。
- 提出轻量 **AgentAdapter**，在不改动原生 agent loop 下支持模型与 harness 后端互换，三角色可独立配置。
- 跨模型（Qwen 3.7-Plus、Claude Opus 4.7、GPT-5.6 Luna）、跨 harness、跨交互域（GUI+CLI、桌面、纯 CLI）取得一致增益，并证明"agent 能力是完整模型-harness 系统的属性"。

## 局限与讨论
- **不能凭空补齐模型本身不具备的能力**：独立审计能发现错误并触发恢复，但无法提供模型缺失的原子能力。在以单步能力（视觉感知、数学推理、编码、算法设计）为瓶颈的分析类任务上增益较小甚至出现回退；增益大的类别通常需要在长轨迹中保留、检查、修订多个相互依赖的环境状态。
- **成本可能显著上升**：对较弱模型（如 Qwen 3.7-Plus 在 OSWorld）token 开销成倍增加（框架总成本取决于任务及底层模型所需的执行/恢复轮数）。
- 官方报告配置使用了不同的权限设置，作者据此仅将其作为参考点，主结论基于匹配的 Claude Code 对比。
