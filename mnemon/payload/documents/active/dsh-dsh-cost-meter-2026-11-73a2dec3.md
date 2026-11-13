---
id: "73a2dec3-6137-411c-b9d2-1f7ec2481436"
title: "DSH 会话费用归因与降本分析（dsh-cost-meter 账本实测，2026-11）"
description: "基于本机 dsh-cost-meter 账本的 DSH 费用结构解析：按 token 计价而非按工具、缓存未命中事故、pwsh 成本归因机制、降本优化清单，以及 dsh-ops 插件的适配性与成本边界。"
status: "active"
created_at: "2026-11-13T03:58:43.343Z"
updated_at: "2026-11-13T03:58:43.343Z"
content_hash: "c9ad384e119a91ccba939d52262bf9b90af7d32d3ef1599f494037ec0851aa88"
source_paths:
  - "."
session_ids:
  - "mnemon-run-v1-12dced39-cb33-44d3-91fe-618ff6a81520"
memory_body_ids:
  []
---

# DSH 会话费用归因与降本分析（本机实测，2026-11-13）

面向"为什么某个工具/某步这么贵"与"怎么降本"的结论性记录。数字是 2026-11-13 当天的账本快照，会过时；**机制与判断逻辑可复用**。

## 一、计价模型（关键事实）

费用按 **token** 收，不按工具或调用次数收。从账本逐条记录反推出的单价：

| 桶 | 单价（$/M token） |
|---|---|
| 未缓存输入 input | 0.30 |
| 缓存重读 cacheRead | 0.006（比 input 便宜 50 倍） |
| 输出 output | 1.20 |
| 非高峰时段 | 各减半（0.15 / 0.60） |

超长上下文会触发 longContext 单价档。因此"某工具贵"只是现象：工具决定的是**它往上下文里塞了多少 token，以及后续每一步重发多少**。

## 二、账本数据结构（dsh-cost-meter）

本机 `dsh-cost-meter` 缓存目录下两类文件：

- 顶层：按日/按维度聚合的 JSON（8–85 KB），每个是 `{version, key, generatedAt, source, value}`。
- `details-cache/`：同名同构但明细更全（最大 350 KB 级）。

`value` 的字段：`cost, apiCost, shares, totalSteps, totalCalls, steps, groups, operations, operationGroups, hierarchy, distribution, tiles`。

`steps[]` 每条含 `sessionId, turn, step, kind(model/tool), tools[], atMs, cost, apiCost, calls, rows[], callRefs[]`；`rows[]` 才是原子计价单元：`{provider, model, bucket: input|cacheRead|output, tokens, rate, cost, priced, plan}`。`callRefs[]` 带 `metering.requestId` 用于对账。

由此可做两类归因：
- **单工具成本**：只含该工具的 step 成本之和。
- **任一工具参与成本**：含该工具的 step 成本之和（多工具有重叠，不能相加）。

## 三、当次实测数字（2026-11）

- 当日：95 次调用，input 467K / cacheRead 17.9M / output 62K，$0.322。
- 一次会话（session-7e16…）：155 steps，$0.481，213 条 step 记录去重后。
- 单工具归因（当日）：`pwsh` 39.9%，`compress` 1.8%，`read` 1.4%，`write` 0.6%，`grep` 0.2%。
- 最贵单步：turn 9 / step 1 = **$0.0942**，input **310,601**、cacheRead 仅 **5,504**、output 853。

## 四、核心结论

1. **成本大头是"每步全量重发"**：每次模型调用都会重发系统提示（本机首轮固定头部约 27K–37K token）+ 全部历史 + 所有工具结果。工具碎、步数多 → 重复发送多。
2. **缓存重读单价极低但量巨大**：17.9M cacheRead ≈ $0.107，占当日 1/3。这段成本由上下文总长决定，缩短单条输出省不掉。
3. **最危险的是缓存未命中 + 长上下文**：上述 $0.0942 那一步 31 万 token 只有 5.5K 命中缓存，等于整份按全价重算。诱因：中途切模型、改系统提示、增删插件导致前缀缓存失效。比"输出太长"严重得多。
4. **pwsh 贵是机制性的**：默认输出冗长（`Get-ChildItem -Recurse`、`Format-Table` 的列宽填充、中文多 token），且这些内容会一直留在上下文里参与后续每一步。

## 五、降本清单（按性价比）

1. **减少工具数量**：每个常驻工具 schema 都在每次请求首部。本机装了 20 个 bundle（含 30+ `ego_*`、20 个 `univer_*`、office 系），卸载用不到的 bundle 是永久性降本。
2. **不打碎前缀缓存**：避免中途换模型 / 改系统提示 / 增删插件。
3. **收紧 pwsh 输出**：优先 `read`/`grep`/`glob` 取代 `Get-Content`/`Select-String`/`Get-ChildItem`；避免 `Format-Table`；长输出重定向到文件后按行区间读；合并命令减少 step；`-Recurse` 前加 `| Select-Object -First N`。
4. **大任务交子代理**：子代理探索过程不进主上下文，只回传结论，是少改配置就能压低"上下文膨胀率"的主要手段。
5. **让压缩真正触发**：`compaction-basic` 默认 `thresholdRatio: 0.8`，在 1M 窗口下要 72–84 万 token 才压缩，等于没开；建议降到 0.5–0.6（`cordis.patch.yml`）。
6. **错峰运行**：非高峰单价减半，长任务收益直接 50%。

## 六、dsh-ops 插件的适配性与成本边界

- **版本约束**：`dsh-ops@0.5.4` 的 `engines.dsh` 为 `>=0.2.2-rc.1 <0.2.3-0`；本机 dsh 为 **0.2.2-rc.1**（`@deepseek-ai/dsh-desktop-runtime` 0.2.2-rc.1），刚好可装；0.4.x 及以下不兼容。
- **痛点对得上**：原生 `file`/`shell` 工具替代 PowerShell 冗长输出；`hide-official` 隐藏官方 `read/write/grep/pwsh` 以缩小常驻 schema；`mode-ptc` 把多步工具调用合并为一段代码执行，**直接减少 step 数**（即减掉最大那块"每步重发"）。
- **代价**：npm 包 unpacked 约 **140 MB / 1,105 文件**（含三平台原生二进制），个人维护；它自带工具 schema + 注入提示，**若不启用 `hide-official` 反而更贵**（固定头部叠加变大）。
- **不解决根因**：它不改变"每步全量重发"的结构，只减少步数与单条输出体积。
- **判断方式**：先做免费优化 1–6，再对同类任务做 A/B，比较「总 step 数 / 总 token / 每任务 $」，看三天账本，而不是指望装上就降价。

## 七、可复用的取证方法

- 直接读 `dsh-cost-meter` 的 details-cache JSON，按 `steps[].rows[]` 的 `tokens × rate` 复算，可独立验证账本自洽（本次复算 input 310601×0.3/M = $0.09318，与账本 $0.0942 一致）。
- 排查"某步异常贵"时先看该步 `input` 与 `cacheRead` 的比例，再看 `callRefs[].longContext`。
- 探针脚本用完即删（本次临时脚本 `_ledger_probe.py` 已清理），不要把探查产物留在工作区。
