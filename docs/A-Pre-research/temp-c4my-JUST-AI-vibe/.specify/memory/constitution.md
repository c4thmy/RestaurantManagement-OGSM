# 餐饮经营分析 BI 中台工具 Constitution

> 适用范围：餐饮行业 BI 中台软件产品全生命周期开发与迭代
> 技术路线：OGSM 落地工具（主界面）+ 传统 WebUI（BI 看板）+ AI Agent Skills 模块化 + 通用数据输入接口
> 核心定位：**软件即 OGSM 落地的数字载体**，BI 数据看板为 G 目标进度提供自动计算支撑，四步法工作台是 OGSM 的执行引擎

---

## Core Principles

### 0. OGSM 驱动设计（OGSM-First）— 不可妥协

**软件的核心价值是让 OGSM 方法论可落地、可追踪、可闭环。所有功能设计必须能追溯到 OGSM 的某个要素。**

OGSM 四要素与软件模块的强制映射：

| OGSM 要素 | 软件模块 | 设计约束 |
|---|---|---|
| O（目的） | OGSM 一张表主界面 | 首页即 OGSM 表，不得将其降级为附加功能 |
| G（目标） | G 目标管理 + BI 看板自动驱动 | G 进度必须由 BI 数据自动计算，禁止纯手工填写 |
| S（策略） | 四步法分析工作台 | 四步法分析结果必须能直接生成 M 行动，不得断链 |
| M（衡量） | 行动清单 + 闭环跟踪 | M 行动完成后必须自动触发关联 G 目标进度重算 |

**OGSM 数据流向（不可打破）**：
```
BI 数据导入 → G 目标进度自动更新 → 偏差触发四步法分析
     ↑                                        ↓
M 行动完成 ←── 行动清单闭环跟踪 ←── Step 4 生成 M 行动
```

**禁止事项**：
- 禁止将 OGSM 一张表设计为"报表页"或"附加功能"
- 禁止 G 目标进度只能手工填写（必须有自动计算路径）
- 禁止四步法分析结果与 OGSM 行动清单断开连接

---

### I. 数据输入接口优先（Data Contract First）— 不可妥协

**所有功能开发必须以数据接口契约为起点，而非 UI 或业务逻辑。**

- 定义通用可扩展的餐饮数据输入标准（见下方"数据接口约定"章节）
- 每个数据源适配器必须实现统一的 `DataSourceAdapter` 接口
- 接口变更必须向后兼容，破坏性变更需升级主版本号
- 数据质量校验在接口层强制执行，不允许脏数据进入业务层
- 支持的输入形式：CSV/Excel 上传、API 推送、数据库直连、手工录入表单

**数据接口契约字段标准（餐饮通用核心字段）：**

```
必填字段（MUST）：
  - record_date: ISO8601 日期
  - store_id: 门店唯一标识
  - category: 数据类别（sales/inventory/cost/staff/customer）
  - amount: 金额/数量（Decimal，精度 2 位）
  - currency: 货币代码（默认 CNY）

可选扩展字段（SHOULD）：
  - dish_id / ingredient_id / supplier_id
  - shift: 班次（morning/afternoon/evening）
  - channel: 销售渠道（dine-in/takeout/delivery）
  - operator_id: 操作员

元数据字段（系统自动填充）：
  - source_type: 数据来源类型
  - ingested_at: 入库时间戳
  - schema_version: 接口版本号
```

---

### II. WebUI 用户体验一致性（UX Consistency）— 不可妥协

**传统 BI WebUI 是用户主要交互界面，必须保持直观、一致、可预期。**

- 任何页面的核心操作路径不超过 **3 步**
- 所有图表组件遵循统一的视觉规范（颜色、字体、间距）
- 响应式布局：支持 1280px 以上桌面端，1024px 平板端降级展示
- 数据加载状态必须有明确的 Loading / Empty / Error 三态反馈
- 关键操作（删除、覆盖导入）必须有二次确认
- 报表导出（Excel/PDF）为标配功能，不得缺失

