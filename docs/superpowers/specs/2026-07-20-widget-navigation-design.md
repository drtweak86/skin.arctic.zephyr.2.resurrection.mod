# Custom Widget Navigation Repair

## Scope

Repair vertical navigation between the custom Continue Watching, Trending, and Top Rated widget rows on the Arctic Zephyr 2 Resurrection home screen. Apply the same navigation behavior to the Movies, TV Shows, and Otaku menu sections without changing widget content or search mappings.

## Current Failure

The custom widget list controls do not define explicit vertical navigation. Kodi accepts a Down action while focus is on Continue Watching control `23101`, but focus remains on that control even when Trending control `993101` and Top Rated control `993102` are populated.

## Design

Add explicit `onup` and `ondown` actions to each custom widget list variant:

- Continue Watching navigates down to Trending when available, otherwise Top Rated, otherwise the main menu.
- Trending navigates up to Continue Watching when available, otherwise the main menu; it navigates down to Top Rated when available, otherwise the main menu.
- Top Rated navigates up to Trending when available, otherwise Continue Watching, otherwise the main menu; it navigates down to the main menu.
- Up from the first available row and Down from the last available row return focus to main-menu control `301`.

Availability checks use each target container's item count or updating state, matching the rows' existing visibility conditions. The links are added to both list and fixed-list variants and therefore cover Movies, TV Shows, and Otaku.

## Safety and Compatibility

Only focus actions in `Includes_CustomWidgets.xml` change. Widget providers, paths, labels, IDs, visibility rules, artwork, and the existing global search mappings remain untouched. A timestamped backup of the file will be made before editing.

## Verification

1. An XML/navigation assertion must fail before the repair because the required focus actions are absent.
2. After editing, XML parsing and the navigation assertion must pass.
3. Reload the active skin without restarting Kodi.
4. Use Kodi JSON-RPC to confirm Down moves focus from `23101` to `993101`, then to `993102`, and another Down returns focus to `301`.
5. Confirm reverse Up navigation and retain the existing widget item counts and search mappings.
