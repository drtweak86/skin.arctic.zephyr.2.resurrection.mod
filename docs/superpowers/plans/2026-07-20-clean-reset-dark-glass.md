# Arctic Zephyr 2 Resurrection Clean Reset and Dark Glass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the damaged Arctic Zephyr 2 Resurrection installation with its verified clean 1.0.51 package, restore only supported menu/widget/search and visual configuration, safely enable Ask Gemini, prefer Flix Poster where supported, install Outline HD weather icons, and add an optional Dark Glass presentation layer.

**Architecture:** A standalone reset utility outside the skin directory owns archive validation, timestamped backup, selective settings restoration, XML generation, state manifests, and rollback. Phase one restores supported configuration through Skin Shortcuts and Kodi settings, with a hard live safety gate around TMDb Helper/Gemini; phase two adds a small conditional skin include whose single setting changes only presentation. Every mutation is preceded by a test and a recoverable checkpoint, and one controlled Kodi restart replaces `ReloadSkin()`.

**Tech Stack:** Python 3.13 standard library (`unittest`, `xml.etree.ElementTree`, `zipfile`, `ast`, `json`, `urllib.request`, `shutil`), Kodi 21.3 Omega JSON-RPC, Kodi skin XML, Skin Shortcuts DATA XML, TMDb Helper 6.16.2, SQLite.

## Global Constraints

- Source archive: `/home/frankie/.kodi/addons/packages/skin.arctic.zephyr.2.resurrection.mod-1.0.51.zip`; reject any archive whose `addon.xml` does not declare ID `skin.arctic.zephyr.2.resurrection.mod` and version `1.0.51`.
- Do not delete replaced data. Move it into `/home/frankie/.kodi/reset-backups/az2r-<UTC timestamp>/` and write a JSON manifest containing source, backup destination, SHA-256 where applicable, and restoration status.
- Never print, copy into the plan, or rewrite the Gemini API key or any other credential. Preserve TMDb Helper `settings.xml` in place.
- Never restore the old skin `settings.xml`, `Includes_CustomWidgets.xml`, modified `Includes.xml`, or generated `script-skinshortcuts-includes.xml` wholesale.
- Widget rows must come from Skin Shortcuts submenu DATA and its native generator; generated control IDs must be unique.
- Main-menu order is Search, Ask Gemini (only after the safety gate), Movies, TV Shows, Anime, Settings.
- Otaku search routes are exactly `plugin://plugin.video.otaku/search_movie/` and `plugin://plugin.video.otaku/search_tv_show/`.
- Ask Gemini route is exactly `plugin://plugin.video.themoviedb.helper/?info=gemini`; a prompt opening alone is not a successful safety test.
- Use `cached_statements=0` for every `sqlite3.connect` call in TMDb Helper's `dbdata.py` and Jurialmunkey's `scache.py` on Python 3.13.
- Flix Poster view ID `526` is preferred only in windows that declare it; unsupported windows keep their native view.
- Preserve Kodi theme `Square`, pre-Dark-Glass colour theme `Darker with dark dialogs`, font `Default`, zoom `0`, provider `weather.openmeteo`, and the approved presentation settings listed in Task 3.
- Weather resource is `resource.images.weathericons.outline-hd` version `0.0.3` or newer, with `weather.icons.path=resource://resource.images.weathericons.outline-hd/`.
- Do not call `ReloadSkin()`. Before controlled restart, require `Player.GetActivePlayers` to return an empty array and reset the idle timer.
- The installed add-on is not a Git repository. Replace plan commit steps with immutable hash checkpoints in the reset manifest; do not initialise a repository inside the installed add-on.

---

## File Structure

- Create `/home/frankie/.kodi/reset-work/az2r_reset.py`: pure validation, backup, settings-filter, Skin Shortcuts XML, XML patch, and rollback operations.
- Create `/home/frankie/.kodi/reset-work/kodi_rpc.py`: authenticated loopback JSON-RPC calls without logging credentials.
- Create `/home/frankie/.kodi/reset-work/tests/test_az2r_reset.py`: unit and fixture tests for the reset transaction and generated XML.
- Create `/home/frankie/.kodi/reset-work/tests/test_sqlite_workaround.py`: AST test for all SQLite connection factories.
- Create `/home/frankie/.kodi/reset-work/tests/test_skin_contract.py`: parsed-XML assertions for menu order, providers, view defaults, weather, and Dark Glass.
- Create `/home/frankie/.kodi/reset-work/state.json`: active transaction state and backup location.
- Create `/home/frankie/.kodi/reset-work/restoration-manifest.json`: selected graphical settings and inclusion/exclusion reasons.
- Replace `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/` from the validated archive, then modify only `1080i/MyVideoNav.xml`, `1080i/Includes.xml`, `1080i/Includes_Global.xml`, `1080i/Includes_Dialog.xml`, `1080i/SkinSettings.xml`, and new `1080i/Includes_DarkGlass.xml`.
- Replace Skin Shortcuts source DATA under `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/`; let Skin Shortcuts regenerate `1080i/script-skinshortcuts-includes.xml`.
- Modify the four SQLite connection calls in `/home/frankie/.kodi/addons/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/files/dbdata.py` and `/home/frankie/.kodi/addons/script.module.jurialmunkey/resources/modules/jurialmunkey/scache.py`.

## Primary References

- Kodi Omega Outline HD resource: `https://kodi.tv/addons/omega/resource.images.weathericons.outline-hd/`.
- Kodi built-ins, including `Container.SetViewMode`, `Skin.ToggleSetting`, `InstallAddon`, and idle-timer actions: `https://kodi.wiki/view/List_of_built-in_functions`.
- Kodi JSON-RPC API: `https://kodi.wiki/view/JSON-RPC_API`.
- CPython Python 3.12/3.13 multithreaded SQLite statement-cache defect: `https://github.com/python/cpython/issues/118172`.
- TMDb Helper cache/Python compatibility evidence: `https://github.com/jurialmunkey/plugin.video.themoviedb.helper/issues/1251`.

### Task 1: Transactional Reset Utility and Archive Validation

**Files:**
- Create: `/home/frankie/.kodi/reset-work/az2r_reset.py`
- Create: `/home/frankie/.kodi/reset-work/tests/test_az2r_reset.py`
- Create at runtime: `/home/frankie/.kodi/reset-work/state.json`

