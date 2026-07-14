# Hooks 强制执行层

> 把 AGENTS.md / rules.md 中的声明性规则转成可执行的强制校验。
> Hooks 是 Agent 行为的"硬边界"：阻断的操作即使声明性规则允许也不能执行。
> 配置文件：根目录 `hooks.json`（独立文件，不再用 config.toml）
> 脚本目录：`.trae/hooks/`
> 日志规范：见 `.trae/logging.md`（NDJSON → stderr，不写本地文件）

## 1. 设计目标

| 目标 | 实现 |
|------|------|
| 强制执行禁止规则 | pre_write / pre_exec 阻断 |
| 关键操作需人工审批 | exit 2 触发 Human-in-the-loop |
| 自动校验代码质量 | pre_commit 跑 ruff + pytest |
| 全程审计留痕 | NDJSON 事件流到 stderr，由外部采集器持久化 |
| 会话级上下文注入 | session_start 输出 project.md + lessons.md 到 stdout |

## 2. Hook 列表

| Hook | 事件 | 退出码 | 作用 |
|------|------|--------|------|
| `pre_write.sh` | PreToolUse | 0/1/2 | 文件写入前：禁止文件 / 需审批文件 |
| `pre_exec.sh` | PreToolUse | 0/1/2 | 命令执行前：危险命令阻断 |
| `pre_commit.sh` | PreToolUse | 0/1 | git commit 前：格式 / ruff / pytest |
| `post_write.sh` | PostToolUse | 0 | 文件写入后审计（不阻断） |
| `pre_prompt.sh` | UserPromptSubmit | 0/1/3 | 用户 prompt 提交前：敏感信息扫描 |
| `session_start.sh` | SessionStart | 0 | 注入项目上下文 + memory |
| `session_stop.sh` | Stop | 0 | 输出 session.stop 事件 |

## 3. 退出码约定

| 码 | 含义 | Trae 行为 |
|----|------|-----------|
| 0 | 通过 | 继续执行 |
| 1 | 阻断 | 终止当前操作 |
| 2 | 需人工审批 | 暂停，等待用户确认 |
| 3 | 警告 | 继续但记录警告 |

## 4. 日志输出

**所有 hook 事件输出到 stderr，格式为 NDJSON**（见 `.trae/logging.md`）。

示例事件：
```json
{"ts":"2026-07-04T10:23:45.123Z","level":"warn","event":"hook.pre_write.blocked","result":"BLOCKED","target":"app/core/security.py","reason":"命中禁止清单","session_id":"sess-123","trace_id":"abc","env":"development"}
```

**stdout 仅用于 session_start 注入项目上下文 + memory**，不输出审计事件。

## 5. 共享库

所有 hook 脚本 source `.trae/hooks/_lib.sh`，调用 `emit_event`：

```bash
source "$(dirname "$0")/_lib.sh"

emit_event "warn" "hook.pre_write.blocked" "BLOCKED" \
    "app/core/security.py" "命中禁止清单"
```

`_lib.sh` 负责：
- JSON 字符串转义
- 时间戳生成（毫秒精度）
- trace_id / session_id / env 注入
- 输出到 stderr

## 6. 配置注册

Hooks 配置在**根目录 `hooks.json`**（独立文件）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "write_file|edit_file|apply_patch",
        "hooks": [{ "type": "command", "command": ".trae/hooks/pre_write.sh" }]
      }
    ],
    "PostToolUse": [...],
    "UserPromptSubmit": [...],
    "SessionStart": [...],
    "Stop": [...]
  }
}
```

**Trae 标准位置说明**：
- Trae 官方项目级 hooks.json 位置是 `.trae/hooks.json`
- 本项目放在根目录 `hooks.json`（更直观，团队易于发现）
- 如果 Trae 不读取根目录，软链即可：
  ```bash
  ln -s ../../hooks.json .trae/hooks.json
  ```

**config.toml 仅保留 feature flag**：
```toml
[features]
trae_hooks = true
```

## 7. 与规则体系的关系

```
声明（约束）   AGENTS.md / rules.md / loop.md / subagents.md / hooks.md
   ↓
配置（映射）   hooks.json（事件 → 脚本）
   ↓
执行（强制）   hooks/*.sh（exit 0/1/2/3）
   ↓
输出（流式）   stderr NDJSON → 外部采集器（Loki/ES）
```

**Hooks 优先原则**：hook 阻断的操作，即使声明性规则允许也以 hook 为准。

## 8. 不再保留本地 audit.log

旧版本 hook 写本地 `.trae/hooks/audit.log`，违反 12-Factor XI。

**新版本**：
- 全部事件输出到 stderr（NDJSON）
- 由外部采集器（Filebeat / Promtail）持久化到 Loki/ES
- 本地不持久化任何运行时数据

如需本地查看事件流（开发期）：
```bash
trae 2> >(jq .)
```
