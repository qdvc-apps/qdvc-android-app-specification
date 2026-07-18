# Specification: Folder-Backed Plaintext Editor Apps for Android

**A reusable blueprint for native Android apps that open a folder of
plaintext-serialisable files and provide a friendly interface to view and edit
them.**

This document is a general specification for building a family of comparable
apps — e.g. notebooks, task lists, recipe collections, config editors — that
share the same shape: point the app at real device folders, browse a hierarchy,
open items into a multitasking switcher, and view/edit them with theming and
fonts under the user's control. Where it gives concrete examples, it draws them
from a notebook-style app that opens folders of Markdown notes; those examples
illustrate the general pattern and are called out as such.

It has two halves, deliberately separated so that design and engineering can
evolve independently:

- **Part A — User experience.** What the app looks like and how it behaves, in
  terms a designer or product owner can review, with no implementation detail.
- **Part B — Technical implementation.** How Part A is realised in Kotlin +
  Jetpack Compose on Android, with the specific patterns and pitfalls that carry
  the design's guarantees.

Throughout, the term **item** denotes whatever the app opens and edits (a note,
a task file, a recipe); **structure** denotes the organising surface (a folder
tree, a tag list); and **workspace** denotes a user-granted device folder that
holds items.

The two goals of this document are (1) to align existing and future apps on a
shared set of best practices, reducing backend technical debt and giving users
a consistent experience, and (2) to make new apps of this style fast to build.

---

# Part A — User-facing experience

This part is intentionally free of technology choices. It describes components,
layout, motion, and behaviour only.

## A1. Overall shape

The app is a single-screen application with a **persistent bottom navigation
bar of four items**. The four items are de-facto tabs. The upper region of the
screen is a **toolbar** (title, optional back button, optional context
actions); the middle is the active tab's content; the bottom is the navigation
bar.

The bottom bar has a fixed meaning for its first and last items, and a
deliberately open meaning for the two in the middle:

1. **Item 1 — Home / structure explorer (prescribed).** The starting point and
   de-facto homepage. It presents the app's structures hierarchically and lets
   the user drill down into them to find items to open.
2. **Item 2 — App-specific surface (not prescribed).** Its meaning depends on
   the app. A common choice is a read-only **View** of the current item.
3. **Item 3 — App-specific surface (not prescribed).** As above. A common
   choice is an editable **Edit** view of the current item.
4. **Item 4 — Multitasking switcher (prescribed).** A switcher between open
   items, equivalent to a browser's tab bar. Items are opened from Item 1's
   structures and closed here.

Items 2 and 3 are the app author's to define. They will typically be surfaces
that act on **the currently-selected item** — for example a read-only View and
an editable Edit of the current item — but the spec does not require this; an
app could instead make them alternative top-level structure browsers (say, a
tag list and a favourites list). What the spec *does* require is that Item 1 is
a home/browse surface and Item 4 is a multitasking switcher.

Where a surface (such as Item 2 or 3) requires a selected item and none is
open, its bottom-bar item is shown **disabled** rather than hidden, so the bar's
shape never changes.

## A2. Item 1 — the home / structure explorer

### Hierarchy and the home root

Structures are hierarchical and are traversed by drilling in and stepping back
out. The **top of the hierarchy is the app's home screen** — the first thing a
new user sees and the anchor everything else hangs from.

A representative hierarchy — the one a notebook-style app would use — looks
like this:

```
Workspace list  (the home root)
  └─ Workspace overview  (a menu: Browse files · All items · Search · Index status)
       ├─ Folder browser  (drills arbitrarily deep through subfolders)
       ├─ All items       (every item in the workspace, flat)
       ├─ Search          (full-text search over the workspace)
       └─ Index status    (state of the search index)
```

Your app's hierarchy may differ, but the pattern holds: a **home root** (a list
of workspaces, or the sole workspace's contents), an optional **overview/menu**
level offering different ways into the same data, and then the **leaf browsers**
that lead to openable items.

### Back navigation (a hard rule)

- Whenever the toolbar shows a **back button**, the **Android system back
  button must do exactly the same thing** — step up one level in the current
  surface — and must **not** close the app or do anything else.
