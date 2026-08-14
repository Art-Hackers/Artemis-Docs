# Troubleshooting & FAQ

This page exists because every problem on it is one somebody actually hit. Nothing here is hypothetical.

---

## "ModuleNotFoundError: No module named 'artemis'"

This means Python can't see the package from wherever you ran the script — almost always one of two causes:

1. **You haven't installed it.** From the project root (the folder containing `pyproject.toml`):
   ```bash
   pip install -e .
   ```
2. **You're running the script from the wrong working directory.** Run it from the project root, not from inside the `examples/` folder itself:
   ```bash
   cd artemis_project     # the folder containing both artemis/ and examples/
   python examples/counter.py
   ```
   A script can't import a package that's a *sibling* of its own folder — only one that Python can find on its import path, which `pip install -e .` arranges for you regardless of your current directory, as long as the install itself succeeded.

If you've done both and it's still not found, run `pip show artemis-ui` to confirm it's actually installed in the Python environment you're running the script with — it's easy to `pip install` into one environment (say, a global install) and then run the script with a different `python` (say, inside a virtual environment), especially on Windows with multiple Python installations.

---

## `AttributeError: 'App' object has no attribute '...'` after updating

This happens when a project folder has a **mix of old and new files** — usually because a new zip was extracted directly on top of an old, already-extracted folder. Extracting a zip only adds and overwrites files; it never deletes ones that were removed or renamed in the new version. If an old example file still references a method that a newer version of the library removed (or renamed), you'll see exactly this error.

**Fix:** delete the whole project folder and extract a fresh zip into an empty location. Don't extract on top of an existing folder when updating.

---

## Why is there no file picker?

Short answer: Artemis shipped one, and pulled it after extensive real-world testing showed it couldn't be made reliable.

Longer answer, for anyone curious or hitting a related issue with `flet.FilePicker` directly: Flet's `FilePicker` control has a real, still-open upstream bug where the client-side listener for that control sometimes never registers in time, producing either `Unknown control: FilePicker` or a `TimeoutException` after Flet's default 10-second wait. This isn't specific to any one Flet version — it's been reported across several recent releases (see `flet-dev/flet` issues `#6040`, `#6251`, `#6422` if you want to read the upstream discussion yourself), and it's noticeably worse on **Python 3.14** specifically (see the [next section](#python-314)).

Several mitigations were tried, in order, against this exact problem, before the feature was removed:

1. Registering the picker on first use, calling `page.update()` immediately after — the officially-recommended pattern. Didn't reliably work.
2. Adding a short delay after registration before the first real request. Helped sometimes, not reliably.
3. Registering the picker eagerly at app startup instead of lazily on first click, to maximize the time available for the client to catch up. Still failed for some users even with a multi-second head start.
4. A retry with a longer backoff, plus catching the failure gracefully so at least the app wouldn't crash even if the picker itself never opened.

Even with all of that, it failed for real users on real machines. Rather than ship a feature that silently works for some people and silently breaks for others depending on their exact Python/Flet/OS combination, it was removed entirely rather than left in a "mostly works" state.

**If you need file access:** the raw Flet control is still there — `art.flet.FilePicker` — if you want to experiment with it yourself, following the same registration pattern described above. Just know you're working against the same open upstream bug; it isn't an Artemis limitation you're routing around, it's a Flet one.

**What's *not* affected:** `app.pick_date`/`app.pick_time` (date/time pickers) and `app.alert`/`app.confirm`/`app.toast` all go through a completely different Flet mechanism (`page.show_dialog(...)`) that doesn't have this problem — they're unaffected and reliable.

---

## Python 3.14

If you're running Python 3.14 and hitting control-registration errors (the `Unknown control` / `TimeoutException` pattern described above, or anything else that looks like a control "isn't known to the client"), this is a real, independently-reported compatibility gap between Flet's internals and Python 3.14's changes — not something specific to Artemis. Python 3.14 is very new; Flet's async/threading internals don't appear to be fully caught up with it yet as of this writing.

