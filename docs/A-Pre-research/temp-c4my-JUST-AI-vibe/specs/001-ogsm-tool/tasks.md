# Tasks: OGSM 落地工具

**Input**: `specs/001-ogsm-tool/` (plan.md + spec.md + data-model.md + contracts/)  
**Branch**: `001-ogsm-tool`  
**Tech Stack**: Python 3.11 + FastAPI / Vue 3 + TypeScript / PostgreSQL + Redis  
**TDD**: 强制 — 测试先写，确认失败后再实现

---

## Phase 1: Setup（项目脚手架）

**Purpose**: 初始化前后端项目结构，确保测试框架可运行

- [ ] T001 初始化 FastAPI 项目，创建 `backend/` 目录结构（app/api/domain/application/infrastructure/skills/workers/tests/）
- [ ] T002 [P] 配置 `backend/pyproject.toml`（black/isort/mypy/bandit/pytest/pytest-asyncio/pytest-cov）
- [ ] T003 [P] 初始化 Vue 3 + TypeScript 项目，创建 `frontend/` 目录结构（src/stores/api/components/pages/）
- [ ] T004 [P] 配置 `frontend/package.json`（eslint/prettier/vitest/playwright/element-plus/echarts/pinia）
- [ ] T005 编写 `docker-compose.yml`（postgres:16 + redis:7 + backend + frontend）
- [ ] T006 [P] 配置 Alembic 迁移框架，创建 `backend/alembic/` 基础结构
- [ ] T007 实现 `backend/app/main.py`（FastAPI 实例 + `/health` 端点 + 错误处理中间件）

**Checkpoint**: `pytest` 和 `npm run test -- --run` 均可运行（0 测试 0 失败）；`GET /health` 返回 200

---

## Phase 2: Foundational（阻塞性基础设施）

**Purpose**: 多租户隔离 + 角色权限，所有用户故事的前提

**⚠️ CRITICAL**: 此阶段完成前，任何用户故事均不可开始

- [ ] T008 [P] [测试先写] 编写 `backend/tests/unit/test_tenant.py`（test_tenant_schema_creation / test_cross_tenant_isolation / test_role_permission_matrix）— 确认失败
- [ ] T009 [P] [测试先写] 编写 `backend/tests/unit/test_auth.py`（test_jwt_tenant_extraction / test_role_visibility_staff）— 确认失败
- [ ] T010 实现 `backend/app/infrastructure/database.py`（多租户 Schema 路由，`tenant_{tenant_id}` 命名）
- [ ] T011 实现 `backend/app/api/v1/deps.py`（TenantContext 数据类 + FastAPI JWT 依赖注入）
- [ ] T012 [P] 实现 Alembic 多租户迁移脚本（`create_tenant_schema` / `run_migrations`）
- [ ] T013 实现 `backend/app/infrastructure/repositories/` 基类（自动注入 schema 前缀）
- [ ] T014 运行 T008/T009 测试，确认全部通过（绿灯）

**Checkpoint**: 跨租户查询被拦截；店长角色只能看到自己的数据

---

## Phase 3: US6 — 数据导入（Priority: P1）

**Goal**: 支持 Excel/CSV 上传、质量检查、确认写入，是所有 BI 数据的基础

**Independent Test**: 上传标准格式 Excel，验证导入后 BI 看板指标数值正确更新

### 测试（TDD — 先写后实现）

- [ ] T015 [P] [US6] 编写 `backend/tests/unit/adapters/test_csv_adapter.py`（11 个场景：正常/缺字段/日期格式错/金额非数字/编码错/空文件/field_mapping/ext字段/stream_parse边界/幂等性）— 确认失败
- [ ] T016 [P] [US6] 编写 `backend/tests/unit/adapters/test_excel_adapter.py`（同上 + 空sheet/多sheet）— 确认失败
- [ ] T017 [P] [US6] 编写 `backend/tests/integration/test_import_flow.py`（上传→质量检查→确认写入→G进度刷新完整链路）— 确认失败

### 实现

