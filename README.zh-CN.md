# fastapi-agents-md

[English](README.md) | **简体中文**

> 一套面向 FastAPI 项目的 `AGENTS.md` 规范，让 AI 编程助手写出分层清晰、类型完整、易维护的代码，而不是把所有逻辑堆进一个文件。

用 AI 做 vibe coding 时，它默认走的是"满足当前这句话的最短路径"：一个巨大的文件、写在路由里的业务逻辑、没有类型注解的函数，以及满屏空 `{}` 响应的 OpenAPI 文档。本仓库把"架构品味"翻译成 AI 每次任务都会读到的**明确、可检查的规则**，并用 **lint 和测试**强制执行其中最重要的部分，不再依赖 AI 的自觉。

## 仓库内容

| 文件 | 用途 |
|------|------|
| [`templates/AGENTS.md`](templates/AGENTS.md) | 规范正文（英文），放到你的项目根目录即可。 |
| [`templates/AGENTS.zh-CN.md`](templates/AGENTS.zh-CN.md) | 中文版，结构和章节编号与英文版一致。 |
| [`templates/pyproject.tooling.snippet.toml`](templates/pyproject.tooling.snippet.toml) | Ruff、mypy、import-linter 的配置，用工具机械地强制执行规则。 |

## 规范涵盖的内容

| # | 章节 | 要点 |
|---|------|------|
| 1 | 工作约定 | 先读后写、非平凡改动先出方案、最小 diff、需求有歧义先问 |
| 2 | 设计原则与设计模式 | SOLID、DRY + 三次法则、KISS、YAGNI、快速失败、命令查询分离、幂等性；Strategy、Facade、Adapter、Repository 等何时用、何时不用；代码坏味道 |
| 3 | 体量与复杂度限制 | 文件约 300 行、函数约 30 行、参数 ≤ 5、圈复杂度 ≤ 10 |
| 4–5 | 项目结构与分层 | 按领域组织、"这个东西该放哪"对照表、controller → service → rules、依赖方向、事务归属 |
| 6–9 | 异步、Pydantic、依赖注入、REST | 阻塞调用陷阱、v2 序列化、用 dependency 做校验、列表必须分页 |
| 10 | 数据库 | 命名、索引、N+1、安全查询、经过审查且可回滚的 Alembic 迁移 |
| 11 | 外部服务集成 | 一个外部服务一个模块、带类型的错误、有上限的重试、在边界处打桩测试 |
| 12–13 | 错误处理、安全与日志 | 领域异常由处理器映射、默认鉴权、密钥处理 |
| 14 | API 文档与测试 | **用返回值注解生成 OpenAPI**（直接返回 schema，另附可选的泛型响应包装写法）、OpenAPI 契约测试、测试金字塔 |
| 15 | 完成标准 | AI 必须运行并汇报的检查项 |
| 16 | Python 约定 | **强制类型注解**（3.12 现代语法）、命名 |
| 17–18 | 命令、速查表 | 一页速查 |

规则分为 **MUST**（评审或工具强制执行）和 **SHOULD**（默认遵循，打破需说明理由）。规则只适用于新增和被修改的代码，因此引入时不需要一次性大重构。

## 快速开始

```shell
# 1. 把规范复制到你的项目根目录
cp templates/AGENTS.md /path/to/your-project/AGENTS.md

# 2. 把工具配置合并进你的 pyproject.toml
#    （参考 templates/pyproject.tooling.snippet.toml）

# 3. 安装规范依赖的开发工具
pip install ruff mypy import-linter pytest pytest-asyncio httpx

# 4. 验证
ruff check app tests && ruff format --check app tests
mypy app
lint-imports
pytest -q
```

**你的工具读取哪个文件？** 原生读取 `AGENTS.md` 的工具无需额外操作。Claude Code 读取的是 `CLAUDE.md`，所以新建一个文件来引用规范：

```shell
echo "@AGENTS.md" > CLAUDE.md
```

具体加载哪个文件名，请以所用工具的文档为准。

## 推荐的工作流

1. **先设计，后编码。** 让 AI 先给出模块结构、抽象、依赖方向和扩展点，不写代码，由你审核。
2. **一次只实现一个模块**，每个模块都带测试。
3. **让工具当裁判。** AI 必须运行 lint、类型检查、import-linter 和测试，并汇报运行了什么（完成标准，§15）。
4. **定期审查。** 让 AI 按 `AGENTS.md` 审查某个目录，先列出违反点（文件、行号），再决定是否修改。

## 按你的项目定制

模板假设使用 **Python 3.12+、FastAPI、Pydantic v2、SQLModel/SQLAlchemy（异步）、Alembic 和 pytest**。使用前请调整：

- [ ] **§4 项目结构**——改成你的真实目录，并同步修改 import-linter 契约。
- [ ] **适用范围与前提**——对照 `AGENTS.md` 开头的必需/可选技术栈检查你的项目。文件里已经说明 `BaseService`、`ResponseModel` 等名称只是*示例*；如果你的项目里有对应实现，仍建议替换成真实名称，让 AI 有具体的参照。
- [ ] **§11 外部服务集成**——如果没有外部集成，直接删掉。
- [ ] **§3 数值限制**——按团队情况调整阈值（以及对应的 Ruff 配置）。
- [ ] **§1 的 git 安全约定**——按你团队的分支模型调整；如有项目专属的 agent skill 或命令，自行补充一节。

## 在已有代码库中引入

1. 运行 `ruff check --statistics app`，看看差距有多大。
2. 用 `per-file-ignores` 修复或记账存量违规；用 mypy 的 `overrides` 豁免历史模块。
3. 对新增和修改的代码执行完整标准。
4. 逐步缩小豁免范围。

## 仓库结构

```
fastapi-agents-md/
├── README.md
├── README.zh-CN.md
├── LICENSE
└── templates/
    ├── AGENTS.md
    ├── AGENTS.zh-CN.md
    └── pyproject.tooling.snippet.toml
```

规范文件特意放在 `templates/` 下：如果仓库根目录放一份 `AGENTS.md`，它会成为约束*本仓库*的 AI 指令，而不是你自己项目的。

## 参与贡献

欢迎提交 Issue 和 Pull Request。请注意：

- 规则要**具体、可检查**（"函数不超过 30 行"优于"函数要短"）；
- 能用工具强制执行的规则，请同时补充到工具配置里；
- 中英文版本保持同步，包括章节编号。

## 许可证

[MIT](LICENSE) © 2026 <你的名字或组织>

## 致谢

其中若干思路——按领域组织项目、把依赖当作请求校验器、异步/同步的常见陷阱——受到社区 FastAPI 指南的启发，尤其是 [zhanymkanov/fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices)。本仓库是独立作品，与其无隶属关系，代码示例均为重新编写。
