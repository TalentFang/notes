# Verification Report — ld-uncertainty-finemapping

**日期**:2026-09-13
**验证对象**:outputs/.drafts/ld-uncertainty-finemapping-revised.md(最终候选,源自 cited)

## 执行的检查

1. **Evidence-auditor 审计**(子代理,无 web 工具;做内部一致性 + 标识级核对):输出 PDF>20MB 不可抓取等限制已记录;发现并解决 Model-X 作者归属冲突(DOI 级验证:doi:10.1111/rssb.12265 = Candès-Fan-Janson-Lv, JRSS-B 80(5):1271–1301,2018);修正 arxiv.gg 域名;修正运行次数算术(上限 5×10⁵);标记 ESNN 孤儿引用。
2. **决定性引文逐字核验**(lead 侧 web 工具):
   - [S4] Wu et al.(PLOS Comp Biol):"over-conservative in most fine-mapping situations" + 选择偏向解释 —— 已抓取正文核验 ✓
   - [S3] FunGen-xQTL:"unstable credible sets" 引文 —— 已抓取正文核验 ✓
   - [S6a/S6b] selection-conditional coverage:NSF PAR 与 arXiv:2403.03868 摘要双重确认 ✓
   - [S5]/[S20] Model-X 作者:DOI + USC PDF 原文确认 ✓
   - [S17] stability selection:元数据确认(doi:10.1111/j.1467-9868.2010.00740.x, JRSS-B 72(4):417–473)✓
   - [S18] SuSiE 官方引用:Wang-Sarkar-Carbonetto-Stephens 2020, JRSS-B 82(5):1273–1300, doi:10.1111/rssb.12388 ✓
3. **Reviewer 验证**(子代理,完整阅读 revised/cited 与全部研究笔记):FATAL 0;MAJOR 1(S6 单来源未核);MINOR 4(§1.1 措辞矛盾、3 处功能声称超证据等级、stability selection 对照过强、S12 警告缺失于摘要)。
4. **修复与 on-disk 复核**:
   - MAJOR:S6 拆为 [S6a]+[S6b] 双来源(摘要级验证)+ 补 [S30](jackknife+,Ann. Statist. 背景佐证)✓
   - MINOR-2:§1.1 结论改写为与 §1.2/1.3 一致("单面板参数化校正已有,可重采样传播空白")✓
   - MINOR-3:RFR/S21/S24/S17 功能描述降级为"元数据级/正文未核"限定 ✓
   - MINOR-4:"几乎一一对应/几乎同构"→"高度相关但不同构"(stability selection 频率向量 vs 单点集中度的差异已写明)✓
   - MINOR-5:摘要 #4 与 C1 表内 [S12] 加"存在性未核实"注记 ✓
   - 复核命令:rg 确认旧措辞(几乎一一对应/形式几乎同构/arxiv.gg/2.5万–100万)消失;python 确认 31 个 [S#] 定义与引用完全一致,无缺失无孤儿 ✓
5. **结构修复**:S30 初始插入位置破坏"未验证段落"引导行,已修复并复核(独立成段 + 未验证列表恢复)✓

## Findings

**FATAL**: 0
**MAJOR**: 1(已修复并复核)
**MINOR**: 4(全部修复并复核)

## 尚未验证(BLOCKED 项,不阻塞交付但应知悉)

- [S12] arXiv:2606.25729 存在性:域名已修正为 arxiv.org,但该 ID(2026-06 编号)未能打开核实——引用前须再验。
- [S13] CP4SBI 的 URL 为审计阶段补入,未独立抓取核验。
- 大量"已检索"来源基于搜索快照/抓取存在性,正文未逐页核读;标识级条目(SuSiE 2.0、MultiSuSiE、RSparsePro、SuSiNE)只有标识。
- 领域知识条目(FINEMAP、CAVIAR、HMM knockoffs、Manski、E-value、GTEx 基准、1000G 规模)无 URL,方法方引用前须自行核实。

## 结论

**Verification: PASS WITH NOTES**(无 FATAL;MAJOR/MINOR 全部修复并经 on-disk 复核;未验证项已全部如实标注,不影响核心结论方向;核心结论由多条独立证据支撑)。