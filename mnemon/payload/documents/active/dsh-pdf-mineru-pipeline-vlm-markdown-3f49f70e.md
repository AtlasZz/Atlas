---
id: "3f49f70e-ac62-4abc-9bd9-03006d4355eb"
title: "dsh-pdf-mineru 插件能力口径：Pipeline/VLM 双引擎、识别方式与 Markdown 产物导出"
description: "dsh-pdf-mineru 0.2.3 的两轴模型（流水线/VLM）、识别方式（自动/文本层/OCR）在各 Provider 上的可用性、请求参数落点、工具限制，以及从本地缓存 full.md 导出 Markdown 的操作路径。"
status: "active"
created_at: "2026-11-09T09:22:19.569Z"
updated_at: "2026-11-09T09:22:19.569Z"
content_hash: "3b6977846308f1aeec05d5c3c4c61c711438460bd184b85d18711eef2ad5201b"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-b4a6f828-1506-48bd-a03a-ca3fc4708233"
memory_body_ids:
  []
---

# dsh-pdf-mineru 插件能力口径（2026-11 核查，v0.2.3）

围绕「Pipeline 与 VLM 有何区别」「自动与 OCR 有何区别」「能否当 Markdown 转换器用」三个问题，对已安装插件做了源码与配置核查。证据来自本机安装包与 profile 配置，但结论的**内核**是 MinerU 服务端语义，插件版本升级后参数名可能变化，动手前宜按文中行号复查一次。

## 证据位置

- 配置层：`C:\Users\ZY-WORK\.dsh\profiles\desktop\cordis.patch.yml`（`- id: dsh-pdf-mineru`，`schemaVersion: 3`）；同目录 `cordis.yml` 仅为空壳，真正的组合在 bundle + patch 层。
- 实现层：`C:\Users\ZY-WORK\.dsh\profiles\desktop\node_modules\dsh-pdf-mineru\lib\index.js`（adapter 与工件落盘）、`lib\client.js`（UI 文案）。
- 关键位置：`index.js` 约 1690–1757（self-hosted legacy v2 提交表单）、3876–3924（self-hosted v1 提交 payload）、4300–4320（官方云提交，`is_ocr` / `model_version`）、9300–9360（工件写盘与默认文件名）；`client.js` 约 2530（`model.pipeline` / `model.vlm` / `field.tier` 文案）。

## 两个轴是正交的，不是精度档位

1. **模型轴**＝用哪种解析引擎。
   - Pipeline：版面检测 → 公式检测/识别 → 表格识别 → OCR，多阶段小模型串联；快、显存低、便宜，页眉页脚/页码规则化处理不"编"，规整公文与单栏论文表现稳。
   - VLM：整页当图片喂多模态大模型，端到端直出 Markdown；复杂图文混排、多栏、密集公式、嵌套表格、低质扫描更强，但慢一个量级、算力贵，且可能漏行或幻觉补字。
   - 官方 README 口径相同：Pipeline 快、资源低、适合常规版面；VLM 对复杂图文混排与特殊公式更好。
2. **识别方式轴**＝这一页的字符从哪来。
   - 自动（auto，默认）：服务端判断，有文本层直接取，取不到（图片页/扫描页）才 OCR。电子版 PDF 一律选它，最快且无识别误差。
   - 文本层（txt）：强制只抽内嵌文字，绝不 OCR，最保真，但纯扫描件几乎读不出内容。
   - OCR（ocr）：强制图像识别，不论有无文本层。适用：扫描/拍照件；文本层是乱码（缺字体、CID 字体、方正宋体嵌入不全——国内公文 PDF 常见）；需要按版面重排而非按原文字流。

## Provider 能力矩阵（决定 UI 上出现哪个开关）

| Provider 类型 | 模型轴 | 识别方式 | 备注 |
|---|---|---|---|
| 官方云 v4（`official-v4`，https://mineru.net/api/v4） | 可选 `pipeline`/`vlm` | 自动 / OCR（**无文本层**） | 语言/公式/表格开关实际不生效，服务端决定 |
| 自部署 3.x（`self-hosted-legacy-v2`，/tasks） | 可选，落地为表单 `backend`（由 `modelMap` 映射，如 pipeline→pipeline、vlm→vlm-engine） | 自动 / 文本层 / OCR（`parse_method`） | 语言/公式/表格生效；只支持单段连续页码范围（`start_page_id`/`end_page_id` 为 0 基） |
| 自部署 4.x（`self-hosted-v1`，/v1/parse/jobs） | 不可选；同位置改为「解析档位」`tier`（flash/basic/standard/advanced） | 自动 / 文本层 / OCR（`ocr_mode`） | 服务端决定引擎；提交时 `output_formats: ["zip"]` |

