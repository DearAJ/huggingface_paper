Group-in-Group Policy Optimization for LLM Agent Training: https://arxiv.org/abs/2505.10978

Proximity-based multi-turn optimization: Practical credit assignment for llm agent training: arXiv preprint arXiv:2602.19225.

How to allocate, how to learn? dynamic rollout allocation and advantage modulation for policy optimization.

Agentic reinforced policy optimization.

Agentic entropy-balanced policy optimization.

From 𝐥𝐨𝐠 𝝅 to 𝝅: Taming divergence in soft clipping via bilateral decoupled decay of probability gradient weight.

Maspo: Unifying gradient utilization, probability mass, and signal reliability for robust and sample-efficient llm reasoning.

# 核心痛点

传统 GRPO 等无 Critic 算法将终点奖励平摊到整个轨迹的所有 step 上，导致（1）成功轨迹中的冗余/低效/错误动作被奖励；（2）失败轨迹中的正确决策被惩罚

+ **环境保真度**：Model 仿真环境，合成tool调用返回值、推理状态和反馈信号
+ **数据质量**：要求覆盖面广、多样性、准确性



# 从稀疏反馈的 Verifier 中得到稠密奖励
> 方向定义：为长程多轮 agent 的**中间轮次**自动生成稠密、可用于 RL 的 credit，尽量少依赖外部监督（gold answer / 任务验证器 / 人工步骤标注 / 训练 PRM / 强 judge）。
>

| **监督等级** | **训练时需要什么** | **代表论文** | **定位** |
| --- | --- | --- | --- |
| **L0 全免** | 连 gold answer 都不用 | **SIOP、ReBel** | 真正的"无 verifier"，蓝海但验证难 |
| **L1 免 critic/PRM/judge** | 仍用 gold answer 做终局锚 | **TRACE、TIPS、ABSeeker、IGPO、iStar** | "无额外价值模型"，主流爆发区 |
| **L2 免人工步骤标注** | 用 outcome reward + 自动过程信号 | GiGPO | 更早的先驱，step 级分组 |


#### <font style="color:rgb(34, 34, 34);">【前驱期 2505–2510】奠基工作</font>
##### <font style="color:rgb(34, 34, 34);">GiGPO (2505.10978) </font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：Semantic Scholar</font><font style="color:rgb(34, 34, 34);"> </font>**366 引用**<font style="color:rgb(34, 34, 34);">、NeurIPS 2025、被 ReBel 等当"最强 step baseline"。是"critic-free step 级信用分配"的事实起点；代码即</font><font style="color:rgb(34, 34, 34);"> </font>`verl-agent`<font style="color:rgb(34, 34, 34);"> </font><font style="color:rgb(34, 34, 34);">框架本身（后续大量工作的基建）。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：group-based RL（GRPO/RLOO）在单轮数学/代码上很成功，但迁到多轮 agent 就失效——一条 agent episode 可长达 50 步、2 万+ token，奖励稀疏且常只在结尾出现，单一轨迹级 advantage 广播到所有 step，抹平了 step 间差异。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786938290052-69a95a67-dd24-48fc-9a4e-e6c3638be6cb.png" width="877" title="" crop="0,0,1,1" id="u8ea93c4e" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：Group-in-Group Policy Optimization </font>**<font style="color:#DF2A3F;">两层嵌套的相对 advantage</font>**<font style="color:#DF2A3F;"></font>
    - **episode 级**<font style="color:rgb(34, 34, 34);">：同任务同初始状态采一组完整轨迹，按总回报算宏观相对 advantage（同 GRPO）；</font>
    - **step 级**<font style="color:rgb(34, 34, 34);">：提出 </font>**anchor state grouping**<font style="color:rgb(34, 34, 34);">——回溯识别多条轨迹里"重复出现的相同环境状态"（因无效动作/绕圈导致反复回到同一网页/房间/场景），把源自同一状态的动作归为一组，算微观相对 advantage。</font>
    - <font style="color:rgb(34, 34, 34);">两者用系数 ω 加权。关键洞察：同任务下轨迹常反复撞到相同状态，这些天然提供了 step 级分组基准，无需额外 rollout 或 critic。</font>
