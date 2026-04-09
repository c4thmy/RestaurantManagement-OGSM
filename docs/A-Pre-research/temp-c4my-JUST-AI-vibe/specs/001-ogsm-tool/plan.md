# Implementation Plan: OGSM 落地工具

**Branch**: `001-ogsm-tool` | **Date**: 2026-04-06 | **Spec**: [spec.md](spec.md)  
**Input**: `specs/001-ogsm-tool/spec.md`

---

## Summary

软件即 OGSM 落地工具。OGSM 一张表是主界面，G 目标进度由 BI 数据自动驱动，四步法工作台是执行引擎，M 行动清单是闭环机制。技术路线：FastAPI 后端 + Vue 3 前端 + PostgreSQL 多租户 Schema 隔离 + Celery 异步任务。

---

## Technical Context

**Language/Version**: Python 3.11 / TypeScript 5.x + Vue 3  
**Primary Dependencies**: FastAPI 0.111 / SQLAlchemy 2.x / Alembic / Celery + Redis / Vue 3 + Element Plus + ECharts / Pinia  
**Storage**: PostgreSQL 16（多租户 Schema 隔离）+ Redis（缓存 + 任务队列）  
**Testing**: pytest + pytest-asyncio / Vitest + Playwright  
**Target Platform**: Linux 服务器（Docker）/ 桌面浏览器（1024px+）  
**Project Type**: Web application（前后端分离）  
**Performance Goals**: OGSM 一张表 P95 < 1s / 看板查询 P95 < 200ms / 四步法每步 P95 < 500ms  
**Constraints**: 多租户数据严格隔离 / 核心操作 ≤ 3 步 / 首屏 LCP < 2.5s / 文件上传 ≤ 5MB  
**Scale/Scope**: 单租户 MVP / 50 并发用户 / 最多 20 家门店对比

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原则 | 检查结果 | 说明 |
|---|---|---|
| 0. OGSM 驱动设计 | ✅ | OGSM 一张表为主界面（FR-001），G 进度自动计算（FR-002），四步法直接生成 M 行动（FR-009/FR-024），M 完成触发 G 重算（FR-012） |
| I. 数据接口优先 | ✅ | DataSourceAdapter 接口覆盖 Excel/CSV，数据质量检查在接口层强制执行（FR-026） |
| II. WebUI 一致性 | ✅ | 核心操作 ≤ 3 步（SC-002/SC-004），三态反馈必须实现，四步法有明确引导（SC-007） |
| III. Skills 模块化 | ✅ | data-import / data-quality-check Skill 覆盖 MVP，report-generator 预留 |
| IV. 测试优先 | ✅ | TDD 强制，业务逻辑 ≥ 80%，数据接口层 ≥ 90% |
| V. 代码质量门禁 | ✅ | black + isort + mypy + bandit / prettier + eslint + tsc |
| VI. 性能基线 | ✅ | OGSM 一张表 < 1s（SC-001），看板 < 2.5s（SC-006），四步法 < 500ms（SC-008） |
| VII. 安全基线 | ✅ | 角色权限控制（FR-028/FR-029），参数化查询，文件类型白名单 |
| VIII. AI 集成原则 | ✅ | 行动推荐 v1 用规则引擎，降级不中断服务 |

---

## Project Structure

### Documentation (this feature)

```text
specs/001-ogsm-tool/
├── plan.md              # 本文件
├── spec.md              # 功能规格
├── research.md          # Phase 0 输出
├── data-model.md        # Phase 1 输出
├── contracts/           # Phase 1 输出
│   ├── ogsm.yaml
│   ├── analysis.yaml
│   ├── dashboard.yaml
│   └── import.yaml
├── checklists/
│   └── requirements.md
└── tasks.md             # /speckit.tasks 输出
```

### Source Code (repository root)

