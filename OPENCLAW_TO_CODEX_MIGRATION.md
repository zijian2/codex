# OpenClaw 本地容器能力迁移到 Codex 的分析

本文档记录当前本地 `openclaw-local` 容器内的插件、工具、MCP 与 skills 清单，并说明它们迁移到本仓库 Codex 时的推荐落点。

结论先行：

- OpenClaw 当前容器内实际可见的插件清单来自 `/app/plugins_bk`，共发现 17 个带 `openclaw.plugin.json` 的插件。
- OpenClaw 当前容器内全局 skills 来自 `/app/skills`，共发现 44 个 skill；插件内还附带 29 个 plugin skill。
- OpenClaw 容器有 MCP runtime、MCP config、MCP proxy、MCP bundle 等运行时代码，也有 `mcporter` skill，但未发现插件清单里直接声明 `.mcp.json` 或 `mcpServers` 的内置 MCP server 配置。
- OpenClaw 的 `api.registerTool(...)`、`channels`、`registerToolHooks(...)` 不能原样复制到 Codex。Codex 的推荐扩展形态是：外部业务工具优先走 MCP server，平台级深度工具可走 native `ToolContributor`，宿主/会话级工具可走 app-server `dynamic_tools`；指令走 skill，生命周期拦截走 hooks，外部聊天入口走 channel gateway + app-server/CLI，打包分发走 `.codex-plugin/plugin.json`。
- 需要特别区分：Codex 的普通 plugin manifest 当前没有直接 `tools` 字段；但 Codex runtime 支持非 MCP 工具，入口是 native `ToolContributor` 和 app-server `dynamic_tools`。

## 调研范围

本次只基于当前本地容器和当前仓库源码，不靠猜测。

已核对的 OpenClaw 容器路径：

- `/app/package.json`
- `/app/dist`
- `/app/plugins_bk`
- `/app/skills`
- `/root/.openclaw`

已核对的 Codex 仓库说明和源码入口：

- `BUILTIN_CAPABILITIES.md`
- `codex-rs/core/src/tools/`
- `codex-rs/codex-mcp/src/`
- `codex-rs/config/src/mcp_types.rs`
- `codex-rs/plugin/src/`
- `codex-rs/core-plugins/src/`
- `codex-rs/hooks/src/`
- `codex-rs/skills/src/`
- `codex-rs/app-server/src/extensions.rs`

## OpenClaw 容器整体结构

容器 `/app` 下关键内容：

| 路径 | 作用 |
| --- | --- |
| `/app/dist` | OpenClaw 构建后的运行时代码，包含 plugin runtime、channel runtime、MCP runtime、web search、image generation、hook runner 等 bundle。 |
| `/app/plugins_bk/extensions` | 本地 extension 插件源码或构建产物。 |
| `/app/plugins_bk/npm/node_modules` | npm 安装的 OpenClaw 插件。 |
| `/app/skills` | OpenClaw 全局内置 skills。 |
| `/root/.openclaw` | 当前容器运行态数据，包括 agents、sessions、artifacts、conversation todos、devices、extensions 等。 |

`/app/dist` 中和扩展相关的运行时文件包括：

- `runtime-plugins.runtime-fLHuT7Vs.js`
- `skills-snapshot.runtime.js`
- `runtime-channel-DfITb-7o.js`
- `channel-config-CWOBJZEd.js`
- `channel.setup-BUM2Et3k.js`
- `channel-inbound-roots-CrAAjhC8.js`
- `message-tool-api-V_QDQrfj.js`
- `web-search-providers.runtime.js`
- `image-generation-provider-yAeWypEo.js`
- `provider-tools-Dn1X46Br.js`
- `slash-plugin-commands.runtime.js`
- `pi-tools.before-tool-call.runtime-DfQIJo5M.js`
- `mcp-config-DYHOkN9M.js`
- `mcp-config-normalize-Df4xMZIV.js`
- `mcp-http-BxjVMJw4.js`
- `mcp-stdio-Vd4sWNp3.js`
- `mcp-cli-Ba1HCfAL.js`
- `bundle-mcp-DPPOalPH.js`
- `extensions/acpx/mcp-proxy.mjs`

这说明 OpenClaw 运行时本身支持 plugin、channel、hook、skill、MCP，但本次容器内“插件清单”没有直接声明内置 MCP server。

## 插件清单

### 总览

| 插件 | 来源 | 类型 | 主要能力 | Codex 迁移建议 |
| --- | --- | --- | --- | --- |
| `agent-relay` | extension | 云端/设备/子 agent relay | agent 上传、会话、设备、日志、代理客户端 | 不建议直接移植为普通插件；若做自有 agent 平台，应重写为平台服务 + Codex extension 或 MCP。 |
| `agent-team` | extension | 工具 + skill | 团队 agent 计划、创建、执行、完成、进度、清理 | 工具封装为 MCP 或 Codex native multi-agent extension；skill 可直接迁移。 |
| `astron-claw` | extension | channel + hook | AstronClaw channel、消息桥接、tool hook | channel 改为外部 gateway；hook 改为 Codex hooks；需要模型调用的动作封装为 MCP。 |
| `astronclaw-gateway` | extension | 云端 gateway | enroll、proxy、session router、supervisor | 平台相关，建议重写为 Codex app-server 外部服务或自研 gateway。 |
| `astronclaw-learning-loop` | extension | 学习闭环 + skill | review、skill 生成、触发、存储 | skill 可迁移；自动学习逻辑建议做 hooks + 外部服务。 |
| `astronclaw-plugins` | extension | 工具 | `artifact_capture` | 优先迁移为 Codex hook；若模型需要主动调用，再做 MCP tool。 |
| `astronmem-cloud-openclaw-plugin` | extension | 云端记忆 | AstronMem 云记忆、同步、召回 | 不建议照搬；替换为 Codex memories extension backend 或独立 MCP。 |
| `base-web-search-tool` | extension | 工具 | `base-web-search` | Codex 已有 `web.run`；如需自有搜索，迁为 MCP 或 web-search extension。 |
| `claw-core-deduct-api` | extension | utility | deduct API bridge/token | 平台计费/扣费相关；自有平台应替换或删除。 |
| `conversation-todo-sync` | extension | 工具 + skill | `astronclaw_todo_create/update/complete/get` | 迁为 MCP server + skill；状态存储改为 Codex 可控目录或外部 DB。 |
| `openclaw-astronclaw-trace` | extension | trace hook | endpoint、batch、debug、enabledHooks | 迁为 Codex hooks 或 app-server telemetry extension。 |
| `static-team` | extension | 工具 | `team_create_task/update_todo/finish_task` | 简单场景迁为 MCP；深度 agent 编排迁为 Codex native extension。 |
| `@openclaw/qqbot` | npm | channel + 工具 + skills | QQ Bot channel、提醒、媒体、channel API | channel 迁为 gateway；`qqbot_*` 工具迁为 MCP；skills 可迁移。 |
| `@soimy/dingtalk` | npm | channel | DingTalk channel、auth、card、feedback | channel 迁为 gateway；发送/查询动作按需做 MCP。 |
| `@tencent-weixin/openclaw-weixin` | npm | channel | 微信 channel、登录、消息、媒体、存储 | channel 迁为 gateway；账号登录和会话状态放外部服务。 |
| `@wecode-ai/weibo-openclaw-plugin` | npm | channel + skills | 微博 channel、搜索、热搜、发布、token、媒体 | channel 迁为 gateway；微博操作做 MCP；skills 可迁移。 |
| `@wecom/wecom-openclaw-plugin` | npm | channel + skills | 企业微信消息、会议、日程、文档、表格、待办 | channel 迁为 gateway；业务 API 做 MCP；skills 可迁移。 |

### 每个插件的具体内容

#### `agent-relay`

来源：`/app/plugins_bk/extensions/agent-relay`

Manifest 信息：

- package: `@openclaw/agent-relay`
- version: `0.1.0`
- plugin id: `agent-relay`
- enabledByDefault: `true`
- activation: `onStartup=false`, `onCommands=["agent-relay"]`
- command alias: `agent-relay`
- config: `apiKey`, `baseUrl`, `defaultTimeout`

主要文件：

- `src/agent/sub-agent.ts`
- `src/archive/pack.ts`
- `src/archive/unpack.ts`
- `src/cli.ts`
- `src/device.ts`
- `src/install.ts`
- `src/log-reporter.ts`
- `src/session/*`
- `src/storage/proxy-client.ts`
- `src/upload.ts`

作用判断：这是 OpenClaw 侧 agent relay 和云端/设备协作插件，不是简单模型工具。

迁移建议：

- 如果只是把某些远程操作给 Codex 调用，拆成 MCP tools。
- 如果要保留 agent relay 生命周期、session、上传、设备注册，应作为自有平台服务重写。
- 如果 fork Codex 做自有 agent 框架，建议在 app-server 扩展层接入，不建议塞进 core tools。

#### `agent-team`

来源：`/app/plugins_bk/extensions/agent-team`

Manifest 信息：

- package: `@openclaw/agent-team`
- version: `1.1.0`
- plugin id: `agent-team`
- enabledByDefault: `true`
- activation: `onStartup=true`
- skills: `./skills`
- contract tools:
  - `team_plan`
  - `team_provision`
  - `team_execute`
  - `team_complete`
  - `team_update_progress`
  - `team_cleanup`

主要文件：

- `skills/agent-team/SKILL.md`
- `skills/agent-team/SKILL-EN.md`
- `src/tools/team-*`

迁移建议：

