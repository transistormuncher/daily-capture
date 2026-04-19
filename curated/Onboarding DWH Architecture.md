# Onboarding — DWH Architecture

Cleaned from raw onboarding notes (captures up to 2026-04-15). Use at work — safe to copy into OneNote / Confluence as needed.

---

## Azure architecture

- SQL Server runs on an Azure VM
- Selektionstool runs on the **dev server** but accesses **production** data
- Tooling: SQL Management Studio
- **ZEIT CDH** (Customer Data Hub) — holds personenbezogene Daten; scope is being reduced. Still used by the selection component.
- **ZEIT DWH** — the main warehouse, layered as below.

## DWH layers

| Layer | Purpose |
|-------|---------|
| **STA** (staging) | 1:1 copy of source per day — always "what's new today" |
| **PSA** (persistent staging) | Historisation |
| **CDW** (core DWH) | First logic, renames, initial transforms; historised. Includes **VDW** (definitions of what belongs together) and **C-VDW** (current valid view) |
| **BLY** (business layer) | Long runtime, heavy logic. Joins across objects (CDW is more object-focused) |
| **ANA** (analytics) | — |
| **SEL** (selection) | — |

### Example (BLY): ABAB
- `v_fp` view for a persisted fact table — analysts write the view, it's written out once daily (historised)
- Control table `dwh_bly_ctrl` steers what runs

## b-telligent meta_dwh framework

How data moves from PSA → CDW:
- Control tables (in `meta_dwh`) hold the logic
- Views build a SQL statement dynamically, which is then executed
- Example: `meta_dwh.v_sql_load_...`
- Historically b-telligent managed everything up to CDW
- Triggering: **SSIS** or stored procedures via **SQL Server Agent Jobs**
- The Selektionskomponente is also scheduled this way (turned off at night)

**Key migration challenge:** translating this generated-SQL control-table logic into dbt.

See [[b-telligent]] — Stefan Rann and Andreas Mittelstedt built and maintained this; they're the contacts for detail questions.

## Engineering — JF notes

### dbt-zeit vs dbt-zeit-workflows (per Kalle)
- Christopher set up everything in Terraform for Zeit Schule (for Yasmin)
- This is the **blueprint** for further Mandanten
- Always **dev / staging / prod**
- `[TF]` prefix = created by Terraform
- Clone-to-staging: **1× daily**
- Clone-to-dev: **every weekend**

### Terraform deployment
- Terraform setup is new, created by Christopher
- Executed via **Atlantis**
- Used for rolling out dbt jobs

### Next steps
- Move Zeit to Terraform, then advise and remaining projects

## Open TODOs

- Add draft-PR flag to `cpr` command + add label
  - Consider interactive prompt: which label (draft vs normal), which execution steps (normal = modified+, minor = just modified, full = full refresh)
- Harmonise sqlfluff between projects
  - Collect fails: jinja, BigQuery dot notation, BigQuery `union by name`
  - Check sqlfluff tickets and Johannes' pre-hook ticket

## Auftragsdaten — orders model

### Bestellungen vs Abos (Z+)
- **Bestellung / Order:** tracked via the "Dankesseite" page load
  - Abo-Herkunft via tracking
  - Logic in the intermediate layer (`intermediate_orders` in ZON)
  - Every order has an associated **offer** (further info on origin/context)
- **Subscription funnel** exists but no one looks at it
- **DPV Abos:** real, confirmed Abos — but only available 3 days later
- Rule: *Abo = always a confirmed Abo*
- **Registrierung** is important for paid content (needed for commenting, following articles)
- **IAP** (In-App Purchase, Apple) is a special case
- **Entitlement:**
  - State immediately after order, before DPV confirmation
  - Also: Zeit employees
  - Print Abo → entitled to digital → "Print Plus": only entitled, no separate digital Abo entry
- `sso_id` = `customer_id`

### Key tables
- `core_fact_orders` — we create synthetic orders for Abos that don't have one
- `core_fact_subscription`

## Migration phase 5

Cleanup and consolidation of ZON and Zeit.
