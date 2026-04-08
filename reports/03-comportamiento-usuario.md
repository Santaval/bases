# Report 3 — Comportamiento del Usuario

## Objetivo

Analizar los patrones de participación para transformar registros brutos en perfiles conductuales. El reporte identifica frecuencias de asistencia, preferencias por tipo de actividad, horarios y días favoritos, permitiendo personalizar la oferta de servicios y optimizar la programación de clases.

---

## Métricas Clave del Reporte

| Métrica | Descripción |
|---|---|
| **Frecuencia semanal** | Promedio de asistencias por semana por usuario |
| **Actividad preferida** | Tipo de actividad más frecuente por usuario |
| **Día preferido** | Día de la semana con mayor asistencia histórica |
| **Hora preferida** | Franja horaria con mayor asistencia |
| **Popularidad de actividades** | Ranking de tipos por total de asistencias |
| **Ocupación promedio** | `(quota - remainingQuota) / quota` por tipo de actividad |
| **Picos de demanda** | Top combinaciones día + hora con mayor asistencia |
| **Rotación de actividades** | Promedio de tipos distintos que un usuario prueba por mes |

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `users_activities` | `user`, `activity`, `createdAt` |
| `activities` | `id`, `activityType`, `date`, `status`, `canceled`, `quota`, `remainingQuota`, `duration`, `scheduleId` |
| `activityTypes` | `id`, `name`, `parentActivity` |
| `users_schedules` | `user`, `schedule`, `createdAt` |
| `schedules` | `id`, `activityType`, `days` (JSON), `hour`, `quota`, `duration` |

> La columna `schedules.days` es de tipo **JSON** en MySQL. Requiere transformación especial en ADF para extraer los días de la semana.

---

## Pipeline ADF: `PL_Report_Behavior`

### Estructura general

```
PL_Report_Behavior
├── ACT_Copy_Bronze
│   ├── Copy_users_activities
│   ├── Copy_activities
│   ├── Copy_activityTypes
│   ├── Copy_schedules
│   └── Copy_users_schedules
├── ACT_DataFlow_Silver
│   ├── DF_Silver_AttendanceBase
│   └── DF_Silver_SchedulePatterns
└── ACT_DataFlow_Gold
    ├── DF_Gold_UserBehaviorProfile
    ├── DF_Gold_ActivityPopularity
    └── DF_Gold_DemandPeaks
```

---

## Paso 1 — Ingesta a Bronze

### Copy_activities
```sql
SELECT id, activityType, date, status, canceled,
       quota, remainingQuota, duration, scheduleId
FROM activities
WHERE canceled = 0
  AND date >= '@{pipeline().parameters.p_watermark_date}'
```

### Copy_activityTypes
```sql
SELECT id, name, parentActivity FROM activityTypes;
```

### Copy_schedules
```sql
SELECT id, activityType, days, hour, quota, duration FROM schedules;
```

### Copy_users_activities y Copy_users_schedules
```sql
-- users_activities
SELECT user, activity, createdAt
FROM users_activities
WHERE createdAt >= '@{pipeline().parameters.p_watermark_date}';

-- users_schedules
SELECT user, schedule, createdAt FROM users_schedules;
```

---

## Paso 2 — Silver

### `DF_Silver_AttendanceBase`

**Source** → `Bronze_users_activities`

1. **Join** con `Bronze_activities` on `activity = activities.id`
   - Retener: `user`, `activities.activityType`, `activities.date AS attended_date`,
     `activities.duration`, `activities.quota`, `activities.remainingQuota`

2. **Join** con `Bronze_activityTypes` on `activityType = activityTypes.id`
   - Retener: `user`, `attended_date`, `type_id = activityType`, `type_name = activityTypes.name`,
     `parent_type = parentActivity`, `duration`, `quota`, `remainingQuota`

3. **Derived Column** — Temporales:
   ```
   attended_weekday   = dayOfWeek(attended_date)      -- 1=Sunday ... 7=Saturday
   attended_hour      = hour(attended_date)
   attended_ym        = toString(year(attended_date)) + '-' + lpad(toString(month(attended_date)), 2, '0')
   attended_week_num  = weekOfYear(attended_date)
   hour_bucket        = iif(attended_hour < 9,   'early_morning',
                          iif(attended_hour < 12, 'morning',
                            iif(attended_hour < 15, 'midday',
                              iif(attended_hour < 19, 'afternoon', 'evening'))))
   weekday_name       = case(attended_weekday,
                          1, 'Sunday', 2, 'Monday', 3, 'Tuesday',
                          4, 'Wednesday', 5, 'Thursday', 6, 'Friday', 'Saturday')
   occupancy_rate     = iif(quota > 0,
                          round(toDouble(quota - remainingQuota) / toDouble(quota) * 100, 2),
                          0)
   ```

