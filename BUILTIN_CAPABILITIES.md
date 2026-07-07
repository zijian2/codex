# Codex 内置工具、插件、MCP 与 Skills 说明

本文档面向维护者和二次开发者，说明本仓库中“工具、插件、MCP、Skills”的实现位置、内置能力、配置方式、扩展方式，以及哪些地方可以改、是否推荐改。

源码入口主要集中在：

- `codex-rs/core/src/tools/`：模型可见工具的规划、注册、分发与执行。
- `codex-rs/tools/src/`：工具抽象、工具参数和工具输出类型。
- `codex-rs/codex-mcp/src/`：MCP 连接、工具同步、资源读取与调用管理。
- `codex-rs/config/src/mcp_types.rs`：用户和插件 MCP 配置结构。
- `codex-rs/plugin/src/` 与 `codex-rs/core-plugins/src/`：插件 manifest、安装、加载、缓存和 marketplace。
- `codex-rs/core-skills/src/` 与 `codex-rs/skills/src/`：Skills 的加载、渲染、注入和系统 Skills 资产。
- `codex-rs/hooks/src/`：Hook 事件、发现、执行和输入输出协议。
- `codex-rs/app-server/src/extensions.rs`：app-server 启动时安装的内置扩展。

## 运行时关系

Codex 发送请求给模型前，会构造一组工具规格。工具来源不是单一文件，而是多个层次合并：

1. 核心工具：shell、apply_patch、图片查看、计划、权限、上下文、MCP 资源等。
2. 扩展工具：web search、image generation、goals、memories、skills 等，由 app-server 安装。
3. MCP 工具：来自用户配置、插件配置或运行时覆盖的 MCP server。
4. 插件能力：插件本身不是直接“工具运行时”，它是一个分发包，可以声明 skills、MCP server、apps/channel connector、hooks。
5. Skills：不是传统函数工具，而是可被发现和读取的指令包；在当前会话里，系统也会把可用 skill 摘要注入上下文。
6. Hosted tools：由模型服务端提供的 hosted web search / image generation 等能力。

核心工具抽象在 `ToolExecutor` 中，每个工具暴露：

- `tool_name()`：工具名。
- `spec()`：发给模型的 JSON schema / tool spec。
- `exposure()`：直接暴露、延迟发现、仅模型直接可见、隐藏。
- `handle()`：实际执行逻辑。

工具规划在 `codex-rs/core/src/tools/spec_plan.rs` 里完成，分发在 `codex-rs/core/src/tools/registry.rs`。

本文档的工具清单按以下源码入口逐项核对：

- `codex-rs/core/src/tools/spec_plan.rs`：`add_tool_sources` 是核心工具入口，顺序是 shell、MCP resource、core utility、collaboration、MCP runtime、extension、dynamic、hosted。
- `codex-rs/core/src/tools/handlers/**`：每个核心工具的 `tool_name()` 和 `spec()`。
- `codex-rs/ext/**/src/**`：每个 extension 的 `ToolContributor::tools()` 和工具名。
- `codex-rs/codex-mcp/src/mcp/mod.rs`、`codex-rs/core/src/mcp.rs`、`codex-rs/ext/mcp/src/**`：内置 / 兼容 MCP server 贡献。
- `codex-rs/core-plugins/src/manifest.rs`、`loader.rs`、`codex-rs/plugin/src/manifest.rs`：插件 manifest 字段、apps、hooks、MCP、skills 的真实解析形状。
- `codex-rs/skills/src/assets/samples/`：仓库随包内置的 system skill 资产。

## 内置模型工具清单

下表列的是从 `add_tool_sources` 和各 handler `tool_name()` 逐项确认的工具。部分工具受 feature、运行模式、模型能力、provider 配置或 app-server 是否启用影响。

