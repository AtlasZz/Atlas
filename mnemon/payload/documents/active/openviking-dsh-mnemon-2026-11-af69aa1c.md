---
id: "af69aa1c-1889-42dc-8498-ea728419c830"
title: "OpenViking 记忆插件与 dsh-mnemon 的共存分析（本机实测，2026-11）"
description: "本机 @openviking/dsh-memory-plugin 0.5.33 与 dsh-mnemon 0.5.42 的实测状态、重叠点、静默失败（pending 队列膨胀）原因，以及三条处置路径（只留 mnemon / 只留 OpenViking / 用 mnemon-provider-openviking 作后端）。"
status: "active"
created_at: "2026-11-08T12:37:49.568Z"
updated_at: "2026-11-08T12:37:49.568Z"
content_hash: "ee626919661a5e5855151127e2e7a58c0f36458f42d7f111c54e91393f73d824"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-1b019b66-d94a-4139-8d2b-940057a8dedd"
memory_body_ids:
  []
---

# OpenViking 记忆插件与 dsh-mnemon 的共存分析（本机实测）

结论先行：两者**技术上不冲突**（不抢工具名、服务名、配置项，两个插件均 enabled/active，无硬报错），但**功能严重重叠**（两套互不同步的长期记忆系统），且本机 OpenViking 侧**根本没有服务端**，正处于"静默失败 + 本地队列持续膨胀"状态。

## 一、本机实测状态（2026-11-08 采样，属时间点快照）

| 项目 | 实测结果 | 证据来源 |
|---|---|---|
| `@openviking/dsh-memory-plugin` | 0.5.33，entry `openviking-memory-runtime`，enabled / active | 插件目录 + 运行时状态 |
| OpenViking 服务端 | 不存在：`127.0.0.1:1933` 不可达；无 `~/.openviking/ovcli.conf`；无 `OPENVIKING_*` 环境变量 | 端口探测、`~/.openviking` 仅含 `pending/`、`state/` |
| 运行时依赖 | `ov` / `ovcli` / `openviking` / `docker` 均 NOT FOUND；`node` 存在 | PATH 探测 |
| MCP 工具 | `mcp__openviking__*` 不可用，`list_mcp_resources(openviking)` 返回 `server is disconnected` | MCP 调用 |
| 待发队列 | `C:\Users\zhouy\.openviking\pending\` 积压约 47 个文件 / 160KB（会话期间由 33 涨到 47，随会话线性增长） | 目录统计 |
| `dsh-mnemon` | 0.5.42，正常：每轮注入 `MNEMON RUNTIME MEMORY SNAPSHOT` + `[MNEMON]` 路由提示 | 运行时上下文 |
| mnemon 记忆空间 | 0 个空间、0 个可用 Provider（`@mnemon-dev/mnemon` CLI 未安装），memoryProvider health 明确提示"无空间可写，MEMORY.md 满了会归档到本地文件" | mnemon 状态接口 |
| OpenViking 技能 | 4 个技能可见（`openviking-memory`、`openviking-skills`、`ov-experience-memory`、`openviking-rule-search`），但背后工具断开 | 技能列表 |

核心现实问题：**技能可见、工具不可用**。模型会被引导去读 `openviking-memory` 技能再调用不存在的 `mcp__openviking__*`，白跑一轮；而 Desktop 不展示插件输出，这种失效从界面上完全看不出来。

## 二、OpenViking 插件的工作方式（决定它为何必须"先有服务端"）

它是**客户端**，不是开箱即用：

1. **先有服务端**：默认 `http://127.0.0.1:1933`。需容器/托管实例部署。本机无 docker，这一步是当前硬门槛。
2. **配置连接**：`C:\Users\zhouy\.openviking\ovcli.conf` 填 `url`、`api_key`（`trusted` 模式还需 `account`、`user`）。Windows 上优先用 `ovcli.conf` 而非环境变量；改完要**完全退出并重开 Desktop**（关窗口只是隐藏）。插件**没有设置页面**，行为参数走 `ovcli.conf` 的 `plugin` 段或 `OPENVIKING_*` 环境变量。
3. **验证生效**：新对话顶部出现 `<openviking-context>` 块，工具列表出现 `mcp__openviking__*`。
4. **运行行为**：
   - 召回：每个带用户输入的步骤前插入相关记忆（`recallTokenBudget` 默认 2000 token）
   - 注入画像：会话开始注入一次 profile（`profileTokenBudget` 默认 **10000** token）
   - 自动捕获：每条消息事件发服务端，超过阈值（`commitTokenThreshold` 默认 20000 token / `commitTurnThreshold` 默认 8 轮）后 commit。**dsh 上 `captureToolResults` 默认 true**——文件内容、命令输出也会上传
   - 工具/技能：`search` / `read` / `remember` / `add_skill` 等 + 上述 4 个技能
   - URI 保护：文件工具指向 `viking://` 会被拦下
5. **关键开关**：`OPENVIKING_AUTO_RECALL`、`OPENVIKING_AUTO_CAPTURE`、`OPENVIKING_CAPTURE_TOOL_RESULTS`（建议关）、`OPENVIKING_PROFILE_TOKEN_BUDGET`、`OPENVIKING_RECALL_TOKEN_BUDGET`、`OPENVIKING_DEBUG=1` + `OPENVIKING_DEBUG_LOG=<file>`（唯一排查手段）。

