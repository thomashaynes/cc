# Weekly Shop: specification

Source of truth for the Weekly Shop app. Read this before changing anything. The current build matches it. Sections marked "Open decisions" are not decided, so ask before assuming.

## 1. Purpose

A phone-first checklist for one person's weekly online grocery shop. It builds a basket checklist from regular items, possibly needed items, one-off extras and chosen meals. The user ticks items off as they add them to the online basket. After the shop, the app shows next week's list and this week's meals.

## 2. User and context

- One user. No sharing. An optional Google sign-in is used only for sync.
- Used on a phone, usually one-handed, while switching between this app and a supermarket site. Taps must be large and forgiving.
- Installed to the home screen. It opens full screen with no browser bar.
- Used on an Android phone and a Windows computer, both in Chrome, with the same data on each.

## 3. Scope

In scope: regulars, possibles, extras, meals with three item types, basket checklist, finish shop, this week's meals, admin, backup and restore, offline use, sync between devices through Google Drive.

Out of scope for now: multiple users, quantities, prices, supermarket integration, notifications, dark mode.

## 4. Delivery

- Static progressive web app. No build step, no dependencies, no backend. The one outside script is Google Identity Services, loaded only when sync is set up (section 10a).
- Files: `index.html` (markup, CSS and JS in one file), `sw.js`, `manifest.webmanifest`, `icon-180.png`, `icon-192.png`, `icon-512.png`.
- Hosted on GitHub Pages from `thomashaynes/cc` at https://thomashaynes.github.io/cc/. Any static host over HTTPS works, but the Google OAuth client must list the host as an authorised JavaScript origin.
- Installs through Add to Home Screen (iPhone, Safari) or Install app (Android, Chrome).
- Works offline after the first load.

## 5. Concepts

An item is a plain string. Names compare case-insensitively after trimming.

| Concept | Meaning |
|---|---|
| Regular | On the list every week. |
| Possible | Shown each week in Plan. On the list only when the user marks it needed. |
| Extra | One-off item for the list being built. Cleared when the shop finishes. |
| Meal | A name plus three kinds of item (below). |
| Plan | The list currently being built. |
| This week | A snapshot of the meals from the last finished shop. |

A meal has:

- **Always**: added whenever the meal is in the plan.
- **If out**: added only when the user marks the item needed for that meal in the plan. Default is not needed.
- **Choices**: groups with a label and options. The user picks one option, or None. A picked option is added. An unpicked group adds nothing and shows "not chosen". Tapping the selected option again clears it.

## 6. Data model

Stored in `localStorage` under `weeklyshop.v1` as one JSON object:

```json
{
  "updatedAt": "2026-10-23T17:31:02.000Z",
  "examples": false,
  "regulars": ["Semi-skimmed milk"],
  "possibles": ["Toilet roll"],
  "meals": [
    {
      "id": "k3j9x2a",
      "name": "Spaghetti bolognese",
      "always": ["Beef mince"],
      "ifOut": ["Olive oil"],
      "groups": [{ "label": "Pasta", "options": ["Spaghetti", "Tagliatelle"] }]
    }
  ],
  "plan": {
    "meals": ["k3j9x2a"],
    "need": { "p|Coffee": true, "m|k3j9x2a|Olive oil": true },
    "picks": { "k3j9x2a|Pasta": "Tagliatelle" },
    "extras": ["Birthday candles"],
    "ticks": { "beef mince": true },
    "skips": { "bread": true }
  },
  "thisWeek": {
    "shoppedAt": "2026-10-23T17:30:00.000Z",
    "meals": [{ "id": "k3j9x2a", "name": "Spaghetti bolognese", "detail": "Tagliatelle", "done": false }]
  }
}
```

Key formats:

- `need`: `p|<possible name>` or `m|<meal id>|<if-out item name>`.
- `picks`: `<meal id>|<group label>`. The value is an option name or `__none`.
- `ticks`: lowercase item name.
- `skips`: lowercase item name. Items removed from this basket without being bought.
- `thisWeek` is `null` until the first shop finishes.
- `updatedAt` is set on every local change. It is an empty string for untouched example data. Sync uses it to decide which copy is newer.

UI state sits in `weeklyshop.ui`: current tab, Plan or Basket mode, admin sub view. Nothing else.

Sync state sits in `weeklyshop.sync`: whether sync is on, whether this device has linked yet, the Google access token and its expiry, and the last sync time. It never syncs itself.

On load, `sanitize()` coerces anything stored into this shape. Keep it that way so old or hand-edited data cannot break the app.

First run with no stored data loads example data (`examples: true`) with two meals already planned. A banner offers Clear examples.

## 7. List building rules

`buildBasket(state)` returns an ordered list of `{ key, name, section, notes[] }`.

1. Add regulars (section "Regulars").
2. Add possibles marked needed (section "Needed").
3. Add extras (section "Needed").
4. For each planned meal, in plan order, add always items, needed if-out items, then picked options (section "For meals"). Skip a pick if it is `__none` or no longer one of the group's options.
5. Merge duplicates by lowercase name. The first section to add an item keeps it. When more than one source wants an item, `notes` lists every meal that wants it, shown under the item. An item with a single source has no notes.
6. Skip meals that no longer exist.

