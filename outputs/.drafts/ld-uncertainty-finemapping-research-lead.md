# Research Notes (Lead/Direct): LD Uncertainty Finemapping

来源:主代理 web_search(alpha_search 因需要 `alpha login` 而 BLOCKED)
日期:2026-09-13

## 搜索词记录

1. `SuSiE fine-mapping LD reference panel mismatch accuracy`
2. `Model-X knockoffs GWAS fine-mapping distribution-free variable selection`
3. `conformal prediction coverage guarantee variable selection genetics fine-mapping`
(后续补充见下文;子代理 T1/T2/T3 笔记另存)

## 关键发现(带来源)

### LD 参考面板不匹配是已知问题,且已有建模

- **SuSiE-RSS 官方文档《Modeling and Accounting for LD Reference Mismatch in Summary Statistics Fine-mapping》**(https://stephenslab.github.io/susieR/articles/rss_mismatch.html):
  - 明确:当 GWAS 汇总统计与外部参考面板 LD 不匹配时,会"inducing high false positives in fine-mapping"
  - 提供了针对有限/不匹配参考面板的模型化校正方法(susie_rss)
- **配套诊断 vignette**(https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html):用于诊断汇总统计与 LD 参考不一致(等位基因编码伪影等)
- **FunGen-xQTL 协议 vignette**(https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html):"The LD reference should match the GWAS ancestry and genome build. A mismatch can produce distorted z-score/LD relationships, unstable credible sets or apparently strong signals" —— 注意原文已出现"不稳定可信集(unstable credible sets)"概念,是被评审方法 C2(脆弱性)的动机来源之一。
- **Weissbrod et al.(bioRxiv 807792)《Functionally-informed fine-mapping...heritability》**:用模拟研究 LD mismatch 对 fine-mapping 性能的影响(目标样本 in-sample LD vs 外部参考面板)

### 覆盖保证 / 校准(C1 相关)

- **Wu et al.《Improving the coverage of credible sets in Bayesian genetic fine-mapping》PLOS Comp Biol(https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829; bioRxiv 781062)**:模拟显示大多数 fine-mapping 方法报告的 credible set 覆盖概率**过度保守(over-conservative)**。直接对标 C1 声称的 1−α 覆盖保证。
- **《Confidence on the focal: conformal prediction with selection-conditional coverage》(https://par.nsf.gov/servlets/purl/10595785)**:边际有效的 conformal 区间在"选择后聚焦单位"上可能失效(selection bias),提出 select-conditional coverage —— 与被评审方法 C1(选择后覆盖)理论层面高度相关。

### 分布自由变量选择(knockoffs,应用于遗传学已是成熟方向)

- **Model-X Knockoffs**:Barber & Candès《Panning for Gold: Model-X Knockoffs for High Dimensional Controlled Variable Selection》(doi:10.1111/rssb.12265)——分布自由 + FDR 控制,高维变量选择
- **多环境 knockoff filter**(https://pmc.ncbi.nlm.nih.gov/articles/PMC11022501/):利用跨环境一致性找稳健关联 —— 注意"跨环境一致"思路与被评审方法"跨 LD 面板一致"相似
- **Second-order group knockoffs in GWAS**(https://pmc.ncbi.nlm.nih.gov/articles/PMC11639161/):conditional testing + FDR,专门针对 GWAS
- **Gene Hunting with Knockoffs for HMM**(Candès 组 pdf)
- **Spatial Knockoff Bayesian Variable Selection in GWAS**(arXiv:2408.10401):贝叶斯变量选择 + knockoffs

### 聚合 / ensemble 思路

- **ESNN《Uncertainty quantification in variable selection for genetic fine-mapping using Bayesian neural networks》(https://pmc.ncbi.nlm.nih.gov/articles/PMC9234235/)**:ensemble of single-effect NN,泛化 SuSiE,对"哪个变量被选中"做不确定性量化 —— 与被评审方法"聚合 PIP 不确定性"思路最近似的方法学邻居。

## 记录的可信度说明

以上均为搜索结果快照中的来源 URL(未全部逐一打开验证正文);子代理笔记与后续 verification 阶段将进一步核实。置信度标注待 cited 阶段完成。

## 待补充方向

- C2 三态判定(sensitivity/robustness 诊断)文献
- 多参考面板 LD 联合(meta-LD、multi-ethnic)先例——T1 子代理负责
- 计算可行性(SuSiE runtime)——T3 子代理负责

---

## 第二轮搜索补充(2026-09-13,响应 mtzg7mjvkx16b4 + mtzg8uk09xb2td)

### LD 面板敏感性(实证证据)

- **Benner et al.《Prospects of Fine-Mapping ... using Summary Statistics from GWAS》AJHG(https://pubmed.ncbi.nlm.nih.gov/28942963/)**:系统评估参考面板 LD 估计 vs 原始个体水平 GWAS 数据的 LD 对 fine-mapping 的影响 —— 早期重要实证证据。
- **RSparsePro《Robust fine-mapping in the presence of linkage disequilibrium mismatch》(https://doi.org/10.1101/2024.10.29.620968)**:专门针对 LD mismatch 的稳健精细定位(概率图模型 + 变分推断)。
- **mapgen(FunGen-xQTL)+ UKBB LD 矩阵**:用预计算 UKBB LD 做 fine-mapping 与 LD mismatch 诊断(xinhe-lab.github.io/mapgen)。

### 多族群/集成"多 LD"先例

- **SuSiEx(Nature Genetics, doi:10.1038/s41588-024-01870-z;github getian107/SuSiEx)**:across-ancestry fine-mapping,联合多个祖源参考面板 —— "多面板"的直接先例。
- **MultiSuSiE(https://pmc.ncbi.nlm.nih.gov/articles/PMC13091671/)**:多祖源 SuSiE,因果效应可随祖源变化,Simulations 显示比单一 Eur SuSiE 更高功效。
- **SuSiNE(Sciety 10.64898/2026.07.31.742084)**:"multi-basin ensembling"应对 LD 歧义/局部最优 —— 集成思想可对标。
- **eLife《The impact of stability considerations on genetic fine-mapping》(https://elifesciences.org/articles/88039)**:稳定性导向补充(stability-guided),源于残差混杂未完全消退。

### PIP 校准 / 覆盖保证(C1 理论对标)

- **Wu et al. PLOS Comp Biol**: credible set 覆盖率过度保守(bioRxiv 781062 / journal.pcbi.1007829)。
- **Replication Failure Rate(RFR)《Improving fine-mapping by modeling infinitesimal effects》NG 2023(doi:10.1038/s41588-023-01597-3)**:通过 down-sampling 评估 fine-mapping 一致性/校准的指标 —— 与"跨重采样一致"诊断强相关。
- **CP4SBI(Phil Trans R Soc A 2025)**:conformal 局部校准 credible sets(SBI 语境) —— 分布自由校准先例。
- **Syring & Martin《Calibrating general posterior credible regions》Biometrika(doi 10.1093/biomet/asy054, https://academic.oup.com/biomet/article/106/2/479/5237467)**:Monte Carlo 调标量参数使 credible region 达名义频率覆盖。
- **Bootstrap coverage calibration(arXiv 2606.25729)**:广义后验 credible set 的 bootstrap 覆盖校准理论。
- **Calibrated Generalized Bayesian Inference(arXiv 2311.15485)**:Gibbs 后验校准。

### 计算可行性

- **SuSiE 2.0(doi:10.1101/2025.11.25.690514)**:模块化重设计,summary stats 应用提速最多 5x;含 SuSiE-ash 改善校准——“强信号与中等效应共存”。
- **UKBB-LD(AWS Open Data,arn:aws:s3:::broad-alkesgroup-ukbb-ld;N≈337K 英裔;2763 个 3Mb 区域)**:公开预计算 LD;All of Us 面板亦提供(Zenodo 16923734)。

### 缺口记录

- C2 阈值(0.9/0.4)的直接对标文献未见;三态标签未找到已命名等价物。
- "跨 LD 面板 PIP 聚合"作为一个正式方法论文(Aggregate-PIP / bagging SuSiE over LD)未见同名文献;SuSiNE 集成维度(多起始点)与被评审方法(多 LD)不同。
- 逐 LD 矩阵跑 SuSiE 的端到端计算成本(Turing/逐园)未见公开基准数字。
---

## 第三轮:作者归属验证(2026-09-13)

- source_check + get_search_content 确认:JRSSB-CFJL18.pdf 内容原文 "E. Candès, Y. Fan, L. Janson and J. Lv"(字母排序,Received January 2017, Final revision November 2017)→ **Model-X "Panning for Gold" 作者 = Candès, Fan, Janson & Lv,JRSS-B 2018**。
- 同结果中 "Controlling the false discovery rate via knockoffs"(doi:10.1214/15-AOS1337,Ann. Statist.)= **fixed-X 版,Barber & Candès 2015**。
- 修正:lead 初稿将 Model-X 记为 "Barber & Candès" 有误;cited/最终版必须以 CFJL18 为准。
