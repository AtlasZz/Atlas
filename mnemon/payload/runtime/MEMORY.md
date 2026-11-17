国土空间总体规划调整只有"动态维护"和"规划修编（修改）"两条法定路径。
§
文档转写时手写批注默认不识别、不转换（用户明确要求记住）。
§
DSH 技能 frontmatter 的 name 必须匹配 /^[a-z0-9]+(?:-[a-z0-9]+)*$/，否则加载器只记一条 "invalid skill name" 警告并静默丢弃（技能不出现在清单，skill 工具也找不到）。技能市场导入的 MinerU 技能原名 "MinerU Document Extractor"（带空格大写）因此失效，已改为 mineru-document-extractor；该目录里重新导入会再次被覆写坏掉。
§
mineru-open-api v0.5.9 已全局安装（C:\Users\zhouy\AppData\Roaming\npm\mineru-open-api，Go 编译的单文件 CLI，装的是 npm 包装）。flash-extract 免 token，2026-11-17 实测可用；extract/crawl 需 MINERU_TOKEN（mineru.net 申请）。DSH 沙箱内 npm 默认缓存 C:\Users\zhouy\AppData\Local\npm-cache 写会 EPERM，全局安装需 danger-full-access。
