# Architecture

## Execution flow

An execution (`sop_pipeline.pipeline.run`) is a linear sequence with three isolated synchronization steps.

```
 1. StorageClient.download_file
    B2 ──────────────────────────────▶ planilha_temp.xlsx (local disk)

 2. EmployeeDataSyncService.sync + .sync_areas
    ExcelReader.read_employees        ── DIM_FUNCIONARIO tab
        │  list[dict]
        ▼
    PostgresClient.upsert_employee  ─▶ funcionarios table (Postgres/Supabase)
    ExcelWriter.save_duplicates     ─▶ DUPLICADOS_REMOVIDOS tab (repeated-email rows)

    name_to_id = {canonical_name: id}, built from funcionarios (Postgres)

    ExcelReader.read_dim_employee_area   ── DIM_FUNCIONARIO_AREA tab
    ExcelReader.read_fato_employee_area  ── FATO_FUNCIONARIO_AREA tab
        │  list[dict]
        ▼
    PostgresClient.upsert_area_and_link ─▶ areas, funcionario_area tables (Postgres/Supabase)

 3. sync_clickup
    ClickUpClient.fetch_tasks(CLICKUP_TEAM_ID, CLICKUP_SPACE_ID) ── pagination by
                               page number, GET /team/{team_id}/task
        │  raw list[dict] (every task in the Space, from any folder)
        ▼
    pipeline._filter_allowed_folders
        │  drops tasks whose folder.id isn't in CLICKUP_FOLDER_IDS
        ▼
    EtlService.transform_tasks    ─▶ list[Task]
    EtlService.transform_details  ─▶ list[TaskDetail]
        │
        ▼
    ExcelWriter.save_tasks    ─▶ tab BASE_TAREFAS
    ExcelWriter.save_tags     ─▶ tabs DIM_ETIQUETAS + FATO_TAREFA_ETIQUETA
    ExcelWriter.save_details  ─▶ tab DETALHES_TAREFA
    PostgresClient.upsert_task / upsert_task_detail / upsert_tag_and_link
                               ─▶ tarefas, detalhes_tarefa, etiquetas, tarefa_etiqueta tables
    PostgresClient.archive_missing_tasks
                               ─▶ marks tarefas.arquivada_em on tasks missing from the fetch (never deletes)

 4. sync_clockify
    ClockifyClient.list_users
        │  for each user:
        ▼
    ClockifyClient.fetch_time_entries(user_id)  ── pagination by header Last-Page
        │  raw list[dict]
        ▼
    EtlService.transform_time_entries ─▶ list[TimeEntry]
        │
        ▼
    ExcelWriter.save_hours       ─▶ tab BASE_HORAS
    PostgresClient.upsert_time_entry ─▶ horas table

 5. StorageClient.upload_file
    planilha_temp.xlsx ──────────────▶ B2   (overwrites the object)

 6. process_alerts (uses Tasks from step 3)
    AlertService.tasks_to_alert  ─▶ list[Task] within alert window
        │
        ▼
    Notifier.send_alert ─▶ POST to Teams webhook for the task's area

 7. Heartbeat
    GET BETTERSTACK_HEARTBEAT_URL   ── only reached if upload succeeded
```

Postgres runs in parallel with the spreadsheet, not instead of it: steps 3
and 4 write the same information to both destinations, one upsert per row
in each.

`ExcelReader.read_employees`, `read_dim_employee_area`, and `read_fato_employee_area`
are thin wrappers over a single generic method, `read_sheet_as_dicts(file_path,
sheet_name, table_name)`, which holds the logic for opening the workbook, finding
the table, and building the list of row dicts. Each wrapper only fixes the sheet
and table name it reads.

Tasks that disappear from the configured Space/folders result (closed out of
scope, moved, deleted) are not removed from Postgres: `PostgresClient.archive_missing_tasks`
stamps `tarefas.arquivada_em` with the current run's timestamp on every row
whose `task_id` didn't come back in the fetch, keeping the full history instead
of deleting it.

## Layers

| Layer | Modules | Rule |
|---|---|---|
| **Clients** | `clients/clickup_client.py`, `clients/clockify_client.py`, `clients/postgres_client.py` | Speak HTTP/SQL and pagination. `PostgresClient` upserts into the Supabase schema through SQLAlchemy; the other two return raw `dict`, without interpreting anything. |
| **Services** | `services/etl_service.py`, `services/alert_service.py`, `services/employee_data_sync_service.py` | Business logic. `EtlService` and `AlertService` do no network or file I/O; `EmployeeDataSyncService` (renamed from `EmployeeSyncService`) is the deliberate exception, since it orchestrates `ExcelReader` and `PostgresClient` to sync identity (`DIM_FUNCIONARIO`) and area links (`DIM_FUNCIONARIO_AREA` + `FATO_FUNCIONARIO_AREA`) into Postgres. |
| **Integrations** | `integrations/excel_writer.py`, `notifier.py`, `storage_client.py` | Pipeline outputs. Each knows one external destination. |
| **Models** | `models/schemas.py` | Contract between layers. Validation via Pydantic. |
| **Config** | `config/settings.py` | Single point that reads the environment. |
| **Errors** | `errors/exceptions.py` | Business exceptions, all under `SopPipelineError`. |

Dependencies flow inward: `pipeline` → `integrations`/`services` →
`models`/`config`. No client knows about `ExcelWriter`, and `ExcelWriter` doesn't
know about ClickUp.

## Failure isolation

The three synchronization steps each run in their own `try/except` inside
`run()`. ClickUp unavailability doesn't prevent Clockify hours collection, and vice versa. Each failure is logged and sent to Sentry, and execution continues.

If `sync_clickup` fails, the alert step is **skipped** with an explicit `warning`, instead of running against an empty list. The distinction matters: "ClickUp didn't respond"
is not the same as "ClickUp has no at-risk tasks", and treating both situations equally would make a broken execution look clean in the log.

## Observability

| Tool | Role |
|---|---|
| **Sentry** | Receives exceptions caught in the three steps, with stack trace. |
| **Better Stack (logs)** | `LogtailHandler` is attached to the root `logging`; all `logger.info/warning/error` go there. |
| **Better Stack (heartbeat)** | A `GET` at the end of execution. Intentionally **outside** any `try/except`: if the spreadsheet upload fails, the line is never reached and Better Stack flags the missing execution. Wrapping it in a `try` would mark a failed execution as successful. |

## Concurrency and scheduling

The pipeline is single-threaded and designed to run on a schedule (cron, GitHub
Actions, etc.). **Two simultaneous executions are not safe**: both would download the
same spreadsheet, write to separate local copies, and the last one to upload would
overwrite the other. The bucket is not used with locks.
