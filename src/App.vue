<template>
  <div class="page" :class="themeClass">
    <div class="container-fluid px-0">
      <header class="header row g-3 align-items-center">
        <div class="col-12 col-md-8">
          <h1>Golden Rate Calculator</h1>
          <p class="subtitle mb-0">
            Transparent pricing for gold &amp; silver pieces with design charge and 12% PH tax.
          </p>
        </div>
        <div class="col-12 col-md-4 text-md-end text-muted small trust-badges">
          <span class="badge-pill">Trusted by local jewelry buyers</span>
          <span class="badge-pill">Designed By Rey Vincent D. Armenta</span>
        </div>
      </header>

      <main class="card calculator-card mt-3">
        <div class="row g-4 align-items-stretch">
          <!-- Live Rates -->
          <section class="col-12 col-lg-4">
            <div class="section-label text-uppercase fw-semibold mb-2">
              Live Rates
            </div>
            <div
              class="card-inner live-card h-100 d-flex flex-column justify-content-between hover-lift"
              :class="{ 'is-loading': isRefreshing }"
            >
              <div v-if="isRefreshing" class="skeleton-stack">
                <div class="skeleton skeleton-pill"></div>
                <div class="skeleton skeleton-price mt-3"></div>
                <div class="skeleton skeleton-text mt-2"></div>
              </div>
              <div v-else>
                <div class="d-flex justify-content-between align-items-center mb-2">
                  <span class="live-pill">Current metal</span>
                  <span class="live-metal">{{ metalLabel }}</span>
                </div>
                <div class="live-value mb-2 rolling-number">
                  <span class="live-prefix">₱</span>
                  <span class="live-main" :key="ratePerGram">
                    {{
                      (ratePerGram || 0).toLocaleString(undefined, {
                        ...currencyFormat,
                        minimumFractionDigits: 2,
                      })
                    }}
                  </span>
                  <span class="live-suffix">/ gram</span>
                </div>
                <p class="live-note mb-0">
                  Update this with your latest in-store or supplier rate for
                  <strong>{{ metalLabel.toLowerCase() }}</strong>.
                </p>
              </div>
              <div class="d-flex justify-content-between align-items-center mt-3 gap-2">
                <button
                  type="button"
                  class="btn-accent-outline flex-grow-1"
                  :disabled="isRefreshing"
                  @click="refreshRates"
                >
                  <span v-if="!isRefreshing">Refresh rate</span>
                  <span v-else>Refreshing…</span>
                </button>
                <button
                  type="button"
                  class="btn-accent-fill flex-grow-1"
                  @click="focusInputs"
                >
                  Calculate
                </button>
              </div>
              <div class="text-muted microcopy mt-2">
                Visual feedback and clear labels help shoppers trust the price breakdown.
              </div>
            </div>
          </section>

          <!-- Inputs -->
          <section class="col-12 col-lg-4">
            <div class="section-label text-uppercase fw-semibold mb-2">
              Input Details
            </div>
            <div class="card-inner h-100 hover-lift" ref="inputCardRef">
              <div class="mb-3">
                <label class="form-label field-label">Metal type</label>
                <div class="segmented" role="radiogroup" aria-label="Metal selection">
                  <button
                    type="button"
                    class="btn-toggle"
                    :class="{ active: metal === 'gold' }"
                    @click="metal = 'gold'"
                  >
                    Gold
                  </button>
                  <button
                    type="button"
                    class="btn-toggle"
                    :class="{ active: metal === 'silver' }"
                    @click="metal = 'silver'"
                  >
                    Silver
                  </button>
                </div>
              </div>

              <div class="mb-3">
                <label for="rate" class="form-label field-label">Rate per gram (₱)</label>
                <input
                  id="rate"
                  type="number"
                  min="0"
                  step="0.01"
                  class="form-control input-soft"
                  v-model.number="ratePerGram"
                  :placeholder="`Enter current ${metalLabel} rate`"
                />
              </div>

              <div class="mb-3">
                <label for="weight" class="form-label field-label">Weight (grams)</label>
                <input
                  id="weight"
                  type="number"
                  min="0"
                  step="0.01"
                  class="form-control input-soft"
                  v-model.number="weightInGrams"
                  placeholder="e.g. 5"
                />
              </div>

              <div>
                <label for="design" class="form-label field-label">Design charge (₱)</label>
                <input
                  id="design"
                  type="number"
                  min="0"
                  step="0.01"
                  class="form-control input-soft"
                  v-model.number="designCharge"
                  placeholder="e.g. 500"
                />
              </div>
            </div>
          </section>

          <!-- Final Total & Breakdown -->
          <section class="col-12 col-lg-4">
            <div class="section-label text-uppercase fw-semibold mb-2">
              Final Total
            </div>
            <div class="card-inner h-100 d-flex flex-column justify-content-between hover-lift">
              <Transition name="fade-slide">
                <div v-if="isValid" key="summary" class="summary">
                  <p class="summary-metal mb-2">
                    {{ metalLabel }} piece ·
                    <strong>{{ weightInGrams?.toFixed(2) }} g</strong>
                  </p>

                  <div class="summary-row">
                    <span>Material cost</span>
                    <span>₱ {{ baseValue.toLocaleString(undefined, currencyFormat) }}</span>
                  </div>
                  <div class="summary-row">
                    <span>Design fee</span>
                    <span>₱ {{ designCharge?.toLocaleString(undefined, currencyFormat) }}</span>
                  </div>
                  <div class="summary-row">
                    <span>Subtotal</span>
                    <span>₱ {{ subtotal.toLocaleString(undefined, currencyFormat) }}</span>
                  </div>
                  <div class="summary-row">
                    <span class="d-inline-flex align-items-center gap-1">
                      12% PH tax
                      <span
                        class="info-dot"
                        title="Standard 12% VAT applied in the Philippines, calculated on subtotal."
                      >
                        i
                      </span>
                    </span>
                    <span class="rolling-number" :key="tax">
                      ₱ {{ tax.toLocaleString(undefined, currencyFormat) }}
                    </span>
                  </div>

                  <div class="summary-total mt-2">
                    <div>
                      <span class="label d-block">Customer total</span>
                      <span class="hint d-block">
                        (rate × grams) + design fee + 12% tax
                      </span>
                    </div>
                    <Transition name="pulse-total">
                      <div class="total-amount rolling-number" :key="total">
                        ₱ {{ total.toLocaleString(undefined, currencyFormat) }}
                      </div>
                    </Transition>
                  </div>
                </div>

                <div v-else key="hint" class="hint-box">
                  <p class="mb-1">
                    Add the live rate, weight, and design fee to preview a fully itemized price.
                  </p>
                  <p class="mb-0 small text-muted">
                    Transparent breakdowns help customers feel confident and ready to checkout.
                  </p>
                </div>
              </Transition>
            </div>
          </section>
        </div>
      </main>

      <footer class="footer mt-3">
        <span class="small text-muted">
          Built to support brand trust for local jewelry &amp; cosmetics sellers.
        </span>
      </footer>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

