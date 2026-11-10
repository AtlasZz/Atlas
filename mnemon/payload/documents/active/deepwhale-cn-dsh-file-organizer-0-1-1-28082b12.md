---
id: "28082b12-25bf-4f1a-842d-ad7510fa3b1f"
title: "「深鲸文件整理」@deepwhale-cn/dsh-file-organizer 插件调研（0.1.1 源码核对）"
description: "npm 上定位的 DSH 文件整理插件：功能、CLI/面板/API 用法、源码结构、安装兼容性，以及 plan 的 duplicates 伪字段、dups 退出码、apply 非幂等风险等已核对问题。"
status: "active"
created_at: "2026-11-10T03:41:36.968Z"
updated_at: "2026-11-10T03:41:36.968Z"
content_hash: "b1305070e854f48f3f006958539c90bd8d10b1c8623a0e9c5b04d7afe7c7d7cf"
source_paths:
  - "pkg/unpacked/package/lib/organize.mjs"
  - "pkg/unpacked/package/bin/cli.mjs"
  - "pkg/unpacked/package/lib/api.js"
  - "pkg/unpacked/package/lib/index.js"
  - "pkg/unpacked/package/package.json"
  - "pkg/unpacked/package/README.md"
  - "pkg/unpacked/package/skill/file-organizer.md"
  - "fetch_pkg.js"
  - "check_npm.js"
  - "find_plugin.py"
session_ids:
  - "mnemon-run-v1-33f4496d-a5f1-4619-8582-fddae5134f1c"
memory_body_ids:
  []
---

# 深鲸文件整理（@deepwhale-cn/dsh-file-organizer）调研

> 2026-11-10 核对。依据 npm registry 元数据与 0.1.1 tarball 解包后的源码逐文件阅读；调研完成时尚未安装到本机 profile，以下行为均针对 0.1.1。

## 1. 身份与获取

- npm 包：`@deepwhale-cn/dsh-file-organizer`，中文名「深鲸文件整理」，latest 0.1.1，最近发布/更新 2026-10-29，registry 共 3 个版本。
- 不在 DSH 插件市场目录（awesome-dsh-plugin.com，抓取时 9714 条）中。按 organizer 搜到的 10 个条目主要是侧栏、会话、设置或桌面快捷方式整理器，均不是通用文件归档工具；此包是从 npm 精确检索定位的。
- 安装：`dsh plugin --profile desktop add @deepwhale-cn/dsh-file-organizer`，装完重启 DSH 生效。
- 兼容：`engines.dsh` 为 `>=0.1.0-rc.6 <0.3.0`（本机 host 0.2.2-rc.1 满足），Node >=18；依赖 `@deepwhale-cn/dsh-guard-core ^0.10.1`，peer 依赖 `@deepseek-ai/dsh-skill *`。
- 元数据：`main` 指向 `lib/index.js`；bin `dsh-file-organizer` 指向 `bin/cli.mjs`；`dsh.bundle.patch` 为 `cordis.patch.yml`；`dsh.client` 声明 `platform: web`、`immediately: true`；tarball 共 10 个文件、解包约 48 KB。

## 2. 功能与安全默认

| 动作 | 行为 |
|---|---|
| `plan` | 只出整理计划（from → to），不写入磁盘；返回 total、moves、conflictTargets、emptyFiles 等 |
| `apply` | 按「类型/年月」建子目录并移动文件；`--by date` 时只按年月；目标同名自动加 `-2`、`-3`，不覆盖 |
| `dups` | 查重：先按大小分桶，再比对 sha256，返回重复组与「可回收字节」 |
| `clean` | 列出空目录与 0 字节文件；默认只报告，`--delete` 才删空目录，`--delete --empty-files` 才删空文件 |

分类规则硬编码在 `lib/organize.mjs` 的 `GROUPS`：doc（docx/doc/pdf/md/txt/rtf/odt/pages）、sheet（xlsx/xls/csv/numbers/ods）、slide（pptx/ppt/key/odp）、image（png/jpg/jpeg/gif/webp/heic/svg/bmp/tif/tiff）、video（mp4/mov/mkv/avi/webm）、audio（mp3/wav/m4a/flac/aac）、archive（zip/rar/7z/tar/gz）、code（js/ts/py/java/go/rs/sh/json/yml/yaml/html/css），其余归 other。

安全默认（源码确认，不是宣传语）：

- 只有 `apply: true` 才移动，只有 `del: true` 才删除；默认全程只读。
- 跳过 `node_modules`、`.git`、`.svn`、`__pycache__`、`.DS_Store`，并跳过所有以 `.` 开头的隐藏项。
- 移动时目标已存在则追加序号，绝不覆盖。
- 全部本机完成，不上传文件。

## 3. 用法

**CLI**

