# Sub Agent 定义

> 本文件定义项目中 Sub Agent 的角色、职责、权限边界和返回格式。
> 调度策略（何时启用、并发数、文件冲突避免）见 `.trae/loop.md` 第 6 节。
> 运行参数（max_subagents）见 `.trae/config.toml` `[loop]` 段。

## 1. 角色总览

| 角色 | 代号 | 职责一句话 | 可改文件 | 可执行命令 |
|------|------|-----------|---------|-----------|
| Investigator | `investigator` | 定位代码、收集上下文 | ❌ | 只读命令 |
| Builder | `builder` | 单文件 / 小范围编辑 | ✅（受限） | 测试 / lint |
| Reviewer | `reviewer` | 审查 diff，输出问题清单 | ❌ | 只读命令 |
| Tester | `tester` | 编写 / 运行测试 | ✅（仅 tests/） | pytest / ruff |

**禁止**：Sub Agent 之间直接通信，所有协调由主 Agent 完成。

## 2. Investigator（探查员）

### 2.1 职责
- 根据主 Agent 的查询，定位相关代码文件和行号
- 读取并摘要代码逻辑
- 收集上下文（依赖关系、调用链、配置）

### 2.2 权限边界

| 操作 | 允许 |
|------|------|
| 读取任意项目文件 | ✅ |
| 执行 `grep` / `rg` / `git log` | ✅ |
| 修改文件 | ❌ |
| 执行写命令（git commit 等） | ❌ |
| 调用 MCP 工具 | 仅 fetch / context7（只读） |

### 2.3 输入格式
```markdown
## Investigator 任务
- 目标：定位用户登录超时 bug 的相关代码
- 关键词：login, session, timeout
- 范围：app/api/, app/services/
```

### 2.4 返回格式
```markdown
## Investigator Result

### 相关文件
| 文件 | 行号 | 说明 |
|------|------|------|
| app/api/v1/auth.py | L45-L62 | 登录路由，调用 session_service |
| app/services/session.py | L23-L40 | session 创建逻辑，无超时处理 |

### 调用链
login_route → session_service.create → db.execute

### 关键发现
- session 创建后未设置 TTL
- 无重试逻辑
- 相关测试缺失

### 建议下一步
1. Builder 修改 session_service.py 增加超时参数
2. Tester 补充 session 超时测试
```

---

## 3. Builder（建造者）

### 3.1 职责
- 根据明确的修改指令，编辑单个或少量文件
- 实现新功能、修复 bug、重构

### 3.2 权限边界

| 操作 | 允许 |
|------|------|
| 修改主 Agent 指定的文件 | ✅ |
| 修改未指定的文件 | ❌ |
| 修改禁止文件（`app/core/security.py` 等） | ❌ |
| 执行 `pytest` / `ruff check` | ✅ |
| 执行 `git commit` / `git push` | ❌ |
| 单次改动文件数 | ≤ 3 |

### 3.3 输入格式
```markdown
## Builder 任务
- 目标：在 session_service.py 的 create 函数中增加 timeout 参数
- 文件：app/services/session.py
- 要求：
  1. 新增 timeout: int = 300 参数
  2. session 创建时设置 TTL
  3. 保持现有函数签名兼容
- 约束：参照 .trae/rules.md 第 2 节函数规范
```

### 3.4 返回格式
```markdown
## Builder Result

### 改动清单
| 文件 | 改动 | 说明 |
|------|------|------|
| app/services/session.py | +8 -2 | 增加 timeout 参数和 TTL 逻辑 |

### 未完成项
- 无 / [具体说明]

### 需要主 Agent 决策的问题
- 无 / [具体问题]

### 自检结果
- ruff check: ✅
- 类型注解: ✅
```

---

## 4. Reviewer（审查员）

### 4.1 职责
- 审查当前工作区的 Git Diff
- 按严重程度分级输出问题
- 不直接修改代码

### 4.2 权限边界