- [ ] T018 [US6] 实现 `backend/app/infrastructure/adapters/base.py`（DataSourceAdapter 抽象基类 + AdapterConfig/ParsedRecord/AdapterResult）
- [ ] T019 [P] [US6] 实现 `backend/app/infrastructure/adapters/csv_adapter.py`（CsvFileAdapter）
- [ ] T020 [P] [US6] 实现 `backend/app/infrastructure/adapters/excel_adapter.py`（ExcelFileAdapter，依赖 openpyxl）
- [ ] T021 [US6] 实现 `backend/app/infrastructure/adapters/registry.py`（工厂函数 get_adapter）
- [ ] T022 [US6] 实现 `backend/app/skills/data_quality_skill.py`（质量评分 + 问题列表）
- [ ] T023 [US6] 实现 `backend/app/skills/data_import_skill.py`（调用 Adapter + 写入 DB）
- [ ] T024 [US6] 实现 `backend/app/workers/import_tasks.py`（Celery 异步导入任务）
- [ ] T025 [US6] 实现 `backend/app/api/v1/import_router.py`（POST /upload + POST /confirm + GET /tasks/{id}，契约见 contracts/import.yaml）
- [ ] T026 [US6] 运行 T015-T017 测试，确认全部通过（覆盖率 ≥ 90%）
- [ ] T027 [P] [US6] 实现前端 `frontend/src/components/import/FileUploader.vue`（文件选择 + 大小校验 ≤5MB）
- [ ] T028 [P] [US6] 实现前端 `frontend/src/components/import/QualityReport.vue`（错误行号 + 原因展示）
- [ ] T029 [P] [US6] 实现前端 `frontend/src/components/import/DataPreview.vue`（前5行预览）
- [ ] T030 [US6] 实现前端 `frontend/src/pages/ImportPage.vue`（3步向导：上传→预览→确认）
- [ ] T031 [US6] 实现前端 `frontend/src/api/import.ts`（调用 import API）

**Checkpoint**: 上传 Excel → 质量检查报告 → 确认写入 → 任务状态轮询，全链路可独立验证

---

## Phase 4: US2 — G 目标管理（Priority: P1）

**Goal**: 财务部录入 G 目标，系统自动关联 BI 指标，进度自动计算

**Independent Test**: 录入一个 G 目标后，在主界面验证进度条出现并显示正确目标值

### 测试（TDD）

- [ ] T032 [P] [US2] 编写 `backend/tests/unit/domain/test_ogsm_progress.py`（test_goal_progress_calculation / test_goal_alert_threshold / test_goal_manual_override / test_goal_no_data_shows_none / test_target_value_zero_raises）— 确认失败
- [ ] T033 [P] [US2] 编写 `backend/tests/unit/domain/test_ogsm_models.py`（test_strategy_execution_progress / test_action_overdue_detection）— 确认失败
- [ ] T034 [US2] 编写 `backend/tests/integration/test_ogsm_flow.py`（test_goal_crud / test_goal_progress_auto_update / test_goal_manual_override_flag）— 确认失败

### 实现

- [ ] T035 [US2] 实现 `backend/app/domain/ogsm_models.py`（OgsmObjective/OgsmGoal/OgsmStrategy/OgsmAction/AnalysisSession/OgsmReview 数据类，见 data-model.md）
- [ ] T036 [US2] 实现 `backend/app/domain/ogsm_progress.py`（calculate_progress / is_alert / calculate_strategy_progress / detect_overdue_actions 纯函数）
- [ ] T037 [US2] 实现 `backend/app/infrastructure/repositories/ogsm_repo.py`（OGSM 四实体 CRUD）
- [ ] T038 [US2] 实现 `backend/app/workers/ogsm_refresh_tasks.py`（Celery 任务：数据导入后触发 G 进度重算）
- [ ] T039 [US2] 实现 `backend/app/application/ogsm_service.py`（G 目标管理业务逻辑 + 手工覆盖逻辑）
- [ ] T040 [US2] 实现 `backend/app/api/v1/ogsm_router.py` G 目标部分（GET/POST/PUT/DELETE /goals + POST /goals/{id}/override，契约见 contracts/ogsm.yaml）
- [ ] T041 [US2] 运行 T032-T034 测试，确认全部通过（绿灯）
- [ ] T042 [P] [US2] 实现前端 `frontend/src/components/ogsm/GoalProgressBar.vue`（进度条 + 完成率 < 80% 标红 + "暂无数据"状态）
- [ ] T043 [P] [US2] 实现前端 `frontend/src/api/ogsm.ts`（调用 OGSM API）
- [ ] T044 [US2] 实现前端 `frontend/src/stores/ogsm.ts`（Pinia store，缓存 OGSM 数据）

**Checkpoint**: 财务部录入 G 目标 → 进度条出现 → 手工覆盖标注"手工覆盖"标识，可独立验证

---

