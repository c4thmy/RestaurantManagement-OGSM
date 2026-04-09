# Feature Specification: 餐饮 BI 中台工具 — OGSM 驱动 MVP

**Feature Branch**: `001-bi-dashboard-mvp`  
**Created**: 2026-04-05  
**Updated**: 2026-04-05 (v2.0 — OGSM 为核心重构)  
**Status**: Draft  
**Input**: 需求分析文档 `docQA/20260405_BI中台工具需求分析.md` v2.0

---

## User Scenarios & Testing

### User Story 1 — 运营总监用 OGSM 一张表主导月度经营复盘（Priority: P1）

运营总监每月初打开系统，看到的是本企业 OGSM 一张表：G 目标进度条由系统自动从 BI 数据计算，S 策略执行进度由 M 行动完成率驱动。总监识别出 G2（净利润率）进度落后，点击后直接进入四步法分析工作台，完成归因后一键生成 M 行动并分配给责任人，全程不超过 3 步核心操作。

**Why this priority**：OGSM 一张表是软件的核心界面，是所有其他模块的入口和汇聚点。没有它，其他功能都是孤立的分析工具。

**Independent Test**：录入一套 OGSM（1个O/3个G/3个S/5个M），导入一个月的经营数据，验证 G 目标进度条自动计算，M 行动状态可更新，整体 OGSM 一张表可正常展示。

**Acceptance Scenarios**:

1. **Given** 已录入 OGSM 目标体系，**When** 导入当月经营数据，**Then** G 目标进度条自动更新（实际值/目标值/完成率），无需手工填写
2. **Given** G2 净利润率进度落后（完成率 < 80%），**When** 查看 OGSM 一张表，**Then** 该 G 目标自动标红，显示"落后 X%"
3. **Given** G 目标标红，**When** 点击该 G 目标，**Then** 跳转到四步法分析工作台，自动加载该指标的历史数据
4. **Given** 四步法分析完成，**When** 点击"生成行动"，**Then** 创建 M 行动并自动关联到对应 S 策略，出现在 OGSM 一张表的 M 列
5. **Given** M 行动已创建，**When** 责任人标记完成，**Then** 关联 G 目标的进度自动重新计算

---

### User Story 2 — 店长用四步法工作台完成一次完整的收入异动分析（Priority: P1）

店长发现本周营收环比下降 12%，但不知道原因。打开四步法工作台，系统引导他完成：识别异常指标 → 渠道/时段拆解 → 客单/客流因素计算 → 生成行动清单。全程有引导提示，不需要数据分析背景。

**Why this priority**：四步法工作台是 OGSM 的执行引擎，解决"店长不会分析"的核心痛点（OGSM 风险 R1）。

**Independent Test**：给定含渠道/时段/客流/客单数据的数据集，验证四步法工作台能完成完整的 4 步流程，最终生成至少 1 条 M 行动。

**Acceptance Scenarios**:

1. **Given** 已有两周经营数据，**When** 进入四步法工作台 Step 1，**Then** 自动展示本周 vs 上周各指标对比，差异率 > 5% 的指标高亮，引导用户选择要分析的异常指标
2. **Given** 选择"总营收"进入 Step 2，**When** 选择"渠道拆解"，**Then** 展示堂食/外卖/自提各渠道收入对比，自动标注贡献最大的异动渠道
3. **Given** 确认"外卖渠道缺口最大"，**When** 进入 Step 3，**Then** 自动计算客单因素和客流因素，以瀑布图展示，标注主要归因（如"客流下降贡献 80%"）
4. **Given** 归因确认为"外卖客流下降"，**When** 进入 Step 4，**Then** 系统推荐 3 条行动模板（如"检查外卖平台曝光/优化套餐/发放优惠券"），用户选择后填写 Who/When，一键生成 M 行动
5. **Given** 生成 M 行动，**When** 查看 OGSM 一张表，**Then** 新行动出现在对应 S 策略下的 M 列

---

### User Story 3 — 财务部上传月度经营数据并通过质量检查（Priority: P1）

财务部每月从 POS 系统导出 Excel，上传到 BI 工具。系统自动校验数据质量，有问题立即提示，不让脏数据进入分析，避免"数据质量差导致分析结论失真"（OGSM 风险 R2）。

**Why this priority**：数据质量是所有 G 目标自动计算的前提，脏数据会导致 OGSM 进度条失真，误导决策。

**Independent Test**：上传含格式错误的 Excel，验证系统拦截并给出具体修复提示；上传正确数据，验证 G 目标进度条自动更新。

**Acceptance Scenarios**:

1. **Given** 用户上传 Excel 文件，**When** 文件解析完成，**Then** 展示前 20 行预览和自动识别的字段映射，用户可手动调整
2. **Given** 文件含 critical 错误（日期格式错误/金额非数字），**When** 执行质量检查，**Then** 列出所有问题（行号+字段+修复建议），阻止导入，不写入数据库
3. **Given** 数据质量评分 < 80，**When** 执行质量检查，**Then** 展示 warning 列表，用户可选择"忽略继续"或"下载修复后重传"
4. **Given** 导入成功，**When** 查看 OGSM 一张表，**Then** 所有关联 BI 指标的 G 目标进度条自动刷新

