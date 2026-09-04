---
name: figma-to-swiftui
description: Implement an iOS screen from Figma as real SwiftUI - native toolbars, TabView, List and alerts instead of stacks that only look right. Audits the frame first and reports which system chrome was drawn by hand. Use when building an iPhone or iPad screen from a Figma design.
---

# A Figma screen as real SwiftUI

Design-to-code normally guesses. It sees a 44pt bar with centred text and writes an `HStack`,
which looks right and behaves wrong: no large-title collapse, no Liquid Glass, no back-swipe,
no Dynamic Type, and it breaks on the next OS release.

Replace guessing with two things: an **audit** of what the frame actually contains, and a rule
that you **read each component's own description and obey it** rather than infer an API from
pixels.

## Before writing any code

Call `get_design_context` on the selection. You need the layout *and* the **description on
every instance**.

Sort what you find into three piles:

1. native chrome backed by a library component
2. native chrome **drawn by hand**
3. genuinely custom content

**If pile 2 is not empty, stop and say so.** Code generated from hand-drawn chrome is wrong
before the first line: a hand-built tab bar becomes an `HStack` of buttons, and the app loses
the real `TabView` along with everything the system does for free.

## Chrome is modifiers, not children

This is the mistake that survives review because the render looks fine. In SwiftUI a navigation
bar is not a view inside the screen - it is a modifier on it.

```swift
NavigationStack {
    List { … }
        .navigationTitle("Folder")
        .toolbarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItemGroup(placement: .topBarTrailing) { … }
        }
        .searchable(text: $query)
}
```

The same holds for the tab bar (`TabView` wraps the screens), the scroll edge effect
(`.scrollEdgeEffectStyle(_:for:)`), and the sheet (`.sheet` on the presenter, not a view in the
tree).

A run of sibling row components is a `List`, not a `VStack` of rows. That single decision buys
separators, swipe actions, selection and the right insets.

## What not to generate at all

iOS draws these. An app that rebuilds them is fighting the system:

| in the design | in code |
|---|---|
| keyboard | `.keyboardType(_:)` on the field |
| page dots | `.tabViewStyle(.page)` |
| dimming behind a modal | nothing - `.sheet` and `.alert` draw it |
| list separators | nothing - `List` draws them |
| status bar | nothing |
| the glass capsule under a tab accessory | nothing - the system draws it |

If the design shows one, that is the designer showing you what the user will see, not asking
you to build it.

## Read the description, do not infer the API

A kit that names its own API is the difference between a guess and a translation. Every
component in [iOS 26 Builder](https://www.figma.com/community/file/922533165060687529) carries
its mapping in its description, checked against the iOS 26 SDK on a live simulator.

```
Toolbar - Top      .toolbar { … } · .toolbarTitleDisplayMode(.inlineLarge)
Tab Bar            TabView with Tab(…); Tab(role: .search) spends one of the five slots
Row                a row in List; the trailing chevron means NavigationLink, not Button
```

Two rules that save real time:

- **A name agreeing is where the check starts, not where it ends.** A layer called
  `face id sheet` is not necessarily the LocalAuthentication alert. Confirm against the
  component's axes and description before reusing it.
- **Verify an API before asserting it.** Apple's own sample code contains errors: a snippet
  writes `.minimized` where the type declares `.minimize`. Prefer the type reference, or grep
  the simulator SDK headers.

## Type and colour

Never emit a point size from the design. `Font.largeTitle`, not `Font.system(size: 34)` - the
number in Figma is what the system draws at the default text size, and hard-coding it breaks
Dynamic Type for everyone who changed theirs.

Never emit a hex for a colour the system owns. `Color.primary`, not `#000000`. See the
`design-tokens` skill for how to tell which is which - it matters more than it looks, because
in iOS 26 `systemBlue` moved and `link` did not.

## Finish honestly

State what you could not translate rather than inventing a view for it. A screen that says
*"the segmented control here is custom, I emitted a placeholder"* is worth more than one that
silently ships an `HStack` pretending to be a `Picker`.
