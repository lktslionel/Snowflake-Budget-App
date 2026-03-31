
# Snowflake Proactive Cost Circuit-Breaker — Design Document

## 1. Overview

This solution implements a **proactive cost enforcement tool** that automatically blocks usage of cost-incurring Snowflake features when a budget threshold is breached. It uses **only native Snowflake capabilities**: Alerts, Stored Procedures, RBAC (Grants/Revocations), Budgets, and Resource Monitors — requiring no external infrastructure.

---

## 2. Architecture

```text
┌─────────────────┐
│  Snowflake       │
│  Budget/Monitor  │──── threshold crossed?
└───────┬─────────┘
        │ YES
        ▼
┌─────────────────┐
│  ALERT object    │  (serverless, runs on CRON)
│  + Stored Proc   │
└───────┬─────────┘
        │ calls
        ▼
┌─────────────────────────────────┐
│  SP: BUDGET_CIRCUIT_BREAKER()   │
│                                 │
│  1. Verify budget exceeded      │
│  2. Log to audit table          │
│  3. SUSPEND warehouses          │
│  4. SUSPEND tasks/pipes         │
│  5. REVOKE USAGE grants         │
│  6. Send notification           │
└─────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────┐
│  SP: BUDGET_CIRCUIT_RESTORE()   │
│  (manual or next billing cycle) │
│                                 │
│  1. GRANT USAGE back            │
│  2. RESUME warehouses/tasks     │
│  3. Log restoration             │
└─────────────────────────────────┘
```

---

## 3. Core Components

### 3.1 Budget Alert (Trigger)

Snowflake **Budgets** or **Resource Monitors** detect when spending hits a configurable threshold. Budgets cover account-wide or object-level spending. Resource Monitors cover warehouse-level compute.

### 3.2 Alert Object (Polling Engine)

A Snowflake **Alert** combines three elements:

| Element   | Role                                                        |
|-----------|-------------------------------------------------------------|
| Schedule  | CRON expression (e.g., every 5 minutes)                     |
| Condition | SQL query against budget/usage views — returns rows if breached |
| Action    | SQL call to the circuit-breaker stored procedure            |

**Execution per tick:**

```text
SCHEDULER fires
  → CONDITION query runs (serverless, near-zero cost)
    → 0 rows returned  → STOP (no action)
    → ≥1 rows returned → ACTION fires (calls stored procedure)
```

### 3.3 Budget Data Sources

| Source | Latency | Granularity |
|--------|---------|-------------|
| `SNOWFLAKE.LOCAL.ACCOUNT_ROOT_BUDGET` | ~1–2 hours | Account-level |
| `SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS` | ~minutes | Per-resource-monitor |
| `SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY` | ~1–2 hours | Per-warehouse, per-hour |
| `SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY` | ~1–2 hours | Per-warehouse, per-day |

The **condition query** reads from these views and evaluates: `Has (spent / limit) exceeded the threshold percentage?`

### 3.4 Privilege Revocation (Enforcement)

The stored procedure uses native RBAC to block all cost vectors:

| Cost Feature | Enforcement Method |
|---|---|
| **Warehouses** | `ALTER WAREHOUSE ... SUSPEND` + `REVOKE USAGE ON WAREHOUSE` |
| **Serverless Tasks** | `ALTER TASK ... SUSPEND` |
| **Snowpipe** | `ALTER PIPE ... SET PIPE_EXECUTION_PAUSED = TRUE` |
| **Materialized Views** | `ALTER MATERIALIZED VIEW ... SUSPEND RECLUSTER` |
| **Cortex/ML Functions** | Revoke grants on specific functions or database access |
| **Auto-Clustering** | `ALTER TABLE ... SUSPEND RECLUSTER` |

### 3.5 Recovery Path

A separate stored procedure (`BUDGET_CIRCUIT_RESTORE`) re-enables access:
- Invoked manually by an admin or automatically at the next billing cycle.
- Re-grants all revoked privileges, resumes suspended objects.
- All actions are logged to the audit table.