Sections display in this order: Regulars, Needed, For meals. Empty sections do not show.

`buildBasket` ignores `skips`. The Shop list view splits its result into the live basket and the removed list. Counts and progress use the live basket only.

## 8. Screens

A fixed bottom tab bar has three tabs: **Shop list**, **This week**, **Admin**. The active tab has a coral bar on top.

### 8.1 Shop list

Shows the example banner when `examples` is true. A two-way switch sits below: **Plan** and **Basket n/total** (n ticked of total items).

**Plan** sections, top to bottom:

1. **Meals.** Each meal is a checkbox row. A ticked meal expands underneath to show its if-out items as toggle chips and each choice group as single-select chips, plus a None chip. Meals with neither show a note that everything is added automatically.
2. **Possibly needed.** Toggle chips.
3. **Extras.** Text input with Add (Enter also adds), list with Remove. Focus stays in the input after adding.
4. **Regulars.** Read-only comma list.
5. A full-width button to view the basket, showing the item count.

**Basket:**

- Progress: ticked count in coral at 44px, label "of N in basket", thin progress bar.
- Items in sections with a ticked/total count per section.
- Each row is a checkbox button. Ticked rows stay in place, struck through and muted, so positions never jump while the user checks the supermarket site.
- Each row has an × button (44px) on the right. It removes the item from this basket without ticking it and clears any tick. The item moves to a **Removed** list below the sections, with a count and a Put back button per item. Put back returns it unticked to its usual place.
- **Finish shop** asks for confirmation inline. The text says how many items are unticked. Confirming does this:
  - `thisWeek` becomes a snapshot of the planned meals. `detail` is the picked options joined by commas.
  - `plan` resets to empty (no meals, needs, picks, extras, ticks or removed items). Regulars and possibles stay.
  - The app switches to the This week tab and Plan mode.
- Empty basket shows a short instruction instead, plus the Removed list if it has items.

### 8.2 This week

- Before the first finished shop: a message saying meals appear here once a shop is finished.
- After: header shows the shop date in en-GB ("Friday 23 October"). Progress "x of N meals cooked". Each meal is a checkbox row with its picked options as a muted note. Ticking marks it cooked.

### 8.3 Admin

A three-way switch: **Regulars**, **Possibles**, **Meals**.

- **Regulars and Possibles:** text input with Add (Enter also adds), list with Remove. Duplicates (case-insensitive) and blank entries are ignored.
- **Meals:** list showing each meal's counts ("3 always, 3 if out, 1 choice") with Edit. Add a meal button.
- **Meal editor:**
  - Name (required).
  - Always needed: textarea, one item per line.
  - Only if you are out: textarea, one item per line.
  - Choose one: repeatable groups, each with a name and an options textarea (one per line). Add a choice and Remove choice buttons.
  - Save, Cancel, and Delete (inline confirm, only for existing meals). Deleting also removes the meal from the plan.
  - Validation: a name is required. A group needs both a name and at least one option, or must be fully empty (empty groups are dropped). Group names must differ, ignoring case. Duplicate lines within a list are removed on save.
  - Typing updates an in-memory draft and does not re-render, so the keyboard and focus stay put.
- **Sync** (hidden while editing a meal, and hidden entirely until a client ID is set): Connect Google Drive, with a note that a list already in Drive replaces the one on this device. Once on: a status line ("Synced with Google Drive at 14:02."), Sync now or Reconnect, and Disconnect.
- **Backup** (hidden while editing a meal): Copy backup puts the full JSON on the clipboard. If the clipboard is blocked, the JSON shows in a read-only textarea to copy by hand. Restore opens a paste box. Pasted text must parse as JSON with `meals` and `regulars` arrays, or the app shows an error and keeps current data. A valid backup replaces everything through `sanitize()`.

## 9. Design

Brand: Effective Strategy (Thomas Haynes). Flat, opaque, square, generous whitespace.

| Token | Value | Use |
|---|---|---|
| `--bg` | `#FFF5F0` | Page background |
| `--fg` | `#1A1A1A` | Text, borders, primary button |
| `--accent` | `#E8745E` | Ticks, selected chips, progress, stats, link underlines |
| `--accent-hover` | `#C85A45` | Hover only |
| `--muted` | `#666666` | Secondary text |
| `--shade` | `#F5E6DD` | Banner, progress track |
| `--deep` | `#EAD3C6` | Row dividers |

