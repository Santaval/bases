# Dashboard 3 — Progreso Físico

## Objetivo

Transformar mediciones clínicas aisladas (peso, porcentaje de grasa y masa muscular) en tendencias temporales significativas por usuario. Categorizar la evolución de cada usuario y permitir un análisis de salud longitudinal que va mucho más allá del dato numérico puntual.

---

## Métricas Clave del Dashboard

| Métrica | Descripción |
|---|---|
| **Δ Peso** | Variación de peso entre evaluaciones consecutivas |
| **Δ Grasa %** | Variación del porcentaje de grasa corporal |
| **Δ Músculo %** | Variación del porcentaje de masa muscular |
| **Tendencia Global** | Clasificación del usuario: Mejora / Estable / Declive |
| **Evaluaciones por período** | Frecuencia de evaluaciones por usuario y mes |
| **Distribución por categoría** | % de usuarios en cada tendencia global |
| **Correlación Engagement–Progreso** | Cruce con ICC (Dashboard 2) |

### Lógica de Clasificación de Tendencia

Para cada usuario, comparando la última evaluación vs. la penúltima:

| Condición | Categoría |
|---|---|
| `Δ_fat < -1` **Y** `Δ_muscle > 0` | **Mejora** |
| `Δ_fat > 1` **O** `Δ_muscle < -1` | **Declive** |
| Resto | **Estable** |

> Umbrales configurables como parámetros del pipeline.

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `evaluations` | `id`, `user`, `date`, `fatPercentage`, `musclePercentage`, `weight`, `evaluator`, `comment` |
| `users` | `id`, `birthdate`, `gender`, `createdAt` |

---

## Pipeline ADF: `PL_PhysicalProgress_Dashboard`

### Estructura general

```
PL_PhysicalProgress_Dashboard
├── ACT_Copy_Bronze
│   ├── Copy_evaluations
│   └── Copy_users (solo columnas demográficas)
├── ACT_DataFlow_Silver
│   └── DF_Silver_Evaluations
└── ACT_DataFlow_Gold
    ├── DF_Gold_EvaluationDeltas
    └── DF_Gold_ProgressSummary
```

---

## Paso 1 — Ingesta a Bronze

### Copy_evaluations
```sql
SELECT id, user, date, fatPercentage, musclePercentage, weight, evaluator
FROM evaluations
ORDER BY user, date ASC
```
> Sin filtro de watermark en primera carga. Para incremental: `WHERE date >= '@{pipeline().parameters.p_watermark_date}'`

### Copy_users (datos demográficos)
```sql
SELECT id, birthdate, gender
FROM users
WHERE verified = 1
  AND banned  = 0
```

---

## Paso 2 — Silver: Enriquecimiento y Ordenación (`DF_Silver_Evaluations`)

**Source** → `Bronze_evaluations`

1. **Join** con `Bronze_users` on `evaluations.user = users.id`
   - Retener: todos los campos de `evaluations` + `birthdate`, `gender`

2. **Derived Column** — `age_at_evaluation`
   ```
   floor(dateDiff('year', birthdate, toDate(date)))
   ```

3. **Derived Column** — `age_group`
   ```
   iif(age_at_evaluation < 25, '<25',
     iif(age_at_evaluation <= 35, '25-35',
       iif(age_at_evaluation <= 50, '36-50', '>50')))
   ```

4. **Derived Column** — `gender_label`
   ```
   iif(gender == 1, 'Male', 'Female')
   ```

5. **Sort** — por `user`, `date` ASC (necesario para la ventana de lag en paso 3)

6. **Derived Column** — columnas temporales:
   ```
   eval_year  = year(toDate(date))
   eval_month = month(toDate(date))
   eval_ym    = toString(eval_year) + '-' + lpad(toString(eval_month), 2, '0')
   ```

7. **Sink** → `DS_Parquet_Silver_evaluations`
   - Path: `curated/physical_progress/evaluations_enriched/`

---

## Paso 3 — Gold: Cálculo de Deltas (`DF_Gold_EvaluationDeltas`)

**Source** → `Silver_evaluations`

> ADF Mapping Data Flow soporta **Window Transformation** con funciones `lag()` para calcular valores previos dentro de una partición ordenada.

1. **Window Transformation**
   - Partition by: `user`
   - Sort: `date` ASC
   - Window columns:
     ```
     prev_fat        = lag(fatPercentage, 1)
     prev_muscle     = lag(musclePercentage, 1)
     prev_weight     = lag(weight, 1)
     prev_eval_date  = lag(date, 1)
     eval_number     = rank()
     ```

2. **Derived Column** — Deltas:
   ```
   delta_fat_pct    = round(fatPercentage - coalesce(prev_fat, fatPercentage), 2)
   delta_muscle_pct = round(musclePercentage - coalesce(prev_muscle, musclePercentage), 2)
   delta_weight_kg  = round(weight - coalesce(prev_weight, weight), 2)
   days_between     = dateDiff('day', coalesce(prev_eval_date, date), date)
   is_first_eval    = iif(eval_number == 1, true(), false())
   ```

