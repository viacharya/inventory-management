<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts</p>
    </div>

    <div v-if="submittedOrder" class="card success-card">
      <div class="success-banner">
        <span class="success-icon">&#10003;</span>
        <span class="success-text">Order <strong>{{ submittedOrder.id }}</strong> submitted successfully</span>
      </div>
      <div class="success-details">
        <div class="success-detail-item">
          <span class="detail-label">Submitted</span>
          <span class="detail-value">{{ formatDate(submittedOrder.submitted_date) }}</span>
        </div>
        <div class="success-detail-item">
          <span class="detail-label">Expected Delivery</span>
          <span class="detail-value">{{ formatDate(submittedOrder.expected_delivery) }}</span>
        </div>
        <div class="success-detail-item">
          <span class="detail-label">Total Cost</span>
          <span class="detail-value"><strong>{{ formatCurrency(submittedOrder.total_cost) }}</strong></span>
        </div>
      </div>
      <div class="success-actions">
        <button class="btn btn-primary" @click="resetForm">Restock Another</button>
      </div>
    </div>

    <template v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-slider-row">
            <input
              type="range"
              v-model.number="budget"
              min="0"
              max="50000"
              step="500"
              class="budget-slider"
            />
            <span class="budget-value">{{ formatBudget(budget) }}</span>
          </div>
          <div class="budget-stats">
            <span class="budget-stat">{{ recommendations.length }} item{{ recommendations.length !== 1 ? 's' : '' }} recommended</span>
            <span class="budget-stat-sep">·</span>
            <span class="budget-stat">Total: {{ formatCurrency(totalCost) }}</span>
          </div>
        </div>
      </div>

      <div v-if="budget > 0" class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="badge info">{{ recommendations.length }}</span>
        </div>

        <div v-if="loading" class="loading">Loading recommendations...</div>
        <div v-else-if="error" class="error">{{ error }}</div>
        <div v-else-if="recommendations.length === 0" class="empty-state">
          No items fit within the selected budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th>Current Demand</th>
                <th>Forecasted Demand</th>
                <th>Qty to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td>{{ item.current_demand }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ item.qty_to_order }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
                <td>{{ item.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-actions">
          <button
            class="btn btn-primary"
            :class="{ disabled: recommendations.length === 0 || submitting }"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>
      </div>

      <div v-else class="card empty-budget-card">
        <p class="empty-state">Adjust your budget to see recommendations</p>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(10000)
    const recommendationsData = ref({ recommendations: [], total_cost: 0, budget: 0, remaining_budget: 0 })
    const loading = ref(false)
    const error = ref(null)
    const submittedOrder = ref(null)
    const submitting = ref(false)

    let debounceTimer = null

    watch(budget, (val) => {
      clearTimeout(debounceTimer)
      if (val === 0) {
        recommendationsData.value = { recommendations: [], total_cost: 0 }
        return
      }
      debounceTimer = setTimeout(async () => {
        loading.value = true
        error.value = null
        try {
          recommendationsData.value = await api.getRestockingRecommendations(val)
        } catch (e) {
          error.value = 'Failed to load recommendations'
          console.error(e)
        } finally {
          loading.value = false
        }
      }, 300)
    }, { immediate: true })

    const recommendations = computed(() => recommendationsData.value.recommendations || [])
    const totalCost = computed(() => recommendationsData.value.total_cost || 0)

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        const order = await api.submitRestockingOrder(recommendations.value, budget.value)
        submittedOrder.value = order
      } catch (e) {
        error.value = 'Failed to submit order'
        console.error(e)
      } finally {
        submitting.value = false
      }
    }

    const resetForm = () => {
      submittedOrder.value = null
      budget.value = 10000
    }

    const formatCurrency = (val) => {
      return '$' + Number(val).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const formatBudget = (val) => {
      return '$' + Number(val).toLocaleString('en-US')
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    return {
      budget,
      recommendations,
      totalCost,
      loading,
      error,
      submittedOrder,
      submitting,
      placeOrder,
      resetForm,
      formatCurrency,
      formatBudget,
      formatDate
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-section {
  padding: 0.5rem 0 0.25rem;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  margin-bottom: 0.75rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  min-width: 120px;
  text-align: right;
}

.budget-stats {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: #64748b;
  font-size: 0.875rem;
}

.budget-stat-sep {
  color: #cbd5e1;
}

.order-actions {
  display: flex;
  justify-content: flex-end;
  padding-top: 1rem;
  margin-top: 0.25rem;
  border-top: 1px solid #e2e8f0;
}

.btn {
  padding: 0.625rem 1.5rem;
  border: none;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-primary {
  background: #2563eb;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled,
.btn-primary.disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

.empty-budget-card {
  border: 1px dashed #cbd5e1;
  background: #f8fafc;
}

.success-card {
  border-left: 4px solid #059669;
}

.success-banner {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding-bottom: 1rem;
  margin-bottom: 1rem;
  border-bottom: 1px solid #d1fae5;
  font-size: 1rem;
  color: #065f46;
}

.success-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  background: #d1fae5;
  border-radius: 50%;
  font-size: 0.875rem;
  font-weight: 700;
  color: #059669;
  flex-shrink: 0;
}

.success-details {
  display: flex;
  gap: 2rem;
  margin-bottom: 1.25rem;
  flex-wrap: wrap;
}

.success-detail-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.detail-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.detail-value {
  font-size: 0.938rem;
  color: #0f172a;
}

.success-actions {
  display: flex;
  justify-content: flex-end;
}
</style>