+ **实验场景**<font style="color:rgb(34, 34, 34);">：ALFWorld（具身）、WebShop（网购）、search-augmented QA；backbone Qwen2.5-1.5B/3B/7B-Instruct。另有 VLM 扩展（Sokoban、EZPoints）。</font>
+ **实验结果**<font style="color:rgb(34, 34, 34);">：比 GRPO 在 ALFWorld </font>**>12%**<font style="color:rgb(34, 34, 34);">、WebShop </font>**>9%**<font style="color:rgb(34, 34, 34);">；QA 上 3B 达 42.1%、7B 达 47.2%；</font>

<font style="color:rgb(34, 34, 34);">全程与 GRPO </font>**同等显存、相同 rollout、几乎零额外时间**<font style="color:rgb(34, 34, 34);">。ω 敏感性：0.8 最佳，0.2–1.0 区间稳健。</font>

+ **<font style="color:rgb(34, 34, 34);">局限</font>**<font style="color:rgb(34, 34, 34);">：锚点分组构建依赖</font><font style="color:#DF2A3F;">状态匹配</font><font style="color:rgb(34, 34, 34);">（State Matching）。如果环境过于复杂或包含大量随机文本噪声，不同轨迹之间可能很难捕捉到完全相同的状态 → singleton 崩溃（正是 ReBel 要修的核心问题）。</font>

<font style="color:rgb(34, 34, 34);"></font>

##### <font style="color:rgb(34, 34, 34);">IGPO (2510.14967) </font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：outcome-only RL 在多轮长程有三大病：</font>
    1. **advantage collapse**<font style="color:rgb(34, 34, 34);">：组内 rollout 答案全对或全错时，组相对 advantage 归零、无梯度信号；</font>
    2. **缺乏细粒度 credit**<font style="color:rgb(34, 34, 34);">：后轮依赖前轮，单轮对错被终局信号糊掉；</font>
    3. **样本效率低**<font style="color:rgb(34, 34, 34);">：整条几千 token 的长轨迹仅获得一个终局标量信号，浪费了丰富的中间语义信息。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786950262558-2ed5da49-340a-4c9d-8c81-fdac1ac240d9.png" width="854" title="" crop="0,0,1,1" id="uaa8ea293" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：</font>将 Agent 与环境的每轮交互视作向真实答案（Ground Truth）递进获取信息增益（Information Gain）的过程<font style="color:rgb(34, 34, 34);">。</font>
    - **信息增益奖励（Turn-level IG Reward）**  
在第 $ t $ 轮交互结束时，计算当前模型生成正确答案 $ a $ 的对数概率相比上一轮的边际增量：

$ r_{i,t}^{\text{IG}} = \log \pi_\theta(a \mid q, o_{i,\le t}) - \log \pi_\theta(a \mid q, o_{i,\le t-1}) $

    - **归一化**与**折扣回报**（Normalization & Discounted Return）  
在 Group 内进行 z-score 归一化，随后引入折扣因子 $ \gamma $ 计算未来累计折扣回报：

$ \tilde{R}_{i,t} = \sum_{k=t}^T \gamma^{k-t} \tilde{r}_{i,k} $

    - **<font style="color:rgb(34, 34, 34);">向量化掩码</font>**<font style="color:rgb(34, 34, 34);">高效计算（Custom Attention Mask）IG</font>
+ **实验结果**<font style="color:rgb(34, 34, 34);">：</font>