**Interfaces:**
- Consumes: cached ZIP and absolute live/backup paths.
- Produces: `validate_archive(path: Path) -> dict[str, str]`, `safe_extract(path: Path, destination: Path) -> Path`, `backup_path(source: Path, backup_root: Path, manifest: dict) -> Path`, `swap_directory(staged: Path, live: Path, backup_root: Path, manifest: dict) -> None`, `write_manifest(path: Path, payload: dict) -> None`, and `rollback(manifest_path: Path) -> None`.

- [ ] **Step 1: Write archive and path-safety tests**

```python
# /home/frankie/.kodi/reset-work/tests/test_az2r_reset.py
import json
import tempfile
import unittest
import zipfile
from pathlib import Path

from az2r_reset import safe_extract, validate_archive


class ArchiveTests(unittest.TestCase):
    def make_zip(self, root: Path, addon_id="skin.arctic.zephyr.2.resurrection.mod", version="1.0.51") -> Path:
        target = root / "skin.zip"
        with zipfile.ZipFile(target, "w") as archive:
            archive.writestr(
                "skin.arctic.zephyr.2.resurrection.mod/addon.xml",
                f'<addon id="{addon_id}" version="{version}" name="AZ2R"/>',
            )
            archive.writestr("skin.arctic.zephyr.2.resurrection.mod/1080i/Includes.xml", "<includes/>")
        return target

    def test_validate_archive_accepts_exact_id_and_version(self):
        with tempfile.TemporaryDirectory() as temp:
            metadata = validate_archive(self.make_zip(Path(temp)))
            self.assertEqual(metadata["id"], "skin.arctic.zephyr.2.resurrection.mod")
            self.assertEqual(metadata["version"], "1.0.51")

    def test_validate_archive_rejects_wrong_version(self):
        with tempfile.TemporaryDirectory() as temp:
            with self.assertRaisesRegex(ValueError, "version"):
                validate_archive(self.make_zip(Path(temp), version="1.0.50"))

    def test_safe_extract_rejects_parent_traversal(self):
        with tempfile.TemporaryDirectory() as temp:
            target = Path(temp) / "bad.zip"
            with zipfile.ZipFile(target, "w") as archive:
                archive.writestr("../escape", "bad")
            with self.assertRaisesRegex(ValueError, "unsafe archive member"):
                safe_extract(target, Path(temp) / "out")
```

- [ ] **Step 2: Run the focused tests and confirm they fail**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_az2r_reset.ArchiveTests -v`

Expected: import failure because `az2r_reset.py` does not exist.

- [ ] **Step 3: Implement validation, extraction, backup, manifest, and rollback**

```python
# /home/frankie/.kodi/reset-work/az2r_reset.py
from __future__ import annotations

import hashlib
import json
import os
import shutil
import zipfile
from pathlib import Path
from xml.etree import ElementTree as ET

ADDON_ID = "skin.arctic.zephyr.2.resurrection.mod"
ADDON_VERSION = "1.0.51"


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for block in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(block)
    return digest.hexdigest()


def validate_archive(path: Path) -> dict[str, str]:
    with zipfile.ZipFile(path) as archive:
        addon_names = [name for name in archive.namelist() if name.endswith("/addon.xml")]
        if addon_names != [f"{ADDON_ID}/addon.xml"]:
            raise ValueError(f"archive addon path mismatch: {addon_names}")
        addon = ET.fromstring(archive.read(addon_names[0]))
        if addon.get("id") != ADDON_ID:
            raise ValueError(f"addon id mismatch: {addon.get('id')}")
        if addon.get("version") != ADDON_VERSION:
            raise ValueError(f"addon version mismatch: {addon.get('version')}")
        return {"id": addon.get("id", ""), "version": addon.get("version", ""), "sha256": sha256(path)}


def safe_extract(path: Path, destination: Path) -> Path:
    destination.mkdir(parents=True, exist_ok=False)
    resolved = destination.resolve()
    with zipfile.ZipFile(path) as archive:
        for member in archive.infolist():
            candidate = (destination / member.filename).resolve()
            if candidate != resolved and resolved not in candidate.parents:
                raise ValueError(f"unsafe archive member: {member.filename}")
        archive.extractall(destination)
    return destination / ADDON_ID


def write_manifest(path: Path, payload: dict) -> None:
    temporary = path.with_suffix(path.suffix + ".tmp")
    temporary.write_text(json.dumps(payload, indent=2, sort_keys=True) + "\n", encoding="utf-8")
    os.replace(temporary, path)


def backup_path(source: Path, backup_root: Path, manifest: dict) -> Path:
    destination = backup_root / source.as_posix().lstrip("/")
    destination.parent.mkdir(parents=True, exist_ok=True)
    shutil.move(str(source), str(destination))
    manifest.setdefault("moves", []).append({"source": str(source), "backup": str(destination), "restored": False})
    return destination


def swap_directory(staged: Path, live: Path, backup_root: Path, manifest: dict) -> None:
    if live.exists():
        backup_path(live, backup_root, manifest)
    live.parent.mkdir(parents=True, exist_ok=True)
    shutil.move(str(staged), str(live))
    manifest.setdefault("created", []).append(str(live))


def rollback(manifest_path: Path) -> None:
    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
    for created in reversed(manifest.get("created", [])):
        target = Path(created)
        if target.exists():
            failed = Path(manifest["backup_root"]) / "failed-current" / target.as_posix().lstrip("/")
            failed.parent.mkdir(parents=True, exist_ok=True)
            shutil.move(str(target), str(failed))
    for move in reversed(manifest.get("moves", [])):
        source, backup = Path(move["source"]), Path(move["backup"])
        if backup.exists() and not source.exists():
            source.parent.mkdir(parents=True, exist_ok=True)
            shutil.move(str(backup), str(source))
            move["restored"] = True
    manifest["status"] = "rolled_back"
    write_manifest(manifest_path, manifest)
```

- [ ] **Step 4: Add and run transaction tests**

Add this test to `ArchiveTests`:

```python
    def test_swap_and_rollback_restore_original_directory(self):
        with tempfile.TemporaryDirectory() as temp:
            root = Path(temp)
            live, staged, backup = root / "live", root / "staged", root / "backup"
            live.mkdir()
            staged.mkdir()
            (live / "marker").write_text("old", encoding="utf-8")
            (staged / "marker").write_text("new", encoding="utf-8")
            manifest = {"backup_root": str(backup), "status": "started"}
            swap_directory(staged, live, backup, manifest)
            state = root / "state.json"
            write_manifest(state, manifest)
            self.assertEqual((live / "marker").read_text(), "new")
            rollback(state)
            self.assertEqual((live / "marker").read_text(), "old")
            self.assertEqual(json.loads(state.read_text())["status"], "rolled_back")
