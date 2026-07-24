# Skin + TMDb Helper Cleanup Pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove confirmed-dead skin windows/settings and TMDb Helper code, and consolidate four duplicated skin template pairs plus one duplicated Python class family, with zero behavior change to anything currently working.

**Architecture:** Two independent codebases, touched in four parts matching the approved spec. Skin XML changes are validated by `python3 -m xml.etree.ElementTree` parsing plus live Kodi restart + visual comparison (no automated test runner exists for Kodi skins). TMDb Helper Python changes are validated by `python3 -m py_compile` / `compileall` plus live Kodi restart and route exercising, following the same verification pattern used earlier this session for the Gemini work.

**Tech Stack:** Kodi skin XML (skin.arctic.zephyr.2.resurrection.mod), Python 3.13 (plugin.video.themoviedb.helper, git repo on branch `refactor/tmdb-helper-cleanup`).

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-21-skin-tmdbhelper-cleanup-design.md` (this skin's docs folder).
- Skin repo is NOT a git repository — no commits possible there; edits are direct file changes, confirmed working via live Kodi test instead of a commit log.
- TMDb Helper repo IS a git repository — commit after each task, following the existing commit-message style on branch `refactor/tmdb-helper-cleanup` (plain sentence summary + body explaining why, `Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>` trailer).
- Never hand-edit `1080i/script-skinshortcuts-includes.xml` (auto-regenerated).
- Before any Kodi restart/skin reload: check `System.ScreenSaverActive` and dismiss Aerial first if active.
- Custom_1115_AutoVis.xml and Custom_1122_MusicFullscreenEnabler.xml must NOT be touched in any task (user confirmed both are actively used).
- `TMDbHelper.EnableCrop` must NOT be touched (confirmed live, read by TMDb Helper's Python monitor code).

---

## Part 1 — Skin: remove confirmed-dead windows/settings

### Task 1: Relocate the watched/progress color picker button

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/SkinSettings.xml:688-693`

**Interfaces:**
- Consumes: existing `Defs_Settings_Button` include, existing color-picker window 1117 (already used by ids 19301/19305 in the same file).
- Produces: a live `Skin.SetString(watchedprogresscolor.name, ...)` entry point in the main Settings menu, replacing the one about to be deleted with Task 2.

- [ ] **Step 1: Read current content around the insertion point**

The `DisableWatchedOverlay` toggle currently reads (lines 688-693):

```xml
                        <control type="radiobutton" id="19402" description="Disable watched indicators">
                            <include>Defs_Settings_Button</include>
                            <label>$LOCALIZE[31317]</label>
                            <selected>!Skin.HasSetting(DisableWatchedOverlay)</selected>
                            <onclick>Skin.ToggleSetting(DisableWatchedOverlay)</onclick>
                        </control>
```

- [ ] **Step 2: Insert the new button immediately after this control**

Add a new `<control type="button" id="19403">` directly after the `</control>` closing the `19402` block (i.e., after line 693), matching the existing color-picker button pattern already used at lines 410-416/418-424 in the same file:

```xml
                        <control type="button" id="19403" description="Watched/progress indicator color">
                            <include>Defs_Settings_Button</include>
                            <label>$LOCALIZE[31573]</label>
                            <label2>$INFO[Skin.String(watchedprogresscolor.name)]</label2>
                            <onclick>SetProperty(colorpick,Watched,Home)</onclick>
                            <onclick>ActivateWindow(1117)</onclick>
                            <enable>!Skin.HasSetting(DisableWatchedOverlay)</enable>
                        </control>
```

Note: the `<enable>` condition drops the `!Skin.HasSetting(EnableClassicIndicators)` half that the original button (in `Custom_1121_EnableIndicators.xml`) had — `EnableClassicIndicators` is deleted in Task 2 and would always evaluate false afterward anyway, so this is a simplification with no behavior change once Task 2 lands. `id="19403"` was confirmed free via grep — verify again if this plan runs much later than it was written:

```bash
grep -rn 'id="19403"' /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/*.xml
```
Expected: no output (only this plan file references it before the edit).

- [ ] **Step 3: Validate XML**

```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/SkinSettings.xml'); print('OK')"
```
Expected: `OK`

- [ ] **Step 4: Defer live verification**

Live verification (does the button appear, does clicking it open the color picker, does picking a color change the indicator) happens in Task 2's verification step, after `Custom_1121_EnableIndicators.xml` is deleted — testing now would be redundant since both changes need to land before the full before/after picture makes sense.

### Task 2: Delete Custom_1121_EnableIndicators.xml

**Files:**
- Delete: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1121_EnableIndicators.xml`

**Interfaces:**
- Consumes: Task 1's relocated button (must land first).
- Produces: nothing — this window and its settings (`EnableClassicIndicators`, `DisableIndicator.New/Episodes/Watched/Progress/Library/MovieSet`) have zero other references anywhere in the skin (verified via grep during brainstorming), so deleting it removes them entirely with no dangling references.

- [ ] **Step 1: Final confirmation these settings are truly unreferenced elsewhere**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
for term in "EnableClassicIndicators" "DisableIndicator\."; do
  echo "--- $term ---"
  grep -rl "$term" "$SKIN"/1080i/*.xml | grep -v "Custom_1121_EnableIndicators.xml"
done
```
Expected: no output for either term (confirms nothing outside the file being deleted references these settings).

- [ ] **Step 2: Delete the file**

```bash
rm /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1121_EnableIndicators.xml
```

- [ ] **Step 3: Confirm no remaining reference to window id 1121**

```bash
grep -rn "ActivateWindow(1121)\|ReplaceWindow(1121)\|Window.IsVisible(1121)\|IsActive(1121)" /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/*.xml /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/shortcuts/*.xml
```
Expected: no output (confirms this was already true before deleting — the window was unreachable).

- [ ] **Step 4: Restart Kodi and verify live**