- `SKILL.md` 可迁移到 Codex skill。
- 工具可以先做成一个 `agent-team-mcp` server，逐个暴露同名 tool。
- 如果要和 Codex multi-agent v2 深度结合，建议写 native extension，落点在 `codex-rs/core/src/tools/handlers/multi_agents_v2` 旁边或新的 extension crate，而不是把 OpenClaw JS runtime 嵌进去。

#### `astron-claw`

来源：`/app/plugins_bk/extensions/astron-claw`

Manifest 信息：

- package: `astron-claw`
- version: `0.2.0`
- plugin id: `astron-claw`
- type: `channel`
- channels: `astron-claw`
- channel config key: `astron-claw`
- config: `allowFrom`, `bridge`, `enabled`, `media`, `name`, `retry`

主要文件：

- `src/channel.ts`
- `src/runtime.ts`
- `src/hooks.ts`
- `src/messaging/*`
- `src/bridge/*`

工具/Hook 线索：

- 源码中有 `registerToolHooks(api)`。

迁移建议：

- Codex 没有 OpenClaw 这种 JS channel runtime。应该把 inbound/outbound channel 做成外部 gateway。
- gateway 接收 AstronClaw 消息后，通过 Codex app-server API 或 CLI 创建/继续会话。
- 发送消息、上传媒体、查询状态等动作如需给模型调用，做成 MCP tools。
- `registerToolHooks` 逻辑迁移到 Codex hooks，例如 `PreToolUse`、`PostToolUse`、`Stop`。

#### `astronclaw-gateway`

来源：`/app/plugins_bk/extensions/astronclaw-gateway`

Manifest 信息：

- package: `astronclaw-gateway`
- version: `0.1.0`
- plugin id: `astronclaw-gateway`
- activation: `onStartup=true`
- config: `agentProxy`, `enabled`, `installation`, `localGateway`, `retry`

主要文件：

- `src/enroll.ts`
- `src/proxy-client.ts`
- `src/session-router.ts`
- `src/supervisor.ts`

迁移建议：

- 这是 OpenClaw 平台耦合较强的 gateway 插件。
- 基于 Codex 做自有 agent 框架时，应设计自己的 gateway 服务，并通过 app-server、MCP 和 hooks 接入。
- 不推荐把它作为 Codex 插件原样迁移。

#### `astronclaw-learning-loop`

来源：`/app/plugins_bk/extensions/astronclaw-learning-loop`

Manifest 信息：

- package: `astronclaw-learning-loop`
- version: `0.0.3`
- plugin id: `astronclaw-learning-loop`
- enabledByDefault: `false`
- config: `enabled`, `review`, `skill`, `store`, `trigger`

主要文件：

- `src/engine.ts`
- `src/review/*`
- `src/skill/*`
- `src/trigger/*`
- `src/skills/skill-creator-automated/SKILL.md`

迁移建议：

- skill 可以迁移。
- review / trigger / store 这类学习闭环建议做外部服务或 Codex extension。
- 如果它需要在工具调用后学习，接 Codex `PostToolUse` / `Stop` hook。

#### `astronclaw-plugins`

来源：`/app/plugins_bk/extensions/astronclaw-plugins`

Manifest 信息：

- package: `astronclaw-plugins`
- version: `0.1.0`
- plugin id: `astronclaw-plugins`
- activation: `onStartup=true`
- contract tool: `artifact_capture`
- config: `artifactCapture`

工具线索：

- `dist/index.js` 中存在 `api.registerTool({ ... })`。
- `/root/.openclaw/artifacts/astronclaw-plugins` 下有运行态 artifact 数据。

迁移建议：

- 如果目标是自动捕获产物，不应让模型主动调用，优先做 Codex `PostToolUse`、`Stop` 或 session lifecycle hook。
- 如果目标是让模型主动登记 artifact，再做 MCP tool。
- Codex 侧可以把 artifact index 写入 workspace、`$CODEX_HOME` 或外部对象存储。

#### `astronmem-cloud-openclaw-plugin`

来源：`/app/plugins_bk/extensions/astronmem-cloud-openclaw-plugin`

Manifest 信息：

- package: `astronmem-cloud-openclaw-plugin`
- version: `0.1.2`
- plugin id: `astronmem-cloud-openclaw-plugin`
- activation: `onStartup=true`, `onCommands=["astronMem"]`
- command alias: `astronMem`
- config:
  - `addMessageMaxPollAttempts`
  - `apiKey`
  - `apiSecret`
  - `astronMemDeductBaseUrl`
  - `astronMemDeductBridgeToken`
  - `astronMemDeductProvider`
  - `baseUrl`
  - `memoryLimitNumber`
  - `pollIntervalMs`
  - `recallThreshold`
  - `runMode`
  - `syncCooldownMs`

迁移建议：

- Codex 已有 memories extension：`codex-rs/ext/memories`。
- 若自有平台继续用 AstronMem，应替换 memories backend，或做一个 `astronmem-mcp`。
- 涉及云端鉴权、扣费、轮询同步的字段不建议直接带入 Codex manifest；应放到 MCP server env 或自有服务配置。

#### `base-web-search-tool`

来源：`/app/plugins_bk/extensions/base-web-search-tool`

Manifest 信息：

- package: `base-web-search-tool`
- version: `0.1.0`
- plugin id: `base-web-search-tool`
- enabledByDefault: `true`
- activation: `onStartup=true`
- contract tool: `base-web-search`
- config: `defaultFullText`, `defaultLimit`, `defaultRerank`, `enabled`, `timeoutMs`

主要文件：

- `index.js`
- `index.test.js`
- `README.md`

迁移建议：

- Codex 已有 `web.run` extension 和 hosted web search。
- 如果要保留 OpenClaw 的搜索 provider、rerank、full text 逻辑，封装为 MCP server。
- 不建议为了一个搜索 provider 修改 Codex core tools。

#### `claw-core-deduct-api`

来源：`/app/plugins_bk/extensions/claw-core-deduct-api`

Manifest 信息：

- package: `@openclaw/claw-core-deduct-api`
- version: `1.0.0`
- plugin id: `claw-core-deduct-api`
- type: `utility`
- config: `bridge_token`, `deduct_api_url`

迁移建议：

- 这是 OpenClaw 云端计费/扣费 utility。
- 如果自研 agent 框架不使用 OpenClaw 云服务，应删除或替换。
- 如果需要保留扣费，应放在自有服务侧，不建议作为模型可见工具。

#### `conversation-todo-sync`

来源：`/app/plugins_bk/extensions/conversation-todo-sync`

Manifest 信息：

- package: `@openclaw/conversation-todo-sync`
- version: `0.1.0`
- plugin id: `conversation-todo-sync`
- enabledByDefault: `true`
- activation: `onStartup=true`
- skills: `./skills`
- contract tools:
  - `astronclaw_todo_create`
  - `astronclaw_todo_update`
  - `astronclaw_todo_complete`
  - `astronclaw_todo_get`

主要文件：

- `skills/conversation-todo-sync/SKILL.md`
- `src/tools/todo-*`

运行态数据：

- `/root/.openclaw/conversation-todos`

迁移建议：

- 这是最适合先迁移的工具类插件。
- 做一个 `conversation-todo-mcp`，保留四个工具名或改成更通用的 `todo_create/update/complete/get`。
- skill 直接迁移，但要把 OpenClaw 路径说明替换为 Codex 的 MCP tool 名和存储位置。
- 如果想和 Codex goal 合并，可以考虑改造到 `codex-rs/ext/goal`，但这属于 native fork。

#### `openclaw-astronclaw-trace`

来源：`/app/plugins_bk/extensions/openclaw-astronclaw-trace`

Manifest 信息：

- package: `@astronclaw/openclaw-astronclaw-trace`
- version: `0.1.0`
- plugin id: `openclaw-astronclaw-trace`
- type: `plugin`
- config: `authorization`, `batchInterval`, `batchSize`, `debug`, `enabledHooks`, `endpoint`, `serviceName`

迁移建议：

- 迁移为 Codex hooks 或 telemetry extension。
- 如果只是上报工具调用、会话结束、错误信息，用 Codex lifecycle hooks 更合适。
- 如果需要结构化事件流和批量上报，做 app-server extension 更稳。

#### `static-team`

来源：`/app/plugins_bk/extensions/static-team`

Manifest 信息：

- package: `@openclaw/static-team`
- version: `1.0.0`
- plugin id: `static-team`
- enabledByDefault: `true`
- activation: `onStartup=true`
- contract tools:
  - `team_create_task`
  - `team_update_todo`
  - `team_finish_task`

主要文件：

- `src/tools/team-*`

迁移建议：

- 简单保留功能：MCP server。
- 深度接入 Codex agent 编排：native extension 或改造 multi-agent v2。

#### `@openclaw/qqbot`

来源：`/app/plugins_bk/npm/node_modules/@openclaw/qqbot`

Manifest 信息：

- package: `@openclaw/qqbot`
- version: `2026.5.7`
- plugin id: `qqbot`
- enabledByDefault: `true`
- channels: `qqbot`
- channel config key: `qqbot`
- skills: `./skills`
- contract tools:
  - `qqbot_channel_api`
  - `qqbot_remind`
- config:
  - `accounts`
  - `allowFrom`
  - `appId`
  - `audioFormatPolicy`
  - `clientSecret`
  - `clientSecretFile`
  - `defaultAccount`
  - `enabled`
  - `markdownSupport`
  - `name`
  - `streaming`
  - `stt`
  - `systemPrompt`
  - `upgradeMode`
  - `upgradeUrl`
  - `urlDirectUpload`
  - `voiceDirectUploadFormats`

Plugin skills:

- `qqbot-channel`
- `qqbot-media`
- `qqbot-remind`

