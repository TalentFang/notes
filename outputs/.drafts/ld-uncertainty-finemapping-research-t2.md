# 研究笔记 T2:分布自由推断(变量选择 / 遗传学)

关联计划:`ld-uncertainty-finemapping`;T2 调研 brief:`outputs/.plans/ld-uncertainty-finemapping-T2.md`
日期:2026-09-13

> **可信度标注约定(重要)**:本 T2 子代理会话未注册 web_search/source_check 工具,**未亲自打开任何网页**。以下内容分两类:
> - **【已检索】** = 来源 URL 出自主代理 lead 检索快照(`outputs/.drafts/ld-uncertainty-finemapping-research-lead.md`)的搜索结果(doi/PMC/arXiv/网页 URL 均为 lead 提供),本轮未逐字核读正文,仅凭 lead 摘要描述 + 领域常识理解;
> - **【未验证——需 verifier 核实】** = 领域知识回忆,本轮未检索到任何来源,**未给出(更未编造)URL 与具体数字**,细节(卷期页/年份/精确表述)需 verifier/lead 补搜核实。
> 所有 URL 均来自 lead 文件,未擅自添加任何新 URL。

## 现状摘要

1. 分布自由的高维变量选择在统计与遗传学中已是成熟方向:Model-X knockoffs【已检索,doi:10.1111/rssb.12265】在任意响应生成机制下提供有限样本 FDR 控制,不假设误差分布,并已被直接应用于 GWAS/遗传研究(多环境 knockoff、group knockoffs、spatial knockoff、HMM knockoff 系列【已检索】)。
2. 共形推断(conformal prediction)提供"分布自由 + 有限样本覆盖"的标准框架,且已有"选择条件覆盖"(selection-conditional coverage)文献【已检索,par.nsf.gov/10595785】明确指出:**边际覆盖保证在"选择后聚焦的单位"上可能失效**——这与 C1"保证真凶变异 ∈ 集合"的声明处于同一理论困境。
3. "使贝叶斯后验/可信集达到名义频率覆盖"的校准构造已有成熟先例:Syring & Martin 的 Monte Carlo 标量校准【已检索,Biometrika】与 bootstrap 覆盖校准【已检索,arXiv:2606.25729】在程序上与 C1 的"校准阈值 q̂"最接近。
4. 现有 fine-mapping credible set 在模拟中被发现**过度保守**(Wu et al., PLOS Comp Biol【已检索】),说明"达到 1−α"不是主要难点,集合的紧致性/功效才是;C1 声称的 1−α 保证需与这一实证结果对话。
5. PIP 本质是贝叶斯后验量,本身不是"分布自由"对象;把 PIP 经多面板/bootstrap 聚合再阈值化,在文献意义上属于"集成/聚合推断"(最近似的方法学邻居是 ESNN【已检索】);未见同名"bagging SuSiE over LD panels"的正式方法(与 lead 缺口记录一致)。
6. C2 的"稳健/脆弱/不可识别"三态:**未找到同名、同参数化(0.9/0.4)的直接先例**;但"脆弱性/稳定性"概念有充分先例(参考面板不匹配导致 "unstable credible sets" 的官方表述【已检索】、stability-guided fine-mapping【已检索】、RFR 重采样一致性【已检索】、以及领域知识中的 stability selection 与部分识别理论【未验证】)。

## 发现列表

### 1. 分布自由变量选择:Model-X knockoffs 及其遗传学应用(Q1)

