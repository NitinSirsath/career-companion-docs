# Career Companion Design System

Design part 1 was built on 18 Oct 2026 (plan approved 17 Oct 2026). Design part 2 restyled it on 22 Oct 2026 from the mock the owner approved that day, boards "Mails list", "Mail conversation" and "Applications list" (the "Render detail pass" at the end of each board's style is the final look). The rules every agent must follow are in [AI_UI_RULES.md](AI_UI_RULES.md).

The base is IBM Carbon (the blue, the status colours, motion times and curves) with the look of the Render dashboard: a top bar with the breadcrumb. Not a copy of Render. The goal is a mature, calm tool, not a showcase.

**18 and 19 Nov 2026 (owner): the Linear iOS look.** Approved local mocks `PHONE-LINEAR-mock/index.html` (phones, with "The phone rules" and "Motion" at the bottom) and `PHONE-LINEAR-mock/desktop.html` (laptops, with "Laptop rules"). Phones follow the Linear iOS app strictly: controls float on glass, an icon-only tab bar, plain rows with thin lines and no boxes, sheets that float, one small status ring in place of coloured status tags, and Linear's motion. See "Phones". Laptops keep their layout and take only small changes: the same rings, person-first mail rows, the facts as pills in one round card, glass "…" menus, the next step as plain text, and the Admin numbers in one line. See "Laptops". One accent blue everywhere. Where the sections below still name an older rule, the 18 Nov rule wins.

**13 Nov 2026 (owner): the new look.** One design for phones, Android and laptops (approved local mock `IOS-LOOK-mock/`, with the laptop before and after pictures in `IOS-LOOK-mock/desktop/`; summary `NEW-LOOK-final.md`). Everywhere: Lucide icons (thin, rounded), the device's own font, rounder corners, frosted glass only on things that float, a refresh icon on every list, and one word, **Reopen**, for putting any closed step back on the list. Phones (and Android) get an iPhone-style frame: see "Phones". Laptops keep their layout, side menu, top bar and 14px text; only the icons, font, corners, glass and motion change, and Dashboard and NorthList Apply get a refresh button. The colours do not change.

**3 Nov 2026 (owner): style v3.** After a friend said they did not like the design, the owner approved a calmer, more modern look for the whole app (approved local mocks `APP-CALM-mock/` and `LOGIN-mock/`, issue COM-433): cooler greys and lighter lines, 14/20 text, softer status tags, 36px controls, rounded corners, cards as soft grey panels, a new Mails list and new sign-in pages. It replaces the square corners of 22 Oct 2026 and the 15/22 text of 26 Oct 2026. The sections below are the current rules.

## Themes

Three choices: **System**, **Light** and **Dark**. System follows the computer's setting. White, Grey, GitHub and Monokai are gone; a saved White opens as System.

| Theme | Page, menu bar, cards, tables | Hover | Selected | Lines | Field borders | Menus | Text |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Light | `#ffffff` | `#f5f5f6` | `#ececee` | `#ededef` | `#e1e1e5` | `#ffffff` | `#18181b` |
| Dark | `#111113` (menu and top bar `#08080a`) | `#18181b` | `#222226` | `#232327` | `#2e2e33` | `#18181b` | `#ededef` |

The theme is chosen in the account menu and on the Account page. Every colour is a token in `src/index.css` (`:root, .theme-light` and `.theme-dark`); components use the Tailwind names of the tokens (`bg-surface`, `text-text-secondary`), never a hex value. Second and third text: Light `#52525b` and `#71717a`, Dark `#a1a1aa` and `#8b8b94` (owner, 3 Nov 2026). One accent blue (owner, 18 Nov 2026), `--accent` (`text-accent`, `bg-accent`): `#2563eb` in Light, `#3b82f6` in Dark, for buttons, links, focus, the chosen tab and the logo's mark. It replaces the IBM blue `#0f62fe`. Dark tags and notices are dark tints with light text (Offer: `#75e0a7` on `#081f13`; an error notice: `#1f0b0a`), never full red, green or blue.

## Surfaces and lines

- The page, the menu bar and tables share the page colour. Lines split rows and sections (`border-border-subtle`); `border-border-default` goes around fields and buttons.
- **Cards are panels** (owner, 3 Nov 2026, as on the Dashboard's Calm look): the `panel` utility in `src/index.css`, a soft grey fill (`surface-subtle`) with 16px corners (13 Nov 2026) and no line, 24px padding. Side cards on a mail, settings sections, the AI provider cards, the Gmail sync strip and the Dashboard cards are panels.
- **Mails stay white:** in a conversation every mail sits on a white card with a thin line and 16px corners, full width, so it reads like mail; the user's own mails carry the blue "You" picture. The reply box is the same.
- **Shadows** only on things that float (dialogs use `shadow-popup`; menus, dropdowns and the phone's glass carry the glass's own shadow since 18 Nov 2026), the faint shadow under buttons, and in Dark the raised page sheet and the open menu item.
- **Rounder corners** (owner, 13 Nov 2026; before: 6 / 8 / 12px): buttons, fields, dropdown boxes, the search box, tabs, chips, filter pills and floating bars are fully round (`rounded-full`); other small boxes 10px (`rounded-md`); groups, cards and panels 16px (`rounded-lg`); menus and pop-overs 14px (18 Nov 2026); dialogs 28px (`rounded-xl`); phone sheets 38px when they float and 34px at the top of a tall one (18 Nov 2026); the laptop's Dark page sheet keeps 12px (`rounded-page`); tags stay pills. Each shared part sets its own corner.
- **Glass** (owner, 13 Nov 2026; the Linear material since 18 Nov 2026): only on things that float over the page. `glass` (the phone's tab bar, bottom bar, round buttons and capsules): a blur with more colour (20px, saturate 1.9), a light tint (`--glass`), a hairline edge, a bright line along the top (`--glass-hi`) and a soft shadow. `glass-menu` (every menu, dropdown and pop-over, phone and laptop, and the laptop Install card): the same glass, a little more solid (`--menu-glass`) so the words read. Floating sheets use `--sheet-glass`. `--glass-on` is the open tab's lighter pill and a segmented control's chosen part. No glow. Pages, cards and dialogs stay solid.
- **Blurred edges** (phones, 18 Nov 2026): content scrolls under the floating controls, and the top and bottom edges blur and fade into the page (`edge-top`, `edge-bottom`; the phone frame draws both).
- **Chosen state:** the open page in the side menu is a pale grey fill with dark text in Light, and a raised item (page colour, thin light outline, small shadow) in Dark, so blue means "you can act here". Segmented tabs keep the light blue tint (`bg-tint`) with blue text; the Mails tabs are underlined (see "Mails list").

## Type

The device's own font (owner, 13 Nov 2026): San Francisco on Apple devices, Roboto or the maker's font on Android, the system font on Windows (`--font-sans` in `src/index.css`; no web font is loaded). The product site (`site/`) keeps Geist. Owner decision, 26 Oct 2026 (approved mock `TYPOGRAPHY-mock.html`): text is larger and stronger on a laptop than in design part 2.

| Use | Size / line height | Weight |
| --- | --- | --- |
| Page title (`h1`) | 26 / 32, letter spacing -0.025em (phones: see "Phones") | semibold |
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

- **The name** (`Wordmark`) wherever there is room: the top bar with the side menu open, the top of the sign-in pages, the top of the phone menu and the privacy page. Letter spacing -0.045em. The "o" is the mark at 0.57em, set 0.011em after the "n", -0.031em before the "r" and 0.017em below the baseline. Screen readers hear "NorthList".
- **The mark alone** (`Logo`) where the name does not fit: the top bar with the side menu collapsed, the browser tab and the phone home screen icon. Also the start-up screen, where the mark moves (see "Opening the app").
- **Never the mark next to the name** (owner, 25 Oct 2026): the sun would show twice.
- **Two drawings of the mark:** three list lines from 24px, and two thicker lines under 24px so they do not blur. Inside the name, the small mark is used under 28px.
- **Colours:** the mark is the blue of `border-interactive`, which is the accent since 18 Nov 2026 (`#2563eb` light, `#3b82f6` dark; before `#0f62fe` and `#4589ff`), the name the main text colour. The app icon is the white mark on a blue square. Flat only: no shadows, glows or gradients; never stretched or recoloured.
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

Up to 767px wide the app is a phone app that follows the Linear iOS app strictly (owner, 18 and 19 Nov 2026; approved local mock `PHONE-LINEAR-mock/index.html`, with "The phone rules" and "Motion" at its bottom). It replaces the iPhone-style frame of 13 Nov 2026 (mock `IOS-LOOK-mock/`) where the two differ. Android gets the same design. The frame is `components/phone/PhoneShell.tsx`; phone-only parts are drawn only on phones (`useIsPhone`), never hidden with CSS.

**The phone rules** (from the mock, used on every screen):

- **Controls float on glass.** Round glass buttons (Back, close), glass capsules for two or three icons, and an icon-only glass tab bar with Search as its own circle. No solid top bars.
- **Content scrolls under the controls;** the top and bottom edges blur and fade.
- **Titles.** A list has a big bold title, in the same row as its glass buttons. A detail page has a small grey line, a big bold title, then the facts as pills in one round card.
- **Detail pages end in a bottom bar:** a glass circle, a glass field, a glass circle.
- **Sheets float** with round corners and a glass × to close. Tall sheets push the page back.
- **No cards on lists,** only thin lines. Mails start with the person, applications with the company. A status is one small ring.
- **Colour.** One accent (our blue), the device's font, body 15.5px, bold titles. Colour only in status rings, the new-mail dot, red for past due and green switches.

**The parts:**

- **Text** (`text-phone-*` in `src/index.css`): `text-phone-title` 30/36 bold (a list's big title), `text-phone-detail` 25/30 bold (a detail page's title), `text-phone-heading` 18/24 bold (a section), `text-phone-row` 15.5/21 (a row's first line), `text-phone-sub` 13.5/18 (its grey lines), `text-phone-label` 13/18 semibold (a plain list's grey label), `text-phone-small` 13/18 (dates and notes). The `phone-type` utility sets body 15.5 and `text-sm` 13.5 (before: 17 and 15). No field is under 16px, so an iPhone never zooms in. `cn()` knows these are sizes, so a text colour next to one does not remove it.
- **Tab bar** (`TabBar.tsx`, inside `BarHost.tsx`): a 50px glass capsule of icons only: Today, Mails, Applications, Settings (the words are for screen readers). The open tab sits in a lighter, wider pill with a blue icon; Mails has a blue dot while there are new job mails. **Search** is its own glass circle beside it: it opens the page's own search (`useTabBarSearch`), else goes to Mails and opens its search. Tapping the open tab goes back to its first page, then to the top. There is no menu drawer and no account button on phones.
- **Top bar** (`PhoneTopBar.tsx`): no bar of its own. On a tab's first page the big title (`PhoneTitle`) sits at the left of the row and the page's one to three icons sit in one glass capsule at the right (`TopBarActions`, `GlassCapsule`, `CapsuleButton`). On a detail page a glass Back circle sits at the left and the capsule at the right. The small title (15.5px) fades into the middle once the big title has scrolled away.
- **Big titles** (`PhoneTitle`): 30px on a list, 25px on a detail page; pulling down at the top makes them a little bigger (iPhone only). `PhoneTitle size="bar"` (or `PageHeader phoneSmallTitle`) is for setup pages (Gmail sync, AI providers, NorthList Apply, the mail sandbox, Account, Appearance, Admin, Free AI money): no big title, the small title always in the middle of the top bar, and the heading kept for screen readers. Owner messages and the offline line come under the title.
- **Bottom bar** (`components/phone/BottomBar.tsx`): a detail page's `PhoneBottomBar` holds a `BarCircle`, one `BarField` and a `BarCircle`, where a tab's first page has the tab bar. A mail: AI reply, "Reply to Priya", Open in Gmail. Admin: "Message to everyone" only. `BarField` draws no glass of its own: the tab bar's glass becomes it. A detail page with no bar (an application) lets the capsule shrink away. `ActionBar` (13 Nov 2026) is replaced.
- **Plain rows** (`components/phone/PlainList.tsx`, like Linear's Settings): `PlainSection` (a small grey label, rows between two thin lines, no box and no icons, a grey note under them, edge to edge), `PlainRow` (a label, a grey second line, grey words or a switch or tick at the right; `tone="blue"` for the one main action, `tone="red"` for a risky one, last on the page) and `PlainLinkRow` (label, grey value, chevron). Rows are at least 50px and grey while touched. They replace the rounded grey groups of 13 Nov 2026 (`PhoneGroup`, `PhoneSection`, `PhoneLinkRow`; do not use them for new pages). `DataTable` with `phonePlain` draws plain rows edge to edge (Applications).
- **Sheets** (`Sheet` in `ui/sheet.tsx`, on Base UI Drawer): a **floating** sheet sits inset on all sides with 38px corners on frosted glass, a glass × at the left and the title in the middle; it is used for every choice, and only its backdrop darkens the page. A **tall** sheet (`tall`: Reply, Write a message) reaches almost to the top with 34px corners and pushes the page behind back: smaller, rounder and darker. `SheetCenterHeader` is a tall sheet's title row: ×, the title, and a round blue Send. Rows sit on `SheetGroup` with no box; a `SheetAction` is 46px with a ring or icon, 16px words, red for a danger and a blue tick for the chosen one. Drag a sheet down and it follows the finger; fling it, or drag it far enough, and it closes. A thin line shows under the title row once the content scrolls under it. With the keyboard up a tall sheet keeps its height and takes the keyboard's room inside it (owner, 15 Nov 2026). Dialogs rise as tall sheets on phones.
- **Menus** ("···" and every pop-over): glass menus with 14px corners that grow out of their button. A finger can slide over an open menu: the highlight pill follows it and letting go picks the item (`useFingerMenus`, on in the phone frame). Risky items are red words, last.
- **Segmented control** (`Tabs` on phones): 37px, 13.5px semibold, grey words, the chosen one on a lighter thumb (`--glass-on`). In progress and NorthList Apply on Applications, and the NorthList Apply tabs.
- **Status ring** (`Ring` in `ui/ring.tsx`): see "Status colours". 15px in rows, 12px on a picture, 17px in sheets and to-dos.

**The screens:**

- **Mails** (`PhoneMailsPage`, `PhoneMailRow`): "Mails ⌄" is the big title (it switches to the other two lists); Filter and ··· in one capsule. The ··· menu starts with when mail last came in and what the AI is doing ("Synced 12:00 PM · 3 new mails wait for AI"), which opens the AI reading status in a sheet, then Refresh, which also asks Gmail for new mail. Each row: the person's picture with the application's status ring on its corner, then three lines: the person and how long ago ("2h", "5d", "Oct 14"), the subject, and the company · what the mail asks · when it is due ("Due Mon", red "Past due Nov 10" once late). A replied or done step says what closed it ("You replied Nov 13"). A new mail has a blue dot before the name. Swiping a row to the right does its first action. The filters in use show as small pills under the title.
- **A mail** (`PhoneMailPage`, `PhoneMailParts`, laid out like Linear's issue view): Done (or Reopen) and ··· in one capsule (··· holds the key dates and the application, then Block and Delete). A grey "who · when" line, the subject big and bold, "For ◔ company · role" (opens the application), then the facts as pills in one round card: the status (or the kind of mail while there is no application), when the step is due, the person, the company, and Probably fake. **The status pill opens the Status sheet** and saves the choice at once, as on the application page. Then "Next step" as one plain row (an open ring, what it asks, when it is due, red once late; a tap opens its sheet) with the AI's reading as one grey line under it, then the mails as activity rows (the picture, "who · when", the first four lines). The bottom bar: AI reply, Reply, Open in Gmail.
- **Reply** (a tall sheet): × at the left, the title, a blue round ↑ Send at the right, faded until there is text. Who gets it as pills with pictures, "Re: …" under it, then the text. Above the keyboard one glass tool bar: "AI" (the AI icon), then the text tools. Reply opens on the text; AI reply opens it with the AI's help ready, and "AI" in the tool bar opens and closes that help.
- **Status sheet** (`PhoneStatusSheet`): the seven statuses with their rings, a tick on the chosen one, "Use the AI's status", and who set it ("Set by you · Nov 18") as one grey line under the list. Picking one closes it.
- **Applications** (`ApplicationsTable` with `phonePlain`): the big title beside one capsule (filters, and ··· with Refresh); Search is the tab bar's circle; In progress and NorthList Apply as a segmented control. A plain list, newest first, company first: the status ring and the company, the latest date at the right; the role and place; then the status, the open to-dos and the next step ("1 to do · Pick a time · Due Mon"), with a red "Past due" in place of the count when the next step is late, or the latest news when nothing is open. Closed ones are faded.
- **An application** (laid out like a mail): a grey "Applied … · via …" line, the big company name, the role and place, the facts as pills in one round card (the status pill opens the Status sheet). Then To do (open steps with an empty ring and their due date, then the finished ones with a green tick, struck through), Mails, and Activity (a status change as its ring and "Moved to Interview"; a tap opens a sheet with the AI note and the row's actions). ··· has Reply, Archive or Restore, and Delete. No bottom bar.
- **Today** (the Dashboard tab; laptops still call it Dashboard): the big title "Today", the date under it, refresh as one glass circle. "Needs you" rows with counts (mails to act on, new job mails, interviews coming up, jobs gone quiet), the period's four numbers in one line, and **Charts and rates** one tap deeper (`/dashboard/charts`, phones only): the period picker (Today follows it), the numbers with their change, the running total, the funnel, the jobs gone quiet, the mailbox and what works best. "Gone quiet" opens that page at its list.
- **Settings** (`SettingsTab.tsx`): plain rows, no boxes and no icons: you at the top (name and email, opens Account); Set up: Gmail sync, AI providers, NorthList Apply (and the mail sandbox in dev mode and for test accounts), each with its state as a grey value; This phone: Appearance, Alerts on this phone (a switch) and Install app (only where the app can be installed); Owner › Admin (only for the owner); Sign out in red.
- **Setup pages** (Gmail sync, AI providers, the mail sandbox, the Account page): the small title in the bar and plain rows: the facts first, one blue action (Sync now), choices with a check on the chosen one, and the risky action red and last (Disconnect, Remove, Reset dev data). `SettingsSection` draws its sections this way on phones.
- **Admin** (Settings › Owner › Admin): four numbers in one line, then Needs a look, Accounts, Free AI money and Messages as plain rows. "Message to everyone" is the bottom bar's field and opens **Write a message**: the same tall sheet as Reply (×, the title, a round blue Send), To as a pill (everyone or one account), plain text up to 500 characters, the alert switch off by default, the sent list under it. An account opens its own page (`/admin/accounts/…`), laid out like a mail (grey line, big name, facts as pills); ··· › Suspend opens a floating sheet, and the reason is required. **Free AI money** (`/admin/money`) has today, this month and what is left, and AI calls per day. On laptops both are on the Admin page itself.
- **Message from NorthList** (`OwnerMessages`): a plain card with the blue dot under the page title, at the top of every page until closed; × closes it on all devices.
- **Install card**: a glass card that floats above the tab bar and slides up like a sheet. iPhone: the Share steps and Got it; Android: Not now and Install.
- **Suspended page**: one calm page with the icon, the title, the reason, Ask to resume and Sign out. No tab bar.
- **Toasts** float just above the tab bar.

**Motion** (from the Linear iOS screen recording of 18 Nov 2026, frames in `PHONE-LINEAR-mock/motion-ref/`):

- **Open a page:** it slides in from the right in about 350ms, smooth, with no bounce (`NAV_SPRING`); the page under it drifts a third of the way left, slower, and darkens. Back, a swipe from anywhere on the page and the phone's own back slide it back, following the finger. Each tab keeps its pages and its place.
- **The tab bar becomes the bottom bar:** while a page slides in, the tab icons blur away and the same glass capsule reshapes into the page's field; the side circles fade in (about 250ms, `ease-morph`). Going back reverses it.
- **Switch tabs:** the lighter pill slides to the tapped tab (about 200ms, a little spring) and the glass swells a little under the finger. The page cross-fades with a soft blur (220ms), no slide.
- **Scroll:** the big title scrolls away and the small title fades into the top bar; content blurs under the top and bottom edges.
- **Pull to refresh** (`usePullToRefresh`, every phone page): pull the page down at the top, a small spinner shows in the gap, let go far enough and the page is fetched again (Mails also asks Gmail), then it springs back.
- **Sheets** slide up in about 300ms and settle with a tiny overshoot (`ease-settle`); closing is a bit faster, and a fling closes as fast as it was flung.
- **Menus** grow out of their button's corner and blur in (about 220ms), and shrink back into it faster.
- **Press:** rows go grey while touched; glass buttons swell a little and spring back.
- **Keyboard:** the reply's tool bar sits on top of the keyboard and moves with it.
- **Loading:** the title shows at once; the rest fades in when ready. No spinner in the middle of a page.
- **Reduce Motion:** slides and swells become short fades (mark a part `motion-fade` to keep its fade); the page behind a sheet does not shrink.

## Laptops

768px and up. Owner, 18 and 19 Nov 2026 (approved local mock `PHONE-LINEAR-mock/desktop.html`, "Laptop rules"): **small changes only.** The layout, the side menu, the top bar, the tables and every page stay as they were.

- **Rings, not coloured labels:** the same status rings, grey icons, blue new-mail dot and red only for past due as on the phone.
- **Mails list** (`MailRow`): the person's picture with the status ring on it; the person, then the subject on the same line; one grey line with the ask and its due date ("→ Pick a time · Due Mon", "↩ You replied Nov 13", "Nothing needed"), red only once past due; the company with the status ring and word; the date; the "…" menu when pointed at. A blue dot marks a new mail.
- **A mail** (`LaptopMailHead`): a grey "who · when" line above the title, a "For ◔ company · role" link under it, then the facts as pills in one round card (the status or the kind of mail, the person, the company, Probably fake, how many mails, New, and the way to the mailbox). The side panels are flat grey cards with small grey headings; the next step has a ring before it and its state in grey words under it ("Needs action", "↩ You replied Nov 13", "Done · Your reply answered it"), with no coloured label.
- **The closed reply box:** one round bar, "Draft" with the first words when text is kept, and "AI" beside it on a job mail, which opens the reply with the AI's choices and the hint ready. "R" opens it too; × and Esc close it.
- **Applications:** the status filter is one small segmented control in the toolbar (`LaptopStatusFilter`); the status is a ring and words; the To do column says "1 to do" in plain words with the next step and its due day under it, red only when past due (`LaptopTodoCell`).
- **Admin:** the four numbers in one line between two thin lines (no tiles); Needs a look and Accounts on the left, Messages and Free AI money on the right; each thing to look at has a ring (an empty one for a request to answer, a grey × for a problem), and a resume request's Resume is a blue word. Accounts say Test and Suspended as plain words (Suspended in red), and Gmail and AI in words, red when broken.
- **Message from NorthList** (`LaptopOwnerMessages`): a plain grey card with the blue dot instead of a blue box, at the top of every page; × closes it everywhere.
- **Install card:** bottom right, on the menu glass (`glass-menu`). **Install app** is in the account menu where the app can be installed.
- **Menus and pop-overs** use the phone's glass and grow from their button. **Motion:** a new section cross-fades in; no slides on laptops.

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
- **A row** (`MailRow`, person first since 18 Nov 2026; see "Laptops"): the person's picture with the application's status ring on it; the person, then the subject on the same line (bold and a blue dot when new); one grey line with the next step and its due date (red only when late), a replied step ("↩ You replied Nov 13"), or "Nothing needed"; the company with the status ring and word ("Not linked" only when no company is known), with "Create application" where there is no application; the date; the "⋮" menu, shown when the row is pointed at or has focus. The whole row opens the mail (the subject is a stretched link; the application and menu sit above it). No header row and no Type column.
- **Phones** (replaced on 13 Nov and again on 18 Nov 2026; see "Phones"): the top bar title was "Job mails ▾", a menu that switches lists; the top bar's search button opens a search row under it with Cancel; one row of chips (Needs action, New only, Filters, which opens a sheet with every filter and the sort) with a sync button at its end; rows show initials, sender and date, the subject in grey, the next step (two lines at most) with its due date on its own line, then the status tag with company and role. "Synced today, …" ends the list.
- **Coloured initials** (`Avatar`): six soft colours; the same name always gets the same colour (`avatarTone`). The user's own picture keeps the blue tint.

## Sign-in pages

Owner, 3 Nov 2026 (approved mock `LOGIN-mock/`, issue COM-438); polished on 7 Nov 2026 (approved mock `LOGIN-POLISH-mock/`, issue COM-446): a filled "Sign in", a picture with no visible bottom corner, a little motion, and a blue band on phones and tablets. `SignInLayout` holds both sign-in pages (`LoginPage`, `TestLoginPage`).

- **Laptops (1024px and up):** two halves in a 12px frame, filling the screen. Left: the name at the top (with the dev or test chip), the form in the middle (at most 380px wide), the footer at the bottom ("Free while in beta. No card needed.", Privacy, northlist.site). Right: a flat blue panel (`#0f62fe`, 16px corners) with the product page headline "Your job search, sorted from your Gmail.", its lead sentence, a faint half sun in the corner and a picture of the Mails list (`SignInPicture`, drawn with the app's own parts, no image file). The picture has 12px corners and is always taller than its space, so the panel's right and bottom edges cut it off and no bottom corner is ever seen. No gradient or glow.
- **The picture moves a little, once** (`lib/signInPicture.ts`): 1.8 s after the page opens, a made-up mail grows in at the top on the pale blue tint, its next step shows the app's grey "Reading the mail" shimmer, and 2.2 s later its step and application appear, "4 mails need you" becomes 5 and the tint fades. A second mail (a rejection, "Nothing needed") comes the same way 4.8 s later. Then nothing more moves. With "reduce motion" on, the picture stays as it starts.
- **Google first:** "Continue with Google" in Google's own dark button with the four-colour "G", then "New here? Continuing with Google creates your account.", then "or sign in with email", Email, Password with "Forgot password?" by its label (it opens the advice in a small note), and the filled blue "Sign in" (it was outlined until 7 Nov 2026). In dev mode there is no Google button and "Create test user" follows the password form.
- **Tablets (768px to 1023px):** a short blue band on top (16px corners, about 292px high): the white name, the headline, and a small Mails card (subject and next step only, with the same motion) that runs past the band's right and bottom edges. Then the form (at most 380px wide), then the footer right after the form. The name above the form is shown on laptops only.
- **Phones (under 768px)** (owner, 15 Nov 2026, approved mock `SIGNIN-PHONE-mock/`): one calm screen that fits without scrolling, between the notch and the home bar (safe areas). In the middle, the app icon (an 84px blue tile with the white mark; 72px under 390px wide) and the headline "Your job search, sorted from your Gmail.". At the bottom, where the thumb is: "Continue with Google" (52px, round) as the one main button, "New here? This creates your account.", a grey "Sign in with email" button (52px) and one footer line, "Free while in beta · Privacy · northlist.site". "Sign in with email" opens a sheet with Email and Password (48px fields, 15px labels), "Forgot password?" by the Password label and the blue "Sign in" (50px). No Mails picture, no lead line, and "Sign in to NorthList" is read by screen readers only. A failed Google sign-in shows its note right above the Google button. Dev mode has no Google button: "Sign in with email" and "Create test user", each in its own sheet. The test account page keeps its form on the page, with the same phone-size fields.
- **Light only**, like northlist.site: while a sign-in page is open, `ThemeProvider` shows the light theme (`setLightOnly`); the saved theme stays and comes back after sign-in.

## Charts and numbers

The Dashboard (owner, 27 Oct 2026; reworked 3 Nov 2026) is the only page with charts. The rules:

- **One chart part:** shadcn's `chart` (`components/ui/chart.tsx`, on Recharts), changed for this app: the popup look for its tooltip (6px corners), and the theme class `.theme-dark`. Bars, columns, lines and small trend lines; no pies, no 3D, no animation on load. Every chart has a legend, a hover readout with every series, keyboard reading, and a hidden table for screen readers.
- **Colours** are tokens in `src/index.css`, for light and dark: `--chart-1` blue, `--chart-2` teal, `--chart-3` purple (Carbon's categorical palette), `--chart-4` grey, `--chart-5` red for rejections; `--chart-grid` for grid lines and `--chart-track` for the empty part of a bar. Dark `--chart-2` is `#009d9a`, one step darker than Carbon's, because `#08bdba` fails the colour-blind lightness check on the dark page (3 Nov 2026).
- **Cards** (the Calm look, owner, 3 Nov 2026) are soft grey panels (`surface-subtle`) with no outline, 8px apart, 24px padding (16px on phones), a 14/20 semibold title and an aside on the right (a note or a link); lines inside a card are the page colour. Sections have a 20/28 semibold title (18/24 on phones). The main number of a card is 32/40 light (28/36 on phones).
- **Small numbers:** a rate shows only from 5 applications; below that, "1 of 3". "Too few" stands in a table cell.
- **Every count leads somewhere:** to the list behind it (Applications with its step, Mails with its filter, Automation).
- **Layout** follows the page's own width (container queries), so it fits with the side menu open or closed. Phones (owner, 18 Nov 2026): the Dashboard tab is **Today**: "Needs you" rows with counts, the period's four numbers in one line, and every chart one tap deeper on **Charts and rates** (`/dashboard/charts`), which also holds the period picker. See "Phones".

## Motion

Carbon's productive curves, as tokens: `ease-standard` (0.2, 0, 0.38, 0.9), `ease-entrance` (0, 0, 0.38, 0.9), `ease-exit` (0.2, 0, 1, 0.9). Since 18 Nov 2026 also `ease-settle` (0.3, 1.18, 0.5, 1: a sheet that settles with a tiny overshoot) and `ease-morph` (0.32, 0.72, 0, 1: a glass shape that changes, no bounce). The phone's Linear motion is listed under "Phones".

| Time | Use |
| --- | --- |
| 70ms | hover and pressed colours |
| 110ms | dropdowns and menus close; tooltips open |
| 150ms | a page fades in (`animate-page-in`) |
| 160ms | menus and dropdowns close |
| 240ms | dialogs come and go; every change of size (below) |
| 220ms | menus, dropdowns and pop-overs open with a small spring (`ease-spring`), growing out of their button's corner and blurring in (18 Nov 2026; 320ms before) |
| 420ms | toasts slide in with the same spring |
| 300ms | phone sheets rise and settle (`ease-settle`, 18 Nov 2026; 450ms before); closing is faster |
| 70ms, then 200ms | a button is pressed in a little (`scale-[0.96]`) and springs back (15 Nov 2026) |
| 200 to 260ms | a filter pill pops in with a small spring and shrinks away when removed |
| 280ms | a list's row that leaves shrinks away where it was; a new row grows in |
| 380ms | a step marked done: the line draws across its words |
| 520ms | an empty list's picture springs in; its words rise in after it |

**Phone pages move on springs** (owner, 13 Nov 2026; timing from the Linear iOS recording, 18 Nov 2026): the slide between pages (about 350ms, no bounce) and the tab bar's pill (about 200ms, a little spring) use the small springs in `lib/spring.ts` (no animation library), so a fling keeps its speed. Tabs cross-fade with a soft blur; laptops cross-fade a new section and never slide. With less motion asked for, they are a short fade.

**Size changes are never a jump** (owner, 21 Oct 2026; approved demo "Motion: today vs new"). Three shared parts do it, so a page never builds its own:

- `Collapse`: a section the user shows and hides (an older mail, the AI's reading, done steps). It grows from no height to its own and fades in, and leaves the same way.
- `Reveal`: something that appears by itself, such as an error under a field. It grows and fades in. `InlineNotification` already uses it.
- `AutoHeight`: a box that follows its content's height when the content jumps (new results, the next step). Every dialog body and the Combobox list already sit in one. A part inside that animates its own height is followed as it moves, so the two never fight.

**Small changes move too** (owner, 15 Nov 2026; approved preview, pictures in `EMPTY-STATES-mock/`). Nothing on screen changes state with a jump:

- Lists: `useListMotion` (`components/ui/useListMotion.ts`) keeps a row that leaves where it was while it shrinks away (for example a mail marked done under "Needs action"), and grows a new row in. It compares rows by their keys. A new filter or page is drawn at once, and so is the last list while the next one loads. The Job mails list (phone and laptop) and Applications (`DataTable` with `motionKey`) use it. Filter pills use it too, so they pop in and shrink away.
- The phone search row slides open and closed (`Collapse`). Cancel and the filter pills dim while pressed.
- The new-mail dot shrinks away once the mail is read. The phone refresh icon finishes its turn instead of snapping back, and also turns while the page checks for new data over a saved copy. The "Job mails ⌄" arrow turns while its menu is open. Copy shows a tick, then resets after 2 seconds. Dashboard numbers rise in, and rise again when they change.

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
| `OverflowMenu` (`menu.tsx`) | the "…" menu for actions that are not the main one; a glass menu (`POPUP_CLASS`, `ITEM_CLASS` in `select.tsx`) that grows out of its button, the highlight a soft pill; danger actions last, in red words |
| `DataTable` | every list, with no outer frame: a header row of grey medium labels (the first label with the row count) over a darker line, rows split by thin lines, underlined names; sorting, filtering and paging stay on the server; skeleton rows while loading; on phones `phoneRow` draws each row and `phonePlain` puts the rows edge to edge with no grey group |
| `Toolbar` | above a table: a bordered search that takes the free width, then each filter as a bordered button ("Action: All"); on phones the filters open in a sheet; `ToolbarSearch` takes an `id` for the phone top bar's search |
| `Tabs` | switching between views of one list: segmented, in a bordered group, each with its count ("Job mails (5)"); on phones an iOS segmented control (37px, 13.5px semibold); `phoneLook="chips"` makes a long set a row of chips on phones |
| `Pagination` | under a list, with no frame: the page, and two small bordered arrow buttons |
| `Ring`, `StatusRing`, `StatusBadge` (`ring.tsx`) | an application status: the ring alone (`Ring kind size label`), a ring with words (`StatusRing look`), or a status as ring and words (`StatusBadge status`); kinds from `lib/statusTags.ts` only (18 Nov 2026) |
| `Tag` | small plain marks that are not a status (Archived, New, Asks you, Needs action) and counts; never an application status |
| `InlineNotification` | a notice inside a page: info, success, warning, error |
| `SectionError` | a part of the page that did not load: plain words and "Try again", never the raw error |
| `EmptyState` | "nothing here", centred: a thin icon in a soft circle, a short title, why, and the one next step (`EMPTY_ACTION`, or `EMPTY_ACTION_QUIET` for "Clear filters") |
| `Skeleton`, `SkeletonList`, `SkeletonRows` | loading: the only place that draws grey loader shapes (a test checks); `SkeletonRows` is shaped like mail rows; each takes a list's side space by itself (see "Loading, empty and error states") |
| `Dialog` | a confirm or a short form: title and description at the top, the buttons at the bottom right (secondary, then the main one) under a line; its body follows its content's height |
| `Collapse` | a section the user shows and hides; the toggle keeps `aria-expanded` |
| `Reveal` | something that appears by itself, such as a field error |
| `AutoHeight` | a box whose content jumps in height (results, steps) |
| `showToast` (`toast.tsx`) | a short confirmation after an action, with Undo where the app can really undo |
| `CodeSnippet` | one line to copy, with a copy button |
| `Avatar` | initials of a person, in a circle; the user ("You") on the blue tint |
| `Sheet`, `SheetGroup`, `SheetAction`, `SheetCenterHeader`, `ActionSheet` (`sheet.tsx`) | phones: a floating sheet (inset, 38px corners, glass, × at the left), or `tall` (pushes the page back); plain rows edge to edge, 46px; a tall sheet's title row with Send; a list of actions |
| `useFingerMenus` | phones: a finger slides over an open menu and lifts to pick (on in the phone frame) |
| `PlainSection`, `PlainRow`, `PlainLinkRow` (`components/phone/PlainList.tsx`) | phones: plain rows like Linear's Settings (see "Phones") |
| `PhoneBottomBar`, `BarCircle`, `BarField` (`components/phone/BottomBar.tsx`) | phones: a detail page's bottom bar: glass circle, field, glass circle |
| `GlassButton`, `GlassCapsule`, `CapsuleButton`, `TopBarActions` (`components/phone/`) | phones: the round glass buttons and the capsule of icons in the top row |
| `Switch` | an on/off setting, as phone settings have |
| `RefreshButton`, `useRefresh` | fetching a list page again without reloading the app |
| `AiIcon` (`ai-icon.tsx`) | the AI's icon |

## Status colours

**Status rings** (owner, 18 Nov 2026, mocks `PHONE-LINEAR-mock/`): one small ring in place of a coloured status tag, on phones and laptops. One map, in `src/lib/statusTags.ts` (`STATUS_RING`; `statusTag()` and `applicationTag()` give `{ label, ring }`), drawn by `Ring` in `ui/ring.tsx`. The ring fills as the application gets further:

| Status | Ring | Colour token |
| --- | --- | --- |
| Applied, Status unknown, No reply yet (NorthList Apply, no status) | empty | `text-ring-grey` |
| Recruiter contact | a quarter | `text-ring-recruiter` (blue) |
| Assessment | half | `text-ring-assessment` (yellow) |
| Interview | three quarters | `text-ring-interview` (orange) |
| Offer | full, with a check | `text-ring-offer` (green) |
| Rejected, Closed | a grey × | `text-ring-grey` |

A to-do has its own empty ring (`kind="todo"`, the second text colour). Sizes: 15px in phone rows, 12px on a picture, 17px in sheets and to-dos, 14px on laptops. The ring is decorative next to its words; alone it takes a `label`.

**Colour only means something:** the rings, the blue new-mail dot, red for past due (`text-late`: `#d93f3f` Light, `#f0625a` Dark) and green switches. Everything else is grey or the text colour.

The coloured status tags of 3 Nov 2026 (gray, cyan, teal, blue, green, red, outline, with a dot) are replaced by the rings. `Tag` (pills, 22px high, 12px medium text, a pale fill, a thin outline and a small dot) stays only for small marks that are not a status, such as Archived, New, Asks you and Needs action.

## Loading, empty and error states

- Opening the app: until it knows who is signed in, the start-up screen (owner, 19 Nov 2026, mock `APP-LOADER-mock/`, direction A, a local file). Only the mark, 56px, on the page colour: in the middle on laptops, 40px above the middle on phones. It never shows the app's shape. The mark is drawn by `index.html` itself, so it is the first thing on screen, in the saved theme (a dark-mode user never sees a white page), and it keeps moving without a restart when the app's code arrives; `components/startup/StartupScreen.tsx` puts the words under it and makes it still (`lib/startupMark.ts`). Motion: nothing for 250ms (a fast start shows nothing); then the half sun rises over the horizon (640ms), the three list lines draw in from the left one after another, then breathe (fade to 28% and back, 1.6s) one after another while it waits. This is the only loader on the screen. After 5 seconds it says "Waking up the server" in the grey text shimmer, that it takes about a minute, and the time waited; after 2 minutes, or with no network, the mark stops and turns grey, and the screen says what happened and offers "Try again" (full width on phones). When the app is ready the mark fades and shrinks a little over the app (180ms). Reduce Motion: no rise or draw; the lines only fade slowly. With no network but a copy saved on this device, the app opens at once, read-only (see "Offline and alerts").
- Loading: skeletons shaped like the content, never a bare "Loading…".
- Loaders and side space (owner, 17 Nov 2026; approved local mock `PHONE-LOADERS-mock/`): a list that runs to the phone's edges uses `phone-bleed` (`src/index.css`), which keeps 20px (`--list-inset`) for its rows and loaders. Every loader comes from `ui/skeleton.tsx` and takes that space by itself, so no loader touches the edge; inside a padded group (`DataTable` on phones) it lines up with the rows' words.
- A saved copy first, as phone mail apps do (owner, 17 Nov 2026): a page shows the copy saved on this device at once (see "Offline and alerts") and fetches new data quietly; the phone ↻ turns while it checks over a copy more than a minute old. The loader shows only when there is nothing to show yet: the first visit, after 7 days or an app update, and the Whole inbox and mail text, which are read live and never saved.
- Empty: `EmptyState` says why and offers the next step (owner, 15 Nov 2026: drawn as a phone draws it, centred, with a thin icon that names the list). The step depends on the state: "Connect Gmail" while Gmail is not connected; "Clear filters" when a search or filter finds nothing, with the search words in the title. Counts and filters that would only show zeros wait until there is something to count (Applications › NorthList Apply hides "0 of 0 replied" and its filters until something is sent).
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
