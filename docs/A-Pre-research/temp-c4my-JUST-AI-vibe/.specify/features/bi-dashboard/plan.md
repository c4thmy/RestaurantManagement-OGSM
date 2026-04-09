# Implementation Plan: 餐饮 BI 中台工具 — OGSM 驱动 MVP

**Branch**: `001-bi-dashboard-mvp` | **Date**: 2026-04-05 | **Spec**: [spec.md](spec.md)  
**Updated**: 2026-04-05 v2.0 — OGSM 为核心重构  
**Input**: `.specify/features/bi-dashboard/spec.md` v2.0

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
**Performance Goals**: OGSM 一张表 P95 < 1s / 看板查询 P95 < 200ms / 四步法 P95 < 500ms  
**Constraints**: 多租户数据严格隔离 / 核心操作 ≤ 3 步 / 首屏 LCP < 2.5s  
**Scale/Scope**: 单租户 500 万行/年 / 50 并发用户 / 最多 20 家门店对比

---

## Constitution Check

| 原则 | 检查结果 | 说明 |
|---|---|---|
| I. 数据接口优先 | ✅ | DataSourceAdapter 接口覆盖 CSV/Excel/API，G 目标进度计算依赖数据质量门禁 |
| II. WebUI 一致性 | ✅ | OGSM 一张表核心操作 ≤ 3 步，四步法工作台有明确引导，三态反馈必须实现 |
| III. Skills 模块化 | ✅ | data-import / data-quality-check / report-generator 覆盖 MVP |
| IV. 测试优先 | ✅ | TDD 强制，业务逻辑 ≥ 80%，数据接口层 ≥ 90% |
| V. 代码质量门禁 | ✅ | black + isort + mypy + bandit / prettier + eslint + tsc |
| VI. 性能基线 | ✅ | OGSM 一张表 < 1s，看板 < 200ms，四步法 < 500ms |
| VII. 安全基线 | ✅ | 多租户隔离 / 参数化查询 / 文件类型白名单 |
| VIII. AI 集成原则 | ✅ | 行动推荐 v1 用规则引擎，AI 摘要附置信度，降级不中断服务 |

---

## Project Structure

### Documentation

```text
.specify/features/bi-dashboard/
├── plan.md              # 本文件
├── spec.md              # 功能规格 v2.0
├── data-model.md        # 数据模型（Phase 1 输出）
├── api-contracts/       # API 接口契约（Phase 1 输出）
│   ├── ogsm.yaml
│   ├── analysis.yaml
│   ├── dashboard.yaml
│   ├── import.yaml
│   └── revenue-cost.yaml
└── tasks.md             # 任务分解（/speckit.tasks 输出）
```

### Source Code

