# 创新性与可行性评审(CITED)
## 面向连锁不平衡不确定性的分布自由因果变异集合构建与精细定位脆弱性诊断

**日期**:2026-09-13
**状态**:cited — 内联引用 + 来源清单;关键引文已经逐字核验;其余标【标识级/未验证】项已如实标注

**引证约定**:正文内 [S#] 对应文末 Sources;证据分级沿用研究笔记:(已检索=来源经搜索/抓取确认存在) / (标识级=DOI/期刊 ID 确认,正文未核) / (未验证=领域知识,无 URL) / (推论=本报告推理)。

---

## 执行摘要

本报告评审如下方法设计的创新性与可行性:给定一个位点的 GWAS 汇总统计(beta、se)与多个 LD 参考面板(如 EUR/AFR 等 + bootstrap),该方法(①)合成一组代表"LD 不确定"的 LD 矩阵;(②)对每个 LD 矩阵独立运行 SuSiE 得到逐变异 PIP;(③)计算两个聚合量——非一致度分数 score(j)=1−mean PIP(j)(用于 C1 覆盖集合)与领先集中度 c=max_j freq(领先变异=j)(用于 C2 三态判定);(④)输出:C1 覆盖集合 {变异: score ≤ q̂}(声称含真凶概率 ≥1−α,自谓"分布自由")与 C2 三态标签(稳健 c≥0.9 / 脆弱 0.4≤c<0.9 / 不可识别 c<0.4)。

**核心结论(基于文献证据的评审判断):**

1. **动机不新(证据充分)**。LD 参考面板不匹配损害精细定位是文献与官方文档公认事实:SusieR 官方文档明确其"inducing high false positives in fine-mapping"并已有模型化校正 [S1];FunGen-xQTL 协议逐字记载"LD reference should match the GWAS ancestry and genome build. A mismatch can produce distorted z-score/LD relationships, unstable credible sets..."[S3]。以"LD 不确定性问题存在"作为卖点不构成贡献。

2. **"LD 不确定性建模"存在直接先例**:SusieR rss_mismatch 单面板参数化校正 [S1];SuSiEx [S22] / MultiSuSiE [S23] 的多面板联合建模。被评审组合的机制差异点是"面板间独立推断后聚合 PIP"+"把 LD 当可重采样的随机量"。该特定编排(多面板 × bootstrap × 逐矩阵 SuSiE × 聚合 PIP)在本次检索范围内未找到同名直接先例(SuSiNE 是"多初始点"集成,维度不同 [S27])。【缺席证据,不可绝对化】

3. **C1 的"分布自由 1−α 覆盖集合"面临两个必须正面回答的理论缺口**:
   - **选择依赖问题**:共形推断文献 [S6a][S6b](已摘要级验证)明确边际覆盖保证在"选择后聚焦单位"上失效(selection-conditional coverage)。C1 的"真凶 ∈ 集合"恰是被挑选/领先变异的聚焦场景;若无条件覆盖论证,1−α 声明在理论上不成立。**本方法最大理论风险点**。【推论+[S6a][S6b]】
   - **分布自由性的来源**:Model-X knockoffs [S5] 的"分布自由"要求 X(基因型/LD)分布已知或可估;被评审方法以"LD 未知"为出发点,需说明分布自由性由哪一步、在何交换性假设下成立。【推论,依据 [S5] 假设】
   - 另 Wu 等 [S4] 实证:多数贝叶斯 fine-mapping 的 CS 覆盖**过度保守**,原因是"fine-mapping 数据集并非从所有因果变异中随机选取,而是从效应量更大的因果变异中选取"(已逐字核验)。因此"达到 1−α"不足为奇;真正贡献点是**紧致性/功效与覆盖的权衡**。

4. **C1 的"校准阈值 q̂"有成熟程序先例**:Syring & Martin 标量校准 [S11]、bootstrap 覆盖校准 [S12](存在性未核实,见来源注)、CP4SBI conformal 校准可信集 [S13]。"校准到名义覆盖"不新;潜在新意仅剩"校准对象 = 跨多 LD × bootstrap 聚合 PIP"这一具体编排 [S11][S12]。

5. **C2 三态判定新颖性相对更高,但阈值动机缺文献支撑**。"不稳定可信集"(协议原文 [S3])、stability-guided fine-mapping [S16]、Replication Failure Rate 重采样一致性 [S15]、以及 stability selection 的"选择频率"概念 [S17](与"领先集中度 c = 跨矩阵领先频率"形式高度相关)构成概念先例;"稳健/脆弱/不可识别"命名与 0.9/0.4 阈值无直接先例——需方法方提供阈值动机或敏感性分析。

6. **可行性:方法学上可行,工程上无直接证据**。多面板输入有工程先例 (SuSiEx [S22]);SuSiE 单区段扩展到数千 SNP 是实务共识,SuSiE 2.0 预印本称 summary-stats 应用提速最高 5 倍 [S25](未核正文);公开参考面板可得(UKBB-LD AWS 开放数据 [S28])。但"多面板 × bootstrap × SuSiE 聚合"整体管线无文献实测成本;数量级估算为 K×B×L 次 SuSiE 运行 [推论]。**必须澄清 bootstrap 的重采样语义**(重采样面板个体 vs 参数扰动):输入只有汇总统计时,标准 bootstrap 无直接作用对象 [推论]。

---

## 1. 现状:精细定位方法与 LD 不确定性

### 1.1 主流方法把 LD 当作"已知且固定"的输入

- **SuSiE**(Wang, Sarkar, Carbonetto & Stephens 2020, JRSS-B 82(5):1273–1300, doi:10.1111/rssb.12388)[S18]:以边际统计量 + LD 矩阵 R 为输入,输出逐 SNP PIP 与名义覆盖 ≥1−α 的可信集;方法本身不把 R 视为随机量。【已检索(论文元数据);正文细节未逐页核】
- **FINEMAP**(Benner et al. 2016, Bioinformatics)、**CAVIAR**(Hormozdiari et al. 2014, Genetics):同为"边际 z + 参考 LD"框架。【未验证-领域知识】
- 结论:主流方法普遍不做"LD 不确定性传播";对 LD 错误的显式处理在文献中已有单面板参数化校正([S1],见 §1.2/§1.3),但"把 LD 当作可重采样的随机量传播到 PIP"的做法仍属空白。【推断】

### 1.2 LD 参考面板不匹配是有据可查的公认问题

- SusieR 官方文档:《Modeling and Accounting for LD Reference Mismatch in Summary Statistics Fine-mapping》明确不匹配"inducing high false positives in fine-mapping",并提供模型化校正(susie_rss)[S1]。配套诊断 vignette 处理等位基因编码伪影等不一致 [S2]。
- FunGen-xQTL 协议(逐字,已核验):"The LD reference should match the GWAS ancestry and genome build. A mismatch can produce distorted z-score/LD relationships, **unstable credible sets** or apparently strong signals that are alignment artifacts."[S3] ——"不稳定可信集"表述已见于公开协议,是 C2 动机的直接文字对应物。
- Weissbrod 等(bioRxiv 807792,PolyFun 预印本)涉及 in-sample LD vs 外部参考面板的影响比较 [S24](内容为元数据级)。RSparsePro(bioRxiv 2024.10.29.620968)专门针对 LD mismatch 的稳健精细定位 [S26]。Benner 等(AJHG 2017)涉及参考面板 LD vs 个体水平 GWAS LD 的系统评估 [S21](内容为元数据级)。

### 1.3 "多面板 / LD 不确定性建模"先例

- **单面板参数化建模**:SusieR rss_mismatch 校正(单一参考面板 + 模型内校正参数化)[S1]。
- **多面板联合建模**:SuSiEx(Nature Genetics 2024)对每个祖源使用各自汇总统计与 LD,联合推断共享因果结构 [S22];MultiSuSiE 为多祖源 SuSiE 扩展 [S23]。二者是"一个模型吃多面板",非"面板间独立推断后聚合"。
- **跨环境一致性**:多环境 knockoff filter 利用跨环境一致性识别稳健关联 [S8]。
- **多初始点集成**:SuSiNE(multi-basin ensembling)用多起始点应对 LD 歧义/局部最优 [S27];集成维度(初始化)不同于被评审方法(多 LD 矩阵)。
- **聚合选择不确定性**:ESNN(ensemble of single-effect Bayesian neural networks)以集成方式泛化 SuSiE,量化"哪个变量被选中"的不确定性 [S7]——方法学邻居,但集成对象是模型而非 LD 矩阵。
- **未找到直接先例**:对 LD 矩阵做 bootstrap/重采样 → 逐矩阵跑 SuSiE → 聚合 PIP 的已发表方法,检索范围内未找到。[缺席证据]

---

## 2. 现状:分布自由推断

### 2.1 分布自由变量选择在遗传学是成熟方向(knockoffs)

- **Model-X knockoffs**(**Candès, Fan, Janson & Lv**, "Panning for Gold", JRSS-B 80(5):1271–1301, 2018, doi:10.1111/rssb.12265)[S5]:对任意 Y|X 模型与误差分布提供有限样本 FDR 控制,不假设误差分布。(注:早期检索快照误记为 "Barber & Candès";正确归属为 CFJL 2018,fixed-X 版为 Barber & Candès 2015, Ann. Statist. 43(5) [S19]。作者归属经 DOI 与全文 PDF 双重确认 [S5][S20]。)
- 遗传学应用:多环境 knockoff [S8]、second-order group knockoffs for GWAS [S9]、Spatial Knockoff Bayesian Variable Selection in GWAS [S10]、HMM knockoffs(Candès 组;Sesia-Sabatti-Candès 系列,细节未核)。
- **关键细微点(推断)**:Model-X 的"分布自由"是相对 Y|X 的;它要求 X 的边际分布(基因型-LD 结构)已知或可估——knockoffs 并不自动免疫 LD 参考面板不确定。与被评审方法"把 LD 不确定作为起点、把分布自由性放在聚合/校准层"的定位形成对照。【推论,依据 [S5] 假设】
- 控制目标不同:knockoff 控制 FDR(期望误选比例),不提供"真凶 ∈ 集合 ≥1−α"覆盖保证;C1 的保证类型是覆盖,亲缘在 conformal/校准文献。【推论】

### 2.2 共形推断:覆盖保证与"选择依赖"问题

- Selection-conditional coverage(《Confidence on the focal: conformal prediction with selection-conditional coverage》)[S6a][S6b,摘要级验证]:边际有效的 conformal 区间在"被选择/聚焦的单元"上会失效(selection bias),需选择条件覆盖。**【C1 理论对话第一优先级对象】**(提示:该结论为共形推断文献的标准主题,另有 jackknife+ 等边际覆盖文献 [S30] 作为背景支撑)
- "conformal fine-mapping / conformal credible set for causal variants"的已发表方法:本次检索未发现。[缺席证据]

### 2.3 校准到名义覆盖:成熟先例

- **Syring & Martin**《Calibrating general posterior credible regions》,Biometrika 106(2):479–486, 2019 [S11]:Monte Carlo 调节标量参数使可信区域达名义频率覆盖——与 C1"校准 q̂"程序最接近。【已检索】
- **Bootstrap coverage calibration**(arXiv:2606.25729)[S12] 与 **Calibrated Generalized Bayesian Inference**(arXiv:2311.15485)[S14]:重采样校准后验可信集是活跃方向。(注意:S12 的 arXiv ID 为 2026-06 编号,与本次评审日期一致,但**存在性尚未核实**——引用前须再验。)
- **CP4SBI**(Phil Trans R Soc A 2025):conformal 局部校准可信集(SBI 语境)[S13]。
- 含义:达到名义覆盖的方法论是现成的,不构成新概念;新颖性须来自"校准对象/校准数据源"。【推断】

---

## 3. 现状:PIP 校准、脆弱性诊断与评估基准

### 3.1 覆盖与校准实证

- **Wu 等**(PLOS Comp Biol, journal.pcbi.1007829)[S4],已逐字核验:"we use simulations to demonstrate that the coverage probabilities are **over-conservative** in most fine-mapping situations... because fine-mapping data sets are not randomly selected from amongst all causal variants, but from amongst causal variants with larger effect sizes."并给出 "adjusted coverage estimator" 方法。
- 含义:① "达到 1−α"不是难点,过度保守意味着集合偏大、分辨率下降;② 对被评审方法:报告应同时报覆盖盈余(或精确校准)与集合大小/分辨率;③ 该文的"选择偏向"解释与 C1 的保证声称形成微妙张力——若 C1 评估也只在"显著/致病位点"上做,则其经验覆盖天然偏高,需校正后比较 [S4]。

### 3.2 脆弱性 / 稳定性诊断先例

- "unstable credible sets"表述见官方协议(§1.2)[S3]。
- eLife《The impact of stability considerations on genetic fine-mapping》:stability-guided 思路已存在 [S16]。
- **Replication Failure Rate**(Nature Genetics 2023, doi:10.1038/s41588-023-01597-3)[S15]:提出以 down-sampling 重采样一致性评估 fine-mapping 可复现性/校准的指标(功能描述为元数据级,正文未核)——与被评审方法"跨矩阵/跨 bootstrap 一致"思路同族。【已检索】
- **stability selection**(**Meinshausen & Bühlmann 2010**, JRSS-B 72(4):417–473, doi:10.1111/j.1467-9868.2010.00740.x)[S17]:以子样本选择频率定义稳健变量,并提供有限样本错误率控制(本报告为元数据级确认;正文机制细节建议方法方自行研读)。与 C2"领先集中度"在形式上**高度相关但不同构**:stability selection 是逐变异的频率向量经阈值化/求和做错误率控制,而 c 是领先变异的单点集中度——评审 C2 新颖性时应以此对照为基线。**【已检索(元数据)】**
- 其他概念近亲(未逐一核验):部分识别理论(Manski,"不可识别"状态形式化)、E-value 敏感性分析(VanderWeele & Ding)。【未验证-领域知识】
- **0.9/0.4 阈值与三态命名**:无直接先例;需方法方提供动机或敏感性分析。[缺席证据]

### 3.3 评估基准惯例与数据可得性

- 模拟基准:真实 LD(1000G 或样本内)模拟 z 值、植入已知因果变异;指标 = CS 经验覆盖 vs 标称 1−α、CS 大小、PIP 分箱校准曲线、含真变异精确度。[未验证-领域知识]
- 真实数据基准:实验支持 eQTL(GTEx、eQTL Catalogue)、UKB 已知位点。[未验证-领域知识]
- 数据可得性:**UKBB-LD**(AWS Open Data, arn:aws:s3:::broad-alkesgroup-ukbb-ld;N≈337K 英裔,2763 个 3Mb 区域)[S28];All of Us / UKB LD 面板 [S29];1000 Genomes Phase 3 五超族群为常用公开面板。[未验证-领域知识]

---

## 4. 可行性分析

### 4.1 组件可行性

- 多面板输入:工程先例充分(SuSiEx [S22] / MultiSuSiE [S23] 均输入多个族群 LD 面板)。
- SuSiE 单区段扩展到数千 SNP:实务共识(未验证经验值);SuSiE 2.0 预印本(bioRxiv 2025.11.25.690514)称 summary-stats 应用提速最高 5 倍 [S25](正文未核)。
- 运行次数:K(面板)×B(bootstrap)×L(位点)次 SuSiE;以 K≈5、B≈50–100、L≈100–1000 计,为 2.5 万–50 万次(上限 5×100×1000 = 5×10⁵);按单次秒–分钟级、集群并行 100–1000 核,小时到天级。[推论,数量级估算,非测量值]
- 内存:L=10^4 SNP 的稠密 p×p 双精度 LD ≈ 0.8 GB/个;多矩阵需注意存储与 IO。[推论]

### 4.2 必须澄清的设计问题(bootstrap 语义)

- 输入只有汇总统计(beta、se)时,标准 bootstrap(重采样观测)无直接作用对象;"对 LD 矩阵做 bootstrap"需明确:重采样面板个体(需面板个体级基因型,1000G 可提供;单族群 ~300–660 人,决定 LD 估计噪声)vs 对 z 统计量/LD 做参数扰动。[推论,置信度中]
- 该语义同时决定统计有效性与实现成本,是可行性的核心质疑点。【推断】

---

## 5. 创新性判定

### 5.1 C1(覆盖集合)判定:**弱-中创新,理论风险高**

| 维度 | 判定 | 依据 |
|------|------|------|
| 问题动机 | 已有(不新) | [S1][S3] |
| "LD 不确定性建模" | 已有先例 | [S1][S22][S23] |
| "校准阈值 q̂ 达名义覆盖" | 已有程序先例 | [S11][S12]*,[S13] |
| 具体编排(多面板×bootstrap×聚合PIP) | 未见直接先例 | 缺席证据 |
| "分布自由"声称 | **需正面论证来源** | §2.1/2.3 [S5] |
| 1−α 覆盖声称 | **受选择依赖问题威胁** | [S6a][S6b] |
| 紧致性/功效权衡 | 报告义务 | [S4] |

判定文字:C1 属"已知构件的新组合";新颖程度取决于能否在(a)选择条件覆盖、(b)分布自由性来源两点提供新的理论论证,以及在模拟中展示**优于"单面板+校正"(rss_mismatch)与"多面板联合"(SuSiEx)的紧致性增益**,而非仅覆盖达标。

### 5.2 C2(三态)判定:**中创新,但需阈值论证**

- 概念先例充分([S3][S15][S16][S17]),"诊断脆弱性"不新;新的是"以跨面板领先集中度做三态自动分级"这一输出形式(未见直接先例)。
- 风险:0.9/0.4 阈值无文献依据;三态命名首次出现。需方法方提供阈值动机、敏感性分析,并与 stability selection [S17](同族但不同构的对照基线)/ 部分识别理论对话,否则有"重新发明或有遗漏"之嫌。

### 5.3 总体判定

**创新性:组合型(低-中)。可行性:方法学上可行,工程上未验证。** 骨架(多面板、聚合、校准、稳定性诊断)均有先例;独立成分级未发现新统计概念。可能成立的贡献是"面向 LD 不确定性、带覆盖性质的集合 + 位点级脆弱性诊断"的**产品化组合**及配套算法细节。作为论文选题成立与否取决于:理论层面解决选择依赖与分布自由来源;实证层面与 rss_mismatch 校正、SuSiEx、单面板 SuSiE 做 head-to-head 并量化紧致性/覆盖权衡;工程层面给出 bootstrap 语义与实测成本。

---

## 6. 证据支撑的注意事项与分歧

- **作者归属(已解决)**:早期 lead 检索记录将 Model-X "Panning for Gold" 记为 Barber & Candès,有误;DOI 注册与论文 PDF 确认为 **Candès, Fan, Janson & Lv, JRSS-B 80(5), 2018** [S5][S20];Barber & Candès 2015 为 fixed-X 版(Ann. Statist. 43(5))[S19]。本报告已按正确归属书写。
- **过度保守 vs 名义覆盖**:Wu 等[S4]报告覆盖过度保守,SuSiE 原论文声称名义 ≥1−α 覆盖 [S18];二者不直接冲突(名义性质 vs 实证行为),但对 C1"保证"口径有直接影响——如实记录,非消解。
- **验证范围**:大量"已检索"来源经搜索/抓取确认存在,但正文未逐页核读;标识级条目(SuSiE 2.0 [S25]、MultiSuSiE [S23]、RSparsePro [S26]、SuSiNE [S27])仅确认标识;领域知识条目(§1.1 FINEMAP/CAVIAR、部分识别、E-value)无 URL,引用时须由方法方自行核实。

---

## 7. 开放问题(Open Questions)

1. C1 的 1−α 是逐位点条件保证还是边际保证?如何回避/论证 selection-conditional 失效 [S6a][S6b]?(最高优先)
2. "分布自由"由哪一步提供?采用何种交换性/重采样论证?与 Model-X 要求"X 分布已知" [S5] 的张力如何化解?
3. bootstrap 的确切对象与算法(个体级重采样 vs 参数扰动)?对小面板 LD 噪声的影响?
4. 0.9/0.4 阈值的动机与敏感性;三态与 stability selection [S17] / 部分识别理论的关系?
5. 与 rss_mismatch 校正版 [S1]、SuSiEx [S22]、单面板 SuSiE [S18] 的 head-to-head 实验设计?
6. 计算成本实测(每阶段耗时/内存)与并行策略?
7. q̂ 校准的统计准则(模拟数据校准?交叉验证?);校准数据集与目标位点的同一性?Wu 等 [S4] 的"选择偏向"提示校准必须校正位点选择效应。

---

## Sources

**[S1]** SusieR 官方文档:《Modeling and Accounting for LD Reference Mismatch in Summary Statistics Fine-mapping》https://stephenslab.github.io/susieR/articles/rss_mismatch.html 【已检索,内容未逐页核读】
**[S2]** SusieR 诊断 vignette《Diagnostic for fine-mapping with summary statistics》https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html 【已检索】
**[S3]** FunGen-xQTL 协议《Fine-mapping GWAS summary statistics with SuSiE RSS》https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html 【已检索;关键引文已逐字核验】
**[S4]** Wu et al.《Improving the coverage of credible sets in Bayesian genetic fine-mapping》PLOS Comp Biol, https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829 【已检索;关键声明已逐字核验】
**[S5]** Candès, Fan, Janson & Lv (2018). *Panning for Gold: Model-X Knockoffs for High Dimensional Controlled Variable Selection*. JRSS-B 80(5), doi:10.1111/rssb.12265 【已检索;作者归属经 DOI + PDF 确认】
**[S6a]** *Confidence on the Focal: Conformal Prediction with Selection-Conditional Coverage.* NSF PAR purl, https://par.nsf.gov/servlets/purl/10595785 【已摘要级验证;PDF>20MB 未全文抓取】
**[S6b]** 同文 arXiv 版:https://arxiv.org/abs/2403.03868 · doi:10.48550/arxiv.2403.03868 【已检索:摘要确认"marginally valid CP intervals may fail to provide valid coverage for the focal unit(s) due to selection bias"】
**[S7]** ESNN:《Uncertainty quantification in variable selection for genetic fine-mapping using Bayesian neural networks》https://pmc.ncbi.nlm.nih.gov/articles/PMC9234235/ 【已检索;正文未核】
**[S8]** Searching for robust associations with a multi-environment knockoff filter, https://pmc.ncbi.nlm.nih.gov/articles/PMC11022501/ 【已检索;正文未核】
**[S9]** Second-order group knockoffs with applications to genome-wide association studies, https://pmc.ncbi.nlm.nih.gov/articles/PMC11639161/ 【已检索;正文未核】
**[S10]** Spatial Knockoff Bayesian Variable Selection in GWAS, arXiv:2408.10401, doi:10.48550/arxiv.2408.10401 【已检索;正文未核】
**[S11]** Syring & Martin (2019). *Calibrating general posterior credible regions*. Biometrika 106(2):479–486, https://academic.oup.com/biomet/article/106/2/479/5237467 【已检索】
**[S12]** A Theory of Bootstrap Coverage Calibration for Generalized Posterior Credible Sets, arXiv:2606.25729, https://arxiv.org/abs/2606.25729 【⚠ 来源域已从 arxiv.gg 修正为 arxiv.org;该 ID(2026-06)存在性尚未核实】
**[S13]** CP4SBI: local conformal calibration of credible sets in simulation-based inference. Phil Trans R Soc A 384, https://royalsocietypublishing.org/rsta/article/384/2327/20250069/483087/CP4SBI-local-conformal-calibration-of-credible 【已检索(提及);URL 为审计阶段补入,未核】
**[S14]** Calibrated Generalized Bayesian Inference, arXiv:2311.15485, https://arxiv.org/html/2311.15485v3 【已检索;正文未核】
**[S15]** Kanai et al.(2023). *Improving fine-mapping by modeling infinitesimal effects*(含 Replication Failure Rate). Nature Genetics, doi:10.1038/s41588-023-01597-3 【已检索;正文未核】
**[S16]** eLife:《The impact of stability considerations on genetic fine-mapping》https://elifesciences.org/articles/88039 【已检索;正文未核】
**[S17]** Meinshausen & Bühlmann (2010). *Stability selection*. JRSS-B 72(4):417–473, doi:10.1111/j.1467-9868.2010.00740.x; https://stat.ethz.ch/Manuscripts/buhlmann/stability.pdf 【已检索(元数据)】
**[S18]** Wang, Sarkar, Carbonetto & Stephens (2020). *A simple new approach to variable selection in regression, with application to genetic fine mapping*. JRSS-B 82(5):1273–1300, doi:10.1111/rssb.12388 【已检索(元数据+官方页面)】
**[S19]** Barber & Candès (2015). *Controlling the false discovery rate via knockoffs*. Ann. Statist. 43(5), doi:10.1214/15-AOS1337 【已检索(元数据)】
**[S20]** Model-X knockoffs 全文 PDF(USC 镜像,确认作者序):https://faculty.marshall.usc.edu/yingying-fan/publications/JRSSB-CFJL18.pdf 【已检索;作者已验证】
**[S21]** Benner et al.(2017). *Prospects of Fine-Mapping Trait-Associated Genomic Regions by Using Summary Statistics from GWAS*. AJHG, PMID 28942963, https://pubmed.ncbi.nlm.nih.gov/28942963/ 【已检索(元数据)】
**[S22]** SuSiEx:《Fine-mapping across diverse ancestries drives the discovery of putative causal variants underlying human complex traits and diseases》Nature Genetics 2024, doi:10.1038/s41588-024-01870-z;github.com/getian107/SuSiEx 【已检索】
**[S23]** MultiSuSiE improves multi-ancestry fine-mapping in All of Us, https://pmc.ncbi.nlm.nih.gov/articles/PMC13091671/ 【标识级,正文未核】
**[S24]** Weissbrod et al.《Functionally-informed fine-mapping and polygenic localization of complex trait heritability》(PolyFun 预印本), doi:10.1101/807792 【已检索(元数据)】
**[S25]** SuSiE 2.0, bioRxiv 2025.11.25.690514, https://www.biorxiv.org/content/10.1101/2025.11.25.690514v1 【标识级,正文未核】
**[S26]** RSparsePro:《Robust fine-mapping in the presence of linkage disequilibrium mismatch》bioRxiv 2024.10.29.620968 【标识级,正文未核】
**[S27]** SuSiNE: Genetic fine-mapping with signed functional priors and multi-basin ensembling, https://sciety.org/articles/activity/10.64898/2026.07.31.742084 【已检索;非常规平台(Sciety activity 页),可用性存疑】
**[S28]** UK Biobank Linkage Disequilibrium Matrices(AWS Open Data), https://registry.opendata.aws/ukbb-ld/ 【已检索】
**[S29]** LD reference panels — All of Us and UK Biobank, doi:10.5281/zenodo.16923734 【已检索(元数据);具体承载数据集未核】

**[S30]** Barber, Candès, Ramdas & Tibshirani (2021). *Predictive inference with the jackknife+*. Ann. Statist. 49(1):486–507, doi:10.1214/20-aos1965 【已检索(元数据);共形/边际覆盖文献背景】

**未验证(领域知识,无 URL,引用前须自行核实):** FINEMAP(Benner et al. 2016, Bioinformatics)、CAVIAR(Hormozdiari et al. 2014, Genetics)、HMM knockoffs(Sesia-Sabatti-Candès)、Manski 部分识别、VanderWeele & Ding E-value、GTEx/eQTL Catalogue 基准、1000 Genomes Phase 3 面板规模细节。