## Phase 5: US1 — OGSM 主界面（Priority: P1）

**Goal**: 运营总监打开软件看到 OGSM 一张表，G 进度自动计算，点击异常 G 跳转分析

**Independent Test**: 录入 O/G/S/M 数据，主界面进度条正确渲染，逾期行动标红

### 测试（TDD）

- [ ] T045 [US1] 编写 `backend/tests/integration/test_ogsm_summary.py`（test_summary_loads_all_entities / test_role_visibility_staff / test_action_complete_triggers_goal_refresh）— 确认失败
- [ ] T046 [P] [US1] 编写前端单元测试 `frontend/src/components/ogsm/__tests__/OgsmTable.spec.ts`（渲染/标红/逾期标注）— 确认失败

### 实现

- [ ] T047 [US1] 实现 `backend/app/api/v1/ogsm_router.py` 完整部分（GET /objectives + /strategies + /actions + GET /summary，含角色过滤）
- [ ] T048 [US1] 实现 `backend/app/api/v1/ogsm_router.py` POST /actions/{id}/complete（标记完成 + 触发 G 重算）
- [ ] T049 [US1] 运行 T045 测试，确认通过（绿灯）
- [ ] T050 [P] [US1] 实现前端 `frontend/src/components/ogsm/ActionItem.vue`（行动清单项 + 状态流转 + 逾期标红）
- [ ] T051 [P] [US1] 实现前端 `frontend/src/components/ogsm/StrategyCard.vue`（策略卡片 + 执行进度）
- [ ] T052 [US1] 实现前端 `frontend/src/components/ogsm/OgsmTable.vue`（OGSM 一张表主组件，组合 O/G/S/M 四区域）
- [ ] T053 [US1] 实现前端 `frontend/src/pages/OgsmPage.vue`（主界面，首页路由，P95 < 1s）
- [ ] T054 [US1] 运行 T046 前端测试，确认通过（绿灯）

**Checkpoint**: OGSM 一张表加载 P95 < 1s；G 进度条自动计算；店长只看自己的 M 行动

---

## Phase 6: US5 — BI 数据看板（Priority: P1）

**Goal**: 高管看到 8 个指标卡片（实际值/G目标/差距），超阈值标红，可跳转四步法

**Independent Test**: 导入数据后，8 个指标卡片正确显示实际值，与 G 目标对比差距

### 测试（TDD）

- [ ] T055 [P] [US5] 编写 `backend/tests/unit/domain/test_metrics.py`（test_revenue_factor_decomposition / test_cost_variance_rate / test_profit_model_thresholds）— 确认失败
- [ ] T056 [US5] 编写 `backend/tests/integration/test_dashboard_api.py`（test_dashboard_shows_goal_vs_actual / test_threshold_alerts / test_revenue_factor_decomposition_api / test_cost_three_line_comparison）— 确认失败

### 实现

- [ ] T057 [US5] 实现 `backend/app/domain/metrics.py`（客单因素/客流因素/损益率/阈值检查，纯函数）
- [ ] T058 [US5] 实现 `backend/app/domain/validators.py`（盈利模型阈值：净利润率/租售比/人工/坪效/人效）
- [ ] T059 [US5] 实现 `backend/app/infrastructure/repositories/sales_repo.py` + `cost_repo.py`
- [ ] T060 [US5] 实现 `backend/app/application/dashboard_service.py`（8 指标聚合 + G 目标对比 + Redis 缓存 5min TTL）
- [ ] T061 [P] [US5] 实现 `backend/app/application/revenue_service.py`（渠道/时段拆解 + 因素分解）
- [ ] T062 [P] [US5] 实现 `backend/app/application/cost_service.py`（三线成本对比 + 费用率结构）
- [ ] T063 [US5] 实现 `backend/app/api/v1/dashboard_router.py`（GET /dashboard/overview，契约见 contracts/dashboard.yaml）
- [ ] T064 [P] [US5] 实现 `backend/app/api/v1/revenue_router.py`（GET /revenue/breakdown + /factor-decomposition）
- [ ] T065 [P] [US5] 实现 `backend/app/api/v1/cost_router.py`（GET /cost/overview + /expenses）
- [ ] T066 [US5] 运行 T055-T056 测试，确认全部通过（绿灯）
- [ ] T067 [P] [US5] 实现前端 `frontend/src/components/common/MetricCard.vue`（实际值/G目标/差距/标红/"查看分析"/"创建行动"按钮）
- [ ] T068 [P] [US5] 实现前端 `frontend/src/components/common/WaterfallChart.vue`（ECharts 瀑布图）
- [ ] T069 [P] [US5] 实现前端 `frontend/src/components/common/TrendChart.vue` + `LoadingState.vue`（三态：Loading/Empty/Error）
- [ ] T070 [US5] 实现前端 `frontend/src/pages/DashboardPage.vue`（8 指标看板，LCP < 2.5s）
- [ ] T071 [P] [US5] 实现前端 `frontend/src/pages/RevenuePage.vue` + `CostPage.vue`

