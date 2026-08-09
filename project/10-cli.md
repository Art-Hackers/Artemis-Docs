# The `artemis` CLI

Installing Artemis registers a real console command, `artemis` — a proper entry point (see `[project.scripts]` in `pyproject.toml`), not something you invoke via `python -m`.

## `artemis new`

```bash
artemis new "My Cool App"
```

Scaffolds a starter project in a new folder named after whatever you pass:

```
My Cool App/
├── assets/
├── main.py
├── pyproject.toml
└── .gitignore
```

**`main.py`** — a complete, runnable starter app: an `App` instance, one registered `"/"` page with a title, some body text, and a button wired to a toast, and the `if __name__ == "__main__": app.run()` entry point. Not a placeholder full of `# TODO` comments — genuinely `python main.py` and it runs.

**`pyproject.toml`** — a minimal project file declaring `artemis-ui` as a dependency, so `pip install -e .` inside the new folder pulls in Artemis correctly.

**`.gitignore`** — pre-populated with `__pycache__/`, `*.pyc`, `.artemis_data/` (the folder `PersistentState` writes to — you generally don't want your local dev preferences committed to version control), and `build/`.

**`assets/`** — created empty; populated automatically the first time you run the new app (see [Theming & Branding → Branding your app icon](04-theming-and-branding.md#branding-your-app-icon)).

### After scaffolding

The command prints the exact next steps:

```bash
cd "My Cool App"
pip install -e .
python main.py
```

### If the folder already exists

`artemis new` refuses to run into a non-empty existing folder — it checks first and exits with a clear message (`'X' already exists and isn't empty - pick a different name or clear it out first.`) rather than silently overwriting or merging into whatever's already there.

## What it doesn't do (yet)

There's currently one subcommand. There's no `artemis run` wrapper (just use `python main.py`, or Flet's own `flet run` — see the note on this in [Troubleshooting](11-troubleshooting-and-faq.md)), and no build wrapper (use `flet build` directly, as covered in [Getting Started](01-getting-started.md#building-a-real-installable-app) — Artemis intentionally doesn't reinvent Flet's build tooling, since it already does that job well).

## Next

**[Troubleshooting & FAQ →](11-troubleshooting-and-faq.md)**
