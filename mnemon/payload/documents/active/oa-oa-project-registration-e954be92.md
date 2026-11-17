---
id: "e954be92-a0f9-493b-bcdf-12da72c67b54"
title: "院内 OA「项目预登记」自动化技能实测：四级联动、字段推测规则与保存校验坑（oa-project-registration）"
description: "把院 OA 项目预登记做成 DSH 技能的实测交接记录：安装路径与 sha1 校验、技能命名与笔误分流决策、七项信息的推测规则、四级联动与 layui 假下拉/顶层 OpenTop 弹层机制、save 与 verify 必须分两次调用、用“项目名称”查询替代“刷新”校验，以及 216 条记录基线的清理验证。"
status: "active"
created_at: "2026-11-17T08:00:18.539Z"
updated_at: "2026-11-17T08:00:18.539Z"
content_hash: "f2ffdee174aae074bc3b9fc60b18d39ac7d3bd94133fb2fdf5a019fb1c3e13cd"
source_paths:
  - "skills/oa-project-registration/SKILL.md"
  - "skills/oa-project-registration/scripts/oa_project_reg.js"
  - ".tmp/projreg-verify.png"
session_ids:
  - "mnemon-run-v1-6d4ed0b0-703c-4f9d-a9f8-5ca997491c88"
memory_body_ids:
  []
---

# 院内 OA「项目预登记」自动化技能（oa-project-registration）实测记录

在院 OA（浙江省国土空间规划研究院，`http://10.50.20.233:8123/OA/`）上，用 dsh-ego-browser 把「项目管理 → 项目预登记」做成可复用技能，并已端到端跑通：填表 → 保存 → 列表核对 → 删除测试记录 → 记录数回到 216 条、筛选与弹层全部复原。同一 OA 环境下的「外出登记」技能见既有文档 `ef833a05-2c7d-49ce-affa-e81abd0baf53`，两套页面同源，iframe 层次与 ego_js 排障经验可直接互参。

## 1. 产物与安装

| 位置 | 说明 |
| --- | --- |
| `C:\Users\ZY-WORK\.dsh\skills\oa-project-registration\SKILL.md` | 安装版技能正文，221 行：触发描述、七项信息从哪来、类型树、单位推测规则、字段对照、页面结构、排错清单 |
| `C:\Users\ZY-WORK\.dsh\skills\oa-project-registration\scripts\oa_project_reg.js` | 安装版脚本，426 行；一段可直接作为 `ego_js` 表达式执行的代码，`action` 取 `fill / save / verify / close` |
| 工作区源件 `skills/oa-project-registration/` | 与安装版 sha1 一致（`067d72fe1ccfc5f561d70b111881ed368b1340ab`），改完需重新同步并比对哈希 |
| `.tmp/projreg-verify.png`、`.tmp/projreg-form-filled.png` | 列表筛选核对截图与填好待保存的表单截图 |

安装目录在工作区外：workspace-write 下按 Windows 风格路径 `mkdir` 会被沙箱拒绝（效果标记 unknown），**不要盲目重试**，先只读确认目录/文件是否已存在，再以 sha1 比对判定安装是否成功。

## 2. 两个需要保留的设计决策

1. **技能名取 ASCII**：DSH 的 frontmatter `name` 必须是纯 ASCII 短横线格式，中文名会被加载器**静默丢弃**（不报错、清单里直接看不到）。因此 `name: oa-project-registration`，把“项目登记／项目预登记／项目备案”等触发词全部写进 description，用户说中文照样命中。
2. **任务书第 3 步“新建外出登记”是沿用外出技能时的笔误**：路径写的是 项目管理→项目预登记，故按**项目预登记**实现，并在技能里加分流说明（真要外出登记 → 转 `oa-outing-application`），避免以后串台。

## 3. 七项信息的处理规则（已写入技能）