```text
backend/
├── app/
│   ├── main.py
│   ├── api/v1/
│   │   ├── deps.py                  # TenantContext 依赖注入 + 角色权限
│   │   ├── ogsm_router.py           # OGSM O/G/S/M CRUD
│   │   ├── analysis_router.py       # 四步法工作台
│   │   ├── dashboard_router.py      # BI 看板
│   │   ├── revenue_router.py        # 收入分析
│   │   ├── cost_router.py           # 成本费用
│   │   └── import_router.py         # 数据导入
│   ├── domain/
│   │   ├── ogsm_models.py           # OGSM 实体（O/G/S/M/Review/AnalysisSession）
│   │   ├── ogsm_progress.py         # G 目标进度自动计算引擎（纯函数）
│   │   ├── metrics.py               # 指标计算（客单/客流因素、损益率）
│   │   ├── analysis_engine.py       # 四步法分析逻辑
│   │   ├── action_recommender.py    # 行动推荐规则引擎
│   │   └── validators.py            # 业务规则校验 + 阈值检查
│   ├── application/
│   │   ├── ogsm_service.py
│   │   ├── analysis_service.py
│   │   ├── dashboard_service.py
│   │   ├── revenue_service.py
│   │   ├── cost_service.py
│   │   └── import_service.py
│   ├── infrastructure/
│   │   ├── database.py              # 多租户 Schema 路由
│   │   ├── repositories/
│   │   │   ├── ogsm_repo.py
│   │   │   ├── sales_repo.py
│   │   │   ├── cost_repo.py
│   │   │   └── import_repo.py
│   │   └── adapters/
│   │       ├── base.py              # DataSourceAdapter 抽象基类
│   │       ├── excel_adapter.py
│   │       ├── csv_adapter.py
│   │       └── registry.py
│   ├── skills/
│   │   ├── data_import_skill.py
│   │   └── data_quality_skill.py
│   └── workers/
│       ├── celery_app.py
│       ├── import_tasks.py
│       └── ogsm_refresh_tasks.py    # G 目标进度定时刷新
├── alembic/
├── tests/
│   ├── unit/
│   │   ├── domain/
│   │   │   ├── test_ogsm_progress.py
│   │   │   ├── test_metrics.py
│   │   │   ├── test_analysis_engine.py
│   │   │   └── test_action_recommender.py
│   │   ├── skills/
│   │   └── adapters/
│   ├── integration/
│   │   ├── test_ogsm_flow.py
│   │   ├── test_analysis_flow.py
│   │   ├── test_import_flow.py
│   │   └── test_dashboard_api.py
│   └── fixtures/
└── pyproject.toml

frontend/
├── src/
│   ├── stores/
│   │   ├── auth.ts
│   │   ├── ogsm.ts
│   │   ├── analysis.ts
│   │   └── dashboard.ts
│   ├── api/
│   │   ├── client.ts
│   │   ├── ogsm.ts
│   │   ├── analysis.ts
│   │   ├── dashboard.ts
│   │   ├── revenue.ts
│   │   ├── cost.ts
│   │   └── import.ts
│   ├── components/
│   │   ├── ogsm/
│   │   │   ├── OgsmTable.vue        # OGSM 一张表主组件
│   │   │   ├── GoalProgressBar.vue  # G 目标进度条（含预警标红）
│   │   │   ├── ActionItem.vue       # M 行动清单项
│   │   │   └── StrategyCard.vue     # S 策略卡片
│   │   ├── analysis/
│   │   │   ├── AnalysisWizard.vue   # 四步法向导容器
│   │   │   ├── Step1Compare.vue
│   │   │   ├── Step2Breakdown.vue
│   │   │   ├── Step3Attribution.vue
│   │   │   └── Step4Decision.vue
│   │   ├── common/
│   │   │   ├── MetricCard.vue       # 指标卡片（实际值/G目标/差距/跳转按钮）
│   │   │   ├── WaterfallChart.vue   # ECharts 瀑布图
│   │   │   ├── TrendChart.vue
│   │   │   ├── LoadingState.vue     # Loading/Empty/Error 三态
│   │   │   └── FilterBar.vue
│   │   └── import/
│   │       ├── FileUploader.vue
│   │       ├── DataPreview.vue
│   │       └── QualityReport.vue
│   └── pages/
│       ├── OgsmPage.vue             # 主界面（首页路由）
│       ├── AnalysisPage.vue
│       ├── DashboardPage.vue
│       ├── RevenuePage.vue
│       ├── CostPage.vue
│       └── ImportPage.vue
└── package.json
```

**Structure Decision**: Web application 前后端分离，后端 FastAPI 四层架构（presentation → application → domain → infrastructure），前端 Vue 3 页面 + 组件分层。与 constitution 技术栈约定完全一致。

---

## Task 分解（9 个 Task，TDD 强制）

### Task 1：项目脚手架与基础设施

**前置条件**：无  
**验收**：`pytest` 和 `npm run test -- --run` 均可运行（0 测试 0 失败）；`GET /health` 返回 200

```
1a. 初始化 FastAPI 项目，配置 pyproject.toml（black/isort/mypy/bandit/pytest）
1b. 初始化 Vue 3 + TypeScript，配置 eslint/prettier/vitest/playwright
1c. 编写 Docker Compose（postgres + redis + backend + frontend）
1d. 配置 Alembic 多租户 Schema 迁移基础
1e. 实现 /health 端点和基础错误处理中间件
```

