# Data Model: OGSM 落地工具

**Branch**: `001-ogsm-tool` | **Date**: 2026-04-06  
**Source**: `spec.md` Key Entities + `research.md` 技术决策

---

## Entity Relationship Overview

```
OgsmObjective (O)
    │ 1:N
    ▼
OgsmGoal (G) ──── kpi_metric ──→ BI 聚合查询（sales_records / cost_records）
    │ 1:N
    ▼
OgsmStrategy (S)
    │ 1:N
    ▼
OgsmAction (M) ──── who ──→ User
    │
    └── kpi_impact ──→ 触发 OgsmGoal.current_value 重算

AnalysisSession ──── triggered_by ──→ OgsmGoal / Alert
    │
    └── actions_generated ──→ OgsmAction[]

OgsmReview ──── period ──→ 季度/月度复盘
```

---

## 实体定义

### OgsmObjective（O — 目的）

```python
@dataclass
class OgsmObjective:
    id: UUID
    tenant_id: str
    title: str                    # 目的标题（鼓舞人心的定性描述）
    description: str | None       # 详细描述
    period: str                   # "2026-Q1" / "2026-annual"
    level: OgsmLevel              # CORPORATE / BRAND / STORE
    store_id: str | None          # level=STORE 时必填
    created_by: UUID
    created_at: datetime
    updated_at: datetime

class OgsmLevel(str, Enum):
    CORPORATE = "corporate"       # 集团
    BRAND     = "brand"           # 品牌
    STORE     = "store"           # 门店
```

**约束**:
- `title` 不可为空，最长 200 字符
- `level=STORE` 时 `store_id` 必填
- 各层级独立，不自动汇总（方案 A）

---

### OgsmGoal（G — 目标）

```python
@dataclass
class OgsmGoal:
    id: UUID
    objective_id: UUID            # 关联 OgsmObjective
    tenant_id: str
    title: str
    kpi_metric: str               # 关联 BI 指标 key（见内置模板）
    target_value: Decimal         # 目标值（不可为 0）
    baseline_value: Decimal       # 基准值
    current_value: Decimal | None # 系统自动填充，None = 暂无数据
    unit: str                     # "%", "元", "次/天", "分"
    period_start: date
    period_end: date
    owner_role: str               # 责任角色
    status: GoalStatus
    manual_override: bool = False # True = current_value 为手工覆盖值
    manual_override_by: UUID | None = None
    manual_override_at: datetime | None = None
    last_refreshed_at: datetime | None = None
    created_by: UUID
    created_at: datetime
    updated_at: datetime

class GoalStatus(str, Enum):
    ACTIVE   = "active"
    ACHIEVED = "achieved"
    FAILED   = "failed"
    ARCHIVED = "archived"
```

**内置 KPI 指标 key**:

| kpi_metric | 描述 | 单位 | 预警阈值 |
|---|---|---|---|
| `net_profit_rate` | 净利润率 | % | < 15% 标红 |
| `total_expense_rate` | 综合费用率 | % | 租售比 > 15% 或人工 > 25% |
| `member_repurchase_rate` | 会员复购率 | % | 差异率 > 5% |
| `revenue_per_staff_day` | 人效 | 元/人/天 | < 1000 |
| `table_turnover_rate` | 翻台率 | 次/天 | < 2 |
| `review_score` | 大众点评评分（手工） | 分 | < 4.7 |
| `total_revenue` | 总营收 | 元 | 差异率 > 5% |
| `gross_margin_rate` | 综合毛利率 | % | 差异率 > 3% |

**约束**:
- `target_value != 0`（防止除零）
- `current_value = None` 时进度条显示"暂无数据"，不显示 0%
- `manual_override=True` 时，自动刷新任务跳过该字段

**进度计算**:
```python
def calculate_progress(goal: OgsmGoal) -> float | None:
    if goal.current_value is None:
        return None  # 暂无数据
    if goal.target_value == 0:
        raise ValueError("target_value 不能为零")
    return float(goal.current_value / goal.target_value)

def is_alert(goal: OgsmGoal) -> bool:
    progress = calculate_progress(goal)
    if progress is None:
        return False
    return progress < 0.8  # 完成率 < 80% 标红
```

---

### OgsmStrategy（S — 策略）

```python
@dataclass
class OgsmStrategy:
    id: UUID
    tenant_id: str
    goal_ids: list[UUID]          # 关联一个或多个 OgsmGoal
    title: str
    description: str | None
    owner_role: str
    execution_progress: float     # 自动计算 = 关联 M 行动完成率
    created_by: UUID
    created_at: datetime
    updated_at: datetime
```

