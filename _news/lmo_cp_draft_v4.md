# Multi-Interval Minimum-Covering Ordinal Conformal Prediction (with Length Regularization) — v4
*(工作标题；base 方法 LMO-CPS，正则化变体 LMO-RCPS)*
 
> **Read this first（给学生）**：本项目**不是**要证明 LMO-CP 全面优于 min-CPS。它只瞄准**多峰** ordinal posterior。第一周目标不是写 paper，而是验证两件事——真实数据里是否有足够多峰、base-region 是否有 oracle gain。**两个 gate 不过就停**（见 §5.1）。
 
> 一句话：min-CPS 精确求解"包含模型最自信标签的最短区间"。LMO-CP 面向多峰预测，去掉"必须含最自信标签"和"只能一段区间"两条限制，找 anchor-free、最多 M 段的最短覆盖区域，用 split conformal 保证覆盖率；M=1 + anchor 退回 min-CPS，再加长度惩罚退回 min-RCPS。
 
这份文档每讲一个设计，都紧跟一段「为什么这么做」。这个项目里几乎每个技术选择都是在"和 min-CPS / min-RCPS 的继承关系"与"和已有工作的区分"之间权衡定下来的，把理由写清楚比把公式写漂亮更重要。
 
---
 
## 读这份文档前要知道的几个词
 
- **有序分类 (ordinal classification)**：标签之间有自然顺序。例：年龄段、病情严重度（轻/中/重）、评分（1–5 星）。"把重度误判成轻度"比"误判成中度"错得更离谱——顺序有意义。
- **共形预测 (conformal prediction, CP)**：不只输出一个标签，而是输出一个**标签集合**，并保证"真标签落在集合里"的概率 ≥ 1−α（如 90%）。这个保证不依赖模型准不准，是统计上严格的。
- **预测集的形状**：传统 ordinal CP 通常追求单个连续区间，因为它最易解释；但在多峰不确定性下，更自然的对象是**少数几个连续区间的并**。本文**不**允许任意零散集合，而是允许**最多 M 个连续区间**（component-constrained），在多峰表达力和 ordinal 可解释性之间折中。这也正是我们和 Chakraborty OPS（任意非连续集）的区别。
- **mode（众数 / 锚点）**：模型概率最高的那个标签。
- **多峰 (multimodal)**：模型概率分布有多个高峰。例：一张模糊 X 光，模型觉得"轻度""重度"都很可能，中间"中度"概率低。
- **覆盖率 (coverage)**：真标签落在预测集里的概率。CP 的核心任务就是保证它 ≥ 1−α。
- **正则化 (regularization)**：目标里加一个惩罚项，在"覆盖"和"集合简洁"之间权衡。min-CPS 的配套方法 min-RCPS 就是加了长度惩罚。
---
 
## 1. Intro（要解决什么，为什么这么解决）
 
### 1.1 起点：min-CPS / min-RCPS
 
min-CPS（AAAI 2026）是有序分类共形预测的强基线。它在每个样本上找一段**最短连续区间**，要求 (1) 覆盖的概率足够大，(2) **必须包含 mode**。它有精确线性时间算法、覆盖保证，平均比之前方法缩小约 14–15% 集合。原论文同时给了 length-regularized 版本 **min-RCPS**：在覆盖约束里减一个长度惩罚，进一步压缩集合，保持同样的 split-conformal 校准。
 
> **为什么从它出发、且连 min-RCPS 一起看**：它已把"有序 + 最短区间 + 精确求解"走通，是公认强基线，且是 min-CPS + min-RCPS 成对提出的。我们做 follow-up，只推广 min-CPS、不处理 regularization 会显得没继承完整。所以一开始就把"能干净退回 min-CPS **和** min-RCPS"当设计约束。
 
### 1.2 min-CPS 的两条假设，在多峰下会吃亏
 
1. **必须包含 mode（anchor）**：若真正合理的区域不在 mode 附近，从 mode 出发会被迫拉长。
2. **只能一段区间**：若概率集中在两个相隔很远的峰上（中间是低概率"山谷"），单段区间会把整段山谷也包进去——而山谷里的标签都不太可能是真值。
具体场景：模糊医学图像，模型在"轻度""重度"两峰都有概率，"中度"概率低。min-CPS 只能输出 [轻度,…,重度] 一大段，把"中度"也算进去。更合理的输出是 {轻度附近} 和 {重度附近} 两小段，跳过中间。
 
