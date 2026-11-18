---
name: adatok
description: Use when the user asks a question that Forest CRM data can answer, about partners or companies (partnerek, cégek), contacts (kapcsolattartók), opportunities (lehetőségek), activities or tasks (tevékenységek, feladatok), invoices (számlák), sales reps (felelős, kereskedő), pipeline, revenue or counts. The data is in the Forest DWH, BigQuery project forestcrm-report-data, dataset public.
---

Always respond to the user in Hungarian.

# Querying the Forest DWH

Answer the user's question with read-only SQL against the `*_wide_hu` views of
`forestcrm-report-data.public`, through the BigQuery connector's `execute_sql_readonly` tool
(`projectId`: `forestcrm-report-data`).

## Steps

Steps 1 and 2 run once per conversation. Steps 3 to 6 run for every question.

### 1. Preflight

Run the `forest-check` skill. If it fails, stop: its message tells the user what to fix.

### 2. Select the tenant

Every question is about exactly one tenant. Row-level access control in BigQuery already limits
each user to the rows they may see, but some users can see several tenants. List them:

```sql
SELECT DISTINCT `tenant_id`, `subscription`
FROM `forestcrm-report-data.public.companies_wide`
ORDER BY `tenant_id`
```

`subscription` is the tenant's name (the Forest customer company).

**Wording:** never say "tenant", "DWH" or "subscription" to the user. In Hungarian, a tenant is a
"Forest-előfizetés" (or just name the company).

- **No rows:** tell the user that their account cannot see any Forest data, and to ask Forest
  for access. Stop.
- **One row:** use that tenant without asking. Name it once in your first answer, e.g.
  "(Példa Kft adatai alapján)".
- **Several rows and the question does not name a tenant:** ask the user to pick. If you have a
  tool that shows the user clickable options, use it; write the plain-text list below only when
  you have no such tool. Ask "Melyik Forest-előfizetés adatairól szól a kérdés?" and label each
  option `subscription (tenant_id)`, e.g. "Példa Kft (1234)". Accept the name, part of the name,
  or the ID.

Keep the selected tenant for the rest of the conversation. Switch only when the user asks for
another tenant. Then name the new tenant in the answer.

### 3. Pick the view

Use these views unless the user explicitly asks for another table:

| View | One row per | Also contains |
|---|---|---|
| `companies_wide_hu` | partner (company) | |
| `contacts_wide_hu` | contact person | their partner |
| `opportunities_wide_hu` | opportunity | its partner (no contact data) |
| `activities_wide_hu` | activity (task) | its partner, contact and opportunity |
| `invoices_wide_hu` | invoice | its opportunity and partner |

Prefer one view over joins: the wide views already carry the related data. Count each entity in
its own view. For example, count partners in `companies_wide_hu`, not in `activities_wide_hu`,
where a partner repeats once per activity.

### 4. Look up the columns

Read [schema.md](schema.md) before your first query in the conversation. Take every column name
from it. Do not query `INFORMATION_SCHEMA` or table metadata up front.

### 5. Write and run the query

Every query must have all of these:

- `WHERE tenant_id = <selected tenant>`, in every query and every subquery or CTE that reads a view.
- Fully qualified names: `` `forestcrm-report-data.public.<view>` ``.
- Only the columns you need. No `SELECT *`.
- No `authorized_emails` anywhere in the output. It is the access-control column.
- Aggregation in SQL (`COUNT`, `SUM`, `GROUP BY`) instead of fetching rows to count them yourself.
  The tool returns at most 3,000 rows.
- A `LIMIT` on queries that list individual records (default 50). If the limit cuts the result,
  say so and give the total count.
- Relative dates against Budapest time: `CURRENT_DATE('Europe/Budapest')`.

Example: activities per sales rep and type, modified in the last 7 days:

```sql
SELECT `tevekenyseg_felelos`, `tevekenyseg_tipus`, COUNT(*) AS db
FROM `forestcrm-report-data.public.activities_wide_hu`
WHERE `tenant_id` = 1234
  AND `tevekenyseg_modositasi_datum` >= DATE_SUB(CURRENT_DATE('Europe/Budapest'), INTERVAL 7 DAY)
GROUP BY 1, 2
ORDER BY db DESC
```

### 6. Answer

- Answer in Hungarian, in the user's terms. Lead with the answer, then the supporting numbers.
- Show amounts with the currency from `..._penznem` (e.g. `lehetoseg_penznem`, `szamla_penznem`).
  A tenant uses one currency, so do not sum across currencies.
- Leave out `..._id` columns unless the user asks. They are internal Forest IDs.
- When you list individual records, the `..._forest_link` column links to the record in Forest.
  Include it when the user will want to open the records.
- Do not show the SQL unless the user asks for it.

## Column meanings

| Column pattern | Meaning |
|---|---|
| `..._felelos` | Owner: an employee of the tenant company ("felelős", "kereskedő", "salesrep"). |
| `..._letrehozo`, `..._modosito` | User who created or last modified the record. |
| `..._modositasi_datum` / `_idopont` | Last change in **Forest** (the source system), not the DWH load time. |
| `..._datum` vs `..._idopont` | `DATE` vs `TIMESTAMP` of the same event. Filter and group on `_datum`. |
| `..._kategoria_lista` | Categories as one string, separated by `" \| "`. Filter with `LIKE '%<category>%'`. |
| `..._id` | Internal Forest IDs. Use for joins and counting distinct records. |

## Values seen in the data

Snapshot from 2026-11-17, across all tenants. Before filtering on other values, check them with
`SELECT DISTINCT` for the selected tenant: status labels can differ per tenant.

- `lehetoseg_statusz` (English codes): `lead`, `offer`, `contract`, `contract_expired`.
  The Forest UI labels them: lead = "Lehetőség", offer = "Ajánlat", contract = "Szerződés",
  contract_expired = "Lejárt szerződés". Use these labels in answers.
- `partner_statusz`: `Új`, `Aktív`, `Inaktív`, `Archív`, `Élő szerződés`.
- `tevekenyseg_statusz`: `Nyitott`, `Teljesített`, `Lemondott`.

## Data freshness

The views hold the state at the **end of the previous day**. They refresh daily, and operations
monitors failures. Do not warn about staleness. If the user asks about today's changes, explain
that today's data appears tomorrow.

## When a query fails

| Error contains | Do this |
|---|---|
| `Unrecognized name` or `Not found` (view or column) | `schema.md` is out of date. Look up that one view with the connector's `get_table_info`, fix the query, retry. Tell the user nothing unless the retry also fails. |
| `Access Denied`, `permission`, `401`, `token` | Run `forest-check` again and follow its messages. |
| Syntax error | Fix the SQL and retry once. |
| Anything else | Tell the user, in Hungarian, that the query failed, and quote the error. |
