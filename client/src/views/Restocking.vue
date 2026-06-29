<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your available budget and review recommended items to restock based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="budget-header-row">
          <span class="budget-label">Available Budget</span>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="500000"
          step="1000"
          v-model.number="budget"
        />
        <div class="utilization-section">
          <div class="utilization-bar-bg">
            <div
              class="utilization-bar-fill"
              :style="{ width: budgetPercent + '%' }"
            ></div>
          </div>
          <div class="utilization-label" v-if="selectedTotal > 0">
            {{ budgetPercent.toFixed(0) }}% of budget allocated
          </div>
          <div class="utilization-label muted" v-else>
            No items selected
          </div>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Restock Recommendations ({{ recommendations.length }} items)</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Category</th>
                <th>Trend</th>
                <th class="col-right">Forecast Demand</th>
                <th class="col-right">In Stock</th>
                <th class="col-right">Qty to Order</th>
                <th>Unit Cost</th>
                <th class="col-right">Est. Cost</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-selected': selectedSkus.has(item.item_sku) }"
              >
                <td class="col-check">
                  <!-- New Set replacement triggers reactivity since Vue 3 doesn't track Set mutations -->
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change="toggleSku(item.item_sku)"
                  />
                </td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.category }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-right">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-right">{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td class="col-right">{{ item.recommended_quantity.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-right">{{ formatCurrency(item.estimated_cost) }}</td>
                <td>{{ item.lead_time_days }}d</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="summary-row">
          {{ selectedItems.length }} items selected &mdash; Total: {{ formatCurrency(selectedTotal) }} of {{ formatCurrency(budget) }} budget
        </div>
      </div>

      <!-- Action Bar -->
      <div class="action-bar">
        <div
          v-if="successOrder"
          class="success-banner"
        >
          Order {{ successOrder.id }} submitted successfully! Estimated delivery in {{ maxLeadTime(successOrder.items) }} days.
        </div>
        <button
          class="btn-primary"
          :disabled="selectedSkus.size === 0 || selectedTotal > budget || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Submitting...' : 'Place Order' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(100000)
    const recommendations = ref([])
    const selectedSkus = ref(new Set())
    const loading = ref(true)
    const submitting = ref(false)
    const error = ref(null)
    const successOrder = ref(null)

    const formatCurrency = (v) =>
      '$' + v.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })

    const selectedItems = computed(() =>
      recommendations.value.filter(r => selectedSkus.value.has(r.item_sku))
    )

    const selectedTotal = computed(() =>
      selectedItems.value.reduce((sum, r) => sum + r.estimated_cost, 0)
    )

    const budgetPercent = computed(() =>
      budget.value > 0 ? Math.min((selectedTotal.value / budget.value) * 100, 100) : 0
    )

    // Greedy auto-selection: sort by cost ASC to guarantee monotonic behavior.
    // More budget always adds items — never removes them. Priority is visible via badges.
    const autoSelect = () => {
      const sorted = [...recommendations.value].sort((a, b) => a.estimated_cost - b.estimated_cost)
      const newSelected = new Set()
      let running = 0
      for (const item of sorted) {
        if (running + item.estimated_cost <= budget.value) {
          newSelected.add(item.item_sku)
          running += item.estimated_cost
        }
      }
      selectedSkus.value = newSelected
    }

    // Re-run auto-selection whenever slider changes; manual checkbox toggles skip this
    watch(budget, () => {
      autoSelect()
    })

    // Replace Set reference so Vue detects the change and re-renders
    const toggleSku = (sku) => {
      const next = new Set(selectedSkus.value)
      if (next.has(sku)) next.delete(sku)
      else next.add(sku)
      selectedSkus.value = next
    }

    const maxLeadTime = (items) =>
      items && items.length ? Math.max(...items.map(i => i.lead_time_days)) : 0

    const placeOrder = async () => {
      submitting.value = true
      successOrder.value = null
      try {
        const items = selectedItems.value.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          category: r.category,
          quantity: r.recommended_quantity,
          unit_cost: r.unit_cost,
          lead_time_days: r.lead_time_days
        }))
        const result = await api.submitRestockingOrder({
          items,
          total_cost: selectedTotal.value
        })
        successOrder.value = result
        selectedSkus.value = new Set()
      } finally {
        submitting.value = false
      }
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations()
        // Run auto-selection after first load
        autoSelect()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      budget,
      recommendations,
      selectedSkus,
      loading,
      submitting,
      error,
      successOrder,
      selectedItems,
      selectedTotal,
      budgetPercent,
      toggleSku,
      placeOrder,
      maxLeadTime,
      formatCurrency
    }
  }
}
</script>

<style scoped>
/* Budget card */
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-header-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.budget-label {
  font-size: 11px;
  font-weight: 900;
  letter-spacing: 0.5em;
  text-transform: uppercase;
  color: var(--color-text-muted);
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: 0.02em;
}

.budget-slider {
  width: 100%;
  margin-bottom: 0.75rem;
  accent-color: var(--color-accent);
}

.utilization-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.utilization-bar-bg {
  height: 6px;
  border-radius: 3px;
  background: var(--color-border);
  overflow: hidden;
}

.utilization-bar-fill {
  background: var(--color-success);
  height: 100%;
  transition: width 0.3s;
  box-shadow: 0 0 8px var(--color-success-bg);
}

.utilization-label {
  font-size: 0.8125rem;
  color: var(--color-text-secondary);
  font-weight: 300;
}

.utilization-label.muted {
  color: var(--color-text-muted);
}

/* Table */
.restock-table {
  table-layout: auto;
  width: 100%;
}

.col-check {
  width: 40px;
  text-align: center;
}

.col-right {
  text-align: right;
}

.sku-cell {
  font-family: monospace;
  font-size: 0.8125rem;
  color: var(--color-accent);
}

/* Selected row highlight in EPAM style */
.row-selected {
  background: rgba(0, 246, 255, 0.06) !important;
}

/* Summary row below table */
.summary-row {
  display: flex;
  justify-content: flex-end;
  padding: 0.75rem;
  font-size: 0.875rem;
  color: var(--color-text-secondary);
  font-weight: 300;
  border-top: 1px solid var(--color-border);
}

/* Action bar */
.action-bar {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 1rem;
  margin-top: 1rem;
}

.success-banner {
  background: var(--color-success-bg);
  border: 1px solid var(--color-success);
  color: var(--color-success);
  padding: 0.75rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 300;
}

/* EPAM primary button */
.btn-primary {
  background: transparent;
  color: var(--color-accent);
  border: 2px solid var(--color-accent);
  padding: 12px 32px;
  border-radius: 40px;
  font-weight: 900;
  font-size: 0.813rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  cursor: pointer;
  transition: all 0.2s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: var(--color-accent);
  color: var(--color-bg);
  box-shadow: 0 0 30px var(--color-accent-glow);
}

.btn-primary:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* EPAM trend badges — scoped override */
.badge.increasing {
  background: var(--color-success-bg);
  color: var(--color-success);
}

.badge.stable {
  background: var(--color-lilac-bg);
  color: var(--color-lilac);
}

.badge.decreasing {
  background: var(--color-error-bg);
  color: var(--color-error);
}
</style>
