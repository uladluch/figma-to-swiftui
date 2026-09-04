---
name: swiftui-to-figma
description: Write changes from Swift code back into a Figma file - a tint, a string, a spacing, a variant. Use when code and design have diverged and the design should follow, or when building a mockup from an existing app. Covers the Figma Plugin API traps that make a write look successful when it did nothing.
---

# Code back into the design

The direction everyone skips, and the one that decides whether this is a generator or a sync.

## The rule that keeps it honest

**Move only what both sides can represent.**

A tint, a string, a spacing, a variant: both sides have these, so they move. An `.onAppear`, a
custom chart, business logic: the design cannot express them, so they are **never touched**.

Get this wrong and the tool regenerates a whole screen from the design, wiping the hand-written
code around it. That is why most design-sync tools are abandoned after the first time they eat
someone's work.

## A token change addresses a variable, not a node

A colour edited in code belongs to the design system. Write it to the Figma **variable** and
every use follows. Writing it onto the one node you happened to find leaves the rest behind and
quietly splits the system in two.

Set the value for **one mode**. A colorset carries light and dark in a single file; a naive
write clobbers the appearance you did not touch.

Compare against a **baseline recorded when the code was generated**, not against a file event.
Without a baseline, the first save reports every token as changed. Editors also write, rename
and touch a file for one save, so debounce and compare content, never react to the event.

## The Plugin API accepts writes it does not perform

This is the expensive part, and none of it throws.

**Write, re-read by id, assert.** A write that cannot be proved did not happen.

**A nested instance silently refuses `resize`.** The call succeeds, throws nothing, and the
value reverts - the size belongs to the outer master. `swapComponent` and `resetOverrides` do
not help. Restructure instead: lift the node out as a sibling, or swap to a variant that is
already the size you need. Reach for restructuring the moment a resize reads back unchanged,
rather than trying a fourth variation of the same call.

**`setProperties` invalidates every handle collected earlier**, including siblings you have not
reached yet. A `findAll` followed by a loop that mutates instances silently skips half of them
and reports success. Re-query for the next match on every iteration; never hold an array across
a mutation.

**`setBoundVariableForPaint` keeps the colour the paint started from and drops alpha.** Binding
a blue variable onto a paint built as black stores black *with a blue binding*, and renders
black. Read the variable's own value, build the paint on it, then bind. A token that carries
alpha imposes its own; a token without alpha leaves the paint's alone.

**A paint bound to a variable cannot keep a blend mode** - Figma forces `NORMAL`, and the
returned object still reports the old value. The node's own `blendMode` is a separate property
and survives, so a single-fill vibrancy layer is fully tokenisable: bind the paint, move the
blend to the node.

**Switching `layoutMode` swaps which axis `counterAxisSizingMode` governs.** A fixed-width
horizontal stack silently becomes width-hug when it turns vertical. Re-assert the size straight
after, and re-check any child that was `FILL`.

**Order matters inside a container.** Append children, then give the parent its auto-layout,
then tell children to `FILL` or `HUG`. Figma rejects `FILL` on a child whose parent is not yet
an auto-layout frame. Auto-layout minimums (`minWidth`, `maxHeight`) are the same: they are
only accepted on a layout node or a child of one.

**An absent property must be reset, not left alone.** `figma.createFrame()` arrives with an
opaque white fill. A layout frame that has no background in the source will keep that white
unless you explicitly clear it - and every measurement will still say the layout is perfect.

## Say what you could not move

A change the design cannot express is not a failure, but silence about it is. Report it:
*"`.onAppear` added in `SettingsView.swift:42` has no design counterpart - left as is."*
