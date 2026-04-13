<template>
  <aside class="sidebar" :class="{ collapsed, open: mobileOpen }">
    <div class="sidebar-brand">
      <slot name="brand">
        <span class="brand-mark"></span>
        <span v-if="!collapsed" class="brand-text">App</span>
      </slot>
    </div>

    <nav class="sidebar-nav" aria-label="Primary">
      <router-link
        v-for="route in routes"
        :key="route.to"
        :to="route.to"
        class="nav-item"
        active-class="active"
      >
        <span class="nav-icon" aria-hidden="true">
          <component :is="route.icon" />
        </span>
        <span v-if="!collapsed" class="nav-label">{{ route.label }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <slot name="footer" />
      <button
        type="button"
        class="collapse-btn"
        :aria-label="collapsed ? 'Expand sidebar' : 'Collapse sidebar'"
        @click="$emit('update:collapsed', !collapsed)"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline v-if="!collapsed" points="15 18 9 12 15 6" />
          <polyline v-else points="9 18 15 12 9 6" />
        </svg>
      </button>
    </div>
  </aside>
</template>

<script>
export default {
  name: 'Sidebar',
  props: {
    routes: { type: Array, required: true },
    collapsed: { type: Boolean, default: false },
    mobileOpen: { type: Boolean, default: false }
  },
  emits: ['update:collapsed', 'update:mobileOpen']
}
</script>

<style scoped>
.sidebar {
  display: flex;
  flex-direction: column;
  background: var(--sidebar-bg);
  border-right: 1px solid var(--sidebar-border);
  min-height: 100vh;
  transition: width var(--dur-med) var(--ease);
  width: var(--sidebar-w);
  position: sticky;
  top: 0;
  max-height: 100vh;
  overflow: hidden;
}
.sidebar.collapsed { width: var(--sidebar-w-collapsed); }

.sidebar-brand {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  padding: var(--sp-4);
  border-bottom: 1px solid var(--c-border);
  min-height: 64px;
}

.sidebar-nav {
  flex: 1;
  padding: var(--sp-3) var(--sp-2);
  display: flex;
  flex-direction: column;
  gap: var(--sp-1);
  overflow-y: auto;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  padding: var(--sp-2) var(--sp-3);
  border-radius: var(--r-md);
  color: var(--c-text-muted);
  text-decoration: none;
  font-size: var(--fs-base);
  font-weight: var(--fw-medium);
  transition: background var(--dur-fast), color var(--dur-fast);
  white-space: nowrap;
  overflow: hidden;
}
.nav-item:hover {
  background: var(--c-surface-2);
  color: var(--c-text);
}
.nav-item.active {
  background: var(--c-accent-soft);
  color: var(--c-accent);
}

.nav-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}
.nav-icon :deep(svg) { width: 18px; height: 18px; }
.nav-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-footer {
  border-top: 1px solid var(--c-border);
  padding: var(--sp-3);
  display: flex;
  flex-direction: column;
  gap: var(--sp-2);
}
.collapse-btn {
  align-self: flex-end;
  background: transparent;
  border: 1px solid var(--c-border);
  border-radius: var(--r-md);
  padding: 6px;
  color: var(--c-text-soft);
  cursor: pointer;
  transition: background var(--dur-fast), color var(--dur-fast);
  display: inline-flex;
}
.collapse-btn:hover {
  background: var(--c-surface-2);
  color: var(--c-text);
}

.sidebar.collapsed .sidebar-footer :deep(.profile-menu),
.sidebar.collapsed .sidebar-footer :deep(.language-switcher) {
  display: none;
}

@media (max-width: 768px) {
  .sidebar {
    position: fixed;
    top: 0;
    bottom: 0;
    left: 0;
    z-index: 200;
    transform: translateX(-100%);
    box-shadow: var(--shadow-lg);
  }
  .sidebar.open { transform: translateX(0); }
  .sidebar.collapsed { width: var(--sidebar-w); }
}
</style>
