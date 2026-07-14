# 沙箱设计

> 为 MCP 服务器和外部工具调用提供进程级隔离，防止 prompt 注入诱导的越权访问。
> 设计对齐 12-Factor：配置走环境变量/config、进程无状态、日志外发、dev/prod 一致。

## 1. 三层沙箱模型

```
┌─────────────────────────────────────────────────────┐
│  Agent 进程（sandbox_mode = workspace-write）       │
│  ┌───────────────────────────────────────────────┐  │
│  │ MCP Sandbox                                  │  │
│  │  ├─ filesystem: allowlist                    │  │
│  │  ├─ network: allowlist                       │  │
│  │  ├─ env: 仅白名单变量注入                    │  │
│  │  └─ capabilities: read/exec, 限制 write      │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │ Tool Sandbox（命令执行）                     │  │
│  │  ├─ pre_exec hook 阻断危险命令               │  │
│  │  └─ OS 级 wrapper（可选）                    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

| 层 | 对象 | 机制 | 配置位置 |
|----|------|------|---------|
| L1 | Agent 进程 | Trae `sandbox_mode` | config.toml |
| L2 | MCP 服务器 | per-MCP sandbox 段 + wrapper 脚本 | config.toml |
| L3 | 工具命令 | pre_exec hook + 可选 OS sandbox | hooks.json + hooks/ |

## 2. per-MCP 沙箱配置 schema

每个 MCP server 可选附加 `[mcp_servers.<name>.sandbox]` 段：

```toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "${PWD}"]
enabled = false

[mcp_servers.filesystem.sandbox]
mode = "read-only"                    # read-only / workspace-write / none
filesystem = ["./"]                   # 允许访问的路径白名单
network = "none"                      # none / allowlist / all
network_allowlist = []                # 网络白名单主机
env_allowlist = ["PWD"]               # 仅注入白名单环境变量
timeout = 60                          # 单次调用超时（秒）
max_memory_mb = 256                   # 内存上限
log_level = "info"                    # stdout 日志级别
```

### 字段说明

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `mode` | string | `none` | `read-only` 文件只读 / `workspace-write` 工作区可写 / `none` 不限制 |
| `filesystem` | string[] | `[]` | 允许访问的路径前缀（相对项目根），空数组=完全禁止文件 |
| `network` | string | `all` | `none` 完全禁网 / `allowlist` 仅白名单 / `all` 不限制 |
| `network_allowlist` | string[] | `[]` | 当 network=allowlist 时生效，域名或 CIDR |
| `env_allowlist` | string[] | `[]` | 仅这些环境变量会传给 MCP 子进程 |
| `timeout` | int | 60 | 单次工具调用超时秒数 |
| `max_memory_mb` | int | 256 | 子进程内存上限 |
| `log_level` | string | `info` | MCP stdout 日志级别 |

## 3. 默认策略矩阵

按最小权限原则预设每个 MCP 的默认沙箱：

| MCP | mode | filesystem | network | env 白名单 |
|-----|------|-----------|---------|-----------|
| filesystem | read-only | `["./"]` | none | `["PWD"]` |
| github | workspace-write | `["./"]` | allowlist: `github.com`, `api.github.com` | `["GITHUB_PERSONAL_ACCESS_TOKEN"]` |
| postgres | read-only | `[]` | allowlist: `${PG_HOST}` | `["DATABASE_URL"]` |
| playwright | none | `[]` | all | `[]` |
| context7 | none | `[]` | allowlist: `context7.com` | `[]` |
| fetch | none | `[]` | all | `[]` |
| memory | workspace-write | `["~/.trae/mcp-memory/"]` | none | `[]` |
| brave_search | none | `[]` | allowlist: `api.search.brave.com` | `["BRAVE_API_KEY"]` |
| sentry | none | `[]` | allowlist: `sentry.io` | `["SENTRY_AUTH_TOKEN"]` |
| linear | none | `[]` | allowlist: `api.linear.app` | `["LINEAR_API_KEY"]` |
| slack | none | `[]` | allowlist: `slack.com`, `www.slack.com` | `["SLACK_BOT_TOKEN"]` |

## 4. 实施机制

### 4.1 路径 A：Trae 原生支持（理想）
若 Trae 支持 `[mcp_servers.<name>.sandbox]` 字段，直接读取配置生效。

### 4.2 路径 B：OS 级 wrapper（兼容，立即可用）

通过 wrapper 脚本启动 MCP 子进程，调用 OS 沙箱工具：

| 平台 | 工具 | 可用性 |
|------|------|--------|
| macOS | `sandbox-exec` | 系统自带 |
| Linux | `bwrap`（bubblewrap） | 需安装 |
| Linux 备选 | `firejail` | 需安装 |
| 通用 | Docker | 重但隔离最强 |

config.toml 改为：
```toml
[mcp_servers.filesystem]
command = ".trae/sandbox/run.sh"
args = ["filesystem", "npx", "-y", "@modelcontextprotocol/server-filesystem", "${PWD}"]
```

`run.sh` 读取对应 MCP 的 sandbox 配置，生成 sandbox-exec profile 并启动。

## 5. macOS sandbox-exec profile

`.trae/sandbox/profiles/` 下提供 3 个模板：

| Profile | 用途 |
|---------|------|
| `readonly.sb` | 文件只读，无网络 |
| `workspace.sb` | 工作区可写，无网络 |
| `network.sb` | 网络白名单（include） |

`run.sh` 根据配置组合这些 profile。

### 5.1 readonly.sb 核心规则

```
(deny file-write*)
(allow file-read* (subpath "${WORKSPACE}"))
(deny network*)
(allow process-info* sysctl-read)
(allow signal)
```

### 5.2 workspace.sb 核心规则

```
(allow file-write* (subpath "${WORKSPACE}"))
(allow file-read* (subpath "${WORKSPACE}"))
(deny network*)
```

### 5.3 network.sb 核心规则

```
(allow network-outbound (remote tcp "${HOST}" 443))
(allow network-outbound (remote tcp "${HOST}" 80))
(deny network-outbound)
```

## 6. 与 12-Factor 的对齐

| Factor | 实施 |
|--------|------|
| III 配置 | 沙箱策略全在 config.toml，不硬编码 |
| IV 支撑服务 | MCP 是 attached resource，sandbox 是连接契约 |
| VI 进程 | MCP 进程无状态，沙箱配置即进程级 |
| VIII 并发 | 每个 MCP 独立进程，沙箱互不影响 |
| XI 日志 | MCP stdout/stderr → Agent → 采集层，沙箱不存日志 |
| XII 管理进程 | 沙箱配置 dev/prod 一致，无特殊 daemon |

## 7. 与规则体系的关系

- 沙箱配置属于**配置层**（config.toml），优先级低于规则层
- 沙箱策略不构成 Agent 行为约束，仅定义进程边界
- pre_exec hook 是工具命令沙箱的强制执行点

## 8. 安全审计

沙箱策略变更应在 PR 中说明理由，建议：
- 沙箱策略收紧：直接合并
- 沙箱策略放宽（如 mode 从 read-only 改为 workspace-write）：必须 code review
- 新增 network_allowlist 条目：必须说明业务理由

## 9. 与日志体系的集成

沙箱 wrapper 脚本（run.sh）输出 NDJSON 事件到 stderr：

```json
{"ts":"...","level":"info","event":"sandbox.start","target":"filesystem","reason":"mode=read-only, fs=./, network=none"}
{"ts":"...","level":"warn","event":"sandbox.violation","target":"filesystem","reason":"尝试访问 /etc/passwd"}
```

格式遵循 `.trae/logging.md` 规范，被采集器统一捕获。
