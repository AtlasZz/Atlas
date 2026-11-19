---
id: "992e9359-3e20-4903-9a3d-0a99d687c55e"
title: "DSH 桌面 profile 插件「单座位槽位」冲突排查与停用流程（prompt-forge vs ui-voice-input，2026-11）"
description: "本机 desktop profile 中 dsh-prompt-forge 与实验性语音输入插件争抢 conversation.input.activity 单座位槽位导致页面插件同步失败：根因、按 id 追加 disabled 的处置、清单/fiberPhase 验证、手动刷新要求与可复用排查步骤。"
status: "active"
created_at: "2026-11-17T10:53:31.882Z"
updated_at: "2026-11-17T10:53:31.882Z"
content_hash: "de0aa766f444bb4839e1e8c660ae98c5d6c56cffce22d52472666b83692900ea"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-2ee2aa30-85f7-474d-9aa6-f46c369ca5a4"
memory_body_ids:
  []
---

# DSH 桌面 profile 插件「单座位槽位」冲突：prompt-forge vs ui-voice-input

> 2026-11-17 本机实测，用户目标：解释设置页报错原因并给出可执行修复。
> 配置文件：`C:\Users\zhouy\.dsh\profiles\desktop\cordis.patch.yml`（desktop profile）。

## 1. 现象

设置 → 内置插件显示「本页面的插件未能完成同步」；页面端注册槽位时报错，✨ 按钮与语音麦克风都不可用。

## 2. 根因

- 客户端插件把入口按钮注册到槽位 `conversation.input.activity`，该槽位是 **single（单座位）**，同一槽位只允许一个注册者。
- `dsh-prompt-forge` 与 `@deepseek-ai/dsh-experimental-client-ui-voice-input` 都注册该槽位，且都未声明 `priority`（默认 0）。**同优先级重复注册会直接抛错，而不是静默覆盖**，于是页面侧插件同步整体失败。
- prompt-forge 的注册是硬编码的（其客户端 `client.js:2811` 附近，未带 priority），升级后行为不变。

## 3. 处置（用户选择：保留 prompt-forge，停用语音 UI）

在 patch 文件末尾追加（patch 按 `id` 匹配已有条目）：

```yaml
- id: ui-voice-input
  name: "@deepseek-ai/dsh-experimental-client-ui-voice-input"
  disabled: true
```

- 只停「抢座位」的前端 UI 插件；转写后端 `speech-to-text`、`speech-to-text-sensevoice`、`api-speech-to-text` 保持启用。
- 要彻底关掉整套语音：再给这三个后端各加 `disabled: true`。
- 恢复麦克风：删掉该条目或改 `disabled: false`，冲突不会复发。

## 4. 验证方式（只读文件不算验证）

- 改动经热更新生效，**无需重启 DSH**。插件清单中：`include:prompt-forge` → `enabled: true`、`fiberPhase: active`；`include:ui-voice-input` → `enabled: false`、`fiberPhase: null`（= 未挂载，不再注册座位）。
- 本机插件清单接口需要鉴权：`/api/plugins`、`/api/settings`、`/api/health` 不带凭据一律 `401 unauthorized`，`/healthz` 为 404。**401 是鉴权缺失，不是插件故障**，别据此误判。

## 5. 页面端必须手动刷新

`dsh-unlag` 日志（`~/.dsh/unlag-reload.log`）明确写着：`plugin reload is manual`、`auto reload is off`。文件改完后浏览器仍缓存着上次加载失败的插件列表，红字不会自己消失 —— 让用户按刷新按钮或刷新窗口即可，**不要为此重启应用**。

## 6. 可复用排查步骤

1. 见到 slot 冲突/「插件未能完成同步」→ 在插件清单里按 `moduleName` 找出两个注册者，确认目标槽位是 single 还是 list。
2. 在 profile 的 `cordis.patch.yml` 末尾追加 `id` + `name` + `disabled: true`（只写 id 不足以唯一定位条目）。
3. 查插件清单确认 `enabled` 与 `fiberPhase` 变化（`fiberPhase: null` 表示已卸载）。
4. 让用户手动刷新页面，再复查设置页与控制台。
5. 升级被保留的插件后复查：若它仍硬编码单座位且不带 priority，冲突只会在重新启用另一方时复现。

## 7. 真正共存的做法（本次未实施）

把 prompt-forge 的按钮改注册到 **list 型槽位 `conversation.input.right`**（可多占），这需要改它的客户端代码并跟进上游（patch 或 issue）。只调 `priority` 只能决定注册先后，无法让两个 single 座位共存。

## 8. 同时发现的配置丢失（备查，未恢复）

- 2026-11-17 18:40 左右 `cordis.patch.yml` 被重写，`image-gen` 自定义 provider（openai-compat）、`ui-theme`、`better-sidebar` 等条目丢失，图片生成回落到默认 `google / gemini-3.1-flash-image`。
- 旧配置备份：`cordis.patch.yml.bak-1794567780474`（11-13 18:34）。
- 用户当时明确选择「不用，先不管」；恢复需从备份挑条目回填，不要整文件覆盖（会再次冲掉后来新增的 `disabled: true`）。
- 环境小坑：本机 Python 无 PyYAML（`ModuleNotFoundError: No module named 'yaml'`），校验 YAML 需换手段（node + js-yaml，或直接读插件清单 JSON）。
