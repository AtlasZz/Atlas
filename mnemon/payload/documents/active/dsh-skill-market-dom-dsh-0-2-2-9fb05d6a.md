---
id: "9fb05d6a-cc71-4761-bd0a-c6c8bdd7a4e7"
title: "dsh-skill-market「技能市场」侧边栏入口不显示：DOM 注入在 DSH 0.2.2 分栏侧边栏上失效的排查记录"
description: "dsh-skill-market 0.32.x 在 DSH 0.2.2-rc.1 上侧边栏入口缺失的原因链：入口靠手工 DOM 注入而非官方 slot，新版侧边栏把「新建会话」移出 logoRow 导致锚点失效；含证据、排除项与推荐修法（sidebar.footer.action）。"
status: "active"
created_at: "2026-11-14T14:17:09.801Z"
updated_at: "2026-11-14T14:17:09.801Z"
content_hash: "6f0673c4d52904d545c9d9aa762a1f1631211511c5241a38d4292c31b0b57024"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-20c644cf-44e0-4363-b671-a704dab74ed1"
memory_body_ids:
  []
---

# dsh-skill-market 侧边栏入口不显示（DSH 0.2.2 本机排查）

时间点：2026-11-14，本机 desktop profile，DSH 0.2.2-rc.1，dsh-skill-market 0.32.1。
结论置信度：机制部分有源码证据；「入口被插到侧边栏最底部」为按 TSX/CSS 推演，**视觉未验证**（无浏览器截图能力）。

## 一句话结论

插件没有被禁用、也没有版本不兼容：它在 `apply` 里**手工往侧边栏 DOM 插一行按钮**（不走官方 slot），而 DSH 0.2.2 的侧边栏改成「分行」结构后，注入锚点全部失效，失败时静默不报错。

## 证据链

1. **插件确实加载成功**
   `plugin_manager list_plugins` 返回 `include:dsh-skill-market / enabled:true / fiberPhase:"active"`；host 端 `src/index.js:944` 声明 `inject = ['connection','skills']`，两个服务在 0.2.2 中都存在（`ctx.skills` 由 `@deepseek-ai/dsh-skill-filesystem/lib/index.js:29` 提供）。
   版本门禁不是原因：`dsh-app-boot/src/plugin-compatibility.ts` 用 `semver.satisfies(..., {includePrerelease:true})`，`0.2.2-rc.1` 满足 `>=0.1.1-rc.1`。

2. **入口是 DOM 注入，不是 slot**
   `dsh-skill-market/src/client.js:1039-1090`：
   - `navSidebarRoot()`：`[class*="sidebarCol"]` → 内部 `[class*="logoRow"]` → 返回 `logoRow.parentElement`
   - `navPlaceEntry()`：要求 `root.querySelector('button[class*="newSession"]')` 存在，再插到 `base.nextElementSibling` 之前
   README 声称使用 `sidebar.footer.action` slot，但代码里只注册了 `shell.overlay`（`client.js:1175`）和 `conversation.input.left`（`:1191`），没有该 slot 注册。

3. **0.2.2 侧边栏结构变化（失效根因）**
   `@deepseek-ai/dsh-client-ui-sidebar/src/client/SidebarRoot.tsx`：
   - `logoRow`（:330）只装品牌区与折叠按钮；
   - 「新建会话」在**独立的** `newSessionRow`（:383），且该行只有一个 renderSlot 子节点，按钮是其父元素的最后一个子元素。
   因此插件的 `button.closest('[class*="logoRow"]')` 返回 `null`，`anchor = button.nextElementSibling` 也是 `null`，最终执行 `root.insertBefore(entry, null)` —— 即把入口**追加到整个侧边栏 `.root` 末尾**（「设置」那排之后），折叠成图标栏（rail）时还会被 CSS 压成约 18px 小圆钮。
   另：`newSessionRow` 仅在 `canStartSession` 为真时渲染；条件不成立时 `navPlaceEntry` 找不到按钮直接 `return false`，**完全静默失败，无兜底**。

4. **已排除项**
   - 类名哈希失效：实际类名为 `mP3x-q_sidebarCol`、`wtg9ga_logoRow`、`wtg9ga_newSession`，选择器可匹配。
   - 1.0.0 品牌改写：1.0.1 已删除，与本次无关。
   - `shell.overlay` / `conversation.input.left` 键改名：0.2.2 仍存在。
   - `dsh.client.inject` 中列出的包不存在：`client-modules` 对缺失依赖只跳过排序边。

## 自证步骤（用户侧 30 秒）

1. 鼠标移到侧边栏**最底部、「设置」行下方**，看是否有「技能市场」；
2. 把侧边栏折叠成图标栏，看最底部是否多出一个小圆钮；
3. 若都没有：设置 → 插件，检查 `dsh-skill-market` 是否标为 page-local 同步失败（唯一能看到客户端半加载失败的入口）。

## 正解方向

- 首选：改为注册官方 `sidebar.footer.action`（list 作用域，声明于 `dsh-client-ui-sidebar` 的 `contract/slots.ts:75`），位置恰在「设置」旁，随侧边栏状态自适应；
- 次选：若继续 DOM 注入，锚点改为插到 `[class*="newSessionRow"]` 之后，并为 `canStartSession` 为假、找不到锚点的情况加显式兜底/日志。

## 备注

排查在工作区留下调试目录 `D:\Workspace\DSHarness\.dsh-dbg\`（asar 解析脚本 + 解包出的宿主源码），未改动 profile 或插件文件。
