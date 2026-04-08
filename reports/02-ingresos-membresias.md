# Report 2 — Ingresos y Membresías

## Objetivo

Analizar la salud financiera del negocio mediante la evolución temporal de ingresos, la distribución por tipo de suscripción y el crecimiento de la base de usuarios. Agrega datos transaccionales para producir indicadores financieros clave como MRR, LTV y tendencias de crecimiento.

---

## Métricas Clave del Reporte

| Métrica | Descripción |
|---|---|
| **MRR** | Monthly Recurring Revenue — ingresos agregados por mes |
| **Nuevo MRR** | Ingresos de usuarios que pagan por primera vez ese mes |
| **Expansión MRR** | Ingresos adicionales por usuarios que mejoran su membresía |
| **Churn MRR** | Ingresos perdidos por cancelaciones estimadas |
| **Ingresos por membresía** | Desglose de MRR por tipo de membresía |
| **Nuevos usuarios activos** | Usuarios que compraron por primera vez en el período |
| **LTV estimado** | Life Time Value promedio por tipo de membresía |
| **Tasa de crecimiento de suscriptores** | MoM growth de `users_memberships` |

---

## Fuentes de Datos

| Tabla MySQL | Columnas utilizadas |
|---|---|
| `transactions` | `id`, `userID`, `activityID`, `value`, `membershipID`, `createdAt` |
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
│   ├── Copy_transactions
│   ├── Copy_memberships
│   ├── Copy_users_memberships
│   ├── Copy_memberships_activityTypes
│   └── Copy_users (id, createdAt)
├── ACT_DataFlow_Silver
│   ├── DF_Silver_Transactions
│   └── DF_Silver_MembershipGrowth
└── ACT_DataFlow_Gold
    ├── DF_Gold_MRR
    ├── DF_Gold_MembershipRevenue
    ├── DF_Gold_NewUsers
    └── DF_Gold_LTV
```

---

## Paso 1 — Ingesta a Bronze

### Copy_transactions
```sql
SELECT id, userID, activityID, value, membershipID, createdAt
FROM transactions
WHERE value > 0
  AND createdAt >= '@{pipeline().parameters.p_watermark_date}'
