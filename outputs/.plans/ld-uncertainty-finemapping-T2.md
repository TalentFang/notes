# Researcher Brief T2: 分布自由推断(变量选择/遗传学)

你是统计推断专项调研员,负责"分布自由(distribution-free)"方向。被评审方法声称构建一个"真凶变异以 ≥1−α 概率被包含"的因果变异集合,且自称是分布自由的(distribution-free,不依赖错误分布假设)。你要查清这个方向已有的文献基础。

## 背景

被评审方法:C1 输出一个覆盖集合(对 PIP 聚合分数取阈值),声称保证真因果变异以 ≥1−α 概率在其中,且声称"分布自由"(对 LD 不确定性不做分布假设)。C2 输出三态判定。

## 你要回答的问题

1. **分布自由的变量选择**:Model-X knockoffs(Barber & Candès 2015; Candès et al. 2018)是不依赖分布假设的高维变量选择,是否已被应用于遗传精细定位/GWAS?(文献:knockoffGWAS、knockoffs for eQTL、FX knockoffs)
2. **共形推断(conformal prediction)**:conformal inference 如何给出"分布自由"的覆盖保证(有限样本、不依赖分布假设);是否有 conformal variable selection / conformal credible sets / conformal p-values 的工作;在统计遗传学或生物医学中是否已有应用。
3. **集合构造与覆盖保证**:文献中"保证真信号以 ≥1−α 概率在集合内"的构造有哪些?(conformal CS、分位数/秩方法、重采样校准、bootstrap calibration、knockoff 的 offset 参数)。
4. **PIP 是否是"分布自由"对象**:SuSiE 的 PIP 本身基于贝叶斯假设;把 PIP 经 bootstrap/多面板聚合后做阈值处理,这在文献里属于什么(聚合推断、bagging、鲁棒 PIP)?是否已有"bagging SuSiE"或"整合多个 LD"的类似尝试?
5. **三态判定(稳健/脆弱/不可识别)**:识别强度、sensitivity analysis、扰动/稳定性分析中是否有类似"三态"分类;conformal 或 knockoff 文献中是否有"无法判定"状态。

## 搜索建议

- "Model-X knockoffs fine-mapping GWAS"
- "conformal inference variable selection"
- "distribution-free credible sets / conformal prediction genetics"
- "bootstrap calibration coverage guarantee variable selection"
- "knockoffGWAS eQTL"
- "uncertainty quantification fine-mapping PIP calibration"
- "conformal p-values subset selection"
- 也用 alpha/论文搜索思路:搜 arXiv/biorxiv 相关标题

## 输出要求

写到你被指定的研究笔记文件,Markdown:
- `## 现状摘要`
- `## 发现列表`(声明 + URL + 置信度)
- `## 对评审对象的启示`:"分布自由覆盖集合"是否已有直接先例(尤其 conformal/knockoff 应用于 finemapping);C1 的 1−α 保证如何与现有理论对话;C2 三态是否有概念先例
- `## 缺口`
- 只引用实际见过的来源,不编造 URL。