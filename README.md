<p align="center">
  <img src="logo.png" alt="AppLaunchFlow" width="120" />
</p>

<h1 align="center">AppLaunchFlow Screenshot Capture Skill</h1>

<p align="center">
  AI agent skill for automated App Store screenshot capture — forked from <a href="https://github.com/ParthJadhav/ios-marketing-capture">ParthJadhav/ios-marketing-capture</a> and adapted for the <a href="https://applaunchflow.com">AppLaunchFlow</a> pipeline.
</p>

---

## What changed from the original

This fork extends the original iOS Marketing Capture skill with:

- **Flutter support** — full screenshot capture path for Flutter apps using offline mode, `simctl io` capture, and in-process locale switching
- **Auto-detection over interviews** — the skill now infers screens, locales, devices, seed data, and appearance from the codebase instead of asking the user a long questionnaire
- **App Store candidate evaluation** — after capture, the skill evaluates screenshots and preselects up to 10 store-ready candidates based on visual strength, feature relevance, and composability
- **AppLaunchFlow MCP handoff** — curated screenshots are automatically handed off to the [AppLaunchFlow MCP](https://github.com/ynnickw/applaunchflow-mcp) for template-based layout generation (device mockups, headlines, backgrounds)
- **8 Flutter-specific gotchas** documented from real projects (Firestore overwriting seed data, Provider type mismatches, animation settle times, etc.)

Everything from the original SwiftUI path is preserved — the 11 SwiftUI gotchas, step-based coordinator, element rendering, and locale looping all work as before.

## What it does

- Builds an in-app capture system for **SwiftUI** and **Flutter** iOS apps — zero production footprint
- Seeds deterministic demo data so every screenshot looks populated and polished
- Navigates to each screen programmatically and snapshots including status bar and safe area
- Renders isolated elements (cards, widgets, charts) with transparency
- Loops every locale automatically — one build, all languages
- Evaluates results and preselects the strongest shots for App Store use
- Hands off to AppLaunchFlow MCP for layout generation (optional)

## Install

### Using npx skills (recommended)

```bash
npx skills add ynnickw/applaunchflow-skill
```

This works with Claude Code, Cursor, Windsurf, OpenCode, Codex, and [40+ other agents](https://github.com/vercel-labs/skills#available-agents).

Install globally (available across all projects):

```bash
npx skills add ynnickw/applaunchflow-skill -g
```

Install for a specific agent:

```bash
npx skills add ynnickw/applaunchflow-skill -a claude-code
```

### Manual (git clone)

```bash
git clone https://github.com/ynnickw/applaunchflow-skill ~/.claude/skills/applaunchflow-skill
```

## Usage

Once installed, the skill triggers automatically when you ask your agent to:

- Capture marketing screenshots for my iOS app
- Generate locale screenshots across all languages
- Render my widgets as isolated PNGs
- Automate App Store screenshot capture

Or just tell the agent what you need:

```
> Capture marketing screenshots for my app across all locales
```

The agent explores your codebase, detects screens/locales/devices automatically, and proceeds. It only asks when something is genuinely ambiguous.

## Example prompts

### Breathwork app (Flutter)

```text
Capture marketing screenshots for my Flutter breathing app.
I want all 4 tabs: Home, Routine, Library, and Progress.
Also capture the technique detail screen and the breathing session player mid-session.
English and German, iPhone 17 simulator.
```

### Coffee app (SwiftUI)

```text
Capture marketing screenshots for my coffee tracking app.
I want Home, Shelf, Coffee Detail, Brew Timer, and Settings.
Also render coffee cards and all my widgets as isolated elements.
Capture across en, de, es, fr, ja.
```

### Minimal prompt

```text
Capture marketing screenshots for my app
```

The skill will auto-detect everything and proceed.

## Output

```
marketing/
    en/
        01-home.png
        02-detail.png
        03-settings.png
        elements/
            card-morning-blend.png
            widget-pulse-small.png
    de/
        ...
```

After capture, the skill evaluates all full-screen shots and produces a shortlist of up to 10 App-Store-ready candidates. If the AppLaunchFlow MCP is configured, it uploads and generates layouts automatically.

## Requirements

### SwiftUI
- Xcode 16+
- iOS 17+ deployment target
- A simulator runtime matching your target iOS version
- Python 3

### Flutter
- Flutter SDK 3.10+
- Xcode (for iOS simulator)
- A booted iOS simulator
- Python 3

## Credits

Forked from [ParthJadhav/ios-marketing-capture](https://github.com/ParthJadhav/ios-marketing-capture), originally built for [Bloom](https://apps.apple.com/us/app/bloom-coffee-shelf-recipe/id6759914524).

## License

MIT
