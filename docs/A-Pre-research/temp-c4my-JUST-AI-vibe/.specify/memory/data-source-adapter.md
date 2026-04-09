# DataSourceAdapter 接口规范

> 所有数据源接入必须实现本文件定义的抽象接口。
> 新餐饮业态或新数据源类型通过实现此接口扩展，禁止修改核心业务层代码。

---

## 设计原则

- **开闭原则**：对扩展开放，对修改关闭。新数据源 = 新 Adapter 类，不改已有代码
- **契约优先**：Adapter 只负责"格式转换 + 初步校验"，业务规则由 domain 层处理
- **幂等性**：同一批数据多次导入结果一致（依赖 `import_mode` 参数）
- **可观测性**：每个 Adapter 必须输出结构化日志，便于排查数据问题

---

## Python 抽象基类定义

```python
# app/infrastructure/adapters/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from decimal import Decimal
from datetime import date, datetime
from enum import Enum
from typing import Any, AsyncIterator

from app.domain.models import DataCategory, DataRecord, TenantContext


class SourceType(str, Enum):
    CSV_UPLOAD   = "csv_upload"
    EXCEL_UPLOAD = "excel_upload"
    API_PUSH     = "api_push"
    DB_DIRECT    = "db_direct"
    MANUAL_FORM  = "manual_form"


class ImportMode(str, Enum):
    APPEND            = "append"
    UPSERT            = "upsert"
    REPLACE_DATE_RANGE = "replace_date_range"


@dataclass
class AdapterConfig:
    """适配器初始化配置，由租户配置层注入"""
    tenant_id: str
    store_id: str
    source_type: SourceType
    schema_version: str = "1.0"
    import_mode: ImportMode = ImportMode.APPEND
    # 业态自定义字段映射：源字段名 → 标准字段名
    field_mapping: dict[str, str] = field(default_factory=dict)
    # 业态自定义校验规则覆盖
    validation_overrides: dict[str, Any] = field(default_factory=dict)
    # 扩展配置（各 Adapter 自定义）
    extra: dict[str, Any] = field(default_factory=dict)


@dataclass
class ParsedRecord:
    """Adapter 解析后的单条标准化记录（进入业务层前的中间态）"""
    record_date: date
    amount: Decimal
    currency: str = "CNY"
    # 可选业务字段
    dish_id: str | None = None
    ingredient_id: str | None = None
    supplier_id: str | None = None
    shift: str | None = None
    channel: str | None = None
    operator_id: str | None = None
    # 业态自定义扩展字段
    ext: dict[str, Any] = field(default_factory=dict)
    # 原始行号/行内容，用于错误定位
    source_row_index: int | None = None
    source_raw: dict[str, Any] = field(default_factory=dict)


@dataclass
class ValidationIssue:
    severity: str          # "critical" | "warning" | "info"
    rule: str              # 触发的规则名
    message: str
    row_index: int | None = None
    field_name: str | None = None


@dataclass
class AdapterResult:
    """Adapter 处理结果，传递给 data-import Skill"""
    success: bool
    records: list[ParsedRecord] = field(default_factory=list)
    issues: list[ValidationIssue] = field(default_factory=list)
    # 统计
    total_rows: int = 0
    parsed_count: int = 0
    skipped_count: int = 0
    # 元数据
    source_type: SourceType = SourceType.CSV_UPLOAD
    schema_version: str = "1.0"
    adapter_name: str = ""
    processed_at: datetime = field(default_factory=datetime.utcnow)

    @property
    def has_critical_issues(self) -> bool:
        return any(i.severity == "critical" for i in self.issues)

    @property
    def quality_score(self) -> float:
        """简单质量评分：无 critical=100，有 warning 按比例扣分"""
        if self.has_critical_issues:
            return 0.0
        if self.total_rows == 0:
            return 100.0
        warning_count = sum(1 for i in self.issues if i.severity == "warning")
        return max(0.0, 100.0 - (warning_count / self.total_rows) * 100)


class DataSourceAdapter(ABC):
    """
    所有数据源适配器的抽象基类。

    实现者职责：
      1. 解析原始数据（文件/API响应/DB查询结果）
      2. 字段映射（源字段 → 标准字段）
      3. 格式校验（类型、必填、范围）
      4. 返回 AdapterResult（不写数据库，不调业务逻辑）

    实现者禁止：
      - 直接操作数据库
      - 调用其他 Skill
      - 持有请求级别以外的状态
    """

    def __init__(self, config: AdapterConfig):
        self.config = config

    @property
    @abstractmethod
    def adapter_name(self) -> str:
        """适配器唯一名称，用于日志和错误追踪"""
        ...

    @property
    @abstractmethod
    def supported_categories(self) -> list[DataCategory]:
        """本适配器支持的数据类别"""
        ...

    @abstractmethod
    async def validate_source(self, source: Any) -> list[ValidationIssue]:
        """
        预校验数据源（不解析全量数据）。
        用于快速失败：文件格式错误、必要列缺失、编码问题等。

        Args:
            source: 原始数据源（文件路径/字节流/API响应体/DB连接）

        Returns:
            校验问题列表，有 critical 级别则后续 parse 不会执行
        """
        ...

    @abstractmethod
    async def parse(self, source: Any, category: DataCategory) -> AdapterResult:
        """
        解析数据源，返回标准化记录列表。

        Args:
            source:   原始数据源
            category: 数据类别（sales/inventory/cost/staff/customer）

        Returns:
            AdapterResult，包含解析结果和所有校验问题
        """
        ...

    @abstractmethod
    async def stream_parse(
        self, source: Any, category: DataCategory, chunk_size: int = 1000
    ) -> AsyncIterator[AdapterResult]:
        """
        流式解析（大文件场景，> 10MB 或 > 5万行时使用）。
        每次 yield 一个 chunk 的 AdapterResult。

        实现要求：
          - 每个 chunk 独立可用，不依赖其他 chunk
          - 内存占用不超过 chunk_size * 单行估算大小
        """
        ...

    def map_fields(self, raw_row: dict[str, Any]) -> dict[str, Any]:
        """
        应用 config.field_mapping 进行字段重命名。
        默认实现：按 mapping 字典替换 key，未配置的字段原样保留。
        子类可覆盖以实现更复杂的映射逻辑。
        """
        if not self.config.field_mapping:
            return raw_row
        mapped = {}
        for src_key, value in raw_row.items():
            target_key = self.config.field_mapping.get(src_key, src_key)
            mapped[target_key] = value
        return mapped

    def get_validation_rule(self, rule_name: str, default: Any = None) -> Any:
        """从 validation_overrides 获取规则，不存在则返回系统默认值"""
        return self.config.validation_overrides.get(rule_name, default)
```

