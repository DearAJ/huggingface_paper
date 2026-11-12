# Thinking Inertia: LLMs Keep Thinking When Told Not To

## 元信息
- **arXiv 链接**: https://arxiv.org/abs/2610.11765 （v2，HTML 全文：https://arxiv.org/html/2610.11765v2）
- **提交日期**: 2026 年 10 月 8 日提交（v1），2026 年 10 月 30 日修订（v2）
- **作者/机构**: Dianqiao Lei、Kevin Qinghong Lin、Pan Lu、Philip Torr、James Zou（5 位作者；机构在已抓取文本中未明确列出，作者含 Stanford/Oxford 背景学者，原文未显式给出机构名）
- **学科分类**: cs.CL（计算与语言）
- **收录**: 被 NeurIPS 2026 接收。项目主页 https://thinking-inertia.github.io/ ；代码 https://github.com/thinking-inertia/code
- **本笔记依据**: arXiv 摘要页 + HTML 全文（摘要、引言、度量定义与主要发现部分；完整实验表格未全部抓取到）

## 背景问题
- LLM 越来越多地内置显式的"思考模式（thinking modes）"，但其对立面"不思考（no-thinking）"受到的关注远少。
- **如何衡量 no-thinking？** 以往工作常用代理指标（proxies）定义不思考：如禁用思考模式、或缺少长推理轨迹。这些代理不可靠——
  - 禁用思考模式仍可能输出推理；
  - 长轨迹可能只是无信息的填充（filler）而非真正推理。
  - 例如 "Let me think about this. The answer should be <ans>" 不含任务特定内容，却会被表层代理判为推理；而一个紧凑的方程式即使输出很短、无特殊 token，也可能暴露显式推理步骤。
- 动机：在推理（长链）之外，"何时直接作答而不展开推理"对效率（延迟/token 成本）与类人自适应推理都很重要。

## 框架设计/方法
将每个响应规范化为"答前文本（pre-answer trace）+ 最终答案"，并在三个层级评估：
1. **Empty-Thinking Rate (ETR)**：离散视角，衡量严格的"仅答案（answer-only）"合规度（是否达到严格的仅答案边界）。
2. **Question–Pre-answer Relevance (QRel.)**：连续视角，用 instruction-aware 的 **Qwen3-Embedding-4B** 计算问题 Q 与答前文本 T 的相似度（提供相关性信号，而非推理的最终证据）。
3. **Explicit Inference Rate (EIR)**：推理视角，用盲判的 **GPT-5.5** 作为 LLM-as-judge 判断 T 中是否含有针对 Q 的显式推理步骤（区分显式推理 vs 改写/相关但非推理文本）。
- 三个层级共同区分：仅答案输出、相关但非推理文本、显式推理。任务准确率（accuracy）独立评估最终答案 A。
- **干预（interventions）谱系**：评估 **6 种 prompting 干预设置**（涵盖原生 no-think 行为、推理再引发、语义抑制、结构前缀强制；M1–M5 以答前文本为 T，M6 为 assistant-prefill 设置）。
- **评测范围**：6 个广泛使用的 LLM（含开源与商用），3 类问题：布尔（Boolean）、多选（multiple-choice）、开放式（open-ended）。
- **人类验证**：人工边界审计印证 ETR；5 位标注者的相关性判断与 QRel. 高度一致；其显式推理标注与主判官 GPT-5.5 的 EIR 达到 **98.8% 平均二元一致率**；EIR 在换用不同模型族的独立判官下仍稳健。

## 实验结果
主要四点发现：
1. **显式 no-think 控制不等于可靠的 no-thinking 控制**：即使前沿 LLM 如 **DeepSeek-V4-Flash（284B）**，在其禁用思考设置下，开放式响应中仍有 **99.9%** 保留可见的显式推理。
2. **"思考惯性（Thinking Inertia）"**：可见推理更多由任务的答案空间决定，而非由开关决定。在原生与严格 no-thinking 设置下，可见推理在布尔任务上最低、多选任务居中、开放式任务最高。
3. **no-thinking–性能权衡**（开放式任务）：更强的仅答案指令能提升仅答案合规、降低可见推理，但也可能降低准确率。具体地，在 5 个完全禁用思考的 LLM 上，严格仅答案 prompting 把平均 ETR 从接近 0 提升到约 **40%**，同时平均准确率下降约 **15 个百分点**。
4. **答案空间很重要**：对同一问题，事实型/候选可见（candidate-visible）的格式比开放生成格式更容易产生仅答案响应；说明该现象不能仅用基准难度或答案长度来解释。
- 总体：布尔与多选任务上准确率相对稳定；开放式任务暴露"仅答案合规 vs 任务准确率"的权衡。

## 创新点
1. 提出把 no-thinking 当作一种**可观测的响应级行为**来研究，而非依赖表层代理，并给出三层级度量（ETR / QRel. / EIR）区分仅答案、相关非推理、显式推理。
2. 发现并命名 **"Thinking Inertia（思考惯性）"**：显式推理在严格控制下仍持续，且随答案空间开放而更普遍。
3. 系统评测 6 种干预 × 6 个 LLM × 3 类问题，揭示 no-thinking–性能权衡与答案空间效应。
4. 用人类研究验证度量有效性（98.8% 与 GPT-5.5 判官一致）。
5. 将 no-thinking 确立为一项与推理能力并列、需系统评估的非平凡能力。

## 局限与讨论
- 摘要/引言未设独立"局限"小节，本小节大部分**原文未显式提供**（完整讨论章节未全部抓取到）。
- 可归纳的讨论点：
  - EIR 依赖 LLM-as-judge（GPT-5.5），虽有人类验证与跨模型族稳健性检查，但仍可能引入判官偏差。
  - QRel. 仅提供相关性信号，不能单独作为推理的最终证据，结论依赖 EIR 与 ETR 结合。
  - 结论基于特定 6 个模型与 3 类任务，对更多模型/任务的泛化性未在摘要中完全展开。
