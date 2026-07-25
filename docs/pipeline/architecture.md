# Arquitetura

> Também publicado como site navegável em
> https://ndelecrodev.github.io/task-time-sync-docs/

## Fluxo de uma execução

Uma execução (`sop_pipeline.pipeline.run`) é uma sequência linear com três etapas
de sincronização isoladas entre si.

```
 1. StorageClient.download_file
    B2 ──────────────────────────────▶ planilha_temp.xlsx (disco local)

 2. EmployeeDataSyncService.sync + .sync_areas
    ExcelReader.read_employees        ── aba DIM_FUNCIONARIO
        │  list[dict]
        ▼
    PostgresClient.upsert_employee  ─▶ tabela funcionarios (Postgres/Supabase)
    ExcelWriter.save_duplicates     ─▶ aba DUPLICADOS_REMOVIDOS (linhas com e-mail repetido)

    name_to_id = {canonical_name: id}, montado a partir de funcionarios (Postgres)

    ExcelReader.read_dim_employee_area   ── aba DIM_FUNCIONARIO_AREA
    ExcelReader.read_fato_employee_area  ── aba FATO_FUNCIONARIO_AREA
        │  list[dict]
        ▼
    PostgresClient.upsert_area_and_link ─▶ tabelas areas, funcionario_area (Postgres/Supabase)

 3. sync_jira
    JiraClient.fetch_tasks(JIRA_JQL)          ── paginação por nextPageToken
        │  list[dict] cru
        ▼
    EtlService.transform_tasks    ─▶ list[Task]
    EtlService.transform_details  ─▶ list[TaskDetail]
        │
        ▼
    ExcelWriter.save_tasks    ─▶ aba BASE_TAREFAS
    ExcelWriter.save_tags     ─▶ abas DIM_ETIQUETAS + FATO_TAREFA_ETIQUETA
    ExcelWriter.save_details  ─▶ aba DETALHES_TAREFA
    PostgresClient.upsert_task / upsert_task_detail / upsert_tag_and_link
                               ─▶ tabelas tarefas, detalhes_tarefa, etiquetas, tarefa_etiqueta
    PostgresClient.archive_missing_tasks
                               ─▶ marca tarefas.arquivada_em nas tarefas que sumiram do JIRA_JQL (nunca apaga)

 4. sync_clockify
    ClockifyClient.list_users
        │  para cada usuário:
        ▼
    ClockifyClient.fetch_time_entries(user_id)  ── paginação por header Last-Page
        │  list[dict] cru
        ▼
    EtlService.transform_time_entries ─▶ list[TimeEntry]
        │
        ▼
    ExcelWriter.save_hours       ─▶ aba BASE_HORAS
    PostgresClient.upsert_time_entry ─▶ tabela horas

 5. StorageClient.upload_file
    planilha_temp.xlsx ──────────────▶ B2   (sobrescreve o objeto)

 6. process_alerts (usa as Tasks da etapa 3)
    AlertService.tasks_to_alert  ─▶ list[Task] dentro da janela de alerta
        │
        ▼
    Notifier.send_alert ─▶ POST no webhook do Teams da área da tarefa

 7. Heartbeat
    GET BETTERSTACK_HEARTBEAT_URL   ── só é alcançado se o upload deu certo
```

O Postgres roda em paralelo à planilha, não no lugar dela: as etapas 3 e 4
gravam as mesmas informações nos dois destinos, um upsert por linha em cada.

`ExcelReader.read_employees`, `read_dim_employee_area` e `read_fato_employee_area`
são wrappers finos sobre um único método genérico, `read_sheet_as_dicts(file_path,
sheet_name, table_name)`, que concentra a lógica de abrir o workbook, achar a
tabela e montar a lista de dicts por linha. Cada wrapper só fixa o nome da aba e
da tabela que lê.

Tarefas que somem do resultado do `JIRA_JQL` (fechadas fora do escopo, movidas,
apagadas) não são removidas do Postgres: `PostgresClient.archive_missing_tasks`
marca `tarefas.arquivada_em` com o timestamp da execução atual em toda linha cujo
`task_id` não veio na busca, mantendo o histórico completo em vez de apagar.

## Camadas

| Camada | Módulos | Regra |
|---|---|---|
| **Clients** | `clients/jira_client.py`, `clients/clockify_client.py`, `clients/postgres_client.py` | Falam HTTP/SQL e paginação. `PostgresClient` faz upsert no schema Supabase via SQLAlchemy; os outros dois devolvem `dict` cru, sem interpretar nada. |
| **Services** | `services/etl_service.py`, `services/alert_service.py`, `services/employee_data_sync_service.py` | Regra de negócio. `EtlService` e `AlertService` não fazem I/O de rede nem de arquivo; `EmployeeDataSyncService` (renomeada de `EmployeeSyncService`) é a exceção deliberada, já que orquestra `ExcelReader` e `PostgresClient` para sincronizar identidade (`DIM_FUNCIONARIO`) e vínculos de área (`DIM_FUNCIONARIO_AREA` + `FATO_FUNCIONARIO_AREA`) com o Postgres. |
| **Integrations** | `integrations/excel_writer.py`, `notifier.py`, `storage_client.py` | Saídas do pipeline. Cada uma conhece um destino externo. |
| **Models** | `models/schemas.py` | Contrato entre as camadas. Validação via Pydantic. |
| **Config** | `config/settings.py` | Único ponto que lê o ambiente. |
| **Errors** | `errors/exceptions.py` | Exceções de negócio, todas sob `SopPipelineError`. |

A dependência é sempre para dentro: `pipeline` → `integrations`/`services` →
`models`/`config`. Nenhum client conhece o `ExcelWriter`, e o `ExcelWriter` não
conhece o Jira.

## Isolamento de falhas

As três etapas de sincronização rodam cada uma no seu próprio `try/except` dentro
de `run()`. Uma indisponibilidade do Jira não impede a coleta das horas do
Clockify, e vice-versa. Cada falha é logada e enviada ao Sentry, e a execução
segue.

Se o `sync_jira` falha, o passo de alertas é **pulado** com um `warning` explícito,
em vez de rodar contra uma lista vazia. A distinção importa: "o Jira não respondeu"
não é a mesma coisa que "o Jira não tem tarefas em risco", e tratar as duas
situações igual fazia uma execução quebrada parecer limpa no log.

## Observabilidade

| Ferramenta | Papel |
|---|---|
| **Sentry** | Recebe as exceções capturadas nas três etapas, com stack trace. |
| **Better Stack (logs)** | `LogtailHandler` é anexado ao `logging` raiz; todos os `logger.info/warning/error` sobem para lá. |
| **Better Stack (heartbeat)** | Um `GET` no fim da execução. Fica **fora** de qualquer `try/except` de propósito: se o upload da planilha falhar, a linha nunca é alcançada e o Better Stack acusa a execução perdida. Colocá-lo dentro de um `try` marcaria como sucesso uma execução que não entregou nada. |

## Concorrência e agendamento

O pipeline é single-threaded e pensado para rodar de forma agendada (cron, GitHub
Actions, etc.). **Duas execuções simultâneas não são seguras**: ambas baixariam a
mesma planilha, escreveriam em cópias locais distintas e a última a subir
sobrescreveria a outra. O bucket não é usado com lock.
