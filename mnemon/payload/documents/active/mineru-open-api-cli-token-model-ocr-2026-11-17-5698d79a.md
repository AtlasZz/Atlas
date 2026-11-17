---
id: "5698d79a-5e06-4c04-bc92-0a3f5a5d9e08"
title: "mineru-open-api CLI 实操口径：安装、token 鉴权与 model/OCR 默认行为（2026-11-17 实测）"
description: "mineru-open-api v0.5.9 在本机安装与 token 配置的落地结果，以及 flash-extract / extract 两种模式下模型与 OCR 参数的真实默认值（含实际请求体证据）和按材料性质选参的对照表。"
status: "active"
created_at: "2026-11-17T03:20:47.767Z"
updated_at: "2026-11-17T03:20:47.767Z"
content_hash: "4b188ffd5a8ed9e6cbf38127b479bb6815340704392eecdbd85cabe91a8bf9ec"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-0a716ff5-658b-46c8-8e0c-34be30d88d87"
memory_body_ids:
  []
---

# mineru-open-api CLI 实操口径（v0.5.9，2026-11-17 本机实测）

本机全局装的是 `mineru-open-api`（npm 分发的 Go 单文件 CLI，`C:\Users\zhouy\AppData\Roaming\npm\mineru-open-api`）。免 token 的 `flash-extract` 与需 token 的 `extract` 均已实测跑通；token 存于 `C:\Users\zhouy\.mineru\config.yaml`（`Token source: config`）。本文只记录**可复用结论**，不含 token 值本身。

## 一、两种模式的硬边界

| | flash-extract | extract（需 token） |
|---|---|---|
| 文件上限 | 10 MB / 20 页 | 200 MB / 600 页 |
| 输入 | pdf、图片、docx、pptx、xlsx | 追加 doc、ppt、html、URL |
| 输出 | 仅 md | `-f md,json,html,latex,docx`（可多选） |
| 模型 | 服务端固定，无 `--model` / `--ocr` 选项 | 可选 `vlm` / `pipeline` / `html` |
| 其他 | 仅 `-o`、`--language`、`--pages` | `--ocr`、`--pages`、批量、`--list`、资源保留 |

## 二、`--model auto` 实际就是 `vlm`

用 `-v` 抓取实际请求体确认，CLI **不按文档类型智能挑选**，只按输入类型固定映射：

| 输入 | 实际发出的 `model_version` |
|---|---|
| PDF（电子版同样） | `vlm` |
| 图片（png/jpg） | `vlm` |
| HTML | `MinerU-HTML` |
| 显式 `--model pipeline` | `pipeline` |

实测请求体：`{"files":[{"name":"text.pdf"}],"model_version":"vlm"}`。

含义：默认路径是精度更高但**极少数情况会改写措辞**的 VLM。法规条文、审查意见、指标表这类不容许改字的材料，默认跑出的结果不能直接当原文引用，须回原件核对，或显式 `--model pipeline`。

## 三、OCR 默认关闭，且不会自动识别扫描件

不传 `--ocr` 时请求体里根本没有 `is_ocr` 字段，服务端按 false 处理。扫描版 PDF 不报错但会返回空壳/乱码——它不会自行判断“图片型 PDF 需要 OCR”。显式传时才发：

```
{"files":[{"is_ocr":true,"name":"text.pdf"}],"language":"en","model_version":"pipeline"}
```

注意 `--ocr` 会自动把模型切到 `pipeline`。

## 四、其余参数的发送条件

| 参数 | 默认 | 何时出现在请求体 |
|---|---|---|
| `--table` | 开 | 仅 `--table=false` 时发 `enable_table:false` |
| `--formula` | 开 | 仅 `--formula=false` 时发 `enable_formula:false` |
| `--language` | `ch` | 仅非默认时发送 |
| `--pages` | 全部 | CLI 本地裁页后再上传（可用它控制额度/时长） |

flash-extract 的请求体只有 `{"file_name": "...", "language":"ch"}`，无模型字段，因此“免 token 模式”是没有选择权的一种固定档位。

## 五、鉴权与排障要点

- `mineru-open-api auth` 支持从 stdin 读 token（`printf '%s\n' "$TOKEN" | mineru-open-api auth`），无需人工交互；写入 `~/.mineru/config.yaml` 在工作区外，DSH 沙箱内需 danger-full-access，否则报 `create config dir: Access is denied`。
- `auth --verify` **只校验 token 格式**，不代表可用；真实可用性必须跑一次 `extract` 验证（本次即用自建 1 页 PDF 验证上传→解析→下载全链路）。
- 文件无论哪种模式都会上传至 mineru.net/aliyun OSS 解析；涉密或未公开的规划/审查材料不宜走此链路，可改用本地 `docx2md` 或直接读 PDF 文本层。
- 纯电子版 PDF 不要加 `--ocr`：变慢且可能在数字页码、表格边框上引入噪声。

## 六、按材料性质的选参对照

| 材料性质 | 参数 |
|---|---|
| 电子版规划文本、审查意见、需逐字准确 | `--model pipeline` |
| 复杂版式、图表跨页、图文混排 | 默认（vlm） |
| 扫描件、盖章页、手写批注 | `--ocr`（自动 pipeline）或 `--ocr --model vlm` |
| 纯英文/日文文献 | `--language en` / `--language japan` |
| 几百页大部头 | `--pages 1-100` 分段 |
| 只取部分页、省额度 | `--pages 1-30` |

## 七、技能侧现状

- `mineru-document-extractor` 技能的 `SKILL.md` 已加 `disable-model-invocation: true`（第 12 行），因此不再进入模型技能清单，只能由用户 `/` 手动唤起。原因是该技能是唯一会把文件上传外部服务器的能力，且原 description 是极端关键词堆砌、误触发率高。
- 不要把 `user-invocable` 设为 false，否则用户自己也无法唤起。
- 该 SKILL.md 的 “Agent rules” 只写了“小文件用 flash-extract、要格式/批量用 extract”，**没有任何 model/OCR 选择指引**，策略层为空，实际选参只能靠调用者按上表指定。

相关文档：`2f980ddd-cafd-46c6-b380-cbfd7a37fa98`（SKILL.md frontmatter 名称校验与调用策略字段，含该技能修复经过）、`3f49f70e-ac62-4abc-9bd9-03006d4355eb`（dsh-pdf-mineru 插件的 Pipeline/VLM 双引擎与 OCR 口径，可与本 CLI 结论互参）。