---

## 内置 Adapter 实现清单

### 1. CsvFileAdapter

```python
# app/infrastructure/adapters/csv_adapter.py

import csv
import io
from decimal import Decimal, InvalidOperation
from datetime import date
from typing import Any, AsyncIterator

from app.infrastructure.adapters.base import (
    DataSourceAdapter, AdapterConfig, AdapterResult,
    ParsedRecord, ValidationIssue, SourceType
)
from app.domain.models import DataCategory

REQUIRED_COLUMNS = {
    DataCategory.SALES:     {"record_date", "amount"},
    DataCategory.INVENTORY: {"record_date", "ingredient_id", "amount"},
    DataCategory.COST:      {"record_date", "amount"},
    DataCategory.STAFF:     {"record_date", "operator_id", "amount"},
    DataCategory.CUSTOMER:  {"record_date", "amount"},
}


class CsvFileAdapter(DataSourceAdapter):
    """
    CSV 文件适配器。
    source 参数：bytes（文件内容）或 str（文件路径）
    """

    @property
    def adapter_name(self) -> str:
        return "csv_file_v1"

    @property
    def supported_categories(self) -> list[DataCategory]:
        return list(DataCategory)

    async def validate_source(self, source: Any) -> list[ValidationIssue]:
        issues = []
        try:
            content = source if isinstance(source, str) else source.decode("utf-8-sig")
            reader = csv.DictReader(io.StringIO(content))
            headers = set(reader.fieldnames or [])
        except UnicodeDecodeError:
            issues.append(ValidationIssue(
                severity="critical",
                rule="encoding_check",
                message="文件编码不支持，请使用 UTF-8 或 UTF-8-BOM 格式"
            ))
            return issues

        # 检查必要列（通过 field_mapping 转换后的列名）
        mapped_headers = {self.config.field_mapping.get(h, h) for h in headers}
        # 此处 category 在 validate_source 阶段未知，检查通用必填列
        if "record_date" not in mapped_headers:
            issues.append(ValidationIssue(
                severity="critical",
                rule="required_column",
                message="缺少必填列：record_date（或通过 field_mapping 映射后的等效列）"
            ))
        if "amount" not in mapped_headers:
            issues.append(ValidationIssue(
                severity="critical",
                rule="required_column",
                message="缺少必填列：amount"
            ))
        return issues

    async def parse(self, source: Any, category: DataCategory) -> AdapterResult:
        content = source if isinstance(source, str) else source.decode("utf-8-sig")
        reader = csv.DictReader(io.StringIO(content))
        records = []
        issues = []
        total = 0

        for idx, raw_row in enumerate(reader):
            total += 1
            mapped = self.map_fields(dict(raw_row))
            parsed, row_issues = self._parse_row(mapped, idx + 2)  # +2: header行=1
            issues.extend(row_issues)
            if parsed:
                records.append(parsed)

        return AdapterResult(
            success=True,
            records=records,
            issues=issues,
            total_rows=total,
            parsed_count=len(records),
            skipped_count=total - len(records),
            source_type=SourceType.CSV_UPLOAD,
            adapter_name=self.adapter_name,
        )

    async def stream_parse(
        self, source: Any, category: DataCategory, chunk_size: int = 1000
    ) -> AsyncIterator[AdapterResult]:
        content = source if isinstance(source, str) else source.decode("utf-8-sig")
        reader = csv.DictReader(io.StringIO(content))
        chunk_records = []
        chunk_issues = []
        chunk_total = 0
        row_offset = 0

        for idx, raw_row in enumerate(reader):
            chunk_total += 1
            mapped = self.map_fields(dict(raw_row))
            parsed, row_issues = self._parse_row(mapped, idx + 2)
            chunk_issues.extend(row_issues)
            if parsed:
                chunk_records.append(parsed)

            if chunk_total >= chunk_size:
                yield AdapterResult(
                    success=True,
                    records=chunk_records,
                    issues=chunk_issues,
                    total_rows=chunk_total,
                    parsed_count=len(chunk_records),
                    skipped_count=chunk_total - len(chunk_records),
                    source_type=SourceType.CSV_UPLOAD,
                    adapter_name=self.adapter_name,
                )
                chunk_records, chunk_issues, chunk_total = [], [], 0
                row_offset += chunk_size

        if chunk_total > 0:
            yield AdapterResult(
                success=True,
                records=chunk_records,
                issues=chunk_issues,
                total_rows=chunk_total,
                parsed_count=len(chunk_records),
                skipped_count=chunk_total - len(chunk_records),
                source_type=SourceType.CSV_UPLOAD,
                adapter_name=self.adapter_name,
            )

    def _parse_row(
        self, row: dict[str, Any], row_num: int
    ) -> tuple[ParsedRecord | None, list[ValidationIssue]]:
        issues = []

        # record_date 解析
        try:
            record_date = date.fromisoformat(str(row.get("record_date", "")).strip())
        except ValueError:
            issues.append(ValidationIssue(
                severity="critical", rule="date_format",
                message=f"record_date 格式错误，期望 YYYY-MM-DD，实际：{row.get('record_date')}",
                row_index=row_num, field_name="record_date"
            ))
            return None, issues

        # amount 解析
        try:
            amount = Decimal(str(row.get("amount", "")).strip())
            min_amount = Decimal(self.get_validation_rule("amount_min", "-999999999.99"))
            max_amount = Decimal(self.get_validation_rule("amount_max", "999999999.99"))
            if not (min_amount <= amount <= max_amount):
                issues.append(ValidationIssue(
                    severity="warning", rule="amount_range",
                    message=f"amount 超出合理范围 [{min_amount}, {max_amount}]：{amount}",
                    row_index=row_num, field_name="amount"
                ))
        except InvalidOperation:
            issues.append(ValidationIssue(
                severity="critical", rule="amount_format",
                message=f"amount 不是有效数字：{row.get('amount')}",
                row_index=row_num, field_name="amount"
            ))
            return None, issues

        # 提取扩展字段（非标准字段全部放入 ext）
        standard_fields = {
            "record_date", "amount", "currency", "dish_id", "ingredient_id",
            "supplier_id", "shift", "channel", "operator_id"
        }
        ext = {k: v for k, v in row.items() if k not in standard_fields and v not in ("", None)}

        return ParsedRecord(
            record_date=record_date,
            amount=amount,
            currency=str(row.get("currency", "CNY")).strip().upper() or "CNY",
            dish_id=row.get("dish_id") or None,
            ingredient_id=row.get("ingredient_id") or None,
            supplier_id=row.get("supplier_id") or None,
            shift=row.get("shift") or None,
            channel=row.get("channel") or None,
            operator_id=row.get("operator_id") or None,
            ext=ext,
            source_row_index=row_num,
            source_raw=row,
        ), issues
```