| 工具 | 作用 | 主要实现位置 | 配置 / 条件 | 是否建议修改 |
| --- | --- | --- | --- | --- |
| `exec_command` | Unified exec shell 工具，执行命令。 | `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs`、`shell_spec.rs` | `tool_environment_mode.has_environment()` 且 `shell_type_for_model_and_features(...) == UnifiedExec`。 | 平台 fork 可以改 shell 协议；业务扩展不建议改。 |
| `write_stdin` | 向已有 Unified exec PTY / 运行中命令写入 stdin 或轮询输出。 | `codex-rs/core/src/tools/handlers/unified_exec/write_stdin.rs`、`shell_spec.rs` | 仅 UnifiedExec 分支加入。 | 平台 fork 可以改；普通能力不建议改。 |
| `shell_command` | 旧 shell 工具。UnifiedExec 时仍隐藏注册用于兼容 dispatch。 | `codex-rs/core/src/tools/handlers/shell/shell_command.rs`、`shell_spec.rs` | shell type 为 Default/Local/ShellCommand 时直接暴露；UnifiedExec 时 `Hidden`。 | 可改但高风险，影响 hooks、权限和执行协议。 |
| `list_mcp_resources` | 列出 MCP server resources。 | `codex-rs/core/src/tools/handlers/mcp_resource/list_mcp_resources.rs` | `context.mcp_tools.is_some()` 时加入。 | 不建议改；新增资源应改 MCP server。 |
| `list_mcp_resource_templates` | 列出 MCP resource templates。 | `codex-rs/core/src/tools/handlers/mcp_resource/list_mcp_resource_templates.rs` | `context.mcp_tools.is_some()` 时加入。 | 不建议改。 |
| `read_mcp_resource` | 读取 MCP resource。 | `codex-rs/core/src/tools/handlers/mcp_resource/read_mcp_resource.rs` | `context.mcp_tools.is_some()` 时加入。 | 不建议改。 |
| `update_plan` | 更新任务计划。 | `codex-rs/core/src/tools/handlers/plan.rs`、`plan_spec.rs` | `add_core_utility_tools` 总是加入。 | 可改 UI/协作语义；业务扩展不建议改。 |
| `request_user_input` | 请求用户输入。 | `codex-rs/core/src/tools/handlers/request_user_input.rs`、`request_user_input_spec.rs` | `config.experimental_request_user_input_enabled` 时加入，且 `DirectModelOnly`。 | 平台可改交互协议；普通扩展不建议改。 |
| `request_permissions` | 请求用户批准权限升级或外部动作。 | `codex-rs/core/src/tools/handlers/request_permissions.rs`、`shell_spec.rs` | `Feature::RequestPermissionsTool`。 | 可以改但安全敏感。 |
| `new_context` | 请求新的上下文窗口。 | `codex-rs/core/src/tools/handlers/new_context_window.rs`、`new_context_window_spec.rs` | `Feature::TokenBudget` 且 `Feature::AutoCompaction`，`DirectModelOnly`。 | 平台可改上下文策略；普通扩展不建议改。 |
| `get_context_remaining` | 查询剩余上下文预算。 | `codex-rs/core/src/tools/handlers/get_context_remaining.rs`、`get_context_remaining_spec.rs` | `Feature::TokenBudget`。 | 平台可改。 |
| `clock.curr_time` | 获取当前时间。 | `codex-rs/core/src/tools/handlers/current_time.rs` | 命名空间为 `clock`，函数名为 `curr_time`。 | 不建议改。 |
| `sleep` | 等待一段时间。 | `codex-rs/core/src/tools/handlers/sleep.rs` | `Feature::SleepTool`。 | 一般不改。 |
| `list_available_plugins_to_install` | 列出可安装插件/connector 候选。 | `codex-rs/core/src/tools/handlers/list_available_plugins_to_install.rs` | `tool_suggest_enabled` 且候选不空，presentation 为 `ListTool`。 | 自研 marketplace 时可改。 |
| `request_plugin_install` | 请求安装插件/connector。 | `codex-rs/core/src/tools/handlers/request_plugin_install.rs` | `tool_suggest_enabled` 且候选不空。 | 自研插件安装流时可改。 |
| `apply_patch` | 以补丁形式编辑文件。 | `codex-rs/core/src/tools/handlers/apply_patch.rs`、`apply_patch_spec.rs` | 有 environment，且 `model_info.apply_patch_tool_type.is_some()`。 | 核心编辑能力，谨慎改。 |
| `test_sync_tool` | 测试同步工具。 | `codex-rs/core/src/tools/handlers/test_sync.rs`、`test_sync_spec.rs` | `model_info.experimental_supported_tools` 包含 `test_sync_tool`。 | 测试/实验用途，不作为产品依赖。 |
| `view_image` | 读取本地图片供模型视觉检查。 | `codex-rs/core/src/tools/handlers/view_image.rs`、`view_image_spec.rs` | 有 environment 时加入。 | 平台可改图片读取策略。 |
| `exec` / `wait` | code mode 协议里的执行与等待工具名。 | `codex-rs/code-mode-protocol/src/lib.rs` | code mode 下使用。 | 不建议改，属于协议面。 |
| `multi_agent_v1.spawn_agent` | 创建子 agent。 | `codex-rs/core/src/tools/handlers/multi_agents/spawn.rs` | multi-agent v1 功能开启且未超过 spawn depth。 | 自研 agent 框架可改。 |
| `multi_agent_v1.send_input` | 给 v1 子 agent 发送输入。 | 同上 | 需要已有 agent。 | 同上。 |
| `multi_agent_v1.resume_agent` | 恢复 v1 子 agent。 | 同上 | 需要已有 agent。 | 同上。 |
| `multi_agent_v1.wait_agent` | 等待 v1 子 agent。 | 同上 | 需要已有 agent。 | 同上。 |
| `multi_agent_v1.close_agent` | 关闭 v1 子 agent。 | 同上 | 需要已有 agent。 | 同上。 |
| `spawn_agent` | multi-agent v2 创建 agent。 | `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs` | `multi_agent_version == V2`；可由 `multi_agent_v2.tool_namespace` 改为命名空间形式。 | 自研 agent 框架很可能要改。 |
| `send_message` | 给 v2 agent 发送消息。 | 同上 | 需要已有 agent。 | 不建议改。 |
| `followup_task` | 给 v2 agent 下发后续任务。 | 同上 | 需要已有 agent。 | 不建议改。 |
| `wait_agent` | 等待 v2 agent。 | 同上 | 需要已有 agent。 | 不建议改。 |
| `list_agents` | 列出 agent。 | 同上 | multi-agent v2。 | 不建议改。 |
| `interrupt_agent` | 中断 agent。 | 同上 | multi-agent v2。 | 不建议改。 |
| `spawn_agents_on_csv` | 基于 CSV 批量创建 agent job。 | `codex-rs/core/src/tools/handlers/agent_jobs/spawn_agents_on_csv.rs` | `Feature::SpawnCsv` 且 collaboration tools enabled。 | 自研批量 agent 可改。 |
| `report_agent_job_result` | agent job worker 上报结果。 | `codex-rs/core/src/tools/handlers/agent_jobs/report_agent_job_result.rs` | `Feature::SpawnCsv` 且当前 session source 是 `agent_job:` 子 agent。 | 自研批量 agent 可改。 |
| MCP runtime tools | 每个 MCP server 暴露的 tool 会被转换成 Codex 工具。 | `codex-rs/core/src/tools/handlers/mcp.rs`、`codex-rs/codex-mcp/src/tools.rs` | `context.mcp_tools` 直接暴露；`context.deferred_mcp_tools` 作为 deferred tools。 | 推荐改 MCP server；框架层才改 handler。 |
| Dynamic tools | app-server 或运行时注入的 function / namespace tools。 | `codex-rs/core/src/tools/handlers/dynamic.rs` | `context.dynamic_tools` 非空。名称不固定。 | 推荐作为平台扩展点。 |
| Extension tools | extension `ToolContributor` 返回的工具。 | `codex-rs/core/src/tools/handlers/extension_tools.rs` | app-server 安装 extension 后由 registry 收集。 | 自研平台推荐通过 extension registry 替换。 |
| `tool_search` | 延迟工具发现。 | `codex-rs/core/src/tools/handlers/tool_search.rs`、`codex-rs/tools/src/tool_discovery.rs` | `search_tool_enabled` 且存在 deferred tool search info。 | 自研插件/connector 发现可改。 |