```

Import `swap_directory`, `write_manifest`, and `rollback` alongside the existing imports. Run:

`cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_az2r_reset -v`

Expected: all archive and transaction tests pass.

- [ ] **Step 5: Record checkpoint**

Run: `sha256sum /home/frankie/.kodi/reset-work/az2r_reset.py /home/frankie/.kodi/reset-work/tests/test_az2r_reset.py`

Expected: two hashes; copy them into `state.json` under `checkpoints.task_1`.

### Task 2: Kodi JSON-RPC Safety Client

**Files:**
- Create: `/home/frankie/.kodi/reset-work/kodi_rpc.py`
- Test: `/home/frankie/.kodi/reset-work/tests/test_az2r_reset.py`

**Interfaces:**
- Consumes: `/home/frankie/.kodi/userdata/guisettings.xml` and loopback Kodi web service.
- Produces: `rpc(method: str, params: dict | None = None, timeout: float = 5.0) -> object`, `active_players() -> list[dict]`, `execute_builtin(command: str) -> None`, `wait_ready(seconds: int = 60) -> None`. Credentials remain local and are never returned or logged.

- [ ] **Step 1: Write a credential-redaction test**

```python
def test_load_auth_returns_header_not_plaintext(self):
    with tempfile.TemporaryDirectory() as temp:
        xml = Path(temp) / "guisettings.xml"
        xml.write_text('<settings><setting id="services.webserverusername">u</setting><setting id="services.webserverpassword">secret</setting></settings>')
        header = load_auth(xml)
        self.assertTrue(header.startswith("Basic "))
        self.assertNotIn("secret", header)
```

Import `load_auth` from `kodi_rpc` and place this method on a `KodiRPCTests(unittest.TestCase)` class.

- [ ] **Step 2: Implement the loopback client**

```python
# /home/frankie/.kodi/reset-work/kodi_rpc.py
import base64
import json
import time
import urllib.request
from pathlib import Path
from xml.etree import ElementTree as ET

GUISETTINGS = Path("/home/frankie/.kodi/userdata/guisettings.xml")
ENDPOINT = "http://127.0.0.1:8080/jsonrpc"


def load_auth(path: Path = GUISETTINGS) -> str:
    root = ET.parse(path).getroot()
    values = {node.get("id"): node.text or "" for node in root.findall("setting")}
    token = base64.b64encode(f"{values.get('services.webserverusername', '')}:{values.get('services.webserverpassword', '')}".encode()).decode()
    return f"Basic {token}"


def rpc(method: str, params=None, timeout: float = 5.0):
    payload = {"jsonrpc": "2.0", "method": method, "id": 1}
    if params is not None:
        payload["params"] = params
    request = urllib.request.Request(
        ENDPOINT,
        data=json.dumps(payload).encode(),
        headers={"Content-Type": "application/json", "Authorization": load_auth()},
    )
    response = json.loads(urllib.request.urlopen(request, timeout=timeout).read())
    if "error" in response:
        raise RuntimeError(json.dumps(response["error"], sort_keys=True))
    return response.get("result")


def active_players():
    return rpc("Player.GetActivePlayers")


def execute_builtin(command: str) -> None:
    rpc("XBMC.ExecuteBuiltin", {"command": command})


def wait_ready(seconds: int = 60) -> None:
    deadline = time.monotonic() + seconds
    while time.monotonic() < deadline:
        try:
            rpc("JSONRPC.Ping", timeout=2)
            return
        except Exception:
            time.sleep(1)
    raise TimeoutError("Kodi JSON-RPC did not become ready")
```

- [ ] **Step 3: Verify syntax and a live ping without displaying configuration**

Run: `python3 -m py_compile /home/frankie/.kodi/reset-work/kodi_rpc.py`

Run while Kodi is open: `cd /home/frankie/.kodi/reset-work && python3 -c 'from kodi_rpc import rpc; print(rpc("JSONRPC.Ping"))'`

Expected: `pong` or Kodi's equivalent success value; no username, password, API key, or token appears.

- [ ] **Step 4: Record checkpoint hashes in `state.json` under `checkpoints.task_2`**

### Task 3: Selective Visual Settings Restoration

**Files:**
- Modify: `/home/frankie/.kodi/reset-work/az2r_reset.py`
- Modify: `/home/frankie/.kodi/reset-work/tests/test_az2r_reset.py`
- Create at runtime: `/home/frankie/.kodi/reset-work/restoration-manifest.json`

**Interfaces:**
- Consumes: backed-up skin `settings.xml`, pristine skin-declared setting references, and `guisettings.xml`.
- Produces: `declared_skin_setting_ids(skin_root: Path) -> set[str]`, `select_visual_settings(old_settings: Path, declared: set[str]) -> tuple[dict[str, str], list[dict]]`, and `write_skin_settings(path: Path, values: dict[str, str]) -> None`.

- [ ] **Step 1: Write tests for the exact allowlist and exclusions**

```python
APPROVED = {
    "focuscolor.name": "ffd50000",
    "gradientcolor.name": "ffd50000",
    "Icons": "colorful",
    "icons.label": "Colorful",
    "enablegeometric": "true",
    "enableclearlogo": "true",
    "showosdclearart": "true",
    "showdate": "true",
    "posterhighlight": "Mix",
    "homemultiflixview": "true",
    "flixhidemenu": "true",
    "disabledprofileinfo": "true",
    "selectbox.thin": "true",
    "kenburnseffect": "true",
}


def test_select_visual_settings_keeps_approved_and_rejects_state(self):
    with tempfile.TemporaryDirectory() as temp:
        source = Path(temp) / "settings.xml"
        source.write_text('<settings version="2"><setting id="focuscolor.name">ffd50000</setting><setting id="SearchTerm">x</setting><setting id="widgetPath.1">plugin://bad</setting><setting id="generated.hash">abc</setting></settings>')
        selected, decisions = select_visual_settings(source, {"focuscolor.name", "SearchTerm", "widgetPath.1", "generated.hash"})
        self.assertEqual(selected, {"focuscolor.name": "ffd50000"})
        self.assertEqual({d["id"] for d in decisions if not d["copied"]}, {"SearchTerm", "widgetPath.1", "generated.hash"})