---

### Task 2：多租户基础设施 + 角色权限（TDD）

**前置条件**：Task 1  
**验收**：`pytest tests/unit/test_tenant.py` 全部通过；跨租户查询被拦截；角色权限正确隔离

```
2a. 先写测试：test_tenant_schema_creation / test_cross_tenant_isolation / test_role_permission_matrix
2b. 实现 TenantContext 数据类和 FastAPI 依赖注入
2c. 实现 Alembic 多租户迁移（create_tenant_schema）
2d. 实现 Repository 基类（自动注入 schema 前缀）
2e. 实现角色权限中间件（高管/运营总监/财务部/店长/营销部）
2f. 运行测试确认通过（绿灯）
```

---

### Task 3：OGSM 核心领域层（TDD）

**前置条件**：Task 2  
**验收**：`pytest tests/unit/domain/test_ogsm_progress.py` 全部通过；G 进度计算公式正确；逾期检测正确

```
3a. 先写测试：
    - test_goal_progress_calculation（实际值/目标值 → 完成率）
    - test_goal_alert_threshold（完成率 < 80% → 标红）
    - test_strategy_execution_progress（M 完成率 → S 执行进度）
    - test_action_overdue_detection（截止日期过后未完成 → 逾期）
    - test_goal_manual_override（手工覆盖 current_value + 标注标识）
    - test_goal_progress_auto_refresh（数据导入后触发 G 进度重算）
3b. 实现 ogsm_models.py（Objective/Goal/Strategy/Action/AnalysisSession/OgsmReview）
3c. 实现 ogsm_progress.py（G 目标进度计算引擎，纯函数）
3d. 实现 ogsm_repo.py（OGSM 数据持久化）
3e. 运行测试确认通过（绿灯）
```

---

### Task 4：OGSM API + 前端主界面（TDD）

**前置条件**：Task 3  
**验收**：`pytest tests/integration/test_ogsm_flow.py` 通过；OGSM 一张表加载 P95 < 1s；G 进度条自动计算；店长只看自己的 M 行动

```
4a. 先写集成测试：
    - test_ogsm_crud（O/G/S/M 增删改查）
    - test_goal_progress_auto_update（导入数据后 G 进度刷新）
    - test_action_complete_triggers_goal_refresh（M 完成 → G 重算）
    - test_role_visibility（店长只看自己的 M 行动）
4b. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/objectives
4c. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/goals（含进度自动计算 + 手工覆盖）
4d. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/strategies
4e. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/actions（含逾期检测 + 角色过滤）
4f. 实现 GET /api/v1/ogsm/summary（OGSM 一张表汇总数据）
4g. 前端：OgsmTable.vue / GoalProgressBar.vue / ActionItem.vue / StrategyCard.vue
4h. 前端：OgsmPage.vue（主界面，首页路由）
4i. 先写前端单元测试，再实现组件
```

---

### Task 5：指标计算领域层（TDD）

**前置条件**：Task 2（可与 Task 3/4 并行）  
**验收**：`pytest tests/unit/domain/test_metrics.py` 全部通过；所有公式与方法论一致

```
5a. 先写测试：
    - test_revenue_factor_decomposition（客单因素 + 客流因素公式）
    - test_cost_variance_rate（原材料损益率）
    - test_profit_model_thresholds（盈利模型阈值检查）
5b. 实现 metrics.py（所有指标计算，纯函数，无副作用）
5c. 实现 validators.py（阈值检查：净利润率/租售比/人工/坪效/人效）
5d. 运行测试确认通过（绿灯）
```

---

### Task 6：数据导入 + DataSourceAdapter（TDD）

**前置条件**：Task 2  
**验收**：`pytest tests/unit/adapters/` 和 `pytest tests/integration/test_import_flow.py` 通过；覆盖率 ≥ 90%；导入成功后 G 进度自动刷新

```
6a. 先写测试（含错误格式、超大文件、字段缺失等边界场景）
6b. 实现 DataSourceAdapter 抽象基类
6c. 实现 ExcelFileAdapter / CsvFileAdapter
6d. 实现 data-import Skill / data-quality-check Skill
6e. 实现 POST /api/v1/import/upload（上传 + 质量检查 + 预览）
6f. 实现 POST /api/v1/import/confirm（确认写入，Celery 异步任务）
6g. 实现导入完成后触发 G 目标进度刷新（Celery 任务链）
6h. 前端：ImportPage.vue（3 步向导：上传→预览→确认）
6i. 前端：QualityReport.vue（质量检查报告，显示错误行号和原因）
```