**内置策略模板**:

| template_id | 策略名称 | 关联 G |
|---|---|---|
| `S_3C` | 3C 指标体系建立 | `net_profit_rate` |
| `S_4STEP` | 四步法推行 | `net_profit_rate`, `total_expense_rate` |
| `S_MODEL` | 盈利模型设计 | `net_profit_rate` |
| `S_MEMBER` | 顾客资产运营 | `member_repurchase_rate` |
| `S_TRAIN` | 团队能力培训 | `revenue_per_staff_day` |

**执行进度计算**:
```python
def calculate_strategy_progress(strategy_id: UUID, actions: list[OgsmAction]) -> float:
    related = [a for a in actions if a.strategy_id == strategy_id]
    if not related:
        return 0.0
    done = sum(1 for a in related if a.status == ActionStatus.DONE)
    return done / len(related)
```

---

### OgsmAction（M — 行动清单）

```python
@dataclass
class OgsmAction:
    id: UUID
    tenant_id: str
    strategy_id: UUID             # 关联 OgsmStrategy
    title: str                    # 行动标题
    what: str                     # 具体做什么
    who: UUID                     # 责任人 user_id
    deadline: date                # 截止日期
    expected_result: str          # 预期效果
    actual_result: str | None     # 实际效果（完成后填写）
    kpi_impact: dict | None       # {"kpi_metric": "net_profit_rate", "expected_delta": 0.02}
    status: ActionStatus
    priority: ActionPriority
    created_from: ActionSource    # MANUAL / ANALYSIS / ALERT
    analysis_session_id: UUID | None  # 来源分析会话
    created_by: UUID
    created_at: datetime
    updated_at: datetime
    completed_at: datetime | None

class ActionStatus(str, Enum):
    PENDING     = "pending"       # 未开始
    IN_PROGRESS = "in_progress"   # 进行中
    DONE        = "done"          # 已完成
    OVERDUE     = "overdue"       # 已逾期

class ActionPriority(str, Enum):
    HIGH   = "high"
    MEDIUM = "medium"
    LOW    = "low"

class ActionSource(str, Enum):
    MANUAL   = "manual"
    ANALYSIS = "analysis"         # 四步法生成
    ALERT    = "alert"            # 异常预警触发
```

**状态流转**:
```
PENDING → IN_PROGRESS → DONE
    ↓           ↓
  OVERDUE    OVERDUE   （截止日期过后自动检测）
```

**逾期检测**（Celery 定时任务，每日 00:05 执行）:
```python
def detect_overdue_actions(actions: list[OgsmAction], today: date) -> list[UUID]:
    return [
        a.id for a in actions
        if a.status in (ActionStatus.PENDING, ActionStatus.IN_PROGRESS)
        and a.deadline < today
    ]
```

---

### AnalysisSession（四步法分析会话）

```python
@dataclass
class AnalysisSession:
    id: UUID
    tenant_id: str
    triggered_by: str             # "goal:{goal_id}" / "alert:{alert_id}" / "manual"
    goal_id: UUID | None          # 关联的 G 目标
    status: SessionStatus
    step1_data: dict | None       # 一比目标：G目标 vs 实际值对比数据
    step2_data: dict | None       # 二拆数据：维度拆解结果
    step3_data: dict | None       # 三定归因：客单/客流因素计算结果
    step4_data: dict | None       # 四做决策：选择的行动模板
    actions_generated: list[UUID] # 生成的 M 行动 ID 列表
    created_by: UUID
    created_at: datetime
    updated_at: datetime

class SessionStatus(str, Enum):
    DRAFT             = "draft"
    STEP1_COMPLETE    = "step1_complete"
    STEP2_COMPLETE    = "step2_complete"
    STEP3_COMPLETE    = "step3_complete"
    ACTIONS_GENERATED = "actions_generated"
```

---

### OgsmReview（复盘记录）

```python
@dataclass
class OgsmReview:
    id: UUID
    tenant_id: str
    period: str                   # "2026-Q1" / "2026-03"
    summary: str                  # 本期总结
    lessons_learned: str | None   # 经验教训
    next_period_adjustments: str | None  # 下期调整方向
    created_by: UUID
    created_at: datetime
```

---

## 数据库 Schema（SQL）

