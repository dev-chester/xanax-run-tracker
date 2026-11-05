<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { Copy, X } from '@lucide/vue'

const quote = ref<{ cost: number; quantity: number; updated: number } | null>(null)
const failed = ref(false)
const now = ref(Date.now())
const movement = ref<'down' | 'up' | ''>('')
const calculator = ref<HTMLElement | null>(null)
const priceButton = ref<HTMLButtonElement | null>(null)
const calculatorOpen = ref(false)
const quantityStorageKey = 'runTrackerBuyQuantity'
function readBuyQuantity(): number {
  try {
    const stored = Number(localStorage.getItem(quantityStorageKey))
    if (Number.isSafeInteger(stored) && stored >= 1) return stored
  } catch {
    // Use the default when browser storage is unavailable.
  }
  return 1
}
const buyQuantity = ref<number | string>(readBuyQuantity())
watch(buyQuantity, value => {
  const quantity = Number(value)
  if (!Number.isSafeInteger(quantity) || quantity < 1) return
  try {
    localStorage.setItem(quantityStorageKey, String(quantity))
  } catch {
    // The quantity still works for this tab when storage is unavailable.
  }
})
const copyStatus = ref('')
const totalCost = computed(() => {
  const quantity = Number(buyQuantity.value)
  if (!quote.value || buyQuantity.value === '' || !Number.isSafeInteger(quantity) || quantity < 1) return null
  const total = quote.value.cost * quantity
  return Number.isSafeInteger(total) ? total : null
})

function openCalculator() {
  copyStatus.value = ''
  calculator.value?.togglePopover()
  positionCalculator()
}

function positionCalculator() {
  const panel = calculator.value
  const anchor = priceButton.value
  if (!panel?.matches(':popover-open') || !anchor) return
  const rect = anchor.getBoundingClientRect()
  const left = Math.max(12, Math.min(rect.right - panel.offsetWidth, window.innerWidth - panel.offsetWidth - 12))
  const top = rect.top - panel.offsetHeight - 8 >= 12
    ? rect.top - panel.offsetHeight - 8 : Math.min(rect.bottom + 8, Math.max(12, window.innerHeight - panel.offsetHeight - 12))
  panel.style.left = `${left}px`
  panel.style.top = `${top}px`
}

async function copyCost() {
  if (totalCost.value === null) return
  try {
    await navigator.clipboard.writeText(String(totalCost.value))
    copyStatus.value = 'Cost copied.'
  } catch {
    copyStatus.value = 'Could not copy cost. Please copy it manually.'
  }
}
let pollTimer: number | undefined
let flashTimer: number | undefined
let ageTimer: number | undefined
let controller: AbortController | undefined
let stopped = false
const numbers = new Intl.NumberFormat('en-US')
const age = computed(() => quote.value ? Math.max(0, Math.floor((now.value / 1000) - quote.value.updated)) : 0)
const ageLabel = computed(() => age.value < 60 ? `${age.value}s ago` : age.value < 3600 ? `${Math.floor(age.value / 60)}m ago` : `${Math.floor(age.value / 3600)}h ago`)
const updatedTitle = computed(() => quote.value ? `YATA South Africa update: ${new Date(quote.value.updated * 1000).toLocaleString()}` : 'Waiting for YATA South Africa stock')

async function poll() {
  controller = new AbortController()
  const timeout = window.setTimeout(() => controller?.abort(), 15000)
  try {
    const response = await fetch('https://yata.yt/api/v1/travel/export/', {
      signal: controller.signal,
      credentials: 'omit',
      cache: 'no-store',
    })
    if (!response.ok) throw new Error(`YATA ${response.status}`)
    const payload = await response.json()
    const south = payload?.stocks?.sou
    const xanax = Array.isArray(south?.stocks)
      ? south.stocks.find((item: { id?: number; name?: string }) => item.id === 206 || item.name === 'Xanax')
      : undefined
    if (!xanax || !Number.isSafeInteger(xanax.cost) || xanax.cost < 0 ||
        !Number.isSafeInteger(xanax.quantity) || xanax.quantity < 0 ||
        !Number.isSafeInteger(south.update) || south.update <= 0 ||
        !Number.isFinite(new Date(south.update * 1000).getTime())) throw new Error('Invalid YATA stock')
    if (stopped) return
    failed.value = false
    // Ignore older snapshots, and compare against the previous accepted quote.
    if (quote.value && south.update < quote.value.updated) return
    if (quote.value && xanax.cost !== quote.value.cost) {
      window.clearTimeout(flashTimer)
      movement.value = xanax.cost < quote.value.cost ? 'down' : 'up'
      flashTimer = window.setTimeout(() => { movement.value = '' }, 2800)
    }
    quote.value = { cost: xanax.cost, quantity: xanax.quantity, updated: south.update }
    now.value = Date.now()
  } catch {
    if (!stopped) failed.value = true
  } finally {
    window.clearTimeout(timeout)
    if (!stopped) pollTimer = window.setTimeout(poll, 5000 + Math.floor(Math.random() * 25001))
  }
}

