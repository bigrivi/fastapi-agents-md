# AGENTS.md — 面向 AI Agent 的 FastAPI 项目开发规范

本文档为在本 FastAPI 代码库中工作的 AI Agent 提供指南。编写或修改代码时请遵循。

- **MUST（必须）**：评审时强制检查（或由工具强制执行）。**SHOULD（应当）**：默认遵循，打破需说明理由。
- 规则适用于**新增和被修改的代码**。不要为了满足规则而大面积重构未触及的代码，发现问题请报告而不是悄悄改掉。

### 适用范围与前提

- **必需技术栈：** Python 3.12+、FastAPI、Pydantic v2、pytest、Ruff。
- **可选（仅当项目使用时适用）：** SQLModel / SQLAlchemy、Alembic、mypy 或 pyright、import-linter、任务队列（如 Celery）、用于外部集成的异步 HTTP 客户端（如 httpx 或 aiohttp）。
- **本文档中的名称只是示例，不保证存在。** `BaseService`、`BaseExternalService`、`ExternalServiceError`、`ResponseModel`、`response_base`、`get_current_user` 等类名/函数名，以及 `app/common/`、`app/modules/`、`app/integrations/` 等包路径，只是用来说明某种模式。**使用前先在代码库中搜索；如果不存在，就沿用项目自己的对应实现——不要因为文档里出现了某个名字，就去导入或凭空创造它。**
- 当本文件与现有代码中清晰、既定的约定冲突时，以现有代码为准，并说明这处差异。

---

## 1. 工作约定（请先阅读）

**编码之前**
1. **先读后写。** 找到最接近的现有模块/功能，沿用它的结构、命名和模式。已有模式时，不要另起炉灶。
2. **非平凡改动先给方案。** 如果改动涉及超过 2 个文件、新增模块、新增外部集成或变更数据库结构，编码前先给出简短方案：要新增/修改的文件、各自职责、依赖方向、测试计划。存在多种合理设计时，等待确认再动手。
3. **有歧义就问，不要猜。** 业务规则不明确时（计算口径、期间/范围规则、权限）请先确认。

**编码过程中**
- 用最小的 diff 解决问题。不要顺手重命名、重新格式化或重构无关代码。
- 一次只处理一个模块/一层；每一步结束时测试都应保持通过。
- 未经询问，不要引入新的第三方依赖。
- 没有具体的第二个使用场景或明确的测试需求，不要新增抽象（基类、Protocol、注册表、工厂）（YAGNI）。
- 不要硬编码密钥、主机地址或 token；不要把它们写进日志。
- 新增代码中的每个参数和返回值都要添加类型注解（§16 *类型注解*）。
- 不要执行破坏性的 git 操作（强制推送、改写历史、删除分支），也不要推送到受保护分支，除非被明确要求。

