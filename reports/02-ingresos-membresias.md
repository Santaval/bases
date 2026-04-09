# Report 2 — Membresías y Utilización de Créditos

## Objetivo

Auditores y stakeholders utilizan este reporte para monitorear el **ciclo de vida de las membresías** y la **tasa de utilización de créditos** (sesiones). El objetivo es medir la retención de usuarios y asegurar que los créditos otorgados por membresías se consuman correctamente según los tipos de actividad.

---

## Métricas Clave

| Métrica | Descripción |
|---|---|
| **Consumo de Sesiones** | Suma de `value` negativo en la tabla `transactions`. |
| **Balance de Créditos** | Suma acumulada de `value` (grants - consumos) por usuario y actividad. |
| **Penetración por Plan** | Distribución de usuarios activos (`users_memberships`) por tipo de membresía. |
| **Engagement Rate** | Relación entre créditos otorgados y créditos consumidos en el periodo. |

---

## Fuentes de Datos (Esquema MySQL)

| Tabla | Rol | Columnas Clave |
|---|---|---|
| `users` | Perfiles | `id`, `name`, `createdAt`, `banned` |
| `memberships` | Catálogo | `id`, `name`, `color` |
| `users_memberships` | Asignación | `user` (FK), `membership` (FK) |
| `transactions` | **Auditoría** | `userID`, `membershipID`, `activityID`, `value` (+/-) |
| `memberships_activityTypes` | Reglas | `membership`, `activity`, `amount`, `period` |

---

## Pipeline ADF: `PL_Report_Membership_Utilization`

### Estructura general

```mermaid
graph TD
    A[MySQL Source] --> B[Bronze Datasets]
    B --> C[Silver Dataflow: DF_Silver_Balances]
    C --> D[Gold Dataflow: DF_Gold_MembershipKpis]
    D --> E[Azure SQL / Power BI]
```

---

## Paso 1 — Ingesta a Bronze (SQL Extract)

### Extract_Transactions
Identifica el consumo real de actividades. Un `value < 0` indica uso de sesión.
```sql
SELECT 
    userID, 
    membershipID, 
    activityID, 
    value, 
    id as transactionID 
FROM transactions;
```

### Extract_User_Plans
Mapea usuarios a sus membresías actuales.
```sql
SELECT 
    user, 
    membership 
FROM users_memberships;
```

### Extract_Users (Nuevos registros)
```sql
SELECT id, name, createdAt
FROM users
WHERE verified = 1 AND banned = 0;
```

---

## Paso 2 — Silver: Cálculo de Utilización

### `DF_Silver_UsageAudit`

**Source** → `Bronze_transactions` JOIN `Bronze_memberships`

1. **Categorización de Valor**:
   - `usage_flag` = `iif(value < 0, 1, 0)`
   - `grant_flag` = `iif(value > 0, 1, 0)`

2. **Join** con `Bronze_memberships` on `membershipID`
   - Obtenemos `membership_name` y `color`.

3. **Window Function** — Saldo por Usuario/Actividad:
   ```
   current_balance = sum(value) over (partitionBy: [userID, activityID], orderBy: transactionID ASC)
   ```

4. **Sink** → `Silver_user_balances`

---

## Paso 3 — Gold: KPIs de Retención

### `DF_Gold_UserLifeCycle`

**Source** → `Silver_user_balances` JOIN `Bronze_users`

1. **Aggregate** por `registration_ym` y `membershipID`:
   - `total_active_users = countDistinct(userID)`
   - `total_sessions_consumed = sum(usage_flag)`
   - `avg_balance_per_user = avg(current_balance)`

2. **Sink** → `DS_AzureSQL_fact_membership_kpis`

---

## Esquema Gold (Azure SQL Target)

### `fact_membership_activity`
```sql
CREATE TABLE fact_membership_activity (
    report_date           DATE          NOT NULL,
    membership_name       VARCHAR(100),
    active_subscribers    INT           DEFAULT 0,
    sessions_consumed     INT           DEFAULT 0,
    total_credits_granted INT           DEFAULT 0,
    etl_run_id           VARCHAR(50),
    PRIMARY KEY (report_date, membership_name)
);
```

---

## Visualización Sugerida (Power BI)

1. **Dashboard de Utilización**: Treemap de `membership_name` por `sessions_consumed`.
2. **Saldo Crítico**: Tabla de usuarios con `current_balance < 2` (Alertas de renovación).
3. **Tendencia de Crecimiento**: Línea de tiempo con `active_subscribers` MoM.

---

> [!NOTE]
> Las inconsistencias previas en los nombres de campos (`user` vs `userID`) han sido corregidas para garantizar la integridad referencial en el pipeline ETL.