4. **Sink** → `Silver_attendance_base`

---

### `DF_Silver_SchedulePatterns`

**Source** → `Bronze_schedules`

> `schedules.days` is stored as JSON array, e.g. `["Monday","Wednesday","Friday"]`.
> ADF Mapping Data Flow can parse JSON arrays using the `parse()` expression or **Flatten transformation**.

1. **Parse Column** — `days_parsed`:
   ```
   parse(days, ['string'])
   ```
   > In ADF Data Flow, use: `parse(days, array(string()))`

2. **Flatten** — explode `days_parsed` into individual rows, one per day:
   - Unroll by: `days_parsed`
   - New column: `schedule_day` (one string value per row: "Monday", "Tuesday", etc.)

3. **Join** con `Bronze_activityTypes` on `schedules.activityType = activityTypes.id`
   - Columna: `type_name`

4. **Derived Column** — `hour_bucket`
   ```
   iif(toInteger(substring(toString(hour), 1, 2)) < 9,   'early_morning',
     iif(toInteger(substring(toString(hour), 1, 2)) < 12, 'morning',
       iif(toInteger(substring(toString(hour), 1, 2)) < 15, 'midday',
         iif(toInteger(substring(toString(hour), 1, 2)) < 19, 'afternoon', 'evening'))))
   ```

5. **Aggregate** — por `schedule_day`, `hour_bucket`, `type_name`
   ```
   schedule_count = count(id)
   avg_quota      = avg(quota)
   ```

6. **Sink** → `Silver_schedule_patterns`

---

## Paso 3 — Gold

### `DF_Gold_UserBehaviorProfile`

**Source** → `Silver_attendance_base`

#### Sub-stream A: Frecuencia y preferencias globales por usuario

1. **Aggregate** — por `user`
   ```
   total_attended          = count(attended_date)
   favorite_type_id        = first(type_id)   -- ver nota abajo
   distinct_types          = countDistinct(type_id)
   total_weeks_with_activity = countDistinct(attended_week_num)
   avg_duration_attended   = avg(duration)
   ```
   > Para `favorite_type_id`: se requiere primero un pre-aggregate por (`user`, `type_id`) → `count` → luego sort desc y `first()`. Hacer esto en dos pasos de Data Flow:
   > - **Aggregate 1**: por `user`, `type_id` → `type_count = count(attended_date)`
   > - **Window**: `rank() over (partitionBy: user, orderBy: type_count DESC)`
   > - **Filter**: `rank == 1`
   > - Luego join con el aggregate principal.

2. **Join** con resultado de favorite_type lookup → obtener `favorite_type_name`

3. **Aggregate** adicional — día y hora favorita (mismo patrón: aggregate → rank → filter → join)

4. **Derived Column** — `weekly_avg_attendance`
   ```
   iif(total_weeks_with_activity > 0,
     round(toDouble(total_attended) / toDouble(total_weeks_with_activity), 2),
     0)
   ```

5. **Derived Column** — `behavior_profile`
   ```
   iif(weekly_avg_attendance >= 4,  'Very Active',
     iif(weekly_avg_attendance >= 2, 'Active',
       iif(weekly_avg_attendance >= 1, 'Occasional', 'Sporadic')))
   ```

6. **Sink** → `DS_AzureSQL_fact_user_behavior_profile`

---

### `DF_Gold_ActivityPopularity`

**Source** → `Silver_attendance_base`

1. **Aggregate** — por `type_id`, `type_name`, `attended_ym`
   ```
   total_attendances    = count(user)
   unique_attendees     = countDistinct(user)
   avg_occupancy_rate   = avg(occupancy_rate)
   total_hours_consumed = sum(duration) / 60
   ```

2. **Window** — ranking mensual por asistencias:
   ```
   monthly_rank = rank() over (partitionBy: [attended_ym], orderBy: total_attendances DESC)
   ```

3. **Aggregate general** — por `type_id`, `type_name` (sin filtro de mes):
   ```
   overall_total_attendances = sum(total_attendances)
   overall_unique_attendees  = countDistinct(unique_attendees)
   ```

4. **Sink** → `DS_AzureSQL_fact_activity_popularity`

---