### 3.6 Notification Layer

When enforcement fires, the stored procedure sends notifications via a **Notification Integration**:

```text
Circuit Breaker SP
       ├──► Email (EMAIL notification integration)
       ├──► Slack/Webhook (WEBHOOK notification integration)
       └──► AWS SNS / Azure Event Grid / GCP Pub/Sub
```

### 3.7 Audit Table

Every enforcement and restoration action is logged to `BUDGET_AUDIT_LOG`:
- Timestamp, action type, object affected, role used, trigger reason.
- Provides full traceability for compliance and debugging.

### 3.8 Configuration Table

A `CIRCUIT_BREAKER_CONFIG` table defines:
- Threshold percentage (e.g., 90%).
- Excluded warehouses/roles (e.g., the admin warehouse itself).
- Which cost features to enforce (granular toggle per feature type).

---

## 4. Timing & Responsiveness

```text
Time ─────────────────────────────────────────────────►

Budget:  40%     55%     72%     85%     91%
          │       │       │       │       │
Alert:   ✓skip  ✓skip  ✓skip  ✓skip  ✗FIRE!
         5min    5min    5min    5min     │
                                         ▼
                                   CIRCUIT BREAKER
                                   PROCEDURE RUNS
```

- **Polling interval**: Configurable (1 min to hours). 5 minutes recommended.
- **Alert evaluation**: Serverless — no warehouse cost when condition is not met.
- **Idempotency**: Procedure checks current state before acting; safe to re-execute.
- **Re-fire prevention**: Once enforced, a status flag prevents redundant execution on subsequent ticks.

---

## 5. Security Model

| Principle | Implementation |
|---|---|
| **Least privilege** | Dedicated `COST_ADMIN` role with only `MANAGE GRANTS` + `MANAGE WAREHOUSES` |
| **Separation of duties** | `COST_ADMIN` cannot query business data — only manage access |
| **Audit trail** | All actions logged with timestamp, actor, and affected objects |
| **No secrets in code** | No credentials stored; runs within Snowflake's native auth context |

---

## 6. Why Not Just Resource Monitors?

| Capability | Resource Monitor | This Solution |
|---|---|---|
| Suspend warehouses | ✅ | ✅ |
| Suspend Tasks | ❌ | ✅ |
| Pause Pipes | ❌ | ✅ |
| Revoke role grants | ❌ | ✅ |
| Suspend reclustering | ❌ | ✅ |
| Block Cortex/Serverless | ❌ | ✅ |
| Custom logic/exclusions | ❌ | ✅ |
| Audit logging | ❌ | ✅ |
| Auto-restore path | ❌ | ✅ |

---

## 7. Prerequisites

- A role with `MANAGE GRANTS`, `MANAGE WAREHOUSES`, and access to `SNOWFLAKE.ACCOUNT_USAGE` (typically granted from `ACCOUNTADMIN`).
- Snowflake Alerts feature (GA).
- Budgets or Resource Monitors configured with thresholds.
- A Notification Integration (for admin alerts).

---

## 8. Known Limitations & Mitigations

### 8.1 Data Latency in Account Usage Views

| Limitation | `ACCOUNT_USAGE` views have a **1–2 hour data lag**. Spending could overshoot the threshold before detection. |
|---|---|
| **Impact** | The circuit breaker may fire late, allowing 1–2 hours of excess spending. |
| **Mitigation** | Set the alert threshold **well below** the hard budget limit (e.g., alert at 80%, hard limit at 100%). Combine with **Resource Monitors** which update faster and provide a native first line of defense for warehouse compute. Use `SNOWFLAKE.ORGANIZATION_USAGE` views where available for improved freshness. |

### 8.2 Alert Polling Interval Gap