---

### III. Skills 模块化封装（Skill-First Modularity）

**固定业务流程必须封装为可复用 Skill，禁止在主流程中内联重复逻辑。**

核心 Skills 清单（必须实现）：

| Skill 名称 | 触发场景 | 输出契约 |
|---|---|---|
| `data-import` | 数据文件上传/API 接入 | 校验报告 + 入库记录数 |
| `data-quality-check` | 任意数据写入前 | 质量评分 + 异常字段列表 |
| `report-generator` | 报表生成请求 | 结构化报表 JSON + 可视化配置 |
| `anomaly-detector` | 定时/手动触发 | 异常列表 + 置信度 + 建议 |
| `forecast-engine` | 预测分析请求 | 预测值 + 置信区间 + 模型说明 |
| `export-packager` | 导出请求 | 文件路径 + 格式 + 大小 |

每个 Skill 必须：
- 有独立的 `SKILL.md` 描述文件
- 有对应的单元测试（覆盖率 ≥ 80%）
- 输入/输出使用 JSON Schema 严格定义（详见 [skills-schema.md](skills-schema.md)）
- 支持 `context: fork` 隔离执行，不污染主会话状态

---

### IV. 测试优先（Test-First）— 不可妥协

**TDD 是强制要求，不是建议。**

- 红绿重构循环严格执行：测试先写 → 确认失败 → 实现代码 → 确认通过
- 禁止在没有对应测试的情况下合并业务代码
- 禁止修改已通过的测试来让代码"通过"
- 覆盖率门禁：核心业务逻辑 ≥ 80%，数据接口层 ≥ 90%
- 双 Agent 对抗测试：`backend-coder` 实现，`test-writer` 独立验证

测试分层要求：

```
单元测试：数据校验逻辑、计算公式、Skill 内部逻辑
集成测试：数据导入→存储→查询→报表 完整链路
E2E 测试：关键用户操作路径（Playwright）
性能测试：高频查询接口 < 200ms（P95）
```

---

### V. 代码质量门禁（Quality Gates）

**所有代码合并前必须通过自动化质量检查，无例外。**

后端（Python）：
- `black` + `isort` 格式化，CI 强制检查
- `mypy` 类型检查，严格模式
- `bandit` 安全扫描，高危漏洞阻断合并
- `pytest --cov` 覆盖率报告，低于阈值阻断合并

前端（TypeScript/Vue）：
- `prettier` + `eslint` 格式化与规范检查
- `vitest` 单元测试，`playwright` E2E 测试
- `tsc --noEmit` 类型检查，零错误要求
- Bundle 体积监控：首屏 JS < 300KB（gzip）

---

### VI. 性能基线（Performance Baseline）

**性能要求是功能需求的一部分，不是优化项。**

| 指标 | 要求 | 测量方式 |
|---|---|---|
| OGSM 一张表加载 | P95 < 1s | API 响应时间（含 G 进度计算） |
| 四步法工作台每步响应 | P95 < 500ms | API 响应时间 |
| 列表/看板查询 | P95 < 200ms | API 响应时间 |
| 多店对比（20家，90天） | P95 < 3s | 端到端时间 |
| 报表生成（< 10万行） | P95 < 3s | 端到端时间 |
| 数据导入（< 5MB） | P95 < 30s | 含质量检查时间 |
| 首屏加载 | LCP < 2.5s | Lighthouse |
| 并发支持 | 50 用户同时操作不降级 | 压测 |

---

### VII. 安全基线（Security Baseline）

- 不存储支付信息、银行卡号等 PCI 数据
- 密码必须使用 bcrypt/argon2 哈希存储，禁止明文
- API 密钥通过环境变量注入，禁止硬编码在代码或配置文件中
- 所有外部输入必须经过参数化查询，禁止 SQL 拼接
- 文件上传限制：类型白名单（csv/xlsx/json）+ 大小上限（50MB）
- CORS 策略：生产环境严格限制来源域名

---

