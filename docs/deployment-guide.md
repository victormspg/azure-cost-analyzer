# Deployment guide

End-to-end setup for the Azure Cost Analyzer (ACA) accelerator on Microsoft Fabric. Follow the steps
in order. Everything is credential-free and idempotent — safe to re-run.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **Fabric capacity** | Any F-SKU (or Trial). The data and consumption workspaces must share one capacity. |
| **Fabric workspace** | Create one (or two for a locked-down topology — see [architecture.md](architecture.md#identity--security)). |
| **Azure permissions** | Cost Management Reader (to configure the FOCUS export) and access to the target storage account. |
| **Storage account (ADLS Gen2)** | Destination for the FOCUS export parquet. |
| **Service principal** *(for Option B)* | An Entra app registration used as the model's fixed-identity connection (configured in step 7). Optionally also used for the storage shortcut. |
| **Workspace Identity** *(optional)* | If you want Fabric to read the storage account under the workspace's own identity, enable **Workspace settings → Workspace identity** and grant it **Storage Blob Data Reader** on the storage account. |
| **Fabric permissions** | Member/Admin on the workspace to create items and connections. |

---

## 1. Configure the FOCUS export in Cost Management

Create the source data feed that the accelerator ingests.

1. In the Azure portal, open **Cost Management → Exports → Create**.
2. Choose the **FOCUS** dataset (cost and usage, FOCUS format).
3. Set the scope (billing account, subscription, or resource group) and a recurring schedule
   (daily recommended). Choose **file overwrite** so each run replaces the latest snapshot.
4. Set the format to **Parquet** with **Snappy** compression.
5. Point the destination at your **ADLS Gen2** storage account + container/path.
6. Run the export once and confirm parquet files land in the container.

See [Microsoft Learn: FOCUS exports](https://learn.microsoft.com/azure/cost-management-billing/dataset-schema/cost-usage-details-focus).

---

## 2. Create the Lakehouse

1. In your workspace, create a **Lakehouse** named **`AzureCostAnalyzer_LH`**.
   (Enable **schemas** if you want the tables under `dbo` — notebook `09` binds `dbo.<table>`.)
2. Note that the notebooks reference this lakehouse by name; keep the name exact or update the
   `%%configure` / parameter cells.

---

## 3. Point at your FOCUS export

Surface the raw export in the lakehouse **without copying** it:

1. In `AzureCostAnalyzer_LH` → **Files**, create a **OneLake shortcut** named `focuscost` to the
   ADLS Gen2 container/path holding your FOCUS parquet.
   - **Authentication** — the shortcut needs a storage connection. Options:
     - **Workspace identity** *(suggested)* — cleanest for production; grant the workspace identity
       **Storage Blob Data Reader** on the storage account (see Prerequisites).
     - **Organizational account** — your Entra user (must have storage read).
     - **Service principal** — with **Storage Blob Data Reader** on the container.
2. Confirm you can browse `Files/focuscost/**.parquet`.

> The default source path parameter is `Files/focuscost`. Override `RawSourcePath` if you use a
> different location.

---

## 4. Import the notebooks

1. Import all 10 notebooks from `notebooks/` into the workspace, **keeping the exact display names**
   (`00_Deploy_ACA_Pipeline` … `09_Build_SemanticModel`). Notebook `00` looks the others up by name.
2. Open each notebook once and **attach the lakehouse** `AzureCostAnalyzer_LH` as the default
   (the `%%configure` cell carries a placeholder binding — attaching replaces it with the real one).

---

## 5. Deploy & run the pipeline

1. Open **`00_Deploy_ACA_Pipeline`** and run it top-to-bottom. It:
   - resolves the workspace + lakehouse,
   - discovers notebooks `01`–`08` by name,
   - creates/updates the Data Factory pipeline **`AzureCostAnalyzer_Pipeline`** (DAG + parameters).
2. Run the pipeline once (set `RUN_AFTER_DEPLOY = True`, or open the pipeline and click **Run**).
   Confirm all activities succeed.
3. Adjust the load window with the pipeline parameters `FromMonth` / `ToMonth` (month offsets from
   *today − 1 day*; `0` = current month).

### Validate the gold layer

In a notebook or the SQL endpoint, confirm the tables exist and carry data:

```python
for t in ["focus_partitioned","gold_cost_summary_daily","gold_cost_summary_monthly",
          "gold_cost_focus_monthly","gold_chargeback_by_tag","dim_tag_key",
          "gold_cost_by_resource","gold_reservations_coverage","gold_reservations_waste",
          "gold_reservations_detail","gold_cost_anomalies","dim_date","dim_month"]:
    print(f"{t:32s}{spark.table(t).count():>12,}")
```

Also inspect the **discovered tags**: `spark.table("dim_tag_key").orderBy("Rank").show()`.

---

## 6. Build the semantic model

1. Open **`09_Build_SemanticModel`** in the **same workspace** as the lakehouse and run it.
   It installs `semantic-link-labs`, then:
   - forces a SQL-endpoint metadata sync,
   - creates (or reuses + schema-syncs) the Direct-Lake-on-SQL model **`AzureCostAnalyzer_SM`**,
   - wires relationships, marks the date table, and (re)creates all measures,
   - refreshes (reframes) the model and runs a sanity DAX query.
2. The last cell prints the model's `workspaceId` + `itemId` — keep them for the optional app.

> Re-running `09` reuses the existing model (same `itemId`) and syncs its schema, so new tag columns
> flow through automatically.

---

## 7. Consumer connection — SSO or fixed identity

Choose how report/app consumers authenticate to the model's SQL analytics endpoint. Both work with
Direct Lake; pick per your governance needs.

### Option A — Single sign-on (SSO)  *(default, simplest)*

The model queries under each caller's own Entra identity — no extra connection to configure.

1. Leave the model's data source on **Single Sign-On (Entra ID)** (the default).
2. Grant each consumer read access to the underlying data (lakehouse / SQL endpoint) **plus**
   **Build + Read** on the model.

Use when consumers are already entitled to the raw data (e.g. the platform / FinOps team).

### Option B — Fixed identity (service principal)  *(recommended for broad sharing)*

Consumers see data **without** any lakehouse/storage permissions — access is brokered by one SP.

1. Open **`AzureCostAnalyzer_SM`** → **Settings** → **Gateway and cloud connections**.
2. The data source defaults to **Single Sign-On (Entra ID)**. Under *Maps to*, choose
   **Create a connection** — the server (`…datawarehouse.fabric.microsoft.com`) and database (a GUID)
   pre-fill; leave them.
3. Connection type **SQL Server**, authentication **Service Principal**, **SSO = OFF**.
4. Map the data source to the new connection → **Apply**.
5. Grant consumers **Build + Read** on the model (and Viewer/Contributor on the consumption
   workspace). They need **nothing** on the lakehouse, SQL endpoint, or storage.

> **Topology**: for broad sharing, keep a locked-down **Data workspace** and a separate
> **Consumption workspace** on the same capacity. See
> [architecture.md](architecture.md#identity--security) for the rationale and the RLS implications.

---

## 8. Schedule

- Open **`AzureCostAnalyzer_Pipeline`** → **Schedule** → set a cadence (e.g. daily 06:00 local), or
  chain it after an upstream pipeline with an *Invoke pipeline* activity.
- Optionally schedule **`09`** (or run it after schema changes) to reframe the model and pick up new
  tag columns.

---

## 9. (Optional) Data Agent

1. Create a **Data Agent** named **`AzureCostAnalyzer_Agent`** in the workspace.
2. Add **`AzureCostAnalyzer_SM`** as a data source.
3. Give it grounding instructions so it answers cost questions accurately. Starting point:

```text
You are AzureCostAnalyzer_Agent, a FinOps assistant over the AzureCostAnalyzer_SM semantic model
(Azure cost data in the FOCUS format). Answer questions about Azure spend, savings, tag-based
chargeback, reservations, and cost anomalies.

Guidance:
- Use `Total Effective Cost` for "cost" unless the user asks for billed or list cost. Savings =
  `Total Savings`; month-over-month = `MoM Cost %`.
- For "cost by <tag>" (team, environment, project, cost center, …), group the WIDE table
  `gold_chargeback_by_tag` by the matching tag column and use `Tag Effective Cost`. The available
  tag columns are listed in `dim_tag_key` — read it to discover the available tags; never assume a
  tag exists.
- Governance questions (untagged spend, coverage): use `Untagged Cost`, `Untagged %`,
  `Untagged Resource Count`, `Tag Count` from `gold_cost_by_resource`.
- Reservations: `RI Coverage %`, `Reservation Utilization %`, `RI Waste Cost`, and the per-commitment
  `gold_reservations_detail` for which reservation is idle.
- Anomalies: `Anomaly Count` and `gold_cost_anomalies` (IsAnomaly, ZScore, Severity).
- Always specify the currency and the time range in your answer. Use `dim_month` for monthly trends
  and `dim_date` for daily. If a question is ambiguous, ask a brief clarifying question.
- Never invent tag names, services, or figures that aren't in the model.
```

Tune the wording to your own tenant (currency, tag vocabulary).

---

## 10. (Optional) RayFin web app

The RayFin web app (cost overview, dynamic-tag chargeback, governance, and anomaly views on top of
`AzureCostAnalyzer_SM`) ships from a **separate repository**.

> 🚧 **Work in progress** — deployment steps for the app will be added here (or linked to the app
> repo) in a future update.

---

## Validation checklist

- [ ] `Files/focuscost` shortcut browses the FOCUS parquet.
- [ ] Pipeline runs green; all 13 tables populated.
- [ ] `dim_tag_key` lists your tenant's real tags, ranked by cost.
- [ ] `09` sanity DAX returns non-blank `Total Effective Cost`, `Untagged %`, `Tag Effective Cost`.
- [ ] *(Option B only)* A consumer with **no** lakehouse/storage access can open a report/app on the
      model and see data (fixed-identity connection working).
- [ ] Queries hit **Direct Lake** (VertiPaq), not DirectQuery fallback (Performance Analyzer).
