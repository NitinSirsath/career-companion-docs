# Career Companion Design System

Design part 1, built on 18 Oct 2026 from the plan the owner approved on 17 Oct 2026 and the approved mock: https://claude.ai/artifact/VWAgcutaXtATYMXhRXc4Mx. The rules every agent must follow are in [AI_UI_RULES.md](AI_UI_RULES.md).

The look is IBM Carbon, made for daily work: layers instead of borders, one type scale, short productive motion, Carbon icons. The goal is a mature, calm tool, not a showcase.

## Themes

Four choices: **System**, **Light** and **Dark** (owner decision, 17 Oct 2026), and **White** (owner decision, 18 Oct 2026). System follows the computer's setting (Light or Dark). Grey, GitHub and Monokai are gone.

| Theme | Carbon theme | Page | Card | Text |
| --- | --- | --- | --- | --- |
| Light | Gray 10 | `#f4f4f4` | `#ffffff` | `#161616` |
| White | White | `#ffffff` | `#f4f4f4` | `#161616` |
| Dark | Gray 100 | `#161616` | `#262626` | `#f4f4f4` |

The theme is chosen in the account menu and on the Account page. Every colour is a token in `src/index.css` (`:root, .theme-light`, `.theme-white` and `.theme-dark`; White only overrides what differs from Light); components use the Tailwind names of the tokens (`bg-surface`, `text-text-secondary`), never a hex value.

## Layers

- **Page** (`bg-background`), then **card** (`bg-surface`: one step lighter than the page in Light and Dark, one step darker in White), then **inner layer** (`bg-layer-accent`, for table headers, code and meters).
- Thin borders (`border-border-subtle`) only between rows and sections. No shadows. No rounded corners, except round status dots and avatars.
- The user's own mails sit on `bg-tint`, a light blue layer.

## Type

IBM Plex Sans and IBM Plex Mono, Carbon's productive scale:

| Use | Size / line height | Weight |
| --- | --- | --- |
| Page title (`h1`) | 28 / 36 | regular |
| Section title in a card (`h2`) | 20 / 28 | regular |
| Settings row title | 16 / 22 | semibold |
| Body | 14 / 20 | regular |
| Label, help and meta text | 12 / 16, letter spacing 0.32px | regular |

Sentence case everywhere ("Sync now", not "Sync Now").

## Width and spacing

- One width for every page: the full width up to **1584px**, centred beyond (`src/routes/__root.tsx`). Laptops use the whole width; margins only appear on very large screens.
- Spacing follows Carbon's 8px grid: 8, 16, 24, 32, 48. Cards use 24px side padding and 20px top and bottom.
- Phones: 16px side gutter, no sideways scroll; tables stack each row's cells.

## Page structure

- `PageHeader` (`components/layout/PageHeader.tsx`): breadcrumb on detail pages, one title, one line of description, actions on the right (at most one primary).
- `Breadcrumb`: "Mails / Lumen Apps". It leads back to the list as the user left it (the list's search is remembered).
- Detail pages: a main column and a 360px side column from `lg` up.
- Settings pages (`SettingsSection`): each section is a row, title and short help on the left third, the controls in a card on the right.

## Motion

Carbon's productive curves, as tokens: `ease-standard` (0.2, 0, 0.38, 0.9), `ease-entrance` (0, 0, 0.38, 0.9), `ease-exit` (0.2, 0, 1, 0.9).

| Time | Use |
| --- | --- |
| 70ms | hover and pressed colours |
| 110ms | dropdowns and menus close; tooltips open |
| 150ms | dropdowns and menus open: they fade and grow out of the side next to their field; a page fades in (`animate-page-in`) |
| 240ms | dialogs and toasts come and go; every change of size (below) |

**Size changes are never a jump** (owner, 21 Oct 2026; demo "Motion: today vs new" on https://claude.ai/artifact/3j4wfD4Mvme1oGQ8p5bCNv). Three shared parts do it, so a page never builds its own:

- `Collapse`: a section the user shows and hides (an older mail, the AI's reading, done steps). It grows from no height to its own and fades in, and leaves the same way.
- `Reveal`: something that appears by itself, such as an error under a field. It grows and fades in. `InlineNotification` already uses it.
- `AutoHeight`: a box that follows its content's height when the content jumps (new results, the next step). Every dialog body and the Combobox list already sit in one. A part inside that animates its own height is followed as it moves, so the two never fight.

Motion is short and quiet and never needed to understand the screen. With "reduce motion" on, it is off, and every part still ends at the right size.

**Tailwind 4 rule.** `translate-*`, `scale-*` and `rotate-*` set the CSS `translate`, `scale` and `rotate` properties, not `transform`. A transition must list the property it moves (`transition-[opacity,translate]`), or use `transition-transform`, which covers all four. A list with only `transform` makes the movement snap while only the fade runs.

## Icons

Carbon icons only (`@carbon/icons-react`), 16px in text and buttons, 20px in the menu. The AI is shown with Carbon's machine-learning model icon, never a sparkle.

## Shared parts (`src/components/ui/`)

| Part | Use it for |
| --- | --- |
| `Button` | primary (one per view), secondary, tertiary, ghost, danger, danger-ghost; `loading` shows a spinner |
| `Select` | a short fixed list; never the browser's own select |
| `Combobox` | a long list or one the server loads: type to search |
| `OverflowMenu` (`menu.tsx`) | the "…" menu for actions that are not the main one; danger actions last and red |
| `DataTable` | every list; sorting, filtering and paging stay on the server; skeleton rows while loading |
| `Toolbar` | search and filters above a table, in one row |
| `Tabs` | switching between views of one list |
| `Tag` | statuses and counts; colours from `lib/statusTags.ts` only |
| `InlineNotification` | a notice inside a page: info, success, warning, error |
| `SectionError` | a part of the page that did not load: plain words and "Try again", never the raw error |
| `EmptyState` | "nothing here": why, and the one next step |
| `Skeleton`, `SkeletonList` | loading |
| `Dialog` | a confirm or a short form; Carbon footer buttons; its body follows its content's height |
| `Collapse` | a section the user shows and hides; the toggle keeps `aria-expanded` |
| `Reveal` | something that appears by itself, such as a field error |
| `AutoHeight` | a box whose content jumps in height (results, steps) |
| `showToast` (`toast.tsx`) | a short confirmation after an action, with Undo where the app can really undo |
| `CodeSnippet` | one line to copy, with a copy button |
| `Avatar` | initials of a person |

## Status colours

One map, in `src/lib/statusTags.ts`. Colour carries meaning only here:

| Status | Tag |
| --- | --- |
| Applied | gray |
| Recruiter contact | cyan |
| Assessment | teal |
| Interview | blue |
| Offer | green |
| Rejected | red |
| Closed | outline |
| Status unknown, Sent by automation, Archived | outline |

The to-do state ("Needs action", yellow) and the mail type never reuse these looks.

## Loading, empty and error states

- Loading: skeletons shaped like the content, never a bare "Loading…".
- Empty: `EmptyState` says why and offers the next step.
- Error: `SectionError` or an error `InlineNotification` in plain words, with "Try again" where it helps. The rest of the page keeps working.
- A save that may or may not have happened is read back from the server and never sent again by itself.