迁移建议：

- QQ inbound/outbound channel 放到外部 gateway。
- `qqbot_channel_api` 和 `qqbot_remind` 做 MCP tools。
- skills 迁移到 Codex plugin `skills` 目录。
- 账号、secret、upload URL、STT 等配置放 gateway/MCP server 的 env，不放进 Codex skill。

#### `@soimy/dingtalk`

来源：`/app/plugins_bk/npm/node_modules/@soimy/dingtalk`

Manifest 信息：

- package: `@soimy/dingtalk`
- version: `3.6.2`
- plugin id: `dingtalk`
- channels: `dingtalk`
- channel config key: `dingtalk`

主要能力：

- DingTalk channel
- auth
- card service
- connection manager
- feedback learning
- messaging

迁移建议：

- DingTalk 消息入口做 gateway。
- 发消息、卡片、审批、反馈学习等动作按需做 MCP。
- 如果没有模型主动操作需求，只需要 gateway 驱动 Codex 会话即可。

#### `@tencent-weixin/openclaw-weixin`

来源：`/app/plugins_bk/npm/node_modules/@tencent-weixin/openclaw-weixin`

Manifest 信息：

- package: `@tencent-weixin/openclaw-weixin`
- version: `2.4.2`
- plugin id: `openclaw-weixin`
- channels: `openclaw-weixin`
- channel config key: `openclaw-weixin`

主要文件：

- `src/api/*`
- `src/auth/login-qr.ts`
- `src/channel.ts`
- `src/messaging/*`
- `src/media/*`
- `src/storage/*`

迁移建议：

- 微信登录、二维码、消息收发和媒体处理都应留在外部 gateway。
- Codex 只接收标准化后的用户消息，并通过 gateway 暴露的 MCP tools 做发送/媒体操作。
- 不建议把微信会话状态放进 Codex core。

#### `@wecode-ai/weibo-openclaw-plugin`

来源：`/app/plugins_bk/npm/node_modules/@wecode-ai/weibo-openclaw-plugin`

Manifest 信息：

- package: `@wecode-ai/weibo-openclaw-plugin`
- version: `2.2.4`
- plugin id: `weibo-openclaw-plugin`
- channels: `weibo`
- channel config key: `weibo`
- skills: `./skills`

Plugin skills:

- `weibo-cron`
- `weibo-crowd`
- `weibo-hot-search`
- `weibo-pic`
- `weibo-search`
- `weibo-status`
- `weibo-token`
- `weibo-video`

迁移建议：

- 微博 channel 做 gateway。
- 搜索、热搜、发布、媒体上传、token 管理等能力做 MCP。
- skills 可以迁移，但需要替换其中 OpenClaw message tool / channel API 的调用说明。

#### `@wecom/wecom-openclaw-plugin`

来源：`/app/plugins_bk/npm/node_modules/@wecom/wecom-openclaw-plugin`

Manifest 信息：

- package: `@wecom/wecom-openclaw-plugin`
- version: `2026.5.7`
- plugin id: `wecom-openclaw-plugin`
- channels: `wecom`
- skills: `./skills`

Plugin skills:

- `wecom-contact-lookup`
- `wecom-doc-manager`
- `wecom-edit-todo`
- `wecom-get-todo-detail`
- `wecom-get-todo-list`
- `wecom-meeting-create`
- `wecom-meeting-manage`
- `wecom-meeting-query`
- `wecom-msg`
- `wecom-preflight`
- `wecom-schedule`
- `wecom-send-media`
- `wecom-send-template-card`
- `wecom-smartsheet-data`
- `wecom-smartsheet-schema`

迁移建议：

- 企业微信 inbound channel 做 gateway。
- 联系人、文档、会议、日程、待办、智能表格、消息发送做 MCP tools。
- skills 迁移为 Codex plugin skills。

## OpenClaw 工具清单

本次从插件 manifest 的 `contracts` 和源码中的 `api.registerTool` 逐项核对到以下工具。

| 工具名 | 来源插件 | 作用 | 推荐 Codex 落点 |
| --- | --- | --- | --- |
| `team_plan` | `agent-team` | 团队任务规划 | MCP 或 native multi-agent extension |
| `team_provision` | `agent-team` | 创建/准备团队执行上下文 | MCP 或 native multi-agent extension |
| `team_execute` | `agent-team` | 执行团队任务 | MCP 或 native multi-agent extension |
| `team_complete` | `agent-team` | 完成团队任务 | MCP 或 native multi-agent extension |
| `team_update_progress` | `agent-team` | 更新团队执行进度 | MCP 或 native multi-agent extension |
| `team_cleanup` | `agent-team` | 清理团队任务资源 | MCP 或 native multi-agent extension |
| `artifact_capture` | `astronclaw-plugins` | 捕获会话/子任务产物 | Codex hook 优先，MCP 次选 |
| `base-web-search` | `base-web-search-tool` | 搜索、全文、rerank | Codex `web.run` 或自定义 MCP |
| `astronclaw_todo_create` | `conversation-todo-sync` | 创建 conversation todo | MCP |
| `astronclaw_todo_update` | `conversation-todo-sync` | 更新 conversation todo | MCP |
| `astronclaw_todo_complete` | `conversation-todo-sync` | 完成 conversation todo | MCP |
| `astronclaw_todo_get` | `conversation-todo-sync` | 获取 conversation todo | MCP |
| `team_create_task` | `static-team` | 创建团队任务 | MCP 或 native extension |
| `team_update_todo` | `static-team` | 更新团队 todo | MCP 或 native extension |
| `team_finish_task` | `static-team` | 完成团队任务 | MCP 或 native extension |
| `qqbot_channel_api` | `@openclaw/qqbot` | QQ Bot channel API | channel gateway + MCP |
| `qqbot_remind` | `@openclaw/qqbot` | QQ Bot 提醒 | MCP |

迁移原则：

- 不要把这些工具直接加进 `codex-rs/core/src/tools/handlers`，除非它们会成为 Codex 的通用核心能力。
- 先用 MCP server 保留行为，再根据运行效果决定是否需要 native extension。
- 需要自动触发的逻辑不要做成模型工具，优先做 hook。

## OpenClaw channel 插件

当前发现的 channel：

| Channel | 来源插件 | 主要用途 | Codex 迁移方式 |
| --- | --- | --- | --- |
| `astron-claw` | `astron-claw` | AstronClaw 消息桥接 | 外部 gateway + Codex app-server/CLI + MCP |
| `qqbot` | `@openclaw/qqbot` | QQ Bot | 外部 gateway + MCP |
| `dingtalk` | `@soimy/dingtalk` | 钉钉 | 外部 gateway + MCP |
| `openclaw-weixin` | `@tencent-weixin/openclaw-weixin` | 微信 | 外部 gateway + MCP |
| `weibo` | `@wecode-ai/weibo-openclaw-plugin` | 微博 | 外部 gateway + MCP |
| `wecom` | `@wecom/wecom-openclaw-plugin` | 企业微信 | 外部 gateway + MCP |

Codex 当前没有 OpenClaw 这种内置 JS channel runtime。Codex plugin 的 `apps` 更接近 connector 声明，不是一个直接执行 channel adapter 的 runtime。

推荐结构：

```text
外部平台消息
  -> channel gateway
  -> Codex app-server API 或 Codex CLI
  -> Codex thread/session
  -> 模型需要主动发消息时调用 MCP tool
  -> channel gateway 执行真实发送
```

如果要把 channel 能力打成 Codex 插件，推荐目录：

```text
my-channel-plugin/
  .codex-plugin/
    plugin.json
  .mcp.json
  .app.json
  skills/
    my-channel/
      SKILL.md
```

`plugin.json` 示例：

```json
{
  "name": "my-channel-plugin",
  "version": "0.1.0",
  "description": "Channel gateway tools and skills for Codex",
  "apps": "./.app.json",
  "mcpServers": "./.mcp.json",
  "skills": "./skills"
}
```

`.mcp.json` 示例：

```json
{
  "mcpServers": {
    "my-channel": {
      "command": "node",
      "args": ["./server.js"],
      "env": {
        "CHANNEL_GATEWAY_URL": "https://gateway.example.com"
      },
      "enabled": true
    }
  }
}
```

注意：`.app.json` 负责 Codex app/connector 侧声明；真正工具仍由 MCP 或 Codex connector 后端提供。

## OpenClaw MCP 情况

本次在容器里确认到 MCP 相关文件：

- `/app/dist/mcp`
- `/app/dist/mcp-cli-Ba1HCfAL.js`
- `/app/dist/mcp-config-DYHOkN9M.js`
- `/app/dist/mcp-config-normalize-Df4xMZIV.js`
- `/app/dist/mcp-http-BxjVMJw4.js`
- `/app/dist/mcp-stdio-Vd4sWNp3.js`
- `/app/dist/bundle-mcp-DPPOalPH.js`
- `/app/dist/extensions/acpx/mcp-proxy.mjs`
- `/app/docs/cli/mcp.md`
- `/app/skills/mcporter`

同时也确认到：

- `/app/plugins_bk` 下的插件 manifest 没有发现 `.mcp.json`。
- `/app/plugins_bk`、`/app/skills` 中没有发现插件式 `mcpServers` 声明。
- 容器用户态 `/root/.openclaw` 中没有发现 `.mcp.json` 或 `mcp.json`。

因此当前容器里的 MCP 更像是 OpenClaw 运行时能力和辅助 skill，而不是已经打包好的内置 MCP server 清单。

迁移到 Codex 时，应该主动为 OpenClaw 工具创建 MCP server，而不是寻找一个现成的 `.mcp.json` 复制过来。