<font style="color:rgb(34, 34, 34);">在 Qwen2.5-3B/7B-Instruct 上，团队在 4 个域内（NQ, TQ, HotpotQA, 2Wiki）与 3 个域外（MusiQue, Bamboogle, PopQA）</font>

    - **<font style="color:rgb(34, 34, 34);">全面超越强基线</font>**<font style="color:rgb(34, 34, 34);">：IGPO 平均得分达到 60.2，显著优于 GRPO（51.9）、PPO（51.5）、DeepResearcher（53.9）以及 StepSearch（46.0）和 GiGPO（47.2）等过程奖励方法。</font>
    - **<font style="color:rgb(34, 34, 34);">小模型提幅显著</font>**<font style="color:rgb(34, 34, 34);">：在 Qwen2.5-3B 上，IGPO 将平均 F1 从 GRPO 的 32.3 提升至 48.9（+16.6 分），极大缓解了小模型因生成全错答案导致的优势塌陷。</font>
    - **<font style="color:rgb(34, 34, 34);">抑制虚假相关性</font>**<font style="color:rgb(34, 34, 34);">（Spurious Correlations）：利用 Gemini-2.5-Pro 仅凭推理轨迹推断答案的测试证明，IGPO 导出的中间思维轨迹信息密度与逻辑质量显著更高，并非依靠“蒙对答案”。</font>



##### <font style="color:rgb(34, 34, 34);">iStar (2509.19199) </font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：首次把"隐式 PRM"用于 agentic RL，是"无显式过程标注"这条思路的关键源头。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：agentic RL 奖励稀疏且</font>**有时根本不可验证**<font style="color:rgb(34, 34, 34);">；已有过程监督常</font><u><font style="color:rgb(34, 34, 34);">靠有偏标注</font></u><font style="color:rgb(34, 34, 34);">、易产生 </font><u><font style="color:rgb(34, 34, 34);">Reward Hacking</font></u><font style="color:rgb(34, 34, 34);">、或者因</font><u><font style="color:rgb(34, 34, 34);">过细而高方差</font></u><font style="color:rgb(34, 34, 34);">，还有的在</font><u><font style="color:rgb(34, 34, 34);">状态几乎不重叠时直接失效</font></u><font style="color:rgb(34, 34, 34);">（GiGPO）。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786948720044-af4fb324-eb7f-4b06-be92-fbca06e193a1.png" width="717" title="" crop="0,0,1,1" id="u7d5bf8ca" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：</font>

**<font style="color:rgb(34, 34, 34);">implicit step rewards (iStar)</font>**<font style="color:rgb(34, 34, 34);">：</font><font style="color:rgb(0, 0, 0);">jointly </font>**<font style="color:rgb(0, 0, 0);">trains an implicit PRM</font>**<font style="color:rgb(0, 0, 0);"> with the policy model (the LLM agent) using trajectories collected online.</font>

    1. **轨迹偏好采集**：策略模型生成多轮采样轨迹，由 outcome verifier 评分，将高分与低分轨迹配对生成 $ (\tau^+, \tau^-) $。
    2. **基于轨迹的 DPO 训练 PRM**：隐式 PRM $ \pi_\phi $ 以旧策略快照 $ \pi_{\theta_{old}} $ 为 reference，通过轨迹级 DPO 目标进行更新：$ \mathcal{J}_{\text{PRM}}(\phi) = -\mathbb{E}_{(\tau^+,\tau^-)}\left[\log \sigma\left(\beta \log \frac{\pi_\phi(\tau^+\vert{}x)}{\pi_{\theta_{old}}(\tau^+\vert{}x)} - \beta \log \frac{\pi_\phi(\tau^-\vert{}x)}{\pi_{\theta_{old}}(\tau^-\vert{}x)}\right)\right] $
    3. **隐式步骤奖励计算**：计算更新后的 PRM 相对于旧策略对当前步骤动作 $ a_t
 $$  的相对偏好： $$ r_\phi(o_{1:t}, a_t) = \beta \log \frac{\pi_\phi(a_t \mid o_{1:t}, x)}{\pi_{\theta_{old}}(a_t \mid o_{1:t}, x)} $
    4. **两级 Advantage 融合**：策略模型更新时结合全局任务成功率与局部步骤贡献：$ A(a_t) = A_E(\tau) + \alpha A_S(a_t) $
+ **实验结果**：