## 内置扩展与扩展工具

app-server 在 `codex-rs/app-server/src/extensions.rs` 中安装一组扩展。扩展可以注册模型工具，也可以注册 MCP server contributor、prompt/context contributor、thread lifecycle contributor 等，所以“扩展”不一定等于“模型工具”。

| 扩展 / 工具 | 作用 | 安装位置 | 配置 / 条件 | 是否建议修改 |
| --- | --- | --- | --- | --- |
| `codex_guardian` | guardian / approval review 相关 extension；不是直接模型工具。 | `codex-rs/ext/guardian/src/lib.rs`、`app-server/src/extensions.rs` | app-server 总是安装。 | 自研安全审查 agent 时可替换。 |
| MCP extension | 贡献 hosted plugin runtime MCP 和 selected executor plugin MCP；不是直接 function tool。 | `codex-rs/ext/mcp/src/lib.rs`、`executor_plugin.rs` | app-server 安装 `install` 和 `install_executor_plugins`。 | 自研云端/插件运行时通常要改。 |
| `web.run` | standalone web search 工具。 | `codex-rs/ext/web-search/src/extension.rs`、`tool.rs` | provider 是 OpenAI，`web_search_mode != Disabled`；core 还要求 standalone web search 可见且 web mode 未关闭。 | 自研搜索建议替换为自有 extension 或 MCP。 |
| `image_gen.imagegen` | standalone image generation 工具。 | `codex-rs/ext/image-generation/src/extension.rs`、`tool.rs` | provider 是 OpenAI、当前 auth 使用 Codex backend；core 还要求 image generation runtime、namespace tools、feature 条件满足。 | 自研图像能力可替换。 |
| `get_goal` | 获取当前目标状态。 | `codex-rs/ext/goal/src/extension.rs`、`tool.rs`、`spec.rs` | state DB 可用，`Feature::Goals` 开启，且 goal runtime `tools_visible()`。 | 自研目标/任务系统可改。 |
| `create_goal` | 创建目标。 | 同上 | 同上。 | 可改。 |
| `update_goal` | 更新目标状态。 | 同上 | 同上。 | 可改。 |
| `memories.add_ad_hoc_note` | 添加长期记忆笔记。 | `codex-rs/ext/memories/src/extension.rs`、`tools/ad_hoc_note.rs` | `Feature::MemoryTool`、`memories.use_memories`、`memories.dedicated_tools`。 | 自研记忆系统可替换 backend 或工具。 |
| `memories.list` | 列出记忆。 | `codex-rs/ext/memories/src/tools/list.rs` | 同上。 | 可改。 |
| `memories.read` | 读取记忆。 | `codex-rs/ext/memories/src/tools/read.rs` | 同上。 | 可改。 |
| `memories.search` | 搜索记忆。 | `codex-rs/ext/memories/src/tools/search.rs` | 同上。 | 可改。 |
| Skills extension | 注入可用 skills 摘要和显式 skill prompt；在有 orchestrator provider 时提供 tools。 | `codex-rs/ext/skills/src/extension.rs` | `include_skill_instructions`、`bundled_skills_enabled`、`orchestrator_skills_enabled` 等配置。 | 自研 skill registry 时建议替换 provider。 |
| `skills.list` | 列出 orchestrator authority 下的 skills。 | `codex-rs/ext/skills/src/tools/list.rs` | 有 orchestrator provider 且当前线程启用 orchestrator skills。 | 可改。 |
| `skills.read` | 读取 orchestrator skill 资源。 | `codex-rs/ext/skills/src/tools/read.rs` | 同上。 | 可改。 |

