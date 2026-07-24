# Skin + TMDb Helper Cleanup Pass

## Context

While building the Ask Gemini feature (2026-07-21), a code-quality survey was run across both the skin (skin.arctic.zephyr.2.resurrection.mod) and the TMDb Helper addon (plugin.video.themoviedb.helper, git repo at `/home/frankie/GitHub/plugin.video.themoviedb.helper`, branch `refactor/tmdb-helper-cleanup`) to find further tune-up/dedup/slimming opportunities, explicitly excluding other addons (Umbrella, Otaku, Magneto). The survey turned up concrete, verified findings in four categories. This spec scopes exactly what to do about each, based on follow-up investigation and decisions made with the user — several initial "looks orphaned" findings turned out to be live features (or vice versa) once traced further, so nothing here is acted on without a verified call site (or lack thereof).

## Part 1 — Skin: remove confirmed-dead windows/settings

Two windows in this family are auto-managed background dialogs (empty `<controls>`, gated purely by their own `<visible>` condition — Kodi shows them automatically without any `ActivateWindow` call) and are both confirmed **live**, not dead:
- `Custom_1115_AutoVis.xml` — gated by `AutoMusicVis`, which has a real working toggle in the main Settings menu (`SkinSettings.xml` id `19516`, lines 970-974). User confirmed this is how they get fullscreen visualization when playing music from Spotify on their phone (via plugin.audio.spotify). **Do not touch.**
- `Custom_1122_MusicFullscreenEnabler.xml` — no gating setting at all; auto-fullscreens on any music playback unless TvTunes/Live TV is active. User confirmed they use and want this. **Do not touch.**

The remaining three are real interactive dialogs (defaultcontrol + buttons) that require an explicit navigation entry point to ever be seen — and none exists anywhere in the skin (verified via grep across all `1080i/*.xml` and `shortcuts/*.xml` for `ActivateWindow`/`ReplaceWindow` referencing their ids):

- **Delete `Custom_1120_EnableInfoButtons.xml`** (id 1120). Configures 3 skin strings (`infobuttons02/03/04`) that are read nowhere else in the skin — the feature these buttons were meant to drive no longer exists. No other file references these strings, so no other cleanup needed.
- **Delete `Custom_1123_EnableSearchLists.xml`** (id 1123). Configures 14 `DisableSearchList.*` settings that are read nowhere else — the native-search filtering this was meant to control was never wired up (or was removed). No other file references these settings.
- **Delete `Custom_1121_EnableIndicators.xml`** (id 1121), but first extract one piece: this dialog is the *only* entry point to open the color picker (window 1117) for a custom "watched/progress" indicator color (`Skin.SetString(watchedprogresscolor.name, ...)`), and that color value is genuinely read live at `Includes.xml:92`. The user has a custom color set and wants to keep the ability to change it. Everything else in this dialog (`EnableClassicIndicators`, the six `DisableIndicator.*` toggles) has zero enforcement anywhere in the codebase — verified the actual indicator-icon selection logic in `Includes_Images.xml:150-178` doesn't check any of them — so those toggles do nothing today and are safe to lose. `DisableWatchedOverlay` (the one setting from this dialog that *is* real) already has its own working toggle elsewhere (`SkinSettings.xml:691-692`), so nothing is lost there either.

  **Action:** add one button to `SkinSettings.xml`, near the existing `DisableWatchedOverlay` toggle (around line 691), that does `SetProperty(colorpick,Watched,Home)` then `ActivateWindow(1117)` — copied from the equivalent button currently in `Custom_1121_EnableIndicators.xml`. Then delete `Custom_1121_EnableIndicators.xml` entirely.

Also remove one dead, always-true no-op condition unrelated to the above: `SkinSettings.xml:968`, `<visible>!Skin.HasSetting(BigMusicVis)</visible>` — `BigMusicVis` is set nowhere in the addon, so this condition is permanently true and meaningless. Delete the `<visible>` line (the control it guards should just always show, matching current de-facto behavior).

