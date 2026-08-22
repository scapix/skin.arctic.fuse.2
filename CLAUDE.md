# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Arctic Fuse 2 (`skin.arctic.fuse.2`) — a Kodi skin by jurialmunkey. It is **pure declarative XML**: there is no Python, no build system, no test suite, and no lint step. "Building" the skin means Kodi parsing the XML at runtime; the only compiled artifact (`media/Textures.xbt`) is committed as a binary.

Note `Readme.md`: this skin is deprecated in favour of [Arctic Fuse 3](https://github.com/jurialmunkey/skin.arctic.fuse.3). Work here is maintenance on the Omega branch.

## Working on the skin

There is no local command that validates a change end-to-end. The practical loop is:

- Well-formedness of any XML you touch: `python3 -c "import xml.etree.ElementTree as E; E.parse('1080i/Home.xml')"` (or `xmllint --noout`). Kodi silently drops a malformed include file, so always check this before committing.
- Real verification requires Kodi with the skin symlinked into `addons/` and `ReloadSkin()` (skin debug options live under Settings → Skin, and `Custom_1194_Debug_Grid.xml` / `Custom_1199_Debug_Overlay.xml` are the in-skin debug windows).
- Version bumps go in `addon.xml` (`version` attribute). Pushing a tag fires `.github/workflows/dispatch_to_repo.yml`, which notifies `jurialmunkey/repository.jurialmunkey` to publish the release. That is the entire CI.
- `.gitattributes` sets `merge=ours` for `addon.xml` and `changelog.txt` — expect merges from upstream not to touch them.

Commit subjects use gitmoji shortcodes: `:zap:` (change/tidy), `:bug:` (fix), `:sparkles:` (new feature), `:bookmark:` (version bump), `:symbols:` (translations/strings), `:hammer:` (refactor), `:test_tube:` (experimental).

## Required companion add-ons

The skin is not self-contained. `addon.xml` `<requires>` pulls in add-ons that the XML actively depends on — breaking a contract with these is the most common source of subtle bugs:

- **`script.skinvariables`** — the code generator. Most dynamic markup in this skin is generated, not hand-written (see below).
- **`script.texturemaker`** — generates recoloured textures at runtime into `special://profile/addon_data/script.texturemaker/ArcticFuse2/`. `Includes_Textures_Script.xml` references those generated paths; `Includes_Textures_Native.xml` holds the non-generated equivalents.
- **`plugin.video.themoviedb.helper`** — supplies artwork, extended metadata, blur/crop images, and the widget monitor. Many `Skin.SetBool(TMDbHelper.*)` toggles are initialised in `shortcuts/skinvariables-startup.json`.
- Resource add-ons for weather icons, studio logos, and the CJK Unicode fontset.

## Architecture

### `1080i/` — the skin itself (~200 XML files)

One `<res>` folder serves every aspect ratio; ratio differences are handled by swapping constant files, not by duplicating layouts.

- **`Includes.xml` is the master manifest.** Every other `Includes_*.xml` is pulled in from here, and **order matters** — constants and expressions load before the includes that consume them. Conditional `<include file=... condition=...>` lines here are how the skin switches whole constant sets: aspect ratio (`Includes_Constants_4x3.xml` … `Includes_Constants_21x9.xml`), home menu style (`HomeStyle_Dialog` / `HomeStyle_Ribbon`), mouse/touch mode, FlixArt size. A new top-level include file is invisible until it is registered in `Includes.xml`.
- **Windows** map to Kodi's built-in window names (`Home.xml`, `MyVideoNav.xml`, `VideoOSD.xml`, …) plus custom windows `Custom_11XX_*.xml`, whose numeric ID is part of the filename and is referenced by `ActivateWindow(1105)`-style calls throughout the XML. Renumbering a custom window means grepping the whole tree.
- **`1080i/IDs`** (extensionless plaintext) is the hand-maintained registry of control-ID conventions. Read it before adding any control ID. Key ranges: `50-59` view item styles (square/landscape/poster/circle/card/board/quad/disc), `5X0` wall views, `5X2`/`5X3` combined views, `55X` list views, `300-340` home page controls, `400` widget grouplist.
- **Layering convention** in filenames: `Includes_Views_*` (view definitions per shape) → `Includes_Layouts.xml` / `Includes_Items.xml` (item layouts) → `Includes_Objects.xml` / `Includes_Furniture.xml` (reusable controls) → `Includes_Constants*.xml` / `Includes_Dimensions.xml` (numbers) / `Includes_Colors.xml` (colour variables) / `Includes_Expressions.xml` (boolean expressions).
- Files named `script-*-includes.xml` are **generated** and pulled in near the end of `Includes.xml`. Do not edit them; `.gitignore` excludes the per-user generated variants.

### Generated XML — `shortcuts/`

This is the part that is non-obvious. `script.skinvariables` reads JSON configs in `shortcuts/` and writes XML into `1080i/`. Changing menus, widgets, or hubs usually means editing the generator inputs, not `1080i/`.

- `skinvariables-generator.json` is the entry point: it lists `genxml` datafiles and writes `1080i/script-skinvariables-generator-includes-{skinuser}.xml`. Its `buildv` field is the build-version stamp — bump it to force every user's includes to regenerate after a template change.
- `generator/data/base/*.xml` describe *what* to generate (home menu, hubs, side menu, search, spotlight, options tray); `generator/data/parts/*.xmltemplate` and `generator/data/build/*.xmltemplate` are the string-substitution templates (`{item_label}`, `{includes_name}`, …) they expand through.
- `skinvariables-shortcut-*.json` are the user-facing menu definitions (home, side, search, context, config, power). `prebuilt/library-basic/` and `prebuilt/tmdb-basic/` are the presets the first-run wizard (`Custom_1180_Wizard.xml`) copies in.
- `skinvariables.xml`, `skinvariables-labels.xml`, `skinvariables-images.xml` are hand-written `<variable>`/`<expression>` sources. They use `{listitem}` as a placeholder and a `containers="50,51,52,..."` attribute; the script expands each into one concrete variable per container ID. That is why `$VAR[Label_Title]` resolves differently per view — do not hand-expand these.
- `builtins/*.json` are small runtime rule-scripts invoked as `RunScript(script.skinvariables,run_executebuiltin=special://skin/shortcuts/builtins/....json,use_rules)`.
- `skinvariables-startup.json` seeds skin settings on first run and every startup; `skinviewtypes.json` maps view IDs to localised names and preview images.
- The **SkinUser** mechanism (`Skin.String(SkinVariables.SkinUser)`) gives each profile its own generated include file; `Includes.xml` chooses between the default and the per-user file at load.

### Everything else

- `language/resource.language.*/strings.po` — skin strings start at `#31000`. Add new strings to `en_gb` only (with the `#: /1080i/File.xml` source comment); other locales come in from translators via PRs.
- `colors/defaults.xml` — colour scheme. Names follow `<area>_<role>_<alpha>`, e.g. `main_fg_70`, `dialog_bg_100`. Alpha suffix is a percentage encoded in the leading hex byte. `Skin default - Light dialogs.xml` is an alternate scheme; keep names in sync across both.
- `fonts/` + `Font.xml` — fontsets are `<include content="Font_Default_*">` with `<param>` overrides; the Unicode fontset substitutes fonts from `resource.font.robotocjksc` and also overrides line-spacing params.
- `extras/` — non-code assets referenced via `special://skin/extras/...`: `icons/` (~1800 FontAwesome-derived PNGs), `playlists/` (smart playlists used as default widget paths), `viewtypes/` & `widgets/` (preview thumbnails shown in selectors), `nodes/`, `profiles/`, `textures/` (source PNGs for texturemaker), `backgrounds/`, `hub/`.