**编码之后**——遵循 [§15 验证与完成标准](#15-验证与完成标准)。

---

## 2. 设计原则与设计模式

### 2.1 SOLID

本项目采用分层架构，service 层同时承担用例逻辑和持久化。请务实地应用这些原则：

| 原则 | 本代码库中的具体规则 |
|------|----------------------|
| **S** 单一职责 | 一个模块/类只有一个变化原因。HTTP 解析、业务规则、持久化、外部调用不得出现在同一个函数里。文件超过约 300 行或函数超过约 30 行必须拆分（见 §3）。 |
| **O** 开闭原则 | 因类型/模式而异的行为（如分摊方式、导出格式），通过**新增**类/函数并注册到 dict/注册表来扩展，而不是继续加长 `if/elif` 链。出现第三种变体时再引入注册表，不要提前做。 |
| **L** 里氏替换 | 项目基类（如 `BaseService`、`BaseExternalService`）的子类必须保持父类的签名、返回类型和错误契约。不得把方法重写成空操作，也不得抛出无关异常。 |
| **I** 接口隔离 | 优先使用小而专一的 `typing.Protocol` 或普通函数，而不是臃肿的基类。除非至少两个领域需要，否则不要往共享基类（如 `BaseService`）里加方法。 |
| **D** 依赖倒置 | 业务代码从外部获取协作对象（`Depends(...)`、构造函数/函数参数），不得在业务方法内部创建客户端、读取环境变量或打开 session。外部系统只能通过 `app/integrations/` 访问。 |

### 2.2 结构性规则

- **函数式内核，命令式外壳。** 把计算和决策放进纯函数（`rules.py`），入参和返回值都是普通数据；IO 由 service 在外层完成。纯代码极易做单元测试。
- **组合优先于继承。** 只继承项目既有的基类（如 `BaseService`、`BaseExternalService`、SQLModel/Pydantic 模型）。
- **函数优于只有一个方法的类**，`Protocol` 优于 ABC，`dataclass`/Pydantic 优于手写容器或传 `dict`。
- **不要过度设计。** 只有一个实现且看不到第二个的地方，不需要接口。简单明确胜过聪明通用。

### 2.3 核心原则

| 原则 | 在本项目中的含义 |
|------|------------------|
| **DRY**（不要重复自己） | DRY 针对的是**知识**，而不是文本：每条业务规则、常量、枚举/状态值、错误信息、配置值和 schema 结构都只有**唯一的权威定义**。schema 从共享基类派生（`UserBase` → `UserCreate` / `UserRead`），而不是重复声明字段；重复的校验抽成 dependency，重复的查询过滤抽成 service 辅助方法，重复的错误映射抽成异常处理器。 |
| **三次法则**（约束 DRY） | 前两次出现重复可以容忍；**第三次**出现时再抽取，并且仅当这些副本因**同一个原因**而变化时才抽取。只是长得像、但变化原因不同的代码属于*偶然重复*——保持分开。 |
| **KISS** | 选择能工作的最简单设计。不要用元编程、基于 `getattr` 的字符串分发、深层继承或炫技式一行流。读者理解一个用例时，跳转的文件不应超过约 2 个。 |
| **YAGNI** | 不要为没人提出的场景添加预留参数、配置开关、钩子或接口。需求真实出现时再加。 |
| **关注点分离** | 每一层、每个模块只回答一个问题（见 §5）。把 HTTP、业务规则、持久化和外部调用混在一处，是最常见的设计失败。 |
| **迪米特法则** | 只与直接协作者交互。避免跨层/跨模块的长调用链（`order.customer.account.owner.email`）。跨领域时通过对方模块的 service，而不是直接伸手进它的模型。同一模块内访问已加载的 ORM 关系属性是可以的。 |
| **快速失败** | 在边界处（Pydantic、dependencies）以及用卫语句做校验；一旦出错就立刻抛出具体异常。绝不对缺失的必填数据悄悄使用默认值，也不要吞掉错误。 |
| **命令查询分离** | 一个函数要么**返回信息**，要么**修改状态**，不要两者兼具。`get_*`、`list_*`、`calculate_*` 没有副作用；`GET` 路由绝不修改数据；`create_*`、`update_*`、`delete_*` 修改状态，最多返回受影响的对象。 |
| **最小惊讶** | 名称与行为一致；没有隐藏的副作用；相似的 service 具有相似的签名；遵循相邻代码的约定。 |
| **显式优于隐式** | 不要有隐式提交、魔法全局变量，也不要在构造函数、属性或导入时执行 I/O。导入模块不得访问网络、数据库或文件系统（实例化 settings 是唯一例外）。 |
| **优先不可变** | 传入 `rules.py` 的数据视为只读（`@dataclass(frozen=True)`、元组、`Mapping`）。不要修改函数入参；返回新值。 |
| **幂等性** | 任何可能被重试的操作（定时任务、webhook、客户端重试、外部调用）都必须能安全地执行两次：唯一键、upsert，或在同一事务内“检查再执行”。 |
| **限定范围的童子军规则** | 让你碰过的代码稍微变好一点（命名、死代码、缺失的类型）——但仅限于你本来就要修改的那几行。这条规则绝不凌驾于 §1 的“最小 diff”原则。 |

### 2.4 原则冲突时怎么办

按以下顺序取舍：
1. **正确性和清晰度**高于一切。
2. **KISS / YAGNI** 高于 DRY 和可扩展性：重复的代价低于错误的抽象。
3. **DRY** 高于预留的灵活性，但必须在满足三次法则之后。
4. **可扩展模式**（注册表、Strategy、工厂）在出现第三种变体时再引入——不要在第一、二种时就引入。

错误抽象的信号：一个共享函数为了迁就越来越不一样的调用方，不断长出布尔开关或 `if caller == ...` 分支。此时应**把它拆回独立的函数**，而不是再加一个参数。

### 2.5 设计模式：何时用、怎么用

模式是工具，不是目标。只有当它要解决的问题确实存在时才使用。

| 模式 | 适用场景 | 在本项目中的形式 | 避免 |
|------|----------|------------------|------|
| **依赖注入** | 任何协作对象（service、客户端、当前用户、session） | FastAPI `Depends(...)`；测试使用 `app.dependency_overrides` | 在业务方法内部创建客户端/session |
| **服务层** | 用例始终如此 | `service.py` 编排一个用例并持有事务 | 把用例逻辑写进路由 |
| **工作单元** | 一个用例要写入多项且必须一起成功 | 每个用例一个 session/事务，由 service 统一提交一次 | 在辅助函数里到处提交 |
| **DTO / Schema 分离** | API 结构与表结构不同 | `Create` / `Update` / `Read` schema 与表模型分开 | 直接返回包含内部字段的表模型 |
| **适配器 / 防腐层** | 与你无法控制其数据格式的系统通信 | `app/integrations/*` 把外部数据和错误转换为内部模型和 `ExternalServiceError` | 让外部字段名和结构渗入领域代码 |
| **外观（Facade）** | 一个用例涉及多个步骤/协作者 | 一个 service 方法隐藏这些步骤；controller 只调用一次 | controller 依次调用多个 service |
| **策略（+ 注册表）** | 某个行为有 3 个及以上可互换的变体 | `dict[str, Callable]` 或 `Protocol` 实现，按 key 选择 | 随变体增多而变长的 `if/elif`；只有一个变体却建 Strategy 类 |
| **工厂** | 选择具体实现需要非平凡逻辑（按配置/类型） | 普通函数 `create_x(kind) -> X`，或返回实现的 dependency | 只生产一种产品的工厂类 |
| **装饰器 / 中间件** | 横切关注点：日志、计时、重试、缓存 | Python 装饰器、FastAPI 中间件或 dependency | 复制粘贴同样的 try/日志/重试代码块 |
| **Repository** | service 的查询变得庞大、被复用，或需要替换/伪造 | 从 `service.py` 中抽出 `queries.py`（或 repository 类） | 默认给每张表都加一层 repository |
| **观察者 / 领域事件** | 副作用（通知、审计）不应让核心用例膨胀 | 提交后触发；`BackgroundTasks` 或队列消费者 | 在没有多个订阅者之前就引入事件总线 |
| **单例** | 无状态且创建成本高的客户端 | 模块级实例（如 `<service>_service`），需要覆盖时通过 dependency 暴露 | 跨请求共享的可变全局状态 |
| **模板方法** | 几乎不用 | 优先使用组合和普通函数；项目既有的基类（如 `BaseExternalService`）是例外 | 带钩子方法的新继承层级 |

### 2.6 需要重构的代码坏味道

修改代码时如果发现以下情况，在本次改动范围内修复，或者上报：

- **上帝 service/类**：超过 300 行或包含许多互不相关的方法 → 按用例拆分。
- **霰弹式修改**：一个小改动需要在很多文件里同步修改 → 知识被重复，或缺少某个概念。
- **特性依恋**：一个方法大部分时间在操作另一个模块的数据 → 移到那个模块的 service。
- **基本类型偏执**：用字符串/整数/dict 代表某个概念 → 使用枚举、值对象或 Pydantic 模型。
- **标志参数**：用 `do_x(..., force=True)` 切换行为 → 拆成两个函数或使用枚举。
- **过长参数列表**（>5）→ 合并为模型。
- **魔法数字/字符串**：重复出现的字面量 → 放进 `constants.py` 的命名常量或枚举。
- **过度泛化**：未被使用的参数、钩子、基类 → 删除。
- **深层嵌套 / 过长的 `if` 阶梯** → 卫语句、提前返回、抽取函数。
- **复述代码的注释** → 改好命名让注释变得多余；注释只保留“为什么”。

其中一部分坏味道可由 Ruff 机械检出：标志参数（`FBT`）、魔法值（`PLR2004`）、被注释掉的代码（`ERA`）、未使用的参数（`ARG`）、`return` 之后多余的 `else`（`RET`），以及错误处理问题（`TRY`、`BLE`）。配置见 `pyproject.tooling.snippet.toml`。

---

## 3. 体量与复杂度限制

| 项目 | 上限 | 超出时的处理 |
|------|------|--------------|
| 文件 | 约 300 行 | 按职责拆分（如 `service.py` → `service.py` + `queries.py` / `rules.py`） |
| 函数/方法 | 约 30 行 | 抽取有名字的辅助函数 |
| 参数个数 | ≤ 5 | 合并为 Pydantic 模型 / dataclass |
| 圈复杂度 | ≤ 10 | 拆分分支，使用提前返回 |
| 嵌套深度 | ≤ 3 | 提前返回 / 卫语句 |

由 Ruff 强制检查（`C901`、`PLR0912`、`PLR0913`、`PLR0915`），配置见 `pyproject.tooling.snippet.toml`。

---

## 4. 项目结构

按**业务领域**组织代码，而不是按文件类型。下方是**推荐**的目录布局：如果项目已有不同的结构，请沿用现有结构，并相应调整 import-linter 契约。

```
app/
├── common/                  # 跨领域、与业务无关的通用代码
│   ├── response.py          #   统一响应包装（如果项目使用）
│   ├── security/auth.py     #   认证依赖（如 JWT）
│   └── services/            #   共享的基础 service（查询与 CRUD 辅助），如有
├── config/
│   └── settings.py          # 全局配置（读取 .env）
├── integrations/            # 所有对外部系统的调用（见 §11）
│   ├── base.py
│   └── <service>.py
├── modules/
│   └── {domain}/            # 如 users/、orders/、billing/
│       ├── models.py        # 数据库表模型（SQLModel/SQLAlchemy）+ 对外读取模型
│       ├── schemas.py       # Pydantic 请求/响应 schema
│       ├── service.py       # 用例 + 持久化（可继承共享的基础 service）
│       ├── rules.py         # 纯业务规则，无 IO（可选）
│       ├── controller.py    # 路由（或 *_api.py）
│       ├── dependencies.py  # 路由依赖 / 校验器
│       ├── exceptions.py    # 领域专属异常
│       ├── constants.py
│       └── config.py        # 领域专属 BaseSettings（可选）
├── utils/                   # 仅放通用的、与领域无关的辅助函数
├── database.py              # Engine / session
├── exceptions.py            # 全局异常 + 异常处理器
└── main.py                  # 应用初始化、路由注册（组装入口）
tests/
├── conftest.py
├── modules/{domain}/        # 与 app/modules 一一对应
└── test_integrations.py
docs/
```

**这个东西该放哪？**

| 我需要…… | 放在 |
|----------|------|
| 解析请求 / 组装响应 | `controller.py` / `*_api.py`、`schemas.py` |
| 校验路径/查询资源是否存在、检查归属 | `dependencies.py` |
| 做业务判断或计算 | `rules.py`（纯函数） |
| 查询/持久化数据、编排用例、持有事务 | `service.py` |
| 调用其他 HTTP 系统 | `app/integrations/<service>.py` |
| 读取环境变量 | `app/config/settings.py`（或领域下的 `config.py`） |
| 跨领域通用辅助（无业务含义） | `app/common/` 或 `app/utils/` |

**导入约定**——跨领域导入时使用显式的模块名：
```python
from app.modules.users import constants as user_constants
from app.modules.notifications import service as notification_service
```

---

## 5. 分层与依赖规则

```
controller / *_api.py  ──►  service  ──►  rules（纯函数）
                              │
                              ├──►  integrations
                              └──►  models
```

| 层 | 做什么 | 不得做什么 |
|----|--------|------------|
| **Controller / 路由** | 声明路由、解析/校验输入、调用**一个** service 方法、包装响应 | 包含业务逻辑、构造查询、接触数据库 session、调用外部服务、提交事务 |
| **Dependencies** | 校验/解析请求级资源（存在性、归属、鉴权） | 修改数据 |
| **Service** | 编排用例、持有事务、持久化、调用 integrations、抛出领域异常 | 感知 HTTP（`Request`、`HTTPException`、状态码） |
| **Rules** | 对普通数据做纯计算 | 做 IO、读配置、使用 session、调用 `datetime.now()`（时间由外部传入） |
| **Integrations** | 与外部 HTTP 系统通信，把传输层错误转换为 `ExternalServiceError` | 导入领域模块 |
| **Models / schemas** | 描述数据结构 | 导入 `service` 或 `controller` |

硬性依赖规则（**MUST**）：
- `app/common/` 绝不导入 `app/modules/` 或 `app/integrations/`。
- `app/integrations/` 绝不导入 `app/modules/`。
- `models.py` / `schemas.py` 绝不导入 `service.py` / `controller.py`。
- 领域之间不得循环导入。如果 A、B 互相需要，把共享部分抽到第三个模块或 `app/common/`。
- 函数内导入（延迟导入）是打破循环依赖的最后手段：必须加注释 `# NOTE: avoids circular import X <-> Y`，并视为值得上报的设计坏味道。

**事务**
- **service** 持有事务边界：一个用例 = 一次提交。Controller 和辅助函数永远不提交。
- 被多个 service 复用的辅助函数不得自行提交。
- 如果数据已写入后外部调用失败，要显式记录该状态（如把相关记录标记为 `FAILED` 并写入 `error_message`），而不是悄悄留下部分数据。

---

## 6. 异步路由

### 规则
- `async def` 路由：**只**使用非阻塞 I/O（`await` 调用）。
- `def` 路由（同步）：用于阻塞 I/O（会自动在线程池中运行）。
- CPU 密集型任务：交给任务队列（如 Celery、Arq、RQ）或进程池。
- 绝不在 `async def` 中调用阻塞代码（`requests`、`time.sleep`、同步数据库驱动、大文件 I/O）。

### 需要避免的常见错误
```python
# 错误：在 async 路由中使用阻塞调用，会卡住整个事件循环
@router.get("/rates/wrong")
async def get_rate_wrong() -> RateRead:
    response = requests.get(RATE_API_URL, timeout=5)
    return RateRead.model_validate(response.json())

# 正确：在 async 路由中使用非阻塞客户端
@router.get("/rates")
async def get_rate() -> RateRead:
    async with httpx.AsyncClient(timeout=5) as client:
        response = await client.get(RATE_API_URL)
    return RateRead.model_validate(response.json())

# 正确：无法避免阻塞代码时使用普通 `def` 路由（FastAPI 会在线程池中运行）
@router.get("/rates/legacy")
def get_rate_legacy() -> RateRead:
    response = requests.get(RATE_API_URL, timeout=5)
    return RateRead.model_validate(response.json())
```

### 在异步环境中使用同步库
```python
from fastapi.concurrency import run_in_threadpool

@router.get("/reports/{report_id}")
async def download_report(report_id: int) -> ReportRead:
    return await run_in_threadpool(legacy_client.fetch_report, report_id=report_id)
```

---

## 7. Pydantic

### 使用内置校验器
```python
from decimal import Decimal
from pydantic import BaseModel, EmailStr, Field

class ProductCreate(BaseModel):
    sku: str = Field(min_length=3, max_length=32, pattern=r"^[A-Z0-9-]+$")
    name: str = Field(min_length=1, max_length=200)
    price: Decimal = Field(gt=0, max_digits=10, decimal_places=2)
    contact_email: EmailStr
```

### 自定义基础模型
创建共享的基础模型以保证序列化一致。Pydantic v2 中 `json_encoders` 已弃用，请改用带注解的序列化类型：
```python
from datetime import UTC, datetime
from typing import Annotated
from pydantic import BaseModel, ConfigDict, PlainSerializer

def to_utc_iso(value: datetime) -> str:
    if value.tzinfo is None:
        value = value.replace(tzinfo=UTC)
    return value.astimezone(UTC).isoformat()

UtcDatetime = Annotated[datetime, PlainSerializer(to_utc_iso, return_type=str)]

class CustomModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
```

### 按领域拆分 BaseSettings
```python
# app/modules/storage/config.py
from pydantic import SecretStr
from pydantic_settings import BaseSettings

class StorageConfig(BaseSettings):
    BUCKET_NAME: str
    ACCESS_KEY: SecretStr      # 密钥使用 SecretStr，避免在日志/repr 中泄露
    PRESIGNED_URL_TTL_SECONDS: int = 300

storage_settings = StorageConfig()
```
- 每新增一个环境变量，必须在同一次改动中更新 `.env.example`。

---

## 8. Dependencies（依赖注入）

### 用于校验，而不只是 DI
返回有类型的对象（模型/schema），而不是 `dict[str, Any]`。
```python
async def valid_invoice_id(invoice_id: int) -> Invoice:
    invoice = await invoice_service.get_by_id(invoice_id)
    if invoice is None:
        raise InvoiceNotFound()
    return invoice

@router.get("/invoices/{invoice_id}")
async def get_invoice(invoice: Invoice = Depends(valid_invoice_id)) -> InvoiceRead:
    return InvoiceRead.model_validate(invoice)
```

### 依赖链
```python
async def valid_owned_invoice(
    invoice: Invoice = Depends(valid_invoice_id),
    current_user: CurrentUser = Depends(get_current_user),
) -> Invoice:
    if invoice.owner_id != current_user.id:
        raise InvoiceNotOwned()
    return invoice
```

### 关键规则
- 依赖在每个请求内会被缓存（同一依赖被多次调用 = 只执行一次）。
- 优先使用 `async` 依赖，避免线程池开销。
- 使用一致的路径变量名，以便复用依赖。
- 测试通过 `app.dependency_overrides` 替换依赖——不要在生产代码里添加仅供测试的分支。

---

## 9. REST 约定

使用一致的路径变量名以复用依赖：
```
# 两个路由都使用 customer_id，可共用 valid_customer_id 依赖
GET /customers/{customer_id}
GET /subscribers/{customer_id}
```
- 资源用名词，集合路径用复数；动作由 HTTP 动词表达。
- **列表接口必须分页且页大小有上限**——绝不返回无界列表。
- 面向用户的过滤条件要声明为显式的 `Query(...)` 参数，以便出现在 `/docs` 中。
- 不要破坏现有响应结构；除非明确要求，只做增量变更。
- **每个路由都通过返回值注解声明响应类型**（`-> XxxRead`，或项目统一包装的参数化类型），保证 OpenAPI 准确——见 §14。

---

## 10. 数据库

### 命名约定
- 使用 `lower_case_snake` 格式。
- 表名用单数：`invoice`、 `customer`、 `invoice_line`。
- 相关表用前缀分组：`billing_plan`、 `billing_account`。
- 日期时间字段后缀 `_at`（`paid_at`）；日期字段后缀 `_date`（`due_date`）。

### 显式设置索引名称
```python
from sqlmodel import SQLModel

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}
# 纯 SQLAlchemy 写法：MetaData(naming_convention=NAMING_CONVENTION)
SQLModel.metadata.naming_convention = NAMING_CONVENTION
```

### SQL 优先
把面向集合的工作交给 SQL，而不是在 Python 里循环：连接、过滤、聚合、排序和分页。只有当它能省掉额外的往返查询时，才在数据库中构建嵌套 JSON（如 PostgreSQL 的 `json_agg`）。

### 查询规则
- 绝不用字符串拼接/f-string 把用户输入拼进 SQL——使用绑定参数 / SQLAlchemy 表达式。
- 避免 N+1：对会被序列化的关系做预加载（to-one 用 `joinedload`，to-many 用 `selectinload`）。
- 外键和用于过滤/排序的列要建索引。
- 大表只查需要的列；分页在 SQL 中做，不要在 Python 中做。

### 迁移（Alembic）
- 迁移保持静态且可回滚；必须实现 `downgrade`。
- 使用带日期前缀、有描述性且包含 revision id 的文件名：`2026-03-14_a1b2c3_add_invoice_due_date_idx.py`
  ```ini
  # alembic.ini
  file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(rev)s_%%(slug)s
  ```
- **提交前务必审查 `--autogenerate` 的产物**，删除无关改动。
- 不要修改已合并/已应用的迁移——新增一个迁移。
- 结构迁移与数据迁移分开。
- 破坏性变更（删表/删列、类型收窄）需要用户明确确认。

---

## 11. 外部服务集成（`app/integrations/`）

*仅当项目需要调用外部 HTTP 系统时适用，否则可跳过本节。*

所有对外部系统的 HTTP 调用都放在 `app/integrations/`。不要把业务相关的外部服务客户端放进 `app/utils/`（那里只放通用辅助函数），也不要在领域模块里内联 HTTP 客户端调用。

```
app/integrations/
├── __init__.py
├── base.py                      # BaseExternalService：共享 HTTP 客户端、URL 拼接、日志、错误处理
└── payment_gateway.py           # 每个外部服务一个文件
```

规则：
- **一个外部服务 = 一个模块。** 模块内只有一个继承 `BaseExternalService` 的类，**每个 URL 对应一个方法**（`create_charge`、`refund_charge` 等）。
- 声明 `service_name`、`base_url`（来自 `settings`）、`default_timeout`；URL 路径作为模块级常量。
- Base URL / 超时 / token 通过 `app/config/settings.py` 从 `.env` 读取
  （如 `PAYMENT_API_URL`、`PAYMENT_API_TIMEOUT`、`PAYMENT_API_TOKEN`）。
  绝不在服务模块里硬编码主机地址。
- `BaseExternalService` 返回解析后的 JSON，并在传输错误、超时、非 2xx、非法 JSON 时抛出 `ExternalServiceError`。领域模块捕获该错误并转换为自己的状态（例如把相关记录标记为 `FAILED` 并写入 `error_message`）。
- 领域模块导入服务单例（`from app.integrations.<service> import <service>_service`）。绝不允许 `app/integrations/` 导入领域模块。
- 当调用方依赖响应结构时，把响应校验成 Pydantic 模型再返回，而不是到处传裸 dict。
- 重试：仅限幂等请求，且要有上限并带退避。非幂等的 POST 绝不自动重试。
- 日志：记录 `service_name`、路径、状态码和耗时。绝不记录 token，也不记录含个人数据的完整请求/响应体。
- 测试在这个边界处打桩（monkeypatch 服务方法，或用与你的客户端匹配的库模拟 HTTP，如 httpx 用 `respx`、aiohttp 用 `aioresponses`）——绝不访问真实网络。

请在 `docs/` 中记录每个集成的请求/响应契约，并在 `tests/test_integrations.py` 中覆盖其请求结构。

---

## 12. 错误处理与响应

- 领域错误是定义在 `{domain}/exceptions.py` 中的异常，继承自 `app/exceptions.py` 里的全局基类。**Service、rules 和 dependencies 抛出领域异常，不构造 HTTP 响应。** 异常 → 状态码/响应体 的映射在异常处理器中完成。
- 捕获**具体的**异常。禁止裸 `except:`，禁止 `except Exception: pass`。如果必须宽泛捕获（例如在任务边界），要带上下文记录日志，并重新抛出或转换。
- `ExternalServiceError` 由调用方转换为领域状态（见 §11）——不得以原始 500 的形式泄露给客户端。
- 沿用项目现有的响应和错误格式（例如统一响应包装，如果项目使用）。不要另造第二种格式。
- 错误信息不得暴露内部细节（SQL、堆栈、token、上游 URL）。

---

## 13. 安全、配置与日志

- **认证：** 每个非公开路由都要声明项目的认证依赖（如 JWT 或会话的 `Depends(...)`）。新增未认证路由必须在 PR/总结中明确说明理由。
- **授权：** 在 dependency 或 service 中检查归属/权限，不要写在 controller 函数体里。
- **密钥：** 只通过 `.env` → settings 获取；使用 `SecretStr`；绝不提交 `.env`；绝不记录密钥、token、密码，或含个人数据的完整请求体。
- **输入：** 在边界处用 Pydantic 校验；绝不信任路径/查询/请求体中的值直接用于 SQL 或文件路径。
- **日志：** 使用项目的 logger，不要用 `print`。记录标识符和结果，带足够排查问题的上下文；不记录个人隐私信息。
- **文档暴露：** 在非 local/staging 环境隐藏 OpenAPI 文档：
  ```python
  DOCS_ENABLED_ENVIRONMENTS = {"local", "staging"}

  def create_app(settings: Settings) -> FastAPI:
      show_docs = settings.ENVIRONMENT in DOCS_ENABLED_ENVIRONMENTS
      return FastAPI(
          title="My API",
          docs_url="/docs" if show_docs else None,
          redoc_url="/redoc" if show_docs else None,
          openapi_url="/openapi.json" if show_docs else None,
      )
  ```

---

## 14. API 文档与测试

### 规范地编写接口文档
```python
@router.post(
    "/invoices",
    summary="Create an invoice",
    description="Creates a draft invoice for an existing customer.",
    status_code=status.HTTP_201_CREATED,
    tags=["Invoices"],
    responses={
        status.HTTP_404_NOT_FOUND: {"model": ErrorResponse, "description": "Customer not found"},
        status.HTTP_409_CONFLICT: {"model": ErrorResponse, "description": "Duplicate invoice number"},
    },
)
async def create_invoice(payload: InvoiceCreate) -> InvoiceRead: ...
```

### 路由返回值注解决定 OpenAPI 文档（MUST）

**每个路由处理函数都必须声明精确的返回类型——FastAPI 正是据此在 `/docs` 和 `/openapi.json` 中生成响应结构。** 缺少它，响应会被文档化成空的 `{}`，前端、API 调用方和自动生成的 SDK 都看不到接口返回了什么。返回值注解还会让 FastAPI 对返回内容做校验和过滤，避免内部字段泄露。

- **直接返回响应 schema**：`-> RefundCreated`、`-> list[RefundRead]`，或项目的分页类型（如 `-> Page[RefundRead]`）。
- **如果项目对所有响应使用统一包装**（如 `{code, msg, data}`），则声明参数化的包装类型：`-> ResponseModel[RefundCreated]`（见下方的包装变体）。沿用现有路由的写法；项目里没有包装时，不要自行引入。
- **响应类型是 Pydantic 响应 schema**（`...Read`、`...Response`），不是带内部字段的表模型，也不是 `dict`。
- **绝不要**省略返回值注解、只写裸的泛型包装（`-> ResponseModel`），或写 `-> dict` / `-> Any`。这些写法都会让文档里的返回内容变成无类型。
- 没有响应体（如 `DELETE`）？使用 `status_code=204` 配 `-> None`。
- 非 200 的成功状态码用 `status_code=` 声明（如创建返回 `201`），错误响应体用 `responses={...}` 声明（见上面的示例）。
- 如果处理函数必须返回注解无法表达的东西（原始 `Response`、流、文件），在装饰器上显式设置 `response_model=` 或 `response_class=`，而不是干脆去掉类型。
- 实际返回的值必须与注解一致——FastAPI 会在运行时校验。
- 请求入参同样要文档化：带类型的路径/查询参数使用 `Query(..., description=...)`，请求体使用 Pydantic schema。

```python
@router.post(
    "/orders/{order_id}/refunds",
    summary="申请订单退款",
    status_code=status.HTTP_201_CREATED,
    dependencies=[Depends(get_current_user)],
    responses={status.HTTP_404_NOT_FOUND: {"model": ErrorResponse}},
)
async def create_refund(
    order_id: int,
    payload: RefundCreate,
) -> RefundCreated:                      # <- 生成 201 响应的 schema
    return await refund_service.create(order_id, payload)

# 错误：以下每种写法都会让 /docs 中的响应变成无类型
async def create_refund(order_id: int, payload: RefundCreate): ...             # 没有注解：响应 schema 为 {}
async def create_refund(order_id: int, payload: RefundCreate) -> dict: ...     # 泛型 object
```

**包装变体（仅当项目使用统一响应包装时适用）。** 包装本身必须是泛型，否则无法写出 `ResponseModel[X]`，文档里的返回内容也就一直是无类型。如果现有包装还不是泛型（例如 `data: Any`），按下面的方式改造，同时保留其字段名，避免影响调用方。`ResponseModel` 和 `response_base` 只是示例名称——请使用项目自己的。

```python
# e.g. app/common/response.py
from pydantic import BaseModel

class ResponseModel[T](BaseModel):
    """Standard envelope. The type parameter becomes the typed `data` in OpenAPI."""

    code: int = 200
    msg: str = "Success"
    data: T | None = None

class Page[T](BaseModel):
    """Paginated payload, used as ResponseModel[Page[OrderRead]]."""

    items: list[T]
    total: int
    page: int
    size: int

class ResponseBase:
    def success[T](self, *, data: T | None = None, msg: str = "Success") -> ResponseModel[T]:
        return ResponseModel[T](code=200, msg=msg, data=data)

    def fail(self, *, code: int, msg: str) -> ResponseModel[None]:
        return ResponseModel[None](code=code, msg=msg, data=None)

response_base = ResponseBase()
```

- 使用 PEP 695 语法（`class ResponseModel[T](BaseModel)`，Python 3.12+，Pydantic v2）。
- `success()` 是泛型方法，返回 `ResponseModel[T]`，这样 mypy/pyright 能检查你传入的 `data` 是否与路由声明的返回类型一致。
- 无数据：`ResponseModel[None]`。列表：`ResponseModel[list[OrderRead]]`。分页：`ResponseModel[Page[OrderRead]]`（如果项目已经在用分页库的 `Page[T]`，如 `fastapi-pagination`，直接用它，不用再定义 `Page`）。
- FastAPI 会按类型参数给 OpenAPI 组件命名（如 `ResponseModel_RefundCreated_`），所以每个接口的 `data` 都会显示为各自具体的 schema。

```python
async def create_refund(order_id: int, payload: RefundCreate) -> ResponseModel[RefundCreated]:
    data = await refund_service.create(order_id, payload)
    return response_base.success(data=data)

# 错误：裸的包装类型，/docs 中的 `data` 无类型
async def create_refund(order_id: int, payload: RefundCreate) -> ResponseModel: ...
```

**如何验证：** 新增或修改路由后，打开 `/docs`，确认响应里显示的是具体的字段，而不是空对象。可以加一个契约测试防止回退——只要有 2xx 响应没有带类型的 schema 就会失败：

```python
from typing import Any
from app.main import app as fastapi_app

def _is_untyped(schema: dict[str, Any], components: dict[str, Any]) -> bool:
    if not schema:
        return True  # 没有返回值注解
    if schema.get("type") == "object" and schema.get("additionalProperties") is True:
        return True  # -> dict
    ref = schema.get("$ref")
    if ref:
        target = components[ref.rsplit("/", 1)[-1]]
        data = target.get("properties", {}).get("data", {})
        return {} in data.get("anyOf", [])  # 裸的 `-> ResponseModel`
    return False

def test_every_2xx_route_documents_a_typed_response() -> None:
    spec = fastapi_app.openapi()
    components = spec.get("components", {}).get("schemas", {})
    offenders: list[str] = []
    for path, operations in spec["paths"].items():
        for method, operation in operations.items():
            for code, response in operation.get("responses", {}).items():
                if not code.startswith("2") or code == "204":
                    continue
                schema = response.get("content", {}).get("application/json", {}).get("schema", {})
                if _is_untyped(schema, components):
                    offenders.append(f"{method.upper()} {path}")
    assert not offenders, f"Routes without a typed response schema: {offenders}"
```
（`$ref` / `data` 这一分支只在项目使用统一响应包装时才有意义——请按实际字段名调整，否则删掉该分支。有意返回文件/流的路由可以加入白名单。）

### 测试金字塔
| 层级 | 测什么 | 怎么测 |
|------|--------|--------|
| 单元 | `rules.py` 和纯辅助函数 | 不连数据库、不走网络，只需要数据。速度快；大多数测试应在这一层。 |
| Service | 用例 + 持久化 | 使用真实测试库；每个测试相互隔离（事务回滚或清表）。对 integrations 打桩。 |
| API | 路由、鉴权、状态码、响应结构 | 使用 `AsyncClient` + `ASGITransport`；用 `app.dependency_overrides` 替换鉴权/服务。 |
| Integration 客户端 | `app/integrations/*` 的请求结构 | 放在 `tests/test_integrations.py`；模拟 HTTP，断言 URL/请求头/请求体/超时及错误映射。 |

### 规则
- 从一开始就使用异步测试客户端：
  ```python
  from collections.abc import AsyncIterator

  import pytest
  from httpx import ASGITransport, AsyncClient

  from app.main import app as fastapi_app

  @pytest.fixture
  async def api_client() -> AsyncIterator[AsyncClient]:
      transport = ASGITransport(app=fastapi_app)
      async with AsyncClient(transport=transport, base_url="http://testserver") as client:
          yield client

  @pytest.mark.asyncio
  async def test_create_invoice_returns_201(api_client: AsyncClient) -> None:
      response = await api_client.post("/invoices", json={"customer_id": 1})
      assert response.status_code == 201
  ```
- 新功能 ⇒ 同一次改动中附带新测试。修 bug ⇒ 先写会失败的回归测试。
- 不仅测正常路径，还要测失败路径（校验错误、未找到、无权限、`ExternalServiceError` → `FAILED`）。
- 一个测试只验证一个行为；测试命名为 `test_<被测单元>_<场景>_<预期>`。
- 通过 fixture/工厂共享准备逻辑；不要复制粘贴准备代码块。
- 测试绝不访问真实网络，也不使用共享/生产数据库。
- 如果测试需要大量 mock 内部实现，说明被测代码职责过多——应修正设计。

---

## 15. 验证与完成标准

修改 Python 代码后，运行：

```shell
ruff check --fix app tests
ruff format app tests
pytest -q                         # 或只跑相关子集：pytest tests/modules/<domain> -q
# 在 pyproject.toml 中配置后：
mypy app                          # 或：pyright
lint-imports                      # import-linter 分层契约
```

“完成”意味着以下**全部**满足：
1. 被改动文件通过 Ruff（lint + format）。
2. 相关测试通过；新行为有对应测试。
3. 类型检查和 import-linter 通过（若已配置）。
4. 没有缺少类型注解的函数（每个参数和返回值都有类型），也没有新增 `Any`、裸 `except`、`print`、硬编码的密钥/主机，也没有不带注释的函数内导入。
5. 每个新增/修改的路由在 `/docs` 中都显示具体的响应结构（带类型的返回值注解）。
6. 迁移（如有）已审查、可回滚，且能干净地应用。
7. 最终回复要说明**运行了哪些检查及其结果**，并列出未能验证的内容。

没有验证就不要声称改动有效。如果某项检查无法运行，请明确说明。

---

## 16. Python 约定

- 使用 Python 3.12+ 语法：`X | None`（而非 `Optional`）、内置泛型（`list[str]`）、`StrEnum`、`pathlib`、f-string、`datetime.now(UTC)`（而非 `utcnow()`）。
- 遵循 PEP 8；使用 Ruff 做 lint 和格式化；启用 Ruff `N`（`pep8-naming`）规则。
- 变量、函数、参数使用 `snake_case`；类使用 `PascalCase`；常量使用 `UPPER_SNAKE_CASE`。
- 布尔变量使用 `is_`、`has_`、`can_` 或 `should_` 前缀。
- **类型注解是强制要求**，每个参数和返回值都要有——见下方[类型注解](#类型注解must)。
- 不使用可变默认参数；不使用通配符导入；不使用 `print`。
- 注释解释**为什么**，而不是做了什么。公开的 service 方法写一行 docstring（用途；不明显时写 `Raises:`）。

### 类型注解（MUST）

`app/` 下的每个函数和方法——路由处理函数、dependencies、service 方法、辅助函数（包括私有函数）、校验器——都必须为**每个参数和返回值**添加类型注解。测试和 Alembic 迁移文件除外。

1. **签名中的一切都要注解**，不返回任何值的函数写 `-> None`。`self` / `cls` 除外。`*args` / `**kwargs` 按元素类型注解（`*args: int`、`**kwargs: str`）。
2. **使用标准的现代语法（Python 3.12+）**，不要使用旧式 `typing` 别名：
   - 用 `str | None`、`int | str`——不要用 `Optional[...]` / `Union[...]`。
   - 用 `list[str]`、`dict[str, int]`、`tuple[int, ...]`、`set[str]`——不要用 `List`、`Dict`、`Tuple`、`Set`。
   - 抽象类型来自 `collections.abc`：`Sequence`、`Mapping`、`Iterable`、`Iterator`、`AsyncIterator`、`Callable`、`Awaitable`。
   - 只有没有内置等价物的才从 `typing` 导入：`Protocol`、`TypedDict`、`Literal`、`Annotated`、`Self`、`Final`、`ClassVar`、`cast`、`TYPE_CHECKING`。
   - 泛型和类型别名使用 PEP 695 语法：`def first[T](items: Sequence[T]) -> T`、`type UserId = UUID`。
3. **参数接受最通用的类型，返回值用具体类型**：入参用 `Sequence[X]` / `Mapping[K, V]` / `Iterable[X]`；返回 `list[X]` / `dict[K, V]`。
4. **返回类型要明确且精确。** `async def` 注解的是 await 之后的结果（`async def get_user() -> User`），而不是 `Coroutine`。生成器使用 `Iterator[X]` / `AsyncIterator[X]`。**路由处理函数必须注解精确的响应类型**（`-> OrderRead`，或项目统一包装的参数化类型）——它决定 OpenAPI 响应结构（见 §14）。Dependencies 返回具体类型（见 §8）。
5. **结构化数据使用有名字的类型**——Pydantic 模型、`dataclass`、`TypedDict` 或 `NamedTuple`——而不是 `dict[str, Any]` 或 `tuple[Any, ...]`。取值固定的集合使用 `StrEnum` 或 `Literal`。
6. **属性也要注解**：类属性（dataclass / Pydantic / SQLModel 字段本身就要求带类型），以及不应被修改的模块级常量（`MAX_PAGE_SIZE: Final = 100`）。
7. **`Any` 是最后手段**：只允许出现在真正动态的边界（如无类型的第三方数据），并用注释说明原因。优先校验成 Pydantic 模型，或在只需透传值时用 `object`。
8. **抑制检查必须具体且有理由**：`# type: ignore[error-code]  # 原因` / `# noqa: ANN401  # 原因`。不允许裸 `# type: ignore`。不要通过把类型弱化为 `Any` 或 `object` 来让检查器闭嘴——修正类型本身。
9. **FastAPI / Pydantic / SQLModel 会在运行时解析注解。** 在路由、schema 和模型模块中，要正常导入所用的类型（不要放在 `if TYPE_CHECKING:` 下），也不要加 `from __future__ import annotations`。只有真正的自引用才使用字符串前向引用。

```python
from collections.abc import Sequence
from decimal import Decimal
from uuid import UUID

def total_amount(items: Sequence[OrderItem], discount: Decimal | None = None) -> Decimal: ...

async def find_user(user_id: UUID) -> User | None: ...

def mark_failed(job: Job, error_message: str) -> None: ...

# 错误
def total(items, discount=None): ...                       # 没有类型注解
def total(items: List[Dict]) -> Optional[Any]: ...         # 旧式别名 + Any
async def find_user(user_id: UUID) -> Coroutine[Any, Any, User]: ...  # 应注解 await 之后的结果
```

由 Ruff 的 `ANN`（缺失参数/返回值注解）和 `UP`（旧式 `typing` 语法）强制检查，另外 mypy 中启用了 `disallow_untyped_defs`——见 `pyproject.tooling.snippet.toml`。

### 命名：使用完整、有意义的单词
```python
# 错误：单字母和被截断的名字
for v in validators: ...
lo = min_date(row)
bucket[v] = v in found

# 正确
for validator in validators: ...
resolved_min_date = min_date(row) if callable(min_date) else min_date
bucket[value] = value in found
```

- 局部变量、循环变量和推导式目标变量使用完整单词
  （`value`、`field`、`record_index`、`query_result`、`prefetch_hook`），
  绝不使用裸的 `v`、`f`、`r`、`e`、`x`，也不使用 `conds`、`fmts`、`dec`、`pre` 这类截断写法。
- 命名要体现“已解析出的值”，而不是计算过程：`resolved_cache_key`、
  `resolved_min_date`、`parsed_decimal`（而不是 `key`、`lo`、`dec`）。
- 谓词变量读起来像断言：`is_allowed`、`distinct_pending`。
- 如果重命名会波及公开调用面，可保留广为接受的领域/API 名称
  （`ctx`、`db`、`row`），并在出现处注明这一例外。
- `ruff check` 通过 `pep8-naming` 强制机械层面的命名规范；
  命名是否有描述性属于代码评审的职责，没有 linter 能检查。

---

## 17. 项目常用命令

### 数据库迁移
```shell
alembic upgrade head                                # 应用迁移
alembic revision --autogenerate -m "description"    # 创建新迁移（提交前务必审查！）
```

### 启动开发服务器
```shell
uvicorn app.main:app --reload --port 8000
```

---

## 18. 速查表

| 场景 | 方案 |
|------|------|
| 非阻塞 I/O | `async def` 路由 + `await` |
| 阻塞 I/O | `def` 路由（同步） |
| 异步中使用同步库 | `run_in_threadpool()` |
| CPU 密集型 | 任务队列 / 进程池 |
| 请求校验 | 带数据库检查的 Dependencies |
| 共享校验 | 依赖链 |
| 按领域划分配置 | 独立的 `BaseSettings` 类 |
| 复杂数据库查询 | 使用 JSON 聚合的 SQL |
| 业务计算 | `rules.py` 中的纯函数 |
| 调用外部系统 | `app/integrations/<service>.py` |
| 现有行为新增一种变体 | 新增类/函数并注册；不要继续扩展 `if/elif` |
| 需要提交事务 | 只在 service 中，每个用例一次 |
| 领域错误 | 抛出领域异常；由处理器映射为 HTTP 响应 |
| 测试依赖外部服务 | 在 integrations 边界处打桩 |
| 同样的逻辑第二次出现 | 先保持重复；第三次出现时再抽取，且仅当这些副本因同一原因变化 |
| 共享函数为迁就两个调用方需要加布尔开关 | 拆成两个函数（或改用枚举） |
| 同一规则/常量/状态在多处使用 | 只保留一处定义（常量、枚举或 `rules.py` 函数），到处导入 |
| 编写任何函数 | 每个参数和返回值都加注解（`-> None` 也要写），使用 `X | None` / `list[str]` 语法 |
| 编写路由处理函数 | 返回值注解写具体的响应类型（`-> XxxRead`）——它生成 OpenAPI 响应 schema |