## OpenClaw 全局 skills

容器 `/app/skills` 下发现 44 个全局 skill：

| Skill | 作用 | 迁移建议 |
| --- | --- | --- |
| `1password` | 使用 1Password CLI。 | 可迁移，确认目标环境有 `op` CLI。 |
| `apple-notes` | 通过 `memo` CLI 操作 Apple Notes。 | 仅 macOS/有依赖时迁移。 |
| `apple-reminders` | 通过 `remindctl` 操作 Apple Reminders。 | 仅 macOS/有依赖时迁移。 |
| `blogwatcher` | 博客/RSS 监控。 | 可迁移，依赖脚本需一起迁。 |
| `blucli` | BluOS CLI。 | 按需迁移。 |
| `camsnap` | RTSP/ONVIF 截图。 | 按需迁移，注意网络/设备权限。 |
| `canvas` | Canvas Skill。 | 可迁移，确认资源文件。 |
| `channel-qr-binding` | 微信/飞书/Lark/钉钉/QQ Bot 的二维码绑定流程。 | 迁移时需改成 channel gateway 的绑定流程。 |
| `clawhub` | ClawHub skills registry CLI。 | OpenClaw 平台相关，不建议原样迁。 |
| `cloud-local-communication` | 云端和本地桥接，用于本地桌面动作。 | 平台相关；若自研云端 agent，需重写桥接。 |
| `coding-agent` | 把代码任务委托给 Codex/Claude/OpenCode/Pi 等后台进程。 | 可参考，但要替换成 Codex multi-agent 或本地进程策略。 |
| `discord` | Discord message tool 操作。 | 做 Discord MCP 后再迁移 skill。 |
| `eightctl` | Eight Sleep pods。 | 按需迁移。 |
| `gh-issues` | 拉取 GitHub issue、委托修复、PR。 | 可迁移，但要对齐 Codex GitHub 工作流。 |
| `github` | GitHub CLI。 | 可迁移，依赖 `gh`。 |
| `healthcheck` | host hardening 检查。 | 可迁移，注意权限。 |
| `imsg` | iMessage/SMS CLI。 | macOS 专用，按需迁移。 |
| `mcporter` | MCP server/tool 管理。 | 可迁移，但需改成 Codex MCP 配置路径和格式。 |
| `model-usage` | 本地模型成本日志。 | OpenClaw 日志格式相关，需改造。 |
| `nano-pdf` | 编辑 PDF。 | 可迁移，确认依赖。 |
| `node-connect` | OpenClaw Android/iOS/macOS node pairing 诊断。 | OpenClaw 平台相关，不建议原样迁。 |
| `obsidian` | Obsidian vault 自动化。 | 可迁移。 |
| `openai-whisper` | 本地 Whisper STT。 | 可迁移。 |
| `oracle` | 第二模型调试/评审 CLI。 | 可迁移，需确认 CLI。 |
| `ordercli` | Foodora 订单。 | 按需迁移。 |
| `peekaboo` | macOS UI 捕获/自动化。 | macOS 专用，按需迁移。 |
| `sag` | ElevenLabs TTS。 | 可迁移，确认密钥。 |
| `session-logs` | 搜索/分析自己的 session logs。 | 需要改成 Codex session 路径和日志格式。 |
| `sherpa-onnx-tts` | 本地离线 TTS。 | 可迁移。 |
| `skill-creator` | 创建/审查 AgentSkills/SKILL.md。 | Codex 已有系统 `skill-creator`，不要重复迁移，必要时合并差异。 |
| `slack` | Slack tool 操作。 | 做 Slack MCP 后再迁移 skill。 |
| `songsee` | 声谱图/音频特征。 | 可迁移。 |
| `sonoscli` | Sonos CLI。 | 按需迁移。 |
| `summarize` | 总结/转写 URL、视频、PDF、本地文件。 | 可迁移，确认依赖和网络策略。 |
| `taskflow` | durable multi-step detached tasks。 | 可迁移思路；执行引擎需重写或改为 Codex goal/multi-agent。 |
| `taskflow-inbox-triage` | TaskFlow inbox triage 示例。 | 依赖 taskflow，按需迁移。 |
| `things-mac` | Things 3 todos。 | macOS 专用，按需迁移。 |
| `tmux` | tmux 远程控制。 | 可迁移。 |
| `video-frames` | ffmpeg 抽帧/剪辑。 | 可迁移，依赖 ffmpeg。 |
| `voice-call` | OpenClaw voice-call plugin。 | OpenClaw 插件相关，需改造为 gateway/MCP。 |
| `wacli` | WhatsApp via wacli。 | 做 WhatsApp gateway/MCP 后迁移。 |
| `weather` | 天气/预报。 | 可迁移；也可改为 MCP。 |
| `weibo-config` | 配置 Weibo channel。 | 需要改成新的 Weibo gateway 配置。 |
| `xurl` | X API。 | 可迁移，确认 API 配置。 |

### 全局 skills 迁移方式

Codex skill 的基本结构也是目录 + `SKILL.md`，所以最容易迁移的是纯指令型 skill。

推荐迁移步骤：

1. 对每个 skill 读取完整 `SKILL.md`。
2. 检查是否引用 `scripts/`、`references/`、`assets/`。
3. 把 OpenClaw 专有命令、路径和 message tool 名称替换成 Codex 对应机制。
4. 放入 Codex 用户级 `$CODEX_HOME/skills`、仓库级 `.codex/skills`，或打包进插件 `skills`。
5. 对需要工具的 skill，先提供 MCP server，再启用 skill。

不建议直接迁移的 skill 类型：

- 强依赖 OpenClaw 云服务、ClawHub、OpenClaw node pairing 的 skill。
- 只适用于 OpenClaw channel runtime 的 skill。
- 依赖 OpenClaw session/artifact 日志格式的 skill，除非先改路径和解析器。

## OpenClaw plugin skills

插件内发现的 skills：

| Plugin | Skills | 迁移建议 |
| --- | --- | --- |
| `agent-team` | `agent-team` | 和 team MCP/native extension 一起迁。 |
| `astronclaw-learning-loop` | `skill-creator-automated` | 可参考，但学习闭环逻辑需重写。 |
| `conversation-todo-sync` | `conversation-todo-sync` | 和 todo MCP 一起迁。 |
| `@openclaw/qqbot` | `qqbot-channel`, `qqbot-media`, `qqbot-remind` | 和 QQ gateway/MCP 一起迁。 |
| `@wecode-ai/weibo-openclaw-plugin` | `weibo-cron`, `weibo-crowd`, `weibo-hot-search`, `weibo-pic`, `weibo-search`, `weibo-status`, `weibo-token`, `weibo-video` | 和 Weibo gateway/MCP 一起迁。 |
| `@wecom/wecom-openclaw-plugin` | `wecom-contact-lookup`, `wecom-doc-manager`, `wecom-edit-todo`, `wecom-get-todo-detail`, `wecom-get-todo-list`, `wecom-meeting-create`, `wecom-meeting-manage`, `wecom-meeting-query`, `wecom-msg`, `wecom-preflight`, `wecom-schedule`, `wecom-send-media`, `wecom-send-template-card`, `wecom-smartsheet-data`, `wecom-smartsheet-schema` | 和 WeCom gateway/MCP 一起迁。 |

这些 plugin skills 迁移后，建议以 Codex plugin 分发，而不是全部塞进全局 skills。这样 tool、skill、hook、app connector 能一起版本化。

## OpenClaw 到 Codex 的概念映射

| OpenClaw 概念 | Codex 对应机制 | 说明 |
| --- | --- | --- |
| `openclaw.plugin.json` | `.codex-plugin/plugin.json` | 字段不同，不能直接复制。 |
| `skills: "./skills"` | `skills` | 可以迁移，但要改 OpenClaw 专有调用说明。 |
| `contracts.tools` / `api.registerTool` | MCP server tools、native extension tools 或 app-server dynamic tools | 外部业务工具推荐 MCP；平台深度集成用 native；宿主会话级工具用 dynamic tools。 |
| `channels` / `channelConfigKeys` | 外部 channel gateway + Codex `apps` 声明 + MCP | Codex 没有同款 JS channel runtime。 |
| `registerToolHooks` | Codex hooks | Codex hook handler 支持 command/prompt/agent，不是 OpenClaw JS hook API。 |
| `activation.onStartup` | Codex plugin 安装/加载 + MCP server 启动 | Codex manifest 没有同样 activation 语义。 |
| `activation.onCommands` / `commandAliases` | slash command、skill、MCP tool 或外部 CLI | 需要按用途重建。 |
| `configSchema` | MCP env/config、Codex config、插件 `interface` | Codex plugin manifest 不原生执行 OpenClaw config schema。 |
| OpenClaw cloud relay/gateway | 自有 gateway/app-server extension | 平台耦合能力应重写。 |
| OpenClaw artifact capture | Codex hooks 或 native telemetry/artifact extension | 自动化捕获不应优先暴露为模型工具。 |

## 基于 Codex 源码的迁移可行性评估

本节只基于当前仓库源码判断，不按概念猜测。

### 结论

OpenClaw 的“平台能力通过插件形式实现，再通过 hook 挂载”的迁移方式在 Codex 里可行，但需要翻译成 Codex 的机制：

```text
OpenClaw plugin runtime
  api.registerTool
  registerToolHooks
  channels
  skills
  configSchema

Codex plugin bundle
  mcpServers
  hooks
  skills
  apps
  interface
```

可行边界：

