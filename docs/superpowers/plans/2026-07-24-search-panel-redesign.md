# Search Panel Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the modal-keyboard-driven search (window 1138) with an inline, always-focused edit control feeding four live per-keystroke result tabs (Movies/TV/People via TMDB Helper, Anime via Otaku), fixing the 2026-07-24 "search box disappears" bug at its root by removing the fragile modal-dialog/Done-button hop from the typing path.

**Architecture:** A new `1080i/Includes_CustomSearch.xml` defines the panel (inline edit control `9199`, a 4-button tab selector writing to `Skin.String(SearchActiveTab)`, and four `panel` controls each polling `plugin://...&query=$INFO[Control.GetLabel(9199)]`). `Custom_1138_Search.xml` is rebuilt to host it. All four existing search entry points are simplified from a 3-action `Skin.Reset`/`Skin.SetString`/`ActivateWindow` chain down to a single `ActivateWindow(1138)`.

**Tech Stack:** Kodi skin XML (`skin.arctic.zephyr.2.resurrection.mod`, Estuary-family skin engine), `plugin.video.themoviedb.helper` (TMDB search), `plugin.video.otaku` (`search_anime/<query>` route, confirmed live via `Files.GetDirectory` on 2026-07-24), Kodi JSON-RPC webserver on `localhost:8080` (user `frankie`) for remote inspection during verification.

## Global Constraints

- Do not modify `1080i/Includes_Object.xml`'s `Object_SearchList_Template` — it's shared with the existing Gemini search feature; the new live-query mechanism (`Control.GetLabel` instead of `Skin.String`) is incompatible with its hardcoded `$INFO[Skin.String($PARAM[term])]` and must not leak into it. Build the four new result panels as self-contained controls in the new file instead.
- New hand-authored XML must NOT live in `1080i/script-skinshortcuts-includes.xml` — that file is regenerated from scratch on every `Home.xml` load and silently wipes hand-authored includes (confirmed prior incident, see `kodi_skinshortcuts_includes_regen` project memory).
- Control IDs `9199` (edit), `9201`–`9204` (tabs), `9210`–`9213` (result panels) must not collide with any existing control ID in `1080i/*.xml`. Verified unused as of 2026-07-24 (Task 1, Step 1).
- The minimum-query-length gate is implemented as **non-empty** (`!String.IsEmpty(...)`), not a strict 2-character threshold — Kodi's skin expression language has no reliable cross-version string-length comparison, and the existing `Custom_1138_Search.xml` guard already uses the same `String.IsEmpty` pattern successfully. This is a deliberate, justified deviation from the spec's "2+ characters" wording.
- Every XML file must be well-formed (verified via `python3 -c "import xml.etree.ElementTree as ET; ET.parse(path)"`) before any live Kodi test.
- After every task that changes registered/loaded XML, the skin must be reloaded (Settings → Interface → Skin → re-select the current skin, or fully restart Kodi) before verification — there is no remote/scriptable way to trigger `ReloadSkin()` from outside Kodi's own UI/keymap context.

---

### Task 1: Create the live search panel include and register it

**Files:**
- Create: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomSearch.xml`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes.xml:41` (insert new `<include file>` line after the `script-skinviewtypes-includes.xml` line)

**Interfaces:**
- Produces: `<include name="Search_Panel_Live">` — the full panel, consumed by Task 2's rebuild of `Custom_1138_Search.xml`.
- Produces: skin string `SearchActiveTab` (values: `Movies`, `TV`, `People`, `Anime`) and control `9199` (edit box label) — both consumed by Task 2 (window onload) and by the four result panels within this same file.

- [ ] **Step 1: Confirm control IDs are free**

Run:
```bash
grep -rn 'id="9199"\|id="9201"\|id="9202"\|id="9203"\|id="9204"\|id="9210"\|id="9211"\|id="9212"\|id="9213"' /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/*.xml
```
Expected: no output (all IDs unused).

