# Dashboard 1 — Retención y Deserción (Churn)

## Objetivo

Visualizar la evolución de la masa de usuarios activos, en riesgo y perdidos a lo largo del tiempo. Permite identificar tendencias de compromiso y detectar posibles fugas de clientes de manera temprana mediante el establecimiento de umbrales de actividad.

---

## Métricas Clave del Dashboard

| Métrica | Descripción |
|---|---|
| **MAU** | Monthly Active Users — usuarios con al menos 1 asistencia en los últimos 30 días |
| **At-Risk Users** | Usuarios sin actividad entre 31 y 60 días |
| **Churned Users** | Usuarios sin actividad en más de 60 días |
| **Churn Rate %** | `churned / (active + at_risk + churned)` por mes |
| **Recovery Rate %** | Usuarios que vuelven a ser activos tras haber estado "at-risk" o "churned" |
| **Cohort Retention** | % de usuarios de cada cohorte mensual que siguen activos N meses después |

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `users` | `id`, `createdAt`, `banned`, `verified` |
| `users_activities` | `user`, `activity`, `createdAt` |
| `users_schedules` | `user`, `schedule`, `createdAt` |
| `activities` | `id`, `date`, `status`, `canceled` |

La **fecha de última actividad** se calcula combinando:
- `users_activities.createdAt` (fecha de reserva/asistencia)
- `activities.date` (fecha real de la actividad)

---

## Pipeline ADF: `PL_Churn_Dashboard`

### Estructura general

```
PL_Churn_Dashboard
├── ACT_Copy_Bronze          (Copy Activities en paralelo)
│   ├── Copy_users
│   ├── Copy_users_activities
│   ├── Copy_activities
│   └── Copy_users_schedules
├── ACT_DataFlow_Silver      (Mapping Data Flow: limpieza)
│   └── DF_Silver_Churn
└── ACT_DataFlow_Gold        (Mapping Data Flow: agregación)
    └── DF_Gold_Churn
```

---

## Paso 1 — Ingesta a Bronze (Copy Activities)

Ejecutar **en paralelo** con la actividad `Execute in parallel` (ForEach no requerido dado el número fijo de tablas).

### Copy_users
- **Source**: `DS_MySQL_users`
- **Sink**: `DS_Parquet_Bronze_users` → `raw/users/year=@year/month=@month/users.parquet`
- **Query incremental**:
  ```sql
  SELECT id, createdAt, banned, verified
  FROM users
  WHERE createdAt >= '@{pipeline().parameters.p_watermark_date}'
  ```

### Copy_users_activities
- **Source**: `DS_MySQL_users_activities`
- **Sink**: `DS_Parquet_Bronze_users_activities`
- **Query incremental**:
  ```sql
  SELECT user, activity, createdAt
  FROM users_activities
  WHERE createdAt >= '@{pipeline().parameters.p_watermark_date}'
  ```

### Copy_activities
- **Source**: `DS_MySQL_activities`
- **Sink**: `DS_Parquet_Bronze_activities`
- **Query**:
  ```sql
  SELECT id, date, status, canceled
  FROM activities
  WHERE date >= '@{pipeline().parameters.p_watermark_date}'
    AND canceled = 0
    AND status = 1
  ```

### Copy_users_schedules
- **Source**: `DS_MySQL_users_schedules`
- **Sink**: `DS_Parquet_Bronze_users_schedules`
- **Query incremental** por `createdAt`.

---

## Paso 2 — Silver: Limpieza y Tipado (`DF_Silver_Churn`)

Mapping Data Flow con las siguientes transformaciones:

### Stream: `stm_last_activity`

**Source** → `Bronze_users_activities` (Parquet)

1. **Join** con `Bronze_activities` on `users_activities.activity = activities.id`
   - Join type: Inner
   - Retener columnas: `user`, `activities.date AS activity_date`

2. **Aggregate**
   - Group by: `user`
   - Columnas derivadas:
     ```
     last_activity_date = max(activity_date)
     total_activities   = count(activity_date)
     ```

3. **Derived Column** — `days_since_last_activity`
   ```
   dayOfMonth(currentDate()) - dayOfMonth(last_activity_date)
   ```
   > Usar `dateDiff('day', last_activity_date, currentDate())` en la expresión ADF.

4. **Derived Column** — `user_status`
   ```
   iif(days_since_last_activity <= 30, 'active',
     iif(days_since_last_activity <= 60, 'at_risk', 'churned'))
   ```

5. **Join** con `Bronze_users` on `user = users.id`
   - Join type: Right Outer (para incluir usuarios sin ninguna actividad)
   - Para usuarios sin actividad: `days_since_last_activity = dateDiff('day', users.createdAt, currentDate())`, `user_status = 'churned'`

