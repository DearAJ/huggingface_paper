# 总览
如何自主从经验中学习？

+ **自主训练的难点：**
    - 生产环境很少提供干净的学习信号
        * 用户满意不意味着合规；
        * 局部参数更新还可能造成能力遗忘、策略漂移或安全退化
        * 若允许模型依据未经验证的反馈直接修改自身参数，错误经验和提示注入就可能被固化，并在后续任务中持续放大
    - 基础模型的周期性训练可以提升通用能力，却无法及时吸收每个 Agent 每天遇到的私有规则、工具变化和局部经验
+ **目前途径：**

记录运行路径，验证结果与过程，从多条轨迹中提取共性，再决定应更新<u>知识</u>、<u>指令</u>、<u>程序</u>还是<u>模型参数</u>。修改前需回归测试和安全检查。

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786539458897-777fa4d5-2532-49ae-b364-8d645221483c.png" width="647" title="" crop="0,0,1,1" id="u039252dc" class="ne-image">



+ **<font style="color:rgb(44, 62, 80);">如何判断好坏</font>**

可靠评价既要看结果，也要检查达成结果的路径。如预先定义评价量表（Rubric），要求验证器逐项给分、引用轨迹证据，并在证据不足时明确表示不确定。

+ **<font style="color:rgb(44, 62, 80);">四种更新方法</font>**

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1786590278302-1458080e-1724-4a11-8309-ec94240120af.png" width="599" title="" crop="0,0,1,1" id="u8c988cbf" class="ne-image">

+ **<font style="color:rgb(44, 62, 80);">验证、发布、修订与淘汰</font>**
    - **回退**：新经验是否与已有的其他经验冲突，原有本来能通过的案例是否出现回退；
    - **泛化能力**：新经验在测试集尚未覆盖的场景中带来的效果提升；
    - **Token 效率**：完成任务消耗的 Token 成本；
    - **安全性**：规则、隐私和拒绝边界是否随进化漂移；
    - **长期工程质量**：维护复杂度、架构一致性、所有权边界、向后兼容性以及未来迁移和调试负担是否恶化。

只解决了当前失败案例的问题，却在其他已有案例或新领域中退化，不是成功的持续学习。

+ **<font style="color:rgb(44, 62, 80);">关键：如何验证</font>**
    - 更新迭代 **harness**，得到更强的 skill/知识库...
    - <font style="color:#DF2A3F;">更新迭代</font>**<font style="color:#DF2A3F;">验证器（更全面的指标）</font>**<font style="color:#DF2A3F;">，得到更强的反馈信号...</font>



# **Evaluator-Policy Co-Adaptation**
## 总览
### 1. 定义
在 alignment 的迭代过程中，evaluator 被更新以应对当前 policy，而 policy 又被重新优化以适应更新后的 evaluator，从而形成$ E_t→π_t→E_{t+1}→π_{t+1 } $的反馈循环。

### 2. 核心动机
假设最开始 evaluator 是在$ D_0 $上训练的，policy 优化之后变成$ \pi_0 \rightarrow \pi_1 $，$ \pi_1 $产生的数据已经逐渐偏离$ D_0 $，出现$ \text{Evaluator distribution shift} $<font style="background-color:#FBF5CB;">：evaluator 原来在 policy 的旧分布上可靠，但 policy 越训越强以后，开始进入 evaluator 没见过、也没有能力准确判断的区域。</font>

### 3. 关键
Evaluator 跟着 policy 变化并不意味着 reward hacking 消失。

相反，**<font style="color:#DF2A3F;">co-adaptation 本身可能制造新的 reward hacking</font>**。evaluator 和 policy 之间存在一个动态博弈：

+ **第一阶段**：policy 找到弱 evaluator$ E_0 $的 blind spot$ \pi_0 \rightarrow \text{exploit}(E_0) $
+ **第二阶段**：发现问题，于是加强 evaluator$ E_0\rightarrow E_1 $
+ **第三阶段**：policy 又开始适应$ E_1 $$ \pi_0\rightarrow\pi_1 $，然后发现$ E_1 $的新 blind spot。
+ **第四阶段**：再次修 evaluator$ E_1\rightarrow E_2 $，policy 再适应。

于是形成 **repeated adaptation**：$ E_0
\rightarrow
\pi_0
\rightarrow
E_1
\rightarrow
\pi_1
\rightarrow
E_2
\rightarrow
\pi_2
\rightarrow\cdots $

### 4. 难点：<font style="background-color:#CEF5F7;">双方共同找到 shortcut</font>
$ \text{co-adaptation}
\neq
\text{guaranteed alignment} $：如果 evaluator 和 policy 适应得过于紧密、又没有外部 grounding，那么整个 loop 可能不是越来越接近真实 objective，而是一起收敛到一个 shared shortcut；

甚至可能$ \text{co-adaptation}
\rightarrow
\text{shared hallucination / shared exploit} $：