- The app only closes on system-back when the user is at the home root, where
  there is no toolbar back button.
- This rule is universal: every screen that shows a toolbar back arrow routes
  system-back through the *same* handler that drives the arrow, so the two can
  never diverge.

### Re-tapping Item 1

Tapping the Item 1 bottom-bar icon **while already on Item 1** jumps straight
back to the top of the hierarchy (the home root), regardless of how deep the
user has drilled. This is the quick "go home" gesture.

### Settings entry point

Because the home root is the app's homepage, the **entry to Settings lives
there**: a menu (overflow) button in the **top-right corner of the toolbar**,
shown only at the home root. Settings is reachable only from here — combined
with the re-tap gesture above, the user is always at most two taps from
Settings (tap Item 1 to go home, tap the menu).

### Motion

Movements through the hierarchy are animated as a **horizontal slide**:
drilling in slides the new level in from the right while the old level eases
left; stepping back reverses it. The animation must be a clean horizontal
slide — it must **not** drift diagonally from a corner (a known failure mode;
see B7).

### Item actions at the home root

The home root offers a clear affordance to **add a workspace** (grant a new
device folder) and, per workspace row, to **remove** it from the list (removing
only the app's pointer to that folder — never deleting the user's files, with a
confirmation dialog that says so).

## A3. Items 2 & 3 — app-specific surfaces

Not prescribed. The most common and recommended choice is a **View / Edit**
pair acting on the current item:

- **View** — a read-only rendering of the current item. Content is presented in
  a fixed-width, single-size layout where structure (headings, emphasis, lists)
  is conveyed through **colour and weight, never through changes in text size**.
  Text is selectable and copyable.
- **Edit** — the same item, editable. A toolbar **Save** action appears **only
  when there are unsaved changes**. While editing, the same syntax colouring is
  applied live to the raw source, so the character grid stays stable and the
  cursor lands where the user expects.

Both View and Edit have **independent font and text-size settings** (see A5).
Both are disabled in the bottom bar until an item is open.

An app in this family is free to give Items 2/3 entirely different meanings, but
should keep the same conventions: a clear toolbar, context actions that appear
only when relevant, and consistent theming.

## A4. Item 4 — the multitasking switcher

The switcher is the phone equivalent of a desktop tab bar.

- It **lists every open item**, marking the current one distinctly (e.g. a dot
  and heavier weight), and shows each item's secondary context (e.g. which
  workspace it belongs to). An item with unsaved edits is marked (e.g. a
  trailing bullet).
- **Tapping** a row makes that item current and takes the user to the
  appropriate surface (e.g. the read-only View, when Items 2/3 are a View/Edit
  pair).
- **Opening** a new item is done from Item 1 (drill to an item and tap it); it
  then appears here and becomes current.
- **Closing** uses a **swipe-to-close** gesture (like the swipe-to-archive
  gesture familiar from mobile email apps). The revealed action must show an
  unmistakable **Close affordance — an X icon above a "Close" label** — not a
  bare coloured rectangle. (The label must be fully visible; a past bug clipped
  it — see B6.)
- Closing an item **with unsaved changes** prompts a warning with **Cancel**
  (keeps the item) and **Close anyway** (discards edits, styled in the error
  colour).
- A toolbar button toggles a **reorder mode**. In reorder mode each row exposes
  **up/down arrows** to move it; the toggle button changes to a confirm
  ("done") affordance. (Drag-to-reorder is a possible future enhancement but is
  explicitly out of scope for now.)
- Empty state: a friendly message telling the user to open something from Item
  1.

The open-item list and the current selection **persist across app restarts**.
On relaunch the list is restored and each item's content re-read from its file;
items whose files have vanished are silently dropped.

## A5. Settings

Settings is a **menu-based, hierarchical** screen: a root list of rows, each
with a **leading icon** and a title (and a subtitle showing the current value),
that navigate into sub-pages. Navigation between Settings pages uses the **same
horizontal slide** animation as Item 1, and obeys the **same back-button rule**
(the toolbar back arrow and system back share one handler; at the Settings root,
back closes Settings).

A typical Settings screen for an app in this family contains:

- **Appearance** — choose Automatic / Light / Dark.
- **Light Mode Style** and **Dark Mode Style** — choose which colour theme is
  used in each mode, from the installed themes (see A6).
- **Fonts** — an independent font choice for each item surface that renders text
  (for a View/Edit pair, one for each). Each offers: the app **default**, any
  **device-installed font**, or a **custom font** (see below). Font rows preview
  themselves in the font they name.
- **Font size** — an independent size for each such surface, adjusted with −/+
  steppers between sensible bounds, with a one-tap **Reset** to default.

### Custom fonts (the four-variant pattern)

When an app supports custom fonts, it follows this exact pattern:

- Each font context (e.g. View and Edit) has **four variant slots**:
  **regular, italic, bold, bold-italic**.
- The user picks **one file per slot**. The chosen file is **copied into the
  app's own storage** (so it cannot vanish if the user later moves or deletes
  the source), and the **original file name is shown** back to the user as the
  slot's label.
- Re-picking a slot overwrites it; a per-slot **clear** action empties it.
- Bold/italic content then renders in the matching face. If only some slots are
  filled, the app degrades gracefully (falling back to the default until at
  least a regular file is set), and tells the user so.

## A6. Theming and colour

- **Light / dark / automatic** mode selection (automatic follows the system).
- **Colour themes** are selectable independently for light and dark mode. Ship a
  sensible spread: at least one default light and dark theme, and it is worth
  including a **pure-black OLED** dark theme (background `#000000`) for
  battery/contrast on OLED screens. Adding a new theme is a data change, not a
  code change (see B5), so shipping several named themes costs almost nothing.
- Syntax/structure highlighting **derives its colours from the active theme**,
  so a new theme restyles content automatically with no extra work.
- **System-bar colour matching.** The Android **status bar** (clock,
  notifications, battery) and **navigation bar** (back/home/recents) are
  coloured to match the app's own bars, so the system chrome blends into the
  app. The approach to use: give the app's top toolbar and bottom navigation bar
  a single shared surface colour, and set both system bars to that same colour;
  then let the light/dark setting drive the contrast of the system-bar icons so
  they never render as plain black on a dark app or vice-versa. (Using one
  colour for both app bars is what lets a single colour match both system bars;
  it is a deliberate simplification and part of what makes the whole top-and-
  bottom of the screen read as one continuous surface.)

## A7. Full-text search and its index

When an app offers full-text search over a workspace:

- Search matches both **item titles** (file names) and **body contents**,
  case-insensitively, and shows a short **snippet** around the match with the
  query terms **emphasised**.
- Search is backed by an **on-device index** so it stays fast on large folders
  and does not re-read every file on every query.
- The index is a **cache, never a source of truth**: if it is missing or fails,
  search and the "all items" list fall back to a live folder scan, so the
  features keep working (just slower).
- The index has a **user-facing status surface** showing whether it is *not
  built*, *building* (with live progress — the current file and a rising
  count), or *ready* (with the count and the time it last rebuilt), plus a
  **Regenerate now** button. The status surface makes clear that regenerating
  touches only the app's private index and never the user's files.
- Freshness: the index refreshes quietly in the background on launch; an in-app
  save updates that item's entry immediately; edits made in other apps are
  picked up on the next launch.

## A8. Cross-cutting UX conventions

- **The user's files stay put.** The app edits files in place in the folders the
  user grants; it never copies items into a private silo or locks them away.
  Destructive-sounding actions (remove workspace, close with unsaved changes)
  are confirmed and clearly worded.
- **Consistent list rows.** Lists use a consistent row: leading icon, primary
  text, optional secondary text, optional trailing action, and a divider.
- **Consistent empty states.** Every list/surface that can be empty shows a
  short, friendly message telling the user what to do next.
- **Restraint in motion.** One animation vocabulary (the horizontal slide) is
  reused for all hierarchical navigation, so the whole app feels of a piece.

---

# Part B — Technical implementation

This part maps Part A onto concrete Android + Kotlin technologies. It is
prescriptive about the patterns that carry the design's guarantees, and it flags
the traps that are easy to fall into — several of these are non-obvious and have
each cost real debugging time to diagnose, so they are documented here with
their root cause rather than just their fix.