```text
backend/
├── app/
│   ├── main.py
│   ├── api/v1/
│   │   ├── deps.py                  # TenantContext 依赖注入
│   │   ├── ogsm_router.py           # OGSM O/G/S/M CRUD
│   │   ├── analysis_router.py       # 四步法工作台
│   │   ├── dashboard_router.py      # BI 看板
│   │   ├── revenue_router.py        # 收入分析
│   │   ├── cost_router.py           # 成本费用
│   │   ├── store_router.py          # 多店对比
│   │   └── import_router.py         # 数据导入
│   ├── domain/
│   │   ├── models.py                # 领域模型
│   │   ├── ogsm_models.py           # OGSM 实体
│   │   ├── metrics.py               # 指标计算（客单/客流因素、损益率）
│   │   ├── ogsm_progress.py         # G 目标进度自动计算引擎
│   │   ├── analysis_engine.py       # 四步法分析逻辑
│   │   ├── action_recommender.py    # 行动推荐规则引擎
│   │   └── validators.py            # 业务规则校验
│   ├── application/
│   │   ├── ogsm_service.py          # OGSM 应用服务
│   │   ├── analysis_service.py      # 四步法分析服务
│   │   ├── dashboard_service.py     # 看板数据聚合
│   │   ├── revenue_service.py       # 收入分析
│   │   ├── cost_service.py          # 成本分析
│   │   └── import_service.py        # 数据导入
│   ├── infrastructure/
│   │   ├── database.py              # 多租户 Schema 路由
│   │   ├── repositories/
│   │   │   ├── ogsm_repo.py
│   │   │   ├── sales_repo.py
│   │   │   ├── cost_repo.py
│   │   │   └── import_repo.py
│   │   └── adapters/
│   │       ├── base.py
│   │       ├── csv_adapter.py
│   │       ├── excel_adapter.py
│   │       └── registry.py
│   ├── skills/
│   │   ├── data_import_skill.py
│   │   ├── data_quality_skill.py
│   │   └── report_generator_skill.py
│   └── workers/
│       ├── celery_app.py
│       ├── import_tasks.py
│       └── ogsm_refresh_tasks.py    # G 目标进度定时刷新
├── alembic/
├── tests/
│   ├── unit/
│   │   ├── domain/
│   │   │   ├── test_ogsm_progress.py   # G 进度计算
│   │   │   ├── test_metrics.py          # 客单/客流因素
│   │   │   ├── test_analysis_engine.py  # 四步法逻辑
│   │   │   └── test_action_recommender.py
│   │   ├── skills/
│   │   └── adapters/
│   ├── integration/
│   │   ├── test_ogsm_flow.py        # OGSM 完整链路
│   │   ├── test_analysis_flow.py    # 四步法完整链路
│   │   ├── test_import_flow.py
│   │   └── test_dashboard_api.py
│   └── fixtures/
└── pyproject.toml

frontend/
├── src/
│   ├── stores/
│   │   ├── auth.ts
│   │   ├── ogsm.ts                  # OGSM 状态
│   │   ├── analysis.ts              # 四步法会话状态
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
│   │   │   ├── GoalProgressBar.vue  # G 目标进度条（含预警）
│   │   │   ├── ActionItem.vue       # M 行动清单项
│   │   │   └── StrategyCard.vue     # S 策略卡片
│   │   ├── analysis/
│   │   │   ├── AnalysisWizard.vue   # 四步法向导容器
│   │   │   ├── Step1Compare.vue     # 一比目标
│   │   │   ├── Step2Breakdown.vue   # 二拆数据
│   │   │   ├── Step3Attribution.vue # 三定归因
│   │   │   └── Step4Decision.vue    # 四做决策
│   │   ├── common/
│   │   │   ├── MetricCard.vue       # 指标卡片（含 G 目标对比）
│   │   │   ├── WaterfallChart.vue   # 瀑布图
│   │   │   ├── TrendChart.vue
│   │   │   ├── BarChart.vue
│   │   │   ├── LoadingState.vue
│   │   │   └── FilterBar.vue
│   │   └── import/
│   │       ├── FileUploader.vue
│   │       ├── FieldMapper.vue
│   │       ├── DataPreview.vue
│   │       └── QualityReport.vue
│   └── pages/
│       ├── OgsmPage.vue             # 主界面：OGSM 一张表
│       ├── AnalysisPage.vue         # 四步法工作台
│       ├── DashboardPage.vue        # BI 看板
│       ├── RevenuePage.vue
│       ├── CostPage.vue
│       ├── StoreComparePage.vue
│       └── ImportPage.vue
└── package.json
```

---

## Task 分解（10 个 Task，TDD 强制）

### Task 1：项目脚手架与基础设施

**前置条件**：无  
**验收**：`pytest` 和 `npm run test -- --run` 均可运行（0 测试 0 失败）；`GET /health` 返回 200

```
1a. 初始化 FastAPI 项目，配置 pyproject.toml（black/isort/mypy/bandit/pytest）
1b. 初始化 Vue 3 + TypeScript，配置 eslint/prettier/vitest/playwright
1c. 编写 Docker Compose（postgres + redis + backend + frontend）
1d. 配置 Alembic 多租户 Schema 迁移
1e. 实现 /health 端点和基础错误处理中间件
```

---

### Task 2：多租户基础设施（TDD）

**前置条件**：Task 1  
**验收**：`pytest tests/unit/test_tenant.py` 全部通过；跨租户查询被拦截

```
2a. 先写测试：test_tenant_schema_creation / test_cross_tenant_isolation / test_tenant_context_injection
2b. 实现 TenantContext 数据类和 FastAPI 依赖注入
2c. 实现 Alembic 多租户迁移（create_tenant_schema / run_migrations）
2d. 实现 Repository 基类（自动注入 schema 前缀）
2e. 运行测试确认通过（绿灯）
```

