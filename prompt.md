Context: this is the task-time-sync-docs repo (MkDocs Material site).
Working tree only, do not commit.

=== TASK: Replace three pages under docs/pipeline/ with the current
task-time-sync content (Jira-to-ClickUp migration) ===

Replace the full content of docs/pipeline/architecture.md and
docs/pipeline/data-model.md (Portuguese/default) with the content
below. Append the design-decisions.md content below to the END of the
existing docs/pipeline/design-decisions.md (entries #1-19 already
there are correct and must not be touched or duplicated).

Apply the same three operations to the English counterparts
(docs/pipeline/architecture.en.md or docs/en/... depending on which
naming convention this repo already uses — check before writing).

--- docs/pipeline/architecture.md (PT) ---
# Arquitetura

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

 3. sync_clickup
    ClickUpClient.fetch_tasks(CLICKUP_TEAM_ID, CLICKUP_SPACE_ID) ── paginação por
                               número de página, GET /team/{team_id}/task
        │  list[dict] cru (toda tarefa do Space, de qualquer pasta)
        ▼
    pipeline._filter_allowed_folders
        │  descarta tarefas cuja folder.id não está em CLICKUP_FOLDER_IDS
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
                               ─▶ marca tarefas.arquivada_em nas tarefas que sumiram da busca (nunca apaga)

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

Tarefas que somem do resultado da busca ao Space/pastas configurados (fechadas
fora do escopo, movidas, apagadas) não são removidas do Postgres:
`PostgresClient.archive_missing_tasks` marca `tarefas.arquivada_em` com o
timestamp da execução atual em toda linha cujo `task_id` não veio na busca,
mantendo o histórico completo em vez de apagar.

## Camadas

| Camada | Módulos | Regra |
|---|---|---|
| **Clients** | `clients/clickup_client.py`, `clients/clockify_client.py`, `clients/postgres_client.py` | Falam HTTP/SQL e paginação. `PostgresClient` faz upsert no schema Supabase via SQLAlchemy; os outros dois devolvem `dict` cru, sem interpretar nada. |
| **Services** | `services/etl_service.py`, `services/alert_service.py`, `services/employee_data_sync_service.py` | Regra de negócio. `EtlService` e `AlertService` não fazem I/O de rede nem de arquivo; `EmployeeDataSyncService` (renomeada de `EmployeeSyncService`) é a exceção deliberada, já que orquestra `ExcelReader` e `PostgresClient` para sincronizar identidade (`DIM_FUNCIONARIO`) e vínculos de área (`DIM_FUNCIONARIO_AREA` + `FATO_FUNCIONARIO_AREA`) com o Postgres. |
| **Integrations** | `integrations/excel_writer.py`, `notifier.py`, `storage_client.py` | Saídas do pipeline. Cada uma conhece um destino externo. |
| **Models** | `models/schemas.py` | Contrato entre as camadas. Validação via Pydantic. |
| **Config** | `config/settings.py` | Único ponto que lê o ambiente. |
| **Errors** | `errors/exceptions.py` | Exceções de negócio, todas sob `SopPipelineError`. |

A dependência é sempre para dentro: `pipeline` → `integrations`/`services` →
`models`/`config`. Nenhum client conhece o `ExcelWriter`, e o `ExcelWriter` não
conhece o ClickUp.

## Isolamento de falhas

As três etapas de sincronização rodam cada uma no seu próprio `try/except` dentro
de `run()`. Uma indisponibilidade do ClickUp não impede a coleta das horas do
Clockify, e vice-versa. Cada falha é logada e enviada ao Sentry, e a execução
segue.

Se o `sync_clickup` falha, o passo de alertas é **pulado** com um `warning`
explícito, em vez de rodar contra uma lista vazia. A distinção importa: "o
ClickUp não respondeu" não é a mesma coisa que "o ClickUp não tem tarefas em
risco", e tratar as duas situações igual fazia uma execução quebrada parecer
limpa no log.

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


--- docs/pipeline/architecture.md (EN) ---
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


--- docs/pipeline/data-model.md (PT) ---
# Modelo de dados

## Modelos Python

Definidos em `src/sop_pipeline/models/schemas.py`. São modelos Pydantic: uma
tarefa do ClickUp que não satisfaça o contrato é descartada com um `warning`, em
vez de derrubar a execução inteira.

### Identidade de colaboradores

Como o ClickUp identifica pessoas por `username`/e-mail do assignee e o Clockify
por e-mail, uma camada de mapeamento normaliza os dois para um nome canônico
usado como chave de junção.