### 1.3 我们的方法 LMO-CP
 
- **去掉 anchor**：不从 mode 出发，而是在最多 M 段区间中求满足概率质量约束的**最短覆盖区域**。
- **允许多区间**：最多 M 段连续区间，自然处理多峰。
- **保留共形保证**：用 split conformal 校准，覆盖率严格 ≥ 1−α。
- **能退回 min-CPS / min-RCPS**：M=1 + anchor 退回 min-CPS；再加长度惩罚退回 min-RCPS。
> **为什么把"能退回"当硬性目标**：我们想把这篇定位成 min-CPS / min-RCPS 的**真正推广**（superset），不是另起炉灶的竞争方法。能精确退回，等于声明"在你原来 work 的地方我和你一样，我只在多峰情形多做一点"。这个目标直接决定了选哪种数学形式（§3.1）和怎么加 regularization（§3.7）。
>
> 注意措辞：我们说的是"最短覆盖区域（满足 mass 约束）"，**不是**"密度最高的区域"——后者会把方法拉回 highest-density region / density conformal 的既有文献，而我们的主对象是 mass-covering，不是 fixed density threshold。这个区别要从一开始守住。
 
---
 
## 2. 和已有工作的关系（必须先讲清，否则 novelty 站不住）
 
> **为什么前置**：多峰、非连续、密度区域在别的设定里都有人做过。不主动切清，审稿人第一反应是"这不就是 XX 的离散版？"我们的 novelty 必须精确落在"离散有序 + 最短覆盖 + 精确组合求解"上。
 
- **vs min-CPS / min-RCPS**：它们是 mode-anchored 单区间的精确最优解。我们不说"更好"，只说面向**互补的多峰情形**——anchored 单区间在那里把长度浪费在低密度山谷上。
- **vs Chakraborty et al. 2024（OPS）**：他们也构造有序的**非连续**集合，但机制是 conformal p-value + multiple testing / FWER control，目标是 coverage。我们的目标是 **efficiency**：区间数预算下求最短覆盖，用精确 DP。同样产出非连续集，出发点和机制都不同。
- **vs 回归里的 highest-density conformal（Sampson–Chan、CHCDS、Tumu 等）**：它们处理连续/几何输出空间。我们处理**离散有序标签线**，预测对象是若干区间的并，能精确组合求解。"highest-density + conformal"不是卖点。
---
 
## 3. Technical Method（方法，以及每步为什么这么设计）
 
### 3.1 核心对象：最短覆盖区域（constraint form）
 
模型对样本 x 输出概率 $p_1(x),\dots,p_K(x)$。"最多 M 段"区域 $B=\bigcup_{j=1}^{m}[l_j,u_j]$（$m\le M$，互不相交），覆盖概率 $P_x(B)=\sum_{y\in B}p_y(x)$，长度 $|B|=\sum_j(u_j-l_j+1)$。
 
给定概率预算 $\tau$，base 版本 **LMO-CPS**：
 
$$R^{\tau}_{M,a}(x)=\arg\min_{B\in\mathcal U_M}|B|\quad\text{s.t.}\quad P_x(B)\ge\tau,\quad[\text{可选 anchor}:\ a(x)=\hat m(x)\in B].$$
 
> **为什么是这个形式，不是"max 概率 − λ·长度"（penalty form）**：penalty form $\max_B[P_x(B)-\lambda|B|]$ 固定 λ 虽 $O(KM)$ 可解，但 **M=1 时不退回 min-CPS**——单峰下它的最优解是"密度超 λ 的那段"，对应分数是原始密度 $p_y$（HPS 风格），而 min-CPS 要包含一个标签所需的是 cumulative mass（APS 风格），两者校准出的集合不同。constraint form 则 M=1 + anchor 时**逐字就是 min-CPS**，分数正好是 cumulative mass。**继承 min-CPS 比省一个 K 因子重要**，所以主方法用 constraint form。
 
