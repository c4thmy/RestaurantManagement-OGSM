# Skills JSON Schema 规范

> 所有 Skill 的输入/输出必须严格遵循本文件定义的 JSON Schema。
> Schema 版本变更需同步更新 `constitution.md` Skills 清单。

---

## 通用约定

- 所有时间字段使用 ISO8601 格式：`"2026-04-05T14:30:00+08:00"`
- 金额字段使用字符串 Decimal，避免浮点精度问题：`"1234.56"`
- 枚举值全部小写下划线：`"dine_in"` 而非 `"DineIn"`
- 错误响应统一结构：`{ "success": false, "error_code": "...", "message": "...", "details": [] }`
- 所有 Skill 响应必须包含 `skill_version` 和 `executed_at` 元数据字段

---

## Skill: `data-import`

### 输入 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "skill/data-import/input/v1",
  "type": "object",
  "required": ["tenant_id", "store_id", "source_type", "payload"],
  "properties": {
    "tenant_id": {
      "type": "string",
      "description": "餐饮企业租户唯一标识",
      "pattern": "^[a-zA-Z0-9_-]{3,64}$"
    },
    "store_id": {
      "type": "string",
      "description": "门店唯一标识，tenant 内唯一",
      "pattern": "^[a-zA-Z0-9_-]{1,64}$"
    },
    "source_type": {
      "type": "string",
      "enum": ["csv_upload", "excel_upload", "api_push", "db_direct", "manual_form"],
      "description": "数据来源类型"
    },
    "schema_version": {
      "type": "string",
      "default": "1.0",
      "description": "数据接口版本号，用于向后兼容路由"
    },
    "payload": {
      "type": "object",
      "required": ["category", "records"],
      "properties": {
        "category": {
          "type": "string",
          "enum": ["sales", "inventory", "cost", "staff", "customer"],
          "description": "数据类别"
        },
        "records": {
          "type": "array",
          "minItems": 1,
          "maxItems": 50000,
          "items": { "$ref": "#/definitions/DataRecord" }
        },
        "import_mode": {
          "type": "string",
          "enum": ["append", "upsert", "replace_date_range"],
          "default": "append",
          "description": "导入模式：追加/更新插入/替换日期范围"
        },
        "date_range": {
          "type": "object",
          "description": "replace_date_range 模式必填",
          "properties": {
            "start": { "type": "string", "format": "date" },
            "end": { "type": "string", "format": "date" }
          }
        }
      }
    },
    "options": {
      "type": "object",
      "properties": {
        "skip_quality_check": {
          "type": "boolean",
          "default": false,
          "description": "跳过质量检查（仅限管理员，生产环境禁用）"
        },
        "dry_run": {
          "type": "boolean",
          "default": false,
          "description": "试运行：校验但不写入"
        },
        "notify_on_complete": {
          "type": "boolean",
          "default": true
        }
      }
    }
  },
  "definitions": {
    "DataRecord": {
      "type": "object",
      "required": ["record_date", "amount"],
      "properties": {
        "record_date": { "type": "string", "format": "date" },
        "amount": { "type": "string", "pattern": "^-?[0-9]+(\\.[0-9]{1,2})?$" },
        "currency": { "type": "string", "default": "CNY", "pattern": "^[A-Z]{3}$" },
        "dish_id": { "type": "string" },
        "ingredient_id": { "type": "string" },
        "supplier_id": { "type": "string" },
        "shift": { "type": "string", "enum": ["morning", "afternoon", "evening", "all_day"] },
        "channel": { "type": "string", "enum": ["dine_in", "takeout", "delivery", "other"] },
        "operator_id": { "type": "string" },
        "ext": {
          "type": "object",
          "description": "业态自定义扩展字段，key-value 结构",
          "additionalProperties": { "type": ["string", "number", "boolean", "null"] }
        }
      }
    }
  }
}
```

### 输出 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "skill/data-import/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at", "summary"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string", "example": "1.0.0" },
    "executed_at": { "type": "string", "format": "date-time" },
    "import_id": {
      "type": "string",
      "description": "本次导入任务唯一 ID，用于追踪和回滚"
    },
    "summary": {
      "type": "object",
      "required": ["total", "imported", "skipped", "failed"],
      "properties": {
        "total": { "type": "integer" },
        "imported": { "type": "integer" },
        "skipped": { "type": "integer" },
        "failed": { "type": "integer" },
        "quality_score": {
          "type": "number",
          "minimum": 0,
          "maximum": 100,
          "description": "数据质量综合评分"
        }
      }
    },
    "quality_report": { "$ref": "#/definitions/QualityReport" },
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "row_index": { "type": "integer" },
          "field": { "type": "string" },
          "error_code": { "type": "string" },
          "message": { "type": "string" }
        }
      }
    }
  },
  "definitions": {
    "QualityReport": {
      "type": "object",
      "properties": {
        "completeness": { "type": "number", "description": "必填字段完整率 0-100" },
        "consistency": { "type": "number", "description": "数据一致性评分 0-100" },
        "anomaly_count": { "type": "integer", "description": "检测到的异常记录数" },
        "duplicate_count": { "type": "integer" },
        "warnings": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    }
  }
}
```