3. **Derived Column** — `progress_category` (evaluación a evaluación):
   ```
   iif(is_first_eval, 'baseline',
     iif(delta_fat_pct < -1 && delta_muscle_pct > 0, 'improving',
       iif(delta_fat_pct > 1 || delta_muscle_pct < -1, 'declining', 'stable')))
   ```

4. **Sink** → `DS_AzureSQL_fact_evaluation_detail`

---

### `DF_Gold_ProgressSummary`

**Source** → `fact_evaluation_detail` (solo registros con `is_first_eval = false`)

1. **Aggregate** — por `user`
   ```
   total_evals        = count(id)
   first_eval_date    = min(eval_date)
   last_eval_date     = max(eval_date)
   total_fat_change   = sum(delta_fat_pct)
   total_muscle_change= sum(delta_muscle_pct)
   total_weight_change= sum(delta_weight_kg)
   improving_count    = countIf(progress_category == 'improving')
   declining_count    = countIf(progress_category == 'declining')
   stable_count       = countIf(progress_category == 'stable')
   ```

2. **Derived Column** — `overall_trend`
   ```
   iif(total_fat_change < -2 && total_muscle_change > 0, 'Mejora',
     iif(total_fat_change > 2 || total_muscle_change < -2, 'Declive', 'Estable'))
   ```

3. **Derived Column** — `avg_eval_interval_days`
   ```
   iif(total_evals > 1,
     divide(dateDiff('day', first_eval_date, last_eval_date), total_evals - 1),
     0)
   ```

4. **Sink** → `DS_AzureSQL_fact_progress_summary`

---

## Esquema Gold (Output Tables)

### `fact_evaluation_detail`
```sql
CREATE TABLE fact_evaluation_detail (
    eval_id              INT           NOT NULL,
    user_id              VARCHAR(20)   NOT NULL,
    eval_date            DATE          NOT NULL,
    eval_ym              VARCHAR(7)    NOT NULL,
    fat_percentage       DECIMAL(5,2)  NOT NULL,
    muscle_percentage    DECIMAL(5,2)  NOT NULL,
    weight_kg            DECIMAL(6,2)  NOT NULL,
    delta_fat_pct        DECIMAL(5,2),
    delta_muscle_pct     DECIMAL(5,2),
    delta_weight_kg      DECIMAL(5,2),
    days_between_evals   INT,
    eval_number          INT           NOT NULL,
    is_first_eval        BIT           NOT NULL DEFAULT 0,
    progress_category    VARCHAR(10),  -- baseline|improving|declining|stable
    age_at_evaluation    INT,
    age_group            VARCHAR(10),
    gender_label         VARCHAR(10),
    etl_run_date         DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (eval_id)
);
```

### `fact_progress_summary`
```sql
CREATE TABLE fact_progress_summary (
    user_id               VARCHAR(20)  NOT NULL,
    total_evals           INT          NOT NULL,
    first_eval_date       DATE,
    last_eval_date        DATE,
    total_fat_change      DECIMAL(6,2),
    total_muscle_change   DECIMAL(6,2),
    total_weight_change   DECIMAL(6,2),
    improving_count       INT          NOT NULL DEFAULT 0,
    declining_count       INT          NOT NULL DEFAULT 0,
    stable_count          INT          NOT NULL DEFAULT 0,
    overall_trend         VARCHAR(10),  -- Mejora|Estable|Declive
    avg_eval_interval_days INT,
    etl_run_date          DATETIME     DEFAULT GETDATE(),
    PRIMARY KEY (user_id)
);
```

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Semanal (lunes a las 05:00 UTC) — las evaluaciones no son diarias
- **Primera carga**: Full load sin watermark (tabla `evaluations` tiene 524 registros)

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| Evolución de peso individual | Line chart | `user_id` (filtro), `eval_date`, `weight_kg` |
| Distribución de tendencias globales | Donut chart | `overall_trend`, count de usuarios |
| Tendencia de grasa vs músculo | Dual-axis line | `eval_date`, `fat_percentage`, `muscle_percentage` |
| Comparativa antes/después | Clustered bar | primera vs última evaluación por usuario |
| Distribución por grupo de edad y tendencia | Stacked bar | `age_group`, `overall_trend`, count |
| Frecuencia de evaluaciones | Bar chart | `eval_ym`, count de evaluaciones |
| Top usuarios con mayor mejora | Table | `user_id`, `total_fat_change`, `total_muscle_change`, `overall_trend` ordenado por mejora |
| Correlación ICC vs Progreso | Scatter plot | `icc_score` (x), `total_fat_change` (y), punto = usuario |
