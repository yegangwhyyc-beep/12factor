# 架构说明

> 本文档描述 12factordemo 框架骨架的系统架构与 12-Factor 实践方式。
> 业务代码就位后，应在本文档补充模块职责、依赖方向、请求处理流程等具体内容。
> Agent 在修改架构相关配置前必须先读本文档。

## 1. 设计原则

本项目严格遵循 [12-Factor App](https://12factor.net/) 规范：

| Factor | 实践 |
|--------|------|
| I. 代码库 | 单一 Git 仓库，多环境靠配置区分 |
| II. 依赖 | `pyproject.toml` 显式声明，`uv` 锁定 |
| III. 配置 | 环境变量注入，禁止硬编码 |
| IV. 后端服务 | PostgreSQL 作为可替换资源（docker-compose） |
| V. 构建/发布/运行 | Docker 镜像多阶段构建（Dockerfile） |
| VI. 进程 | 无状态，会话走 DB |
| VII. 端口绑定 | uvicorn 自绑定，不依赖外部 server |
| VIII. 并发 | 异步 IO（asyncio + asyncpg） |
| IX. 易处理 | 快速启动、优雅关闭 |
| X. 开发生产等价 | dev/staging/prod 同一镜像不同配置 |
| XI. 日志 | 输出 stdout，结构化（structlog / NDJSON） |
| XII. 管理进程 | Alembic migration / 一次性脚本 |

## 2. 框架骨架分层

```
┌─────────────────────────────────────────────────────┐
│              Trae Agent 框架（声明层）              │
│  AGENTS.md  → 项目级行为指令                        │
│  .trae/     → 配置/规则/hooks/sandbox/memory        │
└──────────────────────┬──────────────────────────────┘
                       │ 约束
                       ▼
┌─────────────────────────────────────────────────────┐
│              应用骨架（业务填充层）                  │
│  app/         ← 业务代码就位后填充                  │
│  tests/       ← 单元测试就位后填充                  │
│  migrations/  ← 数据库迁移就位后填充                │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              基础设施（部署层）                      │
│  Dockerfile          → 多阶段构建                   │
│  docker-compose.yml  → PostgreSQL + 应用            │
│  pyproject.toml      → 依赖声明 + 工具链配置        │
│  .env.example        → 环境变量样例                 │
└─────────────────────────────────────────────────────┘
```

## 3. 推荐业务模块职责（业务代码就位后参考）

| 模块 | 职责 | 禁止 |
|------|------|------|
| `app/api/` | HTTP 路由、参数校验、鉴权 | 写业务逻辑 |
| `app/services/` | 业务规则、事务编排 | 直接接 HTTP 请求/响应对象 |
| `app/models/` | ORM 定义、数据库映射 | 写业务逻辑 |
| `app/schemas/` | Pydantic 数据校验 / 序列化 | 写 IO 逻辑 |
| `app/core/` | 配置、日志、安全、异常 | 写业务逻辑 |

## 4. 关键设计决策

### 4.1 异步全链路
- FastAPI + async def 路由
- SQLAlchemy AsyncSession + asyncpg
- httpx.AsyncClient 调用外部服务
- **禁止** requests / time.sleep 等阻塞调用

### 4.2 配置集中管理
- 所有配置通过 `app.core.config.settings` 暴露
- `Settings` 继承 `pydantic-settings.BaseSettings`
- 业务代码禁止 `os.getenv()`

### 4.3 日志
- structlog 输出 JSON 到 stdout
- 每请求自动注入 request_id
- 敏感字段（密码、Token）自动脱敏
- Agent 框架事件走 NDJSON → stderr（见 `.trae/logging.md`）

## 5. 部署架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Nginx     │────▶│   App ×N    │────▶│ PostgreSQL  │
│  (LB/TLS)   │     │ (uvicorn)   │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Redis     │  ← 可选：缓存/限流
                    └─────────────┘
```

- App 无状态，可水平扩展
- 日志由容器收集（stdout → ELK / Loki）
- 数据库连接池：每进程 5-10 连接

## 6. 扩展点

| 场景 | 扩展方式 |
|------|---------|
| 新增资源 | `models/` + `schemas/` + `services/` + `api/v1/` |
| 新增中间件 | `app/main.py` 的 `add_middleware()` |
| 新增外部服务 | `app/services/external/` + httpx client |
| 新增后台任务 | 独立 worker 进程（Celery / ARQ） |
