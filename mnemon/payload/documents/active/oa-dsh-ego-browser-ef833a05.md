---
id: "ef833a05-2c7d-49ce-affa-e81abd0baf53"
title: "院内 OA「外出登记」自动化实测：页面结构、字段机制与提交流程（dsh-ego-browser 技能）"
description: "把院 OA 外出登记做成 DSH 技能的实测记录：三层 iframe 里的表单字段 id 与 laydate/layui 触发方式、提交=保存+「提交流程」两步、审批链 iSelCode 分支、菜单定位坑与 ego_js 排障；含产物路径与未决风险（凭据明文、未做真实提交验证）。"
status: "active"
created_at: "2026-11-16T06:27:34.204Z"
updated_at: "2026-11-16T06:27:34.204Z"
content_hash: "531c5c75460c0ffe79204bd70ab0d0afdd20557b4428add1935f84ffa2eed0f0"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-f9b309c5-d4af-43e5-b28e-dd1a002b3264"
memory_body_ids:
  []
---

# 院内 OA「外出登记」自动化实测记录

目标：让 DSH 用 dsh-ego-browser 在院 OA（浙江省国土空间规划研究院，`http://10.50.20.233:8123/OA/`）走完「外出登记」表单。2026-11-16 已从登录到试填跑通，**未保存、未提交**，表单用「返回」关闭，未留草稿记录。

## 1. 产物位置

- 技能：`C:\Users\ZY-WORK\.dsh\skills\oa-outing-application\SKILL.md`（159 行，约 10.6 KB：触发描述、5 项必填信息、六步流程、字段对照、页面结构、排查清单）
- 脚本：`C:\Users\ZY-WORK\.dsh\skills\oa-outing-application\scripts\oa_outing.js`（311 行，约 12.8 KB）
- 试填截图：同目录 `outing-form-filled.png`
- 脚本用一份 CFG + 四个 action：`fill`（只填并读回）/ `save`（点保存，存草稿，适合截图确认）/ `submit`（点提交，弹出「提交流程」小窗）/ `confirmFlow`（在小窗里点「提交流程」）；`submit`、`confirmFlow` 必须显式 `confirmed: true`。返回 `{ok, action, readback, missing, mismatch, errors, messages, step, hint}`。

## 2. 导航与页面结构（三个坑）

1. **菜单不能用文字定位**：`text=外出登记` 会同时命中标签页和待办表格里的同名文字（报多个匹配）。改用 `[lay-href*="GoOutList"]`；菜单项自身是 `li[cid="304"]` 下 `<a lay-href="/OA//Hrp/Hrp/list?viewName=GoOutList" icfunctionid="30410" isystem="2" icclassid="304">`。
2. **菜单已展开时再点「考勤管理」会收起**，所以流程第一步是"先看可见就直接点，不可见才点一级菜单"。
3. **表单在三层 iframe 里**，`ego_click` 的 CSS 选择器穿不过去；所有字段动作走 `ego_js` 的 DOM 操作（先取到 `src` 含 `GoOutList` 的 iframe 文档）。

## 3. 字段机制（DOM id 与触发方式）

| 字段 | DOM | 要点 |
| --- | --- | --- |
| 外出人员 | 人员选择弹层（`UsersSelect` / `_UsersSelect` / `SetFlowUser` 等全局函数） | 选人后人数自动带出；姓名与 `personCode` 都要读回校验 |
| 开始/结束日期 | `tBeginTime1` / `tEndTime1` | laydate 只读输入框，直接改 value 不会算天数，**必须补发 `change` 触发 `SelectDate()`**，天数由页面自己算 |
| 上午/下午 | 下拉，选项为 `["", "上午", "下午"]` | layui 假下拉 |
| 出行方式 | `FMHrp_GoOut_sOutType` | 取值：网约车 / 单位派车 / 公共交通 / 其他方式；选「其他方式」后 `carVisible=false`、`custom3Visible=true`（用车相关字段只在「单位派车」出现）；改 `select.value` 不联动，要按 layui 方式触发 |
| 外出事由 | `mRemark` | 必填，且**不许编造**；为空时脚本直接拒绝执行并提示先问用户 |

读回校验：填完把表单值读回成 `readback`，与 CFG 逐项比对进 `missing` / `mismatch`，为空才算 `ok`。

## 4. 「提交」与「提交流程」是两步

- 表单底部按钮：`#btnsave`（保存）/ `#btnsubmit`（提交）/ `#btnrefresh`（刷新）/ `#btnreturn`（返回）。
- 点 `提交` = 先保存，再弹出 600×350 的 `RootPath + '/Flow/FlowControl?sViewName=FlowSubmit&options=...'` 窗口（标题「提交流程」，按钮「提交流程」「关闭窗口」）。**窗口里那个「提交流程」才真正发起审批**；`submit` / `confirmFlow` 拆两步就是为此。
- 流程窗口回调用 `parent.layui.flowProcess.result = Result; parent.layui.flowProcess.render();` 通知父页。

## 5. 审批链与 iSelCode 分支

链路固定：**经办人 → 审批人（部门负责人）→ 分管副院长**。填错一个字段，后面三人一起返工，所以"填完先给用户看截图"优先于"急着提交"。

页面脚本 `flowProcess.render({ iSelCode })` 的分支（sal.js）：

- 副院长/分管副院长：单位派车 → `366`（经办人-副院长外出用车），其余 → `365`
- 部门负责人：单位派车 → `354`，其余 → `352`
- 网约车分支（`364` 部门负责人网约车、副院长网约车）在页面源码里**被注释掉**，所以网约车与"其他方式"走同一套代号。

## 6. 已验证与未验证

已验：`fill` 读回 `persons=周宇 / personCode=19 / userCount=1`，`2026-11-17 上午 → 2026-11-17 下午`，天数自动 `1`，`outType=其他方式`，`missing`/`mismatch` 全空（截图为证）。

未验：真实保存与提交会往真实 OA 落单，需用户给出人员/起止时间/事由后再跑，故 `save`/`submit`/`confirmFlow` 尚未在真实单据上走通。

## 7. 排障笔记

- `ego_js` 里 `page.cdp` 不是函数（`page.cdp is not a function`）；page 暴露的键里没有 `cdp`，只能用 `evaluate` 或 `ego_js` 直接操作 DOM。
- layui 假下拉、laydate 只读框：只改 DOM value 不会触发业务联动（天数、用车字段、流程代号），必须补发对应事件或调用页面既有处理函数。
- 若"点提交后没有出现提交流程窗口"，先看 `messages` 与弹层数量，不要重复点击。

## 8. 待决风险

1. **凭据明文**：默认账号密码按用户要求写进了技能，技能目录一旦分享/打包会连同密码带出；建议后续改为"首次登录后依赖持久化登录态，密码另存"。
2. **真跑一单**：待用户提供 人员 / 开始-结束（日期+上午下午）/ 事由，出行方式缺省按「其他方式」。
3. skill-creator 默认的"多子代理跑基准测试"被刻意跳过——每次跑都会在真实 OA 落单子；若要做评测，应改成只跑到「保存」再删草稿的方式。