另外还有 hosted tool spec，它们不是 extension `ToolContributor` 返回的本地工具，而是在 `hosted_model_tool_specs` 中追加：

- hosted web search：provider 支持 hosted search、没有 standalone `web.run` 可用、`web_search_mode` 允许时加入。
- hosted image generation：`Feature::ImageGeneration` 等条件满足，且 standalone `image_gen.imagegen` 不可用时加入。

## MCP

MCP 是推荐的一等扩展方式。只要能力可以放在外部进程、HTTP 服务或已有 MCP server 中，就优先用 MCP。

### MCP 做什么

MCP server 可以提供：

- Tools：被 Codex 转换成模型工具调用。
- Resources：通过 `list_mcp_resources` / `read_mcp_resource` 被读取。
- Resource templates：通过 `list_mcp_resource_templates` 暴露模板化资源。

Codex 负责：

- 按配置启动或连接 MCP server。
- 同步工具列表。
- 规范化工具名和参数 schema。
- 根据 approval mode 决定工具调用是否需要用户批准。
- 把 MCP tool call 分发到对应 server。

### 用户配置

MCP 用户配置在 Codex config 中，结构来自 `codex-rs/config/src/mcp_types.rs`。典型 stdio 配置：

```toml
[mcp_servers.docs]
command = "docs-mcp"
args = ["--root", "/path/to/docs"]
env = { DOCS_TOKEN = "..." }
cwd = "/path/to/project"
enabled = true
required = false
startup_timeout_sec = 10
tool_timeout_sec = 60
enabled_tools = ["search", "read"]
default_tools_approval_mode = "never"

[mcp_servers.docs.tools.delete]
approval_mode = "always"
```

典型 HTTP 配置：

```toml
[mcp_servers.internal]
url = "https://mcp.example.com/mcp"
http_headers = { "x-client" = "codex" }
env_http_headers = { "authorization" = "INTERNAL_AUTH_HEADER" }
bearer_token_env_var = "INTERNAL_MCP_TOKEN"
enabled = true
```

常见字段：

- `command` / `args` / `env` / `env_vars` / `cwd`：stdio server。
- `url` / `http_headers` / `env_http_headers` / `bearer_token_env_var`：HTTP server。
- `environment_id`：绑定环境。
- `startup_timeout_sec` / `startup_timeout_ms`：启动超时。
- `tool_timeout_sec`：工具调用超时。
- `enabled` / `required`：启用和强依赖。
- `supports_parallel_tool_calls`：是否允许并行调用。
- `default_tools_approval_mode`：默认审批策略。
- `enabled_tools` / `disabled_tools`：工具白名单 / 黑名单。
- `tools.<tool>.approval_mode`：单工具审批策略。
- `scopes`、`oauth`、`oauth_resource`：OAuth / scope 相关配置。

### 插件内声明 MCP

插件可以在 `.codex-plugin/plugin.json` 中声明 `mcpServers`。它可以是内联对象，也可以指向一个相对路径，默认文件通常是 `.mcp.json`。

```json
{
  "name": "demo-plugin",
  "version": "0.1.0",
  "mcpServers": "./.mcp.json"
}
```

`.mcp.json`：

```json
{
  "mcpServers": {
    "demo": {
      "command": "demo-mcp",
      "args": ["--stdio"],
      "enabled": true
    }
  }
}
```

插件路径要求是相对插件根目录的路径，通常以 `./` 开头，不允许跳出插件目录。

### 内置 / 兼容 MCP server

这里的“内置 MCP”不是指写死了一堆本地业务工具，而是 Codex 在运行时会自动注册或覆盖的 MCP server 来源。

| 内置 MCP 来源 | server 名 | 作用 | 主要实现位置 | 何时启用 | 是否可改 |
| --- | --- | --- | --- | --- | --- |
| Apps / Connectors 兼容 MCP | `codex_apps` | 把 ChatGPT Apps / Connectors 暴露为一组 MCP tools。也是 channel/app connector 插件最终落到的 MCP server 名。 | `codex-rs/core/src/mcp.rs`、`codex-rs/codex-mcp/src/mcp/mod.rs`、`codex-rs/codex-mcp/src/codex_apps.rs` | `apps_enabled` 为 true，并且通过 auth gating 后才成为 effective server。 | 可以改。做自有云端 connector 平台时，这是最可能需要替换的点。 |
| Hosted plugin runtime MCP | 仍注册为 `codex_apps` | 通过 plugin-service 的 hosted MCP runtime 提供 app/plugin 工具，URL 形如基于 `chatgpt_base_url` 的 `/ps/mcp`。 | `codex-rs/ext/mcp/src/lib.rs`、`codex-rs/codex-mcp/src/mcp/mod.rs` | app-server 安装 `codex_mcp_extension::install`，且 `Feature::Apps` 开启。 | 可以改。自研云端 agent 平台时，通常要替换 base URL、鉴权、路由或整个 contributor。 |
| Legacy apps MCP config | `codex_apps` | 兼容路径，URL 形如基于 `chatgpt_base_url` 的 `/backend-api/wham/apps` 或 `/api/codex/apps`。 | `codex-rs/core/src/mcp.rs`、`codex-rs/codex-mcp/src/mcp/mod.rs` | `McpManager::runtime_config` 中根据 `apps_enabled` 注册 compatibility server；随后 extension overlay 可能覆盖它。 | 可以改。若不用 OpenAI/ChatGPT connectors，应改或关闭。 |
| Selected executor plugin MCP | 插件声明的 server 名 | 线程选中的 executor plugin 可以贡献自己的 MCP servers。 | `codex-rs/ext/mcp/src/executor_plugin.rs`、`provider.rs` | app-server 安装 `install_executor_plugins`，且线程初始化里有 selected capability roots。 | 可以改。自定义 agent 框架若有自己的插件选择/执行环境，需要改这里或实现新的 contributor。 |
| 用户 / 项目 / 插件 MCP | 用户或插件配置的名字 | 普通 MCP server。 | `codex-rs/config/src/mcp_types.rs`、`codex-rs/core-plugins/src/loader.rs`、`codex-rs/codex-mcp/src/catalog.rs` | config 或 plugin manifest 中声明。 | 推荐通过配置/插件新增，而不是改核心。 |