- **F1.【声明】** Model-X knockoffs(Candès-Fan-Janson-Lv"Panning for Gold")实现分布自由的有限样本 FDR 控制:只要 X 的分布已知(或可精确模拟),对任意 Y|X 模型与误差分布,交换性成立,knockoff 统计量 + offset 阈值控制 FDR ≤ q。**来源**:doi:10.1111/rssb.12265【已检索】。**补充说明**:lead 文件将作者写作 "Barber & Candès",按领域知识"Panning for Gold"应为 Candès, Fan, Janson & Lv(JRSS-B 2018);fixed-X 版本才是 Barber & Candès 2015(Ann. Statist.)【未验证——需 verifier 核实归属】。**置信度**:高(核心声明);作者归属待核。
- **F2.【声明】** 关键细微点(**研究推断**):Model-X 的"分布自由"是相对 Y|X 的;它**要求 X 的边际分布已知/可估计**——在遗传学中 X 分布即基因型- LD 结构(常以参考面板估计)。因此 knockoffs 并不自动免疫"LD 参考面板不确定";这与被评审方法把"LD 不确定"作为起点、把分布自由性放在聚合/校准层的定位形成对照。**来源**:doi:10.1111/rssb.12265 的假设(依据 lead 描述 + 领域常识)【已检索 + 推断】。**置信度**:中(推断,需核原文假设表述)。
- **F3.【声明】** Knockoffs 已直接用于遗传学研究:多环境 knockoff filter 利用**跨环境一致性**识别稳健关联——思路与被评审方法"跨 LD 面板一致性"平行。**来源**:PMC11022501【已检索】。**置信度**:中(未读正文)。
- **F4.【声明】** Second-order group knockoffs 专门面向 GWAS,做条件检验 + FDR 控制。**来源**:PMC11639161【已检索】。**置信度**:中(未读正文)。
- **F5.【声明】** Spatial Knockoff Bayesian Variable Selection in GWAS 把贝叶斯变量选择与 knockoff 结合,证明"knockoff × 贝叶斯"混合已有先例(与被评审方法"knockoff/分布自由 + SuSiE"的精神面有交集,但机制不同)。**来源**:arXiv:2408.10401【已检索】。**置信度**:中(未读正文)。
- **F6.【声明】** Candès 组的 HMM knockoffs("Gene hunting with hidden Markov model knockoffs",Sesia-Sabatti-Candès)把 knockoffs 应用到遗传数据(HMM 隐 Markov 结构)。**来源**:lead 文件提及(Candès 组 pdf),**未提供 URL**;卷期/年份按领域知识为 Biometrika 2019【已检索(提及) + 未验证(细节)】。**置信度**:中偏低。
- **F7.【未验证】** 领域知识补充:fixed-X knockoffs(Barber & Candès 2015)、knockoffs + 群体结构 GWAS(Sesia et al., PNAS)、多分辨率因果变异定位(Sesia et al., Nat Commun)、yatu/快速 knockoff 软件等遗传学应用均存在,但本轮未检索核实,不列 URL。**置信度**:低(仅回忆)。

### 2. 共形推断:分布自由覆盖保证与应用(Q2)