---

## Skill: `data-quality-check`

### 输入 Schema

```json
{
  "$id": "skill/data-quality-check/input/v1",
  "type": "object",
  "required": ["tenant_id", "store_id", "category", "records"],
  "properties": {
    "tenant_id": { "type": "string" },
    "store_id": { "type": "string" },
    "category": { "type": "string", "enum": ["sales", "inventory", "cost", "staff", "customer"] },
    "records": { "type": "array", "items": { "$ref": "skill/data-import/input/v1#/definitions/DataRecord" } },
    "rules": {
      "type": "object",
      "description": "自定义校验规则覆盖（可选，不填则使用租户默认规则）",
      "properties": {
        "amount_range": {
          "type": "object",
          "properties": {
            "min": { "type": "string" },
            "max": { "type": "string" }
          }
        },
        "required_fields": { "type": "array", "items": { "type": "string" } },
        "date_range": {
          "type": "object",
          "properties": {
            "not_before": { "type": "string", "format": "date" },
            "not_after": { "type": "string", "format": "date" }
          }
        }
      }
    }
  }
}
```

### 输出 Schema

```json
{
  "$id": "skill/data-quality-check/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at", "passed", "score", "issues"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string" },
    "executed_at": { "type": "string", "format": "date-time" },
    "passed": { "type": "boolean", "description": "是否通过质量门禁（score >= 阈值且无 critical 问题）" },
    "score": { "type": "number", "minimum": 0, "maximum": 100 },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["severity", "rule", "message", "affected_count"],
        "properties": {
          "severity": { "type": "string", "enum": ["critical", "warning", "info"] },
          "rule": { "type": "string", "description": "触发的规则名称" },
          "message": { "type": "string" },
          "affected_count": { "type": "integer" },
          "sample_rows": { "type": "array", "items": { "type": "integer" }, "maxItems": 5 }
        }
      }
    },
    "stats": {
      "type": "object",
      "properties": {
        "null_rate": { "type": "number" },
        "duplicate_rate": { "type": "number" },
        "outlier_rate": { "type": "number" },
        "format_error_rate": { "type": "number" }
      }
    }
  }
}
```

---

## Skill: `report-generator`

### 输入 Schema

```json
{
  "$id": "skill/report-generator/input/v1",
  "type": "object",
  "required": ["tenant_id", "store_ids", "report_type", "date_range"],
  "properties": {
    "tenant_id": { "type": "string" },
    "store_ids": {
      "type": "array",
      "items": { "type": "string" },
      "description": "支持多门店对比，传 ['*'] 表示全部门店"
    },
    "report_type": {
      "type": "string",
      "enum": [
        "sales_summary",
        "cost_analysis",
        "inventory_turnover",
        "waste_rate",
        "staff_efficiency",
        "customer_flow",
        "profit_loss"
      ]
    },
    "date_range": {
      "type": "object",
      "required": ["start", "end"],
      "properties": {
        "start": { "type": "string", "format": "date" },
        "end": { "type": "string", "format": "date" },
        "granularity": {
          "type": "string",
          "enum": ["day", "week", "month", "quarter"],
          "default": "day"
        }
      }
    },
    "filters": {
      "type": "object",
      "properties": {
        "channels": { "type": "array", "items": { "type": "string" } },
        "shifts": { "type": "array", "items": { "type": "string" } },
        "categories": { "type": "array", "items": { "type": "string" } }
      }
    },
    "output_format": {
      "type": "string",
      "enum": ["json", "excel", "pdf", "csv"],
      "default": "json"
    }
  }
}
```