### 2. ExcelFileAdapter（继承 CsvFileAdapter 逻辑）

```python
# app/infrastructure/adapters/excel_adapter.py
# 依赖：openpyxl

import io
from typing import Any, AsyncIterator
import openpyxl

from app.infrastructure.adapters.csv_adapter import CsvFileAdapter
from app.infrastructure.adapters.base import AdapterResult, ValidationIssue, SourceType
from app.domain.models import DataCategory


class ExcelFileAdapter(CsvFileAdapter):
    """
    Excel (.xlsx) 文件适配器。
    复用 CsvFileAdapter 的行解析逻辑，仅覆盖文件读取部分。
    """

    @property
    def adapter_name(self) -> str:
        return "excel_file_v1"

    async def validate_source(self, source: Any) -> list[ValidationIssue]:
        issues = []
        try:
            wb = openpyxl.load_workbook(io.BytesIO(source), read_only=True, data_only=True)
            ws = wb.active
            if ws is None or ws.max_row < 2:
                issues.append(ValidationIssue(
                    severity="critical", rule="empty_sheet",
                    message="Excel 文件为空或只有表头行"
                ))
        except Exception as e:
            issues.append(ValidationIssue(
                severity="critical", rule="file_format",
                message=f"无法解析 Excel 文件：{e}"
            ))
        return issues

    def _excel_to_rows(self, source: bytes) -> list[dict[str, Any]]:
        wb = openpyxl.load_workbook(io.BytesIO(source), read_only=True, data_only=True)
        ws = wb.active
        rows = list(ws.iter_rows(values_only=True))
        if not rows:
            return []
        headers = [str(h).strip() if h is not None else f"col_{i}" for i, h in enumerate(rows[0])]
        return [dict(zip(headers, row)) for row in rows[1:]]

    async def parse(self, source: Any, category: DataCategory) -> AdapterResult:
        raw_rows = self._excel_to_rows(source)
        records, issues = [], []
        for idx, raw_row in enumerate(raw_rows):
            mapped = self.map_fields({k: (str(v) if v is not None else "") for k, v in raw_row.items()})
            parsed, row_issues = self._parse_row(mapped, idx + 2)
            issues.extend(row_issues)
            if parsed:
                records.append(parsed)
        return AdapterResult(
            success=True, records=records, issues=issues,
            total_rows=len(raw_rows), parsed_count=len(records),
            skipped_count=len(raw_rows) - len(records),
            source_type=SourceType.EXCEL_UPLOAD, adapter_name=self.adapter_name,
        )

    async def stream_parse(
        self, source: Any, category: DataCategory, chunk_size: int = 1000
    ) -> AsyncIterator[AdapterResult]:
        raw_rows = self._excel_to_rows(source)
        for i in range(0, len(raw_rows), chunk_size):
            chunk = raw_rows[i:i + chunk_size]
            records, issues = [], []
            for idx, raw_row in enumerate(chunk):
                mapped = self.map_fields({k: (str(v) if v is not None else "") for k, v in raw_row.items()})
                parsed, row_issues = self._parse_row(mapped, i + idx + 2)
                issues.extend(row_issues)
                if parsed:
                    records.append(parsed)
            yield AdapterResult(
                success=True, records=records, issues=issues,
                total_rows=len(chunk), parsed_count=len(records),
                skipped_count=len(chunk) - len(records),
                source_type=SourceType.EXCEL_UPLOAD, adapter_name=self.adapter_name,
            )
```

