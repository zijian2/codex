# Codex App Connector（ChatGPT Apps/Connectors）实现分析报告

> 基于 openai/codex 仓库源码（dev 分支）与 OpenAI 官方文档整理，2026-07。

## 一、什么是 App Connector

App Connector 即 **ChatGPT 生态中的 "Apps"**（2025-12 起官方将 "connectors" 统一更名为 "apps"）：把 Gmail、Google Calendar、Google Drive、Notion、Figma 等第三方服务接入 ChatGPT/Codex，让模型可以直接搜索、读取乃至操作这些服务中的数据。部分 app 提供交互式 UI，部分则是纯数据连接器。

对 Codex CLI 而言，它复用了 ChatGPT 的这套应用生态：

- **托管在 OpenAI 服务端**：应用目录、OAuth 授权、第三方凭据、工具执行全部在 ChatGPT 后端完成，本地不保存任何第三方 token。
- **以 MCP 形式接入**：Codex 把 ChatGPT 后端的 apps 服务当作一个内置的远程 Streamable HTTP MCP server（名为 `codex_apps`）接入，connector 的能力以 MCP tools 的形式暴露给模型。
- **要求 ChatGPT 登录**：只有使用 ChatGPT 账号（Codex backend auth）登录时可用；纯 API key 登录不可用。

## 二、官方文档链接

| 文档 | 链接 | 相关内容 |
|---|---|---|
| Codex 配置参考 | <https://developers.openai.com/codex/config-reference> | `features.apps` 开关、`[apps]` 配置段（enable/approval/destructive 等） |
| Codex App Server 协议 | <https://developers.openai.com/codex/app-server> | `app/list` RPC、`AppInfo` 字段、`$<app-slug>` 提及方式、app 工具审批流程 |
| Codex CLI 功能 | <https://developers.openai.com/codex/cli/features> | CLI/TUI 功能总览（MCP、审批模式等） |
| Codex Changelog | <https://developers.openai.com/codex/changelog> | apps/connectors 相关更新记录 |
| Apps in ChatGPT（帮助中心） | <https://help.openai.com/en/articles/11487775-connectors-in-chatgpt> | apps/connectors 概念、连接与授权方式（connectors → apps 更名说明） |
| App use cases and prompts | <https://help.openai.com/en/articles/12084614-connector-use-cases-and-prompts> | 各 app 的用法示例 |
| Apps SDK：Connect from ChatGPT | <https://developers.openai.com/apps-sdk/deploy/connect-chatgpt> | 开发者如何构建/接入自己的 app（MCP endpoint） |
| Bring your app to ChatGPT | <https://developers.openai.com/codex/use-cases/chatgpt-apps> | 用 Codex 开发 ChatGPT app 的端到端流程 |

官方文档要点摘录：

- `app/list` 用于拉取可用 app；CLI/TUI 中 `/apps` 是面向用户的选择器。
- 每个 app 条目同时包含 `isAccessible`（用户是否已连接可用）和 `isEnabled`（本地 config.toml 是否启用），用于区分"服务端安装/授权状态"与"本地启用状态"。
- 在输入中用 `$<app-slug>` 并附带 `app://<id>` 的 mention item 来点名调用某个 app。
- app 工具调用如有副作用，服务端会通过 `tool/requestUserInput` 发起审批（Accept / Decline / Cancel）；带 `destructive_hint` 的操作总是需要确认。

## 三、总体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      ChatGPT 后端 (OpenAI 托管)               │
│  ┌────────────────────────┐   ┌───────────────────────────┐ │
│  │ 应用目录 Directory API   │   │ codex_apps MCP server      │ │
│  │ /connectors/directory/… │   │ …/backend-api/wham/apps    │ │
│  │ (有哪些 app 可以装)      │   │ (你已连接的 app 有哪些工具) │ │
│  └───────────┬────────────┘   └───────────┬───────────────┘ │
└──────────────┼────────────────────────────┼─────────────────┘
               │ HTTPS + ChatGPT auth        │ Streamable HTTP MCP
               │                             │ + ChatGPT auth