**Fonte editável:** a aba `DIM_FUNCIONARIO` da planilha é onde alguém corrige ou
adiciona colaboradores manualmente. Antes de cada execução, `EmployeeDataSyncService`
lê essa aba com `ExcelReader` e grava as linhas na tabela `funcionarios` do
Postgres via `PostgresClient.upsert_employee`, casando por `clickup_email` ou
`clockify_email` — a coluna, antes chamada `jira_email` por herança do schema
implantado na era Jira, foi renomeada para `clickup_email` (ver
[`design-decisions.md`](design-decisions.md#22)) para deixar de carregar um nome
que não fazia mais sentido pós-migração; ela guarda o e-mail registrado do
colaborador, casado contra o assignee do ClickUp.

**Duplicatas:** a primeira linha a usar um dado `clickup_email` ou `clockify_email`
é sincronizada normalmente; qualquer linha seguinte que repita um dos dois é
tratada como duplicata (`EmployeeDataSyncService._split_duplicates`), recebe um
motivo e é gravada na aba `DUPLICADOS_REMOVIDOS` em vez de ser sincronizada.

**Uso em runtime:** `Settings.load_employee_registry` lê a tabela `funcionarios`
já sincronizada e monta um `EmployeeRegistry`, usado pelo `EtlService` para
normalizar `Task.assignee` (vindo do ClickUp) e `TimeEntry.employee` (vindo do
Clockify) para o nome canônico.

**Foto do colaborador:** `funcionarios.photo_url` guarda a URL pública do
Supabase Storage da foto do colaborador, ou `None` quando nenhuma foto foi
enviada ainda. `EmployeeDataSyncService` propaga o valor lido de `DIM_FUNCIONARIO`
a cada sincronização; é esse campo que o dashboard de indicadores (repositório
separado) consome para exibir a foto de cada pessoa.

**Colaboradores não mapeados:** se um colaborador não é encontrado no registro,
ele recebe um valor sentinela visível (`"Unmapped employee: <email>"`) em vez de
ser descartado silenciosamente. Segue a decisão de design #8: um registro ruim
não derruba a execução inteira, e problemas de qualidade de dados ficam visíveis
no relatório em vez de escondidos.

### `Task`

Uma tarefa do ClickUp normalizada. Chave de upsert: `task_id` (o `id` do
ClickUp).

| Campo | Tipo | Origem no ClickUp |
|---|---|---|
| `task_id` | `str` | `id` |
| `title` | `str` | `name` (default `"No title"`) |
| `assignee` | `str` | `assignees[*].username`/`.email`, normalizados individualmente e concatenados com `", "` |
| `priority` | `Priority` | `priority.priority` (`urgent`/`high`/`normal`/`low`), mapeado para o enum |
| `status` | `str` | `status.status` |
| `area` | `str \| None` | opção resolvida do custom field `drop_down` configurado em `CLICKUP_AREA_FIELD_ID` |
| `creation_date` | `date` | `date_created` (timestamp em milissegundos) |
| `due_date` | `date \| None` | `due_date` (timestamp em milissegundos) |
| `completion_date` | `date \| None` | `date_closed` (timestamp em milissegundos) |
| `task_type` | `TaskType` | fixo em `TaskType.TASK` — ver [`design-decisions.md`](design-decisions.md#22) |
| `creator` | `str \| None` | `creator.username` |
| `update_date` | `date \| None` | `date_updated` (timestamp em milissegundos) |
| `assignee_email` | `EmailStr \| None` | `assignees[0].email`, com fallback em `EmployeeRegistry.get_registered_email` |
| `tags` | `list[str]` | `tags[*].name` |
| `turma` | `str` | `folder.name` — lido direto do ClickUp, nunca digitado por alguém; ver [`design-decisions.md`](design-decisions.md#23) |

**Múltiplos responsáveis:** ao contrário do Jira, o ClickUp permite mais de um
assignee por tarefa. Cada um é normalizado individualmente por
`normalize_employee_identifier` (por e-mail quando presente, senão por
`username`) e os nomes canônicos resultantes são concatenados em `assignee`. Só
o e-mail do **primeiro** assignee alimenta `assignee_email`, porque uma
@menção do Teams só pode apontar para uma pessoa — ver
[`design-decisions.md`](design-decisions.md#22).

**Área (`drop_down` custom field):** o ClickUp devolve `custom_fields` como uma
lista de objetos, cada um com `id`, `name`, `type` e, opcionalmente, `value`.
Para o campo do tipo `drop_down` configurado em `CLICKUP_AREA_FIELD_ID`,
`value` é um **índice** dentro do array `type_config.options` do próprio
campo, não o texto da opção — o índice é resolvido contra esse array para
obter o nome da área. Quando o campo está ausente, não tem chave `value`, ou o
índice não resolve, o resultado é `NO_AREA`.

**Milissegundos:** `date_created`, `due_date`, `date_closed` e `date_updated`
chegam como strings de timestamp Unix em milissegundos (ex.:
`"1753401600000"`), não em ISO 8601 como no Jira.
`EtlService._parse_millis_to_date` converte cada um para uma `date` em
America/Sao_Paulo, tratando `None`.

Campos calculados (`@computed_field`), usados pela regra de alerta e pelo texto da
notificação — **não** são gravados na planilha, que tem as próprias fórmulas:

| Campo | Retorno |
|---|---|
| `days_remaining` | Dias até o prazo; negativo se já venceu; `None` sem prazo. |
| `is_late` | `"SIM"` / `"NÃO"` / `None`. |
| `deadline_status` | Um valor de `DeadlineStatus`. |

`Notifier._build_message` nunca mostra `days_remaining` negativo diretamente:
para uma tarefa vencida, a linha da notificação vira "Tarefa atrasada há N
dia(s)" (com N positivo); para uma tarefa no prazo, "Dias restantes: N
dia(s)"; sem prazo definido, "Dias restantes: Indefinido".

Quando o ClickUp não traz responsável ou área, o ETL usa os textos
`"There is no one responsible."` e `"There is no one area."` em vez de descartar a
linha — a tarefa continua aparecendo no relatório.

### `TimeEntry`

Um apontamento de horas do Clockify. Chave de upsert: `entry_id`.

| Campo | Tipo | Origem no Clockify |
|---|---|---|
| `entry_id` | `str` | `id` |
| `employee` | `str` | e-mail resolvido a partir de `userId` |
| `entry_date` | `date` | `timeInterval.start`, convertido de UTC para America/Sao_Paulo |
| `hours` | `float` (≥ 0) | `timeInterval.duration` (ISO 8601) convertido em horas |

Entradas com `duration` nulo (timer ainda rodando) são ignoradas e recolhidas em
uma execução posterior.

### `TaskDetail`

A descrição longa de uma tarefa, separada porque é um texto grande e fica em aba
própria. Chave de upsert: `task_id`.

| Campo | Tipo | Origem |
|---|---|---|
| `task_id` | `str` | `id` |
| `description` | `str \| None` | `description`, com fallback em `text_content` quando ausente |

### Enums

| Enum | Valores |
|---|---|
| `Priority` | `Highest`, `High`, `Medium`, `Low`, `Lowest` |
| `TaskType` | `Bug`, `Task`, `Story`, `Epic`, `Subtask` |
| `DeadlineStatus` | `Concluído`, `Atrasado`, `Atenção`, `No prazo`, `Sem prazo` |

Os valores de `Priority` são os rótulos históricos do Jira; o ETL mapeia os
quatro níveis de prioridade do ClickUp (`urgent`/`high`/`normal`/`low`) para
eles — ver [`design-decisions.md`](design-decisions.md#22). `TaskType` fica
fixo em `Task` para toda tarefa vinda do ClickUp, pelo mesmo motivo. Os
valores de `DeadlineStatus` são exatamente as strings que a fórmula da coluna
`status_prazo` produz no Excel. **Nenhum desses valores pode ser traduzido** —
só os nomes dos membros do enum.

---

## Mapeamento para a planilha

Convenções do arquivo `.xlsx`: nome de aba em MAIÚSCULAS, nome de tabela em
minúsculas, primeira coluna sempre é o ID usado no upsert.

### Abas escritas pelo Python

| Aba | Tabela | Colunas | Escrita por |
|---|---|---|---|
| `BASE_TAREFAS` | `base_tarefas` | id, titulo, responsavel, area, prioridade, status, data_criacao, prazo, data_conclusao, **dias_restantes**, **atrasado**, **status_prazo**, tipo, criador, data_atualizacao, arquivada_em, turma | `save_tasks` |
| `DETALHES_TAREFA` | `detalhes_tarefa` | id, descricao | `save_details` |
| `BASE_HORAS` | `base_horas` | id, funcionario, data, horas | `save_hours` |
| `DIM_ETIQUETAS` | `dim_etiquetas` | id_etiqueta, nome_etiqueta | `save_tags` |
| `FATO_TAREFA_ETIQUETA` | `fato_tarefa_etiqueta` | id_tarefa, id_etiqueta | `save_tags` |

As três colunas em **negrito** são calculadas por fórmula do Excel; o Python
apenas replica o texto da fórmula quando cria uma linha nova. `arquivada_em`
também foge à regra: quem escreve nela é `ExcelWriter.mark_archived_tasks`, não
`save_tasks`, e é a única coluna ainda tocada numa linha depois que a tarefa
some do ClickUp — todo o resto da linha arquivada fica congelado no último
valor conhecido, por design (ver decisão de design #19).

`Task` → `BASE_TAREFAS` é definido pelo dict `TASK_COLUMN_MAP` em
`integrations/excel_writer.py`. O lado esquerdo é o cabeçalho literal da coluna na
planilha (em português, por isso não é traduzido); o lado direito é o nome do
atributo no modelo:

```python
TASK_COLUMN_MAP = {
    "id": "task_id",
    "titulo": "title",
    "responsavel": "assignee",
    ...
}
```

`DETALHES_TAREFA` e `BASE_HORAS` têm layout fixo e curto, então são escritas por
posição de coluna, sem passar pelo mapa de cabeçalhos.

### Abas que o Python não escreve

Existem no arquivo e são mantidas manualmente ou por fórmula. `DIM_FUNCIONARIO`
é a exceção parcial: ninguém escreve nela por código, mas `ExcelReader` a lê a
cada execução para sincronizar o cadastro com o Postgres (ver "Identidade de
colaboradores" acima).

| Aba | Tabela | Papel |
|---|---|---|
| `DIM_FUNCIONARIO` | `dim_funcionario` | Cadastro de colaboradores (id_funcionario, nome, email), lido por `ExcelReader`. |
| `DIM_FUNCIONARIO_AREA` | `dim_func_area` | Dimensão de áreas (id_area, nome_area). |
| `FATO_FUNCIONARIO_AREA` | `fato_funcionario` | Relação N:N entre colaborador e área. |
| `CALCULOS` | `Tabela6` | Métricas por pessoa (tarefas, concluídas, atrasadas, horas, produtividade). |
| `INDICADORES` | `Tabela7`, `Tabela10`, `Tabela11`, `Tabela12` | KPIs consolidados do dashboard. |

### Relações

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

BASE_HORAS (funcionario) ──── liga-se a DIM_FUNCIONARIO por e-mail
```

## Esquema no Postgres

Definido em `src/sop_pipeline/clients/postgres_client.py` como modelos
SQLAlchemy (ORM), não como SQL cru. O ORM também foi escolhido pelo objetivo
de aprendizado do projeto; ver a decisão de design correspondente. Nomes de
tabela e de coluna espelham o schema já implantado no Supabase e por isso
permanecem em português; os métodos de upsert e o próprio `PostgresClient`
estão em inglês. Este schema roda em paralelo à planilha, não no lugar dela:
toda gravação do pipeline em `tarefas`, `horas`, `etiquetas` etc. tem uma
gravação equivalente na aba correspondente do `.xlsx`.

| Tabela | Papel | Upsert por |
|---|---|---|
| `funcionarios` | Identidade de colaboradores, sincronizada a partir de `DIM_FUNCIONARIO`. Inclui `photo_url`, a URL da foto usada pelo dashboard. | `upsert_employee` |
| `tarefas` | Uma linha por tarefa do ClickUp; `responsavel_id` é `NULL` quando o colaborador não foi mapeado. `arquivada_em` guarda o timestamp em que a tarefa deixou de aparecer na busca (`CLICKUP_SPACE_ID` + `CLICKUP_FOLDER_IDS`, `NULL` enquanto ativa); a linha nunca é apagada. `turma` guarda o nome da pasta do ClickUp (ver [`design-decisions.md`](design-decisions.md#23)), lido direto da API, nunca digitado por alguém. | `upsert_task` (arquivamento: `archive_missing_tasks`) |
| `detalhes_tarefa` | Descrição longa de uma tarefa. | `upsert_task_detail` |
| `horas` | Um apontamento de horas do Clockify; `funcionario_id` é `NULL` quando o colaborador não foi mapeado. | `upsert_time_entry` |
| `etiquetas` | Tags distintas atribuídas a tarefas. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | Associação N:N entre `tarefas` e `etiquetas`. | `upsert_tag_and_link` |
| `areas` | Áreas de atuação dos colaboradores, sincronizadas a partir de `DIM_FUNCIONARIO_AREA`. | `upsert_area_and_link` |
| `funcionario_area` | Associação N:N entre `funcionarios` e `areas`, sincronizada a partir de `FATO_FUNCIONARIO_AREA`; chave composta `funcionario_id` + `area_id`, ambas FK. | `upsert_area_and_link` |


--- docs/pipeline/data-model.md (EN) ---
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
renamed to `clickup_email` (see [`design-decisions.md`](design-decisions.md#22))
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
| `task_type` | `TaskType` | fixed to `TaskType.TASK` — see [`design-decisions.md`](design-decisions.md#22) |
| `creator` | `str \| None` | `creator.username` |
| `update_date` | `date \| None` | `date_updated` (millisecond timestamp) |
| `assignee_email` | `EmailStr \| None` | `assignees[0].email`, falling back to `EmployeeRegistry.get_registered_email` |
| `tags` | `list[str]` | `tags[*].name` |
| `turma` | `str` | `folder.name` — read straight from ClickUp, never user-entered; see [`design-decisions.md`](design-decisions.md#23) |

**Multiple assignees:** unlike Jira, ClickUp allows more than one assignee per
task. Each is normalized individually by `normalize_employee_identifier` (by
email when present, otherwise by `username`), and the resulting canonical
names are joined into `assignee`. Only the **first** assignee's email feeds
`assignee_email`, because a Teams @mention can only target one person — see
[`design-decisions.md`](design-decisions.md#22).

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

`Priority`'s values are Jira's historical labels; the ETL maps ClickUp's four priority levels (`urgent`/`high`/`normal`/`low`) onto them — see [`design-decisions.md`](design-decisions.md#22). `TaskType` is fixed to `Task` for every ClickUp-sourced task, for the same reason. Those of `DeadlineStatus` are exactly the strings the `status_prazo` column formula produces in Excel. **None of these values can be translated** — only the enum member names.

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
| `tarefas` | One row per ClickUp task; `responsavel_id` is `NULL` when the employee could not be mapped. `arquivada_em` holds the timestamp when the task stopped appearing in the fetch (`CLICKUP_SPACE_ID` + `CLICKUP_FOLDER_IDS`, `NULL` while active); the row is never deleted. `turma` holds the ClickUp folder's name (see [`design-decisions.md`](design-decisions.md#23)), read straight from the API, never user-entered. | `upsert_task` (archiving: `archive_missing_tasks`) |
| `detalhes_tarefa` | Long-form task description. | `upsert_task_detail` |
| `horas` | One Clockify time entry; `funcionario_id` is `NULL` when the employee could not be mapped. | `upsert_time_entry` |
| `etiquetas` | Distinct tags assigned to tasks. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | N:N association between `tarefas` and `etiquetas`. | `upsert_tag_and_link` |
| `areas` | Employee work areas, synced from `DIM_FUNCIONARIO_AREA`. | `upsert_area_and_link` |
| `funcionario_area` | N:N association between `funcionarios` and `areas`, synced from `FATO_FUNCIONARIO_AREA`; composite key `funcionario_id` + `area_id`, both FKs. | `upsert_area_and_link` |


--- APPEND to docs/pipeline/design-decisions.md (PT) ---
## 20. `HISTORICO_PROGRESSO.percentual` é gravado como valor, não como fórmula do Excel

`ExcelWriter.save_progress_snapshot` calcula `percentual = concluidas /
total_tarefas` em Python e escreve o resultado na célula. Diferente de
`dias_restantes`, `atrasado` e `status_prazo` (decisão 2), essa coluna não
está de fora do que o Python grava — ela é sempre um valor estático.

**Por quê:** a decisão 2 existe porque aquelas colunas descrevem o estado
*atual* de uma tarefa ainda aberta, e por isso devem recalcular sozinhas toda
vez que alguém abre a planilha. `HISTORICO_PROGRESSO` é o caso oposto: cada
linha é um retrato do progresso em uma data específica (`snapshot_date`), e a
razão de existir dessa aba é justamente preservar esse retrato. Se
`percentual` fosse fórmula, ela recalcularia com os totais de hoje toda vez
que o arquivo fosse aberto, e cada linha antiga passaria a mentir sobre o
que o progresso realmente era na data que ela representa — apagando
silenciosamente o próprio histórico que a aba deveria guardar. Gravar o
valor no momento do snapshot é o que faz da linha um registro histórico de
verdade, em vez de mais uma visão do presente.

## 21. `EtlService._build_task` recorre a `EmployeeRegistry.get_registered_email` quando a fonte de tarefas não informa o e-mail do assignee

Quando um assignee de fato existe mas não vem com e-mail, `_build_task`
primeiro normaliza o identificador bruto (hoje `username` do ClickUp; era
`displayName` do Jira) para o nome canônico via
`normalize_employee_identifier`, e só então chama
`EmployeeRegistry.get_registered_email` com esse nome já canonicalizado,
nunca com o identificador bruto. O fallback não roda no branch de tarefa sem
responsável (lista de assignees vazia), onde `assignee` é o sentinel
`NO_RESPONSIBLE` — chamar `get_registered_email` com um sentinel como se
fosse nome de pessoa não faz sentido e nunca deve acontecer. O método se
chamava `get_jira_email` até a migração para ClickUp (decisão 22); foi
renomeado porque passou a resolver o e-mail registrado do colaborador
independente da fonte de tarefas, e não apenas o do Jira.

**Por quê:** configurações de privacidade de visibilidade de e-mail por
usuário — no Jira Cloud, uma mudança da era GDPR; no ClickUp, uma
possibilidade equivalente — podem deixar o e-mail do assignee nulo mesmo
para alguém corretamente atribuído e visível em qualquer outro lugar da
ferramenta. Usar o nome já canonicalizado, e não o identificador bruto da
fonte, é essencial: os nomes cadastrados na `DIM_FUNCIONARIO` podem divergir
do que a fonte de tarefas retorna, e foi exatamente essa divergência que
causou um bug anterior envolvendo um colaborador chamado "Miguel Felix
Cardozo de Tomy" — buscar pelo nome bruto teria o mesmo problema aqui. Com
isso, o cadastro de colaboradores (`DIM_FUNCIONARIO` / `EmployeeRegistry`)
passa a ser a segunda fonte de verdade para o e-mail de um colaborador,
especificamente para permitir @menções no Teams quando a própria fonte de
tarefas não fornece um e-mail.

## 22. A fonte de tarefas migrou de Jira para ClickUp só na camada de extração; o `Task` continua sendo o contrato

`clients/jira_client.py` foi substituído por `clients/clickup_client.py`, e
`EtlService._build_task`/`transform_details` foram reescritos para o formato
do ClickUp (lista de assignees, `custom_fields` como lista, timestamps em
milissegundos, etc. — ver [`data-model.md`](data-model.md)). O Clockify, a
saída Excel, a saída Postgres, os alertas do Teams, a resolução de identidade
de colaboradores e o arquivamento de tarefas continuam exatamente como
estavam: nenhum desses módulos conhecia o Jira diretamente, só o `Task`
(`models/schemas.py`), e o `Task` não mudou.

**Por quê:** o projeto trocou de ferramenta de gestão de tarefas, mas o
esquema do Postgres, as fórmulas e abas da planilha, e os fluxos de alerta do
Teams não têm motivo para mudar por causa disso — são todos consumidores do
modelo `Task`, não da API de origem. Manter o `Task` intacto (a decisão mais
importante desta migração) transformou uma troca de fornecedor em uma mudança
contida à camada de extração: só o client HTTP e o mapeamento dict→`Task`
precisaram ser reescritos. Isso confirma, na prática, o desenho descrito na
decisão de [arquitetura](architecture.md#camadas): "nenhum client conhece o
`ExcelWriter`, e o `ExcelWriter` não conhece o Jira" — agora vale trocando
"Jira" por "ClickUp".

Identificadores que são **dados**, não código — a coluna `funcionarios.jira_email`
no Postgres, o cabeçalho `jira_email` na aba `DIM_FUNCIONARIO` do Excel, e o
campo `EmployeeMapping.jira_email` que espelha os dois — foram inicialmente
mantidos com o nome antigo, pela mesma razão da decisão 6: renomeá-los
quebraria o casamento em runtime contra o schema já implantado no Supabase e
contra a planilha real, sem erro de importação para avisar. Só o método
`EmployeeRegistry.get_jira_email` foi renomeado nesse momento (decisão 21),
por ser código, não dado.

**Renomeação posterior de `jira_email` para `clickup_email`:** numa tarefa
seguinte, os três identificadores de dados acima foram de fato renomeados —
a coluna `funcionarios.jira_email` e sua check constraint `check_email_jira`
no Postgres, o cabeçalho `jira_email` na aba `DIM_FUNCIONARIO` do Excel, e o
campo `EmployeeMapping.jira_email`, todos passaram a `clickup_email` /
`check_email_clickup`. A justificativa: diferente do rename de código feito
na decisão 21 (adiado até o próprio símbolo deixar de fazer sentido), manter
um identificador de dados chamado `jira_email` por tempo indefinido — agora
que o projeto não fala mais com o Jira — era o tipo de nome que confunde
sem necessidade a próxima pessoa que ler o schema, sem nenhum benefício de
compatibilidade real: a coluna e o cabeçalho podiam ser renomeados atômica e
deliberadamente (migração de schema no Postgres, atualização manual do
cabeçalho no Excel via Claude for Excel, e o campo Python correspondente),
ao contrário de uma renomeação silenciosa e não coordenada. O nome escolhido,
`clickup_email`, segue a convenção já usada pela coluna irmã
`clockify_email` — nomear pelo sistema concreto integrado, não por um termo
genérico como "task_source_email" — e é o mesmo padrão dos identificadores
`CLICKUP_*` já presentes em `Settings` e do módulo `clients/clickup_client.py`.

**Múltiplos responsáveis, e o trade-off do @mention:** diferente do Jira, o
ClickUp permite mais de um assignee por tarefa. Cada assignee é normalizado
individualmente e os nomes canônicos são concatenados em `Task.assignee`
(`"Nicolas Delecrode, Daniel Nogueira"`), decisão do dono do projeto. Mas
`Task.assignee_email` continua sendo um único e-mail — ele alimenta uma única
@menção no Teams, e uma @menção não pode apontar para várias pessoas ao mesmo
tempo — então só o e-mail do **primeiro** assignee é usado. Uma tarefa com
múltiplos responsáveis sempre notifica apenas o primeiro deles por e-mail;
os demais aparecem no relatório (na coluna `responsavel` da planilha e em
`tarefas` no Postgres) mas não recebem @menção direta.

**Trade-off do `task_type`:** o ClickUp não tem um equivalente direto ao
`issuetype` do Jira — o único candidato, `custom_item_id`, só existe quando o
workspace usa a funcionalidade paga de Custom Task Types do ClickUp, o que
não é o caso deste workspace. Como `Task.task_type` é campo obrigatório e o
`Task` não podia mudar, toda tarefa vinda do ClickUp recebe `TaskType.TASK`
fixo, decisão do dono do projeto. Isso significa que o relatório perde a
distinção Bug/Story/Epic/Subtask que existia com o Jira; se o workspace algum
dia adotar Custom Task Types, `_build_task` precisará ser revisitado para
mapear `custom_item_id` em vez de usar o valor fixo.

**Trade-off da prioridade nula:** o ClickUp representa "sem prioridade" como
`priority: null` no payload (em vez do objeto Jira com `name` ausente). O
mapeamento (`urgent`→Highest, `high`→High, `normal`→Medium, `low`→Low) só se
aplica quando `priority` não é nulo; quando é nulo, `Task.priority` fica sem
valor e a validação do Pydantic falha, descartando a tarefa pelo mesmo
caminho que já existia (decisão 8) — mesmo comportamento que uma issue do
Jira sem prioridade sempre teve.

## 23. As pastas do ClickUp sincronizadas são uma allowlist explícita, não "toda pasta do Space"

`ClickUpClient.fetch_tasks` busca todo o Space configurado em
`CLICKUP_SPACE_ID` (`GET /team/{team_id}/task` com `space_ids[]=...`), o que
inclui qualquer pasta que exista ali, de qualquer natureza.
`pipeline._filter_allowed_folders` roda logo em seguida e descarta toda
tarefa cujo `folder.id` não esteja em `CLICKUP_FOLDER_IDS` — uma lista fixa,
configurada em `.env`, dos IDs das pastas que representam de fato uma
"turma" (hoje "Primeiro Ano" e "Segundo Ano") — antes mesmo de `EtlService`
enxergar essas tarefas.

**Por quê:** a alternativa óbvia seria sincronizar automaticamente toda pasta
que existir no Space, sem lista fixa. Isso foi deliberadamente rejeitado: uma
pasta sem relação com uma "turma" pode ser criada no mesmo Space no futuro —
por exemplo, uma pasta de planejamento interno da equipe, ou um experimento
temporário — e nada nela garante que suas tarefas sigam o mesmo contrato
(`turma`, `area`, prioridades) que o resto do pipeline espera. Sem a
allowlist, essa pasta começaria a alimentar o pipeline, os alertas do Teams e
os relatórios apenas por ter sido criada no Space, sem ninguém ter decidido
isso conscientemente.

Essa é a mesma filosofia de "nunca mudar de escopo silenciosamente" que já
aparece neste projeto, só que na direção oposta: a decisão 8 (nunca
descartar um registro silenciosamente) e a decisão 18 (nunca apagar uma
tarefa arquivada silenciosamente) protegem contra **perder** dado sem
aviso; a allowlist de pastas protege contra **ganhar** escopo sem aviso.
Em ambos os casos, o princípio é que uma mudança de escopo — para dentro ou
para fora — deve ser um ato deliberado, não um efeito colateral de algo
que aconteceu em outro sistema (o Jira antes, o ClickUp agora).

**Trade-off:** quando uma nova turma for criada de fato (por exemplo,
"Terceiro Ano"), sincronizá-la exige uma ação manual: alguém precisa
adicionar o ID da nova pasta a `CLICKUP_FOLDER_IDS` e reiniciar a execução
agendada. Não há descoberta automática de novas turmas. Esse atrito é
proposital — é o preço de nunca incluir uma pasta por engano.

--- APPEND to docs/pipeline/design-decisions.md (EN) ---
## 20. `HISTORICO_PROGRESSO.percentual` is written as a value, not an Excel formula

`ExcelWriter.save_progress_snapshot` computes `percentual = concluidas /
total_tarefas` in Python and writes the result into the cell. Unlike
`dias_restantes`, `atrasado`, and `status_prazo` (decision 2), this column
is not left out of what Python writes — it is always a static value.

**Why:** decision 2 exists because those columns describe the *current*
state of a task that is still open, and so they need to recalculate on
their own every time someone opens the spreadsheet. `HISTORICO_PROGRESSO`
is the opposite case: each row is a snapshot of progress on a specific date
(`snapshot_date`), and the entire reason that sheet exists is to preserve
that snapshot. If `percentual` were a formula, it would recompute against
today's totals every time the file is opened, and every past row would
silently start lying about what progress actually was on the date it
represents — erasing the very history the sheet is meant to keep. Writing
the value at snapshot time is what makes the row a genuine historical
record instead of one more view of the present.

## 21. `EtlService._build_task` falls back to `EmployeeRegistry.get_registered_email` when the task source omits the assignee's email

When an assignee actually exists but comes without an email, `_build_task`
first normalizes the raw identifier (today ClickUp's `username`; it used to
be Jira's `displayName`) to the canonical name via
`normalize_employee_identifier`, and only then calls
`EmployeeRegistry.get_registered_email` with that already-canonicalized
name, never with the raw identifier. The fallback does not run on the
no-assignee branch (an empty assignee list), where `assignee` is the
`NO_RESPONSIBLE` sentinel — calling `get_registered_email` with a sentinel
as if it were a person's name makes no sense and must never happen. The
method was named `get_jira_email` until the ClickUp migration (decision
22); it was renamed because it now resolves an employee's registered email
regardless of the task source, not just Jira's.

**Why:** per-user email-visibility privacy settings — a GDPR-era change on
Jira Cloud, an equivalent possibility on ClickUp — can leave the assignee's
email null even for someone correctly assigned and visible everywhere else
in the tool. Using the already-canonicalized name instead of the source's
raw identifier is essential: names registered in `DIM_FUNCIONARIO` can
differ from what the task source returns, and that exact mismatch caused an
earlier bug involving an employee named "Miguel Felix Cardozo de Tomy" —
looking up the raw name here would hit the same problem. This makes the
employee registry (`DIM_FUNCIONARIO` / `EmployeeRegistry`) a second
source of truth for an employee's email, specifically so outbound Teams
@mentions still have one when the task source itself won't provide it.

## 22. The task source migrated from Jira to ClickUp only in the extraction layer; `Task` stayed the contract

`clients/jira_client.py` was replaced by `clients/clickup_client.py`, and
`EtlService._build_task`/`transform_details` were rewritten for ClickUp's
shape (a list of assignees, `custom_fields` as a list, millisecond
timestamps, and so on — see [`data-model.md`](data-model.md)). Clockify, the
Excel output, the Postgres output, Teams alerts, employee identity
resolution, and task archiving stayed exactly as they were: none of those
modules knew Jira directly, only `Task` (`models/schemas.py`), and `Task`
did not change.

**Why:** the project switched task-management tools, but the Postgres
schema, the spreadsheet's formulas and tabs, and the Teams alert flows have
no reason to change because of that — they are all consumers of the `Task`
model, not of the source API. Keeping `Task` intact (the single most
important decision of this migration) turned a vendor swap into a change
contained to the extraction layer: only the HTTP client and the dict→`Task`
mapping needed rewriting. This confirms, in practice, the design described
in the [architecture](architecture.md#layers) decision: "no client knows
about `ExcelWriter`, and `ExcelWriter` doesn't know about Jira" — now reads
the same with "Jira" swapped for "ClickUp".

Identifiers that are **data**, not code — the `funcionarios.jira_email`
column in Postgres, the `jira_email` header on the Excel `DIM_FUNCIONARIO`
tab, and the `EmployeeMapping.jira_email` field that mirrors both — were
initially kept under their old name, for the same reason as decision 6:
renaming them would break the runtime match against the schema already
deployed in Supabase and against the real spreadsheet, with no import error
to warn about it. Only the `EmployeeRegistry.get_jira_email` method was
renamed at that point (decision 21), because it is code, not data.

**Later rename of `jira_email` to `clickup_email`:** in a follow-up task, the
three data identifiers above were in fact renamed — the `funcionarios.jira_email`
column and its `check_email_jira` check constraint in Postgres, the
`jira_email` header on the Excel `DIM_FUNCIONARIO` tab, and the
`EmployeeMapping.jira_email` field, all became `clickup_email` /
`check_email_clickup`. The reasoning: unlike the code rename in decision 21
(deferred until the symbol itself stopped making sense), keeping a data
identifier named `jira_email` indefinitely — now that the project no longer
talks to Jira at all — was the kind of name that needlessly confuses the
next person reading the schema, with no real compatibility benefit: the
column and header could be renamed atomically and deliberately (a Postgres
schema migration, a manual header update in Excel via Claude for Excel, and
the matching Python-side field), unlike an uncoordinated silent rename. The
chosen name, `clickup_email`, follows the convention already used by the
sibling `clockify_email` column — naming after the concrete integrated
system rather than a generic term like "task_source_email" — and matches
the pattern of the `CLICKUP_*` identifiers already present in `Settings` and
the `clients/clickup_client.py` module.

**Multiple assignees, and the @mention trade-off:** unlike Jira, ClickUp
allows more than one assignee per task. Each assignee is normalized
individually and the canonical names are joined into `Task.assignee`
(`"Nicolas Delecrode, Daniel Nogueira"`), a call made by the project owner.
But `Task.assignee_email` is still a single email — it feeds one Teams
@mention, and an @mention can't target several people at once — so only the
**first** assignee's email is used. A task with multiple assignees always
notifies only the first one by email; the rest still show up in the report
(the spreadsheet's `responsavel` column and `tarefas` in Postgres) but don't
get a direct @mention.

**The `task_type` trade-off:** ClickUp has no direct equivalent of Jira's
`issuetype` — the only candidate, `custom_item_id`, only exists when the
workspace uses ClickUp's paid Custom Task Types feature, which this
workspace does not. Since `Task.task_type` is a required field and `Task`
couldn't change, every task from ClickUp gets a fixed `TaskType.TASK`, a
call made by the project owner. This means the report loses the
Bug/Story/Epic/Subtask distinction Jira used to provide; if the workspace
ever adopts Custom Task Types, `_build_task` will need revisiting to map
`custom_item_id` instead of using the fixed value.

**The null-priority trade-off:** ClickUp represents "no priority" as
`priority: null` in the payload (instead of Jira's object with an absent
`name`). The mapping (`urgent`→Highest, `high`→High, `normal`→Medium,
`low`→Low) only applies when `priority` isn't null; when it is,
`Task.priority` is left unset and Pydantic validation fails, discarding the
task through the same path that already existed (decision 8) — the same
behavior a priority-less Jira issue always had.

## 23. Synced ClickUp folders are an explicit allowlist, not "every folder in the Space"

`ClickUpClient.fetch_tasks` fetches the whole Space configured in
`CLICKUP_SPACE_ID` (`GET /team/{team_id}/task` with `space_ids[]=...`),
which includes any folder that exists there, of any kind.
`pipeline._filter_allowed_folders` runs right after and drops every task
whose `folder.id` isn't in `CLICKUP_FOLDER_IDS` — a fixed list, configured
in `.env`, of the folder IDs that actually represent a "turma" (today
"Primeiro Ano" and "Segundo Ano") — before `EtlService` ever sees those
tasks.

**Why:** the obvious alternative would be to automatically sync every
folder that exists in the Space, with no fixed list. That was deliberately
rejected: a folder unrelated to a "turma" could be created in the same
Space later — an internal team-planning folder, say, or a temporary
experiment — and nothing about it guarantees its tasks follow the same
contract (`turma`, `area`, priorities) the rest of the pipeline expects.
Without the allowlist, that folder would start feeding the pipeline, Teams
alerts, and reports just by having been created in the Space, without
anyone having made that call on purpose.

This is the same "never change scope silently" philosophy already present
in this project, just pointed the other way: decision 8 (never silently
discard a record) and decision 18 (never silently delete an archived task)
guard against **losing** data without warning; the folder allowlist guards
against **gaining** scope without warning. In both cases, the principle is
that a scope change — in or out — should be a deliberate act, not a side
effect of something that happened in another system (Jira before, ClickUp
now).

**Trade-off:** when a new turma is actually created (say, "Terceiro Ano"),
syncing it requires a manual action: someone has to add the new folder's ID
to `CLICKUP_FOLDER_IDS` and let the scheduled run pick it up. There's no
automatic discovery of new turmas. That friction is deliberate — it's the
price of never including a folder by accident.

After updating, run `mkdocs build --strict` to confirm nothing broke —
pay particular attention to internal anchor links (data-model.md links
to design-decisions.md#22 and #23; design-decisions.md #22 links to
architecture.md#camadas / #layers). Confirm these anchors resolve
correctly given this site's heading-to-anchor slug conventions.

=== Final output ===

Confirm all three pairs of files were updated (replaced for
architecture/data-model, appended for design-decisions), and report the
strict build result including whether the cross-document anchor links
resolved.