### 3. ApiPushAdapter（接收外部系统推送）

```python
# app/infrastructure/adapters/api_push_adapter.py

from typing import Any, AsyncIterator
from app.infrastructure.adapters.base import (
    DataSourceAdapter, AdapterResult, ParsedRecord,
    ValidationIssue, SourceType
)
from app.domain.models import DataCategory
from decimal import Decimal
from datetime import date


class ApiPushAdapter(DataSourceAdapter):
    """
    API 推送适配器。
    source 参数：已解析的 JSON dict（FastAPI 请求体）
    适用于 POS 系统、ERP、第三方平台的实时数据推送。
    """

    @property
    def adapter_name(self) -> str:
        return "api_push_v1"

    @property
    def supported_categories(self) -> list[DataCategory]:
        return list(DataCategory)

    async def validate_source(self, source: Any) -> list[ValidationIssue]:
        issues = []
        if not isinstance(source, dict):
            issues.append(ValidationIssue(
                severity="critical", rule="payload_type",
                message="API 推送 payload 必须是 JSON 对象"
            ))
            return issues
        if "records" not in source:
            issues.append(ValidationIssue(
                severity="critical", rule="required_field",
                message="payload 缺少 records 字段"
            ))
        elif not isinstance(source["records"], list) or len(source["records"]) == 0:
            issues.append(ValidationIssue(
                severity="critical", rule="records_empty",
                message="records 不能为空数组"
            ))
        return issues

    async def parse(self, source: Any, category: DataCategory) -> AdapterResult:
        raw_records = source.get("records", [])
        records, issues = [], []
        for idx, raw in enumerate(raw_records):
            mapped = self.map_fields(raw)
            try:
                parsed = ParsedRecord(
                    record_date=date.fromisoformat(str(mapped["record_date"])),
                    amount=Decimal(str(mapped["amount"])),
                    currency=str(mapped.get("currency", "CNY")).upper(),
                    dish_id=mapped.get("dish_id"),
                    ingredient_id=mapped.get("ingredient_id"),
                    supplier_id=mapped.get("supplier_id"),
                    shift=mapped.get("shift"),
                    channel=mapped.get("channel"),
                    operator_id=mapped.get("operator_id"),
                    ext=mapped.get("ext", {}),
                    source_row_index=idx,
                    source_raw=raw,
                )
                records.append(parsed)
            except (KeyError, ValueError) as e:
                issues.append(ValidationIssue(
                    severity="critical", rule="parse_error",
                    message=str(e), row_index=idx
                ))
        return AdapterResult(
            success=True, records=records, issues=issues,
            total_rows=len(raw_records), parsed_count=len(records),
            skipped_count=len(raw_records) - len(records),
            source_type=SourceType.API_PUSH, adapter_name=self.adapter_name,
        )

    async def stream_parse(
        self, source: Any, category: DataCategory, chunk_size: int = 1000
    ) -> AsyncIterator[AdapterResult]:
        # API 推送数据量通常较小，直接复用 parse
        yield await self.parse(source, category)
```

