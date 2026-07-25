# Modelo de dados

> Também publicado como site navegável em
> https://ndelecrodev.github.io/task-time-sync-docs/

## Modelos Python

Definidos em `src/sop_pipeline/models/schemas.py`. São modelos Pydantic: uma
issue do Jira que não satisfaça o contrato é descartada com um `warning`, em vez
de derrubar a execução inteira.

### Identidade de colaboradores

Como o Jira identifica pessoas pelo nome de exibição e o Clockify por e-mail, uma
camada de mapeamento normaliza os dois para um nome canônico usado como chave de
junção.

**Fonte editável:** a aba `DIM_FUNCIONARIO` da planilha é onde alguém corrige ou
adiciona colaboradores manualmente. Antes de cada execução, `EmployeeDataSyncService`
lê essa aba com `ExcelReader` e grava as linhas na tabela `funcionarios` do
Postgres via `PostgresClient.upsert_employee`, casando por `jira_email` ou
`clockify_email`.

**Duplicatas:** a primeira linha a usar um dado `jira_email` ou `clockify_email`
é sincronizada normalmente; qualquer linha seguinte que repita um dos dois é
tratada como duplicata (`EmployeeDataSyncService._split_duplicates`), recebe um
motivo e é gravada na aba `DUPLICADOS_REMOVIDOS` em vez de ser sincronizada.

**Uso em runtime:** `Settings.load_employee_registry` lê a tabela `funcionarios`
já sincronizada e monta um `EmployeeRegistry`, usado pelo `EtlService` para
normalizar `Task.assignee` (vindo do Jira) e `TimeEntry.employee` (vindo do
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

Uma issue do Jira normalizada. Chave de upsert: `task_id` (a *key* do Jira).

| Campo | Tipo | Origem no Jira |
|---|---|---|
| `task_id` | `str` | `key` |
| `title` | `str` | `fields.summary` (default `"No title"`) |
| `assignee` | `str` | `fields.assignee.displayName` |
| `priority` | `Priority` | `fields.priority.name` |
| `status` | `str` | `fields.status.name` |
| `area` | `str \| None` | custom field configurado em `JIRA_CUSTOMFIELD_AREA` |
| `creation_date` | `date` | `fields.created` |
| `due_date` | `date \| None` | `fields.duedate` |
| `completion_date` | `date \| None` | `fields.resolutiondate` |
| `task_type` | `TaskType` | `fields.issuetype.name` |
| `creator` | `str \| None` | `fields.creator.displayName` |
| `update_date` | `date \| None` | `fields.updated` |
| `assignee_email` | `EmailStr \| None` | `fields.assignee.emailAddress` |
| `tags` | `list[str]` | `fields.labels` |

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

Quando o Jira não traz responsável ou área, o ETL usa os textos
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
| `task_id` | `str` | `key` |
| `description` | `str \| None` | `fields.description`, achatado do formato ADF para texto puro |

### Enums

| Enum | Valores |
|---|---|
| `Priority` | `Highest`, `High`, `Medium`, `Low`, `Lowest` |
| `TaskType` | `Bug`, `Task`, `Story`, `Epic`, `Subtask` |
| `DeadlineStatus` | `Concluído`, `Atrasado`, `Atenção`, `No prazo`, `Sem prazo` |

Os valores de `Priority` e `TaskType` são exatamente as strings que a API do Jira
devolve. Os de `DeadlineStatus` são exatamente as strings que a fórmula da coluna
`status_prazo` produz no Excel. **Nenhum desses valores pode ser traduzido** — só
os nomes dos membros do enum.

---

## Mapeamento para a planilha

Convenções do arquivo `.xlsx`: nome de aba em MAIÚSCULAS, nome de tabela em
minúsculas, primeira coluna sempre é o ID usado no upsert.

### Abas escritas pelo Python

| Aba | Tabela | Colunas | Escrita por |
|---|---|---|---|
| `BASE_TAREFAS` | `base_tarefas` | id, titulo, responsavel, area, prioridade, status, data_criacao, prazo, data_conclusao, **dias_restantes**, **atrasado**, **status_prazo**, tipo, criador, data_atualizacao | `save_tasks` |
| `DETALHES_TAREFA` | `detalhes_tarefa` | id, descricao | `save_details` |
| `BASE_HORAS` | `base_horas` | id, funcionario, data, horas | `save_hours` |
| `DIM_ETIQUETAS` | `dim_etiquetas` | id_etiqueta, nome_etiqueta | `save_tags` |
| `FATO_TAREFA_ETIQUETA` | `fato_tarefa_etiqueta` | id_tarefa, id_etiqueta | `save_tags` |

As três colunas em **negrito** são calculadas por fórmula do Excel; o Python
apenas replica o texto da fórmula quando cria uma linha nova.

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
| `tarefas` | Uma linha por issue do Jira; `responsavel_id` é `NULL` quando o colaborador não foi mapeado. `arquivada_em` guarda o timestamp em que a tarefa deixou de aparecer no `JIRA_JQL` (`NULL` enquanto ativa); a linha nunca é apagada. | `upsert_task` (arquivamento: `archive_missing_tasks`) |
| `detalhes_tarefa` | Descrição longa de uma tarefa. | `upsert_task_detail` |
| `horas` | Um apontamento de horas do Clockify; `funcionario_id` é `NULL` quando o colaborador não foi mapeado. | `upsert_time_entry` |
| `etiquetas` | Tags distintas atribuídas a tarefas. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | Associação N:N entre `tarefas` e `etiquetas`. | `upsert_tag_and_link` |
| `areas` | Áreas de atuação dos colaboradores, sincronizadas a partir de `DIM_FUNCIONARIO_AREA`. | `upsert_area_and_link` |
| `funcionario_area` | Associação N:N entre `funcionarios` e `areas`, sincronizada a partir de `FATO_FUNCIONARIO_AREA`; chave composta `funcionario_id` + `area_id`, ambas FK. | `upsert_area_and_link` |
