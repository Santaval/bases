# Dashboard 2 — Engagement del Usuario

## Objetivo

Introducir un **Índice de Compromiso Compuesto (ICC)** calculado a partir de variables derivadas que no existen directamente en el sistema fuente: asistencia, consistencia temporal y diversidad de actividades. Permite diferenciar usuarios fidelizados de aquellos en riesgo de abandono.

---

## Métricas Clave del Dashboard

| Métrica | Descripción |
|---|---|
| **ICC** | Índice de Compromiso Compuesto (0–100), ponderación de los tres componentes |
| **Attendance Score** | Frecuencia de asistencia mensual normalizada |
| **Consistency Score** | Semanas consecutivas con al menos 1 asistencia |
| **Diversity Score** | Número de tipos de actividad distintos asistidos |
| **Engagement Segment** | Clasificación del usuario: High / Medium / Low / Inactive |
| **Top Activity Types** | Tipos de actividad con mayor asistencia agregada |
| **Engagement Trend** | Evolución del ICC promedio mes a mes |

### Fórmula del ICC

$$ICC = (0.5 \times \text{AttendanceScore}) + (0.3 \times \text{ConsistencyScore}) + (0.2 \times \text{DiversityScore})$$

Cada sub-score es normalizado en escala 0–100 respecto al máximo del período.

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `users_activities` | `user`, `activity`, `createdAt` |
| `activities` | `id`, `activityType`, `date`, `status`, `canceled` |
| `activityTypes` | `id`, `name` |
| `users_schedules` | `user`, `schedule`, `createdAt` |
| `users` | `id`, `createdAt` |

---

## Pipeline ADF: `PL_Engagement_Dashboard`

### Estructura general

```
PL_Engagement_Dashboard
├── ACT_Copy_Bronze
│   ├── Copy_users_activities
│   ├── Copy_activities
│   ├── Copy_activityTypes
│   └── Copy_users_schedules
├── ACT_DataFlow_Silver
│   └── DF_Silver_Engagement
└── ACT_DataFlow_Gold
    ├── DF_Gold_AttendanceScore
    ├── DF_Gold_ConsistencyScore
    ├── DF_Gold_DiversityScore
    └── DF_Gold_ICC_Aggregate
```

---

## Paso 1 — Ingesta a Bronze

### Copy_users_activities
```sql
SELECT user, activity, createdAt
FROM users_activities
WHERE createdAt >= '@{pipeline().parameters.p_watermark_date}'
```

### Copy_activities
```sql
SELECT id, activityType, date, status, canceled
FROM activities
WHERE canceled = 0
  AND status >= 1
  AND date >= '@{pipeline().parameters.p_watermark_date}'
```

### Copy_activityTypes
```sql
SELECT id, name
FROM activityTypes
WHERE hiddenToUsers = 0
```

### Copy_users_schedules
```sql
SELECT user, schedule, createdAt
FROM users_schedules
WHERE createdAt >= '@{pipeline().parameters.p_watermark_date}'
```

---

## Paso 2 — Silver: Consolidación de Asistencias (`DF_Silver_Engagement`)

**Source** → `Bronze_users_activities`

1. **Join** con `Bronze_activities` on `users_activities.activity = activities.id`
   - Retener: `user`, `activities.activityType`, `activities.date AS attended_date`

2. **Join** con `Bronze_activityTypes` on `activityType = activityTypes.id`
   - Retener: `user`, `attended_date`, `activityType AS type_id`, `activityTypes.name AS type_name`

3. **Filter** — excluir filas con `attended_date` nulo

4. **Derived Column** — columnas temporales:
   ```
   attended_year  = year(attended_date)
   attended_month = month(attended_date)
   attended_week  = weekOfYear(attended_date)
   attended_ym    = toString(attended_year) + '-' + lpad(toString(attended_month), 2, '0')
   ```

5. **Sink** → `DS_Parquet_Silver_attendance_detail`
   - Path: `curated/engagement/attendance_detail/`

---

## Paso 3 — Gold: Cálculo de Sub-Scores

### A) `DF_Gold_AttendanceScore`

**Source** → `Silver_attendance_detail`

1. **Aggregate** — por `user`, `attended_ym`
   ```
   monthly_attendance = count(attended_date)
   ```

2. **Window** — Calcular el máximo en la ventana global para normalización:
   ```
   max_attendance_ever = max(monthly_attendance) over (partitionBy: [])
   ```

3. **Derived Column** — `attendance_score_raw`
   ```
   round(toDouble(monthly_attendance) / toDouble(max_attendance_ever) * 100, 2)
   ```

4. **Sink** → `DS_AzureSQL_stg_attendance_score` (tabla staging)

---

### B) `DF_Gold_ConsistencyScore`

**Source** → `Silver_attendance_detail`

