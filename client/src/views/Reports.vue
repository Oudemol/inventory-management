<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.subtitle') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('reports.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.quarterly.title') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.quarterly.quarter') }}</th>
                <th>{{ t('reports.quarterly.totalOrders') }}</th>
                <th>{{ t('reports.quarterly.totalRevenue') }}</th>
                <th>{{ t('reports.quarterly.avgOrderValue') }}</th>
                <th>{{ t('reports.quarterly.fulfillmentRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td><strong>{{ q.quarter }}</strong></td>
                <td>{{ q.total_orders }}</td>
                <td>${{ formatNumber(q.total_revenue) }}</td>
                <td>${{ formatNumber(q.avg_order_value) }}</td>
                <td>
                  <span :class="getFulfillmentClass(q.fulfillment_rate)">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Trends Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthlyChart.title') }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div v-for="month in monthlyData" :key="month.month" class="bar-wrapper">
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: getBarHeight(month.revenue) + 'px' }"
                  :title="'$' + formatNumber(month.revenue)"
                ></div>
              </div>
              <div class="bar-label">{{ formatMonth(month.month) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Comparison -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthlyTable.title') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.monthlyTable.month') }}</th>
                <th>{{ t('reports.monthlyTable.orders') }}</th>
                <th>{{ t('reports.monthlyTable.revenue') }}</th>
                <th>{{ t('reports.monthlyTable.change') }}</th>
                <th>{{ t('reports.monthlyTable.growthRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(month, index) in monthlyData" :key="month.month">
                <td><strong>{{ formatMonth(month.month) }}</strong></td>
                <td>{{ month.order_count }}</td>
                <td>${{ formatNumber(month.revenue) }}</td>
                <td>
                  <span v-if="index > 0" :class="getChangeClass(month.revenue, monthlyData[index - 1].revenue)">
                    {{ getChangeValue(month.revenue, monthlyData[index - 1].revenue) }}
                  </span>
                  <span v-else>-</span>
                </td>
                <td>
                  <span v-if="index > 0" :class="getChangeClass(month.revenue, monthlyData[index - 1].revenue)">
                    {{ getGrowthRate(month.revenue, monthlyData[index - 1].revenue) }}
                  </span>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.totalRevenue') }}</div>
          <div class="stat-value">${{ formatNumber(totalRevenue) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.avgMonthly') }}</div>
          <div class="stat-value">${{ formatNumber(avgMonthlyRevenue) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.totalOrders') }}</div>
          <div class="stat-value">{{ totalOrders }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.stats.bestQuarter') }}</div>
          <div class="stat-value">{{ bestQuarter }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

const { selectedPeriod, selectedLocation, selectedCategory, selectedStatus, getCurrentFilters } = useFilters()
const { t } = useI18n()

const loading = ref(true)
const error = ref(null)
const quarterlyData = ref([])
const monthlyData = ref([])

// Derived summary stats as computed properties so they stay in sync with data
const totalRevenue = computed(() => monthlyData.value.reduce((sum, m) => sum + m.revenue, 0))
const avgMonthlyRevenue = computed(() => monthlyData.value.length > 0 ? totalRevenue.value / monthlyData.value.length : 0)
const totalOrders = computed(() => monthlyData.value.reduce((sum, m) => sum + m.order_count, 0))
const bestQuarter = computed(() => {
  if (!quarterlyData.value.length) return ''
  return quarterlyData.value.reduce((best, q) => q.total_revenue > best.total_revenue ? q : best).quarter
})

// Cached max revenue for bar height calculations — avoids re-scanning the array on every render
const maxRevenue = computed(() => Math.max(...monthlyData.value.map(m => m.revenue), 0))

const loadData = async () => {
  loading.value = true
  error.value = null
  try {
    const filters = getCurrentFilters()
    const [quarterly, monthly] = await Promise.all([
      api.getQuarterlyReports(filters),
      api.getMonthlyTrends(filters)
    ])
    quarterlyData.value = quarterly
    monthlyData.value = monthly
  } catch (err) {
    error.value = 'Failed to load reports: ' + err.message
  } finally {
    loading.value = false
  }
}

watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], loadData)

onMounted(() => loadData())

function formatNumber(num) {
  // Guard against null/undefined/NaN before formatting
  if (num == null || isNaN(Number(num))) return '0.00'
  var str = num.toString()
  var parts = str.split('.')
  var intPart = parts[0]
  var decPart = parts.length > 1 ? parts[1] : '00'

  var formatted = ''
  var count = 0
  for (var i = intPart.length - 1; i >= 0; i--) {
    if (count > 0 && count % 3 === 0) {
      formatted = ',' + formatted
    }
    formatted = intPart[i] + formatted
    count++
  }

  if (decPart.length === 1) {
    decPart = decPart + '0'
  }
  if (decPart.length > 2) {
    decPart = decPart.substring(0, 2)
  }

  return formatted + '.' + decPart
}

function formatMonth(monthStr) {
  var parts = monthStr.split('-')
  var year = parts[0]
  var month = parts[1]

  var monthNames = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
  var monthIndex = parseInt(month) - 1

  return monthNames[monthIndex] + ' ' + year
}

function getBarHeight(revenue) {
  // Use cached maxRevenue computed property instead of re-scanning the array on every call
  if (maxRevenue.value === 0) return 0
  return (revenue / maxRevenue.value) * 200
}

function getFulfillmentClass(rate) {
  if (rate >= 90) return 'badge success'
  else if (rate >= 75) return 'badge warning'
  else return 'badge danger'
}

function getChangeValue(current, previous) {
  var change = current - previous
  if (change > 0) return '+$' + formatNumber(change)
  else if (change < 0) return '-$' + formatNumber(Math.abs(change))
  else return '$0.00'
}

function getChangeClass(current, previous) {
  var change = current - previous
  if (change > 0) return 'positive-change'
  else if (change < 0) return 'negative-change'
  else return ''
}

function getGrowthRate(current, previous) {
  if (previous === 0) return 'N/A'
  var rate = ((current - previous) / previous) * 100
  var sign = rate > 0 ? '+' : ''
  return sign + rate.toFixed(1) + '%'
}
</script>

<style scoped>
.reports {
  padding: 0;
}

/* Reports uses global .card, .card-header, .card-title from App.vue.
   These scoped overrides only adjust what's specific to this view. */
.reports-table {
  width: 100%;
  border-collapse: collapse;
}

.reports-table th {
  background: var(--color-surface-elevated);
  padding: 0.625rem 0.75rem;
  text-align: left;
  font-size: 11px;
  font-weight: 900;
  letter-spacing: 0.4em;
  text-transform: uppercase;
  color: var(--color-text-secondary);
  border-bottom: 1px solid var(--color-border);
}

.reports-table td {
  padding: 0.75rem;
  border-bottom: 1px solid var(--color-border);
  font-size: 14px;
  font-weight: 300;
  color: var(--color-text-body);
}

.reports-table tr:nth-child(odd) td {
  background: var(--color-bg-subtle);
}

.reports-table tr:nth-child(even) td {
  background: var(--color-surface);
}

.reports-table tr:hover td {
  background: var(--color-surface-elevated);
}

.chart-container {
  padding: 2rem 1rem;
  min-height: 300px;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 250px;
  gap: 0.5rem;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  max-width: 80px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  width: 100%;
}

.bar {
  width: 100%;
  background: linear-gradient(to top, var(--color-accent), var(--color-accent-bg));
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  cursor: pointer;
}

.bar:hover {
  background: linear-gradient(to top, var(--color-accent), var(--color-lilac));
  box-shadow: 0 0 12px var(--color-accent-glow);
}

.bar-label {
  font-size: 0.75rem;
  color: var(--color-text-secondary);
  font-weight: 300;
  text-align: center;
  transform: rotate(-45deg);
  white-space: nowrap;
  margin-top: 1.5rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
  margin-top: 1.5rem;
}

.stat-card {
  background: var(--color-surface);
  border-radius: 12px;
  padding: 28px 24px;
  border: 1px solid var(--color-border);
  box-shadow: 0 4px 24px var(--color-shadow);
  border-left: 4px solid var(--color-accent);
}

.stat-label {
  font-size: 11px;
  font-weight: 900;
  letter-spacing: 0.5em;
  text-transform: uppercase;
  color: var(--color-text-muted);
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--color-text-primary);
  letter-spacing: 0.02em;
}

/* Scoped badge overrides for reports (fulfillment rate badges) */
.badge {
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

.positive-change {
  color: var(--color-success);
  font-weight: 700;
}

.negative-change {
  color: var(--color-error);
  font-weight: 700;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-secondary);
}

.error {
  background: var(--color-error-bg);
  border: 1px solid var(--color-error);
  color: var(--color-error);
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
}
</style>
