---
id: "472242a9-d9a3-4048-9090-5c176ce66f9c"
title: "DSH-IM 微信消息 INTERNAL_UNKNOWN：删除工作区导致残留会话绑定（2026-11-04 排查）"
description: "记录删除 DSH「未分组」工作区后微信私聊持续报 INTERNAL_UNKNOWN 的根因（插件 state.json 仍绑定已删除 session）、现场证据、/new 修复步骤与配置兜底，补充既有 dsh-im 微信文档。"
status: "active"
created_at: "2026-11-04T04:52:00.049Z"
updated_at: "2026-11-04T04:52:00.049Z"
content_hash: "2bb5d79af34b6500679cab94de648cf6c6c86bc84d03e1801ada7814a5ebf792"
source_paths:
  []
session_ids:
  - "26a6e7f3-5069-44c9-a149-1117c8fcb480"
memory_body_ids:
  []
---

# DSH-IM 微信消息 INTERNAL_UNKNOWN：删除工作区导致残留会话绑定（2026-11-04 排查）

> 本文接续既有文档 `27846090-fd84-4760-afc2-bdce7f6b1ed9`（扫码绑定失败）与 `65c1d5f4-16f2-4433-a194-4082dc6589a2`（-14 stale-token 与被动回复限制）。阶段不同：此处绑定本身正常，故障出在「工作区删除后插件仍持有已失效的会话绑定」。

## 现象

- 用户删除 DSH 中一个「未分组」工作区，并新建工作区 `C:\Users\ZY-WORK\.dsh\im` 后，微信私聊机器人发送的任何消息都返回：
  「消息已尝试提交，但任务结果尚未确认。请先到 DSH 查看状态，不要立即重复提交……错误码：INTERNAL_UNKNOWN；参考号：MF-0E119ADE」。
- 12:41:35 出现的「你好！看起来消息可能没打完 🙈」是插件对空/残缺消息的固定问候语，不是模型回答，不能当作链路正常的证据。

## 根因与证据

1. 插件状态 `C:\Users\ZY-WORK\.dsh\integrations\dsh-weixin\accounts\wx_e51d0b…\state.json` 中，微信私聊键 `p2p:o9cq80_…@im.wechat` 绑定到 `session-722dc783-ed48-4c6c-a5d2-883e9260f02a`。
2. 该 session 随被删除的「未分组」工作区一起消失，但插件映射没有同步清理。此后每条入站消息都会在该失效绑定上尝试恢复/新建会话，并在提交阶段失败。
3. 现场 `.dsh\sessions\--C-Users-ZY-WORK-.dsh-im--` 下只有 12:43:17、12:45:46 两次失败尝试留下的空会话；解压后仅有 session 头、没有任何消息，说明 prompt 从未真正落地。
4. `INTERNAL_UNKNOWN` 是插件兜底码：Host 只回通用 `internal` 错误、且插件无法确认会话状态时使用（见 `src/channels/shared/message-failure.mjs:149`；`src/channels/weixin/harness-client.mjs` 约 1470 行会保留原始错误继续上抛）。
5. 附带隐患：`C:\Users\ZY-WORK\.dsh\workspace.json` 的 `defaultWorkspaceId = def0ace9-…` 也指向已删除的工作区；现存 `C:\Users\ZY-WORK\.dsh\im` 对应的工作区 id 以 `4bd65765-` 开头（排查时记录，实操前应按现场完整 id 核对）。

## 修复步骤（按顺序）

1. **在微信里对机器人发送 `/new`**。该命令由插件本地处理，不经过 Harness：`src/channels/weixin/weixin-bridge.mjs` 第 821–826 行直接 `state.clearSession(key)` 并回复「已开启新会话。请发送你的问题。」，用于清除失效的会话绑定。`src/channels/shared/new-command.mjs` 中另一条 `/new` 路径会调用 `harness.prepareConversationDirectory(key, { explicitNew: true })` 后再 `state.clearSession(key)`。
2. 随后再发一句「你好」，插件应新建一个干净会话并正常回答。
3. 若 `/new` 后仍报 `INTERNAL_UNKNOWN`：先备份，再清理 `state.json` 中残留的 `sessions` 映射；同时把 `workspace.json` 的 `defaultWorkspaceId` 改为现存工作区（例如 `C:\Users\ZY-WORK\.dsh\im` 对应项）。这两个文件都在 DSH 工作区之外，写入需用户明确授权。

## 关键文件与代码落点

- 插件数据：`C:\Users\ZY-WORK\.dsh\integrations\dsh-weixin\accounts\wx_e51d0b…\state.json`、`C:\Users\ZY-WORK\.dsh\workspace.json`
- 插件源码（`@xmanrui/dsh-im` 5.10.0，desktop profile）：
  - `src/channels/shared/message-failure.mjs:149` — `INTERNAL_UNKNOWN` 兜底映射
  - `src/channels/weixin/harness-client.mjs` 约 1470 行 — 会话状态不可确认时保留原错误
  - `src/channels/weixin/weixin-bridge.mjs` 821–826 行 — 微信 `/new` 本地清绑定
  - `src/channels/shared/new-command.mjs` — `/new` 的 Harness 路径
  - `src/channels/shared/workspace-session.mjs` — 会话绑定锁、工作区代际栅栏（`workspace-session-stale`）
- 相关文档：`65c1d5f4-16f2-4433-a194-4082dc6589a2`、`27846090-fd84-4760-afc2-bdce7f6b1ed9`

## 状态与教训

- 截至本检查点，修复方案尚未经用户实测确认；`/new` 是纯本地逻辑，预期必定生效，失败再走配置清理兜底。
- 可复用教训：**在 DSH 里删除或重建 IM 机器人所绑定的工作区后，应先在 IM 渠道发送 `/new`**，避免插件继续绑定已随工作区删除的会话，表现为持续的 `INTERNAL_UNKNOWN`。
