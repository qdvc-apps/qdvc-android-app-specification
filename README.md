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
tree, a tag list); **workspace** denotes a user-granted device folder that holds
items; and **slot** denotes one of the four positions in the bottom navigation
bar (so "slot 1" is a place in the bar, never a document).

An item need not be a single file. It is a **document unit**, and there are two
shapes in practice: one file (e.g. `my-note.md`), or one **folder** containing a
canonical `README.md` plus attachments (e.g. a `payloads/` subfolder of images).
Both are first-class. The folder shape has consequences that are called out
where they arise: the predicate for "what counts as an item" becomes *a folder
holding a `README.md`* rather than an extension filter (B3), the switcher, index
and session key off the **folder** identity (B6, B9, B10), and renaming an
item's title renames its folder — which allocates a **new** platform document
id, so anything holding one must be re-pointed (B3).

The two goals of this document are (1) to align existing and future apps on a
shared set of best practices, reducing backend technical debt and giving users
a consistent experience, and (2) to make new apps of this style fast to build.

---

# Part A — User-facing experience

This part is intentionally free of technology choices. It describes components,
layout, motion, and behaviour only.

## A1. Overall shape

The app is a single-screen application with a **persistent bottom navigation bar
of four slots**, which act as de-facto tabs. The upper region of the screen is a
**toolbar** (title, optional back button, optional context actions); the middle
is the active slot's content; the bottom is the navigation bar.

Two things are prescribed about the bar; the rest is the app author's:

1. **Slot 1 is the home / structure explorer.** The starting point and de-facto
   homepage. It presents the app's structures hierarchically and lets the user
   drill down into them to find items to open. This slot is fixed.
2. **A multitasking switcher appears somewhere in the bar** — a switcher between
   open items, equivalent to a browser's tab bar, from which items are also
   closed. **Slot 4 is the conventional home for it** and the recommended
   default, but it is not required to sit there.

The remaining slots are undefined by this spec. The most common arrangement is
`Home · View · Edit · Switcher`, where slots 2 and 3 act on the currently-selected
item; but an app may equally make them alternative top-level structure browsers
(a tag list, a favourites list), or move the switcher to slot 3 and give slot 4
to something else.

### Destination slots and action slots

A bar slot is one of two kinds, and the distinction is worth being explicit
about because they behave differently:

- A **destination slot** switches to a surface and stays there. It shows as
  selected while that surface is current. Slot 1 and the switcher are always
  destinations.
- An **action slot** performs something and hands the user on somewhere else —
  for example a "New" slot that immediately starts a fresh item and drops the
  user into the editing surface. It is **momentary**: it never shows a persistent
  selected state, because the user does not remain "in" it.

Rules for action slots:

- Neither slot 1 nor the switcher may be an action slot.
- At most one action slot, so the bar does not become a toolbar.
- Because the surface an action slot hands off to may not itself be in the bar
  (an Edit surface reached only via "New", say), **the bar can legitimately show
  nothing selected** for as long as the user is on that surface. That is
  acceptable and expected — but it must be *deliberate*, not an accident of a
  missing case, and the app should return to a selected destination as soon as
  the action completes (e.g. on save, go to the View slot).

Where a **destination** slot requires a selected item and none is open, its
bottom-bar item is shown **disabled** rather than hidden, so the bar's shape
never changes. A disabled slot is inert: it does not navigate, and (see A9) it
gives no tactile feedback either.

## A2. Slot 1 — the home / structure explorer

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

### Re-tapping slot 1

Tapping the slot-1 bottom-bar icon **while already on slot 1** jumps straight
back to the top of the hierarchy (the home root), regardless of how deep the
user has drilled. This is the quick "go home" gesture.

### Settings entry point

Because the home root is the app's homepage, the **entry to Settings lives
there**: a menu (overflow) button in the **top-right corner of the toolbar**,
shown only at the home root. Settings is reachable only from here — combined
with the re-tap gesture above, the user is always at most two taps from
Settings (tap slot 1 to go home, tap the menu).

### Motion

Movements through the hierarchy are animated as a **horizontal slide**:
drilling in slides the new level in from the right while the old level eases
left; stepping back reverses it. The animation must be a clean horizontal
slide — it must **not** drift diagonally from a corner (a known failure mode;
see B7).

This is the **depth** vocabulary, and it is reserved for movement *along* a
hierarchy. Switching between bottom-bar slots is a different kind of movement
and gets a different, non-directional treatment — see A8 and B7.

### Item actions at the home root

The home root offers a clear affordance to **add a workspace** (grant a new
device folder) and, per workspace row, to **remove** it from the list (removing
only the app's pointer to that folder — never deleting the user's files, with a
confirmation dialog that says so).

## A3. The app-specific surfaces

Not prescribed. The most common and recommended choice is a **View / Edit**
pair acting on the current item:

- **View** — a read-only rendering of the current item. Content is presented in
  a fixed-width, single-size layout where structure (headings, emphasis, lists)
  is conveyed through **colour and weight, never through changes in text size**.
  Text is selectable and copyable.
- **Edit** — the same item, editable. A toolbar **Save** action appears **only
  when there are unsaved changes**.

### Two shapes of editing surface

There are two legitimate ways to build the editable surface, and which one is
right is determined by **who owns the file's shape**:

1. **Source editing** — for a **free-form, user-owned** file, the user edits the
   raw text and the same syntax colouring is applied live to the source, so the
   character grid stays stable and the cursor lands where the user expects. This
   is the right default for a notebook, a scratchpad, or anything where the user
   may write whatever they like.
2. **Form editing** — for an **app-generated file with a fixed schema** (say a
   `README.md` that always carries a title, an abstract, a payload and an image
   list), the surface exposes those fields directly and the file is
   **regenerated** on save. This trades live source editing for a guarantee that
   the generated file is always well-formed, which is the better bargain when the
   app is the only thing that should be writing that structure.

Pick one deliberately; do not mix them on the same surface. Note that the
`ParagraphStyle`/cursor trap in B8 applies **only to source editing** — a form
surface never has an offset-mapping problem to get wrong.

Both View and Edit have **independent font and text-size settings** (see A5).
Both are disabled in the bottom bar until an item is open.

An app in this family is free to give these slots entirely different meanings, but
should keep the same conventions: a clear toolbar, context actions that appear
only when relevant, and consistent theming.

## A4. The multitasking switcher

The switcher is the phone equivalent of a desktop tab bar. It conventionally
occupies slot 4 of the bottom bar (see A1).

- It **lists every open item**, marking the current one distinctly (e.g. a dot
  and heavier weight), and shows each item's secondary context (e.g. which
  workspace it belongs to). An item with unsaved edits is marked (e.g. a
  trailing bullet).
- **Tapping** a row makes that item current and takes the user to the
  appropriate surface (e.g. the read-only View, where the app has a View/Edit
  pair).
- **Opening** a new item is done from slot 1 (drill to an item and tap it); it
  then appears here and becomes current.
- **Closing** uses a **swipe-to-close** gesture (like the swipe-to-archive
  gesture familiar from mobile email apps). The revealed action must show an
  unmistakable **Close affordance — an X icon above a "Close" label** — not a
  bare coloured rectangle. (The label must be fully visible; a past bug clipped
  it — see B6.)
