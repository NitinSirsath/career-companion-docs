# Career Companion Design System

This document outlines the IBM-inspired frontend design foundation for Career Companion.

## Core Philosophy
- **Precise, Calm, Information-Rich:** The UI should feel like a professional operational tool, not a generic SaaS dashboard.
- **Square Geometry:** `0px` border radius everywhere.
- **Restrained Color:** Use semantic tokens rather than arbitrary HEX values.
- **Hairlines over Shadows:** Hierarchy is achieved through borders (`hairline`, `strong-hairline`) and surface colors (`surface-1`, `surface-2`), avoiding heavy box-shadows.

## Typography
- **Primary Font:** IBM Plex Sans (`var(--font-sans)`)
- We rely on native font weights and tracking to establish hierarchy rather than excessive size differences.

## Tokens (Tailwind)

### Surfaces
- `bg-background`: The canvas (white).
- `bg-surface-1`: Primary elevated surface (cards, sidebars).
- `bg-surface-2`: Secondary surface (hovers, secondary sections).

### Borders
- `border-border`: Standard hairline.
- `border-border-strong`: Stronger emphasis hairline.

### Text
- `text-foreground`: Primary ink.
- `text-muted-foreground`: Secondary ink.

### Action Colors
- `primary`: `#0f62fe` (IBM Blue)
- `success`: `#24a148`
- `warning`: `#f1c21b`
- `destructive`: `#da1e28`
- `info`: `#0f62fe`

## Components
- Reusable primitives are located in `src/components/ui/`.
- Use `Button` variants (`primary`, `secondary`, `tertiary`, `ghost`, `danger`) instead of building custom buttons.
- Use `Badge` for status indication.

## Layout
- **Desktop:** Persistent left sidebar. Content area constrained and centered where appropriate.
- **Mobile:** Bottom navigation bar. Forms and lists should transform to vertical stacks. Do NOT squeeze desktop sidebars onto mobile screens.
