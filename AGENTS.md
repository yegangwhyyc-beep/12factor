# Project: 12factordemo

> 本文件为  Trae（及 Cursor /  等 Agent 工具）的项目级宪法。

## 技术栈

- 语言：Python 3.11+
- Web 框架：FastAPI
- 数据库：PostgreSQL 15 + SQLAlchemy 2.0（异步）
- 包管理：uv（替代 pip / poetry）
- 测试：pytest + pytest-asyncio，覆盖率 ≥ 80%
- 代码风格：ruff（lint + format），2 空格缩进禁用，统一 4 空格

## 目录结构

```
12factordemo/
├── AGENTS.md                # 本文件
├── hooks.json               # Hooks 配置（独立文件，事件→脚本映射）
├── .trae/                  # Trae 项目级配置
│   ├── config.toml          # 模型/审批/沙箱/MCP/Loop 参数（不含 hooks）
│   ├── loop.md              # Agent Loop 行为策略（任务分解/终止/恢复）
│   ├── subagents.md         # Sub Agent 角色定义（investigator/builder/reviewer/tester）
│   ├── hooks.md             # Hooks 定义文档（强制执行层 / Human-in-the-loop）
│   ├── logging.md           # 日志与可观测性（NDJSON / 双通道 / 采集层）
│   ├── sandbox.md           # 沙箱设计（MCP 进程隔离 / 12-Factor 对齐）
│   ├── hooks/               # Hook 脚本（7 个 + 1 共享库）
│   │   ├── _lib.sh          # 共享 emit_event 函数（NDJSON → stderr）
│   │   ├── pre_write.sh     # 文件写入前校验
│   │   ├── pre_exec.sh      # 命令执行前校验
│   │   ├── pre_commit.sh    # git commit 前校验
│   │   ├── post_write.sh    # 文件写入后审计
│   │   ├── pre_prompt.sh    # 用户 prompt 提交前扫描
│   │   ├── session_start.sh # 会话启动注入上下文 + memory
│   │   └── session_stop.sh  # 会话结束输出 session.stop 事件
│   ├── memory.md            # Memory 规范（读取/写入/格式/隐私边界）
│   ├── memory/              # 记忆文件（仅源代码资产入库）
│   │   ├── project.md       # 项目长期记忆（架构决策/技术债/约定）
│   │   └── lessons.md       # 经验库（踩坑/有效模式/Prompt 技巧）
│   ├── rules.md             # 细粒度编码规则（AGENTS.md 的补充）
│   ├── sandbox/             # MCP 沙箱 wrapper + profile 模板
│   │   ├── run.sh           # 沙箱启动脚本（macOS sandbox-exec）
│   │   └── profiles/        # sandbox-exec profile（readonly/workspace/network）
│   ├── skills/              # 可复用 Skill
│   └── prompts/             # 自定义 slash 命令 + Sub Agent Prompt 模板
├── docs/                    # 文档（架构/开发指南）
├── .env.example             # 环境变量样例
├── pyproject.toml           # 依赖声明 + 工具链配置
├── Dockerfile               # 多阶段构建模板
└── docker-compose.yml       # 基础设施（PostgreSQL + 应用）
```

> **业务代码填充区**（业务就位后创建）：
> - `app/` — 应用源码（api/core/models/schemas/services/main.py）
> - `tests/` — 单元测试
> - `migrations/` — Alembic 数据库迁移

## 任务规范

- 所有 PR 提交前必须运行：`uv run ruff check . && uv run pytest`
- 新功能必须附单元测试，覆盖率不低于 80%
- 提交信息格式：`feat/fix/chore/refactor/docs: 简短描述`
- 涉及数据库 schema 改动必须生成 Alembic migration

## 代码规范

- 所有公开函数必须有类型注解 + docstring
- API 路由统一使用 `async def`
- 配置统一通过 `app.core.config.settings` 读取，禁止直接读环境变量
- 遵守 12-Factor：配置走环境变量，日志输出到 stdout，无状态进程

## 禁止操作

- 不修改 `app/core/security.py`（由安全团队维护，需先开 Issue 讨论）
- 不删除 `migrations/` 目录下任何文件
- 不在生产代码中使用 `print()`，统一用 `structlog` 日志
- 不引入新的重型依赖（需在 PR 中说明理由）
- 不硬编码任何密钥、Token、数据库连接串

## 优先参考文件

- 架构说明：`docs/architecture.md`（如存在）
- API 设计：`docs/api.md`（如存在）
- Agent Loop 策略：`.trae/loop.md`（任务分解、终止条件、错误恢复）
- Sub Agent 定义：`.trae/subagents.md`（角色、权限边界、返回格式）
- Hooks 定义：`.trae/hooks.md`（强制执行层、Human-in-the-loop 触发点）
- 日志规范：`.trae/logging.md`（NDJSON 双通道、采集层接入、不写本地文件）
- Memory 规范：`.trae/memory.md`（记忆读取/写入/格式/隐私边界）
- 细粒度编码规则：`.trae/rules.md`（本文件的补充，冲突时以本文件为准）
- 12-Factor 规范：https://12factor.net/

## 冲突优先级

当多份规则冲突时，按以下优先级裁决：

1. `AGENTS.md`（本文件，最高）
2. `.trae/hooks.md` + `.trae/hooks/*.sh`（强制执行，优先于声明）
3. `.trae/loop.md`（Loop 行为策略）
4. `.trae/subagents.md`（Sub Agent 定义）
5. `.trae/rules.md`（编码细则）
6. `.trae/config.toml`（运行参数）
7. `.trae/memory/*.md`（参考层，最低）

> **Hooks 优先原则**：hook 阻断的操作，即使声明性规则允许也以 hook 为准。
> **Memory 参考原则**：memory 与规则冲突时以规则为准，memory 仅提供历史经验。

## 工作流建议

- 修 Bug：先写复现测试 → 修复 → 跑全量测试
- 加功能：先写 schema → service → API → 测试
- 重构：保持测试绿色，小步提交
