# AI UI Implementation Rules

When acting as an AI coding agent modifying the Career Companion frontend, you **MUST** follow these strict rules. The **IBM Carbon design system is the ultimate authority** for this project.

## Positive Rules (Do This)

1. **Read Relevant Documentation:** Always review `DESIGN_SYSTEM.md` and existing component files before starting UI work.
2. **Inspect Existing Components:** Do not reinvent the wheel. Check `src/components/ui/` for existing primitives (e.g., `Button`, `Badge`, `Input`).
3. **Reuse Existing Tokens:** Always use semantic Tailwind variables (e.g., `bg-surface-raised`, `border-border-default`, `text-text-secondary`, `bg-status-success-subtle`).
4. **Identify Gaps Before Implementing:** If a component truly doesn't exist, build a generic, reusable version in `src/components/ui/` using `@base-ui/react` rather than adding inline hacks.
5. **Verify Responsive Behavior:** Mobile is not a smaller desktop. Ensure stacked layouts and use the bottom navigation paradigm.
6. **Implement Strict Accessibility Checks:**
   - Keep visible `focus-visible:ring-2 focus-visible:ring-border-focus` states on interactive elements.
   - Use `cursor-pointer` explicitly on custom buttons for consistent interaction styling (UX convention).
   - Check icon-only buttons for `aria-label`.
   - Verify every form label is correctly associated with its input (`id` and `htmlFor`).
   - Do NOT assume contrast passes WCAG just because semantic tokens are used — **verify the colors**.
   - Do NOT assume `@base-ui/react` automatically makes every custom component accessible; implementation and wiring still matter.

## Negative Constraints & Anti-Vibe-Code (Don't Do This)

These constraints are focused strictly on visual/UI patterns to prevent common "AI drift" away from the IBM Carbon design system.

1. **No Arbitrary Visual Values (Tailwind Drift):** Do NOT invent arbitrary HEX colors, raw tailwind colors (`bg-blue-100`, `text-gray-400`), or heavy custom `box-shadow` styles. Components consume semantic design tokens only.
2. **No Rounded Corners:** The border radius must always be `0px` (`rounded-none`). Do not add `rounded-md`, `rounded-full` (except for specific badges/avatars explicitly defined), or `rounded-lg`.
3. **No Excessive Animations:** Do not add bouncy, springy, or continuous animations. Animations should be subtle and professional (e.g., `transition-colors`, simple opacity fades). 
4. **No Ad-hoc Spacing:** Stick strictly to Tailwind's 4px grid spacing (`p-2`, `m-4`, `gap-4`). Do not invent custom pixel padding values unless absolutely necessary for sub-pixel alignment.
5. **No Layout Inconsistency:** Do not create a new page or modal pattern without verifying how similar layouts are handled in existing pages.