论文在 **WebShop**（电商导航）、**VisualSokoban**（具身/空间规划）及 **SOTOPIA**（开放社交博弈）验证：

    - **超越前沿模型与强 RL 基线**：在 WebShop 成功率达 86.5%（超出 GiGPO 84.1% 与 PRIME 81.5%），VisualSokoban 达 91.7%。
    - **开放答案场景表现突出**：在 SOTOPIA 困难社交场景中，与 GPT-4o 对抗博弈的成功率较 Vanilla RL 提升高达 48%。
    - **高采样与探索效率**：WebShop 上训练仅 105 步即达到 Vanilla RLOO 最终性能（约 2 倍加速），且轨迹探索平均步数明显变短。



#### <font style="color:rgb(34, 34, 34);">【2603】teacher/hindsight 过程势</font>
##### <font style="color:rgb(34, 34, 34);">TIPS (2603.22293)</font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">："用 teacher 似然做过程势"的关键前身。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：传统强化学习通常只给予结果奖励，模型无法区分中间哪一次搜索是有效的、冗余或有误导，易引发训练崩溃或策略偏离。作者设计"简单、稠密、且理论上不改变最优策略(policy-invariant)"的过程奖励。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786952870777-1e4974eb-c676-45d1-8682-f12e3048c9ce.png" width="747" title="" crop="0,0,1,1" id="ua85ce19d" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：</font>
    - <font style="color:rgb(34, 34, 34);">复用正在训练的模型 Checkpoint 作为 teacher，</font>**<font style="color:rgb(34, 34, 34);">dense reward = </font>****teacher 模型下正确答案似然的增加量**<font style="color:rgb(34, 34, 34);">。计算追加当前 Turn 后，模型生成正确答案的对数似然增量：</font>$ \Delta_k = \alpha [\Phi(S_k) - \Phi(S_{k-1})] $<font style="color:rgb(34, 34, 34);">。</font>
    - $ \Phi(S_k) $**<font style="color:rgb(34, 34, 34);">（信息势能函数）</font>**<font style="color:rgb(34, 34, 34);">：</font>$ \Phi(S_k) = \log P(y^* \mid S_k) $<font style="color:rgb(34, 34, 34);">，模型在状态 </font>$ S_k $<font style="color:rgb(34, 34, 34);"> 下对 Gold Answer </font>$ y^* $<font style="color:rgb(34, 34, 34);"> 的条件对数似然，代表当前收集到的上下文信息对推导出正确答案的“信任度/确定性”势能。</font>
    - **不改变原始最优策略**<font style="color:rgb(34, 34, 34);">：</font>$ \sum_{k=1}^T \Delta_k = \alpha [\Phi(S_T) - \Phi(S_0)] $

<font style="color:rgb(34, 34, 34);">总的额外奖励与中间状态无关，防Reward Hacking（为赚取奖励不断进行无意义的搜索）</font>

+ **实验**<font style="color:rgb(34, 34, 34);">：</font>
    - <font style="color:rgb(34, 34, 34);">benchmark：</font><font style="color:rgb(0, 0, 0);">NQ、TriviaQA、PopQA、2WikiMultiHopQA、MuSiQue、HotpotQA、Bamboogle</font>
    - <font style="color:rgb(34, 34, 34);">base model：Qwen-2.5 7B Instruct；对照 GRPO / PPO。</font>
    - **实验结果**<font style="color:rgb(34, 34, 34);">：一致优于 GRPO/PPO 并大幅提升训练稳定性。以 Qwen2.5-7B-Instruct 为例，相对 PPO </font>**平均 EM +11.8%、F1 +13.6%**<font style="color:rgb(34, 34, 34);">。证明 turn 级信息势 shaping 是解稀疏奖励 credit 的有效通用方案。</font>