**Checkpoint**: 8 个指标卡片首屏 < 2.5s；超阈值标红；点击"查看分析"跳转四步法入口

---

## Phase 7: US3 — 四步法分析工作台（Priority: P2）

**Goal**: 运营总监通过四步法分析生成 M 行动，自动关联到 OGSM 一张表

**Independent Test**: 从 OGSM 主界面点击偏差 G 目标，完成四步法，验证生成的 M 行动出现在 OGSM 表 M 列

### 测试（TDD）

- [ ] T072 [P] [US3] 编写 `backend/tests/unit/domain/test_analysis_engine.py`（test_step1_loads_goal_vs_actual / test_step2_channel_breakdown / test_step3_factor_decomposition / test_session_draft_save）— 确认失败
- [ ] T073 [P] [US3] 编写 `backend/tests/unit/domain/test_action_recommender.py`（test_traffic_factor_recommends_marketing / test_unit_price_factor_recommends_menu / test_cost_overrun_recommends_procurement）— 确认失败
- [ ] T074 [US3] 编写 `backend/tests/integration/test_analysis_flow.py`（test_step4_generates_action / test_action_appears_in_ogsm / test_three_trigger_entries / test_session_draft_resume）— 确认失败

### 实现

- [ ] T075 [US3] 实现 `backend/app/domain/analysis_engine.py`（四步法核心逻辑：Step1-4 数据处理）
- [ ] T076 [US3] 实现 `backend/app/domain/action_recommender.py`（规则引擎：归因类型 → 推荐行动模板）
- [ ] T077 [US3] 实现 `backend/app/application/analysis_service.py`（会话管理 + 草稿保存 + 行动生成）
- [ ] T078 [US3] 实现 `backend/app/api/v1/analysis_router.py`（POST /sessions + PUT /sessions/{id}/step/{n} + POST /sessions/{id}/generate-actions，契约见 contracts/analysis.yaml）
- [ ] T079 [US3] 运行 T072-T074 测试，确认全部通过（绿灯）
- [ ] T080 [P] [US3] 实现前端 `frontend/src/components/analysis/Step1Compare.vue`（G目标 vs 实际，三维对比，差异 > 5% 标红）
- [ ] T081 [P] [US3] 实现前端 `frontend/src/components/analysis/Step2Breakdown.vue`（维度拆解 + 主要缺口标注）
- [ ] T082 [P] [US3] 实现前端 `frontend/src/components/analysis/Step3Attribution.vue`（客单/客流因素瀑布图 + 归因确认）
- [ ] T083 [P] [US3] 实现前端 `frontend/src/components/analysis/Step4Decision.vue`（推荐行动模板 + Who/When/预期效果填写）
- [ ] T084 [US3] 实现前端 `frontend/src/components/analysis/AnalysisWizard.vue`（4步向导容器 + 草稿自动保存）
- [ ] T085 [US3] 实现前端 `frontend/src/pages/AnalysisPage.vue`（三种触发入口：OGSM主界面/BI看板/预警）
- [ ] T086 [US3] 实现前端 `frontend/src/stores/analysis.ts`（四步法会话状态管理）
- [ ] T087 [US3] 编写 Playwright E2E 测试 `frontend/e2e/analysis-full-flow.spec.ts`（4步完整流程 → 生成 M 行动 → 出现在 OGSM 表）

**Checkpoint**: 四步法 P95 < 500ms；生成的 M 行动自动出现在 OGSM 一张表 M 列

---

## Phase 8: US4 — 店长执行行动（Priority: P2）

**Goal**: 店长登录只看自己的 M 行动，标记完成后触发 G 进度重算

**Independent Test**: 以店长身份登录，验证只能看到自己的行动，完成后 G 进度更新

### 测试（TDD）

- [ ] T088 [US4] 编写 `backend/tests/integration/test_staff_role.py`（test_staff_sees_only_own_actions / test_complete_action_triggers_goal_refresh / test_overdue_auto_detection）— 确认失败

