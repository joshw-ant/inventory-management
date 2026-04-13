<template>
  <div class="app-shell" :class="{ collapsed: sidebarCollapsed }">
    <header class="mobile-topbar">
      <button class="hamburger" type="button" aria-label="Open navigation" @click="mobileOpen = !mobileOpen">
        <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <line x1="3" y1="6" x2="21" y2="6"/>
          <line x1="3" y1="12" x2="21" y2="12"/>
          <line x1="3" y1="18" x2="21" y2="18"/>
        </svg>
      </button>
      <span class="mobile-title">{{ t('nav.companyName') }}</span>
    </header>

    <div
      v-if="mobileOpen"
      class="mobile-scrim"
      @click="mobileOpen = false"
    ></div>

    <Sidebar
      v-model:collapsed="sidebarCollapsed"
      :mobile-open="mobileOpen"
      :routes="navRoutes"
    >
      <template #brand>
        <span class="brand-mark" aria-hidden="true"></span>
        <span v-if="!sidebarCollapsed" class="brand-text">
          <span class="brand-name">{{ t('nav.companyName') }}</span>
          <span class="brand-sub">{{ t('nav.subtitle') }}</span>
        </span>
      </template>
      <template #footer>
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </template>
    </Sidebar>

    <main class="main">
      <FilterBar />
      <router-view />
    </main>

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
import { ref, onMounted, computed, h, watch } from 'vue'
import { useRoute } from 'vue-router'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import Sidebar from './components/Sidebar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

// Inline SVG icons rendered as Vue components (no v-html — avoids XSS footgun
// if route icons ever come from a non-literal source).
const svg = (children) => h('svg', {
  viewBox: '0 0 24 24',
  fill: 'none',
  stroke: 'currentColor',
  'stroke-width': 2,
  'stroke-linecap': 'round',
  'stroke-linejoin': 'round'
}, children)

const IconHome   = { render: () => svg([h('path', { d: 'M3 9.5L12 3l9 6.5V20a1 1 0 0 1-1 1h-5v-7h-6v7H4a1 1 0 0 1-1-1V9.5z' })]) }
const IconBox    = { render: () => svg([
  h('path', { d: 'M21 16V8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16z' }),
  h('polyline', { points: '3.27 6.96 12 12.01 20.73 6.96' }),
  h('line', { x1: '12', y1: '22.08', x2: '12', y2: '12' })
]) }
const IconList   = { render: () => svg([
  h('line', { x1: '8', y1: '6', x2: '21', y2: '6' }),
  h('line', { x1: '8', y1: '12', x2: '21', y2: '12' }),
  h('line', { x1: '8', y1: '18', x2: '21', y2: '18' }),
  h('circle', { cx: '4', cy: '6', r: '1' }),
  h('circle', { cx: '4', cy: '12', r: '1' }),
  h('circle', { cx: '4', cy: '18', r: '1' })
]) }
const IconDollar = { render: () => svg([
  h('line', { x1: '12', y1: '1', x2: '12', y2: '23' }),
  h('path', { d: 'M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6' })
]) }
const IconTrend  = { render: () => svg([
  h('polyline', { points: '23 6 13.5 15.5 8.5 10.5 1 18' }),
  h('polyline', { points: '17 6 23 6 23 12' })
]) }
const IconChart  = { render: () => svg([
  h('line', { x1: '18', y1: '20', x2: '18', y2: '10' }),
  h('line', { x1: '12', y1: '20', x2: '12', y2: '4' }),
  h('line', { x1: '6', y1: '20', x2: '6', y2: '14' })
]) }

