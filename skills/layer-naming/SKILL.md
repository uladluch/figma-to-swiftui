---
name: layer-naming
description: Name the layers inside a Figma component so an agent reading the file writes the right code. Use when building, renaming or restructuring layers in a Figma iOS design, when generated code comes back full of data-name="Frame 427", and as the last check before calling a component done.
---

# Layer naming

Layer names are machine input. `get_design_context` emits every one of them as `data-name="…"` in
the code your agent receives, so `Frame 2085662666` is not untidy — it is a line of noise inside
the thing you are asking the agent to translate. Twenty of those in one screen and the agent has
no structure to align against, so it falls back to guessing from geometry.

But names are **not** the mapping channel. Component descriptions carry the API; a layer name
carries the **role**. Never write `.toolbar` or `ToolbarItem(placement:)` as a layer name — a layer
is not a call, and the moment the API changes the file lies. Name the role, using the word the SDK
uses for that role.

This pairs with `figma-to-swiftui`: that skill reads structure out of a frame, and this is what
makes the structure readable.

## What must be named

**Tier A — always.** Anything structural: containers, slots, text, images, backgrounds, controls.
These reach the generated code and the layers panel.

**Tier B — numbered siblings.** Repeated slots take `Role N`, one-based, space before the digit:
`Tab 1`, `Dot 3`, `Action 2`. Never zero-based, never `Role1`.

**Tier C — leave alone, deliberately.** Vector geometry *inside* artwork: a `VECTOR`, `ELLIPSE` or
`BOOLEAN_OPERATION` leaf that draws part of an icon. Nobody reads them and renaming them is churn
with no reader — a single icon can hold dozens. A group that *contains* geometry is Tier A: it is
the icon, and it gets the icon name.

## The vocabulary

One word per role. The left column is the only accepted form. The point of fixing it is that an
agent can match on it; three synonyms for one role is the same as no name at all.

| Role | Use | Not |
|---|---|---|
| The surface behind everything | `BG` | Background, Backdrop, Fill |
| Text of a control or button | `Label` | Text, Caption |
| Primary text of a row or card | `Title` | Heading, Header, Name |
| Secondary text under the title | `Subtitle` | Description, Secondary |
| Body copy under a title | `Message` | Description, Body |
| Trailing value on a row | `Detail` | Value, Accessory Text |
| A symbol or glyph | `Icon` | Symbol, Glyph |
| Photographic or artwork content | `Image` | Picture, Photo, Thumbnail |
| The action group | `Buttons` | Actions, Controls, CTA |
| The main content stack | `Content` | Stack, Contents, Container, Wrapper |
| The text block above the actions | `Title and Description` | Copy, Text Block |
| A hairline rule | `Separator` | Divider, Line, Rule |
| Edge slots | `Leading`, `Center`, `Trailing` | Left, Right, Start, End |
| The SF Symbol glyph itself | `Symbol` | Glyph, SF Symbol |
| The typed content of a field | `Value` | Input, Entry |
| A group of chart text | `Labels` | Text, Captions |
| A line in a chart | `Gridline` | Divider, Separator |
| A radial chart segment | `Ring` | Oval, Arc |

Three words look interchangeable and are not. `Icon` is the **slot** — the box a glyph sits in.
`Symbol` is the **SF Symbol text node inside it**, the one holding a private-use character like
`􀋂`. `Image` is artwork that takes no tint. The distinction survives into code as
`Image(systemName:)` versus a bitmap, so keep it honest — collapse `Symbol` into `Icon` and the
layer the agent must recognise as a symbol disappears.

`Divider` is banned outright because it is ambiguous: between rows it is a `Separator`
(`UIColor.separator`), inside a chart it is a `Gridline`. The kit this skill came out of once
carried 108 layers called `Divider`, and every single one was a chart gridline.

Edge words follow the SDK: SwiftUI says `leading` and `trailing`, never left and right.

## Casing and shape

- **Title Case** for layer names: `Search Field`, `Fill + Shadow`.
- Names describe **what the layer is**, not what it looks like: `Separator`, not `Grey Line`.
- No type names as names. `Rectangle`, `Frame`, `Group`, `Vector`, `Oval`, `Path` are Figma types,
  not roles — a Tier A layer carrying one of those is unnamed by definition.
- No trailing numbers unless the layer genuinely repeats among siblings.
- A leading `_` marks a private atom — something assembled into other components rather than
  placed directly on a screen. Useful convention, and iOS 26 Builder uses it throughout, so
  keep it if you are extending that file.

## Variant axes

Axis and value both in **Title Case**: `Type=Grid`, `Selected=True`, `Biometry=Face ID`. Lowercase
axes (`type=grid, selected=false`) are wrong and travel straight into generated prop names.

Axis names say what changes, in the SDK's words where one exists: `Role` (maps to `ButtonRole`),
`State`, `Style`, `Size`, `Type`, `Kind`, `Flow`. Never `Property 1` — that is what Figma leaves
behind after recombining a set; restore it with
`componentSet.editComponentProperty(old, { name: new })`.

## Auditing a component before you call it done

Run this against the component, not against the page — it reports Tier A layers still wearing a
Figma type name, while exempting the geometry leaves that are meant to keep theirs.

```js
const JUNK = /^(Frame|Group|Rectangle|Ellipse|Vector|Line|Union|Subtract|Oval|Path|Image)\s*\d*$/i;
const VECTORY = new Set(['VECTOR','ELLIPSE','LINE','POLYGON','STAR','BOOLEAN_OPERATION']);
const bad = [];
for (const n of comp.findAll(() => true)) {
  const isGeometryLeaf = VECTORY.has(n.type) && (!n.children || !n.children.length);
  if (JUNK.test(n.name) && !isGeometryLeaf) bad.push({ type: n.type, id: n.id, name: n.name });
}
return bad;
```

**Renaming a layer never breaks anything** — component properties bind by node id, not by name.
The one thing to preserve is `componentPropertyReferences`: read it before restructuring a
subtree and put it back afterwards, or text overrides silently reset the next time someone
switches a variant.

## Why not name layers after the SDK

Tempting, and wrong. A layer named `.alert(isPresented:)` claims the layer *is* the call. It is
not, and the claim rots the first time Apple renames something. The component description states
the API once, precisely, and can be checked against the SDK. The layer name only has to answer
“what part is this?”, so the agent can align the structure it sees against the API the description
names.
