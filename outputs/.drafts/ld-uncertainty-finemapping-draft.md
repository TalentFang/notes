# 创新性与可行性评审草稿(DRAFT)
## 面向连锁不平衡不确定性的分布自由因果变异集合构建与精细定位脆弱性诊断

**修订**:PLEASE-CITE(由 cited 阶段替换)
**日期**:2026-09-13
**状态**:draft — 尚未做引用验证与评审;所有来源标注基于研究笔记的证据分级(已检索 / 标识级 / 未验证-领域知识 / 推论)

---

## 执行摘要

本报告评审如下方法设计的创新性与可行性:给定一个位点的 GWAS 汇总统计(beta、se)与多个 LD 参考面板(如 EUR/AFR 等 + bootstrap),该方法(①)合成一组代表"LD 不确定"的 LD 矩阵;(②)对每个 LD 矩阵独立运行 SuSiE 得到逐变异 PIP;(③)计算两个聚合量——非一致度分数 score(j)=1−mean PIP(j)(用于 C1 覆盖集合)与领先集中度 c=max_j freq(领先变异=j)(用于 C2 三态判定);(④)输出两个结果:C1 覆盖集合 {变异: score ≤ q̂}(声称含真凶概率 ≥1−α,自谓"分布自由")与 C2 三态标签(稳健 c≥0.9 / 脆弱 0.4≤c<0.9 / 不可识别 c<0.4)。

**核心结论(预先声明:以下为基于文献证据的评审判断,详见正文):**

1. **动机不新(证据充分)**。LD 参考面板不匹配损害精细定位(假阳性上升、"不稳定可信集")是文献与官方文档公认的事实;官方 SusieR 文档已有针对该问题的模型化校正(rss_mismatch),FunGen-xQTL 协议明确警示"LD 参考应与 GWAS 人群匹配,不匹配会产生不稳定可信集"。以"LD 不确定性问题存在"作为卖点不构成贡献。

2. **"LD 不确定性建模"存在直接先例(rss_mismatch 单一面板校正;SuSiEx/MultiSuSiE 多面板联合建模)**。被评审组合的机制差异点是"面板间独立推断后聚合 PIP"+"把 LD 当可重采样的随机量",而非"意识到 LD 不确定"。该特定编排(多面板 × bootstrap × 逐矩阵 SuSiE × 聚合 PIP)在本次检索范围内未找到同名直接先例(SUSIE 的集成变体 SuSiNE 是"多初始点"集成,维度不同)。

3. **C1 的"分布自由 1−α 覆盖集合"面临两个必须正面回答的理论缺口**:
   - **选择依赖问题**:共形推断文献明确边际覆盖保证在"选择后聚焦单位"上失效(selection-conditional coverage)。C1 的"真凶 ∈ 集合"恰是被挑选/领先变异的聚焦场景;若无条件覆盖论证,1−α 声明在理论上不成立。此为本方法**最大的理论风险点**。
   - **分布自由性的来源**:Model-X knockoffs 式的"分布自由"要求 X(基因型/LD)分布已知;而被评审方法以"LD 未知"为出发点。分布自由性由哪一步、在什么交换性假设下成立,方法设计需正面回答。另 Wu 等(PLOS Comp Biol)显示现有贝叶斯可信集覆盖在模拟中"过度保守",达到 1−α 不足为奇,真正的贡献点是**紧致性/功效与覆盖的权衡**。

4. **C1 的"校准阈值 q̂"有成熟程序先例**(Syring & Martin 标量校准、bootstrap 覆盖校准、CP4SBI conformal 校准可信集),因此"校准到名义覆盖"本身不新;潜在新颖点仅剩"校准对象 = 跨多 LD × bootstrap 聚合 PIP"这一具体编排。

5. **C2 三态判定新颖性相对更高,但阈值动机缺文献支撑**。"不稳定可信集"(官方协议)、stability-guided fine-mapping(eLife)、Replication Failure Rate 重采样一致性(NG 2023)、以及(stability selection 的"选择频率"概念、部分识别理论、带弃权分类)构成概念先例;但"稳健/脆弱/不可识别"命名与 0.9/0.4 阈值在检索范围内无直接先例——需要方法方提供阈值动机或敏感性分析。

