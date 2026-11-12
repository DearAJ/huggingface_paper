# DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

> 说明：本笔记基于 arXiv 摘要页 + HTML 全文（v1）撰写。

## 元信息
- **arXiv 链接**：https://arxiv.org/abs/2609.19969 （v1，HTML 全文：https://arxiv.org/html/2609.19969v1 ）
- **作者**：DeepSeek-AI（团队署名；摘要页列出 Anyi Xu 等，含 et al. 共 590+ 位作者；通讯 research@deepseek.com，提交人 Wenfeng Liang）
- **机构**：DeepSeek-AI
- **日期**：2026-09-17 提交（v1）
- **领域**：cs.CL
- **模型**：https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash

## 背景问题
- 长程智能体（long-horizon agents）的普及使模型负载越来越"输入密集"。即便长上下文计算成本已被大幅降低，**prefill 仍然昂贵，巨大的 KV cache 持续挤压 HBM/SSD 容量与数据传输带宽**。
- 计算、存储、带宽三方面需求共同构成进一步降低部署成本的主要瓶颈。
- 具体地，DeepSeek-V4 用"全局注意力分支 + 局部滑窗注意力(SWA)"：足够长序列下全局 KV 主导运行时 KV 占用（受 HBM 限制），部分 KV 为前缀复用而持久化（受 SSD/主机内存限制）。

## 框架设计 / 方法
- **DeepSeek-V4.1-Flash**：多模态 MoE 模型，**552B backbone 参数 + 196B Engram 参数**，原生支持多模态输入，上下文最长 **100 万 token**。
- **Causal Encoder-Decoder (CED) 架构**（受 YOCO 启发）：40 层语言 backbone = 20 层因果编码器 + 20 层解码器；解码器全局 KV 由编码器最终隐藏态投影得到。使模型 **prefill 每 token 仅激活 8B 参数、decode 激活 16B**，对输入密集的智能体场景更省。CED 把 prefill 复杂度从 O(NL) 降到约 O(NL/2)，近似减半。
- **Compressed Sparse Attention 2 (CSA2)**：跨层复用全局 KV（含 main KV 与 indexer K）与 Top-K 索引。三种静态模式：Full（生成全局 KV 并做索引）、Reindex（复用前层全局 KV，用自己的 indexer Q 重打分选新 Top-K）、Reuse（复用全局 KV 与 Top-K 索引直接做稀疏注意力）。相比 V4 的 CSA-HCA 混合架构，V4.1-Flash 用纯 CSA2，并引入 Hierarchical Sparse Indexer 缩小后续索引层搜索域。
- **FP4 KV 缓存**：训练期即用 FP4 全局 KV，仅轻微性能损失。CSA2 + FP4 把**全局 KV 占用降到约 DeepSeek-V4-Flash 的 1/4**，即 **890 bytes / token**（相对 V4-Flash 约 4×、相对 V1 约 437× 的缩减）。
- **SWA Bounded Replay**（部署优化）：近似重建 SWA KV 状态时只重放最近 n_win 个 token（而非 L×n_win），把**持久化 KV cache 占用进一步降到约 V4-Flash 的 1/8**，仅带来少量 prefill 重算。
- 其它架构扩展：Single-Pass mHC（配 Mega-mHC 部署 kernel，激活内存流量减半）、Engram 条件记忆模块、DSpark 推测解码（半自回归草稿 + 置信度调度验证）、模态特定的无辅助损失负载均衡、DeepSeek-ViT 视觉编码器（2D-RoPE、3×3 pixel-unshuffle，支持约 1344×1344）。
- **预训练**：45T token 多模态语料；稀疏注意力在 64K 序列长度从零训练（无 dense attention warmup）。后训练遵循标准 SFT→RL→OPD，**无算法创新**，全部变化在数据管线。

## 实验结果
- **效率**：上下文从 4K 扩到 1M（256 倍）时，单 token Decode FLOPs 仅增 1/4，远低于 V4-Flash 的增长。每个 CSA2 Reuse Mode 层 prefill 仅 15 个 kernel、decode 仅 11 个。
- **Base 模型**：DeepSeek-V4.1-Flash-Base 的世界知识/推理/编码能力可比 DeepSeek-V4-Pro-Base，在 held-out 上提升 5%–10%，而仅用 **1/3 总参数、1/4 激活参数**。
- **推理**：Codeforces 评分 3471（> V4-Flash 3289、V4-Pro 3348）；MathArena Apex Pass@1 65.6%（持平 Kimi-K3 65.6%、V4-Pro 65.3%）；GPQA Diamond 90.9%（> V4-Flash 89.9%）。
- **智能体**：DeepSWE v1.1 达 74.2%（V4-Flash 54.4%，并超 Opus-5 74.0%、GPT-5.6 Sol 73.0%）；Terminal-Bench 2.1 达 90.6%（> Opus-5 89.1%、GLM-5.3 88.2%）；Automation-Bench 54.8%、Agents' Last Exam 31.8%。Terminal-Bench 3.0/4.0 分别 30.0 / 31.2（在最难的科学向任务上仍落后巨型模型）。
- **跨 scaffold 鲁棒性**：在 Claude Code、Codex、OpenCode、Pi、mini-SWE、DeepSeek Harness 等 6 个 scaffold 家族 8 种配置下评测；Terminal-Bench v2.1 在不同 scaffold 下 84.1–90.6。
- 作者称其已能在绝大多数基准上匹配闭源前沿模型，可完成 >95% 的真实任务。

## 创新点
- 在架构、缓存精度、部署三层面联合优化，把 KV cache 压缩推到新极限：全局 KV 降至 890 bytes/token（≈V4-Flash 的 1/4），持久化 KV 降至 ≈1/8。
- **CED** 架构让 prefill 激活参数（8B）显著小于 decode（16B），针对输入密集的智能体负载降本。
- **CSA2**（跨层 KV + 索引复用 + 分层稀疏索引）与 **FP4 KV**、**SWA Bounded Replay** 等组合式工程创新；并引入 Single-Pass mHC、Engram、DSpark 等扩展。
- 后训练不做算法改动，收益主要来自数据管线与环境构建的大规模自动化。

## 局限与讨论
- 在需要专家级领域知识的科学向智能体任务（如 Terminal-Bench 4.0）上，与巨型模型仍有差距。
- 多模态域整体性能与巨型闭源系统相比仍有明显差距。
- 作者指出标准评测日趋饱和：个别基准分数接近顶级模型（如 Fable-5、GPT-6 Astra）并不代表在最难、高难度推理与长尾案例上达到前沿能力。
- FrontierSWE v2 / ProgramBench golden 子集等多/单智能体结果为"初步（preliminary）"。
