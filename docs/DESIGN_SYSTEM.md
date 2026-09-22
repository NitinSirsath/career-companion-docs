# Career Companion Design System

This document outlines the IBM-inspired frontend design foundation for Career Companion.

## Core Philosophy
- **Precise, Calm, Information-Rich:** The UI should feel like a professional operational tool, not a generic SaaS dashboard.
- **Square Geometry:** `0px` border radius everywhere.
- **Restrained Color:** Components consume semantic design tokens. Themes define semantic token values. Never hardcode colors.
- **Hairlines over Shadows:** Hierarchy is achieved through semantic borders (`border-default`, `border-strong`) and surface colors (`surface`, `surface-subtle`), avoiding heavy box-shadows.

## Themes
The application supports a multi-theme architecture:
- **Light:** Primary IBM-inspired light theme.
- **Dark:** IBM-inspired dark theme.
- **Grey:** Neutral grey-oriented theme.
- **GitHub:** GitHub-inspired color theme.
- **Monokai:** Monokai-inspired color theme.

Theme switching dynamically updates CSS variables defined in `.theme-*` root classes. 

## Typography
- **Primary Font:** IBM Plex Sans (`var(--font-sans)`)
- We rely on native font weights and tracking to establish hierarchy rather than excessive size differences.

## Tokens (Tailwind v4 Semantic Variables)

### Surfaces
- `bg-surface`: The primary canvas.
- `bg-surface-subtle`: Secondary canvas (hovers, backgrounds).
- `bg-surface-raised`: Elevated surface (cards, modals).
- `bg-surface-selected`: Visually distinct active indicator for selected navigation items.
- `bg-surface-disabled`: For explicitly disabled states.

### Borders
- `border-border-subtle`: Very subtle dividing line.
- `border-border-default`: Standard hairline.
- `border-border-strong`: Stronger emphasis hairline.
- `border-border-focus`: Primary interaction indicator.

### Text
- `text-text-primary`: Primary ink.
- `text-text-secondary`: Secondary ink.
- `text-text-tertiary`: Muted ink.
- `text-text-disabled`: Non-interactive ink.
- `text-text-inverse`: Contrast ink against primary action backgrounds.

### Action Colors & States
- `bg-action-primary` & `hover:bg-action-primary-hover`: Main calls to action.
- `bg-action-secondary` & `hover:bg-action-secondary-hover`: Secondary actions.
- `bg-action-danger` & `hover:bg-action-danger-hover`: Destructive actions.

### Status Semantic Colors
- `bg-status-success-subtle text-status-success`: e.g. OFFERS, COMPLETE
- `bg-status-warning-subtle text-status-warning`: e.g. PENDING, ASSESSMENT
- `bg-status-error-subtle text-status-error`: e.g. REJECTED, DISMISSED
- `bg-status-info-subtle text-status-info`: e.g. RECRUITER CONTACT
- `bg-status-neutral-subtle text-status-neutral`: e.g. INTERVIEW, DEFAULT

## Components
- Reusable primitives are located in `src/components/ui/`.
- Use `Button` variants (`primary`, `secondary`, `tertiary`, `ghost`, `danger`) instead of building custom buttons.
- Use `Badge` for status indication mapped semantically.

## Layout
- **Desktop:** Persistent left sidebar. Content area constrained and centered where appropriate.
- **Mobile:** Bottom navigation bar. Forms and lists should transform to vertical stacks. Do NOT squeeze desktop sidebars onto mobile screens.
