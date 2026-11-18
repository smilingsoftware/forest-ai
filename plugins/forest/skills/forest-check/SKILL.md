---
name: forest-check
description: Use before the first query against the Forest DWH (BigQuery project forestcrm-report-data, dataset public) in a conversation, and whenever a BigQuery tool call fails with an access, authentication or "tool not found" error. Checks that the Google Cloud BigQuery connector is installed, connected, and has the permissions the DWH needs.
---

Always respond to the user in Hungarian.

# DWH preflight

The Forest DWH is reached only through the user's claude.ai **Google Cloud BigQuery** connector.
The plugin cannot install or connect it. This check finds out whether the user can query the DWH,
and if not, tells them exactly what to fix.

**Run it once per conversation.** If a probe in this conversation already succeeded, skip the check
unless a later BigQuery call fails with one of the errors in step 3.

## Step 1: Find the connector tool

The query tool is `mcp__claude_ai_Google_Cloud_BigQuery__execute_sql_readonly`.

1. If it is in your tool list, go to step 2.
2. If it is listed as a deferred tool, load it with ToolSearch
   (`select:mcp__claude_ai_Google_Cloud_BigQuery__execute_sql_readonly`), then go to step 2.
3. Otherwise search with ToolSearch for `bigquery execute_sql_readonly`. The prefix can differ
   between Claude surfaces. Accept any tool whose name contains `BigQuery` and ends in
   `__execute_sql_readonly`.
4. If the only BigQuery tool is an `authenticate`-style tool, the connector is added but not
   signed in. Send message **B** and stop.
5. If no BigQuery tool exists at all, send message **A** and stop.

## Step 2: Run the probe query

Call the query tool with exactly these arguments:

- `projectId`: `forestcrm-report-data`
- `query`: ``SELECT table_id FROM `forestcrm-report-data.public.__TABLES__` LIMIT 1``

This reads table metadata only and bills 0 bytes. It tests three things in one call: the connector
is signed in, the user can run jobs in the project, and the user can read the dataset.

If it returns a row, the check passed. Continue with the user's question. Do not mention the
check to the user.

## Step 3: Map the error to a message

Match the error text, in this order. Send the message, then stop: do not try to answer the
question another way.

| Error text contains | Cause | Message |
|---|---|---|
| `bigquery.jobs.create` | User may not run queries in the project | **C** |
| `does not have permission to query table` or `Not found: Dataset` | No read access to the `public` dataset (or it does not exist) | **D** |
| `401`, `unauthenticated`, `invalid_grant`, `token`, `re-authenticate`, `authorization` | Connector sign-in missing or expired | **B** |
| anything else | Unknown | **E** |

## Messages

Use these texts. Keep the English UI labels (Settings, Connectors, Connect) as they are.

**A: connector missing**

> A Forest adattárház eléréséhez a **Google Cloud BigQuery** connector szükséges, de ebben a
> munkamenetben nem érhető el.
>
> 1. Nyisd meg a claude.ai-t: **Settings → Connectors**.
> 2. Keresd meg a **Google Cloud BigQuery** connectort, majd kattints a **Connect** gombra.
> 3. Jelentkezz be azzal a Google-fiókkal, amely hozzáfér a Forest adattárházhoz.
> 4. Indíts új beszélgetést (Claude Code-ban indítsd újra a munkamenetet), és tedd fel újra a kérdést.

**B: connector not signed in**

> A **Google Cloud BigQuery** connector telepítve van, de nincs bejelentkezve, vagy lejárt a
> bejelentkezés.
>
> - claude.ai-on: **Settings → Connectors → Google Cloud BigQuery → Connect**.
> - Claude Code-ban: futtasd a `/mcp` parancsot, és jelentkezz be a BigQuery szerverhez.
>
> Ezután tedd fel újra a kérdést.

**C: no permission to run queries**

> A BigQuery connector működik, de a Google-fiókodnak nincs joga lekérdezést futtatni a
> `forestcrm-report-data` projektben.
>
> Kérd meg a Forest adattárház adminisztrátorát, hogy adja meg a fiókodnak a
> **BigQuery Job User** szerepkört a `forestcrm-report-data` projekten.

**D: no read access to the dataset**

> A BigQuery connector működik, de a Google-fiókod nem fér hozzá a Forest adattárház
> `public` adatkészletéhez.
>
> Kérd meg a Forest adattárház adminisztrátorát, hogy adjon olvasási jogot
> (**BigQuery Data Viewer**) a fiókodnak a `forestcrm-report-data.public` adatkészletre.
> Ellenőrizd azt is, hogy a connectorban a megfelelő Google-fiókkal vagy-e bejelentkezve.

**E: unknown error**

> Nem sikerült elérni a Forest adattárházat. A BigQuery hibaüzenete:
>
> `<the exact error text>`
>
> Ha a hiba nem egyértelmű, küldd el ezt az üzenetet a Forest adattárház adminisztrátorának.

## Common mistakes

- Running the probe with a different `projectId`. The project you pass is the one billed and
  checked for `bigquery.jobs.create`. Always use `forestcrm-report-data`.
- Guessing an answer or using a different data source after the check fails. Stop after the message.
- Re-running the probe before every query. Once per conversation is enough.
