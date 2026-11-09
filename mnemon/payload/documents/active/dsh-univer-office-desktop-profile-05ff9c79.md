---
id: "05ff9c79-b7b2-470a-aed4-f615c77654f8"
title: "dsh-univer-office 使用指南（本机 desktop profile 验证）"
description: "dsh-univer-office 0.4.0 在本机 DSH desktop profile 的安装位置、14 个 univer_* 工具、工作流（草稿 worktree→审阅→确认/导出）、已完成端到端冒烟验证的结论与已知边界。"
status: "active"
created_at: "2026-11-03T07:49:21.338Z"
updated_at: "2026-11-03T07:49:21.338Z"
content_hash: "5578bcdaa7d8d120ac3310f8b3598d74489ec2d8aad90e3f0c464dc7bb7f4093"
source_paths:
  []
session_ids:
  - "32ff6b58-e170-488d-b7b7-9244b7b67297"
memory_body_ids:
  []
---

# dsh-univer-office 使用指南（本机 desktop profile 验证）

> 结论性记录：本机已安装并可用 `dsh-univer-office` 0.4.0，无需额外安装或启动命令即可用自然语言驱动电子表格 / 文档 / 幻灯片 / 多维表格 / 画布。以下为 2026-11 在本机的实际核验结果。

## 1. 安装现状（与官方 README 的关键差异）

- 插件位于 `C:\Users\ZY-WORK\.dsh\profiles\desktop\node_modules\dsh-univer-office`，版本 `0.4.0`。
- 由 `C:\Users\ZY-WORK\.dsh\profiles\desktop\package.json` 的 `dsh.profile.bundles` 列出并加载，同时 `cordis.patch.yml` 以 id `univer` 插入该服务。
- **差异**：官方 README 的安装/卸载命令写的是 `--profile web`；本机实际 profile 是 `desktop`，重装或升级须用：
  - 安装：`dsh plugin --profile desktop add dsh-univer-office`
  - 卸载：`dsh plugin --profile desktop remove dsh-univer-office`
- 改动插件集合后需重启 DSH 进程并刷新浏览器页面（运行中安装不会热加载）。

## 2. 工作流（用户视角）

1. 用自然语言描述目标，可附上待处理的 `.xlsx` / `.docx` / `.pptx`。
2. Agent 先建**隔离草稿（worktree）**，在草稿中编辑，不触碰当前已确认版本。
3. 编辑过程写入实时浮窗（可拖动、缩放、全屏）；文件树中点击 `.univer` 可随时预览。
4. 完成后会话内留下审阅卡片，可「查看 / 对比 / 全屏」，按 Esc 退出全屏。
5. **确认或放弃必须由用户明确提出**，Agent 不得自行确认。
6. 需要交付时说「导出 xlsx / pptx / docx」；草稿与审阅预览内不开放导入导出，导出应在确认后的正式版本上进行。

`univer_*` 工具由 DSH 自动选择，日常不必手动点名；只有排障或精确控制时才显式要求（如「先跑 lint」「截图第 3 页」「打印成 PDF」）。

## 3. 内置工具（14 个）

`univer_new`（新建，不覆盖）、`univer_status`、`univer_worktree`（创建/提交/继续修改/确认/放弃草稿）、`univer_unit`（增删 Sheet/Doc/Slide/Base/Board）、`univer_import`、`univer_inspect`、`univer_execute`、`univer_export`、`univer_lint`（Slide 越界/溢出/重叠）、`univer_compile_svg`、`univer_screenshot`、`univer_print_pdf`、`univer_api`（按关键词查 API 符号）、`univer_resources`（内置图标/Logo/Emoji/插画）。

配套技能：`univer`、`univer-sheet`、`univer-doc`、`univer-slide`、`univer-base`、`univer-board`、`univer-embed`、`univer-cross-unit-formula`、`office`、`office-docx`、`office-xlsx`、`office-pptx`（启用内置技能时后者会替换 DSH 原生 python-docx/openpyxl/python-pptx 路线）。

## 4. 已完成的端到端冒烟验证

在工作区 `D:\Workspace\DSHarness` 下实测通过，链路为：

`univer_new` 建 `smoke-test.univer` → `univer_worktree` 开草稿（draft）→ `univer_unit` 建 Sheet unit → `univer_execute` 写单元格（返回 `committed: true`, `revision: 2`）→ `univer_export` 导出 `smoke-test.xlsx`（6369 字节）→ `univer_screenshot` 得 PNG（352×96, A1:B2）。

要点与坑：

- 渲染类能力（截图 / PDF 打印 / Slide 版式检查 / SVG 文字度量）自动探测到本机 **Microsoft Edge**（`C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`），说明该链路可用，无需另配 `browserExecutablePath`。
- `univer_screenshot` 的 range 需按工具期望格式传入；传 `Sheet1!A1:B2` 会报 `[SCREENSHOT_INVALID_INPUT]: invalid A1 range`，改为不带表名前缀的 `A1:B2` 即成功。
- Gateway 默认端口 9080，被占用时依次试 9081/9082；当次三个端口均未监听，属正常（首次访问才自动拉起本地服务）。
- 清理测试产物时 `.univer` 文件曾被占用进程锁住导致 `Remove-Item` 失败；说明 Gateway/Viewer 可能持有文件句柄，删除前应先关闭相关预览或等待 runtime 空闲。

## 5. 配置与开关

- 侧边栏「插件 → dsh-univer-office」详情页：**自动弹出实时窗口**、**在会话中显示审阅卡片**（关掉后会话更紧凑，文件树预览仍可用）。
- 常用配置项：`gatewayPort`（默认 9080）、`autoStartGateway`、`browserExecutablePath`、`tools`（默认 true）、`skills`（设 false 可保留 DSH 原生 office 技能）、`telemetry`（设 false 或 `DO_NOT_TRACK=1` 关闭）。
- Viewer 与 API 直接挂在 DSH WebServer origin（`/univer-viewer/...`、`/uf/...`），远程部署沿用既有访问入口，无需额外反向代理；Gateway 仅监听 loopback。

## 6. 已知边界（避免期待落空）

- Slide 的母版、版式页、演讲者备注不在编辑范围。
- Board 不支持文件导出（仅可打印）；Board 的思维导图、墨迹、高级编辑未开放。
- 旧版 `.univer` 打开会自动升级到 v3：先在原文件旁备份，在临时副本中迁移验证成功后才替换；升级失败原文件不变。
- 安装 `dsh-better-sidebar` 时，`.univer` 文件树预览改为注册为该插件的文件预览器（本机即此形态）。
- 只安装官方包名 `dsh-univer-office`；`dsh-univer-plugin`、`dsh-univer-office-suite`、`univer-office` 等为防仿冒的 deprecated 占位包。

## 参考（本机路径，位于 DSH 主目录而非工作区）

- 插件 README（中文）：`C:\Users\ZY-WORK\.dsh\profiles\desktop\node_modules\dsh-univer-office\README.zh-CN.md`
- 架构：`.../dsh-univer-office/docs/architecture.md`、`docs/univer-turn-projection-and-surfaces.md`、`docs/viewer-same-origin-deployment.md`
- 技能细则：`.../dsh-univer-office/skills/**/SKILL.md`
