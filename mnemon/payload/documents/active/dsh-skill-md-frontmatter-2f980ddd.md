---
id: "2f980ddd-cafd-46c6-b380-cbfd7a37fa98"
title: "DSH 技能被静默丢弃的排查：SKILL.md frontmatter 名称校验与调用策略字段"
description: "DSH 技能加载器的名称/调用策略 frontmatter 校验规则（含 SKILL_NAME 正则）、非法时只记 warning 并静默忽略的表现，以及 MinerU 技能文件修复与依赖缺失的现状。"
status: "active"
created_at: "2026-11-17T02:31:00.185Z"
updated_at: "2026-11-17T02:31:00.185Z"
content_hash: "7ab9d3220d01e7382d902b12077833265248f3d373244ba62e4d5bdf716fcc9c"
source_paths:
  - "tmp_asar/dsh/node_modules/@deepseek-ai/dsh-skill/lib/index.js"
  - "tmp_asar/dsh/node_modules/@deepseek-ai/dsh-skill-filesystem/lib/index.js"
session_ids:
  - "mnemon-run-v1-540af155-d112-47af-bddb-2df24388cf55"
memory_body_ids:
  []
---

# DSH 技能「装了却不出现在技能清单」的根因口径（2026-11-17 实测）

一个从技能市场导入的技能目录完整（`SKILL.md`、`_meta.json` 都在），但会话技能清单里看不到，`skill` 工具与 `/` 调用都找不到。原因是 frontmatter 校验不通过，加载器**只写一条 warn 日志后丢弃该技能**，没有界面提示。

## 校验规则与证据位置

- 名称语法：`@deepseek-ai/dsh-skill` 的 `SKILL_NAME = /^[a-z0-9]+(?:-[a-z0-9]+)*$/`（`lib/index.js` 约第 18 行，`isSkillName` 在第 30 行）。即 **kebab-case**：小写字母/数字、单连字符分段，不允许大写、空格、下划线、连续或首尾连字符。
- 文件系统 provider：`@deepseek-ai/dsh-skill-filesystem`（本次实例在打包目录 `tmp_asar\dsh\node_modules\...\lib\index.js`）。解析 frontmatter 后依次检查：
  - `name`、`description` 必须为非空字符串，否则 `ignored: frontmatter requires name and description`；
  - `isSkillName(name)` 失败则 `ignored: invalid skill name "<name>"` —— 这条是本次问题的实际原因；
  - `parseInvocationPolicy` 抛错则 `ignored: invalid invocation frontmatter: ...`；
  - YAML 无法解析或缺失 frontmatter 也各自只记 warn 后 return。
- 识别形态：目录包为 `<root>/<skill>/SKILL.md`，扁平技能为 `<root>/*.md`；用户根目录即 `%USERPROFILE%\.dsh\skills`，改动后**热加载生效**，无需重启或新建会话。

## 调用策略 frontmatter（易踩的兼容性坑）

- 规范字段只有两个：`disable-model-invocation`（默认 false）与 `user-invocable`（默认 true）。
- 旧键 `disableModelInvocation` / `modelInvocable` / `userInvocable` 会被显式拒绝并报 `frontmatter field "<legacy>" is unsupported; use "<canonical>"`，导致整条技能被丢弃。
- 布尔值除 true/false 外还接受 `1`/`0` 与字符串 `true/false/yes/no/on/off`（大小写不敏感），其他值抛 `must be a boolean`。

## 本次处置

- 受影响文件：`C:\Users\zhouy\.dsh\skills\miner-u-document-extractor\SKILL.md`，原第 2 行 `name: MinerU Document Extractor`（含大写与空格）→ 改为 `name: mineru-document-extractor`，保留原文件不另建副本；改后清单中出现 `mineru-document-extractor`。
- 同目录其余技能名称经核对均合法，无第二个被静默丢弃者。
- 教训：从市场导入的技能包可能自带不合法 `name`，重装会复发；排查「技能不可用」应先看 host 日志的 `skill file ... ignored` 警告，再看 frontmatter，而不是怀疑安装或缓存。

## 该技能的遗留依赖状态（2026-11-17）

- `mineru-document-extractor` 只是外壳，实际转换依赖外部 CLI `mineru-open-api`（`flash-extract` 免 token，限 10 MB / 20 页；`extract` 需 token）。当时本机 `where mineru-open-api` 无结果、`~/.mineru/config.yaml` 不存在，且沙箱内 `registry.npmjs.org`、`mineru.net`、`baidu.com` 均返回 `000`，故未安装。
- 隐私提示：MinerU 云端模式会把文件上传到 mineru.net 解析；公文/规划类敏感材料应改用本地方式（如 `docx2md`、文本层提取），或先确认可外传。

相关文档：`3f49f70e-ac62-4abc-9bd9-03006d4355eb`（dsh-pdf-mineru 插件的 Pipeline/VLM 与 OCR 口径，可作为 MinerU 本地/插件替代路径参考）。