### 实现

- [ ] T089 [US4] 实现逾期检测 Celery 定时任务 `backend/app/workers/ogsm_refresh_tasks.py`（每日 00:05 扫描逾期行动，更新状态为 overdue）
- [ ] T090 [US4] 验证 `backend/app/api/v1/ogsm_router.py` 中 GET /actions 的角色过滤（staff 只返回 who == current_user_id 的行动）
- [ ] T091 [US4] 运行 T088 测试，确认全部通过（绿灯）

**Checkpoint**: 店长查看行动清单并标记完成 ≤ 3 步；G 进度在完成后自动更新

---

## Phase 9: Polish & 质量门禁

**Purpose**: 全链路验收，性能压测，发布前检查

- [ ] T092 运行 `pytest --cov=app`，确认覆盖率 ≥ 80%（业务逻辑）/ ≥ 90%（数据接口层）
- [ ] T093 [P] 运行 `npm run test -- --run`，确认全部前端单元测试通过
- [ ] T094 运行 Playwright E2E 关键路径：OGSM录入G→导入数据→进度更新 / 四步法完整流程→生成M行动 / 数据导入3步向导 / 店长角色隔离
- [ ] T095 [P] 运行后端质量门禁：`black --check . && isort --check . && mypy app/ && bandit -r app/`
- [ ] T096 [P] 运行前端质量门禁：`npm run lint && tsc --noEmit`
- [ ] T097 k6 压测：50 并发，验证 OGSM 一张表 P95 < 1s，看板 P95 < 200ms，四步法 P95 < 500ms
- [ ] T098 手动验收：对照 `specs/001-ogsm-tool/spec.md` 所有 Acceptance Scenarios 逐一验证（US1-US6 共 18 个场景）

---

## Dependencies & Execution Order

### Phase 依赖关系

```
Phase 1（脚手架）
    └── Phase 2（多租户基础设施）
            ├── Phase 3（数据导入）─────────────────────────────┐
            ├── Phase 4（G 目标管理）                           │
            │       └── Phase 5（OGSM 主界面）                  │
            │                   └── Phase 7（四步法工作台）      │
            │                               └── Phase 8（店长）  │
            └──────────────────────────────── Phase 6（BI 看板）─┘
                                                    └── Phase 9（质量门禁）
```

### 并行机会

- Phase 3（数据导入）与 Phase 4（G 目标管理）可并行（不同文件，无依赖）
- Phase 6（BI 看板）可在 Phase 3 + Phase 4 完成后与 Phase 5 并行
- 每个 Phase 内标 [P] 的任务可并行执行

### 用户故事独立性

| 用户故事 | 前置 Phase | 可独立验证 |
|---|---|---|
| US6 数据导入 | Phase 2 | ✅ 上传→预览→确认→任务状态 |
| US2 G 目标管理 | Phase 2 | ✅ 录入→进度条→手工覆盖 |
| US1 OGSM 主界面 | Phase 4 | ✅ 一张表加载→进度→角色隔离 |
| US5 BI 看板 | Phase 3 + Phase 4 | ✅ 8指标卡片→标红→跳转 |
| US3 四步法工作台 | Phase 5 | ✅ 4步流程→生成M行动→OGSM更新 |
| US4 店长执行行动 | Phase 5 | ✅ 角色过滤→完成→G重算 |

---

## Implementation Strategy

### MVP First（US6 + US2 + US1）

1. Phase 1 + Phase 2（脚手架 + 多租户）
2. Phase 3（数据导入）— 数据基础
3. Phase 4（G 目标管理）— 进度计算核心
4. Phase 5（OGSM 主界面）— 主界面可用
5. **STOP & VALIDATE**: OGSM 一张表可独立演示

### 并行团队策略（3人）

完成 Phase 1 + Phase 2 后：
- Dev A：Phase 3（数据导入）→ Phase 6（BI 看板）
- Dev B：Phase 4（G 目标管理）→ Phase 5（OGSM 主界面）
- Dev C：Phase 7（四步法工作台）→ Phase 8（店长）

---

## Notes

- `[P]` = 不同文件，无依赖，可并行
- `[USn]` = 对应 spec.md 中的用户故事编号
- 每个测试任务必须先确认失败，再实现代码
- 每个 Phase 完成后提交一个 commit
- 覆盖率低于阈值时，Phase 9 不可通过