### 输出 Schema

```json
{
  "$id": "skill/report-generator/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at", "report"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string" },
    "executed_at": { "type": "string", "format": "date-time" },
    "report": {
      "type": "object",
      "required": ["report_id", "report_type", "period", "metrics", "chart_config"],
      "properties": {
        "report_id": { "type": "string" },
        "report_type": { "type": "string" },
        "period": {
          "type": "object",
          "properties": {
            "start": { "type": "string", "format": "date" },
            "end": { "type": "string", "format": "date" },
            "granularity": { "type": "string" }
          }
        },
        "metrics": {
          "type": "array",
          "description": "核心指标列表",
          "items": {
            "type": "object",
            "required": ["key", "label", "value", "unit"],
            "properties": {
              "key": { "type": "string" },
              "label": { "type": "string" },
              "value": { "type": ["number", "string"] },
              "unit": { "type": "string", "description": "元/个/kg/%" },
              "trend": {
                "type": "object",
                "properties": {
                  "direction": { "type": "string", "enum": ["up", "down", "flat"] },
                  "change_rate": { "type": "number", "description": "环比变化率" },
                  "compare_period": { "type": "string" }
                }
              }
            }
          }
        },
        "series": {
          "type": "array",
          "description": "时间序列数据，供图表渲染",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "data": {
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "date": { "type": "string" },
                    "value": { "type": "number" }
                  }
                }
              }
            }
          }
        },
        "chart_config": {
          "type": "object",
          "description": "ECharts option 配置，前端直接消费",
          "properties": {
            "type": { "type": "string", "enum": ["line", "bar", "pie", "scatter", "heatmap"] },
            "option": { "type": "object", "description": "完整 ECharts option JSON" }
          }
        },
        "file_url": {
          "type": "string",
          "description": "非 json 格式时返回文件下载地址"
        }
      }
    }
  }
}
```

---

## Skill: `anomaly-detector`

### 输入 Schema

```json
{
  "$id": "skill/anomaly-detector/input/v1",
  "type": "object",
  "required": ["tenant_id", "store_id", "category", "detect_window"],
  "properties": {
    "tenant_id": { "type": "string" },
    "store_id": { "type": "string" },
    "category": { "type": "string", "enum": ["sales", "inventory", "cost", "staff", "customer"] },
    "detect_window": {
      "type": "object",
      "required": ["start", "end"],
      "properties": {
        "start": { "type": "string", "format": "date" },
        "end": { "type": "string", "format": "date" }
      }
    },
    "sensitivity": {
      "type": "string",
      "enum": ["low", "medium", "high"],
      "default": "medium",
      "description": "检测灵敏度，high 会产生更多 warning 级别告警"
    },
    "baseline_window_days": {
      "type": "integer",
      "default": 30,
      "description": "用于建立基线的历史天数"
    }
  }
}
```

### 输出 Schema

```json
{
  "$id": "skill/anomaly-detector/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at", "anomalies"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string" },
    "executed_at": { "type": "string", "format": "date-time" },
    "anomalies": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["anomaly_id", "severity", "detected_at", "metric", "confidence", "suggestion"],
        "properties": {
          "anomaly_id": { "type": "string" },
          "severity": { "type": "string", "enum": ["critical", "warning", "info"] },
          "detected_at": { "type": "string", "format": "date" },
          "metric": { "type": "string", "description": "异常指标名称" },
          "actual_value": { "type": "number" },
          "expected_range": {
            "type": "object",
            "properties": {
              "min": { "type": "number" },
              "max": { "type": "number" }
            }
          },
          "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 1,
            "description": "置信度 0-1"
          },
          "description": { "type": "string" },
          "suggestion": { "type": "string", "description": "AI 给出的处理建议" },
          "related_records": {
            "type": "array",
            "items": { "type": "string" },
            "description": "关联的数据记录 ID"
          }
        }
      }
    },
    "summary": {
      "type": "object",
      "properties": {
        "critical_count": { "type": "integer" },
        "warning_count": { "type": "integer" },
        "info_count": { "type": "integer" },
        "overall_health": { "type": "string", "enum": ["healthy", "degraded", "critical"] }
      }
    }
  }
}
```

