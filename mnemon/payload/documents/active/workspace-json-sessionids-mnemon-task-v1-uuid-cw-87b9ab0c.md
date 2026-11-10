---
id: "87b9ab0c-a8fc-49da-ab40-ae35a113d8ad"
title: "workspace.json 的 sessionIds 为何混入 mnemon-task-v1 与裸 uuid：成员登记只校验 cwd，不校验 id 形态"
description: "解释「同步后新建会话报 agent-busy」排查中发现的异常 sessionId 来源：attachSession 仅比对 header.cwd、四条写入通道、dsh-mnemon 任务 Agent 借用首个注册工作区当 cwd 且标记 origin=subagent，并给出判据与两个根治方向。"
status: "active"
created_at: "2026-11-10T08:36:18.249Z"
updated_at: "2026-11-10T08:36:18.249Z"
content_hash: "0ff7f23a070528902eff38b9f075a63dd29da8683f3e907f7727a6755e90c467"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-e2d8fb63-8f32-4bbe-bc50-b47291ff9da0"
memory_body_ids:
  []
---

# workspace.json 的 `sessionIds` 为何混入 `mnemon-task-v1-*` 与裸 uuid

> 场景：两台机器同步 `workspace.json` 后无法新建会话（`session/agent-busy`）。用户手动清掉 `sessionIds` 中所有 `mnemon-` 前缀 id 后，发现还剩大量无前缀 id，追问「正常操作是否只会写入 `session-` 开头」。核查 DSH 0.2.x 打包产物与 `dsh-mnemon` 插件源码后的结论。相关：`50220fbb-856d-49a7-a9ed-fff16159d00f`（`sessionIds` 是唯一归属依据）、`d1423f79-4ef2-443a-8564-97840eb28e55`（同步合并取并集）、`d08f20c4-349e-4f30-a9c6-69b48441be95`（跨机迁移约束）。本文补的是**成员登记侧不校验 id 形态**这一点。

## 1. 结论先行

「点新建会话只会产生 `session-<uuid>`」成立，但 `sessionIds` 是**成员关系表**，不是新建会话日志。凡是「在该目录下被创建过的 Agent」都会被登记：普通根会话、会话 fork、子代理、mnemon 后台任务、计划任务会话。写入方**没有任何 id 前缀校验**，因此这三类是机制的正常产物而非数据损坏：

| id 形态 | 来源 | 首帧特征 |
|---|---|---|
| `session-<uuid>` | 根会话；**也包括计划任务 `new`/`new_once` 目标**（`SessionId("session-"+randomUUID())`） | 根会话无 `origin`；计划任务会话同样进名单 |
| 裸 `<uuid>` | DSH 原生子代理（subagent / subagent_fork） | `origin:"subagent"` + `parentSession` 指向父会话 |
| `mnemon-task-v1-*`（及 `mnemon-run-v1-*`） | dsh-mnemon 后台任务 Agent 自造 id | `origin:"subagent"` |

所以「前缀」不是判据，**首帧 header 才是**：`origin === "subagent"` = 隐藏型会话；有 `parentSession` = 子代理；只有一帧、约 470 字节 = 空白占位。

## 2. 写入 `sessionIds` 的四条通道（全都不校验形态）

1. GUI/客户端新建会话：`SessionCommandController.create` 末尾 `workspace.attachSession(sessionId)`（app.asar ≈408804-408836）；
2. 会话分叉 fork：app.asar ≈408947；
3. 计划任务投递 `new`/`new_once`：`resolveTarget` → `joinWorkspace` → `registry.resolveByPath(cwd)?.attachSession(sessionId)`（app.asar ≈2457919 / 2457951-2457959）；
4. 配置管理器导入/同步：`dsh-config-manager` 的 `adapters/workspaces.ts:246-265` 逐 id `attachSession`；同步侧 `sync/workspace-merge.ts` 取并集。对端 id 原样落到本机。

`attachSession`（app.asar ≈2929333-2929351）唯一校验是 `realpathNormalize(header.cwd) === record.path`，随后 `rememberSessionPath` 再写入；**若 id 已在名单中则整段校验被跳过**，直接写。读取侧 `get sessionIds()` 用 `candidateMatches` 过滤（≈2929324），不匹配的只是不显示并打 `filtered session '…' from membership` 警告，记录仍在 JSON 里。

即「是否可见」与「是否是成员」是两套逻辑：可见性看 `origin !== 'subagent'`，成员关系只看 cwd 是否等于工作区路径。

## 3. mnemon 的 id 是怎么进来的

`dsh-mnemon/lib/index.js:13729-13758` 的 `runTaskAgent`：

```js
const sessionId = `mnemon-task-v1-${randomUUID()}`;
this.ctx.agents.create({ sessionId, ...creation, signal });
```

DSH 接受任意字符串 id，前缀是插件自命名，DSH 不预留。关键在于 cwd：`taskAgentDirectories`（`lib/index.js:13817-13823`）在调用方未指定时用 `registeredWorkspaces()[0] ?? process.cwd()`；源码注释说明原因是任务 Agent 跑在对话之外，DSH 0.2.0 的 Web persona 要渲染 `{{cwd}}`，无 cwd 会在第一步就 prompt 组装失败（0.2.1 起还会在每步前检查目录存在性）。于是 cwd 恰好等于 `D:\Workspace\DSHarness`，`attachSession` 的唯一校验必然通过。

插件同时把该会话标成 `origin:"subagent"`（≈13759-13771，注释：否则已结束的任务会一直挂在侧栏当「未命名对话」），于是出现「成员里有、侧栏看不到」。

## 4. 与「新建会话报 agent-busy」的关系

新建会话存在复用判据（app.asar ≈408818）：未归档 + 在工作区名单里即可 `reusable`。若复用到的是一条空白且 `origin=subagent` 的占位（多为同步搬来的对端残留），后续 `hasApiSessionSubagentOwner` 判定失败即返回 `session/agent-busy`。因此**真正该清理的是「空白 + origin=subagent」的成员**，而不是按前缀扫。

## 5. 根治方向

- 给 dsh-mnemon 提 issue：后台任务会话不应 `attachSession` 进工作区，或创建时显式跳过登记；
- 给 DSH 提：`reusable` 判据应排除 `origin === "subagent"`；
- 官方修复前，定期清理「空白 + origin=subagent」成员最省事（清理前完全退出 DSH，先备份 `workspace.json`，否则内存旧名单会覆盖改动）。

## 6. 证据与时效

行号取自本机 2026-11-10 的打包产物：`C:\Users\ZY-WORK\AppData\Local\Programs\DeepSeek Harness\resources\app.asar` 与 `C:\Users\ZY-WORK\.dsh\profiles\desktop\node_modules\dsh-mnemon\lib\index.js`。升级 DSH 或插件后行号/函数名需复查。