---

### Task 3：OGSM 核心领域层（TDD）

**前置条件**：Task 2  
**验收**：`pytest tests/unit/domain/test_ogsm_progress.py` 全部通过；G 进度计算公式正确

```
3a. 先写测试：
    - test_goal_progress_calculation（实际值/目标值 → 完成率）
    - test_goal_alert_threshold（完成率 < 80% → 标红）
    - test_strategy_execution_progress（M 完成率 → S 执行进度）
    - test_action_overdue_detection（截止日期过后未完成 → 逾期）
    - test_goal_progress_auto_refresh（数据导入后触发 G 进度重算）
3b. 实现 ogsm_models.py（OgsmObjective/Goal/Strategy/Action 数据类）
3c. 实现 ogsm_progress.py（G 目标进度计算引擎，纯函数）
3d. 实现 ogsm_repo.py（OGSM 数据持久化）
3e. 运行测试确认通过（绿灯）
```

---

### Task 4：OGSM API + 前端主界面（TDD）

**前置条件**：Task 3  
**验收**：`pytest tests/integration/test_ogsm_flow.py` 通过；OGSM 一张表加载 P95 < 1s；G 进度条自动计算

```
4a. 先写集成测试：
    - test_ogsm_crud（O/G/S/M 增删改查）
    - test_goal_progress_auto_update（导入数据后 G 进度刷新）
    - test_action_complete_triggers_goal_refresh（M 完成 → G 重算）
4b. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/objectives
4c. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/goals（含进度自动计算）
4d. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/strategies
4e. 实现 GET/POST/PUT/DELETE /api/v1/ogsm/actions（含逾期检测）
4f. 实现 GET /api/v1/ogsm/summary（OGSM 一张表汇总数据）
4g. 前端：OgsmTable.vue（OGSM 一张表主组件）
4h. 前端：GoalProgressBar.vue（进度条 + 预警标红）
4i. 前端：ActionItem.vue（行动清单项 + 状态流转）
4j. 前端：OgsmPage.vue（主界面，首页路由）
4k. 先写前端单元测试，再实现组件
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
    - test_store_composite_score（多店综合评分）
5b. 实现 metrics.py（所有指标计算，纯函数，无副作用）
5c. 实现 validators.py（阈值检查）
5d. 运行测试确认通过（绿灯）
```

---

### Task 6：四步法分析工作台（TDD）

**前置条件**：Task 3 + Task 5  
**验收**：`pytest tests/integration/test_analysis_flow.py` 通过；Playwright E2E 测试"4 步完整流程 → 生成 M 行动"通过

```
6a. 先写测试：
    - test_step1_loads_goal_vs_actual（Step 1 自动加载 G 目标对比）
    - test_step2_channel_breakdown（渠道拆解）
    - test_step3_factor_decomposition（客单/客流因素计算）
    - test_step4_generates_action（生成 M 行动并关联 OGSM）
    - test_session_draft_save（中途退出保存草稿）
6b. 实现 analysis_engine.py（四步法核心逻辑）
6c. 实现 action_recommender.py（规则引擎：归因类型 → 推荐行动模板）
6d. 实现 POST /api/v1/analysis/sessions（创建分析会话）
6e. 实现 PUT /api/v1/analysis/sessions/{id}/step/{n}（更新步骤数据）
6f. 实现 POST /api/v1/analysis/sessions/{id}/generate-actions（生成 M 行动）
6g. 前端：AnalysisWizard.vue（4 步向导容器）
6h. 前端：Step1Compare.vue / Step2Breakdown.vue / Step3Attribution.vue / Step4Decision.vue
6i. 前端：WaterfallChart.vue（ECharts 瀑布图，展示因素拆解）
6j. 编写 Playwright E2E 测试
```

---

### Task 7：数据导入 + DataSourceAdapter（TDD）

**前置条件**：Task 2  
**验收**：`pytest tests/unit/adapters/` 和 `pytest tests/integration/test_import_flow.py` 通过；覆盖率 ≥ 90%；导入成功后 G 进度自动刷新