### VIII. AI 能力集成原则（AI-Augmented, Human-Controlled）

**AI 是增强工具，不是替代决策者。**

- AI 分析结果必须附带置信度和数据来源说明
- 预测/异常检测结果以"建议"形式呈现，用户可接受/忽略/反馈
- AI 功能降级策略：模型不可用时，系统退回规则引擎，不中断服务
- 用户反馈数据用于模型迭代，但需明确告知用户并获得同意
- AI 调用的 API Key 通过环境变量管理，变量名由用户指定

---

## 技术栈约定

### 后端
- Python 3.11 + FastAPI + SQLAlchemy 2.x + SQLite（单机）/ PostgreSQL（多租户）
- Alembic 数据库迁移，禁止手动修改 schema
- Celery + Redis 异步任务队列（报表生成、数据导入）
- 分层架构：`presentation → application → domain → infrastructure`

### 前端
- Vue 3 + TypeScript + Vite + Element Plus + Pinia
- ECharts 图表库（统一，禁止混用其他图表库）
- Axios + 统一请求拦截器（错误处理、Token 刷新）
- 组件命名：PascalCase；文件命名：kebab-case

### 数据接口扩展机制
- 新餐饮业态适配：实现 `DataSourceAdapter` 抽象类（详见 [data-source-adapter.md](data-source-adapter.md)）
- 新字段扩展：通过 `schema_version` 版本化，旧版本继续兼容
- 多租户隔离：`store_id` + `tenant_id` 双层隔离（详见 [multi-tenant-isolation.md](multi-tenant-isolation.md)）

---

## Git 工作流

- 分支策略：`main` / `develop` / `feature/xxx` / `fix/xxx`
- 提交规范：Conventional Commits（`feat:` / `fix:` / `test:` / `docs:`）
- PR 要求：必须通过 CI（lint + type-check + test + coverage）才能合并
- 禁止直接推送 `main` 分支，必须通过 PR + Review

---

## 领域知识映射（业务方法论 → 工具功能）

> 来源：`obsidian-cyjyfx-docs/` 领域知识库，所有功能设计必须可追溯到此映射。

### 3C 框架 → 功能模块对应

| 3C 维度 | 方法论核心 | 工具功能模块 | 核心指标 |
|---|---|---|---|
| 自身分析（看自己） | 财务/成本/运营效率 | 经营总览看板 + 成本费用分析 | 营收/毛利率/食材成本率/净利润率/翻台率/人效/坪效 |
| 顾客分析（看用户） | 客流/客单/会员/复购 | 顾客资产分析 | 客流量/客单价/复购率/留存率/新老客占比/CAC |
| 竞争分析（看对手） | 赛道/竞对/差异化 | 竞对分析（P3，预留接口） | 品类市占率/竞对客单/产品结构差异 |

### 四步法 → 工作流设计约束

| 步骤 | 方法论要求 | 工具必须支持 |
|---|---|---|
| 一比目标 | 预算/同比/环比三表，差异率>5%标红 | 自动加载 G 目标 vs 实际值，差异高亮 |
| 二拆数据 | 渠道→时段→来源→商圈逐层下钻 | 收入分析多维下钻，操作≤3步 |
| 三定归因 | 收入异动 = 客单因素 + 客流因素 | 瀑布图因素拆解，后端公式计算 |
| 四做决策 | 行动清单 Who/When/What + 闭环跟踪 | Step 4 直接生成 M 行动写入 OGSM |

### OGSM 内置 G 目标模板（系统预置，不可删除）

| 模板 ID | 目标描述 | 关联 BI 指标 key | 默认阈值 |
|---|---|---|---|
| G_PROFIT | 单店净利润率提升 | `net_profit_rate` | > 15% |
| G_COST | 综合费用率下降 | `total_expense_rate` | 租售比 < 15%，人工 < 25% |
| G_MEMBER | 会员复购率提升 | `member_repurchase_rate` | 目标 +10% |
| G_EFFICIENCY | 人效提升 | `revenue_per_staff_day` | ≥ 1000元/人/天 |
| G_TURNOVER | 翻台率提升 | `table_turnover_rate` | 日上座次数 > 2次 |
| G_SCORE | 大众点评评分 | `review_score` | ≥ 4.7 |