┌──────────────┼─────────────────────────────┼─────────────────┐
│ Codex 本地   ▼                             ▼                 │
│  codex-chatgpt (HTTP 拉目录)      codex-mcp (MCP 连接管理)    │
│         │                             │  工具名规范化/磁盘缓存 │
│         └──────────┬──────────────────┘                      │
│                    ▼                                         │
│      codex-connectors (纯逻辑: merge/filter/policy/cache)     │
│                    ▼                                         │
│   codex-core (accessible 判定 / enable 策略 / 审批 reviewer)  │
│                    ▼                                         │
│   app-server `app/list` RPC ── TUI `/apps` ── 模型工具列表     │
└──────────────────────────────────────────────────────────────┘
```

两条数据通道各司其职：

1. **Directory API**（普通 HTTPS）：告诉客户端"市场上有哪些 app"，用于 `/apps` 列表展示与安装引导。
2. **`codex_apps` MCP server**（Streamable HTTP MCP）：返回当前账号已连接 app 的实际工具，是模型真正调用的通道。

一个 connector 是否 "accessible"，不是查目录得出的，而是**从 MCP server 实际返回的工具列表反推**：有带该 connector 标记的工具即视为已连接。

## 四、关键源码模块

| 模块 | 路径 | 职责 |
|---|---|---|
| `codex-connectors` crate | `codex-rs/connectors/` | 纯逻辑层：目录分页拉取与合并（`lib.rs`）、内存+磁盘双层缓存（`directory_cache.rs`）、目录/可用列表合并（`merge.rs`）、tool-suggest 过滤（`filter.rs`）、app 工具策略（`app_tool_policy.rs`）、accessible 归集（`accessible.rs`） |
| ChatGPT HTTP 客户端 | `codex-rs/chatgpt/src/connectors.rs` | 调用目录 API 的入口：`list_connectors()` / `list_all_connectors_with_options()`，要求 ChatGPT 登录，60s 超时 |
| 内置 MCP server 定义 | `codex-rs/codex-mcp/src/mcp/mod.rs` | `CODEX_APPS_MCP_SERVER_NAME = "codex_apps"`；`codex_apps_mcp_server_config()` 构造 URL/头/超时；`host_owned_codex_apps_enabled()` 开关判定 |
| Codex Apps 专属逻辑 | `codex-rs/codex-mcp/src/codex_apps.rs` | 工具/server info 磁盘缓存（按用户 key SHA1 分片）、工具名与命名空间规范化 |
| MCP 连接管理 | `codex-rs/codex-mcp/src/connection_manager.rs`、`rmcp_client.rs` | 为 `codex_apps` 注入 ChatGPT auth provider；Streamable HTTP 客户端构建 |
| 核心业务 | `codex-rs/core/src/connectors.rs` | accessible connector 判定与缓存、`with_app_enabled_state()` 启用策略、`mcp_approvals_reviewer()` 按 app 审批 |
| catalog 注册 | `codex-rs/core/src/mcp.rs`（~L136） | `apps_enabled` 时把 `codex_apps` 注册进 MCP server catalog |
| App Server RPC | `codex-rs/app-server/src/request_processors/apps_processor.rs` | `app/list`：并发加载目录+accessible、合并、分页、变更通知 |
| 协议类型 | `codex-rs/app-server-protocol/src/protocol/v2/apps.rs` | `AppInfo`、`AppsListParams/Response` 等（含 TS schema 生成） |

## 五、实现细节

### 1. 内置 server 的注册与 URL

- `core/src/mcp.rs`：构建 MCP catalog 时，若 `mcp_config.apps_enabled` 为真，以兼容性内置项注册 `codex_apps`；否则移除。
- URL 推导（`codex-mcp/src/mcp/mod.rs:432-453`）：
  - `https://chatgpt.com` / `https://chat.openai.com` → 自动补 `/backend-api` → `https://chatgpt.com/backend-api/wham/apps`
  - 其他 base → `<base>/api/codex/apps`
- 配置项：`StreamableHttp` transport、30s 启动超时、可选 `X-OpenAI-Product-Sku` 头（企业 SKU）。
- 另有 `hosted_plugin_runtime_mcp_server_config()` 指向 `…/ps/mcp`，服务于 ChatGPT 托管的 plugin runtime（plugin 也可携带 app connector）。

### 2. 鉴权

- 开关判定 `host_owned_codex_apps_enabled()`（`mcp/mod.rs:234`）：`config.apps_enabled && auth.uses_codex_backend()`。
- Bearer token 两条路（优先级从高到低）：
  1. 环境变量 `CODEX_CONNECTORS_TOKEN`（`mcp/mod.rs:423`，测试/特殊场景用）；
  2. ChatGPT 账号运行时 auth provider：`connection_manager.rs:157-199` 仅对 `codex_apps` 这个 server 注入 `codex_model_provider::auth_provider_from_auth`，用登录态 access token 请求。

### 3. 应用目录拉取与缓存

- 端点（`connectors/src/lib.rs:207-255`）：
  - `/connectors/directory/list?external_logos=true`（token 分页，循环取完）
  - 企业账号追加 `/connectors/directory/list_workspace?external_logos=true`（失败静默降级为空）
