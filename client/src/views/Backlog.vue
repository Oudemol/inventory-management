<template>
  <div class="backlog">
    <div class="page-header">
      <h2>{{ t('backlog.title') }}</h2>
      <p>{{ t('backlog.subtitle') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('backlog.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="stats-grid">
        <div class="stat-card danger">
          <div class="stat-label">{{ t('backlog.stats.highPriority') }}</div>
          <div class="stat-value">{{ highPriorityCount }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('backlog.stats.mediumPriority') }}</div>
          <div class="stat-value">{{ mediumPriorityCount }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">{{ t('backlog.stats.lowPriority') }}</div>
          <div class="stat-value">{{ lowPriorityCount }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('backlog.stats.totalItems') }}</div>
          <div class="stat-value">{{ backlogItems.length }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('backlog.table.title') }}</h3>
        </div>
        <div v-if="backlogItems.length === 0" style="padding: 3rem; text-align: center;">
          <p class="backlog-empty">
            ✓ {{ t('backlog.noItems') }}
          </p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('backlog.table.orderId') }}</th>
                <th>{{ t('backlog.table.sku') }}</th>
                <th>{{ t('backlog.table.itemName') }}</th>
                <th>{{ t('backlog.table.quantityNeeded') }}</th>
                <th>{{ t('backlog.table.quantityAvailable') }}</th>
                <th>{{ t('backlog.table.shortage') }}</th>
                <th>{{ t('backlog.table.daysDelayed') }}</th>
                <th>{{ t('backlog.table.priority') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in backlogItems" :key="item.id">
                <td><strong>{{ item.order_id }}</strong></td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.quantity_needed }}</td>
                <td>{{ item.quantity_available }}</td>
                <td>
                  <span class="badge danger">
                    {{ item.quantity_needed - item.quantity_available }} {{ t('backlog.table.unitsShort') }}
                  </span>
                </td>
                <td>
                  <span :class="item.days_delayed > 7 ? 'days-urgent' : 'days-warning'">
                    {{ item.days_delayed }} {{ t('backlog.table.days') }}
                  </span>
                </td>
                <td>
                  <span :class="['badge', getPriorityClass(item.priority)]">
                    {{ item.priority }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Backlog',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const allBacklogItems = ref([])
    const inventoryItems = ref([])

    const { t } = useI18n()

    // Use shared filters
    const { selectedPeriod, selectedLocation, selectedCategory, selectedStatus, getCurrentFilters } = useFilters()

    // Filter backlog based on inventory filters
    const backlogItems = computed(() => {
      if (selectedLocation.value === 'all' && selectedCategory.value === 'all') {
        return allBacklogItems.value
      }

      // Get SKUs of items that match the filters
      const validSkus = new Set(inventoryItems.value.map(item => item.sku))
      return allBacklogItems.value.filter(b => validSkus.has(b.item_sku))
    })

    const highPriorityCount = computed(() => backlogItems.value.filter(i => i.priority === 'high').length)
    const mediumPriorityCount = computed(() => backlogItems.value.filter(i => i.priority === 'medium').length)
    const lowPriorityCount = computed(() => backlogItems.value.filter(i => i.priority === 'low').length)

    const loadBacklog = async () => {
      try {
        loading.value = true
        const filters = getCurrentFilters()

        const [backlogData, inventoryData] = await Promise.all([
          api.getBacklog(),
          api.getInventory({
            warehouse: filters.warehouse,
            category: filters.category
          })
        ])

        allBacklogItems.value = backlogData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load backlog: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Watch all four filters so this page stays consistent with the rest of the app.
    // Backlog doesn't filter by period/status server-side, but warehouse/category
    // drive the client-side SKU filtering via the inventory fetch.
    watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], loadBacklog)

    const VALID_PRIORITIES = new Set(['high', 'medium', 'low'])
    function getPriorityClass(priority) {
      const normalized = (priority || '').toLowerCase()
      return VALID_PRIORITIES.has(normalized) ? normalized : 'info'
    }

    onMounted(loadBacklog)

    return {
      loading,
      error,
      backlogItems,
      highPriorityCount,
      mediumPriorityCount,
      lowPriorityCount,
      getPriorityClass,
      t
    }
  }
}
</script>

<style scoped>
.backlog-empty {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--color-success);
  letter-spacing: 0.02em;
}

.days-urgent { color: var(--color-error); font-weight: 700; }
.days-warning { color: var(--color-warning); font-weight: 700; }
</style>
