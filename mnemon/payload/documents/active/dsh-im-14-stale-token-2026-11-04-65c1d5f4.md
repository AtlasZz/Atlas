---
id: "65c1d5f4-16f2-4433-a194-4082dc6589a2"
title: "DSH-IM 微信渠道换绑成功：-14 stale-token 根因与连接后被动回复限制（2026-11-04 补充）"
description: "更正前一篇微信绑定失败记录：绑定实际可成功，-14 来自服务端旧 bot 替换冲突；并记录连接后主动消息 provider=-2/contextToken=no 的机制、判定与修复步骤。"
status: "active"
created_at: "2026-11-04T04:43:47.090Z"
updated_at: "2026-11-04T04:43:47.090Z"
content_hash: "3364ca851f9010fc8f8ec07cd93b8a60027c66bf78a4f70e9b253da1c9e8130b"
source_paths:
  []
session_ids:
  - "db465f33-9bbf-4577-9b6c-b66c19e4ccaa"
memory_body_ids:
  []
---

# DSH-IM 微信渠道换绑成功与后续限制（2026-11-04 补充）

> 本文是对既有文档 `27846090-fd84-4760-afc2-bdce7f6b1ed9`（《DSH-IM 微信渠道扫码绑定失败排查记录》）的**续记与更正**。前一篇记录的是「扫码无反应、绑定从未成功」阶段；本篇记录同一天后续的突破：绑定成功、`-14 stale-token` 的真实根因，以及连上之后「主动消息发不出去」的原因。阅读时以本篇结论覆盖前篇中「鸿蒙不兼容是首要原因」「绑定从未成功」等已过时判断。

## 一、结论先行

1. **扫码绑定链路本身是通的**：同一台鸿蒙手机、同一个微信号，在后续一次尝试中完成了「扫码 → 微信打开 liteapp 绑定页 → 确认 → 回传凭据」全流程，`config.json` 落盘新 bot，运行时状态检查通过。
2. 之前那次失败**不是**鸿蒙微信打不开 liteapp，而是：账号上还挂着旧的 OpenClaw 连接，微信弹出「当前微信已连接一个 openclaw，继续连接将解除原有的连接」，点击「继续连接」后服务端替换未真正生效，新 bot 处于未激活状态；随后插件拿新 token 调 `notifystart`，微信返回 `-14`，插件映射为 `stale-token`，界面提示「请移除失效接入并重新扫码绑定」。
3. 连接成功后仍会出现红字「微信主动消息发送失败」，诊断形如 `endpoint=sendmessage contextToken=no http=2xx provider=-2`。这**不代表掉线**：iLink 机器人通道以被动回复为主，没有会话上下文（context token）的主动推送会被服务端拒绝。

## 二、失败链与证据（可复现的判定顺序）

时间线（本机，2026-11-04，Asia/Shanghai）：

| 时间 | 事件 |
|---|---|
| 12:31:36 | 一次 `config.json` 写入新账号（botId `wx_…`，`connectedAt` 与 `createdAt` 相同），凭据库同步写入 `DSH_WEIXIN_BOT_TOKEN_<BOTID>` |
| 12:31:55 | 该账号 `accounts/<botId>/state.json` 初始化（准备消息同步） |
| 12:32:02 | 运行时调 `POST ilink/bot/msg/notifystart`，微信返回 `ret=-14` → 插件 `weixin-api.mjs` 把 `-14` 映射为 `stale-token`；`weixin-runtime.mjs` 同样把 `-14` 视为登录凭据失效 |
| 之后 | 删除该失效接入，重新扫码；新账号绑定成功，通道检查通过，`-14` 消失 |

关键代码落点（插件 `@xmanrui/dsh-im` 5.10.0，未改动源码，仅用于判断错误归属）：

- `src/channels/weixin/weixin-api.mjs`
  - `notifyStart()` 请求 `ilink/bot/msg/notifystart`，`providerCode === '-14'` 时抛 `stale-token`（约 834–836 行）。
  - `beginLogin()` 走 `ilink/bot/get_bot_qrcode?bot_type=…`；`pollLogin()` 走 `ilink/bot/get_qrcode_status?qrcode=…`；`normalizeWeixinQrUrl()` 校验 `qrcode_img_content` 必须是 `https:` 的微信域名。
  - `sendText()/sendImage()` 调 `ilink/bot/sendmessage`，仅当有非空 `contextToken` 时才带 `context_token` 字段。
- `src/channels/weixin/weixin-controller.mjs`：扫码登录时把 `login.qrcode`、`login.qrcodeUrl` 写入记录；`#activateAccount` 完成 token/accountId/baseUrl 落地。
- `src/channels/weixin/weixin-runtime.mjs`：`-14` → `stale-token` → 文案「微信登录凭据已失效，请移除账号后重新扫码。」
- `src/channels/weixin/connection-error.mjs`：`stale-token`/`missing-token` 的用户指引为「请移除失效接入并重新扫码绑定。」

其他实测事实：

- 扫码后手机提示「已连接一个 openclaw，继续连接将解除原有的连接」，点一次「继续连接」后同一弹窗再次出现，再点一次即「二维码过期」。**一张二维码只能确认一次**，第二次点击必然过期；因此不要在手机上反复点。
- 二维码有效期约 5 分钟（插件源码常量 `QR_TTL_MS`；本次未在产物 bundle 中再核对，按约 5 分钟对待即可）。过晚扫码在手机端表现为「什么都不发生」，很容易被误判成扫码不识别。
- 失败的尝试会在 `accounts/` 下留下若干空目录（本次现场有 4 个空目录，分别对应 12:13、12:14、12:15、12:31 左右的尝试），不影响功能，但会让现场看起来「绑过很多次」。

