# Arctic Zephyr 2 Resurrection Clean Reset and Dark Glass Design

## Objective

Reset Arctic Zephyr 2 Resurrection to its clean cached `1.0.51` package, restore the user's functional customizations through supported configuration paths, and then add an optional dark translucent glass appearance native to this skin.

## Recovery and Source

Use the locally cached package:

`/home/frankie/.kodi/addons/packages/skin.arctic.zephyr.2.resurrection.mod-1.0.51.zip`

Before replacement, preserve complete timestamped copies of:

- the installed skin add-on directory;
- the skin's `userdata/addon_data` directory;
- `script.skinshortcuts` menu and generated configuration;
- the already verified search configuration and existing backup set;
- `guisettings.xml`, including the selected skin texture theme, colour theme, font, and zoom;
- the installed weather icon resources and active weather icon path.

Validate the cached archive's add-on ID and version before using it. No files are deleted; replaced directories are renamed into the backup set so rollback remains possible.

## Phase One: Clean Functional Baseline

Install the pristine `1.0.51` skin files and start with newly generated skin state. Do not restore modified skin XML wholesale because that would reintroduce the broken custom control topology.

Restore supported user configuration selectively:

- main-menu entries for Search, Ask Gemini, Movies, TV Shows, Anime, and Settings;
- Movies, TV Shows, and Anime/Otaku widget definitions;
- TMDb movie search and TMDb TV search;
- Otaku movie search using `plugin://plugin.video.otaku/search_movie/`;
- Otaku TV search using `plugin://plugin.video.otaku/search_tv_show/`;
- the Multi Flix home layout only after the baseline navigation test passes;
- Flix Poster as the preferred view for compatible video library, plug-in, search, and recommendation containers. Screens that do not expose a Flix view retain their native view.

Widgets must be defined through Skin Shortcuts configuration so its generator creates unique control IDs and supported focus navigation. Do not restore `Includes_CustomWidgets.xml` or directly edit generated `script-skinshortcuts-includes.xml`.

Acceptance checks for phase one:

- Up and Down traverse every populated widget row in Movies, TV Shows, and Anime;
- focus returns to the main menu at the top and bottom;
- empty widget rows do not trap focus;
- all four search providers return results;
- Ask Gemini opens TMDb Helper's prompt and returns mixed movie/TV recommendations without crashing Kodi;
- Multi Flix is the active home layout and compatible video containers open in Flix Poster view;
- the approved colour, icon, artwork, and graphical settings survive the reset;
- Outline HD icons render for current conditions and the multi-day forecast;
- no widget or search path is lost after a Kodi restart;
- Kodi's log contains no new invalid-control, failed-focus, or malformed-skin errors attributable to the restored configuration.

## Ask Gemini Safety Gate

Use TMDb Helper's installed native route:

`plugin://plugin.video.themoviedb.helper/?info=gemini`

Opening the route without a `query` parameter delegates keyboard input, Gemini API access, result mapping, and mixed movie/TV rendering to TMDb Helper. The menu label is **Ask Gemini**. Preserve the existing TMDb Helper settings directory and configured Gemini API key; never copy, print, or rewrite the key.

A live test of this route on Kodi 21.3 with Python 3.13 produced a native `SIGSEGV` in `_sqlite3` while multiple TMDb Helper workers accessed its caches. The same run had previously logged malformed cached Trakt data. Before exposing Ask Gemini:

- preserve a timestamped copy of TMDb Helper's complete add-on-data directory;
- rotate only its generated database/cache directories out of the active path, leaving `settings.xml` and credentials in place;
- apply the Python 3.13 SQLite statement-cache workaround (`cached_statements=0`) to every connection factory used by TMDb Helper's native database and Jurialmunkey shared cache layers;
- preserve pristine copies of every patched source file so the workaround can be reversed or reapplied after an add-on update;
- test the Gemini route directly before adding its main-menu entry.

If the direct Gemini test crashes Kodi again, raises a new native database failure, or cannot return results, restore the patched source files, omit Ask Gemini from the active menu, and continue the skin reset without it. Do not weaken this gate by treating a prompt opening as success; at least one complete recommendation response must render and Kodi must remain stable through a restart.

## Flix Defaults

Enable the skin's existing `HomeMultiFlixView` setting rather than cloning another skin's home layout. Preserve the clean skin's navigation and control IDs.