### 3.2 anchor 是一个开关
 
`anchor=on`（强制 $\hat m\in B$）退回 min-CPS；`anchor=off` 允许跳过 mode、面向多峰。
 
> **为什么做成开关**：一个框架同时覆盖两端。且（定理 1）在 min-CPS 的假设（单峰）下，开和关给同一个解。所以"去掉 anchor 是卖点"和"M=1 退回 min-CPS"不矛盾：anchor 关 + 多峰，才是超出 min-CPS 的地方。
 
### 3.3 怎么精确求解：动态规划
 
对每个长度 $\ell$ 算"长度不超过 $\ell$ 时最多 M 段能圈住的最大概率"：
 
$$P^{*}_M(\ell)=\max_{B\in\mathcal U_M,\,|B|\le\ell}P_x(B).$$
 
base 版本要解 $R^{\tau}_{M}$，找最小 $\ell$ 使 $P^{*}_M(\ell)\ge\tau$。
 
**正则化版本不能简单写成 $P^*_M(\ell)-\gamma\ell$**，因为 $P^*_M(\ell)$ 的 argmax 实际长度可能 < ℓ，那样 $\gamma\ell$ 会算错。正确做法是直接对 regularized mass 建 frontier：
 
$$F^{*}_{M,\gamma}(\ell)=\max_{B\in\mathcal U_M,\,|B|\le\ell}\big[P_x(B)-\gamma|B|\big],$$
 
再找最小 $\ell$ 使 $F^{*}_{M,\gamma}(\ell)\ge\tau$。（$\gamma=0$ 退回 base。）
 
DP 状态 = （看到第 k 个标签，已用 m 段，当前标签是否在一段区间内），逐标签转移，配 backpointer 复原区间；算完所有 $\ell$ 是 $O(K^2M)$。M=1 退回 min-CPS 的线性时间滑窗 $O(K)$。
 
> **为什么用 $O(K^2M)$ DP，而不是 penalty form 那个更快的 $O(KM)$**：要覆盖**完整的**"长度—概率"权衡曲线（Pareto front）。penalty form 扫 λ 只恢复曲线上的 supported 点（凸包上的点），凹陷处够不着；min-CPS 要对每个 $\tau$ 都给解，包括凹陷点。length-indexed 的 $P^*_M(\ell)$ DP 覆盖所有 $\tau$，才和 min-CPS 对齐。有序任务 K 不大，$O(K^2M)$ 跑得动，为继承付这个代价值得。
 
> **为什么 regularized 要单独写 $F^*_{M,\gamma}$**：因为"长度 ≤ ℓ 的最大 mass"的 argmax 长度未必等于 ℓ，直接 $P^*_M(\ell)-\gamma\ell$ 会把惩罚算在错误的长度上。把 regularized mass 整体放进 DP 的 frontier 才正确。附录给 exact-length frontier $P^=_M(\ell)=\max_{|B|=\ell}P_x(B)$ 的等价写法。
 
### 3.4 共形分数：进入区域所需的最小概率预算（精确可算）
 
光"圈住概率 $\tau$"不等于"真标签 90% 落在里面"——模型概率不可信。把区域当**打分器**，再用留出的校准数据标定。定义
 
$$S_M(x,y)=\inf\{\tau:\ y\in R^{\tau}_{M}(x)\},\qquad S_{M,\gamma}(x,y)=\inf\{\tau:\ y\in R^{\tau,\gamma}_{M}(x)\}.$$
 
分数越小，y 在更小预算下就被最优区域选中、越"结构上可信"。
 
> **怎么实现这个 inf（不必做连续 τ，也不必只用 grid）**：constraint form 下，最优长度只在 $\tau=P^*_M(\ell)$ 这 K+1 个值处跳，所以 τ 的 breakpoints 就是 $\{P^*_M(\ell)\}$。于是 entry score 可以**精确**算：遍历长度 $\ell=0,\dots,K$，取每个长度的最优 region（DP backpointer 复原），找**首次包含 y** 的最小 $\ell$，则 $S_M(x,y)=P^*_M(\ell)$；从未被包含则 $+\infty$。这是 $O(K^2M)$ DP 的自然副产品，不引入 grid 超参。（K 很大时，可退而用有限 grid $\mathcal T$ 上的 $\min\{\tau\in\mathcal T:y\in R^\tau_M\}$ 作简化 fallback。）regularized 同理，用 $F^*_{M,\gamma}$ 的 breakpoints。
 
