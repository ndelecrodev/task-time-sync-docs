# Task Time Sync

This site documents the whole system: a pipeline that syncs Jira and
Clockify, and a dashboard the team uses to track the result.

## The two parts

**Pipeline** ([repository](https://github.com/SEU-USUARIO/task-time-sync))

Runs once a day, scheduled via GitHub Actions. Fetches tasks from Jira and
time entries from Clockify, writes both to an Excel file and to a Postgres
database (Supabase), and posts to Teams when a task is close to its
deadline or already late. It also keeps employee identity (name, Jira
email, Clockify email) synced from an editable sheet in the Excel file
itself.

**Dashboard** ([repository](https://github.com/SEU-USUARIO/quimia-dashboard))

A static site that reads straight from the same Postgres database the
pipeline writes to. Each team member logs in with their own email and sees
tasks, hours, and productivity indicators, both their own and the whole
team's.

## How the two parts connect

```
Jira ─┐
      ├─> Pipeline (Python) ─┬─> Excel (spreadsheet, on Backblaze B2)
Clockify ─┘                  └─> Postgres (Supabase) ─> Dashboard (site)
```

The pipeline is the only thing that **writes** to Postgres. The dashboard
only **reads**, it never inserts, updates, or deletes anything, and it's
the database's own Row Level Security that guarantees this, not a rule in
the site's code.

## Where to start

- Want to understand how the pipeline works internally: [Pipeline
  architecture](pipeline/architecture.md).
- Want to know the data format in Excel and Postgres: [Data
  model](pipeline/data-model.md).
- Want to understand why a specific decision was made (and not another):
  [Design decisions](pipeline/design-decisions.md).
- Want to know how the dashboard works, authentication included: [Dashboard
  overview](dashboard/overview.md).