export default {
  name: 'App',
  components: {
    FilterBar,
    Sidebar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const route = useRoute()

    const sidebarCollapsed = ref(false)
    const mobileOpen = ref(false)
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Nav labels are i18n-driven; use computed so they update when locale changes.
    const navRoutes = computed(() => [
      { to: '/',          label: t('nav.overview'),       icon: IconHome },
      { to: '/inventory', label: t('nav.inventory'),      icon: IconBox },
      { to: '/orders',    label: t('nav.orders'),         icon: IconList },
      { to: '/spending',  label: t('nav.finance'),        icon: IconDollar },
      { to: '/demand',    label: t('nav.demandForecast'), icon: IconTrend },
      { to: '/reports',   label: 'Reports',               icon: IconChart }
    ])

    // Close mobile drawer when the user navigates.
    watch(() => route.path, () => { mobileOpen.value = false })

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
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)
        if (isMockTask) {
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)
        if (mockTask) {
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
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
      sidebarCollapsed,
      mobileOpen,
      navRoutes,
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
/* Design tokens */
:root {
  /* Spacing */
  --sp-1: 4px;
  --sp-2: 8px;
  --sp-3: 12px;
  --sp-4: 16px;
  --sp-5: 24px;
  --sp-6: 32px;
  --sp-7: 40px;
  --sp-8: 48px;

  /* Radius */
  --r-sm: 4px;
  --r-md: 6px;
  --r-lg: 10px;
  --r-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.04), 0 1px 3px rgba(15, 23, 42, 0.06);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.06), 0 2px 4px rgba(15, 23, 42, 0.04);
  --shadow-lg: 0 10px 25px rgba(15, 23, 42, 0.08), 0 4px 10px rgba(15, 23, 42, 0.05);

  /* Colors */
  --c-bg: #f8fafc;
  --c-surface: #ffffff;
  --c-surface-2: #f1f5f9;
  --c-border: #e2e8f0;
  --c-border-strong: #cbd5e1;
  --c-text: #0f172a;
  --c-text-muted: #475569;
  --c-text-soft: #64748b;

  --c-accent: #4f46e5;
  --c-accent-soft: #eef2ff;

  --c-success: #059669;
  --c-success-soft: #d1fae5;
  --c-warn: #ea580c;
  --c-warn-soft: #fed7aa;
  --c-danger: #dc2626;
  --c-danger-soft: #fecaca;
  --c-info: #2563eb;
  --c-info-soft: #dbeafe;

  /* Type scale */
  --fs-xs: 12px;
  --fs-sm: 13px;
  --fs-base: 14px;
  --fs-md: 16px;
  --fs-lg: 20px;
  --fs-xl: 24px;
  --fs-2xl: 32px;

  --fw-regular: 400;
  --fw-medium: 500;
  --fw-semibold: 600;
  --fw-bold: 700;

  /* Sidebar */
  --sidebar-w: 240px;
  --sidebar-w-collapsed: 68px;
  --sidebar-bg: #ffffff;
  --sidebar-border: var(--c-border);

  /* Motion */
  --ease: cubic-bezier(0.4, 0, 0.2, 1);
  --dur-fast: 120ms;
  --dur-med: 200ms;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  background: var(--c-bg);
  color: var(--c-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ── App shell ─────────────────────────────────────────────────── */

.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-w) 1fr;
  min-height: 100vh;
  transition: grid-template-columns var(--dur-med) var(--ease);
}
.app-shell.collapsed {
  grid-template-columns: var(--sidebar-w-collapsed) 1fr;
}

.main {
  padding: var(--sp-6);
  min-width: 0;
  overflow-x: hidden;
}

/* Mobile top bar shows a hamburger; hidden on larger screens. */
.mobile-topbar { display: none; }
.mobile-scrim  { display: none; }

@media (max-width: 768px) {
  .app-shell,
  .app-shell.collapsed {
    grid-template-columns: 1fr;
  }
  .main { padding: var(--sp-4); }
  .mobile-topbar {
    grid-column: 1 / -1;
    display: flex;
    align-items: center;
    gap: var(--sp-3);
    padding: var(--sp-3) var(--sp-4);
    border-bottom: 1px solid var(--c-border);
    background: var(--c-surface);
    position: sticky;
    top: 0;
    z-index: 150;
  }
  .hamburger {
    background: none;
    border: none;
    color: var(--c-text);
    padding: 4px;
    cursor: pointer;
    display: inline-flex;
  }
  .mobile-title {
    font-weight: var(--fw-semibold);
    color: var(--c-text);
    font-size: var(--fs-md);
  }
  .mobile-scrim {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.4);
    z-index: 150;
  }
}

/* ── Brand block ───────────────────────────────────────────────── */

