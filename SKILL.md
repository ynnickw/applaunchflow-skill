---
name: ios-marketing-capture
description: Use when the user wants to automate capture of marketing screenshots for a SwiftUI or Flutter iOS app across multiple locales, devices, or appearances. Covers full-screen shots, isolated element renders (carousel cards, widgets), and reproducible output naming. Triggers on marketing screenshots, locale screenshots, widget renders, App Store assets, fastlane-alternative, simctl screenshots, flutter screenshots.
---

# iOS Marketing Capture

## Overview

Automate reproducible marketing screenshot capture for a SwiftUI or Flutter iOS app across multiple locales, with two parallel output streams:

1. **Full-screen captures** — every marketing-relevant screen, with deterministic seeded data, real status bar / safe-area chrome
2. **Element captures** — isolated renders of specific components (cards, widgets, charts) at any scale, with natural background inside rounded corners and transparency outside

This skill is the **capture** step. After capture and verification, it also evaluates the resulting screenshots and preselects up to 10 App-Store-ready candidates before optionally handing them off to the **AppLaunchFlow MCP** for layout generation. This requires the `applaunchflow-mcp` MCP server to be configured.

## Framework Detection

Before starting, determine whether the project is **SwiftUI** or **Flutter**:

- Look for `pubspec.yaml` in the project root → **Flutter**
- Look for `*.xcodeproj` or `Package.swift` with SwiftUI imports → **SwiftUI**

If the project is Flutter, skip to the **Flutter Path** section below. If SwiftUI, continue with the **SwiftUI Path** immediately following.

---

# SwiftUI Path

## Core Approach

**In-app capture mode**, not XCUITest. This is a hard decision that trades off against Fastlane snapshot / XCUITest conventions, and it wins for almost every real project.

Why in-app over XCUITest:

- **No new test target.** Adding a UI test target to an existing Xcode project is fragile pbxproj surgery. Many projects have zero test targets and no xcodegen — adding one by hand is error-prone.
- **Faster iteration.** A UI test takes 30s+ to launch per run. In-app capture is just a relaunch of the installed binary.
- **No `xcodebuild test`.** The whole flow is `xcodebuild build` once, then `simctl launch` per locale. No test-bundle overhead.
- **Access to real app state.** You can call ViewModels, SwiftData, ImageRenderer, and `UIWindow.drawHierarchy` directly. XCUITest can only tap and read accessibility elements.
- **Element renders need in-process anyway.** `ImageRenderer` on widget views or isolated components must run inside the app process — there's no XCUITest equivalent.

How it works:

1. A DEBUG-only `MarketingCapture.swift` file lives in the main app target
2. When launched with `-MarketingCapture 1`, the app seeds data, then a coordinator walks a list of `CaptureStep`s — each step navigates, waits for settle, snapshots, and cleans up
3. PNGs are written to the app's sandbox `Documents/marketing/<locale>/` directory
4. A shell script builds once, installs, then loops locales by relaunching with `-AppleLanguages (xx) -AppleLocale xx`, pulling files out via `simctl get_app_container`

## Process

Work through these steps in order. Do not skip ahead.

### Step 1: Gather requirements

**Default behavior: infer everything from the codebase and act.** Only ask when a detail is genuinely ambiguous and getting it wrong would produce a materially wrong result.

Auto-detect the following during exploration (Step 2) before asking any questions:

- **Screens** — enumerate all user-visible screens from the navigation structure (tabs, routes, pushed views). Default to capturing all marketing-relevant screens. Only ask if the app has 15+ screens and you need to prioritize.
- **Locales** — discover supported locales from `Localizable.xcstrings`, `.arb` files, or `supportedLocales` in the code. Default to all discovered locales. Only ask if there are 10+ locales and you want to confirm scope.
- **Device** — check for a booted simulator via `xcrun simctl list devices booted`. Use it. If none is booted, use iPhone 17 or the latest available. Only ask if no simulator is available.
- **Appearance** — default to light only. Only ask if the app has explicit dark mode theming.
- **Seed data** — determine from exploration whether demo data exists. If not, create a seeder. Don't ask the user to describe their data strategy — audit the code.

If the user gives a vague request like "capture screenshots for my app," proceed with exploration immediately. If they give a specific request like "capture Home, Library, and Progress screens in English and German," respect those constraints.

The only question worth asking upfront if not inferable:

- **Isolated elements** — "I see cards/widgets/charts in your app. Want me to render any of these as isolated elements with transparent backgrounds, or just full-screen captures?"

Skip this too if the user said "all screens" or similar — just capture full screens and mention element rendering as a follow-up option.

### Step 2: Exploration

Before writing any code, explore the codebase enough to answer:

- Does the project use **Xcode synchronized folder groups** (Xcode 16+, `PBXFileSystemSynchronizedRootGroup`)? If yes, new files auto-include in their target — no pbxproj edits needed. Check with `grep -c PBXFileSystemSynchronized <proj>.xcodeproj/project.pbxproj`.
- **What is the root navigation pattern?**
  - `TabView(selection:)` — most common. You need: the `@State selectedTab` binding, tab indices, and which tabs have nested `NavigationStack`.
  - `NavigationStack` (single stack with a router) — you need: the path binding or router object, plus the set of `NavigationLink(value:)` / `.navigationDestination` types.
  - `NavigationSplitView` — you need: the sidebar selection binding, detail column's navigation state.
  - Custom coordinator / UIKit host — you need: the coordinator's `navigate(to:)` method or equivalent.
