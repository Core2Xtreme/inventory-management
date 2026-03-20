<template>
  <div class="restocking">
    <div class="page-header">
      <h2>AI-Assisted Restocking</h2>
      <p>Budget-aware restock recommendations based on demand trends</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-controls">
        <div class="slider-row">
          <input
            type="range"
            min="0"
            max="100000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="budget-summary">
          <span>Available budget: {{ formatCurrency(100000) }}</span>
          <span class="separator">|</span>
          <span>Estimated cost: {{ formatCurrency(estimatedCost) }}</span>
          <span class="separator">|</span>
          <span>Remaining: {{ formatCurrency(budget - estimatedCost) }}</span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ recommendations.length }} items fit budget)</h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        No items match your current budget or demand criteria.
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>SKU</th>
              <th>Item</th>
              <th>Trend</th>
              <th>Recommended Qty</th>
              <th>Unit Cost</th>
              <th>Total Cost</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td><strong>{{ item.sku }}</strong></td>
              <td>{{ item.name }}</td>
              <td>
                <span :class="['badge', item.trend]">
                  {{ capitalize(item.trend) }}
                </span>
              </td>
              <td>{{ item.recommended_qty.toLocaleString() }}</td>
              <td>{{ formatCurrency(item.unit_cost) }}</td>
              <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="card-footer">
        <div v-if="orderSuccess" class="success-banner">
          Restocking order placed successfully.
        </div>
        <div v-if="orderError" class="error-banner">
          {{ orderError }}
        </div>
        <div class="footer-actions">
          <button
            class="place-order-btn"
            :disabled="loading || submitting || recommendations.length === 0"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

// Trend sort priority: increasing first, stable second, decreasing last
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const forecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(false)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    const recommendations = computed(() => {
      // Build sku → unit_cost map from inventory
      const costMap = {}
      for (const item of inventoryItems.value) {
        costMap[item.sku] = item.unit_cost
      }

      // Sort forecasts by trend priority
      const sorted = [...forecasts.value].sort((a, b) => {
        const pa = TREND_PRIORITY[a.trend] ?? 1
        const pb = TREND_PRIORITY[b.trend] ?? 1
        return pa - pb
      })

      const result = []
      let remainingBudget = budget.value

      for (const forecast of sorted) {
        // API returns item_sku and item_name
        const unit_cost = costMap[forecast.item_sku]
        // Skip if no cost data or cost is zero
        if (unit_cost == null || unit_cost === 0) continue

        const recommended_qty = Math.max(1, forecast.forecasted_demand - forecast.current_demand)
        const item_cost = recommended_qty * unit_cost

        let qty = 0
        let total_cost = 0

        if (item_cost <= remainingBudget) {
          // Full quantity fits
          qty = recommended_qty
          total_cost = item_cost
        } else {
          // Try partial quantity
          const partial = Math.floor(remainingBudget / unit_cost)
          if (partial >= 1) {
            qty = partial
            total_cost = partial * unit_cost
          } else {
            continue
          }
        }

        remainingBudget -= total_cost
        result.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          recommended_qty: qty,
          unit_cost,
          total_cost
        })
      }

      return result
    })

    const estimatedCost = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.total_cost, 0)
    )

    const loadData = async () => {
      loading.value = true
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData
        inventoryItems.value = inventoryData
      } catch (err) {
        console.error('Failed to load restocking data:', err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      orderError.value = null
      try {
        await api.submitRestockingOrder({
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity: r.recommended_qty,
            unit_cost: r.unit_cost
          })),
          warehouse: 'San Francisco',
          total_value: estimatedCost.value
        })
        orderSuccess.value = true
        // Reset success message after 5 seconds
        setTimeout(() => { orderSuccess.value = false }, 5000)
      } catch (err) {
        orderError.value = 'Failed to place order'
        console.error('Failed to submit restocking order:', err)
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', {
        style: 'currency',
        currency: 'USD',
        maximumFractionDigits: 0
      })

    const capitalize = (str) =>
      str ? str.charAt(0).toUpperCase() + str.slice(1) : ''

    onMounted(() => loadData())

    return {
      budget,
      loading,
      submitting,
      orderSuccess,
      orderError,
      recommendations,
      estimatedCost,
      placeOrder,
      formatCurrency,
      capitalize
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

.budget-card {
  margin-bottom: 1.5rem;
}

.budget-controls {
  padding: 1rem 1.5rem 1.5rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  height: 4px;
  cursor: pointer;
}

.budget-value {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  min-width: 90px;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
  color: #64748b;
  flex-wrap: wrap;
}

.separator {
  color: #cbd5e1;
}

.card-footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.footer-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.75rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  color: #065f46;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
}

.error-banner {
  background: #fef2f2;
  color: #991b1b;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

/* Trend badge colors */
.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.stable {
  background: #dbeafe;
  color: #1e40af;
}

.badge.decreasing {
  background: #fee2e2;
  color: #991b1b;
}
</style>
