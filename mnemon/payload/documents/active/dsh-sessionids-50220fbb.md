---
id: "50220fbb-856d-49a7-a9ed-fff16159d00f"
title: "DSH 同步后会话落入「未分组/未命名」：sessionIds 名单与标题判定机制及批量补齐"
description: "说明同步 sessions 目录后会话为何落入「未分组/未命名」：sessionIds 是唯一归属依据、标题只在日志内且不拿目录名兜底；给出按裸键去重的批量补齐脚本口径与 ZIP 导入兜底。"
status: "active"
created_at: "2026-11-10T05:25:31.789Z"
updated_at: "2026-11-10T05:25:31.789Z"
content_hash: "952929e7c6cbdd891e839b665e246d7cfe92c0ddee89a5c5c31228227f88cc22"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-e97d530b-626f-4463-9d2e-3e0a63a09c24"
memory_body_ids:
  []
---

# DSH 同步后会话落入「未分组 / 未命名」的成因与修复

> 接续既有文档 `d08f20c4-349e-4f30-a9c6-69b48441be95`（跨设备迁移：首帧 cwd、项目键、workspace.json、ZIP 导入与路径映射）。本文只补该文未覆盖的两点：**工作区归属只认 `sessionIds` 名单**（无任何按路径自动认定），以及**「未命名」来自日志内标题缺失、不用目录名兜底**；并给出按裸键去重的批量补齐做法。

## 1. 归属：`sessionIds` 是唯一依据，DSH 不会按目录名/cwd 自动认领

工作区记录（`%USERPROFILE%\.dsh\storages\workspace.json`）形如：

```json
"8ef481ce-…": { "path": "D:\\Workspace\\DSHarness", "title": "DSHarness",
                "sessionIds": ["session-617b0be5-…", "session-1aa3903e-…"] }
```

- 侧栏「某会话属于某工作区」的唯一判据就是该记录的 `sessionIds` 是否含其 id；不含 → 落入「未分组」（asar 文案 `session.ungrouped` 一类）。
- 只有「新建会话」路径会按 cwd 反查工作区并登记（`registry.resolveByPath(cwd)` 后 `attachSession(sessionId)`，app.asar ≈2457955 / 2466064）。
- **启动、加载、扫描磁盘均无「按目录名或 cwd 自动认定归属」的代码**，所以只同步 `sessions\` 目录 = 只带日志、不带归属；`workspace.json` 才是归属载体。
- 反向排查法：`sessions\<项目键>\` 目录数 vs `sessionIds` 条数。本机实测 55 目录 / 52 id，差集 3 个（2 个 `mnemon-run-v1-…` + `session-2bb04a98-…`）正是「未分组」里的项。`global.archivedSessionIds` 中的会话即使已登记也不显示（在归档区）。

## 2. 命名：标题只存在于会话日志内，「未命名」不拿目录名兜底

- 每个会话目录只有一个 `session.v4.jsonl.zstd`，无旁路元数据文件；标题存在该日志的事件里。
- 无已保存标题时 DSH 显示「未命名 / Untitled」（asar ≈871259 `"session.untitled": "未命名"`；:866267 文档明写不使用目录名兜底）。
- 两种成因：① 源机该会话本来就未保存标题（自动起标题未跑成/会话过短）；② 网盘同步到**正在写入**的 zstd，尾部截断导致标题事件丢失。
- 判别：看源机同一会话是否有标题；源机有、目标机无 ⇒ 日志截断，退出 DSH 后重新完整同步该目录。

## 3. id 命名有两种形态，合并名单必须按裸键去重

本机 55 个会话目录实测：15 个裸 uuid、17 个 `session-` 前缀、23 个 `mnemon-run-v1-…` 等其它前缀；注册表里统一写 `session-<uuid>`（mnemon 类保持原样）。合并时须用 `id -replace '^session-',''` 的裸键判重，否则会写入永远匹配不上的 id（`src/adapters/workspaces.ts` 的 `mergeWorkspaceRecordsById` / `sessionIdKey` 即此口径：`sessionIds` 取并集、按裸键去重）。

## 4. 方案 A：离线批量补 `sessionIds`（两机工作区路径一致时最快）

前提：**DSH 完全退出（含托盘）**，否则退出时内存旧名单会覆盖修改；先备份 `workspace.json`。脚本要点（幂等，可反复跑）：

1. 读 `workspace.json`，按 `path == 目标工作区路径` 找到记录；
2. 用 `HashSet` 装已有 id 的裸键；
3. 遍历 `sessions\--D-Workspace-DSHarness--\*` 目录，裸键不在集合中则追加（裸 uuid 目录补成 `session-<uuid>`，其它前缀原样）；
4. `ConvertTo-Json -Depth 20` + `UTF8Encoding($false)` 写回（避免 BOM/深度截断）。

不改会话日志、不改首帧 cwd —— 两机路径相同时项目键 `--D-Workspace-DSHarness--` 本就一致，缺的纯是名单。若某条仍不显示：查 `archivedSessionIds`，或用「配置管理器 → 环境 → 维护与诊断 → 会话体检」看是否被判「目录与首帧 cwd 不一致」（那类才需「会话布局归位」）。

## 5. 方案 B：GUI ZIP 导入（路径/命名同时对齐）

源机导出勾选「工作区 + 历史会话」→ 目标机导入并在预览阶段做路径映射。可行原因：工作区分区在所有分区收尾后执行 `finalizeImport`，对 `sessionIds` 逐个 `attachSession`，且登记目标 = 备份声明的 id ∪ 本包真正带数据的会话（`workspaces.ts:246-265`），因此**即使会话已在本机、同 id 被跳过不覆盖，归属仍会写入**；同 id 冲突只跳过并报告，不破坏本机日志。

## 6. 长期做法

- 不要只同步 `sessions\`；至少要同时处理 `%USERPROFILE%\.dsh\storages\workspace.json`（`archivedSessionIds` / `pinnedSessionIds` 同在）。
- 两机工作区绝对路径只要有一处不同（盘符/用户名/目录名），`path` 就对不上，只能走方案 B 的路径映射。
- 同步前后都退出 DSH，避免半写 zstd（既是「未命名」常见来源，也可能导致会话读不出）。

## 关键落点

- `workspace.json`：`<DSH_HOME>\storages\workspace.json`（旧版可能是 `<DSH_HOME>\workspace.json`）
- `projectKeyOf`：`dsh-config-manager/src/core/session-select.ts:46`
- 名单合并/导入登记：`dsh-config-manager/src/adapters/workspaces.ts`（`mergeWorkspaceRecordsById`、`finalizeImport`、`registerImportedSessions`）
- 相关文档：`d08f20c4-349e-4f30-a9c6-69b48441be95`
