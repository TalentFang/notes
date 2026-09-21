# 研究简报 T1:精细定位方法与 LD 不确定性现状

> 检索说明:本简报证据分两类——
> (1) **已检索**:来自父会话 web_search 结果(lead 文件,URL 真实出现在上游搜索结果中;上游未逐一打开验证正文,正文细节来自搜索快照摘要,个别细节以"待核实"标注);
> (2) **未验证**:研究员领域知识补充,无 URL、不编造文献细节,置信度相应下调。
> 整理日期:2026-09-13。

## 现状摘要

主流精细定位方法(SuSiE、FINEMAP、CAVIAR、DAP、SuSiEx)都把 LD 矩阵当作**已知且固定的输入**(来自样本内基因型或与 GWAS 人群匹配的外部参考面板),并不在推断中显式传播"LD 本身有误差"这一不确定性。LD 参考面板不匹配是文献公认的问题:会诱发假阳性、扭曲 z-LD 关系、产生不稳定可信集(credible sets);官方 SusieR 文档已提供针对参考面板不匹配的**模型化校正**方案(单一参考面板 + 校正参数化)。多面板/跨族群用法已有先例(SuSiEx 按族群各用各的 LD、共享因果后验),"跨环境一致性"思路在多环境 knockoff 中也已出现;但在已收集的证据范围内,**没有找到"对多个 LD 矩阵做 bootstrap/重采样 → 逐矩阵跑 SuSiE → 聚合 PIP"的直接先例**。SuSiE 的 credible set 名义覆盖 1−α,但模拟研究发现多数方法实际覆盖"过度保守",LD 错误设定下 PIP/CS 的校准行为缺乏系统性量化证据。

## 发现列表

### Q1:主流方法对 LD 输入的要求与假设

- **F1**【SuSiE 假设 LD 已知;R 来自样本内或匹配参考面板;输出 PIP + 1−α credible sets】。SuSiE(Wang et al. 2020,"A simple new approach to variable selection in regression, with application to genetic fine mapping",JRSS-B;arXiv:1903.05792)以 z 统计量 + LD 矩阵 R 为输入,含 AAS/ASIP 两种变体(单效应回归 vs 迭代多效应),对每个效应输出 SNP 的 PIP,并构造名义覆盖率 ≥1−α 的可信集。方法本身不把 R 视为随机量。【来源:领域知识,无 URL(原始论文引用信息待核实)】**【未验证,置信度:中】**。
- **F2**【FINEMAP 假设 LD 已知;参考面板 LD 为标准输入】。FINEMAP(Benner et al. 2016, Bioinformatics,"FINEMAP: efficient variable selection using summary data and external data")用 z 分数 + 外部 LD(通常 1000 Genomes 匹配人群)做贝叶斯变量选择(shotgun stochastic search);后续版本也支持样本内基因型。软件与论文均建议参考面板与 GWAS 人群匹配(具体建议原文待核实)。【来源:领域知识,无 URL(卷期页码待核实)】**【未验证,置信度:中】**。
- **F3**【CAVIAR 假设 LD 已知;用 z 分数 + 参考 LD 计算每个变体因果概率】。CAVIAR(Hormozdiari et al. 2014, Genetics,"Identifying causal variants at loci with multiple signals of association")及 CAVIARBF(Chen et al. 2015, Genetics,近似贝叶斯 + 边际检验统计量)都以边际 z + LD 为输入,LD 视为精确已知。【来源:领域知识,无 URL(卷期页码待核实)】**【未验证,置信度:中】**。
- **F4**【DAP 系列同为"z + 参考 LD"框架】。DAP/DAP-G(Wen 及同事构建,贝叶斯多 SNP 模型,支持汇总统计 + LD;FDR 控制版本为 DAP-G)——具体引用信息待核实。【来源:领域知识,无 URL】**【未验证,置信度:低-中】**。
- **F5**【SuSiEx 是"多 LD 矩阵 + 单一共享后验"的跨族群框架】。SuSiEx 以 SuSiE 为内核,对每个族群使用各自的汇总统计与各自的 LD 矩阵,联合推断共享的因果结构。它是"一个模型吃多面板 LD"的先例,但不是"面板间独立推断后聚合"。【来源:领域知识;预印本/正式发表信息待核实,无 URL】**【未验证,置信度:中(存在性把握较高,细节待核实)】**。
- **F6**【综合判断:主流方法默认 LD 已知,不把 LD 不确定性纳入 PIP/CS 传播;对 LD 错误的处理普遍停留在"换更匹配的面板/做敏感性模拟",而非概率化建模】。此为 F1–F5 的合并推断。【来源:研究员推断,基于 F1–F5】**【推断,置信度:中(受缺口限制)】**。