- **项目名称**：只能由用户给出；缺了直接问，不猜。
- **项目类型**：四级联动「类型 / 子类 / 动作 / 层级」，按名称关键词推断；推不准就只填一级，并在回报里注明“是猜的”。
- **联系单位**：县／县级市 → “XX市/县自然资源和规划局”；设区市的区 → “<地级市>自然资源和规划局<区>分局”。**只能从「选择客户」里选已有单位**，搜不到就把候选发给用户，不新建单位。
- **联系人**：选完单位会自动弹出联系人列表；只有 1 个就直接选，多个或没有就停下来问。
- **联系电话**：默认值先填，选联系人时会被对方手机号覆盖，脚本在最后统一改回默认值。
- **承接部门负责人**：页面按承接部门自动带出，带出的不是预期值就停下问，不硬改。
- **所属地区**：选单位时自动带出；与项目地对不上时用 `provinceChange` 覆盖。

## 4. 页面机制（写脚本必须知道）

- 表单在 iframe 内：`/OA/Pro/Pro/edit?viewName=ProjectPreRegEdit&sTaskDesc=项目预登记&iKeyID=-1`。
- **项目类型、所属地区是 layui 假下拉**：`select` 被包成 `layui-unselect layui-form-select`（外层 `hidden:true`），选项挂在旁边的 `ul/li` 上；直接改 `value` 不会触发联动——一级改完，二级 `dds` 才出现（`规划制定与修改类` → `总体规划/详细规划/专项规划/国土空间设计`）。必须走 layui 的事件／模拟点选，且**改完要等一轮再读回**。
- **「选择客户」「选择联系人」是 OpenTop 弹层，挂在最顶层 document**，不在列表/表单 iframe 内；查元素要从 top 层找，例如 `topLayers` 里出现 `/OA//Sal/Sal/list?...&bSelMode=1` 即联系人弹层已打开。
- `fill` 的返回值是自检结果：`readback`（读回值）+ `missing` / `mismatch` / `errors` + `hint`。**有 missing/mismatch/errors 时不要点保存**，先修 CFG 再跑，否则会带着空字段提交。

## 5. 实测踩到的坑（都已写进技能的排错清单）

1. **保存后 iframe 会跳转到新记录的 `iKeyID`**：同一次 `ego_js` 里再 `await` 必超时。所以 `save` 只负责点「保存」并立即返回，`verify` 另起一次调用。
2. **列表「刷新」按钮不可靠**：有时十几秒才生效（实测一次 `elapsed: 12107ms`），拿它做保存校验会误判成“保存失败”。改用列表自带的**项目名称查询**，实测约 259ms 就能筛出目标行（`共 1 条`）。
3. **保存后总数会滞后**：点击返回时仍读到 `共 216 条`，稍后核对才是 `共 217 条`；不要用单次读数下结论。
4. **ego_js 里没有 `require`**：自写脚本用 `require` 会 `ReferenceError: require is not defined`，ego-browser 直接退出（code 1）。可用的全局有 `process`、`ego`、`navigator`、`crypto` 等，fs 由注入的绑定提供；脚本末尾用 `@@DSH_RESULT@@` 前缀回传 JSON 便于解析。
5. **截图偶发 CDP 超时**（`Page.captureScreenshot` timeout）：先 `ego_page_info`/快照确认页面状态再重试，不要原样重复同一动作。

## 6. 验证与善后（可复用的收尾标准）

- 建 3 条测试记录（总体规划／详细规划／总体规划短名各一），逐条 `fill → save → verify`；`verify` 在筛选后返回 `found:true, matches:1`、行内项目类型与单位均正确。
- 清理后台账：筛选清空、弹层关闭、层数 0，总数回到 `共 216 条` 且 `hasTest:false`，才算善后完成。
- **未做**：真实项目的落库（等用户给项目名称后先推类型/单位/地区确认再提交）；“有技能 vs 无技能”对比评测默认不跑（会在 OA 里反复建记录，需用户明确同意并用专用测试名跑完即删）。

## 下一步建议

用户若要正式使用，先给项目名称，由技能推一版类型树／联系单位／所属地区给用户确认，再执行 `fill → save → verify`；若后续 OA 页面改版，优先复查三处：四级联动的 option 文案、选择客户弹层的 `bSelMode`/`iCustomerID` 参数、列表查询框 `sProjectName` 的 id。