- 平台能力如果只是拦截、审计、产物捕获、trace、补充上下文，适合迁为 Codex plugin `hooks`。
- 平台能力如果要给模型调用业务动作，可以迁为 MCP、native extension tool 或 app-server dynamic tool；选择取决于是否需要深度访问 Codex runtime。
- 平台能力如果只是给模型使用说明和流程，适合迁为 Codex plugin `skills`。
- Channel/inbound message 不能只靠 hook 承接，应该通过 app-server 的 thread/turn API 做 gateway。
- 如果平台能力要改鉴权、connector catalog、memory backend、multi-agent 调度、thread 生命周期，则应做 Codex native extension 或 fork app-server/core。

### 源码依据

| 结论 | 源码依据 | 说明 |
| --- | --- | --- |
| Codex plugin 可以声明 `skills`、`mcpServers`、`apps`、`hooks`、`interface`。 | `codex-rs/core-plugins/src/manifest.rs` 中 `RawPluginManifest` 和 `parse_plugin_manifest`。 | 这证明 Codex plugin 是能力分发包，可以承接 OpenClaw 插件的 skills、MCP、hook 和 app connector 声明。 |
| Codex plugin manifest 当前没有直接 `tools` 字段。 | `codex-rs/plugin/src/manifest.rs` 的 `PluginManifestPaths` 只有 `skills`、`mcp_servers`、`apps`、`hooks`；`codex-rs/plugin/src/load_outcome.rs` 没有 `effective_tools`。 | OpenClaw `api.registerTool` 不能作为 plugin manifest 字段原样迁移；但可以走 MCP、native `ToolContributor` 或 app-server `dynamic_tools`。 |
| `hooks` 支持路径、路径数组、内联对象、内联对象数组。 | `codex-rs/core-plugins/src/manifest.rs` 的 `RawPluginManifestHooks` 和 `resolve_manifest_hooks`。 | OpenClaw 的 hook 插件可以迁成插件内 `hooks/hooks.json`，也可以直接内联到 manifest。 |
| `mcpServers` 支持路径或内联对象。 | `codex-rs/core-plugins/src/manifest.rs` 的 `RawPluginManifestMcpServers` 和 `resolve_manifest_mcp_servers`。 | OpenClaw 的外部业务工具可迁成 MCP server，再由 plugin manifest 声明。 |
| Codex 支持 native extension tools。 | `codex-rs/ext/extension-api/src/contributors.rs` 的 `ToolContributor`，以及 `codex-rs/core/src/tools/spec_plan.rs` 的 `add_extension_tools`。 | 需要深度接入 Codex session/thread/state/metrics 的 OpenClaw 平台工具可以迁成 native Rust extension。 |
| Codex 支持 app-server dynamic tools。 | `codex-rs/app-server-protocol/src/protocol/v2/thread.rs` 的 `dynamic_tools`，`codex-rs/core/src/tools/handlers/dynamic.rs`，`codex-rs/app-server/src/dynamic_tools.rs`。 | 宿主或 channel gateway 可以为单个 thread 提供会话级工具，由 app-server client 响应工具调用。 |
| plugin loader 会实际加载 plugin skills、MCP、apps、hooks。 | `codex-rs/core-plugins/src/loader.rs` 中 `load_plugin_skills`、`load_plugin_mcp_servers_from_manifest`、`load_plugin_apps`、`PluginHookSource`。 | 这证明 manifest 字段不是静态元数据，会进入运行时加载结果。 |
| plugin hook 会被转换为 hook declaration。 | `codex-rs/hooks/src/declarations.rs` 中 `plugin_hook_declarations`。 | Codex 能识别插件内声明的 hook handler，并给每个 handler 生成稳定 key。 |
| Codex hook 事件足够覆盖 OpenClaw 平台挂载点。 | `codex-rs/config/src/hook_config.rs` 的 `HookEventsToml`。 | 支持 `PreToolUse`、`PermissionRequest`、`PostToolUse`、`PreCompact`、`PostCompact`、`SessionStart`、`UserPromptSubmit`、`SubagentStart`、`SubagentStop`、`Stop`。 |
| hook handler 类型不是 OpenClaw JS SDK，而是 `command`、`prompt`、`agent`。 | `codex-rs/config/src/hook_config.rs` 的 `HookHandlerConfig`。 | 迁移时不能原样搬 `registerToolHooks(api)`；要改成命令脚本、prompt hook 或 agent hook。 |
| `PreToolUse` hook 能看到工具名、输入、cwd、model、permission mode、transcript path、tool use id，并能 block/update input/add context。 | `codex-rs/hooks/src/events/pre_tool_use.rs` 的 `PreToolUseRequest` 和 `PreToolUseOutcome`。 | 适合做工具调用前策略、审计、参数修正、权限前置检查。 |
| `PostToolUse` hook 能看到工具输入和工具输出，并能 block/add context/feedback。 | `codex-rs/hooks/src/events/post_tool_use.rs` 的 `PostToolUseRequest` 和 `PostToolUseOutcome`。 | 适合做 artifact capture、trace、学习闭环、工具输出审计。 |
| app-server 提供线程生命周期 API。 | `codex-rs/app-server-protocol/src/protocol/common.rs` 的 `thread/start`、`thread/resume`、`thread/read`、`thread/list` 等。 | channel gateway 可以创建、恢复、读取 Codex thread。 |
| app-server 提供 turn API。 | `codex-rs/app-server-protocol/src/protocol/common.rs` 的 `turn/start`、`turn/steer`、`turn/interrupt`。 | inbound channel 文本消息应通过 `turn/start` 启动一次用户 turn，而不是用 hook 或直接写 history。 |
| `turn/start` 支持文本、图片、本地图片、skill、mention 输入。 | `codex-rs/app-server-protocol/src/protocol/v2/turn.rs` 的 `TurnStartParams` 和 `UserInput`。 | channel gateway 可以把外部消息转为 `UserInput::Text`，图片/媒体可按 URL 或本地文件接入。 |
| `turn/start` 最终提交 `Op::UserInput` 并启动推理。 | `codex-rs/app-server/src/request_processors/turn_processor.rs` 的 `turn_start_inner`。 | 这是 channel gateway 驱动 Codex 回复的主要入口。 |
| app-server 还有实验 realtime API。 | `codex-rs/app-server-protocol/src/protocol/common.rs` 和 `v2/realtime.rs` 的 `thread/realtime/start`、`appendText`、`appendAudio`、`appendSpeech`。 | 语音/实时 channel 可以评估 realtime 路径，但它是 experimental，应谨慎作为核心依赖。 |
| `thread/inject_items` 只追加 raw Responses API items，不启动用户 turn。 | `codex-rs/app-server-protocol/src/protocol/v2/thread.rs` 和 `turn_processor.rs` 的 `thread_inject_items_response_inner`。 | 不适合作为普通 channel inbound 消息入口，除非只是导入历史。 |

## Codex 当前工具与 AstronClaw 差距

本节按源码中的工具注册路径统计。实际会话里暴露哪些工具，还会受 model/provider capability、feature flag、auth、environment、permission profile、multi-agent 配置影响。

### Codex 当前内置工具

| 类别 | 工具或 namespace | 作用 | 源码依据 |
| --- | --- | --- | --- |
| Shell/执行 | `shell_command`、`exec_command`、`write_stdin` | 执行命令、统一 exec 会话写入 stdin。 | `codex-rs/core/src/tools/handlers/shell/shell_command.rs`、`unified_exec/*`、`spec_plan.rs` 的 `add_shell_tools`。 |
| 文件/图片/编辑 | `apply_patch`、`view_image` | 修改文件、查看本地图片。 | `codex-rs/core/src/tools/handlers/apply_patch.rs`、`view_image.rs`。 |
| 计划/上下文/交互 | `update_plan`、`request_permissions`、`request_user_input`、`get_context_remaining`、`new_context`、`sleep`、`clock.curr_time` | 计划管理、权限请求、用户输入、上下文预算、压缩窗口、睡眠、读取当前 UTC 时间。 | `codex-rs/core/src/tools/handlers/{plan,request_permissions,request_user_input,get_context_remaining,new_context_window,sleep,current_time}.rs`。 |
| MCP 资源 | `list_mcp_resources`、`list_mcp_resource_templates`、`read_mcp_resource` | 读取 MCP resource/template。 | `codex-rs/core/src/tools/handlers/mcp_resource/*`。 |
| MCP 工具 | MCP callable namespace tools | 把 MCP server 的 tool 转成模型工具。 | `codex-rs/core/src/tools/handlers/mcp.rs`、`spec_plan.rs` 的 `add_mcp_runtime_tools`。 |
| 插件发现/安装建议 | `tool_search`、`list_available_plugins_to_install`、`request_plugin_install` | 搜索 deferred tools、列出可安装插件、请求插件安装。 | `codex-rs/core/src/tools/handlers/tool_search.rs`、`list_available_plugins_to_install.rs`、`request_plugin_install.rs`。 |
| 多 agent | `multi_agent_v1.spawn_agent/send_input/resume_agent/wait_agent/close_agent`；v2 为 plain tools 或可配置 namespace | 创建、通信、等待、打断、列出 sub-agent。 | `codex-rs/core/src/tools/handlers/multi_agents*`、`multi_agents_spec.rs`。 |
| agent jobs | `spawn_agents_on_csv`、`report_agent_job_result` | CSV 批量 agent jobs 和 worker 回报。 | `codex-rs/core/src/tools/handlers/agent_jobs/*`。 |
| Hosted model tools | `web_search`、`image_generation` | 由 Responses/模型侧托管的 web search 和 image generation。 | `codex-rs/core/src/tools/hosted_spec.rs`、`spec_plan.rs` 的 `hosted_model_tool_specs`。 |
| Native extension tools | `web.run`、`image_gen.imagegen`、`get_goal/create_goal/update_goal`、`memories.*`、`skills.list/read` | Rust extension 贡献的非 MCP 工具。 | `codex-rs/ext/{web-search,image-generation,goal,memories,skills}`、`codex-rs/ext/extension-api/src/contributors.rs`。 |
| Dynamic tools | plain dynamic tool 或自定义 namespace dynamic tool | app-server client 在线程启动时提供、调用时回调 client。 | `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`、`codex-rs/core/src/tools/handlers/dynamic.rs`。 |

