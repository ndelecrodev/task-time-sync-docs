# Architecture

## Flow of a run

A run (`sop_pipeline.pipeline.run`) is a linear sequence with three sync
steps, isolated from each other.

```
 1. StorageClient.download_file
    B2 ──────────────────────────────▶ planilha_temp.xlsx (local disk)

 2. EmployeeSyncService.sync
    ExcelReader.read_employees        ── DIM_FUNCIONARIO sheet
        │  list[dict]
        ▼
    PostgresClient.upsert_employee  ─▶ funcionarios table (Postgres/Supabase)
    ExcelWriter.save_duplicates     ─▶ DUPLICADOS_REMOVIDOS sheet (rows with a repeated email)

 3. sync_jira
    JiraClient.fetch_tasks(JIRA_JQL)          ── pagination via nextPageToken
        │  raw list[dict]
        ▼
    EtlService.transform_tasks    ─▶ list[Task]
    EtlService.transform_details  ─▶ list[TaskDetail]
        │
        ▼
    ExcelWriter.save_tasks    ─▶ BASE_TAREFAS sheet
    ExcelWriter.save_tags     ─▶ DIM_ETIQUETAS + FATO_TAREFA_ETIQUETA sheets
    ExcelWriter.save_details  ─▶ DETALHES_TAREFA sheet
    PostgresClient.upsert_task / upsert_task_detail / upsert_tag_and_link
                               ─▶ tarefas, detalhes_tarefa, etiquetas, tarefa_etiqueta tables

 4. sync_clockify
    ClockifyClient.list_users
        │  for each user:
        ▼
    ClockifyClient.fetch_time_entries(user_id)  ── pagination via the Last-Page header
        │  raw list[dict]
        ▼
    EtlService.transform_time_entries ─▶ list[TimeEntry]
        │
        ▼
    ExcelWriter.save_hours       ─▶ BASE_HORAS sheet
    PostgresClient.upsert_time_entry ─▶ horas table

 5. StorageClient.upload_file
    planilha_temp.xlsx ──────────────▶ B2   (overwrites the object)

 6. process_alerts (uses the Tasks from step 3)
    AlertService.tasks_to_alert  ─▶ list[Task] within the alert window
        │
        ▼
    Notifier.send_alert ─▶ POST to the Teams webhook for the task's area

 7. Heartbeat
    GET BETTERSTACK_HEARTBEAT_URL   ── only reached if the upload succeeded
```

Postgres runs in parallel with the spreadsheet, not in its place: steps 3
and 4 write the same information to both destinations, one upsert per row
in each.

## Layers

| Layer | Modules | Rule |
|---|---|---|
| **Clients** | `clients/jira_client.py`, `clients/clockify_client.py`, `clients/postgres_client.py` | Handle HTTP/SQL and pagination. `PostgresClient` upserts into the Supabase schema via SQLAlchemy; the other two return raw `dict`s, without interpreting anything. |
| **Services** | `services/etl_service.py`, `services/alert_service.py`, `services/employee_sync_service.py` | Business logic. `EtlService` and `AlertService` do no network or file I/O; `EmployeeSyncService` is the deliberate exception, since it orchestrates `ExcelReader` and `PostgresClient` to sync `DIM_FUNCIONARIO`. |
| **Integrations** | `integrations/excel_writer.py`, `notifier.py`, `storage_client.py` | Pipeline outputs. Each one knows about a single external destination. |
| **Models** | `models/schemas.py` | Contract between layers. Validated via Pydantic. |
| **Config** | `config/settings.py` | The only place that reads the environment. |
| **Errors** | `errors/exceptions.py` | Business exceptions, all under `SopPipelineError`. |

Dependencies always point inward: `pipeline` → `integrations`/`services` →
`models`/`config`. No client knows about `ExcelWriter`, and `ExcelWriter`
doesn't know about Jira.

## Failure isolation

Each of the three sync steps runs in its own `try/except` inside `run()`.
A Jira outage doesn't prevent collecting Clockify hours, and vice versa.
Each failure is logged and sent to Sentry, and the run continues.

If `sync_jira` fails, the alert step is **skipped** with an explicit
`warning`, instead of running against an empty list. The distinction
matters: "Jira didn't respond" isn't the same thing as "Jira has no tasks
at risk," and treating the two the same made a broken run look clean in
the log.

## Observability

| Tool | Role |
|---|---|
| **Sentry** | Receives the exceptions caught in the three steps, with stack trace. |
| **Better Stack (logs)** | `LogtailHandler` is attached to the root `logging`; every `logger.info/warning/error` call goes there. |
| **Better Stack (heartbeat)** | A `GET` at the end of the run. It's deliberately **outside** any `try/except`: if the spreadsheet upload fails, that line is never reached and Better Stack flags the run as missed. Putting it inside a `try` would mark a run that delivered nothing as a success. |

## Concurrency and scheduling

The pipeline is single-threaded and designed to run on a schedule (cron,
GitHub Actions, etc.). **Two simultaneous runs are not safe**: both would
download the same spreadsheet, write to separate local copies, and
whichever uploads last would overwrite the other. The bucket isn't used
with a lock.
