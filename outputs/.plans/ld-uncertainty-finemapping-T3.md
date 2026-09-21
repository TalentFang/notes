# Researcher Brief T3: 校准 / 脆弱性 / 基准与可行性

你是评估与基准专项调研员。被评审方法有两个输出:①覆盖集合(声称保证真凶 ≥1−α 在内),②位点级三态判定(稳健/脆弱/不可识别,基于"领先变异集中度"阈值 0.9/0.4)。你需要查清:**现有评估惯例、是否有"脆弱性/稳健性诊断"先例、以及计算可行性证据**。

## 背景

被评审方法:多个 LD 面板 + bootstrap → 逐矩阵 SuSiE → 聚合 PIP。C2 用"领先集中度 c = max_j freq(领先变异 = j)":c≥0.9 稳健;0.4≤c<0.9 脆弱;c<0.4 不可识别。

## 你要回答的问题

1. **精细定位的评估基准与惯例**:
   - 常用基准:已知因果位点(实验验证的 eQTL/GTEx、UKB 终点)、模拟(genetic simulation、个体水平数据模拟 LD)、FM-eQTL benchmark 等
   - 评价指标:credible set 覆盖/大小、PIP 校准/可靠性图、精确定位准确率、"是否包含真变异"
   - cs 报告惯例(SuSiE 输出 CS;如何评估 CS 覆盖)
2. **PIP 校准评估**:是否已有研究评估 SuSiE/FINEMAP PIP 的校准性(在真实或模拟数据);"PIP 跨 LD 面板/跨重采样不一致"是否被用作稳健性指标。
3. **脆弱性/稳健性诊断先例**:
   - genetic fine-mapping 的 sensitivity analysis、扰动 LD、跨面板一致性报告
   - 其他领域的三态判定(如 DAG 识别中"未识别/部分识别/完全识别"等)是否有类似分类思路
   - "fine-mapping robustness" 相关论文
4. **计算可行性**:
   - SuSiE 运行规模/时间的文献报告或经验(单 locus 数千 SNP 时的耗时、内存)
   - 多 LD × 多 bootstrap 的计算成本是否现实(bootstrap 次数、并行化)
5. **数据来源**:1000 Genomes 面板(各族群)、UKB LD 面板的可得性;这些是否在文献中被用于 fine-mapping。

## 搜索建议

- "fine-mapping evaluation benchmark credible set"
- "SuSiE PIP calibration"
- "fine-mapping robustness LD panel"
- "fine-mapping sensitivity analysis"
- "FM-eQTL benchmark SuSiE FINEMAP comparison"
- "1000 genomes LD reference panel fine-mapping"
- "computational cost SuSiE runtime fine-mapping"
- 搜索 SuSiE/FINEMAP 论文的 results/benchmarks 部分

## 输出要求

写到你被指定的研究笔记文件,Markdown:
- `## 现状摘要`
- `## 发现列表`(声明 + URL + 置信度)
- `## 对评审对象的启示`:①C1 覆盖保证如何用现有基准验证;②C2 三态判定在文献中有无对标;③多面板×bootstrap×SuSiE 的可行性是否有证据支撑
- `## 缺口`
- 只引用实际见过的来源,不编造 URL;运行时间等数字若无来源则标"经验值/未验证"。