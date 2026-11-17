国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 技能 frontmatter 的 name 必须匹配 /^[a-z0-9]+(?:-[a-z0-9]+)*$/，否则加载器静默丢弃该技能：不出现在清单，`/` 菜单和 skill 工具都找不到，也没有报错提示。技能市场导入的 MinerU 技能原名 "MinerU Document Extractor"（含空格大写）因此失效；本机已改为 name: mineru-document-extractor，目录仍是 ~/.dsh/skills/miner-u-document-extractor（注意 skills-manager 无该技能管理记录，不会被来源回写覆盖，但重新导入会再次覆写坏掉）。编辑前需 danger-full-access，工作区外的技能目录不在 workspace-write 范围内；改完保存即热加载生效。
§
mineru-open-api v0.5.9 已全局安装在本机（C:\Users\ZY-WORK\AppData\Roaming\npm\mineru-open-api.cmd，已在 PATH）。flash-extract 免 token；MINERU_TOKEN 已配置到 C:\Users\ZY-WORK\.mineru\config.yaml（token source: config），extract 精度模式实测跑通（13 页 PDF → md + images 正常，--model vlm 可用，支持 md/html/latex/docx/json、--ocr --pages 批量）。注意 `auth --verify` 只校验格式，真实验证必须跑一次 extract；写 ~/.mineru 需 danger-full-access。
