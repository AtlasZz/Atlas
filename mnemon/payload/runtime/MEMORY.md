国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 桌面 profile（~/.dsh/profiles/desktop）中 dsh-prompt-forge 与内置实验插件 @deepseek-ai/dsh-experimental-client-ui-voice-input 争夺 conversation.input.activity（single 单座位槽位），两者都不写 priority 默认 0，同优先级注册会直接抛错而非覆盖。用户选择保留 prompt-forge（✨ 按钮 / Ctrl+Alt+E），已在 cordis.patch.yml 末尾加 `- id: ui-voice-input` + `name: @deepseek-ai/dsh-experimental-client-ui-voice-input` + `disabled: true`；语音转写后端 speech-to-text / speech-to-text-sensevoice / api-speech-to-text 仍保持启用。恢复语音输入只需删掉该条目或改 disabled: false。
§
2025-11-17 18:40 左右 ~/.dsh/profiles/desktop/cordis.patch.yml 被重写，image-gen 自定义 provider（openai-compat，https://ai.iisbo.com/v1，gpt-image-2.5-sunburst）、ui-theme、better-sidebar 等条目丢失，图片生成回落到 google/gemini-3.1-flash-image；旧配置备份在 cordis.patch.yml.bak-1794567780474（11-13 18:34）。用户当时选择「先不恢复」。
§
DSH 技能无法由模型加载有两个独立原因（都无报错）：① frontmatter 的 name 必须匹配 /^[a-z0-9]+(?:-[a-z0-9]+)*$/；② frontmatter 若写 `disable-model-invocation: true`，DSH 解析器会令 modelInvocable=false——技能仍出现在用户 `/` 菜单但不在模型清单里，skill 工具返回 "not available for model invocation"。MinerU 技能（市场导入，原名 "MinerU Document Extractor"）两条都中，已改为 name: mineru-document-extractor 并删掉 disable-model-invocation 行；目录仍是 ~/.dsh/skills/miner-u-document-extractor，改完保存即热加载生效。注意 skills-manager 无该技能管理记录，重装不覆盖，但重新从市场导入会再次覆写坏掉（需重做这两处）。改 ~/.dsh 需 danger-full-access。
§
mineru-open-api v0.5.9 已全局安装在本机（C:\Users\ZY-WORK\AppData\Roaming\npm\mineru-open-api.cmd，已在 PATH）。flash-extract 免 token；MINERU_TOKEN 已配置到 C:\Users\ZY-WORK\.mineru\config.yaml（token source: config），extract 精度模式实测跑通（13 页 PDF → md + images 正常，--model vlm 可用，支持 md/html/latex/docx/json、--ocr --pages 批量）。注意 `auth --verify` 只校验格式，真实验证必须跑一次 extract；写 ~/.mineru 需 danger-full-access。
§
OA 项目预登记（10.50.20.233:8123）填表时，表单窗口的 w.provinceChange() 会抛 "Cannot read properties of undefined (reading 'render')"，脚本里用 CFG.region 覆盖所属地区因此不生效；正确做法是按 layui 假下拉逐级点 province→city→county（点 .layui-select-title 再点 dd[value]）。另外客户档案的地区常是单位注册地而非项目地（如“丽水市自然资源和规划局莲都分局”带出 浙江/杭州市），保存前必须核对三级是否等于项目地。
§
OA 项目预登记若某联系单位在客户库里没有联系人（联系人弹窗显示“无数据”），用户给出姓名/手机并明确要求补录时，可在「选择联系人」弹窗点「新增」→ linkManEdit 表单填 *姓名 + 手机 → 保存（关掉编辑层后新联系人立刻出现在该单位列表，勾选后点「选择」即带出 id）；这属于技能默认禁止的新增操作，只在用户明确交代时做。选中联系人会把项目「联系电话」覆盖成对方手机，需要按约定改回。
