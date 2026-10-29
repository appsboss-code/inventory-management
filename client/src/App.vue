<template>
  <div class="app-shell" :class="[`mode-${mode}`, { railed }]">
    <AppSidebar
      ref="sidebarRef"
      :mode="mode"
      :railed="railed"
      :open="overlayOpen"
      @toggle="toggleSidebar"
      @close="overlayOpen = false"
      @show-profile-details="showProfileDetails = true"
      @show-tasks="showTasks = true"
    />

    <div class="app-main">
      <header v-if="mode === 'mobile'" class="mobile-bar">
        <button
          ref="menuBtn"
          type="button"
          class="menu-btn"
          :aria-expanded="String(overlayOpen)"
          aria-label="Open navigation"
          @click="overlayOpen = true"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" aria-hidden="true">
            <path d="M4 6h16M4 12h16M4 18h16" />
          </svg>
        </button>
        <span class="mobile-title">{{ t('nav.companyName') }}</span>
      </header>

      <FilterBar />
      <main class="app-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, computed, watch, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import AppSidebar from './components/AppSidebar.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    AppSidebar,
    ProfileDetailsModal,
    TasksModal
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // ---- Sidebar state ----
    // Breakpoints: >1024 desktop, 769-1024 tablet (icon rail, expands as overlay), <=768 mobile drawer.
    const MOBILE_QUERY = '(max-width: 768px)'
    const TABLET_QUERY = '(min-width: 769px) and (max-width: 1024px)'
    const STORAGE_KEY = 'sidebar-collapsed'
    const route = useRoute()
    const sidebarRef = ref(null)
    const menuBtn = ref(null)

    // localStorage can throw in sandboxed hosts; fall back to per-session state.
    // Returns null when the user has never chosen, so the breakpoint decides.
    const readPref = () => {
      try {
        const v = localStorage.getItem(STORAGE_KEY)
        return v === null ? null : v === '1'
      } catch { return null }
    }
    const writePref = (collapsed) => {
      try { localStorage.setItem(STORAGE_KEY, collapsed ? '1' : '0') } catch { /* storage blocked */ }
    }

    const mobileMql = window.matchMedia(MOBILE_QUERY)
    const tabletMql = window.matchMedia(TABLET_QUERY)
    const computeMode = () => (mobileMql.matches ? 'mobile' : tabletMql.matches ? 'tablet' : 'desktop')
    const mode = ref(computeMode())
    const onMediaChange = () => { mode.value = computeMode() }

    const collapsedPref = ref(readPref())
    // Tablet overlay / mobile drawer open state. Never persisted: it is transient by nature.
    const overlayOpen = ref(false)

    // Rail rendering. The stored choice only drives desktop; on tablet the breakpoint wins
    // (rail by default) so a stale "expanded" choice cannot squeeze the page.
    const railed = computed(() => {
      if (mode.value === 'desktop') return collapsedPref.value === true
      if (mode.value === 'tablet') return !overlayOpen.value
      return false
    })

    const toggleSidebar = () => {
      if (mode.value === 'desktop') {
        collapsedPref.value = !railed.value
        writePref(collapsedPref.value)
      } else {
        overlayOpen.value = !overlayOpen.value
      }
    }

    // Focus management: into the overlay on open, back to its trigger on close.
    watch(overlayOpen, async (open) => {
      await nextTick()
      if (open) {
        sidebarRef.value?.focusFirst()
      } else if (mode.value === 'mobile') {
        menuBtn.value?.focus()
      } else {
        sidebarRef.value?.focusToggle()
      }
    })
    watch(mode, () => { overlayOpen.value = false })
    watch(() => route.path, () => { overlayOpen.value = false })

    const onKeydown = (e) => {
      if (e.key === 'Escape' && overlayOpen.value) overlayOpen.value = false
    }
    onMounted(() => {
      mobileMql.addEventListener('change', onMediaChange)
      tabletMql.addEventListener('change', onMediaChange)
      document.addEventListener('keydown', onKeydown)
    })
    onBeforeUnmount(() => {
      mobileMql.removeEventListener('change', onMediaChange)
      tabletMql.removeEventListener('change', onMediaChange)
      document.removeEventListener('keydown', onKeydown)
    })

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      mode,
      railed,
      overlayOpen,
      sidebarRef,
      menuBtn,
      toggleSidebar,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
/* Shell layout only. Shared page styles live in styles/tokens.css and styles/base.css. */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
  transition: grid-template-columns 0.2s ease;
}
/* Tablet always reserves only the rail width: expanding overlays instead of pushing. */
.app-shell.railed,
.app-shell.mode-tablet { grid-template-columns: var(--sidebar-width-collapsed) minmax(0, 1fr); }
.app-shell.mode-mobile { grid-template-columns: minmax(0, 1fr); }

/* min-width:0 (via minmax above) lets wide tables scroll instead of stretching the grid */
.app-main {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.app-content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max-width);
  margin: 0 auto;
  padding: var(--page-padding);
}

.mobile-bar {
  position: sticky;
  top: 0;
  z-index: var(--z-mobile-bar);
  display: flex;
  align-items: center;
  gap: var(--space-3);
  height: var(--header-height);
  padding: 0 var(--space-4);
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
}
.menu-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  background: transparent;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  color: var(--color-text);
  cursor: pointer;
}
.menu-btn:hover { background: var(--color-hover); }
.menu-btn svg { fill: none; stroke: currentColor; stroke-width: 1.75; stroke-linecap: round; }
.mobile-title {
  font-size: var(--font-size-md);
  font-weight: var(--font-weight-bold);
  color: var(--color-text);
  letter-spacing: -0.01em;
}
</style>
