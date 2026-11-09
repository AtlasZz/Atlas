---
id: "27846090-fd84-4760-afc2-bdce7f6b1ed9"
title: "DSH-IM 微信渠道扫码绑定失败排查记录（本机 desktop profile，2026-11-04）"
description: "本机 @xmanrui/dsh-im 5.10.0 微信扫码绑定失败的实测证据、链路机制、配置/凭据落点与后续排查建议，含鸿蒙 ClawBot 兼容性线索。"
status: "active"
created_at: "2026-11-04T04:22:47.130Z"
updated_at: "2026-11-04T04:22:47.130Z"
content_hash: "436e2c01567bdb735cf77ecbf0bb61319db70bb5036c4905e89fb90affea907b"
source_paths:
  []
session_ids:
  - "a5a5787a-7415-4c3a-8f91-351e339820e7"
memory_body_ids:
  []
---

# DSH-IM 微信渠道扫码绑定失败排查记录（本机 desktop profile，2026-11-04）

## 背景与结论

用户询问 DSH-IM（@xmanrui/dsh-im）是否值得用，并反馈「鸿蒙手机扫码连不上微信」。排查结论：**插件侧与电脑侧链路均正常，问题卡在扫码后的确认/激活环节**，最大嫌疑是鸿蒙微信客户端对微信 ClawBot / iLink 扫码页的兼容性（腾讯自家 WorkBuddy FAQ 亦记录鸿蒙 6 存在该不兼容、鸿蒙 4 可正常连接）。

## 本机环境事实

- `DSH_HOME` = `C:\Users\ZY-WORK\.dsh`，profile 为 `desktop`；DSH Web GUI 监听 `127.0.0.1:25012`。
- 插件实装版本 `@xmanrui/dsh-im` **5.10.0**（npm 最新版，2026-11-03T16:53:33Z 发布；本机即为该版本，无更新可做）。
- 插件 `package.json` 的 `compatibility` 仅声明 `profiles: ["web"]`、核对版本 `0.2.1-rc.2`；宿主实际为 `0.2.1-rc.5`（同一条 rc 线，作者未标注覆盖）。
- `~/.dsh/integrations/` 下只有 `dsh-im` 与 `dsh-weixin` 两个目录 → 用户只试过微信这一个渠道。
- 插件产物路径：`~/.dsh/profiles/desktop/node_modules/@xmanrui/dsh-im/lib/index.js`（约 10 MB 单文件 bundle），README 在 `.../@xmanrui/dsh-im/README.md`（第 96 行渠道概述、第 194 行「设置 → IM机器人」入口、第 262 行侧边栏入口）。

## 绑定从未成功的证据（~/.dsh/integrations/dsh-weixin/）

- `accounts/` 下 **3 个空目录**，时间戳 12:13:09 / 12:14:02 / 12:15:10（约 2 分钟内 3 次尝试，均未走完）。
- `config.json` 中 `"accounts": []`。
- 凭据库 `~/.dsh/.credentials.yaml` 中没有 `DSH_WEIXIN_BOT_TOKEN_*`。
- 群历史库 `group-history.sqlite` 的 groups / messages / attachments 三张表均 0 行 → 从未有消息进出。

## 网络侧实测（排除「网络不通」）

- DNS 可解析 `ilinkai.weixin.qq.com`；TCP 443 对 61.151.230.245 / 180.101.242.203 / 117.89.176.78 均连通。
- 系统 `curl` 因 schannel 报 `SEC_E_NO_CREDENTIALS (0x8009030e)`（客户端证书/凭据问题），不代表网络故障。
- 用本机 Node 直接请求微信 iLink 接口成功：
  - `GET https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3` → 返回 `{"qrcode":"<uuid>","qrcode_img_content":"https://liteapp.weixin.qq.com/q/...&bot_type=3","ret":0}`。
  - `GET .../ilink/bot/get_qrcode_status?qrcode=nonexistent-probe` → `{"ret":0,"status":"expired"}`。
- 结论：DNS / 防火墙 / 代理 / TLS 全程正常，二维码获取与状态查询接口可用。

## 链路机制（易误判点）

微信通道是**电脑主动外连微信 iLink 云端，手机只在微信内扫码确认**；手机与电脑是否同网、同 Wi-Fi 无关。扫码后微信打开 `liteapp.weixin.qq.com` 的 ClawBot 绑定页，卡点在该页面能否打开并完成确认。若「连不上」实际指手机打不开 DSH Web 页面，那是另一回事：默认只监听 `127.0.0.1:25012`，需 `dsh web --host 0.0.0.0 --trusted-host <IP或主机名>` 并放行防火墙。

## 插件源码要点（判断错误归属用）

`lib/index.js` 内：`LOGIN_STATUSES` 白名单校验 `get_qrcode_status` 返回；`beginLogin()` 调 `ilink/bot/get_bot_qrcode?bot_type=${botType}`；`pollLogin()` 调 `ilink/bot/get_qrcode_status`；`normalizeWeixinQrUrl()` 处理 `qrcode_img_content`；异常文案含「微信服务返回了无法识别的扫码状态」「没有返回二维码令牌」。即二维码与轮询链路在插件内是完整实现，问题更可能在微信客户端侧确认环节。

## 原因排序与建议

1. **鸿蒙 6 微信扫码不兼容**（腾讯 WorkBuddy 官方 FAQ 记录；社区同通道产品同样踩坑）。临时方案：借安卓机登录**用户本人**微信完成首次扫码，再回鸿蒙使用；扫码期间电脑须开机不休眠。
2. **微信版本/灰度不足**：微信侧 ClawBot 能力需 8.0.16.51+（鸿蒙版）且账号被灰度；检查「我 → 设置 → 插件」是否有「微信 ClawBot」。
3. **该微信号已绑过其他机器人** → 走 `binded_redirect`，提示已绑定，需解绑或换号。
4. **概念混淆**：把「手机打不开 DSH 页面」误当成 IM 绑定失败。

下一步取证：设置 → IM机器人 → 微信 →「诊断详情 → 复制诊断信息」，关注错误码 `network-error` / `timeout` / `incomplete-login` / `binded_redirect`；同时确认卡点步骤（二维码不显示 / 扫了无反应 / 提示已绑定 / 显示已连接但不回消息）、手机型号、鸿蒙版本、微信版本。

## 插件评估要点（供选型复用）

优点：DSH 生态渠道最全（飞书、微信、钉钉、企业微信含应用、QQ、Slack、Telegram、Discord、WhatsApp、iMessage、Matrix 等），支持多机器人与独立工作区/模型/会话、群历史留存（默认 14 天）、超时补发、文件/图片回传、AI Office Connector，MIT 协议、更新频繁。注意：体积大（约 10 MB bundle）、权限面大；兼容声明只覆盖 web profile；微信通道本质是封装腾讯 iLink / ClawBot 灰度能力，插件无法绕过微信客户端限制。急用可先走免扫码渠道（Telegram BotFather token、飞书）。

## 取证过程中的工具问题

- 内置 Web fetch（DeepSeek `/v1/tools/fetch`）对部分 URL 返回 HTTP 404（request_id 37ff1045…、1f056454…），需在「插件 → 内置插件设置 → Web 搜索」里改 base URL；仅用户可决定端点。
- 有一处 web fetch 超时（30s），换来源后成功。