| Limitation | Between polling ticks, a burst of spending can occur undetected. |
|---|---|
| **Impact** | At a 5-minute interval, up to 5 minutes of uncontrolled compute is possible after a threshold breach. |
| **Mitigation** | Reduce polling interval to **1 minute** for critical environments. Accept that sub-minute enforcement is not possible with native Alerts. For warehouse compute specifically, layer a **Resource Monitor** with `SUSPEND_IMMEDIATE` as a complementary safeguard. |

### 8.3 Serverless Feature Costs Are Hard to Preempt

| Limitation | Some serverless features (Snowpipe Streaming, Search Optimization, auto-clustering) incur costs that can only be stopped **after** they start, not prevented proactively via RBAC. |
|---|---|
| **Impact** | You can suspend these features but cannot prevent the cost of work already in flight. |
| **Mitigation** | Maintain a registry of all serverless features in the config table. The procedure suspends them immediately upon breach. For Search Optimization, use `ALTER TABLE ... DROP SEARCH OPTIMIZATION`. Accept that in-flight work completes at cost. |

### 8.4 No True Event-Driven Trigger

| Limitation | Snowflake Alerts are **poll-based**, not event-driven. There is no native mechanism to fire a procedure the instant a budget threshold is crossed. |
|---|---|
| **Impact** | Reaction time is bounded by `polling_interval + data_latency`. |
| **Mitigation** | If sub-minute response is required, introduce an **external component**: Budget Notification Integration → AWS Lambda / Azure Function → Snowflake API call to invoke the stored procedure. This adds external infra but closes the latency gap. |

### 8.5 MANAGE GRANTS Is a Powerful Privilege

| Limitation | The `COST_ADMIN` role requires `MANAGE GRANTS`, which is a broad privilege that could be misused if the role is compromised. |
|---|---|
| **Impact** | A compromised `COST_ADMIN` role could revoke arbitrary grants. |
| **Mitigation** | Lock `COST_ADMIN` to a **service account** with no interactive login. Apply `NETWORK POLICY` restrictions to limit access. Enable `ACCESS_HISTORY` monitoring on the role. Use `GRANT ... WITH GRANT OPTION` sparingly. Regularly audit role membership. |

### 8.6 Restoring State After Enforcement

| Limitation | The restore procedure must know **exactly** what was revoked. If grants were modified between enforcement and restoration, the restore could grant more or less than the original state. |
|---|---|
| **Impact** | Potential privilege drift after a circuit-breaker cycle. |
| **Mitigation** | The enforcement procedure **snapshots** the current grants state to the audit table before revoking. The restore procedure reads from this snapshot, not from a hardcoded list. This ensures exact restoration regardless of interim changes. |

### 8.7 Cross-Region / Multi-Account Gaps

| Limitation | This solution operates within a **single Snowflake account**. Organizations with multiple accounts need per-account deployment. |
|---|---|
| **Impact** | No centralized enforcement across accounts. |
| **Mitigation** | Use **Snowflake Organization Budgets** for cross-account visibility. Deploy the circuit breaker to each account via scripted automation (Terraform, Snowflake CLI). Centralize notification to a shared Slack channel or PagerDuty integration. |

### 8.8 Alert Object Requires a Running Warehouse (for Action)

| Limitation | While the **condition** evaluates serverlessly, the **action** (calling the stored procedure) requires a warehouse. If all warehouses are already suspended by a previous enforcement, the alert cannot fire again. |
|---|---|
| **Impact** | A race condition where the circuit breaker cannot run because its own warehouse is suspended. |
| **Mitigation** | Designate a **small, dedicated admin warehouse** that is **excluded** from the circuit breaker's suspension list. This warehouse is used only for alert actions and restore operations. Its cost is negligible (XS, auto-suspend 60s). |

---

## 9. Packaging as a Snowflake Native App with Snowpark Container Services

This section describes how to package the circuit-breaker as a **distributable Snowflake Native App** — installable by any Snowflake customer from the Marketplace or via private listing — with an always-on monitoring backend powered by **Snowpark Container Services (SPCS)**.