Check `System.ScreenSaverActive` first; dismiss Aerial if active. Restart Kodi fully. Then:
- Open Settings, navigate to the section containing the watched-indicators toggle (near "Disable watched indicators").
- Confirm a new "Watched/progress indicator color" button appears, showing the currently-set color in its label2.
- Click it, confirm the color picker (window 1117) opens.
- Pick a different color, confirm the watched/in-progress indicator color on a poster actually changes to match.
- Confirm no new skin XML parse errors in `kodi.log`:
  ```bash
  grep -n "Custom_1121\|SkinSettings.xml" /home/franking/.kodi/temp/kodi.log | tail -20
  ```
  (fix the obvious typo in that path before running — it's `/home/frankie/.kodi/temp/kodi.log`)

### Task 3: Delete Custom_1120_EnableInfoButtons.xml

**Files:**
- Delete: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1120_EnableInfoButtons.xml`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — confirmed zero other references anywhere.

- [ ] **Step 1: Final confirmation**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
grep -rl "infobuttons0" "$SKIN"/1080i/*.xml | grep -v "Custom_1120_EnableInfoButtons.xml"
grep -rn "ActivateWindow(1120)\|ReplaceWindow(1120)\|Window.IsVisible(1120)\|IsActive(1120)" "$SKIN"/1080i/*.xml "$SKIN"/shortcuts/*.xml
```
Expected: no output from either command.

- [ ] **Step 2: Delete the file**

```bash
rm /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1120_EnableInfoButtons.xml
```

- [ ] **Step 3: Restart Kodi, confirm no new parse errors**

```bash
grep -n "Custom_1120" /home/frankie/.kodi/temp/kodi.log | tail -10
```
Expected: no output (or only pre-existing lines from before the restart, which won't exist since the file no longer loads).

### Task 4: Delete Custom_1123_EnableSearchLists.xml

**Files:**
- Delete: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1123_EnableSearchLists.xml`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — confirmed zero other references anywhere.

- [ ] **Step 1: Final confirmation**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
grep -rl "DisableSearchList" "$SKIN"/1080i/*.xml | grep -v "Custom_1123_EnableSearchLists.xml"
grep -rn "ActivateWindow(1123)\|ReplaceWindow(1123)\|Window.IsVisible(1123)\|IsActive(1123)" "$SKIN"/1080i/*.xml "$SKIN"/shortcuts/*.xml
```
Expected: no output from either command.

- [ ] **Step 2: Delete the file**

```bash
rm /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1123_EnableSearchLists.xml
```

- [ ] **Step 3: Restart Kodi, confirm no new parse errors**

```bash
grep -n "Custom_1123" /home/frankie/.kodi/temp/kodi.log | tail -10
```
Expected: no output.

### Task 5: Remove the dead BigMusicVis visibility condition

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/SkinSettings.xml:963-969`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — `BigMusicVis` is confirmed set nowhere in the addon, so this condition is permanently true (a no-op).

- [ ] **Step 1: Confirm BigMusicVis is genuinely never set**

```bash
grep -rn "Skin.SetBool(BigMusicVis\|Skin.ToggleSetting(BigMusicVis" /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/*.xml
```
Expected: no output.

- [ ] **Step 2: Remove the dead `<visible>` line**

Current content (lines 963-969):

```xml
                        <control type="radiobutton" id="19515" description="Musicvideo auto info">
                            <include>Defs_Settings_Button</include>
                            <label>$LOCALIZE[31314]</label>
                            <selected>!Skin.HasSetting(DisableMusicVideoAutoInfo)</selected>
                            <onclick>Skin.ToggleSetting(DisableMusicVideoAutoInfo)</onclick>
                            <visible>!Skin.HasSetting(BigMusicVis)</visible>
                        </control>
```

Change to:

```xml
                        <control type="radiobutton" id="19515" description="Musicvideo auto info">
                            <include>Defs_Settings_Button</include>
                            <label>$LOCALIZE[31314]</label>
                            <selected>!Skin.HasSetting(DisableMusicVideoAutoInfo)</selected>
                            <onclick>Skin.ToggleSetting(DisableMusicVideoAutoInfo)</onclick>
                        </control>
```

(Just delete the `<visible>!Skin.HasSetting(BigMusicVis)</visible>` line — the control was already always-visible in practice, this makes it explicit.)

- [ ] **Step 3: Validate XML**

```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/SkinSettings.xml'); print('OK')"
```
Expected: `OK`

- [ ] **Step 4: Restart Kodi, confirm the "Musicvideo auto info" toggle still appears in Settings**

No visual change expected — this is confirming the removal of an always-true condition didn't hide the control.

---

## Part 2 — Skin: dedupe template patterns

### Task 6: Consolidate Object_Info_Line_Label / Flix_Object_Info_Line_Label

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Object.xml:1161-1196` (base template)
- Modify (in Task 9): `Flix_Object_Info_Line`'s internal references to `Flix_Object_Info_Line_Label` — deferred to Task 9 since `Flix_Object_Info_Line_Label` has zero external callers, only internal ones inside `Flix_Object_Info_Line`.

**Interfaces:**
- Consumes: `Object_Variable_Font` include (`Includes_Variable_Object.xml:4`) and `Object_Font` include (`Includes_Variable_Object.xml:60`) — both confirmed unchanged.
- Produces: `Object_Info_Line_Label` gains an optional `font` param that Task 9 will pass when repointing `Flix_Object_Info_Line`'s internal calls.

**Verified finding:** `Flix_Object_Info_Line_Label`'s extra `font` param is inert — both its label controls set `<font>$PARAM[font]</font>` immediately followed by `<include content="Object_Variable_Font">`, which unconditionally emits its own `<font>` tag via `Object_Font` (`Includes_Variable_Object.xml:62`, `<font>$PARAM[item]</font>`). In Kodi skin XML, the later of two same-name simple tags on one control wins, so `Object_Variable_Font`'s font always overrides `$PARAM[font]`. This means adding the `font` param to the base template changes nothing visually — it's dead weight kept only so `Flix_Object_Info_Line`'s internal calls (which pass `font="font_title_mini"` for one entry) don't need their own separate stripped-down template just to drop one inert param.

- [ ] **Step 1: Read current base template**

Current content (lines 1161-1196):

```xml
    <include name="Object_Info_Line_Label">
        <param name="divider" default="true" />
        <param name="visible" default="true" />
        <definition>
            <control type="label">
                <width>auto</width>
                <label>  |  </label>
                <height>40</height>
                <aligny>top</aligny>
                <textcolor>$VAR[ColorHighlight]</textcolor>
                <include content="Object_Variable_Font">
                    <param name="std" value="font_small" />
                    <param name="alt" value="font_tiny" />
                    <param name="usealt" value="$EXP[FontSetIsMediumOrLarger]" />
                </include>

                <visible>!String.IsEmpty(Control.GetLabel($PARAM[id]))</visible>
                <visible>$PARAM[divider]</visible>
                <visible>$PARAM[visible]</visible>
            </control>
            <control type="label" id="$PARAM[id]">
                <width>auto</width>
                <label>$PARAM[label]</label>
                <height>40</height>
                <aligny>top</aligny>
                <textcolor>$PARAM[textcolor]</textcolor>
                <include content="Object_Variable_Font">
                    <param name="std" value="font_small" />
                    <param name="alt" value="font_tiny" />
                    <param name="usealt" value="$EXP[FontSetIsMediumOrLarger]" />
                </include>

                <visible>$PARAM[visible]</visible>
            </control>
        </definition>
    </include>
```

- [ ] **Step 2: Add the (inert but harmless) font param**

Replace with:

```xml
    <include name="Object_Info_Line_Label">
        <param name="divider" default="true" />
        <param name="visible" default="true" />
        <param name="font" default="font_small" />
        <definition>
            <control type="label">
                <width>auto</width>
                <label>  |  </label>
                <height>40</height>
                <aligny>top</aligny>
                <font>$PARAM[font]</font>
                <textcolor>$VAR[ColorHighlight]</textcolor>
                <include content="Object_Variable_Font">
                    <param name="std" value="font_small" />
                    <param name="alt" value="font_tiny" />
                    <param name="usealt" value="$EXP[FontSetIsMediumOrLarger]" />
                </include>

                <visible>!String.IsEmpty(Control.GetLabel($PARAM[id]))</visible>
                <visible>$PARAM[divider]</visible>
                <visible>$PARAM[visible]</visible>
            </control>
            <control type="label" id="$PARAM[id]">
                <width>auto</width>
                <label>$PARAM[label]</label>
                <height>40</height>
                <aligny>top</aligny>
                <font>$PARAM[font]</font>
                <textcolor>$PARAM[textcolor]</textcolor>
                <include content="Object_Variable_Font">
                    <param name="std" value="font_small" />
                    <param name="alt" value="font_tiny" />
                    <param name="usealt" value="$EXP[FontSetIsMediumOrLarger]" />
                </include>

                <visible>$PARAM[visible]</visible>
            </control>
        </definition>
    </include>
```

Note the default `font_small` matches the `std` value already passed to `Object_Variable_Font` in both controls, so even in the (never-reached, since `Object_Variable_Font` always overrides) hypothetical case where the literal `<font>` tag mattered, the default is harmless.

- [ ] **Step 3: Validate XML**

```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Object.xml'); print('OK')"
```
Expected: `OK`

- [ ] **Step 4: Do not delete `Flix_Object_Info_Line_Label` yet**

It's still referenced by `Flix_Object_Info_Line`, which Task 9 consolidates. Deleting it now would break Task 9's starting state. Leave both templates in place until Task 9 completes.

### Task 7: Consolidate Object_Info_Title / Flix_Object_Info_Title

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Object.xml:1540-1554` (base template — add params)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Home.xml:1844-1854` (repoint 1 call site)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_View_52_Showcase.xml:230-238,313-320,370-374` (repoint 3 call sites)
- Delete (final step): `Flix_Object_Info_Title` definition at `Includes_Object.xml:1518-1538`

**Interfaces:**
- Consumes: nothing new.
- Produces: `Object_Info_Title` gains `shadowcolor` (default `""`), `top` (default `""`), `visible` (default `"true"`) params, in addition to its existing `label`/`font`/`titleheight`/`textcolor`.

**Verified per-call-site params (all 4 Flix_Object_Info_Title callers), used to construct the exact replacement below:**
- `Includes_Home.xml:1844`: passes `label`, `font=font_title_large`, `shadowcolor=ff000000`, `titleheight=$PARAM[titleheight]`, `top=80`, `visible=String.IsEmpty(Container($PARAM[id]).ListItem.Art(clearlogo))`.
- `Includes_View_52_Showcase.xml:230`: passes `font=font_title_large`, `shadowcolor=ff000000`, `titleheight=$PARAM[titleheight]`, `top=80`, `visible=!$EXP[Exp_HasClearlogo]`.
- `Includes_View_52_Showcase.xml:313`: passes `font=font_title_medium`, `shadowcolor=ff000000`, `titleheight=$PARAM[titleheight]`, `top=20`.
- `Includes_View_52_Showcase.xml:370`: passes `font=$PARAM[font]`, `titleheight=$PARAM[titleheight]`, `visible=!$EXP[Exp_HasClearlogo] | !Skin.HasSetting(ShowClearlogo)` — **does not** pass `shadowcolor`, relying on Flix's old default of `ff000000`. This is the one call site needing an explicit `shadowcolor` added to preserve current appearance.

- [ ] **Step 1: Add the three missing params to the base template**

Current content (`Includes_Object.xml:1540-1554`):

```xml
    <include name="Object_Info_Title">
        <param name="label" default="$INFO[ListItem.Label]" />
        <param name="font" default="font_title_large" />
        <param name="titleheight" default="80" />
        <param name="textcolor" default="main_fg_100" />
        <definition>
            <control type="label">
                <label>$PARAM[label]</label>
                <height>$PARAM[titleheight]</height>
                <aligny>top</aligny>
                <textcolor>$PARAM[textcolor]</textcolor>
                <font>$PARAM[font]</font>
            </control>
        </definition>
    </include>
```

Replace with:

```xml
    <include name="Object_Info_Title">
        <param name="label" default="$INFO[ListItem.Label]" />
        <param name="font" default="font_title_large" />
        <param name="titleheight" default="80" />
        <param name="textcolor" default="main_fg_100" />
        <param name="shadowcolor" default="" />
        <param name="top" default="" />
        <param name="visible" default="true" />
        <definition>
            <control type="label">
                <label>$PARAM[label]</label>
                <top>$PARAM[top]</top>
                <height>$PARAM[titleheight]</height>
                <aligny>top</aligny>
                <textcolor>$PARAM[textcolor]</textcolor>
                <shadowcolor>$PARAM[shadowcolor]</shadowcolor>
                <font>$PARAM[font]</font>
                <visible>$PARAM[visible]</visible>
            </control>
        </definition>
    </include>
```

`top` and `visible` are risk-free: an absent `<top>` tag and an absent `<visible>` tag are equivalent to `<top></top>` (no offset) and `<visible>true</visible>` respectively, matching the new defaults exactly. `shadowcolor` defaulting to empty is the one point needing live confirmation in Step 4 below — flagged, not assumed.

- [ ] **Step 2: Repoint all 4 Flix call sites**

`Includes_Home.xml:1844-1854`, change:
```xml
                    <include content="Flix_Object_Info_Title">
                        <param name="label" value="$INFO[Container($PARAM[id]).ListItem.Title]" />
                        <param name="font" value="font_title_large" />
                        <param name="shadowcolor" value="ff000000" />
                        <param name="titleheight" value="$PARAM[titleheight]" />
                        <param name="top" value="80" />
                        <param name="visible" value="String.IsEmpty(Container($PARAM[id]).ListItem.Art(clearlogo))" />
```
to:
```xml
                    <include content="Object_Info_Title">
                        <param name="label" value="$INFO[Container($PARAM[id]).ListItem.Title]" />
                        <param name="font" value="font_title_large" />
                        <param name="shadowcolor" value="ff000000" />
                        <param name="titleheight" value="$PARAM[titleheight]" />
                        <param name="top" value="80" />
                        <param name="visible" value="String.IsEmpty(Container($PARAM[id]).ListItem.Art(clearlogo))" />
```
(only the `content="..."` name changes — every param stays identical.)

`Includes_View_52_Showcase.xml:230`, change `content="Flix_Object_Info_Title"` to `content="Object_Info_Title"` — params unchanged (`font`, `shadowcolor`, `titleheight`, `top`, `visible` all already explicit).

`Includes_View_52_Showcase.xml:313`, change `content="Flix_Object_Info_Title"` to `content="Object_Info_Title"` — params unchanged.

`Includes_View_52_Showcase.xml:370-374`, change:
```xml
                <include content="Flix_Object_Info_Title">
                    <param name="font" value="$PARAM[font]" />
                    <param name="titleheight" value="$PARAM[titleheight]" />
                    <param name="visible" value="!$EXP[Exp_HasClearlogo] | !Skin.HasSetting(ShowClearlogo)" />
                </include>
```
to:
```xml
                <include content="Object_Info_Title">
                    <param name="font" value="$PARAM[font]" />
                    <param name="titleheight" value="$PARAM[titleheight]" />
                    <param name="visible" value="!$EXP[Exp_HasClearlogo] | !Skin.HasSetting(ShowClearlogo)" />
                    <param name="shadowcolor" value="ff000000" />
                </include>
```
(this one gains an explicit `shadowcolor` it didn't need before, since it relied on Flix's now-removed default).

- [ ] **Step 3: Delete the Flix_Object_Info_Title definition**

Remove lines 1518-1538 of `Includes_Object.xml` (the entire `Flix_Object_Info_Title` include block, from `<include name="Flix_Object_Info_Title">` through its closing `</include>`).

- [ ] **Step 4: Validate XML on all 3 modified files**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
for f in Includes_Object.xml Includes_Home.xml Includes_View_52_Showcase.xml; do
  python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/$f'); print('$f OK')"
done
grep -rn "Flix_Object_Info_Title" "$SKIN"/1080i/*.xml
```
Expected: 3× `OK`, and the grep returns no output (confirms no dangling references to the deleted template).

- [ ] **Step 5: Restart Kodi, visually compare title rendering**

Check Aerial/screensaver first. After restart, visit: the Home screen's Flix-style widget rows (where `Includes_Home.xml`'s title renders), a Showcase-style content view (`Includes_View_52_Showcase.xml`), and a non-Flix view using `Object_Info_Title` directly — e.g. `Includes_View.xml:113`, reached via any standard video list view. Confirm titles render with the same font, size, color, shadow, and position as before this change in all of them. This is the one task in this plan where a shadow-rendering regression is plausible if the empty-string `shadowcolor` default doesn't behave as expected — if any non-Flix title unexpectedly gains a shadow, revert Step 1's `shadowcolor` addition and investigate before proceeding further.

### Task 8: Consolidate Object_Info_Plot / Flix_Object_Info_Plot

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Object.xml:1578-1593` (base template — add params)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Home.xml:1864-1868` (repoint 1 call site)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_View_52_Showcase.xml:245-249,334-338,342-347` (repoint 3 call sites)
- Delete (final step): `Flix_Object_Info_Plot` definition at `Includes_Object.xml:1556-1576`

**Interfaces:**
- Consumes: nothing new.
- Produces: `Object_Info_Plot` gains `shadowcolor` (default `""`), `width` (default `""`), `maxwidth` (default `""`) params, and its existing hardcoded `top` (10) and `textcolor` (`main_fg_70`) become overridable params with those same values as defaults.

**Verified per-call-site params (all 4 Flix_Object_Info_Plot callers):**
- `Includes_Home.xml:1864`: passes `label`, `maxheight=$PARAM[plotheight]`, `shadowcolor=ff000000`. Does not pass `top` (relies on Flix default 30) or `textcolor`/`width`/`maxwidth` (relies on Flix defaults `main_fg_100`/`auto`/`820`).
- `Includes_View_52_Showcase.xml:245`: passes `height=$PARAM[plotheight]`, `top=10`, `shadowcolor=ff000000`.
- `Includes_View_52_Showcase.xml:334`: passes `height=$PARAM[plotheight]`, `top=10`, `shadowcolor=ff000000`.
- `Includes_View_52_Showcase.xml:342`: passes `height=320`, `top=10`, `shadowcolor=ff000000`.

None of the 4 pass `textcolor`, `width`, or `maxwidth` explicitly — all rely on Flix's hardcoded/default `main_fg_100` / `auto` / `820`. All 4 need these three added explicitly when repointed, since the merged template's defaults will match base's current behavior (`main_fg_70`, no width constraint) instead.

**This is the highest-uncertainty task in this plan.** Base `Object_Info_Plot` currently has no `<width>` element at all (16 call sites across many dialogs rely on inherited/container width). Kodi's exact behavior for `<width max="">` with an empty value is not something that can be verified without live rendering. If Step 5's visual check shows ANY text-wrapping change in a non-Flix plot textbox, stop and revert this task rather than attempting further blind fixes.

- [ ] **Step 1: Add the missing params to the base template, making existing hardcoded values overridable**

Current content (`Includes_Object.xml:1578-1593`):

```xml
    <include name="Object_Info_Plot">
        <param name="maxheight" default="200" />
        <param name="height" default="auto" />
        <param name="label" default="$VAR[Label_Plot]" />
        <definition>
            <control type="textbox">
                <nested />
                <top>10</top>
                <label>$PARAM[label]</label>
                <height max="$PARAM[maxheight]">$PARAM[height]</height>
                <aligny>top</aligny>
                <textcolor>main_fg_70</textcolor>
                <font>font_plotbox</font>
            </control>
        </definition>
    </include>
```

Replace with:

```xml
    <include name="Object_Info_Plot">
        <param name="maxheight" default="200" />
        <param name="height" default="auto" />
        <param name="label" default="$VAR[Label_Plot]" />
        <param name="top" default="10" />
        <param name="textcolor" default="main_fg_70" />
        <param name="shadowcolor" default="" />
        <param name="width" default="" />
        <param name="maxwidth" default="" />
        <definition>
            <control type="textbox">
                <nested />
                <top>$PARAM[top]</top>
                <label>$PARAM[label]</label>
                <height max="$PARAM[maxheight]">$PARAM[height]</height>
                <width max="$PARAM[maxwidth]">$PARAM[width]</width>
                <aligny>top</aligny>
                <textcolor>$PARAM[textcolor]</textcolor>
                <shadowcolor>$PARAM[shadowcolor]</shadowcolor>
                <font>font_plotbox</font>
            </control>
        </definition>
    </include>
```

- [ ] **Step 2: Repoint all 4 Flix call sites, adding the params they implicitly relied on as defaults**

`Includes_Home.xml:1864-1868`, change:
```xml
                <include content="Flix_Object_Info_Plot">
                    <param name="label" value="$INFO[Container($PARAM[id]).ListItem.Plot]" />
                    <param name="maxheight" value="$PARAM[plotheight]" />
                    <param name="shadowcolor" value="ff000000" />
                </include>
```
to:
```xml
                <include content="Object_Info_Plot">
                    <param name="label" value="$INFO[Container($PARAM[id]).ListItem.Plot]" />
                    <param name="maxheight" value="$PARAM[plotheight]" />
                    <param name="shadowcolor" value="ff000000" />
                    <param name="top" value="30" />
                    <param name="textcolor" value="main_fg_100" />
                    <param name="width" value="auto" />
                    <param name="maxwidth" value="820" />
                </include>
```

`Includes_View_52_Showcase.xml:245-249`, change:
```xml
                <include content="Flix_Object_Info_Plot">
                    <param name="height" value="$PARAM[plotheight]" />
                    <param name="top" value="10" />
                    <param name="shadowcolor" value="ff000000" />
                </include>
```
to:
```xml
                <include content="Object_Info_Plot">
                    <param name="height" value="$PARAM[plotheight]" />
                    <param name="top" value="10" />
                    <param name="shadowcolor" value="ff000000" />
                    <param name="textcolor" value="main_fg_100" />
                    <param name="width" value="auto" />
                    <param name="maxwidth" value="820" />
                </include>
```

`Includes_View_52_Showcase.xml:334-338`, same substitution pattern (content name + add `textcolor`/`width`/`maxwidth`):
```xml
                    <include content="Object_Info_Plot">
                        <param name="height" value="$PARAM[plotheight]" />
                        <param name="top" value="10" />
                        <param name="shadowcolor" value="ff000000" />
                        <param name="textcolor" value="main_fg_100" />
                        <param name="width" value="auto" />
                        <param name="maxwidth" value="820" />
                    </include>
```

`Includes_View_52_Showcase.xml:342-347`, same pattern with `height=320`:
```xml
                    <include content="Object_Info_Plot">
                        <param name="height" value="320" />
                        <param name="top" value="10" />
                        <param name="shadowcolor" value="ff000000" />
                        <param name="textcolor" value="main_fg_100" />
                        <param name="width" value="auto" />
                        <param name="maxwidth" value="820" />
                    </include>
```

- [ ] **Step 3: Delete the Flix_Object_Info_Plot definition**

Remove lines 1556-1576 of `Includes_Object.xml` (the entire `Flix_Object_Info_Plot` include block).

- [ ] **Step 4: Validate XML**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
for f in Includes_Object.xml Includes_Home.xml Includes_View_52_Showcase.xml; do
  python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/$f'); print('$f OK')"
done
grep -rn "Flix_Object_Info_Plot" "$SKIN"/1080i/*.xml
```
Expected: 3× `OK`, grep returns no output.

- [ ] **Step 5: Restart Kodi, carefully visually compare plot text rendering**

Check Aerial/screensaver first. Visit a Flix-style Home widget row and a Showcase view (both should render plot text identically to before — bright, shadowed, width-capped at 820). Then visit a **non-Flix** plot rendering — e.g. `Includes_DialogVideoInfo.xml:394`'s video info dialog plot text, or a standard video list's info panel via `Includes_View.xml:121/125`. Confirm the plot text wraps at the same width and displays with the same (dimmer, no-shadow) appearance as before. This is the specific check flagged as highest-risk in this task's header — do not skip it.

### Task 9: Consolidate Object_Info_Line / Flix_Object_Info_Line (and delete Flix_Object_Info_Line_Label)

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Object.xml:1329-1467` (base template — add percentrating pill + green param)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Home.xml:1859-1863` (repoint 1 call site)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_View_52_Showcase.xml:329-331` (repoint 1 call site)
- Delete (final step): `Flix_Object_Info_Line` (`Includes_Object.xml:1198-1327`) and `Flix_Object_Info_Line_Label` (`Includes_Object.xml:1121-1159`, added-to but not yet deleted in Task 6)

**Interfaces:**
- Consumes: `Object_Info_Line_Label` (as modified in Task 6, which must land first).
- Produces: `Object_Info_Line` gains `percentrating` (default `"false"`) and `green` (default `"ff27ae60"`) params, adding the match% pill as an opt-in first item.

**Verified real content differences (not just missing params):**
1. Flix has a "match% pill" first item (percentrating-gated) that base completely lacks.
2. Base has a `fullgenre`-driven choice between two genre-display items (`uid11`/`uid12`) that Flix lacks — Flix always shows the `uid11` variant (`Window(Home).Property(TMDbHelper.ListItem.base_genre)`) unconditionally when not disabled.
3. Base has two season/episode-number items (`uid13`/`uid14`) gated by `Window.IsActive(movieinformation)` that Flix lacks — but since that condition only matches the movie-info dialog context, including them in the merged template is inert for Flix's Home-widget/Showcase callers (they're never in that window).

Only 2 Flix_Object_Info_Line call sites exist: `Includes_Home.xml:1859` (passes `container`, `uid`, `maxheight` — note `maxheight` isn't a real param of either current template; this appears to be a pre-existing latent inconsistency unrelated to this cleanup, and is preserved as-is rather than "fixed" since that's out of this plan's scope) and `Includes_View_52_Showcase.xml:329` (passes only `percentrating=false`).

- [ ] **Step 1: Add the percentrating pill and green param to the base template**

Current content (`Includes_Object.xml:1329-1336`, the param declarations and opening of the definition):

```xml
    <include name="Object_Info_Line">
        <param name="label" default="$VAR[Label_InfoLine_01]" />
        <param name="hdsd" default="Skin.HasSetting(Hide_Codecs)" />
        <param name="nextaired" default="true" />
        <param name="textcolor" default="main_fg_100" />
        <param name="container" default="" />
        <param name="uid" default="661" />
        <param name="fullgenre" default="false" />
        <definition>
            <control type="grouplist">
                <height>40</height>
                <orientation>horizontal</orientation>
                <usecontrolcoords>true</usecontrolcoords>
                <itemgap>0</itemgap>
                <left>0</left>
                <right>0</right>
                <!-- First Items -->
                <!-- Set a fallback so divider is not first item if blank -->
                <include content="Object_Info_Line_Label">
```

Replace the param block and add the pill immediately before the existing first `<include content="Object_Info_Line_Label">` (the one setting `Year`):

```xml
    <include name="Object_Info_Line">
        <param name="label" default="$VAR[Label_InfoLine_01]" />
        <param name="hdsd" default="Skin.HasSetting(Hide_Codecs)" />
        <param name="nextaired" default="true" />
        <param name="textcolor" default="main_fg_100" />
        <param name="container" default="" />
        <param name="uid" default="661" />
        <param name="fullgenre" default="false" />
        <param name="percentrating" default="false" />
        <param name="green" default="ff27ae60" />
        <definition>
            <control type="grouplist">
                <height>40</height>
                <orientation>horizontal</orientation>
                <usecontrolcoords>true</usecontrolcoords>
                <itemgap>0</itemgap>
                <left>0</left>
                <right>0</right>
                <!-- First Items -->
                <!-- Set a fallback so divider is not first item if blank -->
                <include content="Object_Info_Line_Label" condition="!Skin.HasSetting(Infoline.DisableMatch)">
                    <param name="label" value="$VAR[Def_Percentage_Rating]% $LOCALIZE[31589]   " />
                    <param name="font" value="font_title_mini" />
                    <param name="textcolor" value="$PARAM[green]" />
                    <param name="divider" value="false" />
                    <param name="visible" value="!String.IsEmpty(ListItem.Rating) + $PARAM[percentrating]" />
                </include>
                <include content="Object_Info_Line_Label">
```

(This is the exact block from `Flix_Object_Info_Line` lines 1218-1224, unchanged, inserted as the new first item. Its `visible` condition already includes `$PARAM[percentrating]`, so it renders nothing when the param is `"false"` — the default for every existing base caller.)

- [ ] **Step 2: Repoint the 2 Flix_Object_Info_Line call sites**

`Includes_Home.xml:1859-1863`, change:
```xml
                <include content="Flix_Object_Info_Line">
                    <param name="container" value="Container($PARAM[id])." />
                    <param name="uid" value="$PARAM[uid]" />
                    <param name="maxheight" value="$PARAM[plotheight]" />
                </include>
```
to:
```xml
                <include content="Object_Info_Line">
                    <param name="container" value="Container($PARAM[id])." />
                    <param name="uid" value="$PARAM[uid]" />
                    <param name="maxheight" value="$PARAM[plotheight]" />
                    <param name="percentrating" value="true" />
                </include>
```

`Includes_View_52_Showcase.xml:329-331`, change:
```xml
                <include content="Flix_Object_Info_Line">
                    <param name="percentrating" value="false" />
                </include>
```
to:
```xml
                <include content="Object_Info_Line">
                    <param name="percentrating" value="false" />
                </include>
```

- [ ] **Step 3: Delete Flix_Object_Info_Line and Flix_Object_Info_Line_Label**

Remove lines 1198-1327 of `Includes_Object.xml` (`Flix_Object_Info_Line`) and lines 1121-1159 (`Flix_Object_Info_Line_Label`, added-to in Task 6 but never deleted until now since it was still in use).

- [ ] **Step 4: Validate XML**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
for f in Includes_Object.xml Includes_Home.xml Includes_View_52_Showcase.xml; do
  python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/$f'); print('$f OK')"
done
grep -rn "Flix_Object_Info_Line" "$SKIN"/1080i/*.xml
```
Expected: 3× `OK`, grep returns no output (note: this also catches `Flix_Object_Info_Line_Label` since it's a substring match).

- [ ] **Step 5: Restart Kodi, visually compare info-line rendering**

Check Aerial/screensaver first. Visit the same Home widget row and Showcase view checked in Task 7/8 — confirm the info line (year/premiered/MPAA/genre/duration/etc., plus the match% pill on the Home widget specifically) renders identically to before. Visit a non-Flix view using `Object_Info_Line` directly (e.g. `Includes_View_50_List.xml:429`) and confirm no match% pill appears there (it shouldn't, since `percentrating` defaults to `false`) and genre/season-episode display is unchanged.

### Task 10: Consolidate Items_Settings_RatingsMovies / Items_Settings_RatingsTVShows

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Items.xml:2862-2962` (replace both with one parameterized template)
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1150_Dialog.xml:56-57` (call sites)

**Interfaces:**
- Consumes: existing `Dialog_Standard_Button` include, existing `$VAR[Label_CustomRating_*]` variables (unchanged), existing window 1118 (`Custom_1118_EnableRatings.xml`, confirmed live).
- Produces: `Items_Settings_Ratings_Template`, taking one param `contenttype` (`Movies` or `TVShows`).

- [ ] **Step 1: Read current call sites**

`Custom_1150_Dialog.xml:56-57` currently reads:
```xml
                <include condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsMovies)">Items_Settings_RatingsMovies</include>
                <include condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsTVShows)">Items_Settings_RatingsTVShows</include>
```

- [ ] **Step 2: Replace the two duplicate templates with one parameterized template**

In `Includes_Items.xml`, replace lines 2862-2962 (both `Items_Settings_RatingsMovies` and `Items_Settings_RatingsTVShows` in full) with:

```xml
    <include name="Items_Settings_Ratings_Template">
        <param name="contenttype" default="Movies" />
        <definition>
            <control type="button" id="9001">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 01</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_01]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item01,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
            <control type="button" id="9002">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 02</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_02]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item02,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
            <control type="button" id="9003">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 03</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_03]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item03,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
            <control type="button" id="9004">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 04</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_04]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item04,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
            <control type="button" id="9005">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 05</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_05]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item05,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
            <control type="button" id="9006">
                <include>Dialog_Standard_Button</include>
                <label>$LOCALIZE[563] 06</label>
                <label2>$VAR[Label_CustomRating_$PARAM[contenttype]_06]</label2>
                <onclick>SetProperty(ConfigureRatingItem,Item06,Home)</onclick>
                <onclick>SetProperty(ConfigureRatingContent,$PARAM[contenttype],Home)</onclick>
                <onclick>ActivateWindow(1118)</onclick>
            </control>
        </definition>
    </include>
```

- [ ] **Step 3: Update the call sites**

In `Custom_1150_Dialog.xml`, replace lines 56-57:
```xml
                <include condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsMovies)">Items_Settings_RatingsMovies</include>
                <include condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsTVShows)">Items_Settings_RatingsTVShows</include>
```
with:
```xml
                <include content="Items_Settings_Ratings_Template" condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsMovies)">
                    <param name="contenttype" value="Movies" />
                </include>
                <include content="Items_Settings_Ratings_Template" condition="String.IsEqual(Window(home).Property(CustomDialogWindow.items),Settings_RatingsTVShows)">
                    <param name="contenttype" value="TVShows" />
                </include>
```

- [ ] **Step 4: Validate XML**

```bash
SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/Includes_Items.xml'); print('OK')"
python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/Custom_1150_Dialog.xml'); print('OK')"
grep -rn "Items_Settings_RatingsMovies\b\|Items_Settings_RatingsTVShows\b" "$SKIN"/1080i/*.xml
```
Expected: 2× `OK`, grep returns no output (both old names fully removed).

- [ ] **Step 5: Restart Kodi, verify both rating dialogs**

Check Aerial/screensaver first. Open the Movies custom-rating configuration dialog and the TV Shows one (both reached via `Custom_1150_Dialog.xml` with the relevant `CustomDialogWindow.items` property set — check `Custom_1118_EnableRatings.xml` or wherever `Window(home).Property(CustomDialogWindow.items)` gets set to `Settings_RatingsMovies`/`Settings_RatingsTVShows` if the exact navigation path isn't obvious). Confirm both show 6 buttons with the correct Movies/TVShows-specific labels, and that clicking a button still opens window 1118 with the right content type configured (check `Window(Home).Property(ConfigureRatingContent)` reflects `Movies` or `TVShows` correctly, e.g. via `Skin.SetString`/debug or by observing which rating gets configured).

---

## Part 3 — TMDb Helper: remove dead/orphaned code

### Task 11: Remove dead commented-out code blocks

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/database/baseitem_factories/factory.py:9-17`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/update/builder/tags.py:1-4`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/api/tmdb/users.py:4`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/database/baseview_factories/factory.py:1,5`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — pure deletion of inert comments, zero behavior change.

- [ ] **Step 1: factory.py — remove the dead FIXME/commented function**

Current (`baseitem_factories/factory.py:1-19`):
```python
"""
BASEITEM FACTORY

Used to sync detailed data about item mediatype with tmdb_id to cache and then return configured data about item

"""


# FIXME IMAGES
"""
def finalise_image():
    item['infolabels']['title'] = f'{item["infoproperties"].get("width")}x{item["infoproperties"].get("height")}'
    item['params'] = -1
    item['path'] = item['art'].get('thumb') or item['art'].get('poster') or item['art'].get('fanart')
    item['is_folder'] = False
    item['library'] = 'pictures'
"""


def import_movie():
```

Replace with:
```python
"""
BASEITEM FACTORY

Used to sync detailed data about item mediatype with tmdb_id to cache and then return configured data about item

"""


def import_movie():
```

- [ ] **Step 2: tags.py — remove the 4 dead commented imports**

Current (`update/builder/tags.py:1-8`):
```python
# from tmdbhelper.lib.addon.logger import kodi_log
# from tmdbhelper.lib.addon.plugin import get_localized
# from tmdbhelper.lib.api.kodi.rpc import set_tags
# from tmdbhelper.lib.update.common import LibraryCommon
from jurialmunkey.ftools import cached_property
from tmdbhelper.lib.api.kodi.rpc import set_tags
from tmdbhelper.lib.update.builder.userlist import LibraryBuilderUserList
from tmdbhelper.lib.update.builder.media import LibraryBuilder
```
Replace with:
```python
from jurialmunkey.ftools import cached_property
from tmdbhelper.lib.api.kodi.rpc import set_tags
from tmdbhelper.lib.update.builder.userlist import LibraryBuilderUserList
from tmdbhelper.lib.update.builder.media import LibraryBuilder
```

- [ ] **Step 3: users.py — remove the 1 dead commented import**

Current (`api/tmdb/users.py:1-5`):
```python
from tmdbhelper.lib.api.tmdb.api import TMDbAPI, TMDb
from tmdbhelper.lib.api.tmdb.userauthenticator import TMDbUserAuthenticator
from jurialmunkey.ftools import cached_property
# from tmdbhelper.lib.addon.logger import kodi_log

```
Replace with:
```python
from tmdbhelper.lib.api.tmdb.api import TMDbAPI, TMDb
from tmdbhelper.lib.api.tmdb.userauthenticator import TMDbUserAuthenticator
from jurialmunkey.ftools import cached_property

```

- [ ] **Step 4: baseview_factories/factory.py — remove the 2 dead commented imports**

Current (`items/database/baseview_factories/factory.py:1-10`):
```python
# from jurialmunkey.ftools import cached_property
from tmdbhelper.lib.addon.plugin import convert_type
from jurialmunkey.modimp import importmodule
from jurialmunkey.parser import try_int
# from tmdbhelper.lib.addon.logger import kodi_log

"""
BASEVIEW FACTORY

Viewlists to retrieve data from database as a directory of listitems (e.g. cast members list for movie)
```
Replace with:
```python
from tmdbhelper.lib.addon.plugin import convert_type
from jurialmunkey.modimp import importmodule
from jurialmunkey.parser import try_int

"""
BASEVIEW FACTORY

Viewlists to retrieve data from database as a directory of listitems (e.g. cast members list for movie)
```

- [ ] **Step 5: Compile-check all 4 files**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile \
  resources/tmdbhelper/lib/items/database/baseitem_factories/factory.py \
  resources/tmdbhelper/lib/update/builder/tags.py \
  resources/tmdbhelper/lib/api/tmdb/users.py \
  resources/tmdbhelper/lib/items/database/baseview_factories/factory.py
echo "compiled OK"
```
Expected: `compiled OK`, no errors.

- [ ] **Step 6: Commit**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
git add resources/tmdbhelper/lib/items/database/baseitem_factories/factory.py \
  resources/tmdbhelper/lib/update/builder/tags.py \
  resources/tmdbhelper/lib/api/tmdb/users.py \
  resources/tmdbhelper/lib/items/database/baseview_factories/factory.py
git commit -m "$(cat <<'EOF'
Remove dead commented-out code blocks

Four spots with fully commented-out code left over from earlier
refactors: a disabled finalise_image() function under a stale
"FIXME IMAGES" marker, and three files with dead commented-out import
lines whose functionality was already superseded by the active imports
alongside them. No behavior change, pure deletion.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 12: Remove redundant local imports

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/trakt/lists_random.py:39,52`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/script/method/tmdb.py:20`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/api/trakt/sync/datatype.py:477`
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/tmdb/lists_discodir.py:47`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — each of these names is already imported at module level in the same file; the local re-import is dead weight.

- [ ] **Step 1: lists_random.py — remove both redundant `import random` lines**

Line 39 currently reads (inside `sorted_items`):
```python
    @cached_property
    def sorted_items(self):
        import random
        return random.sample(self.filtered_items, self.sample_limit)
```
Change to:
```python
    @cached_property
    def sorted_items(self):
        return random.sample(self.filtered_items, self.sample_limit)
```

Line 52 currently reads (inside `ListTraktRandomised.get_items`):
```python
        if tmdb_type == 'both':
            import random
            items = []
```
Change to:
```python
        if tmdb_type == 'both':
            items = []
```
(Module-level `import random` at line 1 already covers both — confirmed via `grep -n "^import random" resources/tmdbhelper/lib/items/directories/trakt/lists_random.py`.)

- [ ] **Step 2: script/method/tmdb.py — remove the redundant `get_localized` from one import line**

Line 20 currently reads:
```python
    from tmdbhelper.lib.addon.plugin import get_localized, convert_type
```
Change to:
```python
    from tmdbhelper.lib.addon.plugin import convert_type
```
(`get_localized` is already imported at module level, line 5. `convert_type` is genuinely new to this function and stays.)

- [ ] **Step 3: datatype.py — remove the redundant ParallelThread import line entirely**

Line 477 currently reads (inside `item_queue`):
```python
    @cached_property
    def item_queue(self):
        self.main.dialog_progress_bg.max_value = len(self.sd.items)
        from tmdbhelper.lib.addon.thread import ParallelThread
        with ParallelThread(self.sd.items, self.get_items) as pt:
```
Change to:
```python
    @cached_property
    def item_queue(self):
        self.main.dialog_progress_bg.max_value = len(self.sd.items)
        with ParallelThread(self.sd.items, self.get_items) as pt:
```
(Module-level `from tmdbhelper.lib.addon.thread import ParallelThread` at line 7 already covers this.)

- [ ] **Step 4: lists_discodir.py — remove the redundant get_property import line entirely**

Line 47 currently reads (inside `item_search`):
```python
    @property
    def item_search(self):
        from jurialmunkey.window import get_property
        params = self.get_winprop_params()
```
Change to:
```python
    @property
    def item_search(self):
        params = self.get_winprop_params()
```
(Module-level `from jurialmunkey.window import get_property` at line 3 already covers this.)

- [ ] **Step 5: Compile-check all 4 files**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile \
  resources/tmdbhelper/lib/items/directories/trakt/lists_random.py \
  resources/tmdbhelper/lib/script/method/tmdb.py \
  resources/tmdbhelper/lib/api/trakt/sync/datatype.py \
  resources/tmdbhelper/lib/items/directories/tmdb/lists_discodir.py
echo "compiled OK"
```
Expected: `compiled OK`.

- [ ] **Step 6: Commit**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
git add resources/tmdbhelper/lib/items/directories/trakt/lists_random.py \
  resources/tmdbhelper/lib/script/method/tmdb.py \
  resources/tmdbhelper/lib/api/trakt/sync/datatype.py \
  resources/tmdbhelper/lib/items/directories/tmdb/lists_discodir.py
git commit -m "$(cat <<'EOF'
Remove redundant local imports already covered at module level

Four spots re-import a name inside a function/method body that's
already imported at the top of the same file (random x2 in
lists_random.py, get_localized in script/method/tmdb.py -- kept
convert_type on that line since it's genuinely new there, ParallelThread
in datatype.py, get_property in lists_discodir.py). Zero behavior
change.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 13: Delete orphaned module api/trakt/items.py

**Files:**
- Delete: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/api/trakt/items.py`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — confirmed zero importers anywhere in the repo.

- [ ] **Step 1: Final confirmation of zero importers**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
grep -rn "trakt\.items\b\|trakt import items\b\|from tmdbhelper.lib.api.trakt.items" resources/ --include="*.py"
```
Expected: no output.

- [ ] **Step 2: Delete the file**

```bash
git rm resources/tmdbhelper/lib/api/trakt/items.py
```

- [ ] **Step 3: Compile-check the whole tree to catch any missed import**

```bash
python3 -m compileall -q resources/tmdbhelper/lib
echo "exit code: $?"
```
Expected: `exit code: 0`, no output before it (compileall is silent on success).

- [ ] **Step 4: Commit**

```bash
git commit -m "$(cat <<'EOF'
Remove orphaned api/trakt/items.py

TraktItems and its module-level helpers (_sort_itemlist, EPISODE_PARAMS,
SEASON_PARAMS, etc.) are never imported anywhere in the repo -- confirmed
via grep for both the module path and the class name. Superseded by the
ItemListSyncDataFactory architecture in api/trakt/sync/itemlist.py.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 14: Delete orphaned classes

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/trakt/lists_sync.py:140-148` (remove `ListPlaybackProgress`)
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/api/trakt/sync/datasync.py:30-50,114-131` (remove `SyncDataSetters`, `SyncDataGetterProgressCollectedUnHidden`, `SyncDataGetterCalendarUnHidden`, `SyncDataGetterDroppedCollectionUnHidden`, `SyncDataGetterDroppedCalendarUnHidden`)
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/files/bcache.py:17-18` (remove `BasicCacheService`)

**Interfaces:**
- Consumes: nothing.
- Produces: nothing — each class confirmed to have zero subclasses/references anywhere outside its own definition.

- [ ] **Step 1: Final confirmation of zero references for each class**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
for cls in "ListPlaybackProgress" "SyncDataSetters" "SyncDataGetterProgressCollectedUnHidden" "SyncDataGetterCalendarUnHidden" "SyncDataGetterDroppedCollectionUnHidden" "SyncDataGetterDroppedCalendarUnHidden" "BasicCacheService"; do
  echo "--- $cls ---"
  grep -rn "$cls" resources/ --include="*.py"
done
```
Expected: for each class, the only hit is its own `class $cls` definition line — no other references (no instantiation, no subclassing, no import elsewhere).

- [ ] **Step 2: Remove ListPlaybackProgress from lists_sync.py**

Current (`items/directories/trakt/lists_sync.py:138-151`):
```python
    list_properties.plugin_name = '{localized} {plural}'
        return list_properties


class ListPlaybackProgress(ListStandardSync):
    def configure_list_properties(self, list_properties):
        list_properties = super().configure_list_properties(list_properties)
        list_properties.sync_type = 'playback'
        list_properties.sort_by = 'paused'
        list_properties.sort_how = 'desc'
        list_properties.localize = 32196
        list_properties.plugin_name = '{localized} {plural}'
        return list_properties


class ListFavorites(ListStandardSync):
```
Change to (delete the entire `ListPlaybackProgress` class, keep one blank-line gap between the surrounding classes):
```python
    list_properties.plugin_name = '{localized} {plural}'
        return list_properties


class ListFavorites(ListStandardSync):
```

- [ ] **Step 3: Remove SyncDataSetters from datasync.py**

Current (`api/trakt/sync/datasync.py:28-53`):
```python

class SyncDataSetters:
    """ Add-in class to group setter methods for SyncData class """

    def like_userlist(self, user_slug=None, list_slug=None, confirmation=False, delete=False):
        from tmdbhelper.lib.addon.plugin import get_localized
        func = self.delete_response if delete else self.post_response
        response = func('users', user_slug, 'lists', list_slug, 'like')
        if confirmation:
            from xbmcgui import Dialog
            affix = get_localized(32320) if delete else get_localized(32321)
            body = [
                get_localized(32316).format(affix),
                get_localized(32168).format(list_slug, user_slug)
            ] if response.status_code == 204 else [
                get_localized(32317).format(affix),
                get_localized(32168).format(list_slug, user_slug),
                get_localized(32318).format(response.status_code)
            ]
            Dialog().ok(get_localized(32315), '\n'.join(body))
        if response.status_code == 204:
            return response


class SyncDataGetterAll:
```
Change to (delete the entire `SyncDataSetters` class):
```python

class SyncDataGetterAll:
```

- [ ] **Step 4: Remove the 4 orphaned SyncDataGetter classes from datasync.py**

Current (`api/trakt/sync/datasync.py:110-132`, after Step 3's edit has shifted line numbers — re-grep before editing to confirm current positions):
```python
class SyncDataGetterProgressWatchedUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'progress_watched_hidden_at IS NULL', )


class SyncDataGetterProgressCollectedUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'progress_collected_hidden_at IS NULL', )


class SyncDataGetterCalendarUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'calendar_hidden_at IS NULL', )


class SyncDataGetterDroppedWatchedUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'dropped_hidden_at IS NULL AND progress_watched_hidden_at IS NULL', )


class SyncDataGetterDroppedCollectionUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'dropped_hidden_at IS NULL AND progress_collected_hidden_at IS NULL', )


class SyncDataGetterDroppedCalendarUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'dropped_hidden_at IS NULL AND calendar_hidden_at IS NULL', )


class SyncDataGetterAllUnHiddenMoviesToWatch(SyncDataGetterProgressWatchedUnHidden):
```
Change to (delete `SyncDataGetterProgressCollectedUnHidden`, `SyncDataGetterCalendarUnHidden`, `SyncDataGetterDroppedCollectionUnHidden`, `SyncDataGetterDroppedCalendarUnHidden` — keep `SyncDataGetterProgressWatchedUnHidden` and `SyncDataGetterDroppedWatchedUnHidden`, both of which ARE subclassed elsewhere):
```python
class SyncDataGetterProgressWatchedUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'progress_watched_hidden_at IS NULL', )


class SyncDataGetterDroppedWatchedUnHidden(SyncDataGetterAll):
    query_clauses = ('item_type=?', 'dropped_hidden_at IS NULL AND progress_watched_hidden_at IS NULL', )


class SyncDataGetterAllUnHiddenMoviesToWatch(SyncDataGetterProgressWatchedUnHidden):
```

- [ ] **Step 5: Remove BasicCacheService from bcache.py**

Current (`files/bcache.py:8-19`):
```python
class BasicCache(jurialmunkey.bcache.BasicCache):
    _queue_limit = 250
    _simplecache = SimpleCache

    @staticmethod
    def kodi_traceback(exc, log_msg):
        kodi_traceback(exc, log_msg)


class BasicCacheService(BasicCache):
    _queue_limit = 20
```
Change to:
```python
class BasicCache(jurialmunkey.bcache.BasicCache):
    _queue_limit = 250
    _simplecache = SimpleCache

    @staticmethod
    def kodi_traceback(exc, log_msg):
        kodi_traceback(exc, log_msg)
```

- [ ] **Step 6: Compile-check and re-verify no dangling references**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile \
  resources/tmdbhelper/lib/items/directories/trakt/lists_sync.py \
  resources/tmdbhelper/lib/api/trakt/sync/datasync.py \
  resources/tmdbhelper/lib/files/bcache.py
python3 -m compileall -q resources/tmdbhelper/lib
echo "exit code: $?"
for cls in "ListPlaybackProgress" "SyncDataSetters" "SyncDataGetterProgressCollectedUnHidden" "SyncDataGetterCalendarUnHidden" "SyncDataGetterDroppedCollectionUnHidden" "SyncDataGetterDroppedCalendarUnHidden" "BasicCacheService"; do
  grep -rn "$cls" resources/ --include="*.py"
done
```
Expected: `exit code: 0`, no output from the final grep loop (all 7 names fully gone).

- [ ] **Step 7: Commit**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
git add resources/tmdbhelper/lib/items/directories/trakt/lists_sync.py \
  resources/tmdbhelper/lib/api/trakt/sync/datasync.py \
  resources/tmdbhelper/lib/files/bcache.py
git commit -m "$(cat <<'EOF'
Remove orphaned classes with zero references

ListPlaybackProgress (lists_sync.py) is missing from the route table in
addon/consts.py that its siblings ListHistory/ListMostWatched use.
SyncDataSetters (datasync.py) is never mixed into SyncData, which only
inherits SyncDataGetters. SyncDataGetterProgressCollectedUnHidden,
SyncDataGetterCalendarUnHidden, SyncDataGetterDroppedCollectionUnHidden,
and SyncDataGetterDroppedCalendarUnHidden (datasync.py) have no
subclasses, unlike their siblings SyncDataGetterProgressWatchedUnHidden
and SyncDataGetterDroppedWatchedUnHidden which are kept.
BasicCacheService (bcache.py) has zero references anywhere. All
confirmed via grep before removal.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

- [ ] **Step 8: Restart Kodi, exercise Trakt sync**

Since `lists_sync.py` and `datasync.py` are on the Trakt sync hot path, restart Kodi and confirm Trakt history/most-watched/favorites lists still load normally, and that a manual Trakt sync (if one is configured) still completes without new errors in `kodi.log`.

---

## Part 4 — TMDb Helper: dedupe lists_view_db.py

### Task 15: Collapse the Fanart/Poster/Image/Thumb class family

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py:55-80`

**Interfaces:**
- Consumes: `BaseViewFactory`, `convert_type` (both already imported at the top of the file).
- Produces: `ListImageViewBase` (new shared base), with `ListFanart`/`ListPoster`/`ListImage`/`ListThumb` as thin subclasses — same class names, same external behavior, so no route-table changes needed anywhere else.

- [ ] **Step 1: Read current content**

Current (lines 55-80):
```python
class ListFanart(ContainerCacheOnlyDirectory):
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, **kwargs):
        sync = BaseViewFactory('fanart', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('image', 'container')
        return sync.data


class ListPoster(ContainerCacheOnlyDirectory):
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, **kwargs):
        sync = BaseViewFactory('poster', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('image', 'container')
        return sync.data


class ListImage(ContainerCacheOnlyDirectory):
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, **kwargs):
        sync = BaseViewFactory('image', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('image', 'container')
        return sync.data


class ListThumb(ContainerCacheOnlyDirectory):
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, **kwargs):
        sync = BaseViewFactory('thumb', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('image', 'container')
        return sync.data
```

- [ ] **Step 2: Replace with a shared base + thin subclasses**

```python
class ListImageViewBase(ContainerCacheOnlyDirectory):
    view_name = None

    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, **kwargs):
        sync = BaseViewFactory(self.view_name, tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('image', 'container')
        return sync.data


class ListFanart(ListImageViewBase):
    view_name = 'fanart'


class ListPoster(ListImageViewBase):
    view_name = 'poster'


class ListImage(ListImageViewBase):
    view_name = 'image'


class ListThumb(ListImageViewBase):
    view_name = 'thumb'
```

- [ ] **Step 3: Confirm the route table still resolves these class names correctly**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
grep -n "ListFanart\|ListPoster\|ListImage\b\|ListThumb" resources/tmdbhelper/lib/addon/consts.py
```
Expected: shows the existing route entries referencing these class names by string — confirms the class-name-preserving refactor requires no changes there.

- [ ] **Step 4: Compile-check**

```bash
python3 -m py_compile resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
echo "compiled OK"
```
Expected: `compiled OK`.

- [ ] **Step 5: Commit**

```bash
git add resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
git commit -m "$(cat <<'EOF'
lists_view_db.py: collapse Fanart/Poster/Image/Thumb into shared base

These four classes were byte-identical except for the view-name string
passed to BaseViewFactory. Introduces ListImageViewBase with a
view_name class attribute; the four public classes keep their exact
names (route table in addon/consts.py references them unchanged) and
now just set one attribute each. No behavior change.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 16: Collapse the Cast/Crew class family

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py:83-98` (line numbers shift after Task 15 — re-grep before editing)

**Interfaces:**
- Consumes: `ListConfigureOffset` (already defined earlier in the same file, unchanged).
- Produces: `ListPersonListViewBase`, with `ListCast`/`ListCrew` as thin subclasses.

- [ ] **Step 1: Read current content**

```python
class ListCast(ContainerCacheOnlyDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('castmember', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('person', 'container')
        return sync.data


class ListCrew(ContainerCacheOnlyDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('crewmember', tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('person', 'container')
        return sync.data
```

- [ ] **Step 2: Replace with a shared base + thin subclasses**

```python
class ListPersonListViewBase(ContainerCacheOnlyDirectory):
    view_name = None

    @ListConfigureOffset
    def get_items(self, tmdb_id, tmdb_type, season=None, episode=None, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory(self.view_name, tmdb_type, tmdb_id, season, episode, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.container_content = convert_type('person', 'container')
        return sync.data


class ListCast(ListPersonListViewBase):
    view_name = 'castmember'


class ListCrew(ListPersonListViewBase):
    view_name = 'crewmember'
```

- [ ] **Step 3: Compile-check**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
echo "compiled OK"
```
Expected: `compiled OK`.

- [ ] **Step 4: Commit**

```bash
git add resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
git commit -m "$(cat <<'EOF'
lists_view_db.py: collapse Cast/Crew into shared base

ListCast and ListCrew were identical except for the view-name string
('castmember'/'crewmember') passed to BaseViewFactory. Introduces
ListPersonListViewBase carrying the @ListConfigureOffset-decorated
get_items; both public classes keep their exact names and now just set
view_name. No behavior change.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 17: Collapse the Series/Starred*/Crewed* class family

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py` (line numbers shift after Tasks 15-16 — re-grep before editing; originally lines 101-128 and 146-163)

**Interfaces:**
- Consumes: `ListConfigureOffset`, `get_kodi_database` (inherited from `ContainerDefaultCacheDirectory`, unchanged).
- Produces: `ListPersonOrCollectionViewBase`, with `ListSeries`/`ListStarredMovies`/`ListStarredTvshows`/`ListCrewedMovies`/`ListCrewedTvshows` as thin subclasses.

- [ ] **Step 1: Read current content**

```python
class ListSeries(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('seriesmovies', 'collection', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database('movie')
        self.container_content = convert_type('movie', 'container')
        return sync.data


class ListStarredMovies(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('starredmovies', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database('movie')
        self.container_content = convert_type('movie', 'container')
        return sync.data


class ListStarredTvshows(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('starredtvshows', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database('tv')
        self.container_content = convert_type('tv', 'container')
        return sync.data
```
and (originally lines 146-163, now further down after `ListStarredCombined` which stays untouched until Task 18):
```python
class ListCrewedMovies(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('crewedmovies', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database('movie')
        self.container_content = convert_type('movie', 'container')
        return sync.data


class ListCrewedTvshows(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('crewedtvshows', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database('tv')
        self.container_content = convert_type('tv', 'container')
        return sync.data
```

- [ ] **Step 2: Replace ListSeries/ListStarredMovies/ListStarredTvshows with a shared base + thin subclasses**

Replace the `ListSeries`/`ListStarredMovies`/`ListStarredTvshows` block with:
```python
class ListPersonOrCollectionViewBase(ContainerDefaultCacheDirectory):
    view_name = None
    base_tmdb_type = 'person'
    kodi_db_type = 'movie'
    content_type = 'movie'

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory(self.view_name, self.base_tmdb_type, tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        self.kodi_db = self.get_kodi_database(self.kodi_db_type)
        self.container_content = convert_type(self.content_type, 'container')
        return sync.data


class ListSeries(ListPersonOrCollectionViewBase):
    view_name = 'seriesmovies'
    base_tmdb_type = 'collection'


class ListStarredMovies(ListPersonOrCollectionViewBase):
    view_name = 'starredmovies'


class ListStarredTvshows(ListPersonOrCollectionViewBase):
    view_name = 'starredtvshows'
    kodi_db_type = 'tv'
    content_type = 'tv'
```

- [ ] **Step 3: Replace ListCrewedMovies/ListCrewedTvshows (further down the file) with thin subclasses of the same base**

```python
class ListCrewedMovies(ListPersonOrCollectionViewBase):
    view_name = 'crewedmovies'


class ListCrewedTvshows(ListPersonOrCollectionViewBase):
    view_name = 'crewedtvshows'
    kodi_db_type = 'tv'
    content_type = 'tv'
```

- [ ] **Step 4: Compile-check**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
echo "compiled OK"
```
Expected: `compiled OK`.

- [ ] **Step 5: Commit**

```bash
git add resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
git commit -m "$(cat <<'EOF'
lists_view_db.py: collapse Series/Starred*/Crewed* into shared base

ListSeries, ListStarredMovies, ListStarredTvshows, ListCrewedMovies,
and ListCrewedTvshows shared the same BaseViewFactory + get_kodi_database
+ container_content shape, differing only in 2-3 string attributes.
Introduces ListPersonOrCollectionViewBase; all five public classes keep
their exact names and now just set class attributes. No behavior
change.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

### Task 18: Collapse the Combined class family

**Files:**
- Modify: `/home/frankie/GitHub/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py` (line numbers shift after Tasks 15-17 — re-grep before editing; originally `ListStarredCombined` lines 131-144, `ListCrewedCombined` lines 166-179, `ListCreditsCombined` lines 181-196)

**Interfaces:**
- Consumes: nothing new.
- Produces: `ListCombinedViewBase`, with `ListStarredCombined`/`ListCrewedCombined`/`ListCreditsCombined` as thin subclasses.

- [ ] **Step 1: Read current content**

```python
class ListStarredCombined(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('starredcombined', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        try:
            movie_count = len([i for i in sync.data if i and i['infoproperties'].get('tmdb_type') == 'movie'])
            shows_count = len(sync.data) - movie_count
        except TypeError:
            return
        self.kodi_db = self.get_kodi_database('both')
        self.container_content = convert_type('tv', 'container') if shows_count > movie_count else convert_type('movie', 'container')
        return sync.data
```
```python
class ListCrewedCombined(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('crewedcombined', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        try:
            movie_count = len([i for i in sync.data if i and i['infoproperties'].get('tmdb_type') == 'movie'])
            shows_count = len(sync.data) - movie_count
        except TypeError:
            return
        self.kodi_db = self.get_kodi_database('both')
        self.container_content = convert_type('tv', 'container') if shows_count > movie_count else convert_type('movie', 'container')
        return sync.data
```
```python
class ListCreditsCombined(ContainerDefaultCacheDirectory):

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory('creditscombined', 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)

        try:
            movie_count = len([i for i in sync.data if i and i['infoproperties'].get('tmdb_type') == 'movie'])
            shows_count = len(sync.data) - movie_count
        except TypeError:
            return

        self.kodi_db = self.get_kodi_database('both')
        self.container_content = convert_type('tv', 'container') if shows_count > movie_count else convert_type('movie', 'container')
        return sync.data
```

- [ ] **Step 2: Replace all three with a shared base + thin subclasses**

Replace all three class definitions above with:
```python
class ListCombinedViewBase(ContainerDefaultCacheDirectory):
    view_name = None

    @ListConfigureOffset
    def get_items(self, tmdb_id, limit=None, sort_by=None, sort_how=None, offset=None, **kwargs):
        sync = BaseViewFactory(self.view_name, 'person', tmdb_id, filters=self.filters, limit=limit, offset=offset, sort_by=sort_by, sort_how=sort_how)
        try:
            movie_count = len([i for i in sync.data if i and i['infoproperties'].get('tmdb_type') == 'movie'])
            shows_count = len(sync.data) - movie_count
        except TypeError:
            return
        self.kodi_db = self.get_kodi_database('both')
        self.container_content = convert_type('tv', 'container') if shows_count > movie_count else convert_type('movie', 'container')
        return sync.data


class ListStarredCombined(ListCombinedViewBase):
    view_name = 'starredcombined'


class ListCrewedCombined(ListCombinedViewBase):
    view_name = 'crewedcombined'


class ListCreditsCombined(ListCombinedViewBase):
    view_name = 'creditscombined'
```

- [ ] **Step 3: Compile-check the whole file and the whole tree**

```bash
cd /home/frankie/GitHub/plugin.video.themoviedb.helper
python3 -m py_compile resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
python3 -m compileall -q resources/tmdbhelper/lib
echo "exit code: $?"
```
Expected: `exit code: 0`.

- [ ] **Step 4: Confirm the final file only contains the expected class names**

```bash
grep -n "^class " resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
```
Expected: `ListConfigureOffset`, `ListImageViewBase`, `ListFanart`, `ListPoster`, `ListImage`, `ListThumb`, `ListPersonListViewBase`, `ListCast`, `ListCrew`, `ListPersonOrCollectionViewBase`, `ListSeries`, `ListStarredMovies`, `ListStarredTvshows`, `ListCombinedViewBase`, `ListStarredCombined`, `ListCrewedMovies`, `ListCrewedTvshows`, `ListCrewedCombined`, `ListCreditsCombined`, `ListVideos` — all original public class names present, 4 new shared bases added, none of the original public names lost.

- [ ] **Step 5: Commit**

```bash
git add resources/tmdbhelper/lib/items/directories/tmdb/lists_view_db.py
git commit -m "$(cat <<'EOF'
lists_view_db.py: collapse Combined classes into shared base

ListStarredCombined, ListCrewedCombined, and ListCreditsCombined
repeated the exact same movie/show-count try/except and
container_content branching logic verbatim, differing only in the
view-name string passed to BaseViewFactory. Introduces
ListCombinedViewBase; all three public classes keep their exact names
and now just set view_name. No behavior change. This completes the
lists_view_db.py cleanup -- all four duplicated class families in the
file are now single-attribute subclasses of a shared base.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01YAAiVNXQuaYtcUJP8fdRzx
EOF
)"
```

- [ ] **Step 6: Restart Kodi, exercise every affected route**

Restart Kodi (check Aerial/screensaver first). Then browse to, at minimum: a movie or TV show's fanart/poster/image/thumb picker (Task 15's classes), a cast list and a crew list for any movie/show (Task 16's classes), a person's "starred in" movies list and TV shows list, a collection/series list (Task 17's classes), and a person's combined credits view if reachable (Task 18's classes). Confirm every one loads with the same items/count/content-type as before this cleanup — this is the main functional regression check for all of Part 4 in one pass, since all four tasks touch the same file.

---

## Final Verification (all parts)

- [ ] Re-run the full compile sweep on TMDb Helper one more time: `python3 -m compileall -q resources/tmdbhelper/lib && echo OK`.
- [ ] Re-run XML validation on every touched skin file in one pass:
  ```bash
  SKIN=/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
  for f in SkinSettings.xml Includes_Object.xml Includes_Home.xml Includes_View_52_Showcase.xml Includes_Items.xml Custom_1150_Dialog.xml; do
    python3 -c "import xml.etree.ElementTree as ET; ET.parse('$SKIN/1080i/$f'); print('$f OK')"
  done
  ```
- [ ] Confirm `Custom_1120_EnableInfoButtons.xml`, `Custom_1121_EnableIndicators.xml`, `Custom_1123_EnableSearchLists.xml` no longer exist; confirm `Custom_1115_AutoVis.xml` and `Custom_1122_MusicFullscreenEnabler.xml` are untouched (`git diff` has no meaning here since the skin isn't a repo — use `ls -la` timestamps or just re-read them to confirm identical content to before this plan started).
- [ ] Full Kodi restart, run through Home menu end-to-end (Search, Ask Gemini, Movies, TV Shows, Anime, Settings) confirming nothing from this session's earlier Gemini work regressed.
- [ ] `git log --oneline -8` in the TMDb Helper repo to confirm all 8 expected commits (Tasks 11-18) landed on `refactor/tmdb-helper-cleanup`.
