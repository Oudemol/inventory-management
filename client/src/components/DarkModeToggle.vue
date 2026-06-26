<template>
  <button class="dark-mode-toggle" @click="toggleDark" :title="isDark ? 'Switch to light mode' : 'Switch to dark mode'">
    <!-- Sun: shown in dark mode -->
    <svg v-if="isDark" width="20" height="20" viewBox="0 0 20 20" fill="none">
      <circle cx="10" cy="10" r="4" stroke="currentColor" stroke-width="1.5"/>
      <path d="M10 2V4M10 16V18M2 10H4M16 10H18M4.22 4.22L5.64 5.64M14.36 14.36L15.78 15.78M15.78 4.22L14.36 5.64M5.64 14.36L4.22 15.78" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
    </svg>
    <!-- Moon: shown in light mode -->
    <svg v-else width="20" height="20" viewBox="0 0 20 20" fill="none">
      <path d="M17 11.5A7.5 7.5 0 118.5 3a5.5 5.5 0 108.5 8.5z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </button>
</template>

<script>
import { useDarkMode } from '../composables/useDarkMode'

export default {
  name: 'DarkModeToggle',
  setup() {
    const { isDark, toggleDark } = useDarkMode()
    return { isDark, toggleDark }
  }
}
</script>

<style scoped>
.dark-mode-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: transparent;
  border: 1.5px solid var(--color-border);
  border-radius: 8px;
  cursor: pointer;
  color: var(--color-text-secondary);
  transition: all 0.2s ease;
  margin-right: 0.75rem;
  flex-shrink: 0;
}

.dark-mode-toggle:hover {
  background: var(--color-surface-elevated);
  color: var(--color-accent);
  border-color: var(--color-accent);
}
</style>
