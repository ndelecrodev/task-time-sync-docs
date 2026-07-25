# Data model

## Python models

Defined in `src/sop_pipeline/models/schemas.py`. These are Pydantic
models: a Jira issue that doesn't satisfy the contract is dropped with a
`warning`, instead of crashing the whole run.

### Employee identity

Since Jira identifies people by display name and Clockify by email, a
mapping layer normalizes both to a canonical name used as the join key.

**Editable source:** the `DIM_FUNCIONARIO` sheet in the spreadsheet is
where someone corrects or adds employees manually. Before each run,
`EmployeeSyncService` reads that sheet with `ExcelReader` and writes the
rows to the `funcionarios` table in Postgres via
`PostgresClient.upsert_employee`, matching on `jira_email` or
`clockify_email`.

**Duplicates:** the first row to use a given `jira_email` or
`clockify_email` syncs normally; any later row that repeats either one is
treated as a duplicate (`EmployeeSyncService._split_duplicates`), gets a
reason, and is written to the `DUPLICADOS_REMOVIDOS` sheet instead of
being synced.

**Runtime use:** `Settings.load_employee_registry` reads the
already-synced `funcionarios` table and builds an `EmployeeRegistry`, used
by `EtlService` to normalize `Task.assignee` (from Jira) and
`TimeEntry.employee` (from Clockify) to the canonical name.

**Employee photo:** `funcionarios.photo_url` stores the public Supabase
Storage URL for the employee's photo, or `None` when no photo has been
uploaded yet. `EmployeeSyncService` propagates the value read from
`DIM_FUNCIONARIO` on every sync; this is the field the indicators
dashboard (a separate repository) consumes to show each person's photo.

**Unmapped employees:** if an employee isn't found in the registry, they
get a visible sentinel value (`"Unmapped employee: <email>"`) instead of
being silently dropped. This follows design decision #8: a bad record
doesn't crash the whole run, and data quality problems stay visible in the
report instead of hidden.

### `Task`

A normalized Jira issue. Upsert key: `task_id` (the Jira *key*).

| Field | Type | Jira source |
|---|---|---|
| `task_id` | `str` | `key` |
| `title` | `str` | `fields.summary` (default `"No title"`) |
| `assignee` | `str` | `fields.assignee.displayName` |
| `priority` | `Priority` | `fields.priority.name` |
| `status` | `str` | `fields.status.name` |
| `area` | `str \| None` | custom field configured in `JIRA_CUSTOMFIELD_AREA` |
| `creation_date` | `date` | `fields.created` |
| `due_date` | `date \| None` | `fields.duedate` |
| `completion_date` | `date \| None` | `fields.resolutiondate` |
| `task_type` | `TaskType` | `fields.issuetype.name` |
| `creator` | `str \| None` | `fields.creator.displayName` |
| `update_date` | `date \| None` | `fields.updated` |
| `assignee_email` | `EmailStr \| None` | `fields.assignee.emailAddress` |
| `tags` | `list[str]` | `fields.labels` |

Computed fields (`@computed_field`), used by the alert rule and the
notification text, but **not** written to the spreadsheet, which has its
own formulas:

| Field | Returns |
|---|---|
| `days_remaining` | Days until the deadline; negative if already overdue; `None` when there's no deadline. |
| `is_late` | `"SIM"` / `"NÃO"` / `None`. |
| `deadline_status` | A `DeadlineStatus` value. |

`Notifier._build_message` never shows a negative `days_remaining`
directly: for an overdue task, the notification line becomes "Tarefa
atrasada há N dia(s)" (with N positive); for a task within the deadline,
"Dias restantes: N dia(s)"; with no deadline set, "Dias restantes:
Indefinido".

When Jira doesn't provide an assignee or area, the ETL uses the strings
`"There is no one responsible."` and `"There is no one area."` instead of
dropping the row, the task still shows up in the report.

### `TimeEntry`

A Clockify time entry. Upsert key: `entry_id`.

| Field | Type | Clockify source |
|---|---|---|
| `entry_id` | `str` | `id` |
| `employee` | `str` | email resolved from `userId` |
| `entry_date` | `date` | `timeInterval.start`, converted from UTC to America/Sao_Paulo |
| `hours` | `float` (≥ 0) | `timeInterval.duration` (ISO 8601) converted to hours |

Entries with a null `duration` (timer still running) are skipped and
picked up in a later run.

### `TaskDetail`

The long description of a task, kept separate because it's a large text
field and lives in its own sheet. Upsert key: `task_id`.