```bash
dsh-file-organizer --plan  <目录>                         # 只看计划
dsh-file-organizer --apply <目录>                         # 执行：doc/2026-10/xxx.docx
dsh-file-organizer --apply <目录> --by date               # 只按年月
dsh-file-organizer --dups  <目录>                         # 查重
dsh-file-organizer --clean <目录>                         # 只报告空目录/空文件
dsh-file-organizer --clean <目录> --delete --empty-files  # 真正删除
```

加 `--json` 输出结构化结果；`--help` 显示用法。默认 `--by type`。

**侧栏面板**：README 说明侧栏有「深鲸文件整理」入口，填目录、选按类型/按年月，先看计划再执行。宿主 API 注册在 `/dsh-file-organizer`：`GET /api/info` 返回版本，`POST /api/run` 接收 `{root, by, mode, del, emptyFiles}`，`mode` 为 `plan | apply | dups | clean`。

**Agent skill**：包内 `skill/file-organizer.md` 声明了触发场景（“把下载目录按类型整理一下”“这些文件有没有重复的”“空文件夹清一清”），安装后可用自然语言调用 CLI。

## 4. 源码结构与实现要点

| 路径 | 作用 |
|---|---|
| `bin/cli.mjs` | 命令行入口：解析 `--plan/--apply/--dups/--clean/--by/--json`，按结果设置退出码 |
| `lib/organize.mjs` | 内核：`plan`、`organize`、`clean`、`dups`、分类与遍历 |
| `lib/api.js` | 宿主 HTTP 出口：注册 `/dsh-file-organizer` 路由并调用内核 |
| `lib/index.js` | 插件入口：`createPlugin` 注册 skill 与 bin，注入 `webServer` 后注册 API |
| `lib/client.js` | Web 客户端面板（约 32.6 KB，本次未逐行核对） |
| `skill/file-organizer.md` | Agent skill 说明与使用规则 |
| `cordis.patch.yml`、`package.json`、`README.md`、`LICENSE` | 打包/描述文件 |

实现细节：

- `plan` 递归遍历后，目标为 `组名/YYYY-MM/文件名`（`--by date` 时为 `YYYY-MM/文件名`），年月取自文件 mtime；同时统计目标路径冲突。
- `organize` 对每个 move 计算目标；若 `existsSync(dest)` 或本轮已用过该目标，则循环追加 `-2`、`-3`；`mkdirSync(..., {recursive:true})` 后 `renameSync`，失败则退化为复制+删除，仍失败则静默跳过。
- `clean` 自底向上收集空目录（子目录处理完后再判断），空文件单列；删除受 `del` 与 `emptyFiles` 双重开关控制。
- `dups` 跳过 0 字节和单文件桶，按大小分组后对同大小文件算 sha256，只返回多于 1 个路径的组。

## 5. 已核对的问题与注意点

1. **`plan` 返回的 `duplicates` 不是查重结果。** 它与 `emptyFiles` 使用同一个条件 `m.size === 0`，实际列的是 0 字节文件；真正的重复检测只有 `--dups`。若面板或脚本依赖 `plan.duplicates`，会被误导。
2. **`--dups` 发现重复时退出码是 1**（`process.exit(r.groups.length ? 1 : 0)`），0 表示没有重复。脚本或 CI 中不要把 1 当成命令执行失败。
3. **`apply` 非幂等（源码推断，未实机验证）。** 对已经处于 `类型/年月` 目标位置的文件，`plan` 仍会列入 moves，且 `from == to`；`organize` 中 `existsSync(dest)` 命中文件自身，进入加序号分支，会把 `a.docx` 改成 `a-2.docx`。因此同一目录不要反复执行 `--apply`；再次整理前先 `--plan` 核对，或等上游修复。
4. **移动失败被静默跳过。** 单个文件移动/复制失败只被 catch 忽略，最终只能对比返回的 `moved` 与计划 `total` 判断是否有遗漏。
5. **删除必须显式双开关。** `clean` 默认只报告；空目录要 `--delete`，空文件还要再加 `--empty-files`。

## 6. 建议使用流程

1. 先装到测试 profile 或先在临时目录试用，重启 DSH 后确认侧栏面板/CLI 可用。
2. 对真实目录先 `--plan`，人工核对 from → to；注意 `duplicates` 一栏不可信。
3. 确认后只执行一次 `--apply`；需要查重时单独跑 `--dups`。
4. 清理先不带 `--delete` 看报告，确认后再加删除开关。
5. 写脚本时按 `--dups` 的退出码语义处理“有重复”，并检查 `moved` 与 `total` 的差异。

## 7. 证据来源

- 解包源码：`pkg/unpacked/package/` 下的 `lib/organize.mjs`、`bin/cli.mjs`、`lib/api.js`、`lib/index.js`、`package.json`、`README.md`、`skill/file-organizer.md`。
- registry 元数据与 tarball：由工作区脚本 `fetch_pkg.js`、`check_npm.js` 获取；市场目录检索由 `find_plugin.py` 完成。
