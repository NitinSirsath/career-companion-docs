# AI UI Implementation Rules

When you change the Career Companion frontend as an AI coding agent, you **must** follow these rules. They come from design part 1, which the owner approved on 17 Oct 2026 (plan and mock: https://claude.ai/artifact/VWAgcutaXtATYMXhRXc4Mx). The look is IBM Carbon; [DESIGN_SYSTEM.md](DESIGN_SYSTEM.md) has the themes, type, motion and the list of shared parts.

## Owner decisions (17 and 18 Oct 2026)

- Owner decision, 17 Oct 2026: great UX and a clear, working UI come first; looks come second. The look stays IBM Carbon (square corners, IBM Plex, quiet greys, one blue). Behaviour and motion are modern.
- Owner decision, 17 Oct 2026: pages use the full width up to 1584px and are centred only beyond that.
- Owner decision, 17 Oct 2026: one page header and one breadcrumb style on every page.
- Owner decision, 17 Oct 2026: the mail page is a chat-like conversation. Each mail shows only its new text, the quoted history is hidden behind "···", and the AI's reading sits under the mail it read. A reply sent from the app still quotes the earlier mail, as normal mail does; only the app hides it.
- Owner decision, 17 Oct 2026: in a conversation, mails are full width; the user's own mails are tinted and set in from the left. No narrow chat bubbles.
- Owner decision, 17 Oct 2026: TanStack Table for every table; Base UI Select and Combobox for every dropdown; Carbon icons; simple CSS motion built into the shared parts.
- Owner decision, 17 Oct 2026: themes are System, Light (Carbon Gray 10) and Dark (Carbon Gray 100).
- Owner decision, 18 Oct 2026: a fourth theme, White (Carbon White: white page, grey cards), as on ibm.com.
- Owner decision, 17 Oct 2026: the menu can collapse to icons, and the app remembers the choice. Agenda stays out of the menu until it works.
- Owner decision, 18 Oct 2026: the Dashboard is not part of design part 1; it stays as it is and is a [Later] issue.

## Do this

1. **Read first.** Read `DESIGN_SYSTEM.md` and the shared parts in `src/components/ui/` before you change a screen.
2. **Reuse the shared parts.** A list is a `DataTable`; a dropdown is `Select` or `Combobox`; a "…" menu is `OverflowMenu`; a notice is `InlineNotification`; a failed part is `SectionError`; a confirmation is a toast. If a part is missing, add a reusable one to `src/components/ui/` (on `@base-ui/react`), not a one-off in a page.
3. **Use tokens only.** Colours, layers and borders come from the tokens in `src/index.css` (`bg-surface`, `text-text-secondary`, `border-border-subtle`). No hex values and no raw Tailwind colours.
4. **One main button.** One primary (blue) button per page or panel. Everything else is secondary, tertiary, ghost or in a "…" menu. Danger actions sit last in the menu, in red, and ask before they act.
5. **Colour means something.** Colour only for status and for "needs your attention". Status tags come from `src/lib/statusTags.ts` only. The same thing looks the same everywhere.
6. **Human words.** On screen, never system words such as "Email Processed", "PENDING", "COMPLETED" or "RELEVANT", and never a raw server error. Labels, states and dates are tested functions in `src/lib/`. Sentence case.
7. **Every state.** Loading shows a skeleton; empty says why and offers the next step; an error says what failed in plain words and offers "Try again". One failed part never hides the rest of the page.
8. **Safe saves.** A save that may or may not have happened is read back from the server and never sent again by itself. Saves use the revision the page shows, so a change made elsewhere is never overwritten.
9. **Lists keep their place.** Search, filters, sort and page live in the URL, so a reload, Back and the breadcrumb return to the same view.
10. **Phones.** No sideways scroll at 375px, a 16px side gutter, stacked table rows, the bottom navigation.
11. **Accessibility.** Visible focus on every control; an `aria-label` on every icon-only button; every field has a label; check the contrast of new colours; check that Base UI parts are wired correctly (labels, names), do not assume it.

## Not allowed

- Lucide icons or a sparkle icon for AI (use Carbon icons; the AI is the machine-learning model icon).
- A pure white page in the Light theme (its page is `#f4f4f4` with white cards). Only the White theme has a white page, and its cards are grey.
- Em dashes in on-screen text.
- Shadows, and rounded corners (true circles such as status dots and avatars are fine).
- Animation on hover only, bouncy or endless animation, and motion longer than 400ms. Motion uses Carbon times (70 to 400ms) and curves, in CSS only, and turns off when the computer asks for reduced motion.
- "Loading…" text where a skeleton fits.
- The browser's own `<select>`.
- A new page or dialog pattern that differs from the existing ones without the owner's OK.
- PR titles, PR descriptions, branch names and commit messages that name the design part 1 parent issue; name only the PR's own issue.