**Workaround:** run under Python 3.12 or 3.13 instead. You don't need to uninstall 3.14 — install 3.12/3.13 alongside it and create a virtual environment with the older version:

```bash
py -3.12 -m venv venv          # Windows, using the py launcher
venv\Scripts\activate
pip install -e .
```

```bash
python3.12 -m venv venv         # macOS/Linux
source venv/bin/activate
pip install -e .
```

This isn't a permanent state of affairs — it's a snapshot of where Flet's 3.14 support happens to be right now, presumably to be resolved upstream over time.

---

## The window/taskbar icon doesn't change

Covered in detail in [Theming & Branding](04-theming-and-branding.md#a-genuine-platform-limitation-stated-plainly). Short version: this only reliably works on **Windows**, and requires a real `.ico` file at an absolute path — which Artemis handles correctly for you automatically. On **macOS**, the dev-preview dock icon is baked into Flet's pre-built client and can't be changed at all without building a custom client — not fixable from the Python side by Artemis or anyone else. A real `flet build` gets every platform's icon right regardless, since that uses a completely different (and much more reliable) mechanism.

---

## Charts raise a `RuntimeError` about `flet-charts`

Expected behavior, not a bug. `art.LineChart`/`BarChart`/`PieChart` need the optional `flet-charts` package, which Artemis doesn't force onto every project (most apps don't need charts):

```bash
pip install flet-charts
```

or

```bash
pip install -e ".[charts]"
```

The error message itself tells you this — if you're seeing a plain `ImportError` instead of a message suggesting `pip install flet-charts`, something else is wrong; please check you're on a current version of Artemis, since earlier iterations of the chart wrappers may not have had this graceful handling.

---

## Why does clicking a button redraw the *entire* screen?

This is deliberate, not a performance oversight — covered in full in [Core Concepts](02-core-concepts.md#the-re-render-model--the-single-most-important-thing-to-understand). The short version: it trades a small amount of raw rendering efficiency for a much simpler mental model, where your page function is always a fresh, correct description of "what does this screen look like right now" — you never have to manually keep the UI in sync with your data.

For typical app sizes (dozens to low hundreds of controls on screen), this is imperceptible. If you're rendering something genuinely large — thousands of list rows, for instance — you may start to feel it; scoped/partial updates aren't currently part of Artemis's architecture. If you hit this in practice, consider paginating or virtualizing that specific list (only rendering what's actually visible) rather than expecting the framework to solve it transparently.

---

## Is Artemis production-ready?

Depends what you're building, and this deserves an honest answer rather than a marketing one. Artemis is a genuinely large, thoroughly-tested feature set — theming, navigation, forms, state, charts, testing — built and verified incrementally, with real tests (`tests/test_examples.py`, run with `pytest`) and real manual verification against the actual installed `flet` package throughout its development, not written from memory or assumption.

That said: it's young, maintained by (at the time of writing) essentially one continuous development effort rather than a broad community, and it inherits every limitation of the Flet version it's built on — including the genuinely unresolved bugs described elsewhere on this page. For a side project, an internal tool, an MVP, or learning Flet with better defaults, it's solid. For something with a large existing team, long-term multi-year support expectations, or requirements that push against the documented "known rough edges" in the main `README.md`, evaluate carefully and read that section specifically before committing.

---

## Where to look next if your problem isn't here

- The main `README.md` has a "Known rough edges" section covering limitations not specific enough to warrant their own entry on this page.
- Every source file in `artemis/` has a module-level docstring explaining *why* it's built the way it is, not just *what* it does — worth reading directly if you're debugging something unusual.
- `examples/` has a working, runnable app for nearly every feature described across this documentation — if something isn't behaving the way you expect, compare against the matching example first.
- Lastly, you can open an issue in this repository's [Issues section](https://github.com/Art-Hackers/Artemis-Docs/issues)
