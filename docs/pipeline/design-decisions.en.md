# Design decisions

A record of the project's non-obvious choices and the reasoning behind
them.

## 1. The upsert works by ID, scanning the first column

`ExcelWriter._find_row` scans column 1 of the table looking for the ID.
If it finds it, it overwrites the row; if not, it appends a new one.

**Why:** it makes the run **idempotent**. The pipeline runs on a
schedule, and `JIRA_JQL` normally brings back issues that are already in
the spreadsheet. Without the upsert, every run would duplicate rows. With
it, running twice in a row produces exactly the same file.

**Cost:** the search is linear (O(n) per record, O(n²) per run). At the
current scale, a few hundred rows, it's irrelevant, and it keeps the code
free of auxiliary state. If the spreadsheet grows to thousands of rows,
the way forward is building an `{id: row}` index once per sheet, before
the loop.

## 2. Calculated columns live in Excel formulas, not in Python

`dias_restantes`, `atrasado`, and `status_prazo` aren't in
`TASK_COLUMN_MAP`. Python never writes values to them, it only copies the
formula text when it creates a new row.

**Why:** the spreadsheet is the final product and gets opened by people
on days the pipeline didn't run. If these fields were written as static
values, "days remaining" would freeze at the date of the last run and
start lying. As a formula, Excel recalculates on its own every time
someone opens the file.

The `Task` model **also** exposes `days_remaining` / `is_late` /
`deadline_status`, but only for the alert rule and the notification text,
which need the value at the moment of the run.

> Known divergence: the Excel formula uses `MAX(prazo-TODAY(),0)`, so it
> never shows a negative number; `Task.days_remaining` in Python returns
> negative for overdue tasks. Since Python doesn't write to that column,
> the two definitions never overlap, but it's worth knowing when
> comparing the two sides.

## 3. Copying the formula text works because of structured references

`_copy_formula` copies the formula string from row 2 to the new row,
without rewriting any index.

**Why:** the spreadsheet's formulas use structured table references:

```
=IF(base_tarefas[[#This Row],[data_conclusao]]<>"","Concluído", ...)
```

`[#This Row]` resolves relative to the row the formula is in. The same
text is therefore correct on any row, there's no offset to fix, unlike
what would happen with references in the style of `J2`, `J3`.

## 4. Tags use an N:N relationship

A tag became two sheets: `DIM_ETIQUETAS` (one row per distinct tag, with
a surrogate ID) and `FATO_TAREFA_ETIQUETA` (one row per association).

**Why:** the relationship is genuinely many-to-many, a task can have
several tags and a tag is used by several tasks. The alternatives were
worse:

- storing tags concatenated in a single cell (`"bug;urgente"`) would make
  it impossible to count tasks per tag or build a pivot table;
- creating a column per tag would require changing the spreadsheet schema
  every time the team invented a new tag.

With dimensional modeling, a pivot table per tag comes for free, and
`_get_or_create_tag_id` creates the dimension on demand.

The same reasoning applies to `FATO_FUNCIONARIO_AREA`: a person can work
in more than one area.

## 5. `fullCalcOnLoad` is turned on every time the workbook opens

`_open_workbook` always sets `workbook.calculation.fullCalcOnLoad = True`.

**Why:** openpyxl reads the *cached value* of formulas and writes it back
on save. Without this flag, a newly inserted row would show an empty or
stale cache until someone edited the cell. With the flag, Excel
recalculates the whole file the next time it's opened.

## 6. The spreadsheet's identifiers stay in Portuguese

Sheet names (`BASE_TAREFAS`), table names (`base_tarefas`), column
headers (`data_criacao`), and `DeadlineStatus` values (`"Concluído"`)
stay in Portuguese, even with the code entirely in English.

**Why:** these aren't code names, they're **data**. They're looked up at
runtime inside the `.xlsx` file (`workbook["BASE_TAREFAS"]`,
`worksheet.tables["base_tarefas"]`, `column_map["data_criacao"]`) and
compared against what the Excel formulas produce. Translating them would
break the pipeline at runtime, with no import error to warn you.

