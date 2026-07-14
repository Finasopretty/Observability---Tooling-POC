# Observability & Tooling POC

A hands-on Azure sandbox implementing three platform capabilities: **system-health monitoring with alerting**, a **billing-metrics pipeline**, and **IaC-based tenant onboarding**. Built with Azure CLI, PowerShell, T-SQL, and Bicep.

> This is a personal learning sandbox. All identifiers, sample data, and business specifics are fictional/placeholder. No secrets are stored in this repo — the SQL admin password is generated at runtime and resolved from Azure Key Vault by reference.

## Repository layout

```
observability-tooling-poc/
├── README.md
├── .gitignore                          # blocks secrets & local files
├── bicep/
│   └── tenant.bicep                    # Phase 4: parameterized tenant onboarding module
├── sql/
│   ├── 01-schema.sql                   # Phase 3: dim_tenant, dim_billed_feature, fact_monthly_usage
│   ├── 02-seed.sql                     # sample data + reporting queries
│   └── 03-usp_load_monthly_usage.sql   # idempotent collection proc with dry-run
└── scripts/
    ├── setup-sandbox.ps1               # provisions the PaaS stack from scratch
    ├── collect-usage.ps1               # scheduled collection runner (PowerShell)
    └── function_app_reference.py       # production-correct Azure Function equivalent (reference)
```

## Environment constraint

Built on a free-trial subscription with **0 vCPU quota**, so no App Service or VM compute would deploy. The design routes around this by using PaaS services (SQL, Event Hub, Key Vault) that don't draw on vCPU quota, and by validating compute-dependent code with dry-runs (`what-if`, Bicep `deployWebApp=false`) rather than deploying it. Every pattern here is production-shaped; only the host would change in a real environment.

---

## Phase 0–1 — Foundation & monitored stack

`scripts/setup-sandbox.ps1` provisions everything:

- **Log Analytics workspace** — the central sink for all metrics/logs.
- **Azure SQL server + `tenantdb`** (Basic tier) — the SQL layer.
- **Event Hub namespace** — the event/microservice layer (metrics are namespace-level).
- **Key Vault** — holds the SQL admin secret, generated at runtime, never hardcoded.
- **Diagnostic settings** — ship `AllMetrics` from SQL and Event Hub into the workspace.

Resource providers are registered up front and polled until `Registered` to avoid piecemeal failures.

## Phase 2 — Alerting to support

Configured via CLI (see build notes / commands below):

- **Action Group** `support-oncall` with a confirmed email receiver (must be confirmed via Azure's email link or alerts silently don't deliver).
- **`sql-dtu-high`** — fires when SQL DTU averages > 80% over 5 min.
- **`eh-throttling`** — fires when the Event Hub namespace throttles requests.

Validated end to end by pinning the Basic-tier DB's DTU to 100% and watching the alert fire, resolve, and email.

```powershell
# Action group + alerts
az monitor action-group create -g $Rg -n "support-oncall" --short-name "support" `
  --action email SupportEmail "<your-email>"
$AgId    = az monitor action-group show -g $Rg -n "support-oncall" --query id -o tsv
$SqlDbId = az sql db show -g $Rg -s $SqlServer -n tenantdb --query id -o tsv
$EhId    = az eventhubs namespace show -g $Rg -n $EhNamespace --query id -o tsv

az monitor metrics alert create -g $Rg -n "sql-dtu-high" --scopes $SqlDbId `
  --condition "avg dtu_consumption_percent > 80" --window-size 5m `
  --evaluation-frequency 1m --severity 2 --action $AgId

az monitor metrics alert create -g $Rg -n "eh-throttling" --scopes $EhId `
  --condition "total ThrottledRequests > 0" --window-size 5m `
  --evaluation-frequency 1m --severity 2 --action $AgId
```

## Phase 3 — Billing metrics pipeline

Solves the **Client ID (int) vs Tenant ID (UUID)** gap: internal systems key on the UUID, billing keys on the int, and the int isn't reliably populated yet.

**Design:** usage facts reference `tenant_id`; billing reports select `client_id` off `dim_tenant` via the join. Backfilling a `client_id` is a **one-row update**, never a fact-table rewrite. `client_id` is nullable (a filtered unique index enforces "unique if present"), so `WHERE client_id IS NULL` is the live backfill worklist.

Run in order: `01-schema.sql` → `02-seed.sql` → `03-usp_load_monthly_usage.sql`.

The collection proc validates inputs, normalizes to first-of-month, upserts idempotently via `MERGE`, and **defaults to dry-run**. `scripts/collect-usage.ps1` calls it on a schedule, pulling the password from Key Vault; `function_app_reference.py` is the production-correct Azure Function equivalent.

```powershell
# Dry-run (default): logs and changes nothing
./scripts/collect-usage.ps1
# Commit: writes via the proc
./scripts/collect-usage.ps1 -Commit
```

## Phase 4 — Tenant onboarding (Bicep)

`bicep/tenant.bicep` provisions one tenant's stack from a single parameterized command. Uses the `existing` keyword to extend the shared SQL server (never redefine it), and a conditional `if (deployWebApp)` gate so one module serves both quota-limited and full deployments. `location` is a required parameter to force explicit region choice (a DB must match its server's region).

```powershell
$ServerRegion = az sql server show -g $Rg -n $SqlServer --query location -o tsv

# Preview, then deploy
az deployment group what-if -g $Rg --template-file ./bicep/tenant.bicep `
  --parameters tenantName=northgate sqlServerName=$SqlServer location=$ServerRegion

az deployment group create -g $Rg --template-file ./bicep/tenant.bicep `
  --parameters tenantName=northgate sqlServerName=$SqlServer location=$ServerRegion
```

---

## Problems solved (the real learning)

| Problem | Fix |
|---|---|
| Free-trial 0 vCPU quota | Route around with PaaS; validate compute code with dry-runs |
| Unregistered resource providers | Register up front, poll until `Registered` |
| Key Vault RBAC blocks owner | Assign `Key Vault Secrets Officer` to self |
| CLI version drift (EH retention, `--yes`) | Match flags to version; Portal for trivial resources |
| PowerShell execution policy blocks scripts | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| Cross-region SQL constraint | DB must match its server's region |
| Bicep `BCP120` | `existing.location` unresolvable at start — pass region as a parameter |

## Cleanup

```powershell
# Tear down the whole sandbox (irreversible)
az group delete --name azureuser --yes --no-wait
```

## Not included / next steps

- **Custom domain + SSL binding** — needs a real DNS-verifiable domain a sandbox can't provide.
- **Real collection logic** — `collect-usage.ps1` and the Function currently make one representative call; production would iterate real tenants/features from the source system or an ingested CSV.
- **Managed scheduling in Azure** — swap the local Task Scheduler runner for the Function or Elastic Jobs so it runs regardless of any single machine.