- **F8.【声明】** "Confidence on the focal: conformal prediction with selection-conditional coverage"表明:边际有效的 conformal 区间在**被选择/聚焦的单元**上会失效(selection bias),并提出选择条件覆盖。C1 保证"真凶变异以 ≥1−α 概率被包含"恰恰是"聚焦到被选出/领先的变异"这一最脆弱场景,该文献是 C1 理论对话的第一优先级对象。**来源**:https://par.nsf.gov/servlets/purl/10595785【已检索】。**置信度**:高(lead 摘要明确;未读正文)。
- **F9.【声明】** CP4SBI(Phil Trans R Soc A 2025)把 conformal 局部校准用于模拟推断(SBI)语境的可信集——"用 conformal 校准可信集阈值"的直接先例(虽非遗传学)。**来源**:lead 文件提及,未提供 URL【已检索(提及)】。**置信度**:中偏低(细节未核)。
- **F10.【未验证】** 领域知识:split conformal / 分布自由回归预测区间(Lei, G'Sell, Rinaldo, Tibshirani, Wasserman 2018;Papadopoulos 2002)、conformalized quantile regression(Romano, Patterson, Candès 2019)、jackknife+(Barber, Candès, Ramdas, Tibshirani 2021)共同给出"任意交换性数据、任意分布、有限样本 ≥1−α 边际覆盖"的标准机制;conformal p-values(Bates, Candès, Lei, Romano, Sesia 2023)与 conformal e-values 变量选择(Marandon, Lei, Mary, Roquain 2024)提供"共形 + 选择"的 FDR 路线;Lei & Candès 2021 把 conformal 用于反事实/个体处理效应。**未检索核实,不列 URL**。**置信度**:低(仅回忆,需 verifier)。
- **F11.【声明/缺口】** 本轮**未发现** "conformal fine-mapping" 或 "conformal credible set for causal variants" 的已发表方法;conformal 在统计遗传学/生物医学中的应用以预测为主(而非因果变异覆盖集)。**来源**:lead 检索快照 + 本方向检索记录【已检索(缺席证据)】。**置信度**:中(缺席证据,不可绝对化)。

### 3. 集合构造与覆盖保证(Q3)

- **F12.【声明】** Syring & Martin《Calibrating general posterior credible regions》(Biometrika):通过 Monte Carlo 调节**标量参数**(可信水平)使后验可信区域达到**名义频率覆盖**——这是与 C1"校准阈值 q̂ 使集合达到 1−α 覆盖"**程序上最接近的先例**。**来源**:doi:10.1093/biomet/asy054(https://academic.oup.com/biomet/article/106/2/479/5237467)【已检索】。**置信度**:高(声明层面)。
- **F13.【声明】** Bootstrap coverage calibration(广义后验可信集的 bootstrap 覆盖校准)与 Calibrated Generalized Bayesian Inference(Gibbs 后验校准)显示"重采样/后验校准到名义覆盖"是活跃方向。**来源**:arXiv:2606.25729;arXiv:2311.15485【已检索】。**置信度**:中(未读正文,细节未核)。
- **F14.【声明】** Wu et al.《Improving the coverage of credible sets in Bayesian genetic fine-mapping》模拟显示多数 fine-mapping 方法报告的 CS 覆盖**过度保守**。含义:现有"贝叶斯可信集"并非精确 1−α 对象;C1 若声称精确 1−α 需要说明校准机制与数据来源;若只是"达到名义覆盖"则与文献现状一致、不属于新保证。**来源**:https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829【已检索】。**置信度**:高(lead 摘要明确)。
- **F15.【未验证】** 领域知识:knockoff 的 **offset 参数**(offset=1 提供严格有限样本 FDR 控制,offset=0 更高效但控制较弱/需条件)是"牺牲功效换保证强度"的旋钮,概念上平行于 C1 的阈值参数;derandomized knockoffs(Ren & Barber, JRSS-B)把**多次 knockoff 运行**的选择结果经 e-values 聚合,减少内在随机性——这是"把重复运行聚合为稳定选择"的最接近文献,与被评审方法的"反复运行再聚合"做法呼应。**未检索核实,不列 URL**。**置信度**:低(仅回忆)。
- **F16.【声明】** 遗传 fine-mapping 中已有的"置信集合"(如 CAVIAR 风格的 causal variant set,属贝叶斯后验驱动)不是分布自由的有限样本保证对象【未验证——领域知识;T1 将详细覆盖】;在"有限样本、无分布假设、含真信号概率 ≥1−α"这一精确意义上,本轮未发现遗传学直接先例。**置信度**:中(结合 lead 检索与领域常识)。

### 4. PIP 聚合/bagging 与"分布自由"的关系(Q4)

- **F17.【声明】** ESNN(ensemble of single-effect Bayesian neural networks,《Uncertainty quantification in variable selection for genetic fine-mapping using Bayesian neural networks》)是目前**最接近"聚合选择不确定性"的方法学邻居**:以集成方式泛化 SuSiE,对"哪个变量被选中"做不确定性量化。**来源**:PMC9234235【已检索】。**置信度**:中(未读正文,细节未核)。
- **F18.【声明/缺口】** 未见同名方法:"bagging SuSiE over LD panels"/"aggregate PIP over LD" 作为正式方法论论文在 lead 检索与本方向均未出现(lead 缺口记录一致;SuSiEx/MultiSuSiE 是多祖源联合而非"同一汇总统计 + LD 不确定集合";SuSiNE 是多初始点(multi-basin)集成【已检索,均见 lead 文件】,维度与被评审方法的多 LD 不同)。**来源**:lead 文件【已检索】。**置信度**:中(缺席证据)。
- **F19.【未验证 + 推断】** 领域知识:SuSiE 的 CS 由 PIP 在贝叶斯模型下构建,覆盖保证是渐近/近似的(Wang et al. 2020 原论文【未验证】);PIP 是后验概率,**构造上不是分布自由对象**。被评审方法的"分布自由"若成立,必须来自聚合/校准步骤(如 bootstrap 校准、交换性论证),而不能来自 PIP 本身——这是**研究推断**,需评审在方法学上正面回答"分布自由性由哪一步提供"。**置信度**:中(推断)。

### 5. 三态判定(稳健/脆弱/不可识别)概念先例(Q5)

- **F20.【声明】** 官方协议文明确记载 LD 参考面板不匹配 "can produce ... **unstable credible sets** or apparently strong signals"(FunGen-xQTL 协议 vignette)——"不稳定的可信集"是被评审方法 C2"脆弱性"动机的直接文献表述。**来源**:https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html【已检索(lead 转引原文)】。**置信度**:高(引文来自 lead 快照)。
- **F21.【声明】** eLife《The impact of stability considerations on genetic fine-mapping》:stability-guided 的 fine-mapping 思路已存在。**来源**:https://elifesciences.org/articles/88039【已检索】。**置信度**:中(未读正文)。
- **F22.【声明】** Replication Failure Rate(RFR,Nature Genetics 2023):用 **down-sampling 重采样一致性**评估 fine-mapping 校准/可复现性——"跨重采样一致 = 稳健"的诊断先例,与 C2 的"跨 LD 矩阵/跨 bootstrap 一致"思路同构。**来源**:doi:10.1038/s41588-023-01597-3【已检索】。**置信度**:高。
- **F23.【未验证】** 领域知识补充三态概念先例:① stability selection(Meinshausen & Bühlmann 2010, JRSS-B):按子样本选择频率定义稳健变量——与 C2 的"领先集中度 c = 跨矩阵领先频率"在形式上几乎一一对应(研究推断;此为最重要的潜在先例,verifier 必查);② 部分识别/识别集理论(Manski):数据不足以下决定时输出"识别集"而非单点——"不可识别"状态的形式化先例;③ E-value 敏感性分析(VanderWeele & Ding 2017):量化结论对未测混杂的脆弱性;④ 分类中的弃权/选择性预测与 conformal 集合输出(集合大小为 0/1/多 时"不可判定"的天然状态)。**全部未检索核实,不列 URL**。**置信度**:低(仅回忆)。
- **F24.【声明/缺口】** 0.9/0.4 两个阈值与"稳健/脆弱/不可识别"三态命名:lead 检索与领域知识中**均未找到直接先例**(lead 缺口记录一致)。**置信度**:中(缺席证据)。

## 对评审对象的启示

**C1(覆盖集合 + 1−α 保证 + "分布自由")是否已有直接先例?——分三层回答:**

1. **"分布自由变量选择"在遗传学中有强先例(knockoffs 系列,见 F1-F6)**,但目标函数不同:knockoffs 控制 **FDR(误选率)**,不保证"真凶 ∈ 集合 概率 ≥1−α"(覆盖)。C1 的保证类型是**覆盖保证**,其直接理论亲缘在 conformal/校准文献,不在 knockoff 文献。
2. **"校准到名义覆盖"有直接程序先例**:Syring & Martin 标量校准(F12)与 bootstrap 覆盖校准(F13)在逻辑上就是"调一个阈值让集合覆盖率达到 1−α"。因此 C1 的"校准 q̂"不是新概念;**其潜在新意只能是:(a) 校准对象 = 跨多 LD × bootstrap 聚合的 PIP 分数;(b) 校准数据 = 自举/合成 LD 数据自身。** 这两点需与 F13 类文献逐字对比才能判定新颖度。
3. **C1 的"分布自由覆盖真凶"声明面临两个必须正面回答的理论问题**:① **选择依赖**:par.nsf.gov/10595785(F8)证明边际覆盖保证在被选择的聚焦单元上失效——C1 保证的"真凶"恰恰是被挑选/领先的变异,若无选择条件覆盖论证,1−α 声明在理论上站不住;② **校准的源**:分布自由性必须由某个可辩护的交换性/重采样论证提供,而模型-X 式"分布自由"要求 X(即 LD)分布已知(F2),与被评审方法"LD 未知、需要多面板"的出发点存在张力——评审应要求方法说明:分布自由性到底由哪一步、在什么交换性假设下成立。另外 Wu et al.(F14)表明现有 CS 已过度保守,C1 若仅声称"达到 1−α"与现状一致;真正的贡献点是**紧致性/功效与覆盖的权衡**。

**C2(三态)是否有概念先例?——有概念先例,无命名/参数化先例:**

- "脆弱/不稳定"的概念直接见于 F20(unstable credible sets)、F21(stability-guided)、F22(RFR 重采样一致性);
- 形式最接近的机制先例可能是 stability selection 的"选择频率"(F23,未验证,需 verifier 优先核实);
- "不可识别"状态的形式化先例在因果推断部分识别(Manski)中存在;
- 但 **"稳健/脆弱/不可识别"三态命名 + 0.9/0.4 阈值**在检索范围内无直接对应物(lead 缺口记录一致)。这意味着:C2 作为"诊断工具"的新颖性高于 C1,但也意味着阈值动机缺乏文献支撑,评审应要求提供 0.9/0.4 的动机或敏感性分析。

**总体判断(研究推断):** 被评审方法更接近"已知方法学构件(knockoff-式分布自由选择 + conformal/校准式覆盖 + 稳定性诊断)的**特定组合与特定参数化**",而非全新概念;组合层面的新颖点集中在"多 LD 面板 × bootstrap × 聚合 PIP 校准阈值"这一编排,而该编排的核心风险点是 F8 的选择依赖问题与 F2 的 LD 分布假设张力。

## 缺口

- **本轮工具限制**:本子代理无 web 工具,只依赖 lead 检索快照;所有 PMC/arXiv 文献正文未被本子代理打开核读,摘要级声明可信,细节(方法假设、数字、卷期)需 verifier 补核。
- **待补 URL/细节**:HMM knockoffs(F6)与 CP4SBI(F9)在 lead 文件中无 URL;建议 verifier 检索 Sesia-Sabatti-Candès(Biometrika 2019)与 CP4SBI(Phil Trans R Soc A 2025)原文。
- **领域知识条目全部未验证**(F7, F10, F15, F19, F23):优先级最高的是 **stability selection(Meinshausen & Bühlmann 2010)**——它与 C2 的"集中度"在形式上高度同构,直接影响新颖性判定;其次是 derandomized knockoffs(Ren & Barber,与"多次运行聚合"呼应)与 Bates 等 conformal p-values 变量选择。
- **未找到(缺席证据)**:① conformal fine-mapping / conformal credible set for causal variants 的已发表方法;② "bagging SuSiE over LD panels" 同名方法;③ C2 三态命名与 0.9/0.4 阈值的直接文献;④ C1 式"分布自由保证含真概率 ≥1−α"在遗传 fine-mapping 中的直接先例。
- **待核/矛盾**:lead 将 "Panning for Gold"(Model-X)归为 Barber & Candès;按领域知识作者应为 Candès, Fan, Janson & Lv(fixed-X 版才是 Barber & Candès 2015)。已在上文标注,verifier 需确认归属,避免引用错误。
- arXiv:2606.25729 的编号年份(26 开头)与本次评审日期(2026-09)一致,合理;但其具体假设(覆盖的是何种后验、校准数据的交换性要求)未核。

## 来源清单(仅 lead 文件中实际出现的 URL,无新增)

- 保留(直接引用):
  - Model-X knockoffs doi:10.1111/rssb.12265(作者归属待核)——分布自由变量选择基石
  - 多环境 knockoff filter PMC11022501 —— 跨环境一致性先例
  - Second-order group knockoffs GWAS PMC11639161 —— knockoffs 应用于 GWAS
  - Spatial Knockoff Bayesian Variable Selection in GWAS arXiv:2408.10401 —— 贝叶斯 × knockoff 混合
  - Selection-conditional coverage https://par.nsf.gov/servlets/purl/10595785 —— C1 理论对话核心
  - Wu et al. credible set coverage https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829 —— CS 过度保守实证
  - Syring & Martin https://academic.oup.com/biomet/article/106/2/479/5237467 —— 标量校准先例
  - Bootstrap coverage calibration arXiv:2606.25729;Calibrated GBI arXiv:2311.15485 —— 重采样校准
  - RFR doi:10.1038/s41588-023-01597-3 —— 重采样一致性诊断
  - ESNN PMC9234235 —— 聚合选择不确定性的最近方法邻居
  - FunGen-xQTL protocol vignette https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html —— "unstable credible sets"表述
  - eLife stability paper https://elifesciences.org/articles/88039 —— stability-guided fine-mapping
- 提及无 URL(待 verifier 补):HMM knockoffs(Candès 组 pdf);CP4SBI(Phil Trans R Soc A 2025)。
- 领域知识条目(未验证,未列 URL,等待 verifier 检索):Barber & Candès 2015;Ren & Barber derandomized knockoffs;Bates et al. 2023;Marandon et al. 2024;Lei et al. 2018;Romano et al. 2019;Barber et al. 2021;Lei & Candès 2021;Meinshausen & Bühlmann 2010;Manski 部分识别;VanderWeele & Ding 2017;Wang et al. 2020(SuSiE)。
- 无"已检索但拒绝"的来源(lead 快照未提供可判为无关的候选);本轮无独立检索结果。