## B0. Platform, language, and build

- **Language / UI:** Kotlin with **Jetpack Compose** and **Material 3**. No XML
  layouts; the only XML is the manifest, launcher resources, and a minimal base
  theme.
- **Single Activity.** One `ComponentActivity` hosts everything; tabs and
  screens are Composables, not Activities or Fragments.
- **SDK levels:** `minSdk = 26` (Android 8.0), `compile`/`targetSdk = 34`,
  **JDK 17**.
- **Build:** Gradle with the Kotlin DSL. Compose is enabled via
  `buildFeatures.compose` with a pinned `kotlinCompilerExtensionVersion`
  matched to the Kotlin version. A known-good combination is Kotlin `1.9.24`
  with compiler extension `1.5.14` and the Compose BOM `2024.06.00`. Note the
  version coupling: on Kotlin 1.9.x the compiler extension is a separate pinned
  version as shown; if you move to Kotlin 2.0+, the mechanism changes — drop the
  extension version and apply the `org.jetbrains.kotlin.plugin.compose` plugin
  instead.
- **Key dependencies:** Compose BOM (ui, graphics, material3,
  material-icons-extended, foundation); `activity-compose`;
  `lifecycle-viewmodel-compose`; **DataStore Preferences** (settings);
  **DocumentFile** (SAF helpers); **Room** with the **KSP** compiler (search
  index). No third-party markdown or persistence library is required.
- **Manifest:** the hosting Activity sets
  `android:windowSoftInputMode="adjustResize"` so editable surfaces stay above
  the keyboard, and the app enables edge-to-edge.

## B1. Architecture: unidirectional state, one ViewModel

State flows one way and lives in one place.

- A single `AppViewModel` (an `AndroidViewModel`) **owns all application state**
  as `StateFlow`s and exposes plain callbacks to mutate it.
