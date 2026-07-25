# Dashboard

Site estático (HTML, CSS e JavaScript puro, sem framework nem build step)
que mostra tarefas e horas do time, lendo direto do Postgres que o pipeline
mantém. Hospedado no Cloudflare Pages.

## Stack

- **Autenticação:** Supabase Auth, e-mail e senha.
- **Dado:** SDK JavaScript do Supabase (`@supabase/supabase-js`), consultando
  as mesmas tabelas que o pipeline escreve.
- **Gráficos:** Chart.js.
- **Exportação:** CSV (separador `;`, com BOM, para abrir certo no Excel em
  português) e `.xlsx` real via SheetJS.

Não existe backend próprio. Toda a segurança de acesso é responsabilidade do
Postgres, via Row Level Security — o site nunca decide sozinho o que mostrar
ou esconder, ele só pede os dados e o banco filtra.

## Autenticação e controle de acesso

Qualquer pessoa pode criar uma conta (e-mail e senha), mas só quem já está
cadastrado na tabela `funcionarios` (pelo `jira_email` ou `clockify_email`)
consegue ver algum dado depois de logar. Isso é garantido pelas políticas de
RLS de cada tabela, não por uma checagem no código do site.

Se alguém tentar se cadastrar com um e-mail que não está em `funcionarios`,
a tela avisa antes de enviar, mas não bloqueia — a pessoa consegue criar a
conta mesmo assim. O que ela vê depois de logar é sempre vazio, porque
nenhuma política de RLS libera leitura para um e-mail não cadastrado.

Toda tentativa de cadastro com e-mail não autorizado é registrada num log
(`unauthorized_signup_attempts`), via um gatilho no próprio banco — funciona
mesmo se alguém pular a tela de aviso e chamar a API do Supabase Auth
diretamente. O log guarda o e-mail tentado e o horário; não guarda o
endereço IP, porque um gatilho de Postgres não tem acesso a essa informação.

## O que a pessoa vê depois de logar

- **Visão por colaborador:** tarefas, horas, status e prioridade de uma
  pessoa específica, incluindo foto de perfil quando cadastrada (via
  `funcionarios.photo_url`, um link do Supabase Storage).
- **Visão do projeto:** os mesmos indicadores, agregados para o time
  inteiro.
- **Detalhamento de tarefas:** tabela completa, com exportação para CSV ou
  Excel.

## De onde vêm os dados que o dashboard mostra

| O que aparece no dashboard | De onde vem |
|---|---|
| Nome, e-mail, foto | `funcionarios` |
| Tarefas, status, prioridade, prazo | `tarefas` |
| Horas apontadas | `horas` |
| Etiquetas de cada tarefa | `etiquetas` + `tarefa_etiqueta` |
| Área de cada pessoa | Derivada das áreas das tarefas dela (não existe uma tabela dedicada de funcionário-área no Postgres ainda, só no Excel) |

## Limitações conhecidas

- A área mostrada por pessoa é uma aproximação (ver tabela acima), não um
  dado cadastrado diretamente.
- Sem tabela `funcionario_area` no Postgres, alguém sem tarefa nenhuma
  aparece sem área definida.