6. **可行性:方法学上可行,工程上无直接证据**。多面板输入有工程先例(SuSiEx/MultiSuSiE);SuSiE 单区段扩展到数千 SNP 是实务共识,SuSiE 2.0 报告 summary-stats 应用提速最高 5 倍;公开参考面板可得(1000 Genomes / UKBB-LD AWS 开放数据 / All of Us)。但"多面板 × bootstrap × SuSiE 聚合"整体管线的运行成本无文献实测;数量级估算为 K(面板)×B(bootstrap)×L(位点)次 SuSiE 运行。**必须澄清 bootstrap 的重采样语义**(重采样面板个体 vs 参数扰动)——对只有汇总统计的输入,标准 bootstrap 无直接作用对象。

---

## 1. 现状:精确化定位方法与 LD 不确定性

### 1.1 主流方法把 LD 当作"已知且固定"的输入

- SuSiE(Wang et al. 2020,JRSS-B)以 z 统计量 + LD 矩阵 R 为输入,输出逐 SNP PIP 与名义覆盖 ≥1−α 的可信集(CS);方法本身不把 R 视为随机量。【未验证-领域知识;关键事实待核:原论文卷期与模拟细节】
- FINEMAP(Benner et al. 2016, Bioinformatics)、CAVIAR(Hormozdiari et al. 2014, Genetics)同为"边际 z + 参考 LD"框架,LD 视为精确已知。【未验证-领域知识】
- 结论:主流方法不做"LD 不确定性传播";对 LD 错误的处理普遍停留在"换更匹配的面板/敏感性模拟"。这是被评审方法定位的对照基线。【推断,基于上述条目】

### 1.2 LD 参考面板不匹配是有据可查的公认问题

- SusieR 官方文档《Modeling and Accounting for LD Reference Mismatch in Summary Statistics Fine-mapping》:不匹配"inducing high false positives in fine-mapping",并提供模型化校正(susie_rss)。【已检索,置信度高】https://stephenslab.github.io/susieR/articles/rss_mismatch.html
- 配套诊断 vignette 用于识别汇总统计与 LD 不一致(等位基因编码伪影等)。【已检索,置信度高】https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html
- FunGen-xQTL 协议:"The LD reference should match the GWAS ancestry and genome build. A mismatch can produce distorted z-score/LD relationships, **unstable credible sets** or apparently strong signals"——"不稳定可信集"的表述已见于公开协议,是被评审方法 C2 动机的直接文字对应物。【已检索,置信度高】https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html
- Weissbrod 等(bioRxiv 807792,即 PolyFun 预印本)用模拟考察 in-sample LD vs 外部参考面板对 fine-mapping 的影响。【已检索,置信度中】
- RSparsePro(bioRxiv 2024.10.29.620968)专门针对 LD mismatch 的稳健精细定位(概率图模型 + 变分推断)。【标识级,正文未核验】
- Benner 等《Prospects of Fine-Mapping ...》(AJHG 2017)系统评估参考面板 LD vs 个体水平 GWAS LD。【已检索,置信度中】https://pubmed.ncbi.nlm.nih.gov/28942963/

### 1.3 "多面板 / LD 不确定性建模"先例

- **单面板参数化建模**:SusieR rss_mismatch 官方校正(单一参考面板 + 模型内校正参数化)。【已检索,置信度高】
- **多面板联合建模**:SuSiEx(Nature Genetics 2024;github.com/getian107/SuSiEx)对每个祖源使用各自汇总统计与 LD 矩阵,联合推断共享因果结构——"一个模型吃多面板",但非"面板间独立推断后聚合"。MultiSuSiE(All of Us 应用)是多祖源 SuSiE 扩展。【已检索/标识级,置信度中】
- **跨环境一致性**:多环境 knockoff filter 利用跨环境一致性识别稳健关联(PMC11022501)。【已检索,置信度中】
- **多初始点集成**:SuSiNE(multi-basin ensembling,Sciety 10.64898/2026.07.31.742084)用多起始点解决 LD 歧义/局部最优。集成维度(初始化)不同于被评审方法(多 LD 矩阵)。【已检索,置信度中】
- **未找到直接先例**:对 LD 矩阵做 bootstrap/重采样 → 逐矩阵跑 SuSiE → 聚合 PIP 的已发表方法,在本次检索范围内未找到。【缺席证据,不可绝对化】← 这是被评审方法最可能的新颖空间所在。

---

## 2. 现状:分布自由推断

### 2.1 分布自由变量选择在遗传学是成熟方向(knockoffs)

