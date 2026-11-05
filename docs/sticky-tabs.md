# Sticky Tabs

A sticky tab remembers a home URL and a custom name. Navigating away marks the tab as changed; one action returns to the home URL. Reference: Zen (`src/zen/tabs/ZenPinnedTabManager.mjs`), Arc, Dia.

Goal: smallest possible diff against Chromium; reuse existing UI.

## Behavior

- Pinned tabs: always sticky, look unchanged (icon only, no title).
- Other tabs: user makes one sticky → store current URL + title, move it right after pinned tabs. A tab in a group stays in the group; the whole group moves (groups with a sticky tab count as part of the sticky run).
- Leave: committed URL differs from home in any part, `#fragment` included (`GURL` equality in `DidFinishNavigation`, which also fires for same-document/`pushState` changes) → tab marked changed. Sites that rewrite URL on load: user picks "Set current page as sticky URL".
- Reset: load home URL, drop back/forward history, scroll, session storage.

## UI

| Element | Implementation |
|---|---|
| Title | Sticky title replaces page title in `TabData` (`tab_data.cc`); works in horizontal/vertical strip, tooltip, hover card. |
| State dot | Favicon dot via `TabData::sticky_state` → `TabIcon`: green (`kColorAlertLowSeverity`) at home URL, yellow (`kColorAlertMediumSeverityIcon`) after leaving it, none for other tabs. Real attention keeps the accent color. Same for pinned and titled sticky tabs. |
| Back home | Chrome Home command (toolbar ⌂ when enabled, Cmd+Shift+H, History → Home) resets the active sticky tab instead of opening the homepage; hook in `chrome::Home()`. Also menu "Back to sticky URL". |
| Rename | Double-click sticky tab, or menu "Rename…" → centered modal (`chrome::ShowBrowserModal`). Enter saves, Esc cancels. |
| Context menu | "Make tab sticky" / "Remove sticky", "Back to sticky URL" (only when changed), "Set current page as sticky URL", "Rename…". |

Skipped for now: inline title editing, in-tab reset button / favicon click (layout churn → upstream conflicts), edit-URL dialog, tab-anchored rename popup (would require tabs → views cross-layer plumbing).

## Decisions

**Built into Chromium, not an extension.** `chrome.tabs` IDs change on every restart, so an extension can only match stored tabs by guessing (URL, position).

**Storage: session `extra_data` + existing `pinned_tabs` pref.** Only pinned tabs come back when the session isn't restored, so the existing pinned-tab pref is extended instead of adding a new one. Bookmarks rejected (visible, user-editable, async load).

| Store | Contents | Role |
|---|---|---|
| Session `extra_data` (`sticky.home_url`, `sticky.title`) | home URL, title | Travels with the tab: normal restore and Cmd+Shift+T. Written on change (`AddTabExtraData`) and on session rebuild (`BuildCommandsForTab`, which otherwise drops tab extra data). |
| Pref `pinned_tabs` (`PinnedTabCodec`) | `url` = home URL, `sticky_title` | Recovery of pinned tabs when the session isn't restored. Lifecycle (tab close, quit) handled by existing `PinnedTabService`. |

**Startup:**
- Session restored → `browser_tabrestore.cc` parks state on the WebContents; the tab's `StickyTabController` picks it up.
- Session not restored ("Open the New Tab page") → pinned tabs open from `pinned_tabs` at their home URL with their title. Non-pinned sticky tabs are dropped, same as other normal tabs.
- Session restore attempted but returned nothing (session file missing or corrupt with "Continue where you left off" on) → `RestoreOrCreateBrowser` falls back to opening the `pinned_tabs` backup, so sticky pinned tabs are not lost.
- Pinned tab without saved state (legacy, pre-feature pin) → becomes sticky on first load, title follows page until renamed.

**Placement: constrain inserts only.** New tabs that would land before or inside the sticky run open right after it; tabs already landing after it are untouched: `TabStripModel::ConstrainInsertionIndex` lower bound becomes `IndexOfFirstNonStickyTab()` (first non-pinned, non-sticky index; groups with a sticky tab are skipped as a unit). `ConstrainMoveIndex` unchanged — dragging is free. Inserts into an explicit group (link opened from a grouped tab) keep their index so groups stay contiguous.

**Redraw:** sticky state changes trigger `WebContents::NotifyNavigationStateChanged(INVALIDATE_TYPE_TITLE)` — same path as page title changes; refreshes horizontal/vertical tabs, tooltip, hover card.

No backups: Preferences writes are atomic; losing the file means losing the whole profile.