```

Place the method on `VisualSettingsTests(unittest.TestCase)` and import `select_visual_settings` from `az2r_reset`.

- [ ] **Step 2: Implement strict selection rules**

Use case-insensitive IDs for matching. Always write the exact `APPROVED` values above when the pristine XML references that ID. For additional safe settings, include only IDs that the pristine skin references and whose lowercase ID contains one of:

```python
SAFE_TOKENS = ("art", "logo", "clear", "label", "indicator", "osd", "font", "color", "colour", "icon", "posterhighlight", "geometric", "date", "kenburns", "selectbox")
DENY_TOKENS = ("hash", "searchterm", "widgetpath", "widgettarget", "skinshortcuts", "customwidget", "controlid", "homemulti", "netflix", "verticalmenu", "horizontalmenu")
```

Parse all pristine `1080i/*.xml` text for `Skin.String(NAME)` and `Skin.HasSetting(NAME)` to form the declared ID set. Produce one decision per old setting:

```json
{"id":"focuscolor.name","source_value":"ffd50000","declared":true,"copied":true,"reason":"approved"}
```

Write a minimal `<settings version="2">` document with one `<setting id="...">value</setting>` per selected value. Do not copy unknown IDs.

Add these functions to `az2r_reset.py`:

```python
import re

APPROVED = {
    "focuscolor.name": "ffd50000", "gradientcolor.name": "ffd50000",
    "icons": "colorful", "icons.label": "Colorful", "enablegeometric": "true",
    "enableclearlogo": "true", "showosdclearart": "true", "showdate": "true",
    "posterhighlight": "Mix", "homemultiflixview": "true", "flixhidemenu": "true",
    "disabledprofileinfo": "true", "selectbox.thin": "true", "kenburnseffect": "true",
}
SAFE_TOKENS = ("art", "logo", "clear", "label", "indicator", "osd", "font", "color", "colour", "icon", "posterhighlight", "geometric", "date", "kenburns", "selectbox")
DENY_TOKENS = ("hash", "searchterm", "widgetpath", "widgettarget", "skinshortcuts", "customwidget", "controlid", "homemulti", "netflix", "verticalmenu", "horizontalmenu")


def declared_skin_setting_ids(skin_root: Path) -> set[str]:
    pattern = re.compile(r"Skin\.(?:String|HasSetting)\(([^)]+)\)", re.IGNORECASE)
    return {match.group(1).strip() for path in (skin_root / "1080i").glob("*.xml") for match in pattern.finditer(path.read_text(encoding="utf-8", errors="replace"))}


def select_visual_settings(old_settings: Path, declared: set[str]):
    declared_map = {item.casefold(): item for item in declared}
    old = {node.get("id", ""): node.text or "" for node in ET.parse(old_settings).getroot().findall("setting")}
    selected, decisions = {}, []
    for setting_id, source_value in old.items():
        folded = setting_id.casefold()
        is_declared = folded in declared_map
        if folded in APPROVED and is_declared:
            selected[setting_id] = APPROVED[folded]
            reason, copied = "approved", True
        elif is_declared and any(token in folded for token in SAFE_TOKENS) and not any(token in folded for token in DENY_TOKENS):
            selected[setting_id] = source_value
            reason, copied = "safe presentation setting", True
        else:
            reason = "not declared" if not is_declared else "excluded state or layout setting"
            copied = False
        decisions.append({"id": setting_id, "source_value": source_value, "declared": is_declared, "copied": copied, "reason": reason})
    for folded, value in APPROVED.items():
        if folded in declared_map:
            selected.setdefault(declared_map[folded], value)
    return selected, decisions


def write_skin_settings(path: Path, values: dict[str, str]) -> None:
    root = ET.Element("settings", {"version": "2"})
    for setting_id in sorted(values, key=str.casefold):
        ET.SubElement(root, "setting", {"id": setting_id}).text = values[setting_id]
    temporary = path.with_suffix(".xml.tmp")
    ET.ElementTree(root).write(temporary, encoding="utf-8", xml_declaration=True)
    os.replace(temporary, path)
```

- [ ] **Step 3: Run tests**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_az2r_reset -v`

Expected: all tests pass and no fixture contains `SearchTerm` or widget state in selected output.

- [ ] **Step 4: Add `set_kodi_look_and_feel()`**

Implement XML updates that set these `guisettings.xml` IDs only:

```python
KODI_LOOK = {
    "lookandfeel.skintheme": "Square",
    "lookandfeel.skincolors": "Darker with dark dialogs",
    "lookandfeel.skinfont": "Default",
    "lookandfeel.skinzoom": "0",
}
```

Add:

```python
def set_kodi_look_and_feel(path: Path) -> None:
    tree = ET.parse(path)
    nodes = {node.get("id"): node for node in tree.getroot().findall("setting")}
    missing = set(KODI_LOOK) - set(nodes)
    if missing:
        raise ValueError(f"missing Kodi look settings: {sorted(missing)}")
    for setting_id, value in KODI_LOOK.items():
        nodes[setting_id].text = value
    temporary = path.with_suffix(".xml.tmp")
    tree.write(temporary, encoding="utf-8", xml_declaration=True)
    os.replace(temporary, path)
```

Unit-test that unrelated settings and the weather provider are unchanged.

- [ ] **Step 5: Record checkpoint hashes in `state.json` under `checkpoints.task_3`**

### Task 4: Stop Kodi, Back Up All State, and Install the Pristine Skin

**Files:**
- Use: `/home/frankie/.kodi/reset-work/az2r_reset.py`
- Create: `/home/frankie/.kodi/reset-backups/az2r-<UTC timestamp>/manifest.json`
- Replace: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/`

**Interfaces:**
- Consumes: Tasks 1–3 and the validated ZIP.
- Produces: pristine skin directory, complete backup, and rollback manifest with status `pristine_installed`.

- [ ] **Step 1: Preflight all exact sources**

Assert these exist before stopping Kodi:

```text
/home/frankie/.kodi/addons/packages/skin.arctic.zephyr.2.resurrection.mod-1.0.51.zip
/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/addon.xml
/home/frankie/.kodi/userdata/addon_data/skin.arctic.zephyr.2.resurrection.mod/settings.xml
/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/
/home/frankie/.kodi/userdata/guisettings.xml
/home/frankie/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/settings.xml
```

Run `validate_archive` and store archive ID, version, and SHA-256 in the manifest. Abort before any move on failure.

- [ ] **Step 2: Check Aerial/player safety and stop Kodi cleanly**

Run from the RPC client:

```python
players = active_players()
if players:
    raise RuntimeError(f"active Kodi players prevent reset: {len(players)}")
execute_builtin("ResetSystemIdleTimer")
rpc("Application.Quit")
```

Poll `pgrep -af '/usr/lib/.*/kodi.bin|/usr/bin/kodi'` for up to 30 seconds. Abort without modifying files if Kodi remains open.

- [ ] **Step 3: Create one timestamped backup transaction**

