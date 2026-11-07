# Rectangle

Rectangle is a macOS window-management app (a Swift port and successor to Spectacle, with Magnet-style edge snapping). It runs as a menu-bar agent that moves and resizes the frontmost window via macOS Accessibility APIs in response to global keyboard shortcuts, drag-to-edge snapping, title-bar double-clicks, and `rectangle://` URLs.

- **Bundle ID:** `com.knollsoft.Rectangle`
- **Deployment target:** macOS 10.15+ (some features gated on macOS 11/13/26)
- **Language / toolchain:** Swift 5, Xcode project (no `Package.swift` at root)
- **Dependencies (SPM, fetched by Xcode):**
  - [`MASShortcut`](https://github.com/rxhanson/MASShortcut) — global keyboard shortcut recorder/binder. The original repo was archived; the project uses the maintainer's fork. Changes that touch MASShortcut go to the fork.
  - [`Sparkle`](https://sparkle-project.org) — auto-updates.
- **Targets:** `Rectangle` (the app), `RectangleLauncher` (login-item helper, used as a fallback before macOS 13's `SMAppService`), `RectangleTests` (currently a stub — only an empty example test).

## Build / run

Open `Rectangle.xcodeproj` in Xcode and run the `Rectangle` scheme. CI builds via:

```sh
xcodebuild -project Rectangle.xcodeproj -scheme Rectangle archive CODE_SIGN_IDENTITY=- -archivePath build/Rectangle.xcarchive
```

> Note from README: the Liquid Glass app icon requires macOS 26 SDK. On older macOS, delete the "Asset Catalog Other Flags" entry to build locally — but **don't commit that change**.

## High-level architecture

`AppDelegate` is the central wiring point. On launch it:

1. Loads any `~/Library/Application Support/Rectangle/RectangleConfig.json` (and renames it with a timestamp so it's not re-loaded), then NSUserDefaults via `Defaults`.
2. Requests Accessibility permission (`AccessibilityAuthorization`); only after the user grants it does it construct the runtime managers.
3. Builds the status bar menu (`RectangleStatusItem`), with menu items dynamically populated from `WindowAction.active`.
4. Wires Sparkle, login-item state (`LaunchOnLogin` / `SMLoginItemSetEnabled` fallback), and observers for app focus changes and config import.

Once Accessibility is trusted, these collaborators are created:

| Component | Role |
|---|---|
| `WindowCalculationFactory` | Holds singleton `WindowCalculation` instances and the `[WindowAction: WindowCalculation]` lookup. |
| `WindowManager` | Executes a `WindowAction` against the frontmost window: detect screen → run calculation → apply gaps → move via a `WindowMover` chain → record action. |
| `ShortcutManager` | Registers default shortcuts and binds them via `MASShortcutBinder` to `WindowAction.post`; routes notifications to `WindowManager.execute`. Filters through `MultiWindowManager` and `TodoManager` first. |
| `ApplicationToggle` | Per-app shortcut disabling ("Ignore frontmost.app"). Unregisters MAS shortcuts when an ignored app is frontmost; re-registers when not. |
| `SnappingManager` | Listens to mouse events (`EventMonitor` → active or passive), tracks drags, computes the `SnapArea` under the cursor, draws a `FootprintWindow` preview, and posts the snap action on mouse-up. Handles the Mission Control corner kludge and Stage Manager. |
| `TitleBarManager` | Double-click-on-title-bar handling. |
| `TodoManager` | "Todo Mode" — pin a sidebar app/window and reflow the rest. Intercepts actions before the WindowManager. |

### How an action flows end-to-end

The same `WindowAction` enum value is the unit of work for every input source. Each input posts a `Notification` with an `ExecutionParameters` object whose `source` field disambiguates origin:

```
keyboard shortcut  →  MASShortcut → WindowAction.post(source: .keyboardShortcut)
drag-to-edge       →  SnappingManager → WindowAction.postSnap(source: .dragToSnap)
status-bar menu    →  AppDelegate → WindowAction.postMenu(source: .menuItem)
rectangle:// URL   →  AppDelegate.application(_:open:) → WindowAction.postUrl(source: .url)
double-click title →  TitleBarManager → WindowAction.postTitleBar(source: .titleBar)
                          │
                          ▼
                ShortcutManager.windowActionTriggered (single funnel)
                          │
                          ├─ MultiWindowManager.execute  (tile-all, cascade-all, reverse-all)
                          ├─ TodoManager.execute         (todo-mode interception)
                          └─ WindowManager.execute       (default path)
                                  │
                                  ├─ ScreenDetection (cursor-based or window-based)
                                  ├─ WindowCalculationFactory.calculationsByAction[action]
                                  ├─ GapCalculation.applyGaps
                                  └─ WindowMover chain
                                        ├─ StandardWindowMover  /  CenteringFixedSizedWindowMover
                                        └─ BestEffortWindowMover (retry/clamp)
```

`WindowManager` also maintains `AppDelegate.windowHistory` per-window:
- `lastRectangleActions[windowId]` — last action + resulting rect + repeat count, used to detect "subsequent execution" for cycling/traversal.
- `restoreRects[windowId]` — original rect captured before the first Rectangle action, used by the `restore` action.

### Subsequent-execution behaviour

Repeating the same action does different things depending on `Defaults.subsequentExecutionMode`:
- `.resize` — cycle through size variants (e.g. left-half → left-third → left-two-thirds).
- `.acrossMonitor` — move to the same position on the next display.
- `.acrossAndResize` — across displays, then resize when wrapping.
- `.cycleMonitor` — short-circuited up in `ShortcutManager.windowActionTriggered`, swaps to the adjacent screen before delegating.
- `.resizeAndCycleQuadrants` — quadrant cycling.

Calculations branch on this enum (see `LeftRightHalfCalculation.calculate` for the canonical example).

## Code map

```
Rectangle/
  AppDelegate.swift                  Entry point, menu wiring, URL handling, Sparkle, Todo menu
  Defaults.swift                     Typed wrappers around NSUserDefaults (BoolDefault, FloatDefault, JSONDefault, IntEnumDefault, etc.)
  WindowAction.swift                 The 130+ action enum; display names, default shortcuts (Spectacle vs. alternate/Magnet), notification routing
  WindowActionCategory.swift         Menu grouping
  WindowManager.swift                Apply a WindowAction to a window
  ShortcutManager.swift              MASShortcut binding + dispatch funnel
  ApplicationToggle.swift            Per-app shortcut ignore
  ScreenDetection.swift              UsableScreens / adjacent screens / cursor-based detection
  AccessibilityElement.swift         AXUIElement wrapper (frame, role, fullscreen, isResizable, etc.)
  TitleBarManager.swift              Double-click on title bar
  RectangleStatusItem.swift          NSStatusItem singleton
  WindowHistory.swift                Per-windowId restore rect + last-action records
  CycleSize.swift / SubsequentExecutionMode.swift   Repeat-action variants

  WindowCalculation/                 100+ files, one per action. Subclass WindowCalculation, implement calculate / calculateRect.
                                     WindowCalculation.swift hosts WindowCalculationFactory and the action→calc lookup.
  WindowMover/                       Strategy chain that actually pokes AX:
                                       StandardWindowMover, CenteringFixedSizedWindowMover, BestEffortWindowMover, QuantizedWindowMover

  Snapping/
    SnappingManager.swift            Mouse-event monitoring, drag tracking, snap selection
    SnapAreaModel.swift              Configurable snap area assignments (landscape/portrait)
    FootprintWindow.swift            The translucent preview window
    CompoundSnapArea/                Multi-action snap regions (halves/thirds/fourths/sixths/eighths)

  MultiWindow/                       tile-all, cascade-all, reverse-all
  TodoMode/TodoManager.swift         Sidebar Todo Mode
  AccessibilityAuthorization/        Permission prompt + window
  PrefsWindow/                       Storyboard-driven Settings UI
    SettingsViewController.swift     Settings tab (the big one — 66KB)
    PrefsViewController.swift        Shortcut tab (lots of @IBOutlet MASShortcutView)
    SnapAreaViewController.swift
    Config.swift                     JSON import/export of shortcuts + defaults
  Popover/MessageView                Tooltip-ish popovers
  Logging/LogViewer                  In-app log window (alt-click "About" to reach it)
  WelcomeWindow/                     First-launch shortcut-style picker (Spectacle vs. Recommended)
  Utilities/
    AXExtension, CFExtension, CGExtension, NSImageExtension, etc.
    EventMonitor.swift               ActiveEventMonitor (CGEventTap, can mutate events) + PassiveEventMonitor (NSEvent monitor)
    StageUtil.swift                  Stage Manager detection
    MacTilingDefaults.swift          Detection / nudge for macOS built-in tiling
    Debounce.swift, TimeoutCache.swift, RunLoopThread.swift
    NotificationExtension.swift      Custom Notification.Name helpers (used pervasively for cross-component signaling)

  Base.lproj/Main.storyboard         All UI lives here
  mul.lproj/Main.xcstrings           Localizations (single multilingual catalog, ~1.2MB)
  Assets.xcassets, AppIcon.icon      Icons including Big Sur and Liquid Glass variants
  Rectangle.entitlements / RectangleRelease.entitlements
  Info.plist, Rectangle-Bridging-Header.h

RectangleLauncher/                   Tiny helper app, set as login item on macOS <13 to start Rectangle at login
RectangleTests/                      Stub XCTestCase, currently empty
```

## Conventions worth knowing

- **Coordinate flips.** macOS NSScreen frames are bottom-left origin; AX frames are top-left origin. The codebase uses a `screenFlipped` extension on `CGRect` to convert. `WindowCalculation` works in normalized (top-left) coordinates; `WindowMover` works in AX (top-left) — `WindowManager.apply` flips back via `result.calcResult.rect.screenFlipped` before calling movers.
- **Adding a new size/position.** Per CONTRIBUTING.md, new actions go in "Extra Shortcuts" rather than into the main Shortcuts tab. The mechanical steps:
  1. Add a case to `WindowAction` (with a unique `Int` rawValue).
  2. Add it to the `WindowAction.active` array (order is the menu order).
  3. Implement a `WindowCalculation` subclass under `WindowCalculation/`.
  4. Register it in `WindowCalculationFactory.calculationsByAction`.
  5. Add display name / image / category to `WindowAction` extensions.
  6. Add the URL alias in the README's `execute-action` table if relevant.
- **Defaults pattern.** Don't read `UserDefaults.standard` directly — use the `Defaults` typed wrappers. New options become a `static let foo = BoolDefault(key: "foo")` (or `FloatDefault`, `JSONDefault<T>`, `IntEnumDefault<E>`, `OptionalBoolDefault`). `OptionalBoolDefault` distinguishes never-set from explicitly false (`userEnabled` / `userDisabled` / `notSet`).
- **Cross-component signaling.** Components communicate via `NotificationCenter` with names declared as `Notification.Name` extensions in `NotificationExtension.swift`. Patterns: `Notification.Name.foo.post()` to fire, `Notification.Name.foo.onPost { ... }` to subscribe. Don't add direct cross-class references when a notification will do.
- **Localization.** All user-visible strings go through `NSLocalizedString` (`.localized` helper extension is used too). The catalog is `Rectangle/mul.lproj/Main.xcstrings`.
- **Settings UI.** Built in `Main.storyboard` + Cocoa bindings + `@IBOutlet`s. The Settings tab is one large view controller (`SettingsViewController.swift`); editing it usually means editing both the `.swift` and the storyboard.
- **Tests.** There are effectively no tests — `RectangleTests.swift` is stub scaffolding. Don't expect green CI to mean behavior is verified; verify manually in Xcode.
- **External-state quirks.** When a window's frame doesn't match Rectangle's last recorded rect, Rectangle assumes the user moved it externally and clears history. Some apps (iTerm2 character-grid resizing, MATLAB, Adobe Illustrator, AfterEffects, Live2D, DataStudio, thinkorswim, Trader Workstation) misbehave with snapping; bundle IDs are hardcoded in `AppDelegate.checkForProblematicApps` and `SnappingManager.fullIgnoreIds`.
- **Logging.** Use `Logger.log(...)` (in `Logging/`). User can see the log via alt-click on the menu's "About" item.
- **Style.** Match surrounding code per CONTRIBUTING.md. The codebase prefers terse, single-purpose classes (see the per-position calculation files). The license is MIT.

## Important non-obvious behaviours

- The first-launch `WelcomeWindow` selection sets both `alternateDefaultShortcuts` (Spectacle defaults vs. Magnet-ish recommended) **and** `subsequentExecutionMode`. These are migrations, not just preferences — older installs are detected via `Defaults.lastVersion` in `AppDelegate.checkVersion()`, which also runs `MASShortcutMigration.migrate()` and `SnapAreaModel.instance.migrate()` for upgrades.
- `windowSnapping` (the snap-on-drag feature) defaults to enabled but is auto-disabled while certain "problem" apps are frontmost, and globally disabled when a fullscreen window is frontmost — see `SnappingManager.checkFullScreen` / `frontAppChanged`.
- Restore (`restore` action, default `⌃⌥⌫`) restores the rect captured the **first** time the user invoked any Rectangle action on that window, not the previous rect.
- `rectangle://execute-action?name=...` URLs are how scripts/Stream Deck/Shortcuts integrations drive Rectangle. The full list lives in the README; the URL → action mapping is computed live by camelCase-to-kebab-case in `AppDelegate.application(_:open:)` against `WindowAction.active`, so a new action becomes URL-addressable for free.
