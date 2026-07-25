# Task Time Sync — Docs

MkDocs Material site documenting the [task-time-sync](https://github.com/ndelecrodev/task-time-sync)
pipeline and its dashboard: how they work, the data model they share, and
the reasoning behind the non-obvious design choices.

Available in Portuguese (default) and English, via
[mkdocs-static-i18n](https://ultrabug.github.io/mkdocs-static-i18n/).

## Running locally

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open http://127.0.0.1:8000/. Use the language switcher in the header
to toggle between pt and en.

To produce a static build:

```bash
mkdocs build --strict
```

`--strict` fails the build on broken internal links or nav references
pointing at missing files, so it's the right check to run before
publishing.

## Structure

```
docs/
├── index.md / index.en.md
├── assets/                    # favicon, logo
├── dashboard/
│   └── overview.md / overview.en.md
└── pipeline/
    ├── architecture.md / architecture.en.md
    ├── data-model.md / data-model.en.md
    └── design-decisions.md / design-decisions.en.md
```

Every page has a Portuguese source file and an `.en.md` sibling with the
English translation. Sheet names, table names, column names, and other
data-layer literals (see `pipeline/design-decisions.md`, decision #6)
stay untranslated in both languages, since they're read directly at
runtime, not just documentation labels.

Adding a page only requires listing the Portuguese (unsuffixed) file in
`mkdocs.yml`'s `nav:`; the i18n plugin resolves the `.en.md` sibling on
its own.

## License

MIT, see [LICENSE](LICENSE).