Move the live skin directory and skin addon-data directory into the backup. Copy, without changing the sources, `guisettings.xml`, all of `script.skinshortcuts`, all of `plugin.video.themoviedb.helper`, the installed weather resource directories, and `/home/frankie/.kodi/addons/packages/` entries for the selected weather resources. Record every path in `manifest.json`.

Also copy `docs/superpowers/` from the backed-up skin to `/home/frankie/.kodi/reset-work/docs-superpowers/`; it will be copied back after the clean swap so this plan and its design remain available.

- [ ] **Step 4: Extract, validate, and atomically swap**

Create staging with `mktemp -d /home/frankie/.kodi/reset-work/staging.XXXXXX`, call `safe_extract`, parse every staged `*.xml` with `ElementTree`, assert staged `1080i/Includes.xml` does not reference `Includes_CustomWidgets.xml`, then call `swap_directory`.

Copy `/home/frankie/.kodi/reset-work/docs-superpowers/` to the new skin's `docs/superpowers/`. Do not copy any other file from the old skin.

- [ ] **Step 5: Create clean skin state and restoration manifest**

Use Task 3 to generate the new skin `settings.xml` and `/home/frankie/.kodi/reset-work/restoration-manifest.json`. Preserve the backed-up file unchanged. Set transaction status to `pristine_installed`.

- [ ] **Step 6: Offline structural verification**

Run:

`find /home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod -name '*.xml' -print0 | xargs -0 -n1 xmllint --noout`

Expected: exit 0. If `xmllint` is unavailable, run the equivalent `ElementTree.parse` loop. On any failure, call `rollback(state.json)` before launching Kodi.

### Task 5: Python 3.13 SQLite Workaround and TMDb Cache Rotation

**Files:**
- Modify: `/home/frankie/.kodi/addons/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/files/dbdata.py`
- Modify: `/home/frankie/.kodi/addons/script.module.jurialmunkey/resources/modules/jurialmunkey/scache.py`
- Create: `/home/frankie/.kodi/reset-work/tests/test_sqlite_workaround.py`
- Move at runtime: `/home/frankie/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/database_07/`
- Move at runtime if present: `/home/frankie/.kodi/userdata/addon_data/plugin.video.themoviedb.helper/pickle/`

**Interfaces:**
- Consumes: pristine copies stored in the transaction backup and Python AST.
- Produces: four connection factories with `cached_statements=0`; fresh generated caches; Gemini safety verdict `passed` or `failed` in `state.json`.

- [ ] **Step 1: Write the failing AST test**

```python
# /home/frankie/.kodi/reset-work/tests/test_sqlite_workaround.py
import ast
import unittest
from pathlib import Path

FILES = [
    Path("/home/frankie/.kodi/addons/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/files/dbdata.py"),
    Path("/home/frankie/.kodi/addons/script.module.jurialmunkey/resources/modules/jurialmunkey/scache.py"),
]


class SQLiteWorkaroundTests(unittest.TestCase):
    def test_every_sqlite_connect_disables_statement_cache(self):
        found = 0
        for path in FILES:
            tree = ast.parse(path.read_text(encoding="utf-8"), filename=str(path))
            for node in ast.walk(tree):
                if isinstance(node, ast.Call) and isinstance(node.func, ast.Attribute) and node.func.attr == "connect":
                    found += 1
                    values = {kw.arg: kw.value for kw in node.keywords}
                    self.assertIn("cached_statements", values, str(path))
                    self.assertEqual(ast.literal_eval(values["cached_statements"]), 0, str(path))
        self.assertEqual(found, 4)
```

- [ ] **Step 2: Run it and confirm four unpatched calls fail**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_sqlite_workaround -v`

Expected: FAIL because `cached_statements` is absent.

- [ ] **Step 3: Back up and patch only the four call sites**

Copy both source files to `<backup_root>/patched-sources/...` and record their SHA-256. For each existing `sqlite3.connect(...)`, retain every positional argument and existing keyword and add exactly:

```python
cached_statements=0,
```

Do not change pooling, locking, timeouts, schemas, or query code.

- [ ] **Step 4: Rotate only generated TMDb data**

Move `database_07` and `pickle` if present to `<backup_root>/generated-tmdb-cache/`. Leave `settings.xml`, `players/`, and `qrcode/` in place. Record moved paths. Never read or print `settings.xml` values.

- [ ] **Step 5: Run AST and compile verification**

Run:

`cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_sqlite_workaround -v`

Run:

`python3 -m py_compile /home/frankie/.kodi/addons/plugin.video.themoviedb.helper/resources/tmdbhelper/lib/files/dbdata.py /home/frankie/.kodi/addons/script.module.jurialmunkey/resources/modules/jurialmunkey/scache.py`

Expected: PASS and no output from compilation.

- [ ] **Step 6: Save the update-reapplication contract**

Add patched file paths, original hashes, patched hashes, and `reason: "Python 3.13 sqlite statement-cache concurrency workaround"` to `state.json`. This makes add-on-update detection possible without embedding credentials.

### Task 6: Skin Shortcuts Menu, Search, and Widget Sources

**Files:**
- Modify: `/home/frankie/.kodi/reset-work/az2r_reset.py`
- Create/replace: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/mainmenu.DATA.xml`
- Create/replace: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/searchmenu.DATA.xml`
- Create/replace: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/movies.DATA.xml`
- Create/replace: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/tvshows.DATA.xml`
- Create/replace: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/plugin-video-otaku.DATA.xml`
- Remove from active generated state by moving to backup: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/script-skinshortcuts-includes.xml`
- Test: `/home/frankie/.kodi/reset-work/tests/test_skin_contract.py`

**Interfaces:**
- Consumes: Gemini safety verdict and exact provider routes.
- Produces: `write_shortcuts(path: Path, items: list[dict[str, str]]) -> None` and five valid DATA files that Skin Shortcuts can regenerate.

- [ ] **Step 1: Write contract tests for provider routes and menu order**

Parse each DATA file. Assert main labels equal `['Search', 'Ask Gemini', 'Movies', 'TV Shows', 'Anime', 'Settings']` only when `state.json` says Gemini passed; otherwise assert the same order without Ask Gemini. Assert search actions equal:

```python
[
    "DefaultSearch-TMDBMovies",
    "DefaultSearch-TMDBShows",
    "plugin://plugin.video.otaku/search_movie/",
    "plugin://plugin.video.otaku/search_tv_show/",
]
```

Assert submenu widget action URLs equal:

```python
MOVIES = [
    ("Continue Watching", "plugin://plugin.video.themoviedb.helper/?info=trakt_inprogress&tmdb_type=movie&reload=%24INFO%5BWindow%28Home%29.Property%28TMDbHelper.Widgets.Reload%29%5D&widget=true"),
    ("Trending", "plugin://plugin.video.themoviedb.helper/?info=trakt_trending&tmdb_type=movie&widget=true"),
    ("Top Rated", "plugin://plugin.video.themoviedb.helper/?info=top_rated&tmdb_type=movie&widget=true"),
]
TV = [
    ("Continue Watching", "plugin://plugin.video.themoviedb.helper/?info=trakt_inprogress&tmdb_type=tv&reload=%24INFO%5BWindow%28Home%29.Property%28TMDbHelper.Widgets.Reload%29%5D&widget=true"),
    ("Trending", "plugin://plugin.video.themoviedb.helper/?info=trakt_trending&tmdb_type=tv&widget=true"),
    ("Top Rated", "plugin://plugin.video.themoviedb.helper/?info=top_rated&tmdb_type=tv&widget=true"),
]
ANIME = [
    ("Continue Watching", "plugin://plugin.video.otaku/next_up"),
    ("Trending", "plugin://plugin.video.otaku/all_time_trending"),
    ("Top Rated", "plugin://plugin.video.otaku/top_100"),
]
```

- [ ] **Step 2: Implement one XML writer**

`write_shortcuts` creates `<shortcuts>`, then one `<shortcut>` per item with children in this order: `defaultID`, `label`, `label2`, `icon`, `thumb`, `action`. `ElementTree` must escape `&` in serialized plugin URLs. Use `ActivateWindow(Videos,"<route>",return)` for main menu providers and raw routes for submenu list properties, matching the skin's existing Skin Shortcuts generator behavior.

Add:

```python
SHORTCUT_FIELDS = ("defaultID", "label", "label2", "icon", "thumb", "action")


