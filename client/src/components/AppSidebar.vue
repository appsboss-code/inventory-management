<template>
  <!--
    .sidebar-slot is the grid cell. It keeps its width (64px in tablet mode) even when the
    aside inside it goes position:fixed to overlay the page, so content never reflows.
  -->
  <div class="sidebar-slot" :class="`mode-${mode}`">
    <aside
      ref="asideEl"
      class="sidebar"
      :class="[`mode-${mode}`, { railed, open }]"
      :inert="mode === 'mobile' && !open ? '' : undefined"
      @keydown="trapTab"
    >
      <div class="sidebar-top">
        <div class="brand">
          <span class="brand-mark" aria-hidden="true">
            <svg viewBox="0 0 24 24" class="icon">
              <path d="M21 8l-9-5-9 5v8l9 5 9-5V8z" />
              <path d="M3 8l9 5 9-5M12 13v8" />
            </svg>
          </span>
          <span class="brand-text">
            <span class="brand-name">{{ t('nav.companyName') }}</span>
            <span class="brand-sub">{{ t('nav.subtitle') }}</span>
          </span>
        </div>

        <!-- Mobile: the drawer gets a close button; other modes get the rail toggle -->
        <button
          v-if="mode === 'mobile'"
          ref="toggleEl"
          type="button"
          class="icon-btn"
          aria-label="Close navigation"
          @click="$emit('close')"
        >
          <svg viewBox="0 0 24 24" class="icon" aria-hidden="true"><path d="M6 6l12 12M18 6L6 18" /></svg>
        </button>
        <button
          v-else
          ref="toggleEl"
          type="button"
          class="icon-btn"
          :aria-expanded="String(!railed)"
          :aria-label="railed ? 'Expand sidebar' : 'Collapse sidebar'"
          :title="railed ? 'Expand sidebar' : 'Collapse sidebar'"
          @click="$emit('toggle')"
        >
          <svg viewBox="0 0 24 24" class="icon" :class="{ flip: railed }" aria-hidden="true">
            <path d="M15 6l-6 6 6 6" />
          </svg>
        </button>
      </div>

      <nav aria-label="Primary" class="sidebar-nav">
        <router-link
          v-for="item in items"
          :key="item.to"
          :to="item.to"
          class="nav-item"
          :aria-label="item.label"
          :title="railed ? item.label : undefined"
        >
          <svg class="icon nav-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path v-for="d in item.icon" :key="d" :d="d" />
          </svg>
          <span class="nav-label">{{ item.label }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="$emit('show-profile-details')"
          @show-tasks="$emit('show-tasks')"
        />
      </div>
    </aside>

    <!-- Scrim for the mobile drawer and the tablet overlay; aria-hidden, Esc is the keyboard path -->
    <div v-if="mode !== 'desktop' && open" class="scrim" aria-hidden="true" @click="$emit('close')"></div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useI18n } from '../composables/useI18n'
import LanguageSwitcher from './LanguageSwitcher.vue'
import ProfileMenu from './ProfileMenu.vue'

const props = defineProps({
  // 'desktop' | 'tablet' | 'mobile'; derived from matchMedia in App.vue
  mode: { type: String, required: true },
  // Render as the 64px icon rail
  railed: { type: Boolean, default: false },
  // Overlay (tablet) / drawer (mobile) is open
  open: { type: Boolean, default: false }
})

defineEmits(['toggle', 'close', 'show-profile-details', 'show-tasks'])

const { t } = useI18n()

const asideEl = ref(null)
const toggleEl = ref(null)

// Original nav order is preserved (Finance before Demand), not route order.
// "Reports" has no locale key in the app; the literal is kept as-is.
// All icons: 24px grid, 1.75 stroke, round caps/joins (styled via .icon).
const items = computed(() => [
  { to: '/', label: t('nav.overview'), icon: ['M4 4h6v6H4z', 'M14 4h6v6h-6z', 'M4 14h6v6H4z', 'M14 14h6v6h-6z'] },
  { to: '/inventory', label: t('nav.inventory'), icon: ['M21 8l-9-5-9 5v8l9 5 9-5V8z', 'M3 8l9 5 9-5', 'M12 13v8'] },
  { to: '/orders', label: t('nav.orders'), icon: ['M9 4h6a1 1 0 011 1v1H8V5a1 1 0 011-1z', 'M8 6H6a1 1 0 00-1 1v13a1 1 0 001 1h12a1 1 0 001-1V7a1 1 0 00-1-1h-2', 'M9 12h6', 'M9 16h4'] },
  { to: '/spending', label: t('nav.finance'), icon: ['M12 3a9 9 0 100 18 9 9 0 000-18z', 'M12 7v10', 'M14.8 9.5c0-.9-1.2-1.5-2.8-1.5s-2.8.7-2.8 1.8c0 2.4 5.6 1.2 5.6 3.6 0 1.1-1.3 1.8-2.8 1.8s-2.8-.6-2.8-1.5'] },
  { to: '/demand', label: t('nav.demandForecast'), icon: ['M3 17l6-6 4 4 8-8', 'M15 7h6v6'] },
  { to: '/reports', label: 'Reports', icon: ['M14 3H7a2 2 0 00-2 2v14a2 2 0 002 2h10a2 2 0 002-2V8l-5-5z', 'M14 3v5h5', 'M9 13h6', 'M9 17h6'] }
])

