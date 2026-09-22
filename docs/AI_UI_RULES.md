# AI UI Implementation Rules

When acting as an AI coding agent modifying the Career Companion frontend, you **MUST** follow these 10 rules strictly. 

1. **Read Relevant Documentation:** Always review `DESIGN_SYSTEM.md` and existing component files before starting UI work.
2. **Inspect Existing Components:** Do not reinvent the wheel. Check `src/components/ui/` for existing primitives (e.g., `Button`, `Badge`, `Input`).
3. **Reuse Existing Tokens:** Always use semantic Tailwind variables (e.g., `bg-surface-1`, `border-border`, `text-muted-foreground`).
4. **No Arbitrary Visual Values:** Do not invent arbitrary HEX colors, border radii (must be `0px`), or heavy custom `box-shadow` styles.
5. **Check Existing Patterns:** Do not create a new page or modal pattern without verifying how similar layouts are handled (e.g., check `applications.tsx` or `index.tsx`).
6. **Identify Gaps Before Implementing:** If a component truly doesn't exist, document it and build a generic, reusable version in `src/components/ui/` rather than adding inline hacks.
7. **Document Additions:** If you add a new primitive or layout pattern, update the documentation.
8. **Verify Responsive Behavior:** Mobile is not a smaller desktop. Ensure stacked layouts and use the bottom navigation paradigm.
9. **Verify Accessibility:** Ensure semantic HTML, visible keyboard focus (`focus-visible:ring-2`), and text-based status indications (color must not be the only signal).
10. **Verify Reduced Motion:** Animations should be subtle (e.g., `transition-colors`, simple opacity fades). Do not add bouncy or continuous animations.
