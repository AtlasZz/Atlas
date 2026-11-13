---
id: "2e04afa7-beeb-4f92-8eca-0d29cf0d213e"
title: "AnySearch DSH 插件实测：vertical tag 路由问题与可用检索姿势（desktop profile 0.1.8）"
description: "本机 desktop profile 中 @anysearch/anysearch-dsh 0.1.8 的运行状态实测：patch 同时接管 web_search/web_fetch；general.general 在 cn 区错误落到裁判文书网索引（11 次请求仅 6 次有效），social_media.social_media 的 wechatmp/zhihu 通道可用；含排查方法、可复现的调用姿势与未证实部分。"
status: "active"
created_at: "2026-11-12T12:49:56.737Z"
updated_at: "2026-11-12T12:49:56.737Z"
content_hash: "963ebf9771800a19ca26cdd92a06b85c735f2fe3b1e6e7a11814e8fcbeb75d20"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-24692b23-104c-4029-897b-ec01b7ea489a"
memory_body_ids:
  []
---

# AnySearch DSH 插件实测：vertical tag 路由问题与可用检索姿势

> 2026-11-12 实测，目标是回答「这次到底有没有真的调用 AnySearch API、插件运行得好不好」。结论基于当轮真实调用记录、安装文件与凭据引用，不含推测以外的判断。

## 1. 当前接入状态（desktop profile）

- 版本：`@anysearch/anysearch-dsh@0.1.8`，安装在 `desktop` profile 的 profile 级 node_modules 中；同时列入 `package.json` 的 `dsh.profile.bundles`，随 profile 启动加载。
- patch 生效范围（`node_modules/@anysearch/anysearch-dsh/cordis.patch.yml`）：
  ```yaml
  - id: web
    config:
      searchProvider: anysearch
      fetchProvider: anysearch
  - insert:
      - id: web-search-anysearch
        name: '@anysearch/anysearch-dsh'
        config:
          apiKeyEnv: ANYSEARCH_API_KEY
  ```
  即 **`web_search` / `web_fetch` 本身也被改道到 AnySearch**，不只是 3 个专用工具。排查「网页搜索质量飘忽」时必须把默认工具算进同一条链路。
  > 与既有文档 `4b5d1a0b-fa7b-494c-a22c-3b027cc7a6df`（记录的早期状态为 `searchProvider: deepseek-official`、`fetchProvider: http`）相比，当前 profile 已切换到 AnySearch 接管；两文分别是不同时间点的快照，以本文为准判断当前状态。
- 注册工具：`anysearch_capabilities`、`anysearch_search`、`anysearch_batch_search`。调用均无异常。
- 凭据：`.credentials.yaml` 的 `refs` 中存在 `ANYSEARCH_API_KEY`（`as_sk_` 前缀、38 字符），属真实鉴权而非匿名模式，因此结果质量差与配额/匿名降级无关。
- 日志：`C:\Users\zhouy\.dsh\dsh-console.log` 中 anysearch 相关记录 0 条，无崩溃、超时、鉴权失败；batch 调用往返约 1.5s。

## 2. 核心问题：`general` 通道在 `cn` 区错误路由

实测 11 个搜索请求中 6 个有效（≈55%），**5 次无效请求全部集中在 `general` 通道**：

- 返回体 `source: wenshu_court_gov_cn`，即命中的是中国裁判文书网索引；
- 现象与查询词主题无关：问「绘本」「幼儿园」等生活类词条，会被当成案件当事人/案由去捞裁判文书；
- 换 `zone=cn` → `zone=intl`、中文 query → 英文 query，只要仍走 `general.general`，多数情况仍回落裁判文书库（英文 query + intl 一次例外，命中绘本站点与 Crossref 混杂结果）。

判断：这是 AnySearch **服务端 vertical 路由**的问题，不是插件 bug，也不是参数传错——请求参数、tag、zone 均通过 capabilities 校验后下发。因为是远端行为，随服务端或插件版本变化，不应视为永久事实。

## 3. 可用姿势（已验证）

优先使用 `social_media.social_media` 这一 vertical，而非 `general`：

| 用途 | 参数 | 实测效果 |
|---|---|---|
| 中文公众号文章 | `tag=social_media.social_media`，`params={type: "wechatmp"}` | 走搜狗微信，返回标题、摘要、链接，成功率高 |
| 知乎问答 | `tag=social_media.social_media`，`params={type: "zhihu"}` | 直接返回问答**正文全文**，可直接作为内容源 |
| 批量抓取 | `anysearch_batch_search`，同 tag/params，多条 query | 1.5s 级返回，适合一次取多篇 |

调用约束（源码 `lib/tools/search.js` 校验）：`maxResults` 必须是 1–20 的整数；`tag` 必须与 `anysearch_capabilities` 返回的标签完全一致；`params` 只接受字符串/有限数字/布尔标量；`zone` 仅 `cn` | `intl`。**在给 `tag`/`params` 之前先调 `anysearch_capabilities`**，不要凭印象填。

## 4. 排查方法（可复用）

1. `anysearch_capabilities` 拉取可用 tag 及其声明的 params，确认走的是哪个 vertical。
2. 看返回体里的 `source` 字段——它是判断「是否落到预期索引」的最直接证据，比看标题猜更快。
3. 换 vertical 做对照（`general` vs `social_media`），而不是反复换 query 措辞；本次 5 次无效请求就是换措辞的无效尝试。
4. 校验凭据走的是鉴权还是匿名：`lib/client.js` 中 `resolveApiKey` 命中时 `authentication = 'credential'`，否则 `'anonymous'`；可据此排除配额/降级因素。
5. 插件是否真的加载：看 profile 的 `package.json` 的 `dsh.profile.bundles` 与 patch 实际内容，而非只看 `node_modules` 里有包。

## 5. 待确认 / 未证实

- 「`general` 在 `cn` 区固定落到裁判文书网」只在 0.1.8 + 当日服务端观察到，失效条件未知；升级插件后应重新验证而不是沿用结论。
- 是否升级到 npm 最新版、是否把默认 `searchProvider` 改回其他 provider，均未执行，属可选动作；改 patch 前应先备份。
- 本地 `dsh-console.log` 无请求级明细，调用次数是靠 Agent 侧工具调用记录统计的，不是插件自身日志。