```sql
-- 在 tenant_{tenant_id} schema 内执行

CREATE TABLE ogsm_objectives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(200) NOT NULL,
    description     TEXT,
    period          VARCHAR(20) NOT NULL,
    level           VARCHAR(20) NOT NULL CHECK (level IN ('corporate','brand','store')),
    store_id        VARCHAR(64),
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ogsm_goals (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    objective_id          UUID NOT NULL REFERENCES ogsm_objectives(id),
    title                 VARCHAR(200) NOT NULL,
    kpi_metric            VARCHAR(64) NOT NULL,
    target_value          DECIMAL(15,4) NOT NULL CHECK (target_value != 0),
    baseline_value        DECIMAL(15,4),
    current_value         DECIMAL(15,4),
    unit                  VARCHAR(20) NOT NULL,
    period_start          DATE NOT NULL,
    period_end            DATE NOT NULL,
    owner_role            VARCHAR(64),
    status                VARCHAR(20) DEFAULT 'active',
    manual_override       BOOLEAN DEFAULT FALSE,
    manual_override_by    UUID,
    manual_override_at    TIMESTAMPTZ,
    last_refreshed_at     TIMESTAMPTZ,
    created_by            UUID NOT NULL,
    created_at            TIMESTAMPTZ DEFAULT NOW(),
    updated_at            TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ogsm_strategies (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_ids             UUID[] NOT NULL,
    title                VARCHAR(200) NOT NULL,
    description          TEXT,
    owner_role           VARCHAR(64),
    execution_progress   DECIMAL(5,4) DEFAULT 0,
    created_by           UUID NOT NULL,
    created_at           TIMESTAMPTZ DEFAULT NOW(),
    updated_at           TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ogsm_actions (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    strategy_id          UUID NOT NULL REFERENCES ogsm_strategies(id),
    title                VARCHAR(200) NOT NULL,
    what                 TEXT NOT NULL,
    who                  UUID NOT NULL,
    deadline             DATE NOT NULL,
    expected_result      TEXT NOT NULL,
    actual_result        TEXT,
    kpi_impact           JSONB,
    status               VARCHAR(20) DEFAULT 'pending',
    priority             VARCHAR(10) DEFAULT 'medium',
    created_from         VARCHAR(20) DEFAULT 'manual',
    analysis_session_id  UUID,
    created_by           UUID NOT NULL,
    created_at           TIMESTAMPTZ DEFAULT NOW(),
    updated_at           TIMESTAMPTZ DEFAULT NOW(),
    completed_at         TIMESTAMPTZ
);

CREATE TABLE analysis_sessions (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    triggered_by         VARCHAR(256) NOT NULL,
    goal_id              UUID REFERENCES ogsm_goals(id),
    status               VARCHAR(30) DEFAULT 'draft',
    step1_data           JSONB,
    step2_data           JSONB,
    step3_data           JSONB,
    step4_data           JSONB,
    actions_generated    UUID[] DEFAULT '{}',
    created_by           UUID NOT NULL,
    created_at           TIMESTAMPTZ DEFAULT NOW(),
    updated_at           TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ogsm_reviews (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period                      VARCHAR(20) NOT NULL,
    summary                     TEXT NOT NULL,
    lessons_learned             TEXT,
    next_period_adjustments     TEXT,
    created_by                  UUID NOT NULL,
    created_at                  TIMESTAMPTZ DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_ogsm_goals_objective ON ogsm_goals(objective_id);
CREATE INDEX idx_ogsm_goals_kpi ON ogsm_goals(kpi_metric);
CREATE INDEX idx_ogsm_actions_who ON ogsm_actions(who);
CREATE INDEX idx_ogsm_actions_status ON ogsm_actions(status);
CREATE INDEX idx_ogsm_actions_deadline ON ogsm_actions(deadline);
CREATE INDEX idx_analysis_sessions_goal ON analysis_sessions(goal_id);
```

---

## 状态转换图

### OgsmAction 状态机

```
                  ┌─────────────────────────────┐
                  │         PENDING              │
                  └──────────────┬──────────────┘
                                 │ 用户开始执行
                  ┌──────────────▼──────────────┐
                  │       IN_PROGRESS            │
                  └──────┬───────────────┬───────┘
                         │ 标记完成       │ 截止日期过
              ┌──────────▼──────┐  ┌─────▼──────────┐
              │      DONE       │  │    OVERDUE      │
              └─────────────────┘  └────────────────┘
                                         │ 补充完成
                                   ┌─────▼──────────┐
                                   │      DONE       │
                                   └────────────────┘
```

### AnalysisSession 状态机

```
DRAFT → STEP1_COMPLETE → STEP2_COMPLETE → STEP3_COMPLETE → ACTIONS_GENERATED
  ↑______________________________________________|
  （任意步骤可退出保存草稿，下次从当前步骤继续）
```
