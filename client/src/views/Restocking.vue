<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-slider-section">
          <div class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            :min="0"
            :max="maxBudget"
            :step="50"
          />
          <div class="slider-ends">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
          </div>
          <p class="budget-help">{{ t('restocking.budgetHelp') }}</p>
        </div>
      </div>

      <!-- Summary Stat Cards -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.recommendedSpend') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ recommendedSpend.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.itemsRecommended') }}</div>
          <div class="stat-value">{{ itemsRecommended }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.unitsTotal') }}</div>
          <div class="stat-value">{{ unitsTotal }}</div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }}</h3>
        </div>
        <div v-if="recommendations.length === 0" class="no-recommendations">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.gap') }}</th>
                <th>{{ t('restocking.table.recommendedQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in recommendations" :key="row.item_sku">
                <td><strong>{{ row.item_sku }}</strong></td>
                <td>{{ row.item_name }}</td>
                <td>
                  <span :class="['badge', row.trend]">{{ t('trends.' + row.trend) }}</span>
                </td>
                <td>{{ row.current_demand }}</td>
                <td><strong>{{ row.forecasted_demand }}</strong></td>
                <td>+{{ row.gap }}</td>
                <td><strong>{{ row.quantity }}</strong></td>
                <td>{{ currencySymbol }}{{ row.unit_cost }}</td>
                <td>{{ currencySymbol }}{{ row.line_total.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Footer -->
      <div class="order-action-row">
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || placing"
          @click="placeOrder"
        >
          {{ placing ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
        <span v-if="successMessage" class="success-message">{{ successMessage }}</span>
        <div v-if="submitError" class="error">{{ submitError }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const allForecasts = ref([])
    const budget = ref(0)
    const placing = ref(false)
    const successMessage = ref('')
    const submitError = ref('')

    // --- Recommendation algorithm ---

    // Step 1: candidates are forecasts where demand is growing (gap > 0),
    // enriched with their gap value and sorted largest-gap-first so the
    // greedy fill always prioritises the items that need stock most urgently.
    const candidates = computed(() => {
      return allForecasts.value
        .map(f => ({ ...f, gap: f.forecasted_demand - f.current_demand }))
        .filter(f => f.gap > 0)
        .sort((a, b) => b.gap - a.gap)
    })

    // Step 2: maxBudget is the total cost required to fully close every
    // candidate's demand gap, rounded up to the nearest 500. This becomes
    // the slider ceiling so the user can always reach "fund everything".
    const maxBudget = computed(() => {
      const fullCost = candidates.value.reduce((sum, c) => sum + c.gap * c.unit_cost, 0)
      if (fullCost === 0) return 1000
      return Math.ceil(fullCost / 500) * 500
    })

    // Step 3: greedy fill — walk candidates (biggest gap first), allocate as
    // many units as the remaining budget allows (capped at the demand gap so
    // we never over-order). Items that are too expensive to afford even 1 unit
    // are skipped; a cheaper item later in the list may still fit.
    const recommendations = computed(() => {
      let remaining = budget.value
      const result = []

      for (const candidate of candidates.value) {
        const affordableQty = Math.floor(remaining / candidate.unit_cost)
        const qty = Math.min(candidate.gap, affordableQty)

        if (qty >= 1) {
          const line_total = qty * candidate.unit_cost
          result.push({
            item_sku: candidate.item_sku,
            item_name: candidate.item_name,
            trend: candidate.trend,
            current_demand: candidate.current_demand,
            forecasted_demand: candidate.forecasted_demand,
            gap: candidate.gap,
            quantity: qty,
            unit_cost: candidate.unit_cost,
            line_total
          })
          remaining -= line_total
        }
        // Continue even when an item is skipped — a cheaper item may still fit
      }

      return result
    })

    // Summary computeds derived from recommendations
    const recommendedSpend = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_total, 0)
    )
    const budgetRemaining = computed(() => budget.value - recommendedSpend.value)
    const itemsRecommended = computed(() => recommendations.value.length)
    const unitsTotal = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.quantity, 0)
    )

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        allForecasts.value = await api.getDemandForecasts()

        // Initialise slider at roughly half of maxBudget once we know the data,
        // rounded to the nearest 100 for a clean starting value.
        const half = maxBudget.value / 2
        budget.value = Math.round(half / 100) * 100
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placing.value = true
      successMessage.value = ''
      submitError.value = ''

      try {
        const items = recommendations.value.map(r => ({
          item_sku: r.item_sku,
          item_name: r.item_name,
          quantity: r.quantity,
          unit_cost: r.unit_cost
        }))

        const order = await api.createRestockOrder({ budget: budget.value, items })
        successMessage.value = t('restocking.orderPlaced', { orderNumber: order.order_number })
      } catch (err) {
        submitError.value = t('restocking.orderError')
        console.error('Failed to place restock order:', err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      currencySymbol,
      loading,
      error,
      budget,
      placing,
      successMessage,
      submitError,
      maxBudget,
      recommendations,
      recommendedSpend,
      budgetRemaining,
      itemsRecommended,
      unitsTotal,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding: 0.5rem 0;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-ends {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.budget-help {
  font-size: 0.813rem;
  color: #64748b;
}

.no-recommendations {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-style: italic;
}

.order-action-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-top: 1rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-message {
  color: #059669;
  font-weight: 500;
  font-size: 0.938rem;
}
</style>