6. **Select** — columnas finales:
   ```
   user_id, registration_date (users.createdAt),
   last_activity_date, days_since_last_activity,
   user_status, total_activities, banned
   ```

7. **Sink** → `DS_Parquet_Silver_user_status` → `curated/churn/user_status/`

---

## Paso 3 — Gold: Agregación Mensual (`DF_Gold_Churn`)

### Stream: `stm_monthly_churn`

**Source** → `Silver_user_status`

1. **Derived Column** — `registration_month`
   ```
   toString(year(registration_date)) + '-' + lpad(toString(month(registration_date)), 2, '0')
   ```

2. **Derived Column** — `snapshot_month` (mes actual del run)
   ```
   toString(year(currentDate())) + '-' + lpad(toString(month(currentDate())), 2, '0')
   ```

3. **Aggregate** — por `snapshot_month`
   ```
   active_users  = countIf(user_status == 'active')
   atrisk_users  = countIf(user_status == 'at_risk')
   churned_users = countIf(user_status == 'churned')
   total_users   = count(user_id)
   ```

4. **Derived Column** — `churn_rate_pct`
   ```
   round(toDouble(churned_users) / toDouble(total_users) * 100, 2)
   ```

5. **Derived Column** — `atrisk_rate_pct`
   ```
   round(toDouble(atrisk_users) / toDouble(total_users) * 100, 2)
   ```

6. **Sink** → `DS_AzureSQL_fact_churn_monthly`

### Stream: `stm_cohort_retention`

**Source** → `Silver_user_status`

1. **Derived Column** — `cohort_month`
   ```
   toString(year(registration_date)) + '-' + lpad(toString(month(registration_date)), 2, '0')
   ```

2. **Aggregate** — por `cohort_month`, `user_status`
   ```
   user_count = count(user_id)
   ```

3. **Pivot** — transformar `user_status` en columnas: `active`, `at_risk`, `churned`

4. **Derived Column** — `retention_rate_pct`
   ```
   round(toDouble(active) / toDouble(active + at_risk + churned) * 100, 2)
   ```

5. **Sink** → `DS_AzureSQL_fact_cohort_retention`

---

## Esquema Gold (Output Tables)

### `fact_churn_monthly`
```sql
CREATE TABLE fact_churn_monthly (
    snapshot_month   VARCHAR(7)     NOT NULL,  -- 'YYYY-MM'
    active_users     INT            NOT NULL,
    atrisk_users     INT            NOT NULL,
    churned_users    INT            NOT NULL,
    total_users      INT            NOT NULL,
    churn_rate_pct   DECIMAL(5,2)   NOT NULL,
    atrisk_rate_pct  DECIMAL(5,2)   NOT NULL,
    etl_run_date     DATETIME       DEFAULT GETDATE(),
    PRIMARY KEY (snapshot_month)
);
```

### `fact_cohort_retention`
```sql
CREATE TABLE fact_cohort_retention (
    cohort_month        VARCHAR(7)   NOT NULL,  -- 'YYYY-MM'
    active_users        INT          NOT NULL,
    atrisk_users        INT          NOT NULL,
    churned_users       INT          NOT NULL,
    retention_rate_pct  DECIMAL(5,2) NOT NULL,
    etl_run_date        DATETIME     DEFAULT GETDATE(),
    PRIMARY KEY (cohort_month)
);
```

### `dim_user_status` (snapshot diario)
```sql
CREATE TABLE dim_user_status (
    user_id                  VARCHAR(20)  NOT NULL,
    registration_date        DATETIME     NOT NULL,
    last_activity_date       DATETIME,
    days_since_last_activity INT,
    user_status              VARCHAR(10)  NOT NULL,  -- active | at_risk | churned
    total_activities         INT          NOT NULL DEFAULT 0,
    banned                   BIT          NOT NULL DEFAULT 0,
    snapshot_date            DATE         NOT NULL,
    PRIMARY KEY (user_id, snapshot_date)
);
```

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Diario a las 03:00 UTC
- **Parámetro**: `p_watermark_date` se lee desde tabla de control `etl_watermark` en Azure SQL

```sql
-- Tabla de control de watermarks
CREATE TABLE etl_watermark (
    pipeline_name  VARCHAR(100) PRIMARY KEY,
    last_run_date  DATETIME
);
```

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| Tendencia de usuarios activos | Line chart | `snapshot_month`, `active_users`, `atrisk_users`, `churned_users` |
| Churn Rate mensual | Area chart | `snapshot_month`, `churn_rate_pct` |
| Distribución de estado | Donut chart | `user_status`, count |
| Retención por cohorte | Matrix / Heatmap | `cohort_month` (filas), meses transcurridos (columnas), `retention_rate_pct` |
| Usuarios en riesgo esta semana | Card | Filter `atrisk_users` del mes actual |
