# 开发指南

> 本文档帮助开发者快速上手 12factordemo 框架骨架。
> 业务代码就位后，应在本文档补充具体的启动、测试、迁移流程。

## 1. 环境准备

### 1.1 系统要求
- Python 3.11+
- PostgreSQL 15+
- uv（包管理）

### 1.2 安装 uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 1.3 配置环境变量

```bash
cp .env.example .env
# 编辑 .env，填入数据库连接串和 SECRET_KEY
```

## 2. 启动开发

```bash
# 安装依赖（含开发依赖）
uv sync --extra dev

# 启动数据库（如使用 Docker）
docker run -d --name pg -p 5432:5432 \
    -e POSTGRES_PASSWORD=postgres \
    -e POSTGRES_DB=12factordemo \
    postgres:15

# 业务代码就位后启动应用：
# uv run uvicorn app.main:app --reload --port 8000
```

## 3. 日常开发流程

### 3.1 提交前自检

```bash
uv run ruff check .
uv run ruff format .
uv run pytest
```

### 3.2 加新功能（业务代码就位后按顺序）

1. `app/schemas/<resource>.py` — 定义 Pydantic schema
2. `app/models/<resource>.py` — 定义 ORM 模型
3. `app/services/<resource>.py` — 实现业务逻辑
4. `app/api/v1/<resource>.py` — 暴露 HTTP 路由
5. `tests/test_<resource>.py` — 写测试

### 3.3 数据库迁移（业务代码就位后）

```bash
# 生成 migration
uv run alembic revision --autogenerate -m "add user table"

# 执行
uv run alembic upgrade head

# 回滚
uv run alembic downgrade -1
```

## 4. 测试

```bash
# 全量（业务测试就位后）
uv run pytest

# 带覆盖率
uv run pytest --cov=app --cov-report=html
```

## 5. Docker 部署

```bash
docker build -t 12factordemo .
docker run -p 8000:8000 --env-file .env 12factordemo
```

## 6. 框架能力

本项目除业务代码外，还包含 Trae Agent 框架骨架：

| 能力 | 文件 | 说明 |
|------|------|------|
| 项目指令 | `AGENTS.md` | 项目级行为指令 |
| Loop 策略 | `.trae/loop.md` | 任务分解/终止/恢复 |
| Sub Agent | `.trae/subagents.md` | investigator/builder/reviewer/tester |
| Hooks | `hooks.json` + `.trae/hooks/*.sh` | 强制执行层 / Human-in-the-loop |
| 编码细则 | `.trae/rules.md` | 命名/函数/错误处理 |
| 运行参数 | `.trae/config.toml` | 模型/审批/沙箱/MCP |
| 日志规范 | `.trae/logging.md` | NDJSON 双通道 |
| 沙箱设计 | `.trae/sandbox.md` | MCP 进程隔离 |
| Memory | `.trae/memory.md` | project/lessons/session |
