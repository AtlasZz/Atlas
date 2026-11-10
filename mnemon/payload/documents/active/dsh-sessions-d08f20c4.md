---
id: "d08f20c4-349e-4f30-a9c6-69b48441be95"
title: "DSH 会话跨设备迁移：为何直接拷 sessions 目录看不见，以及导入/导出与路径映射的正确路径"
description: "DSH 侧栏会话可见性的三条硬约束（首帧 cwd 与目录项目键一致、workspace.json 的 path+sessionIds 登记、日志版本匹配），以及用 dsh-config-manager 导出 ZIP、导入时做路径映射的推荐流程与常见坑（本机 v0.2.0-rc.2 / v4 日志核查）。"
status: "active"
created_at: "2026-11-10T04:43:17.684Z"
updated_at: "2026-11-10T04:43:17.684Z"
content_hash: "a869f41371daf48a147f8110ba94c5349356f8ac789c3a41375f8e3a8e7d2990"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-1710892e-82c5-4b40-837d-e7e1a71f59a4"
memory_body_ids:
  []
---

# DSH 会话跨设备迁移：可见性约束与正确流程

场景：把一台电脑上的 DSH 会话（以及工作区）搬到另一台，或换机恢复。本机核查环境：DSH 0.2.0-rc.2，`DSH_HOME` 默认 `%USERPROFILE%\.dsh`，会话日志为 `session.v4.jsonl.zstd`（首帧 version=4）。证据来自本机安装的 `dsh-config-manager` 源码（`<DSH_HOME>\profiles\desktop\node_modules\dsh-config-manager\src\...`），升级后行号宜复查。

## 1. 为什么「把目录拷过去」经常看不见

一条会话要在侧栏对应工作区里出现，必须同时满足三条，缺一条就表现为「数据在、对话没了」：

1. **日志位置必须与首帧 cwd 自洽**：`<DSH_HOME>\sessions\<项目键>\<sessionId>\session.v4.jsonl.zstd`，且目录段项目键必须等于 `projectKeyOf(首帧 cwd)`。DSH 启动时会校验「日志位置 == projectKeyOf(header.cwd)/id」，不一致直接拒绝启动/加载。
   - `src/adapters/sessions.ts:218-231`（分区收尾 issue #45 的注释即完整不变量说明），`src/core/session-select.ts:46`（`projectKeyOf`）。
2. **工作区必须登记**：`~\.dsh\storages\workspace.json`（或 `~\.dsh\workspace.json`，视版本）中要有 `path` 等于该 cwd 的工作区（DSH 按 `realpath` 比较），且该会话 id 列在其 `sessionIds` 里。
   - 迁移改写首帧时必须写入**逐字一致的本机原生路径**：工作区一侧由 DSH 按 realpath 落库（Windows 为反斜杠原生形），而通用路径映射函数输出正斜杠 —— 两者只差分隔符时目前靠 DSH 的 realpath 兜底，属已知脆弱点（`adapters/sessions.ts:538-560`）。
3. **格式代次匹配**：日志版本必须与目标机 DSH 支持的一致（本机 v4）。v0–v3 的旧日志在 v4 目标机上会被拒收，需先把源机升级到同版本。

两台设备用户名/盘符通常不同，直接复制来的首帧 `cwd` 仍是源机路径，第 1、2 条必然对不上。**正解：走插件导出/导入并在导入时做路径映射改写 cwd，而不是用网盘同步 `sessions` 目录。**

## 2. 推荐流程：ZIP 导入 + 在线路径映射

源设备（导出）：
1. 侧栏打开配置管理器（导航：首页 / 导出 / 导入 / 同步 / 市场），进「导出」→「自定义」。
2. 勾选 **工作区** 与 **历史会话**（会话区默认不勾选，标签「历史会话（默认关闭，含敏感内容）」）；可一并勾设置/插件/提示词/技能/MCP。
3. 安全选项设密码（会话含聊天原文，建议加密），导出到 `<DSH_HOME>\dsh-config-manager\exports\dsh-config-<时间戳>.zip`，再放网盘。

目标设备（导入）：
4. 下载 ZIP 到本机；**建议先退出 DSH** 再操作，避免写冲突。
5. 启动 DSH → 配置管理器 → 导入，走六步：选择 / 分析 / 预览与决策 / 确认 / 执行 / 完成。分析阶段只读。
6. 预览与决策（关键）：逐项冲突决策（保留当前 / 使用备份）；**路径映射**把源机 `D:\Workspace\...`、`C:\Users\<旧用户名>\...` 逐条映射到本机真实路径（「选择本机路径」）；留空即跳过该路径。凭据值不随包带出，需手填或留空。
7. 确认导入，保持「失败时整体回滚（推荐）」勾选（执行前自动建快照）。
8. 完成后**重启 DSH**（界面会提示）。

插件收尾会做三件事，顺序固定：① 改写各 generation 会话首帧 cwd → ② 按新 cwd 把日志归位到 `sessions\<新项目键>\<id>\` → ③ 目录搬不动就**回滚首帧**，绝不留半套（`adapters/sessions.ts:538-575`）。附件 `<sessionId>\attachments\` 随会话一起搬。

## 3. 备选：已经用网盘同步了原始 sessions 目录

仅当两台机工作区绝对路径完全相同（同盘符同目录）时，直接合并 `sessions` + 合并 `workspace.json` 的 `sessionIds` 即可。路径不同则：

1. 把另一台机的 `sessions` 内容放进本机 `<DSH_HOME>\sessions`（插件只扫这一个根，可用环境变量改 DSH_HOME）。
2. 配置管理器首页 →「从其它 agent 导入」→ 选「DSH 会话日志（V4 代次）」→ 开始导入（`src/foreign/dsh4.ts` 与 dsh 源共用同一扫描器与装配器）。
3. 它只读产出「sessions + 工作区连带」分区包，自动送入同一导入向导，后续同第 2 节 6–8 步。

限制：格式不符会被跳过（报「格式不受支持」）；同 id 冲突**不覆盖**本机，只跳过并报告；子代理会话（`origin=subagent`）不单独显示，必须连父会话一起导入。

## 4. 常见坑

- **只导会话不导工作区** → 数据进去了但侧栏不显示，两个分区必须一起勾。
- **路径映射留空** → 首帧仍是旧路径，落不进任何工作区。
- **没关 DSH 就同步** → zstd 日志可能半写；同步前后都先退出 DSH。
- **版本不一致** → 旧版日志被 v4 目标机拒收，先统一版本。
- 事后若确已出现「目录与首帧 cwd 不一致」：配置管理器「环境 → 维护与诊断 → 会话体检」里的「会话布局归位」是写动作（搬运/隔离，需 DSH 停止或走门前置判定），等价命令离线为 `dcm sessions repair --apply`（`src/cli/actions.ts:729`、`src/ui/session-layout-view.ts`）；但「未登记工作区」「容器/首帧读不出」「格式超前」这几类插件不修，需回到第 2 节重新导入或人工处置。

## 5. 可用工具入口

- `config_backup`：白名单含 `workspaces`、`sessions`，支持加密。
- `config_import_plan`：只读生成计划，含冲突与待映射路径。
- `config_import_apply`：需 planId + confirm；计划变化会拒绝执行。