def write_shortcuts(path: Path, items: list[dict[str, str]]) -> None:
    root = ET.Element("shortcuts")
    for item in items:
        shortcut = ET.SubElement(root, "shortcut")
        for field in SHORTCUT_FIELDS:
            ET.SubElement(shortcut, field).text = item.get(field, "")
    temporary = path.with_suffix(".xml.tmp")
    ET.ElementTree(root).write(temporary, encoding="utf-8", xml_declaration=True)
    os.replace(temporary, path)


def activate(route: str) -> str:
    return f'ActivateWindow(Videos,"{route}",return)'
```

Main actions are:

```text
Search: Skin.SetString(SearchTerm)
Ask Gemini: ActivateWindow(Videos,"plugin://plugin.video.themoviedb.helper/?info=gemini",return)
Movies: ActivateWindow(Videos,"plugin://plugin.video.themoviedb.helper/?info=dir_movie&tmdb_type=None&widget=true",return)
TV Shows: ActivateWindow(Videos,"plugin://plugin.video.themoviedb.helper/?info=dir_tv&tmdb_type=tv&widget=true",return)
Anime: ActivateWindow(Videos,"plugin://plugin.video.otaku/",return)
Settings: ActivateWindow(Settings)
```

Keep the existing skin/TMDb/Otaku icons. Do not include Live TV or Radio.

- [ ] **Step 3: Generate files and move stale generated XML aside**

Write all five DATA files through temporary siblings and `os.replace`. Move the generated `script-skinshortcuts-includes.xml` into `<backup_root>/generated-skinshortcuts/` if present. Do not hand-edit it.

- [ ] **Step 4: Run offline contract tests**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_skin_contract -v`

Expected: all DATA XML parses, order/routes match, and no active `Includes_CustomWidgets.xml` reference exists.

### Task 7: Flix Poster Default and Outline HD Weather

**Files:**
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/MyVideoNav.xml`
- Modify: `/home/frankie/.kodi/userdata/addon_data/skin.arctic.zephyr.2.resurrection.mod/settings.xml`
- Modify after Kodi stops: `/home/frankie/.kodi/userdata/guisettings.xml`
- Test: `/home/frankie/.kodi/reset-work/tests/test_skin_contract.py`

**Interfaces:**
- Consumes: parsed clean skin XML and RPC add-on state.
- Produces: Flix Poster as the first/default video view, Multi Flix enabled, Outline HD installed/selected, unchanged OpenMeteo provider.

- [ ] **Step 1: Write failing Flix contract tests**

Assert `MyVideoNav.xml` has `<defaultcontrol always="true">526</defaultcontrol>`, `<views>` starts with `526,`, view include `View_526_Flix_Poster` remains present, and control 300 has `<ondown>526</ondown>`. Assert no non-video window file is changed to reference view 526.

- [ ] **Step 2: Apply the minimal video-only XML patch**

Change only:

```xml
<defaultcontrol always="true">526</defaultcontrol>
<views>526,50,525,528,529,500,501,502,504,503,51,510,517,511,512,513,514,516,515,52,520,527,521,522,524,53,523</views>
...
<control type="list" id="300">
    <ondown>526</ondown>
```

Do not add `Container.SetViewMode(526)` globally. Kodi's stored per-path view remains authoritative after the user explicitly changes it, and unsupported windows cannot receive 526.

- [ ] **Step 3: Set Multi Flix and approved visual values**

Verify the generated skin settings contain `homemultiflixview=true` and every Task 3 approved setting. Reject simultaneous true values for `HomeMultiNetflix`, conflicting horizontal/vertical menu layout modes, or old custom widget state.

- [ ] **Step 4: Install Outline HD through Kodi if absent**

After Kodi is launched for the first baseline boot, query `System.HasAddon(resource.images.weathericons.outline-hd)`. If false, execute:

`InstallAddon(resource.images.weathericons.outline-hd)`

Poll for up to 120 seconds and require its installed `addon.xml` version to compare at least `0.0.3`. If Kodi cannot install it, stop and request network/add-on-manager approval; do not substitute an unverified icon pack.

- [ ] **Step 5: Select weather icons without changing provider**

With Kodi stopped, set skin setting `weather.icons.path` to `resource://resource.images.weathericons.outline-hd/`. Assert the `weather.openmeteo` provider selection in Kodi settings is unchanged. Do not patch `MyWeather.xml` or the top bar.

