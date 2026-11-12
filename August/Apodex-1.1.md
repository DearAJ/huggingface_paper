# Apodex 1.1: Scaling Agentic Intelligence for Complex Work

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2608.23283 （HTML 全文：https://arxiv.org/html/2608.23283v2）
- **作者/机构**：Apodex Team（署名为 B. An 等共 75 位作者；通讯/提交人 Xinyu Wang）。官方网站 https://www.apodex.ai ，GitHub https://github.com/ApodexAI/FrontierAgent ，模型权重在 Hugging Face apodex/apodex-11 集合。
- **提交日期**：2026 年 8 月 24 日（v1），2026 年 8 月 25 日修订为 v2（本笔记依据 v2）
- **学科分类**：cs.AI（兼 cs.CL、cs.LG）
- **全文获取情况**：已获取 HTML 全文（方法、评测细节均来自正文）

## 背景问题（要解决什么痛点）
- 通用语言模型能推理、综合知识，但"复杂工作"还要求：持续与文件、信息源、可执行代码交互，维护状态、从失败中恢复、交付可验证的成果。
- 论文把这种能力定义为 **working capability（工作能力）**：面向真实目标的、持续且可验证的进展。其衡量单位是"完成的工作"而非一次性回答。
- 现有工作（ReAct、学习式工具调用、交互式 agent 评测）暴露了"答案质量"与"可靠行动"之间的差距：模型即使能说出正确答案，也常无法在变化的环境中长时程地把推理转化为持续工作。

## 框架设计 / 方法
Apodex 1.1 沿两个互补的扩展维度发展工作能力，并以共享执行框架 + AgentOS 支撑。

- **任务契约形式化**：用统一契约 ℰ=(𝒲, W₀, q, 𝒜, 𝒯, Ω, 𝐁, D, V_D) 描述任务。其中 𝒲 为工作区状态空间、W₀ 初始工作区、q 目标、𝒜 动作集、𝒯 状态转移算子、Ω 观测接口、𝐁 资源预算向量、D 交付契约、V_D 任务级验证器。每步动作后 W_{t+1}=𝒯(W_t,a_t)、o_{t+1}=Ω(W_t,a_t,W_{t+1})；最终由 S_D=V_D(W₀,W_H,τ_H) 判定交付是否合格。
- **维度一：Environment Scaling（环境扩展）**：扩展可执行的文件（File）、搜索（Search）、代码（Code）三类世界的多样性、保真度与可验证性。原则是"forward cheap, inverse expensive"——生成器用潜在状态或参考程序廉价地构造并求解世界，agent 只看到渲染后的工作区并须在约束下恢复路径。
- **维度二：Agentic Coordination Scaling（智能体协调扩展）**：训练模型分解长时程任务、委派并行子任务、整合异步结果、修订共享计划、重规划。由 **Agent Team** 在运行时实现：主（lead）agent 先全局推理、分解问题、按需构造专用子 agent（团队由问题诱导而非固定角色目录）。1.1 新增四项能力：异步人工介入、非对称验证（asymmetric verification）、自适应 Max Team Effort（只对薄弱/有争议/关键结论追加独立调查）、证据接地的综合（evidence-grounded synthesis，构建"主张-证据图"再由写作 agent 产出交付物）。
- **显式任务板（Task Board）**：1.1 把分解从内部推理变为显式任务状态，写在 run-scoped 的任务板上（记录目标、依赖、解决状态、指派、返回证据/产物），放在 LLM 上下文之外。
- **AgentOS 执行底座**：维护持久工作区 W_t=(F_t, Q_t, C_t, I_t, G_t, K_t)（文件态、检索证据、可执行态与日志、产物索引、依赖图、运行控制态），提供分级压缩（tiered compaction）、软/硬超时、异步控制通道、受控产物交付（发布租约 + 清单）等机制。
- **训练**：统一 SFT 混合先建立推理、工具、恢复、交付、多 agent 协调的共同行为；随后 Agentic RL 在可执行环境轨迹与协调轨迹上改进长时程决策。真实任务、benchmark 错误、运行失败被归类为能力缺口，Task Pipeline 转成新任务，再由 Environment Scaling 提供对应世界与验证器（论文称此为受控工程循环意义上的 self-evolution，而非无约束自我修改）。
- **两种评测模式**：ReAct（最小脚手架，反映底层模型策略）与 Agent Team（衡量训练后的协调行为 + 额外有组织计算带来的系统级增益）。