- Model-X knockoffs("Panning for Gold",JRSS-B)提供任意 Y|X 模型下的有限样本 FDR 控制,不假设误差分布。【已检索,置信度高】doi:10.1111/rssb.12265(作者归属待核:按领域知识应为 Candès, Fan, Janson & Lv;fixed-X 版才是 Barber & Candès 2015——T2 笔记指出 lead 记录有归属冲突,需核)
- 遗传学应用:多环境 knockoff(PMC11022501)、second-order group knockoffs for GWAS(PMC11639161)、Spatial Knockoff Bayesian variable selection in GWAS(arXiv:2408.10401)、HMM knockoffs(Candès 组;细节无 URL 待核)。
- **关键细微推断**:Model-X 的"分布自由"是相对 Y|X 的;它要求 X 的边际分布(即基因型-LD 结构)已知或可估——**knockoffs 并不自动免疫 LD 参考面板不确定**。这与被评审方法把"LD 不确定"作为起点、把分布自由性放在聚合/校准层的定位形成对照。【推断,置信度中】
- 控制目标不同:knockoff 控制 FDR(期望误选比例),不给出"真凶 ∈ 集合 ≥1−α"的覆盖保证。C1 的保证类型是覆盖,亲缘在 conformal/校准文献。【推断】

### 2.2 共形推断:覆盖保证与"选择依赖"问题

- 边际有效的 conformal 区间在"被选择/聚焦的单元"上会失效(selection bias),需要 selection-conditional coverage(par.nsf.gov/10595785)。**【C1 理论对话第一优先级对象,置信度高,已检索】**
- "conformal fine-mapping / conformal credible set for causal variants"的已发表方法:本次检索未发现。【缺席证据】
- 领域知识中的 conformal 谱系(split conformal、CQR、jackknife+、conformal p-values 等)未逐一核验,不引具体 URL。【未验证】

### 2.3 校准到名义覆盖:成熟先例

- Syring & Martin《Calibrating general posterior credible regions》(Biometrika):Monte Carlo 调节标量参数使可信区域达名义频率覆盖——与 C1"校准 q̂"程序最接近。【已检索,置信度高】https://academic.oup.com/biomet/article/106/2/479/5237467
- Bootstrap coverage calibration(arXiv:2606.25729)与 Calibrated Generalized Bayesian Inference(arXiv:2311.15485):重采样校准后验可信集是活跃方向。【已检索,置信度中】
- CP4SBI(Phil Trans R Soc A 2025):conformal 局部校准可信集(SBI 语境)。【已检索(提及),若引用需补 URL】
- 含义:达到名义覆盖的方法论是现成的,不构成新概念;新颖性须来自"校准对象/校准数据源"。【推断】

---

## 3. 现状:PIP 校准、脆弱性诊断与评估基准

### 3.1 覆盖与校准实证

- Wu 等(PLOS Comp Biol):模拟显示**大多数** fine-mapping 方法的 CS 覆盖概率**过度保守**。【已检索,置信度高】https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829
- 含义:① "达到 1−α"不是难点,过度保守意味着集合偏大、分辨率下降;② 对被评审方法:报告应同时报覆盖盈余(或精确校准)与集合大小/分辨率,只报"覆盖达标"不充分。【推断】

### 3.2 脆弱性 / 稳定性诊断先例

- "unstable credible sets"表述见官方协议(§1.2)。【已检索,置信度高】
- eLife《The impact of stability considerations on genetic fine-mapping》:stability-guided 思路已存在(PMC/elifesciences.org/articles/88039)。【已检索,置信度中】
- Replication Failure Rate(NG 2023,doi:10.1038/s41588-023-01597-3):down-sampling 重采样一致性作为 fine-mapping 校准/可复现性指标——与 C2"跨矩阵/跨 bootstrap 一致"思路同构。【已检索,置信度高】
- **最接近的形式先例(stability selection)**:Meinshausen & Bühlmann 的 stability selection 用子样本选择频率定义稳健变量——与 C2"领先集中度 = 跨矩阵领先频率"在形式上几乎一一对应。**该条目为未验证的领域知识,是评审 C2 新颖性之前必须核实的最高优先文献**。【未验证,需补检】
- 其他概念近亲:部分识别理论(Manski 等,"不可识别"状态的形式化)、E-value 敏感性分析(VanderWeele & Ding)、带弃权/模糊区的分类。【未验证,领域知识】
- **0.9/0.4 阈值与三态命名**:无直接先例。需方法方提供动机或敏感性分析。【缺席证据】

### 3.3 评估基准惯例

