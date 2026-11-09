---
id: "a8d8fd8f-4a4d-49e2-a755-18f0d03c3679"
title: "dsh-mnemon 可选增强组件（通用策略与三个叠加策略）启用评估（desktop profile 2026-11）"
description: "本机「可组合记忆」里默认关闭的 4 个组件（strategy-general / auto-capture / light-context / scoped）各自作用、默认关闭时的实际后果，以及为何当前数据量下建议全部保持关闭、唯一值得试的路径。"
status: "active"
created_at: "2026-11-08T13:58:19.332Z"
updated_at: "2026-11-08T13:58:19.332Z"
content_hash: "6f61265020b65f9c7b4a54f396fe5eb5fc066d4254be3b2c9c8ff4b10baa022a"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-cd46410e-e987-49da-aeac-309f608e8879"
memory_body_ids:
  []
---

# dsh-mnemon 可选增强组件启用评估（desktop profile，2026-11-08 实机核对）

> 回答「插件 → 可组合记忆」里 4 个默认为「关闭」的组件有什么用、是否有必要打开。结论：**当前配置下 4 个都不必打开**；只有「主动记录」在补装 CLI 并建好记忆空间后才值得试。三层记忆结构与日常运维见 `91943929-4291-4ca0-8170-28e95c15cb55`；OpenViking 关系见 `af69aa1c-1889-42dc-8498-ea728419c830`。

## 1. 组件构成：1 个备选主策略 + 3 个叠加增强

| 包 | 中文名 | 性质 | 作用 | 当前开启的实际后果 |
|---|---|---|---|---|
| `dsh-mnemon-strategy-general` | 通用策略 | **替换**主策略 | 角色无关（role-agnostic）：所有可用 Source 共用一份预算，由模型自行决定怎么用每个来源 | ⚠️ 不驱动任何自动维护：`idleReviewMs` 空闲审查与运行时记忆容量归档同时失效。设置值保留，切回分层策略后恢复 |
| `dsh-mnemon-strategy-auto-capture` | 主动记录 | 叠加 | 每轮提示模型判断本轮是否有值得长期记下的内容（最多 1 条）；无后台任务、不新增写权限 | 本机空跑：默认只针对记忆空间（`durable-evidence` + `remember`），而本机 0 个记忆空间、CLI 未装（`providerReady=false`） |
| `dsh-mnemon-strategy-light-context` | 轻量上下文 | 叠加 | 给所有「每轮注入」内容设共享投影预算上限，省上下文 | 运行时记忆仅约 241 字节，无压力；上限过小会把运行时记忆整个隐藏，且无自动扩容途径 |
| `dsh-mnemon-strategy-scoped` | 范围组合 | 叠加 | 多来源时指定谁优先、谁只读，并收窄写入范围 | 每类来源只有 1 个（运行时 1、项目档案 1、记忆空间 0），不额外配置时行为与现状一致 |

当前主策略是 `mnemon-strategy-default-three-tier`（分层策略），空闲审查 30 秒生效。通用策略属替换而非叠加：同一时间只能运行一个主策略。

## 2. 为什么「主动记录」默认不会写项目档案

- 项目档案 Source 的角色是 `narrative`，对外动作是 `create` / `manage`，**没有 `remember`**。
- 记忆空间 Source 才是 `durable-evidence` + `remember`，但本机无 Provider、无空间，`providerReady=false`，默认配置找不到可写目标。
- 要让它工作：先 `npm i -g @mnemon-dev/mnemon` 并建/激活一个记忆空间；或在它自己的设置页把 `actionIds` 改为项目档案动作（`create` / `manage`）。后一条等于改变其默认语义，需先确认界面支持。

## 3. 启停安全性

- 组件开启即时生效；**切换主策略不需要迁移数据**；关闭不会删除已有记忆。
- 试错成本低，但当前数据量（项目档案 5 份、用户画像 1 条、记忆空间 0 个）下，通用/轻量/范围三者只增加变量不增加能力，建议保持关闭。

## 4. 若将来要试的推荐顺序

1. 安装 CLI：`npm i -g @mnemon-dev/mnemon`；
2. 建一个记忆空间并激活（确认 `providerReady` 变为 true）；
3. 开启「主动记录」观察一两周，嫌吵随时关；
4. 其余三个暂不开启，除非出现明确需求：需要非分层策略（通用）、注入内容超预算（轻量）、同类来源变多需排序/限写（范围）。

## 5. 证据来源

- 包元数据：`C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon-strategy-{general,auto-capture,light-context,scoped}\package.json`（description 与 peerDependencies）、同目录 `cordis.patch.yml`、`README.md`。
- 定义与校验：`C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon\lib\definitions-DU7MLBhu.js`（角色/能力/ID 校验规则）。
- 运行时状态与文案：`...\dsh-mnemon\lib\client.js`（`providerReady` 计算、无记忆空间提示、各组件文案键）。
