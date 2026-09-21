# Provenance: 面向连锁不平衡不确定性的分布自由因果变异集合构建与精细定位脆弱性诊断

- **Date:** 2026-09-13
- **Rounds:** 3 rounds + 1 audit round + 1 review round
  - R1: 3 researcher 子代理并行(T1 精细定位/LD 现状、T2 分布自由推断、T3 校准/基准/可行性)
  - R2: lead 直接 web_search 补充(3 轮共 9 查询)+ source_check + fetch_content 抓取核验
  - R3: 关键引文逐字核验(Wu、FunGen-xQTL、selection-conditional coverage)
  - 审计:evidence-auditor 内部一致性审计(无 web 工具,标识级核对)
  - 评审:reviewer 完整验证(cited 文件)
- **Sources consulted:** ~30 个来源(搜索/快照/抓取),见最终报告 Sources [S1]–[S30]
- **Sources accepted:** [S1]–[S30](含 2 个摘要级验证、多个元数据级);未验证领域知识条目单列
- **Sources rejected:** arxiv.gg 域名版 [S12](修正为 arxiv.org);lead 记录中 "Barber & Candès" 对 Model-X 的错误归属(修正为 CFJL 2018)
- **Verification:** PASS WITH NOTES(0 FATAL;1 MAJOR + 4 MINOR 全部修复并经 on-disk 复核;未验证项已如实标注)
- **Plan:** outputs/.plans/ld-uncertainty-finemapping.md
- **Research files:**
  - outputs/.drafts/ld-uncertainty-finemapping-research-lead.md(lead 直接检索笔记,含 3 轮搜索记录与作者归属验证)
  - outputs/.drafts/ld-uncertainty-finemapping-research-t1.md(精细定位与 LD 不确定性)
  - outputs/.drafts/ld-uncertainty-finemapping-research-t2.md(分布自由推断)
  - outputs/.drafts/ld-uncertainty-finemapping-research-t3.md(校准/脆弱性/基准/可行性)
  - outputs/.drafts/ld-uncertainty-finemapping-draft.md(初始草稿)
  - outputs/.drafts/ld-uncertainty-finemapping-cited.md(引用版)
  - outputs/.drafts/ld-uncertainty-finemapping-revised.md(最终候选,已交付)
  - outputs/.drafts/ld-uncertainty-finemapping-verification.md(验证报告)

## 运行缺陷与降级记录

1. **子代理无 web 工具**:3 个 researcher 与 evidence-auditor 运行时仅注册 read/write/contact_supervisor,无 web_search;降级为"读 lead 检索笔记 + 领域知识,来源分级标注(已检索/标识级/未验证)";所有权威来源由 lead 侧 web 工具负责最终核验。
2. **alpha_search 不可用**:需 `alpha login`,已跳过;论文检索由 web_search 替代。
3. **标准 verifier agent 不存在**:以 evidence-auditor + lead 侧逐字核验替代引用验证;reviewer 正常执行。
4. **PDF 抓取限制**:par.nsf.gov(S6)PDF >20MB,改为 arXiv 摘要级验证 + 双来源。
5. **首次 workflow 脚本 bug**:`const runs` 遮蔽全局 runs 导致 ReferenceError;已修复重启。
6. **审计修复 S30 插入位置错误**:破坏"未验证"段引导行;已修复并复核。

## 未验证项(BLOCKED 清单)

- [S12] arXiv:2606.25729 存在性未核实(域名已修正,ID 待验)
- [S13] CP4SBI URL 未独立抓取
- 标识级条目正文(SuSiE 2.0、MultiSuSiE、RSparsePro、SuSiNE)
- 领域知识条目(FINEMAP、CAVIAR、HMM knockoffs、Manski、E-value、GTEx 基准、1000G 规模细节)