- How are **deep links** routed? Find the `onOpenURL` handler and the enum/switch that maps URLs to navigation state.
- Where are **demo data seeders** defined? Trace the code path from the debug button (if any) to the function that actually writes to `ModelContext`. If no seeder exists, see "Creating a demo data seeder" below.
- Do **widgets** live in a separate target? Are the widget view files and entry types in the main app target too? (Almost certainly no — they need to be added if you want to render them via ImageRenderer.)
- Does the app use **Live Activities** / ActivityKit? If yes, flag this as a known gotcha (see below).
- Does the app use **SwiftData + CloudKit sync** (`cloudKitDatabase: .automatic`)? If yes, flag as a known gotcha.
- Does any view need to be **captured in a non-default state**? (e.g. a timer mid-countdown, a form partially filled, a chart with specific values). If yes, each needs a `static var` priming mechanism (see "Priming view state" below).
- **What locales are supported?** Discover from `Localizable.xcstrings`:
  ```bash
  python3 -c "import json; d=json.load(open('<path>/Localizable.xcstrings')); langs=set(); [langs.update(v.get('localizations',{}).keys()) for v in d['strings'].values()]; print(sorted(langs))"
  ```
  Or from `.lproj` directories: `ls -d *.lproj | sed 's/.lproj//'`. Default to all discovered locales.
- **What device is currently booted?** Check with `xcrun simctl list devices booted`. Use the booted simulator. If none is booted, default to iPhone 17.

### Step 3: Present design to user

Before writing code, summarize your plan in this structure. Get explicit approval before proceeding:

1. Architecture (in-app capture mode, single file, DEBUG-gated)
2. File list (exact paths you'll create / modify)
3. Screen-by-screen capture plan (how each screen is reached — tab index, navigation path, sheet trigger)
4. Capture ordering rationale (which screens must come before others — see gotcha #5)
5. Element rendering approach (which components, how they'll be wrapped)
6. Output layout (folder structure, naming convention)
7. Known gotchas relevant to this project (flagged from Step 2)
8. Primed states needed (which views, what static vars)

### Step 4: Implement

Use the templates in `templates/` as starting points. They are **reference patterns**, not copy-paste scaffolding — every project has different navigation, models, and views. The templates show the building blocks; you compose them for the target app.

Key files to produce:

- `<AppName>/Debug/MarketingCapture.swift` — the whole capture system, DEBUG-only. Contains:
  - `MarketingCapture` enum (launch arg parsing, output helpers, window snapshot, priming vars)
  - `MarketingCaptureCoordinator` class (walks `[CaptureStep]` and snapshots each)
  - `MarketingElementHarness` enum (ImageRenderer renders of cards, widgets, charts)
- `<AppName>/ContentView.swift` (or wherever the root view lives) — DEBUG hook that seeds data and runs the coordinator.
- Any views that need primed states — DEBUG-gated `.onAppear` hooks and `.onReceive` dismiss listeners.
- `scripts/capture-marketing.sh` — build + install + per-locale loop.
- `.gitignore` — add `marketing/`.

### Step 5: Verify iteratively

Do **not** hand the script to the user and wait. Run it yourself against a simulator and verify at least one locale before declaring done. Read the output PNGs with the Read tool to visually verify each screen shows what you expect. Common runtime issues are listed in "Known Gotchas" below.

When you find an issue, fix it, rerun the whole script (not just the failing locale — fixes can regress earlier locales), and re-verify visually.

### Step 6: Evaluate and preselect App Store candidates

After Step 5 verification passes, evaluate the captured **full-screen** screenshots and produce a shortlist of up to **10** files that are strongest for App Store use.

Do not skip this step when the user wants store-ready assets. The App Store upload limit is 10 screenshots per size class, so the skill should reduce a large capture set into a curated marketing sequence.

Rules:

- Evaluate **full-screen captures only**. Ignore `elements/` renders for shortlist selection.
- Use the **primary locale** as the source of truth for selection quality unless the user explicitly asks for locale-specific curation.
- Prefer a coherent story over raw completeness. The shortlist should feel like a landing page sequence, not a dump of every screen.
- Aim for **6-10** screenshots when possible. Use fewer only if the app genuinely has fewer strong marketing screens.
- Every selected screenshot must earn its slot. If two images communicate the same feature, keep the stronger one.

Selection criteria:

- **Instant clarity**: a user should understand the screen's value in under 2 seconds.
- **Feature relevance**: the screen should represent a feature worth marketing, not just navigation chrome.
- **Visual strength**: populated UI, good hierarchy, clear focal point, attractive data density.
- **Distinctiveness**: each chosen screenshot should add a new message or product angle.
- **Composability**: the screen should work well inside App Store layouts with space for headlines and framing.
- **State quality**: avoid loading states, empty states, placeholder copy, debug controls, or accidental overlays unless the empty state itself is intentionally marketable.

Hard exclusions:

- Settings screens unless the app's core value is configuration or customization.
- Redundant variants of the same list/detail state with no new marketing value.
- Transitional modals, alerts, toasts, permission prompts, and half-dismissed sheets.
- Screens whose meaning depends on prior explanation.
- Screens with visibly fake, broken, clipped, or sparse seed data.

Story-building heuristic:

1. Start with the clearest hero/value screen.
2. Follow with the most important core workflow screens.
3. Include proof-of-value screens: results, analytics, progress, personalization, or output.
4. Include secondary differentiators only if they add something new.
5. End with a strong closing screen only if it is genuinely compelling.

Deliverables:

- A ranked shortlist of up to 10 filenames with one-line reasons for each.
- A curated folder at `marketing/<primaryLocale>/app-store-shortlist/` containing only the selected full-screen PNGs, renamed in presentation order.
- If multiple locales were captured, mirror the same shortlist filenames into each locale's `app-store-shortlist/` folder when the corresponding localized PNG exists. If a locale is missing a counterpart, flag it explicitly instead of guessing.

Implementation notes:

- Inspect the actual PNGs visually before selecting. Do not infer quality from filenames alone.
- Use local image viewing/reading tools on the generated files.
- Keep the original capture output untouched. The shortlist is an additional curated export.
- Rename shortlisted files with stable sequence numbers, e.g. `01-hero.png`, `02-plan-overview.png`, `03-progress.png`.

### Step 7: Generate App Store screenshot layouts (optional)

After Step 6 shortlist evaluation is done, ask the user:

> "Screenshots captured and verified. Would you like me to generate App Store screenshot layouts using AppLaunchFlow? This creates professional marketing layouts with device frames, headlines, and backgrounds."

If the user declines, stop here — the capture pipeline is done.

#### 7a. Create project

Use the `create_project` MCP tool. Infer values from the codebase explored in Step 2:

- `appName` — the Xcode scheme or product display name. Ask only if ambiguous.
- `platform` — `"ios"`
- `category` — infer from the app's purpose (e.g. "Health & Fitness", "Productivity"). Ask only if genuinely unclear.
- `appDescription` — one-sentence summary inferred from exploration.

Store the returned `projectId`.

#### 7b. Upload screenshots

Use the `upload_screenshots` MCP tool:

- `projectId` — from 7a
- `deviceType` — `"mobile"`
- `platform` — `"ios"`
- `sources` — `[{ "path": "marketing/<primaryLocale>/app-store-shortlist/" }]` where `<primaryLocale>` is the first entry in the `LOCALES` array

The tool auto-expands directories, picking up only top-level image files.

Upload only the curated **primary locale** shortlist. These are the source screenshots for layout generation. Element renders (cards, widgets) are not used by `generate_layouts`.

#### 7c. Select a template

Use the `browse_templates` MCP tool:

- `deviceType` — `"phone"`
- `title` — the app name from 7a

This opens a visual gallery. Wait for the user to select a template. Store the returned `templateId`.

#### 7d. Generate layouts

Use the `generate_layouts` MCP tool:

- `generationId` — the `projectId` from 7a (`generationId` = `projectId`)
- `templateId` — from 7c (if selected)
- `deviceType` — `"phone"`

**Present the returned `editorUrl` to the user.** This is their entry point to review and refine layouts in the visual editor.

#### 7e. Translate layouts (if multiple locales)

If the capture ran for more than one locale, ask:

> "Your screenshots were captured in N locales: [list]. Would you like me to translate the App Store layout text for the other locales?"

If yes, use the `translate_layouts` MCP tool:

- `generationId` — the `projectId`
- `targetLanguages` — all captured locale codes except the primary locale
- `layouts` — `["mobile"]`

**Important:** `translate_layouts` translates the **marketing copy** in the layouts (headlines, subtitles), not the app UI text. The app UI is already localized in the per-locale screenshots. If the user wants each locale's layout to show that locale's own shortlisted screenshots instead of translated headlines over the primary-locale shortlist, this requires uploading shortlisted screenshots per locale and generating separate layouts — flag this as an advanced workflow.

#### Error handling

- **Auth failures on any MCP call:** Tell the user to verify the AppLaunchFlow MCP server is configured and authenticated (`npx @applaunchflow/mcp@latest auth login`).
- **`upload_screenshots` fails:** Verify files exist at `marketing/<locale>/` by listing the directory.
- **`generate_layouts` fails:** Call `list_screenshots` to verify uploads succeeded.
- **`browse_templates` returns a URL instead of completing interactively:** Present the gallery URL to the user and ask them to reply with the template name or id.

## Architecture: Step-Based Capture

The coordinator drives capture by walking a list of `CaptureStep` values. Each step is self-contained: it knows how to navigate to its screen, how long to wait, and how to clean up afterward.

```swift
struct CaptureStep {
    let name: String                        // output filename, e.g. "01-home"
    let navigate: @MainActor () -> Void     // put the app in the right state
    let settle: Duration                    // wait for animations/loads
    let cleanup: (@MainActor () -> Void)?   // tear down before next step
}
```

The coordinator is a simple loop:

```swift
for step in steps {
    step.navigate()
    try? await Task.sleep(for: step.settle)
    if let image = MarketingCapture.snapshotKeyWindow() {
        MarketingCapture.writePNG(image, name: step.name)
    }
    step.cleanup?()
    try? await Task.sleep(for: .milliseconds(400))  // cleanup animation
}
```

### Building steps for different navigation patterns

**TabView app** (most common):
```swift
// Simple tab switch — just set the index
CaptureStep(name: "01-home", navigate: { setTab(0) }, settle: .milliseconds(1800), cleanup: nil)

// Tab + presented sheet
CaptureStep(
    name: "05-timer-setup",
    navigate: {
        setTab(3)
        pendingBrewRecipe = someRecipe
    },
    settle: .milliseconds(2000),
    cleanup: {
        NotificationCenter.default.post(name: MarketingCapture.dismissSheetNotification, object: nil)
        pendingBrewRecipe = nil
    }
)
```

**NavigationStack + router app:**
```swift
// Push a route onto the stack
CaptureStep(
    name: "02-detail",
    navigate: { router.push(.itemDetail(item)) },
    settle: .milliseconds(1800),
    cleanup: { router.popToRoot() }
)
```

**NavigationSplitView app:**
```swift
// Select sidebar item, then detail
CaptureStep(
    name: "03-detail",
    navigate: {
        sidebarSelection = .recipes
        detailSelection = recipes.first
    },
    settle: .milliseconds(1800),
    cleanup: { detailSelection = nil }
)
```

### Ordering: the stacking rule

**Capture any screen that needs a "clean" navigation state BEFORE screens that push onto the same stack.** Nested `NavigationPath` / `@State` inside child views can't be popped from the coordinator. So:

```
Good:  Shelf (clean list) → Coffee Detail (pushes onto shelf's stack)
Bad:   Coffee Detail → Shelf (stack still has detail pushed)
```

If two screens share a NavigationStack, capture the root-level view first.

## Priming View State

Some screens need to be captured in a specific non-default state — a timer mid-countdown, a chart with particular values, a form half-filled. The pattern:

1. Add a `static var` to `MarketingCapture` for each priming value:
   ```swift
   /// Set by the coordinator before presenting the timer view.
   /// The view reads this in .onAppear to jump to a specific elapsed time.
   static var pendingElapsedSeconds: Int?

   /// Set to true to show the assessment overlay on the timer.
   static var pendingShowAssessment: Bool = false
   ```

2. In the target view, add a DEBUG-gated `.onAppear` that reads the priming value:
   ```swift
   .onAppear {
       #if DEBUG
       if MarketingCapture.isActive, let elapsed = MarketingCapture.pendingElapsedSeconds {
           phase = .active
           timerVM.elapsedTime = TimeInterval(elapsed)
           timerVM.start()
           DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) { timerVM.pause() }
       }
       #endif
   }
   ```

3. In the coordinator, set the var before navigating:
   ```swift
   CaptureStep(
       name: "06-timer-midway",
       navigate: {
           MarketingCapture.pendingElapsedSeconds = 75
           openTimerSheet(someRecipe)
       },
       settle: .milliseconds(2400),
       cleanup: {
           MarketingCapture.pendingElapsedSeconds = nil
           NotificationCenter.default.post(name: MarketingCapture.dismissSheetNotification, object: nil)
       }
   )
   ```

## Creating a Demo Data Seeder

If the app has no existing demo data mechanism, create one. Place it in `<AppName>/Debug/DemoDataSeeder.swift`, wrapped in `#if DEBUG`.

Guidelines:
- Seed **enough data that every captured screen looks populated**. Audit the screen list against the seed.
- Use realistic content: real place names, plausible numbers, varied states (some items "running low", some "fresh", some with images, some without).
- If the app uses SwiftData, write directly to the `ModelContext`. If Core Data, use the managed object context. If a REST backend, seed via the local cache/store layer.
- Make seeding **idempotent** — check if data already exists before inserting. The store persists across simulator relaunches, and re-seeding per locale causes CloudKit sync churn and crashes.
- Include enough variety to fill different UI states: empty states should NOT appear unless they're a marketing screen.

Minimal shape:
```swift
#if DEBUG
enum DemoDataSeeder {
    static func seedIfEmpty(in context: ModelContext) {
        let existing = (try? context.fetchCount(FetchDescriptor<Item>())) ?? 0
        guard existing == 0 else { return }

        // Items with varied states
        let items = [
            Item(name: "...", status: .active, ...),
            Item(name: "...", status: .lowStock, ...),
            // ...enough to fill every screen
        ]
        items.forEach { context.insert($0) }
        try? context.save()
    }
}
#endif
```

## Element Rendering

Elements are rendered via `ImageRenderer` at 3x scale with transparency outside rounded corners.

### Cards / list rows

```swift
@MainActor
static func renderCards(items: [Item], theme: AppTheme) {
    let cardWidth: CGFloat = 380

    for item in items {
        let card = ItemCard(item: item, theme: theme)
            .padding(.horizontal, 16)
            .padding(.vertical, 12)
            .frame(width: cardWidth)
            .background(theme.background)
            .clipShape(RoundedRectangle(cornerRadius: 20, style: .continuous))

        let renderer = ImageRenderer(content: card)
        renderer.scale = 3
        renderer.isOpaque = false
        renderer.proposedSize = .init(width: cardWidth, height: nil)

        guard let image = renderer.uiImage else { continue }
        MarketingCapture.writePNG(image, name: "card-\(slugify(item.name))", subfolder: "elements")
    }
}
```

### Widgets

Widget views require special handling because they normally run inside WidgetKit's process and rely on system-provided padding and backgrounds.

```swift
@MainActor
static func renderWidget(
    name: String,
    size: CGSize,
    cornerRadius: CGFloat? = nil,
    @ViewBuilder content: () -> some View
) {
    let isAccessory = size.height <= 80
    let radius = cornerRadius ?? (isAccessory ? 8 : 22)
    let contentPadding: CGFloat = isAccessory ? 0 : 16

    let view = content()
        .padding(contentPadding)
        .frame(width: size.width, height: size.height)
        .background(theme.background)
        .clipShape(RoundedRectangle(cornerRadius: radius, style: .continuous))
        .environment(\.colorScheme, .light)

    let renderer = ImageRenderer(content: view)
    renderer.scale = 3
    renderer.isOpaque = false
    renderer.proposedSize = .init(width: size.width, height: size.height)

    guard let image = renderer.uiImage else { return }
    MarketingCapture.writePNG(image, name: name, subfolder: "elements")
}

// Standard iPhone widget sizes (points, iPhone 14-17 size class)
enum WidgetSize {
    static let small  = CGSize(width: 170, height: 170)
    static let medium = CGSize(width: 364, height: 170)
    static let large  = CGSize(width: 364, height: 382)
    static let accessoryCircular    = CGSize(width: 76, height: 76)
    static let accessoryRectangular = CGSize(width: 172, height: 76)
    static let accessoryInline      = CGSize(width: 257, height: 26)
}

// Usage:
renderWidget(name: "widget-pulse-small", size: WidgetSize.small) {
    PulseSmallView(entry: PulseEntry(
        date: Date(),
        count: 2,
        streak: 5,
        lastItemName: "Morning Routine"
    ))
}
```

### Charts / standalone views

Any SwiftUI view can be rendered as an element. Wrap it the same way — explicit size, background, corner clip:

```swift
@MainActor
static func renderChart() {
    let chart = MyChartView(values: ChartData.sample)
        .frame(width: 420, height: 420)
        .background(theme.background)
        .clipShape(RoundedRectangle(cornerRadius: 32, style: .continuous))

    let renderer = ImageRenderer(content: chart)
    renderer.scale = 3
    renderer.isOpaque = false
    renderer.proposedSize = .init(width: 420, height: 420)

    guard let image = renderer.uiImage else { return }
    MarketingCapture.writePNG(image, name: "chart-overview", subfolder: "elements")
}
```

## Known Gotchas

These are all real bugs that bit a real project. Treat this list as load-bearing.

### 1. Live Activities persist across app launches

ActivityKit Live Activities **outlive process termination**. If your app starts a Live Activity during capture (e.g. via a timer's `start()`), then the next locale's relaunch will inherit it. Combined with a fresh seed that deletes the models the stale LA references, you get SwiftData persisted-property assertions.

Fix: call `<ActivityManager>.shared.endImmediately()` at the very start of the marketing capture block, before touching data. Also call `timerVM.stop()` (or whatever properly ends the LA) in the view's `onDisappear` when in capture mode.

### 2. Don't re-seed on every locale

Seeding SwiftData + CloudKit per locale causes sync churn and crashes. The SwiftData store persists across relaunches — the data is locale-agnostic demo content, so seed **once** on the first run and skip subsequent runs:

```swift
contentVM.fetchItems()
if contentVM.allItems.isEmpty {
    DemoDataSeeder.seedIfEmpty(in: modelContext)
    contentVM.fetchItems()
}
```

### 3. ViewModels that setup before the seed hold stale snapshots

If the root view's `onAppear` calls `someVM.setup(modelContext:)` **before** the marketing seed runs, the VM holds a snapshot of the empty store. After seeding, call `someVM.refresh()` (or its equivalent fetch method) for every VM whose data you need.

### 4. Setting a trigger binding to nil does NOT dismiss a sheet

If a parent view presents a `.fullScreenCover(item: $request)` and `request` is driven by an internal `@State`, then setting the *trigger* binding (e.g. `pendingItem = nil`) does nothing to the cover. The cover stays up, and your next screenshot captures it instead of the screen you navigated to.

Fix: broadcast a dismiss signal via NotificationCenter, and have the presented view listen:

```swift
// MarketingCapture.swift
static let dismissSheetNotification = Notification.Name("MarketingCapture.dismissSheet")

// In presented view body
.onReceive(NotificationCenter.default.publisher(for: MarketingCapture.dismissSheetNotification)) { _ in
    dismiss()
}
```

Then in the step's `cleanup`, post the notification and allow **at least 900ms** for the cover animation to complete before the next step begins.

### 5. NavigationPath can't be popped from outside

If a child view holds `@State private var navigationPath = NavigationPath()` and a deep link pushes onto it, the coordinator can't reach in to pop. Solution: **reorder your capture sequence** so screens that push onto a stack come AFTER screens that need a clean stack. Example: capture Shelf first, then push into Coffee Detail — don't do it the other way around.

### 6. Widget views normally live in the extension target only

If the user's widget views are only in the widget extension target, you can't reference them from `MarketingCapture.swift` in the main app target. You need to either:

- **(a)** Add the widget view files (and their entry types and any shared helpers) to the main app target's membership. If the project uses synchronized folder groups, this means editing `PBXFileSystemSynchronizedBuildFileExceptionSet.membershipExceptions`. **CRITICAL GOTCHA: `membershipExceptions` is an INCLUSION list, not an exclusion list.** Files listed there ARE members of the target, not excluded from it. Read this twice before editing.
- **(b)** Skip widget rendering from the capture harness and let the user do them manually.

You'll also need to exclude `<App>WidgetBundle.swift` from the main app target (it has `@main` and conflicts with the app's `@main`).

### 7. `ImageRenderer` + `ProgressView(value:total:)` = prohibited symbol

Without an explicit style, `ProgressView` determinate renders as a red circle-with-slash when composited through ImageRenderer. Fix: `.progressViewStyle(.linear)` on the ProgressView. It's a no-op in normal rendering and fixes the render glitch.

### 8. `.containerBackground(for: .widget)` is a no-op outside widget context

When you render a widget view via ImageRenderer in the app, its `.containerBackground` does nothing — the widget's background is transparent, and pixels outside the content are bare. You must wrap the widget render with an explicit background color + rounded rect clip:

```swift
content()
    .padding(16)  // widget container normally provides this
    .frame(width: size.width, height: size.height)
    .background(theme.background)
    .clipShape(RoundedRectangle(cornerRadius: 22, style: .continuous))
```

Home-screen widget corner radius on iPhone: ~22pt. Lock-screen accessory radius: ~8pt.

### 9. iPhone 8 Plus is gone on iOS 26

If the user asks for a "6.5\" iPhone" (legacy App Store size), note that iOS 26+ simulators don't include iPhone 8 Plus / iPhone 11 Pro Max. Options: (a) install an older iOS runtime via Xcode > Settings > Platforms, or (b) fall back to a modern 6.1\" like iPhone 17 for iOS 26 design features.

### 10. Locale launch arguments

Pass `-AppleLanguages (xx) -AppleLocale xx` at every `simctl launch`. The parens around the language code are mandatory (it's a plist array literal). Use `Locale.current.language.languageCode?.identifier` for folder naming — it's more robust than `Locale.current.identifier` which may include region suffixes like `en_US`.

### 11. SwiftUI animations in ImageRenderer

`ImageRenderer` captures a single frame — it doesn't wait for animations. If your component has an `.onAppear` animation (chart drawing, number counting up), the render may capture the initial state. Either disable the animation in capture mode or add an explicit delay before rendering:

```swift
try? await Task.sleep(for: .milliseconds(500))  // let onAppear animations finish
let renderer = ImageRenderer(content: view)
```

---

# Flutter Path

## Core Approach

**Separate entry point + simctl capture.** Flutter apps can't use SwiftUI's `UIWindow.drawHierarchy` or `ImageRenderer`. Instead, the skill creates a standalone Dart entry point (`lib/main_screenshots.dart`) that:

1. Initializes Firebase (or other backend) but puts state management in **offline mode** — no Firestore listeners, no auth gates
2. Seeds demo data directly into the app's state services
3. Auto-navigates through all screens using a coordinator with timed steps
4. Writes sentinel files to `/tmp/` to signal the shell script when each screen is ready
5. A shell script runs `flutter run -t lib/main_screenshots.dart`, watches for sentinels, and captures each screen via `xcrun simctl io <device> screenshot`

Why this approach over integration_test:

- **No Firebase permission issues.** Integration tests create a fresh app install with a new anonymous user. Firestore security rules often reject these users, causing `permission-denied` errors that crash the test. The offline-mode approach bypasses Firestore entirely.
- **No test framework overhead.** `flutter test integration_test/` has flaky `pumpAndSettle` behavior with animations, Firebase streams, and timers. The separate entry point runs the real app widget tree.
- **Real rendering.** Screenshots via `simctl io` capture the full native frame — status bar, safe area, system chrome — exactly as a user would see it.
- **Simpler debugging.** The app runs normally in debug mode with hot reload available. You can iterate on the coordinator without rebuilding.

## Process (Flutter)

Work through these steps in order.

### Step 1: Gather requirements (same as SwiftUI)

Infer everything from the codebase and act. Same auto-detect-first principle as the SwiftUI path — only ask when genuinely ambiguous. For Flutter, locale discovery comes from `supportedLocales` in `MaterialApp` or `.arb` files in `lib/l10n/`.

### Step 2: Exploration (Flutter-specific)

Before writing any code, explore the codebase enough to answer:

- **What state management is used?** Provider, Riverpod, Bloc, GetX, or vanilla? You need to know how to create an instance with seeded data and provide it to the widget tree.
- **What is the root navigation pattern?**
  - `NavigationBar` / `BottomNavigationBar` with `IndexedStack` — most common. You need: the tab index state, the list of screen widgets.
  - `GoRouter` / `Navigator 2.0` — you need: the router configuration and how to push routes programmatically.
  - `Navigator.push` (imperative) — you need: the route builders for each screen.
- **What backend services does the app depend on?** Firebase Auth, Firestore, Supabase, REST APIs? Each service that runs at startup needs to be either initialized or bypassed.
- **Does the state management class accept dependency injection?** Look for constructor parameters like `AppState({AuthService? authService, ...})`. If the state class can be constructed with mock/null services, the offline approach is straightforward.
- **Where are locales defined?** Check `supportedLocales` in `MaterialApp`, look for `.arb` files, `Localizable.xcstrings`, or custom localization systems.
- **Is there existing demo/seed data?** Check for test fixtures, debug menus, or seed functions.
- **Does the app have onboarding or auth gates?** You need to bypass these in the screenshot entry point.

### Step 3: Present design to user (same as SwiftUI)

Same structure — get approval before writing code.

### Step 4: Implement (Flutter)

Use the templates in `templates/` as starting points. Key files to produce:

#### 4a. Add `offlineMode` to the state management class

Add a flag to the app's primary state class that skips all backend initialization:

```dart
class AppState extends ChangeNotifier {
  AppState({
    this.offlineMode = false,
    String? overrideUserName,
  }) {
    if (offlineMode) {
      _isLoading = false;
      if (overrideUserName != null) _userName = overrideUserName;
      return; // Skip Firebase/Firestore/auth initialization
    }
    _initialize(); // Normal startup
  }

  final bool offlineMode;
```

For Riverpod apps, create an override provider. For Bloc, create a pre-seeded bloc instance.

Also add a public method to trigger stats/cache computation if the class has private preloading:

```dart
/// Public so screenshot mode can trigger after seeding.
void refreshStats() => _preloadStats();
```

#### 4b. Create `lib/main_screenshots.dart`

This is the screenshot entry point. Structure:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // If needed

  final appState = AppState(offlineMode: true, overrideUserName: 'Luna');
  _seedDemoData(appState);
  appState.refreshStats(); // Trigger cached stat computation

  runApp(_ScreenshotApp(appState: appState));
}
```

Key rules:
- Provide the state object as the **real type** the screens expect (e.g., `Provider<AppState>.value(value: appState)`). Don't create a mock subclass — the screens use `context.watch<AppState>()` and the type must match.
- Skip auth wrappers and splash screens. Go directly to the main shell widget.
- Use `IndexedStack` with tab index state for the coordinator to switch tabs without rebuilding.
- For detail screens (pushed via Navigator), use `setState` to swap an overlay widget rather than `Navigator.push` — this keeps the coordinator in control.

#### 4c. Build the coordinator

The coordinator is a `StatefulWidget` that auto-navigates through screens on a timer and writes sentinel files:

```dart
class _ScreenshotCoordinator extends StatefulWidget { ... }

class _ScreenshotCoordinatorState extends State<_ScreenshotCoordinator> {
  int _tabIndex = 0;
  bool _showingOverlay = false;
  Widget? _overlayScreen;

  Future<void> _runCaptureSequence() async {
    const pause = Duration(seconds: 3);

    // Tab screens
    for (final (index, name) in [(0, '01-home'), (1, '02-routine'), ...]) {
      setState(() => _tabIndex = index);
      await Future.delayed(const Duration(seconds: 2));
      _writeSentinel(name);
      await Future.delayed(pause);
    }

    // Detail screens via overlay
    setState(() {
      _showingOverlay = true;
      _overlayScreen = DetailScreen(item: someItem);
    });
    await Future.delayed(const Duration(seconds: 2));
    _writeSentinel('05-detail');
    await Future.delayed(pause);
    setState(() { _showingOverlay = false; _overlayScreen = null; });
  }
}
```

Sentinel files are written to `/tmp/<app>-screenshots/<name>.ready`. A `_done` sentinel signals completion.

#### 4d. Seed demo data

Create a `_seedDemoData(AppState appState)` function that populates all services directly:

```dart
void _seedDemoData(AppState appState) {
  final techniques = appState.techniqueService.all;

  // Session logs — 60 days of realistic history
  for (int daysAgo = 0; daysAgo < 60; daysAgo++) {
    if (daysAgo % 7 == 6) continue; // Skip some days
    appState.logService.addLog(SessionLog(
      id: 'demo-$daysAgo',
      dateTime: DateTime.now().subtract(Duration(days: daysAgo)),
      // ...
    ));
  }

  // Routines, favorites, etc.
  appState.routineService.itemsList.addAll([...]);
}
```

Rules:
- Seed **enough data** that every screen looks populated. Audit each captured screen against the seed.
- Use realistic content: plausible names, varied states, mixed completion statuses.
- Access services directly (`.addLog()`, `.itemsList.addAll()`) rather than going through methods that trigger Firestore writes.

#### 4e. Create `scripts/capture-screenshots.sh`

The shell script coordinates `flutter run` with `simctl io` screenshot capture:

```bash
flutter run -t lib/main_screenshots.dart -d "$DEVICE_ID" --no-hot &
FLUTTER_PID=$!

# Wait for each sentinel, capture screenshot
for SCREEN in "01-home" "02-routine" ...; do
    while [ ! -f "/tmp/app-screenshots/$SCREEN.ready" ]; do sleep 0.5; done
    sleep 0.5  # Let rendering settle
    xcrun simctl io "$DEVICE_ID" screenshot "$OUT_DIR/$SCREEN.png"
done
```

**Critical:** Use absolute paths for screenshot output. `flutter run` may reset the shell's working directory.

#### 4f. Multi-locale capture (Flutter)

Unlike SwiftUI where each locale requires a separate `simctl launch`, Flutter can switch locales without relaunching. There are two approaches — pick the one that fits the project.

##### Approach A: `--dart-define` per-run (preferred for ≤ 3 locales)

Add a compile-time locale override to `main_screenshots.dart`:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();

  final appState = AppState(offlineMode: true, overrideUserName: 'Luna');

  // Allow locale override via --dart-define=SCREENSHOT_LOCALE=de
  const localeOverride = String.fromEnvironment('SCREENSHOT_LOCALE');
  if (localeOverride.isNotEmpty) {
    appState.setLocale(Locale(localeOverride));
  }

  _seedDemoData(appState);
  appState.refreshStats();
  runApp(_ScreenshotApp(appState: appState));
}
```

Then run the capture script once per locale with a different output directory:

```bash
# English (default)
flutter run -t lib/main_screenshots.dart -d "$DEVICE_ID" --no-hot

# German
flutter run -t lib/main_screenshots.dart -d "$DEVICE_ID" --no-hot \
  --dart-define=SCREENSHOT_LOCALE=de
```

The shell script stays unchanged — just point `OUT_DIR` to `marketing/de` for the second run. The coordinator, sentinel logic, and screen list are all identical.

Why this is often better:

- **Zero coordinator changes.** The coordinator captures screens exactly once per run — no outer locale loop, no locale-prefixed sentinels.
- **Simpler debugging.** Each locale is a separate run. If German fails, you re-run just German.
- **No risk of locale switch not taking effect.** Some widgets cache text on first build — a fresh process ensures clean localization.

Downside: each locale is a separate `flutter run`, so there's build overhead. For 2-3 locales this is negligible; for 10+ locales, use Approach B.

##### Approach B: In-process locale loop (better for 4+ locales)

The coordinator loops through all locales in a single run, switching `appState.currentLocale` between them:

```dart
Future<void> _runCaptureSequence() async {
  final appState = context.read<AppState>();
  final locales = [const Locale('en'), const Locale('de')]; // From supportedLocales

  for (final locale in locales) {
    final langCode = locale.languageCode;

    // Switch locale and let the widget tree rebuild
    await appState.setLocale(locale);
    await Future.delayed(const Duration(seconds: 2));

    // Capture all screens for this locale
    for (final (index, name) in [(0, '01-home'), (1, '02-routine'), ...]) {
      setState(() => _tabIndex = index);
      await Future.delayed(const Duration(seconds: 2));
      _writeSentinel('$langCode/$name'); // e.g. "en/01-home", "de/01-home"
      await Future.delayed(const Duration(seconds: 3));
    }

    // Detail/overlay screens...
  }
  _writeDone();
}
```

**Sentinel naming convention:** `<locale>/<screen>` — e.g., `en/01-home.ready`, `de/01-home.ready`.

**Shell script adaptation for multi-locale:**

```bash
LOCALES=(en de)
SCREENS=("01-home" "02-routine" "03-library" "04-progress")

for L in "${LOCALES[@]}"; do
    mkdir -p "$ABS_OUT_DIR/$L"
    for SCREEN in "${SCREENS[@]}"; do
        while [ ! -f "$SENTINEL_DIR/$L/$SCREEN.ready" ]; do sleep 0.5; done
        sleep 0.5
        xcrun simctl io "$DEVICE_ID" screenshot "$ABS_OUT_DIR/$L/$SCREEN.png"
        echo "    ✓ $L/$SCREEN"
    done
done
```

**Locale discovery:** Auto-detect from the app's `supportedLocales`:

```bash
grep -oP "Locale\('([a-z]{2})'\)" lib/main.dart | grep -oP "'[a-z]{2}'" | tr -d "'"
```

Or from `.arb` files:

```bash
ls lib/l10n/app_*.arb | sed 's/.*app_\(.*\)\.arb/\1/'
```

### Step 5: Verify iteratively (same as SwiftUI)

Run the script, read the PNGs visually, fix issues, re-run.

### Step 6: Evaluate and shortlist (same as SwiftUI)

Same criteria and process.

### Step 7: Generate App Store layouts (same as SwiftUI)

Same AppLaunchFlow MCP workflow.

## Flutter Known Gotchas

These are all real bugs encountered during Flutter screenshot capture.

### F1. Firestore streams overwrite seeded data

If the state management class sets up Firestore stream listeners (`getSessionLogsStream().listen(...)`) that call methods like `updateLogsFromFirestore([])`, those listeners will fire even in "offline" mode and clear your seeded demo data with empty results.

Fix: The `offlineMode` flag must `return` from the constructor **before** any stream listeners are set up. The flag must prevent `_initialize()`, `_initializeUser()`, and any `authStateChanges.listen()` from running.

### F2. Provider type must match exactly

Screens use `context.watch<AppState>()` which requires a `Provider<AppState>`. If you create a `MockAppState` subclass, the type won't match and `Provider.of<AppState>()` will throw. Always use the **real class** with an offline flag, not a mock subclass.

### F3. Firebase must still be initialized

Even in offline mode, `Firebase.initializeApp()` must be called before creating service objects. `AuthService()` accesses `FirebaseAuth.instance` and `FirestoreService()` accesses `FirebaseFirestore.instance` at field-declaration time — these crash if Firebase isn't initialized, even if you never use them.

### F4. Stats caching prevents demo data from showing

If the state class uses a `_cachedStats` object (populated by an async `_preloadStats()` method), the cached stats will be null after offline construction and all stat getters will return 0. Call `refreshStats()` after seeding data, and ensure `_preloadStats()` doesn't depend on Firestore.

### F5. `flutter run` resets shell working directory

When running `flutter run` in the background from a shell script, the working directory may be reset when the process exits. Always use **absolute paths** for `simctl io screenshot` output. Relative paths silently write to the wrong directory.

### F6. Overlay screens vs Navigator.push for detail views

Using `Navigator.push` from the coordinator to show detail screens works but makes it harder to dismiss them reliably. The overlay approach (swap a widget in the coordinator's `build()` method) is more reliable:

```dart
if (_showingOverlay && _overlayScreen != null) {
  return _overlayScreen!;
}
return Scaffold(body: IndexedStack(...), bottomNavigationBar: ...);
```

This avoids Navigator stack management issues and gives the coordinator full control.

### F7. Animations need time to settle

Flutter entry animations (from `flutter_animate`, `AnimatedContainer`, etc.) need time to complete before capture. Use at least 2 seconds of settle time after switching tabs, and 8+ seconds for screens with countdowns or complex animations.

### F8. Locale switching in Flutter

Flutter handles locale via the `MaterialApp.locale` property, not via `simctl launch` arguments. Two approaches exist (see section 4f):

- **Approach A (`--dart-define`):** Pass `--dart-define=SCREENSHOT_LOCALE=de` at build time and read it with `String.fromEnvironment('SCREENSHOT_LOCALE')`. Each locale is a separate `flutter run`. No coordinator changes needed. Preferred for ≤ 3 locales.
- **Approach B (in-process loop):** The coordinator switches `appState.currentLocale` programmatically and captures all screens per locale in a single run. Better for 4+ locales to avoid repeated build overhead.

With either approach, verify that locale switching actually took effect by comparing file sizes between locales (see Verification Checklist). Some widgets cache text on first build — if using Approach B and seeing identical screenshots across locales, switch to Approach A.

## Output Layout

```
marketing/
    <locale>/           e.g. en, de, es, fr, ja
        01-home.png
        02-<screen>.png
        ...
        NN-<screen>.png
        app-store-shortlist/
            01-<selected-screen>.png
            02-<selected-screen>.png
            ...
        elements/
            card-<name>.png
            widget-<family>-<size>.png
            chart-<name>.png
```

Put `marketing/` in `.gitignore`. These are outputs, not source.

## Verification Checklist

Before declaring the capture pipeline done, verify:

- [ ] All locales produced N files (where N = screens + elements)
- [ ] File sizes differ between locales (confirms translations actually render — if `en/settings.png` and `de/settings.png` are byte-identical, locale switching didn't take effect)
- [ ] Read 2-3 screens visually for the primary locale and confirm they show the expected content
- [ ] Read the same screens for at least one other locale and confirm localized strings are present
- [ ] Read at least one widget render and one card render to verify backgrounds and corners look right (SwiftUI only)
- [ ] No screenshot shows a screen from a *different* step (the most common bug — an undismissed sheet from the previous step)
- [ ] No screenshot shows empty state where data should be populated (Flutter gotcha F1 — Firestore streams clearing seeded data)
- [ ] User name is not "Guest" or a default placeholder (use `overrideUserName` parameter)
- [ ] Stats, streaks, and cached values are populated (call `refreshStats()` after seeding — Flutter gotcha F4)
- [ ] A ranked shortlist of up to 10 primary-locale full-screen screenshots was produced with explicit reasons
- [ ] `marketing/<primaryLocale>/app-store-shortlist/` exists and contains only the curated full-screen shots in presentation order
- [ ] If multiple locales matter for delivery, corresponding shortlist folders were mirrored or any missing counterparts were called out explicitly
- [ ] If Step 7 was run: editor URL was presented to the user and layouts were generated successfully

## Templates

### SwiftUI

- `templates/MarketingCapture.swift.template` — skeleton of the capture file with step-based coordinator. Reference the body of this skill for the patterns to apply.
- `templates/capture-marketing.sh.template` — skeleton of the shell script. Replace the bundle ID, scheme name, and simulator name for each project.

### Flutter

- `templates/main_screenshots.dart.template` — standalone screenshot entry point with offline AppState, demo data seeder, and auto-navigation coordinator.
- `templates/capture-screenshots-flutter.sh.template` — shell script that runs `flutter run` in the background and captures via `simctl io` synchronized with sentinel files.
