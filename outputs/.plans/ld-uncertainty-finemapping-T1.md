# Researcher Brief T1: 精细定位方法与 LD 不确定性现状

你是统计遗传学/精细定位专项调研员。目标是评估一个方法的**创新性**:该方法用"多个参考面板 LD + bootstrap"合成一组 LD 矩阵,逐矩阵跑 SuSiE,再聚合 PIP。你需要查清**文献中已经存在什么**。

## 背景(给评审对象,不必复述在输出里)

被评审方法:输入 GWAS 汇总统计(beta, se)+ 多个 LD 参考面板 → 建模 LD 不确定性(多面板 + bootstrap)→ 逐 LD 矩阵跑 SuSiE 得每个 SNP 的 PIP → 聚合。

## 你要回答的问题

1. **主流精细定位方法对 LD 输入的要求与假设**:SuSiE(Wang et al. 2020, AAS/ASIP)、FINEMAP(Benner et al.)、CAVIAR(Hormozdiari)、DAP、SuSiEx。它们的 LD 矩阵来自哪里(参考面板、样本内)?是否假设 LD 已知/无误?
2. **LD 参考面板选择/不匹配已知会怎样影响结果**:搜索 LD panel mismatch、LD misspecification、population-specific LD 对 fine-mapping 的影响;不同族群面板(EUR vs AFR vs EAS)的 LD 差异研究;LD 估计误差(样本内小样本)的影响。
3. **是否已有"LD 不确定性建模"的先例**:
   - 多个 LD 面板联合使用 / 元分析 LD
   - bootstrap LD 矩阵 / LD 重采样
   - 跨族群/多族群 fine-mapping(SuSiEx、其他 multi-ethnic 方法)是否天然就是"多面板"
   - 对 LD 做扰动/引入噪声来测试稳健性的文献
4. **SuSiE 输出在模型错误设定下的行为**:PIP 校准、credible set 覆盖在 LD 错误/单因果假设违背时是否失效;已有任何关于 SuSiE 稳健性的讨论。

## 搜索建议(可自由扩展)

- "fine-mapping LD reference panel mismatch"
- "linkage disequilibrium misspecification fine-mapping"
- "SuSiE credible set coverage"
- "multi-ethnic fine-mapping SuSiEx"
- "bootstrap LD matrix / LD uncertainty genetic"
- "reference panel population-specific LD difference EUR AFR"
- 在 web_search 用 3-6 个不同角度查询,善用 queries 多查询参数
- 也可以搜索关键论文标题:SuSiE / FINEMAP / CAVIAR 原始论文页面(PMC、arXiv、biorxiv)

## 输出要求

写到你被指定的研究笔记文件(见任务指令),Markdown 结构:
- `## 现状摘要`(3-6 句话)
- `## 发现列表`:每条含【声明 + 来源 URL + 置信度(高/中/低)】
- `## 对评审对象的启示`:LD 不确定性建模是否已存在(具体指向哪篇文献),如果存在,被评审组合的新颖点还剩什么
- `## 缺口`(没找到证据的地方)
- **只引用你实际访问/搜索结果中出现的来源,不要编造 URL**。拿不准的标记为"未验证"。