> **为什么分数这么定义**：其一，和 min-CPS 对得上——M=1 + anchor 时它正好是"从 mode 累积到 y 的概率质量"。其二，把"长度—概率"几何压成标量，方便校准。分数单位是 **mass，不是 density**——这是它能退回 min-CPS（APS 风格）而 penalty form 退不回（HPS 风格）的原因。
 
> **确定性 tie-breaking（必须 operational，否则 $S_M$ 不是稳定函数）**：当存在多个最短最优 $B$，按以下顺序唯一选定：
> 1. **anchor-on 时，优先包含 anchor 的解**（保证 reduction：plateau/ties 下不会选到不含 mode 的等价解）；
> 2. 最大 $P_x(B)$；
> 3. 最少 components；
> 4. 最小 hull length；
> 5. lexicographically leftmost。
>
> anchor-off 时跳过第 1 条。关键不是这组顺序唯一正确，而是必须确定——否则分数不稳定、复现和覆盖论证都出问题。
 
### 3.5 用 split conformal 保证覆盖率
 
校准集上算每个真标签分数 $S_M(X_i,Y_i)$，取 $\widehat q=Q_{1-\alpha}(\{S_i\})$，输出 $\widehat C(x)=\{y:S_M(x,y)\le\widehat q\}$。标准 split conformal 给 $\mathbb P\{Y\in\widehat C(X)\}\ge 1-\alpha$。
 
> **为什么不需要 radial monotonicity**：min-CPS 的覆盖证明走 nested-set route，"嵌套"需要 radial monotonicity。我们走 label-level 打分器路线：只要 $S_M$ 在校准前冻结、是 $(x,y)$ 的确定函数，split CP 自动给覆盖，**和嵌套无关**。这是相对 min-CPS 的干净好处，但要诚实——只是换证明路线绕开假设，不是更强的覆盖定理。
 
> **必须讲明**：多峰下 $R^{\tau}_{M}$ 随 $\tau$ 增大**不一定嵌套**（可能跳峰），所以 $\widehat C(x)$ 一般**不等于** $R^{\widehat q}_M(x)$，而是"曾在某 $\tau\le\widehat q$ 被选中过"的标签的并。这不影响覆盖率，但不能让人误以为最终集合是单一最短区域。只有 M=1 + anchor + 单峰时区域才嵌套，$\widehat C=R^{\widehat q}_1$，正好是 min-CPS 的校准结果。
 
### 3.6 控制最终区间数：最小补集投影
 
