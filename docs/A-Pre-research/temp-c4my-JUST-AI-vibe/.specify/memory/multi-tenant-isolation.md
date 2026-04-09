# 多餐饮企业数据隔离技术方案

> 适用场景：BI 中台工具同时服务多家餐饮企业（连锁品牌、加盟商、独立餐厅）
> 隔离目标：数据安全隔离 + 配置独立 + 性能互不影响

---

## 隔离模型选择

### 三种模式对比

| 模式 | 描述 | 适用场景 | 本项目选择 |
|---|---|---|---|
| 独立数据库（Silo） | 每个租户独立 DB 实例 | 高安全要求、大客户 | 可选（企业版） |
| 独立 Schema（Bridge） | 同一 DB，不同 Schema/前缀 | 中等规模、成本敏感 | **默认模式** |
| 共享表（Pool） | 所有租户共享表，`tenant_id` 区分 | 小微客户、SaaS 标准版 | 轻量模式 |

**本项目默认采用"独立 Schema + 共享基础设施"的 Bridge 模式**，兼顾隔离强度与运维成本。企业版客户可升级为 Silo 模式。

---

## 核心隔离层设计

### 1. 租户标识体系

```
TenantID 规则：
  - 格式：^[a-z][a-z0-9_]{2,31}$（小写字母开头，3-32位）
  - 示例：haidilao_001、mcdonald_cn、local_bbq_shop
  - 不可变：创建后不允许修改
  - 全局唯一：跨所有部署环境唯一

StoreID 规则：
  - 格式：^[a-zA-Z0-9_-]{1,64}$
  - 在 tenant 内唯一（不要求全局唯一）
  - 示例：store_beijing_001、sz_nanshan_02

完整数据定位键：tenant_id + store_id + record_date + category
```

### 2. 数据库 Schema 隔离（Bridge 模式）

```sql
-- 每个租户独立 Schema
CREATE SCHEMA tenant_haidilao_001;
CREATE SCHEMA tenant_mcdonald_cn;

-- 每个 Schema 内结构相同，通过 Alembic 多租户迁移管理
-- 表名示例：tenant_haidilao_001.sales_records
--           tenant_haidilao_001.inventory_records
--           tenant_haidilao_001.stores

-- 共享基础表（不含业务数据）放在 public schema
-- public.tenants          -- 租户注册信息
-- public.tenant_configs   -- 租户配置
-- public.audit_logs       -- 跨租户审计日志（仅系统管理员可读）
```

**Schema 命名规则：** `tenant_{tenant_id}`，由系统自动创建，禁止手动操作。

### 3. 应用层强制隔离（Row-Level Security 补充）

```python
# 所有数据库查询必须通过 TenantContext 注入
# 禁止在业务代码中直接拼接 tenant_id 字符串

class TenantContext:
    """请求级别的租户上下文，通过依赖注入传递"""
    tenant_id: str
    store_ids: list[str]  # 当前用户有权访问的门店列表
    role: TenantRole       # owner / manager / staff / readonly

# FastAPI 依赖注入示例
async def get_tenant_context(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> TenantContext:
    payload = verify_jwt(token)
    # 从 JWT 中提取 tenant_id，不信任请求体中的 tenant_id
    return TenantContext(
        tenant_id=payload["tenant_id"],
        store_ids=payload["store_ids"],
        role=TenantRole(payload["role"])
    )

# Repository 层强制注入 schema
class SalesRepository:
    def __init__(self, db: AsyncSession, ctx: TenantContext):
        self._db = db
        self._schema = f"tenant_{ctx.tenant_id}"

    async def get_records(self, date_range: DateRange) -> list[SalesRecord]:
        # 自动使用租户 schema，无需业务层关心
        return await self._db.execute(
            text(f"SELECT * FROM {self._schema}.sales_records WHERE ...")
        )
```

### 4. PostgreSQL Row-Level Security（双重保险）

```sql
-- 在共享表模式下额外启用 RLS
ALTER TABLE sales_records ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON sales_records
    USING (tenant_id = current_setting('app.current_tenant_id'));

-- 应用连接时设置
SET app.current_tenant_id = 'haidilao_001';
```

---

## 租户配置独立化

### 配置分层

```
Level 1 - 系统默认配置（代码内置，所有租户共享）
Level 2 - 租户级配置（public.tenant_configs，覆盖系统默认）
Level 3 - 门店级配置（tenant_{id}.store_configs，覆盖租户配置）
Level 4 - 用户级偏好（tenant_{id}.user_preferences，仅影响 UI）
```

### 租户配置表结构

