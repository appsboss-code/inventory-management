# saas-sidebar-redesign - Reference

Scaffolds to adapt, not paste blindly. Replace placeholder values with the app's own colors and fonts found in Phase 0.

## Recon recipes

Run from the client source directory (e.g. `client/src`).

```bash
# Most-used hex colors
grep -rhoE '#[0-9a-fA-F]{6}\b' . | tr 'A-F' 'a-f' | sort | uniq -c | sort -rn | head -15

# Distinct values per property (a high count means no scale)
for p in padding gap border-radius box-shadow; do
  echo "$p: $(grep -rhoE "^\s*$p:[^;]+;" . | sed 's/^\s*//' | sort -u | wc -l)"
done

# Header-height coupling
grep -rnE 'top:\s*[0-9]+px|calc\(100vh|position:\s*(sticky|fixed)' .

# Header-anchored dropdowns
grep -rnE 'position:\s*absolute' . | grep -E 'right|left'

# Existing CSS variables and breakpoints
grep -rnE 'var\(--|@media' . | wc -l

# Shared class redefinitions across views
grep -rnE '^\s*\.(page-header|card|card-header|card-title|stats-grid|stat-card)\b' .
```

## tokens.css

```css
:root {
  /* Colors: use the values the app already uses; names below are roles. */
  --color-text: #0f172a;
  --color-text-muted: #64748b;
  --color-border: #e2e8f0;
  --color-surface: #ffffff;
  --color-bg: #f8fafc;
  --color-primary: #2563eb;
  --color-primary-tint: #eff6ff;
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-danger: #dc2626;

  /* Spacing: 4px base */
  --space-1: 4px;  --space-2: 8px;  --space-3: 12px; --space-4: 16px;
  --space-5: 20px; --space-6: 24px; --space-7: 32px; --space-8: 48px;

  --radius-sm: 6px; --radius-md: 10px; --radius-lg: 14px; --radius-pill: 9999px;

  --shadow-card: 0 1px 2px rgba(15, 23, 42, 0.06), 0 1px 3px rgba(15, 23, 42, 0.08);
  --shadow-raised: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-overlay: 0 20px 50px rgba(15, 23, 42, 0.15);

  --font-size-xs: 0.75rem; --font-size-sm: 0.875rem; --font-size-md: 1rem;
  --font-size-lg: 1.25rem; --font-size-xl: 1.875rem;

  --sidebar-width: 240px;
  --sidebar-width-collapsed: 64px;
  --content-max-width: 1600px;
  --page-padding: var(--space-6);
  --header-height: 0px; /* becomes a real value only on mobile, where a slim top bar exists */

  --modal-sm: 480px; --modal-md: 640px; --modal-lg: 900px;
}

@media (max-width: 768px) {
  :root { --header-height: 56px; --page-padding: var(--space-4); }
}
```

## Root layout

```vue
<template>
  <div class="app-shell" :class="{ collapsed }">
    <AppSidebar v-model:collapsed="collapsed" v-model:drawer-open="drawerOpen"
                @show-profile-details="$emit('show-profile-details')" />
    <div class="app-main">
      <header class="mobile-bar"> <!-- visible below 768px only -->
        <button class="icon-btn" :aria-expanded="drawerOpen" aria-label="Open navigation"
                @click="drawerOpen = true">
          <svg viewBox="0 0 24 24" width="20" height="20" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16" stroke="currentColor" stroke-width="2" stroke-linecap="round" fill="none"/>
          </svg>
        </button>
      </header>
      <slot name="toolbar" /> <!-- sticky filters etc. -->
      <main class="app-content"><router-view /></main>
    </div>
  </div>
</template>

<style>
.app-shell { display: grid; grid-template-columns: var(--sidebar-width) 1fr; min-height: 100vh; background: var(--color-bg); }
.app-shell.collapsed { grid-template-columns: var(--sidebar-width-collapsed) 1fr; }
.app-main { min-width: 0; } /* lets wide tables scroll instead of stretching the grid */
.app-content { max-width: var(--content-max-width); padding: var(--page-padding); margin: 0 auto; }
.mobile-bar { display: none; }
@media (max-width: 768px) {
  .app-shell, .app-shell.collapsed { grid-template-columns: 1fr; }
  .mobile-bar { display: flex; align-items: center; height: var(--header-height); padding: 0 var(--space-4);
                background: var(--color-surface); border-bottom: 1px solid var(--color-border); }
}
@media (prefers-reduced-motion: reduce) { .app-shell * { transition: none !important; } }
</style>
```

