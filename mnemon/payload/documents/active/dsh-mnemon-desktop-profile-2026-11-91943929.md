---
id: "91943929-4291-4ca0-8170-28e95c15cb55"
title: "dsh-mnemon 记忆系统本机使用与运维指南（desktop profile，2026-11 实机核对）"
description: "dsh-mnemon「记忆系统」的三层结构、三处界面入口、本机实际存储布局与备份方式、容量整理与去重语义、常见坑，以及记忆空间开启的前置条件与建议。"
status: "active"
created_at: "2026-11-08T12:57:29.605Z"
updated_at: "2026-11-08T12:57:29.605Z"
content_hash: "44f9813329b124f903a05d7df2b267abf787cce12b1a77c35d8cc77bcbb2f6bd"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-0f2cab6d-aa30-4efe-a535-da1271b21fc6"
memory_body_ids:
  []
---

# dsh-mnemon 记忆系统本机使用与运维指南

> 面向本机 desktop profile 的实操记录，2026-11-08 实机核对（含读 `C:\Users\zhouy\.mnemon` 数据与插件 `lib/client.js` 文案键）。界面里不出现 "mnemon" 字样，中文名是「记忆系统」，侧边栏找不到插件名属正常。

## 1. 三层记忆，各管一件事

| 层 | 界面名称 | 职责 | 触发方式 |
|---|---|---|---|
| 第一层 | 运行时记忆 | 用户画像（USER.md）+ 工作记忆（MEMORY.md），每轮注入 | 自动，始终在上下文 |
| 第二层 | 项目档案 | 一份份可检索的 Markdown 成果/总结 | 按需检索（document search） |
| 第三层 | 记忆空间 | 大规模、可建关系的长期知识库 | 按需召回，需挂在 Provider 上 |

关键事实：**前两层开箱可用；第三层必须有已启用的 Provider 才能创建**。项目记录优先放项目档案，运行时记忆只放稳定规则与画像，避免同一知识写两层。

## 2. 三处界面入口

1. **侧栏「记忆系统」**：管理后台，四个页签 —— 状态 / 运行时记忆 / 项目档案 / 记忆空间；可查看、手改、删除。
2. **每条回复下方的「本回合记忆」行**：列出本回合用到的记忆工具；右侧「存入记忆」按钮把该回复交给独立任务 Agent 判断是否值得保存、去重提炼后写入，不占用当前对话。保存位置可选：由 Agent 决定（默认）/ 工作记忆 MEMORY.md / 用户画像 USER.md / 记忆空间（无空间时置灰）。
3. **设置 → 插件 → 可组合记忆**：配置层，管 Provider 开关、存储根、策略、空闲审查；日常无需改动。

另：技能管理单独显示在侧边栏，不属于记忆系统（`client.js` 中 `skills.sidebarHint` 明确写了这一点）。

## 3. 本机实况（2026-11-08 采样，属时间点快照，会漂移）

- 存储根：`C:\Users\zhouy\.mnemon\`
  - `runtime\`（USER.md / MEMORY.md 投影）
  - `documents\`（`index.json` + `active\` / `archived\` 托管副本）
  - `skills\`（含 `trash\`）
  - `state\`、`machine.json`（机器 id `344f1caa…`，label `ZY-HOME`，来源 hostname）
- 运行时记忆：1 条（用户画像）。
- 项目档案：3 份 —— dsh-univer-office 使用指南、44 页 PPT 视觉转写工作流、OpenViking 与 dsh-mnemon 共存分析。
- 记忆空间：0 个，0 个已启用 Provider；`@mnemon-dev/mnemon` CLI 未安装（`commandFound: false`）。
- 插件版本 0.5.42，正常每轮注入 `MNEMON RUNTIME MEMORY SNAPSHOT`。
- 同期 OpenViking 0.5.33 无服务端、pending 队列膨胀，与 mnemon 无功能冲突，详见文档 `af69aa1c-1889-42dc-8498-ea728419c830`。

## 4. 存储与备份

- **备份 = 直接复制整个 `C:\Users\zhouy\.mnemon` 文件夹**，换机/重装拷回即可；无需导出或同步。设置里的「同步」默认关闭。
- `USER.md` / `MEMORY.md` 是程序生成的**投影文件**，手改会被覆盖；修改应走界面或让 Agent 调 `mnemon_runtime_memory`。`memories.json` 才是运行时记忆的真实存储。
- 删除项目档案不可撤销：同时移除档案记录与托管副本，不建立冷索引（`documents.deleteDescription`）。

## 5. 容量、整理与去重语义

- 运行时记忆有容量上限，写满时**自动整理**：有可用记忆空间则先归档到空间；没有空间则把移出条目的内容落到**本地归档**（`runtime.result.localArchive` 文案），再写入新条目。属正常行为，不是丢数据。
- 用户画像满时需要任务 Agent 压缩（`saveAction.place.userFull`），不保留存档；因此画像条目要短。
- 单条内容不能含 `§`（条目分隔符），且有单条长度上限。
- 「内容已存在，没有重复写入」= 去重生效，不是失败。
- 在插件页关闭 mnemon **不会删除数据**，`.mnemon` 内容原样保留。

## 6. 记忆空间：是否开启

- 开启前置：本机装全局 CLI `npm install --global @mnemon-dev/mnemon`（提供本地图库），或启用第三方 Provider（如 `mnemon-provider-openviking`）并配置服务。
- 本机建议：**数据量 < 10 条时先不开**。前两层已覆盖"记住偏好 / 找回成果"的日常需求；记忆空间适合数百条以上且需要按关系检索的重型场景，提前开启只增加复杂度。

## 7. 使用场景速查

| 需求 | 说法 | 落点 |
|---|---|---|
| 记固定偏好 | “记住：我写材料习惯用一是/二是/三是，不要小标题套小标题” | 运行时记忆 |
| 记项目规矩 | “记住这个项目的规矩：审查意见按问题—依据—建议三段写，依据引到条款号” | 工作记忆 / 项目档案 |
| 新对话续接 | “上次那个 44 页 PPT 的转写，当时定的工作流调出来” | 项目档案检索 |
| 纠正/别乱记 | “把运行时记忆里关于 XX 那条删了”“这个别记，临时的事” | 运行时记忆 |

## 8. 证据来源

- 存储与状态：`C:\Users\zhouy\.mnemon\documents\index.json`、`runtime\memories.json`、`machine.json`。
- 行为与文案契约：`C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon\lib\client.js`（`saveAction.*`、`runtime.result.*`、`documents.*`、`skills.sidebarHint`、`turnTail.*`）、`lib\index.js`。
- OpenViking 共存与 pending 膨胀：见 `af69aa1c-1889-42dc-8498-ea728419c830`。