const isOverlay = computed(() => props.mode !== 'desktop' && props.open)

// Keep Tab inside the sidebar while it is an overlay, so keyboard users
// cannot tab into the (visually dimmed) page behind the scrim.
const trapTab = (e) => {
  if (e.key !== 'Tab' || !isOverlay.value || !asideEl.value) return
  const focusable = asideEl.value.querySelectorAll('a[href], button:not([disabled])')
  if (!focusable.length) return
  const first = focusable[0]
  const last = focusable[focusable.length - 1]
  if (e.shiftKey && document.activeElement === first) {
    e.preventDefault()
    last.focus()
  } else if (!e.shiftKey && document.activeElement === last) {
    e.preventDefault()
    first.focus()
  }
}

// Called by App.vue after the overlay opens / closes.
defineExpose({
  focusFirst: () => asideEl.value?.querySelector('.nav-item')?.focus(),
  focusToggle: () => toggleEl.value?.focus()
})
</script>

<style scoped>
.sidebar-slot {
  position: sticky;
  top: 0;
  height: 100vh;
  height: 100dvh;
  z-index: var(--z-sidebar);
}

/* The slot's grid cell stays; in mobile the aside and scrim are fixed so the wrapper adds no box. */
.sidebar-slot.mode-mobile {
  display: contents;
}

/* In tablet overlay the aside lifts out of the slot; the slot paints the rail's surface underneath. */
.sidebar-slot.mode-tablet {
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
}

/* Not overflow:auto here: that would clip the footer dropdowns. Only .sidebar-nav scrolls. */
.sidebar {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  background: var(--color-surface);
  border-right: 1px solid var(--color-border);
  padding: var(--space-4) var(--space-3);
  transition: transform 0.2s ease, width 0.2s ease;
}

/* Tablet expanded = overlay: fixed so the page does not squeeze. */
.sidebar.mode-tablet.open {
  position: fixed;
  top: 0;
  left: 0;
  width: var(--sidebar-width);
  height: 100vh;
  height: 100dvh;
  box-shadow: var(--shadow-overlay);
}

/* Mobile: off-canvas drawer */
.sidebar.mode-mobile {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  width: min(var(--sidebar-width), 85vw);
  transform: translateX(-100%);
}
.sidebar.mode-mobile.open {
  transform: translateX(0);
  box-shadow: var(--shadow-overlay);
}

.scrim {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.4);
  z-index: -1; /* below the aside within the slot's stacking context */
}
.sidebar-slot.mode-mobile .scrim {
  /* slot is display:contents, so the scrim joins the root context and needs its own layer */
  z-index: calc(var(--z-sidebar) - 1);
}

.sidebar-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-2);
  margin-bottom: var(--space-4);
  min-height: 40px;
}

.brand {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  min-width: 0;
}

.brand-mark {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  flex-shrink: 0;
  border-radius: var(--radius-sm);
  background: var(--color-primary-tint);
  color: var(--color-primary);
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}
.brand-name {
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-bold);
  color: var(--color-text);
  letter-spacing: -0.01em;
  line-height: 1.2;
}
.brand-sub {
  font-size: var(--font-size-xs);
  color: var(--color-text-muted);
  line-height: 1.3;
}

.icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.75;
  stroke-linecap: round;
  stroke-linejoin: round;
}
.icon.flip { transform: rotate(180deg); }

.icon-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  flex-shrink: 0;
  background: transparent;
  border: 1px solid transparent;
  border-radius: var(--radius-sm);
  color: var(--color-text-muted);
  cursor: pointer;
}
.icon-btn:hover { background: var(--color-hover); color: var(--color-text); }

.sidebar-nav {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--color-text-muted);
  text-decoration: none;
  font-size: var(--font-size-sm);
  font-weight: var(--font-weight-medium);
  transition: background-color 0.15s ease, color 0.15s ease;
}
.nav-item:hover { background: var(--color-hover); color: var(--color-text); }
/* Active state tints the whole pill. A one-sided accent stripe is deliberately not used. */
.nav-item.router-link-exact-active {
  background: var(--color-primary-tint);
  color: var(--color-primary);
  font-weight: var(--font-weight-semibold);
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  margin-top: var(--space-3);
  padding-top: var(--space-3);
  border-top: 1px solid var(--color-border);
}

/* Icon rail: 64px wide, 8px side padding leaves 47px inner for 40px controls. */
.sidebar.railed {
  padding-left: var(--space-2);
  padding-right: var(--space-2);
}
.sidebar.railed .sidebar-top {
  flex-direction: column;
  justify-content: flex-start;
  gap: var(--space-2);
}
.sidebar.railed .brand-text {
  /* Visually hidden but still read by screen readers */
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}
.sidebar.railed .nav-item {
  justify-content: center;
  padding: var(--space-2);
}
.sidebar.railed .nav-label {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
  white-space: nowrap;
}
</style>
