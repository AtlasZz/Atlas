国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 技能 frontmatter 的 name 必须匹配 /^[a-z0-9]+(?:-[a-z0-9]+)*$/，否则加载器只记一条 "invalid skill name" 警告并静默丢弃（技能不出现在清单，skill 工具也找不到）。技能市场导入的 MinerU 技能原名 "MinerU Document Extractor"（带空格大写）因此失效，已改为 mineru-document-extractor；该目录里重新导入会再次被覆写坏掉。
§
mineru-open-api v0.5.9 已全局安装（C:\Users\zhouy\AppData\Roaming\npm\mineru-open-api，Go 编译单文件 CLI + npm 壳）。flash-extract 免 token 可用；2026-11-17 已配置 MINERU_TOKEN 到 C:\Users\zhouy\.mineru\config.yaml，extract 精度模式实测跑通（上限 200MB/600 页，支持 vlm/pipeline/html、--ocr、--pages、md/html/latex/docx/json）。注意 `auth --verify` 只校验 token 格式，真实验证必须跑一次 extract。DSH 沙箱内 npm 默认缓存 EPERM、写 ~/.mineru 需 danger-full-access。