### 与 AstronClaw/OpenClaw 对比缺口

| AstronClaw/OpenClaw 能力 | Codex 当前是否内置 | 缺口 | 推荐补齐方式 |
| --- | --- | --- | --- |
| `artifact_capture` | 无同名模型工具 | Codex 有 hooks，但没有 AstronClaw artifact capture 业务逻辑。 | 优先 `PostToolUse`/`Stop` hook + 外部 artifact store；模型需主动调用时再做 MCP/native tool。 |
| `base-web-search` | 有 `web.run` 和 hosted `web_search` | Codex 搜索绑定 OpenAI/Codex 搜索 API；没有 OpenClaw 的自定义 provider/rerank/full-text pipeline。 | 见下一节 web-search 补齐。 |
| `astronclaw_todo_create/update/complete/get` | 无 | Codex 没有 conversation todo 业务工具。 | MCP server 或 app-server `dynamic_tools`；深度接 thread 生命周期才做 native extension。 |
| `team_plan/provision/execute/complete/update_progress/cleanup` | 部分有 multi-agent，但无 AstronClaw team 语义 | Codex multi-agent 偏通用 sub-agent 调度，没有 AstronClaw team/task state machine。 | 简单版用 MCP/dynamic tools；要管 agent 生命周期则 fork/新增 native extension。 |
| `team_create_task/update_todo/finish_task` | 无同名工具 | Codex 没有 static-team 业务状态。 | MCP/dynamic tools。 |
| `qqbot_channel_api`、`qqbot_remind` | 无 | Codex 无 QQ channel runtime 和提醒业务工具。 | channel gateway + MCP/dynamic tools + skill。 |
| 微信/微博/企业微信/钉钉 channel | 无同款 channel runtime | Codex app-server 是通用宿主协议，不是 OpenClaw channel SDK。 | 外部 channel gateway 接 app-server thread/turn；发送/媒体/查询能力做 MCP 或 dynamic tools。 |
| AstronMem 云记忆 | 有 `memories.*` extension，但 backend 不同 | Codex memories 是自身扩展，不等于 AstronMem 云同步。 | 简单查询/写入用 MCP/dynamic tools；要原生召回则替换 `ext/memories` backend。 |

### Web-search 如何补齐

Codex 现有两套 web-search：

- Hosted `web_search`：`spec_plan.rs` 在 provider 支持 `web_search` 且 `web_search_mode != disabled` 时加入，由 `codex-rs/core/src/tools/hosted_spec.rs` 创建。
- Standalone namespace `web.run`：`codex-rs/ext/web-search/src/extension.rs` 在 `config.model_provider.is_openai()` 且 `web_search_mode != Disabled` 时贡献工具；`tool.rs` 暴露 `web.run`，实际通过 `SearchClient` 调 Codex/OpenAI 搜索 API。

如果要补齐 OpenClaw `base-web-search` 的 provider/rerank/full-text 能力，有三种路径：

1. 最小侵入：做一个自定义 MCP server，例如 `search.query`、`search.fetch`、`search.rerank`，再用 Codex plugin `mcpServers` 声明。优点是不用改 Codex core，适合替代 OpenClaw 搜索插件。
2. 平台级集成：新增 Rust native `ToolContributor`，仿照 `codex-rs/ext/web-search`，但 backend 换成自有搜索服务。优点是能接 Codex auth、telemetry、thread store；缺点是要编译进 app-server。
3. Channel/gateway 私有搜索：在 `thread/start.dynamicTools` 注入搜索工具，由 app-server client 执行。适合某个宿主、某个 channel 临时提供搜索能力。

不建议直接改 `codex-rs/core/src/tools/hosted_spec.rs` 来替换搜索，除非目标是 fork 出自有 Codex 发行版并彻底替换 OpenAI hosted tool。

### Namespace tool 清单

Codex 中 `ToolSpec::Namespace`/`ToolName::namespaced` 的内置来源包括：

| Namespace | 工具 | 来源 |
| --- | --- | --- |
| `web` | `run` | `codex-rs/ext/web-search/src/tool.rs`。 |
| `image_gen` | `imagegen` | `codex-rs/ext/image-generation/src/tool.rs`。 |
| `clock` | `curr_time` | `codex-rs/core/src/tools/handlers/current_time.rs`。 |
| `skills` | `list`、`read` | `codex-rs/ext/skills/src/tools/*`。 |
| `memories` | `add_ad_hoc_note`、`list`、`read`、`search` | `codex-rs/ext/memories/src/lib.rs`、`tools/*`。 |
| `multi_agent_v1` | `spawn_agent`、`send_input`、`resume_agent`、`wait_agent`、`close_agent` | `codex-rs/core/src/tools/handlers/multi_agents*`。 |
| 可配置 multi-agent v2 namespace | `spawn_agent`、`send_message`、`followup_task`、`wait_agent`、`interrupt_agent`、`list_agents` | `codex-rs/core/src/tools/handlers/multi_agents_v2/*`；也可作为 plain tools。 |
| MCP callable namespace | MCP server tools | `codex-rs/core/src/tools/handlers/mcp.rs`，由 MCP tool info 的 `callable_namespace` 决定。 |
| Dynamic namespace | app-server client 声明的 namespace tools | `codex-rs/core/src/tools/handlers/dynamic.rs`。 |

注意：hosted `web_search` 和 hosted `image_generation` 是 Responses hosted tool spec，不是 `web.run` 这种 namespace function tool。

### OpenAI/ChatGPT 强绑定点与替换清单

这部分只按本次调研范围统计：工具、插件、skills、MCP、app-server。`model-provider` 和 `login/auth` 是这些模块的底层依赖，不在表里单独列为模块，但会在“强绑定内容”中说明依赖关系。

| 模块 | 能力 | 强绑定内容 | 源码依据 | 替换建议 |
| --- | --- | --- | --- | --- |
| 工具 | Hosted `web_search` | Responses hosted web search，依赖 provider capability 和 `web_search_mode`。 | `codex-rs/core/src/tools/spec_plan.rs`、`codex-rs/core/src/tools/hosted_spec.rs`。 | 自有搜索不要改 hosted spec，优先 MCP/native `ToolContributor`/dynamic tools。 |
| 工具 | Standalone `web.run` | `config.model_provider.is_openai()`，通过 `SearchClient` 调 Codex/OpenAI 搜索 API。 | `codex-rs/ext/web-search/src/extension.rs`、`tool.rs`。 | 复制 extension 形态但替换 backend，或改成 MCP server。 |
| 工具 | Hosted/standalone image generation | 需要 `current_auth_uses_codex_backend()`，并依赖 OpenAI image API。 | `codex-rs/ext/image-generation/src/extension.rs`、`tool.rs`、`codex-rs/core/src/tools/spec_plan.rs`。 | 自有图片生成走自有 native extension/MCP；保留相同 `image_gen.imagegen` namespace 也可以，但 backend 必须替换。 |
| 工具 | `tool_search` / `request_plugin_install` | 搜索和安装建议会关联 marketplace/recommended plugins；remote plugin suggestion 可能走远端插件目录。 | `codex-rs/core/src/tools/handlers/tool_search.rs`、`request_plugin_install.rs`。 | 自有平台用本地 marketplace 或自有插件目录生成 discoverable tools，不直接依赖 OpenAI remote recommendation。 |
| MCP | Codex Apps MCP 文件上传 | Apps tool 的 `openai/fileParams` 会上传到 OpenAI file storage，且明确要求 ChatGPT/Codex backend auth。 | `codex-rs/core/src/mcp_openai_file.rs`。 | 自有生态媒体/附件不要走 OpenAI file upload；由 channel gateway/MCP server 自己处理文件存储和签名 URL。 |
| MCP | Apps/connector MCP 可用性 | app/connectors 会根据是否使用 Codex backend 裁剪。 | `codex-rs/core/src/session/turn_context.rs`、`session/session.rs` 中 `apps_enabled_for_auth(...)`。 | 自有平台如需 connector catalog，应实现自己的 app registry 和 auth gating，或先禁用 apps。 |
| 插件 | Remote plugin sync / curated remote plugin | 远端插件启用/卸载、推荐插件 cache、remote marketplace 依赖 remote service config 和 auth。 | `codex-rs/core-plugins/src/manager.rs`、`codex-rs/app-server/src/request_processors/plugins.rs`。 | 第一阶段只用 local marketplace/local plugin；后续替换为自有插件仓库和安装状态服务。 |
| 插件 | Plugin install 后的 app/MCP auth flow | 安装后会启动 plugin MCP OAuth、计算 `apps_needing_auth`。 | `codex-rs/app-server/src/request_processors/plugins.rs`。 | 自有插件系统需要重写 app auth/OAuth gating；业务 secret 放 gateway/MCP env。 |
| Skills | Orchestrator-owned skills | `skills.list/read` 只读 orchestrator-owned skills，并通过 Codex Apps MCP server authority 路由。 | `codex-rs/ext/skills/src/tools/mod.rs`、`provider/orchestrator.rs`。 | 自有平台如果不使用 Codex orchestrator，就关闭 orchestrator skills，改用 host/user/plugin skills 或自有 skill provider。 |
| Skills | Bundled/system skills | 部分系统 skill 面向 OpenAI/Codex 工作流，如 `openai-docs`、`plugin-creator`、`skill-installer`。 | `codex-rs/skills/src/assets/samples/*`、运行时 skill 列表。 | 自有发行版应审查系统 skills，替换 OpenAI 文档、插件市场、安装源说明。 |
| App-server | Account/login/rate limit/usage/workspace messages | ChatGPT 登录、账户、限额、用量、workspace message、credit nudge。 | `codex-rs/app-server/src/request_processors/account_processor.rs`、`app-server/README.md`。 | 替换成自有账号中心和计费服务；不需要时隐藏/禁用这些 RPC。 |
| App-server | Cloud config bundle | app-server 可用 ChatGPT auth 加载云端配置 bundle。 | `codex-rs/app-server/src/config_manager.rs`、`codex_cloud_config::cloud_config_bundle_loader`。 | 自有平台应提供自己的 `ThreadConfigLoader`/config bundle loader，或使用本地 config-only 模式。 |
| App-server | Feedback upload | 反馈上传会关联 ChatGPT user/account 信息，并走 Codex feedback 管线。 | `codex-rs/app-server/src/request_processors/feedback_processor.rs`。 | 替换为自有反馈/日志上传接口；或关闭 `feedback_enabled`。 |
| App-server | Attestation / Guardian 宿主能力 | app-server/core 有 attestation provider、Guardian review/session 机制，部分能力面向 Codex 托管安全链路。 | `codex-rs/core/src/session/mod.rs`、`thread_manager.rs`、`guardian/*`、`codex-rs/ext/guardian/src/lib.rs`。 | 自有平台要么关闭相关 feature，要么实现自己的审批/风控 agent 和 attestation provider。 |

