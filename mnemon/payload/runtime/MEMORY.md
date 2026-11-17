国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 技能 frontmatter 的 name 必须匹配 /^[a-z0-9]+(?:-[a-z0-9]+)*$/，否则加载器静默丢弃该技能：不出现在清单，`/` 菜单和 skill 工具都找不到，也没有报错提示。技能市场导入的 MinerU 技能原名 "MinerU Document Extractor"（含空格大写）因此失效；本机已改为 name: mineru-document-extractor，目录仍是 ~/.dsh/skills/miner-u-document-extractor（注意 skills-manager 无该技能管理记录，不会被来源回写覆盖，但重新导入会再次覆写坏掉）。编辑前需 danger-full-access，工作区外的技能目录不在 workspace-write 范围内；改完保存即热加载生效。
§
mineru-open-api v0.5.9 已全局安装在本机（C:\Users\ZY-WORK\AppData\Roaming\npm\mineru-open-api.cmd，已在 PATH）。flash-extract 免 token；MINERU_TOKEN 已配置到 C:\Users\ZY-WORK\.mineru\config.yaml（token source: config），extract 精度模式实测跑通（13 页 PDF → md + images 正常，--model vlm 可用，支持 md/html/latex/docx/json、--ocr --pages 批量）。注意 `auth --verify` 只校验格式，真实验证必须跑一次 extract；写 ~/.mineru 需 danger-full-access。
§
OA 项目预登记（10.50.20.233:8123）填表时，表单窗口的 w.provinceChange() 会抛 "Cannot read properties of undefined (reading 'render')"，脚本里用 CFG.region 覆盖所属地区因此不生效；正确做法是按 layui 假下拉逐级点 province→city→county（点 .layui-select-title 再点 dd[value]）。另外客户档案的地区常是单位注册地而非项目地（如“丽水市自然资源和规划局莲都分局”带出 浙江/杭州市），保存前必须核对三级是否等于项目地。
§
OA 项目预登记若某联系单位在客户库里没有联系人（联系人弹窗显示“无数据”），用户给出姓名/手机并明确要求补录时，可在「选择联系人」弹窗点「新增」→ linkManEdit 表单填 *姓名 + 手机 → 保存（关掉编辑层后新联系人立刻出现在该单位列表，勾选后点「选择」即带出 id）；这属于技能默认禁止的新增操作，只在用户明确交代时做。选中联系人会把项目「联系电话」覆盖成对方手机，需要按约定改回。