That's why `TASK_COLUMN_MAP` exists: it's the explicit boundary between
the data world (left side, Portuguese) and the code world (right side,
English).

## 7. The heartbeat stays outside error handling

See [`architecture.md`](architecture.md#observability): the heartbeat
should only fire once the spreadsheet has actually reached the bucket.
It's the last line of `run()` and isn't wrapped in `try/except`,
precisely so an earlier failure prevents it from running.

## 8. Invalid records are dropped, they don't stop the run

`transform_tasks`, `transform_details`, and `transform_time_entries`
catch `ValidationError` and `KeyError` **per record**, log a `warning`,
and move on.

**Why:** a single issue with a missing field can't cost the entire day's
sync. The warning stays in Better Stack for later investigation.

**Trade-off:** a systemic error (for example, `JIRA_CUSTOMFIELD_AREA`
pointing to a field that no longer exists) would show up as "every task
got skipped," with the run ending in apparent success. That's why each
drop is logged at **ERROR** and sent to Sentry, and the end-of-sync log
carries the count: `"Jira: N issues fetched, M tasks written, K
discarded"`. A `K` above zero is a sign of a structural problem,
typically a new priority or type in Jira that doesn't exist in the
`models/schemas.py` enums.

**Careful when touching this `except`:** it explicitly lists
`AttributeError` and `TypeError` (the `RECORD_ERRORS` tuple) in addition
to `ValidationError`/`KeyError`. Without them, an unexpected `null` field
from Jira throws `AttributeError`, escapes the loop, and takes down the
whole batch, the opposite of the per-record isolation this decision
promises. That was exactly the bug: `priority: null` on a single issue
caused every other one to get lost.

## 9. Custom exceptions cover only the I/O edges

`errors/exceptions.py` defines `ExcelWriteError`, `NotificationError`,
and `StorageError`, raised in `integrations/`. `TaskValidationError` is
defined, but the ETL **still** uses `except (ValidationError, KeyError):
continue`.

**Why:** swapping the `continue` for a `raise` would change behavior, it
would start aborting the transformation on the first bad record, which is
exactly the opposite of decision 8. The custom exceptions only apply
where there was already generic error propagation up to the
`try/except` in `run()`, always with `raise ... from error` to preserve
the original cause.

## 10. Packages don't re-export anything in `__init__.py`

Each `__init__.py` has only a docstring; modules are imported by their
full path.

**Why:** `config/settings.py` instantiates `Settings()` on import, which
reads and validates the `.env`. With re-exports in
`integrations/__init__.py`, importing `ExcelWriter` would pull in
`Notifier`, which would pull in the settings, and that would suddenly
require a full `.env` just to write to a local spreadsheet. Without
re-exports, `ExcelWriter` is testable in isolation.

## 11. Status validation in Excel is a manual copy of the Jira workflow

The validation dropdown on `BASE_TAREFAS.status` uses the list `Backlog,
To Do, In Progress, Code Review, Testing, Done`, copied from the workflow
configured in Jira on 2026-07-22.

**Why:** unlike `tipo` (`TaskType`, an enum validated in
`models/schemas.py`), `status` is free text coming straight from Jira,
there's no enum on the Python side for that column. A fixed list in the
code would run the same risk that already materialized with `tipo`
(issue `QT-6` with `task_type="Function"`, outside the enum, got
silently dropped from the report). That's why the status list doesn't
live in the code: it lives only in Excel's dropdown validation, and has
to be copied manually from Jira.

**Trade-off:** this validation only protects manual edits to the
spreadsheet. The pipeline overwrites `status` on every run with the value
coming straight from Jira, without going through Excel's validation, so a
new status in Jira shows up in the report even if it's not in the
dropdown list, but manually editing the cell with a value outside the
list is blocked.

**Careful when touching this:** if the Jira workflow changes (a status
renamed, added, or removed), this list needs to be updated manually in
Excel. There's no automatic sync between the two.

## 12. Employee identity moved from EMPLOYEES_JSON to a Postgres table, with Excel as an editable front-end

Employee mapping used to live in an environment variable
(`EMPLOYEES_JSON`), a JSON blob read when `Settings` initialized. Today
`Settings.load_employee_registry` reads the `funcionarios` table in
Postgres, and `EmployeeSyncService` is what keeps that table up to date,
syncing it from the `DIM_FUNCIONARIO` sheet in the spreadsheet before
each run.

**Why:** a JSON blob in an environment variable could only be edited by
someone with access to the `.env` who knew the correct syntax; a typo
broke reading the entire employee registry. Moving the source of truth to
a relational table, editable through a spreadsheet the team already uses
day to day (`DIM_FUNCIONARIO`), removes the technical barrier to keeping
the registry up to date. Postgres also makes that registry available to
other consumers, like the dashboard.

**Trade-off:** pipeline startup now depends on the database being
reachable. `load_employee_registry` propagates `SQLAlchemyError` when the
connection fails, something the local JSON blob never had to consider.
Syncing employees also became one more step at the start of every run,
and it has to finish before any name normalization.

## 13. SQLAlchemy was chosen as the ORM, not Core or raw psycopg

`PostgresClient` uses SQLAlchemy declarative classes (`Funcionarios`,
`Tarefas`, etc.) and `Session` objects, instead of writing SQL directly
with `psycopg` or using SQLAlchemy's own `Core` layer.

**Why:** part of the motivation is the project's learning goal. This
project also doubles as a space to practice ORM on a real case, with
upserts, relationships, and constraints. As a side effect, the ORM also
keeps SQL out of the rest of the code: upserts in `PostgresClient` become
Python attribute assignment (`existing.canonical_name = ...`) instead of
a hand-built `UPDATE ... SET`.

**Trade-off:** each upsert opens its own `Session` and does a `SELECT`
before the `INSERT`/`UPDATE` (see `upsert_employee`, `upsert_task`,
etc.), which is less efficient than a native Postgres `INSERT ... ON
CONFLICT`. For the pipeline's current volume (a few hundred rows per run)
this isn't a problem; worth revisiting if the volume grows.

## 14. `person.area` on the dashboard is derived from task areas, not from a per-employee area table in Postgres

Excel has `FATO_FUNCIONARIO_AREA`, a dedicated N:N relationship between
employee and area (see decision 4). The Postgres schema has no
equivalent: there's no `funcionario_area` table. When the dashboard needs
a person's area, it derives that value from the areas of the tasks
assigned to them in `tarefas.area`.

**Why:** `FATO_FUNCIONARIO_AREA` originated in Excel for the
spreadsheet-based dashboard; replicating that table when migrating the
indicators to Postgres would mean maintaining yet another synced
registry, without there being a need today that task area doesn't
already cover. In practice, whoever works mostly on tasks in one area
also belongs to it.

**Trade-off:** an employee with no tasks assigned in a given period has
no derivable area in Postgres, unlike Excel, where the person's area is
registered explicitly. If this gap starts to matter, the way forward is
adding a `funcionario_area` table in Postgres mirroring
`FATO_FUNCIONARIO_AREA`.

## 15. The alert shows "Tarefa atrasada há N dia(s)" instead of the negative number from `days_remaining`

`Task.days_remaining` is negative by design for overdue tasks (decision
2). `Notifier._build_message`, though, never exposes that negative
number in the text: when `days_remaining < 0`, it swaps the label to
"Tarefa atrasada há" and shows `abs(days_remaining)`.

**Why:** "-3 dias restantes" requires the reader to do the mental math
(negative means late, and by how many days). "Tarefa atrasada há 3
dia(s)" communicates the same information without that extra step, for
text that gets read quickly inside a Teams notification.

**Careful when touching this:** the sign flip (`abs()`) and the label
swap need to move together. Changing one without the other produces a
message like "Dias restantes: -3 dia(s)" or "Tarefa atrasada há -3
dia(s)", both inconsistent with the rest of the text.