### Q2:LD 参考面板不匹配的影响

- **F7**【不匹配诱发高假阳性;官方已有模型化校正】。SusieR 官方文档《Modeling and Accounting for LD Reference Mismatch in Summary Statistics Fine-mapping》明确:GWAS 汇总统计与外部参考面板 LD 不匹配会"inducing high false positives in fine-mapping",并提供针对有限/不匹配参考面板的建模与校正方法(susie_rss)。【来源:已检索,https://stephenslab.github.io/susieR/articles/rss_mismatch.html (正文细节来自上游快照,方法学术语待核实)】【**已检索,置信度:高**】。
- **F8**【配套诊断工具存在】。SusieR 诊断 vignette 用于识别汇总统计与 LD 参考不一致(等位基因编码伪影等)。【来源:已检索,https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html】【**已检索,置信度:高**】。
- **F9**【实操协议明确要求 LD 与 GWAS 人群/基因组版本匹配,否则可信集不稳定】。FunGen-xQTL 协议 vignette:"The LD reference should match the GWAS ancestry and genome build. A mismatch can produce distorted z-score/LD relationships, unstable credible sets or apparently strong signals"——注意"unstable credible sets(不稳定可信集)"这一表述已存在于文献,是被评审方法"LD 扰动下 CS 稳定性"动机的直接文字对应物。【来源:已检索,https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html】【**已检索,置信度:高**】。
- **F10**【权威模拟研究考察过 LD mismatch 对 fine-mapping 性能的影响】。Weissbrod et al. 预印本(bioRxiv 807792)《Functionally-informed fine-mapping…heritability》(即 PolyFun 论文预印本)用模拟比较目标样本 in-sample LD vs 外部参考面板对 fine-mapping 性能的影响。【来源:已检索,标题与 ID 来自上游结果(bioRxiv 807792;正式发表为期刊版 PolyFun,具体卷期待核实)】【**已检索,置信度:中**】。
- **F11**【人群间 LD 差异是公认事实:EUR/AFR/EAS 面板不可互换,AFR LD 衰减更快、单倍型多样性更高,EAS/EUR 连锁块更大;1000 Genomes 多祖先面板的分层即为此设计】。若用错人群面板,等效于 F7 的不匹配场景。【来源:领域知识,无 URL(量化对比数据未验证)】【**未验证,置信度:低-中**】。

### Q3:"LD 不确定性建模"是否有先例

- **F12**【最接近的"LD 不确定性建模"先例 = SusieR rss_mismatch 官方校正】。它把"参考 LD 与目标人群不一致"显式纳入模型并做校正,属于 SuSiE 家族内部对 LD 不确定性的(参数化)建模——即"意识到并处理 LD 不确定性"这件事已存在,被评审方案的机制(多面板 + 重采样而非单面板参数化)才是可能差异点。【来源:已检索 F7 + 研究员解读】**【已检索+解读,置信度:中-高(机制细节待核实)**】。
- **F13**【多面板联合使用已有先例:SuSiEx 按人群各用各的 LD 矩阵,但共享单一后验,不是"面板间独立建档 + 聚合"】。因此"多 LD 面板进模型"不新,"聚合式使用"才是可能的差异化点。【来源:未验证 F5 + 推断】【**未验证,置信度:中**】。
- **F14**【"跨环境/跨面板一致性作为稳健性信号"在多环境 knockoff filter 中已出现】。多环境 knockoff filter(PMC11022501)利用跨环境一致性识别稳健关联——思路与被评审方案"跨 LD 面板一致"同族,但是 knockoff/FDR 框架而非贝叶斯 PIP 框架。【来源:已检索,https://pmc.ncbi.nlm.nih.gov/articles/PMC11022501/】【**已检索,置信度:中**】。
- **F15**【"对变量选择做不确定性量化 + 集成"最近邻居 = ESNN】。ESNN《Uncertainty quantification in variable selection for genetic fine-mapping using Bayesian neural networks》(PMC9234235)用 single-effect 神经网络集成泛化 SuSiE 并对"哪个变量被选中"做不确定性量化——集成聚合位置较近,但集成的对象是模型/初始化,而非 LD 矩阵。【来源:已检索,https://pmc.ncbi.nlm.nih.gov/articles/PMC9234235/】【**已检索,置信度:中**】。
- **F16**【bootstrap/重采样 LD 矩阵用于 fine-mapping:未找到直接先例】。在已收集证据(上游检索 + 领域知识)范围内,没有发现"对 LD 重采样(面板级或基因型级 bootstrap)→ 逐矩阵跑 SuSiE → 聚合 PIP"的已发表方法。【来源:检索未见;属"未找到"而非"不存在"】【**置信度:中(缺口,见下)**】。
- **F17**【LD 扰动/替换做稳健性测试是已知验证风格】。F7 的校正演示与 F10 的模拟都属于"改换/错配 LD 观察输出变化"的稳健性测试方式;但"注入核基噪声扰动 LD 后系统评估 PIP/CS 稳定性"的专门文献未找到。【来源:已检索 F7/F10 + 推断】【**置信度:中**】。

