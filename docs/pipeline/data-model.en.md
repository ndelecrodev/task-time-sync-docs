# Data model

## Python models

Defined in `src/sop_pipeline/models/schemas.py`. They are Pydantic models: a
ClickUp task that doesn't satisfy the contract is discarded with a `warning`,
instead of bringing down the entire execution.

### Employee identity

Since ClickUp identifies people by the assignee's `username`/email and
Clockify by email, a mapping layer normalizes both to a canonical name used
as the join key.

**Editable source:** the workbook's `DIM_FUNCIONARIO` tab is where someone
corrects or adds employees by hand. Before each run, `EmployeeDataSyncService`
reads that tab with `ExcelReader` and writes the rows into Postgres'
`funcionarios` table via `PostgresClient.upsert_employee`, matched by either
`clickup_email` or `clockify_email` — the column, previously named
`jira_email` as inherited from the schema deployed in the Jira era, was
renamed to `clickup_email` (see [`design-decisions.md`](design-decisions.md#22-the-task-source-migrated-from-jira-to-clickup-only-in-the-extraction-layer-task-stayed-the-contract))
to stop carrying a name that no longer made sense after the migration; it
holds the employee's registered email, matched against the ClickUp assignee.

**Duplicates:** the first row to use a given `clickup_email` or `clockify_email`
is synced normally; any later row that repeats either one is treated as a
duplicate (`EmployeeDataSyncService._split_duplicates`), gets a reason attached,
and is written to the `DUPLICADOS_REMOVIDOS` tab instead of being synced.

**Runtime use:** `Settings.load_employee_registry` reads the already-synced
`funcionarios` table and builds an `EmployeeRegistry`, used by `EtlService` to
normalize `Task.assignee` (from ClickUp) and `TimeEntry.employee` (from
Clockify) to the canonical name.

**Employee photo:** `funcionarios.photo_url` holds the public Supabase Storage
URL of the employee's photo, or `None` when no photo has been uploaded yet.
`EmployeeDataSyncService` forwards the value read from `DIM_FUNCIONARIO` on every
sync; this is the field the reporting dashboard (a separate repository)
consumes to show each person's photo.

**Unmapped employees:** if an employee is not found in the registry, they
receive a visible sentinel value (`"Unmapped employee: <email>"`) instead of
being dropped silently. This follows design decision #8: a single bad record
does not take down the whole run, and data quality issues are visible in the
report rather than hidden.

### `Task`

A normalized ClickUp task. Upsert key: `task_id` (ClickUp's `id`).

| Field | Type | Origin in ClickUp |
|---|---|---|
| `task_id` | `str` | `id` |
| `title` | `str` | `name` (default `"No title"`) |
| `assignee` | `str` | `assignees[*].username`/`.email`, each normalized individually and joined with `", "` |
| `priority` | `Priority` | `priority.priority` (`urgent`/`high`/`normal`/`low`), mapped onto the enum |
| `status` | `str` | `status.status` |
| `area` | `str \| None` | resolved option of the `drop_down` custom field configured in `CLICKUP_AREA_FIELD_ID` |
| `creation_date` | `date` | `date_created` (millisecond timestamp) |
| `due_date` | `date \| None` | `due_date` (millisecond timestamp) |
| `completion_date` | `date \| None` | `date_closed` (millisecond timestamp) |
| `task_type` | `TaskType` | fixed to `TaskType.TASK` — see [`design-decisions.md`](design-decisions.md#22-the-task-source-migrated-from-jira-to-clickup-only-in-the-extraction-layer-task-stayed-the-contract) |
| `creator` | `str \| None` | `creator.username` |
| `update_date` | `date \| None` | `date_updated` (millisecond timestamp) |
| `assignee_email` | `EmailStr \| None` | `assignees[0].email`, falling back to `EmployeeRegistry.get_registered_email` |
| `tags` | `list[str]` | `tags[*].name` |
| `turma` | `str` | `folder.name` — read straight from ClickUp, never user-entered; see [`design-decisions.md`](design-decisions.md#23-synced-clickup-folders-are-an-explicit-allowlist-not-every-folder-in-the-space) |

**Multiple assignees:** unlike Jira, ClickUp allows more than one assignee per
task. Each is normalized individually by `normalize_employee_identifier` (by
email when present, otherwise by `username`), and the resulting canonical
names are joined into `assignee`. Only the **first** assignee's email feeds
`assignee_email`, because a Teams @mention can only target one person — see
[`design-decisions.md`](design-decisions.md#22-the-task-source-migrated-from-jira-to-clickup-only-in-the-extraction-layer-task-stayed-the-contract).

**Area (`drop_down` custom field):** ClickUp returns `custom_fields` as a list
of field objects, each with `id`, `name`, `type`, and optionally `value`. For
the `drop_down` field configured in `CLICKUP_AREA_FIELD_ID`, `value` is an
**index** into that field's own `type_config.options` array, not the option's
text — the index is resolved against that array to get the area name. When
the field is absent, has no `value` key, or the index doesn't resolve, the
result is `NO_AREA`.

**Milliseconds:** `date_created`, `due_date`, `date_closed`, and
`date_updated` arrive as millisecond Unix-timestamp strings (e.g.
`"1753401600000"`), not ISO 8601 like Jira's.
`EtlService._parse_millis_to_date` converts each to a `date` in
America/Sao_Paulo, handling `None`.

Computed fields (`@computed_field`), used by the alert rule and notification text — are **not** written to the spreadsheet, which has its own formulas:

| Field | Return |
|---|---|
| `days_remaining` | Days until deadline; negative if overdue; `None` if no deadline. |
| `is_late` | `"SIM"` / `"NÃO"` / `None`. |
| `deadline_status` | A value of `DeadlineStatus`. |

`Notifier._build_message` never shows a negative `days_remaining` directly:
for an overdue task, the notification line reads "Tarefa atrasada há N
dia(s)" (with a positive N); for a task still within its deadline, "Dias
restantes: N dia(s)"; with no deadline set, "Dias restantes: Indefinido".

When ClickUp provides no assignee or area, the ETL uses the texts
`"There is no one responsible."` and `"There is no one area."` instead of discarding the
row — the task still appears in the report.

### `TimeEntry`

A Clockify time entry. Upsert key: `entry_id`.

| Field | Type | Origin in Clockify |
|---|---|---|
| `entry_id` | `str` | `id` |
| `employee` | `str` | email resolved from `userId` |
| `entry_date` | `date` | `timeInterval.start`, converted from UTC to America/Sao_Paulo |
| `hours` | `float` (≥ 0) | `timeInterval.duration` (ISO 8601) converted to hours |

Entries with null `duration` (timer still running) are ignored and collected in a
later execution.

### `TaskDetail`

The long description of a task, separated because it's a large text and goes in its own tab. Upsert key: `task_id`.

| Field | Type | Origin |
|---|---|---|
| `task_id` | `str` | `id` |
| `description` | `str \| None` | `description`, falling back to `text_content` when absent |

### Enums

| Enum | Values |
|---|---|
| `Priority` | `Highest`, `High`, `Medium`, `Low`, `Lowest` |
| `TaskType` | `Bug`, `Task`, `Story`, `Epic`, `Subtask` |
| `DeadlineStatus` | `Concluído`, `Atrasado`, `Atenção`, `No prazo`, `Sem prazo` |

`Priority`'s values are Jira's historical labels; the ETL maps ClickUp's four priority levels (`urgent`/`high`/`normal`/`low`) onto them — see [`design-decisions.md`](design-decisions.md#22-the-task-source-migrated-from-jira-to-clickup-only-in-the-extraction-layer-task-stayed-the-contract). `TaskType` is fixed to `Task` for every ClickUp-sourced task, for the same reason. Those of `DeadlineStatus` are exactly the strings the `status_prazo` column formula produces in Excel. **None of these values can be translated** — only the enum member names.

---

## Mapping to the spreadsheet

Conventions for the `.xlsx` file: tab name in UPPERCASE, table name in lowercase, first column is always the ID used in upsert.

### Tabs written by Python

| Tab | Table | Columns | Written by |
|---|---|---|---|
| `BASE_TAREFAS` | `base_tarefas` | id, titulo, responsavel, area, prioridade, status, data_criacao, prazo, data_conclusao, **dias_restantes**, **atrasado**, **status_prazo**, tipo, criador, data_atualizacao, arquivada_em, turma | `save_tasks` |
| `DETALHES_TAREFA` | `detalhes_tarefa` | id, descricao | `save_details` |
| `BASE_HORAS` | `base_horas` | id, funcionario, data, horas | `save_hours` |
| `DIM_ETIQUETAS` | `dim_etiquetas` | id_etiqueta, nome_etiqueta | `save_tags` |
| `FATO_TAREFA_ETIQUETA` | `fato_tarefa_etiqueta` | id_tarefa, id_etiqueta | `save_tags` |

The three columns in **bold** are calculated by Excel formula; Python
only copies the formula text when creating a new row. `arquivada_em` breaks
the pattern too: it's written by `ExcelWriter.mark_archived_tasks`, not
`save_tasks`, and it's the only column still touched on a row once the task
has dropped out of ClickUp — every other field on an archived row stays frozen
at its last known value, by design (see design decision #19).

`Task` → `BASE_TAREFAS` is defined by the dict `TASK_COLUMN_MAP` in
`integrations/excel_writer.py`. The left side is the literal column header in the
spreadsheet (in Portuguese, so it's not translated); the right side is the attribute name in the model:

```python
TASK_COLUMN_MAP = {
    "id": "task_id",
    "titulo": "title",
    "responsavel": "assignee",
    ...
}
```

`DETALHES_TAREFA` and `BASE_HORAS` have short fixed layouts, so they are written by
column position, without going through the header map.

### Tabs that Python doesn't write

They exist in the file and are maintained manually or by formula.
`DIM_FUNCIONARIO` is the partial exception: nothing writes to it in code, but
`ExcelReader` reads it on every run to sync the registry into Postgres (see
"Employee identity" above).

| Tab | Table | Role |
|---|---|---|
| `DIM_FUNCIONARIO` | `dim_funcionario` | Employee registry (id_funcionario, nome, email), read by `ExcelReader`. |
| `DIM_FUNCIONARIO_AREA` | `dim_func_area` | Area dimension (id_area, nome_area). |
| `FATO_FUNCIONARIO_AREA` | `fato_funcionario` | N:N relationship between employee and area. |
| `CALCULOS` | `Tabela6` | Metrics per person (tasks, completed, overdue, hours, productivity). |
| `INDICADORES` | `Tabela7`, `Tabela10`, `Tabela11`, `Tabela12` | Consolidated KPIs from the dashboard. |

### Relationships

```
BASE_TAREFAS (id)
   │ 1:1
   ├──────────▶ DETALHES_TAREFA (id)
   │
   │ 1:N
   └──────────▶ FATO_TAREFA_ETIQUETA (id_tarefa) ──N:1──▶ DIM_ETIQUETAS (id_etiqueta)

DIM_FUNCIONARIO (id_funcionario)
   │ 1:N
   └──────────▶ FATO_FUNCIONARIO_AREA (id_funcionario) ──N:1──▶ DIM_FUNCIONARIO_AREA (id_area)

BASE_HORAS (funcionario) ──── links to DIM_FUNCIONARIO by email
```

## Postgres schema

Defined in `src/sop_pipeline/clients/postgres_client.py` as SQLAlchemy models
(ORM), not raw SQL. The ORM was also chosen for the project's learning goal;
see the corresponding design decision. Table and column names mirror the
schema already deployed in Supabase and stay in Portuguese for that reason;
the upsert methods and `PostgresClient` itself are in English. This schema
runs in parallel with the spreadsheet, not instead of it: every pipeline
write to `tarefas`, `horas`, `etiquetas`, and so on has a matching write to
the corresponding `.xlsx` tab.

| Table | Role | Upserted by |
|---|---|---|
| `funcionarios` | Employee identity, synced from `DIM_FUNCIONARIO`. Includes `photo_url`, the photo URL consumed by the dashboard. | `upsert_employee` |
| `tarefas` | One row per ClickUp task; `responsavel_id` is `NULL` when the employee could not be mapped. `arquivada_em` holds the timestamp when the task stopped appearing in the fetch (`CLICKUP_SPACE_ID` + `CLICKUP_FOLDER_IDS`, `NULL` while active); the row is never deleted. `turma` holds the ClickUp folder's name (see [`design-decisions.md`](design-decisions.md#23-synced-clickup-folders-are-an-explicit-allowlist-not-every-folder-in-the-space)), read straight from the API, never user-entered. | `upsert_task` (archiving: `archive_missing_tasks`) |
| `detalhes_tarefa` | Long-form task description. | `upsert_task_detail` |
| `horas` | One Clockify time entry; `funcionario_id` is `NULL` when the employee could not be mapped. | `upsert_time_entry` |
| `etiquetas` | Distinct tags assigned to tasks. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | N:N association between `tarefas` and `etiquetas`. | `upsert_tag_and_link` |
| `areas` | Employee work areas, synced from `DIM_FUNCIONARIO_AREA`. | `upsert_area_and_link` |
| `funcionario_area` | N:N association between `funcionarios` and `areas`, synced from `FATO_FUNCIONARIO_AREA`; composite key `funcionario_id` + `area_id`, both FKs. | `upsert_area_and_link` |