### 9.1 Why a Native App?

| Benefit | Description |
|---|---|
| **Zero external infra** | Consumer installs entirely within their Snowflake account |
| **Marketplace distribution** | Publish once, any customer can install with one click |
| **Security sandbox** | App runs in an isolated application database with explicit privilege grants from the consumer |
| **Versioning & upgrades** | Provider pushes new versions; consumers upgrade in-place |
| **Monetization** | Optional paid listing via Snowflake Marketplace billing |

### 9.2 High-Level Architecture (Native App + SPCS)

```text
┌─────────────────────────────────────────────────────────────┐
│                    CONSUMER ACCOUNT                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              NATIVE APP (installed)                     │  │
│  │                                                        │  │
│  │  ┌──────────────┐   ┌──────────────────────────────┐   │  │
│  │  │  Streamlit   │   │  SPCS Container Service      │   │  │
│  │  │  Dashboard   │   │  ┌────────────────────────┐  │   │  │
│  │  │  (Config UI) │   │  │  Monitor Loop (Python) │  │   │  │
│  │  │              │   │  │  - Poll budget views    │  │   │  │
│  │  │  • Thresholds│   │  │  - Evaluate rules      │  │   │  │
│  │  │  • Exclusions│   │  │  - Call enforcement SPs │  │   │  │
│  │  │  • Audit log │   │  │  - Send notifications   │  │   │  │
│  │  │  • Status    │   │  │  - Health heartbeat     │  │   │  │
│  │  └──────────────┘   │  └────────────────────────┘  │   │  │
│  │                     └──────────────────────────────┘   │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │           APP-OWNED OBJECTS                       │  │  │
│  │  │  • CIRCUIT_BREAKER_CONFIG (table)                 │  │  │
│  │  │  • BUDGET_AUDIT_LOG (table)                       │  │  │
│  │  │  • GRANT_SNAPSHOT (table)                         │  │  │
│  │  │  • BUDGET_CIRCUIT_BREAKER() (stored proc)         │  │  │
│  │  │  • BUDGET_CIRCUIT_RESTORE() (stored proc)         │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                          │                                    │
│            Consumer grants privileges to app:                │
│            • MANAGE GRANTS                                   │
│            • MANAGE WAREHOUSES                               │
│            • IMPORTED PRIVILEGES on SNOWFLAKE db             │
│            • CREATE COMPUTE POOL (for SPCS)                  │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 Why SPCS Instead of Alerts?

The Alert-based approach (Section 3.2) works well for standalone deployment. SPCS adds value for the Native App packaging:

| Concern | Alert-Based | SPCS-Based |
|---|---|---|
| **Polling frequency** | Minimum ~1 min CRON | Sub-second loop possible |
| **Complex logic** | Limited to SQL | Full Python (rule engine, ML anomaly detection) |
| **Stateful monitoring** | Stateless per tick | In-memory state, trend tracking, rate-of-change detection |
| **Multi-signal correlation** | Requires multiple alerts | Single loop correlates warehouses + tasks + pipes |
| **Health monitoring** | No self-awareness | Heartbeat endpoint, auto-restart on failure |
| **UI backend** | N/A | Serves Streamlit API callbacks |
| **Extensibility** | New SQL per feature | Plugin architecture in Python |

### 9.4 SPCS Container Design

```text
┌─────────────────────────────────────────┐
│         SPCS Service Container           │
│         (Python application)             │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │         Main Monitor Loop          │  │
│  │                                    │  │
│  │  while True:                       │  │
│  │    budget = query_budget_views()   │  │
│  │    config = read_config_table()    │  │
│  │    state  = evaluate_rules(        │  │
│  │               budget, config,      │  │
│  │               historical_trend)    │  │
│  │                                    │  │
│  │    if state.breached:              │  │
│  │      call_circuit_breaker_sp()     │  │
│  │      send_notification()           │  │
│  │                                    │  │
│  │    write_heartbeat()               │  │
│  │    sleep(interval)                 │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │      Advanced Detection Engine     │  │
│  │                                    │  │
│  │  • Rate-of-change analysis         │  │
│  │    (spending acceleration)         │  │
│  │  • Anomaly detection               │  │
│  │    (unusual warehouse patterns)    │  │
│  │  • Predictive breach estimation    │  │
│  │    (ETA to threshold at current    │  │
│  │     burn rate)                     │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │        Health / API Endpoint       │  │
│  │                                    │  │
│  │  GET /health   → 200 OK           │  │
│  │  GET /status   → enforcement state │  │
│  │  POST /restore → trigger restore  │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 9.5 Native App Project Structure