`TMDbHelper.EnableCrop` was also flagged by the survey as "unreferenced in the skin" — confirmed this is a real, live, cross-addon setting: TMDb Helper's Python side (`monitor/images.py:541`, `monitor/imgmon.py:34,42`, `monitor/player.py:356`) reads it via `get_condvisibility`. **Do not touch.**

## Part 2 — Skin: dedupe template patterns

Two duplication clusters in `Includes_Object.xml` / `Includes_Items.xml`, both following the same shape we already fixed today in `Object_SearchList_Template` (a "Flix" variant added extra params by hardcoding a parallel copy of the template instead of parameterizing the original):

1. **`Flix_Object_Info_Line_Label`/`Object_Info_Line_Label`, `Flix_Object_Info_Line`/`Object_Info_Line`, `Flix_Object_Info_Title`/`Object_Info_Title`, `Flix_Object_Info_Plot`/`Object_Info_Plot`** (`Includes_Object.xml`, base templates around lines 1161-1600, Flix variants around lines 1121-1570). Each Flix variant differs from its base only by a small set of extra params (`font`, `shadowcolor`, `top`, `visible`, `fullgenre`/`percentrating`) that were never added to the base template. Fix: add the missing params to each base template with defaults that reproduce current Flix-variant behavior exactly (verify each default against the Flix version's hardcoded value — don't guess), repoint all ~10 Flix call sites at the base template with explicit param values matching what the Flix variant used to hardcode, then delete the 4 `Flix_*` template definitions. Net: ~250 lines removed, zero visual change if defaults are chosen correctly.

2. **`Items_Settings_RatingsMovies`/`Items_Settings_RatingsTVShows`** (`Includes_Items.xml:2862-2962`), both consumed from `Custom_1150_Dialog.xml`. Identical 6-button block except the `$VAR[Label_CustomRating_Movies_0N]`/`$VAR[Label_CustomRating_TVShows_0N]` label variable and the `SetProperty(ConfigureRatingContent,Movies/TVShows,Home)` onclick value. Fix: create one `Items_Settings_Ratings_Template` taking a `contenttype` param (`Movies`/`TVShows`), call it twice from `Custom_1150_Dialog.xml`, delete the two originals. Net: ~100 lines removed.

Both require live Kodi verification after the change (visual comparison before/after for the Flix views and the ratings dialog) since they touch rendered UI, not just internal logic.

## Part 3 — TMDb Helper: remove dead/orphaned code

Mechanical, no behavior change, low risk:

- Delete the dead commented-out `finalise_image()` block and its `# FIXME IMAGES` comment: `items/database/baseitem_factories/factory.py:9-17`.
- Delete 4 dead commented-out import lines: `update/builder/tags.py:1-4`, `api/tmdb/users.py:4`, `items/database/baseview_factories/factory.py:1,5`.
- Remove 4 redundant local imports (already imported at module level, re-imported inside a function/method for no reason): `items/directories/trakt/lists_random.py:39,52` (`import random` ×2), `script/method/tmdb.py:20` (`get_localized`), `api/trakt/sync/datatype.py:477` (`ParallelThread`), `items/directories/tmdb/lists_discodir.py:47` (`get_property`).
- Delete orphaned module `api/trakt/items.py` (212 lines) — `TraktItems` class and helpers, confirmed zero imports anywhere in the repo, superseded by the `ItemListSyncDataFactory` architecture in `api/trakt/sync/itemlist.py`.
- Delete orphaned classes, each confirmed to have zero references/subclasses anywhere outside their own definition:
  - `ListPlaybackProgress` (`items/directories/trakt/lists_sync.py:140`) — sibling classes are registered in `addon/consts.py`'s route table, this one isn't.
  - `SyncDataSetters` (`api/trakt/sync/datasync.py:30`) — never mixed into `SyncData` (which only inherits `SyncDataGetters`).
  - `SyncDataGetterProgressCollectedUnHidden`, `SyncDataGetterCalendarUnHidden`, `SyncDataGetterDroppedCollectionUnHidden`, `SyncDataGetterDroppedCalendarUnHidden` (`api/trakt/sync/datasync.py:114,118,126,130`) — unlike sibling getter classes, these have no subclasses and no wiring in the `get_all_*_getter` methods.
  - `BasicCacheService` (`files/bcache.py:17`).