### Q4:SuSiE 输出在模型错误设定下的行为

- **F18**【credible set 名义覆盖 ≠ 实际覆盖;多数方法报告过度保守】。Wu et al.《Improving the coverage of credible sets in Bayesian genetic fine-mapping》(PLOS Comp Biol)模拟显示大多数 fine-mapping 方法报告的 credible set 覆盖概率过度保守(over-conservative)——直接关系到"1−α 覆盖保证"类声称(即使 LD 正确时也未必精确)。【来源:已检索,https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829;bioRxiv 781062】**【已检索,置信度:中-高**】。
- **F19**【LD 错误设定下假阳性上升 → PIP/CS 可信度受损】。F7 文档(不匹配 → 高假阳性)与 F9(不稳定可信集)是被评审方法脆弱性动机的直接证据。【来源:已检索,F7/F9】【**已检索,置信度:高**】。
- **F20**【SuSiE 对"单因果"假设的依赖:多效应时用 ASIP 迭代;但 LD 错误下 PIP 校准、CS 拆分/合并行为缺乏系统性研究】。SuSiE 原论文的可信集覆盖性质(名义 ≥1−α)在正确 LD 模拟下验证;对 LD 错误设定下覆盖率的量化评估,在已收集证据内只找到 F18/F19 间接证据。【来源:领域知识(原论文模拟细节待核实)+ 已检索 F18/F19】【**未验证+已检索,置信度:中**】。
- **F21**【选择后条件覆盖的理论警示】。conformal 预测文献《Confidence on the focal: conformal prediction with selection-conditional coverage》指出:边际有效的区间在"选择后聚焦单元"上可能失效,需 selection-conditional coverage——对被评审方案"选定 LD 面板/选定 CS 后的覆盖保证"构成理论层面的对照与风险提示。【来源:已检索,https://par.nsf.gov/servlets/purl/10595785】【**已检索,置信度:中**】。

## 对评审对象的启示

1. **动机不新,机制是可能的差异化点**。"LD 参考面板不匹配有害"(假阳性、CS 不稳定)已有多个权威来源(F7/F9/F10),被评审方案不应以"发现 LD 不匹配问题"为卖点。
2. **"LD 不确定性建模"本身已有直接先例**:SusieR 官方 rss_mismatch 文档(F7)就是 SuSiE 家族内对参考 LD 不匹配的模型化处理。被评审组合与它的区别是**机制路径**:官方做法 = 单一参考面板 + 模型内校正参数化;被评审做法 = 多面板 + bootstrap 重采样 + 面板间聚合。评审几乎必然要求 head-to-head 比较,这是必须准备的对标。
3. **"多个 LD 矩阵"不新**:SuSiEx(F5)已经是"每人群一个 LD 矩阵 + 联合共享后验";"多环境一致性"在多环境 knockoff(F14)中也已出现。被评审组合的新颖点应聚焦于:**不共享后验、逐 LD 矩阵独立推断后聚合 PIP**,以及**把 LD 当作可重采样的随机量显式传播不确定性**——此二者在已收集证据中没有直接先例(F16)。
4. **聚合不确定性量的最近邻居是 ESNN(F15)**:它做的是模型层面的集成不确定性,不是 LD 层面的;可在评审材料中作为"方法学邻居"划清边界。
5. **覆盖/校准主张要小心**:Wu et al.(F18)显示多数方法的报告覆盖过度保守;F21 提示"选择后覆盖"需要条件化保证。被评审方案若声称"聚合后覆盖仍为 1−α"或"选择后覆盖有效",需要额外理论或模拟证据,不能默认继承 SuSiE 的名义性质。
6. **输出形式的可能新颖点**:LD 扰动下 CS 稳定性/三态分级(与 F9"不稳定可信集"、F14"跨环境一致"呼应)——但需注意 F9 已把"不稳定"作为问题陈述过,新颖性在"如何建模与量化"而非"不稳定现象存在"。