## 三、实测有效的修复步骤

1. **在手机微信里主动断开旧的 ClawBot/OpenClaw 连接**：微信 →「我」→「设置」→「插件」→「微信 ClawBot」，找到已连接的 OpenClaw/设备，断开或移除。代价是原 OpenClaw 通道失效，需确认可接受，或改用另一个微信号绑定 DSH。
2. **在 DSH 侧删掉失效接入**：设置 →「IM机器人」→ 微信 → 移除账号。面板删不掉时，等价于清理：
   - `%USERPROFILE%\.dsh\integrations\dsh-weixin\config.json` 中对应 `accounts[]` 条目；
   - `~/.dsh/.credentials.yaml` 中对应的 `DSH_WEIXIN_BOT_TOKEN_<BOTID>`；
   - `%USERPROFILE%\.dsh\integrations\dsh-weixin\accounts\<botId>\` 残留目录。
   该操作不可逆，删除前先确认 botId 与 tokenRef 对得上。
3. **重新生成二维码后尽快扫，并且只点一次确认**；扫完立即回电脑看面板状态应从「等待扫码」推进到「已连接」。
4. 断开旧连接后仍稳定复现 `-14`，再考虑借安卓机用**同一微信号**完成首次绑定（绑定跟随账号而非设备），成功后回鸿蒙微信正常聊天；这是兜底，不是首选。

成功判据（三项同时满足）：

- `config.json` 出现唯一一条新账号，`connectedAt` 为本次时间；
- 对应 `accounts/<botId>/state.json` 已生成，且 `contextTokens.credentialHash` 与当前凭据一致；
- 诊断信息不再出现 `-14` / `stale-token`，且能通过一次真实入站消息得到回复。

## 四、连上之后：为什么「检查连接」会报主动消息失败

- 「检查连接」按钮走 `sendConnectionTest(text)`：目标优先取最近一次入站消息的 `toUserId`，没有则回退到 `config.ownerUserId`；要求运行时 `ready`，否则抛 `Weixin runtime is not connected`。它会**真的发一条消息**，不是纯探测。
- 此时尚未有人在微信里对机器人说过话，`state.json` 里没有该用户的 context token，请求不带 `context_token`；微信返回 HTTP 2xx 但在业务层拒绝（`provider=-2`，插件文案：「微信主动消息发送失败。长轮询在线不代表消息可以发送。」）。
- 因此正确验收方式是：**先在微信里给机器人发一句「你好」**，看是否回复；回了就说明链路通，红字可忽略。
- context token 的持久化与失效：
  - 存于 `accounts/<botId>/state.json` 的 `contextTokens.users[userId].token`，并与 `credentialHash` 绑定；换绑/换凭据会重置 users（`bindContextTokens()`）。
  - 仅接受非空 `contextToken` 且带合法 `userId` 才记录（`rememberContextToken`）。
  - `recentOutboundMessages` 保留 30 天（`WEIXIN_RECENT_OUTBOUND_TTL_MS = 30 * 24 * 60 * 60 * 1_000`），用于回复引用等场景。
- `-2` 不只有「缺 context token」一种含义，插件自身的提示还包括会话有效期、发送额度、消息内容等原因；会话静默过久失效后，让对方再发一条消息即可恢复。**不要靠反复发心跳/反复点检查连接续期**，插件明确劝阻。

## 五、环境与路径速查

- 集成数据根：`%USERPROFILE%\.dsh\integrations\dsh-weixin\`（`config.json`、`workspaces.json`、`accounts\<botId>\state.json`）。
- 凭据库：`~/.dsh/.credentials.yaml`，键名 `DSH_WEIXIN_BOT_TOKEN_<BOTID>`，值为 `<accountId>:<secret>` 形态（不要把值贴到聊天或文档里）。
- 插件产物：`~/.dsh/profiles/desktop/node_modules/@xmanrui/dsh-im/`（`lib/index.js` 为单文件 bundle；源码在 `src/channels/weixin/`）。
- 本机：Windows `win32`，Node `24.21.0`，插件 `5.10.0`，DSH Web `127.0.0.1:25012`。
- 微信渠道默认工作区是插件给的草稿目录 `C:\Users\ZY-WORK\.dsh\im`；要在微信里让 DSH 干活，应先在面板「选择目录」改成实际项目目录（如 `D:\Workspace\DSHarness`），否则代理会在空目录里操作。

## 六、对前篇文档的更正清单

- ❌「绑定从未成功」→ 只对 12:13–12:31 那几次成立；12:38 之后已成功绑定并连上。
- ❌「鸿蒙 6 微信扫码不兼容是首要原因」→ 本次同一台鸿蒙机成功走完扫码与 liteapp 确认，兼容性应降级为兜底假设，优先排查「旧 OpenClaw 连接未解除」。
- ✅ 仍成立：二维码/轮询接口可用；失败时先看诊断错误码；`-14` 属于凭据侧而非网络侧。
- ⚠️ 未从片段确认用户当时是否真的在手机里执行了「断开旧 ClawBot」；但服务端替换冲突这一机制与 `-14` 的对应关系有代码与时间线双重证据。