## 实验结果
评测分三层：公开 benchmark、内部结构化检索/研究交付 benchmark、HDS6（评价执行过程质量）。核心数字（↑越高越好，分数保留一位小数）：

**专业工作 / 金融（表 3）**
- APEX-Agents：Apodex 1.1 ReAct 34.4、Agent Team 38.5（Apodex 1.0 仅 16.5）；对比 GPT-5.4 33.3、Claude-Opus-4.6 33.0、Gemini-3.1-Pro 32.0。
- GDPVal（胜率）：ReAct 69.5、Agent Team 78.8（Apodex 1.0 为 59.3）。
- FrontierFinance：ReAct 48.7、Agent Team 54.3（Apodex 1.0 为 40.3）；对比 Claude-Fable-5 49.2、Kimi-K3(max) 48.8、GPT-5.6-Sol 46.8。论文称 Agent Team 在该比较表中取得所列最强值。
- YC-Bench（最终净值 USD）：ReAct 为 $1,038,255。

**科学研究 / 通用推理（表 4）**
- FrontierScience-Research：ReAct 55.0、Agent Team 63.3（Apodex 1.0 为 28.3），平均 rubric 分分别 6.7 / 6.9（60 个研究级任务，≥7/10 判通过）。
- BioMysteryBench（17 题 Human-difficult 集）：ReAct 23.5%（4/17）、Agent Team 35.3%（6/17）；Claude Opus 5 报告 49.4%。
- Humanity's Last Exam：ReAct 53.2、Agent Team 56.1。
- DeepSearchQA（F1）：ReAct 88.2、Agent Team 92.4。

**35B Mini 模型（表 5）**
- FrontierFinance / FrontierScience-Research / APEX-Agents：1.1 mini ReAct 40.0 / 45.0 / 24.2；Agent Team 50.2 / 51.7 / 27.7（1.0 mini ReAct 为 33.2 / 25.0 / 15.4）。

**数学（表 6，MathArena 协议）**
- IMO2025 / IMO2026 / USAMO2026：1.1 Agent Team 36.5 / 30.5 / 26.5（参考阈值 35 / 29 / 25，均超过，含 IMO 金牌线）；ReAct 24.3 / 18.5 / 16.0；1.0 Agent Team 仅 12.5 / 13.0 / 5.8。
- IMO-ProofBench：Agent Team Basic 96.7% / Advanced 63.3%（ReAct 80.0% / 46.4%）。

**代码（表 7）**
- Terminal-Bench 2.1：Apodex 1.1 为 70.8（Gemini 3.6 Flash 91.9、Kimi K3 88.3、Claude Opus 5 77.3）。
- SWE-bench Verified：Apodex 1.1 为 77.7（Claude Opus 5 92.2、Kimi K3 80.8、DeepSeek V4 Pro 80.6）。

## 创新点
- 提出并形式化 **working capability**，以"完成的工作"（任务契约 + 验证器）为 agent 智能的衡量单位，而非单次回答。
- 提出两个互补的扩展维度：**Environment Scaling**（扩展可执行文件/搜索/代码世界的多样性、保真度、可验证性）与 **Agentic Coordination Scaling**（把分解、委派、异步整合、重规划作为模型行为来训练）。
- 设计统一执行 harness + **AgentOS** 持久运行时，使轨迹采集、重放、训练、运行时执行共用一套执行契约；引入显式任务板、异步介入、非对称验证、自适应 Max Team Effort、受控产物交付等机制。
- 统一 SFT + Agentic RL 的训练闭环，把环境轨迹与协调轨迹转化为单一策略能力。
- 以显著更小的模型达到"领先性能带"，并提供 35B 的 Apodex 1.1 Mini 证明可本地部署的高效性。

## 局限与讨论
- 论文强调 self-evolution 仅指"受管控的工程循环"，并非模型无约束自我修改——是作者主动设的边界/澄清。
- 在代码类 benchmark（Terminal-Bench 2.1 70.8、SWE-bench Verified 77.7）上并非最强，落后于 Claude Opus 5、Kimi K3 等；在 BioMysteryBench 上也明显落后 Claude Opus 5（35.3% vs 49.4%）。
- 多个专有对比系统的参数量未公开，因此作者刻意以"性能带（performance band）"而非精确排名来陈述其模型规模效率结论。
- 评测刻意不压成单一聚合分，强调工作能力是多目标（正确性、产物完整性、溯源保真、恢复能力、对介入的响应、墙钟时间、总算力），单一答案分只能观测到其投影。
