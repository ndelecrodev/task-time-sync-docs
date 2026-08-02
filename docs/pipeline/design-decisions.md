# Decisões de design

> Também publicado como site navegável em
> https://ndelecrodev.github.io/task-time-sync-docs/

Registro das escolhas não óbvias do projeto e do porquê delas.

## 1. O upsert é feito por ID, varrendo a primeira coluna

`ExcelWriter._find_row` percorre a coluna 1 da tabela procurando o ID. Se achar,
sobrescreve a linha; se não, acrescenta uma nova.

**Por quê:** torna a execução **idempotente**. O pipeline roda de forma agendada e
o `JIRA_JQL` normalmente traz de novo issues que já estão na planilha. Sem o
upsert, cada execução duplicaria linhas. Com ele, rodar duas vezes seguidas
produz exatamente o mesmo arquivo.

**Custo:** a busca é linear (O(n) por registro, O(n²) na execução). Para a ordem
de grandeza atual — algumas centenas de linhas — é irrelevante, e mantém o código
sem estado auxiliar. Se a planilha crescer para milhares de linhas, o caminho é
construir um índice `{id: linha}` uma vez por aba, antes do laço.

## 2. As colunas calculadas ficam em fórmula do Excel, não em Python

`dias_restantes`, `atrasado` e `status_prazo` não estão no `TASK_COLUMN_MAP`. O
Python nunca escreve valores nelas — só copia o texto da fórmula quando cria uma
linha nova.

**Por quê:** a planilha é o produto final e é aberta por pessoas em dias em que o
pipeline não rodou. Se esses campos fossem gravados como valores estáticos,
"dias restantes" ficaria congelado na data da última execução e passaria a mentir.
Como fórmula, o Excel recalcula sozinho toda vez que alguém abre o arquivo.

O modelo `Task` **também** expõe `days_remaining` / `is_late` / `deadline_status`,
mas apenas para a regra de alerta e para o texto da notificação, que precisam do
valor no momento da execução.

> Divergência conhecida: a fórmula do Excel usa `MAX(prazo-TODAY(),0)`, então
> nunca mostra número negativo; o `Task.days_remaining` em Python devolve negativo
> para tarefas vencidas. Como o Python não escreve nessa coluna, as duas
> definições nunca se sobrepõem, mas é bom saber ao comparar os dois lados.

## 3. Copiar o texto da fórmula funciona por causa da referência estruturada

`_copy_formula` copia a string da fórmula da linha 2 para a linha nova, sem
reescrever nenhum índice.

**Por quê:** as fórmulas da planilha usam referência estruturada de tabela:

```
=IF(base_tarefas[[#This Row],[data_conclusao]]<>"","Concluído", ...)
```

`[#This Row]` resolve relativo à linha em que a fórmula está. O mesmo texto é,
portanto, correto em qualquer linha — não existe deslocamento a corrigir, ao
contrário do que aconteceria com referências no estilo `J2`, `J3`.

## 4. Etiquetas usam relação N:N

Uma etiqueta virou duas abas: `DIM_ETIQUETAS` (uma linha por etiqueta distinta,
com ID surrogate) e `FATO_TAREFA_ETIQUETA` (uma linha por associação).

**Por quê:** a relação é genuinamente muitos-para-muitos — uma tarefa pode ter
várias etiquetas e uma etiqueta é usada por várias tarefas. As alternativas eram
piores:

- guardar as etiquetas concatenadas em uma célula (`"bug;urgente"`) impediria
  contar tarefas por etiqueta ou montar tabela dinâmica;
- criar uma coluna por etiqueta exigiria alterar o schema da planilha toda vez que
  o time inventasse uma etiqueta nova.

Com a modelagem dimensional, uma tabela dinâmica por etiqueta sai de graça, e
`_get_or_create_tag_id` cria a dimensão sob demanda.

O mesmo raciocínio vale para `FATO_FUNCIONARIO_AREA`: uma pessoa pode atuar em
mais de uma área.

## 5. `fullCalcOnLoad` é ligado em toda abertura do workbook

`_open_workbook` sempre define `workbook.calculation.fullCalcOnLoad = True`.

**Por quê:** o openpyxl lê o *valor em cache* das fórmulas e o regrava ao salvar.
Sem essa flag, uma linha nova recém-inserida ficaria exibindo cache vazio ou
desatualizado até alguém editar a célula. Com a flag, o Excel recalcula o arquivo
inteiro na próxima abertura.