type Metal = 'gold' | 'silver'

const metal = ref<Metal>('gold')
const ratePerGram = ref<number | null>(null)
const weightInGrams = ref<number | null>(null)
const designCharge = ref<number | null>(null)
const isRefreshing = ref(false)
const inputCardRef = ref<HTMLElement | null>(null)

const metalLabel = computed(() => (metal.value === 'gold' ? 'Gold' : 'Silver'))
const themeClass = computed(() =>
  metal.value === 'gold' ? 'theme-gold' : 'theme-silver',
)

const isValid = computed(
  () =>
    ratePerGram.value !== null &&
    ratePerGram.value > 0 &&
    weightInGrams.value !== null &&
    weightInGrams.value > 0 &&
    designCharge.value !== null &&
    designCharge.value >= 0,
)

const baseValue = computed(
  () => (ratePerGram.value || 0) * (weightInGrams.value || 0),
)
const subtotal = computed(
  () => baseValue.value + (designCharge.value || 0),
)
const taxRate = 0.12
const tax = computed(() => subtotal.value * taxRate)
const total = computed(() => subtotal.value + tax.value)

const currencyFormat: Intl.NumberFormatOptions = {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
}

function refreshRates() {
  if (isRefreshing.value) return
  isRefreshing.value = true

  // Simulate a short network refresh for live market rates
  setTimeout(() => {
    // Keep existing rate if set; otherwise, seed a sensible default
    if (ratePerGram.value === null || ratePerGram.value === 0) {
      ratePerGram.value = metal.value === 'gold' ? 3500 : 45
    }
    isRefreshing.value = false
  }, 900)
}

function focusInputs() {
  if (inputCardRef.value) {
    const firstField = inputCardRef.value.querySelector<HTMLInputElement>(
      'input[type="number"]',
    )
    firstField?.focus()
  }
}

onMounted(() => {
  // Seed a gentle default to avoid an empty hero card
  ratePerGram.value = metal.value === 'gold' ? 3500 : 45
})
</script>