---

### User Story 4 — 高管查看 OGSM 总表，30 秒内识别战略执行状态（Priority: P1）

集团高管每周打开系统，30 秒内看清：哪个 G 目标落后、哪条 S 策略执行不力、哪些 M 行动逾期。不需要点击任何分析页面，OGSM 一张表本身就是完整的战略执行状态图。

**Why this priority**：高管是 OGSM 的最终决策者，他们的使用体验决定工具能否在企业内推广。

**Independent Test**：录入完整 OGSM 数据，设置部分 G 落后/M 逾期，验证高管打开 OGSM 一张表后能在 30 秒内识别所有异常状态。

**Acceptance Scenarios**:

1. **Given** OGSM 数据完整，**When** 高管打开系统，**Then** 首页即为 OGSM 一张表，无需额外导航
2. **Given** G2 完成率 < 80%，**When** 查看 OGSM 一张表，**Then** G2 行标红，显示"落后 X%，距截止还有 Y 天"
3. **Given** 某 M 行动已逾期，**When** 查看 OGSM 一张表，**Then** 该行动标红，显示"逾期 X 天，责任人：XXX"
4. **Given** 查看 OGSM 一张表，**When** 切换查看维度（集团/品牌/门店），**Then** 对应层级的 OGSM 数据展示，支持多层级下钻

---

### User Story 5 — 运营总监做多店对比，找标杆店复制成功经验（Priority: P2）

运营总监需要对比旗下 10 家门店，找出标杆店（综合评分最高）和问题店，提取可复制的成功经验，并将改善行动写入 OGSM 的 M 列。

**Why this priority**：多店对比是 OGSM S3（盈利模型设计）的核心数据支撑，标杆店的参数是盈利模型的常量来源。

**Independent Test**：录入 10 家门店数据，验证系统自动计算综合评分，标注标杆店和问题店，支持对比结果直接生成 M 行动。

**Acceptance Scenarios**:

1. **Given** 多门店数据已导入，**When** 打开多店对比页，**Then** 展示所有门店综合评分排名，自动标注 Top1（标杆）和 Bottom1（问题店）
2. **Given** 选择标杆店和问题店对比，**When** 查看对比结果，**Then** 展示两店在各指标上的差距，差距最大的指标排在最前
3. **Given** 对比分析完成，**When** 点击"生成改善行动"，**Then** 系统推荐基于标杆店数据的改善行动模板，用户确认后写入 OGSM M 列

---

### Edge Cases

- OGSM 中 G 目标关联的 BI 指标暂无数据：进度条显示"暂无数据"，不显示 0%，避免误判
- 同一 G 目标关联多个 BI 指标：取加权平均或用户指定主指标
- M 行动责任人离职/账号注销：行动状态变为"待重新分配"，通知上级
- 四步法分析中途退出：自动保存草稿，下次进入可继续
- 导入数据与已有数据日期重叠：提示用户选择导入模式（追加/覆盖/跳过重复）
- OGSM 目标期末未完成：自动归档到历史记录，不影响新周期 OGSM

---

## Requirements

### Functional Requirements

**OGSM 核心模块**
- **FR-001**: 系统 MUST 支持 OGSM 四层结构（O/G/S/M）的录入、编辑和展示
- **FR-002**: 系统 MUST 自动从 BI 数据计算 G 目标的当前实际值和完成进度，无需手工填写
- **FR-003**: G 目标完成率 < 80% 时，系统 MUST 自动标红并显示落后幅度
- **FR-004**: M 行动完成后，系统 MUST 自动触发关联 G 目标的进度重新计算
- **FR-005**: 系统 MUST 支持内置 G 目标模板（G_PROFIT/G_COST/G_MEMBER/G_EFFICIENCY/G_TURNOVER）
- **FR-006**: 系统 MUST 支持 OGSM 多层级（集团/品牌/门店），各层级独立维护
- **FR-007**: M 行动逾期时，系统 MUST 自动标红并发送站内通知给责任人和上级

**四步法分析工作台**
- **FR-010**: 系统 MUST 提供 4 步引导式分析流程（Step 1-4），每步有明确操作提示
- **FR-011**: Step 1 MUST 自动加载当前 G 目标 vs 实际值，差异率 > 5% 的指标高亮
- **FR-012**: Step 2 MUST 支持渠道/时段/商圈/顾客类型四个拆解维度，操作 ≤ 3 步
- **FR-013**: Step 3 MUST 自动计算客单因素和客流因素，以瀑布图展示
- **FR-014**: Step 4 MUST 支持从分析结果直接生成 M 行动，自动关联到 OGSM
- **FR-015**: 分析会话 MUST 自动保存草稿，支持中途退出后继续

