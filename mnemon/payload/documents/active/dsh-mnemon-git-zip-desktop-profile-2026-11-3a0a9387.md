---
id: "3a0a9387-3717-4526-8005-bb4051b27d2b"
title: "dsh-mnemon 跨设备记忆同步实操指南（Git 同步 / ZIP 迁移，desktop profile 2026-11 核对）"
description: "dsh-mnemon「记忆系统」在两台电脑之间同步记忆的实操记录：Git 同步的两种凭据方式、默认分支与子目录、合并/删除/冲突语义、「记忆整理」与备份历史、ZIP 迁移，以及隐私红线与常见错误提示。补充既有《本机使用与运维指南》中未覆盖的跨设备同步部分。"
status: "active"
created_at: "2026-11-08T13:13:13.814Z"
updated_at: "2026-11-08T13:13:13.814Z"
content_hash: "3ad3b7b61ac64ed7efbe6436cd2cbdb289519be6c87e1307fe3ee79a13a69f52"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-3e148fb8-0457-46ec-971b-bccbdb77b696"
memory_body_ids:
  []
---

# dsh-mnemon 跨设备记忆同步实操指南

> 面向本机 desktop profile（mnemon 0.5.42，2026-11-08 核对）。本文只回答「家里电脑和单位电脑怎么把记忆合到一起」，不重复三层记忆结构、三处界面入口、容量整理等日常内容——那些见既有文档 `91943929-4291-4ca0-8170-28e95c15cb55`（《dsh-mnemon 记忆系统本机使用与运维指南》）。该文档只写到「备份 = 复制 `.mnemon` 文件夹、设置里的同步默认关闭」，跨设备同步的完整语义在本文。

## 1. 结论

- 官方通道是设置页里的 **「Git 同步」**：把记忆数据写进一个自建 Git 仓库，另一台机器连同一仓库、同一分支/子目录后再同步。
- 同步是**合并式**，不是覆盖式：**先合并仓库里的记忆，再推送本机的**（文案键 `config.syncNever`）。
- 单机搬家/一次性迁移用 **「备份与迁移」ZIP** 更省事；长期双机才值得配 Git 同步。
- 不推荐把 `.mnemon` 目录直接放进 OneDrive/坚果云让两台同时读写：记忆每轮都在写，冲突副本难排查；插件也没为此设计。
- 同步默认关闭（`syncEnabled` 默认 `false`，见 `lib/index.js`），需要手动在设置页开启。

## 2. 前置条件

1. 系统里要有 Git；缺失时界面提示「同步需要 Git {required} 或更高版本」（`config.syncGitMissing`）。本机 2026-11-08 为 `git version 2.55.0`。
2. 一个 **私有** Git 仓库。选到公开仓库时插件会明示「{name} 是公开仓库：同步后任何人都能看到这些记忆」（`config.syncPublicBody`）——警告不等于拦截，仍要自己把关。
3. 每台机器各配一次凭据；两台尽量装同一版本 mnemon（本机 0.5.42）。
4. Host 必须支持同步通道，否则显示「当前 DSH 主机不支持 Mnemon 仓库同步通道。」（`config.syncUnavailable`）。

## 3. 配置入口与两种凭据方式

**设置 → 插件 → 可组合记忆 → mnemon 设置页 → 「Git 同步」**，说明文案为「把上述组件的数据同步到 Git 仓库；不含第三方 Provider 与密钥」（`config.syncSimpleDescription`）。

| 连接方式 | 操作 | 关键点 |
|---|---|---|
| **GitHub 账号** | 「使用 GitHub 登录」→ 设备码流程（获取设备码 → 「复制并打开 GitHub」→ 粘入确认 → 等待授权 → 「已登录 @{login}」） | 登录后可直接选择或新建仓库；**授权只保存在这台电脑**（`config.syncGitHubHint`）；可随时退出登录；凭据存储只读时会提示无法保存 |
| **访问令牌** | 选令牌方式，填仓库地址 + 访问令牌 | 推送必须有所需凭据，否则「推送需要 GitHub 登录或访问令牌」（`config.syncNoCredential`）；令牌只留本机，**不随同步外传** |

常见失败与含义（`client.js` 文案键）：

- `syncIssueAuthHttps`：需要凭据，去登录或填令牌。
- `syncIssueRejectedGitHub` / `syncIssueRejectedGitHubToken`：GitHub 拒绝登录态 / 拒绝保存的令牌 → 重新登录或换令牌。
- `syncIssueRejectedGitHubEnvironment`：GitHub 拒绝了 `{env}` 环境变量里的令牌 → 换掉它。
- `syncIssueNotFoundGitHub`：仓库地址找不到；私有仓库要先有权限。
- 断开连接（`config.syncDisconnectBody`）：之后不再同步、自动同步关闭，**本机记忆和仓库内容都不删**，保存的访问令牌一并移除。

## 4. 默认写入位置（一般不用改）