- Closing an item **with unsaved changes** prompts a warning with **Cancel**
  (keeps the item) and **Close anyway** (discards edits, styled in the error
  colour).
- A toolbar action enters a **reorder mode** (see the shared pattern in A4a).
- Empty state: a friendly message telling the user to open something from
  slot 1.

The open-item list and the current selection **persist across app restarts**.
On relaunch the list is restored and each item's content re-read from its file;
items whose files have vanished are silently dropped.

## A4a. Reorder mode (a shared pattern)

Any list whose order the user controls — the switcher's open items, or an item's
own internal structure — uses the same reorder mode, so the gesture is learned
once.

- **Entering** is an explicit mode, reached from a toolbar action, not an
  always-on gesture. Entering a mode that takes over the screen is worth
  confirming physically (see A9).
- **Drag-to-reorder is the canonical gesture.** The user **presses and holds** a
  row, then drags it; rows part to show where it will land, and the list
  auto-scrolls when the finger approaches an edge. Press-and-hold rather than an
  immediate drag is deliberate, not fussiness: an immediate drag would swallow
  the list's own scroll gesture, leaving a list taller than the screen impossible
  to scroll while in the mode.
- **Up/down arrows per row remain an acceptable fallback** where drag is
  disproportionate — a small fixed list, or a first cut of a new app. They are
  cheaper and more robust, and an app is not wrong for shipping them; drag is
  simply the better experience where the list is long enough to matter.
- **While in the mode**, ordinary row interactions (tap to open, swipe to close)
  are suspended, and any always-visible destructive control is hidden. The bottom
  navigation bar **slides out of the way** and slides back on exit, so the mode
  reads as taking over the screen.
