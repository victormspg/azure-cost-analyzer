# Azure Cost Analyzer (ACA)

A FinOps accelerator for **Microsoft Fabric** that turns raw **Azure Cost Management FOCUS**
exports into a governed, analytics-ready **medallion lakehouse**, a **Direct Lake semantic model**,
and — optionally — a **conversational Data Agent** and a lightweight **web app**.

Built on the open **[FinOps FOCUS](https://focus.finops.org/)** billing specification, so it
works with any Azure billing account and adapts to **whatever cost-allocation tags you
actually use** — nothing is hard-coded.

---

## What you get

| Layer | Artifact | Purpose |
|---|---|---|
| **Ingestion** | `AzureCostAnalyzer_Pipeline` (Data Factory) | Loads FOCUS exports → silver → gold, on a schedule |
| **Lakehouse** | `AzureCostAnalyzer_LH` | Bronze/silver/gold Delta tables (medallion) |
| **Model** | `AzureCostAnalyzer_SM` (Direct Lake on SQL) | 13 tables, relationships, ~30 measures |
| **Data Agent** *(optional)* | `AzureCostAnalyzer_Agent` | Natural-language Q&A over the model |
| **App** *(optional)* | RayFin web app *(separate repo)* | Cost, chargeback, governance & anomaly views |

You can stop at the **model** (use it from Power BI, Excel, or any XMLA client), or add the
Data Agent and app for a turnkey experience.

---

## Key capabilities

- **Cost overview** — effective / billed / list cost, savings, month-over-month trend.
- **Dynamic tag chargeback** — group and filter cost by *any* combination of your own tags,
  with **no double counting** (one row per resource-month).
- **Governance** — untagged cost / resource count, tag coverage, inline remediation targets.
- **Reservations & savings plans** — coverage %, utilization %, idle (wasted) commitment cost,
  per-commitment detail so you know *which* reservation to right-size.
- **Anomaly detection** — rolling z-score spike detection per subscription/service.
- **FOCUS-native** — all metrics derive from FOCUS columns + a few Azure `x_*` extensions.

---

## Repository layout

```
aca-accelerator/
├── README.md                     ← you are here
├── LICENSE
├── .gitignore
├── docs/
│   ├── deployment-guide.md       ← end-to-end setup (start here)
│   ├── architecture.md           ← components, data flow, identity/security
│   └── data-model.md             ← every table, column, relationship & measure
└── notebooks/
    ├── README.md                 ← notebook DAG, parameters, run order
    ├── 00_Deploy_ACA_Pipeline.ipynb
    ├── 01_Load_CostManagement_Focus_Data.ipynb
    ├── 02_Gold_Calendar.ipynb
    ├── 03_Gold_FocusMonthly.ipynb
    ├── 04_Gold_CostSummary.ipynb
    ├── 05_Gold_ByTag.ipynb
    ├── 06_Gold_Reservations.ipynb
    ├── 07_Gold_Anomalies.ipynb
    ├── 08_Gold_CostByResource.ipynb
    └── 09_Build_SemanticModel.ipynb
```

---

## Quickstart

1. **Prerequisites** — a Fabric capacity, a workspace, and an Azure Cost Management **FOCUS export**
   landing in ADLS Gen2 (or OneLake). See [docs/deployment-guide.md](docs/deployment-guide.md).
2. **Import** the 10 notebooks in `notebooks/` into your workspace (keep the exact names).
3. **Create** a Lakehouse named `AzureCostAnalyzer_LH` and point a shortcut at your FOCUS export.
4. **Run** `00_Deploy_ACA_Pipeline` to build the ingestion pipeline, then run the pipeline.
5. **Run** `09_Build_SemanticModel` to create the Direct Lake model.
6. *(Optional)* Wire up the **Data Agent** and/or the **RayFin app**.

Full step-by-step instructions, identity/security setup, and a validation checklist are in the
**[deployment guide](docs/deployment-guide.md)**.

---

## Design principles

- **Credential-free & portable** — every notebook uses parameterized placeholders; no tenant,
  workspace, capacity, or resource IDs are baked in. Bind the Lakehouse at import time.
- **Idempotent** — every notebook is safe to re-run (create-or-update, drop-then-readd measures).
- **Dynamic by data** — tag dimensions are discovered from the FOCUS `Tags` JSON at runtime, so the
  same code fits any tenant's tagging strategy.
- **Direct Lake first** — the model reads Delta directly through the SQL analytics endpoint (no
  import/refresh lag). Consumers can connect via **single sign-on (SSO)** or a **fixed service
  principal** identity — both approaches are covered in the [deployment guide](docs/deployment-guide.md).

---

## License

Released under the [MIT License](LICENSE).
