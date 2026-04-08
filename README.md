# ETL Dashboards & Reports — Azure Data Factory

This guide covers the full ETL design and implementation for each dashboard and report, using **Azure Data Factory (ADF)** as the orchestration and transformation engine over the `default` MySQL database.

---

## Architecture Overview

```
MySQL Source DB (default)
        │
        ▼
[ADF Copy Activity] ─── Linked Service: AzureMySql
        │
        ▼
Azure Data Lake Storage Gen2 (Raw Zone / Bronze)
        │
        ▼
[ADF Mapping Data Flow] ─── Transformations
        │
        ▼
Azure Synapse Analytics / Azure SQL DB (Gold Zone)
        │
        ▼
Power BI (Dashboards & Reports)
```

### Layers

| Layer | Storage | Description |
|---|---|---|
| **Bronze** | ADLS Gen2 `raw/` container | Raw copy of source tables (Parquet/CSV) |
| **Silver** | ADLS Gen2 `curated/` container | Cleaned, typed, deduplicated data |
| **Gold** | Azure SQL DB / Synapse | Aggregated fact and dimension tables ready for reporting |

---

## Source Database Tables Reference

| Table | Key Columns |
|---|---|
| `users` | `id`, `createdAt`, `birthdate`, `gender`, `verified`, `banned` |
| `users_activities` | `user`, `activity`, `membershipID`, `createdAt` |
| `users_schedules` | `user`, `schedule`, `membershipID`, `createdAt` |
| `users_memberships` | `user`, `membership` |
| `activities` | `id`, `activityType`, `date`, `status`, `canceled`, `scheduleId`, `quota`, `remainingQuota`, `duration` |
| `activityTypes` | `id`, `name`, `requireMembership`, `parentActivity` |
| `schedules` | `id`, `activityType`, `days` (JSON), `hour`, `quota`, `duration` |
| `memberships` | `id`, `name`, `color` |
| `memberships_activityTypes` | `membership`, `activity`, `amount`, `period` |
| `evaluations` | `id`, `user`, `date`, `fatPercentage`, `musclePercentage`, `weight` |
| `transactions` | `id`, `userID`, `activityID`, `value`, `membershipID`, `createdAt` |
| `tickets` | `id`, `title`, `color` |
| `users_tickets` | `user`, `ticket` |

---

## Dashboards

| # | Name | Guide |
|---|---|---|
| 1 | Retención y Deserción (Churn) | [dashboards/01-retencion-desercion.md](dashboards/01-retencion-desercion.md) |
| 2 | Engagement del Usuario | [dashboards/02-engagement-usuario.md](dashboards/02-engagement-usuario.md) |
| 3 | Progreso Físico | [dashboards/03-progreso-fisico.md](dashboards/03-progreso-fisico.md) |

## Reports

| # | Name | Guide |
|---|---|---|
| 1 | User 360 | [reports/01-user-360.md](reports/01-user-360.md) |
| 2 | Ingresos y Membresías | [reports/02-ingresos-membresias.md](reports/02-ingresos-membresias.md) |
| 3 | Comportamiento del Usuario | [reports/03-comportamiento-usuario.md](reports/03-comportamiento-usuario.md) |

---

## Shared ADF Components

### Linked Services
- **LS_MySQL_Source** — Azure Database for MySQL (or self-hosted IR for on-premise). Connection string pointing to the `default` database.
- **LS_ADLS_Bronze** — Azure Data Lake Storage Gen2 for raw ingestion.
- **LS_ADLS_Silver** — Same ADLS, `curated/` container.
- **LS_AzureSQL_Gold** — Azure SQL Database (or Synapse Dedicated Pool) for Gold layer output.

### Datasets (reusable)
- **DS_MySQL_[TableName]** — MySQL source dataset per table.
- **DS_Parquet_Bronze_[TableName]** — Parquet sink in `raw/[tablename]/` with date partitioning (`year=YYYY/month=MM/`).
- **DS_AzureSQL_[OutputTable]** — Gold layer sink datasets.

### Global Pipeline Parameters
| Parameter | Type | Example |
|---|---|---|
| `p_watermark_date` | String | `2025-01-01` |
| `p_run_date` | String | `@utcNow()` |
| `p_bronze_container` | String | `raw` |

### Trigger Strategy
- **Full load** (first run): Tumbling Window trigger, `p_watermark_date = 1900-01-01`
- **Incremental load** (daily): Schedule trigger at 02:00 UTC, using `createdAt` or `updatedAt` watermarks