替换优先级建议：

1. 工具：先替换 web search、image generation、插件推荐/安装建议工具。
2. MCP：再替换 Apps MCP、OpenAI file upload、connector auth gating。
3. 插件/skills：改成本地 marketplace、自有插件仓库、自有 skill provider，审查系统 skills。
4. App-server：最后替换 account、cloud config、feedback、Guardian/attestation 等平台服务。

### 工具如何横向扩展

推荐顺序：

1. 外部业务 API：MCP server。适合微信、微博、QQ、钉钉、企业微信、搜索、todo、CRM、内部系统；plugin manifest 用 `mcpServers` 声明。
2. 宿主/会话级工具：app-server `dynamic_tools`。适合 channel gateway 临时注入“发消息、上传媒体、查上下文”等工具，避免改 Codex core。
3. 平台级深度工具：native `ToolContributor`。适合要读写 Codex thread/session store、接 telemetry、替换 memory/search/image backend、管理 agent 生命周期的功能。
4. 真正通用核心工具：core handler。只建议 fork 自有 Codex 框架时少量加入，避免污染上游升级面。

### Skills 安装、卸载与生态接入

Codex 的 skill 不是工具执行 runtime，本质是 `SKILL.md` 指令和附属资源：

- `ext/skills` 暴露 `skills.list`、`skills.read`，用于发现/读取 orchestrator-owned skills，不负责安装卸载。
- plugin 可以通过 manifest `skills` 字段打包 skills；`core-plugins/src/loader.rs` 会加载 plugin skills。
- plugin 安装/卸载由 app-server `plugin/install`、`plugin/uninstall` 触发，最终进入 `core-plugins/src/manager.rs` 的 `install_plugin` / `uninstall_plugin`。
- 单独 skill 也可以按文件系统方式放入 Codex skills 目录；内置 `skill-installer` skill 是操作说明/脚本型能力，不是 Rust runtime API。

微信、微博等生态接入的最小侵入形态：

1. channel gateway 独立进程/服务：负责 webhook、鉴权、媒体、回调、重试、平台限流。
2. gateway 调 app-server：把外部消息转换为 Codex thread/turn；Codex 不直接实现微信/微博 SDK。
3. gateway 或业务服务提供 MCP/dynamic tools：发送消息、上传媒体、查联系人、查群、查热搜、发布微博等。
4. skills 只写“如何使用这些工具”的流程说明，随 plugin 打包；不要把 token、secret、业务状态写进 skill。
5. 可观测性用 hooks/app-server telemetry 统一埋点；业务系统自己记录 channel message id、平台错误码、重试状态。

开发平台接入建议：

- 第一阶段：local marketplace + plugin manifest + MCP + skills + hooks，先跑通，不改 core。
- 第二阶段：app-server gateway 承接 channel 和账号体系，把动态能力通过 `dynamic_tools` 暴露。
- 第三阶段：只有当需要替换 OpenAI/Codex 云端账号、memory backend、search backend、multi-agent 调度、plugin remote service 时，才 fork/新增 native extension 或改 app-server。

### 平台插件迁移评估

| OpenClaw 平台能力 | Codex 可行落点 | 可行性 | 依据和限制 |
| --- | --- | --- | --- |
| artifact capture | `PostToolUse` / `Stop` hook | 高 | `PostToolUseRequest` 有 `tool_input`、`tool_response`、`transcript_path`；`Stop` 可在回合结束后汇总。 |
| trace / telemetry | `PreToolUse`、`PostToolUse`、`SessionStart`、`Stop` hook | 高 | hook 事件覆盖会话开始、工具前后和结束；适合异步 command hook 上报。 |
| learning loop | `PostToolUse` / `Stop` hook + 外部服务 | 中高 | hook 可收集数据和反馈，但自动修改 skills/策略需要外部存储、审核和版本管理。 |
| conversation todo | MCP tools、dynamic tools 或 skill | 高 | 外部服务用 MCP；如果由 app-server client/gateway 承接，可用 dynamic tools；状态存在 MCP server、gateway 或外部 DB。 |
| team/static-team | MCP tools、dynamic tools 或 native multi-agent extension | 中 | 简单任务管理用 MCP/dynamic tools；如果要接 Codex multi-agent 生命周期，需要 native extension。 |
| AstronMem 云记忆 | MCP、dynamic tools 或替换 `codex-rs/ext/memories` backend | 中 | 只做查询/写入可 MCP/dynamic tools；要变成 Codex 原生记忆召回，需要改 memories extension。 |
| agent relay / gateway | app-server 外部 gateway 或 native extension | 中 | 如果只是外部设备/云端转发，可 gateway；如果要控制 session router、auth、租户、调度，应改 app-server。 |
| deduct / billing | app-server/native platform layer | 中 | 不建议作为模型工具或普通 hook；计费应在 gateway/app-server/service 层做强制执行。 |

### Channel 迁移评估

Channel 大概率应通过 app-server 迁移，这个判断有源码依据。

推荐路径：

```text
企业微信/钉钉/微信/微博/QQ Bot
  -> channel gateway
  -> app-server JSON-RPC
     1. thread/start 或 thread/resume
     2. turn/start 提交 UserInput
     3. 订阅/读取 thread events 或 thread/read
  -> gateway 把 Codex 输出发回外部 channel
```

普通文本/图片 channel：

- 创建或恢复会话：`thread/start` / `thread/resume`。
- 发送用户消息：`turn/start`，输入类型用 `UserInput::Text`、`Image` 或 `LocalImage`。
- 追加活跃回合 steering：`turn/steer`，但必须带 `expected_turn_id`，只适合已有活跃 turn。
- 读取结果：订阅 server notification，或用 `thread/read` / `thread/turns/list` 做补偿读取。
- 导入历史：可用 `thread/inject_items`，但它不会启动 turn，不适合作为实时消息入口。

实时语音/流式 channel：

- 可评估 `thread/realtime/start`、`thread/realtime/appendText`、`thread/realtime/appendAudio`、`thread/realtime/appendSpeech`。
- 源码标注为 experimental；生产迁移应先用普通 `turn/start` 跑通，再单独评估 realtime 稳定性。

Channel 不建议迁成 hook，原因：

- hook 是 Codex 内部生命周期回调，不是外部消息入口。
- `UserPromptSubmit` 能拦截用户 prompt，但前提是 prompt 已经进入 Codex turn 流程。
- 外部 channel 的账号鉴权、消息去重、媒体下载、会话映射、重试、回执都应在 gateway 处理。

Channel plugin 可以打包为 Codex plugin，但 plugin 只负责声明：

- `skills`：告诉 agent 如何使用该 channel。
- `mcpServers`：暴露发送消息、上传媒体、查询联系人等工具。
- `hooks`：做 trace、artifact、策略检查。
- `apps`：如果要出现在 Codex connector/app 管理面里，声明 connector metadata。

真正 inbound channel runtime 仍建议在 app-server 外部。

## 推荐迁移架构

### 轻量迁移：保留能力，不 fork Codex core

适合先把 OpenClaw 工具和 skills 跑起来。

```text
Codex
  loads plugin.json
  -> loads skills
  -> starts MCP servers
  -> runs hooks

External services
  -> channel gateways
  -> business APIs
  -> memory/search/todo/team services
```

落地方式：

1. 每个业务域一个 Codex plugin。
2. 每个 plugin 里放 `.mcp.json`、`skills/`、必要的 `hooks/`。
3. OpenClaw tool 按场景选择 MCP server、native ToolContributor 或 app-server dynamic tools。
4. OpenClaw channel 逐个移植为外部 gateway。
5. OpenClaw cloud 配置改为 MCP env 或 gateway env。

优点：

- 不需要改 Codex core。
- 升级 Codex 成本低。
- 工具隔离清晰。

缺点：

- 和 Codex 内部 multi-agent、goal、memory 的深度整合较弱。

