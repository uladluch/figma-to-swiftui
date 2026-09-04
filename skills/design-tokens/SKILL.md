---
name: design-tokens
description: Turn Figma colour, type and spacing variables into Swift and an asset catalog, and sync edits back. Use when setting up a design system in an Xcode project, generating Colors.swift or Assets.xcassets from Figma, or when a colour changed on one side and should reach the other. Knows which tokens iOS owns and must never be copied as values.
---

# Design tokens, both directions

The obvious move is to export every Figma colour into `Assets.xcassets` and reference it from
Swift. It produces an app that looks right today and drifts from the system on the next iOS
release.

That is not hypothetical. **In iOS 26 `systemBlue` moved to `#0088FF` while `link` stayed
`#007AFF`** after years of being the same value. Every app carrying its own copy of "the blue"
silently stopped matching the system.

So the first job is not conversion. It is deciding, per token, **who owns the value**.

## Four classes, three different fates

| class | what it is | in code | syncs back |
|---|---|---|---|
| **sdk** | a colour iOS supplies | `Color.primary`, `Color(uiColor: .systemGray3)` | no - regenerated |
| **vibrancy** | a `UIVibrancyEffectStyle` | not a colour at all | no |
| **recipe** | one layer of a material | nothing | no |
| **own** | this design system's | a colorset with light and dark | **yes** |

**sdk** tokens are emitted as *references*. The system supplies light and dark and keeps them
correct across releases. Writing the hex instead freezes today's value into the app.

**vibrancy** tokens name `UIVibrancyEffect(blurEffect:style:)`, applied inside a
`UIVisualEffectView`. SwiftUI has no equivalent. The hex in Figma is an approximation of an
effect - emitting it as a colour is wrong twice over.

**recipe** tokens are one layer of a stacked material: Liquid Glass, a system blur. In code
these are `.glassEffect()` or `Material`, never a fill. A recipe colour that reaches an asset
catalog is a glass layer masquerading as a brand colour.

**own** tokens are the only ones an app actually owns, and therefore the only ones where
"changed in Xcode" and "changed in Figma" are both meaningful.

## Classifying without a table

Do not keep a list of token names - it drifts the moment one is renamed. **Read the
description.** A well-built kit states the mapping on the variable itself:

```
Labels/Primary      "SwiftUI Color.primary · UIKit UIColor.label — text that contains primary content."
Overlays/Default    "Design-system convention, not a public SDK colour. Dimming behind modal content."
Labels - Vibrant/…  "UIVibrancyEffectStyle.label — applied with UIVibrancyEffect(blurEffect:style:)"
```

The rules, in order:

1. description says **"not a public SDK colour"** → `own`
2. description names **`UIVibrancyEffectStyle`** → `vibrancy`
3. the variable lives in a collection **hidden from publishing** → `recipe`
4. description names **`SwiftUI Color.X`** or **`UIKit UIColor.Y`** → `sdk`, and that symbol is
   the expression to emit
5. otherwise → `own`

Rule 3 is easy to miss and expensive: a kit keeps its material recipes in a private collection
precisely because they are not palette. Without it, 37 glass layers arrive as app colours.

If a description carries a qualifier the expression cannot hold, **say so in a doc comment
rather than dropping it**. `UIColor.systemBackground resolved at UIUserInterfaceLevel.elevated`
emits the plain colour - iOS raises the level itself inside presented content - but silently
producing the same expression for two different tokens is how a dark-mode difference disappears.

## Typography maps to Font cases, never to numbers

```swift
static let largeTitle = Font.largeTitle      // right
static let largeTitle = Font.system(size: 34) // breaks Dynamic Type for everyone who changed it
```

The ramp is `.largeTitle .title .title2 .title3 .headline .body .callout .subheadline
.footnote .caption .caption2`. Emit the case. The point size in Figma is what the system
happens to draw at the default text size; it is not the value.

Spacing is the opposite: SwiftUI ships no spacing scale, so a spacing scale belongs to the
project and round-trips like any `own` token.

## What to generate

```
DesignSystem/Colors.swift        sdk references + own colorset accessors
DesignSystem/Typography.swift    the ramp as Font cases
DesignSystem/Spacing.swift       plain CGFloat constants
Assets.xcassets/<Name>.colorset/Contents.json   own tokens only, light + dark appearances
DesignSystem/kit-tokens.json     the manifest that makes the round trip addressable
```

A colorset carries both appearances in one file: the light entry has no `appearances` key, the
dark one carries `[{"appearance": "luminosity", "value": "dark"}]`. Components are sRGB floats
as strings.

**The manifest is what makes code-to-design possible.** Without it a changed colorset is just a
file, and working out which Figma variable it came from is guesswork. With it:

```
Assets.xcassets/OverlaysDefault.colorset  →  Overlays/Default  →  a variable id
```

## Checks worth running

Two failures are silent and both are cheap to catch:

- **Two tokens emitting the same Swift expression.** Legitimate sometimes, a dropped qualifier
  otherwise. Report them; do not merge them.
- **Two tokens producing the same Swift identifier.** That does not compile. Fail loudly.

Then compile. `swiftc -typecheck` against the simulator SDK and `actool` on the catalog take
seconds and are the only proof that any of this is real.

## Code back to design

A colour edited in `Assets.xcassets` belongs to the **design system**, so it is written to the
Figma **variable**, not to a node. Writing it onto one node leaves every other use behind.

Compare against a baseline recorded when the code was generated, never against the file event
itself. Without a baseline the first save reports every token as changed; with one, a save that
changes nothing reports nothing.

Set the value for **one mode**. A colorset carries light and dark in a single file, and a naive
write clobbers the appearance you did not touch.
