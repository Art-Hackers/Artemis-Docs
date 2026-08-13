# Theming & Branding

## How theming works, under the hood

Artemis's theming is built on Material 3's "seed color" system: give Flutter a single color, and it derives an entire coherent palette — surfaces, text colors, hover states, contrast-appropriate variants for both light and dark mode — automatically. This is why `theme="sunset"` alone, with nothing else configured, produces something that looks deliberately designed rather than the flat grey of a default, unstyled Flet app.

```python
app = art.App("My App", theme="ocean")
```

`theme` accepts either a name from the built-in palette (below) or any raw hex color string (`theme="#22D3EE"`).

## Named palettes

56 named palettes ship out of the box, grouped roughly by mood:

| Group | Names |
|---|---|
| Cool | `indigo`, `ocean`, `sky`, `cobalt`, `royal`, `teal`, `cyan`, `slate`, `steel`, `midnight`, `graphite` |
| Green | `forest`, `mint`, `emerald`, `lime`, `olive` |
| Warm | `sunset`, `rose`, `cherry`, `crimson`, `amber`, `gold`, `orange`, `coral`, `clay`, `sand` |
| Bold | `grape`, `violet`, `magenta`, `fuchsia`, `plum` |
| Custom | `silver`, `hackergreen`, `neonpink`, `cyberpunk`, `laser`, `electriclime` |
| Monochrome | `jetblack`, `charcoal`, `snow`, `pearl` |
| Vibrant Accents | `electricblue`, `acidgreen`, `hotmagenta`, `tangerine` |
| Others | `midnightblue`, `matrix`, `vaporwave`, `ultraviolet`, `toxic`, `solaris`, `deepspace`, `supernova`, `arctic`, `amethyst`, `crimsonflare` |

The full mapping (name → hex) lives in `artemis.PALETTES`, a plain dict, if you want to inspect or extend it:

```python
>>> import artemis as art
>>> art.PALETTES["indigo"]
'#6366F1'
```

Every widget that accepts a color parameter (`Text(color=...)`, `Box(bg=...)`, `Badge(color=...)`, and so on) accepts either a palette name or a raw hex string interchangeably — Artemis resolves palette names internally wherever a color is expected.

## Light and dark mode

By default, Artemis follows the operating system's light/dark setting. Force one explicitly with `dark_mode`:

```python
art.App("My App", theme="indigo", dark_mode=True)    # always dark
art.App("My App", theme="indigo", dark_mode=False)   # always light
art.App("My App", theme="indigo", dark_mode=None)    # follow the system (default)
```

## Full manual color control

For anyone who wants an exact look rather than a named vibe, four more keywords give you direct control over specific colors rather than just the seed:

```python
app = art.App(
    "My App",
    theme="indigo",          # still drives anything you don't override below
    background="#0B1220",    # the color behind everything
    surface="#161F32",       # the color of cards/boxes sitting on that background
    text="#E5E7EB",          # the default foreground/text color on that surface
    primary="#818CF8",       # the accent color used for buttons, switches, etc.
)
```

Each one you set becomes a **fixed, absolute color in both light and dark mode** — that's the point of overriding it. Anything you leave as `None` still comes from the seed color, light/dark variation included.

This is not an Artemis-invented trick layered on top of Flet — it's exactly how Flet's own `ColorScheme` overriding works. Flet's `Theme` accepts both a `color_scheme_seed` *and* a partial `ColorScheme` object at the same time; any field you explicitly set on that `ColorScheme` takes precedence over what the seed would have generated, and any field you leave unset still comes from the seed. Artemis's four keywords are just friendlier names for constructing that partial `ColorScheme` (and setting `Theme.scaffold_bgcolor`/`card_bgcolor` for the page-level background) instead of building the object by hand.

Concretely, the mapping is:

| Artemis keyword | What it actually sets |
|---|---|
| `background` | `Theme.scaffold_bgcolor` |
| `surface` | `ColorScheme.surface` + `Theme.card_bgcolor` |
| `text` | `ColorScheme.on_surface` + `ColorScheme.on_surface_variant` |
| `primary` | `ColorScheme.primary` |