```text
cost-circuit-breaker/
├── manifest.yml
├── setup_script.sql
├── README.md
│
├── streamlit/
│   └── dashboard.py
│
├── containers/
│   ├── Dockerfile
│   ├── monitor/
│   │   ├── main.py
│   │   ├── rules_engine.py
│   │   ├── enforcement.py
│   │   ├── notifications.py
│   │   └── health.py
│   └── requirements.txt
│
├── stored_procedures/
│   ├── circuit_breaker.sql
│   ├── circuit_restore.sql
│   └── grant_snapshot.sql
│
├── tables/
│   ├── config.sql
│   ├── audit_log.sql
│   └── grant_snapshot.sql
│
└── service/
    └── spec.yml
```

### 9.6 Key manifest.yml Privileges

The Native App must declare the privileges it needs from the consumer:

```yaml
manifest_version: 1
artifacts:
  setup_script: setup_script.sql
  default_streamlit: streamlit/dashboard.py

privileges:
  - MANAGE GRANTS:
      description: "Required to revoke/restore grants on cost-incurring objects"
  - MANAGE WAREHOUSES:
      description: "Required to suspend/resume warehouses"
  - IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE:
      description: "Required to read ACCOUNT_USAGE budget and metering views"
  - CREATE COMPUTE POOL:
      description: "Required to run the SPCS monitoring container"
  - CREATE WAREHOUSE:
      description: "Required to create the dedicated admin warehouse"

references:
  - NOTIFICATION_INTEGRATION:
      label: "Notification target for alerts"
      description: "Email, webhook, or cloud notification integration"
      privileges: [USAGE]
```

### 9.7 SPCS Service Specification (spec.yml)

```yaml
spec:
  containers:
    - name: cost-monitor
      image: /db/schema/app_repo/cost-monitor:latest
      resources:
        requests:
          cpu: 0.5
          memory: 256M
        limits:
          cpu: 1
          memory: 512M
      env:
        POLL_INTERVAL_SECONDS: "60"
        LOG_LEVEL: "INFO"
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 30
  endpoints:
    - name: monitor-api
      port: 8080
      public: false
```

### 9.8 Setup Script Flow (on install)

```sql
-- 1. Create app-owned schema and objects
CREATE SCHEMA IF NOT EXISTS core;
CREATE TABLE core.circuit_breaker_config (...);
CREATE TABLE core.budget_audit_log (...);
CREATE TABLE core.grant_snapshot (...);

-- 2. Create stored procedures
CREATE PROCEDURE core.budget_circuit_breaker() ...;
CREATE PROCEDURE core.budget_circuit_restore() ...;

-- 3. Create image repository and compute pool
CREATE IMAGE REPOSITORY IF NOT EXISTS app_repo;
CREATE COMPUTE POOL IF NOT EXISTS cost_monitor_pool
  MIN_NODES = 1 MAX_NODES = 1
  INSTANCE_FAMILY = CPU_X64_XS
  AUTO_SUSPEND_SECS = 0;

-- 4. Create the SPCS service
CREATE SERVICE core.cost_monitor_service
  IN COMPUTE POOL cost_monitor_pool
  FROM @app_stage/service
  SPECIFICATION_FILE = 'spec.yml';

-- 5. Create dedicated admin warehouse (excluded from enforcement)
CREATE WAREHOUSE IF NOT EXISTS COST_ADMIN_WH
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- 6. Grant Streamlit access
GRANT USAGE ON SCHEMA core TO APPLICATION ROLE app_user;
```

