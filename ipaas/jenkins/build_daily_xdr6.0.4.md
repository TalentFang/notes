# build_daily_xdr6.0.4 流水线分析

## 基本信息

| 项目 | 值 |
|------|-----|
| 类型 | MultiJob 多任务流水线 |
| 插件 | jenkins-multijob-plugin |
| 定时触发 | `H 1 * * *`（每天凌晨 1 点） |
| 执行方式 | 顺序执行（SEQUENTIAL） |
| 失败策略 | 继续执行（ALWAYS） |
| 产品版本参数 | `PRODUCT_VERSION=v6.0.5` |
| 邮件通知 | 构建失败时发送 |

## 流水线结构

### 预处理

```bash
rm -rf /var/jenkins_home/repo/com/ais/maxs
rm -rf /var/jenkins_home/repo/com/ais/framework
echo SOURCE_BUILD_NUMBER=${BUILD_NUMBER} > ${WORKSPACE}/Buildinfo
echo PRODUCT_VERSION=${PRODUCT_VERSION} >> ${WORKSPACE}/Buildinfo
```

### 阶段 1：Java 后端模块

| 序号 | 任务名 | 说明 |
|------|--------|------|
| 1 | maxs-dependencies | 基础依赖包 |
| 2 | maxs-pm-api-linkone | PM API |
| 3 | maxs-soar-api-linkone | SOAR API |
| 4 | maxs-asset-api-linkone | 资产 API |
| 5 | maxs-intel-api-linkone | 情报 API |
| 6 | maxs-commons-xdr | 公共模块 |
| 7 | maxs-monitor | 监控模块 |
| 8 | python-offline | Python 离线包 |
| 9 | lkone-maxs-pm | PM 模块 |
| 10 | lkone-maxs-intel | 情报模块 |
| 11 | lkone-maxs-rv | RV 模块 |
| 12 | maxs-rule-engine_lo | 规则引擎 |
| 13 | maxs-rule-lo6.x | 规则 LO 6.x |
| 14 | data-process-engine-xdr | 数据处理引擎 |

### 阶段 2：Web 前端模块

| 序号 | 任务名 | Node.js | 需要 node_modules |
|------|--------|---------|-------------------|
| 15 | lkone-frontend | v12.22.12 | ★ 是 |
| 16 | lkone-ssa-web | v12.22.12 | ★ 是 |
| 17 | lkone-vis-web | v12.22.12 | ★ 是 |
| 18 | lkone-maxs-web | - | 否 |
| 19 | lkone-maxs-soar | - | 否 |
| 20 | lkone-maxs-bi-web | v12.22.12 | ★ 是 |

### 阶段 3：其他模块

| 序号 | 任务名 | 说明 |
|------|--------|------|
| 21 | maxs-ts-alone | TS 独立模块 |
| 22 | maxs-upgrade | 升级模块 |
| 23 | maxs-asset6.0.4 | 资产 6.0.4 |
| 24 | lkone-maxs-pboc-report | 央行报表 |
| 25 | lkone-maxs-bi | BI 模块 |
| 26 | lkone-maxs-bi-engine | BI 引擎 |
| 27 | lkone-maxs-dm | DM 模块 |
| 28 | lkone-maxs-datareport | 数据报表 |
| 29 | data-access-xdr | 数据接入 |
| 30 | maxs-ops | 运维模块 |
| 31 | linkone-maxs-szr | SZR 模块 |
| 32 | ipaas-precheck | 预检查 |

### 完成后

- 触发 `build_trigger_job`
- 构建失败时发送邮件通知

## 定时触发配置

### 当前配置

```
H 1 * * *
```

### Jenkins Cron 语法

```
┌───── 分钟 (0-59)
│ ┌───── 小时 (0-23)
│ │ ┌───── 日 (1-31)
│ │ │ ┌───── 月 (1-12)
│ │ │ │ ┌───── 星期 (0-7, 0和7都是周日)
│ │ │ │ │
H 1 * * *
```

### H 符号说明

`H` 是 Jenkins 的哈希符号，不是固定值：

- `H 1 * * *` → 在 1:00-1:59 之间随机选一个时间
- `0 1 * * *` → 固定在 1:00 整

使用 `H` 可以避免所有定时任务同时触发，分散服务器负载。

### 常用定时表达式