1. **Aggregate** — por `user`, `attended_ym`, `attended_week`
   ```
   active_week = count(attended_date) > 0 → 1 else 0
   ```
   > En ADF Mapping Data Flow usar `countDistinct(attended_week)` agrupado por `user`, `attended_ym`

2. **Aggregate** — por `user`, `attended_ym`
   ```
   active_weeks_in_month = countDistinct(attended_week)
   ```

3. **Derived Column** — `max_possible_weeks = 4` (constante)

4. **Derived Column** — `consistency_score_raw`
   ```
   round(toDouble(active_weeks_in_month) / 4.0 * 100, 2)
   ```

5. **Sink** → `DS_AzureSQL_stg_consistency_score`

---

### C) `DF_Gold_DiversityScore`

**Source** → `Silver_attendance_detail`

1. **Aggregate** — por `user`, `attended_ym`
   ```
   distinct_types = countDistinct(type_id)
   ```

2. **Window** — máximo global de tipos distintos para normalizar:
   ```
   max_types_ever = max(distinct_types) over (partitionBy: [])
   ```

3. **Derived Column** — `diversity_score_raw`
   ```
   round(toDouble(distinct_types) / toDouble(max_types_ever) * 100, 2)
   ```

4. **Sink** → `DS_AzureSQL_stg_diversity_score`

---

### D) `DF_Gold_ICC_Aggregate`

**Source** → `stg_attendance_score` (join con los otros dos staging)

1. **Join** — `stg_attendance_score` LEFT JOIN `stg_consistency_score` on (`user`, `attended_ym`)
2. **Join** — resultado LEFT JOIN `stg_diversity_score` on (`user`, `attended_ym`)
3. **Derived Column** — `icc_score`
   ```
   round(
     (0.5 * coalesce(attendance_score_raw, 0)) +
     (0.3 * coalesce(consistency_score_raw, 0)) +
     (0.2 * coalesce(diversity_score_raw, 0)),
   2)
   ```

4. **Derived Column** — `engagement_segment`
   ```
   iif(icc_score >= 70, 'High',
     iif(icc_score >= 40, 'Medium',
       iif(icc_score >= 10, 'Low', 'Inactive')))
   ```

5. **Sink** → `DS_AzureSQL_fact_engagement_monthly`

---

## Esquema Gold (Output Tables)

### `fact_engagement_monthly`
```sql
CREATE TABLE fact_engagement_monthly (
    user_id                VARCHAR(20)   NOT NULL,
    period_ym              VARCHAR(7)    NOT NULL,  -- 'YYYY-MM'
    monthly_attendance     INT           NOT NULL DEFAULT 0,
    attendance_score       DECIMAL(5,2)  NOT NULL DEFAULT 0,
    active_weeks_in_month  INT           NOT NULL DEFAULT 0,
    consistency_score      DECIMAL(5,2)  NOT NULL DEFAULT 0,
    distinct_activity_types INT          NOT NULL DEFAULT 0,
    diversity_score        DECIMAL(5,2)  NOT NULL DEFAULT 0,
    icc_score              DECIMAL(5,2)  NOT NULL DEFAULT 0,
    engagement_segment     VARCHAR(10)   NOT NULL,  -- High|Medium|Low|Inactive
    etl_run_date           DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (user_id, period_ym)
);
```

### `fact_engagement_summary` (vista agregada para KPI cards)
```sql
CREATE VIEW fact_engagement_summary AS
SELECT
    period_ym,
    AVG(icc_score)                                        AS avg_icc,
    COUNT(CASE WHEN engagement_segment = 'High'     THEN 1 END) AS high_users,
    COUNT(CASE WHEN engagement_segment = 'Medium'   THEN 1 END) AS medium_users,
    COUNT(CASE WHEN engagement_segment = 'Low'      THEN 1 END) AS low_users,
    COUNT(CASE WHEN engagement_segment = 'Inactive' THEN 1 END) AS inactive_users
FROM fact_engagement_monthly
GROUP BY period_ym;
```

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Mensual (1er día del mes a las 04:00 UTC, o diario si se requiere granularidad diaria)
- Depende de: `PL_Churn_Dashboard` no tiene dependencia directa, puede correr en paralelo.

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| Tendencia del ICC promedio | Line chart | `period_ym`, `avg_icc` |
| Distribución de segmentos | Stacked bar / Donut | `period_ym`, `High/Medium/Low/Inactive` counts |
| Top usuarios por ICC | Table | `user_id`, `icc_score`, `engagement_segment`, filtrado por mes |
| Dispersión Attendance vs Consistency | Scatter plot | `attendance_score` (x), `consistency_score` (y), punto = usuario, color = `engagement_segment` |
| Heatmap de actividad semanal | Matrix | semana (columna), usuario (fila), `active_weeks` (valor) |
| Evolución del segmento de un usuario | Line chart | filtro por `user_id`, `period_ym`, `icc_score` |
