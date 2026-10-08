# 桌面端 AI 编码 Agent 技术方案

> 状态：待评审 | 基线：基于 openai/codex 开源仓库（codex-rs，Apache-2.0）构建自有产品
> 目标平台：Windows + macOS 双端同等优先，Linux 预留

## 1. 背景与目标

基于 codex-rs 的 `app-server` 能力，打造自有品牌的桌面端 AI 编码 Agent：

- Agent 核心能力（会话编排、工具执行、沙箱、审批）100% 复用官方 `codex app-server`，不重写
- **不使用 ChatGPT 登录**，账号体系走自有 OAuth
- 行为定制深度为 **L2（行为配置）**：通过配置定制行为，不 fork codex-rs 源码
- 部署形态为**纯桌面端**：agent 跑在用户本机，本期不搭云端执行

### 已确认的决策记录

| 决策项 | 结论 |
|---|---|
| 技术路线 | Tauri 2.x 壳 + `codex app-server` 子进程（路线 B，产品级） |
| 行为定制 | L2：config 层配置（审批策略、MCP 预置、模型目录等），不 fork |
| 账号体系 | 自有 OAuth（authorization code 流程），不用 ChatGPT OAuth |
| 模型接入 | OpenAI 兼容协议（自选服务商/自建网关），`wire_api` 可配 |
| 部署 | 纯桌面，零服务器；未来云执行用现成 `exec-server` 扩展 |
| 平台 | Windows + macOS 双端同等优先（M4 同时交付），Linux 预留 |
| 二次开发许可 | codex-rs 为 Apache-2.0，保留 NOTICE 即可 |

## 2. 总体架构

```
┌─ Tauri App（自有品牌）───────────────────────────────┐
│  WebView 前端 (React + TS)                            │
│   登录页 · 聊天流 · diff 视图 · 审批弹窗 · 设置       │
│        ▲ Tauri IPC / event                            │
│  Rust host                                            │
│   ├─ Supervisor   codex 进程守护·版本检测·崩溃恢复    │
│   ├─ RpcClient    JSON-RPC 2.0 封装·通知分发·节流     │
│   ├─ SessionMgr   thread 生命周期·多窗口共享单后端    │
│   └─ ConfigWriter 生成/维护 ~/.codex/config.toml      │
└──────────────┬────────────────────────────────────────┘
               │ stdio（按行 JSON）
        codex app-server（随 App 分发的官方二进制）
               │
   ├─ 模型 API ←→ 自有模型网关（自有 OAuth 守门）
   ├─ 沙箱执行（macOS Seatbelt / Windows restricted-token·MXC）
   ├─ MCP 工具 / skills / hooks
   └─ rollout 会话持久化（SQLite，自动）
```

## 3. 账号与鉴权设计（自有 OAuth）

### 3.1 推荐路径 A：codex 原生 `gateway_oauth`（自有 OAuth 守门模型网关）

codex-rs 原生支持把模型网关的鉴权交给一个标准 OAuth2 authorization code 服务
（见 `codex-rs/model-provider-info/src/gateway_oauth.rs` 的 `GatewayOAuthConfig`）：

```toml
# ~/.codex/config.toml（由 App 的 ConfigWriter 生成）
model = "your-model"
model_provider = "my-gateway"

[model_providers.my-gateway]
name = "My Agent Gateway"
base_url = "https://gateway.your-domain.com/v1"
wire_api = "chat"

[model_providers.my-gateway.gateway_oauth]
authorization_url = "https://accounts.your-domain.com/oauth2/authorize"
token_url = "https://accounts.your-domain.com/oauth2/token"
client_id = "<desktop-app-client-id>"
scopes = ["model:invoke"]
redirect_port = 1455
delivery = { kind = "header", name = "Authorization", scheme = "Bearer" }
```

登录流程（全部用 app-server 现成 RPC，App 只做 UI）：

```
initialize（capabilities 显式声明 explicitGatewayOauth: true）
→ account/gatewayOAuth/read     查询登录状态（required / notReady / succeeded）
→ account/gatewayOAuth/login    拿到 authUrl → 系统浏览器打开自有登录页
→ 用户登录自有账号 → OAuth 回调 redirect_port → app-server 保存凭证
← account/gatewayOAuth/changed  通知（started/succeeded/failed）
→ account/gatewayOAuth/read     确认 succeeded，进入主界面
```