`codex_apps` 的关键细节：

- 常量名在 `codex-rs/codex-mcp/src/mcp/mod.rs`：`CODEX_APPS_MCP_SERVER_NAME = "codex_apps"`。
- bearer token 环境变量是 `CODEX_CONNECTORS_TOKEN`。
- URL 由 `chatgpt_base_url` 推导：
  - apps 兼容路径：`codex_apps_mcp_server_config(...)`
  - hosted plugin runtime：`hosted_plugin_runtime_mcp_server_config(...)`
- `host_owned_codex_apps_enabled` 会检查 `apps_enabled` 和当前 auth 是否使用 Codex backend。
- `McpCatalogBuilder` 会处理 config、plugin、selected plugin、compatibility、extension 之间的覆盖和冲突。

### 修改建议

- 推荐：新增普通外部能力时，写 MCP server，并通过用户 config 或插件 `mcpServers` 接入。
- 推荐：基于 Codex 做自己的 agent 平台时，把云端相关 MCP contributor 明确 fork 掉，包括 `codex_apps` URL、token、auth gating、apps feature 开关和 hosted plugin runtime contributor。
- 谨慎：修改 `codex-rs/codex-mcp/src/connection_manager.rs`，适合改变 MCP 工具同步、运行时覆盖、工具缓存、auth 状态或调用管理。
- 谨慎：修改 `codex-rs/core/src/mcp.rs`，这是 config/plugin/extension MCP catalog 的合并点。
- 不推荐：为了某个业务工具去改 `codex-rs/core/src/tools/handlers/mcp.rs` 或核心工具规划，除非是修 Codex 的 MCP 框架本身。

## Plugins

插件是能力分发包。它本身不是单一运行时接口，而是把 skills、MCP、apps/channel connector、hooks 和元数据打包给 Codex。

### 插件 manifest

标准位置：

- `.codex-plugin/plugin.json`
- 兼容位置：`.claude-plugin/plugin.json`

核心 parser 支持的主要字段：

- `name`
- `version`
- `description`
- `keywords`
- `skills`
- `mcpServers`
- `apps`
- `hooks`
- `interface`

`interface` 中实际解析的字段包括：

- `displayName`
- `shortDescription`
- `longDescription`
- `developerName`
- `category`
- `capabilities`
- `websiteURL` / `websiteUrl`
- `privacyPolicyURL` / `privacyPolicyUrl`
- `termsOfServiceURL` / `termsOfServiceUrl`
- `defaultPrompt`
- `brandColor`
- `composerIcon`
- `logo`
- `logoDark`
- `screenshots`

示例：

```json
{
  "name": "acme-tools",
  "version": "0.1.0",
  "description": "ACME internal tools for Codex",
  "keywords": ["acme", "internal"],
  "skills": "./skills",
  "mcpServers": "./.mcp.json",
  "apps": "./.app.json",
  "hooks": "./hooks/hooks.json"
}
```

默认约定：

- `skills` 未显式声明但存在 `skills/` 目录时，可作为插件 skills 来源。
- `mcpServers` 常用默认文件是 `.mcp.json`。
- `apps` 常用默认文件是 `.app.json`。
- `hooks` 常用默认文件是 `hooks/hooks.json`。

### 插件做什么

插件可以：

- 打包 MCP server 配置。
- 打包一个或多个 skill。
- 声明 app/channel connector。
- 声明 hooks。
- 进入 marketplace / local store，被安装、缓存、同步和卸载。

### 仓库内置插件

本仓库内置的是插件框架、安装/加载/同步逻辑和测试样例；没有一组实际随仓库发布的业务插件目录作为固定内置插件包。插件 marketplace 和远程 curated 插件是在运行时同步、缓存和安装的。

### 修改建议

- 推荐：新增业务能力时创建插件包，而不是改核心。
- 推荐：插件里组合 `skills + mcpServers + hooks + apps`，让安装者一次获得完整能力。
- 可以修改：`codex-rs/core-plugins/src/loader.rs`、`manifest.rs`、`store.rs` 等用于改变插件系统能力。
- 不推荐：为一个具体业务插件修改插件框架。除非你要新增 manifest 语义或修 framework bug。

