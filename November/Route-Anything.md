# Route Anything: A Study of Model and Policy Routing for General-Purpose AI Systems（Route Anything：面向通用 AI 系统的模型与策略路由研究）

## 元信息

- **论文标题**：Route Anything: A Study of Model and Policy Routing for General-Purpose AI Systems
- **链接**：
  - arXiv 摘要页：https://arxiv.org/abs/2611.05275
  - arXiv HTML 全文：https://arxiv.org/html/2611.05275v1
- **作者**：Pengfei Zhou、Yixing Ma、Hexin Wang（通讯）、Weidong Tang、Zhenglin Wan、Wangbo Zhao、Yang You（通讯）
- **机构**：National University of Singapore, HPC-AI Lab（新加坡国立大学 HPC-AI 实验室）；InfRec, Cardinal AI Lab；University of California, Berkeley；Hong Kong University of Science and Technology。通讯邮箱：wanghexin@infrec.tech、yangyou@nus.edu.sg。
- **提交日期**：2026 年 11 月 3 日（v1）
- **学科分类**：cs.CV（Computer Vision and Pattern Recognition）
- **许可**：CC BY-NC-SA 4.0
- **笔记说明**：基于 arXiv 摘要页与 HTML 全文（含第 1–2 节正文、关键发现列表、附录 A 定理证明、附录 B 数据矩阵与路由器细节、Table 6 逐基准表）抓取撰写。以下数据均来自抓取到的正文与表格。

## 背景问题

- AGI 需要在多样的领域与模态上理解、生成、推理与行动。与其依赖单一模型掌握所有能力，一条更有前景的路径是**把一个异构生态（专用模型、工具、策略，统称"候选 candidate"）编排成统一的 AI 系统**。
- 核心问题：AI 系统如何为每个任务**动态选择并协调最有能力的候选**，同时平衡性能与计算成本？
- 现有路由研究大多只针对语言模型（如升级到更强模型的 cascade、从偏好中学习的 router、连接二者的质量–成本公式），**跨模态路由是否有效仍是开放问题**。本文提出统一框架 **Route Anything** 来系统研究跨模态的模型与策略路由。
- 研究围绕三个问题展开：**路由能带来多大增益？学习型 router 效果如何？何时"更便宜的路由"就已足够好？**

## 框架设计/方法

### 1. 统一的路由理论（第 2 节 / 附录 A）
- 把资源价格与质量约束同最优的模型/策略选择联系起来；刻画"router 所能观测到的信息如何决定其可达性能"。
- 候选运行候选 u 产生价值 V_u ∈ [0,1]（基准指标）、消耗资源 D_u > 0（美元或秒）；router g 在状态下按概率选择候选；定义效率 η(g) = E_g[V] / E_g[D]。
- **信息阶梯（Proposition 3，Information ladder）**：router 可达性能是一条从"最佳固定候选（best fixed candidate）"到"逐输入 oracle（per-input oracle）"的阶梯——V*(σ₁) ≤ V*(σ₂) ≤ V*(x) ≤ E[max_u W_u]。观测越细，潜在收益越高；最后一段差距是任何可观测状态都无法预测的"结果噪声"。
- **价格、参考与地板（Theorem 2）**：在有价格时，最优 router 在每个状态只选最大化 v(s,u) − λc(s,u) 的候选；用 Dinkelbach 方法 + 线性规划对偶证明；在"质量地板 Q ≥ −ε"约束下，最优路由等价于在由参考模型与容忍度确定的单一价格 λ 下路由。
- **学结果而非学赢家（Proposition 4，Outcomes, not winners）**：应回归每个候选的结果期望 E[V_u|x]（最小化交叉熵/平方误差得到 m_u(x)=E[V_u|x]），而非分类"谁是赢家"；后者可能损失近一半价值区间（因胜者标签常被延迟/平局打破规则主导）。
- 还提供"认证质量地板（Proposition 5，Certified quality floor）"等结果。

