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
[`design-decisions.md`](design-decisions.md#22-a-fonte-de-tarefas-migrou-de-jira-para-clickup-so-na-camada-de-extracao-o-task-continua-sendo-o-contrato)) para deixar de carregar um nome
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
| `task_type` | `TaskType` | fixo em `TaskType.TASK` — ver [`design-decisions.md`](design-decisions.md#22-a-fonte-de-tarefas-migrou-de-jira-para-clickup-so-na-camada-de-extracao-o-task-continua-sendo-o-contrato) |
| `creator` | `str \| None` | `creator.username` |
| `update_date` | `date \| None` | `date_updated` (timestamp em milissegundos) |
| `assignee_email` | `EmailStr \| None` | `assignees[0].email`, com fallback em `EmployeeRegistry.get_registered_email` |
| `tags` | `list[str]` | `tags[*].name` |
| `turma` | `str` | `folder.name` — lido direto do ClickUp, nunca digitado por alguém; ver [`design-decisions.md`](design-decisions.md#23-as-pastas-do-clickup-sincronizadas-sao-uma-allowlist-explicita-nao-toda-pasta-do-space) |

**Múltiplos responsáveis:** ao contrário do Jira, o ClickUp permite mais de um
assignee por tarefa. Cada um é normalizado individualmente por
`normalize_employee_identifier` (por e-mail quando presente, senão por
`username`) e os nomes canônicos resultantes são concatenados em `assignee`. Só
o e-mail do **primeiro** assignee alimenta `assignee_email`, porque uma
@menção do Teams só pode apontar para uma pessoa — ver
[`design-decisions.md`](design-decisions.md#22-a-fonte-de-tarefas-migrou-de-jira-para-clickup-so-na-camada-de-extracao-o-task-continua-sendo-o-contrato).

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
eles — ver [`design-decisions.md`](design-decisions.md#22-a-fonte-de-tarefas-migrou-de-jira-para-clickup-so-na-camada-de-extracao-o-task-continua-sendo-o-contrato). `TaskType` fica
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
| `tarefas` | Uma linha por tarefa do ClickUp; `responsavel_id` é `NULL` quando o colaborador não foi mapeado. `arquivada_em` guarda o timestamp em que a tarefa deixou de aparecer na busca (`CLICKUP_SPACE_ID` + `CLICKUP_FOLDER_IDS`, `NULL` enquanto ativa); a linha nunca é apagada. `turma` guarda o nome da pasta do ClickUp (ver [`design-decisions.md`](design-decisions.md#23-as-pastas-do-clickup-sincronizadas-sao-uma-allowlist-explicita-nao-toda-pasta-do-space)), lido direto da API, nunca digitado por alguém. | `upsert_task` (arquivamento: `archive_missing_tasks`) |
| `detalhes_tarefa` | Descrição longa de uma tarefa. | `upsert_task_detail` |
| `horas` | Um apontamento de horas do Clockify; `funcionario_id` é `NULL` quando o colaborador não foi mapeado. | `upsert_time_entry` |
| `etiquetas` | Tags distintas atribuídas a tarefas. | `upsert_tag_and_link` |
| `tarefa_etiqueta` | Associação N:N entre `tarefas` e `etiquetas`. | `upsert_tag_and_link` |
| `areas` | Áreas de atuação dos colaboradores, sincronizadas a partir de `DIM_FUNCIONARIO_AREA`. | `upsert_area_and_link` |
| `funcionario_area` | Associação N:N entre `funcionarios` e `areas`, sincronizada a partir de `FATO_FUNCIONARIO_AREA`; chave composta `funcionario_id` + `area_id`, ambas FK. | `upsert_area_and_link` |
