# Memory 规范

> 本文件定义 Trae Agent 的记忆体系：何时读、何时写、格式约定、隐私边界。
> Memory 是**参考层**，提供历史经验作为 Agent 决策输入，不强制执行。
> 区别于规则层（AGENTS.md / rules.md 是约束）和执行层（hooks 是强制）。

## 1. 设计目标

| 目标 | 实现方式 |
|------|---------|
| 跨会话保持上下文 | project.md 持久化项目决策 |
| 累积经验避免重复踩坑 | lessons.md 记录有效模式和教训 |
| 会话级任务跟踪 | sessions/<id>.md 记录单次任务进度 |
| 自动注入与采集 | session_start 读取、session_stop 写入 |

## 2. Memory 类型

| 类型 | 文件 | 生命周期 | 入库 | 用途 |
|------|------|---------|------|------|
| 项目记忆 | `memory/project.md` | 永久 | ✅ | 架构决策、团队约定、技术债 |
| 经验记忆 | `memory/lessons.md` | 永久 | ✅ | 踩过的坑、有效 prompt 模式 |
| 会话记忆 | `memory/sessions/<id>.md` | 单会话 | ❌ | 当前任务进度、TODO、关键变量 |

## 3. 读取时机

### 3.1 会话启动（SessionStart hook）
`session_start.sh` 自动执行：
1. 读取 `memory/project.md` 全文 → 注入为 additionalContext
2. 读取 `memory/lessons.md` 全文 → 注入为 additionalContext
3. 若存在 `memory/sessions/<prev_session_id>.md`，读取上一会话摘要

注入格式：
```
[项目记忆]
<project.md 内容>

[经验记忆]
<lessons.md 内容>
```

### 3.2 任务执行中
Agent 可主动 `cat .trae/memory/project.md` 查阅架构决策。
不自动注入 sessions/ 下其他会话的记忆（避免污染上下文）。

## 4. 写入时机

### 4.1 会话结束（Stop hook）
`session_stop.sh` 自动执行：
1. 收集 git status 改动文件列表
2. 输出 `session.stop` NDJSON 事件到 stderr（含 changed_files 字段）
3. 由外部采集器（Filebeat/Promtail）路由到 Loki/ES 持久化

**不再写本地 sessions/<id>.md**，遵守 12-Factor XI。

### 4.2 主动写入（仅 project / lessons）
Agent 在任务执行中发现重要决策时，可主动追加到 `memory/project.md`：
```bash
echo "- [2026-07-04] 决策：session 用 JWT 而非 server-side session（12-Factor VI）" \
    >> .trae/memory/project.md
```

会话级短期状态（任务进度、TODO）不要写本地文件，改为输出事件流：
```bash
emit_event "info" "task.progress" "IN_PROGRESS" "task-name" "完成 50%"
```

## 5. 文件格式约定

### 5.1 project.md
```markdown
# 项目记忆

## 架构决策
- [YYYY-MM-DD] 决策内容（一句话）— 原因

## 技术债
- [YYYY-MM-DD] 债务描述 — 计划还款时间

## 团队约定
- 约定内容
```

### 5.2 lessons.md
```markdown
# 经验库

## 有效模式
- [YYYY-MM-DD] 场景 → 做法

## 踩过的坑
- [YYYY-MM-DD] 现象 → 原因 → 解决

## Prompt 技巧
- 场景 → 有效 prompt 模式
```

### 5.3 sessions/<id>.md
```markdown
# Session: <session_id>
- 时间：YYYY-MM-DD HH:MM
- 任务：一句话描述
- 改动文件：file1, file2
- 关键决策：...
- 未完成：...
- 下一会话建议：...
```

## 6. 隐私边界

**禁止写入 memory 的内容**：
- 密钥、Token、密码（即使是测试用）
- 用户个人数据
- 完整代码片段（只记决策与经验，代码在 git 里）

**允许写入**：
- 架构决策及理由
- 踩坑现象与解决方案关键词
- 任务进度与 TODO
- 有效的 prompt 模式

## 7. 冲突优先级

Memory 是**参考层**，优先级最低：

1. AGENTS.md（约束）
2. hooks.md + hooks/*.sh（强制）
3. loop.md（策略）
4. subagents.md（角色）
5. rules.md（细则）
6. config.toml（参数）
7. **memory/*.md（参考，最低）**

> Memory 与规则冲突时，以规则为准。Memory 仅提供历史经验，不构成约束。

## 8. 维护要求

- `project.md` 和 `lessons.md` 入库，团队共享
- `sessions/` 目录 gitignore，不共享
- 每月 review 一次 `lessons.md`，把过时条目归档到 `lessons.archive.md`
- `project.md` 单文件不超过 200 行，超出则按主题拆分

## 9. 与 hooks 的集成

| Hook | 行为 |
|------|------|
| `session_start.sh` | 读 project.md + lessons.md 注入 stdout；输出 session.start 事件到 stderr |
| `session_stop.sh` | 输出 session.stop 事件（含 changed_files）到 stderr，不写本地文件 |

集成代码见 `.trae/hooks/session_start.sh` 和 `session_stop.sh`。
日志格式见 `.trae/logging.md`。

## 10. 与 MCP memory server 的关系

| 来源 | 用途 |
|------|------|
| `.trae/memory/*.md`（本规范） | 文件式记忆，可读、可 git 跟踪、团队共享 |
| MCP `memory` server（config.toml） | 知识图谱式记忆，跨工具共享，默认禁用 |

两者互补：文件式适合项目级长期记忆，MCP 适合实体关系图谱。