```sql
-- public.tenant_configs
CREATE TABLE public.tenant_configs (
    tenant_id       VARCHAR(32) PRIMARY KEY REFERENCES public.tenants(id),
    business_type   VARCHAR(32) NOT NULL,  -- 'hotpot'/'fastfood'/'cafe'/'bbq'/'other'
    currency        CHAR(3) DEFAULT 'CNY',
    timezone        VARCHAR(64) DEFAULT 'Asia/Shanghai',
    fiscal_year_start_month INT DEFAULT 1,
    data_retention_days INT DEFAULT 730,   -- 数据保留天数
    quality_score_threshold DECIMAL(5,2) DEFAULT 80.0,
    custom_fields   JSONB DEFAULT '{}',    -- 业态自定义字段定义
    feature_flags   JSONB DEFAULT '{}',    -- 功能开关
    ai_model_env_var VARCHAR(128),         -- AI API Key 环境变量名（不存 Key 本身）
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 业态自定义字段示例

```json
// 火锅业态（hotpot）的 custom_fields
{
  "custom_fields": {
    "table_turnover_rate": { "type": "number", "label": "翻台率", "unit": "次/天" },
    "pot_type": { "type": "enum", "label": "锅底类型", "values": ["麻辣", "清汤", "鸳鸯"] },
    "avg_consumption_per_person": { "type": "decimal", "label": "人均消费", "unit": "元" }
  }
}

// 快餐业态（fastfood）的 custom_fields
{
  "custom_fields": {
    "drive_through_orders": { "type": "integer", "label": "得来速订单数" },
    "avg_service_time_seconds": { "type": "integer", "label": "平均服务时长", "unit": "秒" }
  }
}
```

---

## 数据访问权限矩阵

```
角色              | 本租户数据 | 跨门店 | 跨租户 | 系统配置
------------------|-----------|--------|--------|--------
tenant_owner      | 全部读写   | 全部   | 禁止   | 只读
store_manager     | 本店读写   | 禁止   | 禁止   | 禁止
staff             | 本店写入   | 禁止   | 禁止   | 禁止
readonly_analyst  | 本租户只读 | 全部   | 禁止   | 禁止
system_admin      | 全部只读   | 全部   | 全部   | 全部读写
```

**JWT Payload 结构：**

```json
{
  "sub": "user_uuid",
  "tenant_id": "haidilao_001",
  "store_ids": ["store_bj_001", "store_bj_002"],
  "role": "store_manager",
  "iat": 1743840000,
  "exp": 1743926400
}
```

---

## 租户生命周期管理

### 创建流程

```
1. POST /api/v1/admin/tenants  →  创建 public.tenants 记录
2. 系统自动执行：
   a. CREATE SCHEMA tenant_{id}
   b. 运行 Alembic migration 在新 schema 建表
   c. 插入默认 tenant_configs
   d. 创建 tenant_owner 账号
3. 返回租户初始化完成状态
```

### 数据迁移（租户间）

```
禁止直接跨租户复制数据。
如需迁移（如加盟商独立为品牌方），必须：
1. 导出源租户数据（export-packager skill）
2. 人工审核导出内容
3. 通过 data-import skill 导入目标租户
4. 审计日志记录全程
```

### 租户注销

```
1. 标记为 status='pending_deletion'（软删除，保留 30 天）
2. 30 天后：
   a. 导出全量数据归档（加密存储 7 年，合规要求）
   b. DROP SCHEMA tenant_{id} CASCADE
   c. 删除 public.tenants 记录
3. 不可逆操作，需双人审批
```

---

## 性能隔离策略

### 防止"嘈杂邻居"问题

```python
# 每个租户的异步任务队列独立限速
CELERY_TASK_RATE_LIMITS = {
    "data_import": "100/hour",      # 每租户每小时最多 100 次导入
    "report_generate": "500/hour",  # 每租户每小时最多 500 次报表
    "forecast": "50/hour"           # 预测任务计算密集，严格限速
}

# 数据库连接池按租户分配
# 标准租户：最大 5 个连接
# 企业租户：最大 20 个连接
# 系统保留：10 个连接（管理操作）
```

### 大租户数据分区

```sql
-- 超过 100 万行的租户，sales_records 按月分区
CREATE TABLE tenant_haidilao_001.sales_records (
    id          UUID DEFAULT gen_random_uuid(),
    record_date DATE NOT NULL,
    ...
) PARTITION BY RANGE (record_date);

-- 自动创建月度分区（由 Celery 定时任务管理）
CREATE TABLE tenant_haidilao_001.sales_records_2026_04
    PARTITION OF tenant_haidilao_001.sales_records
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
```

---

## 审计与合规

```sql
-- public.audit_logs（所有租户共享，仅系统管理员可读）
CREATE TABLE public.audit_logs (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_id   VARCHAR(32) NOT NULL,
    store_id    VARCHAR(64),
    user_id     UUID NOT NULL,
    action      VARCHAR(64) NOT NULL,  -- 'data_import'/'report_export'/'config_change'
    resource    VARCHAR(256),
    ip_address  INET,
    user_agent  TEXT,
    result      VARCHAR(16),           -- 'success'/'failure'/'blocked'
    detail      JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 保留策略：审计日志保留 7 年（合规要求）
-- 索引：(tenant_id, created_at) 复合索引
```

---

**Version**: 1.0.0 | **Last Updated**: 2026-04-05
