# Artemis Documentation

**Artemis** is a batteries-included layer on top of [Flet](https://flet.dev) for building Android and desktop apps in Python. Flet gives you the primitives (it's Flutter, wearing a Python trench coat); Artemis gives you the everyday app-shell stuff — theming, navigation, forms, state, dialogs — as a handful of plain functions instead of a hundred lines of boilerplate you'd otherwise rewrite in every new project.

This is the documentation for the version of Artemis in this project (`artemis.__version__` — check yours with `python -c "import artemis; print(artemis.__version__)"`). If you were expecting a specific version number and this doesn't match, that's worth double-checking; documentation drifts out of sync with code the moment either one changes without the other, so treat the *installed package* as the source of truth and this doc as a faithful description of it.

Nothing here is aspirational. Every example on every page has actually been run against the real library — most of them through `artemis.testing.TestApp` (see [Testing](09-testing.md)) — before being written down. Where something is flaky or has a real limitation, that's stated plainly rather than glossed over. If you only remember one thing from this whole doc set, let it be this: **every Artemis widget returns a plain Flet control.** There's no parallel universe of Artemis-only concepts to learn. If you already know Flet, you already know 80% of Artemis. If you don't, everything you learn here transfers directly.

## Reading order

If you're brand new, read these roughly in order — each one assumes the last:

1. **[Getting Started](project/01-getting-started.md)** — install it, run your first app, understand the shape of an Artemis project.
2. **[Core Concepts](project/02-core-concepts.md)** — the mental model: `App`, pages, `State`, and *why* the re-render-on-every-click thing works the way it does. Read this one properly; everything else makes more sense once you have this.
3. **[Widget Reference](project/03-widgets.md)** — every widget Artemis ships, what it wraps, every parameter, with runnable examples.
4. **[Theming & Branding](project/04-theming-and-branding.md)** — palettes, full manual color control, your app's icon/logo.
5. **[Navigation](project/05-navigation.md)** — routes, route params, the back stack, bottom tabs, the side drawer, route guards, page transitions.
6. **[Forms & Validation](project/06-forms-and-validation.md)** — `Field`, `Form`, the built-in validators, writing your own.
7. **[Data & Networking](project/07-data-and-networking.md)** — `State` vs `PersistentState` vs `AsyncData`, fetching from an API, async event handlers.
8. **[Dialogs & Feedback](project/08-dialogs-and-feedback.md)** — toasts, alerts, confirmations, date/time pickers, keyboard shortcuts, clipboard.
9. **[Testing](project/09-testing.md)** — `artemis.testing.TestApp`, how to write real tests for your app without opening a window.
10. **[The `artemis` CLI](project/10-cli.md)** — scaffolding a new project.
11. **[Troubleshooting & FAQ](project/11-troubleshooting-and-faq.md)** — the specific, concrete problems people actually hit, and what fixes them. Start here if something's broken.

If you already know what you're looking for, every page stands reasonably well on its own — jump straight in.

## The five-second pitch

```python
import artemis as art

app = art.App("Hello", theme="ocean")

@app.page("/")
def home(page):
    return art.Column([
        art.Title("Hello, Artemis"),
        art.Button("Say hi", on_click=lambda e: app.toast("Hi!")),
    ], center=True)

app.run()
```

Run that on desktop and you get a real native-feeling window (not a browser wrapped in a box) with a proper Material 3 color scheme, correct fonts, and rounded modern controls — all from eleven lines. Run the *same file* through `flet build apk` and it's an Android app, because underneath, it's still Flet the entire way down. Artemis doesn't fork Flet, doesn't wrap it in a webview, and doesn't invent a new rendering engine. It's a thin, honest layer that removes repetition.

## What Artemis is not

Worth saying plainly, since it saves confusion later:

- **Not a replacement for Flet.** You will drop into raw `flet` (available as `art.flet`) regularly — for icons, for anything Artemis hasn't wrapped yet, for fine control. That's expected and fine, not a failure of the abstraction.
- **Not a no-code tool.** You're still writing Python. Artemis just means less of it.
- **Not trying to hide Flet's rough edges by pretending they don't exist.** Where Flet itself has a real limitation (the `FilePicker` saga is the canonical example — see the [Troubleshooting](11-troubleshooting-and-faq.md) page), Artemis says so directly instead of shipping something that quietly breaks on your machine and not the original developer's.

## Where things live

```
your_project/
├── your_app/              (or wherever you keep the artemis/ package)
│   ├── artemis/            the library itself
│   └── ...
├── main.py                 your app's entry point
├── assets/                 auto-created on first run (branding, etc.)
├── .artemis_data/           auto-created if you use PersistentState
├── pyproject.toml
└── tests/                  if you're using artemis.testing
```

Next: **[Getting Started →](project/01-getting-started.md)**
