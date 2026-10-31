# Career Companion Design System

Design part 1 was built on 18 Oct 2026 (plan approved 17 Oct 2026). Design part 2 restyled it on 22 Oct 2026 from the mock the owner approved that day, boards "Mails list", "Mail conversation" and "Applications list" (the "Render detail pass" at the end of each board's style is the final look). The rules every agent must follow are in [AI_UI_RULES.md](AI_UI_RULES.md).

The base is IBM Carbon (the blue, the status colours, square corners, motion times and curves, Carbon icons) with the look of the Render dashboard: one flat background split by thin lines, a top bar with the breadcrumb, and the Geist font. Not a copy of Render. The goal is a mature, calm tool, not a showcase.

## Themes

Three choices: **System**, **Light** and **Dark**. System follows the computer's setting. White, Grey, GitHub and Monokai are gone; a saved White opens as System.

| Theme | Page, menu bar, cards, tables | Hover | Selected | Lines | Field borders | Menus | Text |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Light | `#ffffff` | `#f4f4f4` | `#e0e0e0` | `#e0e0e0` | `#c6c6c6` | `#ffffff` | `#161616` |
| Dark | `#0d0d0d` | `#141414` | `#272727` | `#272727` | `#3d3d3d` | `#141414` | `#f0f0f0` |

The theme is chosen in the account menu and on the Account page. Every colour is a token in `src/index.css` (`:root, .theme-light` and `.theme-dark`); components use the Tailwind names of the tokens (`bg-surface`, `text-text-secondary`), never a hex value. Dark takes Render's near-black and greys (owner, 22 Oct 2026), with the IBM blue kept: second-line text `#b3b3b3`, third `#8f8f8f`. Its tags and notices are dark tints with light text, as Render's chips are (Offer: `#6fdc8c` on `#0b2a17`; an error notice: `#2a0d10`), never full red, green or blue.

## Surfaces and lines

- One flat colour for the page, the menu bar, cards and tables. Lines split them: `border-border-subtle` between rows and sections and around cards; `border-border-default` around fields and buttons, and under a table's header row.
- Cards (side panels, settings, the mails of a conversation) are a thin line on the page colour, 24px padding.
- In a conversation every mail sits on the same card, full width; the user's own mails carry the blue "You" picture.
- Only things that float get a shadow: menus, dropdowns and dialogs use `shadow-popup` (a soft shadow, darker in Dark), because a flat page has no layers to set them apart.
- **Chosen state:** the chosen menu item and the chosen tab have the light blue tint (`bg-tint`: `#edf5ff`, Dark `#1c2638`) with blue text (`text-tint-text`). Menu items and tabs are medium weight, chosen or not (owner, 26 Oct 2026).
- No rounded corners, except true circles: status dots and avatars.

## Type

Geist (free, loaded from Google Fonts in `index.html`). Owner decision, 26 Oct 2026 (approved mock `TYPOGRAPHY-mock.html`): text is larger and stronger on a laptop than in design part 2.

| Use | Size / line height | Weight |
| --- | --- | --- |
| Page title (`h1`) | 28 / 36, letter spacing -0.02em (22 / 28 on phones) | semibold |
| Section title (`h2`) | 20 / 28 | medium |
| Card and side panel title | 15 / 22 | semibold |
| Names in tables | 15 / 22, underlined with a light line | medium (semibold while new or unread) |
| Body, table cells, descriptions (`text-sm`) | 15 / 22 | regular |
| Buttons, tabs, the side menu | 14 / 18 to 20 | medium |
| Fields, filter buttons, dropdown items, the top bar breadcrumb | 14 / 20 | regular |
| Small text: help, meta, dates, field labels (`text-xs`) | 13 / 18 | regular |
| Tags | 12 / 16 | medium |
| Table header row | 13 / 16, grey (`text-text-secondary`), sentence case | medium |
| Plain-text mail in a conversation | 15, line height 1.6 | regular |

`text-sm` is 15 / 22 and `text-xs` is 13 / 18 in this app (set in `src/index.css`), not Tailwind's sizes. Controls (buttons, tabs, the menu, fields, dropdowns, filter buttons, the breadcrumb, toasts) use `text-[14px]`, so they keep their heights. The page uses the browser's own font smoothing (no `antialiased`, which makes Geist too thin on a Mac). No uppercase text and no extra letter spacing. Geist Mono is only for code, tokens and the small count after a table's first header label. There is no small label above page titles. Sentence case everywhere ("Sync now", not "Sync Now").

## Width and spacing

- One width for every page: the full width up to **1584px**, centred beyond (`src/routes/__root.tsx`).
- Page side padding 48px on laptops, 16px on phones. Spacing follows the 8px grid: 8, 16, 24, 32, 48.
- Phones: no sideways scroll; lists are one tappable row or card each (see "Phones" below).

## Logo

The logo is the name "northlist" in lowercase Geist Bold, with the **Horizon mark** as its "o": a half sun over list lines (owner, 25 Oct 2026; approved mock `LOGO-W3-final-mock.html`, a local file). Both are in `components/layout/Logo.tsx`: `Wordmark` is the name, `Logo` is the mark alone.

- **The name** (`Wordmark`) wherever there is room: the top bar with the side menu open, the sign-in pages' top bar, the top of the phone menu, the start-up screen and the privacy page. Letter spacing -0.045em. The "o" is the mark at 0.57em, set 0.011em after the "n", -0.031em before the "r" and 0.017em below the baseline. Screen readers hear "NorthList".
- **The mark alone** (`Logo`) where the name does not fit: the top bar with the side menu collapsed, the browser tab and the phone home screen icon.
- **Never the mark next to the name** (owner, 25 Oct 2026): the sun would show twice.
- **Two drawings of the mark:** three list lines from 24px, and two thicker lines under 24px so they do not blur. Inside the name, the small mark is used under 28px.
- **Colours:** the mark is the blue of `border-interactive` (`#0f62fe` light, `#4589ff` dark), the name the main text colour. The app icon is the white mark on a blue square. Flat only: no shadows, glows or gradients; never stretched or recoloured.
- **Space:** keep empty space of at least half the mark's width around it.
- **Files:** the masters are in [`brand/`](brand/): `northlist-mark.svg`, `northlist-mark-small.svg`, `northlist-mark-white.svg`, `app-icon.svg` and `favicon.svg`. The frontend's `public/` has the browser tab icons (`favicon.svg`, `favicon.ico`), the iPhone home screen icon (`apple-touch-icon.png`, 180px), the installed app icons (below) and the link preview image (`og-image.png`, 1200 × 630), named in `index.html`.
- **Installed app** (owner, 29 and 30 Oct 2026; approved mock `PWA-icons-mock.html`, a local file): people can install NorthList from the browser menu; it opens in its own window with no browser bar. `public/manifest.webmanifest` names the app icons, made from `brand/app-icon.svg`: `icon-192.png` and `icon-512.png` are the blue tile with rounded corners (112 of 512) and a see-through outside, because computers do not round icons themselves; `icon-maskable-512.png` is the full blue square, with the mark inside the safe middle circle, for phones that cut icons into their own shape. Remake all three when the app icon changes.
- **Window bar colour:** the installed window's bar is the page colour, `#ffffff` in Light and `#0d0d0d` in Dark. `ThemeProvider` sets the `theme-color` meta whenever the theme changes, so the bar follows NorthList's own theme switch, not only the computer's setting. The Android start screen stays white in Dark: the manifest allows one colour only.

## App shell

- **Top bar** (`components/layout/TopBar.tsx`), 56px with a line under it, across the app: the name in a cell as wide as the side menu (288px), or only the mark in a 56px square cell when the menu is collapsed; then the breadcrumb; on the right the dev mode or test account chip ("Dev mode · mail sandbox", yellow, sentence case) and the round account button.
- **Account menu** (`AccountMenu.tsx`), from the account button: name and email, Account, Theme (System, Light, Dark), Log out.
- **Side menu** (`Sidebar.tsx`), under the top bar, 288px: dark item text with grey icons, the daily pages (Dashboard, Mails, Applications; the Dashboard is the home page, owner 27 Oct 2026, and the logo leads to it), a "Settings" group label in small sentence-case text, then the settings pages; "Collapse menu" on its own row at the bottom with a line above it. Collapsed, it is 56px of icons.
- **Phones:** see "Phones" below.
- The sign-in pages have their own top bar with the name and the chip.

## Phones

Up to 767px wide the app looks and works like a phone app (approved mock boards "Option B: Mails", "Option B: a mail" and "Option B: Applications", square corners). The phone parts are drawn only on phones (`useIsPhone` in `components/layout/useIsPhone.ts`), so a laptop never has them hidden in the page.

- **Top bar**, 56px, a row of square 48px cells split by thin lines: the menu, the page name, the page's own buttons (search, and "+" on Applications), the "Dev" or "Test" chip, the account button. A page puts its buttons there with `TopBarActions` and `TopBarButton` / `TopBarSearch` (`components/layout/TopBarButtons.tsx`).
- **Menu** (`PhoneMenu.tsx`): the menu cell opens a drawer from the left with every page: the daily pages, the "Settings" label, the settings pages and the mail sandbox. A tap on a page closes it.
- **Bottom bar** (`MobileNav.tsx`): Dashboard, Mails and Applications only, the chosen one on the blue tint. No "More": the rest is in the menu.
- **Page header**: the title is in the top bar, so the page shows no title and no description (`PageHeader phoneCompact`). The sync line is one short row under the tabs.
- **Lists**: `DataTable` takes `phoneRow`, one row or card per item, with no header row. A job mail is one row edge to edge (`PhoneMailRow`): picture, sender, date, subject, what it needs, the tags. Swiping it to the right does its first action, such as "Mark done", with Undo in the toast. An application is a bordered card (`PhoneApplicationCard`).
- **Filters**: the search box and the "Filters" button stay; Mails adds two one-tap chips, "Needs action" and "New only". Applications shows its status tabs as a row of chips that scrolls sideways.
- **A mail**: the job panel comes first, the next step open and "Key dates" and "Application" folded. Reply is a floating button above the bottom bar; it opens the reply full screen with a close cell. Closing keeps the text.
- **Dialogs** cover the whole screen. **Toasts** sit above the bottom bar.

## Page structure

- `PageHeader` (`components/layout/PageHeader.tsx`): one title, one line of description, actions on the right (at most one primary), and a line under it across the content, with a 24px gap below it on every page. Pages do not pass `mb-0` to it inside a `space-y-*` parent: since Tailwind 4, `mb-0` beats `space-y-*` and the gap is lost. A page that spaces its parts with a flex `gap` sets the gap so the header is still 24px from what follows. A detail page puts a row of facts under the title (for a mail: company, role, status, how many mails).
- `Breadcrumb` (`components/layout/Breadcrumb.tsx`): written in the page, shown in the top bar: "Mails › BrightPath Analytics", with the section's icon on the first level. Every detail page names itself: a mail shows its company (read from the mail, or its application's), else its subject. It leads back to the list as the user left it. A page without one shows its section in the top bar.
- Detail pages: a main column and a 360px side column from `lg` up.
- Settings pages (`SettingsSection`, owner 22 Oct 2026): each section has its title and short help above, and the controls in a card across the full width. Sections are spaced apart with no line between them. There is no left label column.
- Gmail sync (owner 22 Oct 2026, mock board "Gmail sync: compact"): the sync line and "Sync now" in the page header, as on Mails; one bordered strip with two cells, Account ("Connected as …", Disconnect) and Look back (the drop-down and one help line), stacked on phones; then the notices, and "Mails the app has read" with its tabs on the same line as its title.
- AI providers (owner 22 Oct 2026, mock board "AI providers: several connected, one active"): "Reads your mail now" is a card for the active provider with its key buttons always in sight and one row per model role (name and help, the drop-down, the state tag); "Other connected providers" is a list with "Make active" and a "⋯" menu; "Use today" and "Try it" are two cards side by side. Keys are entered only in dialogs.