## 缺口

- **未找到**(在已收集证据内,需后续定向补检索验证):
  - bootstrap/重采样 LD 矩阵用于 fine-mapping 的先例(F16);
  - 多参考面板 LD 的显式"meta-LD / 集成"方法(SuSiEx 是联合建模,非集成);
  - "对 LD 注入噪声/扰动 → PIP/CS 变化"的系统稳健性研究(F17);
  - SuSiE 在 LD 错误设定下 CS 覆盖率/校准的**定量**评估(只找到过度保守 F18 与假阳性 F19 两个间接证据);
  - 上游检索词中"bootstrap LD matrix / LD uncertainty genetic""multi-ethnic fine-mapping SuSiEx"等角度的结果未出现在 lead 文件中,属于未覆盖方向。
- **未验证的文献细节**(领域知识,无 URL,均待核实):SuSiE(F1)、FINEMAP(F2)、CAVIAR/CAVIARBF(F3)原始论文的卷期页码与模拟结论;DAP/DAP-G(F4)的规范引用;SuSiEx(F5)的预印本/正式发表信息;rss_mismatch(F7)背后的正式论文与方法学术语;F11 的人群 LD 差异量化数据。

## 来源清单

**已检索(来自上游 lead 文件,可引用):**
- SusieR 官方文档:Modeling and Accounting for LD Reference Mismatch (https://stephenslab.github.io/susieR/articles/rss_mismatch.html)
- SusieR 诊断 vignette (https://stephenslab.github.io/susieR/articles/susierss_diagnostic.html)
- FunGen-xQTL summary stats fine-mapping 协议 (https://statfungen.github.io/xqtl-protocol/summary_stats_finemapping_vignette.html)
- Weissbrod et al., Functionally-informed fine-mapping…(bioRxiv 807792,上游未给出 URL 字符串)
- Wu et al., Improving the coverage of credible sets…, PLOS Comp Biol (https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1007829;bioRxiv 781062)
- Confidence on the focal: conformal prediction with selection-conditional coverage (https://par.nsf.gov/servlets/purl/10595785)
- Barber & Candès, Panning for Gold: Model-X Knockoffs (doi:10.1111/rssb.12265)
- 多环境 knockoff filter (https://pmc.ncbi.nlm.nih.gov/articles/PMC11022501/)
- Second-order group knockoffs in GWAS (https://pmc.ncbi.nlm.nih.gov/articles/PMC11639161/)
- Spatial Knockoff Bayesian Variable Selection in GWAS (arXiv:2408.10401)
- ESNN: Uncertainty quantification in variable selection…BNN (https://pmc.ncbi.nlm.nih.gov/articles/PMC9234235/)
- Gene Hunting with Knockoffs for HMM(Candès 组,上游仅标题)

**未验证(领域知识,无 URL,引用细节待核实):** SuSiE(Wang et al. 2020)、FINEMAP(Benner et al. 2016)、CAVIAR(Hormozdiari et al. 2014, Genetics)、CAVIARBF(Chen et al. 2015)、DAP/DAP-G、SuSiEx、人群间 LD 差异(1000 Genomes 多祖先设计)。

## 下一步

1. 定向补检索(建议父会话或后续子代理):"bootstrap LD fine-mapping"、"multiple LD reference panels ensemble / meta-LD"、"SuSiE credible set coverage under LD misspecification"、"SuSiEx 正式引用"、"rss_mismatch 背后正式论文(非 vignette)"。
2. 评审材料准备:把"与 SusieR 官方 rss_mismatch 校正版的对比实验"和"单面板 vs 多面板聚合的增益量化"列为被评审方案必做验证项。

## 矛盾/争议点

- F18 显示大多数方法的报告覆盖"过度保守",而 SuSiE 原论文声称名义 1−α 覆盖——二者并不直接冲突(名义性质在正确设定下成立、实证偏保守),但对被评审方案"覆盖保证"声明的口径有直接影响,列为需注意的矛盾面,记录在案而非消解。