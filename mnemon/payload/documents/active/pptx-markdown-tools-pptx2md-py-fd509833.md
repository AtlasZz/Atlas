---
id: "fd509833-8f1a-4a69-9543-ddbf871d3e6c"
title: "离线 PPTX→Markdown 转换方案（tools/pptx2md.py）与验证方法"
description: "2026-11-07 结论：本机无法访问 PyPI 安装 markitdown，改用 python-pptx 自建 pptx2md.py 输出 MarkItDown 兼容 Markdown；含标题推断、阅读顺序、画布外元素、备注与断链校验方法，以及 LibreOffice 与沙箱 ACL 两个环境障碍。"
status: "active"
created_at: "2026-11-07T12:56:35.238Z"
updated_at: "2026-11-07T12:56:35.238Z"
content_hash: "92fd7d8196893ae63ce6f2f1444647cc2741df133c4a34575f0a0eb7b9f9fd34"
source_paths:
  - "tools/pptx2md.py"
  - "【20260820zhu】0820配置方案和组合供应交流.md"
session_ids:
  - "mnemon-run-v1-d263419f-0942-4698-b15b-aa138a9e26e6"
memory_body_ids:
  []
---

# 离线 PPTX→Markdown 转换方案（tools/pptx2md.py）

记录时间：2026-11-07。工作目录 `D:\harness`。

## 背景与结论

需求是把一份 56 页的中文政府汇报 PPT（`【20260820zhu】0820配置方案和组合供应交流.pptx`，约 151 MB，桌面路径）转成 Markdown，保留正文、图片、表格和演讲者备注。

- 技能文档建议 `pip install 'markitdown[all]'`，但**本机 markitdown CLI 未安装，且无法访问 PyPI 及阿里云/腾讯云等镜像**（curl 在 TLS 握手阶段即被中断）。
- 结论：不依赖网络，用 MarkItDown 处理 PPTX 时所用的同一底层库 `python-pptx` 自建转换脚本 `tools/pptx2md.py`，输出沿用 MarkItDown 的约定（`<!-- Slide number: N -->` 分页标记、每页一个 `##` 标题、逐段一行、备注跟在页面内容后），使结果与 markitdown 产物结构一致，便于后续替换回官方工具。

## 工具设计要点

- **标题推断**：这份 PPT 56 页全部未使用标题占位符，所以先尝试 `slide.shapes.title`，为空时按"字号优先、其次靠上、再靠左"在非页脚文本中择优；仍无则退化为最靠上的短文本。标题取首行，首行过短（≤4 字）时与次行合并，超长截断加省略号。
- **阅读顺序**：按版面几何排序（先上→下，再左→右）；面积占比 ≥ 阈值的全版背景图排到页末，避免大图切碎正文。
- **画布外元素**：放映时不显示的元素（`t >= slide_h` 等）不混入正文，单独收进 `<details>` 折叠块，并给出条数；本例 25 页共 130 处、约 3447 字。这类内容常含讲稿式长句，混排会严重干扰正文可读性。
- **标题形状的剩余段落**：被判定为页标题的形状，其余段落原先被整体丢弃，导致 4 个段落漏转；修正后只消费标题文本、其余段落仍按正文输出。这是一类易漏的 bug（标题与正文混在同一个 text frame 里）。
- **图片**：按 blob 的 SHA1 去重，命名 `slideNN_imgNN.ext`，默认导出到与 md 同名的 `_assets` 目录，链接用相对路径；无 alt 时用"第N页图片M"，背景图前缀"背景图"。
- **其他**：表格转 Markdown 表格、图表读 series、备注清洗掉纯页码行、页脚类文本按前缀过滤。

## 验证方法（可复用）

不要只看"转换成功"，要对源文件逐段核对：

- 遍历源 PPTX 每个 `text_frame` 段落，检查清洗后的文本是否出现在对应页的 Markdown 中，统计 missing —— 本例 804 段，最终 `missing 0`（修复前 missing 4）。
- 统计图片链接：总链接数、唯一文件数、断链数 —— 本例 90 链接 / 84 唯一文件 / `broken 0`。
- 输出规模与结构计数：headings 数应等于页数（56）、`<details>` 块数、备注条数（45）、字符数与文件字节数。

## 环境障碍（两个，均已在本次绕开或修复）

1. **LibreOffice 不可用**：想用 `soffice --headless --convert-to pdf` 渲染幻灯片做视觉核对，报 `The application cannot be started` / `The component manager is not available`（`C:/Program Files/LibreOffice/program/services.rdb` 不存在），退出码 `-1073741819`。即装了 LibreOffice 也无法用其做渲染核对，需改用 python-pptx 几何/文本层面的自校验。
2. **工作区沙箱权限**：`D:\harness` 缺少 `WRITE_OWNER`，沙箱一启动就报 `SetNamedSecurityInfoW failed (Win32 5): grantWrite`，连 `Get-Location` 都失败。为当前用户补一条完全控制权限后恢复；改权限前已生成备份与回滚脚本，放在 `D:\harness-acl-recovery\`（`acl-backup-<guid>.json.ps1`）。若再次遇到同类启动失败，先查 ACL 而非怀疑 shell 或工具本身。

## 产物与备注

- 转换脚本：`tools/pptx2md.py`
- 转换结果：`【20260820zhu】0820配置方案和组合供应交流.md`（56 页，约 87.6 KB，38004 字符）+ 同名 `_assets` 目录（84 张去重图片，约 95 MB）
- 脚本可参数化：仅纯文字模式（不引用图片）、或把图片内嵌 base64 / 输出自包含 HTML，均为可选的后续变体。

## 已知边界

- 标题推断是启发式，对无标题占位符、且首屏为大号装饰文字的设计稿可能判断错，需人工抽查若干页标题。
- 图片去重按内容哈希，同一图片在不同页复用只导出一次，链接数会多于文件数（90 vs 84），这是预期行为，不是断链。
- 输出面向"可读的 Markdown 存档"，不做版式还原；需要版式保真时应走 PDF/图片路线。