配置解析实现位置（本机 desktop profile）：
- `C:\Users\zhouy\.dsh\profiles\desktop\node_modules\@openviking\dsh-memory-plugin\config.mjs` — `MCP_SERVER_NAME = "openviking"`（该字符串是模型侧契约，改名等于重命名全部 MCP 工具）；`ovcli.conf` 的 `plugin` 段优先于 host input，连接字段例外（host 指定更具体，优先于凭据链）。
- `...\shared\credentials.mjs` — 默认 `~/.openviking/ovcli.conf` 与 `~/.openviking/ov.conf`，凭据链顺序 host → env(未 pin) → ovcli.conf → `plugin.<harness>`，默认端口回退 1933。
- `...\shared\config-schema.mjs` — 全部参数声明与默认值。

## 三、与 dsh-mnemon 的关系

**不冲突之处**：工具命名空间不同（`mnemon_*` vs `mcp__openviking__*`）、服务注册不同（`mnemon` vs `openvikingMemory`）、两者都只在 `agent/pre-step` 追加消息（OpenViking 用 `prepend:true` 排在最后，mnemon 的 View 先固定、后追加，顺序安全）、两份 `cordis.patch.yml` 互不干涉。

**重叠之处**：

| 维度 | dsh-mnemon | OpenViking 插件 |
|---|---|---|
| 定位 | 记忆框架（Source/Strategy/统一 View） | 单一记忆后端 + MCP 工具 |
| 存储 | 本地文件（USER.md/MEMORY.md/文档）+ 可插拔 Provider | 远端 OpenViking 服务 |
| 写入时机 | 引导式（`writebackMode: guided`，自动捕获关闭） | 全自动（每条消息捕获，按阈值 commit） |
| 注入 | 每轮 USER.md/MEMORY.md 快照 + 路由提示 | 首轮 profile（≤10k）+ 每步 recall（≤2k） |
| 技能 | 技能管理层（版本/进化/市场） | 4 个固定技能 |

## 四、并存的具体风险（按严重度）

1. **上下文预算被吃两遍**：mnemon 每轮快照 + OpenViking 首轮 ≤10k profile、每步 ≤2k recall，固定开销叠加。
2. **记忆分裂**：两库互不同步，同一事实可能只进其一；无仲裁机制判断谁更权威。
3. **重复存储**：mnemon 受控写入 vs OpenViking 全量捕获（且 dsh 默认带工具输出）。
4. **静默失败 + 队列膨胀**（本机正在发生）：服务端不通时不报错、不提示，只往 `~/.openviking/pending` 堆；将来服务端可用时 `replayPendingQueue` 会一次性灌入陈旧会话，带来索引成本与过期记忆污染。
5. **模型行为漂移**：两套"何时该记、该查哪里"的指令方向相反（"只写新事实" vs "全部捕获"）。
6. **工具/技能面双份**：每轮在 tool/skill 列表占位，增加选择负担。

无崩溃级冲突——不会互相搞挂，问题是叠加成本与行为混乱。

## 五、三条处置路径

**A. 只留 mnemon（推荐；本机实际在用的就是它）**
停用 OpenViking 插件开关；清空 `C:\Users\zhouy\.openviking\pending\`（纯出站队列，一条都未成功写入，删除不损任何已有记忆）。收益：立即少一份每轮开销、少 4 个不可用技能、停止写盘垃圾。代价：无。注意两个动作分别需要动插件开关（需审批）与写工作区外目录（受文件沙箱限制）。

**B. 只留 OpenViking**
需先部署服务端并装 `ov`/容器运行时。成本最高；收益是语义召回 + 跨会话经验沉淀。适合确实要远端语义记忆时。

**C. 官方设计的组合方式（若要用 OpenViking 作后端）**
不是两个插件并存，而是 `dsh-mnemon` + `dsh-mnemon-provider-openviking`：把 OpenViking 降级为 mnemon 的一个记忆空间 Provider——一套框架、一个后端、一份预算。本机 `dsh-mnemon-provider-openviking` 已在 `node_modules` 中（mnemon Starter 自带，当前未配置）。步骤：服务端可达 → 在「插件 → 可组合记忆 → 记忆空间」填 endpoint / apiKey / account / user 并启用。

## 六、可复用的排查手法

- 判断 OpenViking 是否真的可用：`Test-NetConnection 127.0.0.1 1933`；查 `~/.openviking/ovcli.conf` 是否存在；调一次 `list_mcp_resources("openviking")` 看是否 `server is disconnected`；统计 `~/.openviking/pending` 文件数与体积（增长即代表静默失败）。
- 判断 mnemon 存储能力：看 memoryProvider 健康输出中的 `memorySpaces` / `providers` 计数；为 0 时 MEMORY.md 满后只能本地归档，不会进远端的 Memory Space。
- 插件行为参数一律查 `shared/config-schema.mjs`，不要凭 README 猜测默认值（README 与本机版本存在差异）。
