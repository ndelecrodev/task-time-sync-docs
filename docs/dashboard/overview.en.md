# Dashboard

A static site (plain HTML, CSS, and JavaScript, no framework, no build
step) that shows the team's tasks and hours, reading straight from the
Postgres database the pipeline maintains. Hosted on Cloudflare Pages.

## Stack

- **Authentication:** Supabase Auth, email and password.
- **Data:** Supabase's JavaScript SDK (`@supabase/supabase-js`), querying
  the same tables the pipeline writes to.
- **Charts:** Chart.js.
- **Export:** CSV (`;` separator, with BOM, so it opens correctly in
  Portuguese Excel) and real `.xlsx` via SheetJS.

There's no backend of its own. All access security is Postgres's
responsibility, via Row Level Security, the site never decides on its own
what to show or hide, it just requests the data and the database filters
it.

## Authentication and access control

Anyone can create an account (email and password), but only someone
already registered in the `funcionarios` table (by `jira_email` or
`clockify_email`) can see any data after logging in. This is enforced by
each table's RLS policies, not by a check in the site's code.

If someone tries to sign up with an email that isn't in `funcionarios`,
the screen warns them before submitting, but doesn't block it, the person
can still create the account. What they see after logging in is always
empty, because no RLS policy grants read access to an unregistered email.

Every signup attempt with an unauthorized email is logged
(`unauthorized_signup_attempts`), via a trigger in the database itself,
which works even if someone skips the warning screen and calls the
Supabase Auth API directly. The log stores the attempted email and the
timestamp; it doesn't store the IP address, because a Postgres trigger has
no access to that information.

## What a person sees after logging in

- **Per-employee view:** tasks, hours, status, and priority for a specific
  person, including a profile photo when one is registered (via
  `funcionarios.photo_url`, a Supabase Storage link).
- **Project view:** the same indicators, aggregated for the whole team.
- **Task breakdown:** a full table, with export to CSV or Excel.

## Where the data shown on the dashboard comes from

| What shows up on the dashboard | Where it comes from |
|---|---|
| Name, email, photo | `funcionarios` |
| Tasks, status, priority, deadline | `tarefas` |
| Logged hours | `horas` |
| Tags on each task | `etiquetas` + `tarefa_etiqueta` |
| Each person's area | Derived from the areas of their tasks (there's no dedicated employee-area table in Postgres yet, only in Excel) |

## Known limitations

- The area shown per person is an approximation (see table above), not
  data registered directly.
- Without a `funcionario_area` table in Postgres, someone with no tasks at
  all shows up with no area defined.
