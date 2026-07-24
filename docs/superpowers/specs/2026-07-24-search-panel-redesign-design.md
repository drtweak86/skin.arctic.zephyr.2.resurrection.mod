# Search Panel Redesign

## Context

Search (and Ask Gemini, which shares the same trigger chain) currently opens the
native modal `DialogKeyboard.xml`, then on confirm activates window `1138`
(`Custom_1138_Search.xml`), which reads `Skin.String(SearchTerm)` once and builds
two static TMDB Helper queries (movie/tv).

Debugging a "search box appears then disappears" report on 2026-07-24 traced the
failure to this modal flow: `Custom_1138_Search.xml` has `<onload
condition="String.IsEmpty(Skin.String(SearchTerm))">Back</onload>`, which is
correct-by-design for a cancelled search, but the modal keyboard's
`<defaultcontrol always="true">300</defaultcontrol>` (the "Done" button — this is
stock Kodi/Estuary convention, confirmed unchanged across every skin variant on
this system, not a bug) combined with remote-app text injection proved fragile in
practice for this user's setup (Kodi mobile remote app). Live inspection via
`XBMC.GetInfoLabels` during a paused mid-type state showed the edit control (312)
empty and focus already on "Done", i.e. no reliable window to inject text before
the modal's default-focus/submit boundary is crossed.

Separately, the user asked whether search could behave more like
`skin.arctic.fuse.mod` (KD637's Arctic Fuse Mod 2), which they use elsewhere and
which shares `script.skinvariables` as a common dependency with this skin ("both
jurialmunkey skins at the core"). Comparison confirmed Fuse 2's search
(`Includes_Search.xml`, ~1000 lines) is a fundamentally different architecture: a
live-search panel with an **inline** `<control type="edit" id="9099">` sitting
directly in the panel (not a separate modal dialog), category tabs, live
per-keystroke result refresh, and a dedicated search-history UI.

Checked resurrection.mod against Fuse 2's supporting includes
(`Categories_Selector`, `DialogCustom_Favourites`, `Widget_End_Bumper`,
`Furniture_Bottom_Right`, `Animation_ConditionalFade`,
`View_Furniture_Bottom_ViewLine`): none exist in this skin (only
`Object_Hidden_List` does). A literal port is not viable; only the underlying
mechanism (inline live edit control instead of modal dialog) is worth adopting,
rebuilt with resurrection.mod's own existing include vocabulary.

## Decision

Adopt Fuse 2's core mechanism — an inline, always-present edit control feeding
live per-keystroke result containers — rebuilt natively for resurrection.mod,
without porting Fuse 2's additional search-history panel, widget-preview rows, or
no-results animation states (deferred as a possible follow-up; out of scope here).

This is also the fix for the 2026-07-24 flakiness: removing the modal
keyboard/Done-button round trip from the typing path removes the exact boundary
where text injection from a remote app was getting lost.

## Architecture

- New file `1080i/Includes_CustomSearch.xml`, registered via `<include
  file="Includes_CustomSearch.xml" />` in `1080i/Includes.xml` (after the
  `script-skinshortcuts-includes.xml` line). Kept out of
  `script-skinshortcuts-includes.xml` because that file is regenerated from
  scratch on every `Home.xml` load and silently wipes hand-authored includes
  (see `kodi_skinshortcuts_includes_regen` memory / prior incident on this skin).
- Window `1138` (`Custom_1138_Search.xml`) is rebuilt to host the new panel in
  place of the current header+two-list layout.
- Inline `<control type="edit" id="9199">` (ID confirmed unused elsewhere in this
  skin) lives directly in the panel. Selecting it with a physical/IR remote still
  opens the native on-screen keyboard (standard Kodi behavior for edit controls,
  unchanged) — this design does not remove that fallback. Remote apps using
  `Input.SendText` write into it directly while it is simply focused, with no
  modal dialog or "Done" button boundary in the path.
- Tab row: a plain horizontal `grouplist` with 4 entries — Movies, TV, People,
  Anime. Not Fuse 2's `Categories_Selector` (does not exist here); a minimal
  equivalent built from controls this skin already has.
- 4 result containers (one per tab), following the existing
  `Object_SearchList_Template` pattern already used by TMDB search/Gemini search
  in `Includes_Object.xml`.
- Content sources:
  - Movies/TV/People: `plugin://plugin.video.themoviedb.helper?info=search&tmdb_type={movie|tv|person}&query=` (unchanged from today).
  - Anime: `plugin://plugin.video.otaku/search_anime/<query>` — Otaku's own
    native search route (confirmed present in `OtakuBrowser.py`/`Main.py`;
    handles its own pagination and search history independently of the skin).

## Data Flow

Keystroke → edit control `9199` label updates live → each tab's container
`content` param (`plugin://...&query=$INFO[Control.GetLabel(9199)]`)
auto-re-evaluates on label change, using the same live-content mechanism already
proven in `Object_SearchList_Template` (`Includes_Object.xml:104`, currently used
with `Skin.String(SearchTerm)` — swapped here for `Control.GetLabel(9199)` so it
updates per keystroke rather than only on modal-dialog confirm) → results refresh
per tab with no separate submit step.

## Trigger Points

All four existing search entry points collapse from the current 3-action chain
(`Skin.Reset(SearchTerm)` → `Skin.SetString(SearchTerm)` → `ActivateWindow(1138)`)
down to a single `ActivateWindow(1138)`, with the new window's `onload` handling
initial focus to control `9199`:

- `shortcuts/overrides.xml` (the central `<override action="Skin.SetString(SearchTerm)">` definition, and the `search` shortcut entry at line 214)
- `shortcuts/mainmenu.DATA.xml` (main menu search shortcut)
- `1080i/Includes_Home.xml` (Home corner-icon search button)
- `1080i/Includes_Items.xml` (PVR-context item list search entry)

## Error Handling

- Queries shorter than 2 characters do not trigger a plugin call (avoids
  spamming TMDB Helper/Otaku per keystroke on 1-character input, and avoids the
  empty-query `GetDirectory` errors observed in `kodi.log` during the 2026-07-24
  incident).
- Below the 2-character threshold (including empty), containers show a plain
  placeholder state — no error dialog.
- Backspacing back below the threshold clears results cleanly via the same gate.
- No custom "no results" animation or search-history UI in this iteration
  (Approach B / follow-up scope).

## Testing

Manual only — no automated test harness exists for Kodi skin XML in this
project.

1. Type via the Kodi mobile remote app: confirm text appears live in the edit
   control and all 4 tabs' results update per keystroke.
2. Type via a physical/IR remote (native on-screen keyboard fallback): confirm
   this path still works unchanged.
3. Confirm Anime tab results are playable through Otaku identically to Otaku's
   own native search.
4. Confirm all 4 entry points (Home corner icon, PVR context item, and the two
   `overrides.xml`/`mainmenu.DATA.xml`-driven shortcuts) open the new panel
   correctly.
5. Confirm empty/1-character queries produce no `GetDirectory` errors in
   `kodi.log`.

## Out of Scope (possible follow-up)

- Fuse 2-style dedicated search-history panel/dialog.
- Widget-preview "spotlight" rows per category.
- No-results animation states.
- Any change to Otaku's own search behavior/history (addon-side, unaffected by
  this skin change).
