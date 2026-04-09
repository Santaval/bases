# Report 2 — Ingresos y Membresías

## Objetivo

Analizar la salud financiera del negocio mediante la distribución por tipo de suscripción y el crecimiento de la base de usuarios. Produce indicadores clave de membresías y tendencias de crecimiento de suscriptores.

---

## Métricas Clave del Reporte

| Métrica | Descripción |
|---|---|
| **Nuevos usuarios activos** | Usuarios registrados por primera vez en el período |
| **Tasa de crecimiento de suscriptores** | MoM growth de `users_memberships` |

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `memberships` | `id`, `name`, `color` |
| `users_memberships` | `user`, `membership` |
| `memberships_activityTypes` | `membership`, `activity`, `amount`, `period` |
| `users` | `id`, `createdAt`, `banned` |

---

## Pipeline ADF: `PL_Report_Revenue`

### Estructura general

```
PL_Report_Revenue
├── ACT_Copy_Bronze
│   ├── Copy_memberships
│   ├── Copy_users_memberships
│   ├── Copy_memberships_activityTypes
│   └── Copy_users (id, createdAt)
├── ACT_DataFlow_Silver
│   └── DF_Silver_MembershipGrowth
└── ACT_DataFlow_Gold
    └── DF_Gold_NewUsers
```

---

## Paso 1 — Ingesta a Bronze

### Copy_memberships
```sql
SELECT id, name FROM memberships;
```

### Copy_users_memberships
```sql
SELECT user, membership FROM users_memberships;
```

### Copy_memberships_activityTypes
```sql
SELECT membership, activity, amount, period FROM memberships_activityTypes;
```

### Copy_users (solo nuevos)
```sql
SELECT id, createdAt
FROM users
WHERE verified = 1
  AND banned   = 0
  AND createdAt >= '@{pipeline().parameters.p_watermark_date}'
```

---

## Paso 2 — Silver

### `DF_Silver_MembershipGrowth`

**Source** → `Bronze_users_memberships` JOIN `Bronze_memberships`

1. **Join** on `users_memberships.membership = memberships.id`

2. **Join** con `Bronze_users` on `users_memberships.user = users.id`
   - Columnas: `user`, `name AS membership_name`, `users.createdAt AS user_registration`

3. **Derived Column** — `registration_ym`
   ```
   toString(year(user_registration)) + '-' + lpad(toString(month(user_registration)), 2, '0')
   ```

4. **Aggregate** — por `registration_ym`, `membership_name`
   ```
   new_subscribers = count(user)
   ```

5. **Window** — acumulado por `membership_name`
   ```
   cumulative_subscribers = cumSum(new_subscribers) over (partitionBy: [membership_name], orderBy: registration_ym ASC)
   ```

6. **Sink** → `Silver_membership_growth`

---

## Paso 3 — Gold

### `DF_Gold_NewUsers`

**Source** → `Bronze_users`

1. **Aggregate** — por `registration_ym` (derivar de `createdAt`)
   ```
   new_users_count = count(id)
   ```

2. **Window** — acumulado:
   ```
   cumulative_users = cumSum(new_users_count) over (orderBy: registration_ym ASC)
   ```

3. **Sink** → `DS_AzureSQL_fact_user_growth`

---

## Esquema Gold (Output Tables)

### `fact_user_growth`
```sql
CREATE TABLE fact_user_growth (
    registration_ym       VARCHAR(7)    NOT NULL,
    new_users_count       INT           NOT NULL DEFAULT 0,
    cumulative_users      INT           NOT NULL DEFAULT 0,
    etl_run_date          DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (registration_ym)
);
```

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Mensual (1er día del mes a las 02:00 UTC) para cierre contable

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| Crecimiento de suscriptores | Line chart | `registration_ym`, `cumulative_users`, `new_users_count` |
| Nuevos usuarios por mes | Bar chart | `registration_ym`, `new_users_count` |
| KPIs del mes | Cards | Nuevos suscriptores, total acumulado, crecimiento MoM |
