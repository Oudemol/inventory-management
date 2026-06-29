<template>
  <div class="app">
    <header class="top-nav">
      <div class="nav-container">
        <div class="logo">
          <h1>{{ t('nav.companyName') }}</h1>
          <span class="subtitle">{{ t('nav.subtitle') }}</span>
        </div>
        <nav class="nav-tabs">
          <router-link to="/" :class="{ active: $route.path === '/' }">
            {{ t('nav.overview') }}
          </router-link>
          <router-link to="/inventory" :class="{ active: $route.path === '/inventory' }">
            {{ t('nav.inventory') }}
          </router-link>
          <router-link to="/orders" :class="{ active: $route.path === '/orders' }">
            {{ t('nav.orders') }}
          </router-link>
          <router-link to="/spending" :class="{ active: $route.path === '/spending' }">
            {{ t('nav.finance') }}
          </router-link>
          <router-link to="/demand" :class="{ active: $route.path === '/demand' }">
            {{ t('nav.demandForecast') }}
          </router-link>
          <router-link to="/restocking" :class="{ active: $route.path === '/restocking' }">
            Restocking
          </router-link>
          <router-link to="/reports" :class="{ active: $route.path === '/reports' }">
            Reports
          </router-link>
        </nav>
        <LanguageSwitcher />
        <DarkModeToggle />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </header>
    <FilterBar />
    <main class="main-content">
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
import { ref, onMounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import { useDarkMode } from './composables/useDarkMode'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'
import DarkModeToggle from './components/DarkModeToggle.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher,
    DarkModeToggle
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    useDarkMode()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)

    // Use mock tasks from currentUser only — no /api/tasks endpoint exists yet
    const tasks = computed(() => currentUser.value.tasks)

    const addTask = (taskData) => {
      currentUser.value.tasks.push({
        id: Date.now().toString(),
        ...taskData,
        status: 'pending'
      })
    }

    const deleteTask = (taskId) => {
      const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
      if (index !== -1) {
        currentUser.value.tasks.splice(index, 1)
      }
    }

    const toggleTask = (taskId) => {
      const task = currentUser.value.tasks.find(t => t.id === taskId)
      if (task) {
        task.status = task.status === 'pending' ? 'completed' : 'pending'
      }
    }

    return {
      t,
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
/* EPAM Design System — light mode defaults; html.dark overrides for dark mode */
:root {
  --color-bg: #FBFAFA;
  --color-surface: #FFFFFF;
  --color-surface-elevated: #F5F5F5;
  --color-border: rgba(0, 0, 0, 0.10);
  --color-text-primary: #060606;
  --color-text-secondary: rgba(0, 0, 0, 0.55);
  --color-text-muted: rgba(0, 0, 0, 0.35);
  --color-text-body: rgba(0, 0, 0, 0.80);
  --color-bg-subtle: #FAFAFA;
  --color-accent: #0047FF;
  --color-accent-glow: rgba(0, 71, 255, 0.12);
  --color-accent-bg: rgba(0, 71, 255, 0.08);
  --color-lilac: #8453D2;
  --color-success: #036B58;
  --color-warning: #CB3E01;
  --color-error: #C50303;
  --color-info: #0078C2;
  --color-lime: #5B7B62;
  --color-success-bg: rgba(3, 107, 88, 0.12);
  --color-warning-bg: rgba(203, 62, 1, 0.12);
  --color-error-bg: rgba(197, 3, 3, 0.12);
  --color-info-bg: rgba(0, 120, 194, 0.12);
  --color-lilac-bg: rgba(132, 83, 210, 0.12);
  --color-shadow: rgba(0, 0, 0, 0.10);
}

html.dark {
  --color-bg: #060606;
  --color-surface: #111111;
  --color-surface-elevated: #1a1a1a;
  --color-border: rgba(255, 255, 255, 0.08);
  --color-text-primary: #ffffff;
  --color-text-secondary: rgba(255, 255, 255, 0.55);
  --color-text-muted: rgba(255, 255, 255, 0.35);
  --color-text-body: rgba(255, 255, 255, 0.80);
  --color-bg-subtle: #0d0d0d;
  --color-accent: #00F6FF;
  --color-accent-glow: rgba(0, 246, 255, 0.30);
  --color-accent-bg: rgba(0, 246, 255, 0.08);
  --color-lilac: #B896FF;
  --color-success: #00F4A8;
  --color-warning: #FF7701;
  --color-error: #FF4B98;
  --color-info: #7BA8FF;
  --color-lime: #D5E662;
  --color-success-bg: rgba(0, 244, 168, 0.15);
  --color-warning-bg: rgba(255, 119, 1, 0.15);
  --color-error-bg: rgba(255, 75, 152, 0.15);
  --color-info-bg: rgba(123, 168, 255, 0.15);
  --color-lilac-bg: rgba(184, 150, 255, 0.15);
  --color-shadow: rgba(0, 0, 0, 0.50);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'MuseoSansCyrl', 'Calibri', 'Trebuchet MS', system-ui, sans-serif;
  background: var(--color-bg);
  color: var(--color-text-body);
  font-weight: 300;
  letter-spacing: 0.02em;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.top-nav {
  background: var(--color-surface);
  border-bottom: 1px solid var(--color-border);
  position: sticky;
  top: 0;
  z-index: 100;
}

.nav-container {
  max-width: 1600px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  padding: 0 2rem;
  height: 70px;
}

.nav-container > .nav-tabs {
  margin-left: auto;
  margin-right: 1rem;
}

.nav-container > .language-switcher {
  margin-right: 1rem;
}

.logo {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.logo h1 {
  font-size: 1.375rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: 0.02em;
}

.subtitle {
  font-size: 0.813rem;
  color: var(--color-text-secondary);
  font-weight: 300;
  padding-left: 0.75rem;
  border-left: 1px solid var(--color-border);
}

/* EPAM underline tab pattern */
.nav-tabs {
  display: flex;
  gap: 0;
  border-bottom: 1px solid var(--color-border);
  align-self: stretch;
}

.nav-tabs a {
  padding: 12px 20px;
  color: var(--color-text-secondary);
  text-decoration: none;
  font-weight: 300;
  font-size: 15px;
  transition: all 0.2s ease;
  position: relative;
  display: flex;
  align-items: center;
  white-space: nowrap;
}

.nav-tabs a:hover {
  color: var(--color-text-primary);
}

.nav-tabs a.active {
  color: var(--color-accent);
  font-weight: 700;
}

.nav-tabs a.active::after {
  content: '';
  position: absolute;
  bottom: -1px;
  left: 0;
  right: 0;
  height: 2px;
  background: var(--color-accent);
  box-shadow: 0 0 8px var(--color-accent-glow);
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--color-text-primary);
  margin-bottom: 0.375rem;
  letter-spacing: 0.02em;
}

.page-header p {
  color: var(--color-text-secondary);
  font-size: 0.938rem;
  font-weight: 300;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

/* EPAM card style */
.stat-card {
  background: var(--color-surface);
  padding: 28px 24px;
  border-radius: 12px;
  border: 1px solid var(--color-border);
  box-shadow: 0 4px 24px var(--color-shadow);
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: rgba(255, 255, 255, 0.15);
}

/* EPAM section label (caps) */
.stat-label {
  color: var(--color-text-muted);
  font-size: 11px;
  font-weight: 900;
  text-transform: uppercase;
  letter-spacing: 0.5em;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: 0.02em;
}

.stat-card.warning .stat-value {
  color: var(--color-warning);
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-error);
}

.stat-card.info .stat-value {
  color: var(--color-accent);
}

/* EPAM card */
.card {
  background: var(--color-surface);
  border-radius: 12px;
  padding: 28px 24px;
  border: 1px solid var(--color-border);
  box-shadow: 0 4px 24px var(--color-shadow);
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: 0.02em;
}

.table-container {
  overflow-x: auto;
  border: 1px solid var(--color-border);
  border-radius: 10px;
  overflow: hidden;
}

table {
  width: 100%;
  border-collapse: collapse;
}

/* EPAM table header */
thead {
  background: var(--color-surface-elevated);
}

th {
  text-align: left;
  padding: 0.625rem 0.75rem;
  font-size: 11px;
  font-weight: 900;
  letter-spacing: 0.4em;
  text-transform: uppercase;
  color: var(--color-text-muted);
}

/* EPAM alternating row colors */
td {
  padding: 0.5rem 0.75rem;
  font-size: 14px;
  font-weight: 300;
  color: var(--color-text-body);
}

tbody tr:nth-child(odd) {
  background: var(--color-bg-subtle);
}

tbody tr:nth-child(even) {
  background: var(--color-surface);
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: var(--color-surface-elevated);
}

/* EPAM filled badge tags */
.badge {
  display: inline-block;
  padding: 4px 10px;
  border-radius: 6px;
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.badge.success {
  background: var(--color-success-bg);
  color: var(--color-success);
}

.badge.warning {
  background: var(--color-warning-bg);
  color: var(--color-warning);
}

.badge.danger {
  background: var(--color-error-bg);
  color: var(--color-error);
}

.badge.info {
  background: var(--color-info-bg);
  color: var(--color-info);
}

.badge.increasing {
  background: var(--color-success-bg);
  color: var(--color-success);
}

.badge.decreasing {
  background: var(--color-error-bg);
  color: var(--color-error);
}

.badge.stable {
  background: var(--color-lilac-bg);
  color: var(--color-lilac);
}

.badge.high {
  background: var(--color-error-bg);
  color: var(--color-error);
}

.badge.medium {
  background: var(--color-warning-bg);
  color: var(--color-warning);
}

.badge.low {
  background: var(--color-info-bg);
  color: var(--color-info);
}

.loading {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-secondary);
  font-size: 0.938rem;
}

.error {
  background: var(--color-error-bg);
  border: 1px solid var(--color-error);
  color: var(--color-error);
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