:::color1
1. evaluator 学到一个 proxy： E(x)≈看起来像正确答案
2. policy 学会 π → generate convincing-looking answers
3. 更新 evaluator 时，如果新 evaluator 的训练数据主要来自这个已经被 policy 改造过的 distribution，那么 evaluator 可能进一步学习：“这些看起来很像正确答案的东西就是好答案。”
4. $ \text{policy shortcut}
\rightarrow
\text{evaluator learns shortcut}
\rightarrow
\text{policy learns evaluator's new shortcut} $

:::



policy 的能力足够强，它可以主动利用 evaluator 能力覆盖不足的区域。

+ [Weak-to-Strong Generalization](https://arxiv.org/abs/2312.09390)<font style="color:rgb(0, 0, 0);">：</font>弱 supervisor** **<font style="color:rgb(0, 0, 0);">能监督 strong model 的部分能力，但存在未约束的部分。</font>

<font style="color:rgb(0, 0, 0);">重点：</font>**<font style="color:rgb(0, 0, 0);">弱监督能让强模型学到多少，以及还有多少能力没有被弱监督有效控制</font>**<font style="color:rgb(0, 0, 0);">。</font>

+ [Super(ficial)-alignment](about:blank)<font style="color:rgb(0, 0, 0);">：强模型可能主动利用 evaluator 没能力判断的部分进行 hack。</font>
+ [Monitoring Reasoning Models](about:blank)<font style="color:rgb(0, 0, 0);">：policy 不仅找到 reward hack，还学会规避 monitor</font>



**范式发展**

+ **<font style="color:rgb(0, 0, 0);">Iterative, online evaluator–policy co-evolution</font>**
    - **<font style="color:rgb(0, 0, 0);">起点 - OAIF (2402.04792)：</font>**<font style="color:rgb(0, 0, 0);">DPO 让 LLM 在线打分</font>
    - **<font style="color:rgb(0, 0, 0);">发展 - Self-Rewarding Language Models (2401.10020)</font>**<font style="color:rgb(0, 0, 0);">: 自己产生 preference 后更新自己</font>
+ **<font style="color:rgb(0, 0, 0);">Limits of co-evolution and adversarial evaluator adaptation.</font>**
    - **<font style="color:rgb(0, 0, 0);">URPO (2507.17515)</font>**<font style="color:rgb(0, 0, 0);">：</font>没有<font style="color:rgb(0, 0, 0);">外部 grounding 可能共同收敛到 shared shortcuts</font>
    - **<font style="color:rgb(0, 0, 0);">转变 - </font>**<font style="color:rgb(0, 0, 0);">“E 跟着 π 更新” 变成 “E 专门针对 π 的 exploit 更新”</font>
    - **APO (2311.08045)**：对抗训练，LLM和奖励模型通过极限优化交替更新



### 5. adaptation 分类
1. **Defensive adaptation：**Evaluator 找出 policy 新学会的 exploit X

$ E_{t+1}=E_t+\text{adversarial training against }\pi_t $

2. **Passive adaptation**：Evaluator 用 policy 最近的输出训练自己。

问题是：$ \pi_t
\rightarrow
D_t
\rightarrow
E_{t+1} $里的$ D_t $已经是 policy 影响后的数据。

如果没有外部 ground truth，evaluator 很可能把 policy 创造出来的 shortcut 当成正确规律。

### 6. 解决方向
不要只是同步更新，而应该$ \text{adversarial evaluator adaptation}
 $：<font style="color:#DF2A3F;">evaluator 的更新目标不是更好地拟合 policy 当前产生的数据，而是</font>**<font style="color:#DF2A3F;">专门针对 policy 的 exploit 分布去修补 evaluator blind spots。</font>**

:::info
**APO **把 alignment 写成 reward model 和 LLM 间的 **alternating min-max game**：$ \min_E\max_\pi
\quad \text{某种 evaluator-policy objective} $

+ $ \pi:
\quad
\text{努力寻找 evaluator 的漏洞} $
+ $ E:
\quad
\text{努力修补这些漏洞} $

co-evolution

+ 不是从 policy 变 → evaluator 跟着变
+ 而是 policy attack → evaluator defend → policy attack again → evaluator defend again ...

这种 **competitive game** 比单纯的 passive synchronization 更能够迫使 evaluator 持续修复 blind spots。

:::

### 7. 总结
真正的 research question 是：$ \text{How can }E_{t+1}\text{ improve with }\pi_t
\text{ without learning }\pi_t\text{'s shortcuts?} $

evaluator 不是一个单纯的 **reward function**，而是训练系统中的另一个 **adaptive player**



## **更新 evaluator/oversight 的手段**
+  重新训练 reward model 
+  加一些 adversarial examples ：专门产生某种 evaluator 容易误判的输出，把它们加进训练/测试体系。
+  修改 judge 的评价标准
+  修改 constitution 
+  加新的 monitor





# <font style="color:rgb(34, 34, 34);">Verifier Papers</font>
> 子方向定义：研究如何构造 / 评估 / 加固「可信」的 evaluator·verifier，使其在 agent RL 与 LLM 后训练中提供**可靠、抗 reward-hacking** 的奖励/反馈信号。 
>

---

## <font style="color:rgb(34, 34, 34);">定义</font>
<font style="color:rgb(34, 34, 34);">后训练靠 evaluator 给策略产出打分。但打分器本身会失效、会被钻空子：</font>

+ <font style="color:rgb(34, 34, 34);">判别式 RM 打分不透明、OOD 掉点；</font>
+ <font style="color:rgb(34, 34, 34);">LLM-as-Judge 对"质量接近的候选"分辨力塌缩；</font>
+ <font style="color:rgb(34, 34, 34);">PRM 被证明本质是"流畅度检测器"而非"推理验证器"（Reward Under Attack）；</font>
+ <font style="color:rgb(34, 34, 34);">用不可信 verifier 做 RL → 把 reward hacking 直接放大进策略。</font>

**边界厘清：**

+ **vs RLVR**<font style="color:rgb(34, 34, 34);">：RLVR 是"用可验证结果当奖励"的训练范式（结果级）；本方向研究"验证过程本身 + verifier 自身可靠性"，是 RLVR 的上层质量保证。</font>
+ **vs 一般 reward modeling**<font style="color:rgb(34, 34, 34);">：后者侧重"拟合偏好/正确性"；本方向侧重"可信度、可攻击性、抗 hacking"——是 reward modeling 的</font>**鲁棒性视角**<font style="color:rgb(34, 34, 34);">。</font>
+ **vs credit assignment**<font style="color:rgb(34, 34, 34);">：本方向是 L0（无 verifier 长程 credit assignment）的</font>**奖励供给层与质检层**<font style="color:rgb(34, 34, 34);">。</font>

---

### <font style="color:rgb(34, 34, 34);">Novelty 的三条核心轴</font>
1. **<font style="color:rgb(34, 34, 34);">信号来源（verifier 长什么样）</font>**

从"黑箱打分"到"会调工具的验证 agent"，可解释性与算力成本递增。

| **分类** | **形态** | **代表** |
| --- | --- | --- |
| 1 判别式 RM | 标量打分，黑箱 | 传统 BT-RM，被当 baseline |
| 2 生成式 verifier<br/> (GenRM/GRM) | 用文本生成"评分+批评"，<br/>可解释、吃推理算力 | GenRM、GRAM、DeepSeek-GRM、FlexiVe |
| 3 agentic verifier | 多轮、调工具、双向核对的"验证 agent" | AgentV-RL、Argos、RewardAgent |
| 4 符号/规则 oracle | 确定性规则验证中间步，不用神经判官 | VPRM、VPR |


2. **<font style="color:rgb(34, 34, 34);">攻防立场</font>**

**本方向最有辨识度的轴。**半数论文不是在"造更好的 verifier"，而是在"证明 verifier 会坏 / 检测它何时坏"。

| **立场** | **在做什么** | **代表** |
| --- | --- | --- |
| 1 建设 | 造更强/更可信的 verifier | GenRM 系、AgentV-RL、VPRM |
| 2 拆解 | 证明 verifier 可被攻破、量化脆弱性 | Reward Under Attack、LLMs Gaming Verifiers |
| 3 检测 | 无监督发现 hacking、建 benchmark | TRACE×2、RHB、From Rebound to Remedy |
| 4 加固 | 对抗训练/表征干预把漏洞补回去 | APRM、Adv-RM、Advantage Modification |


3. **<font style="color:rgb(34, 34, 34);">可信度的保证机制</font>**

novelty 常藏在<font style="color:#DF2A3F;">"用什么办法保证可信"</font>里。

    1. <font style="color:rgb(34, 34, 34);"></font>**对抗共演化**<font style="color:rgb(34, 34, 34);">：生成器造难样本、verifier 学识别（APRM、Adv-RM）。</font>
    2. <font style="color:rgb(34, 34, 34);"></font>**符号锚定**<font style="color:rgb(34, 34, 34);">：把可信度外包给确定性 oracle（VPR、VPRM）。</font>
    3. <font style="color:rgb(34, 34, 34);"></font>**随策略共演化**<font style="color:rgb(34, 34, 34);">：verifier 跟着 policy 的输出分布一起更新，保持相对分辨力（DynamicRubric）。</font>
    4. **自验证内化**<font style="color:rgb(34, 34, 34);">：把验证能力训进策略，消除 generator-verifier 分布错配（ReVeal、PAG、Self-Verify）。</font>
    5. **可解释信号**<font style="color:rgb(34, 34, 34);">：用"推理努力度""同构扰动"等外部可观测量当可信度代理（TRACE、IPT）。</font>

**用三条轴定位**<font style="color:rgb(34, 34, 34);">：一篇强 novelty 的工作，通常是"在某条轴上开了新档位"（如 1.4 符号 oracle、C3 共演化），而非在 1.2 里再调一版 GenRM。</font>



## <font style="color:rgb(34, 34, 34);">时间线</font>
### <font style="color:rgb(34, 34, 34);">【奠基期 2401–2504】生成式 verifier 起点</font>
##### <font style="color:rgb(0, 0, 0);">Self-Rewarding Language Models (2401.10020)</font>


##### **GenRM (2410.12832)  **<font style="color:#117CEE;">裁判要像人类一样写出推理过程（CoT）的范式</font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：</font>生成式奖励模型的起点，被后续 GRAM/DeepSeek-GRM 反复引用；
+ **核心问题**<font style="color:rgb(34, 34, 34);">：“判别式标量”RM 在 OOD 数据上鲁棒和泛化弱；RLAIF 可能和人类便偏好不完全一致。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787125647381-e51154d7-9b69-48c3-bb8d-e545821151a9.png" width="502" title="" crop="0,0,1,1" id="ue0d39f03" class="ne-image">

+ **三种 Reward Model**：
    - **Bradley-Terry RM（传统标量裁判）**：输入$ (x, y_1, y_2) $，通过后接的打分头输出实数分数 $ R $。
    - **GenRM (无CoT的生成式裁判）**：输入$ (x, y_1, y_2) $，不写理由，直接看模型下一个 Token 生成 $ I_1 $（选A）还是 $ I_2 $（选B）的概率大小。
    - **CoT-GenRM（带CoT的生成式裁判）**：输入$ (x, y_1, y_2) $，生成推理文字$ r $和指示标记 $ I $。
+ **STaR-DPO 算法**
    1. **自生成推理链（CoT Generation）**：

输入 Prompt$ x $和两个候选答案$ y_1, y_2 $，LLM 写出理由：“为什么答案$ y_1 $比$ y_2 $好？”

    2. **Bootstrapping（自我提炼/筛选）**：

采用类似 STaR 的思想：如果模型推理得出的结论和数据集里真实的“人类偏好胜者”一致，就把这条推理链（CoT）保留下来作为正样本；如果不一致，就扔掉。

    3. **结合 DPO 优化（STaR-DPO）**：

用保留下来的高质量“推理链 + 判决结论”，结合 DPO 微调模型。

+ **实验结果**
    1. **OOD 泛化能力**：GenRM 的准确率超传统标量 RM 10% ~ 45%
    2. **Inference-time Compute Scaling**：对同一比较采样多次，做多数投票，GenRM 的打分准确率随着推理算力的增加呈直线上升。



##### **<font style="color:#DF2A3F;">DeepSeek-GRM / SPCT (2504.02495) </font>****<font style="color:rgb(34, 34, 34);"> </font>**<font style="color:#117CEE;">小模型 GRM 通过测试期多花算力得到高质量奖励</font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：</font>**本批 HF 关注度第一**<font style="color:rgb(34, 34, 34);">，推理期缩放代表作。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：</font>通用领域的奖励建模比数学/代码难，没有确定性 ground truth，判别式 RM 在开放任务上打分不透明、易偏、OOD 掉点，易偏误。

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787131348314-d0328411-6cce-492b-9d70-1e8ef09b4d19.png" width="715" title="" crop="0,0,1,1" id="uc81fc052" class="ne-image">

+ **创新点：Self-Principled Critique Tuning (SPCT) **

<font style="color:#DF2A3F;">一个好的评判要先有</font>**<font style="color:#DF2A3F;">"评分原则(principle)"</font>**<font style="color:#DF2A3F;">，再据原则写</font>**<font style="color:#DF2A3F;">"批评(critique)"</font>**<font style="color:#DF2A3F;">、最后</font>**<font style="color:#DF2A3F;">给分</font>**<font style="color:#DF2A3F;">。</font>

    1. **Self-Principle Generation**

给定输入 Prompt$ x
 $与候选回复$ y
 $，模型首先分析该具体情境下的潜在风险与评价要点，自发出具一组针对该样本的**定制化原则（Principle）**。

    2. **Principled Critique Generation**

模型将候选回复$ y $与上述自建原则逐条比对，生成具体、针对性强的批**评意见（Critique）**，明确指出回复在哪一条原则上达标、在哪一条上犯了错，并给分。

    3. **Filtering & Self-Verification**：把"评判质量"变成可随采样次数提升的量
        1. **并行采样**：同一输入多次采样原则+批评，聚合；
        2. **meta-RM 投票**：训练一个 meta 奖励模型对多份批评投票筛选，过滤低质判断。
    4. **对齐微调（Tuning Phase）**

利用筛选出的高质量数据对模型进行 SFT 或偏好对齐微调（如 DPO/PPO）。

+ **实验结果**
    - DeepSeek-GRM 在多个 RM benchmark 上优于既有方法且未见明显领域偏差；
    - **关键结论**：推理期缩放（小 GRM + 多采样 + meta 投票）的收益可超过训练期把模型做大，在固定预算下，把算力花在测试期评判比花在堆参数更划算。



##### **<font style="color:rgb(34, 34, 34);">RewardAgent (2502.19328) </font>**<font style="color:#117CEE;">agentic </font>**<font style="color:#117CEE;">用工具（跑代码/查搜索）拿实证打分</font>**
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：ACL 2025；"agentic reward modeling"最早形态。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：RM 忽视可验证信号。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787133221260-f626c3f4-7b20-47b9-b7d5-3ce648060b37.png" width="665" title="" crop="0,0,1,1" id="uf16e6809" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：把奖励系统做成 agent</font>
    1. **<font style="color:rgb(0, 0, 0);">Router</font>**<font style="color:rgb(34, 34, 34);">：主动规划验证步骤。</font>
    2. **Verify Agents**<font style="color:rgb(34, 34, 34);">：与外界（代码解释器、搜索引擎、数据库）交互，拿到了</font>**真实客观的执行日志**。
    3. **Judger**<font style="color:rgb(34, 34, 34);">：将收集到的真实日志与原回答进行比对，生成一份有凭有据的 Critique，并算出最终奖励分。</font>

<font style="color:rgb(34, 34, 34);">基于上述轨迹，构建包含</font>`<font style="color:rgb(34, 34, 34);">[Plan -> Tool Call -> Observation -> Critique]</font>`<font style="color:rgb(34, 34, 34);">的多轮交互数据，并通过 SFT 与偏好微调（DPO/RL）训练 RM 模型。</font>

+ **实验结果**<font style="color:rgb(34, 34, 34);">：</font>
    - **代码与数学领域碾压**：在 Code / Math / Fact-checking 等需要客观验证的 Benchmark 上，ARM 的评估准确率远超仅靠 CoT 的静态 GenRM。
    - **根除“裁判幻觉”**<font style="color:rgb(34, 34, 34);">：有真实环境反馈（如代码输出或搜索实证）做锚定，传统 RM 经常出现的“凭空捏造错误”或“不懂装懂给高分”现象被大幅遏制。</font>



### <font style="color:rgb(34, 34, 34);">【爆发期 2505–2506】self-verification 集中涌现</font>
##### **<font style="color:#DF2A3F;">ReVeal (2506.11442) </font>**<font style="color:#117CEE;">细粒度过程验证（Process Verification）与递归纠错</font>
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：把"生成-验证-工具评估"结构化为多轮 RL 轨迹；</font>**TAPO 做逐轮 credit assignment**<font style="color:rgb(34, 34, 34);">。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：</font>
    - **Outcome-only Trap**：只看最终结果对错，无法精确定位**哪一步发生了逻辑滑坡**。
    - **All-or-Nothing Refinement**<font style="color:rgb(34, 34, 34);">：一旦判定错误，策略模型需要从头生成，无法基于错误节点做局部修复。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787142288187-9929e56e-6086-45d6-b78b-a369b7522a71.png" width="654" title="" crop="0,0,1,1" id="u9487f4f8" class="ne-image">

+ **创新点**<font style="color:rgb(34, 34, 34);">：</font>
    - **<font style="color:rgb(0, 0, 0);">ReVeal</font>**<font style="color:rgb(0, 0, 0);">：将长程推理拆解为“多轮 Iterative Generation-Verification 回路”</font>
        1. **Autonomous Test Case Generation：**<font style="color:rgb(0, 0, 0);">模型生成候选代码和写针对该题目的单元测试用例。</font>
        2. **Tool Execution & Environment Observation：**<font style="color:rgb(0, 0, 0);">将生成的代码与测试用例送入外部工具（如 Python 沙盒）进行编译运行，获取真实的执行日志、报错堆栈或 Pass/Fail 运行结果。</font>
        3. **Turn-Level Dense Rewards & TAPO：**<font style="color:rgb(0, 0, 0);">引入针对“代码质量”与“验证准确性”的密集奖励信号，并通过 TAPO 算法进行逐轮信用分配，防止模型生成垃圾代码来套取验证奖励。</font>
        4. **Co-Evolution of Generation & Verification：**<font style="color:rgb(0, 0, 0);">在多轮 RL 训练中，代码生成能力与自验证能力互相促进：</font>**更准确的测试用例逼出更高质量的代码，而更稳健的代码又提升了自验证的可靠性**<font style="color:rgb(0, 0, 0);">。</font>
    - **<font style="color:rgb(0, 0, 0);">TAPO：</font>****Turn-level Advantage Policy Optimization **- 基于 PPO
        * **<font style="color:rgb(0, 0, 0);">Token-level return</font>**<font style="color:rgb(0, 0, 0);">：pure Monte Carlo</font>
        * **<font style="color:rgb(0, 0, 0);">Turn-level return</font>**<font style="color:rgb(0, 0, 0);">：</font>
            + 代码质量奖励$ r_{\text{code}}^{(t)} $<font style="color:rgb(0, 0, 0);">：把 Agent 当前生成的代码送入沙箱，用隐藏测试集（Oracle）测出来的代码通过率。依据编译/语法错误或运行时崩溃、通过率打分。</font>
            + 验证准确性奖励$ r_{\text{verif}}^{(t)} $<font style="color:rgb(0, 0, 0);">：用 Agent 自身生成的单元测试用例去测试</font>**已知正确的标准代码（Oracle）和已知有 Bug 的变异代码**。依据误报、漏报、高区分度打分。
+ **实验结果**<font style="color:rgb(34, 34, 34);">：</font>在代码领域最权威的 **LiveCodeBench** 等基准上取得突破：
    - **Test-Time Scaling 突破上限**<font style="color:rgb(34, 34, 34);">：</font>
        * <font style="color:rgb(34, 34, 34);">训练阶段只用 </font>**3 turns **<font style="color:rgb(34, 34, 34);">轨迹，但测试推理阶段在 </font>**20+ Turns 的超长推理循环中持续演进修正代码**<font style="color:rgb(34, 34, 34);">。</font>
        * **Pass@1 **<font style="color:rgb(34, 34, 34);">从第 1 轮的 </font>**36.9%**<font style="color:rgb(34, 34, 34);"> 持续增长升至第 19 轮的 </font>**42.4%**<font style="color:rgb(34, 34, 34);">。</font>
    - **探索边界扩展（Pass@k 指标显著提升）**<font style="color:rgb(34, 34, 34);">：在 Pass@k 表现上大幅碾压 Qwen2.5-Coder 32B 等。</font>
    - **超越顶级开源 RL 模型**<font style="color:rgb(34, 34, 34);">：</font>超越了 **DeepSeek-R1-Zero-Qwen-32B** 和 CTRL 等单轮/多轮 RL 基线。



### <font style="color:rgb(34, 34, 34);">【转折期 2510】综述立项 + reward hacking 检测起势</font>
##### **<font style="color:rgb(34, 34, 34);">PRM 综述 (2510.08049) </font>**<font style="color:#117CEE;">方向被学术共同体正式承认的标志</font>
+ <font style="color:rgb(34, 34, 34);">过程数据 → PRM 构建 → 测试期缩放与 RL 应用，覆盖 math/code/text/多模态/机器人/agent。</font>

<font style="color:rgb(34, 34, 34);"></font>

##### **<font style="color:#DF2A3F;">TRACE (2510.01367) </font>****<font style="color:#117CEE;">Detecting Implicit Reward Hacking by Measuring Reasoning Effort</font>**
+ **<font style="color:#DF2A3F;">重要度依据</font>**<font style="color:#DF2A3F;">：ICLR 2026 Oral</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：</font>**Implicit Hacking **-** **模型的 CoT 看起来合理五无害，但实际上利用了漏洞得到答案。

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787229002868-16bcd4ea-2ce7-46d7-a52d-a38754c2c0de.png" width="723" title="" crop="0,0,1,1" id="u9925773a" class="ne-image">

+ **<font style="color:rgb(34, 34, 34);">创新点</font>**<font style="color:rgb(34, 34, 34);">：</font>
    - **<font style="color:rgb(34, 34, 34);">key insight</font>**<font style="color:rgb(34, 34, 34);">：reward hacking 需要的 reasoning effort 远低于真正推导难题。非 hack 模型通常最后拿到结果，而 hack 模型早期就能拿到奖励，之后的 CoT 是冗余。</font>
    - **TRACE（Truncated Reasoning AUC Evaluation）**<font style="color:rgb(34, 34, 34);">：</font>
    - <font style="color:rgb(34, 34, 34);">用</font><font style="color:#DF2A3F;">"</font>**<font style="color:#DF2A3F;">推理努力度(reasoning effort)</font>**<font style="color:#DF2A3F;">"</font><font style="color:rgb(34, 34, 34);">当可信度代理。靠推理得到正确答案的模型，</font>**<font style="color:rgb(34, 34, 34);">截断其 CoT 会显著掉正确率</font>**<font style="color:rgb(34, 34, 34);">（它依赖那些推理步）；靠捷径/记忆拿高分的模型，截断 CoT 正确率几乎不变（推理是"表演"）。</font>
        1. **梯度截断**：取 10%, 20%, ..., 100% 的 CoT 前缀
        2. **强行输出**：强制模型基于截断前缀直接补全答案
        3. **计算期望奖励**$ E[R*] $:在每个截断点多次采样并求平均得分
        4. **绘制“得分-长度”曲线**：计算曲线下面积 AUC (TRACE Score)

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787229284154-69fb1dd0-b7b3-4356-b41b-ff244c1e8da5.png" width="213" title="" crop="0,0,1,1" id="YQ6ZJ" class="ne-image">

+ **<font style="color:rgb(34, 34, 34);">实验结果</font>**<font style="color:rgb(34, 34, 34);">：数学上比 </font>**<font style="color:rgb(34, 34, 34);">72B</font>**<font style="color:rgb(34, 34, 34);"> CoT 监控器检测增益 </font>**<font style="color:rgb(34, 34, 34);">>65%</font>**<font style="color:rgb(34, 34, 34);">，代码上比 </font>**<font style="color:rgb(34, 34, 34);">32B</font>**<font style="color:rgb(34, 34, 34);"> 监控器 </font>**<font style="color:rgb(34, 34, 34);">>30%</font>**<font style="color:rgb(34, 34, 34);">。即轻量的"努力度探针"胜过重量级监控模型。</font>

<font style="color:rgb(34, 34, 34);"></font>

### <font style="color:rgb(34, 34, 34);">【爆发期 2511–2605】攻防 + agentic + 可验证过程奖励</font>
##### **<font style="color:#DF2A3F;">Reward Under Attack (2603.06621) </font>****<font style="color:#117CEE;">评测 RM 鲁棒性</font>**
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：ICML 2026；</font>**"verifier 可信度"关键的拆解性证据**<font style="color:rgb(34, 34, 34);">。</font>
+ **核心问题**<font style="color:rgb(34, 34, 34);">：</font><font style="color:rgb(0, 0, 0);">如何衡量 RM 的鲁棒性。</font>

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787546772304-235315a2-0f2b-474b-8f38-632d32e64317.png" width="1452" title="" crop="0,0,1,1" id="ufa0d9947" class="ne-image">

+ **创新点：****<font style="color:rgb(34, 34, 34);">静态扰动分析框架（Static Perturbation Analysis）</font>**<font style="color:rgb(34, 34, 34);">，施加越来越大的干扰，量化 RM 的安全边界</font>
    - **<font style="color:rgb(34, 34, 34);">语义保持（Semantics-preserving edits）</font>**<font style="color:rgb(34, 34, 34);">：保持推理正确性</font>
        * **<font style="color:rgb(34, 34, 34);">改写（rephrasing）</font>**<font style="color:rgb(34, 34, 34);">：改变用词和句法</font>
        * **<font style="color:rgb(34, 34, 34);">冗余度变化（verbosity changes）</font>**<font style="color:rgb(34, 34, 34);">：增加或删除冗余语言</font>
    - **<font style="color:rgb(34, 34, 34);">语义破坏</font>**<font style="color:rgb(34, 34, 34);">（</font>**<font style="color:rgb(34, 34, 34);">Semantics-altering attacks）</font>**<font style="color:rgb(34, 34, 34);">：引入逻辑错误</font>
        * **<font style="color:rgb(34, 34, 34);">问题重排（question shuffling）</font>**<font style="color:rgb(34, 34, 34);">：将推理轨迹与不相关的提示词配对</font>
        * **<font style="color:rgb(34, 34, 34);">推理幻觉（reasoning hallucination）</font>**<font style="color:rgb(34, 34, 34);">：在推理步骤中注入错误的假设</font>
+ **实验**<font style="color:rgb(34, 34, 34);">：</font>
    - **<font style="color:rgb(34, 34, 34);">Metrics</font>**
        * **偏好反转率**<font style="color:rgb(34, 34, 34);">：施加扰动后，原始正负样本对在 RM 判定中相对偏好关系发生倒置的比例。</font>
        * **分数漂移量**<font style="color:rgb(34, 34, 34);">：针对同一回答输入，在扰动前后 RM 标量得分/预测概率的偏移绝对值。</font>
    - **实验结果：**
        * **对语义保持敏感**：主流 RM 过度依赖 Style Bias 作为打分依据，出现得分偏移。
        * **对语义改变型扰动鉴伪不足**：部分 RM 给扰动后的错误回答打出高分。
        * <font style="color:rgb(34, 34, 34);">具 CoT 的 GenRM 对对表层格式的鲁棒性优于标量 RM；但当扰动注入，GenRM 易被带偏。</font>

<font style="color:rgb(34, 34, 34);"></font>

##### **<font style="color:rgb(34, 34, 34);">reward hacking (2604.13602) </font>****<font style="color:#117CEE;">综述</font>**
+ **重要度依据**<font style="color:rgb(34, 34, 34);">：提出 Proxy Compres</font>sion Hypothesis：hacking 源于目标压缩、优化放大与** evaluator-policy 共适应**的相互作用。后者正是 DynamicRubric/对抗训练要治的根因。
+ 双向调适的两个方向
    1. **Evaluator**$ \rightarrow $**Policy：策略模型根据评估器的反馈信号更新权重。**策略模型会逐渐吸收评估器的偏好特征（如推理格式、特定句式），同时也会主动去探索并利用评估器的鉴伪盲区（Reward Hacking）。
    2. **Policy**$ \rightarrow $**Evaluator：**随着策略模型能力的提升，其输出分布会发生漂移（Distribution Shift），产生静态评估器未曾见过的复杂回答或新型逻辑错误。为了保持评估精准度，Evaluator 必须动态升级（如引入策略模型最新 Hard Negatives、更新评分规则、补充工具调用能力），**重新标定评估边界。**

<font style="color:rgb(34, 34, 34);"></font>

**<font style="color:#DF2A3F;">AgentV-RL (2604.16004) </font>****<font style="color:#117CEE;">双向逻辑核验</font>**

    - **重要度依据**<font style="color:rgb(34, 34, 34);">：ACL 2026</font>
    - **核心问题**：
        * Error Propagation & False Positive：RM 在大体正确的数据上训练，当面对包含推导错误的假答案时，极易被表面看似合理的推导误导。
        * Lack of External Grounding：面对高强度计算或知识密集型任务，Verifier 缺乏与代码解释器等外部符号工具的结合，容易产生幻觉，导致验证结果不稳定。

<img src="https://intranetproxy.alipay.com/skylark/lark/0/2026/png/237956535/1787551639606-3d3af9b5-9a30-460c-82e7-49c3cd51b847.png" width="663" title="" crop="0,0,1,1" id="E7VIu" class="ne-image">

    - **创新点**<font style="color:rgb(34, 34, 34);">：</font>
        * **<font style="color:rgb(34, 34, 34);">双向协作智能体架构</font>**
            + Forward Agent - 充分性检查：从已知条件推导至结论，一步步验证逻辑链條的充分性。
            + Backward Agent - 必要性检查：从最终结论倒推已知条件，防遗漏必要条件或逻辑倒装。
        * **<font style="color:rgb(34, 34, 34);"> Agentic Verifier：</font>**<font style="color:rgb(34, 34, 34);">Plan-Validate-Verdict</font>
            1. <font style="color:rgb(34, 34, 34);">Planning：智能体将求解过程拆解为原子化、可验证的子步骤序列 </font>$ \Pi=\{v_1, v_2, ..., v_n\} $<font style="color:rgb(34, 34, 34);"></font>
            2. <font style="color:rgb(34, 34, 34);">Validation：ReAct 多轮互动，可自主调用 Python 解释器，将运行结果并入推理链中核对。</font>
            3. <font style="color:rgb(34, 34, 34);">Verdict： 汇总证据，输出点评和裁决，并将 True Token 的 Logit 概率作为置信度得分。</font>
        * **AgentV-RL**：将上述多智能体协作能力整合进单 LLM 中的训练方案
            1. **合成轨迹采样：**过滤高质量推理数据集，由大模型扮演正向/反向 Agent 自动生成带 Python 工具调用的验证轨迹，并通过二元正确性控制筛选高质量数据。
            2. **拒绝采样微调：**在 15K 合成数据上进行 SFT，使模型掌握多轮拆解、思考及工具调用的交互。
            3. **强化学习：** 使用 **GRPO **在 50K 样本上利用可验证的二元结果奖励进行强化学习。
        * **Test Time Scaling**
            + **Parallel Scaling / Best-of-N：**Verifier 对 N 个候选解答打分，选择综合置信度最高的解答。
            + **Sequential Scaling / Iterative Refinement：**Verifier 输出自然语言 Critique（修改意见），指导 Actor 模型逐步修正错误。
    - ** 实验结果**
        * **数据集：**GSM8K、MATH500、Gaokao2023、AIME24 等多个数学与逻辑推理基准
        * **并行采样（Best-of-N）**表现大幅超越同等及更大规模 RM
        * **串行修正（Sequential Refinement）**表现精准纠错能力，快速收敛且稳定性

<font style="color:rgb(34, 34, 34);"></font>

### <font style="color:rgb(34, 34, 34);">【最新 2607】评测器-策略共演化</font>
+ **<font style="color:rgb(34, 34, 34);">DynamicRubric (2607.20083) </font>**<font style="color:#117CEE;">共演化新工作</font>
    - 见[详读](https://aliyuque.antfin.com/idsn7l/yihwt7/rueesx9gk0yw2s68)。












|  | **权重级自进化** | **产物级进化** |
| --- | --- | --- |
| **进化对象** | 模型自身权重/参数$ \theta $ | 外部产物（代码、训练脚本、算法、方案等） |
| **评判指标** | 同等参数与算力下对比 Base vs Evolved；需报**逐迭代曲线** | 产物质量 vs Human SOTA；“性能-预算” Scaling 曲线 |
| benchmark | 任务量大、可复现、可自动核验、单轨迹成本低； | 优化过程开放、带可运行评估脚本、具备人类基线 |


本文档关注**前者，权重级自进化**：利用模型自身生成的任务、轨迹，多轮迭代改写权重。

# 1 价值判断
### **<font style="color:rgba(0, 0, 0, 0.88);">1.1 自进化闭环四要素</font>**
```plain
① 任务生成 (Task Proposer) →  ② 轨迹采集 (Policy Rollout)
        ↑                                ↓
 ④ 参数更新 (Updater)    ←    ③ 打分 (Evaluator)
```

+ 大部分的自进化工作基于上图展开，各自在 ①、②、③ 部分做出设计。
+ 然而，随着 policy model 能力增强：
    1. 开始进入 evaluator 没见过、也没能力准确判断的区域。
    2. policy model 不再真正优化任务目标，而开始优化 evaluator 的漏洞。
    3. evaluator 需要和 **<font style="color:rgba(0, 0, 0, 0.88);">policy model 共同进化</font>**<font style="color:rgba(0, 0, 0, 0.88);">。</font>
+ <font style="color:rgba(0, 0, 0, 0.88);">近期，自进化的闭环进一步被证明会</font>**<font style="color:rgba(0, 0, 0, 0.88);">坍</font>********<font style="color:#000000;">塌</font>**<font style="color:#000000;">。</font>
    - <font style="color:rgba(0, 0, 0, 0.88);">具体表现：随着进化轮数增加，policy model 的能力越进化越差。</font>
    - <font style="color:rgba(0, 0, 0, 0.88);">Verification Horizon 2606.26300(</font><font style="color:#117cee;">Qwen, 53upvote</font><font style="color:rgba(0, 0, 0, 0.88);">）：验证器和生成器要 co-evolve.</font>
+ <font style="color:rgba(0, 0, 0, 0.88);">行业与学界</font><font style="color:#df2a3f;">关注度</font><font style="color:rgba(0, 0, 0, 0.88);">：</font>
    - <font style="color:rgba(0, 0, 0, 0.88);">学界：</font><font style="color:#000000;">AbsoluteZero（起点工作</font><font style="color:#117cee;"> NIPS2025 </font><font style="color:#df2a3f;background-color:rgb(247, 246, 244);">spotlight</font><font style="color:#117cee;">，1.9k star</font><font style="color:#000000;">）；</font><font style="color:rgb(44, 58, 74);background-color:rgb(255, 253, 250);">R-Zero (</font><font style="color:#117cee;background-color:rgb(255, 253, 250);">Tencent AI Lab, ICLR2026, 179cite</font><font style="color:rgb(44, 58, 74);background-color:rgb(255, 253, 250);">)；</font><font style="color:#000000;">DR Tulu (</font><font style="color:#117cee;">ICML2026 </font><font style="color:#df2a3f;background-color:rgb(247, 246, 244);">oral</font><font style="color:#117cee;">，huggingface 64 upvote</font><font style="color:#000000;">）</font>
    - <font style="color:rgba(0, 0, 0, 0.88);">工业界：相关工作已有行业落</font><font style="color:#000000;">地：DynamicRubric（</font><font style="color:#117cee;">腾讯微信 </font><font style="color:#000000;">260720）</font>

**分析判断 - 关键瓶颈：****<font style="color:#df2a3f;">Evaluator</font>**

系统能否实现持续且可靠的自我提升，核心取决于**评估信号的质量**。

若 Evaluator 存在偏见、Reward Hacking 或能力上限，Policy 的训练和推理扩展就会停滞或产生退化。因此，Evaluator（包含 verifier, a reward model, execution feedback, a proof checker, a meta-evaluator）是整个闭环中最关键的研发瓶颈。

过去的问题是“如何让 Policy 自我进化”；现在的问题进一步变成“当 Policy 越来越强时，谁来判断它是否真的变强？”；Evaluator 已经不应再被视为自进化系统中的一个静态模块，而被视为**决定整个自进化闭环能否持续、稳定、可靠运行的核心研究对象**。



### 1.2 Evaluator
核心瓶颈：

1. **Reward hack**：
    - 现象：模型生成能力已超过验证能力，验证比生成难。
2. **错误放大**：
    - 现象：Hack 产物会回流进下一轮的训练池，误差被指数级放大。
    - Misalignment 2511.18397 (<font style="color:#117cee;">Anthropic, 85cite</font>): 模型学会 hack，错误泛化到各场景；
    - <font style="color:rgba(0, 0, 0, 0.88);">Monitoring 2503.11926 (</font><font style="color:#117cee;">OpenAI, 405cite</font><font style="color:#000000;">):</font><font style="color:rgba(0, 0, 0, 0.88);"> 检测 CoT 为奖励，模型学会混淆式 hack</font>

如何让进化更稳：

1. **一个好的 Evaluator**：判断准确、判断有区分度；
2. **进入下一轮的数据**：难度刚好卡在 Policy 能力边界。



---

# <font style="color:rgb(34, 34, 34);">2 </font>grounding 来源
| <font style="color:rgb(34, 34, 34);">方法</font> | **锚信号** | **锚从哪来（具体来源）** | **怎么注入 evaluator 训练** |
| --- | --- | --- | --- |
| **DynamicRubric**<font style="color:rgb(34, 34, 34);"> (2607.20083 tencent)</font> | ranked anchor data $ \mathcal{A} $<font style="color:rgb(34, 34, 34);">：带人工排序的响应集</font> | 人类标注偏好数据<font style="color:rgb(34, 34, 34);">（</font><font style="color:rgb(0, 0, 0);">Nectar Zhu et al.</font><font style="color:rgb(34, 34, 34);">）</font> | <font style="color:rgb(34, 34, 34);">训练目标 = </font>**discriminability 目标 + anchor 目标**<font style="color:rgb(34, 34, 34);">（权重 </font>$ \lambda $<font style="color:rgb(34, 34, 34);">）。</font><br/><font style="color:rgb(34, 34, 34);">anchor 项强制 rubric 分数排序与人工排序一致，防止 generator 为拉大 gap 而编造标准。另有</font>**冻结的 DR-Verifier**<font style="color:rgb(34, 34, 34);">（不训练）执行 yes/no 判定，切断"生成者兼裁判"的闭环</font> |
| **EvoRubrics**<font style="color:rgb(34, 34, 34);"> (2606.23038 PKU)</font> | _golden rubric _$ R^* $<font style="color:rgb(34, 34, 34);">：预先构造的参考评分标准</font> | <font style="color:rgb(34, 34, 34);">人工/专家编写的 golden rubric（HealthBench 等域内标准）</font> | <font style="color:rgb(34, 34, 34);">两条通道：</font>**alignment reward**<font style="color:rgb(34, 34, 34);"> </font>$ r^{align}_j=\text{Sim}(R_j,R^*)/10 $<font style="color:rgb(34, 34, 34);"> 约束生成 rubric 不漂移；</font>**constructiveness reward**<font style="color:rgb(34, 34, 34);"> 用 </font>$ R^* $<font style="color:rgb(34, 34, 34);"> 度量"按新 rubric 修订后答案是否真变好"。去掉 </font>$ R^* $<font style="color:rgb(34, 34, 34);"> 的自监督变体仍有效但失去外部性</font> |
| **DR Tulu**<font style="color:rgb(34, 34, 34);"> (2511.19399 ICML26)</font> | 检索到的外部文档证据 | <font style="color:rgb(34, 34, 34);">训练中</font>实时检索<font style="color:rgb(34, 34, 34);">（search 工具返回的真实资料）</font> | <font style="color:rgb(34, 34, 34);">rubric 构造时纳入检索到的新信息与对比响应，使评分标准绑定可核查事实而非模型内部知识</font> |
| **DR-Rubric**<font style="color:rgb(34, 34, 34);"> (2606.01091)</font> | 多轮 agentic 检索得到的领域事实<font style="color:rgb(34, 34, 34);"></font> | <font style="color:rgb(34, 34, 34);">外部搜索 API + 权威来源</font> | <font style="color:rgb(34, 34, 34);">Stage I 检索证据→Stage II 蒸馏为</font>**原子可验证约束**<font style="color:rgb(34, 34, 34);">；另用 </font>**reward polarization**<font style="color:rgb(34, 34, 34);"> 作坍缩预警（越阈值即停），是对 co-evolution 的</font>**过程性外部约束** |
| **OnlineRubrics**<font style="color:rgb(34, 34, 34);"> (2510.07284)</font> | reference policy 的响应<font style="color:rgb(34, 34, 34);">作为对照基准</font> | <font style="color:rgb(34, 34, 34);">训练前固定的 reference model（非当前 policy）</font> | <font style="color:rgb(34, 34, 34);">用"当前 policy vs. reference policy"成对比较在线抽取标准；锚在</font>**未被当前优化污染的旧分布**<font style="color:rgb(34, 34, 34);">上</font> |
| **AMARIS**<font style="color:rgb(34, 34, 34);"> (2605.18592)</font> | 纵向训练证据<font style="color:rgb(34, 34, 34);">：历史 rollout 分析、历次 rubric 编辑及其效果</font> | <font style="color:rgb(34, 34, 34);">持久化评估记忆库（系统自身历史，非外部真值）</font> | <font style="color:rgb(34, 34, 34);">检索历史记录来修订 rubric，可</font>**评估以往编辑是否有效**<font style="color:rgb(34, 34, 34);">、避免震荡。</font><br/>⚠<font style="color:rgb(34, 34, 34);"> 锚是"时间维度的自我一致性"，</font>**不是外部真值** |
| **PasoDoble**<font style="color:rgb(34, 34, 34);"> (2511.11881)</font> | ground-truth 答案 + 通过率区间 | <font style="color:rgb(34, 34, 34);">Proposer 生成问题时必须附带可验证答案；预训练语料提供知识</font> | <font style="color:rgb(34, 34, 34);">只有当问题有效、Solver 通过率落在指定区间才计分——把可验证性写进</font>**博弈规则**<font style="color:rgb(34, 34, 34);">本身</font> |
| **Self-Guide**<font style="color:rgb(34, 34, 34);"> (2604.03098)</font> | 环境 reward | <font style="color:rgb(34, 34, 34);">agent 交互的真实环境反馈</font> | ⚠<font style="color:rgb(34, 34, 34);"> 内部 self-guidance 负责稠密化信号，但</font>**最终仍需环境 reward 兜底**<font style="color:rgb(34, 34, 34);">；内部 reward 本身无外部性，是 URPO 警示的风险区</font> |


### <font style="color:rgb(34, 34, 34);">锚点的三种形态（按"policy 能否伪造"排序）</font>
1. **不可伪造的硬锚**<font style="color:rgb(34, 34, 34);">：ground-truth 答案、执行结果、检索到的可核查事实（PasoDoble、DR Tulu、DR-Rubric）——policy 无法通过"说得像"来满足。</font>
2. **人工标注的排序锚**<font style="color:rgb(34, 34, 34);">：ranked anchor data、golden rubric（DynamicRubric、EvoRubrics）——</font>**当前主流做法**<font style="color:rgb(34, 34, 34);">。人类偏好本身是稀缺的，故只用少量数据作为"校准锚"，而非全量监督；关键设计是把它作为</font>**约束项**<font style="color:rgb(34, 34, 34);">而非主目标，与 discriminability 目标共同优化。</font>
3. **结构性弱锚**<font style="color:rgb(34, 34, 34);">：reference policy 旧分布、历史记忆自一致性（OnlineRubrics、AMARIS）——不引入新外部信息，只保证"不偏离得太快"，防漂移但不防系统性共谋。</font>

### <font style="color:rgb(34, 34, 34);">关键机制观察</font>
+ **锚不是用来直接打分的，是用来约束 evaluator 更新方向的。**<font style="color:rgb(34, 34, 34);"> DynamicRubric 的 anchor 项、EvoRubrics 的 alignment reward 都是</font>**正则项**<font style="color:rgb(34, 34, 34);">，主目标仍是判别力。这正是"既让 E 跟上 π，又不让 E 学到 π 的 shortcut"的工程解法。</font>
+ **切断"生成者兼裁判"**<font style="color:rgb(34, 34, 34);">：DynamicRubric 让 </font>**DR-Verifier 保持冻结**<font style="color:rgb(34, 34, 34);">（论文明确列为局限："We also keep DR-Verifier frozen rather than training a dedicated verifier"），只训练 rubric 生成器。验证环节不参与共演化，天然阻断闭环共谋。经验证该设计对验证器选择鲁棒（换 Llama-3.1-8B 等不同族仍成立）。</font>
+ **锚的稀缺性决定路线选择**<font style="color:rgb(34, 34, 34);">：人工排序锚昂贵→只能用作校准；检索证据可规模化→成为开放域主流；执行结果最可靠→但只覆盖可验证域。</font>

## 
---

# 3 分布偏移
+ **进化偏移**：模型一旦更新，模型自己产生的经验分布就变了；而下一轮训练信号又来自这个已经改变的经验分布。于是错误、偏好和偶然模式可以通过“模型 → 数据 → 训练 → 模型”的闭环逐轮放大，最终把模型推离原来的行为分布，甚至造成分布坍缩。
+ **OPD**：学生已经产生了自己的 prefix 分布，但教师被要求在这些 prefix 上提供监督；随着学生与教师的行为/思维路径差异累积，学生访问的状态逐渐落到教师不熟悉的区域，于是教师监督本身开始变得不可靠。

这两个概念**表面上都叫 distribution shift，但方向不同**：

$ \boxed{
\text{自进化：distribution shifts across evolution rounds}
} $

$ \boxed{
\text{OPD：distribution mismatch between student occupancy and teacher reliability}
} $

最近的 multi-turn OPD 工作实际上明确把这两者联系起来了：它将问题描述为 **student occupancy 与 teacher reliability 之间的 two-sided distribution shift**。([Microsoft](https://www.microsoft.com/en-us/research/publication/multi-turn-on-policy-distillation-with-prefix-replay/?utm_source=chatgpt.com))

## RSI
假设第 0 轮模型是$ \pi_0 $，它 rollout 出来的轨迹分布是$ \tau \sim p_{\pi_0}(\tau) $,从这些历史轨迹提取训练信号，把模型更新成$ \pi_1 $。于是下一轮模型 rollout 的轨迹变成$ \tau \sim p_{\pi_1}(\tau) $...

$ \pi_0 \rightarrow \pi_1 \rightarrow \pi_2 \rightarrow \cdots $

$ p_{\pi_0}(\tau)
\neq
p_{\pi_1}(\tau)
\neq
p_{\pi_2}(\tau). $

这就是**自进化过程中的分布漂移（distribution shift / distribution drift）**。

关键：**训练数据的来源本身就是模型，而模型一变，之后产生的数据分布也跟着变。**

$ \boxed{
\text{模型}
\rightarrow
\text{rollout分布}
\rightarrow
\text{训练信号}
\rightarrow
\text{模型更新}
\rightarrow
\text{新的rollout分布}
} $

因此“多轮后崩塌”经常可以从这个角度理解：

$ p_{\pi_0}(\tau)
\rightarrow
p_{\pi_1}(\tau)
\rightarrow
p_{\pi_2}(\tau)
\rightarrow \cdots $

每一轮都可能<font style="color:#DF2A3F;">把模型推向一个</font>**<font style="color:#DF2A3F;">越来越窄、越来越偏的行为分布</font>**。

:::info
最开始不同解法很多，几轮自训练之后，模型可能偏向某种**“看起来容易得到高 reward”**的轨迹，变成：

+ 某种正确模板：70%
+ 其他正确解法：10%
+ 错误轨迹：20%

再下一轮可能变成：

+ 某一种模板：95%
+ 其他行为：5%

:::

这时我们说模型的<font style="color:#117CEE;"> </font>**<font style="color:#117CEE;">occupancy / trajectory distribution 发生了坍缩或偏移</font>**。

所以在“自进化多轮崩塌”语境下，分布偏移最值得关注的其实是：

$ \boxed{
\text{模型自己的行为分布随着进化不断改变}
} $

---

## OPD
OPD（On-Policy Distillation）核心：**学生生成轨迹，教师在学生的状态上提供 token-level supervision。**

即：$ s_t \sim \pi_{\text{student}} $，教师给出$ \pi_{\text{teacher}}(a_t\mid s_t). $

所以 OPD 特意解决了传统 offline distillation 的一个问题：

传统 SFT：**训练时学生看到的是教师产生的 prefix，测试时学生自己产生 prefix。**长序列里，这个差异会不断累积，这就是典型的 train-test distribution shift / exposure bias。([arXiv](https://arxiv.org/abs/2605.14071?utm_source=chatgpt.com))

### OPD 为什么叫“on-policy”？
OPD 把“前缀/状态的采样权”给了学生$ s_t \sim d_{\pi_S} $。然后$ \text{teacher}(a_t|s_t) $

它试图做到：

$ \boxed{
\text{训练时的 state/prefix distribution}
\approx
\text{学生实际 inference 时的 distribution}
} $

这也是 OPD 相比 teacher-generated SFT 的**核心优势**。

### 问题来了：这并不意味着“教师和学生分布完全一致”
一道多步题。教师可能走：$ s_0 \rightarrow s_1^T \rightarrow s_2^T \rightarrow s_3^T $；学生可能走：$ s_0 \rightarrow s_1^S \rightarrow s_2^S \rightarrow s_3^S. $

1. 在第一步$ s_1^S \approx s_1^T $可能没问题。
2. 到了第三、第四步，学生只要前面有一点不同$ s_3^S \neq s_3^T $；教师被要求回答**自己不太会遇到的 prefix/state**。

$ s_t \sim d_{\pi_S} $，但教师自然熟悉的是$ s_t \sim d_{\pi_T}. $，于是：$ \boxed{
d_{\pi_S}\neq d_{\pi_T}
} $

这就是 OPD 现在很多工作所说的<font style="color:#DF2A3F;"> </font>**<font style="color:#DF2A3F;">student-teacher distribution gap / mismatch</font>**。

近期对 OPD 的系统研究明确指出，student-teacher distributional gap 会导致教师在学生访问到的状态上提供<font style="color:#DF2A3F;">不可靠或错位的指导</font>；另一些多轮 OPD 工作进一步把它描述成一种<font style="color:#DF2A3F;">随 horizon 增长而加剧的 prefix-level shift</font>。

### OPD 中“两种分布”
#### A. 状态 / prefix 分布
$ d_{\pi_S}(s)
\quad\text{vs.}\quad
d_{\pi_T}(s) $

> **学生实际走到了哪些状态，教师平时又会走到哪些状态。**
>

这是 multi-turn OPD 里非常关键的 distribution shift。

#### B. 同一个状态下的 action / token 分布
到了相同的$ s_t $，还可以比较：

$ \pi_S(a|s_t)
\quad\text{vs.}\quad
\pi_T(a|s_t). $

> **学生和教师在同一个状态下，对下一个 token / action 的概率分布是不是接近。**
>

OPD 本身就在利用教师的 token-level distribution 去推动学生：

$ \pi_S(\cdot|s_t)
\rightarrow
\pi_T(\cdot|s_t). $

近期研究发现，成功的 OPD 中起主要作用的是 student-visited states 上**双方高概率 token 的逐渐对齐**。([arXiv](https://arxiv.org/abs/2604.13016?utm_source=chatgpt.com))

---

### 为什么长 horizon 会让 OPD 的 distribution shift 越来越严重？
可以把它想成一个误差累积过程。

1. 第一步：$ s_1^S \approx s_1^T $
2. 第二步：$ s_2^S = f(s_1^S,a_1^S) $。只要$ s_1^S\neq s_1^T $，就可能$ s_2^S\neq s_2^T. $
3. 到了第三步$ s_3^S $是在一个已经不同的 prefix 上继续产生的。
4. 随着$ t $增大，$ D(d_{\pi_S}^{(t)},d_{\pi_T}^{(t)}) $往往会越来越大。

最近 multi-turn OPD 工作提出 **prefix trap**：越往后用完全 student-on-policy 的 prefix，确实越贴近学生自己的分布，但这些 prefix 对教师来说可能越来越 out-of-distribution，于是教师监督可靠性下降。([Microsoft](https://www.microsoft.com/en-us/research/publication/multi-turn-on-policy-distillation-with-prefix-replay/?utm_source=chatgpt.com))

---

## 总结：同一个“反馈闭环问题”
### 自进化：**模型自我更新导致**
$ \pi_k
\rightarrow
d_{\pi_k}
\rightarrow
\text{训练信号}
\rightarrow
\pi_{k+1} $

关键：$ d_{\pi_k}\rightarrow d_{\pi_{k+1}} $**。模型更新导致自己的数据/行为分布发生漂移。**

### OPD：学生与老师之间
$ d_{\pi_S}
\rightarrow
\text{teacher supervision}
\rightarrow
\pi_S' $

关键：$ d_{\pi_S} vs.
d_{\pi_T} $**。学生的行为分布与教师可靠监督分布之间出现错位。**

### 区别
| **场景** | **“分布偏移”的含义** | **比较对象** |
| --- | --- | --- |
| 自进化多轮 | 模型每轮更新后，自己的 rollout / state / trajectory distribution 改变 | $ d_{\pi_k} $vs$  d_{\pi_{k+1}} $ |
| SFT | 训练数据来自教师，但 inference 来自学生 | $ d_{\pi_T} $vs$ d_{\pi_S} $ |
| Multi-turn OPD | 学生虽然 on-policy，但学生访问的 prefix 越来越偏离教师熟悉的 prefix | $ d_{\pi_S}(s_{1:t}) $vs$ d_{\pi_T}(s_{1:t}) $ |
| OPD token alignment | 同一个 student state 下，学生和教师的 action/token distribution 不一致 | $ \pi_S(a|s_t) $vs.$ \pi_T(a|s_t) $ |


**.**