### 盈利模型阈值（硬编码为系统预警基准）

```
利润率 > 15%        → 低于此值：净利润率指标标红
租售比 < 15%        → 超过此值：空间费用率指标标红
人工成本率 < 25%    → 超过此值：人员费用率指标标红
坪效 > 100元/平米   → 低于此值：坪效指标预警
人效 > 1000元/人/天 → 低于此值：人效指标预警
日上座次数 > 2次    → 低于此值：翻台率指标预警
投资回收周期 < 租期2/3 → 盈利模型沙盘中标注
```

### OGSM 目标体系 → 功能优先级依据

| OGSM 目标 | 对应功能 | 交付阶段 |
|---|---|---|
| G1：3C指标体系建立 | OGSM 一张表 + 经营总览看板 + 收入/成本分析 | MVP（P1） |
| G2：单店盈利改善 | 四步法工作台 + 成本费用分析 | MVP（P1） |
| G3：综合费用率下降 | 成本费用分析（食材成本率/损耗率） | MVP（P1） |
| G4：会员复购率提升 | 顾客资产分析 | P2 |
| G5：团队分析能力 | 四步法工作台（引导式，降低使用门槛） | P1/P2 |
| G6：盈利模型建立 | 盈利模型沙盘（三档压力测算） | P3 |

---

## 规范文档索引

| 文档 | 内容 | 路径 |
|---|---|---|
| 本宪章 | 核心原则、技术栈、治理规则 | `constitution.md` |
| Skills JSON Schema | 全部 Skill 输入/输出契约 | [skills-schema.md](skills-schema.md) |
| 多租户隔离方案 | 数据隔离、权限矩阵、租户生命周期 | [multi-tenant-isolation.md](multi-tenant-isolation.md) |
| DataSourceAdapter 接口 | 适配器抽象类、内置实现、扩展示例 | [data-source-adapter.md](data-source-adapter.md) |
| 需求分析报告 | 领域知识 → 功能需求全景 + 优先级 | [../../docQA/20260405_BI中台工具需求分析.md](../../docQA/20260405_BI中台工具需求分析.md) |
| MVP 功能规格 | 用户故事 + 验收场景 + 功能需求 | [../../.specify/features/bi-dashboard/spec.md](../features/bi-dashboard/spec.md) |
| MVP 实施计划 | 技术方案 + 任务分解 + 项目结构 | [../../.specify/features/bi-dashboard/plan.md](../features/bi-dashboard/plan.md) |

---

## Governance（治理规则）

1. 本宪章优先级高于所有其他开发规范文档
2. 宪章修订需要：书面说明修改原因 + 影响评估 + 迁移方案
3. 所有 PR Review 必须验证是否符合宪章约束
4. **OGSM 原则（Principle 0）是最高优先级**：任何功能设计若与 OGSM 数据流向冲突，必须重新设计
5. 新增 Skill 必须在本宪章 Skills 清单中登记，并在 [skills-schema.md](skills-schema.md) 补充 JSON Schema
6. 数据接口字段变更必须更新 `schema_version` 并通知所有适配器维护方
7. 新增数据源类型必须实现 [data-source-adapter.md](data-source-adapter.md) 定义的抽象接口并注册到 `registry.py`
8. 新增餐饮企业租户必须遵循 [multi-tenant-isolation.md](multi-tenant-isolation.md) 的创建流程，禁止手动建表
9. G 目标进度计算逻辑变更必须同步更新 `ogsm_progress.py` 的单元测试，覆盖率保持 ≥ 80%
10. 违反"不可妥协"原则的代码，CI 自动阻断，不允许人工覆盖

---

**Version**: 1.2.0 | **Ratified**: 2026-04-05 | **Last Amended**: 2026-04-05