| 表达式 | 含义 |
|--------|------|
| `H 1 * * *` | 每天凌晨 1 点（当前配置） |
| `H 2 * * *` | 每天凌晨 2 点 |
| `H 0 * * 1-5` | 工作日凌晨 |
| `H/30 * * * *` | 每 30 分钟 |
| `0 8 * * 1` | 每周一早 8 点 |
| `H 0 * * 1` | 每周一凌晨 |

### 修改定时

Jenkins Web UI 操作步骤：

1. 点击 `build_daily_xdr6.0.4` → 配置
2. 找到 "Build Triggers" → "Build periodically"
3. 修改 Schedule 字段
4. 保存

## 关键配置说明

### continuationCondition: ALWAYS

即使某个子任务失败，流水线也会继续执行后续任务。这意味着：

- 前端构建失败不会阻止后端模块的构建
- 所有 32 个任务都会尝试执行
- 最终结果取决于所有任务的汇总状态

### 执行方式: SEQUENTIALLY

所有子任务按顺序依次执行，不会并行。这保证了：

- 依赖关系正确（如 maxs-dependencies 必须先构建）
- 资源不会冲突
- 构建时间较长（所有任务串行）

### 邮件通知

构建失败时发送邮件给以下人员：

- zhang.hz@asiainfo-sec.com
- lu.yj@asiainfo-sec.com
- hong.liang@asiainfo-sec.com
- cao.zl@asiainfo-sec.com
- lin.yiming@asiainfo-sec.com
- zhang.ym3@asiainfo-sec.com
- lv.pei@asiainfo-sec.com
- li.ss@asiainfo-sec.com
- yue.yy@asiainfo-sec.com
- shenjie3@asiainfo-sec.com
- fanwj@asiainfo-sec.com
- lv.yf@asiainfo-sec.com
- liuzuo@asiainfo-sec.com
- zhangxl@asiainfo-sec.com

## build_trigger_job（构建后触发）

`build_daily_xdr6.0.4` 构建完成后自动触发此任务。

### 基本信息

| 项目 | 值 |
|------|-----|
| 类型 | MultiJob 多任务流水线 |
| 触发方式 | 由 build_daily_xdr6.0.4 触发 |
| 执行方式 | 并行执行（PARALLEL） |
| 失败策略 | 任一失败则停止（SUCCESSFUL） |

### 子任务

```
build_trigger_job
├── build_trigger_job2 ──→ build_uosv25_arm (UOS ARM 构建)
├── build_uosv25_x86 (UOS x86 构建)
├── upgrade_shell_6.0.4 (升级脚本)
└── build_noniata_uosv25_x86 (非 SATA UOS x86 构建)
```

| 序号 | 任务名 | 说明 | 失败策略 |
|------|--------|------|----------|
| 1 | build_trigger_job2 | 触发 ARM 构建 | 失败停止 |
| 2 | build_uosv25_x86 | UOS v25 x86 构建 | 继续执行 |
| 3 | upgrade_shell_6.0.4 | 升级脚本 6.0.4 | 继续执行 |
| 4 | build_noniata_uosv25_x86 | 非 SATA UOS x86 构建 | 失败停止 |

### build_trigger_job2

进一步触发 ARM 构建：

| 序号 | 任务名 | 说明 |
|------|--------|------|
| 1 | build_uosv25_arm | UOS v25 ARM 构建 |

## 完整触发链路

```
build_daily_xdr6.0.4 (每天凌晨 1 点)
    │
    ├── [顺序执行] 32 个子任务（Java 后端 + Web 前端 + 其他）
    │
    └── [完成后] 触发 build_trigger_job
                    │
                    └── [并行执行] 4 个构建任务
                        ├── build_trigger_job2 → build_uosv25_arm
                        ├── build_uosv25_x86
                        ├── upgrade_shell_6.0.4
                        └── build_noniata_uosv25_x86
```

## 配置文件位置

| 任务 | 宿主机路径 |
|------|-----------|
| build_daily_xdr6.0.4 | /home/opt/jenkens/jobs/build_daily_xdr6.0.4/config.xml |
| build_trigger_job | /home/opt/jenkens/jobs/build_trigger_job/config.xml |
| build_trigger_job2 | /home/opt/jenkens/jobs/build_trigger_job2/config.xml |
