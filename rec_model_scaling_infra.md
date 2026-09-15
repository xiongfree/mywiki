# 推荐系统 Scaling&Infra：25 篇核心论文逐篇深度详解

> 本文档覆盖 25 篇推荐系统 Scaling 核心论文，每篇按"做了什么 / 为什么这么做 / 原始论文深度新见解 / 工业验证数据 / 对 Scaling 的意义"组织，之后是八个横向维度：跨论文对比、瓶颈依赖链、工程实践映射、前瞻性分析、Roofline 分析、"三层前提"框架、主线间交叉引用、深度新见解汇总。
>
> **关于引用可靠性**：正文见解以论文原文与可核实的公开材料为依据。凡属本文档归纳或推断的内容均就地标注（形如"这是本文档的归纳而非论文结论"）；跨论文的并置与对照关系除明确标注引用外，均为本文档的整理视角，不代表论文之间存在引用关系。数值存在多版本差异的论文（如 SORT、HeMix、RankUp）已注明版本口径，引用前请核对目标版本原文。

---

## 目录

- [第一部分：25 篇论文逐篇深度详解](#第一部分25-篇论文逐篇深度详解)
  - [主线一：Token 化（4 篇）](#主线一token-化4-篇)
  - [主线二：统一交互 Block（9 篇）](#主线二统一交互-block9-篇)
  - [主线三：长序列（3 篇）](#主线三长序列3-篇)
  - [主线四：推理解耦（2 篇）](#主线四推理解耦2-篇)
  - [主线五：底层基建（3 篇）](#主线五底层基建3-篇)
  - [主线六：生成式推荐（2 篇）](#主线六生成式推荐2-篇)
  - [主线七：表征健康（2 篇）](#主线七表征健康2-篇)
- [第二部分：八个补充维度](#第二部分八个补充维度)
  - [维度一：跨论文对比分析](#维度一跨论文对比分析)
  - [维度二：瓶颈依赖链（八层因果链）](#维度二瓶颈依赖链八层因果链)
  - [维度三：工程实践映射](#维度三工程实践映射)
  - [维度四：前瞻性分析](#维度四前瞻性分析)
  - [维度五：GEMM 统一度与 Roofline 量化分析](#维度五gemm-统一度与-roofline-量化分析)
  - [维度六："三层前提"框架](#维度六三层前提框架)
  - [维度七：主线间交叉引用关系](#维度七主线间交叉引用关系)
  - [维度八：从原始论文提取的 14 个深度新见解](#维度八从原始论文提取的-14-个深度新见解)

---

# 第一部分：25 篇论文逐篇深度详解

## 主线一：Token 化（4 篇）

### 1. OneTrans — 统一跨场景 Token 化范式

**论文信息**：arXiv:2510.26104, WWW 2026, ByteDance

**做了什么**

OneTrans 提出推荐系统的统一 Token 化框架，将多场景、多任务推荐统一到单一 Transformer 架构。核心创新四件套：Auto-Split Tokenizer（自动分割分词器）、Mixed Parameterization（混合参数化）、Pyramid Stack（金字塔堆叠）、Cross-Request KV Caching（跨请求 KV 缓存）。所有场景的输入统一到同一 Token 序列空间，实现跨场景知识迁移和参数共享。

**为什么这么做**

传统推荐系统中不同场景各自维护独立模型，特征工程重复、参数无法共享、在线服务成本高昂。Auto-Split Tokenizer 解决不同场景特征维度不一致问题；Mixed Parameterization 在共享和场景专属参数间取得平衡；Pyramid Stack 通过分层抽象实现渐进式特征交互；Cross-Request KV Caching 解决在线推理重复计算。

**原始论文深度新见解**

- **Auto-Split Tokenizer 的本质**：非简单特征拼接，而是将变长特征序列自动分割为等长 Token 块，使不同场景输入统一到同一序列空间，实现 Token 级别跨场景注意力而非传统特征级交互。
- **Mixed Parameterization 的金字塔分配**：底层 Embedding 和浅层 Transformer 全场景共享（通用模式），中层按场景子集分组（半专属），顶层完全场景专属（定制化输出）。参数分配策略与 Pyramid Stack 结构对齐。
- **Cross-Request KV Caching 的工程价值**：同一用户在不同请求间历史行为序列高度重叠，缓存已计算 KV 对可减少约 30% 运行时间和约 50% 内存占用，这是 OneTrans 工程落地的关键。
- **Causal Attention + Cross-Request 的一致性**：因果注意力保证请求内时序正确性，跨请求缓存复用历史 KV 不违反因果约束——历史请求的 KV 对当前请求是"已发生"的，可安全复用。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| Feeds GMV/用户 | +5.68% | 电商推荐 |
| Feeds 订单/用户 | +4.35% | 电商推荐 |
| Mall GMV/用户 | +3.67% | 电商推荐 |
| Mall 订单/用户 | +2.58% | 电商推荐 |
| 冷启动订单/用户 | +13.59% | 电商推荐 |

**对 Scaling 的意义**：OneTrans 证明了"统一 Token 空间 + 混合参数化"可以替代多场景独立模型，参数共享使跨场景 Scaling 不再受场景数量线性扩展的约束。

---

### 2. MDL — Multi-Distribution Learner via Tokenization

**论文信息**：arXiv:2602.07520, ByteDance Search

**做了什么**

MDL（Multi-Distribution Learning）受 LLM Prompting 范式启发，把场景信息与预估目标信息 Token 化，当作 Prompt Token 从最底层注入统一交互 Block，而不是当作辅助输入或门控信号。

Token 化产出三类 Token：特征 Token `T_f`（用户、商品、序列、交叉特征按语义分组后投影到固定维度）、场景 Token `T_s ∈ ℝ^{(N_s+1)×d_s}`、任务 Token `T_t ∈ ℝ^{N_t×d_t}`，后两类由场景/任务相关特征经 Pertoken FFN 变换得到。场景 Token 数等于场景数、任务 Token 数等于预估任务数；除此之外还额外构造一个跨场景共享的**全局场景 Token** `t_{s,global}` 专门学习场景间共性知识。

交互分三类：(1) **特征 Token 自交互**——直接沿用 RankMixer 的 `PertokenFFN(LN(TokenMixing(T_f) + T_f))`，论文注明这部分可替换为其他特征交互方法；(2) **Domain-aware Attention**——cross-attention 的变体，以场景/任务 Token 作 query、特征 Token 作 key 和 value，QKV 投影本身也是 Pertoken FFN 形式，作用是让场景/任务自适应地激活相关的特征子空间；(3) **Domain-fused Module**——两步走，先按实例所属场景从全量场景 Token 中选出对应的那几个（场景可重叠，故可能多选）并连同 `t_{s,global}` 一起 mean pooling，再把聚合后的 `t_{s,avg}` 融合进每个任务 Token，从而实现"特定场景下的特定任务"的差异化预测。

**为什么这么做**

论文点出的两个短板与常见叙述不同，值得照原文记：(1) **大规模模型参数利用不足**——场景/任务信号与复杂特征交互模块的交互太有限，通常只作用在中间层或输出层（比如 MMoE 的 gate、STAR 的 tower），庞大的特征交互参数空间没被这些信号真正激活；(2) 场景与任务难以在统一框架下联合建模。

MDL 的应对逻辑是把注入点从"中间/输出层"提前到"最底层的 Token 化"：一旦场景与任务成为 Token，它们就能逐层、自底向上地与特征信息充分交互，从而"prompt"并激活整个参数空间。这是它自称受 prompting 启发的实际含义——不是写自然语言指令，而是把分布标识做成能参与每一层计算的一等公民。

**原始论文深度新见解**

- **"Prompt"在这里指注入位置，不是指令语义**：LLM 的 Prompt Token 是指令，MDL 的 Prompt Token 是分布标识——当前样本属于哪个场景、要预估哪个目标。真正被借用的是"信息从输入侧进入、逐层参与计算"这一机制，而非 in-context learning 那套语义。论文自己的表述是 bottom-up、layer-wise 地 prompt 并激活参数空间。
- **方向性很关键：场景/任务当 query，特征当 key/value**：Domain-aware Attention 是单向的 cross-attention，场景/任务 Token 去"查询"特征 Token，选择性地放大与自己相关的特征子空间；特征 Token 侧则由自交互层独立演化。这个方向不能反——如果让特征当 query，得到的是"特征关注哪个场景"，与"按场景激活特征"的意图正相反。另需注意交互对象是全体特征 Token，不限于行为序列。
- **Domain-fused 是实例级的稀疏选择加均值池化，不是"汇聚到输出"**：它做的是先按实例所属场景挑出对应场景 Token（场景重叠时可多选）再连同全局场景 Token 一起 mean pooling，最后加进任务 Token。设计意图是"特定场景下的特定任务"这个组合的差异化，而全局场景 Token 的存在保证了共性知识不会被稀疏选择切断——这是同时兼顾共享与差异的地方。
- **注意力可视化是这篇论文少见的可解释性证据**：论文画出了任务 Token 与特征 Token 之间、以及场景 Token 与特征 Token 之间的注意力分布（横轴为特征 Token 索引，纵轴为平均注意力权重），并按层、按任务分别展示。这类图给"场景/任务确实在选择不同特征子空间"提供了直接证据，而不只是靠指标涨跌反推。
- **查询变更率是搜索场景特有的代理指标**：除 QAUC 外，MDL 用"查询变更率"（用户改写搜索词的频率）衡量效果——变更率下降说明首轮结果更贴合意图。这是搜索业务才有的信号，做纯推荐场景无法照搬。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| LT30 | +0.0626% | 抖音搜索，为期一个月的线上 A/B |
| 查询变更率（change query rate） | −0.3267% | 抖音搜索，同一实验 |
| 离线 QAUC（Query-level AUC） | 单列/双列/站内搜索三场景、Click/Like/Fav 三任务共 9 项全面优于最强基线，相对提升 +0.23%~+0.77% | 工业数据集 |
| 部署范围 | 全量部署，日服务数亿用户 | 生产环境 |

> 注：LT30 在论文中未展开定义，此处不作释义，引用时请核对原文。对照基线包括 RankMixer、SharedBottom、MMoE、STAR、HMoE、PEPNet。

**对 Scaling 的意义**：MDL 的 Scaling 论证是直接做出来的——通过调隐层维度 d 和层数 L 扫参数量与 FLOPs，与 MMoE 对比发现两点：MDL 始终更好，且**参数量/FLOPs 越大，MDL 相对 MMoE 的增益越大**。第二点才是关键，它说明门控式方案的瓶颈不在容量而在信号注入方式：gate 只在局部起作用，堆再多参数也激活不了；Token 化让分布信号逐层参与，参数才用得上。这与论文开篇列的第一个短板"参数利用不足"首尾呼应。另一层意义是场景/任务的扩展代价从"加 tower/加分支"变成"加 Token"，架构不必随场景数增长而改动。

---

### 3. UniMixer — 统一 Attention / TokenMixer / FM 三大范式

**论文信息**：arXiv:2604.00590, Kuaishou

**做了什么**

UniMixer 提出统一理论框架，将推荐系统三大 Scaling 范式——attention-based（HiFormer、FAT、HHFT）、TokenMixer-based（RankMixer、TokenMixer-Large）、factorization-machine-based（Wukong、Kunlun）——纳入同一参数化体系。注意这里的 FM 指 Factorization Machine（因子分解机），不是 Foundation Model。

核心链条是：先证明 TokenMixer 那套基于规则的 token 混合等价于乘一个排列矩阵 `W_perm`，再观察到 `W_perm` 有四条性质——可压缩（可分解为 Kronecker 积 `W_perm = G ⊗ I`，其中 G 是全局混合矩阵、I 是局部恒等阵）、双随机（每行每列和为 1）、稀疏（每行每列恰一个非零元）、以及 T=H 时对称。直接参数化 G 和 I 把参数量从 O(T²D²) 降到 O(T⁴+(D/T)²)。

在此基础上的 UniMixing 模块做了三件事：(1) 放弃 T/D 的划分，改用块（block size B、块数 (L//B)²）来描述排列矩阵，把恒等的局部阵换成每块独立可学的 `W_B^i`，得到 `UniMixing(X) = reshape((W_G ⊗ {W_B^i}) flatten(X), 1, L)`——`W_B^i` 控块内交互、`W_G` 控块间交互，且不再要求 token 数等于 head 数；(2) 用 Sinkhorn-Knopp 迭代（先取指数保正、再交替按行列归一化）满足双随机性，用温度系数控制稀疏度，用 (W+Wᵀ)/2 满足对称性；(3) 优化计算流水线，避免重构 `W_perm` 时产生 [TD, TD] 的巨大中间变量。另有轻量版 UniMixing-Lite 用基矩阵组合 + 低秩近似进一步压缩，以及沿用外部工作的 SiameseNorm 解决深层训练稳定性。

**为什么这么做**

论文的出发点是推荐与 NLP 的一个根本差异：LLM 里所有 token 共享同一 embedding 空间，而推荐输入是类别、数值、ID、交叉、序列等来自不同语义空间的异构特征，在两个异构语义空间之间算内积相似度缺乏物理意义，所以 Transformer 不能直接搬过来。围绕这一点工业界分裂成三条路线，各有短板：attention-based 注意力权重尖锐稀疏、梯度传播困难且是二次复杂度；TokenMixer-based 高效但混合模式是写死的规则、缺乏可学习性与场景适应性，还硬性要求 T=H；FM-based 受限于显式低阶交互，规模上去后收益有限。

UniMixer 的判断是三者本质都是"全局混合模式 × 局部混合结果"的不同实例化——把 TokenMixer 的固定排列改成可学习的参数化矩阵，就能在同一框架内取三者之长，而不必在三种离散架构间做非此即彼的选择。

**原始论文深度新见解**

- **`W_perm = G ⊗ I` 描述的是原始 TokenMixer，UniMixer 的贡献恰恰是把 I 换掉**：分解式里 G 是全局混合矩阵、I 是恒等阵，意味着原版 TokenMixer 只在块间做重排、块内什么都不做。UniMixer 把恒等阵替换为每块独立可学的 `W_B^i`，块内也获得了各自的交互模式——这才是"从规则驱动到数据驱动"的实际落点。论文附录给了一个 12 维数值例子，把 TokenMixer 的重排写成 4×4 全局混合矩阵 ⊗ 3×3 局部混合矩阵，后者在原版中正是单位阵。
- **三大范式的归约是逐个给出的，不是笼统类比**：论文明确指出，当块数 L//B 取作 T 且 `W_V^{ih}` 与 `W_B^i` 同维时，UniMixer 的局部交互投影就等价于异构注意力层的 value 投影，而 `W_G` 的维度与角色和注意力权重一致——差别只在 `W_G` 额外要满足双随机、稀疏与对称约束。FM 一侧则是：Wukong 的核心交互项 `XI(XI)ᵀY` 恰好对应注意力模块中 `W_Q = W_K = I` 的特例。所以"统一"是通过约束的松紧来实现的，注意力是最松的一端、TokenMixer 是最紧的一端。
- **UniMixing-Lite 的两级压缩，以及"更小反而更好"**：局部一侧不再给每个块配一套独立参数，而是准备一组基矩阵、按块学不同的组合系数；全局一侧用低秩近似。结果是参数更少但效果更好——表中 UniMixer-Lite-4-Blocks 用 38.2M 参数 /1.26T FLOPs 就拿到 AUC 0.752327，已超过 135.5M 的 RankMixer（0.749329）和 101.5M 的 UniMixer（0.750238）。这说明原版 UniMixer 的局部交互模式存在冗余，压缩掉冗余同时起到了正则作用。消融显示收益递减：基矩阵数量增加有效但边际下降，全局低秩的秩数存在一个折中点。
- **SiameseNorm 是引入的外部方法，解决的是 Pre-Norm/Post-Norm 之争**：SiameseNorm 不是 UniMixer 自己提出的，论文明确引用他人工作。它的机制是每层维护两条耦合的流 X̄ 和 Ȳ（初始化时都等于输入 embedding），以此化解 Pre-Norm 与 Post-Norm 的张力。引入它的动机是 RankMixer 缺少面向深层架构的设计，沿深度方向 Scaling 收益有限；TokenMixer-Large 用间隔残差加辅助损失去补，论文认为那没治本。
- **温度控制的是稀疏度，且是全模型最关键的一环**：温度越低权重越稀疏，但梯度也随之变稀疏、变弱甚至不稳定，容易陷入局部最优；而实验又显示稀疏性对效果有显著正贡献，所以稀疏不可放弃。做法是线性退火，从 τ=1.0 逐步降到 0.05。消融表明去掉温度系数带来约 0.1645pt 的 AUC 下降，是所有组件里最大的一项。
- **0.142 属于 Lite 版，不是 UniMixer 本体**：论文给出三条完整拟合式（纵轴是 ΔAUC，横轴分参数量与 FLOPs 两套）——RankMixer 为 0.002718·Params^0.116043，UniMixer 为 0.003032·Params^0.131973，UniMixer-Lite 为 0.003767·Params^0.141903。所以常被引用的"0.142 vs 0.116"是 **Lite 版**对 RankMixer，UniMixer 本体是 0.132。论文强调在两个常数中缩放指数才是主导项，而 Lite 版在指数和系数上都最高。引用提示：这几个指数出自 UniMixer 自身的对照实验，纵轴用 ΔAUC、横轴用参数量或 FLOPs，与 LLaTTE 表 3 的 α（纵轴 NE、横轴算力）口径不同，两组数字不可直接横向比较。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| CAD（Cumulative Active Days，D1–D30 累积活跃天数，剔除安装当天） | 多场景平均 +15% 以上 | 快手多个广告投放场景 |
| AUC | UniMixer-Lite-4-Blocks 0.752718（84.5M 参数），较 RankMixer（0.749329 / 135.5M）+0.8141pt | 离线，快手广告数据 |
| 缩放指数（Params） | UniMixer-Lite 0.141903；UniMixer 0.131973；RankMixer 0.116043 | 离线实验 |
| 数据规模 | 7 亿以上用户样本、覆盖整年，任务为预测首次激活后次日是否回访 | 快手广告 |

**对 Scaling 的意义**：UniMixer 证明了三大范式可在统一框架下连续插值，Scaling 时无需在架构间离散切换——通过调节 G 的参数化形式可以平滑过渡，这是"架构 Scaling"的理论基础。

**异构特征空间是这条路线的真正前提**

UniMixer 之所以绕开注意力、改走参数化排列矩阵，根子在论文开篇就点明的那句话：推荐的特征空间天生异构，在两个异构语义空间之间计算内积相似度缺乏合理的物理意义。LLM 里所有 token 共享一个 embedding 空间，所以 QKᵀ 算出来的相似度是可解释的；推荐里把一个"用户年龄"的向量和一个"商品类目"的向量做内积，得到的数值没有对应的语义。

这解释了 TokenMixer 路线为什么会存在——它用固定规则重排来实现异构特征交互，正是为了回避这个内积。也解释了 UniMixer 的定位：它没有回到内积，而是保留"不算相似度"这一前提，只把重排规则从写死改成可学。约束（双随机、稀疏、对称）的作用就是让学出来的矩阵仍然近似一个排列，不至于退化成一个普通的全连接层。

需要提示的是：UniMixer 全文没有 geometric / hypersphere 相关表述，并未主张"通过统一 Embedding 设计与超球面归一化把不同域特征对齐到同一空间"。异构特征需要空间对齐是一个成立的工程认知，但不宜挂到这篇论文名下。

### 4. TokenFormer — 分层注意力与非线性交互增强

**论文信息**：arXiv:2604.13737, Tencent（TokenFormer: Unify the Multi-Field and Sequential Recommendation Worlds）

**做了什么**

TokenFormer 解决静态多字段特征和用户行为序列特征被硬塞进同一个 Token stream 后，静态特征把序列表征"拖低"的问题（Sequential Collapse Propagation，SCP）。它是一个同构的 decoder-only 架构，把非序列字段 F、行为序列 T、目标特征 V 拉平成一条 token 流，堆叠 L 个 Unified Interaction Block（UIB），目标是在单一计算流形内原生建模 F↔F、T↔T、F↔T 等全部六类交互。

三项设计：(1) **统一 Token 流 + RoPE 位置索引**——不用类型 embedding 区分 token 种类，而是把静态特征映射到位置 p=0、行为 token 映射到时间索引、目标 token 映射到序列末端之后，让 RoPE 通过相对位置差隐式区分类型；论文的理由是加法式位置编码会迫使语义 embedding 与低秩位置向量相加，引入秩污染。(2) **BFTS（Bottom-Full-Top-Sliding）注意力调度**——浅层用完整因果注意力建立全局跨域交互，深层切到滑动窗口注意力（SWA）且窗口逐层收缩；另外在全注意力层结束后，模型完全停止关注前 M 个非序列 token，强制跨域交互在早期层内完成。(3) **NLIR（Non-Linear Interaction Representation）门控**——`Ĩ = σ(XW_g) ⊙ A`，即用当前层输入（残差路径上的原始输入）学出门控信号去逐元素调制注意力输出，再走残差；门控是单边的（只作用于一侧），目的是打破注意力的线性混合、维持秩丰富度。UIB 内 NLIR 之后还接 SwiGLU FFN。

**为什么这么做**

前几篇 Token 化论文解决的都是"如何把异构特征拼成统一 Token 序列"，但没有正面回答"不同信息密度、不同秩的 Token 强行混合是否会互相污染"这个在实践中直接影响效果天花板的问题。TokenFormer 指出：低秩的静态特征和高秩的行为序列特征不加区分地全局混合，低秩信息会像噪声一样把整体表征的有效维度拉低。

**原始论文深度新见解**

- **SCP 的成因是低基数与频次偏斜，不是"信息熵注入"**：论文的说法很具体——工业场景中许多非序列特征因低基数或频次偏斜导致信息丰度低，其 embedding 天然倾向于占据低维子空间；在解耦模型里这种坍缩被安全隔离在非序列侧，而统一模型中这些坍缩倾向的静态 token 通过共享算子直接与序列 token 交互，把序列表示也拖进坍缩。论文是从低基数与频次偏斜这一具体成因论证的，未使用"信息熵注入""信息论意义上的维度坍缩"这类理论化表述。
- **SCP 是通过一张图上的取舍关系确立的，值得照原样理解**：论文 Fig. 1 左轴用互信息（MI）衡量判别力、右轴用奇异值谱与有效秩衡量维度鲁棒性。纯序列 Transformer 维度高但判别力弱（没用上静态信号）；朴素联合建模 MI 明显上升，说明静态特征确实提供了有价值的预测线索，但谱衰减也明显变陡。所以 SCP 不是"静态特征没用"，而是"有用但有副作用"——这正是需要结构性修补而非直接丢弃的原因。
- **BFTS 的关键不只是窗口收缩，还有对静态 token 的硬性截断**：论文给出一个直接证据——深层网络中序列 token 会反常地把大量注意力权重分配给非序列位置，而这种跨域注意力在深层并无实质收益。因此在 l_f 层全注意力之后，模型完全停止关注前 M 个非序列 token。这与 OneTrans 的 Pyramid Stack 思路一致。收缩窗口的具体大小论文未给出设定方法，属于需要按数据集调的超参。
- **NLIR 的门控信号来自残差路径的输入，不是注意力输出自身**：`G = XW_g` 取的是当前层输入 X，再 `σ(G) ⊙ A` 调制注意力输出。这个来源很关键：如果门控由注意力输出自己产生，就只是一个逐元素非线性变换；由输入产生则构成一条独立的乘性通路。第三方开源实现中 `W_g` 初始化为 0、`b_g` 初始化为 1.0，使门控在训练初期接近恒等，避免早期破坏已有信息流。需要说明的是，论文对"为何乘性门控能防坍缩"给的是经验证据（分层谱分析显示 BFTS 与 NLIR 各自独立带来维度恢复），而非形式化证明；已有第三方评审指出 NLIR 与紧随其后的 SwiGLU FFN 都是逐元素乘，两者的职责边界论文交代得不够清楚。
- **消融数字显示两个组件贡献几乎相当**：相对 Vanilla Transformer，单加 NLIR 是 +4.87‰ AUC，单用 BFTS 是 +4.91‰，两者叠加时 MI 增益超过 4.9‰。论文还做了顺序实验（2F2S vs 2S2F）验证"先全注意力后滑窗"这个方向不能反。
- **与主线七"表征健康"的关联**：TokenFormer 的 Sequential Collapse Propagation 是表征坍缩在 Token 化层的表现，而 RankUp/RankElastor 应对的 effective rank 阻尼振荡（该观察出自先前的 RankMixer 研究）是同类风险在交互 Block 层的表现。两篇论文之间没有互相引用，把它们并置是本文档的归纳，用意是提示表征坍缩可能同时出现在这两个环节。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 离线 AUC（User-Centric 设定） | TokenFormer-T 0.85967（+5.00‰）、TokenFormer-L 0.86282（+8.15‰），对比 Transformer 基线 | KuaiRand-27K |
| 离线 AUC（New Impression Only 设定） | TokenFormer-S* 0.85161（+11.42‰），对比 Transformer* 基线 | KuaiRand-27K |
| 消融 | NLIR 单独 +4.87‰、BFTS 单独 +4.91‰ | KuaiRand-27K |
| 线上 GMV | +4.03% | 微信视频号广告系统 A/B |
| 线上 AUC | +0.22% | 腾讯广告生产基线对比 |

> 注：上表数值均取论文报告口径。离线评测只用了 KuaiRand-27K 这一个公开数据集（约 2.7 万用户，19k/2k/5k 划分），规模偏小；工业侧结果论文未完整列表。第三方评审也指出基线覆盖偏窄（User-Centric 设定只比了 Transformer 与 HSTU 变体，New Impression Only 只比了 OneTrans 与 HyFormer），"SOTA"的适用范围有限。

**对 Scaling 的意义**：TokenFormer 的贡献是把"共享 backbone 统一为什么容易失败"从一句经验之谈变成了一个可诊断的失效模式，并配了对应的结构补丁。它说明统一 Token 化只是必要的第一步——不同信息密度的 Token 混合需要显式的层次管理，否则"统一"本身会成为表征坍缩的源头，Token 化层的 Scaling 不能只堆 Token 数量。它引入的诊断方法（分层有效秩轨迹 + 奇异值谱 + 互信息）本身也有复用价值，可以作为判断统一架构是否健康的通用工具。

需要如实标注局限：BFTS 与 NLIR 的单个组件（滑动窗口注意力、门控交互）在 LLM 侧都是成熟技术，架构新颖度有限；论文对 SCP 缓解机制只给了经验证据而非形式化证明；小数据集上收益会饱和，窗口大小与 NLIR 超参需按数据集调。

---

## 主线二：统一交互 Block（9 篇）

### 5. RankMixer — Parameter-Free Multi-Head Token Mixing

**论文信息**：arXiv:2507.15551, ByteDance

**做了什么**

RankMixer 提出 parameter-free Multi-Head Token Mixing，用它替代二次复杂度的 self-attention。核心架构：(1) Multi-Head Token Mixing——把每个 Token 的 embedding 切成 H 个 head，再把这些 head 跨 Token 重组成新的混合 Token，论文将该模块整体描述为 splitting / shuffling / projection 且不引入可学习参数；(2) Per-Token FFN（PFFN）——为每个 Token 位置分配独立参数的 FFN，用于建模不同特征子空间并缓解特征空间支配（inter-feature-space domination）问题；注意它与 MLP-Mixer 的 channel-mixing 不同，后者在所有 Token 间共享同一组参数；(3) Sparse-MoE 扩展——用 ReLU routing 加 DTSI（Dense Train, Sparse Infer）动态路由，解决专家训练不足与不均衡，把模型推到 10 亿参数量级。MFU 从 4.5% 提升到 45%。

**为什么这么做**

标准 Attention 的 O(L²) 复杂度是推荐系统 Scaling 的核心瓶颈——序列长度增长时计算量二次爆炸。论文的论证不是"位置和类型决定关系"，而是：推荐特征空间高度异构（用户 ID 与物品 ID 等不在同一语义空间），在异构空间之间算内积相似度本身缺乏意义，还会带来跨空间 ID 相似度建模的组合爆炸；同时 attention 的 score 矩阵是 activation-to-activation GEMM，严重 memory-bound。因此改用无参数的固定 mixing。需要说明的是，"parameter-free mixing 效果不差于 self-attention"在论文中是靠内部消融支撑的领域假设，没有公开数据或理论证明。

**原始论文深度新见解**

- **Parameter-Free Mixing 的数学本质**：mixing 等价于一个固定排列作用于切分后的 head 矩阵——排列模式与输入内容无关，这是它区别于 attention 的根本点。真正的可学习变换由后续 PFFN 承担，因此可以把职责理解为"mixing 负责路由（谁和谁交互）、PFFN 负责变换（交互后如何映射）"。
- **Token 数量 T 的双向权衡**：论文明确讨论了 Token 化的粒度问题——特征域常达数百个，若一特征一 Token，则每个 Token 分到的参数和算力被切得极碎，重要特征建模不足且 GPU 利用率低；若 Token 太少（极端是单个 Token），结构退化成普通 DNN，失去对不同特征子空间的独立表征，还会出现主导特征掩盖其他特征。RankMixer 的做法是按领域知识把特征聚成语义连贯的簇，拼接后再切成固定维度、数量适中的 Token。消融显示：若让所有 PFFN 共享同一份输入向量，效果显著下降——这反证了特征子空间切分与独立建模的必要性。
- **ReLU Routing 配 DTSI 才是完整方案**：vanilla top-k softmax MoE 在高稀疏度下会退化，原因是被路由到的专家训练不足。RankMixer 改用 ReLU router 加 L1 稀疏惩罚——ReLU 输出天然含零，激活专家数由数据自适应决定而非硬性 top-k，并用一个正则项把平均激活比例约束在预算附近。更关键的是配套的 DTSI（Dense Train, Sparse Infer）：训练时走 dense 让所有专家都得到充分梯度，推理时才稀疏激活，用训练成本换专家质量。论文报告该组合在 8 倍稀疏度下仍保持精度、推理吞吐提升约 50%。注意这也正是 TokenMixer-Large 后来要改掉的地方——DTSI 只省推理不省训练。
- **MFU 4.5%→45% 的十倍提升根源**：关键在于算子的算术强度结构变了。attention 中 QKV 投影与输出投影是权重矩阵参与的 GEMM（权重跨样本共享，compute-bound），但 QK^T 与 A·V 是 activation-to-activation GEMM——两个操作数都随样本变化、无法跨 batch 复用，算术强度只有 O(1) 量级，落在 memory-bound 区。RankMixer 去掉这两个算子后，mixing 是纯数据重排、PFFN 是规整大 GEMM，整个拓扑变成 GEMM-first。这也解释了推荐场景为何比 LLM 更吃亏：QK^T 的算术强度随 Token 数 T 增长，而推荐的 T 只有几十，结构性地锁在 memory-bound 一侧。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| MFU | 4.5% → 45% | 短视频推荐 |
| 参数量扩展 | 论文摘要记作 100x，贡献列表记作 70x（含 MFU 提升与量化），均为推理延迟基本不变前提下 | 抖音信息流 |
| A/B 活跃天数 | +0.3% | 短视频推荐 |
| A/B 时长 | +1.08% | 短视频推荐 |

**对 Scaling 的意义**：RankMixer 证明了推荐系统 Token 交互可以完全摆脱注意力机制，O(L) 复杂度 + 45% MFU 使序列长度 Scaling 不再受计算墙约束，参数量 Scaling 不再受效率墙约束。它是统一交互 Block 的奠基之作。

---

### 6. TokenMixer-Large — Mixing-and-Reverting 大规模扩展

**论文信息**：arXiv:2602.06563, ByteDance

**做了什么**

TokenMixer-Large 将 RankMixer 范式扩展到 7B（在线服务）/ 15B（离线训练）参数规模。核心创新：(1) Mixing-and-Reverting 机制解决残差语义错位——mixing 后的 Token 语义空间偏移，直接残差连接会导致语义混叠，Reverting 操作将混合后的 Token 映射回原始语义空间再做残差；(2) 层间残差 + 辅助损失解决深层梯度消失；(3) PerToken SwiGLU 替代标准 FFN；(4) Sparse Per-token MoE 实现"先 Enlarge 后 Sparse"——先扩大 dense 参数到目标规模，再引入稀疏路由保持推理计算量可控（1:2 稀疏比）；(5) Global Token 机制 + 语义 Group-wise Tokenizer。

**为什么这么做**

直接扩大 RankMixer 规模会遇到两个问题：(1) 深层 mixing 累积导致 Token 语义空间漂移，残差连接的"捷径"不再是捷径而是噪声源；(2) 7B+ 参数的 dense 模型推理成本不可接受。Mixing-and-Reverting 解决第一个问题，Sparse MoE 解决第二个问题。"先 Enlarge 后 Sparse"则把 RankMixer 的"Dense Train, Sparse Infer"升级为"Sparse Train, Sparse Infer"，让训练成本也一起降下来。

**原始论文深度新见解**

- **Mixing-and-Reverting 的理论动机**：标准残差 `y = x + f(x)` 假设 f(x) 与 x 在同一语义空间。但 Token Mixing 改变了 Token 的语义表征（如行为 Token 经过 mixing 后混入了 Query Token 的信息），此时 `y = x + f(x)` 中的 x 和 f(x) 语义错位。Reverting 通过逆映射 `f_revert(f_mix(x)) ≈ x` 将混合结果拉回原始空间，使残差连接恢复语义一致性。
- **"先 Enlarge 后 Sparse"的真实动机是降训练成本**：先把 dense 模型扩到目标规模，再把 SwiGLU 拆分为多个专家做稀疏化。关键不在于训练更稳，而在于稀疏度在训练期就已固定，从而实现"Sparse Train, Sparse Infer"。这正是对 RankMixer 的针对性修补——RankMixer 用的是"Dense Train, Sparse Infer"，只省推理不省训练。配套设计包括 Gate-Value Scaling（α 与稀疏度成反比，保证被激活专家获得足够梯度）、Down-Matrix Small Init（stddev 1→0.01，初始化时近似恒等映射）和常驻的 shared expert（去掉它约损失 0.02% AUC）。
- **Global Token 借鉴 BERT [CLS]**：论文把它类比为 BERT 的 [CLS] token——一组可学习的全局上下文 Token，参与每层 mixing 以聚合全局信息，区别在于作用于 Token Mixing 而非 Attention。注意不要与 LONGER 的 global token 混淆：后者是由候选 item、用户画像等基础特征聚合而成、用作 cross attention 的 query 锚点，两者来源与用途都不同。
- **语义 Group-wise Tokenizer**：将异构 Token 按语义分组（如"行为组""特征组""Query 组"），组内先做 mixing，组间通过 Global Token 间接交互。这减少了 mixing 的有效序列长度，降低了计算量。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 电商订单 | +1.66% | 电商推荐 |
| 电商 GMV | +2.98% | 电商推荐 |
| 广告 ADSS | +2.0% | 广告推荐 |
| 直播营收 | +1.4% | 直播推荐 |

**对 Scaling 的意义**：TokenMixer-Large 验证了 RankMixer 范式可扩展到 7B/15B 参数——parameter-free mixing 在超大规模下依然保持高效率，Mixing-and-Reverting 是深层 Scaling 的必要组件。

---

### 7. MixFormer — Co-Scaling Dense and Sequence

**论文信息**：arXiv:2602.14110, ByteDance

**做了什么**

MixFormer 提出 Co-Scaling Up Dense and Sequence，在统一 Transformer 的单一参数空间内联合建模序列行为和特征交互。架构三段式：(1) Query Mixer——论文受 RankMixer 启发，用无参数的 HeadMixing 替代 self-attention（理由是非序列特征来自高度异构的语义空间与超大稀疏 ID 域，用内积相似度算注意力权重本质上不可靠），再对每个 query head 过一个独立的 SwiGLU FFN；(2) Cross Attention——Query Mixer 的 N 个输出头直接作为 N 个 query head（无需额外投影矩阵，因为每个 head 已代表一个特定子空间），序列侧每个行为先经当前层的 SwiGLU FFN 变换再投影成 K/V；(3) Output Fusion——对注意力输出再过 per-head 独立 SwiGLU FFN 做非线性融合。配套 user-item decoupling 策略降低冗余计算，部署于抖音和抖音极速版。

**为什么这么做**

传统推荐将序列建模（如 HSTU/Transformer 行为序列）和特征交互（如 DCN/DeepFM 特征交叉）分离为两个模块，各自独立 Scaling。MixFormer 认为二者应在同一参数空间联合 Scaling——序列行为提供时序信号，特征交互提供上下文信号，两者互补且共享底层表征。Co-Scaling 避免了"序列模块和特征模块各自扩大但交互不足"的问题。

**原始论文深度新见解**

- **Query Token 的切分方式是均匀切分，不按语义分组**：论文先把所有非序列特征各自 embedding 后拼接成 e_ns，再**均匀**划分为 N 个连续子向量，每个投影到 D 维。这与 RankMixer 的 group-wise（按语义聚簇）和 MTGR 的一特征一 Token 都不同——MixFormer 得到的是一组数量固定、维度统一的紧凑特征 Token。注意它不按人口属性、设备信息、场景上下文等语义类别分组。
- **"统一参数化"指的是 FFN 被两类信息复用，不是共享 Embedding**：论文所称 unified parameterization 的实质是 Query Mixer 与 Output Fusion 中的 per-head SwiGLU FFN 同时服务于序列与非序列信息，因此没有哪部分参数被专门划给序列或划给稠密特征——这才是"不必在两者间分配容量"的由来。需要提醒的是，有第三方评审指出该表述有夸大之处：稠密路径与序列路径之间并未真正共享权重（KV 编码用的是每层独立的 SwiGLU FFN），"统一"更多体现在架构组织而非权重共享。引用该卖点时宜留意。
- **User-Item Decoupling 恰恰是靠 masking 实现的**：做法是先把非序列特征的 head 拆成 user 侧与 item 侧两个子集（论文实际设置为 1:1），然后在 HeadMixing 操作中**屏蔽 user head → item head 方向的信息流**，使最终的 user head 不含任何 item 信息，于是它在 RLB（请求级批处理）中可以安全地跨候选 item 复用——这一解耦正是靠 masking 达成的，而非纯架构切分。论文报告 UI-MixFormer 在不同候选集规模下相比原始 MixFormer 有 30%+ 的推理加速。另可对比：OneTrans 只做了序列特征的 KV Caching，未覆盖 user 侧非序列特征的重复计算。
- **动机是 OneTrans 式统一架构的算力代价**：论文把已有工作分为分离式与统一式两类——分离式限制了双向信息流与延迟优化，而统一式的 OneTrans 虽然效果好，但不加改造时计算量过大（尤其 Cross-Attention 部分），难以满足工业级低延迟要求。MixFormer 的定位就是在保持统一架构的同时把算力压回可部署区间，这也是它必须配 user-item decoupling 的原因。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 在线 A/B | 活跃天数与应用内使用时长稳定提升 | 抖音 + 抖音极速版 |
| 推理加速 | UI-MixFormer 相比原始 MixFormer 30%+ | 不同候选集规模 |

> 注：论文摘要与正文说明活跃天数、使用时长等参与度指标获得一致提升，但本文档未收录其具体数值，引用时请核对原文表格。

**对 Scaling 的意义**：MixFormer 证明了序列建模和特征交互可以在单一参数空间联合 Scaling，打破了"序列模块 + 特征模块"分离式架构的 Scaling 边界。

---

### 8. Kunlun — GDPA 重构与 CompSkip 计算跳过

**论文信息**：arXiv:2602.10016, Meta

**做了什么**

Kunlun 提出统一推荐架构设计，包含低层和高层两组优化。低层模块优化（提升硬件利用率）：(1) GDPA（Generalized Dot-Product Attention）把 PFFN 重写为点积注意力形式以便融合成单 kernel；(2) HSP（Hierarchical Seed Pooling）用可学习 seed 加 Kronecker 压缩做序列摘要；(3) 滑动窗口注意力利用推荐序列的近期偏好局部性，把 O(T²) 降为 O(T·w)，QPS +31.1%、FLOPs −29.5%。高层计算重分配：(1) CompSkip 按层选择性跳过组件（FLOPs −43.1%、QPS +35%）；(2) Event-level Personalization 按事件类型重要性差异化分配算力。MFU 从 17% 提升到 37%（NVIDIA B200），缩放效率翻倍，部署于 Meta GEM 广告模型。

**为什么这么做**

论文的动机不是笼统的成本节约，而是一个可分解的问题定式：把 scaling efficiency 拆成算法有效性（每 FLOP 带来多少 NE 收益）与计算效率（MFU）两个相乘的因子，并指出联合建模序列与非序列特征的架构之所以拿不到可预测的幂律曲线，是因为这两个因子同时偏低——推荐模型 MFU 只有 3-15%（LLM 是 40-60%），成因是异构特征空间导致 embedding 维度小、张量形状不规则、算子 memory-bound；同时对所有组件均匀扩容会因各层各事件类型的收益差异而边际递减。低层优化修 MFU，高层重分配修资源配置，两者都做到才可能出现可预测的 scaling law。

**原始论文深度新见解**

- **GDPA 的"广义"是指把 PFFN 改写成点积注意力**：GDPA 不是统一多种注意力变体的配置接口。它做的是把 InterFormer 的 Personalized FeedForward Network 重新表达为一个 cross-attention 算子——序列 token 作 Query，由非序列上下文特征生成的两个权重矩阵作 Key 和 Value，并用 per-head 激活替代 softmax、温度取序列最大长度。这样做的目的是让计算模式与 FlashAttention 兼容，从而能写成单个融合 Triton kernel，把原先大量"小而不规则的矩阵乘"变成规整的融合算子，PFFN 模块 MFU 最高提升 6 倍。附带收益是引入残差后 PFFN 可以稳定堆叠，这是原始形式不具备的。
- **HSP 的两阶段压缩**：seed 不是"挑出来的高重要性特征"，而是一组**可学习的 seed embedding**。第一阶段用这些 seed 对序列做注意力，得到 seed 级表示；第二阶段用 SumKronLinear——一个基于 rank-k Kronecker 积的压缩层——把 seed 进一步压成最终 token。Kronecker 结构的作用是在省参数的同时保留跨维度相关性，比可分离因子分解更强。它替代的是 PMA 里用随机 query 做摘要的做法。"分层"指 seed 生成与 seed 压缩这两级，不是粗细粒度的多级池化。
- **CompSkip 跳的是层内组件，不是按输入稀疏性动态跳过**：它的依据是深层网络中组件存在冗余，采用的是"隔层交替"这种按层选择的模式——例如偶数层跳过 self-attention 但重新计算 HSP 摘要，奇数层跳过 HSP/PFFN 但执行 self-attention。这是层级的结构化配置，与运行时检测输入零值无关，也不是逐样本的动态路由。论文报告 FLOPs 减少 43.1%、QPS 提升 35% 且模型质量不降。
- **Event-level Personalization 是按事件类型分配算力**：它解决的不是"个性化粒度"问题，而是资源分配问题。不同交互类型（点击、购买、曝光）的信号质量差异很大——点击序列通常比曝光序列更有预测力。于是给重要事件类型更大的模型维度、更多注意力头、更多摘要 token 和更大的滑动窗口，次要类型用精简配置。这属于论文两大高层创新中"计算资源重分配"的一半，与 CompSkip 并列。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| MFU | 17% → 37%（NVIDIA B200） | Meta Ads |
| 缩放效率 | 2 倍（相对 SOTA） | Meta Ads |
| Topline 指标 | +1.2% | Meta Ads（GEM 广告模型）|
| QPS（滑动窗口注意力） | +31.1%，FLOPs −29.5% | 消融实验 |
| CompSkip | FLOPs −43.1%，QPS +35% | 消融实验 |
| GDPA | PFFN 模块 MFU 最高提升 6 倍 | 消融实验 |

> 注：MFU 17%→37% 与 2 倍缩放效率出自论文摘要；+1.2% 是论文所述的 topline 指标提升，非 NE 口径；QPS、FLOPs、CompSkip、GDPA 各项来自消融实验，分别对应单个模块的贡献，不可相加。

**对 Scaling 的意义**：Kunlun 证明了低层算子统一（GDPA）+ 高层计算跳过（CompSkip）的组合可以将 MFU 翻倍，这是"效率 Scaling"——在不增加参数量的前提下通过计算效率提升实现有效 Scaling。

---

### 9. ULTRA-HSTU — SLA 半局部注意力与动态拓扑

**论文信息**：arXiv:2602.16986（论文标题 *Bending the Scaling Law Curve in Large-Scale Recommendation Systems*）, Meta

**做了什么**

ULTRA-HSTU 是 HSTU 的下一代设计，思路上受 DeepSeek-V2 启发，走的是模型与系统协同设计。它的立场值得先讲清楚：面对 O(L²) 瓶颈，许多同期工作改用 cross-attention 绕开，论文认为那样会牺牲 self-attention 的表征能力，因此**选择保留 self-attention 而非消除它**，转去解决它的工程可行性。三组创新：(1) 输入序列优化——item 与 action 表征改为相加合并（配异构 action 编码），序列长度减半、attention FLOPs 降 4 倍，并加 candidate masking 防 action 信息泄漏；同时提出 LBSL（Load-Balanced Stochastic Length，负载均衡随机长度采样），在随机长度采样时施加每 rank 算力负载约束以减少 straggler，训练吞吐 +15%。(2) SLA（Semi-Local Attention）——把 causal attention 限制为"最近 K1 个 token 的局部窗口"加"全局 prefix 中的 K2 个 token"，复杂度降为 O((K1+K2)·L)。(3) 动态拓扑——Attention Truncation（前 N1 层跑全序列，之后只保留尾部片段进入后续层）与 MoT（Mixture of Transducers，多路行为通道各自独立的 transducer 栈，输出拼接后送统一 tower）。系统侧配 BF16/FP8/INT4 混合精度与定制 SLA kernel（扩展 FlashAttention V3，适配 SiLU 注意力与非标准 mask，覆盖 H100 与 AMD MI300）。相对原始 HSTU，训练缩放效率 5.3 倍、推理缩放效率 21.4 倍。

**为什么这么做**

论文的核心判断是：self-attention 对于扩大算力与加深架构仍然是更优且必需的，问题只在于 O(L²) 让它在生产环境（亚秒级延迟、每天数十亿次推荐）不可行。这与依赖 cross-attention 降复杂度的路线形成明确分野——后者省了算力但削弱了表征能力。SLA 的设计因此不是"近精确、远摘要"的分层压缩，而是把注意力 mask 结构化为线性稀疏：K1 覆盖近期局部模式，K2 覆盖全局 prefix 中的若干 token 以锚定长期兴趣。论文强调这一点区别于通用稀疏注意力——后者通常只关注局部窗口，而推荐场景同时需要局部行为模式与全局用户偏好。

**原始论文深度新见解**

- **SLA 双窗口不涉及压缩或低秩**：K2 窗口里是**全局 prefix 中真实存在的 token**，不是"每个压缩 Token 代表一段历史摘要"，SLA 也没有做低秩 attention——它做的只是把 mask 从全因果改成"局部窗口 + 全局 prefix"两块，被选中的位置仍是标准 attention 计算。开源实现印证了这一点：effective_k2 = max(sla_k2, contextual_seq_len)，即 contextual prefix 自动成为全局 prefix 的一部分。SLA 单独贡献训练/推理缩放指数各 1.39 倍和 1.69 倍，叠加 Attention Truncation 后推理侧进一步到 1.8 倍。
- **Item + Action 相加的真实收益是序列长度减半**：原始 HSTU 把 item 与 action 表征**交错排列**，这等于把序列长度翻了一倍；ULTRA-HSTU 直接相加 x_{i,j} = I_{i,j} + a_{i,j}，有效序列长度减少 2 倍、attention FLOPs 降约 4 倍，并借异构 action 编码同时容纳隐式与显式信号。附带一个易被忽略的工程细节：因为 action 被并入同一 token，必须加 candidate masking 防止 action 信息泄漏到候选侧。论文未从"行为类型是物品修饰符、相加等价于语义空间偏移"的角度论证，引用时不宜作此解释。
- **LBSL 解决的是分布式 straggler，不是 padding 浪费**：全称 Load-Balanced Stochastic Length（负载均衡随机长度）。背景是同步分布式训练中各 rank 拿到的序列长度不同，长序列 rank 拖慢整个 step——即 straggler 问题。LBSL 的做法是在**随机长度采样阶段**就施加每 rank 的算力负载约束，使各 rank 负载均衡，训练吞吐 +15%。注意它不是"按长度分桶、每桶 padding 到桶长、不同桶分配不同 GPU"那类做法。
- **"Bending the curve"指整条缩放曲线变陡，与语义特征无关**：论文标题里的 bending 指的是——通过效率优化，让模型质量随算力投入的提升速度显著加快，即缩放曲线被"掰弯"向上。论文将 scaling efficiency 形式化定义为"模型性能与算力成本拟合直线的斜率"，并报告训练缩放指数提升 2.08 倍、推理缩放指数提升 4.59 倍。它没有提出"语义特征超过某序列长度阈值后边际收益下降、ID 特征继续上升"这一发现。附带说明：LLaTTE 也用 bend the curve 一词且同样指曲线变陡，两者方向一致，但自变量与归因不同（此处归因于架构与系统协同优化，LLaTTE 归因于语义特征），不宜当作同一发现互相印证。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 训练缩放效率 | 5.3 倍（缩放指数 2.08 倍） | vs 原始 HSTU |
| 推理缩放效率 | 21.4 倍（缩放指数 4.59 倍） | vs 原始 HSTU |
| 消费与互动指标 | +4%~8% | 生产环境，服务数十亿用户 |
| Topline 指标 | +0.217%（Online 1）、+0.037%（Online 2） | 30 天在线 A/B |
| 训练吞吐 | +15%（LBSL）、+70%（全套系统优化） | 内部基准 |
| 推理吞吐 | +50%（全套系统优化） | 内部基准 |
| 部署规模 | 18 层 self-attention × 16k 序列，数百张 H100 | 生产环境 |

> 注：5.3 倍与 21.4 倍是**缩放效率**（性能-算力拟合斜率之比），不是端到端加速比，引用时不宜简化为"快 21 倍"。

**对 Scaling 的意义**：ULTRA-HSTU 给出的是一条与"放弃 self-attention"相反的路线——通过 SLA 把复杂度降到线性、再叠加动态拓扑与混合精度，让 self-attention 在 16k 序列、18 层、数百张 H100 的生产配置下可行。它对 Scaling 讨论的独特贡献在于把 scaling efficiency 定义为可测量的拟合斜率，从而使"架构改进是否真的改善了缩放"变成一个可验证的命题，而非只看单点指标。

---

### 10. HyFormer — 长序列与特征交互的交替优化

**论文信息**：arXiv:2601.12681（v1 2026-01-19，v2 2026-01-23）, ByteDance（AML + Search）

**做了什么**

HyFormer 针对的是当时已成主流的"两阶段"范式——先用 LONGER 这类 query-token 序列压缩器把长行为序列压好，再用 RankMixer 这类 token-mixing 模块与稠密特征融合。论文认为这种流水线强制了一种**压缩的、后融合的、单向的**交互模式：序列被过早压缩，后续的特征交互无法反过来影响序列建模。HyFormer 把两件事塞进同一个骨干，构造成**交替优化**过程，每层由两个模块串起来并逐层堆叠：

- **Query Generation**：把所有非序列特征向量与一个序列级摘要（对行为序列表征做 pooling 得到）拼成 "Global Info"，再经多个轻量 FFN 投影出一组 query token。深层复用上一层的 query，使其带着越来越丰富的语义去追问长序列。
- **Query Decoding**：用这些 query token 通过 **cross-attention** 去读取长序列的逐层 key-value 表征，抽取 target-aware 信息。序列侧的编码方式是可插拔的三档——Full Transformer（容量最高）、LONGER 式高效编码（把复杂度从 O(L²) 降到 O(L·L')）、以及 decoder 式无注意力的轻量前向（给延迟敏感场景）。
- **Query Boosting**：把解码后的 query 与非序列特征嵌入拼接，做 MLP-Mixer 式的 token mixing——每个 query token 切成若干 channel 子空间，同一子空间跨所有 token 位置聚合信息，再用逐 token 的轻量 FFN 精炼，并加残差连接稳定优化。
- **多序列独立建模**：面对视频观看、商品购买等多条异构序列时，每个 block 内对每条序列用一组专属 query token 独立处理，跨序列交互在 query 层面的 token mixing 中完成，避免简单合并序列带来的退化。

**为什么这么做**

论文列了两条具体动机。一是现有序列 transformer 的 query 表示过于简化——用来聚合长序列的 query token 通常只来自候选相关或全局特征的一个小子集，能利用的上下文有限；而直接增加 query 数量，会在 KV-Cache 与 M-Falcon 这类服务机制下显著拖垮效率。二是序列压缩后的 token 与异构非序列 token 的交互只发生在模型很靠后的位置，跨特征推理被推迟到压缩之后，信息已经损失。交替优化就是对这两点的直接回应：query 侧信息尽量丰富，且让交互与压缩互相迭代而非单向排队。

**原始论文深度新见解**

- **它的时间位置和批评对象必须说清楚，否则会读反**：HyFormer 是 2026 年 1 月的工作，**晚于** LONGER（2505）与 RankMixer（2507），且摘要开篇就把 "LONGER 压缩 + RankMixer 融合" 点名为它要改的既有范式。所以 HyFormer 不是这条技术线的起点或分叉点，而是**在两阶段范式跑成主流之后提出的合并方案**。若把它当作"分叉前的关键节点、同时启发了 RankMixer 与 ULTRA-HSTU"，引用方向就反了。
- **真正的发明点是"交替"而不是"统一"**：把序列建模与特征交互放进一个 backbone，OneTrans、MTGR 都做过。HyFormer 的差别在于它不是一次性把所有 token 铺平做 self-attention，而是让 query 反复往返——decode 一次（从序列取信息）、boost 一次（在 query 之间和非序列特征之间混合），下一层的 query 因此比上一层"问得更准"。论文自己的表述是 alternating optimization，双向信息流由此产生。
- **Global Token 与 LONGER 同名但职责不同**：LONGER 的 global token 是注意力的 query 锚点，用于稳定长上下文；HyFormer 的 Global Tokens 是**由非序列特征扩展而来的共享语义接口**，承担的是"让非序列信息去解码序列"的桥梁角色。两篇都是字节的工作、用词相同，容易混读。
- **序列编码器被降级为可替换组件，这是个架构态度**：HyFormer 明确支持 Full Transformer / LONGER 式 / 无注意力轻量三种序列编码，意味着论文把"长序列怎么编码"当作正交的成本旋钮，而把自己的贡献锚定在 query 侧的交替机制上。这与主线三那些以序列编码本身为主体的工作形成分工。
- **消融数据支持"query 侧信息量是关键"这个主张**：把 query 限制为仅目标特征时 AUC 掉 0.34%，去掉跨序列 pooling token 掉 0.20%，多序列独立建模相比合并序列涨 0.27%。三个数字都指向同一件事——query 携带的上下文越丰富，长序列解码越有效，这正是论文批评既有做法"query 过于简化"的实证支撑。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| Query-level AUC | 0.6489（相对 LONGER+RankMixer 基线 0.6478 提升 0.74% 相对值） | 抖音搜索，十亿级工业数据集 |
| FLOPs | 3.9×10¹²（低于多数对比方案；基线 LONGER+RankMixer 为 3.5×10¹²） | 同参数/算力预算对比 |
| 消融：query 仅用目标特征 | AUC −0.34% | 离线 |
| 消融：去掉跨序列 pooling token | AUC −0.20% | 离线 |
| 消融：多序列独立建模 vs 合并 | AUC +0.27% | 离线 |
| Scaling 行为 | 随参数与 FLOPs 增长表现优于基线 | 离线 |
| 在线 A/B | 论文表述为"significant gains over deployed SOTA"，未给出具体百分比 | 高流量生产系统 |

> 注：论文对比表中 MTGR/OneTrans 这类统一 Block 方案表现反而不如两阶段基线（0.6480 vs 0.6478，且 FLOPs 6.6 明显更高），论文归因于它们在特征交互侧依赖效率较低的 self-attention 或对多序列的合并方式不佳。在线 A/B 只给了定性表述，引用时不宜编造数值。

**对 Scaling 的意义**：HyFormer 的贡献是把"长序列压缩"与"特征交互"这两条本来分属主线三和主线二的工作重新缝在一起，并给出一个具体的缝法——交替而非并列。它的 Scaling 曲线优于 LONGER+RankMixer 组合，说明在同等参数与算力预算下，**信息流的拓扑（能否双向、能否迭代）本身就是一个可以换取效果的设计维度**，而不只是算子效率和参数量的问题。它也提示统一 Block 不等于自动更好——同为统一方案的 MTGR/OneTrans 在这份对比里并没有跑赢两阶段基线。

---

### 11. EST — 高效可扩展统一建模

**论文信息**：arXiv:2602.10811（论文标题 *EST: Towards Efficient Scaling Laws in Click-Through Rate Prediction via Unified Modeling*）, Alibaba 淘天集团

**做了什么**

EST（Efficiently Scalable Transformer）的目标是**完全统一建模**——把所有原始输入放进单一序列处理，不做任何有损聚合。论文的靶子很明确：工业界为了效率普遍采用早期聚合（Early Aggregation）或部分统一建模，即先用 DIN 之类模块把长行为序列压成定长向量再与其他特征交互。论文指出这种有损聚合丢弃了 token 级细粒度信号，形成信息瓶颈，直接限制了靠增大参数规模换性能的空间。EST 由此从 CTR 与 LLM 的两点本质差异出发：行为特征与非行为特征之间的**信息密度不对称**，以及内容型信号的**模态特定先验**。两大组件：(1) LCA（Lightweight Cross-Attention）——剪掉冗余的自交互，把算力集中到高影响的跨特征依赖上；(2) CSA（Content Sparse Attention）——利用内容相似度动态挑选高信号行为。

**为什么这么做**

标准 Transformer 对所有 Token 一视同仁做 self-attention，但推荐场景中：(1) 非行为特征（画像、上下文）数量多但信息密度低，互相做 self-attention 是冗余计算；(2) 行为序列中大量低信号行为（快速划过、误触）浪费注意力容量。EST 通过"非行为特征用 LCA 轻量交互、行为特征用 CSA 稀疏筛选"实现计算资源的精准分配。

**原始论文深度新见解**

- **LCA 的依据是信息密度不对称**：论文的观察是少量非行为特征（用户画像、候选广告信息）信息密度极高，而海量行为序列充满低价值噪声——在这种不对称结构下做全量 self-attention 是极度冗余的。LCA 因此剪除冗余的自交互，只保留高影响的跨特征依赖。论文并未以"传统 cross-attention 以行为为 Query、EST 反转为以特征为 Query"这种对比方式论证，引用时不宜套用。
- **CSA 的内容相似性筛选**：CSA 不是简单地按行为时间衰减加权，而是计算行为的内容表示与当前候选物品的内容相似性，只对高相似性行为做精确 attention。低相似性行为被聚合为"背景行为 Token"参与计算但不占用精确注意力容量。这等价于"内容相关的 top-k attention"。
- **信息密度不对称**：论文的核心先验是行为序列的信息密度高度不均——少数高信号行为承载了大部分有效注意力，CSA 据此把注意力预算集中到高信息密度行为上，实现"计算资源精准分配"。注意论文未给出"前 20% 贡献 80%+"这类实测比例，引用时请以原文表述为准。
- **模态特定先验指的是行为不止是离散 ID**：论文的第二个观察是用户行为还携带丰富的多模态内容（如图文），这类内容型信号具有可利用的模态先验——CSA 正是据此用内容相似度而非单纯的时序或 ID 匹配来筛选高信号行为。可核实材料中没有"ID 特征用低秩 attention、文本特征用标准 attention"这类具体参数化方案。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| RPM（Revenue Per Mille，千次展示收入） | +3.27% | 淘宝展示广告 |
| CTR | +1.22% | 淘宝展示广告 |

**对 Scaling 的意义**：EST 的核心主张是"有损聚合本身就是 Scaling 的天花板"——只要还在用早期聚合压缩行为序列，token 级信号的丢失就会让增大参数规模换不来相应收益。它给出的解法是先做到完全统一建模，再用 LCA 与 CSA 把算力按信息密度重新分配回来。论文报告 EST 呈现稳定的幂律缩放关系，使性能随规模可预测提升——这一点比单纯的效率提升更关键，因为可预测性才是投资决策的前提。

---

### 12. HeMix — 异构混合与 Query-Mixed 兴趣提取

**论文信息**：arXiv:2602.09387, Alibaba AMAP（论文标题随版本变化：v1/v2 为 *Query-Mixed Interest Extraction and Heterogeneous Interaction*，v3 改为 *HeMix: Scaling Industrial Ranking Models with Heterogeneous Token Mixing*）

**做了什么**

HeMix 针对两个挑战：(C1) 已有序列 Token 化无法同时刻画上下文感知与上下文无关的用户意图；(C2) 主流交互机制既昂贵又语义同质，在严格在线延迟约束下限制预测质量。对应两大组件：(1) Query-Mixed Interest Extraction——用动态 Query 与固定 Query 分别在**全局行为序列**和**实时行为序列**上提取上下文感知与上下文无关兴趣；(2) HeteroMixer Block 替代 self-attention，三阶段流水线：Multi-Head Token Fusion（按语义分组拼接成 mix-token）→ Heterogeneous Mixed-Token Interaction（每个 mix-token 各自过专用 MLP，配低秩分解）→ Group-Aligned Reconstruction（重塑回原 Token 空间）。参数量从约 100M 平滑扩展到约 1500M，做法是**独立扩展块深度与 Token 维度**，无需重新设计架构。

**为什么这么做**

传统推荐行为序列建模要么只做上下文感知建模（如 target attention，兴趣随候选物品变化），要么只做上下文无关建模（如 self-attention，提取静态兴趣）。两者各有优劣但通常分离实现。HeMix 认为用户兴趣是"静态偏好 + 动态调整"的叠加，应在同一框架内联合建模。HeteroMixer 则解决异构 Token（行为、特征、Query）在同一空间混合时的语义冲突问题。

**原始论文深度新见解**

- **动态 Query + 固定 Query 分工，且作用于两类序列**：动态 Query 随请求上下文变化，提取上下文感知兴趣；固定 Query 不随请求变化，提取上下文无关兴趣。关键是这两类 Query 同时作用于**全局（长期）序列**与**实时序列**两个来源。论文消融给出两条明确结论：去掉固定 Query 会显著掉点，说明上下文无关兴趣不可省；全局与实时序列的联合建模同样关键，其中实时序列的影响尤其大。论文未把这组分工类比为"positional encoding 与 content encoding 的互补"。
- **HeteroMixer 三阶段的信息流**：阶段一多头 Token 融合将异构 Token 映射到统一表示空间；阶段二异构混合 Token 交互让不同类型 Token 交换信息（行为 ↔ 特征 ↔ Query）；阶段三组对齐重构将混合后的 Token 重新分组对齐到原始语义组，保持输出可解释性。三阶段确保"混合不混乱"。
- **低秩异构交互的具体形式**：每个 mix-token 的变换写作 Ĝ_m = σ(G_m·W_L)·W_R，其中 W_L ∈ R^{d_h×d_r}、W_R ∈ R^{d_r×d_h} 且 d_r << d_h——即先降维再升维的低秩瓶颈。复杂度从标准注意力的 O(N²·d_T) 降为 O(N·d_T·d_r)。HeteroMixing 引入的额外参数量约为 2L·N·d_T·d_r，占比很小，这是它可平滑扩参的原因。
- **100M → 1.5B 的 Scaling 验证有具体数字**：在 AMAP 超过 44 亿条记录的数据集上，HeMix-Small（约 100M）相对 DLRM 基线 CTR-AUC +1.64%、CVR-AUC +1.63%；HeMix-Large（约 1.5B）提升到 CTR-AUC +2.34%、CVR-AUC +2.38%。即 15 倍扩参带来约 0.7 个百分点的额外 AUC 增益——收益持续但明显递减，这一点比"效果持续提升"的笼统说法更有参考价值。另需注意 HeMix-Small 在优于最强竞品的同时 GFLOPs 更低。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| CTR-AUC（约100M） | +1.64%（vs DLRM，且 GFLOPs 低于最强竞品） | 离线，44 亿条记录 |
| CTR-AUC（约1.5B） | +2.34%（CVR-AUC +2.38%） | 离线 |
| GMV | +3.61% | 高德 APP 在线 A/B |
| PV_CTR | +2.78% | 高德 APP 在线 A/B |
| UV_CVR | +2.12% | 高德 APP 在线 A/B |

> 注：该论文在线数值随版本变动较大——v1 记作 +0.61% GMV / +2.32% PV_CTR / +0.81% UV_CVR，v2 记作 +3.61% / +2.78% / +2.12%，v3 又记作 +0.88% / +2.74% / +0.84%。本文档采用 v2 口径，引用前务必核对目标版本。

**对 Scaling 的意义**：HeMix 的可扩展性来自一个工程上很实用的性质——扩容只需独立调整块深度与 Token 维度两个旋钮，不必重新设计架构。低秩瓶颈使额外参数量随规模增长得很慢，因此 100M→1.5B 的扩展能在延迟约束内落地。但它的收益曲线也提示了现实边界：15 倍参数换来约 0.7 个百分点 AUC，递减明显。

---

### 13. SORT — 请求中心范式与生成式预训练

**论文信息**：arXiv:2603.03988, Alibaba International / AliExpress

**做了什么**

SORT（Systematically Optimized Ranking Transformer）针对的是两个核心障碍：**高特征稀疏性**与**低标签密度**。后者是关键——LLM 里每个 token 位置都能通过 next-token prediction 提供监督，而排序模型的稀疏二值标签只挂在目标 item 上，海量用户历史序列没有直接标签，这种信息不对称让十亿级参数空间极易过拟合。四项主要优化：(1) Request-centric sample organization——同一请求下的多个候选合并处理，共享用户画像与历史序列，消除请求无关特征的重复计算；(2) Local attention——历史行为序列的 token 只在局部窗口内做注意力，复杂度从 O(L²) 降为 O(L)，论文实验中窗口 256 表现最佳，甚至略优于标准因果注意力；(3) Query pruning——在较高层渐进剪掉距候选 token 较远的 query，算力近乎减半且效果反而变好，论文推测这与用户偏好的时间衰减先验吻合；(4) Generative pre-training——用生成式（next-item prediction）任务补充标签密度以缓解过拟合。模块改进包括 tokenization 引入特殊 token 作特征边界、MHA 加 QKNorm 与 attention gate 稳定训练、FFN 替换为 MoE 层以在不增算力的前提下扩容量。

**为什么这么做**

论文把问题拆成两层。计算层面：传统 impression-centric 采样下每个候选独立成样本，请求无关特征（用户画像、历史序列）被反复计算；长序列的 O(L²) 注意力还有大量算力花在与标签无关的交互上。Request-centric、local attention、query pruning 三者共同把算力重新分配到与标签相关的交互上。信息层面：稀疏二值标签无法正则化十亿级参数空间，这是过拟合的根因，也是"模型越大反而越差"的来源。生成式预训练通过 next-item prediction 提高标签密度来解决它——注意这与"减少有标注数据依赖"不是一回事，论文的目标是补充监督信号密度，而非省标注。

**原始论文深度新见解**

- **Request-centric 是把冗余计算摊薄，而非改变复杂度阶数**：N 个候选原本各自重算一遍用户画像与历史序列编码，合并后这部分只算一次。省下的是常数倍的重复量（用户侧从 N 次降到 1 次），候选侧仍是 O(N)。论文指出 HSTU 已用"单样本内处理多候选"摊薄训练成本，MTGR、GenRank 与本文都沿用了这一范式，因此这不是 SORT 独有的创新点。
- **Local attention 作用于历史序列而非候选之间**：局部窗口限制的是**历史行为 token** 的注意力范围（最优窗口 256），不是"让相邻候选交换信息学习相对优势"。论文也没有把 label 描述为相对排序。一个值得注意的实验结果是：窗口 256 的局部注意力略优于标准因果注意力——即限制注意力范围不仅省算力，还带来了有益的归纳偏置。
- **生成式预训练沿用 GPSD 而非 masked prediction**：预训练任务是 **next-item prediction**（不是 masked behavior prediction），在用户点击序列上单独训练一个模型，再把学到的 item embedding 迁移到排序模型并**冻结**。这套做法出自同一批作者的 GPSD 工作，其动机是稀疏 item ID 特征会阻碍模型扩展性；冻结稀疏 embedding 能显著缩小训练与测试的泛化差距。GPSD 报告 dense 参数从 13K 扩到 0.3B 的过程中性能持续提升并贴合幂律。
- **MoE 用的是数据驱动路由，与 OneTrans 的规则路由形成对照**：论文明确把 OneTrans 的独立参数建模视为"基于规则的路由 MoE"，而 SORT 刻意避开规则路由、改用数据驱动路由。这是两条路线的分歧点，也是理解 SORT 定位的关键：论文自述其相对 DLRM 更靠近 LLM 一端，主动放弃领域专用交互模块以换取更通用的排序范式。论文中没有"按点击/停留/购买等行为类型细粒度分离专家"的设计。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| MFU | 22%（v1/v2）→ 45%（v3）；论文未标注 GPU 型号 | 训练系统 |
| 线上 orders | +6.35%（v1/v2）→ +7.47%（v3） | AliExpress |
| 线上 buyers | +5.97%（v1/v2）→ +6.67%（v3） | AliExpress |
| 线上 GMV | +5.47%（v1/v2）→ +8.65%（v3） | AliExpress |
| 延迟 | −44.67%（v1/v2）→ −62%（v3） | AliExpress |
| 吞吐 | +121.33%（v1/v2）→ +589%（v3） | AliExpress |
| 部署范围 | 全量部署，服务 AliExpress 所有用户 | 生产环境 |

> 注：该论文 v3（2026-08-31）对几乎所有指标做了大幅上修，MFU 从 22% 改为 45%、吞吐从 +121% 改为 +589%。本文档保留两版数值以便追溯，引用时必须指明版本。另：公开摘要与可核实材料中均无"CTR-AUC +0.51pt"与"FLOPs 188G vs 322G"这两项数据。

**对 Scaling 的意义**：SORT 最值得记的一点是它把"低标签密度"确立为排序模型 Scaling 的独立瓶颈——不是算力不够也不是架构不行，而是稀疏二值标签撑不住十亿级参数空间，导致模型越大越差。生成式预训练之所以关键，是因为它提高的是监督信号密度而非算力效率。论文同时在数据量、模型规模、序列长度三个维度上验证了可扩展性，这种多维验证比单维曲线更有说服力。

---

## 主线三：长序列（3 篇）

### 14. Make It Long — 端到端 10K 长序列直接处理

**论文信息**：arXiv:2511.06077, ByteDance

**做了什么**

Make It Long 选择直接在线端到端处理 10K 长序列，而非拆分为两阶段异步架构。三大创新：(1) STCA（Single-Token Cross-Attention）——用 Cross-Attention 替代 Self-Attention，候选作 Query、历史行为作 Key/Value，复杂度从序列内部全对全的 O(L²) 降为候选与序列之间的 O(M×L)（M 为候选数，L 为序列长度，M≪L）；(2) RLB（Request-Level Batch）——同一请求内 B 个候选共享 User 编码，聚合成 batch 统一前向，避免对同一用户重复计算 B 次；(3) 长度外推训练——短窗口训练、长窗口直接推理，无需在训练时就使用全长度序列。

**为什么这么做**

相比拆分两阶段架构（如 LLaTTE 的 upstream/online 异步）引入的额外复杂度和信息时延，端到端路径可获得更实时精确的建模效果。STCA 的优势来自"候选数远小于序列长度"这一工业场景特性；RLB 的优势来自"同请求内 User 不变"这一业务事实；长度外推训练能生效的前提是 STCA 和 RLB 已解决底层算力问题。

**原始论文深度新见解**

- **STCA 的复杂度优势根因**：传统 Self-Attention 对 L 个 Token 做全对全交互是 O(L²)，但推荐场景中真正需要的是"B 个候选 × L 长序列"的交互，不是"L × L"的序列内部交互。STCA 把问题从"序列自建模"重构为"候选查询序列"：原本每个候选都要付 O(L²) 的序列自注意力，改为候选作 query 只需 O(L) 地读一遍序列，B 个候选合计 O(B·L)。这不是近似，而是问题定义的降维。
- **"同请求内共享用户侧计算"被多篇工作独立采用**：同一思路在本文档收录的其他论文中反复出现——Compute Only Once 的 UG-Sep 做用户侧与商品侧解耦复用，LONGER 的 KV cache serving 把用户序列 KV 跨候选复用，SORT 则把训练侧的计算粒度从候选改为请求。四者机制与所处环节并不相同（推理缓存 / KV 复用 / 训练粒度），共同点是都利用了"同一请求内用户侧不变"这一业务事实。需要说明的是，这些论文之间未必存在引用关系，"独立收敛"是本文档的归纳。
- **长度外推训练的生效条件**：短窗口训练、长窗口推理之所以可行，前提是 STCA 已将复杂度从 O(L²) 降为 O(B·L)——线性复杂度下，推理时序列变长只是线性增加计算量，不会引发 O(L²) 式的爆炸。如果用 Self-Attention，长度外推会因为 O(L²) 而不可行。换句话说，长度外推在这里能成立，很大程度上依赖 STCA 已经把复杂度压到线性——这是本文档对论文三个创新点之间关系的解读，论文本身未把两者表述为严格的依赖关系。
- **与 LLaTTE 的路线分歧**：LLaTTE 选择两阶段异步架构（upstream 异步计算 + online 实时消费），Make It Long 选择端到端直接处理。两条路线的权衡点是"信息时延 vs 算力解耦"——LLaTTE 用时间换算力（异步有延迟但算力解耦），Make It Long 用算力换实时性（端到端无延迟但需要在线算力支撑 10K 序列）。这两条路线不宜直接比优劣：LLaTTE 面向 Meta 广告排序、序列可达 5000 且上游算力是在线的 45 倍以上，Make It Long 面向字节场景，两者的延迟预算与算力结构都不同。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 序列长度 | 10K | 字节推荐 |
| 端到端延迟 | 可控（STCA+RLB 降复杂度） | 在线服务 |

> 注：论文未公布具体业务效果指标。

**对 Scaling 的意义**：Make It Long 证明端到端 10K 长序列可行，STCA+RLB 的组合使长序列 Scaling 不必依赖两阶段架构——在线算力足够时可选择端到端路径获得更实时精确的建模效果。RLB 的跨论文收敛验证说明该洞察是推荐场景的结构性优势。

---

### 15. LONGER — Global Token + Token Merge 长序列优化

**论文信息**：arXiv:2505.04421, ByteDance, RecSys'25

**做了什么**

LONGER 提出长序列优化 Transformer，核心机制：(1) Global token 机制——把候选 item、用户画像、UID、高阶交叉特征等基础信息聚合成少量 Token，作为注意力的 query 锚点稳定长上下文下的注意力（论文提到可缓解 attention sink）；(2) Token merge——把序列中**相邻**的 K 个 Token 分组，组内用轻量 InnerTransformer 压成一个 Token，序列长度按因子 K 缩短；(3) 混合注意力——第一层用 cross causal attention（query = global token + 最近 k 个 Token）去读取压缩后的全序列，之后叠加多层 self causal attention；(4) KV cache serving——用户侧序列的 KV 预计算一次、跨候选复用；(5) 混合精度训练与激活重计算，以及稠密/稀疏参数统一在 GPU 上更新的全同步训练服务框架。已在字节超过 10 个场景全量部署。

**为什么这么做**

长序列（100k+ Token）的完整 self-attention 计算不可承受，而完全局部注意力丢失全局信息。LONGER 的设计哲学是"保留全局信息但不全量计算全局注意力"——global token 以 O(1) 额外 Token 携带全局上下文，token merge 以信息无损方式压缩序列长度，混合注意力按层深度分配注意力范围。

**原始论文深度新见解**

- **Global Token 的作用是稳定注意力，不是充当容量瓶颈**：论文对 global token 的定位是"锚点"——它们由候选 item、用户画像等基础特征聚合而成，带有全局感受野，用作 cross attention 的 query，从而缓解长上下文下的注意力不稳定（attention sink）。相比 SIM 只用目标 item 作 query，global token 让 query 侧信息更丰富。论文未给出"8~16 个 global token 为最优区间"这类结论。
- **Token Merge 按位置分组，不按语义相似度筛选**：论文的做法是把序列按固定因子 K 切成相邻组，组内跑一个轻量 InnerTransformer 再压成单个 Token——分组依据是相邻位置，不存在"语义相似度超过阈值才合并"的判据。InnerTransformer 的作用是比简单池化更好地保留组内局部语义。效率上，论文给出 L=2048、d=32、K=4 时 FLOPs 减少 42.8%，注意力代价由 O(L²) 降到 O((L/K)²)。
- **混合注意力的实际拓扑是"先 cross 后 self"**：论文的结构是 1 层 cross causal attention + N 层 self causal attention——第一层用少量 query（global token 加最近 k 个 Token）跨注意到压缩后的完整长序列，把长序列信息先吸收进少数 Token，后续多层再在这些 Token 内部做 self-attention 提取高阶交叉。这不是"浅层 local、深层 global"的窗口分配。消融显示用"最近 k 个"作 query 优于可学习 query 或均匀采样 query。
- **KV cache serving 才是推理侧的关键**：论文让用户序列的 key/value 预计算一次并在同一请求的多个候选间复用，把扩长序列带来的在线吞吐下降从 −40% 收窄到 −6.8%。论文中没有"训练用 100k、推理用 10k 的长度解耦"这一说法——它的量级是端到端建模到 10,000。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 离线 AUC | 0.85290（相对 base +1.57%，相对最强 Transformer 基线 +0.21%） | 52 亿样本 CVR 任务 |
| 在线 A/B（广告） | 广告主指标 +1.06%~2.10% | 字节广告 |
| 在线 A/B（电商） | 人均订单与 GMV +4.6%~7.9% | 字节电商 |
| FLOPs | −42.8%（L=2048, d=32, K=4） | 相对 vanilla Transformer |
| 在线吞吐退化 | 从 −40% 收窄到 −6.8%（KV cache serving） | 在线服务 |
| 部署规模 | 字节 10+ 场景全量部署，服务数十亿用户 | 广告 / 电商 |

> 注：论文平台口径下 AUC 提升 0.1% 即视为显著；但相对最强 Transformer 基线的 0.0018 AUC 差距是否超出随机种子间波动，论文未给出多次运行的方差，引用时宜留意。

**对 Scaling 的意义**：LONGER 的 token merge + 混合注意力把端到端序列建模推到 10,000 量级，KV cache serving 则让扩长序列的在线吞吐代价从 −40% 收窄到 −6.8%——真正解除长序列 Scaling 的推理侧约束的是缓存复用，而不是压缩本身。

---

### 16. LLaTTE — 推荐系统的 Scaling Law

**论文信息**：arXiv:2601.20083

**做了什么**

LLaTTE 系统验证了推荐系统的 Scaling Law，发现序列建模的质量改善随算力投入呈可预测的**对数线性**关系（ΔNE ∝ −α·log₁₀C），据此可对不同扩展维度排定投资优先级。注意这与 LLM 侧常说的幂律形式不同，下文 α 即该对数线性拟合的斜率。核心发现：(1) 语义特征（内容 embedding）是 Scaling 的前提而非增量——它把缩放曲线"掰弯"得更陡（bend the curve），只用 ID 特征时深度扩展很快饱和；另有一个独立发现是宽度存在阈值，d 低于约 256 时加深度收益有限；(2) 序列长度是"最陡峭"的 Scaling 维度（下游模型对总 FLOPs 的斜率 α=0.265，高于深度 0.200 和宽度 0.133）；(3) 两阶段架构——异步 upstream 用户模型处理长历史（序列侧 FLOPs 为在线模型的 45 倍以上）并把用户表征缓存为固定长度向量（生产中 d=2048），在线 ranker 读取该缓存；(4) 转移率 τ≈50%——upstream 的 NE 改善约有一半能传导到下游排序。

**为什么这么做**

LLM 领域的 Scaling Law（Kaplan et al. 2020）揭示了"模型大小、数据量、计算量"之间的幂律关系，指导了 LLM 的 Scaling 投资决策。推荐系统缺乏类似的理论指导——扩大模型是否有效？哪个维度（宽度、深度、序列长度、数据量）的 Scaling 收益最大？LLaTTE 填补了这一空白。

**原始论文深度新见解**

- **缩放斜率 α=0.265 的含义**：论文拟合的是对数线性形式 ΔNE(C) ∝ −α·log₁₀C，即归一化熵的改善量与计算量的对数成正比，α 是该直线的斜率（每 10 倍算力带来的 NE 改善），而非"损失 ∝ 序列长度的幂次"。按总 FLOPs 计，下游模型三个维度的斜率为序列长度 0.265 > 深度 0.200 > 宽度 0.133；按序列侧 FLOPs 计则为 0.238 / 0.106 / 0.091。序列长度在两种口径下都最陡，这是"序列长度优先"结论的实际来源。
- **宽度是深度的前置条件，不是收益衰减点**：论文的发现是 d 低于约 256 时模型受表征容量制约，此时加深度收益很小；一旦宽度跨过该阈值，深度扩展才开始有效。反过来，宽度 1024 但很浅的配置消耗更多序列 FLOPs 却拿到更小的 NE 改善。所以宽度的角色是"解锁其他维度"，而"弯曲"（bend the curve）说的是另一件事——语义特征让所有维度的曲线整体变陡。
- **两阶段的转移率 τ≈50%**：upstream 的 NE 改善约一半传导到下游。损失主要来自固定带宽的信息瓶颈——upstream 必须把数千个事件压缩进单个 2048 维向量，且缺少候选上下文，只能产出通用表征。值得注意的是论文把 50% 视为偏高的好结果（同类多阶段系统典型值为 25-30%），并非退化信号。论文还发现在等 FLOPs 下把算力分配给序列还是深度（Seq-heavy vs Model-heavy）对 τ 影响不显著。
- **"最陡峭驱动力"的实践含义**：序列长度的 Scaling 指数最陡峭，意味着在固定计算预算下，投资序列长度扩展的 ROI 最高。这解释了 LONGER、ULTRA-HSTU 等长序列论文的工业价值——它们直接攻击最陡峭的 Scaling 维度。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| CVR | +4.3% | Meta 广告（Facebook Feed 与 Reels）|
| NE（归一化熵） | -0.25% | Meta 旗舰广告排序模型 |
| 转移率 | ≈50% | 上下游对照实验 |

**对 Scaling 的意义**：LLaTTE 为推荐系统 Scaling 提供了可操作的投资优先级——序列长度是最陡峭的驱动维度，语义特征是让曲线整体变陡的前提条件（而非某个维度上的收益拐点），宽度需先跨过约 256 的阈值才能解锁深度收益，两阶段架构的转移率约 50%。需注意 d≈256 是宽度阈值，与语义特征的"弯曲"是两个独立发现，不应混为一谈。

---

## 主线四：推理解耦（2 篇）

### 17. Compute Only Once — UG-Sep 用户侧计算复用

**论文信息**：arXiv:2602.10455, ByteDance

**做了什么**

Compute Only Once 提出 UG-Sep（User-Group Separation），首次在 TokenMixer 类 dense 交互模型中实现用户侧计算复用。四层设计：

(1) **Token 分离**——底层特征抽取拆成两支，U 侧产出 U-token、G 侧（候选/混合特征）产出 G-token，保证进入交互模块的输入已严格分区。
(2) **交互分离**——在 Multi-Head Token Mixing 的 Mixup 之后施加二值 mask，令前 c_u 个输出 token 只含 U 侧信息；随后把 PertokenAFFN 拆成 U-token 走的 Reusable PertokenAFFN 与 G-token 走的 Non-Reusable PertokenAFFN，前者每用户每请求只算一次，复杂度从 O(C)（C 为候选数）降到 O(1)。
(3) **分离式残差**——金字塔结构中各层 U/G token 数会变，直接残差会重新把两侧缠回去，故改用 cross-attention 形式的残差（Mixup+FFN 输出作 query、输入 token 作 key/value），并在其中再加一道 UG mask 阻断 G→U。
(4) **Information Compensation + W8A16**——当 U:G 比例失衡（U 远多于 G）时 masking 掉的 G 相关维度会引起掉点，用补偿策略自适应重建被抑制的用户-物品交互；masking 大幅削减 U 侧 FLOPs 后模型转为 memory-bound，再叠加 W8A16 weight-only 量化缓解带宽瓶颈。

论文实际报告的线上延迟降幅是 11.5%~22.0%（摘要口径为"最高 20%"），部署于抖音 Feed、红果 Feed、穿山甲广告、千川广告四个场景。

**为什么这么做**

传统 dense 推荐模型中用户特征和候选物品特征在底层就交互，导致每个候选都需重新计算用户侧表示——推理复杂度 O(B)（B 为候选集大小）。UG-Sep 通过 masking 在架构层面隔离用户侧和物品侧，使用户侧计算与候选集大小无关——O(B) → O(1)。

**原始论文深度新见解**

- **只需切断 G→U 单向，不必切断双向**：这是容易读错的地方。目标只是让 U-token 对不同候选保持不变，因此约束是 U-token 不接收 G 侧信息；G-token 仍可自由吸收 U 侧信息。如果误以为要双向隔离成两座孤岛，就变成了双塔模型，dense 交互的表达力也就无从谈起。UG-Sep 的取巧正在于这个不对称性。
- **U:G 比例是效率与精度的直接旋钮，且论文选的是"几乎不掉点"而非"不掉点"**：Table 1 显示抖音 Feed 在 1:2 时 ΔAUC +0.002%、1:1 时 −0.004%（延迟 −20.0%）、3:1 时 −0.013%；红果 1:1 为 −0.018%（延迟 −11.5%）；千川 1:1 为 −0.024%（延迟 −22.0%）。论文明确说明 0.01%~0.03% 的 AUC 下降经严格 A/B 验证对线上几乎无影响。所以准确的说法是"以可忽略的精度代价换取两位数延迟下降"，而非无损。
- **Information Compensation 的触发条件是比例失衡，不是普遍必需**：消融显示 1:2 时不加补偿 ΔAUC 为 +0.00%、1:1 时 −0.01%，此时补偿并无必要；而 3:1 时不加补偿掉到 −0.06%、加了补偿回到 −0.02%。也就是说 U 侧占比越高、被 mask 掉的 G 相关维度越多，补偿才越有价值。第三方评审也指出论文缺一组"只做 masking 不做补偿"的完整对照。
- **cross-attention 出现在残差路径，不在补偿模块**：它不是 Information Compensation 的机制，而是分离式残差的实现方式；其作用是**再加一道 mask 阻止 G 侧流向 U 侧**，与"从 G-token 提取信息补偿 U-token"的方向恰好相反。
- **训练也一起加速，这一点常被漏掉**：Table 2 显示抖音模型在 U:G = 1:2/1:1/3:1 时训练分别加速 5.50%/8.60%/14.8%。与 RankMixer 的 DTSI（只省推理不省训练）相比，UG-Sep 的收益覆盖训练与推理两侧。
- **W8A16 是在 masking 之后才有意义的第二级优化**：顺序不能颠倒。masking 先把 U 侧 FLOPs 砍掉，模型瓶颈从 compute-bound 转为 memory-bound，此时 weight-only 量化才对症。Table 4 给出叠加 W8A16 后的 GEMM 级延迟降幅 40.0%~55.0%（按 M/N/K 配置不同）——注意这是算子级数字，不等于端到端延迟降幅。选 W8A16 而非 W8A8 也是因为瓶颈在权重搬运带宽，压权重即可，压激活收益有限而风险更高。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 线上延迟 | −20.0%（抖音 Feed）/ −11.5%（红果 Feed）/ −12.7%（穿山甲广告）/ −22.0%（千川广告），均为 U:G=1:1 | 四个业务场景 A/B |
| ΔAUC | +0.002% ~ −0.033%（随 U:G 比例变化），论文称 0.01%~0.03% 的下降对线上几乎无影响 | 同上 |
| 训练加速 | +5.50%（1:2）/ +8.60%（1:1）/ +14.8%（3:1） | 抖音推荐模型 |
| GEMM 级延迟（叠加 W8A16） | −40.0% ~ −55.0%，按 M/N/K 配置不同 | 算子级基准，非端到端 |

> 注：摘要口径为"延迟最高降低 20%"，正文 Table 1 的实际区间是 11.5%~22.0%。

**对 Scaling 的意义**：UG-Sep 把长序列模型早已享有的 KV Cache 式复用，第一次搬进了 dense 交互架构——这两类模型此前在推理经济性上并不对等，长序列可以缓存用户历史表示，TokenMixer 却每来一个候选就得把用户侧重算一遍。U 侧复杂度 O(C)→O(1) 之后，候选集规模与用户侧算力解耦，dense 模型继续加宽加深的空间也就打开了。它同时提示一条更一般的设计原则：在网络早期就把稳定计算路径与高频变化路径隔开，稳定的那条才有机会被缓存。

---

### 18. FreeScale — SM-Free 通信与流水线优化

**论文信息**：arXiv:2604.24073, Meta, MLSys 2026

**做了什么**

FreeScale 面向序列推荐模型的分布式训练，针对的是用户交互历史（UIH）长度差异悬殊带来的资源闲置。三大组件：(1) **Sequence Load Balancing**——估算每个样本的计算复杂度后重新分配，缓解 straggler；(2) **Prioritized Embedding Updates**——识别相邻训练迭代之间的 collision rows（冲突行），据此排定通信优先级，让嵌入通信尽可能与计算重叠；(3) **SM-Free Communication**——改走 CPU-RDMA，绕开 NCCL 对 GPU SM 的占用，解决通信与计算重叠时的 GPU 资源争抢。基于 PyTorch + TorchRec 实现，配套定制 Triton kernel。在 256 张 H100、最大 UIH 长度 21,000 的真实负载上，相对 vanilla TorchRec 将 exposed communication 降低 90.3%。

**为什么这么做**

大规模分布式训练的瓶颈不是计算而是通信——GPU 规模扩大时通信量线性增长，传统 NCCL 通信使用 GPU SM 发起 RDMA，占用计算资源导致通信时 GPU 无法全力计算。FreeScale 的 SM-Free 通信将通信从 GPU 卸载到 CPU，实现真正的计算-通信重叠。

**原始论文深度新见解**

- **为什么推荐模型不能照搬 LLM 的序列并行**：论文交代得很清楚——DLRM 面向高吞吐流量设计，单样本计算量相对轻，序列并行或上下文并行引入的额外通信开销会超过收益。所以 FreeScale 走的是"重排样本 + 藏通信"，而不是把长序列切开分发。这解释了整套方案为何全在数据编排与通信层面动手，没碰模型结构。
- **稀疏度与 straggler 是同一个根因的两个表现**：论文用 padding 比例定义 sparsity（把 batch 内所有样本补齐到最长 UIH 所需的填充占比），用各 rank 平均空转时间占迭代总时长的比例定义 straggler 百分比。UIH 长度分布长尾 → padding 多 → 各 rank 实际计算量参差 → 快的等慢的。看清这一点就知道负载均衡不是锦上添花，而是这类模型的第一性问题。
- **冲突行的价值在于"哪些必须同步"而非"哪些可以省"**：嵌入表并行更新时，跨迭代命中同一行的更新必须保序，非冲突行则无此约束。FreeScale 据此把通信排出优先级，让不必阻塞的部分先与计算重叠。论文 Figure 9 给出一个佐证：TorchRec 的 exposed communication 时长基本恒定，FreeScale 则与 collision rate 呈线性关系——说明剩余的暴露通信确实只来自真正的冲突部分，机制按预期生效。
- **SM-Free 的动机是 NCCL 会抢 SM，不是 CPU 更快**：集群规模超过单个 NVL domain 时，NCCL 回退到占用 SM 的实现，还要求预注册内存（需精细内存规划或显式拷贝）。改由 CPU 发起 RDMA，图的是把 SM 完整让给计算，代价是多一次 D2H/H2D 搬运。Figure 10 显示 SM-Free 在这一场景下优于 NCCL。定制 Triton kernel 也有实测支撑：Figure 7 中 world size 512 时 Triton-Dispatch 0.1ms 对 PyTorch-Dispatch 0.6ms，Triton-Combine 59ms 对 PyTorch-Combine 48.9ms，收益主要在 dispatch 侧。
- **"90.3%"的口径需要说清**：论文正文的表述是 exposed communication（暴露的、未被计算掩盖的通信）相对 vanilla TorchRec 减少 90.3%，摘要中概括为 computational bubbles 减少最高 90.3%。不能据此推出"GPU 利用率从约 50% 提升到约 95%"或"256 GPU 有效算力接近线性扩展"——暴露通信减少 90.3% 不等于气泡总量减少 90.3%，更不能反推出利用率的绝对值。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| Exposed communication | −90.3%（相对 vanilla TorchRec） | 256 张 H100，最大 UIH 长度 21,000、batch 128 |

> 注：论文公开的量化结论只有这一项；"掉队者 −9 倍""通信开销 −9 倍"在论文中没有对应出处。straggler 与负载均衡的效果以 Figure 8 曲线形式给出，未给出单一倍数。

**对 Scaling 的意义**：FreeScale 指出序列推荐的分布式训练存在一类 LLM 侧不显著的瓶颈——用户历史长度天然长尾，导致 padding 浪费与 rank 间空转，而这个问题无法用序列并行解决（额外通信开销超过收益）。它给出的三件事都在模型之外：把样本按估算复杂度重排、按冲突关系给嵌入通信排优先级、把通信从 SM 上挪走。含义是长序列方向的 Scaling 受制于数据编排与通信调度，光有架构设计不足以把算力吃满。已被 MLSys 2026 接收，实现基于 PyTorch/TorchRec，工程可复现性相对较好。

---

## 主线五：底层基建（3 篇）

### 19. Design Once — SMT 标准化模型模板

**论文信息**：arXiv:2603.24963, Meta AI

**做了什么**

Design Once 提出 Standard Model Template（SMT），把推荐模型的组件标准化为经过验证的可组合模块，使技术传播复杂度从 O(n·2^k) 降到 O(n+k)（n 为模型数，k 为技术数）。

两个关键机制容易被漏读：(1) **迭代解耦**——把模型演进拆成 Template Iteration（集中做创新与泛化验证）与 Model Iteration（高效实例化具体模型）两条独立轨道，创新只在模板层做一次；(2) **MMO（Multi-Model Optimization）**——新技术不在单个模型上调参，而是在一个**代表性模型集合**上联合优化，导出一套全局稳健的超参，再铺到全量模型。这两点合起来才是复杂度降阶的实际实现方式，也是"降低回归率"成为关键指标的原因。

在 Meta 生产广告排序生态、跨四个全球开发周期的评估结果：离线 NE（Normalized Entropy）平均改善 0.63%（服务算力持平），线上累计提升 0.86%，每模型迭代工程时间减少 92%，技术-模型配对采用吞吐提高 6.3 倍——相当于让 5 倍数量的模型受益于新技术。

**为什么这么做**

Meta 内部有数百个推荐模型和数十项可用技术，每项技术需要在每个模型适配。传统方式下 k 项技术在 n 个模型中的传播复杂度为 O(n·2^k)——每项新技术需在每种模型组合上测试适配，组合爆炸使传播成本指数增长。SMT 的标准化接口将技术传播从"为每个模型定制适配"变为"一次开发、全模型可用"，复杂度降为 O(n+k)。

**原始论文深度新见解**

- **2^k 的来源是"组合验证"，不是"逐个适配"**：这里容易读浅。论文 Figure 1 描述的传统流程是：对每个模型，工程师先在 k 项技术上各自试错调参，筛出有效的子集，**再评估这些技术所有可能的组合**（最多 2^k 种）才选定最终方案。指数项来自技术之间的相互作用必须成对成组地验证，而非来自技术数量本身。更麻烦的是论文点出的一个动态效应：任何针对某模型的新技术 T_j 都必须先验证与该模型既有技术池的兼容性，**所以模型越老、技术池越厚，增量技术反而越难上线**。这解释了为什么单纯"加人加机器"解决不了问题——成本随时间复利式增长。
- **降阶靠的是把组合验证移到模板层做一次**：n+k 里的 k 是模板层验证 k 项技术，n 是实例化 n 个模型。关键前提是模板层验证过的组合在各模型上仍然成立——这正是 MMO 用代表性模型集导出全局稳健超参要保证的事，也是论文强调最小化回归率的原因。如果模板层的结论在下游频繁失效，复杂度就会退回去。
- **"neutral serving capacity"是约束条件，不是效果描述**：它指的是在服务算力持平（不额外多花推理资源）的前提下取得 0.63% 的 NE 改善。neutral 修饰的是成本侧而非收益侧，把它读成"跨场景的中性提升、不恶化任何场景"就反了。这个约束让结论更有分量：不是用更多算力换来的效果。
- **反直觉的地方在于标准化通常被认为要付代价**：常识预期是通用模板不如逐模型精调，收益应该为负、只是被工程效率补偿掉。论文的实测是**离线 NE 与线上都为正**（+0.63% / 累计 +0.86%），这才构成对"多样化目标必然需要多样化设计"的实质挑战。一个合理的解释是：逐模型独立试错容易陷入局部最优且难以吸收外部创新，而模板层集中优化能把最佳实践更快铺开——效率与效果在这里不是取舍关系。
- **第三方评审对因果归因的质疑值得如实标注**：这是一项跨四个开发周期的生产环境对照研究，非随机实验。评审指出论文未说明模型如何分组、是否做了随机化或匹配、如何平衡同期的平台/数据/基础设施变更，也未给统计检验与误差范围，因此三项数字难以严格归因于 SMT 本身。引用时宜作为工程实践的经验证据，而非受控实验结论。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 离线 NE（Normalized Entropy） | 平均改善 0.63%，服务算力持平 | Meta 生产广告排序，四个全球开发周期 |
| 线上累计提升 | +0.86% | 同上 |
| 每模型迭代工程时间 | −92% | 同上 |
| 技术-模型配对采用吞吐 | 6.3 倍（约 5 倍数量的模型得以受益于新技术） | 同上 |

> 注：论文摘要用 cross-entropy 表述该指标，解读材料则记为 Normalized Entropy，二者在广告排序语境下通常指同一族指标，引用时建议照摘要原文写 cross-entropy。该研究为生产环境跨周期对照，非随机对照实验。

**对 Scaling 的意义**：这篇处理的是全篇唯一一个组织与流程层面的 Scaling 瓶颈——被 Scaling 的不是模型参数，而是**模型的数量**。当生态里有数百个模型、数十项技术时，真正的约束变成了创新的扩散速度：研究产出快于落地能力，技术就会积压，而且模型越老、既有技术池越厚，新技术越难挤进去，成本随时间复利增长。SMT 的应对是把组合验证从"每模型各做一遍"移到模板层做一次。它同时提供了一个反直觉的经验证据：标准化没有以效果为代价换效率，两者同向。需要注意结论的适用前提——这套做法的价值随模型数量 n 与技术数量 k 增长而放大，对只维护少数几个模型的团队，收益远不如 Meta 这种量级明显。

---

### 20. LoopFM — Foundation Model 知识迁移

**论文信息**：arXiv:2605.29280, Meta

**做了什么**

LoopFM（Learning frOm HistOrical RePresentations of FM）把 Foundation Model 的历史中间层表征物化为 Vertical Model 的输入特征。注意 VM 是 **Vertical Model（垂直模型，即线上小模型）**，不是 Value Model。

三个可独立配置的模块化阶段：(1) **Extraction**——从 FM 指定层取历史时点的中间嵌入；(2) **Compression**——AutoEncoder 降维 + 量化（如 INT4）压低存储成本；(3) **Structuring**——按 grouping key（用户 ID、物品 ID 等）把压缩后的嵌入组织成序列或图，本文聚焦 user-keyed 时序序列。关键约束是**只用历史嵌入**，因此线上服务无需实时调用 FM，也不要求 FM 与 VM 之间存在架构耦合。

理论侧给出信息增益分解与迁移率下界：`TR ≥ (I_temporal + I_cross − C_compression) / ΔNE_FM`，其中 I_temporal 为时序历史信息、I_cross 为跨特征信息、C_compression 为压缩损失。三条结论是：下界随 FM 相对 VM 的特征差距单调上升（Corollary 4）、随压缩质量改善而变紧（Theorem 2）、信息增益随序列长度非减（Theorem 5）。

**为什么这么做**

论文的起点是一个可量化的退化现象，不是泛泛的"FM 太大部署不了"。工业推荐普遍是两层架构：万亿参数级 FM 离线学表征，紧凑 VM 在毫秒级延迟下出预测，中间靠知识蒸馏（KD）以标量软标签衔接。论文定义**迁移率 TR = ΔNE_VM / ΔNE_FM**（VM 捕获到的 FM 提升占比），并观察到随 FM 扩到多万亿参数，TR 持续恶化。

诊断是**带宽瓶颈**：单个标量预测要把 FM 学到的跨域特征、多层交互模式、上下文信号全压进一个数字；FM 越强，它学到的与一个标量能传达的之间落差越大。再者 FM 通常吃比 VM 更丰富的跨域特征，这个 feature gap 是标量 KD 根本跨不过去的。所以 LoopFM 的定位不是替代 KD，而是**并联一条高带宽通道**，论文实验也显示两者是互补关系。

**原始论文深度新见解**

- **"只用历史嵌入"是整套方案成立的支点**：这一条约束同时买到三样东西——线上不必实时跑 FM（省延迟与算力）、FM 与 VM 无需架构耦合（两侧可独立迭代）、以及 VM 可以用新鲜的历史嵌入更频繁更新而不必重训 FM。反过来它也划定了适用边界：LoopFM 传递的是"截至上次快照 FM 对该用户的理解"，对需要实时反映当前上下文的信号无能为力。论文的消融里专门有一项 checkpoint freshness，说明快照新鲜度确实是个需要管的变量。
- **迁移率下界随 feature gap 单调上升，这是最反直觉也最有用的一条**：常识会觉得师生差距越大蒸馏越难（论文也确实引用了 Cho & Hariharan 关于容量差距下 KD 退化的结论），但 LoopFM 的 Corollary 4 指出，FM 相对 VM 的**特征差距**越大，这条高带宽通道的迁移率下界反而越高。两者并不矛盾：标量 KD 的瓶颈在带宽，差距越大越传不动；而直接搬中间表征时，FM 独有的那部分跨域信息正是 VM 拿不到的净增量。推论是 LoopFM 最该用在 FM 与 VM 特征集差异大的场合，而不是两者特征几乎相同的场合。
- **增益分解的三项要按"净增量"读**：`I_temporal + I_cross − C_compression`——前两项是收益（时序历史、跨特征），第三项是压缩带来的信息损失，必须扣除。Theorem 2 说压缩越好下界越紧，Theorem 5 说序列越长增益非减。这给出了明确的调优方向：优先改善压缩质量与拉长序列，而不是一味换更大的 FM。注意第三项是压缩损失，不是"FM 与 VM 都已捕获的冗余信息"，两者不是一回事。
- **"迁移率翻倍"缺少绝对值，引用时需谨慎**：论文只说在 KD 之上把迁移率大致翻倍，没有给出翻倍前后的具体数值，也没有"KD 通常 10-20%、LoopFM 达到 20-40%"这类区间。第三方评审同样指出这一点：翻倍是支撑论文意义的核心主张，却未报告基线与改进后的 TR 数值，也没把增益分解公式套到观测到的 delta 上。
- **公开基准三个数据集的收益差异很大**：TaobaoAd AUC +6.4%（区间 6.1%~6.6%）、KuaiVideo +1.0%（0.6%~1.6%）、Amazon Electronics +0.5%（0.02%~1.14%）。常被单独引用的"+6%"是三者中最高的那个，Amazon 上区间下界几乎为零。这个跨数据集的落差本身与理论一致——feature gap 越小，可迁移的净增量越少。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| AUC | +6.4%（6.1%~6.6%） | TaobaoAd（公开基准） |
| AUC | +1.0%（0.6%~1.6%） | KuaiVideo（公开基准） |
| AUC | +0.5%（0.02%~1.14%） | Amazon Electronics（公开基准） |
| 转化率 | 首个半年 +0.5%；次半年两次独立上线分别 +1.03%、+1.22% | Meta 工业系统（十亿级样本、万亿参数 FM） |
| 迁移率 | 在 KD 之上约翻倍（论文未给出翻倍前后的绝对数值） | 工业系统 |

> 注：Matryoshka 表征学习、"中上部 60-80% 深度层效果最佳"、"总压缩约 128 倍"三项未能在论文或可核实材料中找到出处，已从正文删除。论文的压缩阶段写的是 AutoEncoder 加量化（如 INT4），层选择列为消融项之一但未公开最优区间。

**对 Scaling 的意义**：LoopFM 处理的是一个容易被忽略的 Scaling 断点——FM 一侧的 Scaling 收益能不能真正落到线上。当迁移率随 FM 变大而持续下滑时，继续加大 FM 就成了亏本买卖，瓶颈不在 FM 也不在 VM，而在两者之间的传输带宽。把标量通道换成表征通道之后，FM 的规模上限重新只受离线成本约束，且理论上迁移率下界随特征差距上升，意味着 FM 越是往"更多跨域特征"方向扩张，这条通道的价值越大。代价是引入了一套嵌入快照的存储与新鲜度管理，属于用离线存储换在线延迟的取舍。

---

### 21. CAdam — Confidence-Based Optimization

**论文信息**：arXiv:2411.19647, Tencent, v2 June 2025

**做了什么**

CAdam 提出 Confidence-Based Optimization for Online Learning，解决 Adam 优化器在推荐在线学习中的两大问题：(1) 过时动量和平均平方梯度导致分布变化适应慢；(2) 数据噪声敏感性。核心机制：每次参数更新前评估动量 m_i 与梯度 g_i 的一致性——若对齐（m_i·g_i > 0）执行 Adam 原始更新，若不对齐（m_i·g_i ≤ 0）暂时扣留更新并监测后续迭代以区分真实分布漂移与噪声。关键细节是**扣留而不清除**：暂停该维度的更新但保留动量，让动量自然衰减——若只是噪声，下一步梯度大概率重新与动量对齐；若是真实漂移，动量最终会翻转到新方向。方法不引入任何额外超参，计算开销可忽略，可直接替换现有训练流程中的 Adam。腾讯生产环境 48 小时 A/B（模型规模最高 3300 亿参数）平均 GAUC 提升 0.30%，已在 16 个业务场景稳定运行超过九个月。

**为什么这么做**

推荐系统使用在线学习（实时更新模型），数据分布随时间漂移（用户兴趣变化、热门内容更替）。Adam 的动量 m 和平均平方梯度 v 是历史累积的，当分布漂移时这些历史统计量"过时"——动量指向旧分布的梯度方向，与新分布的梯度方向冲突，导致适应慢。直接降低动量衰减率又会对噪声敏感（单次噪声梯度被放大）。

**原始论文深度新见解**

- **判据是逐参数维度的符号一致性，不是全局的**：一致性检查对每一个参数维度独立进行——同一步内，一部分维度照常更新，另一部分被扣留。这一点决定了 CAdam 的粒度优势：分布漂移通常只影响部分特征对应的参数，全局性的动量重置会连带牺牲那些本来没受影响的维度。
- **"扣留"与"清零"的区别是这篇论文的实际发明点**：检测到冲突时把动量清零是直觉做法，但单次噪声梯度同样会触发冲突，清零等于让噪声抹掉了积累的历史。CAdam 只是暂停该维度的更新、让动量自然衰减，把判断权交给后续若干步：噪声会自行归位，真实漂移则会持续施压直到动量翻转。等于用"等一拍"换取了区分噪声与漂移的能力，而这个区分在单步内是原理上做不到的。
- **论文的立论建立在一个具体的分布漂移图景上，值得照原样理解**：作者举的例子是同一平台上早间以老年用户为主、晚间以在职人群为主——**用户偏好函数 f̂(x) 本身相对稳定，变的是输入 x 的分布**。这个设定很重要：它意味着漂移不必然要求模型忘掉旧知识，只是要求优化器别再沿着旧输入分布下的方向硬推。这也解释了为何"暂停"而非"重置"是对症的处置。
- **"Confidence"指的是对梯度方向的信心，不是对预测的信心**：容易误读成模型输出置信度。这里的置信度是优化器层面的——当前梯度与历史动量同向则视为可信、执行更新，反向则视为待观察。
- **零超参、零额外状态是它能上生产的关键**：CAdam 不新增任何需要调的超参，也不需要额外的状态张量，计算开销可忽略，可即插即用替换 Adam。对于已经跑着数百个在线学习模型的平台，这个性质比多几个点的离线收益更有分量——已在腾讯 16 个场景稳定运行九个月以上，本身就是这一点的证据。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| GAUC | 平均 +0.30% | 腾讯生产环境 48 小时 A/B，模型规模最高 3300 亿参数 |
| 部署规模 | 16 个业务场景，稳定运行 9 个月以上 | 腾讯 |
| 噪声鲁棒性 | Criteo-x4-001 注入 1% 标签噪声下，Adam 明显退化而 CAdam 接近无噪声水平 | 公开基准 |
| GMV | 论文表述为"substantial increases"，未给出具体百分比 | 腾讯推荐 |

> 注：论文摘要只说 GMV 大幅提升而未给数值，可核实材料补充了 GAUC +0.30%、3300 亿参数、16 场景 9 个月等细节。离线验证覆盖合成任务（L1-sudden 等突变/线性/正弦漂移）、CIFAR-10 图像分类与 Criteo 推荐基准。

**对 Scaling 的意义**：这是全篇唯一一篇在优化器层面谈 Scaling 的工作，补上了一个容易被架构讨论盖过的环节——模型再大，在线学习下如果优化器沿着过时方向推，规模优势也兑现不出来。它在 3300 亿参数模型上验证有效，且零超参、零额外状态，意味着规模扩张时不需要重新调参，这对同时维护大量在线模型的平台是实际的部署价值。反过来也要看清它的边界：CAdam 解决的是漂移适应与噪声鲁棒，不改变模型容量或表达力，收益量级（GAUC +0.30%）与架构类改动不在同一个数量级。

---

## 主线六：生成式推荐（2 篇）

### 22. OneRec — 生成式端到端推荐

**论文信息**：V1 会议版 arXiv:2502.18965；技术报告 arXiv:2506.13695；V2 技术报告 arXiv:2508.20900，Kuaishou

**做了什么**

OneRec 提出生成式端到端推荐范式，从判别式（打分 + 筛选候选集）转向生成式（直接生成推荐序列）。需要先区分两代架构，二者机制不同不可混谈：

**V1（arXiv:2506.13695）**：Encoder-Decoder 架构。(1) 协同感知的多模态语义 ID tokenizer——用 RQ-Kmeans 残差量化把每个视频编码为 3 层由粗到细的语义 ID，输入端融合标题、标签、ASR、OCR、封面与视频帧（多模态特征提取用 miniCPM-V-8B，压缩用 QFormer），并融入用户行为的协同信号；(2) Encoder 做多尺度用户建模（静态特征、短期序列、有效观看序列、终身序列），Decoder 用 MoE 扩容并以 Next Token Prediction 训练，采用 session-wise 生成；(3) 强化学习偏好对齐——偏好奖励（P-Score）、格式奖励、工业场景奖励三类，优化算法是 **ECPO（Early-Clipped GRPO）**，对负优势样本施加更严格的梯度截断。

**V2（arXiv:2508.20900）**：针对 V1 的算力错配提出 **Lazy Decoder-Only** 架构。V1 在 context 长度 512 时，97.66% 的 FLOPs 花在序列编码上，真正做生成决策的 target decoding 只占 2.34%，所以扩不上去。V2 直接去掉 Encoder，用户上下文当作静态条件经 Lazy Cross-Attention 注入且不参与 Loss，配合层间 KV 共享与 GQA，总计算量降 94%、训练资源降 90%，从而在同等预算下扩到 8B。后训练改用真实用户反馈（观看时长）而非奖励模型代理信号，含 Duration-Aware Reward Shaping（按视频时长分桶取分位数，消除长视频天然占优的偏置）与 Adaptive Ratio Clipping，论文另提出 GBPO（Gradient-Bounded Policy Optimization），用 BCE 梯度为 RL 梯度设界以防梯度爆炸。

> 注：V1 是 Encoder-Decoder，Lazy Decoder-Only 属于 V2 的改动，两代不可混谈。语义 ID 用的是 RQ-Kmeans 残差量化，不是 VLM + Q-Former 做层次分解。

**为什么这么做**

论文列出的三条动因值得照原样理解，因为它们决定了 OneRec 改的是什么。一是算力碎片化：快手链路中算力最重的精排模型 SIM，训练/推理 MFU 只有 4.6%/11.2%，而级联架构下超过一半的服务资源花在阶段间通信与存储而非高精度计算上。二是目标冲突：平台要同时优化用户、创作者、生态的数百个目标，各阶段互相掣肘。三是技术代差：级联结构接不住 Scaling Law 与强化学习这类进展。

需要注意"突破候选集上限"只是其中一个侧面，且论文的表述更克制——它指出预训练只是拟合过去系统的曝光分布，因此无法突破原系统天花板，真正用来突破的手段是 RL 偏好对齐，不是生成式本身。

**原始论文深度新见解**

- **语义 ID 是残差量化的产物，层次性来自残差逐级细化**：RQ-Kmeans 每一级对上一级的残差做聚类，三级得到 3 个 token。层次性因此是残差结构的自然结果，而非人为规定"第 1 位是粗类目、第 2 位是细类目"。这个区别有实际后果：语义相近的物品共享前缀，冷启动物品的 ID 由内容与协同信号生成，因此新物品无需积累行为即可被生成出来。论文另提到量化的"沙漏现象"——码字分布不均衡，是 RQ-VAE 一类方法的已知问题。
- **97.66% 这个数字是 V2 全部改动的出发点，也是本篇最值得记住的一条**：Encoder 处理 512 长度的用户序列，Decoder 只预测 3 个语义 token，于是算力几乎全花在编码上下文而不是做决策。序列越长失衡越严重（3000 token 时 target decoding 仅占 0.41%）。这解释了为什么 V2 的解法不是把 Encoder 做小，而是整个去掉——瓶颈不在 Encoder 效率低，而在"投入编码的算力不产生生成质量的边际收益"。1B 规模下 GFLOPs 从 296.4 降到 18.9，收敛 loss 反而略好。
- **"Lazy"的含义是上下文只算一次、跨层复用，不是跳层**：Lazy Cross-Attention 去掉了 cross-attention 内部的 key/value 投影层，直接用 Context Processor 预计算好的、各层共享的 (k_l, v_l)，再配 GQA 让多个 query head 共享 KV head。它省的是"每层重复投影上下文"的开销，与"某些层跳过前向"无关。
- **Duration-Aware Reward Shaping 解决的是时长偏置，不是奖励稀疏**：短视频即使质量很高，观看时长上限也就是视频本身长度，直接比绝对时长会系统性偏向长视频。做法是按视频时长分桶，取观看时长在该桶内的分位数作为偏好信号。这样奖励反映的是内容质量而非视频长度。
- **RL 会带来"挤压效应"这个副作用，论文的处理方式很有代表性**：负优势的梯度把概率质量挤向当前最优输出，导致生成的语义 ID 序列大量落到词表中不存在对应视频的位置，即非法生成。OneRec 的对策是再加一个格式奖励。更有意思的是它的消融结论：用概率最大的 k 个样本训练会诱发 reward hacking（总体合法性先升后降），随机挑 k 个则两条曲线同步上升。这说明奖励项之间会互相干扰，加奖励不是单调改善。
- **Scaling Law 的成立范围要说清楚**：V1 在 0.015B→2.633B 区间观察到训练损失持续下降，V2 在 0.1B→8B 区间收敛 loss 从 3.57 降到 3.19。但 V2 论文自己指出，这条下降并不完全符合 scaling law 的形式，仍需在架构、数据组织与预训练方法上继续探索。对外引用时不宜简化成"生成式推荐已验证 Scaling Law"。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| MFU（训练/推理） | 23.7% / 28.8%（对照精排 SIM 的 4.6% / 11.2%） | 快手推荐，V1 |
| OPEX | 降至传统级联链路的 10.6% | 快手推荐，V1 |
| App 停留时长 | +0.54%（主站）/ +1.24%（极速版），需叠加 RM Selection | 快手推荐，V1 |
| LT7 | +0.05% / +0.08%（快手体系内 0.01% 即具统计显著性） | 快手推荐，V1 |
| GMV / 订单量 / 购买用户数 | +21.01% / +17.89% / +18.58%，新客获取 +23.02% | 快手本地生活，V1，已全量 |
| 流量承接 | 约 25% QPS | 快手/快手极速版 |
| 计算量与规模 | 总计算量 −94%、训练资源 −90%，扩至 8B；收敛 loss 3.57（0.1B）→3.19（8B） | V2 |
| App 停留时长（V2） | +0.467% / +0.741% | 快手/快手极速版 |

> 注：OPEX 的口径是"降至传统方案的 10.6%"（即节约约 90%），不是"−25%"。停留时长两个数值分属主站与极速版，且论文明确是"OneRec with RM Selection"配置下取得；纯生成式 OneRec 单独上线只是打平原级联系统。V1 与 V2 的数据不可混用。

**MFU 从个位数抬到 23.7% 靠什么**

这部分是 OneRec 最容易被低估的贡献，但需要按论文的层次讲，不宜包装成并列的"四大突破"。

首要来源是架构本身而非某项算子技巧：快手精排模型积累了 15000+ 个算子，结构过于零碎，无法像 LLM 那样做深度优化，单算子计算密度低到让显存带宽成为天花板。换成类 LLM 的 Encoder-Decoder 后关键算子数量压缩 92% 至约 1200 个，配合更大的模型规模抬高计算密度，同时链路重构释放了延迟压力。**先有结构规整，才谈得上后面的优化**。

训练侧的具体手段：同一请求下的多条曝光样本共享用户与 context 特征，按请求 ID 分组即可避免在 context 序列上重复算 FFN，再用变长 flash attention 省掉重复的 KV 访存；Embedding 一侧自研 **SKAI** 框架把训练全流程放在 GPU 上完成，消除 GPU/CPU 同步中断，其中 **UGMMU（统一 GPU 内存管理）** 用于大幅减少 kernel 数量，缓存侧采用**时间加权 LFU** 利用数据的时间局部性，另有 Embedding 预取流水线把参数传输与计算重叠。并行策略是数据并行 + ZERO1 + 梯度累积，选 ZERO1 的原因是 dense 参数小到单卡可容纳完整参数与梯度。

推理侧围绕大 beam size（通常 512）做计算复用：同一请求下 encoder 侧特征在所有 beam 上一致，故 encoder 只前向一次；cross-attention 的 key/value 在 beam 间共享；decoder 内部用 KV cache 缓存历史步。再叠加 Float16 混合精度、MoE/Attention/BeamSearch 的 kernel 融合，以及按负载实时调整 batch 大小的动态 Batching。推理硬件是 NVIDIA L20，用 TensorRT 编译计算图并以自定义插件实现 cross-attention 与 MoE，配合 batching 与 MPS 拿到 5 倍吞吐提升。

> 注：几个容易误传的组件名与归属需澄清——SKAI 是 Embedding 训练框架而非通信调度器，UGMMU 是它的内存管理组件而非独立创新，缓存算法是时间加权 LFU（无"TLFU-P"这一名称）。"cuDF 基 UGMMU""50TB 级 Embedding""Warp 级 Kernel Fusion""三阶段静态通信调度""FA2 Cross-Attention 定制""FA2 Block 启动方向改为 Q 维度"在论文中均无对应，"把动态决策转化为静态决策"也不是论文的表述。

**对 Scaling 的意义**：OneRec 的贡献是把推荐系统放回了能被 Scaling 的形状上。级联架构下算力碎片化、算子零碎、MFU 长期个位数，规模化的收益还没到模型就先漏在了阶段间通信里；换成端到端生成式之后，MFU 抬到 23.7%/28.8%、OPEX 降到传统链路的 10.6%，LLM 社区的训练与 serving 技术才第一次能直接搬过来用。

更值得记住的是 V1 到 V2 这一步暴露的规律：**架构统一之后，瓶颈会转移到算力在架构内部的分配上**。V1 已经是端到端了，但 97.66% 的算力花在编码上下文而非生成决策，模型规模因此扩不上去；V2 把这笔预算挪给 decoder 才扩到 8B。这说明"能不能 Scaling"不只取决于总算力，还取决于算力有没有花在产生边际收益的那部分计算上。

边界也要说清楚：V2 论文自述 loss 下降并不完全符合 scaling law，且 V1 明确指出推理阶段的 step scaling 效果不明显——即 OneRec 尚不具备强推理能力，这与 LLM 的 test-time scaling 有实质差距。

---

### 23. GR4AD — 生成式广告推荐

**论文信息**：arXiv:2602.22732, Kuaishou Technology

**做了什么**

GR4AD（Generative Recommendation for ADvertising）把生成式推荐做进广告场景，四项设计分别对应架构、学习与服务：(1) **UA-SID（Unified Advertisement Semantic ID）**——先用广告领域指令微调的 MLLM（Qwen3-VL-7B）产出统一广告嵌入 UAE，再用 MGMR RQ-Kmeans 量化为语义 ID；(2) **LazyAR（Lazy Autoregressive decoder）**——把自回归依赖推迟到第 K 层注入，前 K 层并行计算，为"短序列、多候选"生成降推理成本；(3) **VSL（Value-Aware Supervised Learning）+ RSPO（Ranking-Guided Softmax Preference Optimization）**——前者按价值给样本加权并预测 eCPM token，后者是 list-wise 的强化学习，直接对 NDCG 这类列表级指标优化；(4) **Dynamic beam serving**——按生成层级与实时负载调整 beam 宽度。

> 注：VSL 与 RSPO 的全称按论文原文为准，不是"Variance-aware Sequence Learning"与"Reward Shaping with Preference Optimization"。该文亦无 KDD'25 的会议归属。

**为什么这么做**

论文给出的三条理由，都是"LLM 那套配方直接搬过来不够用"的具体表现。一是分词：广告创意把视频属性、商品详情与 B2B 广告主元数据混在一起，而转化目标、广告账户这类商业信号无法从内容语义里读出来，此前也没有端到端微调过的广告 LLM 嵌入。二是学习范式：广告优化的是列表级业务指标（eCPM、NDCG），逐条样本的 next-token 监督刻画不了。三是实时服务：生成式并没有解除 DLRM 时代的延迟约束，系统仍要在高流量下于严格延迟预算内产出多个高质量候选（论文口径是单卡 L20 上 500+ QPS、延迟 100ms 以内），这与 LLM 交互式使用中"单条回复可以慢"的场景根本不同。

需要修正一个方向性判断：UA-SID 编码的是**转化类型、账户 ID 这类非语义业务特征**，用来降低语义 ID 碰撞、区分内容相同但投放策略不同的广告；它并不编码出价、预算、ROI 约束，也不承担"生成时满足竞价约束"的职责。业务价值对齐由 VSL 与 RSPO 在学习目标一侧完成。

**原始论文深度新见解**

- **UA-SID 的"统一"是指语义与非语义信号在同一串 ID 里，靠的是最后一层换掉量化方式**：UAE 侧用六类指令模板微调 Qwen3-VL-7B 覆盖直播、实物商品、虚拟服务等异构广告形态，再用 Swing 估计的共现物品对做 InfoNCE 对比学习注入协同信号，图到图检索 R@1 从 QARM 的 0.541、原始 Qwen3-VL-7B 的 0.769 提升到 0.896。量化侧的 MGMR 有两层含义：multi-resolution 指码本逐级变小（16384/4096/1024），前面的残差熵更高所以给更大词表；multi-granularity 指**把最后一级的向量量化直接替换为对非语义业务特征（物品/账户 ID、转化类型）的哈希映射**。动机很具体——内容一样的广告，因广告主转化目标不同，投放轨迹会有实质差异，靠内容语义分不开。
- **LazyAR 的巧处在于它利用了"难算的位置不贵、贵的位置不难"这个不对称**：生成语义 ID 时，第一位最难预测但 beam 宽度为 1、计算量小；后面几位好预测，却要在数百条 beam 上重复算。LazyAR 于是把自回归依赖推迟——总共 L 层里前 K 层并行计算，只处理用户历史与结构化特征形成潜表示，前一个 token 的嵌入直到第 K 层才通过轻量融合算子注入。L=9、K=6 的配置下吞吐提升 117%，效果仅有边际损失。训练时靠辅助的 MTP（multi-token prediction）损失，让这些并行层在拿不到前序 token 的情况下仍学得会预测。它放松的是**层间**依赖，不是用低秩近似替代隐状态。
- **VSL 是价值感知的样本加权加一个 eCPM token，与方差无关**：权重按 `w = w_user · w_behavior` 组合，即依用户长期价值与交互深度加权，而不是平等对待每次点击或转化；模型还在生成广告 token 的同时预测一个"eCPM token"，损失写作 `L_NTP = L_SID + λ_e · L_eCPM`，完整目标再叠加支撑 LazyAR 的 `L_MTP`。此前"按生成方差调学习率"的描述与论文无关。
- **RSPO 的落点是把 LambdaLoss 那套排序思想搬进 RL 目标**：不做 pairwise 比较，而是直接针对 NDCG 这类列表级指标做 list-wise 优化。更值得注意的是 VSL 与 RSPO 不是先后两阶段而是同时更新，权重按排序差异分数 `A^(i)` 动态分配——模型当前排序离理想排序远时偏向监督学习求稳，接近时偏向 RL 求进一步的价值优化。RSPO 单项带来 3.86% 的收入增益。
- **Dynamic beam serving 的调节依据是流量负载与生成层级，不是请求难易**：流量感知的 TABS 在非高峰放宽 beam 以探索更多候选、高峰收窄以守住延迟；渐进式 beam 调度则让前几个 token 从较小宽度起步，随生成逐步确定再放大。配套还有 beam 间共享 KV cache 与 FP8 低精度推理。判据是系统侧的，与"用户兴趣是否明确"无关。
- **实时索引这一项容易被忽略但对广告很关键**：GR4AD 用双向的 UA-SID ↔ 物品 ID 索引替代了向量最近邻检索，新广告创建后索引可在秒级更新，使其立即可被生成出来。这是生成式范式在广告冷启动上的实际优势——论文报告中小广告主投放提升 17.5%，正来自内容语义 ID 带来的泛化能力。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 广告收入 | 最高 +4.2%（对照现有 DLRM 链路） | 快手广告，4 亿+用户，已全量部署 |
| 中小广告主投放 / 转化率 | +17.5% / +10.17% | 快手广告 |
| RSPO 单项收益 | +3.86% 收入 | 消融 |
| LazyAR 吞吐 | +117% QPS（收入仅边际损失） | L=9, K=6 |
| UA-SID 检索质量 | 图到图 R@1 0.896（QARM 0.541，原始 Qwen3-VL-7B 0.769） | 离线 |
| 服务约束 | 单卡 L20 500+ QPS，延迟 <100ms | 线上 |
| 模型 Scaling | 0.03B→0.32B，训练损失与线上收入单调改善 | 线上 A/B |

**对 Scaling 的意义**：GR4AD 的价值不在"把 OneRec 搬到广告"，而在于它给出了**固定服务预算下的 Scaling 路径**。广告场景的延迟与吞吐约束是硬的，所以论文里模型规模（0.03B→0.32B）与推理算力（beam 宽度）这两条 Scaling 曲线都是在成本约束下测的，且都出现收益递减。LazyAR 的作用正是把省下来的推理预算腾给规模扩张——这与主线四"推理解耦"的思路同源：**先把算力从不产生边际收益的位置挪走，Scaling 才有空间**。

需要注意它的边界：论文自述生成式架构对算力的原始需求仍高于精简的 MLP 模型，参数规模上限（0.32B）也远低于 OneRec 的 8B，说明广告实时场景下能扩到多大仍受服务预算而非算法约束。

---

## 主线七：表征健康（2 篇）

> **正交诊断框架**
>
> 本线不含独立论文，是把主线二内 RankMixer → RankUp → RankElastor 这条链的贡献，提炼为一条独立的方法论主线。核心原理是：effective rank 与 MFU 正交——MFU 回答"算力有没有被浪费"，effective rank 回答"表达能力有没有被浪费"，两者可以独立退化。
>
> 一个架构完全可能 MFU 拉满、同时表征严重坍缩，此时效果天花板不是被算力卡住而是被表征健康卡住，而这一点从 MFU 上完全看不出来。前六条主线都在回答"如何用更少算力做更大模型"，这条线追问的是另一个问题："新增参数是否真正转化为表达能力"。需要说明归属：有效秩沿层数呈阻尼振荡这一观察出自先前的 RankMixer 研究，RankUp 与 RankElastor 是在此观察之上提出对策的工作。
>
> 这个视角不局限于 RankMixer 的 MetaFormer 结构——RankUp 论文本身就把观察范围列到了 RankMixer、HiFormer、MixFormer、TokenMixer-Large 等一族模型，对 Kunlun 与 ULTRA-HSTU 这类 Self-Attention 变体是否存在各自版本的秩退化，本文档未见论文验证，属待考问题。把它归纳为"效率 + 表征质量"双维度评估，是本文档的提炼而非某篇论文的结论。
>
> 主线七与主线一 TokenFormer 的 Sequential Collapse Propagation 关注的是同一类风险的不同环节：前者是 MetaFormer 骨干沿层深的有效秩退化，后者是静态特征与序列特征混合时对序列表征的拖低。两篇论文之间没有互相引用，把它们并置是本文档的归纳——共同提示表征坍缩可能同时出现在 Token 化层和交互 Block 层。

### 24. RankUp — 有效秩阻尼振荡与五机制

**论文信息**：arXiv:2604.17878, Tencent / WeChat

**做了什么**

RankUp 解决 MetaFormer 推荐模型的有效秩（Effective Rank）退化问题——token 表征的有效秩沿**层数**（across layers）呈阻尼振荡轨迹，不随深度持续上升、深层甚至下降，导致表征坍缩。注意这条轨迹的自变量是层深，不是训练步数。提出五机制：(1) Randomized Permutation Splitting——随机置换算子 σ 打乱稀疏特征索引，分散高相关特征到不同 Token，减少 Token 间相关性；(2) Multi-embedding Representation Paradigm——K 个独立嵌入表，同一类别信号多几何视角表示；(3) Global Token Integration——Token mixing 中与全局上下文信息交互；(4) Crossed Pretrained Embedding Tokens——跨域/场景/任务预训练嵌入融合；(5) Task-Specific Token Decoupling——多目标梯度干扰缓解。部署于微信视频号/公众号/朋友圈。

**为什么这么做**

推荐模型的表征健康问题长期被忽视——模型效果提升但表征空间可能在"坍缩"（有效秩下降意味着表征维度利用不充分，信息容量退化）。RankUp 的出发点不是自己发现该现象——论文明确写的是"Prior studies on RankMixer reveal"，即阻尼振荡由先前的 RankMixer 研究观察到，RankUp 是受此启发提出对策，从特征组织、嵌入表示、全局上下文、预训练迁移、多目标解耦五个层面切入。

**原始论文深度新见解**

- **阻尼振荡是层间现象，不是训练动力学**：有效秩定义为 erank(H_l)=exp(−Σ p_i ln p_i)，p_i 为表征矩阵奇异值的归一化占比。振荡沿网络层深展开：越往深层有效秩不能持续上升、甚至退化。论文把成因归结为 MetaFormer 两类算子的固有性质——token mixer 只能提供有界的 rank expansion，per-token FFN 则表现出 rank-contractive 行为。它不是"训练初期上升、中期振荡、后期下降"的训练过程曲线，阻尼也与优化器动量无关。
- **Randomized Permutation Splitting 对照的是语义分组**：论文的对照基线是按语义把特征归组（semantic grouping），那样做会把共线的、长尾的特征聚到同一个 Token，形成低秩瓶颈。随机置换算子 σ 打乱稀疏特征索引后，高相关特征被分散到不同 Token，论文用互信息差异矩阵（M_Randomized − M_Semantic）验证了 Token 间冗余下降，且该结论在 K=48 与 K=64 两种聚类数下一致。
- **Multi-embedding 扩的是初始自由度**：单个 Embedding 表 ψ:F→R^d 只给一组映射，RankUp 用 K 个独立表 ψ_1..ψ_K，让同一稀疏特征被投影到多个子空间，直接抬高初始表征矩阵 H_0 的自由度，缓解早期阶段的表征坍缩。论文的落点是"输入侧自由度"而非 ensemble 式的预测多样性。
- **五机制的互补性**：机制 1-2 作用于输入侧（降相关、扩自由度），机制 3-4 注入全局上下文与跨域先验，机制 5 缓解多目标梯度干扰。论文的消融显示各机制对 Realtime AUC 均有独立增益，其中任务专属 token 对深层秩保持的作用最明显。论文并未断言五者"缺一不可"。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| GMV（视频号/公众号/朋友圈） | +3.41% / +4.81% / +2.12% | 微信广告三场景全量部署 |
| Order Task GMV | +7.18% | 微信广告 |
| 新广告冷启动（公众号） GMV | +9.67% | 微信广告 |

> 注：朋友圈 GMV 一项在 arXiv 不同版本中分别记作 +2.12% 与 +2.21%，引用时请以所引版本为准。论文主实验为微信视频号广告的 CVR 预估（日均 2000 万样本、1200+ 稀疏特征、32 个优化目标）。

**对 Scaling 的意义**：RankUp 证明了表征健康是 Scaling 的隐性前提——模型扩大但如果有效秩下降（表征坍缩），增加的参数只是拟合噪声。五机制确保 Scaling 过程中表征空间不坍缩。

---

### 25. RankElastor — 参数化全混合与 GLU 改进

**论文信息**：arXiv:2605.23191, Tencent + HKUST-GZ, KDD 2026

**做了什么**

RankElastor 解决 RankMixer 嵌入坍缩问题。通过实证分析和理论洞察，识别 rigid token mixing 和 P-FFN 模块为阻尼振荡轨迹的联合成因。两组件：(1) Parameterized Full Mixing——将 block-mixing 推广为 `vec(M^T) = LN((W+I)vec(X^T))`，W∈R^{TD×TD} 可学习混合矩阵，对应 grid scale d=1 最细粒度，Theorem 3.1 证明比 block-mixing 更强表达力，移除 Kronecker 约束 `(W⊗I_{d*})` 对表达细粒度高秩 coordinate interactions 的限制；(2) GLU-improved P-FFNs——GLU 风格 FFN 模块稳定表示频谱。代码开源 https://github.com/vasile-paskardlgm/RankElastor。

**为什么这么做**

RankMixer 的 parameter-free mixing 虽然高效，但固定排列模式无法适应不同特征组合的交互需求——某些特征需要"强混合"（全连接），某些需要"弱混合"（局部），rigid mixing 一视同仁导致表征频谱退化。RankElastor 通过参数化 mixing 和 GLU FFN 恢复表征频谱的多样性。

**原始论文深度新见解**

- **Parameterized Full Mixing 的理论推广**：RankMixer 的 block-mixing 是 `vec(M^T) = LN(W_block ⊗ I_{d*})vec(X^T)`，Kronecker 约束 `⊗ I_{d*}` 限制了 mixing 的细粒度——block 内 Token 只做恒等映射。RankElastor 移除此约束，W 直接作用于 TD×TD 全空间，grid scale d=1 意味着每个 Token 的每个维度都可独立混合——这是"最细粒度全混合"。
- **Theorem 3.1 的表达力证明**：Theorem 3.1 证明 Parameterized Full Mixing 的表达力严格强于 block-mixing——block-mixing 是 Full Mixing 的特例（当 W 的 block 对角部分非零、off-block 部分为零时退化为 block-mixing）。Full Mixing 可以表达 block-mixing 无法表达的高阶 coordinate interactions。
- **GLU-improved P-FFN 的频谱稳定**：标准 FFN 的 ReLU/GELU 激活可能导致频谱偏移（高频成分被抑制）。GLU（Gated Linear Unit）的门控形式为 `FFN(x) = (W1 x) ⊙ σ(W2 x)`，⊙ 为逐元素乘（注意与本篇 mixing 部分的 Kronecker 积 ⊗ 区分）。门控让不同频率成分经由可调通路通过，从而抑制频谱偏移、稳定有效秩。需要说明的是，"高频/低频成分走不同门控通路"是对该机制的直观解释，论文的论证落在表示频谱的稳定性上。
- **与 RankUp 的互补关系**：RankUp 从特征组织（输入层）解决有效秩问题，RankElastor 从 mixing 参数化（中间层）解决。两者攻击不同环节——RankUp 防止输入层特征相关性导致坍缩，RankElastor 防止中间层 mixing 模式刚性导致坍缩。

**工业验证数据**

| 指标 | 数值 | 场景 |
|------|------|------|
| 代码开源 | https://github.com/vasile-paskardlgm/RankElastor | 离线实验 |

> 注：论文以理论分析和离线实验为主，未公布线上业务效果指标。

**对 Scaling 的意义**：RankElastor 指出 parameter-free mixing 虽然高效但刚性，Parameterized Full Mixing 在恢复表达力的同时保持可接受效率。需要注意两点限定：一是"参数化是表征健康的必要条件"这一说法过强，论文证明的是表达力的严格包含关系（Theorem 3.1），不是表征健康的必要性；二是该工作以理论分析和离线实验为主，尚无线上业务验证，因此与 RankUp 并置时不宜称为"完整解决方案"。

---

# 第二部分：八个补充维度

## 维度一：跨论文对比分析

### 1. 统一交互 Block 三方对比

| 维度 | RankMixer | TokenMixer-Large | UniMixer |
|------|-----------|-------------------|----------|
| Mixing 机制 | Parameter-free（splitting/shuffling/projection） | Parameter-free + Mixing-and-Reverting | Parameterized（块间 W_G × 块内 W_B^i，受双随机/稀疏/对称约束） |
| 参数规模 | ~100M~1B | 7B/15B | 论文报告区间 38M~140M |
| MoE 支持 | ReLU routing Sparse-MoE | Sparse Per-token MoE（1:2） | 无（Lite 版走基组合 + 低秩近似） |
| 残差处理 | 标准残差 | Mixing-and-Reverting | SiameseNorm 双耦合流（引自外部工作，非本文提出） |
| MFU | 45% | 高（延续 RankMixer） | 论文未报告 |
| Scaling 指数（Params） | 0.116 | 待验证 | 0.132（本体）/ 0.142（Lite） |
| 核心贡献 | 奠基：证明无注意力可行 | 扩展：7B/15B 验证 | 统一：三大范式理论框架 |

**对比分析**：RankMixer 是奠基之作，证明 parameter-free mixing 可行且高效；TokenMixer-Large 验证该范式可扩展到 7B+，Mixing-and-Reverting 是深层扩展的关键；UniMixer 从理论层面统一 Attention/TokenMixer/FM，提供架构 Scaling 的连续插值框架。三者形成"奠基→扩展→统一"的递进关系。

### 2. 注意力策略六方对比

| 方法 | 复杂度 | 机制 | 远期信息 | 适用场景 |
|------|--------|------|----------|----------|
| Standard Attention | O(L²) | 全序列 QKV | 完整保留 | 短序列（<16k） |
| SLA（ULTRA-HSTU） | O((K1+K2)·L) | 局部窗口 K1 + 全局 prefix 中 K2 个真实 token（不做压缩） | 由 K2 锚定的 prefix token 承载 | 中长序列（16k~100k） |
| LONGER 混合注意力 | token merge 后 O((L/K)²) | 1 层 cross causal（global token + 最近 k 个作 query）+ N 层 self causal | Global token 作 query 锚点携带 | 端到端 10,000 量级 |
| CSA（EST） | O(L·k) | 内容相似性 top-k | 高信号行为保留 | 稀疏行为序列 |
| STCA（Make It Long） | O(L·d) | 单 Token 交叉注意力降维 | 逐 Token 交互保留 | 端到端 10K 长序列 |
| Query Decoding（HyFormer） | 取决于所选序列编码器（三档可插拔） | 非序列特征扩展成的 query 经 cross-attention 读取逐层 KV，与 Query Boosting 交替迭代 | 由 query 携带的全局上下文决定 | 长序列与异构特征需双向交互时 |
| RankMixer Mixing | O(L) | 无注意力 | 固定模式 mixing | 效率优先场景 |

**对比分析**：注意力策略沿"全量→结构化稀疏→内容稀疏→无注意力"的谱系演进。SLA 与 LONGER 都属结构化稀疏但手段不同——SLA 改 mask 形状（局部窗口加全局 prefix，不压缩序列），LONGER 改序列长度（先 token merge 压缩，再用少量 query 跨注意）；CSA 属内容稀疏，按与候选的内容相似度动态选 token；STCA 换问题定义，用候选查询序列把序列内部的 O(L²) 自注意力整个去掉；RankMixer 则彻底不算注意力权重。选择取决于序列长度、候选数与延迟预算。需注意 ULTRA-HSTU 明确反对"用 cross-attention 绕开 self-attention"，与 STCA、LONGER 的路线存在立场分歧，不是同一谱系上的渐进关系。HyFormer 则不在这条谱系上选边——它把序列编码器做成三档可插拔（Full Transformer / LONGER 式 / 无注意力轻量），把自己的贡献放在 query 侧的交替迭代，等于承认"序列怎么编码"是可替换的成本旋钮。

### 3. 推理优化策略对比

| 方法 | 优化层面 | 核心机制 | 复杂度变化 |
|------|----------|----------|------------|
| UG-Sep（Compute Only Once） | 架构 masking | 单向阻断 G→U + 分离式残差 + Information Compensation（比例失衡时才需要） | 用户侧 O(C) → O(1)，ΔAUC −0.004%~−0.033% |
| MixFormer user-item decoupling | 架构设计 | Query Mixer + Cross Attention 分解 | 用户侧 O(1) |
| Cross-Request KV Caching（OneTrans） | 缓存复用 | 跨请求 KV 对复用 | 运行时间 -30%，显存 -50% |
| Lazy Decoder-Only（OneRec V2） | 上下文编码开销 | 去掉 Encoder，上下文作静态条件经 Lazy Cross-Attention 注入，层间共享 KV + GQA | 总计算量 −94%，训练资源 −90% |
| LazyAR（GR4AD） | 自回归层间依赖 | 前 K 层并行、第 K 层才注入前序 token，辅以 MTP 损失 | 吞吐 +117%（L=9, K=6） |
| CompSkip（Kunlun） | 计算跳过 | 按层交替跳过 self-attention 或 HSP/PFFN 组件 | FLOPs −43.1%，QPS +35% |
| RLB（Make It Long） | 请求级批量 | 同请求内候选共享用户侧计算 | 用户侧 O(1) |

**对比分析**：推理优化有两条路径——架构层面（UG-Sep、MixFormer、RLB 从设计上分离用户侧和物品侧，CompSkip 按层裁剪组件）和运行时层面（KV Caching、Lazy 在执行时复用或延迟计算）。架构层面的优化更彻底但需要重新设计模型，运行时层面的优化可叠加到现有模型上。RLB 与 UG-Sep 独立收敛到相同的"用户侧复用"洞察，但 RLB 作用于请求内批量、UG-Sep 作用于跨候选 masking，两者可叠加。

一个贯穿多篇的共同前提是：**用户侧计算之所以可复用，仅因为它在一次请求内对不同候选保持不变**。因此这类方法的实现要点都是单向阻断物品侧到用户侧的信息流，而非双向隔离——双向隔离会退化成双塔，dense 交互的表达力也就没了。另需注意口径差异：表中的复杂度与降幅分属不同层级，UG-Sep 的 O(1) 是用户侧算子级、CompSkip 的 −43.1% 是 FLOPs、KV Caching 的 −30% 是运行时间，彼此不可直接相加或横向比较。这类优化也几乎都不是严格无损，UG-Sep 明确以 0.01%~0.03% 量级的 AUC 下降换取两位数延迟收益。

### 4. Token 化策略对比

| 方法 | Token 化对象 | 统一方式 | 场景扩展 |
|------|-------------|----------|----------|
| OneTrans | 所有场景特征 | Auto-Split Tokenizer + Mixed Param | 跨场景统一 |
| MDL | 场景+任务信息 | Prompt Token 注入 | 多分布统一 |
| OneRec SID | 视频多模态内容 + 用户行为协同信号 | RQ-Kmeans 残差量化为 3 层语义 ID | 生成式统一 |
| GR4AD UA-SID | 广告多模态内容 + 非语义业务特征 | 指令微调 MLLM 出 UAE，再 MGMR RQ-Kmeans；末级换为业务特征哈希 | 降碰撞、区分同内容不同投放 |
| LoopFM | FM 历史中间层嵌入 | AutoEncoder 降维 + 量化（如 INT4） | FM 知识统一 |
| TokenFormer | 静态特征+行为序列 | BFTS 分层 Token 化 | 异构 Token 分层管理 |

**对比分析**：Token 化策略的演进方向是"统一更多类型的信息"——OneTrans 统一多场景特征，MDL 统一多分布信息，OneRec/GR4AD 统一多模态内容，LoopFM 统一 FM 知识，TokenFormer 统一异构 Token 但通过分层管理避免污染。生成式这一侧还多出一层约束：语义 ID 既要保住内容语义的层次性，又要避免碰撞，GR4AD 把末级量化换成业务特征哈希就是为此付的代价——它牺牲了最后一级的语义性来换取可区分度。Token 化是 Scaling 的前提——只有统一到同一 Token 空间才能在同一架构内 Scaling，但 TokenFormer 提示统一后必须有显式的层次管理机制。

---

## 维度二：瓶颈依赖链（八层因果链）

推荐系统 Scaling 的八层瓶颈形成因果依赖链，上游瓶颈的解决是下游瓶颈可解的前提：

```
第1层：特征交叉容量不足
  └→ 第2层：显存墙（模型放不下）
       └→ 第3层：通信瓶颈（多卡训练时通信占用计算）
            └→ 第4层：推理 O(B)（候选集大小影响延迟）
                 └→ 第5层：算子级效率（MFU 低）
                      └→ 第6层：序列长度限制（O(L²) 爆炸）
                           └→ 第7层：压缩一致性（量化/压缩后效果退化）
                                └→ 第8层：表征健康（有效秩坍缩）
```

**注意**：上述八层是概念上的依赖递进关系，实际工程中各层之间不一定严格串行。例如，表征健康问题（第 8 层）可以独立于压缩一致性（第 7 层）发生——RankUp 论文讨论的有效秩坍缩发生在未量化的场景中。此链条的意义在于梳理"上游瓶颈解决后下游瓶颈才凸显"的逻辑递进，而非暗示各层只能按顺序出现。

**逐层解析**：

- **第1层→第2层**：特征交叉容量不足（DCN/DeepFM 交叉深度有限）→ 需要更大模型（统一交互 Block 或全注意力交叉）→ 模型放不下（显存墙）
- **第2层→第3层**：显存不够 → 多卡分布式训练 → 通信量随卡数线性增长 → 通信占用 GPU SM
- **第3层→第4层**：通信瓶颈解决后训练可 Scaling → 但推理时每候选独立计算 → O(B) 延迟
- **第4层→第5层**：推理 O(B) 通过 UG-Sep 降为 O(1) → 但单次计算 MFU 低 → 算子级效率不足
- **第5层→第6层**：MFU 提升后计算变快 → 但序列长度 O(L²) 仍限制 → 长序列瓶颈
- **第6层→第7层**：长序列通过 SLA/LONGER 解决 → 但模型变大后需量化压缩 → 压缩一致性
- **第7层→第8层**：压缩后效果退化 → 这时容易误判为"压缩损失"而非表征问题 → 有效秩坍缩作为独立风险需要显式监测（RankUp 论文中的坍缩发生在未量化模型上）

**每层对应的论文**：第1层（MixFormer Co-Scaling、UniMixer 参数化交互）、第2层（TokenMixer-Large 的 Sparse MoE 把 15B 压回可服务规模）、第3层（FreeScale 的 SM-Free 通信与嵌入通信优先级）、第4层（Compute Only Once 的 UG-Sep、Make It Long 的 RLB）、第5层（RankMixer MFU 4.5%→45%、Kunlun GDPA）、第6层（SLA、LONGER、Make It Long 的 STCA）、第7层（UG-Sep 叠加的 W8A16、LoopFM 的 AE+INT4 压缩）、第8层（RankUp/RankElastor 表征健康）。

> 注：第 2 层与第 3 层此前都记作 FreeScale，实为重复——FreeScale 解决的是通信与负载均衡（第 3 层），显存侧的代表性工作应为 TokenMixer-Large 的稀疏化。

---

## 维度三：工程实践映射

### 论文 → 工程能力映射

| 论文 | 工程能力 | 可复用组件 |
|------|----------|------------|
| RankMixer | Parameter-free Token Mixing | reshape/shuffle 算子 + Per-Token FFN |
| TokenMixer-Large | 大规模 Mixing 训练 | Mixing-and-Reverting + Sparse MoE |
| SLA | 长序列注意力 | 双窗口 K1+K2 attention kernel |
| LONGER | 超长序列训练 | Global token + token merge |
| UG-Sep | 推理解耦 | Masking + Information Compensation |
| FreeScale | 分布式训练 | SM-Free 通信 (CPU-RDMA) |
| Design Once | 模型工程 | SMT 标准化模板接口 |
| CAdam | 在线优化 | Confidence-based 更新策略 |
| LoopFM | FM 知识迁移 | AutoEncoder + 量化压缩管线（Extraction/Compression/Structuring 三阶段） |
| OneRec | 生成式推荐 | RQ-Kmeans SID + Encoder-Decoder（V1）/ Lazy Decoder-Only（V2）|
| TokenFormer | 分层 Token 化 | BFTS 分层注意力 + NLIR gate |
| Make It Long | 端到端长序列 | STCA 单 Token 交叉注意力 + RLB 请求级批量 |

### 部署成熟度评估

分档依据统一为论文自报的落地状态，而非贡献类型：

| 成熟度 | 论文 | 依据 |
|--------|------|------|
| 全量部署（论文明示全量/多场景上线） | RankMixer, Kunlun, ULTRA-HSTU, OneTrans, SORT, HeMix, LONGER（10+ 场景）, MDL（日服务数亿用户）, RankUp（微信广告三场景）, Compute Only Once（四场景）, GR4AD（4 亿+用户）, OneRec（本地生活全量，主 Feed 承接约 25% QPS）, CAdam（16 场景运行 9 个月+）, Design Once（Meta 广告排序生态四个开发周期） | 论文报告线上 A/B 且已全量或多场景铺开 |
| 生产验证（有线上结果，规模或范围未明示全量） | TokenMixer-Large, MixFormer, TokenFormer, LLaTTE, LoopFM, UniMixer, EST | 论文报告线上 A/B 或生产环境结果 |
| 大规模实验 | FreeScale（256 H100）, Make It Long | 大规模系统实验，论文未报告业务指标 |
| 理论/框架贡献为主 | RankElastor | 以理论分析与离线实验为主 |

> 注：上表按论文自报的落地状态排列，而非按贡献类型。几处易误判的：TokenFormer 有微信视频号广告 A/B（GMV +4.03%），LLaTTE 是 Meta 旗舰广告排序模型的生产结论，LONGER 为字节 10+ 场景全量部署，三者均非"理论框架"。

---

## 维度四：前瞻性分析

### 1. 生成式推荐将重构推荐系统架构

OneRec 和 GR4AD 证明了生成式推荐在工业规模下可行——一个承接快手主站约 25% QPS，一个在 4 亿+用户的广告系统全量部署。若这一方向延续，影响会体现在三处：
- 候选集定义不再是瓶颈（生成阶段不再受上游检索漏斗约束）
- 推荐系统架构与 LLM 架构靠拢（自回归 + Next Token Prediction；具体形态仍在演化，OneRec V1 是 Encoder-Decoder，V2 才转为 Lazy Decoder-Only）
- 推荐系统可直接复用 LLM 基础设施（训练框架、推理优化、硬件设计）

但"全面转向"这个判断需要打折看待，两篇论文自己给出了三条阻力：V2 自述 loss 下降并不完全符合 scaling law；V1 明确指出推理阶段 step scaling 效果不明显，即尚不具备强推理能力；GR4AD 则指出生成式架构的原始算力需求仍高于精简 MLP，其参数规模上限（0.32B）受服务预算而非算法约束。判别式链路在延迟敏感场景短期内不会消失。

### 2. 统一 Token 空间是 Scaling 的终极形态

OneTrans（多场景统一）、MDL（多分布统一）、MixFormer（序列+特征统一）、LoopFM（FM 知识统一）都指向同一方向——将更多类型的信息统一到同一 Token 空间。未来可能出现"全统一 Token 空间"：场景+任务+行为+特征+FM 知识+多模态内容全部 Token 化，在单一 Transformer 内联合建模。

### 3. 表征健康成为 Scaling 的隐性约束

RankUp 和 RankElastor 揭示了 Scaling 的隐性约束——模型增大但表征坍缩则无效。未来"表征健康监测"可能成为标准训练流程的一部分，类似 loss curve 监控。有效秩、频谱分布等指标将纳入模型评估体系。

### 4. 训练-推理解耦使 Scaling 不受部署约束

UG-Sep（用户侧 O(C)→O(1)）、LONGER（用户序列 KV 预计算后跨候选复用）、LoopFM（FM 离线产出表征、VM 在线消费历史快照）共同指向同一范式：把稳定且昂贵的那部分计算移出在线关键路径。三者的解耦程度并不相同——UG-Sep 与 LONGER 仍在同一模型内，只是复用请求内不变的中间结果；LoopFM 才是真正的跨模型解耦，让离线 FM 的规模上限只受离线成本约束。需要说明的是，"训练 100B、推理 1B 压缩版"是对该趋势的外推，本文档收录的论文中并无这一量级的实例。

### 5. 工程 Scaling 与模型 Scaling 同等重要

Design Once（O(n·2^k)→O(n+k)）和 FreeScale（SM-Free 通信）证明，工程效率的 Scaling 与模型参数的 Scaling 同等重要。如果新技术传播太慢或训练集群效率太低，模型 Scaling 的成果无法落地。

---

## 维度五：GEMM 统一度与 Roofline 量化分析

### Roofline 模型基础

Roofline 模型通过"算术强度"（Arithmetic Intensity，AI = FLOPs / Bytes）判断计算是 memory-bound 还是 compute-bound：

- **AI < 拐点**：memory-bound（带宽限制，GPU 计算单元空闲）
- **AI > 拐点**：compute-bound（算力限制，带宽充分利用）
- **拐点位置**：AI_拐点 = 峰值算力 / 峰值带宽

### 推荐模型的 GEMM 统一度

推荐模型的计算可分解为多个 GEMM（矩阵乘法）操作。GEMM 统一度指"多大比例的计算可表示为大规模连续 GEMM"：

| 架构 | GEMM 统一度 | 相对算术强度 | 瓶颈类型 | MFU |
|------|------------|----------|----------|-----|
| Standard Attention | 低（KV cache 不连续访问） | 低 | memory-bound | 4.5% |
| RankMixer | 高（reshape/shuffle 连续 + 大 GEMM） | 高 | 偏 compute-bound | 45% |
| SLA | 中（双窗口部分连续） | 中 | 混合 | 15-25% |
| TokenMixer-Large | 高（延续 RankMixer） | 高 | 偏 compute-bound | 40%+ |

**分析**：本表只做相对比较，不给绝对算术强度数值。原因是 Roofline 拐点随硬件差异很大——A100（BF16）的拐点约在 150 以上，H100 更高，因此单看一个绝对数字无法判断落在拐点哪一侧，跨架构套用同一组数字更容易误导。

RankMixer 的 MFU 十倍提升根源在于 GEMM 统一度：parameter-free mixing 的 reshape/shuffle 全是连续内存操作，后续 Per-Token FFN 则是规整的大 GEMM，整体算术强度从不规则访存主导的 memory-bound 区显著上移，落到更接近峰值利用率的区间。需要说明的是，MFU 从 4.5% 到 45% 是论文报告的实测值，而"跨越 Roofline 拐点"是对该结果的机制解释，不是论文给出的 Roofline 测量。

### Kunlun 的 Roofline 优化

Kunlun 的 GDPA + HSP + CompSkip 三层优化对应 Roofline 的三个改善方向：
- GDPA 把 PFFN 改写为点积注意力并融合成单 kernel → 消除大量小而不规则的矩阵乘 → 提升 GEMM 统一度（PFFN 模块 MFU 最高 6 倍）
- HSP 分层池化 → 减少输入长度 → 降低 memory-bound 区的计算量
- CompSkip 按层跳过组件 → 减少总 FLOPs 而非提升单算子强度 → FLOPs −43.1%、QPS +35%

---

## 维度六："三层前提"框架

推荐系统 Scaling 需要同时满足三层前提，任一层成为短板都会让边际收益急剧递减：

### 架构前提

模型架构必须支持高效 Scaling——parameter-free mixing（RankMixer）、可学习的参数化混合（UniMixer）、统一 Token 空间（OneTrans/MDL）、长序列线性复杂度（SLA/LONGER）、推理 O(1)（UG-Sep）。没有架构前提，增加参数和序列长度只会遇到计算墙和延迟墙。

### 数据前提

数据质量和多样性必须支撑 Scaling——LLaTTE 的 Scaling Law 要求"足够多样"的数据，两阶段 Scaling 的转移率 τ≈50% 要求 upstream 数据与 online 数据有分布重叠。LoopFM 的 FM 知识迁移要求 FM 训练数据与推荐场景相关。没有数据前提，模型扩大只拟合噪声。

### 表征前提

表征空间必须健康——RankUp 的有效秩阻尼振荡、RankElastor 的嵌入坍缩都表明，如果表征空间在 Scaling 过程中坍缩，增加的参数只是拟合噪声。表征前提要求：(1) 输入层特征去相关（RankUp 的 Randomized Permutation Splitting）、(2) 中间层混合模式不僵化（RankElastor 的 Parameterized Full Mixing 与 GLU-improved P-FFN）、(3) 输出层多目标解耦（RankUp 的多目标相关设计）。TokenFormer 揭示的异构 token 秩污染属于同一层前提在 Token 化环节的表现。

> 归属说明：UniMixer 不属于"表征前提"这一档。它的参数化动机是统一 attention / TokenMixer / FM 三大范式，让写死的排列规则变为可学习，从而在同一框架内连续插值——出发点是推荐特征空间异构、异构空间之间算内积缺乏物理意义，属于架构前提。论文中并未讨论有效秩、频谱退化或嵌入坍缩。RankElastor 才是以表征健康为直接目标的工作。

### 三层前提的依赖关系

```
架构前提（能否高效计算）
  ∧ 数据前提（是否有足够信息）
      ∧ 表征前提（信息是否被有效编码）
          → 有效 Scaling
```

三层前提近似 AND 关系——任一层成为短板，Scaling 的边际收益就急剧递减。严格说它不是二元的开关：数据前提部分不满足时模型仍可能有小幅收益，只是投入产出比迅速恶化。这也解释了为什么有些模型扩大后效果不提升——不是参数不够，而是某一层前提成了瓶颈。

---

## 维度七：主线间交叉引用关系

以下按主线分组梳理各主线与其他主线的关键交叉点：

**主线一（Token 化）与其他主线**
- 与主线二：MDL 以 RankMixer 为 backbone；UniMixer 统一框架包含 FM 范式
- 与主线三：OneTrans 的 KV caching 天然支持长序列
- 与主线四：OneTrans 的 Cross-Request KV 复用是推理解耦的前置
- 与主线六：OneRec 的语义 ID（SID）本身就是 Token 化
- 与主线七：TokenFormer 的 Sequential Collapse Propagation 是表征坍缩在 Token 化层的具体表现

**主线二（统一交互 Block）与其他主线**
- 与主线三：SLA 双窗口注意力是交互策略的长序列变体；HyFormer 直接以这条缝为论点——它批评"LONGER 压缩（主线三）+ RankMixer 融合（主线二）"的组合式接法，主张两层合并为逐层交替，是本文档中唯一一篇把主线二与主线三的**衔接方式**本身当作问题来做的工作
- 与主线四：MixFormer 通过在 HeadMixing 中屏蔽 user→item 信息流，使 user 侧计算可在 RLB 中跨候选复用
- 与主线五：RankMixer 的高 MFU 依赖连续 GEMM 算子优化
- 与主线六：生成式的 Decoder 同样需要在单 Block 内混合上下文与目标信息，但走的是 cross-attention 而非 Token Mixing 算子
- 与主线七：RankElastor 直接改进 RankMixer 的表征健康

**主线三（长序列）与其他主线**
- 与主线二：HyFormer 认为长序列压缩不应是独立前置阶段——序列过早压缩会让后续特征交互失去反向影响序列建模的机会，故把压缩与融合改为逐层交替；这与主线三其余三篇"先压好再交出去"的隐含前提直接冲突
- 与主线四：LONGER 的 KV cache serving 把用户序列 KV 预计算后跨候选复用；Make It Long 的 RLB 与 UG-Sep 独立收敛到请求级复用
- 与主线五：长序列训练依赖 FreeScale 的分布式通信优化
- 与主线七：长序列增大了有效秩坍缩的风险面

**主线四（推理解耦）与其他主线**
- 与主线五：推理优化依赖底层算子效率；部署标准化依赖 Design Once
- 与主线六：OneRec V2 的 Lazy Cross-Attention 与 GR4AD 的 LazyAR 同属"把不产生边际收益的算力挪走"，但优化的是训练/生成侧的算力分配，与 UG-Sep 的用户侧请求级复用不是一回事
- 与主线七：U/G 分离可能损失信息，需 Information Compensation 保障表征质量

**主线五（底层基建）与其他主线**
- 与主线六：生成式推荐的 MFU 提升首先来自架构规整（算子数 15000+ → 约 1200），其上才是 SKAI/UGMMU 这类 Embedding 训练优化与算子融合
- 与主线七：CAdam 改善的是在线学习中梯度方向的准确性（过时动量 / 噪声鲁棒），不直接涉及有效秩或表征坍缩，但优化方向更准可间接减缓训练中的表征退化

**主线六（生成式推荐）与主线七（表征健康）**
- 生成式的自回归架构天然产生更长序列，表征坍缩风险更高；但语义 ID 的层次编码也可能为表征多样性提供新的保障

---

## 维度八：从原始论文提取的 14 个深度新见解

> 以下 14 个见解是对第一部分各论文"原始论文深度新见解"的提炼与跨论文视角的再组织。部分内容与第一部分有交叉，此处侧重跨论文对比和 Scaling 全局意义。

### 见解 1：RankMixer 的"路由与变换分离"原则

**新认知**：RankMixer 的关键不在于"省掉参数"，而在于把"路由"（谁和谁交互）与"变换"（交互后如何映射）拆开——路由由与输入内容无关的固定重排承担，变换交给 Per-Token FFN。这样 attention 里两个 memory-bound 的 activation-to-activation GEMM（QK^T 与 A·V）被彻底移除，剩下的都是权重参与的规整 GEMM，容量增长与访存代价因此解耦。

**对 Scaling 的意义**：路由与变换分离是 O(L) 复杂度的架构基础——固定路由不需要计算注意力权重，计算量与序列长度线性相关。

### 见解 2：UniMixer 的"连续插值"统一框架

**新认知**：UniMixer 不是"又一种 mixing"，而是指出 Attention 与 TokenMixer 处在同一参数化空间的两端，区别在于**约束的松紧**而非结构本身。论文的归约是具体的：`W_G` 的维度和角色与注意力权重一致，只是额外要求双随机、稀疏、对称；把这三条约束全部松掉，`W_G` 就退化为普通的注意力权重；把它们收到极致（每行每列恰一个非零元且固定不学），就退回原版 TokenMixer 的固定排列。FM 一侧则对应注意力中 `W_Q = W_K = I` 的特例。

**对 Scaling 的意义**：既然差别只是约束强度，那么"选架构"这件事就从离散决策变成了可调超参——用温度系数控制稀疏度、用 Sinkhorn-Knopp 控制双随机性，就能在效率端与表达力端之间移动。论文的实证结论是这个中间态比两个端点都好：受约束的可学排列（UniMixer-Lite）在参数效率和缩放指数上同时超过 RankMixer 与无约束基线。需要提醒的是，"连续插值"是理论框架层面的性质，论文并没有做"训练过程中动态从 TokenMixer 端滑向 Attention 端"这类实验。

### 见解 3：MDL 的"Prompt Token 作为分布标识"范式

**新认知**：MDL 将 LLM 的 Prompt 范式映射到推荐——LLM Prompt 是"任务指令"，MDL Prompt Token 是"分布标识"。这不是简单搬运，而是语义层面的迁移：LLM 通过 Prompt 告诉模型"做什么任务"，MDL 通过 Prompt Token 告诉模型"当前来自什么分布"。

**对 Scaling 的意义**：Prompt Token 范式使场景/任务 Scaling 从架构层（加 tower）降为输入层（加 Token），Scaling 代价从 O(场景×任务) 降为 O(场景+任务)。

### 见解 4：LLaTTE 的"序列长度最陡峭"发现

**新认知**：LLaTTE 拟合出的对数线性斜率显示，序列长度是最陡峭的 Scaling 维度——按总 FLOPs 计 α=0.265，明显高于深度 0.200 和宽度 0.133。这意味着在固定算力预算下，投资长序列的 ROI 最高。

**对 Scaling 的意义**：这直接指导 Scaling 投资决策——优先扩展序列长度，其次宽度，最后深度。也解释了为什么 SLA/LONGER 等长序列论文的工业价值巨大。

### 见解 5：TokenMixer-Large 的"先 Enlarge 后 Sparse"与 Sparse Train Sparse Infer

**新认知**：TokenMixer-Large 把 RankMixer 的"Dense Train, Sparse Infer"升级为"Sparse Train, Sparse Infer"——先把 dense 模型扩到目标规模，再拆 SwiGLU 为多专家做稀疏化，训练期稀疏度已固定，从而同时省训练和推理。核心不是"避免冷启动"，而是 DTSI 只省推理不省训练这个瓶颈必须打掉。配套设计包括 Gate-Value Scaling（保证被激活专家获得足够梯度）、Down-Matrix Small Init（初始化时近似恒等映射）和常驻 shared expert。

**对 Scaling 的意义**：这为 7B+ 推荐模型的训练提供了可操作路径——训练与推理同时稀疏，训练成本不再随总参数量线性增长。

### 见解 6：MixFormer 的"Co-Scaling"打破模块分离

**新认知**：传统推荐把序列建模和特征交互分成两个独立参数化的模块，在算力预算固定时必须在两者间做次优的容量分配。MixFormer 的思路是让同一组 per-head SwiGLU FFN 同时服务序列与非序列信息，从而不必划分专用参数。需要留意的是，"统一参数化"被第三方评审指出有夸大——稠密与序列路径之间并未真正共享权重，统一更多体现在架构组织层面。

**对 Scaling 的意义**：Co-Scaling 消除了"序列模块和特征模块各自扩大但交互不足"的问题，使 Scaling 不受模块边界的约束。

### 见解 7：SLA 双窗口的"近局部、远全局"分层策略

**新认知**：SLA 的 K1+K2 双窗口不是"近精确、远摘要"——K2 选中的是全局 prefix 中真实存在的 token，不做压缩或低秩近似，被选中位置的 attention 计算与 K1 窗口内完全一致。两个窗口的区别在于感受野而非精度：K1 覆盖近期行为的局部模式，K2 锚定全局 prefix 中的长期兴趣。论文还强调这区别于通用稀疏注意力——后者通常只做局部窗口，推荐场景需要同时捕获局部行为模式与全局用户偏好。

**对 Scaling 的意义**：结构化稀疏使序列长度 Scaling 不受 O(L²) 约束，同时 K2 保证了长期信息的完整保留——是"选择性关注"而非"信息压缩"。

### 见解 8：LoopFM 的"FM 知识净增量"分解

**新认知**：LoopFM 的迁移率下界分解为 `TR ≥ (I_temporal + I_cross − C_compression) / ΔNE_FM`——I_temporal 是时序历史信息、I_cross 是跨特征信息、C_compression 是压缩损失。这三项的结构揭示了 FM 知识迁移的"净增量"：不是 FM 全部知识都有用，必须扣除压缩引入的信息损失。

**对 Scaling 的意义**：净增量分解指导 FM 知识迁移的优化方向——增加 I_temporal 和 I_cross（FM 独有的时序和交叉知识），降低 C_compression（改善压缩质量），同时拉长序列（Theorem 5 保证信息增益非减）。

### 见解 9：Compute Only Once UG-Sep 首次实现 Dense 模型用户侧复用

**新认知**：用户侧计算复用在推荐系统中一直存在（如缓存用户 embedding），但只在"浅层"复用——只缓存最终用户表示。UG-Sep 首次在 dense 交互模型中实现"深层"复用——缓存多层 Transformer 的 U-tokens 输出，G-tokens 只做轻量级交互。这要求 U/G 严格隔离，是架构层面的创新而非简单缓存。

**对 Scaling 的意义**：UG-Sep 将推理复杂度从 O(B) 降为 O(1)，使候选集大小不再影响延迟。这是推理 Scaling 的关键——无论候选集多大，用户侧只算一次。

### 见解 10：FreeScale SM-Free 通信避免 GPU SM 争用

**新认知**：传统 GPU 通信（NCCL）使用 GPU SM 发起 RDMA，占用计算资源——通信时 GPU 无法全力计算。FreeScale 使用 CPU 发起 RDMA（D2H→RDMA→H2D），完全释放 GPU SM。这不是"优化通信"，而是"将通信从 GPU 卸载到 CPU"，实现真正的计算-通信重叠。

**对 Scaling 的意义**：SM-Free 通信是训练 Scaling 的系统前提——GPU 规模扩大时通信量线性增长，如果不解决 SM 争用，计算效率会随规模下降。

### 见解 11：Design Once 组合爆炸减少 O(n·2^k)→O(n+k)

**新认知**：推荐系统的工程瓶颈不只是计算效率，还有"技术传播效率"——每项新技术需要在每个场景适配，组合爆炸使传播成本随场景×技术指数增长。SMT 的标准化接口将指数降为线性，这是组织层面的 Scaling。

**对 Scaling 的意义**：技术传播效率是"工程 Scaling"的核心——如果新技术传播太慢，研究速度会超过落地速度，造成技术积压。SMT 确保研究成果能快速全场景部署。

### 见解 12：OneRec 判别式筛选→生成式创建的范式转变

**新认知**：OneRec 的范式转变不是"更好的排序模型"，而是"取消排序"——直接生成推荐列表，无需预定义候选集。这类似于 NLP 从 BERT（判别式）到 GPT（生成式）的转变。生成式推荐突破了候选集定义的限制——模型直接自回归生成目标 item 的语义 ID 序列，不再依赖上游检索交付的候选集。需要说明的是，OneRec 生成的是已有 item 的语义 ID，而非凭空创造新 item；"突破候选集"指的是取消了检索阶段的漏斗约束，不是扩充了物品库。

**对 Scaling 的意义**：架构与 LLM 靠拢之后，LLM 的训练框架、推理优化与硬件红利可以直接复用，这是 MFU 从个位数跃到 20%+ 的前提。但"遵循相似的 Scaling Law"要留余地：OneRec 在 0.015B→2.633B（V1）与 0.1B→8B（V2）区间观察到 loss 单调下降，V2 论文却明确说这条曲线并不完全符合 scaling law 的形式；推理阶段的 step scaling 也尚未显现。可以说的是生成式让推荐系统重新具备了被 Scaling 的形状，还不能说它已经复刻了 LLM 的 Scaling 规律。

### 见解 13：TokenFormer 的"Sequential Collapse Propagation"

**新认知**：值得单独拎出来的不是"统一要分层管理"这个结论，而是这篇论文把一个模糊的工程直觉转成了可测量的诊断量。此前"统一 backbone 效果不如精心设计的异构架构"只能靠离线指标反推，TokenFormer 用分层有效秩轨迹 + 奇异值谱 + 互信息三件套，同时给出了坍缩的位置（哪一层开始谱衰减变陡）、代价（有效秩下降幅度）与收益（MI 上升幅度），从而把取舍摊开在同一张图上。它还顺带给出了一个可直接查的信号：深层里序列 token 向非序列位置分配的注意力权重异常偏高，而这部分跨域注意力在深层并无实质收益——这既是坍缩的证据，也是"深层直接切断静态 token"这一处置的依据。

**对 Scaling 的意义**：这套诊断方法的价值超出 TokenFormer 本身，可以作为评估任何统一架构是否健康的通用工具——在决定"要不要把两类异构信号塞进同一个 backbone"之前先量一下谱，比事后看 AUC 更早发现问题。与主线七的呼应也因此更具体：RankUp/RankElastor 关注的是沿层深的有效秩振荡与嵌入坍缩，TokenFormer 关注的是异构 token 之间的秩污染，二者共用同一套度量语言，可以放在同一张诊断面板上。

> 提示：本节与主线一 TokenFormer 正文存在内容重叠，正文侧重架构与机制，本节侧重方法论层面的可复用性。

### 见解 14：Make It Long 的"RLB 跨论文普适性"——三方独立收敛

**新认知**：Make It Long 的 RLB（Request-Level Batch）将同一请求内 B 个候选的用户侧计算合并为一次，这与 Compute Only Once 的 UG-Sep、SORT 的请求中心范式形成"三方独立收敛"——三篇论文从不同角度（端到端长序列、Dense 模型解耦、生成式预训练）独立发现了相同的工程洞察：用户侧计算应在请求级复用。RLB 的独特之处在于它在端到端 10K 长序列的极端条件下验证了这一洞察的可行性，证明即使序列极长，用户侧复用仍是降延迟的关键。

**对 Scaling 的意义**：三方独立收敛说明"用户侧计算复用"不是某篇论文的特殊技巧，而是推荐系统推理 Scaling 的结构性需求。未来任何推荐架构的推理优化都应将此作为基线设计原则，而非可选优化。

---

## 附录：25 篇论文工业验证数据汇总表

| # | 论文 | 核心指标 | 数值 | 场景 |
|---|------|---------|------|------|
| 1 | OneTrans | Feeds GMV/Feeds订单/Mall GMV/Mall订单/冷启动 | +5.68%/+4.35%/+3.67%/+2.58%/+13.59% | 电商 |
| 2 | MDL | LT30/查询变更率 | +0.0626%/-0.3267% | 抖音搜索 |
| 3 | UniMixer | CAD（D1–D30 累积活跃天数）/AUC/缩放指数 | 多场景均值 +15% 以上 / 0.752718（84.5M，+0.8141pt vs RankMixer）/ 0.142（Lite）、0.132（本体） | 快手广告投放 |
| 4 | TokenFormer | 线上 GMV/线上 AUC/离线 AUC | +4.03% / +0.22% / +8.15‰（KuaiRand-27K, TokenFormer-L vs Transformer） | 微信视频号广告 |
| 5 | RankMixer | MFU/参数扩展/活跃天数/时长 | 4.5%→45%/100x(摘要)或70x(贡献)/+0.3%/+1.08% | 抖音信息流 |
| 6 | TokenMixer-Large | 订单/GMV/ADSS/直播营收 | +1.66%/+2.98%/+2.0%/+1.4% | 电商/广告/直播 |
| 7 | MixFormer | 在线A-B/推理加速 | 活跃天数与使用时长提升（具体值见原文）/UI-MixFormer 30%+ | 抖音+抖音极速版 |
| 8 | Kunlun | MFU/Topline/缩放效率/CompSkip | 17%→37%(B200)/+1.2%/2×/FLOPs−43.1% | Meta Ads(GEM) |
| 9 | ULTRA-HSTU | 训练/推理缩放效率/消费/topline | 5.3×/21.4×（缩放斜率比，非加速比）/+4%~8%/+0.217% | Meta 生产环境 |
| 10 | HyFormer | Query-AUC/FLOPs/消融 | 0.6489（vs LONGER+RankMixer 基线 0.6478，+0.74% 相对）/ 3.9×10¹²（基线 3.5×10¹²）/ query 仅目标特征 −0.34% | 抖音搜索；在线 A/B 论文仅定性表述 |
| 11 | EST | RPM/CTR | +3.27%/+1.22% | 淘宝展示广告 |
| 12 | HeMix | CTR-AUC/GMV/PV_CTR/UV_CVR | +1.64%(100M)→+2.34%(1.5B)/+3.61%/+2.78%/+2.12%（v2口径） | 高德 APP |
| 13 | SORT | MFU/orders/GMV/延迟/吞吐 | 22%→45% / +6.35%→+7.47% / +5.47%→+8.65% / −44.67%→−62% / +121.33%→+589%（v1v2→v3） | AliExpress |
| 14 | Make It Long | 序列长度/端到端延迟 | 10K/可控(STCA+RLB降复杂度) | 字节推荐 |
| 15 | LONGER | AUC/在线A-B/FLOPs/吞吐退化 | 0.85290(+1.57% vs base)/广告+1.06~2.10%、电商+4.6~7.9%/−42.8%/−40%→−6.8% | 字节广告与电商 |
| 16 | LLaTTE | CVR/NE/转移率 | +4.3%/-0.25%/≈50% | Meta 广告 |
| 17 | Compute Only Once | 线上延迟/训练加速/GEMM级(含W8A16) | −11.5%~−22.0%（四场景）/ +5.5%~+14.8% / −40%~−55% | 抖音·红果Feed/穿山甲·千川广告 |
| 18 | FreeScale | Exposed communication | −90.3%（相对 vanilla TorchRec） | 256 H100 训练，最大 UIH 21,000 |
| 19 | Design Once | 离线交叉熵/工程时间/传播吞吐/线上累计 | +0.63%（算力持平）/ −92% / 6.3× / +0.86% | Meta 广告排序，四个开发周期 |
| 20 | LoopFM | AUC/转化率/迁移率 | +6.4%（TaobaoAd）、+1.0%（KuaiVideo）、+0.5%（Amazon）/ +0.5%、+1.03%、+1.22%（三次上线）/ 约翻倍（未给绝对值） | 三个公开基准 / Meta 工业系统 |
| 21 | CAdam | GAUC/部署规模/GMV | +0.30%（3300亿参数，48h A/B）/ 16 场景运行 9 个月以上 / 论文称大幅提升未给数值 | 腾讯推荐 |
| 22 | OneRec | MFU(训练/推理)/OPEX/停留时长/GMV | 23.7% / 28.8% / 降至传统链路 10.6% / +0.54%(主站)、+1.24%(极速版) / +21.01%(本地生活) | 快手，V1；V2 计算量 −94% 并扩至 8B |
| 23 | GR4AD | 广告收入/中小广告主/吞吐 | 最高 +4.2% / +17.5% / LazyAR +117% QPS | 快手广告，4 亿+用户全量 |
| 24 | RankUp | GMV(视频号/公众号/朋友圈)/Order/冷启动 | +3.41%/4.81%/2.12%/+7.18%/+9.67% | 微信 |
| 25 | RankElastor | 开源代码 | github.com/vasile-paskardlgm/RankElastor | 离线实验 |

---

> **文档说明**：本文档覆盖 25 篇推荐系统 Scaling 核心论文。在原有三维度分析基础上从原始论文 arXiv 全文深入阅读，补充了工业验证数据、原始论文深度见解、跨论文对比分析、瓶颈依赖链、工程实践映射、前瞻性分析、GEMM/Roofline 量化、三层前提框架、主线交叉引用矩阵共八个补充维度。工业验证数据以原始论文的线上实验结果为准；跨论文对比与维度分析属本文档归纳，部分推断与归类已在对应段落标注。