- [ ] **Step 2: Write `Includes_CustomSearch.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<includes>

    <include name="Search_Panel_Live">
        <control type="group">
            <left>60</left>
            <top>60</top>

            <control type="label">
                <description>Search hint</description>
                <width>1800</width>
                <height>40</height>
                <font>font_small</font>
                <textcolor>dialog_fg_70</textcolor>
                <label>$LOCALIZE[137]</label>
            </control>

            <control type="edit" id="9199">
                <description>Search query</description>
                <top>50</top>
                <width>1800</width>
                <height>75</height>
                <align>left</align>
                <font>font_small_bold</font>
                <textcolor>dialog_fg_70</textcolor>
                <focusedcolor>$VAR[ColorSelected]</focusedcolor>
                <texturefocus colordiffuse="$VAR[ColorHighlight]" border="5">diffuse/box.png</texturefocus>
                <texturenofocus colordiffuse="dialog_fg_12" border="5">diffuse/box.png</texturenofocus>
                <onright>9201</onright>
                <ondown>9201</ondown>
            </control>

            <control type="grouplist" id="9200">
                <top>145</top>
                <width>1800</width>
                <height>60</height>
                <orientation>horizontal</orientation>
                <itemgap>10</itemgap>

                <control type="button" id="9201">
                    <width>200</width>
                    <height>60</height>
                    <label>$LOCALIZE[20338]</label>
                    <onclick>Skin.SetString(SearchActiveTab,Movies)</onclick>
                    <onup>9199</onup>
                    <onleft>9199</onleft>
                    <onright>9202</onright>
                    <ondown>9210</ondown>
                </control>
                <control type="button" id="9202">
                    <width>200</width>
                    <height>60</height>
                    <label>$LOCALIZE[20343]</label>
                    <onclick>Skin.SetString(SearchActiveTab,TV)</onclick>
                    <onup>9199</onup>
                    <onleft>9201</onleft>
                    <onright>9203</onright>
                    <ondown>9211</ondown>
                </control>
                <control type="button" id="9203">
                    <width>200</width>
                    <height>60</height>
                    <label>$LOCALIZE[133]</label>
                    <onclick>Skin.SetString(SearchActiveTab,People)</onclick>
                    <onup>9199</onup>
                    <onleft>9202</onleft>
                    <onright>9204</onright>
                    <ondown>9212</ondown>
                </control>
                <control type="button" id="9204">
                    <width>200</width>
                    <height>60</height>
                    <label>Anime</label>
                    <onclick>Skin.SetString(SearchActiveTab,Anime)</onclick>
                    <onup>9199</onup>
                    <onleft>9203</onleft>
                    <ondown>9213</ondown>
                </control>
            </control>

            <control type="panel" id="9210">
                <description>Movies results</description>
                <top>220</top>
                <width>1800</width>
                <height>700</height>
                <orientation>horizontal</orientation>
                <visible>String.IsEqual(Skin.String(SearchActiveTab),Movies) + !String.IsEmpty(Control.GetLabel(9199))</visible>
                <onup>9201</onup>
                <itemlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture>$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>dialog_fg_70</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </itemlayout>
                <focusedlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture diffuse="$VAR[ColorHighlight]">$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>$VAR[ColorSelected]</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </focusedlayout>
                <content>plugin://plugin.video.themoviedb.helper?info=search&amp;tmdb_type=movie&amp;query=$INFO[Control.GetLabel(9199)]</content>
            </control>

            <control type="panel" id="9211">
                <description>TV results</description>
                <top>220</top>
                <width>1800</width>
                <height>700</height>
                <orientation>horizontal</orientation>
                <visible>String.IsEqual(Skin.String(SearchActiveTab),TV) + !String.IsEmpty(Control.GetLabel(9199))</visible>
                <onup>9202</onup>
                <itemlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture>$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>dialog_fg_70</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </itemlayout>
                <focusedlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture diffuse="$VAR[ColorHighlight]">$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>$VAR[ColorSelected]</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </focusedlayout>
                <content>plugin://plugin.video.themoviedb.helper?info=search&amp;tmdb_type=tv&amp;query=$INFO[Control.GetLabel(9199)]</content>
            </control>

            <control type="panel" id="9212">
                <description>People results</description>
                <top>220</top>
                <width>1800</width>
                <height>700</height>
                <orientation>horizontal</orientation>
                <visible>String.IsEqual(Skin.String(SearchActiveTab),People) + !String.IsEmpty(Control.GetLabel(9199))</visible>
                <onup>9203</onup>
                <itemlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture>$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>dialog_fg_70</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </itemlayout>
                <focusedlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture diffuse="$VAR[ColorHighlight]">$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>$VAR[ColorSelected]</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </focusedlayout>
                <content>plugin://plugin.video.themoviedb.helper?info=search&amp;tmdb_type=person&amp;query=$INFO[Control.GetLabel(9199)]</content>
            </control>

            <control type="panel" id="9213">
                <description>Anime results</description>
                <top>220</top>
                <width>1800</width>
                <height>700</height>
                <orientation>horizontal</orientation>
                <visible>String.IsEqual(Skin.String(SearchActiveTab),Anime) + !String.IsEmpty(Control.GetLabel(9199))</visible>
                <onup>9204</onup>
                <itemlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture>$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>dialog_fg_70</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </itemlayout>
                <focusedlayout width="280" height="420">
                    <control type="image">
                        <width>280</width>
                        <height>380</height>
                        <aspectratio>scale</aspectratio>
                        <texture diffuse="$VAR[ColorHighlight]">$INFO[ListItem.Art(poster)]</texture>
                    </control>
                    <control type="label">
                        <top>385</top>
                        <width>280</width>
                        <height>35</height>
                        <font>font_mini_bold</font>
                        <textcolor>$VAR[ColorSelected]</textcolor>
                        <label>$INFO[ListItem.Label]</label>
                    </control>
                </focusedlayout>
                <content>plugin://plugin.video.otaku/search_anime/$INFO[Control.GetLabel(9199)]</content>
            </control>

        </control>
    </include>

</includes>
```