- 模拟基准:真实 LD(1000G 或样本内基因型)模拟 z 值、植入已知因果变异;指标 = CS 经验覆盖 vs 标称 1−α、CS 大小、PIP 分箱校准曲线、含真变异精确度。【未验证-领域知识,置信度中】
- 真实数据基准:实验支持 eQTL(GTEx、eQTL Catalogue)、UKB 已知位点、FM-eQTL 性质基准。【未验证-领域知识,置信度中】
- 数据可得性:1000 Genomes Phase 3 五超族群为常用公开面板;UKBB-LD(AWS Open Data,arn:aws:s3:::broad-alkesgroup-ukbb-ld,N≈337K 英裔,2763 个 3Mb 区域)与 All of Us LD 面板公开可得(registry.opendata.aws/ukbb-ld;Zenodo 16923734)。【已检索,置信度高】

---

## 4. 可行性分析

### 4.1 组件可行性

- 多面板输入:工程先例充分(SuSiEx/MultiSuSiE 均输入多个族群 LD 面板)。【已检索】
- SuSiE 单区段扩展到数千 SNP:实务共识(未验证经验值);SuSiE 2.0 预印本(bioRxiv 2025.11.25.690514)称 summary-stats 应用提速最高 5 倍(正文未核验)。【标识级/未验证】
- 运行次数:K(面板)×B(bootstrap)×L(位点)次 SuSiE;若 K≈5、B≈50–100、L≈100–1000,则约 2.5 万–100 万次;按单次秒–分钟级、集群并行 100–1000 核,小时到天级(纯数量级估算,非测量值)。【推论,置信度低-中】
- 内存:L=10^4 SNP 的稠密 p×p 双精度 LD ≈ 0.8GB/个;多矩阵需注意存储与 IO。【推论,置信度低-中】

### 4.2 必须澄清的设计问题(bootstrap 语义)

- 输入只有汇总统计(beta、se)时,标准 bootstrap(重采样观测)无直接作用对象;"对 LD 矩阵做 bootstrap"需明确语义:重采样面板个体(需要面板个体级基因型,1000G 可提供;面板 ~300–660 人/族群,决定 LD 估计噪声)vs 对 z 统计量/LD 做参数扰动。【推论,置信度中】
- 该语义同时决定统计有效性与实现成本,是被评审方法可行性的核心质疑点。【推断】

---

## 5. 创新性判定

### 5.1 C1(覆盖集合)判定:**弱-中创新,理论风险高**

| 维度 | 判定 | 依据 |
|------|------|------|
| 问题动机 | 已有(不新) | §1.2 官方文档/协议 |
| "LD 不确定性建模" | 已有先例 | rss_mismatch 校正;SuSiEx 多面板 |
| "校准阈值 q̂ 达名义覆盖" | 已有程序先例 | Syring & Martin;bootstrap 校准 |
| 具体编排(多面板×bootstrap×聚合PIP) | 未见直接先例 | 缺席证据 |
| "分布自由"声称 | **需正面论证来源** | §2.1/2.3 |
| 1−α 覆盖声称 | **受选择依赖问题威胁** | §2.2 selection-conditional coverage |
| 紧致性/功效权衡 | 报告义务 | §3.1 Wu et al. 过度保守 |

判定文字:C1 属于"已知构件的新组合",新颖程度取决于能否在 (a) 选择条件覆盖、(b) 分布自由性的来源两点上提供新的理论论证,以及在模拟中展示**优于"单面板+校正"(rss_mismatch)与"多面板联合"(SuSiEx)的紧致性增益**(而非仅覆盖达标)。

### 5.2 C2(三态)判定:**中创新,但需阈值论证**

- 概念先例充分(不稳定可信集、stability-guided、RFR、stability selection 选择频率),说明"诊断脆弱性"不新;新的是**以跨面板领先集中度做三态自动分级**这一输出形式(未找到直接先例)。
- 风险:0.9/0.4 阈值无文献依据;三态命名首次出现;需方法方提供阈值动机、敏感性分析,并与部分识别(stability selection/识别集)理论对话。
- 提示:stability selection(Meinshausen & Bühlmann 2010)形式几乎同构,若方法方未引用,将被视为"重新发明或有遗漏"。

### 5.3 总体判定