- Fonts: Libre Baskerville Bold for the page heading (34px, coral full stop). Inter for everything else (17px body, 15px secondary, 14px notes). Load from Google Fonts with system fallbacks. The service worker caches them.
- Corners: square. Exceptions: buttons 2px, inputs 4px, chips fully rounded.
- No shadows, gradients, emoji or icons. One exception: the × character on basket rows, which the user asked for. The checkbox is a 28px square with a CSS-drawn tick on a coral fill.
- Sentence case everywhere. UK English.
- Tap targets 44px or more. Rows 60px minimum. Inputs 17px so iOS does not zoom.
- Single light theme. The brand has no dark palette.
- Layout: one column, 560px maximum, 16px side gutters, safe-area insets on all edges. Content needs bottom padding so the tab bar never covers the last row.
- Respect `prefers-reduced-motion` (the only animation is the progress bar width).
- Visible focus outline on every control.

## 10a. Sync

- Google Identity Services gives an access token for the `drive.appdata` scope. The app can only see its own hidden folder in the user's Drive, not their files.
- Data lives in one file, `weeklyshop.json`, in that folder. It holds the same JSON as `weeklyshop.v1`.
- A sync lists the file, downloads it and compares `updatedAt`. The later copy wins: newer Drive data replaces local data, newer local data is uploaded. No file yet means upload.
- On a device's first link, the Drive copy wins whatever the dates, so a fresh device cannot overwrite real data with examples.
- Syncs run on start, after each change (1.5 seconds after the last one), when the app comes back into view, when the device comes back online, and every minute while the app is visible.
- Google tokens last one hour and need a tap to renew, because browsers block the sign-in window without one. When the token has expired, a bar at the top of every tab says "Not synced. Tap to reconnect Google Drive." A failed sync shows "Sync failed. Tap to try again."
- Disconnect revokes the token and stops syncing. Local data stays.
- Offline, the app works as before and syncs once back online.
- Setup: a Google Cloud project with the Drive API enabled, an OAuth consent screen with the owner as a test user, and a web OAuth client whose authorised JavaScript origin is the host. Its client ID goes in `GOOGLE_CLIENT_ID` in `index.html`. Client IDs are not secret.

## 10. Offline and updates

`sw.js` has a `CACHE` constant. Change it on every release so installed copies update.

- Page loads: network first, fall back to the cached `index.html`.
- Own files and Google Fonts: cached copy first, refreshed in the background.
- Add new files to the `SHELL` list in `sw.js`.
- Google sign-in and Drive requests are not cached. They go straight to the network.

## 11. Implementation notes

- Vanilla JS in one IIFE. HTML comes from string templates. Escape every user string with `esc()`, including inside attributes.
- Interaction uses event delegation. Controls carry `data-act` plus data attributes (`data-id`, `data-name`, `data-key`, `data-i`, `data-list`, `data-v`). One `act()` switch handles every action.
- Every state change calls `commit()`: save, then re-render the active view and restore scroll position. Tab and mode changes scroll to the top.
- Each rendered row is a `<button>` with `role="checkbox"` and `aria-checked`.
- `window.__buildBasket` is exposed for tests. Remove it if a build step is ever added.

## 12. Acceptance checks

These pass today in a simulated browser (jsdom). Keep them passing.

1. First load shows Shop list with the example banner.
2. Always items appear. If-out items do not, until marked.
3. Marked if-out items, picked options, needed possibles and extras all appear.
4. Unticking a meal removes its items.
5. An item wanted by two sources shows once, with the meal names as notes.
6. Ticking an item updates the progress count.
7. Finish shop asks first and reports unticked items.
8. Finishing moves meals to This week with their picks, and clears the plan.
9. A meal can be ticked cooked on This week.
10. The new list starts with nothing ticked and no meals.
11. The meal editor rejects a choice with no options, then saves once it has them. Duplicate lines are removed.
12. Clear examples empties all data.
13. Backup restores correctly. A bad backup is rejected and leaves data alone.
14. State survives a reload.
15. No script errors.

Sync checks, run against a mocked Google sign-in and Drive:

16. Connecting with no Drive file creates one in the app folder.
17. A change uploads after the delay. Another device picks it up when it comes back into view. The later save wins.
18. A device that changed its data before its first link still takes the Drive copy and does not overwrite Drive.
19. An expired token or a 401 shows the reconnect bar. Tapping it renews without the consent screen. Disconnect stops uploads and keeps local data.
20. Removing a basket item takes it out of the list and the counts, clears its tick and shows it under Removed. Put back restores it unticked.

Still to verify on a real phone: layout at 360 to 430px wide, tab bar and safe areas, install flow on iPhone and Android, offline launch, and clipboard behaviour in standalone mode.

## 13. Known gaps

- Items cannot be renamed or reordered. Remove and re-add.
- Renaming a choice group clears its pick for that meal.
- Without sync, data lives in one browser on one device. Clearing site data erases it. Backup is manual.
- Sync is whole file, last save wins. Changes made on two devices between syncs do not merge; the older set is lost.
- After an hour, sync needs a tap to reconnect.
- No undo.

## 14. Open decisions

Not decided. Ask before building any of these.

- Quantities or notes per item.
- Ordering or grouping the basket by supermarket aisle.
- Assigning meals to days.
- Whether extras or needed possibles should carry over between weeks.
- Whether to wrap it as a native app later.
- Whether to start empty instead of with example data.