---

## Adapter 注册与工厂

```python
# app/infrastructure/adapters/registry.py

from app.infrastructure.adapters.base import DataSourceAdapter, AdapterConfig, SourceType
from app.infrastructure.adapters.csv_adapter import CsvFileAdapter
from app.infrastructure.adapters.excel_adapter import ExcelFileAdapter
from app.infrastructure.adapters.api_push_adapter import ApiPushAdapter

_REGISTRY: dict[SourceType, type[DataSourceAdapter]] = {
    SourceType.CSV_UPLOAD:   CsvFileAdapter,
    SourceType.EXCEL_UPLOAD: ExcelFileAdapter,
    SourceType.API_PUSH:     ApiPushAdapter,
    # 新增适配器在此注册，无需修改其他代码
}


def get_adapter(config: AdapterConfig) -> DataSourceAdapter:
    """
    工厂函数：根据 source_type 返回对应 Adapter 实例。
    新业态只需：1) 实现 DataSourceAdapter 子类  2) 在此注册
    """
    adapter_class = _REGISTRY.get(config.source_type)
    if adapter_class is None:
        raise ValueError(
            f"未注册的数据源类型：{config.source_type}。"
            f"已支持：{list(_REGISTRY.keys())}"
        )
    return adapter_class(config)
```

