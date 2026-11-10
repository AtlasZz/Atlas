---
id: "d1423f79-4ef2-443a-8564-97840eb28e55"
title: "dsh-config-manager 同步 workspace.json 的增量语义：记录级三方合并、sessionIds 并集与「不会整文件替换」的边界"
description: "回答「用配置管理器同步 workspace.json 是增量还是替换」：writeRecord 无覆盖写通道、ZIP 导入逐条 upsert、同步通道按记录 id 三方合并且 sessionIds 恒取并集；并列出快照恢复、显式选「使用远端」、改 path 不生效等真实风险点。"
status: "active"
created_at: "2026-11-10T05:40:48.428Z"
updated_at: "2026-11-10T05:40:48.428Z"
content_hash: "79445e4c83e851a6bb45bb5ddea849049ffb877ebd53de987baf34b03711b8bb"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-267c216d-538f-4d0f-bbac-7c2dded5d3a6"
memory_body_ids:
  []
---

# dsh-config-manager 同步 workspace.json：增量语义与边界

> 场景：两台机器共用 DSH，需要把「哪个会话属于哪个工作区」搬到另一台。用户问：除了同步两个日志目录，是否还要更新 `workspace.json`；配置管理器是增量更新还是整文件替换（担心本机已有信息被删）。结论：**配置管理器的两条通道都是增量语义，整文件覆盖只发生在「网盘直接同步文件」或「快照恢复/回滚」**。证据为 `<DSH_HOME>\profiles\desktop\node_modules\dsh-config-manager\src` 下源码（本机版本），升级后行号/函数名宜复查。

## 1. 前提：归属只存在 workspace.json 里

`sessions\` 目录只装日志，「谁属于哪个工作区」仅由 `<DSH_HOME>\storages\workspace.json` 中各记录的 `sessionIds` 决定。因此跨机迁移要么同步该文件，要么在目标机补登记 id，要么让插件导入时补 `attachSession`。详见 `50220fbb-856d-49a7-a9ed-fff16159d00f` 与 `d08f20c4-349e-4f30-a9c6-69b48441be95`。

## 2. 为什么不存在「整份替换工作区表」

DSH 宿主侧没有「覆盖写 workspace.json」的 API。插件的写入走 `writeRecord`（`src/index.ts:629-639`）：

- 已存在同 id 记录：**只允许改标题**（`existing.setTitle`），`path` 与 `sessionIds` 不会被覆盖，函数直接 return；
- 不存在：`registry.create(record.path, record.title)` 新建。

所以「整份替换/删除」只可能来自快照恢复或回滚这类显式破坏性操作（会先要求确认），不属于同步路径。

## 3. 通道 A：导出 / 导入 ZIP —— 逐条 upsert，纯增量

| 情况 | 行为 |
|---|---|
| 包内工作区，本机无同 id | 新建（安全路径下连目录一起建），`sessionIds` 先空，收尾阶段补登记 |
| 本机有同 id、内容一致 | 跳过 |
| 本机有同 id、内容不同 | 列为冲突逐条裁决；受 `writeRecord` 限制，实际只会改标题 |
| 本机独有（包内没有） | 完全不动，不会被删 |
| 会话归属 | 所有分区收尾后，对备份声明的 `sessionIds` ∪ 包内实际带的会话逐个 `attachSession`（`src/adapters/workspaces.ts:246-265`），**只增不减** |

结论：ZIP 导入不会删本机工作区，也不会清空已有 `sessionIds`。

## 4. 通道 B：git / WebDAV / S3 / Gist 同步 —— 记录级三方合并

`workspaces` 分区有专门合并器 `src/sync/workspace-merge.ts`（issue #127），因为通用 `mergeJsonValues` 对数组一律「双侧都改且不逐字相等 → 冲突」，而 `sessionIds` 天生两侧各自增长，会导致每轮都冲突并静默丢掉一侧归属。

规则（`workspace-merge.ts` 文件头与 `mergeRecord`）：

- **`sessionIds` 恒取两侧并集**：本地在前、远端新增按原序追加，按小写裸 id 去重（`unionSessionIds`）。理由：挂接只是提示，多余 id 最多 attach 不到会话；少了 id 会让本机已有会话从工作区消失。
- 其它字段（`path`/`title`/`createdAt`/`updatedAt`/未来字段）：两侧相同取该值；仅一侧相对祖先变了取变更侧；两侧都变或无祖先 → **冲突**（点号路径 `workspaces.<id>.<field>`，`reason` 为 `both-modified` 或 `no-ancestor`），保留本地值供对照，绝不静默取舍。
- 仅单侧存在的记录：有祖先且该侧与祖先逐字相同 → 跟随删除；无祖先 → 保留（无法区分「对方删了」与「本机独有」）；有祖先且该侧改过 → `modify-delete` 冲突。
- 整分区若产生冲突则整体不半应用，交用户裁决（`previewMerge` / `applyMergeResolutions`，`src/sync/sync-engine.ts:497-501`）。
- `sessions` 分区默认不参与同步（设备相关 + 手动 `syncOptIn`），所以同步只搬名单不搬日志。

分区级决策链（`src/sync/merge.ts`）：`skip` → 双侧相同 → `useRemote`/`keepLocal` → `merge`（JSON 递归 / 行级三方 / 记录级钩子）→ `conflict`。`merge` 在 `risk.ts` 中与 useRemote/keepLocal 同属「决策明确」，可在低风险阈值内自动应用。

## 5. 真正需要提防的三件事

1. **快照恢复 / 回滚**：整文件恢复，会覆盖并删除，属破坏性操作。
2. **冲突时显式选「使用远端」**：整分区取远端值，可能丢掉本机独有 id；选默认合并结果（useMerged）才是并集。
3. **改 `path` 对已存在工作区不生效**（`writeRecord` 只改标题）。两机绝对路径不同时，要让新路径生效只能「从工作区列表移除再重新添加」——移除工作区不删会话文件，这些会话回到「未分组」，再挂回去即可。

另：`workspaces` 等分区默认不参与自动同步，需在同步页分区选择里显式勾「工作区」后再手动推送/拉取；网盘支持 WebDAV（坚果云、Nextcloud）时优先用 WebDAV 通道，而不是让网盘直接同步 `workspace.json` 文件（文件级「最后写入者胜」才是唯一会静默丢名单的路径）。WebDAV 限流问题见 `dc99f112-748d-4732-8416-e944e7365f01`。

## 关键落点

- 覆盖写缺席：`dsh-config-manager/src/index.ts:629-639`（`writeRecord`）
- 记录级三方合并：`src/sync/workspace-merge.ts`（`mergeWorkspacesSection`、`mergeRecord`、`unionSessionIds`）
- 分区决策链：`src/sync/merge.ts`；文本/JSON 合并：`src/sync/text-merge.ts`
- 裁决与应用：`src/sync/sync-engine.ts:497-501`、`1590-1651`
- ZIP 导入登记：`src/adapters/workspaces.ts:246-265`