- The Activity **collects** those flows and passes plain values + callbacks down
  into **stateless Composable screens**. Screens hold only transient UI state (a
  text field's contents, which sub-page is open) — never anything that must
  survive a tab switch or process death.
- Repositories sit behind the ViewModel, one per concern:
  - a **settings repository** (DataStore) for preferences and persisted session
    state,
  - an **item repository** (SAF) for listing/reading/writing/creating items,
  - an **index repository** (Room) for fast list/search,
  - a **theme repository** for loading themes.

Reading rule for maintainers: *what state exists* → read the ViewModel; *how
something is stored or fetched* → read the matching repository; screens are just
projections of that state.

Recommended package layout:

```
app/
  MainActivity.kt          host; edge-to-edge; tab switching; pickers; back handling
  AppViewModel.kt          all UI state; the Item-1 navigation state machine
  model/                   plain data types + enums, no Android dependencies
  data/
    SettingsRepository.kt  DataStore preferences + persisted session
    ItemRepository.kt      all SAF access
    ThemeRepository.kt     loads/caches JSON themes from assets
    IndexRepository.kt     Room FTS index: list, search, background sync
    index/                 Room entities, DAO, database
  util/                    highlighter, font discovery/slot management
  ui/
    theme/                 ColorScheme construction from a theme spec
    components/             bottom bar, shared rows, transformations
    <surface>/             one package per bottom-bar surface + settings
```

## B2. Navigation and back handling

- **Bottom bar:** a Material 3 `NavigationBar` with four `NavigationBarItem`s
  keyed off a `Tab` enum. Surfaces that need a selected item pass
  `enabled = false` when none is open. Set the bar's `containerColor` to
  `surface` and `tonalElevation = 0.dp` so it matches the top app bar (this is
  what makes the single-surface system-bar match in B4 work).
- **Item-1 hierarchy as a state machine.** Model the home surface's levels as an
  enum (e.g. `WORKSPACES, OVERVIEW, FOLDERS, ALL_ITEMS, SEARCH, INDEX_STATUS`)
  held in a `BrowseState` in the ViewModel, with a folder **stack** for the
  arbitrary-depth level. A single `browseUp()` function encodes every back-step;
  it returns whether it consumed the action.
- **The back-button rule, enforced structurally.** For each surface that shows a
  toolbar back arrow, wire the arrow's `onClick` and an
  `androidx.activity.compose.BackHandler` to **the same lambda**. Never give a
  screen an independent `BackHandler` that does something different from its
  arrow. On Item 1, enable a `BackHandler` exactly when
  `currentTab == item1 && mode != homeRoot`, calling `browseUp()`. In Settings,
  the toolbar arrow and a `BackHandler` share one `goBack` lambda that steps up
  a page or calls `onClose()` at the root.
- **Re-tap to go home:** in the bottom bar's `onSelect`, detect
  "tapped Item 1 while already on Item 1" and reset the hierarchy state to its
  root.
- **Insets:** enable `enableEdgeToEdge()`. Let each screen's `TopAppBar` consume
  the top status-bar inset, and apply only the **bottom** inset at the
  `Scaffold` level, or you will double the top padding.

## B3. The Storage Access Framework (read before touching file access)

This is the most constraint-driven part of the app and the easiest to break
with a "cleaner" rewrite.

- A **workspace is a tree URI** the user granted via
  `ActivityResultContracts.OpenDocumentTree()`. Immediately call
  `takePersistableUriPermission(...)` with read+write so the grant survives
  reboots. All access goes through `ContentResolver`, **never** `java.io.File`.
- **Stay inside the granted tree.** Address a folder by its **document id within
  that tree**, and get its children with
  `DocumentsContract.buildChildDocumentsUriUsingTree(treeUri, docId)`. **Do not
  fabricate a fresh tree URI for a subfolder** — a tree URI the user never
  granted carries no permission and silently returns zero children. Thread
  `rootTreeUri` + `docId` around rather than passing self-contained subfolder
  tree URIs.
- **SAF calls are IPC** to the storage provider and are far costlier than
  ordinary file reads; cost scales with the number of files/folders. This
  expense is the entire reason the search index exists — avoid any feature that
  walks the whole tree on a hot path.
- Split scans into a **cheap half** (list `documentId`, `displayName`,
  `lastModified`, `size` only — never bodies) and **body reads** done one file
  at a time only when needed.
- Do every SAF operation on `Dispatchers.IO` and **catch exceptions per file**,
  so one unreadable file can't abort a whole scan.
- Reads and writes: `contentResolver.openInputStream(uri)` and
  `openOutputStream(uri, "wt")` (the `"wt"` truncates so the file is fully
  replaced). Create items with `DocumentsContract.createDocument(...)`. Check
  existence with `DocumentFile.fromSingleUri(...).exists()` when restoring a
  session.

Adapting to a different item type: the only file-type-specific parts of the SAF
layer are the predicate that decides which files count as items (e.g. an
extension filter such as `.md`/`.markdown` for a notebook app) and the MIME type
passed when creating a new item. Everything else — tree traversal, reads,
writes, existence checks — is identical across apps in this family.

## B4. System-bar colour matching

In a `SideEffect` inside the themed content, set both bars to the app's surface
colour and drive icon contrast from the resolved light/dark boolean:

```kotlin
val barColor = MaterialTheme.colorScheme.surface.toArgb()
val window = (view.context as Activity).window
window.statusBarColor = barColor
window.navigationBarColor = barColor
val controller = WindowInsetsControllerCompat(window, view)
controller.isAppearanceLightStatusBars = !darkTheme   // light theme → dark icons
controller.isAppearanceLightNavigationBars = !darkTheme
```

Resolve `darkTheme` from the theme mode: `AUTOMATIC → isSystemInDarkTheme()`,
else the explicit choice. Because the top app bar and bottom `NavigationBar`
both use `surface`, one colour matches both system bars.

## B5. Theming (JSON themes as data)

- Each theme is a **plain JSON file** in `assets/themes/`, one file per theme,
  with an `id`, a display `name`, a `dark` boolean, and a `colors` object of hex
  strings for the handful of Material roles the app uses (`background`,
  `surface`, `surfaceVariant`, `onBackground`, `onSurfaceVariant`, `outline`,
  `primary`, `onPrimary`, `secondary`, `onSecondary`, `error`).
- A `ThemeRepository` lists the assets directory, parses each file with
  `org.json` (built into Android — no dependency), **skips malformed files**,
  caches the result, and sorts them. Light themes populate the Light Mode Style
  list, dark ones the Dark Mode Style list, keyed off `dark`.
- A theme spec becomes a Compose `ColorScheme` by starting from
  `lightColorScheme()`/`darkColorScheme()` and `copy()`-ing the roles across.
  Map both `surface` and `onSurface` sensibly; a simple, robust choice is to
  point `onSurface` at the theme's `onBackground` value (so text reads correctly
  on both `background` and `surface`, which are close in these themes) rather
  than requiring the JSON to specify an `onSurface` separately.