onMounted(() => {
  void poll()
  ageTimer = window.setInterval(() => { now.value = Date.now() }, 1000)
  window.addEventListener('resize', positionCalculator)
  window.addEventListener('scroll', positionCalculator, true)
})
onUnmounted(() => {
  stopped = true
  controller?.abort()
  window.clearTimeout(pollTimer)
  window.clearTimeout(flashTimer)
  window.clearInterval(ageTimer)
  window.removeEventListener('resize', positionCalculator)
  window.removeEventListener('scroll', positionCalculator, true)
})
</script>

<template>
  <div class="xanax-ticker" :title="updatedTitle" aria-label="South Africa Xanax stock">
    <div class="ticker-values">
      <span class="ticker-symbol">SA · XAN</span>
      <button ref="priceButton" type="button" class="ticker-price" :class="movement" :disabled="!quote" aria-haspopup="dialog" :aria-expanded="calculatorOpen" aria-controls="xanax-buy-calculator" aria-label="Open live Xanax buy cost calculator" @click="openCalculator">
        <span aria-hidden="true">{{ movement === 'down' ? '↓' : movement === 'up' ? '↑' : '' }}</span>
        {{ quote ? `$${numbers.format(quote.cost)}` : '—' }}
      </button>
      <span class="ticker-quantity">Qty <b>{{ quote ? numbers.format(quote.quantity) : '—' }}</b></span>
    </div>
    <small class="ticker-updated" :class="{ warning: failed || (quote && age > 300) }">
      {{ failed ? (quote ? 'Update failed · showing last quote · ' : 'YATA unavailable · retrying') : '' }}{{ quote ? `YATA updated ${ageLabel}` : failed ? '' : 'Loading YATA…' }}
    </small>
  </div>
  <Teleport to="body">
    <div id="xanax-buy-calculator" ref="calculator" popover="auto" role="dialog" class="buy-calculator" aria-labelledby="buy-calculator-title" @toggle="calculatorOpen = calculator?.matches(':popover-open') ?? false; positionCalculator()">
      <div class="calculator-heading">
        <h2 id="buy-calculator-title">Xanax buy cost</h2>
        <button type="button" class="calculator-close" aria-label="Close calculator" @click="calculator?.hidePopover()"><X :size="16" /></button>
      </div>
      <p class="calculator-price">Live: <strong>{{ quote ? `$${numbers.format(quote.cost)}` : '—' }}</strong> each</p>
      <div class="calculator-quantity">
        <label for="xanax-buy-quantity">Buy qty</label>
        <input id="xanax-buy-quantity" v-model="buyQuantity" type="number" min="1" step="1" autofocus @input="copyStatus = ''" />
      </div>
      <p v-if="totalCost === null" class="calculator-validation">Enter a valid whole quantity of at least 1.</p>
      <div class="calculator-total">
        <div><span>Total cost</span><output for="xanax-buy-quantity">{{ totalCost === null ? '—' : `$${numbers.format(totalCost)}` }}</output></div>
        <button type="button" class="calculator-copy" :disabled="totalCost === null" aria-label="Copy total cost" title="Copy total cost" @click="copyCost"><Copy :size="15" /></button>
      </div>
      <p v-if="failed" class="calculator-validation">YATA update failed. Using the last received price.</p>
      <p v-if="copyStatus" class="calculator-copy-status" role="status">{{ copyStatus }}</p>
    </div>
  </Teleport>