#### <font style="color:rgb(34, 34, 34);">【L0 突破期 2605】甩掉 gold answer</font>
##### <font style="color:rgb(34, 34, 34);">SIOP (2605.04984) </font>
+ 提出了一个用于**无校验器（Verifier-Free）** 场景下 LLM Agent 多轮交互信用分配的强化学习框架。
+ **核心问题**<font style="color:rgb(34, 34, 34);">：</font>
    - **监督错配与稀疏奖励**：结果奖励单一，存在严重探索效率低下与梯度冲突问题。
    - **过程监督依赖标准答案**：现有的轮次级奖励塑形方法需要计算中间轮次对“标准答案”或“标准动作”概率的提升。但在大多数开放式 Agent 场景中，缺乏固定的 Gold Answer 或稳定可用的任务校验器。
    - **无校验器 RL 缺乏细粒度归因**：虽然能从模型自身输出分布提取自监督信号（如语义熵、多数投票），但通常只作用于最终回答或整条 Trajectory。

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786955649443-95723d81-0c08-43b8-b9a4-d89e38c45e95.png" width="1027" title="" crop="0,0,1,1" id="SJPzA" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：</font>
    - **<font style="color:rgb(34, 34, 34);">洞见</font>**<font style="color:rgb(34, 34, 34);">：对同一 query 采样多条 rollout，其最终答案本身构成结构化分布：有的反复出现、语义一致、被检索证据支持（可信），有的是低共识噪声。</font>
    - **<font style="color:rgb(34, 34, 34);">核心</font>**<font style="color:rgb(34, 34, 34);">：把语义等价的 Final Outcomes 聚成"</font>**<font style="color:rgb(34, 34, 34);">语义结果模式(Semantic Outcome Mode)</font>**<font style="color:rgb(34, 34, 34);">"，每个 mode 当潜在未来状态，从而把无监督的 outcome 可靠性提升为 turn 级 credit。</font>
    - **<font style="color:rgb(34, 34, 34);">流程</font>**<font style="color:rgb(34, 34, 34);">：</font>
        1. **<font style="color:rgb(34, 34, 34);">采参与聚类</font>**<font style="color:rgb(34, 34, 34);">：采 K 条轨迹，将语义等价的回答合并为一个聚类 c。每个聚类拥有一个原始采样频率</font>$ m(c\vert{}q) = \frac{N_c}{K}
 $，即在采样的 K 条轨迹中，有多少比例的最终回答属于语义模式 c。
        2. **<font style="color:rgb(34, 34, 34);">计算外部证据支持度</font>**<font style="color:rgb(34, 34, 34);">：Agent 在多轮交互中会使用工具检索出外部文档或观察结果。SIOP 利用语义蕴含模型计算这些</font>**<font style="color:#DF2A3F;">检索到的文档对回答聚类 c 的支持程度</font>**<font style="color:#DF2A3F;">，得到外部证据得分 r(c, q)</font>
        3. <font style="color:rgb(34, 34, 34);">结合经验采样频率 </font>$ m(c\vert{}q)
 $<font style="color:rgb(34, 34, 34);"> 与外部文档支持度 </font>$ r(c, q)
 $<font style="color:rgb(34, 34, 34);">，合成</font>**<font style="color:rgb(34, 34, 34);">校准目标分布 </font>**$ q_\theta(c\vert{}q)
 $<font style="color:rgb(34, 34, 34);">：</font>$ q_\theta(c\vert{}q) = \frac{m(c\vert{}q) \cdot \exp\left( \alpha \cdot r(c,q) \right)}{\sum_{c' \in \mathcal{C}(q)} m(c'\vert{}q) \cdot \exp\left( \alpha \cdot r(c',q) \right)} $
            + **<font style="color:rgb(34, 34, 34);">低频对答案</font>**<font style="color:rgb(34, 34, 34);">：既模型偶然采样到了 1-2 次正确答案</font>$ c_{\text{right}} $<font style="color:rgb(34, 34, 34);">（</font>$ m $<font style="color:rgb(34, 34, 34);">很低），但检索到的事实文档强烈支持它（</font>$ r $<font style="color:rgb(34, 34, 34);">很高）。指数项</font>$ \exp(\alpha \cdot r) $<font style="color:rgb(34, 34, 34);">会产生放大效应，拉高</font>$ q_\theta(c_{\text{right}}\vert{}q) $<font style="color:rgb(34, 34, 34);">的权重。</font>
            + **<font style="color:rgb(34, 34, 34);">高频错答案</font>**<font style="color:rgb(34, 34, 34);">：如果某个回答虽然出现频率高，但检索到的文档完全不支持（模型在“高置信度幻觉”），校准机制压低其权重</font>$ q_\theta(c\vert{}q) $
        4. **<font style="color:rgb(34, 34, 34);">奖励计算</font>**<font style="color:rgb(34, 34, 34);">：第 t 轮的动作提高了预测出“高可靠回答 c”的概率</font>$ r_{t,\text{proc}} \propto q_\theta(c\vert{}q) \cdot \left[ \log \tilde{p}_\theta(c \mid \tau_{\le t}) - \log \tilde{p}_\theta(c \mid \tau_{\le t-1}) \right] $
+ **实验**<font style="color:rgb(34, 34, 34);">：</font>
    - **Benchmark**：<font style="color:rgb(0, 0, 0);">NQ、TriviaQA、HotpotQA、2WikiMultiHopQA、MuSiQue、Bamboogle、PopQA</font>
    - **<font style="color:rgb(0, 0, 0);">Base model</font>**<font style="color:rgb(0, 0, 0);">：Qwen3-4B 和 Qwen3-8B </font>
    - **实验结果**
        * **全面超越无校验器基线**<font style="color:rgb(34, 34, 34);">：在 4B/8B Backbone 上，SIOP 的 Exact Match 与 F1 均显著优于 TTRL 与 EMPO。在多跳推理数据集（如 2WikiMultiHopQA、HotpotQA）上提升尤为明显。</font>
        * **逼近有监督上限**<font style="color:rgb(34, 34, 34);">：在不依赖任何标注答案的情况下，SIOP 逼近了有监督算法 IGPO 的表现，甚至在部分多跳任务上超越了基于真实答案的 GRPO。</font>
        * **行为动态优化**<font style="color:rgb(34, 34, 34);">：训练指标表明，SIOP 引导模型进行了更精准的检索，减少了无意义的重复搜索、缩短了输出长度、降低了生成熵，表现出更高的检索效率。</font>

#### 【2607–2608】L1 在工程中
##### <font style="color:rgb(0, 0, 0);">GiGPO 后作 - HGPO：Hierarchy-of-Groups Policy Optimization for Long-Horizon Agentic Tasks</font>
##### TRACE (2607.13988) 
+ 提出了一套无 Critic、无 Judge 的 Agent 强化学习信用分配（Credit Assignment）框架，通过在工具调用边界估计状态价值变化，为长程（Long-horizon）Agent 训练提供密集的 Turn 级奖励信号。  
+ 背景问题
    - 结果奖励（Outcome Reward）的局限：在多轮工具交互的长程任务中，终局奖励（Outcome-only）过于稀疏且方差极高。如果整条轨迹仅凭最终对错赋值，失败轨迹中有效的中间动作会被一锤子打死，成功轨迹中的冗余操作也会被错误奖励。  
    - 过程监督（Process Supervision）成本高：传统 PRM 或 LLM-as-a-Judge 依赖昂贵的步骤级人工/模型打标，且训练出的 PRM 容易出现评分漂移（Score Drift）和 Reward Hacking。  
+ 核心创新
    - 冻结参考模型（Frozen Reference Model）作为探针：无需训练独立的 Critic 或 PRM，直接利用初始化参考模型 $ \pi_{\text{ref}} $ 评估每个步骤前缀（Prefix）下标准答案 $ y^* $ 的对数概率 $ \bar{\ell}_k $。  
    - 对数比例状态价值（Log-Ratio State Value）：将状态价值定义为 $ V(S_k) = \log\frac{-\bar{\ell}_0 + \epsilon}{-\bar{\ell}_k + \epsilon} $，用以衡量当前动作相对初始状态缩小答案不确定性 Gap 的比例。  
    - 消元 TD 信用分配（Telescoping TD Credit）：以相邻状态价值差值 $ \delta_k = V(S_{k+1}) - V(S_k) $ 作为 Temporal-Difference (TD) Turn 级奖励。一阶 TD 在数学上具有 Telescoping（项相消）性质，中间冗余的工具调用无法虚高累积奖励。  
    - 联合优化（Joint Optimization）：结合 $ K $ 步折扣 TD 信用与 GRPO 的 Outcome Advantage，以终局正确性为 Anchor，同时兼顾中间步骤的有效性。  
+ 实验结果

论文采用 Pure RL 训练范式（无 SFT 冷启动、无 Web 实时数据、无强模型 Judge）： 

| **模型与方法** | **BrowseComp-Plus** | **BrowseComp** | **GAIA** | **xbench-DeepSearch** | **四项平均** |
| --- | --- | --- | --- | --- | --- |
| **Qwen3-4B Base** | 7.2 | 3.3 | 24.2 | 19.0 | 13.4 |
| Qwen3-4B GRPO | 30.0 | 5.1 | 38.8 | 44.0 | 29.5 |
| **Qwen3-4B TRACE** | **35.6** | **6.7** | **44.6** | **49.0** | **34.0** |
| **Qwen3-30B-A3B Base** | 8.4 | 4.4 | 34.1 | 20.0 | 16.7 |
| Qwen3-30B-A3B GRPO | 36.4 | 10.8 | 45.6 | 37.0 | 32.5 |
| **Qwen3-30B-A3B TRACE** | **42.6** | **12.9** | **52.0** | **45.0** | **38.1** |


    - **显著超越基线**：在闭卷 BrowseComp-Plus 上，4B 与 30B 模型分别提升至 35.6 与 42.6，均大幅领先传统 GRPO、GSPO 和 GiGRPO。  
    - **零样本开卷泛化**：尽管仅在闭卷合成数据上训练，学习到的搜索策略可无缝迁移至 Open-Web 检索及跨语言任务（xbench-DeepSearch 达 45.0）。  
    - **训练效率高**：相比Outcome-only 方法，TRACE 曲线起飞更早、收敛更快，且能更快在早期训练中建立长程有效交互。





  


# GRPO 变体
+ **<font style="color:rgb(34, 34, 34);">方法论："GRPO 骨架 + 一个修修补补的 advantage"</font>**
    - <font style="color:rgb(34, 34, 34);">把 advantage 切到 turn/step/action 粒度 (ARPO、GTPO、Turn-PPO、TL-GRPO、GAGPO、CAPO、MICA、IGPO)</font>
    - <font style="color:rgb(34, 34, 34);">改分组/归一化(Stratified GRPO、RC-GRPO、GAGPO、MICA、TL-GRPO、F-GRPO)</font>
    - <font style="color:rgb(34, 34, 34);">在 KL 上做文章(FRPO、FIPO)</font>
    - <font style="color:rgb(34, 34, 34);">把 GRPO 移植到新模态/新领域(Stepwise-Flow-GRPO、TL-GRPO、F-GRPO)</font>
+ **基准与数据集**<font style="color:rgb(34, 34, 34);">：</font>
    - <font style="color:rgb(34, 34, 34);">baseline 互踩 GRPO/PPO/RLOO/REINFORCE++/GiGPO;</font>
    - <font style="color:rgb(34, 34, 34);">benchmark 集中在 WebShop、ALFWorld、HotpotQA、BFCLv4、GAIA、AIME </font>
    - <font style="color:rgb(34, 34, 34);">增量以 1~5 个点为主(除 FIPO、RC-GRPO 少数例外)</font>



# 信号来源分类
| **信号来源** | **机制** | **代表** | **监督档** |
| --- | --- | --- | --- |
| ①答案分布/语义共识 | 多 rollout 聚类 + 可靠性 | SIOP | L0 |
| ②信念一致性 | 预测信念 vs 后续观测 | ReBel | L0 |
| ③参考/teacher 模型似然 | gold 前缀似然的 TD/势差 | TRACE、TIPS、IGPO | L1 |
| ④隐式 PRM（DPO） | 轨迹 DPO → step reward | iStar | L1 |
| ⑤答案反向回溯线索 | 从答案倒推关键步 | ABSeeker | L1 |
| ⑥状态/信念分组 | 组内相对 advantage | GiGPO、ReBel | L2/L0 |