- [ ] **Step 6: Run offline contracts**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_skin_contract -v`

Expected: Flix, Multi Flix, graphical values, provider, and icon path tests pass.

### Task 8: Direct Ask Gemini Safety Gate Before Menu Exposure

**Files:**
- Modify at runtime: `/home/frankie/.kodi/reset-work/state.json`
- Conditionally regenerate: `/home/frankie/.kodi/userdata/addon_data/script.skinshortcuts/mainmenu.DATA.xml`
- Inspect: `/home/frankie/.kodi/temp/kodi.log`

**Interfaces:**
- Consumes: Task 5 workaround, clean TMDb caches, JSON-RPC client, and a fixed harmless test query.
- Produces: `gemini.status=passed` only after a complete result renders and Kodi survives restart; otherwise restored sources and an active menu without Ask Gemini.

- [ ] **Step 1: Launch Kodi and wait for baseline generation**

Start Kodi using the same command/session mechanism already used on this machine, call `wait_ready(60)`, and allow Skin Shortcuts to generate. Do not use `ReloadSkin()`.

- [ ] **Step 2: Verify generated widget identity before Gemini**

Parse generated `script-skinshortcuts-includes.xml`. Assert Movies row IDs are `23101`, `23102`, `23103`; TV row IDs are `33101`, `33102`, `33103`; Anime row IDs are `43101`, `43102`, `43103`; assert all nine IDs are unique and each associated `<content>`/list property contains its expected route. If generation uses a different native prefix, accept it only when all nine IDs are unique and the generated Up/Down focus chain reaches the next populated row and returns to menu control 301 at both ends.

- [ ] **Step 3: Run a fixed-query Gemini route directly**

Activate:

```text
plugin://plugin.video.themoviedb.helper/?info=gemini&query=Recommend%20one%20popular%20movie%20and%20one%20popular%20TV%20show
```

Poll the active Videos container for up to 120 seconds. Success requires `Container.NumItems >= 1`, a non-empty focused item label, and a result item with either IMDb or TMDb metadata. The keyboard-only prompt route is tested later; it does not satisfy this gate.

- [ ] **Step 4: Inspect only relevant, redacted log evidence**

Search lines written after the test started for `SIGSEGV`, `_sqlite3`, `malformed`, `Traceback`, `ERROR`, and `plugin.video.themoviedb.helper`. Do not print full URLs, headers, settings, or tokens. A new native/database error fails the gate.

- [ ] **Step 5: Controlled stability restart**

Require `active_players() == []`, call `ResetSystemIdleTimer`, then `Application.Quit`. Confirm the process exits and start Kodi once. After `wait_ready(60)`, require the home window and generated widgets to load without a new crash log.

- [ ] **Step 6: Branch on the gate**

On success, set `gemini.status=passed` and regenerate mainmenu DATA with Ask Gemini in position 2. On failure, restore both patched source files from `<backup_root>/patched-sources/`, set `gemini.status=failed` with a redacted reason, regenerate the menu without Ask Gemini, and continue. Never keep the menu entry active after a failed gate.

- [ ] **Step 7: Test the no-query user path only after success**

Open `plugin://plugin.video.themoviedb.helper/?info=gemini`, verify one keyboard dialog appears, cancel it, and confirm Kodi remains responsive. Do not automate entry of or inspect the configured API key.

### Task 9: Functional Baseline Navigation and Provider Verification

**Files:**
- Inspect: generated `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/script-skinshortcuts-includes.xml`
- Inspect: `/home/frankie/.kodi/temp/kodi.log`
- Update: `/home/frankie/.kodi/reset-work/state.json`

**Interfaces:**
- Consumes: completed phase-one configuration.
- Produces: `phase_one.status=passed` or immediate rollback.

- [ ] **Step 1: Verify all menu and widget rows visually and through labels**

For Movies, TV Shows, and Anime, focus the first populated widget row, send Down through all three rows, then back to the main menu; repeat upward. At each step query `System.CurrentControlId`, `System.CurrentControl`, and focused label. Fail if focus ID/label does not change within two seconds, an empty row traps focus, or the top/bottom cannot return to menu control 301.

- [ ] **Step 2: Verify four searches**

Use a fixed ordinary query for TMDb Movies, TMDb TV, Otaku Movies, and Otaku TV. Each must open the intended provider, produce at least one item, and return normally. Record provider route and item count only; do not store entered search text in skin settings.

- [ ] **Step 3: Verify Flix behavior**

Open one TMDb list, one Otaku list, one search result list, and Gemini results when enabled. Require view 526 in compatible `MyVideoNav` containers. Open Settings and Weather and confirm neither is assigned 526 and both retain focusable native controls.

- [ ] **Step 4: Verify weather**

Open Weather and confirm the current condition and at least one daily forecast icon resolve beneath `resource://resource.images.weathericons.outline-hd/`. Accept the bundled white top-bar icon where the pristine skin hardcodes it.

- [ ] **Step 5: Verify restart persistence and logs**

Perform one controlled restart using the player-empty check. Re-run menu order, one row transition in each category, icon path, and Flix default checks. Search new log lines for invalid-control, failed-focus, malformed-skin, XML parse, or provider errors.

- [ ] **Step 6: Gate phase two**

Set `phase_one.status=passed` only if every check passes. If navigation/provider mappings cannot be made stable from clean supported configuration, call `rollback(state.json)`, restart Kodi, and verify the original skin/menu returns. Do not start Dark Glass after a failed baseline.

### Task 10: Optional Native Dark Glass Layer

**Files:**
- Create: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_DarkGlass.xml`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes.xml`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Global.xml`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/Includes_Dialog.xml`
- Modify: `/home/frankie/.kodi/addons/skin.arctic.zephyr.2.resurrection.mod/1080i/SkinSettings.xml`
- Test: `/home/frankie/.kodi/reset-work/tests/test_skin_contract.py`

**Interfaces:**
- Consumes: phase-one-passed state and existing skin textures/colors.
- Produces: one `DarkGlass` radiobutton and shared `DarkGlass_GlobalOverlay` and `DarkGlass_DialogOverlay` includes. The global overlay sits above artwork and below all menus, widget rows, and information content; the dialog overlay sits above the normal dialog texture and below dialog controls. The setting changes presentation only.

- [ ] **Step 1: Write failing Dark Glass contract tests**

Assert:

```python
assert includes_file_reference_count("Includes_DarkGlass.xml") == 1
assert skin_settings_toggle("DarkGlass") == {
    "selected": "Skin.HasSetting(DarkGlass)",
    "onclick": "Skin.ToggleSetting(DarkGlass)",
}
assert all_dark_glass_controls_visible_condition() == "Skin.HasSetting(DarkGlass)"
assert dark_glass_textures() == {"common/white.png", "common/dialog.png"}
assert no_dark_glass_control_has_id()
```