---

### Task 7：BI 看板 + 收入/成本分析（TDD）

**前置条件**：Task 5 + Task 6  
**验收**：`pytest tests/integration/test_dashboard_api.py` 通过；8 个指标卡片显示实际值/G 目标值/差距；超阈值自动标红；首屏 LCP < 2.5s

```
7a. 先写集成测试：
    - test_dashboard_shows_goal_vs_actual（指标卡片含 G 目标对比）
    - test_threshold_alerts（超阈值标红）
    - test_revenue_factor_decomposition_api
    - test_cost_three_line_comparison
7b. 实现 GET /api/v1/dashboard/overview（8 个指标 + G 目标对比 + 预警）
7c. 实现 GET /api/v1/revenue/breakdown（渠道/时段拆解）
7d. 实现 GET /api/v1/revenue/factor-decomposition（客单/客流因素）
7e. 实现 GET /api/v1/cost/overview（三线成本 + 损益率）
7f. 实现 GET /api/v1/cost/expenses（费用率结构）
7g. 实现 Redis 缓存层（5 分钟 TTL，导入后主动失效）
7h. 前端：MetricCard.vue（含 G 目标对比 + "查看分析"/"创建行动"按钮）
7i. 前端：DashboardPage.vue / RevenuePage.vue / CostPage.vue
```

---

### Task 8：四步法分析工作台（TDD）

**前置条件**：Task 3 + Task 5  
**验收**：`pytest tests/integration/test_analysis_flow.py` 通过；Playwright E2E "4 步完整流程 → 生成 M 行动 → 出现在 OGSM 表" 通过

```
8a. 先写测试：
    - test_step1_loads_goal_vs_actual
    - test_step2_channel_breakdown（渠道拆解 + 主要缺口标注）
    - test_step3_factor_decomposition（客单/客流因素计算）
    - test_step4_generates_action（生成 M 行动并关联 OGSM）
    - test_session_draft_save（中途退出保存草稿）
    - test_three_trigger_entries（三种触发入口均可进入工作台）
8b. 实现 analysis_engine.py（四步法核心逻辑）
8c. 实现 action_recommender.py（规则引擎：归因类型 → 推荐行动模板）
8d. 实现 POST /api/v1/analysis/sessions
8e. 实现 PUT /api/v1/analysis/sessions/{id}/step/{n}
8f. 实现 POST /api/v1/analysis/sessions/{id}/generate-actions
8g. 前端：AnalysisWizard.vue（4 步向导容器）
8h. 前端：Step1Compare.vue / Step2Breakdown.vue / Step3Attribution.vue / Step4Decision.vue
8i. 前端：WaterfallChart.vue（ECharts 瀑布图，展示因素拆解）
8j. 编写 Playwright E2E 测试
```

---

### Task 9：发布前质量门禁

**前置条件**：Task 4-8 全部完成  
**验收**：所有自动检查通过，手动验收清单完成

```
9a. pytest --cov=app（覆盖率 ≥ 80%）
9b. npm run test -- --run（全部通过）
9c. Playwright E2E 关键路径：
    - OGSM 一张表：录入 G → 导入数据 → 进度自动更新
    - 四步法：4 步完整流程 → 生成 M 行动 → 出现在 OGSM 表
    - 数据导入：上传→预览→确认→G 进度刷新
    - 角色隔离：店长只看自己的 M 行动
9d. black --check . && isort --check . && mypy app/ && bandit -r app/
9e. npm run lint && tsc --noEmit
9f. k6 压测：50 并发，OGSM 一张表 P95 < 1s，看板 P95 < 200ms
9g. 手动验收：对照 spec.md 所有 Acceptance Scenarios 逐一验证
```

---

## Complexity Tracking

| 决策 | 理由 | 更简单方案被拒绝的原因 |
|---|---|---|
| G 目标进度自动计算 | OGSM 核心价值：不需要手工填写进度 | 手工填写：用户负担重，数据不准确 |
| 四步法分析会话持久化 | 支持中途退出继续，支持草稿保存 | 无状态：用户每次重新开始，体验差 |
| 行动推荐规则引擎（非 AI） | v1 快速交付，规则可解释，不依赖外部 API | 纯 AI 推荐：延迟高，v1 不稳定 |
| 多层级 OGSM 各自独立 | 用户选择方案 A：各层独立维护，管理简单 | 自动汇总：数据模型复杂，MVP 阶段过度设计 |