---

## 新业态扩展示例（奶茶店）

```python
# app/infrastructure/adapters/bubble_tea_adapter.py
# 示例：奶茶连锁特有字段（杯型、温度、甜度、小程序订单）

from app.infrastructure.adapters.csv_adapter import CsvFileAdapter
from app.infrastructure.adapters.base import ParsedRecord, ValidationIssue
from typing import Any


class BubbleTeaAdapter(CsvFileAdapter):
    """
    奶茶业态适配器。
    扩展字段：cup_size / temperature / sweetness / mini_program_order_id
    这些字段通过 ext 字典传递，不修改 ParsedRecord 基础结构。
    """

    @property
    def adapter_name(self) -> str:
        return "bubble_tea_v1"

    def _parse_row(
        self, row: dict[str, Any], row_num: int
    ) -> tuple[ParsedRecord | None, list[ValidationIssue]]:
        parsed, issues = super()._parse_row(row, row_num)
        if parsed is None:
            return None, issues

        # 奶茶业态特有校验
        cup_size = row.get("cup_size", "").strip()
        if cup_size and cup_size not in ("small", "medium", "large"):
            issues.append(ValidationIssue(
                severity="warning", rule="cup_size_enum",
                message=f"cup_size 值不在枚举范围内：{cup_size}",
                row_index=row_num, field_name="cup_size"
            ))

        # 扩展字段写入 ext（不改变基础结构）
        parsed.ext.update({
            k: row[k] for k in ("cup_size", "temperature", "sweetness", "mini_program_order_id")
            if k in row and row[k] not in ("", None)
        })
        return parsed, issues

# 注册到 registry（在 registry.py 中添加一行）：
# SourceType.CSV_UPLOAD: BubbleTeaAdapter  ← 租户级覆盖，通过 tenant_config 动态路由
```

---

## 测试要求

每个 Adapter 实现必须包含以下测试用例：

```python
# tests/infrastructure/adapters/test_csv_adapter.py

import pytest
from decimal import Decimal
from datetime import date

# 必须覆盖的测试场景：
# 1. 正常数据解析（全字段）
# 2. 最小必填字段解析
# 3. record_date 格式错误 → critical issue
# 4. amount 非数字 → critical issue
# 5. amount 超出范围 → warning issue（不阻断）
# 6. 编码错误（GBK文件）→ critical issue
# 7. 空文件 → critical issue
# 8. field_mapping 字段重命名
# 9. ext 扩展字段正确提取
# 10. stream_parse chunk 边界（行数恰好整除/不整除 chunk_size）
# 11. 幂等性：同一文件解析两次结果相同
```

---

## 文件目录结构

```
app/infrastructure/adapters/
├── __init__.py
├── base.py                  # 抽象基类（本文件定义）
├── registry.py              # 工厂注册表
├── csv_adapter.py           # CSV 适配器
├── excel_adapter.py         # Excel 适配器
├── api_push_adapter.py      # API 推送适配器
└── extensions/              # 业态扩展适配器
    ├── bubble_tea_adapter.py
    ├── hotpot_adapter.py
    └── fastfood_adapter.py

tests/infrastructure/adapters/
├── test_csv_adapter.py
├── test_excel_adapter.py
├── test_api_push_adapter.py
└── fixtures/
    ├── sample_sales.csv
    ├── sample_inventory.xlsx
    └── sample_api_push.json
```

---

**Version**: 1.0.0 | **Last Updated**: 2026-04-05