- [ ] **Step 3: Verify the new file is well-formed XML**

Run:
```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomSearch.xml')"
```
Expected: no output, exit code 0.

- [ ] **Step 4: Register the new file in `Includes.xml`**

In `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes.xml`, find:
```xml
    <include file="script-skinshortcuts-includes.xml"/>
    <include file="script-skinvariables-includes.xml" />
    <include file="script-skinviewtypes-includes.xml" />
```
Change to:
```xml
    <include file="script-skinshortcuts-includes.xml"/>
    <include file="script-skinvariables-includes.xml" />
    <include file="script-skinviewtypes-includes.xml" />
    <include file="Includes_CustomSearch.xml" />
```

- [ ] **Step 5: Verify `Includes.xml` is still well-formed**

Run:
```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes.xml')"
```
Expected: no output, exit code 0.

- [ ] **Step 6: Commit**

```bash
cd /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
git add 1080i/Includes_CustomSearch.xml 1080i/Includes.xml
git commit -m "Add live search panel include (Search_Panel_Live)"
```
(If this directory is not a git repo, skip this step — confirmed not under version control as of 2026-07-24; note this to the user instead of failing silently.)

---

### Task 2: Rebuild `Custom_1138_Search.xml` to host the live panel

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1138_Search.xml` (full rewrite)

**Interfaces:**
- Consumes: `Search_Panel_Live` include (Task 1, `Includes_CustomSearch.xml`), control `9199` (for `defaultcontrol`).
- Produces: window `1138` behavior consumed by Task 3's trigger-point updates (they now just need `ActivateWindow(1138)`, no pre-set skin string).

- [ ] **Step 1: Read the current file to confirm nothing else references it**

Run:
```bash
grep -rln "Custom_1138_Search\|skinshortcuts-template-search" /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/*.xml /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/shortcuts/*.xml
```
Expected: only `Custom_1138_Search.xml` itself and `shortcuts/template.xml` (the `skinshortcuts-template-search` submenu source, which becomes dead code after this change but is left in place — it's the skinshortcuts-managed group definition, not something this window will reference anymore; removing it is out of scope and risks breaking the skinshortcuts `searchmenu` group registration elsewhere).

- [ ] **Step 2: Replace the full contents of `Custom_1138_Search.xml`**

Note: Kodi has no builtin that clears a specific edit control's text (`ClearProperty` only
applies to window/container properties, not control content). Window `1138` is loaded with
`KEEP_IN_MEMORY` (confirmed in `kodi.log` for the original version of this file), so control
`9199`'s typed text will persist across re-openings until the user backspaces it or types over
it. This is an accepted, honest limitation for this iteration — not a regression from today's
behavior, since the old modal keyboard also always opened blank regardless, so this is a minor
UX difference (previous query stays until cleared) rather than a broken feature. Revisit only if
the user reports it as annoying in practice.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<window type="window" id="1138">
    <defaultcontrol always="true">9199</defaultcontrol>
    <onload>Skin.Reset(SearchActiveTab)</onload>
    <onload>Skin.SetString(SearchActiveTab,Movies)</onload>
    <controls>
        <include>Global_Background</include>
        <include>Topbar</include>

        <control type="group">
            <include>Animation_FadeInOut</include>
            <include>View_Group</include>
            <include>Search_Panel_Live</include>
        </control>

        <include>Object_PlotOverlay</include>
    </controls>
</window>
```

- [ ] **Step 3: Verify well-formed XML**

Run:
```bash
python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Custom_1138_Search.xml')"
```
Expected: no output, exit code 0.

- [ ] **Step 4: Commit**

```bash
cd /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
git add 1080i/Custom_1138_Search.xml
git commit -m "Rebuild window 1138 around the live search panel"
```
(Skip if not a git repo, per Task 1 Step 6 note.)

---

### Task 3: Simplify all four search trigger points to a single `ActivateWindow(1138)`

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/shortcuts/overrides.xml:37-41`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Home.xml:1454-1456`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Items.xml:1970-1972`

**Interfaces:**
- Consumes: window `1138` from Task 2 (now self-initializing via its own `onload`, requires no pre-set `Skin.String(SearchTerm)`).

`shortcuts/mainmenu.DATA.xml:44-45` needs **no edit** — it calls bare `Skin.SetString(SearchTerm)`, which is intercepted and expanded by the `overrides.xml` override being changed in Step 1 below, so fixing the override fixes this call site too.

- [ ] **Step 1: Update the `overrides.xml` override block**

In `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/shortcuts/overrides.xml`, find:
```xml
    <override action="Skin.SetString(SearchTerm)">
        <action>Skin.Reset(SearchTerm)</action>
        <action>Skin.SetString(SearchTerm)</action>
        <action>ActivateWindow(1138)</action>
    </override>
```
Change to:
```xml
    <override action="Skin.SetString(SearchTerm)">
        <action>ActivateWindow(1138)</action>
    </override>
```

- [ ] **Step 2: Update the Home corner-icon search button**

In `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Home.xml`, find:
```xml
            <param name="icon" value="special://skin/extras/icons/search.png" />
            <onclick>Skin.Reset(SearchTerm)</onclick>
            <onclick>Skin.SetString(SearchTerm)</onclick>
            <onclick>ActivateWindow(1138)</onclick>
```
Change to:
```xml
            <param name="icon" value="special://skin/extras/icons/search.png" />
            <onclick>ActivateWindow(1138)</onclick>
```

- [ ] **Step 3: Update the PVR-context item list search entry**

In `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Items.xml`, find:
```xml
                <label>$LOCALIZE[137]</label>
                <icon>special://skin/extras/icons/search.png</icon>
                <onclick>Skin.Reset(SearchTerm)</onclick>
                <onclick>Skin.SetString(SearchTerm)</onclick>
                <onclick>ActivateWindow(1138)</onclick>
```
Change to:
```xml
                <label>$LOCALIZE[137]</label>
                <icon>special://skin/extras/icons/search.png</icon>
                <onclick>ActivateWindow(1138)</onclick>
```

- [ ] **Step 4: Verify all three edited files are well-formed**

Run:
```bash
for f in shortcuts/overrides.xml 1080i/Includes_Home.xml 1080i/Includes_Items.xml; do
  python3 -c "import xml.etree.ElementTree as ET; ET.parse('/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/$f')" && echo "$f OK"
done
```
Expected: `shortcuts/overrides.xml OK`, `1080i/Includes_Home.xml OK`, `1080i/Includes_Items.xml OK`.

- [ ] **Step 5: Commit**

```bash
cd /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod
git add shortcuts/overrides.xml 1080i/Includes_Home.xml 1080i/Includes_Items.xml
git commit -m "Simplify search entry points to a single ActivateWindow(1138)"
```
(Skip if not a git repo, per Task 1 Step 6 note.)

---

### Task 4: End-to-end verification

**Files:** none (verification only).

**Interfaces:**
- Consumes: everything from Tasks 1-3.

- [ ] **Step 1: Reload the skin**

Ask the user to reload the skin (Settings → Interface → Skin → re-select the current skin) or fully restart Kodi, since `Includes.xml` registration changes require a full skin reload to pick up.

- [ ] **Step 2: Check for XML/include errors after reload**

Run:
```bash
grep -in "invalid include\|error.*Includes_CustomSearch\|error.*Custom_1138_Search\|Skin has invalid" /home/frankie/.kodi/temp/kodi.log | tail -30
```
Expected: no output.

- [ ] **Step 3: Confirm the Home corner-icon search opens window 1138 with the edit control focused**

Ask the user to press the search icon on Home, then run:
```bash
curl -s -u frankie:dingleberry -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"XBMC.GetInfoLabels","params":{"labels":["System.CurrentWindow","System.CurrentControlID"]},"id":1}' http://localhost:8080/jsonrpc
```
Expected: `"System.CurrentControlID":"9199"` and a window label matching the search window.

- [ ] **Step 4: Confirm live typing updates results, via the Kodi mobile remote app (the original repro device)**

Ask the user to type a query (e.g. "naruto") via the app's text box, without hitting confirm, and report what they see on screen (results should appear on the Movies tab as they type, since it's the default active tab).

Run, to cross-check server-side:
```bash
curl -s -u frankie:dingleberry -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"XBMC.GetInfoLabels","params":{"labels":["Control.GetLabel(9199)"]},"id":1}' http://localhost:8080/jsonrpc
```
Expected: the label reflects the characters typed so far (live, not empty) — this is the direct fix verification for the 2026-07-24 bug, where the equivalent check showed a permanently empty label.

- [ ] **Step 5: Confirm all four tabs return results**

Ask the user to switch to TV, People, and Anime tabs in turn (with a query already typed) and confirm each shows results. For Anime specifically, confirm at least one result is playable through Otaku.

- [ ] **Step 6: Confirm the fallback physical/IR remote path still works**

Ask the user (or note for later testing if no IR remote is available right now) to confirm that selecting control `9199` with a D-pad/IR remote and pressing OK still opens Kodi's native on-screen keyboard, and that typing there also updates results live.

- [ ] **Step 7: Confirm no `GetDirectory` errors on empty/short queries**

Run:
```bash
grep -in "GetDirectory - Error getting plugin://plugin.video.themoviedb.helper.*query=$\|GetDirectory - Error getting plugin://plugin.video.otaku/search_anime/$" /home/frankie/.kodi/temp/kodi.log | tail -20
```
Expected: no output (the `!String.IsEmpty(...)` visibility guard on each panel should prevent the plugin from ever being queried with an empty string).

- [ ] **Step 8: Confirm the other three trigger points still work**

Ask the user to try the main-menu search shortcut and, if accessible, the PVR-context search entry, confirming both open window 1138 the same way as the Home corner icon.