### 9.9 Consumer Installation Experience

```text
1. Find "Cost Circuit-Breaker" on Snowflake Marketplace
2. Click "Get" → Review requested privileges → "Grant"
3. App installs → Streamlit dashboard opens automatically
4. Configure:
   ├── Set budget threshold (e.g., 90%)
   ├── Select features to enforce (warehouses, tasks, pipes, etc.)
   ├── Add exclusions (admin warehouse, critical pipelines)
   └── Connect notification integration
5. Click "Activate" → SPCS monitor starts polling
6. View real-time status and audit history in dashboard
```

### 9.10 Native App Specific Limitations

| Limitation | Impact | Mitigation |
|---|---|---|
| **SPCS compute cost** | The always-on container incurs SPCS credits (~0.5 credits/hr for XS) | Use smallest instance family; expose config to pause/resume the service during off-hours |
| **Privilege escalation surface** | `MANAGE GRANTS` inside an app context is powerful | App runs in a sandboxed database; consumer reviews privileges before granting; all actions are audited |
| **Image updates require provider push** | Bug fixes or new features need a new app version | Use semantic versioning; enable auto-upgrade for patch versions |
| **SPCS cold start** | If compute pool scales to 0, first poll has ~60s delay | Set `AUTO_SUSPEND_SECS = 0` for always-on, or accept the cold start trade-off for cost savings |
| **Consumer must own a compute pool** | Some consumers may not have SPCS enabled | Offer a **fallback mode** using native Alerts (Section 3.2) as an alternative when SPCS is unavailable |
| **Cross-account app sharing** | Each account needs its own installation | Distribute via Marketplace listing; provide Terraform module for bulk deployment |

---

## 10. References

- [Snowflake Budgets](https://docs.snowflake.com/en/user-guide/budgets)
- [Snowflake Resource Monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
- [Snowflake Alerts](https://docs.snowflake.com/en/user-guide/alerts)
- [Snowflake Stored Procedures](https://docs.snowflake.com/en/sql-reference/stored-procedures-overview)
- [GRANT / REVOKE Privileges](https://docs.snowflake.com/en/sql-reference/sql/grant-privilege)
- [Notification Integrations](https://docs.snowflake.com/en/user-guide/notifications/notification-integrations)
- [Account Usage Views](https://docs.snowflake.com/en/sql-reference/account-usage)
- [Metering History](https://docs.snowflake.com/en/sql-reference/account-usage/metering_history)
- [ALTER WAREHOUSE](https://docs.snowflake.com/en/sql-reference/sql/alter-warehouse)
- [ALTER TASK](https://docs.snowflake.com/en/sql-reference/sql/alter-task)
- [ALTER PIPE](https://docs.snowflake.com/en/sql-reference/sql/alter-pipe)
- [Network Policies](https://docs.snowflake.com/en/user-guide/network-policies)
- [Snowflake Native Apps Framework](https://docs.snowflake.com/en/developer-guide/native-apps/native-apps-about)
- [Native App manifest.yml Reference](https://docs.snowflake.com/en/developer-guide/native-apps/creating-manifest)
- [Native App Setup Script](https://docs.snowflake.com/en/developer-guide/native-apps/creating-setup-script)
- [Snowpark Container Services Overview](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview)
- [SPCS Service Specification](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/specification-reference)
- [CREATE COMPUTE POOL](https://docs.snowflake.com/en/sql-reference/sql/create-compute-pool)
- [CREATE SERVICE](https://docs.snowflake.com/en/sql-reference/sql/create-service)
- [Snowflake Marketplace Publishing](https://docs.snowflake.com/en/developer-guide/native-apps/publishing)