## Skills

Skill 是一个带 `SKILL.md` 的指令包。它不是 shell 函数，也不是 MCP tool；它用于告诉模型“什么时候使用这套流程、要读哪些参考文件、如何执行”。

### Skill 做什么

Skill 适合：

- 固化某类任务流程。
- 提供领域知识、约束和模板。
- 引导模型读取额外参考资料。
- 封装“该如何使用某个外部工具或 API”的操作手册。

Skill 不适合：

- 执行需要强一致性的外部动作。
- 暴露结构化函数调用。
- 替代 MCP server。

### 系统内置 Skills

`codex-rs/skills/src/assets/samples/` 中的系统 skills 会作为样例/系统资产安装。当前仓库内置系统 skills 包括：

| Skill | 作用 | 仓库内文件 |
| --- | --- | --- |
| `imagegen` | 指导何时使用图片生成/编辑能力；包含 CLI、网络、image API、prompting 参考和图片处理脚本。 | `SKILL.md`、`agents/openai.yaml`、`references/`、`scripts/`、`assets/`、`LICENSE.txt` |
| `openai-docs` | 查询 OpenAI / Codex 官方文档时的流程和来源约束；包含最新模型、prompting、升级参考和脚本。 | `SKILL.md`、`agents/openai.yaml`、`references/`、`scripts/`、`assets/`、`LICENSE.txt` |
| `plugin-creator` | 创建 Codex 插件目录、manifest 和 marketplace 条目；包含 plugin spec、安装更新说明和校验/创建脚本。 | `SKILL.md`、`agents/openai.yaml`、`references/`、`scripts/`、`assets/` |
| `skill-creator` | 创建或更新 skill；包含 openai.yaml 参考、初始化和校验脚本。 | `SKILL.md`、`agents/openai.yaml`、`references/`、`scripts/`、`assets/`、`license.txt` |
| `skill-installer` | 安装 curated 或 GitHub 来源的 skills；包含 GitHub 安装和列表脚本。 | `SKILL.md`、`agents/openai.yaml`、`scripts/`、`assets/`、`LICENSE.txt` |

本仓库还包含 repo-local skills，位于 `.codex/skills/`，例如：

- `babysit-pr`
- `code-review`
- `code-review-breaking-changes`
- `code-review-change-size`
- `code-review-context`
- `code-review-testing`
- `codex-bug`
- `codex-issue-digest`
- `codex-pr-body`
- `path-types`
- `pushing-ci-changes`
- `remote-tests`
- `test-tui`
- `update-v8-version`

这些是当前仓库工作流相关的 skills，不是通用运行时工具。

### Skill 配置和集成

常见目录结构：

```text
skills/
  my-skill/
    SKILL.md
    references/
      details.md
    scripts/
      helper.sh
```

插件中声明：

```json
{
  "name": "acme-skills",
  "version": "0.1.0",
  "skills": "./skills"
}
```

`SKILL.md` 应包含：

- skill 名称和触发条件。
- 使用流程。
- 需要读取的参考文件。
- 可复用脚本或模板的位置。
- 边界条件：什么时候不要使用该 skill。

### 修改建议

- 推荐：新增领域流程时优先新增 skill。
- 推荐：把 skill 放进插件，和对应 MCP server 一起分发。
- 可以修改：系统 skills 资产，但只适合修改 Codex 自带能力。
- 不推荐：把需要确定性执行的逻辑只写进 skill。执行逻辑应放 MCP / CLI / 服务端，skill 只负责教模型如何使用。

## Hooks

Hook 是生命周期事件拦截器。它可以在工具调用前后、压缩前后、会话开始、子 agent 启停等阶段运行命令或触发 agent/prompt handler。

### Hook 事件

源码中的事件名包括：

- `PreToolUse`
- `PermissionRequest`
- `PostToolUse`
- `PreCompact`
- `PostCompact`
- `SessionStart`
- `UserPromptSubmit`
- `SubagentStart`
- `SubagentStop`
- `Stop`

其中支持 matcher 的事件包括：

- `PreToolUse`
- `PermissionRequest`
- `PostToolUse`
- `PreCompact`
- `PostCompact`
- `SessionStart`
- `SubagentStart`
- `SubagentStop`

### Hook handler 类型

当前配置模型中可见的 handler 类型包括：

- `command`：运行本地命令。可带 `command`、`commandWindows` / `command_windows`、`timeout`、`async`、`statusMessage` 等字段。
- `prompt`：prompt 类型 hook。
- `agent`：agent 类型 hook。

最常用、最稳定的是 `command`。

### 用户配置 Hook

TOML 形状：

```toml
[[hooks.PreToolUse]]
matcher = "exec_command"

[[hooks.PreToolUse.hooks]]
type = "command"
command = "./hooks/check-command.sh"
timeout = 5
```

JSON 形状：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "exec_command",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/check-command.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "exec_command",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/audit-output.sh"
          }
        ]
      }
    ]
  }
}
```

### Hook 类型插件接入

推荐把 hook 放进插件，而不是要求用户手写全局配置。

插件结构：

```text
acme-policy-plugin/
  .codex-plugin/
    plugin.json
  hooks/
    hooks.json
    check-command.sh
    audit-output.sh