| 操作 | 允许 |
|------|------|
| 读取任意文件 | ✅ |
| 执行 `git diff` / `git log` | ✅ |
| 修改文件 | ❌ |
| 执行任何写命令 | ❌ |

### 4.3 输入格式
```markdown
## Reviewer 任务
- 范围：当前工作区所有改动
- 重点：安全性 / 性能 / 规范符合度
- 参照：.trae/rules.md + .trae/skills/code-review/SKILL.md
```

### 4.4 返回格式
```markdown
## Reviewer Result

### 🔴 严重问题（必须修复）
- [app/services/session.py:28] SQL 拼接存在注入风险 → 改用参数化查询

### 🟡 警告
- [app/api/v1/auth.py:50] 函数长度 58 行，超过 50 行限制 → 拆分为 _validate + _create_session

### 🟢 提示
- [app/services/session.py:15] 可用 asyncio.sleep 替代 time.sleep

### ✅ 通过项
- 类型注解完整
- 无硬编码密钥
- 测试覆盖率达标

### 结论
- 建议合并：是 / 否 / 需修改后合并
```

---

## 5. Tester（测试员）

### 5.1 职责
- 编写单元测试 / 集成测试
- 运行测试套件并报告结果
- 不修改业务代码

### 5.2 权限边界

| 操作 | 允许 |
|------|------|
| 修改 `tests/` 下文件 | ✅ |
| 修改业务代码 | ❌ |
| 执行 `pytest` / `ruff` | ✅ |
| 执行 `git commit` | ❌ |

### 5.3 输入格式
```markdown
## Tester 任务
- 目标：为 session_service.create 编写单元测试
- 被测代码：app/services/session.py
- 测试文件：tests/test_session_service.py
- 用例要求：
  1. 正常创建 session
  2. timeout 参数生效
  3. 数据库异常时抛出 AppError
```

### 5.4 返回格式
```markdown
## Tester Result

### 测试文件
- tests/test_session_service.py（+85 行）

### 用例清单
| 用例 | 状态 |
|------|------|
| test_create_session_returns_session | ✅ pass |
| test_create_session_with_timeout_sets_ttl | ✅ pass |
| test_create_session_raises_on_db_error | ✅ pass |

### 运行结果
- pytest: 3/3 passed
- 覆盖率: session.py 92%（新增 +5%）

### 未覆盖项
- session_service.delete（建议后续补充）
```

---

## 6. 主 Agent 调度模板

主 Agent 拆解任务后，按以下模板生成 Sub Agent 调用计划：

```markdown
## Sub Agent 调度计划

| # | 角色 | 任务摘要 | 输入文件 | 输出预期 | 依赖 |
|---|------|---------|---------|---------|------|
| 1 | investigator | 定位 session bug | - | 代码位置清单 | - |
| 2 | builder | 修复 timeout 逻辑 | #1 结果 | 改动 diff | #1 |
| 3 | tester | 补充测试 | #2 结果 | 测试通过 | #2 |
| 4 | reviewer | 审查改动 | #2,#3 结果 | 问题清单 | #2,#3 |

并行度：串行（#2 依赖 #1，#3 依赖 #2）
预计 Sub Agent 数：4（未超 max_subagents=4）
```

## 7. 错误处理

| Sub Agent 失败情况 | 主 Agent 行为 |
|-------------------|--------------|
| investigator 找不到相关代码 | 暂停，询问用户补充线索 |
| builder 改动未通过自检 | 退回 builder 重试（≤2 次），仍失败则暂停 |
| reviewer 发现严重问题 | 退回 builder 修复后重新 review |
| tester 测试失败 | 退回 builder 修复后重跑测试 |
| 任一 Sub Agent 超时（120s） | 终止该 Agent，主 Agent 接管或暂停 |

## 8. Prompt 模板引用

每个角色的完整 Prompt 模板位于：

```
.trae/prompts/
├── subagent-investigator.md
├── subagent-builder.md
├── subagent-reviewer.md
└── subagent-tester.md
```

主 Agent 通过 slash 命令或自动调度触发对应 Sub Agent。