## 6. Os identificadores da planilha ficam em português

Nomes de aba (`BASE_TAREFAS`), nomes de tabela (`base_tarefas`), cabeçalhos de
coluna (`data_criacao`) e os valores de `DeadlineStatus` (`"Concluído"`) seguem em
português, mesmo com o código todo em inglês.

**Por quê:** não são nomes de código, são **dados**. São procurados em tempo de
execução dentro do arquivo `.xlsx` (`workbook["BASE_TAREFAS"]`,
`worksheet.tables["base_tarefas"]`, `column_map["data_criacao"]`) e comparados com
o que as fórmulas do Excel produzem. Traduzi-los quebraria o pipeline em runtime,
sem erro de importação para avisar.

Por isso o `TASK_COLUMN_MAP` existe: ele é a fronteira explícita entre o mundo dos
dados (esquerda, português) e o mundo do código (direita, inglês).

## 7. O heartbeat fica fora do tratamento de erro

Ver [`architecture.md`](architecture.md#observabilidade): o heartbeat só deve
disparar quando a planilha efetivamente chegou ao bucket. Ele é a última linha de
`run()` e não está protegido por `try/except` justamente para que uma falha
anterior o impeça de rodar.

## 8. Registros inválidos são descartados, não interrompem a execução

`transform_tasks`, `transform_details` e `transform_time_entries` capturam
`ValidationError` e `KeyError` **por registro**, logam um `warning` e seguem.

**Por quê:** uma única issue com campo faltando não pode custar a sincronização
inteira do dia. O warning fica no Better Stack para investigação posterior.

**Trade-off:** um erro sistemático (por exemplo, o `JIRA_CUSTOMFIELD_AREA`
apontando para um campo que não existe mais) se manifestaria como "todas as
tarefas foram puladas", com a execução terminando em sucesso aparente. Por isso o
descarte é logado em **ERROR** e mandado pro Sentry, e o log de fim de sync traz a
contagem: `"Jira: N issues fetched, M tasks written, K discarded"` — um `K` acima
de zero é sinal de problema estrutural, tipicamente uma prioridade ou tipo novo no
Jira que não existe nos enums de `models/schemas.py`.

**Cuidado ao mexer nesse `except`:** ele lista explicitamente `AttributeError` e
`TypeError` (a tupla `RECORD_ERRORS`) além de `ValidationError`/`KeyError`. Sem
eles, um campo `null` inesperado do Jira estoura `AttributeError`, escapa do laço e
derruba o lote inteiro — o oposto do isolamento por registro que essa decisão
promete. Foi exatamente esse o bug: `priority: null` numa única issue fazia todas
as outras se perderem.

## 9. As exceções customizadas cobrem só as bordas de I/O

`errors/exceptions.py` define `ExcelWriteError`, `NotificationError` e
`StorageError`, levantadas em `integrations/`. `TaskValidationError` está definida
mas o ETL **continua** usando `except (ValidationError, KeyError): continue`.

**Por quê:** trocar o `continue` por um `raise` mudaria comportamento — passaria a
abortar a transformação no primeiro registro ruim, que é justamente o oposto da
decisão 8. As exceções entram só onde já havia propagação de erro genérica até o
`try/except` do `run()`, sempre com `raise ... from error` para preservar a causa
original.

## 10. Os pacotes não reexportam nada nos `__init__.py`

Cada `__init__.py` tem só a docstring; os módulos são importados pelo caminho
completo.

**Por quê:** `config/settings.py` instancia `Settings()` no import, o que lê e
valida o `.env`. Com reexportação em `integrations/__init__.py`, importar o
`ExcelWriter` puxaria o `Notifier`, que puxaria as settings — e passaria a exigir
um `.env` completo só para escrever numa planilha local. Sem reexportação, o
`ExcelWriter` é testável de forma isolada.

## 11. A validação de status no Excel é uma cópia manual do workflow do Jira

O dropdown de validação em `BASE_TAREFAS.status` usa a lista `Backlog, To Do,
In Progress, Code Review, Testing, Done`, copiada do workflow configurado no
Jira em 22/07/2026.

**Por quê:** diferente de `tipo` (`TaskType`, um enum validado em
`models/schemas.py`), `status` é texto livre vindo direto do Jira — não existe
enum no lado Python para essa coluna. Uma lista fixa no código correria o
mesmo risco que já se materializou com `tipo` (a issue `QT-6` com
`task_type="Function"`, fora do enum, foi descartada silenciosamente do
relatório). Por isso a lista de status não vive no código: vive só na
validação de dropdown do Excel, e precisa ser copiada manualmente do Jira.

**Trade-off:** essa validação protege apenas edição manual da planilha. O
pipeline sobrescreve `status` a cada execução com o valor vindo direto do
Jira, sem passar pela validação do Excel — então um status novo no Jira
aparece no relatório mesmo sem estar na lista do dropdown, mas editar a célula
manualmente com um valor fora da lista é bloqueado.

**Cuidado ao mexer nisso:** se o workflow do Jira mudar (status renomeado,
adicionado ou removido), essa lista precisa ser atualizada manualmente no
Excel. Não há sincronização automática entre as duas.

## 12. Identidade de colaboradores saiu de EMPLOYEES_JSON para uma tabela no Postgres, com o Excel como front-end editável

Antes, o mapeamento de colaboradores vivia em uma variável de ambiente
(`EMPLOYEES_JSON`), um blob JSON lido na inicialização de `Settings`. Hoje
`Settings.load_employee_registry` lê a tabela `funcionarios` do Postgres, e
`EmployeeDataSyncService` é quem mantém essa tabela, sincronizando-a a partir da
aba `DIM_FUNCIONARIO` da planilha antes de cada execução.

**Por quê:** um blob JSON em variável de ambiente só podia ser editado por
quem tinha acesso ao `.env` e sabia a sintaxe correta; um erro de digitação
quebrava a leitura do cadastro inteiro de colaboradores. Mover a fonte de
verdade para uma tabela relacional, editável através de uma planilha que a
equipe já usa no dia a dia (`DIM_FUNCIONARIO`), tira a barreira técnica de
manter o cadastro atualizado. O Postgres também deixa esse cadastro
disponível para outros consumidores, como o dashboard.

**Trade-off:** a inicialização do pipeline agora depende de o banco estar
acessível. `load_employee_registry` propaga `SQLAlchemyError` quando a
conexão falha, algo que o blob JSON local nunca precisou considerar. A
sincronização de colaboradores também virou um passo a mais no início de
cada execução, e precisa terminar antes de qualquer normalização de nomes.

## 13. SQLAlchemy foi escolhido como ORM, não Core ou psycopg cru

`PostgresClient` usa classes declarativas do SQLAlchemy (`Funcionarios`,
`Tarefas`, etc.) e objetos `Session`, em vez de escrever SQL diretamente com
`psycopg` ou usar a camada `Core` do próprio SQLAlchemy.

**Por quê:** parte da motivação é o objetivo de aprendizado do projeto. Este
projeto também é um espaço para praticar ORM em um caso real, com upserts,
relações e constraints. Como efeito colateral, o ORM também tira SQL do
resto do código: os upserts em `PostgresClient` viram atribuição de atributo
Python (`existing.canonical_name = ...`) em vez de `UPDATE ... SET`
montado à mão.

**Trade-off:** cada upsert abre sua própria `Session` e faz um `SELECT`
antes do `INSERT`/`UPDATE` (ver `upsert_employee`, `upsert_task`, etc.), o
que é menos eficiente que um `INSERT ... ON CONFLICT` nativo do Postgres.
Para o volume atual do pipeline (algumas centenas de linhas por execução)
isso não é um problema; vale revisitar se o volume crescer.

## 14. `person.area` no dashboard é derivado das áreas das tarefas, não de uma tabela de área por colaborador no Postgres

O Excel tem `FATO_FUNCIONARIO_AREA`, uma relação N:N dedicada entre
colaborador e área (ver decisão 4). O schema do Postgres não tem
equivalente: não existe uma tabela `funcionario_area`. Quando o dashboard
precisa da área de uma pessoa, deriva esse valor a partir das áreas das
tarefas atribuídas a ela em `tarefas.area`.

**Por quê:** `FATO_FUNCIONARIO_AREA` nasceu no Excel para o dashboard
baseado em planilha; replicar essa tabela ao migrar os indicadores para o
Postgres significaria manter mais um cadastro sincronizado, sem que exista
hoje uma necessidade que a área das tarefas não cubra. Na prática, quem
trabalha majoritariamente em tarefas de uma área também pertence a ela.

**Trade-off:** um colaborador sem tarefas atribuídas em um período não tem
área derivável no Postgres, diferente do Excel, onde a área da pessoa é
cadastrada explicitamente. Se essa lacuna passar a importar, o caminho é
adicionar uma tabela `funcionario_area` no Postgres espelhando
`FATO_FUNCIONARIO_AREA`.

**Atualização:** a tabela `funcionario_area` foi criada no Postgres (ver
decisão 17), mas para sincronizar `FATO_FUNCIONARIO_AREA` via
`EmployeeDataSyncService.sync_areas`, não para alimentar o `person.area` do
dashboard descrito acima. A lacuna do trade-off ("colaborador sem tarefa não
tem área derivável") segue existindo enquanto o dashboard não passar a
consultar essa tabela.

## 15. O alerta mostra "Tarefa atrasada há N dia(s)" em vez do número negativo de `days_remaining`

`Task.days_remaining` é negativo por design para tarefas vencidas (decisão
2). `Notifier._build_message`, porém, nunca expõe esse negativo no texto:
quando `days_remaining < 0`, troca o rótulo para "Tarefa atrasada há" e
mostra `abs(days_remaining)`.

**Por quê:** "-3 dias restantes" exige que quem lê faça a conta mental
(negativo significa atrasado, e por quantos dias). "Tarefa atrasada há 3
dia(s)" comunica a mesma informação sem esse passo extra, para um texto lido
rapidamente dentro de uma notificação do Teams.

**Cuidado ao mexer nisso:** a inversão de sinal (`abs()`) e a troca de
rótulo precisam andar juntas. Mudar uma sem a outra produz uma mensagem como
"Dias restantes: -3 dia(s)" ou "Tarefa atrasada há -3 dia(s)", ambas
incoerentes com o resto do texto.

## 16. `EmployeeSyncService` virou `EmployeeDataSyncService`, com um `read_sheet_as_dicts` genérico em vez de três leituras quase iguais

A sincronização de colaboradores ganhou uma segunda responsabilidade: além de
`DIM_FUNCIONARIO`, `EmployeeDataSyncService.sync_areas` agora lê também
`DIM_FUNCIONARIO_AREA` e `FATO_FUNCIONARIO_AREA`. Para isso, `ExcelReader`
ganhou um método genérico, `read_sheet_as_dicts(file_path, sheet_name,
table_name)`, que abre o workbook, acha a tabela e monta a lista de dicts por
linha. `read_employees`, `read_dim_employee_area` e `read_fato_employee_area`
chamam esse helper, cada um só fixando o nome da aba e da tabela que lê.

**Por quê:** as três leituras faziam a mesma sequência de passos (abrir
workbook, achar tabela, mapear colunas, iterar linhas), variando só a aba e a
tabela. Manter três cópias dessa lógica criaria três lugares para corrigir o
mesmo bug se o formato de leitura mudasse. O nome do serviço também mudou de
`EmployeeSyncService` para `EmployeeDataSyncService` porque a classe deixou de
sincronizar só identidade; manter o nome antigo passaria a ser enganoso.

**Trade-off:** nenhum digno de nota. `read_sheet_as_dicts` cobre exatamente o
mesmo contrato que `read_employees` já tinha antes da extração; é uma
reorganização de método, não uma mudança de comportamento.

## 17. Vínculo colaborador-área usa duas tabelas no Postgres (`areas` + `funcionario_area`), não uma coluna de texto

`upsert_area_and_link` resolve ou cria uma linha em `Area` (tabela `areas`) e
depois resolve ou cria a linha de associação em `FuncionarioArea` (tabela
`funcionario_area`, chave composta `funcionario_id` + `area_id`), em vez de
gravar o nome da área direto numa coluna de `Funcionarios`.

**Por quê:** mesmo raciocínio da decisão 4 para `etiquetas`/`tarefa_etiqueta`:
um colaborador pode atuar em mais de uma área, então a relação é N:N, não N:1.
Uma coluna de texto em `funcionarios` só comportaria uma área por colaborador
e exigiria duplicar o nome da área em cada linha, sem uma dimensão para
agrupar por área depois.

## 18. Tarefas que somem do `JIRA_JQL` são arquivadas por timestamp, não apagadas

`PostgresClient.archive_missing_tasks` roda ao fim de `sync_jira` e marca
`tarefas.arquivada_em = now()` em toda linha cujo `task_id` não apareceu na
busca da execução atual e que ainda não tinha sido arquivada; a linha nunca é
apagada.

**Por quê:** segue a mesma filosofia da decisão 8, nunca descartar
silenciosamente. Uma falha transitória do Jira ou uma `JIRA_JQL` mal
configurada pode fazer a lista de issues devolvida vir vazia ou incompleta;
sem o arquivamento por timestamp, um `DELETE` nesse momento apagaria tarefas
que continuam existindo no Jira, e o próximo `sync_jira` bem-sucedido não
teria como recuperar o que foi perdido. Marcar com timestamp em vez de
apagar deixa o problema visível e reversível.

**Cuidado ao mexer nisso:** o conjunto usado para decidir o que arquivar precisa ser all_ids_from_jira (toda issue key devolvida pela busca crua ao Jira, antes de qualquer validação), não valid_ids (as tarefas que já passaram pela validação do Pydantic, decisão 8). Usar valid_ids faria uma issue descartada por validação (priority/tipo fora do enum, por exemplo) ser arquivada como se tivesse desaparecido do Jira, mesmo continuando ativa lá — confundindo "não veio nessa busca" com "veio, mas falhou na validação". Essa foi, de fato, uma versão inicial com esse bug, corrigida antes de entrar em produção.

## 19. O arquivamento também é marcado na planilha, numa coluna que só recebe essa escrita

`BASE_TAREFAS` ganhou a coluna `arquivada_em`, espelhando a coluna de mesmo
nome em `tarefas` (decisão 18). `ExcelWriter.mark_archived_tasks` roda ao
fim de `sync_jira`, depois de `archive_missing_tasks`, e busca as tarefas já
arquivadas com `PostgresClient.get_archived_tasks` para escrever a data de
arquivamento na planilha.

**Por quê:** antes dessa mudança, uma tarefa arquivada ficava marcada só no
Postgres; quem abrisse a planilha não tinha como saber que uma linha de
`BASE_TAREFAS` correspondia a uma tarefa que já sumiu do Jira, a não ser
consultando o banco diretamente. Escrever a mesma marca na planilha deixa
essa informação visível para quem só usa o Excel.

**Trade-off:** `mark_archived_tasks` escreve exclusivamente a célula de
`arquivada_em`; não chama `_write_task_row` nem qualquer outro caminho que
regrave `titulo`, `status`, `prazo` ou qualquer outro campo da linha. Isso é
proposital: uma tarefa arquivada não recebe mais atualizações do Jira, então
seus outros campos devem continuar congelados no último valor real que
tinham antes de a tarefa desaparecer da busca, não serem sobrescritos ou
zerados. Se um `task_id` vindo do Postgres não tiver linha correspondente em
`BASE_TAREFAS` (não deveria acontecer, já que a tarefa foi escrita lá antes
de ser arquivada), o método pula essa tarefa em vez de lançar erro.

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

## 21. `EtlService._build_task` recorre a `EmployeeRegistry.get_jira_email` quando o Jira não informa o e-mail do assignee

Quando `fields.assignee.emailAddress` vem nulo mas há um assignee de fato
(`fields.assignee` não é `None`), `_build_task` primeiro normaliza o
`displayName` do Jira para o nome canônico via
`normalize_employee_identifier`, e só então chama
`EmployeeRegistry.get_jira_email` com esse nome já canonicalizado, nunca
com o `displayName` bruto. O fallback não roda no branch de tarefa sem
responsável (`fields.assignee is None`), onde `assignee` é o sentinel
`NO_RESPONSIBLE` — chamar `get_jira_email` com um sentinel como se fosse
nome de pessoa não faz sentido e nunca deve acontecer.

**Por quê:** as configurações de privacidade de visibilidade de e-mail por
usuário do Jira Cloud (uma mudança da era GDPR) podem deixar
`emailAddress` nulo mesmo para um assignee corretamente definido e
visível em qualquer outro lugar do Jira — isso é comportamento esperado
do Jira, não um dado faltando por falha deste projeto. Usar o nome já
canonicalizado, e não o `displayName` bruto do Jira, é essencial: os
nomes cadastrados na `DIM_FUNCIONARIO` podem divergir do `displayName`
que o Jira retorna, e foi exatamente essa divergência que causou um bug
anterior envolvendo um colaborador chamado "Miguel Felix Cardozo de
Tomy" — buscar pelo nome bruto teria o mesmo problema aqui. Com isso, o
cadastro de colaboradores (`DIM_FUNCIONARIO` / `EmployeeRegistry`) passa
a ser a segunda fonte de verdade para o e-mail de um colaborador,
especificamente para permitir @menções no Teams quando o próprio Jira
não fornece um e-mail.