优点：

- token 的获取、存储、**自动刷新、失效撤销**全部由 app-server 托管，App 零维护
- 登录即额度：账号体系与模型网关天然绑定（一个账号一份模型额度）
- 上游协议有明确的产品级契约（含错误码、取消、多连接语义）

对自有服务端的要求：标准 OAuth2 authorization code 端点 + token 端点；
token 通过 HTTP Header（默认 Bearer）或 Cookie 注入模型请求。

### 3.2 备选路径 B：App 层自有账号（OAuth 只管 App，不管模型）

若自有 OAuth 仅用于 App 登录/订阅管理，模型走第三方 API key：

- App 自行完成 OAuth（WebView 或系统浏览器），token 存系统钥匙串
- spawn app-server 时把模型 key 以环境变量注入（对应 provider 的 `env_key`）
- token 刷新逻辑在 App 侧维护

适合"账号管订阅计费、模型完全外采"的形态。两条路径可并存：
A 管模型鉴权，B 管应用账号。

### 3.3 凭证存储

- 模型网关 token：app-server 自管（路径 A 无需处理）
- App 账号 token / API key（路径 B）：系统钥匙串（macOS Keychain / Windows
  Credential Manager，参考 codex-rs `keyring-store` crate 的做法），绝不写入明文配置

## 4. 模型接入设计

| 来源 | 配置方式 | 阶段 |
|---|---|---|
| 自有网关（OpenAI 兼容） | `base_url` + `gateway_oauth`，`wire_api = "chat"` | 主力方案 |
| 第三方 OpenAI 兼容（DeepSeek/GLM/Qwen 等） | `base_url` + `env_key` | 备选/高级设置 |
| 本地模型（Ollama / LM Studio） | 原生支持，自动发现 | 隐私/离线卖点 |
| 自有模型目录 | provider 的 `model_catalog_url` 指向自有目录服务 | L2 定制关键点 |

L2 定制要点：`model_catalog_url` 让 `model/list` 返回**自己的模型列表**，
配合自有网关实现"模型市场"式产品体验，无需 fork。

## 5. 模块划分

### Rust 侧（src-tauri，4 个模块）

| 模块 | 职责 | 要点 |
|---|---|---|
| `supervisor` | 启动/监控 `codex app-server` | 指数退避重启 → 最近 threadId `thread/resume`；启动前 `--version` 校验 |
| `rpc` | JSON-RPC 2.0 客户端 | 行分帧；请求 id ↔ pending map；通知多路分发；initialize 时声明 `explicitGatewayOauth` |
| `session` | thread 生命周期 | 多窗口共享单 backend；thread 按 id 路由；会话列表分页 |
| `config` | 配置生成器 | 首启写 `config.toml`（provider/gateway_oauth/审批策略默认值）；key 不落明文 |

### 前端（React + TS）

| 模块 | 内容 |
|---|---|
| Auth | 自有 OAuth 登录页（调 gatewayOAuth RPC + 打开浏览器）、登录态展示 |
| Chat | 消息流、工具调用卡片、markdown、delta 150ms 批量渲染 |
| Approval | exec 批准/拒绝、apply-patch 确认、沙箱提权（一等 UI，非确认框） |
| Onboarding | 选工作目录 + project trust 确认 + 模型来源向导（网关已登录则跳过） |
| Sessions | `thread/list` 历史、`thread/resume` 恢复、归档 |
| Settings | 模型（自有 catalog）、沙箱模式、MCP 服务器管理 |

## 6. 协议对接（关键流程）

类型对齐 `codex-rs/app-server-protocol`（上游有 ts-rs 导出可参考）。

### ① 启动握手

```
spawn codex app-server
→ initialize { clientInfo, capabilities: { explicitGatewayOauth: true } }
→ configRequirements/read · model/list（自有 catalog）
→ account/gatewayOAuth/read（登录态检查，未登录走 ②）
```

### ② 登录（自有 OAuth）

见 3.1 流程图。登录取消用 `account/gatewayOAuth/cancel`；监听 `account/gatewayOAuth/changed`。

### ③ 对话主循环

```
thread/start { cwd, approvalPolicy, sandboxMode }
  → thread/started { threadId }
turn/start { input }
  ← item/* 通知流（agent 消息 delta、命令执行、文件修改）
  ← execCommandApproval / applyPatchApproval（服务端请求 → 审批 UI → 响应）
  ← turn/completed { usage }
```