Also snapshot all `<onup>`, `<ondown>`, `<onleft>`, `<onright>`, `<onclick>`, `<content>`, and `<views>` values before this task and require the same values afterward, except the already-approved MyVideoNav changes.

- [ ] **Step 2: Add the dedicated include file**

Create:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<includes>
    <include name="DarkGlass_GlobalOverlay">
        <control type="image">
            <visible>Skin.HasSetting(DarkGlass)</visible>
            <texture colordiffuse="99000000">common/white.png</texture>
        </control>
    </include>
    <include name="DarkGlass_DialogOverlay">
        <control type="image">
            <visible>Skin.HasSetting(DarkGlass)</visible>
            <texture border="20" colordiffuse="D0101010">common/dialog.png</texture>
        </control>
    </include>
</includes>
```

These are neutral overlays, not new controls with IDs; focus colors and text colors remain the preserved red-accent, dark-dialog palette.

- [ ] **Step 3: Register the include once**

In clean `Includes.xml`, directly after `<include file="Includes_Dialog.xml" />`, add:

```xml
<include file="Includes_DarkGlass.xml" />
```

- [ ] **Step 4: Insert overlays only into shared presentation layers**

In `Includes_Global.xml`, locate the unique `<include name="Global_Background">` and add `<include>DarkGlass_GlobalOverlay</include>` immediately before its existing `<include>Object_NowPlaying</include>`. This draw order places the translucent layer over fanart/backgrounds and under home menus, all generated widget rows, and information controls. In `Includes_Dialog.xml`, locate the unique `<include name="Dialog_Background_Standard">`; inside its first group add `<include>DarkGlass_DialogOverlay</include>` immediately after the image whose texture has `colordiffuse="dialog_overlay"`. Require both anchors to resolve exactly once or abort without writing either file. Never inject overlays into generated Skin Shortcuts controls.

- [ ] **Step 5: Add one native Skin Settings toggle**

In the Home/Appearance settings group immediately after the Home Layout button, add a unique unused control ID confirmed by parsing all clean skin XML (use `19990` only if absent):

```xml
<control type="radiobutton" id="19990" description="Dark Glass">
    <include>Defs_Settings_Button</include>
    <label>Dark Glass</label>
    <selected>Skin.HasSetting(DarkGlass)</selected>
    <onclick>Skin.ToggleSetting(DarkGlass)</onclick>
</control>
```

Do not call `Skin.Theme`, change geometry, or change any focus action.

- [ ] **Step 6: Run XML and invariance tests**

Run: `cd /home/frankie/.kodi/reset-work && python3 -m unittest tests.test_skin_contract -v`

Run the full skin XML parse command from Task 4.

Expected: all files parse; exactly one toggle/include registration exists; navigation/provider snapshots are unchanged.

- [ ] **Step 7: Record phase-two patch hashes**

Store original and modified hashes of all five skin XML files and the new include under `checkpoints.task_10`. This is the removal/update-reapplication map.

### Task 11: Final Controlled Restart, Visual Acceptance, and Rollback Drill

**Files:**
- Inspect/update: `/home/frankie/.kodi/reset-work/state.json`
- Inspect: `/home/frankie/.kodi/temp/kodi.log`
- Use: `/home/frankie/.kodi/reset-work/az2r_reset.py`

**Interfaces:**
- Consumes: phase-one passed and Dark Glass XML contracts.
- Produces: verified final installation plus a tested rollback command.

- [ ] **Step 1: Run all offline tests immediately before restart**

Run:

`cd /home/frankie/.kodi/reset-work && python3 -m unittest discover -s tests -v`

Expected: all tests pass. Record the command, UTC time, exit code, and test count in `state.json`.

- [ ] **Step 2: Restart safely without Aerial orphaning**

Require `Player.GetActivePlayers == []`, run `ResetSystemIdleTimer`, quit Kodi, confirm process exit, and start it once. Never issue `ReloadSkin()`.

- [ ] **Step 3: Verify Dark Glass on and off**

With Dark Glass off, capture the current theme/menu/dialog state and confirm it matches the clean dark appearance. Toggle it on and inspect home menu, widget backplates, one dialog, and one video information panel: artwork remains visible, text and red focus retain contrast, and geometry/focus do not move. Toggle it off and confirm the original clean appearance returns.

- [ ] **Step 4: Re-run the phase-one smoke checks**

Check all six menu entries when Gemini passed (five otherwise), Up/Down across the three rows in each category, all four search providers, Ask Gemini prompt/result when enabled, Flix Poster in compatible video containers, and Outline HD weather icons.

- [ ] **Step 5: Verify Aerial lifecycle**

Start Aerial normally, confirm a video begins, exit on input, and require `Player.GetActivePlayers` to become empty. Confirm no separate/orphaned Aerial playback remains. Do not modify its playlist behavior in this project.

- [ ] **Step 6: Inspect bounded log lines**

Inspect only lines written since this restart. Require no new `SIGSEGV`, `_sqlite3`, invalid-control, failed-focus, malformed-skin, or XML parse error attributable to this work. Redact query URLs and never emit credentials/tokens.

- [ ] **Step 7: Confirm rollback is executable**

Run a dry check that every `manifest.moves[].backup` path exists and every current destination is known. Document the actual recovery command:

`cd /home/frankie/.kodi/reset-work && python3 -c 'from pathlib import Path; from az2r_reset import rollback; rollback(Path("/home/frankie/.kodi/reset-work/state.json"))'`

Do not execute it after a successful acceptance run. If acceptance fails, stop Kodi safely, execute it, restart, and verify the original state.

- [ ] **Step 8: Mark completion**

Set `status=complete`, record installed skin version, Gemini gate verdict, weather resource version, test results, backup root, and modified file hashes. Keep all backups until the user explicitly asks to remove them.

## Self-Review Results

- Spec coverage: Tasks 4 and 11 cover backup/rollback; Tasks 5 and 8 cover the Gemini crash and safety gate; Task 6 covers final menu order, four search paths, and native Skin Shortcuts widgets; Tasks 7 and 9 cover Multi Flix, Flix Poster, visuals, and Outline HD; Task 10 covers the removable Dark Glass layer; Tasks 4, 8, and 11 cover Aerial-safe restarts.
- Placeholder scan: the plan contains no deferred implementation markers. Task 10's two exact shared anchors must resolve once and deliberately fail closed if the pristine skin structure differs.
- Type consistency: reset, RPC, XML-writer, state, and rollback names are consistent across consuming tasks; `gemini.status` and `phase_one.status` are the only phase gates.
- Repository constraint: no Git commit is planned because the installed add-on is not a repository; each task records SHA-256 checkpoints instead.