- 分支：`mnemon-sync`；仓库内子目录：`mnemon/`；作者名：`dsh-mnemon sync`（常量定义于 `lib/index.js`）。
- 配置落在数据目录的 `sync-git.json`（`MNEMON_SYNC_CONFIG_FILE`）；令牌可用环境变量 `MNEMON_SYNC_GIT_TOKEN` 提供。
- 「高级设置」可改分支名、子目录、机器名称；机器名默认取主机名（本机 `ZY-HOME`），备份历史里显示为「来自 {name}」（`config.syncMachineHostnameNote`）。改机器名只影响之后的同步，已有版本保留原名。
- 自动同步间隔上限 7 天（`MNEMON_SYNC_MAX_AUTO_BACKUP_MINUTES = 10080`）；关闭自动同步时，只在点「立即同步」时才同步（`config.syncAutoHintOff`）。

## 5. 第一次连通的操作顺序（两台机器各做一遍）

1. **连接**：成功后状态显示「已连接 · 首次同步会创建 {branch} 分支」（首次同步会建 `mnemon-sync` 分支）。
2. **先检查/对比，别急着同步**：看三种提示——「本机与远端一致，不需要同步」/「远端记录了 N 条删除，同步时会在本机应用」/「N 个主题两边写法不同：同步会保留两种写法」。
3. **点「立即同步」**：顺序是先合并远端、再推送本机，所以新装的那台（空数据）不会覆盖已有的那台。
4. **到仓库网页确认**：能看到 `mnemon-sync` 分支和 `mnemon/` 目录即成功；成功提示为「已同步：合并了 {count} 条远端记忆，推送了 {commit}」（`config.syncDoneMerged`）。
5. **再决定自动同步**：跑顺一轮后再开「按间隔后台同步」；自动同步失败会在设置页显示上次失败原因（`config.syncAutoFailed`）。

## 6. 合并语义与冲突处理（最该放心的部分）

| 情形 | 界面提示 | 结果 |
|---|---|---|
| 同一主题两边写法不同 | 「{count} 个主题两边写法不同：同步会保留两种写法」 | **不二选一、不丢数据**；之后用「记忆整理」（`config.reconcileTitle`）合并——模型读合并后的记忆，**只提建议，等你审查后才写入** |
| 一台机器删过某条 | 「远端记录了 {count} 条删除，同步时会在本机应用」 | 删除会传播，另一端跟着删 |
| 远端有本机以前删掉的条目 | 「远端有 {count} 条记忆是本机以前删掉的，同步不会把它们加回」 | 不会“复活”你已删除的内容 |
| 两台同时同步 | 「另一个同步操作还在进行」/「仓库又被其他设备更新了，请再同步一次」 | 隔几分钟再点一次即可，不是损坏 |
| 完全一致 | 「本机与远端一致，不需要同步」 | 无需操作 |

补充：默认「同步时先合并仓库里的记忆，再推送本机的」（`config.syncNever`），这是它比“手工拷文件夹/云盘”安全的核心原因。

## 7. 备份历史与回滚线索

每次同步在仓库里留一个版本（`config.syncHistory`，标题「备份历史」），条目带机器名与时间（「来自 {name}」）。出问题时可按仓库 commit 历史回看；「备份历史」为空表示仓库里还没有备份（`config.syncHistoryEmpty`）。

## 8. ZIP 备份与迁移（一次性/应急）

- 同一设置页的「备份与迁移」提供 **导出 ZIP / 导入 ZIP**（`config.packImportZip` / `config.packExportZip`），ZIP 含「上述组件的数据，不含第三方 Provider 与密钥」。
- 导入是**安全合并到数据目录**（「已将 ZIP 安全合并到 {root}」），不是覆盖；Host 不支持时提示「当前 DSH 主机不支持 Mnemon ZIP 备份通道。」
- 适合作一次性搬家或应急；长期双机仍走 Git 同步。

## 9. 隐私与合规红线

- 记忆里可能含单位内部信息、人名职务、审查意见等。**只用私有仓库**；有保密要求时优先单位内网 Git，不上公网托管。
- 令牌不要贴在聊天、文档或仓库里：填在界面输入框，或用 `MNEMON_SYNC_GIT_TOKEN` 环境变量。
- 公开仓库的警告只是提示，插件不会替你拦截。

## 10. 证据来源

- `C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon\lib\index.js`：同步常量（`MNEMON_SYNC_GIT_TOKEN`、`MNEMON_SYNC_DEFAULT_BRANCH = "mnemon-sync"`、`MNEMON_SYNC_DEFAULT_SUBDIR = "mnemon/"`、`MNEMON_SYNC_CONFIG_FILE = "sync-git.json"`、`MNEMON_SYNC_MAX_AUTO_BACKUP_MINUTES = 10080`）、`syncEnabled` 默认 `false`、分支名校验（`syncBranchValid`）。
- `C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon\lib\client.js`：界面文案键 `config.sync*`、`config.pack*`、`config.reconcile*`（约 2894–3093 行；英文对照约 3720–3919 行）。
- `C:\Users\zhouy\.dsh\profiles\desktop\node_modules\dsh-mnemon\cordis.patch.yml`：Starter 默认配置（`writebackMode: guided`、`recallMode: guided`、`idleReviewMs: 30000`、`remoteAccess: read-only`）与 `mnemon-source-*` 组件分层。
- 相关文档：`91943929-4291-4ca0-8170-28e95c15cb55`（本机使用与运维指南）、`af69aa1c-1889-42dc-8498-ea728419c830`（OpenViking 共存分析）。