因此设置页在 4.x 把「模型」换成了「解析档位」——同一层级的不同表现形式，二者不并存。

## 请求参数落点（便于日后核对/排障）

- 官方云：payload `is_ocr`、`model_version`，另带 `page_range`。
- self-hosted v1：`ocr_mode`、`tier`、`files[].page_range`、`output_formats:["zip"]`。
- self-hosted legacy v2：multipart 表单 `files`、`backend`、`parse_method`、`lang_list`、`formula_enable`、`table_enable`、`return_md` / `return_middle_json` / `return_model_output` / `return_content_list` / `return_images`（按 `requiredArtifacts` 集合裁剪），未就绪服务返回健康状态异常即抛 `PROVIDER_UNAVAILABLE`。

## 工具面的硬约束

- `read_pdf` 与 `async_parse_pdf` **都不接受** `ocr`、`model` 之类的临时参数；要换引擎或识别方式只能在 Plugins → dsh-pdf-mineru →「服务与解析」里改并保存。Agent 无法对单个文件"这次用 OCR"。
- 改解析配置会改变缓存身份，同一 PDF 下次重新上传解析（不复用旧结果）——改错可重跑，但也意味着不能靠旧缓存省钱。
- `read_pdf` 单次内联预算 12000 字符（`output.maxInlineChars`）、最多 6 张图，带 cursor 续读。用它"复述"整篇 Markdown 会被切分且可能丢表格，**不要**这样产出转换结果。

## 当 Markdown 转换器用的正确姿势

产品定位是「让 Agent 带证据读文档」（按块/按页/按检索读 + 后台整份解析），不是导出器；但完整结果包已落盘，所以"能转，绕一步"：

1. 解析一次该 PDF（任一工具触发均可）。
2. 调 `read_pdf({ file_path: "...", focus: "artifacts" })` 取 `markdown_path`（指向 `full.md`）与 `manifest_path`。
3. 工件目录：`$DSH_HOME/cache/pdf-mineru/<result-id>/files/<fileId>/`，内含 `full.md`、`layout.json`、`content_list.json`、`model.json`、`images/img_<n>.png`（默认文件名由 `defaultArtifactFileName` 给出：markdown→`full.md`，layout→`layout.json`，model-output→`model.json`，content-list→`content_list.json`，images→`images/img_<i>.png`）。
4. 直接复制 `full.md` 改名 `.md` 即为转换结果，比让模型复述保真得多。

三个坑：

- **图是相对路径**：`full.md` 引用 `images/xxx.jpg`，只拷 `.md` 会丢图，要把整个 `files/<fileId>/`（含 `images/`）一起拷走。
- **缓存即产物**：设置页「存储 → 清空缓存」或 `stagingTtlMs`（默认 86400000）清理后即消失，先拷再清。
- **数据外发**：激活官方云时 PDF 上传到 mineru.net；涉密或未公开材料应切到本地 Provider（3.x :18000 / 4.x :8000），并先确认服务在跑。

## 与既有 OCR 路线的衔接

此前扫描件转写在无 PDF/OCR 库的 Windows 环境下走的是 Windows OCR 一段式流程，并踩过 markitdown 对无 `/Font` 层 PDF 返回空文本的坑（见《生态保护红线管理办法（征求意见稿）》2026-11 新版转写文档 `3efde35a-b061-4654-8264-da586a9d4c66`）。MinerU 的 OCR 正是一步到位，不需要再"pypdf 抽图 + rapidocr"两段拼，可作为该场景的替代路径。

## 本机当前状态（核查时）

- 插件 `dsh-pdf-mineru@0.2.3` 已装且 active，`read_pdf` / `async_parse_pdf` 在工具表中。
- 三个 Provider 均已填写，**激活 `mp_official`（官方云 v4）**；`MINERU_API_KEY` 存在于 DSH 凭据库（环境变量未设置，走凭据存储）。
- 默认：`model=pipeline`、`ocr=false`、`parseMethod=auto`、`language=ch`、公式/表格开启；`storage.cacheEnabled=true`。
- 缓存目录为空 → 尚未成功解析过任何文档；本地 3.x/4.x 两个地址仅填写未验证。
- 选型建议：规整公文/规划/报告用 Pipeline；仅在版面极乱、表格跨页嵌套、公式密集时切 VLM，且凡引用数值或公式都用 `read_pdf(view:"page")` 回原页核对。