- **Adding a theme is a data change:** drop in a new JSON file and it appears
  automatically; the selected theme is persisted by `id`, with defaults for
  light and dark. A pure-black OLED theme is just a dark theme whose background
  is `#000000`.
- Keep syntax colours **derived from `MaterialTheme.colorScheme`** (see B8) so a
  new theme restyles content for free.

## B6. The multitasking switcher (swipe-to-close + reorder)

- Render the list with a `LazyColumn` keyed by a stable item identity.
- **Swipe-to-close:** place the red close action *behind* the row and translate
  the foreground row horizontally to reveal it. The revealed strip must contain
  an **X icon above a "Close" label**, centred in the strip.
  - **Unit-mismatch trap:** the reveal strip is sized in **dp** but
    `graphicsLayer { translationX = ... }` is in **pixels**. If you slide the row
    by the dp *number* interpreted as pixels, then on a typical ~2.75× density
    screen a 96 dp strip is exposed by only ~35 px — a sliver — and the "Close"
    label is clipped, so the action reads as a bare coloured rectangle. Convert
    with `LocalDensity`:
    `val revealPx = with(LocalDensity.current) { revealWidthDp.toPx() }`, and
    translate by `revealPx` so the exposed strip exactly matches the action's
    width.
- **Unsaved-changes guard:** closing an item whose draft differs from its saved
  content opens an `AlertDialog` with **Cancel** and **Close anyway** (the
  latter tinted `error`).
- **Reorder mode:** a toolbar `IconButton` toggles a local `reordering` flag
  (icon swaps between a reorder glyph and a confirm check). In reorder mode,
  disable swipe/tap and show per-row **up/down** `IconButton`s guarded by
  `canMoveUp`/`canMoveDown`; each calls a ViewModel `move(from, to)`.
- **Session persistence:** persist only item **identity** (uri, display name,
  context label) and the current selection to DataStore whenever the list or
  selection changes. On launch, restore the list, re-read each file's content,
  and drop any whose file no longer exists.

## B7. Hierarchical navigation animation (avoid the diagonal drift)

Use `AnimatedContent` with a horizontal slide, keyed by navigation **depth** (or
mode) so direction can be derived (`targetState > initialState` ⇒ going deeper):

```kotlin
val deeper = target > initial
val spec = if (deeper)
    (slideInHorizontally(tween(280)) { it } + fadeIn()) togetherWith
        (slideOutHorizontally(tween(280)) { -it / 4 } + fadeOut())
else
    (slideInHorizontally(tween(280)) { -it / 4 } + fadeIn()) togetherWith
        (slideOutHorizontally(tween(280)) { it } + fadeOut())
spec.using(SizeTransform(clip = false) { _, _ -> snap() })
```

- **Diagonal-drift trap:** `AnimatedContent`'s default `SizeTransform`
  **animates the container's height** while the horizontal slide runs, so when
  consecutive screens have different content heights the content appears to
  drift in diagonally from a corner instead of sliding straight across. Snapping
  the size (`SizeTransform { _, _ -> snap() }`) makes the size change instantly,
  so only the clean horizontal slide animates. This is the single most common
  cause of a "janky" hierarchy transition and is easy to miss because it only
  shows up when adjacent screens differ in height.
- Reuse this identical spec for **both** Item-1 navigation and Settings
  navigation so all hierarchy motion matches.

## B8. Content rendering / syntax highlighting

The concrete example here is Markdown highlighting, but the pattern generalises
to any structured plaintext (task syntax, config keys, wiki markup):

