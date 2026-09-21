# figma-to-swiftui

Skills that teach your coding agent to turn an iOS Figma design into **real SwiftUI** — and to
keep the two in sync afterwards.

Not a plugin, not a service, not an account. Three markdown files your agent reads.

```
/plugin marketplace add uladluch/figma-to-swiftui
/plugin install figma-to-swiftui@figma-to-swiftui
```

Works with any agent that can reach your Figma file and your code. In Claude Code that means
the Figma MCP server plus your repository — nothing else to install.

---

## Why this exists

Design-to-code normally guesses, because its input is anonymous geometry. It sees a 44pt bar
with centred text and writes an `HStack`. That looks right and behaves wrong: no large-title
collapse, no Liquid Glass, no back-swipe, no Dynamic Type — and it breaks on the next OS
release.

These skills replace guessing with two things: an **audit** of what a frame actually contains,
and the rule that the agent **reads each component's own description and obeys it**.

They get exact when the design carries its mapping. Every component in
**[iOS 27 Builder](https://www.figma.com/community/file/922533165060687529)** — free, 121k
users, no plugin — names its SwiftUI or UIKit API in its own description, checked against the
iOS 27 SDK on a live simulator. The iOS 26 edition stays published separately for apps still
shipping against 26. With those present the agent stops inferring and starts
translating.

---

## What's inside

| skill | direction | what it knows |
|---|---|---|
| **design-tokens** | both | which colours iOS owns and must never be copied as values |
| **figma-to-swiftui** | design → code | chrome is modifiers, not children; what not to build at all |
| **swiftui-to-figma** | code → design | the Plugin API writes that look successful and did nothing |
| **figma-layer-roles** | design → code | the fixed role vocabulary an agent can actually match on |

---

## The one idea worth reading first

Most tokens in an iOS design system are **not values**. They are names for colours the system
supplies.

In iOS 26 `systemBlue` moved to `#0088FF` while `link` stayed `#007AFF`, after years of being
identical. iOS 27 moved `opaqueSeparator` the same way. Every app that had exported "its blue" into an asset catalog silently stopped
matching the system.

So a token is sorted before it is converted:

| class | in code | syncs back |
|---|---|---|
| **sdk** | `Color.primary`, `Color(uiColor: .systemGray3)` | no — regenerated |
| **vibrancy** | a `UIVibrancyEffect`, not a colour at all | no |
| **recipe** | one layer of a material — `.glassEffect()`, `Material` | no |
| **own** | a colorset with light and dark | **yes** |

In the iOS 27 Builder kit that split is **50 / 7 / 17 / 15** across 89 colours — counted by
running the rules below over the live file, not from memory. Only fifteen belong to the app;
exporting the other seventy-four as values is the bug this avoids. (The recipe count halved
when the kit moved to iOS 27 and its twenty Liquid Glass variables were folded into the
primitives, which is why a number like this is worth re-counting rather than quoting.)

Typography follows the same logic: `Font.largeTitle`, never `Font.system(size: 34)`. The number
in Figma is what the system draws at the default text size — hard-coding it breaks Dynamic Type
for every reader who changed theirs.

---

## Verified, not asserted

Every claim here was measured rather than remembered:

- the generated Swift **compiles** — `swiftc -typecheck` against the iOS 27.0 SDK, zero errors
- the asset catalog **compiles** — `actool`, zero errors, dark appearances present
- an app built on the generated tokens **runs on the simulator**, and its background measures
  `#f2f2f7` in light and `#000000` in dark, because the system supplies the value

The Plugin API traps in `swiftui-to-figma` are not folklore. Each one cost an afternoon: a
nested instance that accepts `resize()` and reverts it, a `setProperties` that invalidates every
handle you collected, a paint binding that keeps the colour you started from, and 887 frames
that installed opaque white because an absent property was left at its creation default.

---

## Using it

Open a screen in Figma, then ask your agent in plain language:

> Implement the selected frame as SwiftUI.

> Generate the design system into `DesignSystem/` and the asset catalog.

> I changed `OverlaysDefault` in Xcode — put it back into Figma.

The skills load themselves when the task matches. You do not invoke them by name.

---

## Licence

Apache 2.0. Use it commercially, fork it, ship it — keep the attribution.

The kit it is built around is free on
[Figma Community](https://www.figma.com/community/file/922533165060687529).