For content windows, prefer the existing **Flix Poster** view (control/view ID `526`) wherever `MyVideoNav.xml` exposes it. Apply the preference through Kodi's supported view-state mechanism for existing paths and the clean skin's compatible default for new video paths. Do not force view `526` into settings, PVR, music, pictures, or any other window that does not declare it. A fallback native view must remain focusable when Flix Poster is unavailable.

## Visual Preference Preservation

Do not restore the old skin `settings.xml` wholesale. Parse the backup and copy only the approved presentation settings into the clean skin state after phase-one navigation passes.

Retain these current Kodi look-and-feel values:

- texture theme: `Square`;
- colour theme before enabling Dark Glass: `Darker with dark dialogs`;
- font: `Default`;
- skin zoom: `0`.

Retain these current Arctic Zephyr 2 Resurrection presentation values:

- `focuscolor.name=ffd50000`;
- `gradientcolor.name=ffd50000`;
- `Icons=colorful` and `icons.label=Colorful`;
- `enablegeometric=true`;
- `enableclearlogo=true`;
- `showosdclearart=true`;
- `showdate=true`;
- `posterhighlight=Mix`;
- `homemultiflixview=true`;
- `flixhidemenu=true`;
- `disabledprofileinfo=true`;
- `selectbox.thin=true`;
- `kenburnseffect=true`.

Also preserve unrelated safe graphical preferences for artwork visibility, labels, indicators, OSD presentation, fonts, colour names, and icon choices when their setting IDs exist unchanged in the pristine skin. Explicitly exclude generated hashes, temporary search text, widget paths/targets, Skin Shortcuts state, menu-layout booleans that conflict with Multi Flix, custom control IDs, and any setting that points to `Includes_CustomWidgets.xml` or the failed navigation experiment.

Generate a machine-readable restoration manifest recording every copied setting, its source value, and whether the clean skin declared the same setting ID. Unknown or obsolete settings remain only in the backup and are not injected into the clean profile.

## Weather Icons

Use the Kodi Omega resource add-on `resource.images.weathericons.outline-hd` (Weather Icons - Outline HD, version `0.0.3` or newer compatible release). Install it through Kodi's add-on manager if absent, then set:

`weather.icons.path=resource://resource.images.weathericons.outline-hd/`

The skin already selects weather resources through `script.image.resource.select` and reads numbered PNG condition codes from the chosen resource path, so no weather-window XML override is needed. Keep `weather.openmeteo` as the provider. Verify current-condition, daily forecast, and top-bar icons; the top bar may continue using the skin's bundled white icons where its pristine XML deliberately hard-codes them.

## Phase Two: Native Dark Glass Theme

Create Dark Glass as an optional Arctic Zephyr 2 Resurrection setting rather than copying Arctic Horizon 2 layouts or dependencies.

The theme will use:

- the skin's existing neutral textures and color variables;
- dark translucent panel overlays with consistent opacity;
- a subtle native gradient/noise treatment where an existing compatible texture is available;
- readable foreground, selected, and disabled text colors;
- unchanged geometry, control IDs, widget providers, and navigation.

Place new theme behavior in a dedicated include/color file and expose one toggle in Skin Settings. Existing controls consume the theme through shared color/texture variables, limiting duplication and making the feature removable. No live blur effect is assumed because Kodi's skin engine does not provide a portable real-time backdrop blur.

Acceptance checks for phase two:

- the toggle changes menus, widget backplates, dialogs, and information panels consistently;
- artwork remains visible through the dark translucent layer;
- focused items retain clear contrast;
- disabling Dark Glass restores the clean skin appearance;
- navigation and all phase-one provider mappings remain unchanged.

## Aerial Screensaver Safety

Do not call `ReloadSkin()` while Aerial is active or has a video player. Aerial 3.0.3 can leave its playback loop orphaned when the skin reload destroys its modal window. Before any required reload, verify `Player.GetActivePlayers` is empty and reset Kodi's idle timer. Prefer one controlled Kodi restart for final integration verification. Confirm Aerial starts normally, exits on input, and leaves no active player afterward.

## Rollback

If the clean baseline cannot reproduce the requested layout or mappings, restore the complete pre-reset directories from the timestamped backup. Dark Glass remains a separate optional layer and can be removed without affecting restored menus, widgets, or search providers.