### 平台迁移：基于 Codex 做自己的 agent 框架

适合你明确要做自己的 agent 平台、替换云端相关能力。

应重点改或替换：

| 区域 | Codex 位置 | 为什么要改 |
| --- | --- | --- |
| hosted apps / connectors MCP | `codex-rs/ext/mcp/src/lib.rs`, `codex-rs/core/src/mcp.rs`, `codex-rs/codex-mcp/src/mcp/mod.rs` | 默认面向 Codex/OpenAI backend，URL、token、auth gating 都可能不适合自有平台。 |
| auth/backend 配置 | `codex-rs/core/src/config`, `codex-rs/config` | 自有平台需要自己的鉴权、租户、endpoint。 |
| plugin marketplace / 安装 | `codex-rs/plugin`, `codex-rs/core-plugins` | 自有插件仓库、版本、签名、cache 策略可能不同。 |
| skills provider | `codex-rs/ext/skills`, `codex-rs/core-skills` | 自有 skill registry、内置 skill、权限和来源策略需要替换。 |
| memories backend | `codex-rs/ext/memories` | 如果使用 AstronMem 或自有记忆服务，需要换 backend。 |
| multi-agent / jobs | `codex-rs/core/src/tools/handlers/multi_agents_v2`, `agent_jobs` | OpenClaw 的 team/relay/taskflow 若要成为平台原生能力，需要接这里。 |
| hooks/telemetry | `codex-rs/hooks`, app-server extension | trace、artifact、learning loop 需要更强生命周期事件时要改。 |

不建议改：

- 为单个业务 API 修改 `codex-rs/core/src/tools/handlers`。
- 为单个 channel 把 OpenClaw JS channel runtime 嵌入 Codex core。
- 把云端 secret 写进 skill 或 plugin manifest。

## 各能力的推荐迁移顺序

### 第一阶段：低风险直接迁移

1. `conversation-todo-sync`
2. `base-web-search-tool`
3. `static-team`
4. `agent-team` 的 skill 和基础 tools
5. 可独立运行的全局 skills，例如 `github`、`tmux`、`video-frames`、`summarize`

原因：这些能力主要是工具或指令，和 OpenClaw channel/cloud runtime 耦合较低。

### 第二阶段：channel gateway

1. `qqbot`
2. `wecom`
3. `dingtalk`
4. `openclaw-weixin`
5. `weibo`
6. `astron-claw`

原因：这些不能按 Codex 插件原样迁移，需要先确定外部消息如何映射到 Codex thread/session，以及 Codex 如何回写消息。

### 第三阶段：平台能力替换

1. `agent-relay`
2. `astronclaw-gateway`
3. `astronmem-cloud-openclaw-plugin`
4. `claw-core-deduct-api`
5. `openclaw-astronclaw-trace`
6. `astronclaw-learning-loop`
7. `astronclaw-plugins` artifact pipeline

原因：这些涉及云端、计费、记忆、trace、artifact、学习闭环，是自有 agent 框架的核心平台层，不应简单照搬。

## 公开 Codex / MCP 支持情况

本节补充公开搜索结果，用来判断哪些 OpenClaw 内置或第三方能力可以直接换成已有 Codex/MCP 方案。

### 已找到明确支持 Codex 或可直接接 Codex 的官方版本

| 能力 | 公开支持情况 | 对当前迁移的影响 |
| --- | --- | --- |
| GitHub | GitHub 有 `github/github-mcp-server`，仓库说明为 GitHub official MCP Server，并在安装说明中列出 Codex 安装指南。 | OpenClaw 的 `github`、`gh-issues` skill 不必完整移植工具层；优先接 GitHub MCP，再保留/改写 skill 工作流。 |
| Figma | Figma 官方 MCP server 文档明确列出 Codex by OpenAI，支持 remote/desktop server，并提供 Figma skills for MCP。 | 如果后续要迁移设计/前端相关 skill，应优先用 Figma 官方 MCP + Figma skills，不建议自写 Figma tool。 |
| 1Password | 1Password 官方文档提供 1Password MCP Server beta，并明确写到 MCP clients such as Codex。 | OpenClaw 的 `1password` skill 不应只按 CLI skill 迁移；优先接 1Password 官方 MCP Server，CLI 作为 fallback。 |
| MCP reference servers | Model Context Protocol 官方 examples 列出 `filesystem`、`git`、`memory`、`time`、`fetch` 等 reference servers。 | OpenClaw 中通用文件、Git、记忆、时间、fetch 类能力不需要从 OpenClaw 搬工具；可直接配 Codex MCP。 |

参考链接：

- GitHub official MCP Server: `https://github.com/github/github-mcp-server`
- Figma MCP server: `https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server`
- 1Password MCP Server: `https://www.1password.dev/environments/mcp-server`
- MCP example servers: `https://modelcontextprotocol.io/examples`

### 找到 MCP 生态替代，但不是对应厂商官方 Codex 插件

| 能力 | 公开支持情况 | 对当前迁移的影响 |
| --- | --- | --- |
| Slack | MCP 官方 reference 里旧 Slack server 已归档，并标注现在由 Zencoder 维护；另有 Slackbot 支持 MCP 的新闻，但这不是 OpenClaw `slack` skill 的直接 Codex 插件替代。 | 可以用 MCP 方案替代一部分 Slack 操作，但要审查维护方、权限和工具范围。 |
| Brave Search / web search | MCP reference 曾有 Brave Search server，并标注已被 official server 替代；Codex 自身也有 `web.run` / hosted web search。 | `base-web-search-tool` 不应优先移植；先用 Codex 内置搜索或官方搜索 MCP。 |

### 未找到公开 Codex/MCP 官方版本的 OpenClaw channel 插件

以下插件/能力本次没有搜到明确的官方 Codex 版本、`.codex-plugin` 版本或厂商官方 MCP server：

- `@wecom/wecom-openclaw-plugin`
- `@soimy/dingtalk`
- `@tencent-weixin/openclaw-weixin`
- `@wecode-ai/weibo-openclaw-plugin`
- `@openclaw/qqbot`

判断：

- 这些仍应按“外部 channel gateway + MCP tools + Codex skills”迁移。
- 不能假设它们已经有可直接安装到 Codex 的官方版本。
- 如果后续发现厂商发布了 MCP server，应优先接官方 MCP，而不是继续维护 OpenClaw channel runtime 的移植版。

### 查询限制

本次 npm registry 直查在当前环境中没有及时返回，因此没有把 npm registry metadata 作为依据。结论主要来自公开 web 搜索、官方 GitHub/文档页面，以及本地容器内插件 manifest。

## Codex plugin 打包模板

一个迁移后的插件建议这样组织：

```text
my-openclaw-migrated-plugin/
  .codex-plugin/
    plugin.json
  .mcp.json
  hooks/
    hooks.json
    post-tool-use.sh
  skills/
    my-skill/
      SKILL.md
      references/
      scripts/
  server/
    package.json
    src/
      index.ts
```

`plugin.json`：

```json
{
  "name": "my-openclaw-migrated-plugin",
  "version": "0.1.0",
  "description": "Migrated OpenClaw capability for Codex",
  "skills": "./skills",
  "mcpServers": "./.mcp.json",
  "hooks": "./hooks/hooks.json"
}
```

`.mcp.json`：

```json
{
  "mcpServers": {
    "my-openclaw-tools": {
      "command": "node",
      "args": ["./server/dist/index.js"],
      "env": {
        "MY_SERVICE_BASE_URL": "https://api.example.com"
      },
      "enabled": true,
      "tool_timeout_sec": 60
    }
  }
}
```

`hooks/hooks.json`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "./hooks/post-tool-use.sh"
          }
        ]
      }
    ]
  }
}
```

## 迁移检查清单

迁移一个 OpenClaw 插件时逐项检查：

1. Manifest：记录 `id`、`type`、`activation`、`channels`、`skills`、`contracts`、`configSchema`。
2. 工具：找 `api.registerTool` 和 manifest `contracts.tools`，逐个决定迁为 MCP、native `ToolContributor` 还是 app-server `dynamic_tools`，并为所选路径编写对应 schema/实现。
3. Skills：读取完整 `SKILL.md`，检查引用的脚本、资源和 OpenClaw 专有命令。
4. Channel：把 inbound/outbound、鉴权、媒体、状态存储拆到 gateway。
5. Hooks：找 `registerToolHooks` 或 trace/artifact 逻辑，映射到 Codex hook event。
6. Config：secret 放 env；非 secret 放 Codex config 或 gateway config；不要放进 skill。
7. State：OpenClaw 的 `/root/.openclaw/...` 状态路径改成 Codex 可控路径或外部 DB。
8. 权限：MCP tool 的高风险动作设置 approval mode。
9. 测试：先测 MCP tool，再测 skill 调用说明，再测 channel gateway。

## 本次最重要的迁移判断

- OpenClaw 的工具可以改，但不要优先改 Codex 内置 core tool。外部业务工具先封装成 MCP；平台深度工具走 native ToolContributor；宿主会话级工具走 dynamic tools。
- OpenClaw 的 skills 大多可以改并迁移，但要先读完整 `SKILL.md`，不能只搬目录名。
- OpenClaw 的 MCP 当前不是一组现成可复制的 server 配置；需要按工具主动生成 MCP server。
- OpenClaw 的 channel 插件不能直接迁成 Codex plugin runtime；应改成外部 gateway，再用 MCP/apps/skills 接入。
- OpenClaw 的云端 relay、gateway、deduct、AstronMem、trace、artifact、learning loop 是做自有 agent 框架时真正需要替换的部分。