```
7a. 先写测试（11 个场景，见 data-source-adapter.md）
7b. 实现 DataSourceAdapter 抽象基类
7c. 实现 CsvFileAdapter / ExcelFileAdapter / ApiPushAdapter
7d. 实现 data-import Skill / data-quality-check Skill
7e. 实现 POST /api/v1/import/upload（上传 + 质量检查 + 预览）
7f. 实现 POST /api/v1/import/confirm（确认写入，异步任务）
7g. 实现导入完成后触发 G 目标进度刷新（Celery 任务链）
7h. 前端：ImportPage.vue（3 步向导：上传→预览→确认）
7i. 前端：QualityReport.vue（质量检查报告）
```

---

### Task 8：BI 看板 + 收入/成本分析（TDD）

**前置条件**：Task 5 + Task 7  
**验收**：`pytest tests/integration/test_dashboard_api.py` 通过；指标卡片显示实际值/G 目标值/差距；超阈值自动标红

```
8a. 先写集成测试：
    - test_dashboard_shows_goal_vs_actual（指标卡片含 G 目标对比）
    - test_threshold_alerts（超阈值标红）
    - test_revenue_factor_decomposition_api（收入因素拆解 API）
    - test_cost_three_line_comparison（三线成本对比）
8b. 实现 GET /api/v1/dashboard/overview（8 个指标 + G 目标对比 + 预警）
8c. 实现 GET /api/v1/revenue/breakdown（渠道/时段拆解）
8d. 实现 GET /api/v1/revenue/factor-decomposition（客单/客流因素）
8e. 实现 GET /api/v1/cost/overview（三线成本 + 损益率）
8f. 实现 GET /api/v1/cost/expenses（费用率结构）
8g. 实现 Redis 缓存层（5 分钟 TTL）
8h. 前端：MetricCard.vue（含 G 目标对比 + 跳转四步法按钮 + 创建行动按钮）
8i. 前端：DashboardPage.vue / RevenuePage.vue / CostPage.vue
```

---

### Task 9：多店对比（TDD）

**前置条件**：Task 5  
**验收**：`pytest tests/integration/test_store_compare_api.py` 通过；20 家门店对比 P95 < 3s；标杆店自动识别

```
9a. 先写集成测试：test_store_ranking / test_benchmark_identification / test_generate_action_from_compare
9b. 实现 GET /api/v1/stores/compare（多店横向对比 + 综合评分）
9c. 实现 GET /api/v1/stores/benchmark（标杆店/问题店识别）
9d. 实现 POST /api/v1/stores/compare/generate-action（从对比结果生成 M 行动）
9e. 前端：StoreComparePage.vue（排名表格 + 雷达图 + 生成行动按钮）
```

---

### Task 10：发布前质量门禁

**前置条件**：Task 4-9 全部完成  
**验收**：所有自动检查通过，手动验收清单完成

```
10a. pytest --cov=app（覆盖率 ≥ 80%）
10b. npm run test -- --run（全部通过）
10c. Playwright E2E：关键路径全部通过
     - OGSM 一张表：录入 G → 导入数据 → 进度自动更新
     - 四步法：4 步完整流程 → 生成 M 行动 → 出现在 OGSM 表
     - 数据导入：上传→预览→确认→G 进度刷新
10d. black --check . && isort --check . && mypy app/ && bandit -r app/
10e. npm run lint && tsc --noEmit
10f. k6 压测：50 并发，OGSM 一张表 P95 < 1s，看板 P95 < 200ms
10g. 手动验收：对照 spec.md 所有 Acceptance Scenarios 逐一验证
```

---

## Complexity Tracking

| 决策 | 理由 | 更简单方案被拒绝的原因 |
|---|---|---|
| G 目标进度自动计算 | OGSM 核心价值：不需要手工填写进度 | 手工填写：用户负担重，数据不准确，失去自动化价值 |
| 四步法分析会话持久化 | 支持中途退出继续，支持团队复盘 | 无状态：用户每次重新开始，体验差 |
| 行动推荐规则引擎（非 AI） | v1 快速交付，规则可解释，不依赖外部 API | 纯 AI 推荐：延迟高，成本高，v1 不稳定 |
| PostgreSQL 独立 Schema | 数据隔离是核心安全要求 | 共享表：RLS 配置复杂，遗漏过滤会导致数据泄露 |
