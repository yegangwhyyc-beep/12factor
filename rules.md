# Coding Rules

> 本文件是 AGENTS.md 的细粒度补充，定义编码层面的强制规则。
> Trae 在生成 / 修改代码时必须遵守，冲突时以 AGENTS.md 为准。

## 1. 命名规范

| 类型 | 规则 | 示例 |
|------|------|------|
| 模块 / 文件 | snake_case | `user_service.py` |
| 类 | PascalCase | `UserService` |
| 函数 / 变量 | snake_case | `get_user_by_id` |
| 常量 | UPPER_SNAKE | `MAX_RETRY_COUNT` |
| 私有成员 | 前缀 `_` | `_validate_input` |
| Pydantic 字段 | snake_case，序列化时转 camelCase | `user_id` → `userId` |
| 异常类 | 后缀 `Error` | `UserNotFoundError` |
| Enum 成员 | UPPER_SNAKE | `UserRole.ADMIN` |

**禁止**：单字母变量（除循环索引 `i/j/k`）、匈牙利记号、缩写到无法理解。

## 2. 函数规范

```python
# ✅ 正确：有类型注解 + docstring + 单一职责
async def get_user_by_id(
    user_id: int,
    *,
    include_deleted: bool = False,
) -> User:
    """根据 ID 查询用户。

    Args:
        user_id: 用户唯一标识。
        include_deleted: 是否包含已软删除的用户，默认 False。

    Returns:
        User 对象。

    Raises:
        UserNotFoundError: 用户不存在时抛出。
    """
    ...
```

**规则**：
- 函数长度 ≤ 50 行，超过必须拆分
- 参数数量 ≤ 5 个，超过必须封装为 schema 对象
- 优先使用关键字参数（`*` 强制）
- 公开函数必须有 docstring（Google 风格）
- 私有工具函数可省略 docstring，但需有注释说明意图

## 3. 错误处理

### 3.1 异常分层

```
BaseException
└── app.core.exceptions.AppError           # 所有业务异常基类
    ├── UserNotFoundError                   # 404
    ├── ValidationError                     # 422
    ├── AuthenticationError                 # 401
    ├── AuthorizationError                  # 403
    └── ConflictError                       # 409
```

### 3.2 规则

- **禁止**裸 `except:`，必须 `except SomeError as e:`
- **禁止**吞异常（`except: pass`），至少要记日志
- 业务错误抛 `AppError` 子类，由全局异常处理器转 HTTP 响应
- 外部调用（HTTP / DB）必须 catch 并转为业务异常
- 不允许在 `finally` 中 return

```python
# ✅ 正确
try:
    resp = await httpx.AsyncClient().get(url)
    resp.raise_for_status()
except httpx.HTTPError as e:
    logger.warning("external_api_failed", url=url, error=str(e))
    raise ExternalServiceError(f"调用外部服务失败: {e}") from e

# ❌ 错误
try:
    resp = await httpx.AsyncClient().get(url)
except:
    pass
```

## 4. 日志规范

- 使用 `structlog`，**禁止** `print()` 和 `logging` 原生模块
- 日志输出到 stdout（12-Factor），由容器收集
- 必须使用结构化字段，便于查询

```python
# ✅ 正确：结构化日志
logger.info("user_created", user_id=user.id, source="api")

# ❌ 错误：f-string 拼接
logger.info(f"User {user.id} created from api")
```

**日志级别**：
- `debug`：开发调试信息
- `info`：业务关键节点（用户创建、订单完成）
- `warning`：可恢复异常、降级行为
- `error`：不可恢复错误，需告警
- `critical`：系统级故障

**禁止**：日志中打印密码、Token、身份证号、手机号明文。

## 5. 异步编程规范

- API 路由、service、repository 全部 `async def`
- **禁止**在 async 函数中调用同步阻塞 IO（`time.sleep`、`requests.get`）
  - 用 `asyncio.sleep`、`httpx.AsyncClient`
- 数据库操作用 `SQLAlchemy AsyncSession`
- CPU 密集型任务用 `asyncio.to_thread()` 或 `run_in_executor()`
- **禁止**未 await 的协程（pyright 会警告）