- 处理：过滤 `visibility == "HIDDEN"`；同 id 跨来源字段级合并（名称/描述/logo/branding/评分等，`merge_directory_app`）；生成安装链接 `https://chatgpt.com/apps/<slug>/<id>`；按名称排序。
- 缓存：进程内 static 缓存（TTL 1 小时，`CONNECTORS_CACHE_TTL`）+ 磁盘缓存，key = `(chatgpt_base_url, account_id, chatgpt_user_id, is_workspace_account)`，按用户隔离；磁盘缓存带 schema 版本，不匹配即失效删除。

### 4. 工具发现、命名与缓存

- `codex_apps` server 返回的每个 MCP tool 通过 meta key（`MCP_TOOL_CODEX_APPS_META_KEY`）携带 `connector_id` / `connector_name`；`synthetic_link` 标记的占位工具会被 app 列表过滤（`core/src/connectors.rs:496-508`）。
- 命名规范化（`codex-mcp/src/codex_apps.rs:69-141`）：
  - 剥离工具名中的 connector 前缀（`gmail_search_messages` → `search_messages`）；
  - 命名空间改写为 `codex_apps__<connector>`，模型最终看到形如 `codex_apps__gmail__search_messages`；
  - 名称经 `sanitize_name` 处理以满足 Responses API 的 `^[a-zA-Z0-9_-]+$` 约束。
- 磁盘缓存：`~/.codex/cache/codex_apps_tools/<sha1(user_key)>.json` 与 `cache/codex_apps_server_info/…`，启动时先读缓存（免等远端），连接成功后回写；schema 版本号（当前 tools 缓存 v4）不符即失效。

### 5. accessible 判定与列表合并

- `core/src/connectors.rs`：
  - `accessible_connectors_from_mcp_tools()` 把工具按 connector 归组得到"已连接"列表；进程内缓存 1 小时（按账号 key）。
  - 首次为空时最多等 server ready 30s（`CONNECTORS_READY_TIMEOUT_ON_EMPTY_TOOLS`）；支持 `force_refetch` 硬刷新。
- `chatgpt/src/connectors.rs::list_connectors()`：`tokio::join!` 并发取"全部目录"与"accessible"，`merge_connectors_with_accessible()` 合并——目录已加载完时会剔除不在目录中的 accessible 项（目录加载中则保留，避免闪烁）。
- `app-server` 的 `app/list` 走同样的合并逻辑并分页，数据变化时向客户端推送 list-updated 通知。

### 6. 启用策略与审批

配置（`config.toml`，详见官方 config-reference）：

```toml
[features]
apps = true                      # 实验性总开关

[apps._default]
enabled = true                   # 所有 app 的默认启用态
default_tools_approval_mode = "prompt"   # auto | prompt | approve
destructive_enabled = false      # destructive_hint 工具默认允许/拒绝
open_world_enabled = true        # open_world_hint 工具默认允许/拒绝
approvals_reviewer = "user"      # user | auto_review

[apps.gmail]
enabled = true
[apps.gmail.tools.send_email]
approval_mode = "approve"        # 工具级覆盖
```

源码对应：

- `with_app_enabled_state()`（`core/src/connectors.rs:510`）：用户层 `[apps]` 配置决定 `is_enabled`；管理端 requirements 中 `enabled = false` 的 app 强制禁用（管理策略优先）。
- `mcp_approvals_reviewer()`（`core/src/connectors.rs:548`）：仅对 `codex_apps` server 按 connector id 查 per-app / default 的 `approvals_reviewer`，且必须通过 requirements 校验才生效。
- 工具级策略在 `connectors/src/app_tool_policy.rs`（`AppToolPolicyEvaluator`）：综合 enabled / approval_mode / destructive_hint / open_world_hint 决定放行、询问或拒绝。

## 六、端到端流程小结

1. 用户以 ChatGPT 账号登录，`features.apps` 开启 → `codex_apps` 内置 MCP server 进入 catalog。
2. 会话启动：先读磁盘工具缓存立即可用，同时后台连接 `…/backend-api/wham/apps` 刷新工具列表并回写缓存。
3. 用户 `/apps`（或客户端调 `app/list`）：目录 API + accessible 列表合并展示；未连接的 app 给出 `chatgpt.com/apps/...` 安装链接，授权在网页端完成。
4. 模型调用 `codex_apps__<app>__<tool>` → 标准 MCP tool call → ChatGPT 后端代理执行第三方 API；有副作用时按 `[apps]` 策略触发审批。
5. 也可在输入里用 `$<app-slug>` 显式点名某个 app。

## 七、注意事项

- 该功能官方标注为 **experimental**（`features.apps`）。
- 仅 ChatGPT 登录可用；API key 模式下 `codex_apps` 会从有效 server 列表中移除。
- 所有缓存（目录、工具、accessible）均按 账号+base_url 隔离，切换账号不会串数据。
- 本地 `is_enabled` 与服务端 `is_accessible` 是两个独立维度：前者由 config 控制，后者由是否完成 OAuth 连接决定。
