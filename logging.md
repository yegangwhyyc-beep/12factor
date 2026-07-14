# 日志与可观测性

> 12-Factor XI 升级：Agent 进程无状态，日志全部流式输出，持久化由外部采集层负责。
> 本文件定义日志通道、NDJSON schema、采集层接入方式。

## 1. 双通道设计

| 通道 | 用途 | 格式 | 采集器 |
|------|------|------|--------|
| `stdout` | 业务日志（structlog） | NDJSON | Filebeat / Promtail |
| `stderr` | 审计事件（hooks） | NDJSON | Filebeat / Promtail（独立路由） |

**为什么分开**：stdout 给应用日志采集器，stderr 给安全/审计采集器，路由到不同存储池，便于独立保留期与权限管控。

## 2. NDJSON Schema

### 2.1 强制字段

```json
{
  "ts": "2026-07-04T10:23:45.123Z",
  "level": "info",
  "event": "hook.pre_write.blocked",
  "trace_id": "abc-123",
  "session_id": "sess-456",
  "env": "development"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `ts` | string | ISO 8601 UTC，毫秒精度 |
| `level` | string | debug / info / warn / error |
| `event` | string | 点分命名：`<source>.<action>.<result>` |
| `trace_id` | string | 请求/会话追踪 ID |
| `session_id` | string | Trae 会话 ID |
| `env` | string | development / staging / production |

### 2.2 可选字段

| 字段 | 适用事件 | 说明 |
|------|---------|------|
| `target` | hook.* | 操作对象（文件路径/命令） |
| `result` | hook.* | PASSED / BLOCKED / NEEDS_APPROVAL / WARN |
| `reason` | hook.* | 阻断/警告原因 |
| `agent` | subagent.* | main / investigator / builder / reviewer / tester |
| `changed_files` | session.stop | 改动文件列表（JSON 数组） |
| `duration_ms` | 任意 | 耗时 |

## 3. 事件命名约定

`<source>.<action>.<result>`

| 事件 | 触发 |
|------|------|
| `hook.pre_write.passed` | 文件写入前校验通过 |
| `hook.pre_write.blocked` | 命中禁止文件 |
| `hook.pre_write.needs_approval` | 命中需审批文件 |
| `hook.pre_exec.blocked` | 危险命令阻断 |
| `hook.pre_exec.needs_approval` | 危险命令需审批 |
| `hook.pre_commit.passed` | commit 校验通过 |
| `hook.pre_commit.blocked_*` | commit 校验失败（format/ruff/pytest） |
| `hook.post_write.written` | 文件写入后审计 |
| `hook.pre_prompt.warn_secret` | prompt 含疑似密钥 |
| `session.start` | 会话启动 |
| `session.stop` | 会话结束（含改动文件） |
| `app.error` | 应用异常 |

## 4. trace_id 传播

Trae 在会话启动时生成 trace_id，通过 `TRAE_TRACE_ID` 环境变量注入到所有 hook 脚本。

```bash
# hook 脚本读取
TRACE_ID="${TRAE_TRACE_ID:-unknown}"
```

如果未设置，使用 "unknown"，由采集器标记为缺失（便于发现配置问题）。

## 5. 采集层接入

### 5.1 Docker（开发环境，json-file + Filebeat）

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

容器日志被 Filebeat/Promtail 采集后路由到 Loki/ES。

### 5.2 Kubernetes（生产环境，Loki + Promtail）

```yaml
# promtail-config.yaml
scrape_configs:
  - job_name: 12factordemo
    static_configs:
      - targets: [localhost]
        labels:
          job: 12factordemo
          __path__: /var/log/containers/*12factordemo*.log
```

### 5.3 本地开发（不部署采集器）

实时查看 stderr 事件流：
```bash
trae 2> >(jq .)
```

或分开收集两个通道：
```bash
trae > app.log 2> audit.log
```

## 6. 存储层

| 系统 | 用途 | 保留期 | 适用 |
|------|------|--------|------|
| Loki | 实时查询 + 告警 | 30 天 | 开发 / 小规模生产 |
| Elasticsearch | 全文搜索 + 审计 | 90 天 | 合规要求高 |
| ClickHouse | 审计分析 | 180 天 | 大规模审计 |
| S3 / OSS | 冷归档 | 1 年+ | 合规归档 |

## 7. 与本地文件的关系

| 文件 | 性质 | 处置 |
|------|------|------|
| `.trae/memory/project.md` | 源代码资产（团队决策） | ✅ 入库 git |
| `.trae/memory/lessons.md` | 源代码资产（经验库） | ✅ 入库 git |
| `.trae/memory/sessions/*.md` | ❌ 不再生成 | 走 stderr 事件流 |
| `.trae/hooks/audit.log` | ❌ 不再生成 | 走 stderr 事件流 |

**原则**：业务运行时数据全部走日志通道，由外部系统持久化。本地只保留"源代码资产"。

## 8. 与规则体系的关系

本文件属于**可观测性层**：

- 不构成 Agent 行为约束，仅定义输出格式
- 优先级独立于 AGENTS.md 规则链
- hooks 脚本必须遵守本文件的 NDJSON schema

## 9. 共享 emit 库

所有 hook 脚本 source `.trae/hooks/_lib.sh`，调用 `emit_event` 函数输出 NDJSON：

```bash
source "$(dirname "$0")/_lib.sh"

emit_event "warn" "hook.pre_write.blocked" "BLOCKED" \
    "app/core/security.py" "命中禁止清单"
```

`_lib.sh` 负责 JSON 转义、时间戳、trace_id 注入。