.brand-mark {
  width: 32px;
  height: 32px;
  border-radius: var(--r-md);
  background: linear-gradient(135deg, var(--c-accent) 0%, #7c3aed 100%);
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}
.brand-text {
  display: flex;
  flex-direction: column;
  line-height: 1.2;
  min-width: 0;
}
.brand-name {
  font-weight: var(--fw-semibold);
  font-size: var(--fs-base);
  color: var(--c-text);
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.brand-sub {
  font-size: 11px;
  color: var(--c-text-soft);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* ── Shared primitives (used across views) ─────────────────────── */

.page-header {
  margin-bottom: var(--sp-5);
}
.page-header h2 {
  font-size: var(--fs-xl);
  font-weight: var(--fw-bold);
  color: var(--c-text);
  margin-bottom: var(--sp-1);
  letter-spacing: -0.02em;
}
.page-header p {
  color: var(--c-text-soft);
  font-size: var(--fs-base);
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: var(--sp-4);
  margin-bottom: var(--sp-5);
}

.stat-card {
  background: var(--c-surface);
  padding: var(--sp-5);
  border-radius: var(--r-lg);
  border: 1px solid var(--c-border);
  box-shadow: var(--shadow-sm);
  transition: border-color var(--dur-fast), box-shadow var(--dur-fast);
}
.stat-card:hover {
  border-color: var(--c-border-strong);
  box-shadow: var(--shadow-md);
}
.stat-label {
  color: var(--c-text-soft);
  font-size: var(--fs-xs);
  font-weight: var(--fw-semibold);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: var(--sp-2);
}
.stat-value {
  font-size: var(--fs-2xl);
  font-weight: var(--fw-bold);
  color: var(--c-text);
  letter-spacing: -0.02em;
  line-height: 1.1;
}

.stat-card.warning .stat-value { color: var(--c-warn); }
.stat-card.success .stat-value { color: var(--c-success); }
.stat-card.danger .stat-value  { color: var(--c-danger); }
.stat-card.info .stat-value    { color: var(--c-accent); }

.card {
  background: var(--c-surface);
  border-radius: var(--r-lg);
  padding: var(--sp-5);
  border: 1px solid var(--c-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--sp-5);
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--sp-4);
  padding-bottom: var(--sp-3);
  border-bottom: 1px solid var(--c-border);
}
.card-title {
  font-size: var(--fs-md);
  font-weight: var(--fw-semibold);
  color: var(--c-text);
  letter-spacing: -0.01em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--fs-base);
}
thead {
  background: var(--c-surface-2);
  border-top: 1px solid var(--c-border);
  border-bottom: 1px solid var(--c-border);
}
th {
  text-align: left;
  padding: var(--sp-2) var(--sp-3);
  font-weight: var(--fw-semibold);
  color: var(--c-text-muted);
  font-size: var(--fs-xs);
  text-transform: uppercase;
  letter-spacing: 0.06em;
}
td {
  padding: var(--sp-3);
  border-top: 1px solid var(--c-border);
  color: var(--c-text);
  font-size: var(--fs-sm);
}
tbody tr {
  transition: background-color var(--dur-fast);
}
tbody tr:hover {
  background: var(--c-surface-2);
}

.badge {
  display: inline-block;
  padding: var(--sp-1) var(--sp-2);
  border-radius: var(--r-md);
  font-size: var(--fs-xs);
  font-weight: var(--fw-semibold);
  text-transform: uppercase;
  letter-spacing: 0.04em;
}
.badge.success    { background: var(--c-success-soft); color: var(--c-success); }
.badge.warning    { background: var(--c-warn-soft);    color: var(--c-warn); }
.badge.danger     { background: var(--c-danger-soft);  color: var(--c-danger); }
.badge.info       { background: var(--c-info-soft);    color: var(--c-info); }
.badge.increasing { background: var(--c-success-soft); color: var(--c-success); }
.badge.decreasing { background: var(--c-danger-soft);  color: var(--c-danger); }
.badge.stable     { background: #e0e7ff;              color: #3730a3; }
.badge.high       { background: var(--c-danger-soft);  color: var(--c-danger); }
.badge.medium     { background: var(--c-warn-soft);    color: var(--c-warn); }
.badge.low        { background: var(--c-info-soft);    color: var(--c-info); }

.loading {
  text-align: center;
  padding: var(--sp-8);
  color: var(--c-text-soft);
  font-size: var(--fs-base);
}

.error {
  background: #fef2f2;
  border: 1px solid var(--c-danger-soft);
  color: #991b1b;
  padding: var(--sp-4);
  border-radius: var(--r-md);
  margin: var(--sp-4) 0;
  font-size: var(--fs-base);
}
</style>