</template>

<style scoped>
.xanax-ticker { --ticker-muted: #dedede; --ticker-symbol: #ffe29a; color: #fff; text-align: right; text-shadow: none; }
:global(html[data-theme="discord"] .xanax-ticker) { --ticker-muted: #c8cbd1; --ticker-symbol: #c4caff; color: #f2f3f5; }
.ticker-values { display: flex; align-items: center; justify-content: flex-end; gap: 10px; flex-wrap: wrap; font-variant-numeric: tabular-nums; }
.ticker-symbol { color: var(--ticker-symbol); font-size: 11px; font-weight: 700; letter-spacing: .06em; }
.ticker-price { display: inline-flex; gap: 3px; align-items: center; border: 0; background: transparent; color: inherit; border-radius: 4px; padding: 4px 6px; font-family: ui-monospace, monospace; font-size: 16px; font-weight: 700; }
.ticker-price:hover:not(:disabled) { outline: 1px solid currentColor; }
.ticker-price > span { width: 10px; font-size: 13px; }
.ticker-quantity { color: var(--ticker-muted); font-size: 11px; }
.ticker-quantity b { color: #fff; font-family: ui-monospace, monospace; font-size: 13px; }
.ticker-updated { display: block; color: var(--ticker-muted); margin-top: 3px; font-size: 11px; }
.warning { color: #ffda85; }
.down { color: #9bf2bd; animation: price-drop 2.8s ease-out; }
.up { color: #ffb8b8; animation: price-rise 2.8s ease-out; }
@keyframes price-drop { from { background: rgb(60 190 110 / 24%); } to { background: transparent; } }
@keyframes price-rise { from { background: rgb(230 80 80 / 24%); } to { background: transparent; } }
@media (prefers-reduced-motion: reduce) { .down, .up { animation: none; } }
@media (max-width: 600px) { .xanax-ticker { text-align: left; } .ticker-values { justify-content: flex-start; } }
.buy-calculator { position: fixed; inset: auto; margin: 0; width: min(240px, calc(100% - 24px)); max-height: calc(100dvh - 24px); overflow: auto; padding: 10px; border: 1px solid #aaa; border-radius: 4px; background: linear-gradient(#fafafa, #e9e9e9); color: #333; box-shadow: 0 5px 18px #0005; text-align: left; }
.buy-calculator::backdrop { background: transparent; pointer-events: none; }
.calculator-heading { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-bottom: 2px; }
.calculator-heading h2 { margin: 0; font-size: 12px; color: inherit; }
.calculator-close { display: grid; place-items: center; padding: 4px; border: 0; border-radius: 4px; color: inherit; background: transparent; }
.buy-calculator p { font-size: 11px; line-height: 1.35; }
.calculator-price { margin: 0 0 8px; }
.calculator-quantity { display: flex; align-items: center; justify-content: space-between; gap: 8px; }
.buy-calculator label { font-size: 11px; font-weight: 700; }
.buy-calculator input { width: 110px; height: 28px; padding: 4px 6px; font-size: 12px; border: 1px solid var(--border); border-radius: 4px; }
.calculator-total { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-top: 8px; }
.calculator-total span { display: inline; font-size: 11px; margin-right: 6px; }
.calculator-total output { font-size: 14px; font-weight: 700; overflow-wrap: anywhere; }
.calculator-total > div { min-width: 0; }
.calculator-copy { display: inline-flex; align-items: center; justify-content: center; flex-shrink: 0; width: 28px; height: 28px; padding: 4px; border: 0; border-radius: 4px; background: #9c7a1e; color: #fff; }
.calculator-copy-status { margin: 8px 0 0; }
.calculator-validation { color: #ae2e24; }
:global(html[data-theme="discord"] .buy-calculator) { background: #313338; color: #f2f3f5; border-color: #3f4147; }
:global(html[data-theme="discord"] .calculator-heading h2) { color: #f2f3f5; }
:global(html[data-theme="discord"] .calculator-copy) { background: #5865f2; }
:global(html[data-theme="discord"] .calculator-validation) { color: #ffb8b8; }
</style>