**创新性:组合型(低-中)。可行性:方法学上可行,工程上未验证。** 该设计的骨架(多面板、聚合、校准、稳定性诊断)均有先例;独立成分级未发现新的统计概念。可能成立的贡献是"面向 LD 不确定性、带覆盖性质的集合 + 位点级脆弱性诊断"这一**产品化组合**,以及与之配套的算法细节(如集中度计算、q̂ 校准的实现)。作为论文选题,其能否成立取决于:理论层面解决选择依赖与分布自由来源;实证层面与 rss_mismatch 校正、SuSiEx、单面板 SuSiE 做 head-to-head,并量化紧致性/覆盖权衡;工程层面给出 bootstrap 语义与实测成本。

---

## 6. 证据支撑的注意事项与分歧

- **作者归属分歧**:lead 检索记录将 "Panning for Gold"(Model-X)记为 Barber & Candès;T2 笔记按领域知识认为应为 Candès, Fan, Janson & Lv,JRSS-B 2018(fixed-X 为 Barber & Candès 2015 Ann. Statist.)。**引用前必须核实**(PLEASE-CITE)。【未验证】
- **过度保守 vs 名义覆盖**:Wu et al. 报告覆盖过度保守,SuSiE 原论文声称名义 ≥1−α 覆盖;二者不直接冲突(名义性质 vs 实证行为),但对 C1 的"保证"口径有直接影响。【矛盾记录】
- **缺失验证**:大量"已检索"来源来自搜索快照摘要,正文未逐页核读;SuSiE 2.0/MultiSuSiE/RSparsePro/eLife 88039/RFR 仅标识级;领域知识条目全部未验证。所有引用必须经 cited 阶段逐条核实(PLEASE-CITE)。

---

## 7. 开放问题(Open Questions)

1. C1 的 1−α 是逐位点条件保证还是边际保证?如何回避/论证 selection-conditional 失效?(最高优先)
2. "分布自由"由哪一步提供?采用何种交换性/重采样论证?与 Model-X 要求"X 分布已知"的张力如何化解?
3. bootstrap 的确切对象与算法(个体级重采样 vs 参数扰动)?对小面板(300–660 人)的 LD 噪声影响如何?
4. 0.9/0.4 阈值的动机与敏感性;三态与 stability selection / 部分识别理论的关系?
5. 与 rss_mismatch 校正版、SuSiEx、单面板 SuSiE 的 head-to-head 实验设计(紧致性/覆盖/功效)?
6. 计算成本实测(每阶段耗时/内存)与并行策略?
7. q̂ 校准的统计准则(用模拟数据校准?用观测数据交叉验证?校准数据集与目标位点的同一性)?

---

## 8. 初步来源清单(待 cited 阶段核实)

- SusieR rss_mismatch 文档 — https://stephenslab.github.io/susieR/articles/rss_mismatch.html
- SusieR diagnostic vignette — https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html
- FunGen-xQTL summary stats vignette — https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html
- Wu et al., PLOS Comp Biol — https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829
- Selection-conditional coverage — https://par.nsf.gov/servlets/purl/10595785
- Model-X knockoffs — https://doi.org/10.1111/rssb.12265
- Multi-environment knockoff filter — https://pmc.ncbi.nlm.nih.gov/articles/PMC11022501/
- Group knockoffs GWAS — https://pmc.ncbi.nlm.nih.gov/articles/PMC11639161/
- Spatial knockoff GWAS — https://doi.org/10.48550/arxiv.2408.10401
- ESNN — https://pmc.ncbi.nlm.nih.gov/articles/PMC9234235/
- Syring & Martin, Biometrika — https://academic.oup.com/biomet/article/106/2/479/5237467
- Bootstrap coverage calibration — https://arxiv.gg/abs/2606.25729
- Calibrated GBI — https://arxiv.org/html/2311.15485v3
- RFR, NG 2023 — https://doi.org/10.1038/s41588-023-01597-3
- eLife stability — https://elifesciences.org/articles/88039
- SuSiEx paper — https://doi.org/10.1038/s41588-024-01870-z
- MultiSuSiE — https://pmc.ncbi.nlm.nih.gov/articles/PMC13091671/
- RSparsePro — https://doi.org/10.1101/2024.10.29.620968
- SuSiE 2.0 — https://www.biorxiv.org/content/10.1101/2025.11.25.690514v1
- Weissbrod et al. — https://doi.org/10.1101/807792
- Benner AJHG 2017 — https://pubmed.ncbi.nlm.nih.gov/28942963/
- SuSiNE — https://sciety.org/articles/activity/10.64898/2026.07.31.742084
- UKBB-LD — https://registry.opendata.aws/ukbb-ld/
- All of Us / UKB LD 面板 — https://doi.org/10.5281/zenodo.16923734