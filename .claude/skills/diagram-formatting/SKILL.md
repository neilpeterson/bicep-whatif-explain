---
name: diagram-formatting
description: Ensures ASCII/box-drawing diagrams follow consistent formatting standards.
---

When creating or editing ASCII diagrams (architecture diagrams, flowcharts, component layouts), apply these formatting standards.

## When to Use This Skill

- Creating architecture diagrams in markdown documents
- Editing existing ASCII diagrams
- Reviewing specifications that contain box-drawing diagrams
- Any time box-drawing characters (─, │, ┌, ┐, └, ┘, ├, ┤, ┬, ┴, ┼) are used

## Formatting Standards

### 1. Border Alignment

All vertical borders must align consistently. The right edge of nested boxes should have consistent spacing to the outer border.

**❌ Incorrect** - Inconsistent spacing before right border:
```
┌─────────────────────────────────────────────┐
│                  Layer 1                    │
│  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Box A     │  │       Box B         │  │
│  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────┤
│                  Layer 2                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐  │
│  │  Box C  │  │  Box D  │  │    Box E    │  │
│  └─────────┘  └─────────┘  └─────────────┘  │
└─────────────────────────────────────────────┘
```
Note: Layer 1 has 1 space before `│`, Layer 2 has 2 spaces before `│`

**✅ Correct** - Consistent spacing (2 spaces) before right border:
```
┌───────────────────────────────────────────┐
│                 Layer 1                   │
│  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Box A     │  │       Box B         │ │
│  └─────────────┘  └─────────────────────┘ │
├───────────────────────────────────────────┤
│                 Layer 2                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────────┐  │
│  │  Box C  │ │  Box D  │ │    Box E    │  │
│  └─────────┘ └─────────┘ └─────────────┘  │
└───────────────────────────────────────────┘
```

### 2. Consistent Inner Padding

Use consistent padding (typically 2 spaces) between the outer border and nested content on both left and right sides.

```
│  ┌─────────┐  ┌─────────┐  │
   ^^                     ^^
   2 spaces on each side
```

### 3. Horizontal Divider Alignment

Horizontal dividers (`├` and `┤`) must span the full width and align with the outer corners.

**❌ Incorrect:**
```
┌──────────────────────┐
│       Section 1      │
├─────────────────────┤
│       Section 2      │
└──────────────────────┘
```

**✅ Correct:**
```
┌──────────────────────┐
│       Section 1      │
├──────────────────────┤
│       Section 2      │
└──────────────────────┘
```

## Verification Checklist

When reviewing a diagram, verify:

- [ ] All rows have the same total character width
- [ ] Left padding is consistent (typically 2 spaces after `│`)
- [ ] Right padding is consistent (typically 2 spaces before `│`)
- [ ] Horizontal lines (`─`) connect properly to corners and junctions
- [ ] Nested boxes maintain internal alignment
- [ ] Text is centered or consistently aligned within boxes

## Future Standards

_Additional diagram formatting rules will be added here._