### `DF_Gold_DemandPeaks`

**Source** → `Silver_attendance_base`

1. **Aggregate** — por `weekday_name`, `hour_bucket`, `type_name`
   ```
   attendance_count = count(user)
   unique_users     = countDistinct(user)
   avg_occupancy    = avg(occupancy_rate)
   ```

2. **Window** — ranking global:
   ```
   demand_rank = rank() over (orderBy: attendance_count DESC)
   ```

3. **Filter** — `demand_rank <= 20` (Top 20 combinaciones más concurridas)

4. **Sink** → `DS_AzureSQL_fact_demand_peaks`

---

## Esquema Gold (Output Tables)

### `fact_user_behavior_profile`
```sql
CREATE TABLE fact_user_behavior_profile (
    user_id                    VARCHAR(20)   NOT NULL,
    total_attended             INT           NOT NULL DEFAULT 0,
    distinct_activity_types    INT           NOT NULL DEFAULT 0,
    weekly_avg_attendance      DECIMAL(4,2)  NOT NULL DEFAULT 0,
    total_weeks_with_activity  INT           NOT NULL DEFAULT 0,
    favorite_type_id           VARCHAR(20),
    favorite_type_name         VARCHAR(100),
    favorite_weekday           VARCHAR(15),
    favorite_hour_bucket       VARCHAR(20),
    avg_duration_attended_min  DECIMAL(6,2),
    behavior_profile           VARCHAR(20),  -- Very Active|Active|Occasional|Sporadic
    etl_run_date               DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (user_id)
);
```

### `fact_activity_popularity`
```sql
CREATE TABLE fact_activity_popularity (
    type_id                    VARCHAR(20)   NOT NULL,
    type_name                  VARCHAR(100),
    period_ym                  VARCHAR(7)    NOT NULL,
    total_attendances          INT           NOT NULL DEFAULT 0,
    unique_attendees           INT           NOT NULL DEFAULT 0,
    avg_occupancy_rate         DECIMAL(5,2),
    total_hours_consumed       DECIMAL(10,2),
    monthly_rank               INT,
    etl_run_date               DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (type_id, period_ym)
);
```

### `fact_demand_peaks`
```sql
CREATE TABLE fact_demand_peaks (
    weekday_name      VARCHAR(15)   NOT NULL,
    hour_bucket       VARCHAR(20)   NOT NULL,
    type_name         VARCHAR(100)  NOT NULL,
    attendance_count  INT           NOT NULL DEFAULT 0,
    unique_users      INT           NOT NULL DEFAULT 0,
    avg_occupancy     DECIMAL(5,2),
    demand_rank       INT,
    etl_run_date      DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (weekday_name, hour_bucket, type_name)
);
```

---

## Nota sobre el campo JSON `schedules.days`

El campo `schedules.days` almacena un array JSON como `["Monday","Wednesday","Friday"]`. En ADF Mapping Data Flow:

1. El tipo de la columna debe definirse como `string` en el dataset origen.
2. Usar la transición **Parse** con tipo `array(string())`.
3. Aplicar **Flatten** transformation para explotar el array a filas individuales.
4. El resultado es una fila por cada día habilitado en el schedule.

Si el Integration Runtime no soporta directamente el parse JSON nativo en el Data Flow, se puede usar una **Script Activity** previa con Azure SQL para materializar el explode en una tabla temporal `stg_schedule_days`.

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Semanal (domingo a las 04:00 UTC) para análisis de comportamiento semanal
- Puede compartir la ingesta Bronze con `PL_Engagement_Dashboard` (mismo set de fuentes)

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| Ranking de actividades | Horizontal bar | `type_name`, `total_attendances` (mes seleccionado) |
| Heatmap de demanda | Matrix | `weekday_name` (columnas), `hour_bucket` (filas), `attendance_count` (valor + color) |
| Distribución de perfiles conductuales | Donut chart | `behavior_profile`, count de usuarios |
| Evolución de popularidad por tipo | Line chart | `period_ym`, `total_attendances`, segmentado por `type_name` |
| Ocupación promedio por actividad | Bar chart | `type_name`, `avg_occupancy_rate` |
| Top usuarios por actividad | Table | `user_id`, `total_attended`, `favorite_type_name`, `behavior_profile` |
| Preferencias día/hora | Stacked bar | `weekday_name`, `hour_bucket`, attendance |
| Comparativa de diversidad | Box plot / Scatter | `distinct_activity_types` por usuario, agrupado por `behavior_profile` |