审批请求/响应类型：`ExecCommandApprovalParams/Response`、
`ApplyPatchApprovalParams/Response`（app-server-protocol/src/protocol/v1.rs）。

### ④ 会话持久化

后端自动写 rollout/SQLite；前端只调 `thread/list`（分页、归档过滤）+ `thread/resume`。

### ⑤ 工具管理

`mcpServerStatus/list`、`mcpServer/oauth/login`、`config/mcpServer/reload`。

## 7. L2 行为配置清单（不 fork 可达的定制面）

| 定制项 | 配置位置 |
|---|---|
| 默认审批策略 / 沙箱模式 | `approval_policy` / `sandbox_mode`（config.toml） |
| 预置 MCP 服务器 | `mcp_servers`（config.toml） |
| 项目级行为提示 | AGENTS.md / 项目级 `.codex` 目录 |
| 模型列表（"模型市场"） | `model_providers.*.model_catalog_url` |
| 特性开关 | `features` + `experimentalFeature/enablement/set` |
| 网关鉴权方式 | `gateway_oauth.delivery`（Header/Cookie） |

L3（改系统提示词/内置工具/agent 流程）明确不在本期，需 fork 或 extension API，另立项。

## 8. 里程碑

| 阶段 | 交付 | 验收标准 |
|---|---|---|
| M1 跑通闭环 | spawn + 握手 + 自有 OAuth 登录 + 对话流渲染 | 选目录 → 登录 → 对话改一个文件成功 |
| M2 安全闭环 | 审批弹窗 + 沙箱模式 + trust + 钥匙串存储 | 默认只读沙箱下命令需逐条批准 |
| M3 产品化 | 历史会话/恢复、diff 视图、MCP/模型设置页 | 重启 App 恢复上次会话 |
| M4 发布 | macOS（.dmg）+ Windows 安装包、codex 二进制更新通道、崩溃恢复 | 两端全新机器一键安装可用 |

## 9. 风险与对策

| 风险 | 对策 |
|---|---|
| app-server 协议演进快 | 二进制与 App 解耦 + 锁定支持版本区间；RPC 层版本探测；不依赖 experimental API（gateway_oauth 除外，它有明确契约） |
| 自有 OAuth 服务端适配 | M1 前先用最小 OAuth 服务（标准 code flow）联调；delivery 先用 Header+Bearer |
| WebView 依赖 | Windows 安装包内置 WebView2 引导；macOS 用系统 WKWebView，零依赖 |
| 用户环境差异（git/PATH） | 首启环境自检（参考 cli 的 doctor 子命令思路） |
| 长会话膨胀 | 交给后端 compaction/rollout；前端只渲染可视窗口 |
| 跨平台沙箱差异 | macOS 走 Seatbelt 开箱即用（最接近官方默认行为）；Windows 设置页暴露沙箱模式（restricted-token / MXC），差异不透传给用户 |
| macOS 签名与公证 | 需要 Apple Developer ID + notarization（否则 Gatekeeper 拦截）；随包分发的 codex 二进制同样要过公证；CI 需 macOS 构建机 |

## 10. 待确认项

1. 自有 OAuth 服务端是否已具备 authorization code + token 端点？（决定 M1 能否直接联调）
2. 模型网关与 OAuth 的关系：token 直接守网关（路径 A）还是账号只管订阅（路径 B）？
3. 项目落位：建议新建 `d:\codex-desktop`（与 codex 仓库并列）。

## 附：关键代码索引（codex-rs）

| 内容 | 位置 |
|---|---|
| app-server 启动与 transport（stdio/ws） | `codex-rs/app-server/src/main.rs` |
| JSON-RPC 协议类型 | `codex-rs/app-server-protocol/src/protocol/v1.rs` |
| 自定义 provider 结构 | `codex-rs/model-provider-info/src/lib.rs` |
| 自有 OAuth 网关配置 | `codex-rs/model-provider-info/src/gateway_oauth.rs` |
| 审批策略类型 | `codex-rs/protocol/src/`（AskForApproval 等） |
| 远程执行（未来扩展） | `codex-rs/exec-server/README.md` |
| 环境自检参考 | `codex-rs/cli/src/doctor.rs` |
