---
name: saas-sidebar-redesign
description: Redesigns a Vue 3 + vue-router app into a modern SaaS interface with a collapsible left sidebar instead of a top nav bar, design tokens, consistent spacing, and a polished professional look. Use when asked to redesign the UI, modernize the look, move navigation to a sidebar, replace the top nav, or normalize spacing and styling in a Vue app. This is an explicit request for the SaaS look, so it overrides frontend-design's advice to avoid SaaS-template layouts.
---

# SaaS Sidebar Redesign

Restyles the structure of a Vue 3 app (shell, navigation, spacing, cards) while preserving its palette, fonts, data flow and behavior. Work through the phases in order and do not skip Phase 0.

Scaffolds and grep recipes are in [reference.md](reference.md).

## Principles

- **Preserve identity, change structure.** Derive tokens from the colors and fonts the app already uses. Never swap in a new palette.
- **Behavior is untouched.** No changes to API calls, data handling, routes, i18n keys, auth or event wiring.
- **Report first, then edit.** Phase 0 ends with a short findings report and change list.
- **Small steps, build between them.** Run the build after Phases 1, 2 and 3.

## Phase 0 - Recon (read-only)

Locate and record:

1. Router file and the route-to-view mapping.
2. Root component: where the nav lives, what else is in the header (language switcher, profile menu, filters), which events and refs the header wires up.
3. Nav labels and how they are produced (i18n keys vs literals) and how the active link is determined.
4. Global styles and any per-view redefinitions of shared classes (page header, card, stat grid, table).
5. Anything positioned relative to header height: `position: sticky | fixed`, `top: <n>px`, `calc(100vh - ...)`.
6. Dropdowns and popovers anchored to the header (`position: absolute` with `right: 0`).
7. Design rules in project docs (palette, "no emojis", icon or chart conventions).
8. Counts of distinct hex colors, padding, gap, radius and shadow values (recipes in reference.md).
9. The project's style API: `<script setup>` vs `setup()`, scoped vs global styles.

Report: what was found, what will change, and anything out of scope that was spotted (see Guardrails). Then continue.

## Phase 1 - Design tokens

Create a tokens stylesheet (e.g. `src/styles/tokens.css`) and import it once in the entry file.

- Colors: name the app's own most-used values (text, muted text, border, surface, background, primary, primary tint, success, warning, danger). Keep the exact hex values already in use.
- Spacing: 4px base scale (`--space-1` 4px through `--space-8` 48px).
- Radius: three steps (small, medium, large) plus pill.
- Shadow: three steps (card, raised, overlay).
- Type: size scale and weight tokens. Keep the existing font family.
- Layout: `--sidebar-width`, `--sidebar-width-collapsed`, `--content-max-width`, `--page-padding`, `--header-height` (0 if there is no slim header).

Replace only the highest-frequency literals mechanically. Leave one-offs for Phase 3. Run the build.

## Phase 2 - Shell with sidebar

Add a sidebar component and change the root component to a two-column layout (sidebar, main).

**Navigation**
- Build items from the existing nav, preserving its labels, i18n keys and order. Do not generate them from raw route order.
- Use `router-link` active state and `aria-current="page"` instead of hand-written path comparisons.
- Each item gets an inline SVG icon. No emojis and no icon-font or icon-library dependency.
- **Active item style is a full background tint on the whole item (pill or rounded rectangle). Never use a single-edge colored stripe or border** (left, top or bottom) on the active item or on any card, panel or row.

**Header contents**
- App title goes at the top of the sidebar.
- Move language switcher and profile menu to the sidebar footer or a slim content header.
- Re-anchor their dropdowns so they open upward or sideways and are not clipped by the sidebar edge.
- Keep every existing event, ref and composable wiring exactly as it was.

**Offsets**
- Replace every hardcoded header-height offset found in Phase 0 (sticky filter bars, `calc(100vh - ...)`) with tokens.
- Filters or toolbars that were sticky under the top nav stay sticky at the top of the content area.

Run the build, then do a quick visual check of one route.

## Phase 3 - Normalize

- Move duplicated shared classes (page header, card, card header, stat grid, stat card, table) into one shared stylesheet driven by tokens. Delete the per-view overrides that only restate them. Keep view-specific styling that is genuinely different.
- Snap padding, gap, radius and shadow values to the token scale. Where a value falls between steps, pick the nearest and note it.
- Give modals a small set of width tokens (e.g. small, medium, large) instead of per-file pixel values.
- Page content uses one container: consistent max width, page padding and vertical rhythm between sections.
- Off-palette one-offs (a stray accent color used in a few places) are reported, not silently replaced, unless they are plainly a mistake.

Run the build.

## Phase 4 - Responsive and accessibility

- **Collapse:** a toggle shrinks the sidebar to an icon rail. Labels become tooltips (`title` plus visually hidden text). Persist the choice in `localStorage` inside try/catch, because sandboxed hosts block storage.
- **Mobile:** below about 768px the sidebar becomes an off-canvas drawer with a scrim, a menu button in a slim top bar, Esc and scrim click to close, focus moved into the drawer on open and restored on close.
- **A11y:** `<nav aria-label>` landmark, visible `:focus-visible` rings, toggle buttons expose `aria-expanded`, sufficient contrast for muted text, respect `prefers-reduced-motion`.
- Content grids (`minmax()` auto-fit) and tables reflow or scroll horizontally on narrow screens.

## Phase 5 - Verify

Prove the work; do not assume it.

1. Build the app with the project's build script. This is a compile gate only.
2. Start the dev servers and drive every route in a real browser (Playwright MCP if available). Capture screenshots at desktop expanded, desktop collapsed and mobile widths. Check the console for errors.
3. Exercise: open and close each modal, both header dropdowns from their new location, every filter, and the language switch if the app has one.
4. Compare against any pre-change screenshot.
5. Run a code review on the diff.
6. If no browser tooling is available, say so plainly and report only what the build proved.

Report results per route, and list anything not verified.

## Guardrails

- **Delegation:** if the project requires a Vue specialist agent for `.vue` work (e.g. `vue-expert`), delegate significant `.vue` edits to it.
- **Style API:** match each component's existing API (`<script setup>` or `setup()`); never mix them within one component.
- **Dependencies:** add none. Do not touch registry config or lockfiles, and do not commit credentials, internal hostnames or private registry URLs.
- **Scope:** report, do not fix, things noticed on the way: untranslated labels, nav order differing from route order, views with no route, hardcoded API base URLs.
- **Docs:** if project docs say where styles live and that has changed, update that line.
- **Comments:** comment non-obvious logic (why a value was snapped, why a dropdown opens upward).
- **Visual rules from project docs win** over anything here (palette, "no emojis", chart style).
- **Git:** work on a feature branch. Commit each verified phase. Never push or merge without the user's direct word.
