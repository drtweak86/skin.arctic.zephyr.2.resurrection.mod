# Custom Widget Navigation Repair Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Up and Down reliably traverse the Continue Watching, Trending, and Top Rated home widget rows, skip unavailable rows, and return to the main menu at the ends.

**Architecture:** Keep the existing custom widget containers and add mutually exclusive conditional navigation actions to each list/fixed-list control. Validate the XML structure with a focused Python test, reload the active skin, and exercise the real controls through Kodi JSON-RPC.

**Tech Stack:** Kodi 21 skin XML, Python 3 standard library, Kodi JSON-RPC

## Global Constraints

- Modify only focus navigation in `1080i/Includes_CustomWidgets.xml`.
- Preserve widget providers, paths, labels, IDs, visibility, artwork, and global search mappings.
- Cover both list and fixed-list variants for Movies, TV Shows, and Otaku.
- Back up the production XML before editing because the installed add-on is not a Git repository.

---

### Task 1: Add and verify explicit widget-row navigation

**Files:**
- Create: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/tests/test_custom_widget_navigation.py`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomWidgets.xml`
- Back up: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomWidgets.xml.navigation-backup-20260720`

**Interfaces:**
- Consumes: custom widget container IDs `23101`, `993101`, and `993102`; main-menu control ID `301`; Kodi conditions `Container(ID).NumItems` and `Container(ID).IsUpdating`.
- Produces: deterministic `onup` and `ondown` actions on all six list/fixed-list variants of each custom row.

- [ ] **Step 1: Create a recoverable backup**

Run:

```bash
cp -p /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomWidgets.xml /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomWidgets.xml.navigation-backup-20260720
```

Expected: the backup exists and `cmp` reports no difference.

- [ ] **Step 2: Write the failing structural test**

Create a standard-library `unittest` that parses the XML, selects only `include` elements whose `content` is `Object_Widget_Spotlight`, groups them by their `param name="id"`, asserts six variants per target ID, and requires these exact navigation tuples:

```python
EXPECTED = {
    "23101": {
        "onup": [(None, "SetFocus(301)")],
        "ondown": [
            ("Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating", "SetFocus(993101)"),
            ("![Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating] + [Integer.IsGreater(Container(993102).NumItems,0) | Container(993102).IsUpdating]", "SetFocus(993102)"),
            ("![Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating] + ![Integer.IsGreater(Container(993102).NumItems,0) | Container(993102).IsUpdating]", "SetFocus(301)"),
        ],
    },
    "993101": {
        "onup": [
            ("Integer.IsGreater(Container(23101).NumItems,0) | Container(23101).IsUpdating", "SetFocus(23101)"),
            ("![Integer.IsGreater(Container(23101).NumItems,0) | Container(23101).IsUpdating]", "SetFocus(301)"),
        ],
        "ondown": [
            ("Integer.IsGreater(Container(993102).NumItems,0) | Container(993102).IsUpdating", "SetFocus(993102)"),
            ("![Integer.IsGreater(Container(993102).NumItems,0) | Container(993102).IsUpdating]", "SetFocus(301)"),
        ],
    },
    "993102": {
        "onup": [
            ("Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating", "SetFocus(993101)"),
            ("![Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating] + [Integer.IsGreater(Container(23101).NumItems,0) | Container(23101).IsUpdating]", "SetFocus(23101)"),
            ("![Integer.IsGreater(Container(993101).NumItems,0) | Container(993101).IsUpdating] + ![Integer.IsGreater(Container(23101).NumItems,0) | Container(23101).IsUpdating]", "SetFocus(301)"),
        ],
        "ondown": [(None, "SetFocus(301)")],
    },
}
```

For each selected include, compare `[(node.get("condition"), node.text) ...]` for `onup` and `ondown` with `EXPECTED[id]`.

- [ ] **Step 3: Run the test and verify RED**

Run:

```bash
python3 /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/tests/test_custom_widget_navigation.py -v
```

Expected: FAIL because the selected widget controls currently have no `onup` or `ondown` elements.

- [ ] **Step 4: Add the minimal navigation XML**

For every `Object_Widget_Spotlight` include with ID `23101`, insert the `EXPECTED["23101"]` `onup` and `ondown` elements immediately before its existing `onback`. Repeat for IDs `993101` and `993102` using their corresponding expected elements. Do not alter the heading includes or any other element.

- [ ] **Step 5: Run static verification and verify GREEN**

Run:

```bash
python3 /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/tests/test_custom_widget_navigation.py -v
python3 -c 'import xml.etree.ElementTree as ET; ET.parse("/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_CustomWidgets.xml"); print("XML OK")'
```

Expected: the unit test passes and XML parsing prints `XML OK`.

- [ ] **Step 6: Reload the skin and verify live navigation**

Use Kodi JSON-RPC `Input.ExecuteAction` with action `reloadskin`, then query `XBMC.GetInfoLabels` until the Home window and widget counts are restored. Focus Movies and control `23101`, issue Down three times, and query `System.CurrentControlId` after each action.

Expected sequence with all three rows populated: `23101` → `993101` → `993102` → `301`. Then refocus `993102`, issue Up twice, and expect `993102` → `993101` → `23101`.

- [ ] **Step 7: Confirm protected behavior remains unchanged**

Run `cmp`-based or parsed comparisons against the backup to confirm changes are limited to `onup`/`ondown` elements. Query the three widget counts and verify they remain populated. Confirm the existing TMDb movie/TV and Otaku movie/TV search routes are still present in the active Skin Shortcuts configuration.

Expected: only navigation elements differ; widgets retain items; all four search mappings remain intact.

- [ ] **Step 8: Record the local installation result**

The installed add-on has no `.git` directory, so no commit is possible. Retain the timestamped backup and report the modified file, test output, and live control-ID sequence to the user.
