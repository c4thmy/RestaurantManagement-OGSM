# Research: OGSM 落地工具

**Branch**: `001-ogsm-tool` | **Date**: 2026-04-06  
**Purpose**: 解决 Technical Context 中的技术选型问题，为 Phase 1 设计提供依据

---

## 1. OGSM 进度自动计算引擎

**Decision**: 纯函数实现，独立于 ORM，由 Celery 任务触发刷新

**Rationale**:
- G 目标进度 = `current_value / target_value`，current_value 从 BI 聚合查询得出
- 纯函数设计（`ogsm_progress.py`）便于单元测试，无副作用
- 数据导入完成后通过 Celery 任务链异步触发刷新，不阻塞导入响应
- 手工覆盖时写入 `manual_override=True` 标志，下次自动刷新跳过该字段

**Alternatives considered**:
- 实时计算（每次请求重算）：OGSM 一张表 P95 < 1s 要求下，20+ 个 G 目标同时聚合查询压力过大，拒绝
- 数据库触发器：难以测试，与 Alembic 迁移冲突，拒绝

---

## 2. 四步法分析会话持久化

**Decision**: 数据库持久化（`analysis_sessions` 表），支持草稿保存和恢复

**Rationale**:
- 四步法完整流程预计 10-15 分钟，中途退出需保存进度
- 会话数据（step1_data/step2_data/step3_data）存为 JSONB，灵活扩展
- 会话状态：`draft` → `step1_complete` → `step2_complete` → `step3_complete` → `actions_generated`
- 生成的 M 行动 ID 列表存入 `actions_generated` 字段，支持回溯

**Alternatives considered**:
- 前端 localStorage：刷新/换设备丢失，多人协作无法共享，拒绝
- Redis 临时存储：TTL 到期丢失，无法支持长期草稿，拒绝

---

## 3. 行动推荐规则引擎（v1）

**Decision**: 硬编码规则引擎，基于归因类型映射推荐行动模板

**Rationale**:
- v1 快速交付，规则来自方法论文档，可解释性强
- 规则结构：`归因类型 → 推荐行动模板列表`
  - 客流因素为主 → `["营销引流活动", "渠道优化（外卖平台曝光）", "会员召回"]`
  - 客单因素为主 → `["菜单结构调整", "套餐设计优化", "高毛利品推荐"]`
  - 成本超标 → `["食材采购优化", "损耗管控", "供应商谈判"]`
- 用户可编辑推荐行动后再生成，不强制接受

**Alternatives considered**:
- Claude API 智能推荐：延迟高（1-3s），成本高，v1 不稳定，降级策略复杂，推迟到 v2

---

## 4. 多租户 Schema 隔离

**Decision**: PostgreSQL 独立 Schema（Bridge 模式），命名规则 `tenant_{tenant_id}`

**Rationale**: 已在 `multi-tenant-isolation.md` 详细定义，直接复用
- MVP 阶段单租户，但数据模型和 Repository 层按多租户设计，避免后期重构
- TenantContext 通过 JWT 注入，不信任请求体中的 tenant_id

---

## 5. BI 看板缓存策略

**Decision**: Redis 缓存，TTL 5 分钟，数据导入后主动失效

**Rationale**:
- 看板 8 个指标聚合查询较重，缓存可将 P95 从 ~800ms 降至 < 200ms
- 数据导入完成后 Celery 任务主动删除相关缓存 key，保证数据新鲜度
- 缓存 key 格式：`dashboard:{tenant_id}:{store_id}:{date_range}`

---

## 6. 角色权限模型

**Decision**: JWT 携带角色信息，FastAPI 依赖注入层强制校验

**Rationale**: 已在 `multi-tenant-isolation.md` 定义权限矩阵，本功能新增 OGSM 专属规则：
- `tenant_owner` / `store_manager`（运营总监/高管/财务部）：可查看全部 OGSM 数据
- `staff`（店长）：只能查看分配给自己的 M 行动（`action.who == current_user_id`）
- G 目标数据：仅 `owner` / `manager` 角色可见

---

## 7. 数据导入触发 G 进度刷新

**Decision**: Celery 任务链：`import_task → ogsm_refresh_task`

**Rationale**:
- 导入完成后异步触发，不阻塞用户操作
- `ogsm_refresh_task` 重新聚合所有关联 BI 指标，更新 `ogsm_goals.current_value`
- 前端通过轮询 `GET /api/v1/ogsm/goals/{id}/progress` 获取最新进度（或 WebSocket，v2 考虑）

---

## 8. 前端状态管理

**Decision**: Pinia store 分模块（ogsm / analysis / dashboard），与 constitution 约定一致

**Rationale**:
- `ogsm` store：缓存 OGSM 一张表数据，M 行动状态变更后局部更新
- `analysis` store：管理四步法会话状态，支持步骤间数据传递
- `dashboard` store：缓存看板数据，导入完成后触发刷新

---

## 所有 NEEDS CLARIFICATION 已解决

| 项目 | 解决方式 |
|---|---|
| 多层级 OGSM 汇总方式 | 用户选择方案 A：各层独立，不自动汇总 |
| G 进度计算触发时机 | 数据导入完成 + M 行动完成，均触发异步刷新 |
| 四步法会话存储 | 数据库持久化，支持草稿 |
| 行动推荐机制 | v1 规则引擎，v2 考虑 AI |
