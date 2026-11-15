---
id: "b7c59b01-37ef-4a88-8120-dc4fa665e108"
title: "dsh-mineru 拖拽 docx 恒报“请先打开一个会话”根因与修复（DSH 0.2.2-rc.1）"
description: "本机 dsh-mineru 0.1.15 拖拽/粘贴桥接在 DSH 0.2.2-rc.1 上必然失败的代码级根因（读取不存在的 sessions.list.current）、原生拖放被 capture 阶段拦截的连带影响、上游版本核对结论，以及四种处置方案与热修补丁。"
status: "active"
created_at: "2026-11-13T12:06:43.269Z"
updated_at: "2026-11-13T12:06:43.269Z"
content_hash: "edab639708903b7ceea6bf64c47480ecbf4ec01aa0163fc66cda73b766e173bb"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-36d9a902-5792-4d77-b2d9-eaf739fa62c9"
memory_body_ids:
  []
---

# dsh-mineru 拖拽 docx 恒报“请先打开一个会话”根因与修复

结论：该提示**不是 DSH 本体**发出的，而是本机插件 **dsh-mineru 0.1.15** 的拖拽桥接逻辑在 **DSH 0.2.2-rc.1** 上必然走失败分支；与“是否打开了会话”无关，重开会话/重启 DSH/换 docx 文件都无效。属于插件兼容性回归，不是 DSH 拖放功能坏了。

## 一、报错来源

- 文案表：`C:/Users/zhouy/.dsh/profiles/desktop/node_modules/dsh-mineru/lib/client.js` 第 18 行
  `"toast.noSession": "请先打开一个会话, 再拖入文档."`
- 触发点：同文件 `installFileDropBridge` 内的 `intake()`（约 :676-681），取不到当前会话 id 即弹 toast 并 `return`。
- 用户口述的“打开一个绘画／拖入文件”即这句（会话→绘画、文档→文件，同音）。

## 二、代码级根因

:654 取当前会话：

```js
var currentSession = function () { return ctx.sessions.list.getSnapshot().current; };
```

但 DSH 0.2.2-rc.1 的 `sessions.list` 快照只 set 四个字段，没有 `current`：

```js
// dsh-api-session-controller/.../client/sessions/service.js:566
this.list.set({ ids, byId, phase, projectionsBySession });
```

因此 `sessionId` 恒为 `undefined`，每次拖入都命中 `if (!sessionId) { toast(...); return; }`。

## 三、原生拖拽同时“失灵”的原因

- 插件在 `document` 上以 **capture 阶段**注册 `dragenter/dragover/drop/dragend/paste`（约 :753-757），`onDrop` 内 `preventDefault() + stopPropagation()`。
- DSH 原生附件拖放监听挂在 `document` 的 **bubble 阶段**，捕获阶段即被掐断 → 原生处理收不到事件，表现为“拖进去只有一句错误提示，连文件卡片都不出现”。
- 仅白名单扩展名（:644 `SUPPORTED_DROP_EXT`，含 `.doc/.docx/.pdf/...`）会被处理，其他文件拖入完全无反应；图片扩展名另见 `IMAGE_DROP_EXT`；上限 `DROP_MAX_BYTES = 210MB`。
- 插件还带一个兜底：`killOverlay()` 合成 `dragend` 事件，用于清除 OS 拖拽导致卡住的 composer 遮罩。

## 四、版本核对（升级不管用）

| 项 | 结果 |
|---|---|
| 本机版本 | 0.1.15（2026-10-24） |
| npm 最新 | 0.1.18（2026-11-11） |
| 上游 CHANGELOG 0.1.16–0.1.18 | 仅修 TOKEN 测试、设置面板入口、结果 CDN 劫持，**无本条修复** |

## 五、四种处置方案

**方案 A｜绕开拖拽（最省事）**：拖拽入口坏但 `mineru_parse` / `mineru_batch_parse` / `mineru_task` 工具、mineru-tools 技能、`/plugin/mineru/upload` 路由均正常。把文件放磁盘任意位置，直接在对话里写“用 mineru_parse 解析 `D:\...\x.docx`”，或放进工作区后用 `@` 引用。代价：手打路径。

**方案 B｜停用插件，回退原生拖拽**：插件市场停用/卸载 dsh-mineru 并重启，docx 走原生附件通道（原生有 `kind:"file"` 通用附件与上传进度卡片）。代价：失去 MinerU 解析，docx 进对话后能否被读取取决于 DSH 文档处理，通常还得转文本。

**方案 C｜热修补丁（保留插件又修好拖拽，长期推荐）**：把 :654 改为：

```js
var currentSession = function () {
  var snap = ctx.sessions.list.getSnapshot() || {};
  if (snap.current) return snap.current;
  var ui = null;
  try { ui = ctx.get("uiSession"); } catch { /* not provided */ }
  var key = ui && ui.current && ui.current.value && ui.current.value.key;
  if (key) return key;                       // 主视图当前会话（DSH 的真实来源）
  var byId = snap.byId || {}, ids = snap.ids || [];
  for (var i = 0; i < ids.length; i++) {     // 兜底：被主视图持有的会话
    var e = byId[ids[i]];
    if (e && e.retainedBy && (e.retainedBy.mainView || 0) > 0) return ids[i];
  }
  return void 0;
};
```

步骤：① 退出 DSH → ② 备份 `lib/client.js` → ③ 替换该行 → ④ 重启 DSH → ⑤ 拖 docx 验证。
预期：提示“已上传 N 个文件”，输入框草稿出现“我拖入了以下文件（已保存到工作区）- <路径>”，**不自动发送**（`fillDraft` 只写草稿）。
注意：插件升级/重装会覆盖补丁，需重打；`uiSession` 段用 `try/catch` 保证该服务不存在时安全退化。文件在工作区之外（`%USERPROFILE%\.dsh\profiles\desktop\node_modules`），打补丁需要一次工作区外写权限。

**方案 D｜治本**：向上游提 issue——`installFileDropBridge` 读取 `sessions.list.getSnapshot().current`（DSH 0.2.x 无此字段）导致拖拽恒报 noSession，附方案 C 的改法。

## 六、当前状态

截至本次排查结束，补丁**尚未应用**（助手仅提出代打，等待用户授权工作区外写权限）。本会话内用于分析的 `_scan_out.txt`、`_att.js`、`_svc.js`、`_ui_plugin.md`、`_practices.md` 等为临时取证文件，可删除。

## 七、可复用要点

- 排查此类“插件级 UI 失效”应先定位文案字符串所在包，再核对该包读取的宿主快照/服务字段是否在当前 DSH 版本仍存在，最后才怀疑用户状态与文件格式。
- DSH 中 `ctx.sessions.list` 快照字段会随版本变化；判断“当前会话”的稳定来源是 `uiSession` 服务或快照里的 `retainedBy.mainView` 标记。
- document 上 capture 阶段的 `preventDefault/stopPropagation` 会静默吃掉宿主 bubble 阶段的原生拖放，是插件拖放桥接类 bug 的共性风险点。