| Field | Type | Source |
|---|---|---|
| `task_id` | `str` | `key` |
| `description` | `str \| None` | `fields.description`, flattened from ADF format to plain text |

### Enums

| Enum | Values |
|---|---|
| `Priority` | `Highest`, `High`, `Medium`, `Low`, `Lowest` |
| `TaskType` | `Bug`, `Task`, `Story`, `Epic`, `Subtask` |
| `DeadlineStatus` | `Concluído`, `Atrasado`, `Atenção`, `No prazo`, `Sem prazo` |

The `Priority` and `TaskType` values are exactly the strings the Jira API
returns. The `DeadlineStatus` values are exactly the strings the
`status_prazo` column formula produces in Excel. **None of these values
can be translated**, only the enum member names.

---

## Mapping to the spreadsheet

Conventions in the `.xlsx` file: sheet name in UPPERCASE, table name in
lowercase, first column is always the ID used for the upsert.

### Sheets written by Python

| Sheet | Table | Columns | Written by |
|---|---|---|---|
| `BASE_TAREFAS` | `base_tarefas` | id, titulo, responsavel, area, prioridade, status, data_criacao, prazo, data_conclusao, **dias_restantes**, **atrasado**, **status_prazo**, tipo, criador, data_atualizacao | `save_tasks` |
| `DETALHES_TAREFA` | `detalhes_tarefa` | id, descricao | `save_details` |
| `BASE_HORAS` | `base_horas` | id, funcionario, data, horas | `save_hours` |
| `DIM_ETIQUETAS` | `dim_etiquetas` | id_etiqueta, nome_etiqueta | `save_tags` |
| `FATO_TAREFA_ETIQUETA` | `fato_tarefa_etiqueta` | id_tarefa, id_etiqueta | `save_tags` |

The three columns in **bold** are calculated by Excel formula; Python
only copies the formula text when it creates a new row.

`Task` → `BASE_TAREFAS` is defined by the `TASK_COLUMN_MAP` dict in
`integrations/excel_writer.py`. The left side is the literal column
header in the spreadsheet (in Portuguese, which is why it isn't
translated); the right side is the attribute name on the model:

```python
TASK_COLUMN_MAP = {
    "id": "task_id",
    "titulo": "title",
    "responsavel": "assignee",
    ...
}
```

`DETALHES_TAREFA` and `BASE_HORAS` have a short, fixed layout, so they're
written by column position, without going through the header map.

### Sheets Python doesn't write

They exist in the file and are maintained manually or by formula.
`DIM_FUNCIONARIO` is the partial exception: nothing writes to it from
code, but `ExcelReader` reads it on every run to sync the registry with
Postgres (see "Employee identity" above).

| Sheet | Table | Role |
|---|---|---|
| `DIM_FUNCIONARIO` | `dim_funcionario` | Employee registry (id_funcionario, nome, email), read by `ExcelReader`. |
| `DIM_FUNCIONARIO_AREA` | `dim_func_area` | Area dimension (id_area, nome_area). |
| `FATO_FUNCIONARIO_AREA` | `fato_funcionario` | N:N relationship between employee and area. |
| `CALCULOS` | `Tabela6` | Per-person metrics (tasks, completed, overdue, hours, productivity). |
| `INDICADORES` | `Tabela7`, `Tabela10`, `Tabela11`, `Tabela12` | Consolidated dashboard KPIs. |

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

Defined in `src/sop_pipeline/clients/postgres_client.py` as SQLAlchemy
models (ORM), not raw SQL. The ORM was also chosen for the project's
learning goal; see the corresponding design decision. Table and column
names mirror the schema already deployed on Supabase, which is why they
stay in Portuguese; the upsert methods and `PostgresClient` itself are in
English. This schema runs in parallel with the spreadsheet, not in its
place: every pipeline write to `tarefas`, `horas`, `etiquetas`, etc. has
an equivalent write to the corresponding sheet in the `.xlsx`.

| Table | Role | Upsert via |
|---|---|---|
| `funcionarios` | Employee identity, synced from `DIM_FUNCIONARIO`. Includes `photo_url`, the photo URL used by the dashboard. | `upsert_employee` |
| `tarefas` | One row per Jira issue; `responsavel_id` is `NULL` when the employee wasn't mapped. | `upsert_task` |
| `detalhes_tarefa` | Long description of a task. | `upsert_task_detail` |
| `horas` | A Clockify time entry; `funcionario_id` is `NULL` when the employee wasn't mapped. | `upsert_time_entry` |
| `etiquetas` | Distinct tags assigned to tasks. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | N:N association between `tarefas` and `etiquetas`. | `upsert_tag_and_link` |