### 2. 全模态轨迹矩阵（Omni-Modal Trajectory Matrix，第 3 节 / 附录 B）
- 让候选池中每个候选在**相同输入**上运行，构成 25 个基准、涵盖 7 大类：
  - **text QA**：MMLU-Pro、MMLU-Redux、SuperGPQA、GPQA、AA-LCR、AA-Omniscience、HLE（文本子集）；
  - **image-text QA**：MMMU-Pro、MMMU、VLM-ExecRouterBench；
  - **image generation**：GenEval2、Qwen-Image-Bench、LongT2I、OneIG、WISE；
  - **video generation**：VBench-2.0；**speech generation**：Seed-TTS 英文与中文；
  - **code and workplace agents**：AA-Briefcase、AutomationBench、GDP.pdf、GDPval、SciCode、Terminal-Bench；
  - **robot manipulation**：RoboTwin 2.0（50 个双臂任务，clean 与 randomized 两种场景）。
- 规模：**374,401 次评分运行（scored runs）**，涵盖 **103,307 个任务**、**452 个候选配置**（候选在模型、harness、effort 上有差异），每个池 3–7 个候选。
- 候选举例：语言模型 Claude Opus 5.5、GPT-6.1 Sol、DeepSeek V4.1 Flash、GPT-6 Astra；图像生成器 FLUX 3、Seedream 5.0 Pro、MAI-Image 2.6、GPT Image 2.5、Qwen-Image 3 Pro、Grok Imagine 2.0；视频生成器 MiniMax H3、Seedance 2.5、Wan 3.0；语音生成器 Seed Audio 1.0、Fish S2.1 Pro、Gemini 3.8 Flash TTS；机器人策略 π0.5、GigaBrain、Spatial Forcing、X-VLA、X-WAM、PatchWAM、ME-Dex（均在 50 个 RoboTwin 任务上联合训练）。
- **数据划分（附录 B.2）**：全集 104,050 个决策，按来源组以 70/10/20、种子 42 划分（保证相同输入不跨划分；RoboTwin 按任务族划分为 35 训练/5 开发/10 测试），得到训练 72,351 / 开发 10,630 / 测试 21,069 条记录；第 4 节 router 研究使用首批发布的 18 个基准。

### 3. 学习型 Router（附录 B.3）
- **能力树 router（capability-tree router）**：叶子由 GPT-6.1 Sol 在训练输入上标注（只看输入、输出要求与叶定义，不看候选结果）；分类器为 **Qwen3.5-4B + LoRA（rank 8）**，在全部 18 个基准上一起训练 3 epoch，batch 16，学习率 1e-4（adapter）/1e-3（head）。
- **结果 router 与类型 router（在 11 个基准上）**：Qwen3.5-4B + LoRA（rank 16, α=32），AdamW，学习率 2e-5 / 3e-4，6 epoch，多种子（42/13/2026/7）。结果 router 损失为"归一化结果交叉熵 + 4× 任务内分差平方误差"；类型 router 预测 68 种能力类型之一。

## 实验结果

**关键发现（Key findings，来自第 1 节与各节；测试集为 24 个基准 19,814 条测试输入，部分实验用 16 个基准）**：