## Changing the theme while the app is running

```python
app.set_theme(theme_name=None, dark_mode=<unset>, background=<unset>, surface=<unset>, text=<unset>, primary=<unset>)
```

Every argument is optional and independent — pass only what you want to change:

```python
app.set_theme("forest")                  # just change the palette
app.set_theme(dark_mode=True)             # just force dark mode
app.set_theme(primary="#EF4444")          # just nudge the accent color
```

**A subtlety worth knowing:** arguments you don't pass at all keep their current value — but arguments you pass **explicitly as `None`** clear that override back to pure seed-derived color:

```python
app.set_theme(background=None, surface=None, text=None)  # remove all three overrides
```

This distinction (omitted vs. explicitly `None`) is deliberate, and lets you build a genuine "reset to defaults" button without needing to track the original values yourself.

```python
# a settings-screen theme picker
art.Row([
    art.Button("Ocean", on_click=lambda e: app.set_theme("ocean")),
    art.Button("Forest", on_click=lambda e: app.set_theme("forest")),
])
```

Pair this with `PersistentState` (see [Data & Networking](07-data-and-networking.md#persistentstate)) to remember the user's theme choice across restarts — see `examples/dashboard.py` for a complete working version of exactly this pattern.

## Fonts

```python
art.App("My App", theme="indigo", font="Inter")
```

`font` accepts any Google Fonts name — Flet fetches it for you. Artemis's own default (used if you don't set `font`) is `"Poppins"`.

## Page transitions

```python
app = art.App("My App", transitions="cupertino")
```

Accepts `"fade"`, `"cupertino"`, `"zoom"`, or `"none"`. Applies to every `app.go()`/`app.back()` push and pop, across every platform, in one keyword — rather than configuring Flutter's `PageTransitionsTheme` per-platform yourself.

## Branding your app icon

A brand-new Flet app shows a generic default icon. The first time `app.run()` starts, Artemis fixes this automatically: it drops a default Artemis logo (a crescent-moon mark) into your project's `assets/` folder, creating that folder if it doesn't exist, under three filenames:

- **`logo.png`** — the source image.
- **`icon.png`** — a copy of it, under the exact filename `flet build` looks for when generating the real installed-app icon (the actual Android launcher icon, the Windows/macOS icon, etc.) — so a real build is already correctly branded, not just the dev preview.
- **`logo.ico`** (Windows only) — a proper multi-resolution `.ico`, used for the desktop window/taskbar icon.

None of this ever overwrites a file you've already placed there yourself.

### Using your own logo

Just put your own `logo.png` in the `assets/` folder (next to your script) before running. Artemis detects it's already there and leaves it alone — your version is used automatically, no configuration needed.

If you have Pillow installed (`pip install pillow` or the `[icons]` extra), Artemis also auto-converts your custom `logo.png` into a matching `logo.ico`, so your own branding shows up in the Windows dev-preview window icon too, not just Artemis's default. Without Pillow, Artemis falls back to its own default `.ico` for that one specific case (the window icon on Windows) — or you can supply your own `logo.ico` directly, which Artemis will also leave untouched.

### Custom filename or folder

```python
app = art.App("My App", logo="brand.png")   # looks for assets/brand.png instead
app.run(assets_dir="static")                 # looks in static/ instead of assets/
```

### A genuine platform limitation, stated plainly

While running via `python main.py` (not a real build), the window/taskbar icon can only reliably be overridden on **Windows**, and only via an absolute path to a real `.ico` file — Artemis handles that correctly under the hood. On **macOS**, the dock icon during dev preview is baked into Flet's own pre-built desktop client and cannot be changed at all short of building a custom white-labeled Flet client — this is a Flet limitation, not an Artemis one. Either way, a real `flet build`/`flet pack` gets the icon right on every platform, since that path uses `assets/icon.png`, which behaves correctly everywhere.

## Next

**[Navigation →](05-navigation.md)**
