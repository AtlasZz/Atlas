---
id: "23a04d2a-546b-43e6-bf27-77f492c81d42"
title: "dsh-mnemon 与 OpenViking DSH 插件：区别、共存风险与选型结论"
description: "2026-11-07 调研结论：OpenViking dsh-memory-plugin 是远程记忆服务桥接层，dsh-mnemon 是本地优先完整记忆框架；OpenViking 同时是 mnemon 的 provider。含本机安装审计、双装冲突分析与选型建议。"
status: "active"
created_at: "2026-11-07T10:16:11.801Z"
updated_at: "2026-11-07T10:16:11.801Z"
content_hash: "e3a2e7afbf25fc93c937482ea6c8a370531bcdf6be22b6d0b3038f6ab742bdef"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-1978b46a-f364-43f6-bc9c-967b80f0499b"
memory_body_ids:
  []
---

# dsh-mnemon 与 OpenViking DSH 插件：区别、共存风险与选型结论

> 调研时间：2026-11-07。面向问题：「两个插件有什么区别、能否同时安装、哪个更好用」。上游文档与 npm 信息可能随版本变化，安装前应重新核对版本与接口。

## 结论先行

两者不是同类替代品，而是不同层：

- **@openviking/dsh-memory-plugin**：DSH 的**客户端桥接插件**。自身不存储记忆，把 DSH 接到远程 OpenViking 服务：每步自动召回、捕获会话事件、到阈值提交抽取，并暴露服务端 MCP 工具（`mcp__openviking__search/read/tree/grep/remember/add_skill…`）。记忆抽取、图谱、`viking://` 资源树都在服务端。
- **dsh-mnemon**：**本地优先的完整记忆系统**。三层结构——运行时热记忆（USER.md / MEMORY.md 每轮注入）、项目文档、Memory Spaces；外加 Strategy/Source/Provider 可组合架构、Memory System 界面、回合记忆条、一键保存、备份与 Git 同步。

关键交叠：**OpenViking 本身是 dsh-mnemon 支持的 9 个 Provider 之一**。因此「用 OpenViking 当后端」和「安装 OpenViking 插件」是两条不同路径。

## 两条 OpenViking 接入路径的差别

`dsh-mnemon-provider-openviking` 适配器要求可达的 OpenViking 端点，支持 `POST /api/v1/content/write`（`mode: "create"`、`wait: true`），标签通过 `POST /api/v1/content/set_tags` 设置；作用域固定为 `viking://user/<user>/memories`。Agent 写入的是唯一命名的 Markdown 文件，适配器在返回 `stored` 前校验索引完成并做全文回读；**它不创建抽取会话**。

对比之下，OpenViking 插件走 `dsh-<session-id>` 会话 + 服务端异步抽取。同一后端被两条路径写入时，同一事实会以「会话抽取结果」和「直接 Markdown 资源」两种形态各存一份。

## 本机审计（2026-11-07）

- 活动 profile 为 `desktop`（非 `web`）；`C:\Users\zhouy\.dsh\profiles\web\package.json` 的 dependencies 为空。
- DSH 版本 0.2.2-rc.1；dsh-mnemon 0.5.42 已安装并启用，10 个组件行全部挂载（runtime / documents / memory-spaces / skills + 5 个 strategy）。
- 9 个 provider 包已在磁盘上（`C:\Users\zhouy\.dsh\profiles\desktop\node_modules\`）：openviking、mem0、honcho、supermemory、hindsight、byterover、holographic、mnemon-native、retaindb。
- OpenViking 插件未安装。

即：现有记忆栈是 mnemon，OpenViking 后端可在 Memory Spaces 中直接启用，无需再装插件。

## 能否同时安装

**技术上可以。** 包名、entryId、工具命名空间（`mnemon_*` vs `mcp__openviking__*`）均不冲突，各自注册独立 lifecycle 与 skills provider。

**但不建议同时作为记忆层运行**，有三处实际冲突：

1. **每轮双份记忆上下文**：mnemon 注入 USER/MEMORY 投影，OpenViking 注入 profile + 记忆索引 + 召回块 + 技能目录；叠加后每轮多出数千 token，且可能得到两套不一致的写入指令。
2. **双份捕获**：mnemon 的 auto-capture 与 OpenViking 的 session/event 捕获会记录同一段对话，产生重复事实，OpenViking 侧还要为抽取多付一次 LLM 成本。
3. **同一后端重复写**：两条写入路径语义不同（异步抽取 vs 直接 `.md` + 回读校验），同一事实会重复落库。

若要评估 OpenViking 插件，应使用独立 profile 保持桌面环境干净，例如 `dsh plugin --profile ovtest add @openviking/dsh-memory-plugin`，不要改动现有 `desktop` profile。

## 选型建议

- **单人单机、Windows 桌面、已在用 mnemon** → mnemon 更合适：零运维、有可视化界面、数据在本地、可备份/同步，迁移成本为零。短板是跨机器、跨 AI 工具共享需依赖 Git 同步，且发版较快、偶有版本磨合。
- **同时使用 Claude Code / Codex / Cursor / DSH，需要一套服务端共享记忆 + 资源树 + 技能库** → OpenViking 方案更合适。服务端抽取与图谱更强，代价是维护一个服务、配置模型/嵌入密钥、每次写入产生 token 成本，且 DSH 侧无可视化界面，查记忆只能靠工具。
- **想要 OpenViking 的存储 + mnemon 的界面** → 不装插件，在 mnemon Memory Spaces 中新建 OpenViking provider（endpoint、targetUri，可选 apiKey/account/user）。注意该路径是直接写 Markdown，不触发服务端异步抽取，能力比插件窄。

**最终建议**：保留 dsh-mnemon，不安装 `@openviking/dsh-memory-plugin`。只有确定要跨多个 AI 编程工具共享服务端记忆时，才切换到 OpenViking 插件；切换时应关闭或卸载 mnemon 的记忆源，而不是两者并行。

## 证据与局限

- 本地证据：`C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon-source-memory-spaces\package.json`（版本 0.5.25，peer 依赖 dsh-mnemon ^0.5.19，devDeps 列出 9 个 provider）、各 provider 的 `package.json`、web profile 的 `package.json`。
- 上游证据（截至 2026-11-07）：dsh-mnemon 的 `docs/en/memory-providers.md`（Provider 矩阵：OpenViking 为既有 HTTP 服务 + `viking://` 根，召回/浏览可用，图为投影节点，写入为异步抽取，遗忘为受控硬删除）、`plugins/dsh-mnemon-provider-openviking/README.md`、npm 包页面。
- 局限：本次部分网页抓取通过 DeepSeek fetch 端点返回 HTTP 404，部分细节来自搜索结果摘要而非完整原文；第三方声明与版本约束应以官方仓库当前内容为准。未对 OpenViking 服务做实际连通性测试。