ORDER BY createdAt ASC
```

> `value` puede ser negativo (reembolsos o créditos). Filtrar `value > 0` para ingresos. Mantener negativos en stream separado para churn revenue.

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

### `DF_Silver_Transactions`

**Source** → `Bronze_transactions`

1. **Derived Column** — columnas temporales:
   ```
   tx_year  = year(createdAt)
   tx_month = month(createdAt)
   tx_ym    = toString(tx_year) + '-' + lpad(toString(tx_month), 2, '0')
   ```

2. **Left Join** con `Bronze_memberships` on `transactions.membershipID = memberships.id`
   - Columna añadida: `membership_name`

3. **Derived Column** — `is_income` y `is_refund`
   ```
   is_income = iif(value > 0, true(), false())
   is_refund = iif(value < 0, true(), false())
   ```

4. **Window Transformation** — primera transacción por usuario:
   - Partition by: `userID`
   - Sort: `createdAt` ASC
   - Window column: `user_tx_rank = rank()`

5. **Derived Column** — `is_new_customer`
   ```
   iif(user_tx_rank == 1, true(), false())
   ```

6. **Sink** → `Silver_transactions_enriched`

---

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

### `DF_Gold_MRR`

**Source** → `Silver_transactions_enriched` (filtro: `is_income = true`)

1. **Aggregate** — por `tx_ym`
   ```
   total_revenue        = sum(value)
   transaction_count    = count(id)
   unique_paying_users  = countDistinct(userID)
   new_customer_revenue = sumIf(is_new_customer == true, value)
   returning_revenue    = sumIf(is_new_customer == false, value)
   ```

2. **Window** — MoM growth:
   ```
   prev_month_revenue = lag(total_revenue, 1) over (orderBy: tx_ym ASC)
   ```

3. **Derived Column** — `mom_growth_pct`
   ```
   iif(prev_month_revenue > 0,
     round((toDouble(total_revenue) - toDouble(prev_month_revenue)) / toDouble(prev_month_revenue) * 100, 2),
     0)
   ```

4. **Sink** → `DS_AzureSQL_fact_mrr`

---

### `DF_Gold_MembershipRevenue`

**Source** → `Silver_transactions_enriched` (filtro: `is_income = true`)

1. **Aggregate** — por `tx_ym`, `membershipID`, `membership_name`
   ```
   revenue_by_membership = sum(value)
   tx_count              = count(id)
   unique_users          = countDistinct(userID)
   ```

2. **Derived Column** — `revenue_share_pct` (dentro del mes):
   - Requiere join con `fact_mrr` para obtener `total_revenue` del mes
   - Alternativa: Window `sum(revenue_by_membership) over (partitionBy: [tx_ym])` para el total

3. **Sink** → `DS_AzureSQL_fact_membership_revenue`

---

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

### `DF_Gold_LTV`

**Source** → `Silver_transactions_enriched`

1. **Aggregate** — por `userID`, `membershipID`, `membership_name`
   ```
   user_lifetime_revenue = sum(value)
   first_purchase        = min(createdAt)
   last_purchase         = max(createdAt)
   purchase_count        = count(id)
   ```

2. **Derived Column** — `customer_lifetime_days`
   ```
   dateDiff('day', toDate(first_purchase), toDate(last_purchase))
   ```

3. **Aggregate** — por `membership_name` (para LTV promedio):
   ```
   avg_ltv_per_membership = avg(user_lifetime_revenue)
   avg_lifetime_days      = avg(customer_lifetime_days)
   total_customers        = count(userID)
   ```

4. **Sink** → `DS_AzureSQL_fact_ltv_by_membership`

---

## Esquema Gold (Output Tables)

### `fact_mrr`
```sql
CREATE TABLE fact_mrr (
    period_ym             VARCHAR(7)    NOT NULL,
    total_revenue         INT           NOT NULL DEFAULT 0,
    transaction_count     INT           NOT NULL DEFAULT 0,
    unique_paying_users   INT           NOT NULL DEFAULT 0,
    new_customer_revenue  INT           NOT NULL DEFAULT 0,
    returning_revenue     INT           NOT NULL DEFAULT 0,
    prev_month_revenue    INT,
    mom_growth_pct        DECIMAL(6,2),
    etl_run_date          DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (period_ym)
);
```

### `fact_membership_revenue`
```sql
CREATE TABLE fact_membership_revenue (
    period_ym             VARCHAR(7)    NOT NULL,
    membership_id         VARCHAR(20)   NOT NULL,
    membership_name       VARCHAR(100),
    revenue               INT           NOT NULL DEFAULT 0,
    transaction_count     INT           NOT NULL DEFAULT 0,
    unique_users          INT           NOT NULL DEFAULT 0,
    etl_run_date          DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (period_ym, membership_id)
);
```

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

### `fact_ltv_by_membership`
```sql
CREATE TABLE fact_ltv_by_membership (
    membership_id         VARCHAR(20)   NOT NULL,
    membership_name       VARCHAR(100),
    avg_ltv               DECIMAL(10,2) NOT NULL DEFAULT 0,
    avg_lifetime_days     INT           NOT NULL DEFAULT 0,
    total_customers       INT           NOT NULL DEFAULT 0,
    etl_run_date          DATETIME      DEFAULT GETDATE(),
    PRIMARY KEY (membership_id)
);
```

---

## Trigger

- **Tipo**: Schedule Trigger
- **Frecuencia**: Mensual (1er día del mes a las 02:00 UTC) para cierre contable
- Adicionalmente: Schedule diario para MRR acumulado en tiempo real

---

## Visualización Sugerida (Power BI)

| Visual | Tipo | Campos |
|---|---|---|
| MRR mensual | Area chart | `period_ym`, `total_revenue` |
| Crecimiento MoM | Line + bar combo | `period_ym`, `total_revenue`, `mom_growth_pct` |
| Ingresos por membresía | Stacked bar | `period_ym`, `membership_name`, `revenue` |
| Market share de membresías | Pie chart | `membership_name`, `revenue` (mes seleccionado) |
| Crecimiento de suscriptores | Line chart | `registration_ym`, `cumulative_users`, `new_users_count` |
| LTV por membresía | Bar chart | `membership_name`, `avg_ltv` |
| KPIs del mes | Cards | MRR actual, nuevos clientes, MoM growth, users activos pagando |