---

## Skill: `forecast-engine`

### 输入 Schema

```json
{
  "$id": "skill/forecast-engine/input/v1",
  "type": "object",
  "required": ["tenant_id", "store_id", "metric", "forecast_days"],
  "properties": {
    "tenant_id": { "type": "string" },
    "store_id": { "type": "string" },
    "metric": {
      "type": "string",
      "enum": ["daily_revenue", "customer_count", "ingredient_consumption", "waste_rate", "staff_cost"],
      "description": "预测目标指标"
    },
    "forecast_days": {
      "type": "integer",
      "minimum": 1,
      "maximum": 90,
      "description": "预测未来天数"
    },
    "training_window_days": {
      "type": "integer",
      "default": 90,
      "description": "训练数据窗口天数"
    },
    "include_factors": {
      "type": "array",
      "items": { "type": "string", "enum": ["holiday", "weather", "promotion", "weekday_pattern"] },
      "description": "纳入预测的影响因子"
    }
  }
}
```

### 输出 Schema

```json
{
  "$id": "skill/forecast-engine/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at", "forecast"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string" },
    "executed_at": { "type": "string", "format": "date-time" },
    "forecast": {
      "type": "object",
      "required": ["metric", "model_info", "predictions"],
      "properties": {
        "metric": { "type": "string" },
        "model_info": {
          "type": "object",
          "properties": {
            "algorithm": { "type": "string", "description": "使用的算法（如 Prophet/ARIMA/LLM）" },
            "accuracy_mape": { "type": "number", "description": "历史回测 MAPE 误差率" },
            "training_samples": { "type": "integer" }
          }
        },
        "predictions": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["date", "value", "lower_bound", "upper_bound"],
            "properties": {
              "date": { "type": "string", "format": "date" },
              "value": { "type": "number", "description": "预测值" },
              "lower_bound": { "type": "number", "description": "置信区间下界（80%）" },
              "upper_bound": { "type": "number", "description": "置信区间上界（80%）" },
              "confidence": { "type": "number", "minimum": 0, "maximum": 1 }
            }
          }
        },
        "insights": {
          "type": "array",
          "items": { "type": "string" },
          "description": "AI 生成的预测洞察说明"
        }
      }
    }
  }
}
```

---

## Skill: `export-packager`

### 输入 Schema

```json
{
  "$id": "skill/export-packager/input/v1",
  "type": "object",
  "required": ["tenant_id", "export_type", "source"],
  "properties": {
    "tenant_id": { "type": "string" },
    "export_type": { "type": "string", "enum": ["excel", "pdf", "csv", "json"] },
    "source": {
      "type": "object",
      "description": "导出数据来源，二选一",
      "oneOf": [
        {
          "required": ["report_id"],
          "properties": { "report_id": { "type": "string" } }
        },
        {
          "required": ["query"],
          "properties": {
            "query": {
              "type": "object",
              "properties": {
                "category": { "type": "string" },
                "date_range": { "type": "object" },
                "store_ids": { "type": "array", "items": { "type": "string" } }
              }
            }
          }
        }
      ]
    },
    "options": {
      "type": "object",
      "properties": {
        "filename": { "type": "string" },
        "include_charts": { "type": "boolean", "default": true },
        "locale": { "type": "string", "default": "zh-CN" },
        "expires_in_seconds": { "type": "integer", "default": 3600 }
      }
    }
  }
}
```

### 输出 Schema

```json
{
  "$id": "skill/export-packager/output/v1",
  "type": "object",
  "required": ["success", "skill_version", "executed_at"],
  "properties": {
    "success": { "type": "boolean" },
    "skill_version": { "type": "string" },
    "executed_at": { "type": "string", "format": "date-time" },
    "file": {
      "type": "object",
      "properties": {
        "download_url": { "type": "string", "description": "预签名下载地址" },
        "filename": { "type": "string" },
        "format": { "type": "string" },
        "size_bytes": { "type": "integer" },
        "expires_at": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

---

**Schema Version**: 1.0.0 | **Last Updated**: 2026-04-05