```

`.codex-plugin/plugin.json`：

```json
{
  "name": "acme-policy",
  "version": "0.1.0",
  "description": "ACME command policy hooks",
  "hooks": "./hooks/hooks.json"
}
```

`hooks/hooks.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "exec_command",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/check-command.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "exec_command",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/audit-output.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

插件 `hooks` 字段也可以是：

- 单个路径：`"hooks": "./hooks/hooks.json"`
- 路径数组：`"hooks": ["./hooks/pre.json", "./hooks/post.json"]`
- 内联对象：`"hooks": { "hooks": { ... } }`
- 内联对象数组。

修改建议：

- 推荐：策略校验、审计、命令重写、阻止危险操作，用 hook 插件。
- 推荐：hook 脚本保持小而确定，复杂逻辑放到 MCP server 或独立 CLI。
- 谨慎：`PreToolUse` 可以影响工具是否执行，属于安全敏感路径。
- 不推荐：为了单个团队策略修改 `codex-rs/hooks/src/`。团队策略应作为插件分发。

## Apps / Channel Connector 插件

源码里没有一个正式命名为 `channel plugin` 的独立插件类型。最接近“channel 类型插件”的机制是插件里的 `apps` 声明：插件声明 app connector，Codex 通过 apps / connector 体系把外部 app 能力接入运行时。

如果你说的 channel 是“某个外部应用、ChatGPT connector、业务通道或 app channel”，推荐按 apps connector 插件接入。

### Channel / Apps 插件结构

```text
acme-channel-plugin/
  .codex-plugin/
    plugin.json
  .app.json
  .mcp.json
  skills/
    acme-channel/
      SKILL.md
```

`.codex-plugin/plugin.json`：

```json
{
  "name": "acme-channel",
  "version": "0.1.0",
  "description": "ACME channel connector for Codex",
  "apps": "./.app.json",
  "mcpServers": "./.mcp.json",
  "skills": "./skills"
}
```

`.app.json`：

```json
{
  "apps": {
    "acme-channel": {
      "id": "connector_acme_channel",
      "category": "Productivity"
    }
  }
}
```

`.mcp.json`：

```json
{
  "mcpServers": {
    "acme-channel": {
      "command": "acme-channel-mcp",
      "args": ["--stdio"],
      "enabled": true
    }
  }
}
```

这里的 `apps` 声明负责把插件与 connector id 关联；实际工具能力通常仍由 MCP server 或 app connector 后端提供。源码中的 `PluginAppFile` 只解析：

- 顶层 `apps`
- 每个 app 的 `id`
- 每个 app 的可选 `category`

插件 loader 会读取 `.app.json`，得到 app 名称、connector id 和 category；空 `id` 会被忽略。

### Channel 插件推荐方式

- 如果只是外部 API 工具：只做 MCP。
- 如果需要在 Codex UI / app 列表里作为一个 connector 出现：做 `apps` 声明。
- 如果还需要模型知道怎么用这个通道：同时附带 skill。
- 如果需要调用前后安全策略：再附带 hooks。

不推荐发明新的“channel plugin” manifest 字段，除非你准备修改插件 schema、loader、app-server 协议、UI 和安装流程。

## 新增能力时怎么选

| 需求 | 推荐形式 | 理由 |
| --- | --- | --- |
| 暴露一个外部 API / 数据库 / SaaS 操作 | MCP server | 结构化工具调用、可审批、可隔离、可独立升级。 |
| 教模型按固定流程完成任务 | Skill | 低成本、可读、适合说明和流程。 |
| 同时分发 MCP、skills、hooks、apps | Plugin | 一次安装，版本化，便于 marketplace / cache 管理。 |
| 拦截工具调用、做审计或策略控制 | Hook plugin | 生命周期原生支持，适合 PreToolUse / PostToolUse。 |
| 接入 app/channel connector | Plugin `apps` + MCP + skill | apps 负责 connector 声明，MCP 负责工具，skill 负责使用说明。 |
| 修改 Codex 自身基础行为 | Core tool / extension | 只有改平台能力时才走这条路。 |
| 需要长期、本地或服务端状态 | Extension 或 MCP server | 取决于状态属于 Codex 还是外部系统。 |

## 是否可以修改内置能力

可以。前面的“不建议改”主要针对“新增一个业务工具”的场景；如果目标是基于 Codex 开发自己的 agent 框架或私有云端平台，部分内置扩展、Skills 和 MCP 不但可以改，而且通常应该改。

需要优先审视的云端 / 平台耦合点：

