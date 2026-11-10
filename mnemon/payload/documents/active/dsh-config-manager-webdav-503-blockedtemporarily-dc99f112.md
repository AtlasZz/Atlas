---
id: "dc99f112-748d-4732-8416-e944e7365f01"
title: "dsh-config-manager WebDAV 同步报 503 BlockedTemporarily（坚果云限流）诊断与处置"
description: "排查 DSH 配置管理器（dsh-config-manager）向坚果云 WebDAV 推送时返回 503 BlockedTemporarily 的根因：服务端账号级限流，而非凭据/路径/DSH 故障；含请求量估算、代码层证据（写路径不重试、MKCOL 单次、加密快照不外置 blob、blob 缺失硬失败）与分级处置方案。"
status: "active"
created_at: "2026-11-10T03:12:30.435Z"
updated_at: "2026-11-10T03:12:30.435Z"
content_hash: "bfe562d3affde8450724ac3ec01e194868baf274ee8c52a6765bdb35ae292a7e"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-e0ad3104-90a0-4192-8314-0e7c2ac339c5"
memory_body_ids:
  []
---

# dsh-config-manager WebDAV 同步 503 BlockedTemporarily（坚果云限流）诊断

> 场景：本机 DSH desktop profile 的「配置同步」走 webdav 通道（坚果云 `dav.jianguoyun.com`），推送时失败，用户问「这是什么问题」。本文记录根因判定、证据与处置顺序。与 `3a0a9387-3717-4526-8005-bb4051b27d2b`（《dsh-mnemon 跨设备记忆同步实操指南》）无关：那篇讲 mnemon 记忆的 Git/ZIP 同步，本文讲 dsh-config-manager 的 WebDAV 配置同步。

## 1. 结论

不是地址/账号填错，也不是 DSH 缺陷，而是**坚果云服务端对该账号做的临时请求频率封禁**：响应体带 `<s:exception>BlockedTemporarily</s:exception>`、`<s:message>Too many requests are received recently</s:message>`，命名空间为 `xmlns:s="http://ns.jianguoyun.com"`，即坚果云自己的错误形态。触发后一段时间内该账号**所有** WebDAV 请求都会被 503 挡回，直到时间窗滑过；重试不会加速解封。

判定依据（错误码分野）：鉴权失败为 401/403；目录/路径问题多为 404/409/405；`503 + BlockedTemporarily` 才对应限流。

## 2. 为什么这套配置容易撞上限流

- 本地 webdav 通道为**高级模式**，勾选了 11 个分区：`settings / ui / providers / plugins / prompts / skills / agentPresets / agentInstructions / self / mcp / sessions`，其中 `sessions` 还指定了 18 个具体会话（`C:\Users\ZY-WORK\.dsh\dsh-config-manager\sync\sync-selection.json`）。
- 该通道的远端布局是「**一个文件一次 PUT**」：文件类分区里每个文件先 PUT 一个 `<sha256>` blob，再 PUT 快照与索引（`lib/sync/blob-store.js` 的 `sectionToBlobRefs`）。
- 单次推送的请求量 ≈ `MKCOL` + `GET index.json` + `GET blobs-index.json` + 每个新增/变更文件 1 次 `PUT` + `PUT 快照` + `PUT index.json`。会话文件多且几乎每次都变，几十到几百个请求会挤在一两分钟内完成。
- 社区/客服口径：坚果云 WebDAV 按账号限频，免费版约 600 次/30 分钟、付费版约 1500 次/30 分钟（此数字来自论坛转述与同类插件反馈，**未在本机实测**）。同类插件（Obsidian Remotely Save、Nutstore Sync）的 503 成因与之相同。

## 3. 代码层证据（为什么它「不会自己好」）

路径均在 `C:\Users\ZY-WORK\.dsh\profiles\desktop\node_modules\dsh-config-manager\lib\sync\` 下：

- **写路径刻意不重试**：`withSyncRetry` 只包裹幂等读；`upload/delete` 禁止重试，因为「PUT 快照」与「PUT 索引」之间失败会留下孤儿文件（`transport.js:50-56`）。
- **MKCOL 只发一次**，且 503 不在「已存在/已建好」的可接受状态集内，直接上抛（`webdav-transport.js:693-714`、`:69`）。
- **加密快照永不外置成 blob**：开启加密后 sections 整体是密文，一次推送只剩 `MKCOL + GET index + PUT 快照 + PUT index` ≈ 4 个请求（`webdav-transport.js:557`）。
- **blob 取不到时硬失败**：任一 blob 缺失即抛错，不静默降级为空分区，以避免「同步成功但对话没了」的老问题形态（`blob-store.js:38-52`，注释指向 issue #45）。
- 自动同步配置存于 `sync-autosync.json`，默认 `interval='30m'`、`startupMinIntervalMs=5min`，含 `consecutiveFailures` 与冷却窗口判断（`lib/sync/autosync-config.js`）；执行记录写 `sync-history.json`（`lib/sync/sync-history.js`）。

## 4. 处置顺序（从低成本到长期）

1. **先停手等 30–60 分钟**，之后只推一次；连续点击会继续消耗额度。
2. **削减推送内容**：把 `sessions` 从 webdav 分区移除（文件最多、变动最频繁），只留少量小文件分区，请求数可从几百降到个位数。
3. **若必须同步会话则开加密**：密文快照不拆 blob，单次约 4 个请求，基本不会触发限流。代价：无法做分区级合并预览，换机需输入密码。
4. **长期**：大量小文件同步与坚果云 WebDAV 限频天然相克，可改用 git 私有仓 / S3 / gist 通道。
5. 若开着自动同步，限流期间会按 `30m` 间隔反复撞同一轮失败，可临时关闭或拉长间隔。

## 5. 未验证项（使用时注意）

- 600/1500 次每 30 分钟的额度为外部转述，非官方文档原文，也未在本机复现计数。
- 本次失败是否由「自动同步定时触发 + 手动重试」叠加造成，未从 `sync-history.json` 内容确认（当时只读到文件结构，未读到条目）。
- 本机 webdav 账号等敏感字段不入文档；重配时直接看 `sync/sync-config.json`（`transport: "webdav"`，schemaVersion 3）。