**BI 数据看板**
- **FR-020**: 系统 MUST 展示 8 个核心指标，每个指标卡片显示实际值/G 目标值/差距
- **FR-021**: 系统 MUST 支持预算/同比/环比三维对比，差异率 > 5% 自动标红
- **FR-022**: 系统 MUST 支持从指标卡片直接跳转到四步法工作台
- **FR-023**: 系统 MUST 支持从指标卡片直接创建 M 行动
- **FR-024**: 看板数据刷新延迟 MUST ≤ 5 分钟

**收入分析**
- **FR-030**: 系统 MUST 支持收入按渠道/时段/来源/商圈多维下钻
- **FR-031**: 系统 MUST 自动计算客单因素和客流因素，以瀑布图展示
- **FR-032**: 系统 MUST 支持同店同比/环比/定基比三种时间对比

**成本费用分析**
- **FR-040**: 系统 MUST 展示标准/理论/实际三线成本对比，自动计算原材料损益率
- **FR-041**: 系统 MUST 对超出盈利模型阈值的指标自动标注（利润率<15%/租售比>15%/人工>25%）
- **FR-042**: 系统 MUST 支持按食材品类/供应商下钻成本分析

**数据导入**
- **FR-050**: 系统 MUST 支持 CSV/Excel 文件上传，单文件上限 50MB
- **FR-051**: 系统 MUST 执行数据质量检查，输出质量评分和分级问题列表
- **FR-052**: critical 问题或质量评分 < 80 时，系统 MUST 阻止写入
- **FR-053**: 系统 MUST 支持导入预览（前 20 行）和字段映射配置
- **FR-054**: 导入成功后，系统 MUST 自动触发关联 G 目标进度刷新

**多店对比**
- **FR-060**: 系统 MUST 支持最多 20 家门店横向指标对比
- **FR-061**: 系统 MUST 自动计算综合评分，标注标杆店和问题店
- **FR-062**: 系统 MUST 支持从对比结果直接生成 M 行动

**通用**
- **FR-070**: 系统 MUST 实现多租户数据隔离，用户只能访问本租户数据
- **FR-071**: 系统 MUST 实现角色权限控制（owner/manager/staff/readonly）
- **FR-072**: 系统 MUST 支持报表导出（Excel/PDF）

### Key Entities

- **OgsmObjective**：O 目的，属于 Tenant，包含 title/description/period/level
- **OgsmGoal**：G 目标，属于 Objective，包含 kpi_metric/target_value/current_value/period/status
- **OgsmStrategy**：S 策略，关联多个 Goal，包含 title/description/execution_progress
- **OgsmAction**：M 行动，属于 Strategy，包含 what/who/deadline/status/kpi_impact/created_from
- **AnalysisSession**：四步法分析会话，包含 4 步数据快照和生成的行动列表
- **Tenant**：餐饮企业租户，包含 business_type/currency/timezone 等配置
- **Store**：门店，属于 Tenant，包含 name/address/area_sqm/seat_count
- **SalesRecord**：销售记录，包含 record_date/store_id/channel/shift/revenue/customer_count
- **CostRecord**：成本记录，包含 record_date/store_id/category/standard/theoretical/actual
- **ImportJob**：导入任务，包含 status/quality_score/total_rows/imported_rows
- **Budget**：预算数据，包含 store_id/period/metric_key/budget_value

---

## Success Criteria

### Measurable Outcomes

- **SC-001**: 运营总监打开系统，30 秒内能识别出哪个 G 目标落后（用户测试通过率 ≥ 90%）
- **SC-002**: 店长完成一次完整四步法分析（从进入工作台到生成 M 行动）≤ 10 分钟
- **SC-003**: G 目标进度条在数据导入后 5 分钟内自动更新
- **SC-004**: OGSM 一张表加载时间 P95 < 1s
- **SC-005**: 四步法工作台每步操作响应时间 P95 < 500ms
- **SC-006**: 数据质量检查准确率 ≥ 95%（critical 问题不漏报）
- **SC-007**: 系统支持 50 个并发用户同时操作不降级
- **SC-008**: 从 BI 看板异常指标到生成 M 行动，操作步骤 ≤ 5 步

---

## Assumptions

- 初期目标用户为中小连锁餐饮（5-50 家门店），单租户数据量不超过 500 万行/年
- OGSM 目标体系由运营总监/财务部手工录入，系统提供模板辅助
- G 目标进度自动计算依赖 BI 数据准确性，数据质量门禁是前提
- 外卖平台数据 v1 阶段通过手工导出 Excel 上传，不做自动同步
- 移动端 v1 不支持，仅支持 1024px 以上屏幕
- AI 分析摘要和行动推荐依赖外部 LLM API，API Key 由用户通过环境变量配置
- 行动推荐模板 v1 基于规则引擎（归因类型 → 推荐行动），不依赖 AI
- OGSM 多层级 v1 支持集团/门店两级，品牌层级 v2 再加