- **Leaving** is explicit: **Cancel** and **Save**, each of which confirms first.
  The Android system back button triggers the same discard confirmation as Cancel
  (per A2's back rule).
- **The draft order is never persisted.** Nothing is written until Save, and the
  draft is not part of saved session state — so quitting or force-killing
  mid-reorder discards it, which is the correct outcome and costs no code to
  achieve.
- Reorder lists deliberately **start at the top** rather than restoring a
  previous scroll position (see A8).

## A5. Settings

Settings is a **menu-based, hierarchical** screen: a root list of rows, each
with a **leading icon** and a title (and a subtitle showing the current value),
that navigate into sub-pages. Navigation between Settings pages uses the **same
horizontal slide** animation as slot 1, and obeys the **same back-button rule**
(the toolbar back arrow and system back share one handler; at the Settings root,
back closes Settings).

A typical Settings screen for an app in this family contains:

- **Appearance** — choose Automatic / Light / Dark. *(Baseline.)*
- **Light Mode Style** and **Dark Mode Style** — choose which colour theme is
  used in each mode, from the installed themes (see A6). *(Baseline.)*
- **Font size** — an independent size for each surface that renders text,
  adjusted with −/+ steppers between sensible bounds, with a one-tap **Reset**
  to default. *(Baseline.)*
- **Fonts** — an independent font *choice* for each such surface: the app
  **default**, any **device-installed font**, or a **custom font** (see below).
  Font rows preview themselves in the font they name. *(Opt-in.)*

### Typography tiers

The three typography features are tiered, and an app is complete without the
upper two:

1. **Per-surface font size** is the **baseline** — expected of every app in the
   family, and cheap.
2. **Device-font discovery** is **opt-in**. It costs a startup scan and a font
   registry.
3. **Custom font files** (the four-variant pattern below) are **opt-in**, and
   only worth it for apps whose whole point is long-form reading or writing.

State which tier you implement, so a missing font picker reads as a scoping
decision rather than an omission.

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
- **The user's folders are always the system of record.** The index holds nothing
  that does not already exist on disk, which is what makes it safe to delete and
  rebuild at any time. There are two valid ways to lean on it — a search-only
  cache, or a full read model the whole UI reads from — and the choice is a
  technical one (see B10). Whichever is used, it is never the truth.
- The index has a **user-facing status surface** showing whether it is *not
  built*, *building* (with live progress — the current file and a rising
  count), or *ready* (with the count and the time it last rebuilt), plus a
  **Regenerate now** button. The status surface makes clear that regenerating
  touches only the app's private index and never the user's files.
- Freshness: the index refreshes on launch; an in-app save updates that item's
  entry immediately; edits made in other apps are picked up on the next launch or
  on an explicit **Regenerate now**. If the app does not watch the folders while
  running, say so somewhere the user can find it, so a stale list reads as a known
  limit rather than a bug.

## A8. Cross-cutting UX conventions

- **The user's files stay put.** The app edits files in place in the folders the
  user grants; it never copies items into a private silo or locks them away.
  Destructive-sounding actions (remove workspace, close with unsaved changes)
  are confirmed and clearly worded.
- **Consistent list rows.** Lists use a consistent row: leading icon, primary
  text, optional secondary text, optional trailing action, and a divider. Where a
  row needs a status marker (e.g. *open*, or a sub-type such as *chat*), use one
  shared **outlined pill** component — transparent fill, border and text in the
  foreground colour — rather than inventing a second style per surface, and
  rather than encoding status in the text itself with parenthetical suffixes or
  italics.
- **Consistent empty states.** Every list/surface that can be empty shows a
  short, friendly message telling the user what to do next.
- **Restraint in motion — two vocabularies, and only two.** Movement *along* a
  hierarchy (drilling in and out of structures, and between Settings pages) uses
  the **horizontal slide** of A2. Movement *between* bottom-bar slots uses a
  single **non-directional** transition — the incoming surface arrives and the
  outgoing one recedes, with no left/right implication. Keeping these separate is
  not decoration: slots have no depth relation to one another, so sliding
  sideways between them would assert an ordering that does not exist. Every other
  transition reuses one of these two.
- **Lists keep their place.** Returning to a surface restores the scroll position
  the user left it at, per item where a surface can show different items. Three
  cases deliberately reset to the top instead, because starting at the top is
  right for each: **search results**, **reorder mode**, and **Settings**.
- **Two surfaces must never disagree about the same fact.** Where the same
  underlying fact is shown in two places — a "last updated" date on a list row
  and the same date inside the item, say — both must be derived from the same
  **resolved state**, not computed independently from raw history. Two surfaces
  that are each internally consistent but disagree with one another read as a bug
  to the user, and are much harder to diagnose than a single wrong number.
- **Never silently substitute.** If the thing the user was acting on has gone —
  deleted from under the app, or a permission revoked — show the **empty state**
  for that surface. Do not quietly fall back to some other item that happens to
  be open: a fallback that changes *which* item the user is acting on, without
  telling them, invites them to act on the wrong one. An honest empty surface is
  always the safer failure.
- **Failed writes are never silent.** If a write to the user's folder fails, any
  optimistic change already shown is rolled back and the user is told. A change
  that appears to have happened but did not is the worst outcome available.

## A9. Tactile feedback (haptics)

Short vibrations confirm that an action landed. Done well they are almost
invisible; done carelessly they feel like a broken phone. The rules below exist
to make them consistent across the family and across apps generally.

- **The system's own touch-feedback setting is honoured absolutely.** A user who
  has turned haptics off is never overridden, and the app needs no vibration
  permission to do this correctly (see B14). On a device with no vibrator,
  everything below simply does nothing.
- **Feedback marks a change, not a touch.** The test is whether *what the app is
  showing* changed, or whether something was committed. Tapping a bar slot,
  opening an item, toggling an item's state, entering a mode: yes. Opening a
  menu, scrolling, focusing a text field: no. Feedback on everything is
  indistinguishable from feedback on nothing.
- **A tap must feel like other apps' taps.** If the app's tap is fainter than the
  rest of the phone, users read it as broken rather than subtle — and the fix is a
  single constant, not a redesign (see B14 for the exact rungs).
- **A repeating sensation takes the faintest weight and must be gated on
  crossing a boundary** — a dragged row passing its neighbour, a swipe crossing
  the point where releasing would act. Gating on a boundary is what stops a
  continuous gesture buzzing continuously.
- **Two things stay silent, deliberately.** A **disabled** control gives no
  feedback, because buzzing would imply something happened when nothing did. A
  gesture that **grabbed nothing** — a long press on empty space below the last
  row — gives none either. A cancelled drag counts as a drop: the finger has
  left, and the row is going somewhere.
- **Confirming is not the same as tapping.** Where an action opens a confirmation
  dialog, the *button that opens the dialog* takes an ordinary tap and the
  *commit* takes Confirm (or Reject, where the commit throws work away), so the
  weight tells the user which side of the decision they are on.

### The six sensations

Having more than one sensation is what makes haptics informative rather than
merely present. Six are enough for every app in this family:

| Sensation | Weight | Used for |
| --- | --- | --- |
| **Tap** | normal click | navigating; pressing a button; toggling a state |
| **Step** | faintest tick | one notch of a continuous gesture |
| **Pick up** | heavy | a dragged row leaving its slot |
| **Drop** | medium | a dragged row being released |
| **Confirm** | heavy / platform confirm | a confirmation accepted; a mode entered |
| **Reject** | heavy / platform reject | work discarded; an action refused |

B14 gives the exact platform constant behind each, and the recommended mapping
from events to sensations.

## A10. Launch and the loading screen

Most apps in this family can show their home surface immediately and fill it in
as data arrives. An app whose state is **derived** from many files on disk (see
B10's projection pattern) cannot: until the read finishes there is genuinely
nothing truthful to show. Where that is the case:

- **Block visibly, and narrate.** Rather than an opaque spinner, show the read
  narrating itself — a running transcript of what is being read, with a rising
  count. The transcript is not decoration; it is the difference between "this app
  is slow" and "this app is reading my 400 files", and it is where a problem (an
  unreadable folder, a missing expected subfolder, a revoked permission) becomes
  visible instead of manifesting later as mysteriously absent data.
- **Block only once, at launch**, and for an explicit **Regenerate** from the
  index status surface. Nothing in ordinary use may put this screen back on
  screen.
- **Never block on a write.** Writes go to disk immediately and the UI reflects
  them at once (see A8's rule on failed writes); only the initial read earns a
  blocking screen.

---

# Part B — Technical implementation

This part maps Part A onto concrete Android + Kotlin technologies. It is
prescriptive about the patterns that carry the design's guarantees, and it flags
the traps that are easy to fall into — several of these are non-obvious and have
each cost real debugging time to diagnose, so they are documented here with
their root cause rather than just their fix.

**On naming:** Part A's generalized vocabulary (*item*, *structure*,
*workspace*) is what's prescribed. The concrete identifiers that appear in the
code and examples below — class names like `ItemRepository`, and Markdown-app
specifics such as `notes_fts`, `docId`, or the `.md` filter — are
**illustrative**: they show the pattern in a real, working form. Adopt the
*pattern*, and name things to fit your own app; where an identifier is fixed by
a platform or library (e.g. SQLite's `rowid`, `DocumentsContract` columns), that
is called out explicitly.

## B0. Platform, language, and build

- **Language / UI:** Kotlin with **Jetpack Compose** and **Material 3**. No XML
  layouts; the only XML is the manifest, launcher resources, and a minimal base
  theme.
- **Single Activity.** One `ComponentActivity` hosts everything; tabs and
  screens are Composables, not Activities or Fragments.
- **SDK levels:** `minSdk = 26` (Android 8.0), `compile`/`targetSdk = 34`,
  **JDK 17**.
- **Build:** Gradle with the Kotlin DSL. On Kotlin 2.0 and later — the default
  for anything built going forward — the Compose compiler ships with Kotlin, so
  you enable Compose by applying the **`org.jetbrains.kotlin.plugin.compose`**
  plugin with its version matched to your Kotlin version (declare it in the
  version catalog, apply it in each module that uses Compose), alongside
  `buildFeatures.compose = true`. You do **not** set
  `kotlinCompilerExtensionVersion` — that field is removed under this mechanism,
  and compiler options go in a `composeCompiler {}` block instead. (On AGP
  9.0+, Kotlin support is built in, so the separate `kotlin.android` plugin may
  not be needed either.)
- **Legacy build note (pre-2.0 Kotlin only):** before Kotlin 2.0 the Compose
  compiler was versioned independently and enabled via
  `composeOptions { kotlinCompilerExtensionVersion = "…" }`, pinned to a version
  compatible with your Kotlin release. A concrete known-good combination on that
  older toolchain is Kotlin `1.9.24` + compiler extension `1.5.14` + Compose BOM
  `2024.06.00`; keep it as an anchor only if you are deliberately staying on
  Kotlin 1.9.x, and prefer the 2.0+ plugin mechanism above otherwise.
  *Note for consistency with existing apps:* the apps currently implementing this
  spec are all still on that legacy combination, so a new app that wants to match
  its siblings byte-for-byte will find the pinned-extension form in them. The 2.0+
  mechanism above is still the right choice for anything new; a migration of the
  existing apps is outstanding.
- **Key dependencies:** Compose BOM (ui, graphics, material3,
  material-icons-extended, foundation); `activity-compose`;
  `lifecycle-viewmodel-compose`; **DataStore Preferences** (settings);
  **DocumentFile** (SAF helpers); **Room** with the **KSP** compiler (search
  index). No third-party markdown or persistence library is required.
- **Manifest:** the app enables edge-to-edge, and the hosting Activity sets
  `android:windowSoftInputMode="adjustNothing"`. That second choice is
  counter-intuitive and is explained in full in B2 — briefly, `enableEdgeToEdge()`
  already opts the window out of the legacy resize, so declaring `adjustResize`
  buys nothing while implying the window handles the keyboard, and the two
  mechanisms together double-count it. Under edge-to-edge, `WindowInsets.ime` is
  the single source of truth for the keyboard.

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

**Where this pattern stops scaling:** one ViewModel owning all state is the
right call at the four-surface scale this spec targets, but it is not unbounded.
As surfaces multiply — many independent feature areas, each with substantial
state of its own — the single ViewModel becomes a bottleneck: a growing
God-object that is hard to reason about and a magnet for merge conflicts when
several people touch it. The signal to split is when surfaces stop sharing
state rather than when the file merely gets long; at that point, give
independent feature areas their own scoped ViewModels (keeping the same
unidirectional-state and repository discipline within each) rather than
stretching the single one further.

Recommended package layout:

```
app/
  MainActivity.kt          host; edge-to-edge; tab switching; pickers; back handling
  AppViewModel.kt          all UI state; the slot-1 navigation state machine
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
- **Action slots (A1):** an action slot still needs a `Tab` entry, because the
  `when (tab)` dispatch must be exhaustive — but that branch is effectively
  unreachable, since selecting it runs its action (which immediately sets some
  *other* tab) rather than rendering anything. Give it a branch anyway rather
  than an `else`, so adding a real destination later cannot silently fall through
  it. Intercept the slot in the bar's `onSelect` (`if (selected == NEW)
  startNewItem() else selectTab(selected)`); do not route it through the ordinary
  tab-selection path.
- **Slot-1 hierarchy as a state machine.** Model the home surface's levels as an
  enum (e.g. `WORKSPACES, OVERVIEW, FOLDERS, ALL_ITEMS, SEARCH, INDEX_STATUS`)
  held in a `BrowseState` in the ViewModel, with a folder **stack** for the
  arbitrary-depth level. A single `browseUp()` function encodes every back-step;
  it returns whether it consumed the action.
- **The back-button rule, enforced structurally.** For each surface that shows a
  toolbar back arrow, wire the arrow's `onClick` and an
  `androidx.activity.compose.BackHandler` to **the same lambda**. Never give a
  screen an independent `BackHandler` that does something different from its
  arrow. On slot 1, enable a `BackHandler` exactly when
  `currentTab == slot1Tab && mode != homeRoot`, calling `browseUp()`. In Settings,
  the toolbar arrow and a `BackHandler` share one `goBack` lambda that steps up
  a page or calls `onClose()` at the root.
- **Re-tap to go home:** in the bottom bar's `onSelect`, detect
  "tapped slot 1 while already on slot 1" and reset the hierarchy state to its
  root.
### Insets: exactly one owner per edge

This app shape has **nested `Scaffold`s** — an outer one in `MainActivity` owning
the bottom `NavigationBar`, and an inner one per screen owning that screen's
`TopAppBar`. Compose will happily let both apply the same system-bar inset, and
the result is a phantom strip of blank space that obscures content. Every app
built to this spec so far has hit it. Read this before adding a `Scaffold`.

The contract is that **each inset edge has exactly one owner**:

| Edge | Owner |
| --- | --- |
| Top (status bar) | each screen's `TopAppBar`, via its own default `windowInsets` |
| Bottom (navigation bar) | the outer `Scaffold`'s bottom bar (`NavigationBar` consumes it internally) |
| IME (keyboard) | one `imePadding()` on the single bottom-most element that must clear it |

Which yields these rules:

- The **outer** `Scaffold` contributes nothing:
  `contentWindowInsets = WindowInsets(0, 0, 0, 0)`. Its `padding` then reserves
  the bottom bar's height and nothing else.
- Do **not** re-apply the bottom system-bar inset to the content container. The
  `NavigationBar` already consumes it; adding
  `windowInsetsPadding(WindowInsets.systemBars.only(Bottom))` on the content is a
  second, additive cause of the same gap.
- Every **inner** `Scaffold` must contribute **no bottom inset**. Define one
  shared value and use it everywhere, so the exceptions are visible:

  ```kotlin
  /** Inner screens sit inside the app's bottom bar, which owns the bottom edge. */
  val topOnlyInsets: WindowInsets
      @Composable get() = WindowInsets.systemBars.only(WindowInsetsSides.Top)
  ```

- **A full-screen route is the exception** and must use the default insets, not
  `topOnlyInsets`. Settings, rendered as an overlay *above* the outer `Scaffold`
  rather than inside it, has no bottom bar beneath it and therefore does own its
  own bottom edge. Using a named helper for the nested case is what makes this
  exception legible: a `Scaffold` that does not mention `topOnlyInsets` is
  announcing that it is not nested.
- **Zeroing an inner `Scaffold`'s content insets does not break the top.**
  `contentWindowInsets` affects only the *content*'s padding; a `TopAppBar`
  applies the status-bar inset from its **own** `windowInsets` parameter
  regardless. This looks wrong when you read it, which is why it is written down —
  `WindowInsets(0, 0, 0, 0)` and `topOnlyInsets` are equivalent in a screen that
  has a top bar. Prefer `topOnlyInsets` for the reason above, and note that a
  nested screen *without* a top bar genuinely does need the top inset from
  somewhere.

### The keyboard (IME)

The keyboard is a separate trap from the one above, it is harder, and getting it
wrong produces a gap that looks identical — so it is easy to misdiagnose as more
of the same. Two independent mechanisms cause it:

1. **Double-counting between the window and Compose.** `enableEdgeToEdge()` sets
   `decorFitsSystemWindows = false`, which opts the window out of the legacy
   resize; `WindowInsets.ime` then reports the keyboard for you to apply
   manually. Declaring `windowSoftInputMode="adjustResize"` as well asks the
   platform to *also* resize, and where it does, an `imePadding()` on top adds the
   keyboard's height a second time. **Declare `adjustNothing`** (B0) so there is
   one mechanism, and apply the IME inset exactly once.
2. **Over-lifting past the app's own bottom bar.** This is the non-obvious one. A
   composer anchored to the bottom of a nested screen already sits above the
   entire app `NavigationBar` — its **content height plus** the system nav-bar
   inset. `imePadding()` lifts by the keyboard's full height, measured from the
   *screen* bottom, so the composer ends up too high by the whole bar. The
   intuitive repair,
   `windowInsetsPadding(WindowInsets.ime.exclude(WindowInsets.navigationBars))`,
   subtracts only the **inset** and leaves the bar's own content height —
   typically the larger part, and a visible gap of a centimetre or more. There is
   no inset that represents "the height of my own navigation bar", so subtraction
   cannot be made to work.

The resolution is to remove what is underneath instead of subtracting it:

```kotlin
// MainActivity's outer Scaffold
bottomBar = {
    // Hide the app's own bar while the keyboard is up, so a bottom-anchored
    // input can sit directly on the keyboard with nothing between them.
    if (!WindowInsets.isImeVisible) {   // @ExperimentalLayoutApi
        AppBottomBar(...)
    }
},
```

With nothing below it, the composer's plain `imePadding()` seats it exactly on
the keyboard.

- **Apply this unconditionally**, even in an app with no bottom-anchored input.
  It is the same amount of code either way, hiding the bar while typing is
  desirable behaviour in its own right (it returns a bar's worth of height to the
  content), and it means the first person to add a bottom-anchored input inherits
  a working arrangement instead of rediscovering the above from scratch.
- **One consequence to check when adopting `adjustNothing`:** the window no
  longer resizes, so any input the keyboard could cover must be reachable by other
  means. In practice that means a scrolling form carries a single `imePadding()`
  on its scrolling column, so the focused field can be scrolled clear; inputs near
  the top of the screen (a search field in a toolbar) need nothing. Audit the
  app's text fields once when making this change.
- **Never stack IME handling.** One `imePadding()` per screen, on the
  bottom-most element that must clear the keyboard. An `imePadding()` on both a
  message list and its composer shrinks the list by the keyboard height twice.

### Preserving scroll position across slot switches

Switching bottom-bar slots through an `AnimatedContent` **discards the outgoing
slot's composition**. Any `LazyListState` that was implicitly remembered inside
that screen is therefore rebuilt at zero on every return, so lists silently lose
their place. `rememberSaveable` does **not** fix this — it survives configuration
change and process death, not *leaving composition*.

Hoist the state **above** the `AnimatedContent` and pass it in. Where a surface
can show different items, key it, and bound its memory:

```kotlin
@Composable
private fun rememberKeyedListState(key: String?, retain: Set<String>): LazyListState {
    val states = remember { mutableMapOf<String, LazyListState>() }
    val fallback = rememberLazyListState()
    // Bounded: forget positions for keys that are no longer open/present.
    LaunchedEffect(retain) { states.keys.retainAll(retain) }
    return key?.let { states.getOrPut(it) { LazyListState() } } ?: fallback
}
```

The same loss happens *within* a surface whose levels are swapped by their own
`AnimatedContent` (B7), so hoisted state is needed per level too, not only per
slot. Per A8, search results, reorder mode and Settings are deliberately left to
reset to the top — pass them their own unkeyed state rather than exempting them
by accident.

## B3. The Storage Access Framework (read before touching file access)

This part of the app carries constraints that are not obvious from the code, and
a "cleaner"-looking rewrite that ignores them will break in ways that fail
silently rather than loudly. Read this section before changing anything here.

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

### Appending to a file another client may also write

Some apps keep an append-only file — an event log, an audit trail — that a
desktop sibling may also write to. Appending with `openOutputStream(uri, "wa")`
is far cheaper than rewriting (one row's worth of writing rather than the whole
file), and it carries a **data-loss trap** that destroys existing data rather
than merely failing:

- `"wa"` writes onto the file's **final byte**. If the last line has no
  terminator — this app always writes one, but another client need not — the new
  content is spliced onto it, **fusing two valid records into one malformed record
  and destroying both**.
- So **classify the final byte first** and supply the missing separator before
  appending. Accept **CR as well as LF** as a terminator, so CRLF and bare-CR
  files are handled.
- **Find the tail by streaming to the end of the file, not by seeking to
  `size - 1`** using a size from a directory listing. That size is stale
  precisely when it matters — when another client has just written.
- If the tail cannot be determined, **fall back to the rewrite path rather than
  appending blind.**
- And guard the fallback: a read helper that returns an empty list for **both** an
  empty file and a **failed read** will make the rewrite path truncate a file it
  could not read. Distinguish the two and abort rather than write. This is a
  latent bug that only fires on the day a read fails, which is the day you can
  least afford it.

### Adapting to a different item shape

The file-type-specific parts of the SAF layer are small:

- **The item predicate.** For a file-shaped item, an extension filter (e.g.
  `.md`/`.markdown`). For a **folder-shaped** item (see the vocabulary in the
  intro), *a folder containing a `README.md`* — deliberately tolerant, so a folder
  created by a sibling app is recognised.
- **The MIME type** passed when creating a new item, or
  `DocumentsContract.Document.MIME_TYPE_DIR` plus a child `README.md` for the
  folder shape.
- **Renaming, for folder-shaped items.** Renaming an item's title renames its
  folder, and a rename **allocates a new document id**. Anything holding the old
  id — the open-item list, index rows, the current selection — must be re-pointed,
  not dropped. Make identity matching fall back from document id to a **stable
  app-level id** (a frontmatter `id`, a filename prefix) so an item renamed
  *outside* the app is re-pointed rather than treated as deleted and re-created.

Everything else — tree traversal, reads, writes, existence checks — is identical
across apps in this family.

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
- **Swipe haptics:** if the swipe does not *latch* open — if releasing past a
  threshold closes immediately — then the moment worth signalling is **crossing
  that threshold, in either direction**, so the user can feel the point of no
  return arrive and depart. Signal the close itself with Confirm (A9, B14).
- **Handle `onDragCancel`.** An interrupted gesture (an incoming call, a system
  dialog) that is not handled leaves the row stuck part-swiped. It is one line and
  it is always forgotten.
- **Session persistence:** persist only item **identity** (uri, display name,
  context label) and the current selection to DataStore whenever the list or
  selection changes. On launch, restore the list, re-read each file's content,
  and drop any whose file no longer exists — but see A8: if the *current* item is
  among them, clear the selection and show the empty state rather than falling
  back to another open item.

### Reorder mode (A4a)

Compose ships no reorderable list, so drag-to-reorder is hand-written: pointer
tracking, live gap animation, index remapping, and edge autoscroll. Budget for it
needing a round of on-device tuning, and keep as much of it as possible testable.

- **Begin the drag on a long press** — `detectDragGesturesAfterLongPress`, not
  `detectDragGestures`. A plain drag consumes the `LazyColumn`'s scroll gesture,
  so a list taller than the screen becomes unscrollable while in the mode. The
  long press is also what makes the Pick-up haptic apt (B14).
- **The reordering list must contain *only* draggable rows.** The index
  arithmetic maps a finger position to `LazyListState.layoutInfo.visibleItemsInfo`
  entries and assumes **list index equals row index**; a header, spacer or info
  panel inside the same lazy list breaks that mapping **silently** — rows land one
  slot out. Pin any header *above* the list rather than inside it, and say so in a
  comment on the component, because the constraint is invisible at the call site.
- **Hoist the mode into the ViewModel**, not into local screen state. Both the
  bottom bar (which slides away) and the `BackHandler` (which must raise the
  discard confirmation) need to see it.
- **Keep the draft order out of persisted state.** Hold it as Compose state keyed
  on the mode, so entering re-seeds it from the source of truth and leaving drops
  it. A force-quit then discards the draft for free — no code, no cleanup path.
- **Extract the pure arithmetic.** A `movedItem(list, from, to)` function is
  ordinary Kotlin and can be unit-tested exhaustively; the gesture and layout
  layer cannot. Getting the arithmetic out of the Composable leaves the untestable
  surface as small as possible, which matters because this is the component most
  likely to need blind iteration.
- **Feed the haptics hooks** (`onPickUp`, `onStep` per boundary crossed, `onDrop`,
  and Confirm/Reject on Save/Discard) — see B14.
- Where drag is disproportionate, per-row **up/down** `IconButton`s guarded by
  `canMoveUp`/`canMoveDown`, each calling a ViewModel `move(from, to)`, remain a
  supported fallback (A4a).

## B7. Navigation animation (avoid the diagonal drift)

### Depth transitions (hierarchy and Settings)

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
  so only the clean horizontal slide animates. It is easy to miss because it
  only shows up when adjacent screens differ in height — a transition can look
  perfect in testing and drift in production once the content changes.
- Reuse this identical spec for **both** slot-1 navigation and Settings
  navigation so all hierarchy motion matches.

### Lateral transitions (between bottom-bar slots)

Slots have no depth relation, so a directional slide would assert an ordering
that does not exist (A8). Use a **non-directional** transition instead — the
incoming surface scales up and fades in while the outgoing one recedes slightly
and fades aside:

```kotlin
AnimatedContent(
    targetState = tab,
    transitionSpec = {
        val enter = scaleIn(initialScale = 0.92f, animationSpec = tween(280)) +
            fadeIn(animationSpec = tween(280))
        val exit = slideOutHorizontally(animationSpec = tween(220)) { -it / 6 } +
            fadeOut(animationSpec = tween(220))
        (enter togetherWith exit)
            .using(SizeTransform(clip = false) { _, _ -> snap() })
    },
    label = "slot-switch",
) { shownTab -> /* … */ }
```

- **The `SizeTransform { snap() }` guard is required here too.** It is the same
  failure mode as above, and slots differ in content height far more than adjacent
  hierarchy levels do, so omitting it here is *more* visible, not less.
- **This `AnimatedContent` is what discards composition** on a slot switch, which
  is why list state must be hoisted above it — see B2's scroll-position
  subsection. The two are the same structure viewed from different angles.

## B8. Content rendering / syntax highlighting

The concrete example here is Markdown highlighting, but the pattern generalises
to any structured plaintext (task syntax, config keys, wiki markup).

**Scope:** the read-only path below applies to any app that renders item content.
The editable path applies only to a **source-editing** surface (A3). An app whose
editable surface is a **structured form** that regenerates the file has no
`VisualTransformation`, no offset mapping, and therefore none of the second path's
traps — it has the different obligation of **round-tripping its own format
losslessly**, so that a value the form cannot represent is preserved rather than
silently dropped on the next save. Cover that round trip with unit tests over the
real writer and the real parser (B13).

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
  never file bodies — so restored content is always read fresh from disk. For a
  folder-shaped item that identity is the **folder**, and it should include a
  stable app-level id alongside the platform document id, so an item renamed
  outside the app is re-pointed rather than dropped (B3).
- **Do not persist transient modes.** A reorder draft, a search query, or "am I in
  reorder mode" must stay out of DataStore — that is what makes a force-quit
  discard them correctly with no cleanup code (A4a).

## B10. The on-device database (Room + FTS4)

### Two patterns, both valid

The reason for an on-device database is B3: SAF calls are IPC and their cost
scales with the number of files. There are **two established ways** to spend that
database, and they are equally valid — pick by the shape of your data, not by
preference.

**Pattern 1 — index as a search cache.** Room backs *search* and the *all items*
list. Everything else reads through SAF on demand: opening an item reads its file,
saving writes it. Reads from the index **return null when no usable index exists**
so callers fall back to a **live SAF scan** — so search and the flat list keep
working, just slower, if the index is missing or broken.

**Pattern 2 — projection-first.** Room is the app's **read model for everything**.
At launch the app reads the workspaces in full — every item, and every auxiliary
file its state is derived from — into the database, and from then on **the UI reads
only from the projection**. Opening an item, switching between open items, changing
an item's state, showing its history: all SQL, no SAF. Writes go the other way and
go **immediately**.

**Which to choose:**

| Choose Pattern 1 when… | Choose Pattern 2 when… |
| --- | --- |
| An item is self-contained: reading it is one file read | An item's state is **derived** from many files (e.g. replaying an append-only log) |
| The app is the primary or only writer | A desktop sibling edits the same tree, so correctness of writes dominates |
| Launch should be instant and search can warm up behind it | A blocking read at launch is acceptable in exchange for a UI that never touches SAF |
| Item bodies are large relative to the number of items | There are many small files and per-file IPC cost dominates |

Pattern 2's benefit is that no ordinary interaction can be slow, because none of
them touch the filesystem. Its cost is the blocking launch read, which is why it
comes with a user-facing loading screen (A10). Pattern 1's benefit is a cheap
launch and a degradation path that keeps working with no index at all.

**The invariant both share:** the projection or index holds nothing that does not
already exist on disk, so it is always **disposable** — it can be deleted or
rebuilt without data loss. That is what makes
`fallbackToDestructiveMigration()` (below) correct rather than reckless.

### Pattern 2: rules that make projection-first safe

If the UI reads only from the projection, the write path is where all the risk
concentrates. These are not optional:

- **Writes are never queued, batched, or handed to a background scheduler.** Every
  create, edit, reorder and state change is written to the user's folder as soon as
  the user performs it. The files are shared with other clients; a deferred write
  is a write that can be lost, and no amount of local durability substitutes for
  the bytes being on disk.
- **State changes are optimistic, and roll back.** Patch the projection first so
  the UI responds on the next frame, *then* write. If the write fails, **undo the
  projection patch and tell the user** (A8). A change that shows in the UI but
  never reached disk is the failure this rule exists to prevent.
- **Structural changes go to disk first.** Creating or renaming is where naming
  rules and uniqueness are actually validated — by the filesystem and by the
  format's own constraints — so write first, then refresh the projection **from
  the result** rather than from what you intended to write.
- **Never derive state from a snapshot taken at launch.** Distinguish facts with
  different lifetimes: *which workspaces were successfully read this session* is a
  session fact and may be held as a set; *whether an item exists* changes the
  moment the user creates one, and must be **queried live from the projection**
  every time. Conflating the two makes an item created since launch look deleted —
  and pruning logic will then dutifully drop it.
- **A workspace that failed to read is not an empty workspace.** Treating a
  revoked permission or an I/O error as "everything was deleted" will empty the
  user's open-item list. Leave failed workspaces untouched by any pruning pass.

### Shared mechanics (both patterns)

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
- **The database is disposable.** Build it with
  `fallbackToDestructiveMigration()`; a schema change simply rebuilds it from a
  scan, so you generally never write a `Migration`. Bump the version and let it
  rebuild.
- **Freshness — Pattern 1 (cache):**
  - Reads (`listAll`, `search`) **return null when no usable index exists** so
    callers fall back to a live SAF scan.
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
- **Freshness — Pattern 2 (projection):**
  - *On launch,* ingest each workspace **in full**, behind the loading screen
    (A10). There is no partial-read fast path: the UI has no SAF fallback to
    degrade to, so it must not render before ingest completes.
  - *On every write,* update the projection as part of the same operation (see the
    optimistic/rollback and disk-first rules above). Nothing waits for the next
    launch.
  - *External edits* are picked up on the next launch or an explicit **Regenerate
    now**, which re-runs ingest behind the same loading screen. Say so in the UI
    (A7).
  - Serialise ingest per workspace with a `Mutex`, for the same reason as above.
- **Live status:** during a build, push per-file progress (current file + rising
  count) into a `StateFlow`; conflation keeps the UI responsive on large
  workspaces. Keep any per-file work cheap and on `Dispatchers.IO`.
- **Query building & display:** turn user input into a safe MATCH expression —
  split on whitespace, strip FTS operator characters, AND the tokens as prefix
  terms (`foo*`, so "foo" also matches "foobar"). Merge body matches (FTS) with
  title matches (a `LIKE` on the name, since FTS indexes only bodies),
  de-duplicated, body matches first. Take snippets from SQLite's `snippet()`,
  strip its delimiter characters, and re-emphasise query terms in the UI.

### Room and KSP traps

Three failures that are cheap to avoid and expensive to diagnose, because two of
them surface in *generated* code and the third does not surface at all.

- **Never name a DAO parameter after a Java reserved word.** Room's KSP processor
  generates **Java**, so a parameter called `new` produces
  `_stmt.bindString(_argIndex, new);` and the build fails inside
  `SomeDao_Impl.java` with a syntax error that points at a file you did not write.
  The Kotlin looks perfectly legal. Use `oldName`/`newName` and note why in a
  comment, and audit every DAO parameter, entity property and projection field
  against the Java keyword list — not just the ones you happen to be adding.
- **Never call a synchronous Room query from the main thread.** Without
  `allowMainThreadQueries()` (which you should not add) Room throws, so a single
  synchronous call reached from a UI callback crashes the app on that action every
  time. Run the query on `Dispatchers.IO` and post the result into a flow. Watch
  for this in "refresh some status" helpers, which look cheap and are usually where
  it creeps in.
- **An identifier field must name exactly what it identifies.** A write result
  carrying a field like `docId` meaning "the folder this write touched" is a
  latent defect: for some operations that folder is the *item*, for others its
  *parent*. A caller reading it as one when it is the other will delete the right
  row and re-insert the wrong thing — and the flaw **passes review**, because for
  the majority of operations the two meanings coincide. Name the field for the
  thing (`itemDocId`, `containerDocId`), or return both. This class of bug corrupts
  only the projection, so it "fixes itself" on restart, which makes it look
  intermittent rather than systematic.

Two habits that pay for themselves here: audit every `@Query`'s `:bindings`
against the method's parameters and every projection field against the selected
columns whenever the schema changes, and keep pure logic (state folds, query
building, ordering arithmetic) **outside** Room and Compose so it can be unit
tested without either (B13).

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
  behave identically); closing an item with unsaved changes; reordering a list and
  quitting mid-reorder (the draft must be gone); opening a bottom-anchored input to
  check the keyboard seats correctly (B2); returning to a list to check it kept its
  place (B2); and switching themes and fonts.
- If you do add tests, the plain `model/` types and the
  query-building/highlighting helpers are pure Kotlin with no Android
  dependencies, so they are the cheapest and most valuable to cover first.

### Design for the untestable parts

Compose layout, gesture handling and haptics cannot be verified without a device,
and a Room schema cannot be verified without running codegen. That is a reason to
**shrink** those surfaces, not to shrug at them:

- **Push pure logic out of Composables and out of Room.** Ordering arithmetic,
  progress summaries, state folds over a log, date formatting, query building — all
  ordinary Kotlin, all exhaustively testable, and all of it commonly written inside
  a Composable or a DAO where it cannot be. Move it to `model/`/`util/` or a
  companion object and test it directly against the **production** function, not a
  reimplementation of it.
- **Inject the clock** into anything that formats or compares times. It is the only
  way to test "today", midnight boundaries, daylight-saving shifts, and future
  timestamps from a client with a wrong clock — each of which is a real defect
  class, and none of which is reachable otherwise.
- **Verify byte-level file behaviour against the app's own parser.** For an append
  path (B3), the useful test asserts that a round trip through the real writer and
  the real parser survives a terminated file, an unterminated one, an empty one,
  CRLF, and a bare CR.
- **Declare what remains unverified.** When a change touches layout, gestures,
  haptics or Room codegen and the environment cannot compile or run them, say so in
  the commit message and name the single value or line to adjust if the result feels
  wrong on device. A change presented as verified when it is not is worse than one
  honestly labelled unverified.

### A shipped manual test plan

Because so much of this app shape is only checkable by hand, the manual plan is
worth treating as a **deliverable in the repository** rather than tribal knowledge:

- Keep it in the repo (e.g. `manual-tests/`) in a real, machine-readable format
  rather than as prose, so it can be validated and imported.
- **Organise suites by behaviour, not by repository layout** — loading a workspace,
  moving around, changing state, authoring, what lands on disk, search, appearance
  and feedback.
- **Add a regression case for every defect found in development, and have it state
  what the behaviour used to be.** A regression case that only asserts the correct
  behaviour stops making sense once nobody remembers the bug.
- Mark unreviewed cases honestly as drafts. An unreviewed case marked "approved" is
  worse than an honest draft.
- Ship a **dependency-free validator** for the plan's own format that exits
  non-zero, so a hook or CI job catches the mistakes hand-editing introduces —
  values outside a vocabulary, steps with no expected result, numbering drift. And
  verify the validator itself two ways: that it passes known-good input, and that it
  catches faults deliberately injected into a copy.

## B14. Haptic feedback

Implements A9.

- **Route everything through `View.performHapticFeedback`, never
  `Vibrator`/`VibrationEffect`.** Three reasons, each independently sufficient: it
  honours the system's own touch-feedback setting, so a user who turned haptics off
  is not overridden; it requires **no `VIBRATE` permission**, so the app can keep an
  empty `uses-permission` list; and it no-ops quietly on a device with no vibrator.
- **Do not pass `FLAG_IGNORE_GLOBAL_SETTING`.** If haptics are off, they stay off.
- **Put every call behind one small class**, obtained from a composable helper, so
  the vocabulary is enumerable and a weight can be re-tuned in one place:

  ```kotlin
  class Haptics(private val view: View) {
      fun tap()     = perform(HapticFeedbackConstants.VIRTUAL_KEY)
      fun step()    = perform(HapticFeedbackConstants.CLOCK_TICK)
      fun pickUp()  = perform(r(HapticFeedbackConstants.GESTURE_START, HapticFeedbackConstants.LONG_PRESS))
      fun drop()    = perform(r(HapticFeedbackConstants.GESTURE_END, HapticFeedbackConstants.CONTEXT_CLICK))
      fun confirm() = perform(r(HapticFeedbackConstants.CONFIRM, HapticFeedbackConstants.LONG_PRESS))
      fun reject()  = perform(r(HapticFeedbackConstants.REJECT, HapticFeedbackConstants.LONG_PRESS))

      /** API 30+ constant, else a minSdk-26-safe equivalent. */
      private fun r(api30: Int, fallback: Int) =
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) api30 else fallback

      private fun perform(constant: Int) = view.performHapticFeedback(constant)
  }

  @Composable
  fun rememberHaptics(): Haptics {
      val view = LocalView.current
      return remember(view) { Haptics(view) }
  }
  ```

### The three intensity rungs

Record these in a comment beside the definitions above, so nobody has to
re-derive them in order to move a sensation up or down:

| Constants | Sensation |
| --- | --- |
| `CONTEXT_CLICK`, `CLOCK_TICK` | faint tick |
| `VIRTUAL_KEY`, `KEYBOARD_TAP` | normal click — the standard button press |
| `LONG_PRESS` | heavy click |

- **Trap — `CONTEXT_CLICK` is the platform's *lightest* predefined effect**
  (roughly `EFFECT_TICK`). It is the natural-looking choice for "a light tick on
  tap" and it is wrong: on many devices it is barely perceptible, so the app reads
  as having broken haptics while every other app on the phone feels correct. Taps
  take **`VIRTUAL_KEY`**, which is what other apps use for a button press. If a
  sensation still needs more weight on real hardware, moving it one rung is a
  one-word change — which is the whole point of the table above.
- **API-level guards.** With `minSdk = 26`, `VIRTUAL_KEY`, `KEYBOARD_TAP`,
  `CLOCK_TICK`, `CONTEXT_CLICK` and `LONG_PRESS` are all available unconditionally.
  `GESTURE_START`, `GESTURE_END`, `CONFIRM` and `REJECT` are **API 30+** and each
  needs a version guard with a named fallback. Check every constant against
  `minSdk` when adding one; the compiler will not.
- **`pickUp()`'s fallback is apt, not merely tolerable.** The drag begins on a long
  press (B6), so `LONG_PRESS` is the correct sensation on older devices rather than
  a degradation.
- **Put the call at the interaction site, not in the ViewModel.** Haptics are a
  property of the gesture, and the same ViewModel callback may be reached from a
  tap, a restore, or another app's write — only the first should buzz.

### Recommended event mapping

A default, so a new app does not relitigate it:

| Event | Sensation |
| --- | --- |
| Tapping an enabled bar slot | `tap` |
| Opening an item from the home surface, a search result, or the switcher | `tap` |
| Tapping a row that navigates to another surface | `tap` |
| Toggling an item's state | `tap` |
| A button that *opens* a confirmation dialog | `tap` |
| Committing a confirmation | `confirm` |
| Discarding work (a draft, a reorder) | `reject` |
| Entering a screen-taking-over mode (e.g. reorder) | `confirm` |
| A dragged row leaving its slot | `pickUp` |
| A dragged row released, or the drag cancelled | `drop` |
| A dragged row passing a neighbour | `step` |
| A swipe crossing (or re-crossing) its commit threshold | `step` |
| A swipe-to-close actually closing a row | `confirm` |
| Tapping a **disabled** control | *(silent)* |
| A long press that grabbed no row | *(silent)* |
| Opening a menu; scrolling; focusing a field | *(silent)* |

---

# Appendix — the traps this spec exists to prevent

A concise checklist of the non-obvious mistakes this spec is designed to
prevent. Each maps to a section above.

### Layout and motion

- **Diagonal navigation animation** — caused by `AnimatedContent`'s default
  `SizeTransform` animating height. Snap the size. Applies to lateral slot
  transitions as much as to depth ones. (B7)
- **Phantom gap above the bottom bar** — nested `Scaffold`s both applying the
  bottom system-bar inset. One owner per edge: inner `Scaffold`s contribute no
  bottom inset, and the content container must not re-apply it either. (B2)
- **Doubling the top inset** — the `TopAppBar` consumes the status-bar inset from
  its own `windowInsets`, so zeroing an inner `Scaffold`'s content insets is safe;
  applying the top inset a second time is not. (B2)
- **A gap between the keyboard and a bottom-anchored input** — two causes.
  `adjustResize` plus edge-to-edge double-counts the IME (declare `adjustNothing`);
  and `imePadding()` over-lifts past the app's own `NavigationBar` by the bar's
  whole height, which `ime.exclude(navigationBars)` does **not** fix. Hide the bar
  while the IME is visible. (B0, B2)
- **Lists losing their scroll position** — the slot-switching `AnimatedContent`
  discards composition, and `rememberSaveable` does not help. Hoist list state above
  it, keyed and bounded. (B2)
- **Clipped "Close" label on swipe** — dp/px unit mismatch between the reveal
  strip width and `translationX`. Convert dp→px via `LocalDensity`. (B6)
- **A header inside a reorderable lazy list** — the index arithmetic assumes list
  index equals row index, and breaks **silently**. Pin headers outside the list.
  (B6)
- **Plain drag instead of long-press drag** — swallows the list's scroll gesture,
  making a long list unscrollable in reorder mode. (B6)
- **System back not mirroring the toolbar back** — one shared handler per
  surface; never an independent divergent `BackHandler`. (A2, B2)

### Storage and data

- **Fabricated subfolder tree URIs returning nothing** — navigate by tree +
  document id only. (B3)
- **SAF work on the main thread / aborting a whole scan on one bad file** — keep
  it on `Dispatchers.IO` and catch per file. (B3)
- **Appending onto an unterminated final line** — `"wa"` writes onto the last
  byte, fusing two records into one malformed one and destroying both. Classify the
  final byte (CR *or* LF) and supply the separator first; find the tail by
  streaming, not from a stale listed size. (B3)
- **A read helper that returns empty for both "empty" and "failed"** — makes a
  rewrite fallback truncate a file it could not read. (B3)
- **Assuming a rename keeps its document id** — it does not; re-point everything
  holding one, and match by a stable app-level id as a fallback. (B3)
- **Reaching for FTS5** — Room has no `@Fts5`; use FTS4. (B10)
- **Renaming the FTS `rowId`** — it must stay the SQLite rowid. (B10)
- **Treating the index or projection as truth** — it holds nothing that is not
  already on disk. Pattern 1 keeps a live-scan fallback; Pattern 2 keeps the write
  path immediate and rollback-safe. (B10)
- **A DAO parameter named `new`** — Room's KSP processor generates Java, so it
  fails in generated code. Audit against the Java keyword list. (B10)
- **A synchronous Room query on the main thread** — throws; crashes the action
  every time. (B10)
- **One id field meaning two different things** — passes review because the two
  meanings coincide in the common cases, then deletes the right row and inserts the
  wrong thing. (B10)
- **Deriving "does this exist" from a launch-time snapshot** — an item created
  since launch looks deleted, and pruning drops it. Query live. (B10)
- **Treating a workspace that failed to read as empty** — empties the user's open
  list on a revoked permission. (B10)

### Content and behaviour

- **`ParagraphStyle` in the editor breaking the cursor** — hanging indents are
  read-only; keep the editor's identity offset mapping. Applies to source-editing
  surfaces only. (A3, B8)
- **Two surfaces disagreeing about the same fact** — derive both from the same
  resolved state, never independently from raw history. (A8)
- **Silently substituting another item when the current one is gone** — show the
  empty state instead; substitution invites acting on the wrong item. (A8)
- **A write that fails silently after an optimistic UI update** — roll back and
  tell the user. (A8, B10)
- **Using `CONTEXT_CLICK` for taps** — it is the platform's faintest effect, so
  the app reads as having broken haptics. Use `VIRTUAL_KEY`. (B14)
- **An API-30 haptic constant without a version guard** — `GESTURE_START`,
  `GESTURE_END`, `CONFIRM`, `REJECT` all need one at `minSdk 26`. (B14)
- **Haptic feedback on a disabled control** — implies something happened when
  nothing did. (A9, B14)
- **Unhandled `onDragCancel`** — leaves a swiped row stuck part-open. (B6)