- **Colour and weight only, never size.** Communicate structure by colouring and
  weighting tokens; keep one font size so the character grid is stable. Force
  every run to the user's chosen `FontFamily` so bold/italic runs can't fall
  back to a different face.
- **Derive colours from the theme** via a small `SyntaxColors` mapped from
  `MaterialTheme.colorScheme` (headings ← primary, code ← secondary, etc.), so
  themes restyle highlighting automatically.
- **Two rendering paths that must not be merged** (a subtle, documented trap):
  1. **Read-only surface (View):** may use a per-line `ParagraphStyle` with a
     `TextIndent` for hanging indents. Because a Compose `ParagraphStyle`
     behaves "as if it had line feeds at its start and end", this path must
     **not** also emit literal `\n` between lines, or every line double-spaces.
  2. **Editable surface (Edit):** rendered through a `VisualTransformation` on a
     `BasicTextField` with **`OffsetMapping.Identity`**. This path must keep an
     exact 1:1 character mapping with the source — literal `\n`, **no**
     `ParagraphStyle` — or the cursor lands in the wrong place. Consequently,
     hanging indents are a View-only feature; do not "fix" the editor by adding
     `ParagraphStyle` to it.
- Make the read-only surface selectable with `SelectionContainer`. On the
  editor, apply `imePadding()` so the content stays reachable above the
  keyboard. Show the **Save** action only when `draft != saved`.

## B9. Persistence (DataStore) and session restore

- Use **DataStore Preferences** for all settings: theme mode, per-mode theme
  ids, per-surface font id, per-surface font size, custom-font slot names, the
  workspace list (+ an explicit order), the open-item list, and the current
  selection.
- Store the **workspace list** as a set of encoded `uri‹sep›name` strings plus a
  separate **order** string, so the home list keeps a stable, user-visible
  order.
- Persist session state (open items, current selection) as **identity only** —
  never file bodies — so restored content is always read fresh from disk.

## B10. The search index (Room + FTS4)

- Two Room entities: a **content/metadata table** (one row per item, keyed by an
  autogenerated `Long` `rowId`, with a **unique index on
  `(workspaceUri, docId)`** and the full body in a `content` column), and an
  **`@Fts4(contentEntity = …)` shadow** over that `content` column. Room
  generates the FTS virtual table **and the sync triggers**, so you only ever
  write the content table; searches **join on `rowId = notes_fts.rowid`**.
- **Use FTS4, not FTS5.** Room supports `@Fts3`/`@Fts4` but has **no `@Fts5`**
  (FTS5 isn't guaranteed on the SQLite of older supported Android versions).
  FTS4 gives the inverted-index matching, snippets, and ranking needed. Do not
  "upgrade" the annotation; there is nothing to upgrade to. Only if you truly
  need an FTS5-only feature do you drop to hand-written raw SQL.
- Keep `rowId: Long` exactly as-is — Room maps it to SQLite's true `rowid`,
  which the FTS content-table linkage requires; rename/retype it and the join
  breaks silently.
- Also keep a small **per-workspace meta table** (last-regenerated time, count)
  for the status surface.
- **The index is a disposable cache.** Build the database with
  `fallbackToDestructiveMigration()`; a schema change simply rebuilds it from a
  scan, so you generally never write a `Migration`. Reads
  (`listAll`, `search`) **return null when no usable index exists** so callers
  fall back to a live SAF scan.
- **Freshness model:**
  - *On launch,* run a cheap **reconcile** per workspace off the main thread:
    walk the tree for `(docId, lastModified)` only, re-read bodies **only** for
    new/changed files, prune rows whose files disappeared, update meta.
  - *On in-app save,* update just that item's row by document URI so it's
    searchable immediately.
  - *External edits* are picked up on the next launch's reconcile (the accepted
    staleness window).
  - *Manual regenerate* clears the workspace's rows and rebuilds.
  - Serialise reconcile per workspace with a `Mutex` so launch-time and manual
    runs can't collide.
- **Live status:** during a build, push per-file progress (current file + rising
  count) into a `StateFlow`; conflation keeps the UI responsive on large
  workspaces. Keep any per-file work cheap and on `Dispatchers.IO`.
