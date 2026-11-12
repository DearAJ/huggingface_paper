# ProxMO：Proximity-Based Multi-Turn Optimization

## 元信息
- **arXiv**：https://arxiv.org/abs/2602.19225 （v1，2026-02-22）
- **标题**：Proximity-Based Multi-Turn Optimization: Practical Credit Assignment for LLM Agent Training
- **作者/机构**：Yangyi Fang、Jiaye Lin、Xiaoliang Fu、Cong Qin、Haolin Shi、Chang Liu、Peilin Zhao（通讯）；Tsinghua University（清华大学）、Fudan University、Peking University、Lanzhou University、Tencent Inc.、Shanghai Jiao Tong University 等（含同等贡献）
- **代码**：https://anonymous.4open.science/r/proxmo
- **笔记来源**：基于 arXiv 摘要页 + HTML 全文（方法、公式与主结果表格均抓取到）

## 背景问题
多轮 LLM agent 已是生产系统（客服、电商辅助、交互任务管理）关键，需从大量随机噪声中区分高价值信息信号以实现样本高效训练。现实中：琐碎任务失败可能只是随机抖动，高难任务成功才是真正能力突破。但已有 group-based 策略优化**僵硬地只用离散 batch 内的统计偏差**，任务难度波动时常误分配信用，体现在两级：
1. **episode 级（上下文盲归一化）**：标准 z-score 对 75% 成功率组里的失败与 25% 成功率组里的成功赋予相近 advantage 幅度，忽略信息异质性。
2. **step 级（硬边界分组）**：现有方法（GiGPO）用精确匹配/相似度阈值做离散分组 → 严格准则产生 singleton（无法比较基线），宽松准则又对语义差异大的状态等权，存在不可调和的权衡。

## 框架设计 / 方法
ProxMO 在两级注入全局上下文，即插即用兼容标准 GRPO。

1. **Episode 级：成功率感知的 advantage 调制（PSC, Polarized Signal Controller）**（式 6-7）：
   设组经验成功率 p、失败率 d=1−p，缩放权重
   w(R,p) = 1 + β·f(R,p)，
   f(R,p) = Sigmoid(d^α) − 0.5（若 R=1，成功）；Sigmoid(−p^α) − 0.5（若 R=0，失败）。
   调制后 Ã^E(τ_i) = w(R(τ_i), p)·A^E(τ_i)。
   效果：低成功率组里的成功被**放大**（巩固罕见突破），高成功率组里的失败被**衰减**（减少噪声惩罚）。β 控强度、α 控陡度。
2. **Step 级：邻近度软聚合（PSA, Proximity-based Soft Aggregation）**（式 8-11）：
   不再把状态切成离散组，而是按语义邻近度连续加权聚合所有状态的回报作基线。
   - 折扣回报 R_t^(i) = Σ_{k=t}^T γ^{k-t} r_k^(i)。
   - 用 **TF-IDF 向量 + L2 归一化余弦相似度** sim(s_t^(i), s_t^(j)) 衡量状态语义相似度（高效可扩展）。
   - 温度缩放 softmax 权重 w_ij（限制在同任务 episode 组 𝒢(i) 内），软基线 B_t^(i)=Σ_j w_ij R_t^(j)，step 级 advantage A^S(a_t^(i)) = R_t^(i) − B_t^(i)。
   - τ→0 退化为最近邻（近似精确匹配），τ→∞ 退化为 episode 级基线；实用 τ=0.1。
3. **统一目标**（式 12-13）：A(a_t^(i)) = Ã^E(τ_i) + ω·A^S(a_t^(i))（ω=1 默认），用 clipped PPO 目标 + KL 惩罚优化。

超参：α=4.0、β=0.1、τ=0.1、γ=0.95、ω=1.0、N=8。

## 实验结果
**基准**：ALFWorld（具身，3827 实例、6 类家务）、WebShop（网购，110 万商品、12k 指令）。
**基线**：闭源 GPT-4o、Gemini-2.5-Pro；prompting ReAct、Reflexion；RL 的 GRPO、GiGPO。
**backbone**：Qwen2.5-1.5B/7B-Instruct，3 seed 平均（表 1）。

- **Qwen2.5-1.5B**：
  - ALFWorld All：GRPO 70.3、GiGPO 85.2、**ProxMO 90.6**（相对 GRPO +28.9%）。
  - WebShop Score / Succ.：GRPO 73.1/52.2、GiGPO 81.7/62.3、**ProxMO 85.3/67.1**（Succ. 相对 GRPO +28.5%）。
  - 子任务上提升最大的是长程/易混任务：Look +85.8%、Pick2 +74%、Cool +41.6%（相对 GRPO）。
- **Qwen2.5-7B**：
  - ALFWorld All：GRPO 79.8、GiGPO 89.5、**ProxMO 94.5**（相对 GRPO +18.4%）。
  - WebShop Score / Succ.：GRPO 79.2/67.2、GiGPO 85.5/74.8、**ProxMO 87.2/76.5**（Succ. 相对 GRPO +13.8%）。
- 论文称：训练后的 1.5B/7B 小模型在部分指标上可匹配甚至超过 GPT-4o、Gemini-2.5-Pro（参照：GPT-4o ALFWorld All 48.0 / WebShop Succ. 23.7；Gemini-2.5-Pro 60.3 / 35.9）。
- **超参敏感性**（图 3）：α、β、τ 在较宽区间内稳定，最优 α=4.0、β=0.1、τ=0.1，且同一配置跨不同规模/任务近最优 → 利于大规模部署（免繁琐调参）。
- 消融（摘要/正文）：两机制独立且协同有效，计算开销可忽略。

## 创新点
- 面向工业部署的实用框架：episode 级成功率感知调制 + step 级邻近度软聚合，把全局上下文（任务难度、语义邻近）注入信用分配。
- 用连续软聚合替代 GiGPO 的硬边界分组，消除 singleton 退化与等权缺陷；用 TF-IDF 余弦相似度，轻量高效可扩展。
- 即插即用兼容标准 GRPO，开销可忽略，便于现有工业训练管线低摩擦接入。

## 局限与讨论
- step 级相似度用 TF-IDF（词频统计），对深层语义/同义改写的刻画弱于嵌入模型（原文为效率权衡而选 TF-IDF）。
- 仅在 ALFWorld、WebShop 两个具身/网购基准验证，未覆盖搜索增强 QA 等其他 agentic 场景。
- 代码为匿名仓库链接（投稿期）。
- episode 级调制假设二值奖励 R∈{0,1}；α、β、τ 虽稳健但仍需设定。
