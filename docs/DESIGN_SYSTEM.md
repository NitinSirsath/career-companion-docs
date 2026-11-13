# Career Companion Design System

Design part 1 was built on 18 Oct 2026 (plan approved 17 Oct 2026). Design part 2 restyled it on 22 Oct 2026 from the mock the owner approved that day, boards "Mails list", "Mail conversation" and "Applications list" (the "Render detail pass" at the end of each board's style is the final look). The rules every agent must follow are in [AI_UI_RULES.md](AI_UI_RULES.md).

The base is IBM Carbon (the blue, the status colours, motion times and curves) with the look of the Render dashboard: a top bar with the breadcrumb. Not a copy of Render. The goal is a mature, calm tool, not a showcase.

**13 Nov 2026 (owner): the new look.** One design for phones, Android and laptops (approved local mock `IOS-LOOK-mock/`, with the laptop before and after pictures in `IOS-LOOK-mock/desktop/`; summary `NEW-LOOK-final.md`). Everywhere: Lucide icons (thin, rounded), the device's own font, rounder corners, frosted glass only on things that float, a refresh icon on every list, and one word, **Reopen**, for putting any closed step back on the list. Phones (and Android) get an iPhone-style frame: see "Phones". Laptops keep their layout, side menu, top bar and 14px text; only the icons, font, corners, glass and motion change, and Dashboard and NorthList Apply get a refresh button. The colours do not change.

**3 Nov 2026 (owner): style v3.** After a friend said they did not like the design, the owner approved a calmer, more modern look for the whole app (approved local mocks `APP-CALM-mock/` and `LOGIN-mock/`, issue COM-433): cooler greys and lighter lines, 14/20 text, softer status tags, 36px controls, rounded corners, cards as soft grey panels, a new Mails list and new sign-in pages. It replaces the square corners of 22 Oct 2026 and the 15/22 text of 26 Oct 2026. The sections below are the current rules.

## Themes

Three choices: **System**, **Light** and **Dark**. System follows the computer's setting. White, Grey, GitHub and Monokai are gone; a saved White opens as System.

| Theme | Page, menu bar, cards, tables | Hover | Selected | Lines | Field borders | Menus | Text |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Light | `#ffffff` | `#f5f5f6` | `#ececee` | `#ededef` | `#e1e1e5` | `#ffffff` | `#18181b` |
| Dark | `#111113` (menu and top bar `#08080a`) | `#18181b` | `#222226` | `#232327` | `#2e2e33` | `#18181b` | `#ededef` |

The theme is chosen in the account menu and on the Account page. Every colour is a token in `src/index.css` (`:root, .theme-light` and `.theme-dark`); components use the Tailwind names of the tokens (`bg-surface`, `text-text-secondary`), never a hex value. Second and third text: Light `#52525b` and `#71717a`, Dark `#a1a1aa` and `#8b8b94` (owner, 3 Nov 2026). The IBM blue stays for buttons and links. Dark tags and notices are dark tints with light text (Offer: `#75e0a7` on `#081f13`; an error notice: `#1f0b0a`), never full red, green or blue.

## Surfaces and lines

- The page, the menu bar and tables share the page colour. Lines split rows and sections (`border-border-subtle`); `border-border-default` goes around fields and buttons.
- **Cards are panels** (owner, 3 Nov 2026, as on the Dashboard's Calm look): the `panel` utility in `src/index.css`, a soft grey fill (`surface-subtle`) with 16px corners (13 Nov 2026) and no line, 24px padding. Side cards on a mail, settings sections, the AI provider cards, the Gmail sync strip and the Dashboard cards are panels.
- **Mails stay white:** in a conversation every mail sits on a white card with a thin line and 16px corners, full width, so it reads like mail; the user's own mails carry the blue "You" picture. The reply box is the same.
- **Shadows** only on things that float (menus, dropdowns and dialogs use `shadow-popup`), the faint shadow under buttons, and in Dark the raised page sheet and the open menu item.
- **Rounder corners** (owner, 13 Nov 2026; before: 6 / 8 / 12px): buttons, fields, dropdown boxes, the search box, tabs, chips, filter pills and floating bars are fully round (`rounded-full`); other small boxes 10px (`rounded-md`); groups, cards, panels and menus 16px (`rounded-lg`); dialogs and sheets 28px (`rounded-xl`; a tall sheet 22px at the top); the laptop's Dark page sheet keeps 12px (`rounded-page`); tags stay pills. Each shared part sets its own corner.
- **Frosted glass** (owner, 13 Nov 2026): only on things that float over the page: the phone's tab bar, action bars and round top buttons (`glass` utility), menus and dropdown lists (`glass-menu`), and toasts. Quiet: a blur and a hairline edge, no shine or glow. Pages, cards and dialogs stay solid.
- **Chosen state:** the open page in the side menu is a pale grey fill with dark text in Light, and a raised item (page colour, thin light outline, small shadow) in Dark, so blue means "you can act here". Segmented tabs keep the light blue tint (`bg-tint`) with blue text; the Mails tabs are underlined (see "Mails list").

## Type

The device's own font (owner, 13 Nov 2026): San Francisco on Apple devices, Roboto or the maker's font on Android, the system font on Windows (`--font-sans` in `src/index.css`; no web font is loaded). The product site (`site/`) keeps Geist. Owner decision, 26 Oct 2026 (approved mock `TYPOGRAPHY-mock.html`): text is larger and stronger on a laptop than in design part 2.

| Use | Size / line height | Weight |
| --- | --- | --- |
| Page title (`h1`) | 26 / 32, letter spacing -0.025em (22 / 28 on phones) | semibold |
| Section title (`h2`) | 20 / 28 | medium |
| Card and side panel title | 15 / 22 | semibold |
| Names in tables | 15 / 22, underlined with a light line | medium (semibold while new or unread) |
| Body, table cells, descriptions (`text-sm`) | 14 / 20, letter spacing -0.006em | regular |
| Buttons | 13 / 18 | medium |
| Tabs, the side menu | 14 / 20 | medium for the chosen one, else regular |
| Fields, filter buttons, dropdown items, the top bar breadcrumb | 14 / 20 | regular |
| Small text: help, meta, dates, field labels (`text-xs`) | 13 / 18 | regular |
| Tags | 12 / 16 | medium |
| Table header row | 13 / 16, grey (`text-text-secondary`), sentence case | medium |
| Plain-text mail in a conversation | 15, line height 1.6 | regular |

`text-sm` is 14 / 20 and `text-xs` is 13 / 18 in this app (set in `src/index.css`), not Tailwind's sizes. Dates and counts use numbers of equal width (`tabular-nums`). Controls (buttons, tabs, the menu, fields, dropdowns, filter buttons, the breadcrumb, toasts) use `text-[14px]`, so they keep their heights. The page uses the browser's own font smoothing (no `antialiased`). No uppercase text and no extra letter spacing. The device's mono font (`font-mono`) is only for code, tokens and the small count after a table's first header label. Phones have their own sizes: see "Phones". There is no small label above page titles. Sentence case everywhere ("Sync now", not "Sync Now").

## Width and spacing

- One width for every page: the full width up to **1584px**, centred beyond (`src/routes/__root.tsx`).
- Page side padding 48px on laptops, 16px on phones. Spacing follows the 8px grid: 8, 16, 24, 32, 48.
- Phones: no sideways scroll; lists are one tappable row or card each (see "Phones" below).

## Logo

The logo is the name "northlist" in lowercase bold (Geist on the product site and in the brand files; the app's own font in the app), with the **Horizon mark** as its "o": a half sun over list lines (owner, 25 Oct 2026; approved mock `LOGO-W3-final-mock.html`, a local file). Both are in `components/layout/Logo.tsx`: `Wordmark` is the name, `Logo` is the mark alone.

- **The name** (`Wordmark`) wherever there is room: the top bar with the side menu open, the top of the sign-in pages, the top of the phone menu, the start-up screen and the privacy page. Letter spacing -0.045em. The "o" is the mark at 0.57em, set 0.011em after the "n", -0.031em before the "r" and 0.017em below the baseline. Screen readers hear "NorthList".
- **The mark alone** (`Logo`) where the name does not fit: the top bar with the side menu collapsed, the browser tab and the phone home screen icon.
- **Never the mark next to the name** (owner, 25 Oct 2026): the sun would show twice.
- **Two drawings of the mark:** three list lines from 24px, and two thicker lines under 24px so they do not blur. Inside the name, the small mark is used under 28px.
- **Colours:** the mark is the blue of `border-interactive` (`#0f62fe` light, `#4589ff` dark), the name the main text colour. The app icon is the white mark on a blue square. Flat only: no shadows, glows or gradients; never stretched or recoloured.
- **Space:** keep empty space of at least half the mark's width around it.
- **Files:** the masters are in [`brand/`](brand/): `northlist-mark.svg`, `northlist-mark-small.svg`, `northlist-mark-white.svg`, `app-icon.svg` and `favicon.svg`. The frontend's `public/` has the browser tab icons (`favicon.svg`, `favicon.ico`), the iPhone home screen icon (`apple-touch-icon.png`, 180px), the installed app icons (below) and the link preview image (`og-image.png`, 1200 × 630), named in `index.html`.
- **Installed app** (owner, 29 and 30 Oct 2026; approved mock `PWA-icons-mock.html`, a local file): people can install NorthList from the browser menu; it opens in its own window with no browser bar. `public/manifest.webmanifest` names the app icons, made from `brand/app-icon.svg`: `icon-192.png` and `icon-512.png` are the blue tile with rounded corners (112 of 512) and a see-through outside, because computers do not round icons themselves; `icon-maskable-512.png` is the full blue square, with the mark inside the safe middle circle, for phones that cut icons into their own shape. Remake all three when the app icon changes.
- **Window bar colour:** the installed window's bar is the page colour, `#ffffff` in Light and `#0d0d0d` in Dark. `ThemeProvider` sets the `theme-color` meta whenever the theme changes, so the bar follows NorthList's own theme switch, not only the computer's setting. The sign-in pages are always light, and so is the bar there (`setLightOnly`). The Android start screen stays white in Dark: the manifest allows one colour only.

## App shell

- **Laptops** (`components/layout/LaptopShell.tsx`); phones have their own frame (see "Phones").
- **Top bar** (`components/layout/TopBar.tsx`), 56px with a line under it in Light, across the app: the name in a cell as wide as the side menu (288px), or only the mark in a 56px square cell when the menu is collapsed; then the breadcrumb; on the right the dev mode or test account chip ("Dev mode · mail sandbox", yellow, sentence case) and the round account button.
- **Account menu** (`AccountMenu.tsx`), from the account button: name and email, Account, Theme (System, Light, Dark), Log out.
- **Side menu** (`Sidebar.tsx`), under the top bar, 288px: dark item text with grey icons, the daily pages (Dashboard, Mails, Applications; the Dashboard is the home page, owner 27 Oct 2026, and the logo leads to it), a "Settings" group label in small sentence-case text, then the settings pages; "Collapse menu" on its own row at the bottom with a line above it. Collapsed, it is 56px of icons. On tablet widths (768 to 1199px) it starts collapsed until the user chooses (owner, 3 Nov 2026); the choice is remembered.
- **Phones:** see "Phones" below.
- **Dark mode** (owner, 3 Nov 2026): the side menu and top bar sit on near-black (`--canvas`, `#08080a`) without lines, and the page sits on a raised sheet in the page colour with 12px corners, 8px from the right and bottom edge, a thin light outline and a soft shadow (`--sheet-shadow`). Light keeps the plain white menu and top bar with thin lines (owner: keep it simple).
- The sign-in pages have no top bar (see "Sign-in pages").

## Phones

Up to 767px wide the app is a phone app in today's iPhone style (owner, 13 Nov 2026; approved local mock `IOS-LOOK-mock/`). Android gets the same design. The frame is `components/phone/PhoneShell.tsx`; phone-only parts are drawn only on phones (`useIsPhone`), never hidden with CSS.

- **Text:** 17px for rows and mail text, 15px for second lines (`text-sm` on phones), 13px for small notes (the `phone-type` utility). No field is under 16px, so an iPhone never zooms in.
- **Tab bar** (`TabBar.tsx`): a floating glass bar with Dashboard, Mails, Applications and Settings: line icons when off, filled and blue when on, a soft highlight that slides to the chosen tab. It shows on each tab's first page; a page opened from it has its own action bar instead. There is no menu drawer and no account button on phones.
- **Settings tab** (`/settings`, `SettingsTab.tsx`): Gmail sync, AI providers, NorthList Apply, the mail sandbox (dev mode and test accounts), Account, Appearance (`/settings/appearance`), alerts on this phone as a switch, and Sign out, as grouped rows with small coloured icon squares.
- **Top bar:** no bar of its own, only round glass buttons over the page (`GlassButton`): Back on the left on a page opened from another; on the right the page's own buttons, put there with `TopBarActions`: on a list, refresh, search and filters in one glass capsule (`GlassCapsule`, `CapsuleButton`, `PhoneRefresh`); on a mail, one ⓘ. A soft blur fades in under the buttons as the page scrolls.
- **Big titles** (`PhoneTitle`): 34px on a tab's first page, 28px on a page opened from it. They scroll under the top bar, where the small title then shows; pulling down at the top makes them a little bigger (iPhone only). `PageHeader` draws one on phones by itself.
- **Moving between pages** (`usePhoneStack.ts`, `lib/phoneStack.ts`): a page slides in from the right; the page under it moves a third of the way left and darkens. Back, a swipe from anywhere on the page (not only the edge) and the phone's own back gesture or button slide back; let go halfway and the page springs back. Each tab keeps its pages and its place; tapping the open tab goes back to its first page, then to the top. Safari's own back swipe is not drawn twice.
- **Sheets** (`Sheet` in `ui/sheet.tsx`, on Base UI Drawer): they float above the page with round corners, spring up, follow the finger when dragged down and close on a fling; the page behind shrinks a little (fully behind a tall sheet such as Reply, half behind a short one). An action list is an `ActionSheet`; rows sit on a `SheetGroup`. Dialogs rise as tall sheets on phones.
- **Lists:** rows on one soft grey group with thin lines between them (`PhoneGroup`, `PhoneSection`, `PhoneLinkRow`; `DataTable` with `phoneRow`). The filters in use show as small pills under the title; the search opens as a row under the title, with Cancel. In progress and NorthList Apply on Applications are a segmented control.
- **Mails:** "Job mails ⌄" is the big title and switches the lists; under it one status line: when mail last came in and what the AI is doing ("Synced 12:00 PM · 3 new mails wait for AI ›"), which opens the AI reading status in a sheet. Each mail is four short lines with no picture (who and when, the subject, the next step, the status and company), with a blue dot in the margin while new. Swiping a row to the right does its first action. The day's summary sits under the list.
- **A mail** (`PhoneMailPage.tsx`): the big subject and the facts, the next step as one row (its sheet has "I did this" or Reopen, the way to its mail, and why it closed), the way to make the application while there is none, then the mails edge to edge with older ones folded to one line. Key dates and the application are behind ⓘ. A floating action bar: Reply (Draft once text is kept), Done or Reopen, AI reply, More. Reply is a tall sheet: Close and Send at its top, the AI's help (reply choices, an optional hint, "Write with AI") above the text, and the text tools at its bottom; AI reply opens it with the hint box ready.
- **An application:** the big company name, the role and the status (a tap opens the status sheet), then grouped sections: Next steps, Conversations, Details, People, History. A floating action bar: Reply, Status, More.
- **Refresh** on every list: Dashboard, Mails (also asks Gmail for new mail), Applications, Gmail sync and NorthList Apply. It spins while the page is fetched again and says "Up to date."
- **Toasts** float just above the tab bar.
- **Less motion asked for:** pages and sheets fade instead of moving, and the page behind a sheet does not shrink.

## Page structure

- `PageHeader` (`components/layout/PageHeader.tsx`): one title, one line of description, actions on the right (at most one primary), and a line under it across the content, with a 24px gap below it on every page. Pages do not pass `mb-0` to it inside a `space-y-*` parent: since Tailwind 4, `mb-0` beats `space-y-*` and the gap is lost. A page that spaces its parts with a flex `gap` sets the gap so the header is still 24px from what follows. A detail page puts a row of facts under the title (for a mail: company, role, status, how many mails).
- `Breadcrumb` (`components/layout/Breadcrumb.tsx`): written in the page, shown in the top bar: "Mails › BrightPath Analytics", with the section's icon on the first level. Every detail page names itself: a mail shows its company (read from the mail, or its application's), else its subject. It leads back to the list as the user left it. A page without one shows its section in the top bar.
- Detail pages: a main column and a 360px side column from `lg` up.
- Settings pages (`SettingsSection`, owner 22 Oct 2026): each section has its title and short help above, and the controls in a card across the full width. Sections are spaced apart with no line between them. There is no left label column.
- Gmail sync (owner 22 Oct 2026, mock board "Gmail sync: compact"): the sync line and "Sync now" in the page header, as on Mails; one bordered strip with two cells, Account ("Connected as …", Disconnect) and Look back (the drop-down and one help line), stacked on phones; then the notices, and "Mails the app has read" with its tabs on the same line as its title.
- AI providers (owner 22 Oct 2026, mock board "AI providers: several connected, one active"): "Reads your mail now" is a card for the active provider with its key buttons always in sight and one row per model role (name and help, the drop-down, the state tag); "Other connected providers" is a list with "Make active" and a "⋯" menu; "Use today" and "Try it" are two cards side by side. Keys are entered only in dialogs.

## Mails list

Owner, 3 Nov 2026 (approved mock `APP-CALM-mock/`, issue COM-437). The Mails page is one list, **newest first** like a mail app; "Needs action first" stays in Sort.

- **Header:** "Mails", and under it "4 mails need you, 2 are past due. Everything else is up to date." (`MailSummary`, counted by `GET /api/conversations/summary` over all mails, not one page). "4 mails need you" turns the Needs action filter on and off. Synced time and "Sync now" on the right.
- **AI notice:** one slim line (`AIAccessNotice compact`) with the same words and the same ways to fix it.
- **Toolbar** (laptops), one line that stays on screen while the list scrolls: underlined tabs Job mails (with count), Whole inbox, Restricted; on the right the search box ("Search mails"), a "Filters" button with the number in use that opens a menu with Action, Type, Application and New only, and the sort. Chosen filters show as chips under it with "Clear all".
- **A row** (`MailRow`): coloured initials; the subject (bold and a blue dot when new); the next step with a blue arrow and its due date (red when late), a replied step with a reply mark, a done step in grey with a green check and its reason, or "Nothing needed"; then sender, company, type and mail count in small grey text. On the right the application (company, status tag, role) or "Not linked" with "Create application"; the date; the "⋮" menu, shown when the row is pointed at or has focus. The whole row opens the mail (the subject is a stretched link; the application and menu sit above it). No header row and no Type column.
- **Phones:** the top bar title is "Job mails ▾", a menu that switches lists; the top bar's search button opens a search row under it with Cancel; one row of chips (Needs action, New only, Filters, which opens a sheet with every filter and the sort) with a sync button at its end; rows show initials, sender and date, the subject in grey, the next step (two lines at most) with its due date on its own line, then the status tag with company and role. "Synced today, …" ends the list.
- **Coloured initials** (`Avatar`): six soft colours; the same name always gets the same colour (`avatarTone`). The user's own picture keeps the blue tint.

## Sign-in pages

Owner, 3 Nov 2026 (approved mock `LOGIN-mock/`, issue COM-438); polished on 7 Nov 2026 (approved mock `LOGIN-POLISH-mock/`, issue COM-446): a filled "Sign in", a picture with no visible bottom corner, a little motion, and a blue band on phones and tablets. `SignInLayout` holds both sign-in pages (`LoginPage`, `TestLoginPage`).

- **Laptops (1024px and up):** two halves in a 12px frame, filling the screen. Left: the name at the top (with the dev or test chip), the form in the middle (at most 380px wide), the footer at the bottom ("Free while in beta. No card needed.", Privacy, northlist.site). Right: a flat blue panel (`#0f62fe`, 16px corners) with the product page headline "Your job search, sorted from your Gmail.", its lead sentence, a faint half sun in the corner and a picture of the Mails list (`SignInPicture`, drawn with the app's own parts, no image file). The picture has 12px corners and is always taller than its space, so the panel's right and bottom edges cut it off and no bottom corner is ever seen. No gradient or glow.
- **The picture moves a little, once** (`lib/signInPicture.ts`): 1.8 s after the page opens, a made-up mail grows in at the top on the pale blue tint, its next step shows the app's grey "Reading the mail" shimmer, and 2.2 s later its step and application appear, "4 mails need you" becomes 5 and the tint fades. A second mail (a rejection, "Nothing needed") comes the same way 4.8 s later. Then nothing more moves. With "reduce motion" on, the picture stays as it starts.
- **Google first:** "Continue with Google" in Google's own dark button with the four-colour "G", then "New here? Continuing with Google creates your account.", then "or sign in with email", Email, Password with "Forgot password?" by its label (it opens the advice in a small note), and the filled blue "Sign in" (it was outlined until 7 Nov 2026). In dev mode there is no Google button and "Create test user" follows the password form.
- **Tablets and phones (under 1024px):** a short blue band on top (16px corners, about 292px high): the white name, the headline, and a small Mails card (subject and next step only, with the same motion) that runs past the band's right and bottom edges. Then the form (at most 380px wide), then the footer right after the form. The name above the form is shown on laptops only.
- **Light only**, like northlist.site: while a sign-in page is open, `ThemeProvider` shows the light theme (`setLightOnly`); the saved theme stays and comes back after sign-in.

## Charts and numbers

The Dashboard (owner, 27 Oct 2026; reworked 3 Nov 2026) is the only page with charts. The rules:

- **One chart part:** shadcn's `chart` (`components/ui/chart.tsx`, on Recharts), changed for this app: the popup look for its tooltip (6px corners), and the theme class `.theme-dark`. Bars, columns, lines and small trend lines; no pies, no 3D, no animation on load. Every chart has a legend, a hover readout with every series, keyboard reading, and a hidden table for screen readers.
- **Colours** are tokens in `src/index.css`, for light and dark: `--chart-1` blue, `--chart-2` teal, `--chart-3` purple (Carbon's categorical palette), `--chart-4` grey, `--chart-5` red for rejections; `--chart-grid` for grid lines and `--chart-track` for the empty part of a bar. Dark `--chart-2` is `#009d9a`, one step darker than Carbon's, because `#08bdba` fails the colour-blind lightness check on the dark page (3 Nov 2026).
- **Cards** (the Calm look, owner, 3 Nov 2026) are soft grey panels (`surface-subtle`) with no outline, 8px apart, 24px padding (16px on phones), a 14/20 semibold title and an aside on the right (a note or a link); lines inside a card are the page colour. Sections have a 20/28 semibold title (18/24 on phones). The main number of a card is 32/40 light (28/36 on phones).
- **Small numbers:** a rate shows only from 5 applications; below that, "1 of 3". "Too few" stands in a table cell.
- **Every count leads somewhere:** to the list behind it (Applications with its step, Mails with its filter, Automation).
- **Layout** follows the page's own width (container queries), so it fits with the side menu open or closed. Phones: one column, the three "needs you now" tiles as rows with the number on the right, the five numbers two per row.

## Motion

Carbon's productive curves, as tokens: `ease-standard` (0.2, 0, 0.38, 0.9), `ease-entrance` (0, 0, 0.38, 0.9), `ease-exit` (0.2, 0, 1, 0.9).

| Time | Use |
| --- | --- |
| 70ms | hover and pressed colours |
| 110ms | dropdowns and menus close; tooltips open |
| 150ms | a page fades in (`animate-page-in`) |
| 160ms | menus and dropdowns close |
| 240ms | dialogs come and go; every change of size (below) |
| 320ms | menus and dropdowns open with a small spring (`ease-spring`), growing out of their button (13 Nov 2026) |
| 420ms | toasts slide in with the same spring |
| 450ms | phone sheets rise and settle (`ease-sheet`) |

**Phone pages move on springs** (owner, 13 Nov 2026): the slide between pages and the tab bar's highlight use a small spring in `lib/spring.ts` (no animation library), so a fling keeps its speed. With less motion asked for, they are a short fade. |

**Size changes are never a jump** (owner, 21 Oct 2026; approved demo "Motion: today vs new"). Three shared parts do it, so a page never builds its own:

- `Collapse`: a section the user shows and hides (an older mail, the AI's reading, done steps). It grows from no height to its own and fades in, and leaves the same way.
- `Reveal`: something that appears by itself, such as an error under a field. It grows and fades in. `InlineNotification` already uses it.
- `AutoHeight`: a box that follows its content's height when the content jumps (new results, the next step). Every dialog body and the Combobox list already sit in one. A part inside that animates its own height is followed as it moves, so the two never fight.

Motion is short and quiet and never needed to understand the screen. With "reduce motion" on, it is off, and every part still ends at the right size.

**Tailwind 4 rule.** `translate-*`, `scale-*` and `rotate-*` set the CSS `translate`, `scale` and `rotate` properties, not `transform`. A transition must list the property it moves (`transition-[opacity,translate]`), or use `transition-transform`, which covers all four. A list with only `transform` makes the movement snap while only the fade runs.

## Icons

Lucide icons only (`lucide-react`, owner, 13 Nov 2026), imported by their `…Icon` names (`CheckIcon`), thin lines (1.75, set once in `main.tsx`), 16px in text and buttons, 20px in the side menu, 22 to 26px in the phone's bars. The AI is shown with `AiIcon` (`components/ui/ai-icon.tsx`): the shape of Carbon's machine-learning model, drawn thin. Never a sparkle.

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
| `Sheet`, `SheetGroup`, `SheetAction`, `ActionSheet` (`sheet.tsx`) | phones: a sheet from the bottom on Base UI Drawer; rows on a group; a list of actions |
| `Switch` | an on/off setting, as phone settings have |
| `RefreshButton`, `useRefresh` | fetching a list page again without reloading the app |
| `AiIcon` (`ai-icon.tsx`) | the AI's icon |

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
| Status unknown, No reply yet (NorthList Apply, no status), Archived | outline |

The to-do state ("Needs action", yellow) and the mail type never reuse these looks.

Tags (owner, 3 Nov 2026) are pills, 22px high, 12px medium text, with a pale fill, a thin outline of the same colour family (`--tag-*-ring`) and a small dot before the word.

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
