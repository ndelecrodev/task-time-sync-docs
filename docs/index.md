# Task Time Sync

Este site documenta o sistema completo: um pipeline que sincroniza Jira e
Clockify, e um dashboard que o time usa para acompanhar o resultado.

## As duas partes

**Pipeline** ([repositório](https://github.com/SEU-USUARIO/task-time-sync))

Roda uma vez por dia, agendado via GitHub Actions. Busca tarefas no Jira e
apontamentos de hora no Clockify, grava os dois num arquivo Excel e num banco
Postgres (Supabase), e avisa no Teams quando uma tarefa está perto do prazo
ou atrasada. Também mantém a identidade dos funcionários (nome, e-mail no
Jira, e-mail no Clockify) sincronizada a partir de uma aba editável do
próprio Excel.

**Dashboard** ([repositório](https://github.com/SEU-USUARIO/quimia-dashboard))

Site estático que lê direto do mesmo banco Postgres que o pipeline escreve.
Cada pessoa do time loga com o próprio e-mail e vê tarefas, horas e
indicadores de produtividade, tanto os próprios quanto os do time inteiro.

## Como as duas partes se conectam

```
Jira ─┐
      ├─> Pipeline (Python) ─┬─> Excel (planilha, no Backblaze B2)
Clockify ─┘                  └─> Postgres (Supabase) ─> Dashboard (site)
```

O pipeline é a única coisa que **escreve** no Postgres. O dashboard só
**lê** — nunca insere, atualiza ou apaga nada, e é o Row Level Security do
próprio banco que garante isso, não uma regra do código do site.

## Por onde começar

- Quer entender como o pipeline funciona por dentro: [Arquitetura do
  pipeline](pipeline/architecture.md).
- Quer saber o formato dos dados no Excel e no Postgres: [Modelo de
  dados](pipeline/data-model.md).
- Quer entender por que uma decisão específica foi tomada (e não outra):
  [Decisões de design](pipeline/design-decisions.md).
- Quer saber como o dashboard funciona, autenticação incluída: [Visão geral
  do dashboard](dashboard/overview.md).