Adapt to the project's style API. If the root uses `setup()`, convert only what is needed and keep it consistent within the component.

## AppSidebar.vue skeleton

```vue
<template>
  <aside class="sidebar" :class="{ collapsed, open: drawerOpen }">
    <div class="sidebar-top">
      <span class="brand">{{ t('nav.companyName') }}</span>
      <button class="icon-btn collapse-btn" :aria-expanded="!collapsed"
              aria-label="Toggle sidebar width" @click="$emit('update:collapsed', !collapsed)">
        <!-- chevron svg -->
      </button>
    </div>

    <nav aria-label="Primary" class="sidebar-nav">
      <router-link v-for="item in items" :key="item.to" :to="item.to" class="nav-item"
                   :title="collapsed ? item.label : undefined">
        <svg class="nav-icon" viewBox="0 0 24 24" aria-hidden="true"><path :d="item.icon" /></svg>
        <span class="nav-label">{{ item.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <!-- existing LanguageSwitcher and ProfileMenu go here; re-anchor their dropdowns to open upward -->
    </div>
  </aside>
  <div v-if="drawerOpen" class="scrim" @click="$emit('update:drawerOpen', false)" />
</template>

<style scoped>
.sidebar { position: sticky; top: 0; height: 100vh; display: flex; flex-direction: column;
           background: var(--color-surface); border-right: 1px solid var(--color-border);
           padding: var(--space-4) var(--space-3); transition: width .2s ease; overflow-y: auto; }
.nav-item { display: flex; align-items: center; gap: var(--space-3); padding: var(--space-2) var(--space-3);
            border-radius: var(--radius-sm); color: var(--color-text-muted); text-decoration: none; }
.nav-item:hover { background: var(--color-bg); color: var(--color-text); }
/* Active state tints the whole item. A one-sided accent stripe is deliberately not used. */
.nav-item.router-link-exact-active { background: var(--color-primary-tint); color: var(--color-primary); font-weight: 600; }
.nav-item:focus-visible, .icon-btn:focus-visible { outline: 2px solid var(--color-primary); outline-offset: 2px; }
.sidebar.collapsed .nav-label, .sidebar.collapsed .brand { display: none; }
@media (max-width: 768px) {
  .sidebar { position: fixed; inset: 0 auto 0 0; width: var(--sidebar-width); z-index: 40;
             transform: translateX(-100%); transition: transform .2s ease; }
  .sidebar.open { transform: translateX(0); }
  .scrim { position: fixed; inset: 0; background: rgba(15, 23, 42, .4); z-index: 30; }
}
</style>
```

Notes:
- Build `items` from the existing nav in its original order. Use `router-link-exact-active` for the root path (`/`) so it is not active on every route.
- Icons: simple 24px stroke paths drawn inline; keep one stroke width and style across all of them.
- Drawer behavior in script: on open, move focus to the first nav item; on `Escape`, close and return focus to the menu button; close on route change.

## Persisted collapse state

```js
const KEY = 'sidebar-collapsed'
const read = () => { try { return localStorage.getItem(KEY) === '1' } catch { return false } }
const write = (v) => { try { localStorage.setItem(KEY, v ? '1' : '0') } catch { /* storage blocked: state is per-session only */ } }
```

## Dropdowns that moved to the sidebar footer

```css
/* Open upward and inside the sidebar width so the menu is not clipped by the viewport edge or overflow. */
.dropdown-menu { position: absolute; bottom: calc(100% + var(--space-2)); left: 0; right: 0; }
.sidebar.collapsed .dropdown-menu { left: calc(100% + var(--space-2)); right: auto; bottom: 0; min-width: 200px; }
```

The sidebar uses `overflow-y: auto`, which clips absolutely positioned children. If a menu is clipped, either render it with `<Teleport to="body">` or set the footer's menu `position: fixed` using the trigger's bounding rect.

## Phase 5 checklist

| Check | Pass condition |
|---|---|
| Build | Exit 0, no SFC compile errors |
| Routes | Every route renders, active item highlighted, console clean |
| Widths | 1440 expanded, 1440 collapsed, 390 mobile drawer |
| Header features | Both dropdowns open fully visible, modals open and close |
| Filters | Toolbar sticky at content top, not overlapped |
| Locale | Switching language relabels nav without layout break |
| Tokens | Distinct padding/radius/shadow counts drop sharply; no new off-palette hex |
| Rules | No emojis in UI; no single-edge color accents |