- Leave `CProfiler` (`addon/logger.py:31`) alone — zero current references, but it's a plausible deliberate manual-debugging utility, not obviously dead code; removing it isn't worth the small risk of losing a debugging tool for negligible line-count benefit.

## Part 4 — TMDb Helper: dedupe `lists_view_db.py`

`resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py` has several classes that are byte-identical except for one or two string literals:
- `ListFanart`/`ListPoster`/`ListImage`/`ListThumb` (lines 55-80) — differ only in the view-name string passed to `BaseViewFactory`.
- `ListCast`/`ListCrew` (lines 83-98) — differ only in `'castmember'`/`'crewmember'` and `container_content` type.
- `ListSeries`/`ListStarredMovies`/`ListStarredTvshows`/`ListCrewedMovies`/`ListCrewedTvshows` (lines 101-163) — same `BaseViewFactory(view_name, tmdb_type, tmdb_id, ...)` + `get_kodi_database` + `container_content` shape, differing only in 2-3 string params.
- `ListStarredCombined`/`ListCrewedCombined`/`ListCreditsCombined` (lines 131-196) — the exact same movie/show-count try/except block copy-pasted three times verbatim.

Fix: introduce a small shared base (class attributes for `view_name`/`tmdb_type` plus one method for the movie/show-count branching used by the three "Combined" classes), collapsing all of the above into much shorter subclasses that just set attributes. ~120 lines removed, identical behavior. Needs test coverage/manual verification of each affected route (fanart/poster/image/thumb lists, cast/crew lists, starred/crewed lists) since this changes control flow, not just moving code around.

## Explicitly deferred (not in this pass)

- **`mappings.py`'s `ItemMapperMethods`** (`items/database/mappings.py`, ~750-line single class mixing art/credits/genre/TV-specific/translation concerns). Confirmed to be a genuine "god class" with cleanly separable responsibilities (could become `ArtMapperMethods`/`CreditsMapperMethods`/`TVMapperMethods` mixins), but this is a multi-file restructuring that touches a heavily-used class — scope as its own future project with its own design/plan, not bundled into this cleanup pass.
- **The ~28 remaining TODO/FIXME/HACK comments** across the TMDb Helper codebase — all are either speculative "maybe someday" feature notes or require domain/product knowledge to resolve correctly (e.g. Trakt query-logic uncertainty, season-finale detection heuristics). None are safe mechanical fixes. Leave them as-is.

## Verification

**TMDb Helper (Part 3 & 4):**
- `python3 -m py_compile` on every changed file.
- `python3 -m compileall` across `resources/tmdbhelper/lib/` to catch any import breakage from deleted modules/classes.
- Grep-verify (again, post-change) that nothing references the deleted module/classes.
- Exercise the affected routes live in Kodi after restart: browse to a fanart/poster/image/thumb list, a cast/crew list, and a starred/crewed list (Part 4); confirm Trakt sync/history still works normally (Part 3, since `lists_sync.py` and `datasync.py` were touched).

**Skin (Part 1 & 2):**
- `xmllint --noout` (or ElementTree parse) on every changed/new XML file.
- Restart Kodi, confirm no new skin XML parse errors in `kodi.log`.
- Confirm Settings menu still shows the `DisableWatchedOverlay` toggle plus the newly-added watched-color picker button, and that picking a color still visibly changes the watched-progress indicator color.
- Visually compare a Flix view and a non-Flix (base) view before/after the Part 2 template consolidation to confirm no visual regression.
- Visually compare the Ratings configuration dialog (Movies and TV Shows tabs) before/after Part 2's Ratings template consolidation.
- Confirm AutoVis (Spotify playback) and MusicFullscreenEnabler still behave exactly as before (untouched, but worth one confirmation pass since they're in the same file family being edited).