- **Query building & display:** turn user input into a safe MATCH expression —
  split on whitespace, strip FTS operator characters, AND the tokens as prefix
  terms (`foo*`, so "foo" also matches "foobar"). Merge body matches (FTS) with
  title matches (a `LIKE` on the name, since FTS indexes only bodies),
  de-duplicated, body matches first. Take snippets from SQLite's `snippet()`,
  strip its delimiter characters, and re-emphasise query terms in the UI.

## B11. Fonts (device fonts + the four-slot custom pattern)

- **Device fonts:** discover once at startup, off the main thread, by scanning
  the known system font directories (`/system/fonts`, `/product/fonts`, …) for
  `.ttf`/`.otf`, loading each with `Typeface.createFromFile` wrapped in
  `FontFamily`, de-duplicated by a prettified display name. Persist a font
  choice as a stable id (the file path), with sentinels for "default" and
  "custom".
- **Custom fonts (the four-variant pattern from A5):** eight fixed slots overall
  — four variants (regular, italic, bold, bold-italic) × each font context —
  copied into `filesDir/custom_fonts/{context}_{variant}.ttf`, with a sidecar
  `.name` file per slot recording the original file name. **Copy** the file
  (rather than holding a SAF URI) so it can't vanish between sessions. Validate
  it's a real font (`Typeface.Builder(pfd.fileDescriptor).build()`) before
  accepting. Assemble present variants into one `FontFamily(List<Font>)` so
  bold/italic content uses the right face; fall back to the default until at
  least the regular slot is set.
- **Font sizes** are per-surface floats with defined default/min/max/step and a
  paired line-height ratio, exposed with −/+ steppers and a reset in Settings.

## B12. Settings screen implementation

- Model Settings pages as an enum with a **depth** function; drive the same
  `AnimatedContent` slide (B7) between them.
- One `goBack` lambda drives both the toolbar arrow and a `BackHandler` (B2).
- Build rows from a few shared composables (a navigation row with leading icon +
  title + subtitle + chevron; a single-choice row with a trailing check; a
  stepper row). Font choice rows **preview themselves** in the font they
  represent.

## B13. Testing and verification

- An automated suite is optional for an app of this size; whether or not you
  have one, the paths most worth exercising by hand are: granting and removing a
  workspace; the "all items" list and search on a **large** folder (this is
  where the SAF-cost and indexing decisions actually matter); editing and saving
  an item then confirming the edit is immediately searchable; regenerating the
  index from the status surface; deep folder drill-down using **both** the
  toolbar back arrow and the **Android system** back button (to confirm they
  behave identically); closing an item with unsaved changes; and switching
  themes and fonts.
- If you do add tests, the plain `model/` types and the
  query-building/highlighting helpers are pure Kotlin with no Android
  dependencies, so they are the cheapest and most valuable to cover first.

---

# Appendix — the traps this spec exists to prevent

A concise checklist of the non-obvious mistakes this spec is designed to
prevent. Each maps to a section above.

- **Diagonal navigation animation** — caused by `AnimatedContent`'s default
  `SizeTransform` animating height. Snap the size. (B7)
- **Clipped "Close" label on swipe** — dp/px unit mismatch between the reveal
  strip width and `translationX`. Convert dp→px via `LocalDensity`. (B6)
- **System back not mirroring the toolbar back** — one shared handler per
  surface; never an independent divergent `BackHandler`. (A2, B2)
- **Fabricated subfolder tree URIs returning nothing** — navigate by tree +
  document id only. (B3)
- **`ParagraphStyle` in the editor breaking the cursor** — hanging indents are
  read-only; keep the editor's identity offset mapping. (B8)
- **Reaching for FTS5** — Room has no `@Fts5`; use FTS4. (B10)
- **Renaming the FTS `rowId`** — it must stay the SQLite rowid. (B10)
- **Treating the index as truth** — it is a cache; always keep the live-scan
  fallback working. (B10)
- **SAF work on the main thread / aborting a whole scan on one bad file** — keep
  it on `Dispatchers.IO` and catch per file. (B3)
- **Doubling the top inset** — the `TopAppBar` already consumes the status-bar
  inset; apply only the bottom inset at the `Scaffold`. (B2)