```python
# ✅ 正确
async def fetch_user(user_id: int) -> User:
    async with db.AsyncSession() as session:
        return await session.get(User, user_id)

# ❌ 错误：阻塞调用
async def fetch_user(user_id: int) -> User:
    time.sleep(1)  # 阻塞 event loop
    return requests.get(f"/users/{user_id}").json()
```

## 6. 数据库规范

- 所有表必须有 `id`（BigInt 主键）、`created_at`、`updated_at`
- 软删除用 `deleted_at` 字段，不物理删除业务数据
- 查询必须分页（`limit` + `offset`），单次查询 ≤ 100 条
- **禁止** N+1 查询，用 `selectinload` / `joinedload` 预加载关联
- 写操作必须在事务中
- Migration 必须可回滚（`upgrade` + `downgrade`）

## 7. API 设计规范

- RESTful 风格，资源用复数：`/users`、`/orders`
- 版本前缀：`/api/v1/users`
- 响应统一封装：

```python
class ApiResponse(BaseModel, Generic[T]):
    code: int = 0          # 0 表示成功，非 0 表示业务错误码
    message: str = "ok"
    data: T | None = None
```

- HTTP 状态码语义化：200/201/204/400/401/403/404/409/422/500
- 分页响应统一结构：

```python
class PageResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
```

- 所有端点必须有鉴权 decorator，公开端点显式标注 `@public`

## 8. 测试规范

- 文件命名：`test_<module>.py`
- 函数命名：`test_<行为>_<条件>_<预期>`
- 每个公开函数至少 3 个用例：正常、边界、异常
- **禁止**依赖测试执行顺序
- **禁止**访问真实数据库 / 网络，用 `pytest-asyncio` + `unittest.mock`
- Fixture 复用，避免重复 setup

```python
# ✅ 命名清晰
async def test_get_user_raises_not_found_when_id_not_exist():
    ...

# ❌ 命名模糊
async def test_user():
    ...
```

## 9. 配置规范

- 所有配置集中在 `app/core/config.py` 的 `Settings` 类
- 使用 `pydantic-settings` 从环境变量读取
- **禁止**在业务代码中 `os.getenv()`
- 敏感配置（密钥、连接串）只读环境变量，不写默认值

```python
# ✅ 正确
class Settings(BaseSettings):
    database_url: PostgresDsn
    secret_key: SecretStr  # 无默认值，强制注入

# ❌ 错误
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://localhost/dev")
```

## 10. Git 提交规范

提交信息格式（Conventional Commits）：

```
<type>(<scope>): <subject>

<body>

<footer>
```

- `type`：feat / fix / chore / refactor / docs / test / perf / ci
- `scope`：模块名（可选），如 `auth`、`user`、`order`
- `subject`：祈使句，≤ 50 字符，不加句号
- `body`：说明 why，不在 what
- `footer`：BREAKING CHANGE / Closes #123

```
feat(auth): 支持 OAuth2 登录

新增 Google / GitHub 第三方登录，复用现有 JWT 颁发逻辑。

Closes #142
```

## 11. 禁止清单（汇总）

| # | 禁止项 | 替代方案 |
|---|--------|---------|
| 1 | `print()` | `logger.info()` |
| 2 | 裸 `except:` | `except SpecificError:` |
| 3 | `os.getenv()` 在业务代码 | `settings.xxx` |
| 4 | 同步阻塞 IO 在 async 函数 | 异步库 / `to_thread` |
| 5 | 硬编码密钥 / Token | 环境变量 / SecretStr |
| 6 | `requests` 库 | `httpx.AsyncClient` |
| 7 | 物理删除业务数据 | 软删除 `deleted_at` |
| 8 | `time.sleep()` 在 async | `asyncio.sleep()` |
| 9 | 函数 > 50 行 | 拆分 |
| 10 | 单测依赖执行顺序 | Fixture 隔离 |
| 11 | `eval()` / `exec()` | 显式逻辑 |
| 12 | `shell=True` subprocess | 参数列表 |