## Charts and numbers

The Dashboard (owner, 27 Oct 2026) is the only page with charts. The rules:

- **One chart part:** shadcn's `chart` (`components/ui/chart.tsx`, on Recharts), changed for this app: square corners, the popup look for its tooltip, and the theme class `.theme-dark`. Bars only; no pies, no 3D, no animation on load.
- **Colours** are tokens in `src/index.css`, for light and dark: `--chart-1` blue, `--chart-2` teal, `--chart-3` purple (Carbon's categorical palette), `--chart-4` grey, `--chart-5` red for rejections; `--chart-grid` for grid lines and `--chart-track` for the empty part of a bar. A step's colour follows its status tag.
- **Cards** are a thin line on the page colour, 24px padding (16px on phones), a 14/20 semibold title and an aside on the right (a note or a link). The main number of a card is 32/40 light (28/36 on phones).
- **Small numbers:** a rate shows only from 5 applications; below that, "1 of 3". "Too few" stands in a table cell.
- **Every count leads somewhere:** to the list behind it (Applications with its step, Mails with its filter, Automation).
- **Phones:** one column; the five numbers two per row.

## Motion

Carbon's productive curves, as tokens: `ease-standard` (0.2, 0, 0.38, 0.9), `ease-entrance` (0, 0, 0.38, 0.9), `ease-exit` (0.2, 0, 1, 0.9).

| Time | Use |
| --- | --- |
| 70ms | hover and pressed colours |
| 110ms | dropdowns and menus close; tooltips open |
| 150ms | dropdowns and menus open: they fade and grow out of the side next to their field; a page fades in (`animate-page-in`) |
| 240ms | dialogs and toasts come and go; every change of size (below) |

**Size changes are never a jump** (owner, 21 Oct 2026; approved demo "Motion: today vs new"). Three shared parts do it, so a page never builds its own:

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
| `Button` | primary (blue, one per view), secondary (bordered, on the page colour), tertiary, ghost, danger, danger-ghost; `loading` shows a spinner |
| `Select` | a short fixed list; never the browser's own select |
| `Combobox` | a long list or one the server loads: type to search |
| `OverflowMenu` (`menu.tsx`) | the "…" menu for actions that are not the main one; danger actions last and red |
| `DataTable` | every list, with no outer frame: a header row of grey medium labels (the first label with the row count) over a darker line, rows split by thin lines, underlined names; sorting, filtering and paging stay on the server; skeleton rows while loading |
| `Toolbar` | above a table: a bordered search that takes the free width, then each filter as a bordered button ("Action: All"); on phones the filters open in a sheet; `ToolbarSearch` takes an `id` for the phone top bar's search |
| `Tabs` | switching between views of one list: segmented, in a bordered group, each with its count ("Job mails (5)"); `phoneLook="chips"` makes a long set a row of chips on phones |
| `Pagination` | under a list, with no frame: the page, and two small bordered arrow buttons |
| `Tag` | statuses and counts; colours from `lib/statusTags.ts` only |
| `InlineNotification` | a notice inside a page: info, success, warning, error |
| `SectionError` | a part of the page that did not load: plain words and "Try again", never the raw error |
| `EmptyState` | "nothing here": why, and the one next step |
| `Skeleton`, `SkeletonList` | loading |
| `Dialog` | a confirm or a short form: title and description at the top, the buttons at the bottom right (secondary, then the main one) under a line; its body follows its content's height |
| `Collapse` | a section the user shows and hides; the toggle keeps `aria-expanded` |
| `Reveal` | something that appears by itself, such as a field error |
| `AutoHeight` | a box whose content jumps in height (results, steps) |
| `showToast` (`toast.tsx`) | a short confirmation after an action, with Undo where the app can really undo |
| `CodeSnippet` | one line to copy, with a copy button |
| `Avatar` | initials of a person, in a circle; the user ("You") on the blue tint |

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

- Opening the app: until it knows who is signed in, the start-up screen (`components/startup/StartupScreen.tsx`, approved mock boards "Start-up screen (laptop)" and "Start-up screen (phone)"): the name and a thin moving line, centred on the page colour. It never shows the app's shape. After 5 seconds it says the server is waking up (about a minute) with the time waited; after 2 minutes, or with no network, it says what happened and offers "Try again" (full width on phones). With no network but a copy saved on this device, the app opens at once, read-only (see "Offline and alerts").
- Loading: skeletons shaped like the content, never a bare "Loading…".
- Empty: `EmptyState` says why and offers the next step.
- Error: `SectionError` or an error `InlineNotification` in plain words, with "Try again" where it helps. The rest of the page keeps working.
- A save that may or may not have happened is read back from the server and never sent again by itself.

## Offline and alerts

Owner, 29 and 30 Oct 2026 (approved mock `PUSH-OFFLINE-mock.html`, a local file).

- **The offline line** (`components/layout/OfflinePage.tsx`): under the top bar, on every page while offline: a grey strip (`bg-surface-subtle`, a line under it) with the offline icon, "You're offline." and, in secondary text, "Showing what was here at 10:42 AM." ("yesterday at", "on Oct 28 at"), from the oldest data on the page. It goes away when the device is back online, and the page refreshes by itself.
- **A page with no copy:** "No copy of this page yet", one sentence and "Try again" (tertiary), centred, with the large offline icon. The page stays mounted under it and shows once its data comes.
- **What is kept** (`lib/offlineCopy.ts`): the data of the Dashboard, Mails, a conversation, Applications, an application and the Account page, and what they show around them, for the signed-in user only. A copy is dropped after 7 days or with a new version of the app, and wiped on sign-out. The service worker (`public/sw.js`) keeps the app's files.
- **A change offline:** a toast with the offline icon instead of the tick: "You're offline. Nothing was changed." A reply: "You're offline. Your reply was not sent. It stays here; send it when you're back online." No other message next to it, and nothing is sent later.
- **Alerts on this device** (`components/auth/AlertsSection.tsx`, Account page, after "Appearance"): a state line with a small round dot (green on, grey off, red blocked) and one button: "Turn on alerts" (tertiary) or "Turn off" (ghost). Other states: blocked in the browser (how to allow it), iPhone or iPad before the app is on the Home Screen (three steps), a browser without alerts (one line). Hidden while the server has no alert keys. A button, not a switch: corners are square here.
- **An alert:** the company as the title, the step as the text, the app icon, and on Android the small one-colour mark `public/alert-badge.png` (the small mark in white on a see-through square). A tap opens that application. The first alert after turning on: "Alerts are on" / "You'll get an alert here when something needs you."