1. **路由潜在增益大（第 3 节）**：逐输入 oracle 比"逐基准最佳固定候选"平均高 **5.4 分**，范围从语音生成的 0.7 分到视频生成与机器人操作的 18–19 分。即便事后选"逐能力类型最佳候选"，也只弥合了 8% 的差距。
2. **学习型 router 超过最佳固定候选（第 4.1 节）**：整体高 **0.3 分**，每个输出组最高高 **2.1 分**。一个跨 16 个基准的能力树 router 得 **79.98**，固定候选 79.68，oracle 84.80；增益主要来自 agents、视频与图像生成；在 GenEval2 与 Qwen-Image-Bench 上，124 个 router 配置无一显著超过固定生成器。
3. **机器人上 router 能"读场景"并省时（第 4.2 节）**：从 3 帧相机图像判断——clean 场景派给 PatchWAM、randomized 场景派给 ME-Dex，成功率 **71.6%** vs 最佳单一策略 70.5%，且**策略时间少 24%**。
4. **学结果而非学赢家（第 4.3 节）**：相同输入与骨干下，回归各策略成功率达 71.6% 机器人成功率，而分类赢家仅 42.8%（因 94% 的赢家标签由延迟平局打破规则决定）。
5. **在相同输入上比较候选（第 4.4 节，matched comparisons）**：要求每个训练输入都有分数的固定基线会淘汰在 1,122 条 HLE 中缺 11 条分数的 Opus 5.5，并把学习型 router 的 −0.4 分劣势转为 +2.4 分优势。
6. **更便宜的路由往往够用（第 5.1 节）**：相对参考模型，路由在四个 testbed 上平均提升效率 **8.82×**，其中代码 **25.5×**、文本 QA **7.1×**。在 CodeRouterBench 上，为零质量损失拟合的 router 从不调用 Opus 参考；认证零损地板在全部 200 个划分上保持、中位 19.4×。图像生成中，价格路由在半分以内把成本降 26%。
7. **更丰富候选/更细状态仅在超过被替代者时才值得（第 5.2 节）**：在 CodeRouterBench 上，一次失败使下一模型成功率大致减半；免费的编译器门控带来 4.09× 美元效率，而按标价计费的 LLM 裁判只有 0.30×。

**逐基准潜在增益（Table 6 摘录，Fixed→Oracle 的 Gap）**：
- 视频 VBench-2.0 18.93（62.43→81.36）、机器人 RoboTwin 2.0 18.40（70.45→88.85）、Terminal-Bench 15.38、GDP.pdf 15.00、GenEval2 13.69 等为高增益项；
- 语音 Seed-TTS(en) 0.61、LongT2I 0.75、Seed-TTS(zh) 0.72 等为低增益项。

## 创新点

1. **统一的路由理论**：把资源价格、质量约束与最优模型/策略选择联系起来，并用"信息阶梯"刻画可观测信息如何决定可达性能；给出含价格、参考与认证地板的最优路由刻画（Dinkelbach + LP 对偶）。
2. **全模态轨迹矩阵**：452 个候选配置在 25 个基准的相同输入上运行、374,401 次评分运行，首次在"同输入矩阵"层面系统量化跨模态（文本/图文理解、图像/视频/语音生成、代码与办公智能体、机器人操作）路由。
3. **量化跨模态路由潜力**：给出潜在增益（比最佳固定候选高 5.4 分）、学习型 router 的实际增益（整体 +0.3、视频 +2.1），并指出任务类型标签只能捕获约 8% 的潜在增益。
4. **"学结果而非学赢家"的训练目标洞见**，以及"相同输入下匹配比较（matched comparisons）"对公平评测的重要性。
5. **效率结论**：路由相对参考模型平均提效 8.82×、代码最高 25.5×，并在机器人上以少 24% 策略时间匹配/超过最佳单一策略。

## 局限与讨论

- 作者自述的边界：**多数矩阵未记录成本**，因此第 2–4 节的多数实验在 λ=0（无成本）下进行，成本侧仅由含价格记录的两个代码/文本 testbed（如 CodeRouterBench）补充。
- **学习型 router 的整体增益较小（+0.3 分）**，且在图像生成（GenEval2、Qwen-Image-Bench）上没有任何配置显著超过固定生成器；增益高度依赖具体输出组/模态。
- **评测目标与基线选择显著影响测得增益**（第 4.4 节：是否要求每个输入都有分数、是否在相同输入上匹配比较，会把结论从劣势翻转为优势），说明路由评测对协议敏感。
- 已退役模型 Fable 5.1 被排除出所有评测，用其训练的 router 以 Opus 5.5 替代（附录 B.3），属实现层面的处理约束。
- 本笔记未抓取到的内容：第 3–5 节完整表格（如 Table 1–5 的全部数值）、附录 A 后续命题的细节与附录 B 其余部分，仅依据抓取到的正文、关键发现与 Table 6 概述。（原文部分表格数值未完整抓取）