| 模块 | 为什么可能要改 | 主要位置 | 建议 |
| --- | --- | --- | --- |
| Hosted apps / connectors MCP | 默认面向 ChatGPT / Codex backend 的 Apps connector。自有平台通常有自己的 connector 网关、token、租户和权限模型。 | `codex-rs/ext/mcp/src/lib.rs`、`codex-rs/codex-mcp/src/mcp/mod.rs`、`codex-rs/core/src/mcp.rs` | 推荐改或替换 contributor。 |
| Web search extension | 默认 web 搜索能力可能依赖 OpenAI/hosted web 工具和当前认证。 | `codex-rs/ext/web-search/`、`codex-rs/app-server/src/extensions.rs` | 自研搜索/知识库建议换成自有 MCP 或自有 extension。 |
| Image generation extension | 默认图片生成走当前内置 image generation extension。 | `codex-rs/ext/image-generation/` | 如果你的平台不用 OpenAI 图像能力，应替换或关闭。 |
| Skills extension providers | 当前组合 executor provider 与 orchestrator provider。 | `codex-rs/ext/skills/`、`codex-rs/app-server/src/extensions.rs` | 自研 skill registry / marketplace 时可以替换 provider。 |
| System bundled skills | 默认系统 skills 包括 OpenAI docs、plugin creator、skill creator 等。 | `codex-rs/skills/src/assets/samples/` | 自有发行版可以删减、替换或增加自己的系统 skills。 |
| Plugin marketplace / curated sync | 默认会同步/读取 curated plugin marketplace。 | `codex-rs/core-plugins/src/startup_sync.rs`、`installed_marketplaces.rs`、`marketplace*.rs` | 自有 marketplace 应替换来源和签名/缓存策略。 |
| Auth manager / backend client | 多个 extension 会依赖 Codex auth 或 backend URL。 | `codex-rs/login/`、`codex-rs/backend-client/`、各 extension install 参数 | 自有云端必须明确替换认证和 API base URL。 |
| Guardian / approval review | MCP 和工具审批可能接入 guardian review。 | `codex-rs/ext/guardian/`、`codex-rs/core/src/guardian/`、`codex-rs/core/src/mcp_tool_call.rs` | 如果有自己的安全审查 agent，应替换这里。 |
| Memories / Goals | 默认实现使用 Codex 本地/状态扩展。 | `codex-rs/ext/memories/`、`codex-rs/ext/goal/` | 可保留本地实现，也可替换成服务端状态。 |

仍然要区分目标：

- 改业务能力：不推荐改内置工具。用 MCP、skill、plugin、hook。
- 改工具协议：可以改 `codex-rs/core/src/tools/` 和 `codex-rs/tools/src/`，但这是核心行为变更，需要测试覆盖。
- 改 MCP 框架：优先在 `codex-rs/codex-mcp/src/connection_manager.rs` 附近改，减少跨层传参。
- 改插件框架：改 `codex-rs/core-plugins/src/` 和 `codex-rs/plugin/src/`。
- 改 Skills 框架：改 `codex-rs/core-skills/src/` 和 `codex-rs/skills/src/`。
- 改 Hooks 框架：改 `codex-rs/hooks/src/`，但 hook 是安全敏感路径。
- 改 app-server 暴露 API：改 `codex-rs/app-server-protocol/` 和 `codex-rs/app-server/`，并同步 schema。

普通扩展的一般推荐顺序：

1. Skill：只需要知识和流程。
2. MCP：需要确定性工具调用。
3. Plugin：需要打包分发多个能力。
4. Hook plugin：需要生命周期策略。
5. Extension / core tool：只有平台级能力才修改核心。

自研 agent 框架的推荐顺序：

1. 先决定是否保留 `app-server` extension registry。如果保留，就通过 `ExtensionRegistryBuilder` 替换默认 extension 安装清单。
2. 替换云端相关 extension：MCP hosted runtime、web search、image generation、auth/backend client。
3. 替换默认 skills 来源和 bundled skills。
4. 保留 MCP 作为外部工具协议，让业务工具仍通过 MCP/plugin 接入。
5. 只有当模型工具协议或会话编排不满足需求时，再改 core tools 和 session orchestration。

## 最小可用组合示例

### 只有 Skill

```text
my-plugin/
  .codex-plugin/plugin.json
  skills/my-workflow/SKILL.md
```

```json
{
  "name": "my-workflow",
  "version": "0.1.0",
  "skills": "./skills"
}
```

### Skill + MCP

```text
my-plugin/
  .codex-plugin/plugin.json
  .mcp.json
  skills/my-api/SKILL.md
```

```json
{
  "name": "my-api",
  "version": "0.1.0",
  "skills": "./skills",
  "mcpServers": "./.mcp.json"
}
```

### Hook 插件

```text
my-policy/
  .codex-plugin/plugin.json
  hooks/hooks.json
  hooks/pre-tool-use.sh
```

```json
{
  "name": "my-policy",
  "version": "0.1.0",
  "hooks": "./hooks/hooks.json"
}
```

### Channel / App Connector 插件

```text
my-channel/
  .codex-plugin/plugin.json
  .app.json
  .mcp.json
  skills/my-channel/SKILL.md
```

```json
{
  "name": "my-channel",
  "version": "0.1.0",
  "apps": "./.app.json",
  "mcpServers": "./.mcp.json",
  "skills": "./skills"
}
```

## 结论

这个仓库的扩展体系可以理解为：

- 工具是模型能调用的函数入口。
- MCP 是推荐的外部工具协议。
- Plugin 是分发格式。
- Skill 是模型行为说明包。
- Hook 是生命周期拦截器。
- Apps 是 app/channel connector 的声明方式。

如果只是新增业务能力，不建议修改内置工具；优先走 MCP、Skill、Plugin、Hook 和 Apps 声明。如果目标是 fork Codex 做自己的 agent 框架，则应明确审视并替换云端耦合的 extension、MCP contributor、auth/backend、marketplace 和 bundled skills，再决定是否改 core tools。
