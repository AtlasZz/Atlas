DSH 默认工作区记录（%USERPROFILE%\.dsh\storages\workspace.json 的 tables.workspaces）被删除后不会自动重建：global.defaultWorkspaceId 作为「首次使用身份」被保留（源码注释 retained after its registration is deleted），initializeDefault() 见到该字段就直接返回 entities.get(id)，不做创建。恢复办法：GUI 侧栏「添加工作区」重新登记 %USERPROFILE%\Documents\deepseek-harness\default-workspace（title 为自动标题 default-workspace 时 UI 会本地化显示为「默认工作区」）。
§
本机 DSH profile（%USERPROFILE%\.dsh\profiles\desktop）装的 dsh-skill-market@1.0.0 会在运行时改写品牌：侧栏字标「deepseek harness」→「兴湘智能体」、鲸鱼 logo→橙色图、favicon 同换。常量硬编码在 node_modules/dsh-skill-market/src/client.js：BRAND_NAME_TEXT(L1159)、BRAND_LOGO_DATA_URL(L1156)、mountBrand(L1245) 调用点 L1306，带 MutationObserver 自愈；插件无 config 开关，只能改代码或卸载。用户 2026-11-04 明确选择暂不修改。
§
表格已生成，保存在两个位置： - 工作区：[生态保护红线管理办法修改意见汇总表.xlsx](D:\Workspace\DSHarness\生态保护红线管理办法修改意见汇总表.xlsx) - 项目文件夹：[生态保护红线管理办法修改意见汇总表.xlsx](D:\S-生态保护红线\文件撰写\生态保护红线管理办法修改意见汇总表.xlsx) **表格结构**：表头下方共有13行数据（正文加附件共13条意见），共五列：序号、部门、意见建议、理由、采纳情况。采纳情况统一填写为“采纳”；部门按您的要求填写为**浙江省国土空间规划研究院**；意见建议一列将每条意见压缩为可直接报送的表述，理由一列保留原文理由；表格已设置自动换行、边框和页面横向打印设置，可直接打印，也可以在 Word 或 Excel 中继续编辑。 两点说明： 1. 如果这份汇总表最终要以**浙江省自然资源厅（省厅）**名义报部，部门一列就需要整体替换，请您告诉我，我马上改。 2. 表中共有13条意见：第六条、第七条、第八条、第九条、第十一条、第十三条、第十四条（一）（三）（六）（七）、附件第5点（海上养殖、203、一二三产融合）、第十四条（七）与附件第14点。检查后没有缺漏。
§
本机 DSH 沙箱下 pip 安装第三方包会报 Errno 13：根因是 os.mkdir(mode=0o700) 创建的目录对该进程不可写，pip 用 tempfile.mkdtemp 建的 pip-unpack-* 临时目录因此写不进去（0o755/0o777 正常）。解法：在工作区放 sitecustomize.py 把 os.mkdir 的 0o700 改写为 0o755，PYTHONPATH 指向它，再 pip install --target 到工作区内目录并把该目录加入 PYTHONPATH。另：markitdown 对纯扫描件 PDF 返回空文本（PDF 无 /Font 层），需先用 pypdf 提取内嵌 JPEG 页图、再用 rapidocr_onnxruntime OCR。
§
国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
Windows 上 DSH 沙箱（workspace-write）默认禁止写 `C:\Users\zhouy\.dsh\skills`，把脚本拷进技能目录需用 pwsh 的 sandbox_permissions=danger-full-access；而 create_skill 工具本身可以正常写入该目录。
§
PowerShell 的 `Get-Content` 按 GBK 解码 UTF-8 无 BOM 文件，中文会乱码；判断中文文件内容要用 read/grep 工具，写文件用 write/edit，别用 PowerShell 读写文本。Python 运行前设 `$env:PYTHONIOENCODING='utf-8'`。
§
PPT「版式讲解」转换器（Python + python-pptx）已固化为 skill `pptx-layout-narration`，脚本在 `C:\Users\zhouy\.dsh\skills\pptx-layout-narration\scripts\`（pptx_layout.py 解析层 / pptx_narrate.py 讲解生成 / verify_md.py 校验）。工作副本在 `D:\harness\tools\`。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 桌面 profile（C:\Users\ZY-WORK\.dsh\profiles\desktop）装 CN 版提示词增强必须用 npm 别名：在 profile 目录跑 `pnpm add dsh-prompt-enhance@npm:dsh-prompt-enhance-cn@0.3.5`。原因：dsh-prompt-enhance-cn 的 cordis.patch.yml 里 insert.name 写的是上游名 dsh-prompt-enhance，按真名装会 ERR_MODULE_NOT_FOUND 并回滚（0.3.5 仍未修）。之后在插件管理器/市场点更新会重装成 -cn 而再次失效，需重做别名。