最终集合区间数**不一定** ≤ M（全局校准 + 不嵌套）。若交付要严格控制，把相邻区间间**最短的几个空隙**填上，直到区间数 ≤ M′：$C_{\rm out}(x)=\Pi^{\rm sup}_{M'}(C_{\rm raw}(x))$，$C_{\rm raw}\subseteq C_{\rm out}$。
 
> **为什么安全也最优**：填空隙只让集合**变大**，覆盖率不变（超集不丢真标签）；填最短空隙是"凑到 M′ 段的最小扩张"。于是区分 **M（打分器内部预算）和 M′（输出预算）**，把"区间数不受控"变成可解释旋钮。
 
### 3.7 正则化版本：LMO-RCPS（min-RCPS 的类比）
 
$$R^{\tau,\gamma}_{M,a}(x)=\arg\min_{B\in\mathcal U_M}|B|\quad\text{s.t.}\quad P_x(B)-\gamma|B|\ge\tau,\quad[a(x)\in B\ \text{if anchor-on}].$$
 
$\gamma=0$ 退回 LMO-CPS；**M=1 + anchor + 长度惩罚 退回 min-RCPS** 可行集形式。分数/校准与 base 完全平行（§3.4–3.5）。
 
> **为什么加、为什么正则项只用长度 $|B|$**：完整继承 min-CPS→min-RCPS 的 lineage。但正则项只敢用长度，因为三种惩罚和多峰卖点关系不同：
> - **长度惩罚 $\gamma|B|$**：和多峰**兼容**——在"覆盖多少 mass"和"区间多长"间权衡，不针对分段；多峰下多段总长（$W$）本就比单段 hull（$W+G$）短，长度惩罚不会把多段推回单段。进主线。
> - **段数惩罚 $\eta(\#\text{comp}-1)$**：和多峰**冲突**——它惩罚的"多段"正是我们的卖点，$\eta$ 大就退回单段。本质是"往 min-CPS 退"的旋钮，**不是效率提升**。进附录，诚实定位。
> - **gap 惩罚 $|\text{Hull}(B)|-|B|$**：冲突**最直接**——跨大山谷正是多峰核心场景。不加。
 
> **为什么用可行集正则化、而非 penalty-form 目标**：可行集形式保 constraint-form 的"阈值 + 最短"结构，M=1 退回 min-RCPS；penalty-form $\max_B[P_x(B)-\lambda|B|]$ 是 argmax，退不回、只覆盖 supported 点，降为附录伴侣（§3.8）。
 
> **为什么是 corollary、一个诚实提醒**：不让这篇变成"正则项设计 paper"，主 theorem 仍是 DP + reduction + validity + projection + 几何。诚实：min-RCPS 原文报告过适当正则再压几个百分点，**但那是单区间结果，不能当"我们多区间也会提升"的论据**。正则化版本定位是 **lineage 完整 + 可控 trade-off**，不是效率卖点。
 
### 3.8 penalty form 去哪了
 
penalty form $\max_B[P_x(B)-\lambda|B|]$ 进附录作 Lagrangian 伴侣：固定 λ 可 $O(KM)$ 快速求解，可作计算加速。但它只恢复 Pareto front 的 supported（凸包）点，**不是对每个 $\tau$ 的精确等价**，也**不是** LMO-RCPS（后者是可行集正则化、能退回 min-RCPS），不当主方法、不混为一谈。
 
### 3.9 数据划分协议（soundness 关键）
 
四份不重叠数据，各司其职：
 
| 数据 | 用途 |
| --- | --- |
| $D_{\rm train}$ | 训练 ordinal 模型 |
| $D_{\rm tune}$ | 选择 $M,\gamma,M'$、tie-breaking 变体、projection 策略（以及若用 grid，则选 $\tau$-grid） |
| $D_{\rm cal}$ | **只**计算最终 conformal 分位数 $\widehat q$ |
| $D_{\rm test}$ | **只**报告结果 |
 
> **为什么必须写死**：如果用同一份数据既选结构/超参又算校准阈值，会破坏 exchangeability、毁掉覆盖保证。这是 reviewer 对这类项目最常抓的 soundness 漏洞（"你是不是用 calibration set 同时调结构和阈值？"），提前分清楚就堵住了。所有超参在 $D_{\rm tune}$ 上定、在 $D_{\rm cal}$ 前冻结，覆盖保证才严格成立。
 
---
 
## 4. Theory（定理，以及每条为什么这么陈述）
 
### 定理 1（退回 min-CPS）
 
constraint form 下，M=1 + anchor 时 $R^{\tau}_{1}$ 就是 min-CPS 样本级问题。在 **strict radial monotonicity**（严格单调、唯一 mode）下，$R^{\tau}_1$ 嵌套，于是 $\{y:S_1(x,y)\le\widehat q\}=R^{\widehat q}_1(x)$，**整条流程（问题、$O(K)$ 算法、分数、校准）就是 min-CPS**。同样条件下，不打开 anchor 的 M=1 最优解也自动含 mode（只要 tie-breaking 按 §3.4，anchor 优先），与 anchored 解一致。
 
> **为什么要 strict + tie-breaking**：radial monotonicity 允许相等（plateau）。有等概率标签、出现并列最短区间时，anchor-free 可能因 tie 选到不含 mode 的等价区间。所以正式陈述要么假设严格单调、要么用 §3.4 的 anchor-优先 tie-break。min-CPS 的覆盖定理本就建立在 radial monotonicity（含唯一 mode）上，reduction 落在它自己的假设域内，恰好覆盖 min-CPS 所有能保证生效的情形。
 
### 推论 1′（退回 min-RCPS）
 
定理 1 设定上，把约束换成 $P_x(B)-\gamma|B|\ge\tau$（LMO-RCPS，$c(B)=|B|$），M=1 + anchor 退回 min-RCPS 可行集构造；$\gamma=0$ 退回 LMO-CPS。固定 $\gamma$（或在 $D_{\rm tune}$ 上选定后冻结），entry-score 的 split conformal 给边际覆盖。
 
### 定理 2（精确求解 + 复杂度）
 
DP 精确求出 $R^{\tau}_{M}$ 与 $R^{\tau,\gamma}_M$（base 用 $P^*_M(\ell)$，regularized 用 $F^*_{M,\gamma}(\ell)$），复杂度 $O(K^2M)$；M=1 退回 $O(K)$。
 
### 命题 3（确定性的几何增益）
 
设高概率质量集中在 $m\le M$ 个被山谷隔开的峰上，各峰宽度之和 $W$，山谷总长 $G$。则任何要同时盖住这些峰的**单段区间**长度至少 $W+G$；一个 **M 段区域**只需 $W$。结构性节省是 $G$。
 
> **为什么写成纯几何，不说"真标签在非 mode 峰所以更短"**：方法构造集合时**不知道真标签在哪**，只看模型概率几何。把"真标签落点"（随机事件）和"区域长度"（确定几何量）混在一句里逻辑错。正确分两步：先说确定的几何事实（单区间多付 $G$，多区间不用）；再单独一句经验结论——真标签确实集中在这些峰上时，几何节省转化为更小的共形集。前者是定理，后者是实验问题。
 
### 定理 4（覆盖保证）
 
固定 (M, γ, M′, 超参)、或在 $D_{\rm tune}$ 上选好后冻结，则 $\widehat C(x)$ 满足边际覆盖 ≥ 1−α（标准 split conformal）。
 
> **关于"全局校准稀释多峰增益"，要诚实**：全局只用一个 $\widehat q$，若数据大多单峰，$\widehat q$ 被它们主导，多峰样本增益可能被摊薄。可用**按多峰程度分层校准**（Mondrian）缓解，它给分层内覆盖率。但**必须说清：分层校准不"恢复区域嵌套性"，也不保证"每个样本增益被保住"**——它只是可选经验手段，不是覆盖率所必需，覆盖率本身已由 split CP 给出。
 
### 命题 5（最小补集投影）
 
填最短空隙得到的，是有序线上最小的 M′ 段超集，且保覆盖（§3.6）。
 
### 退化检查（M=∞）
 
不限区间数时，constraint form 最优解是"按概率从高到低选标签，直到累积质量达 $\tau$"，即离散 top-density / HPS oracle 集合。它可等价写成某密度 cutoff 下的集合，但 cutoff 由 $\tau$ 和排序决定，不是外部 λ。
 
> **为什么讲、为什么强调 cutoff 由 τ 决定**：讲 M=∞ 是为说明卖点是**有限 M 的有序拓扑约束**，不是密度阈值本身。强调 cutoff 由 τ 决定，是因为 constraint form 主线下没有外部 λ——带 λ 的写法留给 penalty form 附录，别混进主线。
 
### 附录定理（penalty 伴侣 / 精确路径 / 段数惩罚）
 
- **penalty 伴侣**：$V(\lambda)=\max_\ell[P^*_M(\ell)-\lambda\ell]$ 是 $K+1$ 条直线的上包络；折点是相邻长度的**边际平均密度**（不是单个 $p_y$）。penalty form 沿包络只恢复 supported（凸包）点，是伴侣，不是逐 $\tau$ 等价、也不是 min-RCPS analogue。
- **段数惩罚**：$P_x(B)-\gamma|B|-\eta(\#\text{comp}(B)-1)\ge\tau$。**诚实定位**：$\eta$ 是"简洁性 ↔ 多峰适应性"的 controlled trade-off 旋钮，$\eta$ 增大把输出推回单区间（趋近 min-CPS），**不是**效率提升；任何"变好"都来自在不该用多区间的数据上退回单区间。gap 惩罚直接罚跨山谷（主场景），不纳入。
---
 
## 5. Experiment Design（实验设计，以及为什么这么设计）
 
围绕"多峰"，不能只报平均集合大小。
 
### 5.1 先做两个门控（Gate 0 / Gate 1，第一周任务）
 
> **为什么放最前**：增益**只在多峰时出现**（命题 3）。投入完整对比前先用两个便宜诊断回答"有没有戏"，kill-fast。**这是学生第一周的任务，不是实验后半段的 analysis。**
 
- **Gate 0：真实数据多峰审计。** 统计 strong-multimodal 占比（local maxima 数、valley ratio、top-2 峰间距、次峰质量、山谷质量占比）。若只有 2–3%，不适合当主线。
- **Gate 1：base-region oracle 增益。** 共形校准**之前**，比较 anchored min-CPS / anchored min-RCPS / 无 anchor M=1 / M=2 / M=∞ 的 base 长度。若 base 层都没明显增益，校准只会进一步吃掉它，不往下走。
### 5.2 合成数据（必做）
 
$$p(y\mid x)=\omega\cdot\text{Bump}(\mu_1,\sigma_1)+(1-\omega)\cdot\text{Bump}(\mu_2,\sigma_2)+\epsilon,$$
 
可控：峰间距、山谷深度、不平衡度 $\omega$、峰数、标签数 $K$、校准集大小。
 
> **为什么必做**：真实数据"有多峰"既不可控也不易诊断。合成能精确调节多峰程度，干净展示"峰越远、山谷越深，M=2 相对单区间增益越大"，同时验证覆盖率达标。它是把命题 3 的几何增益"看得见"的地方。
 
### 5.3 真实数据（与 min-CPS 对齐）
 
UTKFace（年龄）、IMDB（年龄）、Avocado（价格档）、Electric Motor Temperature（温度档）——min-CPS 同款，apple-to-apple。
 
### 5.4 对比方法（主表只放四个，避免爆炸）
 
主表：**min-CPS、min-RCPS、LMO-CPS (M=2)、LMO-RCPS (M=2)**——两两对照"加多区间"和"加正则化"两个维度。扩展表：Ordinal APS、Chakraborty OPS、LMO-CPS (M=1)（=anchor-free min-CPS）、投影前 vs 后。附录 ablation：长度惩罚 $\gamma$ 的 sensitivity（主文只报 tuning 后结果）；段数惩罚 $\eta$ 的 sensitivity（诚实展示是 compactness↔multimodality 的 trade-off，不是提升）。
 
### 5.5 必报指标（每个回答一个具体质疑）
 
覆盖率（必须 ≥ 1−α）；平均集合大小；**多峰子组集合大小**（主战场）；单峰子组退化幅度（trade-off 可控）；最终区间数；top-1 排除率（回答"凭什么不含最自信标签"）；hull 大小；运行时间。
 
> **为什么强调分层、不只报平均**：诚实讲，这方法在单峰样本上可能和 min-CPS 持平甚至略差（全局 $\widehat q$ + 多区间的代价），价值集中在多峰子组。只报平均会抹平故事。分层报告（单峰 / 弱多峰 / 强多峰）才能既展示增益又暴露代价，让结论站得住。
 
### 5.6 投 AAAI 的最低成立标准（写死，作为 go/no-go 硬条件）
 
1. 理论 spine 成立：exact DP、min-CPS/min-RCPS reduction、split CP validity、projection 命题、多峰几何命题。
2. synthetic 漂亮：随 mode separation / valley depth 增大，M=2 对单区间优势系统增强。
3. 真实数据不能只靠 synthetic：至少一个 min-CPS 同款数据集有足够 strong-multimodal subgroup。
4. subgroup gain 硬：strong-multimodal subgroup 相对 min-RCPS 至少 10–15% size reduction。
5. overall 不受伤：总体平均不比 min-RCPS 差超过 1–2%，最好持平或小赢。
6. top-1 exclusion 可解释：不高到让人觉得违背 ordinal 决策直觉。
7. projection 不吃光 gain：M′-投影后仍有可见收益。
8. Chakraborty OPS 进 extended baseline：即使主表不放，也要 appendix 比较或实现讨论。
---
 
## 一句话总览
 
> **LMO-CP** 把 min-CPS 从 mode-anchored 单区间，推广到 anchor-free、最多 M 段的最短覆盖区域；对每个概率预算 $\tau$ 用 DP 精确求解，把（非嵌套的）区域路径转成 label-level 进入分数 $S_M(x,y)$（可在 $\{P^*_M(\ell)\}$ 断点上精确计算），split conformal 校准得有限样本覆盖，必要时投影成最小 M′ 段超集。**LMO-RCPS** 把 mass 约束换成 mass−长度约束，继承 min-RCPS。M=1 + anchor + 单峰特例精确退回 min-CPS（加长度惩罚退回 min-RCPS）；瞄准互补的多峰情形——那里 anchored 单区间把长度浪费在低密度山谷上。
 
### 设计决策一览
 
- 主方法用 constraint form（不用 penalty form）→ 只有它的 M=1 退回 min-CPS。
- anchor 做成开关 → 一框架同为 min-CPS（开）和多峰推广（关）。
- DP 用 $O(K^2M)$ → 覆盖完整 Pareto front，K 小代价可接受。
- 分数用 mass 而非 density → 退回 min-CPS（APS 风格）的关键。
- 分数在 $\{P^*_M(\ell)\}$ 断点上精确算 → 不必连续 τ、也不必只 grid 近似。
- regularized DP 用 $F^*_{M,\gamma}(\ell)$ → 不能写成 $P^*(\ell)-\gamma\ell$（argmax 长度未必等于 ℓ）。
- 确定性 tie-break，anchor 优先 → 保 reduction、保分数稳定。
- 数据四分 train/tune/cal/test → 堵 soundness 漏洞。
- split conformal 而非 nested-τ → 不依赖 radial monotonicity（只是换证明路线）。
- 明说非嵌套下 $\widehat C\ne R^{\widehat q}_M$ → 不误导读者。
- M / M′ 双预算 + 最短空隙投影 → 区间数变可解释旋钮，保覆盖。
- 正则化只加长度惩罚（主线），段数/gap 惩罚进附录 → 后两者罚的正是卖点。
- 正则化用可行集形式、是 corollary、不照搬 min-RCPS 数字 → lineage 完整而非效率卖点。
- 命题 3 写成纯几何 → 不混真标签落点与确定长度。
- 删"分层恢复嵌套" → overclaim；分层只给分层覆盖。
- 两个前置 gate + 八条成立标准 → 增益只在多峰，先 kill-fast。
---
 
## 附录 A：学生执行清单
 
### Week 1 deliverables
 
1. **DP 实现**：`solve_dp_base(p, tau, M, anchor=None)`、`solve_dp_reg(p, tau, M, gamma, anchor=None)`，返回 `components, length, mass, regularized_mass`。状态 `dp[k][m][ell][s]`（`s=1` 表示标签 k 被选且当前区间打开），转移 = skip / 续区间 / 开新区间，backpointer 复原区间；anchor-on 用额外状态 `has_anchor` 或后筛只接受含 anchor 的解。**brute-force 自检**：$K\le 12$ 时枚举所有最多 M 段区域，验证 DP 输出完全一致。
2. **分数构造**：`entry_score(p, y, M, gamma=0, anchor=None)`（遍历长度断点精确算 $S_M$）、`build_raw_set(p, qhat)`。
3. **投影**：`project_to_Mprime_components(C_raw, M_prime)`；单测：填最短空隙 = 最小超集。
4. **合成数据**：双峰 ordinal 生成器，控 separation / valley depth / imbalance。
5. **Gate 0 / Gate 1 memo**：多峰审计 + base-region oracle gain + stop/go 决策。
### Week 2 deliverables
 
1. split conformal 原型；2. min-CPS / min-RCPS baseline；3. LMO-CPS / LMO-RCPS (M=2) 对比；4. 投影前 vs 后；5. top-1 排除率；6. subgroup 结果表。
> 留白、等 pilot 后再补：定理证明细节、命题 3 增益常数、intro 卖点措辞强度。
 