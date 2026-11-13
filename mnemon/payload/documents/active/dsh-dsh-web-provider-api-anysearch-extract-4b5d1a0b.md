---
id: "4b5d1a0b-fa7b-494c-a22c-3b027cc7a6df"
title: "DSH 网页搜索/抓取链路梳理：dsh-web provider 选择、新版 API 开关与 AnySearch extract 的挂载方式"
description: "本机 DSH（desktop profile）网页能力实测结论：web_search 与 web_fetch 同属 @deepseek-ai/dsh-web 插件；「启用新版 Web 搜索 API」开关只切换新版 /v1/tools/* 与 Messages 回退两条通道；AnySearch 的 extract 不注册独立工具，只作为 web_fetch 的 fetch provider，当前 fetchProvider 为 http 故未走 AnySearch。含代码路径、错误码映射与局限。"
status: "active"
created_at: "2026-11-12T12:29:26.586Z"
updated_at: "2026-11-12T12:29:26.586Z"
content_hash: "6823bdf71f45ad6df9d17b45387eb48594f8034d9aca3becea517b3dd1dd77e9"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-c1e2a570-9a4b-4242-8983-a4c5a768bd17"
memory_body_ids:
  []
---

> 说明：本文引用的源文件均位于本机 DSH profile 目录 `C:\Users\zhouy\.dsh\profiles\desktop\`，在会话 workspace 之外，故未登记为 sourcePaths，路径直接写在正文中。

## 1. 问题

用户连续追问三件事，本文给出可复用的结论：

1. 「DSH 自带的网页搜索插件是不是已经关了？」
2. 「我关掉的是什么新版的搜索？」
3. 「anysearch 为什么无法抓取全文，不是有 extract 么？」

## 2. 结论

- **插件没关。** `web_search` / `web_fetch` 同属 `@deepseek-ai/dsh-web` 一个插件，关闭会让两个工具一起消失；实测两工具都在注册表中且能返回结果。
- **用户关掉的是 `web-deepseek-api` 开关**（设置页「启用新版 Web 搜索 API」），它只切换搜索/抓取的**通道**，不停用插件。默认关闭；关闭后搜索回退到 Messages API 由辅助模型自行多轮搜索（配置中 `maxUses: 5`，注释写明「仅适用于 Messages API」）。
- **AnySearch 的 extract 一直存在，但不叫 `anysearch_extract`。** 它被注册为 `web_fetch` 的一个 fetch provider；要真正用上必须让 `web.fetchProvider = anysearch`，而本机当前是 `http`（进程内直连 GET，不渲染 JS、无浏览器指纹）。

## 3. 证据与代码路径

Profile 根目录：`C:\Users\zhouy\.dsh\profiles\desktop\`（当前 GUI 跑的是 desktop profile，另有 `web` profile，互不影响）。

- `cordis.patch.yml:55-59`
  ```yaml
  - id: web
    name: '@deepseek-ai/dsh-web'
    config:
      searchProvider: deepseek-official
      fetchProvider: http      # ← 决定 web_fetch 走哪家
  ```
  无 `disabled: true`，`cordis.yml` 根条目为空、未被改写；`package.json` 的 bundles 中 `@deepseek-ai/dsh-web-app` 与 `@anysearch/anysearch-dsh` 都在。
- 新版 API 对应两个端点：`POST https://api.deepseek.com/v1/tools/search` 与 `/v1/tools/fetch`。开关关闭后抓取改走 `fetchProvider: http`，云端抓取路径（含此前 Reddit 抓取 404 那类问题）不再使用。
- `node_modules/@anysearch/anysearch-dsh/lib/index.js:64-68`
  ```js
  ctx.web.registerSearchProvider(new AnySearchProvider(client));      // id: 'anysearch'
  ctx.web.registerFetchProvider(new AnySearchFetchProvider(client));   // id: 'anysearch'
  if (ctx.tools.get('web_fetch') === undefined) { applyWebFetchTool(...) }
  ```
- `lib/fetch-provider.js:31` 才是调用 `/v1/extract` 的位置：`mapAnySearchExtractResponse(await this.client.extract({ url: request.url }, signal))`。
- `lib/provider.js:5`：`export const ANYSEARCH_PROVIDER_ID = 'anysearch'`——即 `fetchProvider` 应填的 id。
- `README.zh-CN.md:123` 明确说明不额外注册 `anysearch_extract`，提取统一通过原生 `web_fetch` 暴露。
- 错误码映射（`lib/fetch-provider.js:41-51`）：`invalid_extract_url→WEB_INVALID_URL`、`extract_target_blocked→WEB_BLOCKED_URL`、`extract_content_too_large→WEB_FETCH_TOO_LARGE`、`extract_unsupported_content→WEB_UNSUPPORTED_CONTENT_TYPE`、`extract_timeout→WEB_FETCH_TIMEOUT`、`aborted/extract_canceled→WEB_ABORTED`。

## 4. 容易混淆的三组概念

| 容易混淆 | 实际关系 |
|---|---|
| `web-search-deepseek`（旧版，基于 Messages）vs `web-deepseek-api`（新版 Search/Fetch API） | 两个不同配置命名空间，名字相近但不是一回事 |
| `anysearch_search({includeContent:true})` vs extract | 前者只是带回搜索索引里的 `content` 片段（受 `maxRenderedContentChars` 限制，默认 12000 字符）；全文提取是另一个接口 `POST /v1/extract` |
| 关「搜索」vs 关「抓取」 | 同一插件内两个 provider 配置项，可单独切换；插件本身仍在 |

## 5. 切换到 AnySearch 全文提取

两条路径：
1. GUI：`@deepseek-ai/dsh-web` 设置项里把「抓取提供方」选为 AnySearch；
2. 改配置：`cordis.patch.yml` 中 `fetchProvider: anysearch` 后重启（该文件在 workspace 之外，需完全访问授权）。

## 6. 局限（不要过度承诺）

- `lib/fetch-provider.js:11-17` 注释与实现显示：Extract 公开成功响应省略源站状态与截断元数据，适配层把 `truncated` 固定为 `false`、`statusCode` 固定为 `200`——**服务端真截断了模型侧也看不出来**。
- `extract_target_blocked` 等错误码的存在说明 AnySearch 自身也会拒抓部分站点（付费墙、风控站）。
- 回退到 Messages 通道的代价：搜索质量与延迟取决于辅助模型（最多 5 次），更易跑偏或漏搜；`fetchProvider: http` 遇 Cloudflare/登录墙站点大概率 403 或空正文，且受本机出口网络影响（此前本机直连 Reddit 与 GitHub API 均失败过）。
- anysearch 搜索有独立垂直领域，不依赖 DSH 搜索通道，不受该开关影响，仍是最稳定的搜索来源。
