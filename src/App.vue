<script setup>
import { ref, watch, onMounted, onUnmounted } from 'vue'

import PackageSelector from './components/PackageSelector.vue'
import AiVisual from './components/AiVisual.vue'
import AlaCart from './components/AlaCart.vue'
import RenovationAssistant from './components/RenovationAssistant.vue'
import RenovationQuotation from './components/RenovationQuotation.vue'

// Shared quotation state
const selectedPackage = ref(null)
const selectedShopItems = ref([])
const packageCreditSavings = ref(0)

// Property details
const selectedProperty = ref('')
const selectedPropertyIsCustom = ref(false)
const selectedSize = ref('')
const selectedBedrooms = ref('')


// ---------------------------------------------------------------
// BRIDGE: PackageSelector and AlaCart are legacy components. They keep
// their data in window.* and announce changes with CustomEvents, so
// v-model never fires. We listen to those events here and copy the
// values into the refs that RenovationQuotation reads.
// ---------------------------------------------------------------
const cloneItems = items => (items || []).map(i => ({ ...i }))

function onPackage(e) {
  const d = e.detail
  selectedPackage.value = d ? { ...d } : null
  if (d?.property !== undefined) selectedProperty.value = d.property || ''
  if (d?.propertyIsCustom !== undefined) selectedPropertyIsCustom.value = !!d.propertyIsCustom
}
function onProperty(e) {
  const d = e.detail || {}
  selectedPropertyIsCustom.value = !!d.isCustom
  selectedProperty.value = d.isCustom ? (d.customProperty || '') : (d.property || '')
  selectedSize.value = d.size || ''
  selectedBedrooms.value = d.bedrooms || ''
}
function onCustomName(e) {
  selectedProperty.value = e.detail?.property || ''
  selectedPropertyIsCustom.value = true
}
function onCart(e) {
  selectedShopItems.value = cloneItems(e.detail?.items)
}

// Register before children mount so no early event is missed
window.addEventListener('packageChange', onPackage)
window.addEventListener('propertySelected', onProperty)
window.addEventListener('customPropertyNameChanged', onCustomName)
window.addEventListener('cart:updated', onCart)

onMounted(() => {
  // Pick up anything that was set before App mounted
  if (window.selectedPackage) selectedPackage.value = { ...window.selectedPackage }
  if (window.selectedShopItems?.length) selectedShopItems.value = cloneItems(window.selectedShopItems)
  if (window.selectedProperty) selectedProperty.value = window.selectedProperty
  selectedPropertyIsCustom.value = !!window.selectedPropertyIsCustom
  if (window.selectedSize) selectedSize.value = window.selectedSize
  if (window.selectedBedrooms) selectedBedrooms.value = window.selectedBedrooms
})
onUnmounted(() => {
  window.removeEventListener('packageChange', onPackage)
  window.removeEventListener('propertySelected', onProperty)
  window.removeEventListener('customPropertyNameChanged', onCustomName)
  window.removeEventListener('cart:updated', onCart)
})

// Quotation -> catalog: when qty is changed/removed in the quotation,
// push it back to AlaCart so its cards and counters stay in sync.
watch(selectedShopItems, items => {
  window.selectedShopItems = cloneItems(items)
  window.dispatchEvent(new CustomEvent('cart:external'))
}, { deep: true })

// Reset quotation after successful submission
function resetQuotation() {
  selectedPackage.value = null
  selectedShopItems.value = []
  packageCreditSavings.value = 0
  selectedProperty.value = ''
  selectedPropertyIsCustom.value = false
  selectedSize.value = ''
  selectedBedrooms.value = ''
  window.selectedShopItems = []
  window.dispatchEvent(new CustomEvent('cart:external'))
}
</script>

<template>
  <div class="min-h-screen bg-[#faf9f6]">

    <!-- NAVBAR -->
    <header
      class="sticky top-0 z-50 bg-[#faf9f6] border-b border-stone-200/80 shadow-sm transition-all"
    >
      <div class="max-w-7xl mx-auto px-6 h-20 flex items-center justify-between">

        <div class="flex items-center gap-3">
          <span class="text-xl font-bold tracking-tight uppercase font-serif text-stone-900">
            NextGen
          </span>
        </div>

        <nav class="hidden md:flex items-center gap-8 text-sm font-medium text-stone-600">
          <a href="#home" class="hover:text-[#4a5d45] transition">
            Dashboard
          </a>

          <a href="#package-section" class="hover:text-[#4a5d45] transition">
            Packages
          </a>

          <a href="#ai-section" class="hover:text-[#4a5d45] transition">
            3D Designer
          </a>

          <a href="#alacart-section" class="hover:text-[#4a5d45] transition">
            A La Carte
          </a>

          <a href="#quotation-section" class="hover:text-[#4a5d45] transition">
            Quotation
          </a>
        </nav>

        <div>
          <a
            href="#package-section"
            class="bg-[#4a5d45] hover:bg-[#3b4a37] text-white px-5 py-2.5 rounded-lg text-sm font-medium shadow-sm transition"
          >
            View Packages
          </a>
        </div>

      </div>
    </header>

    <!-- HOME LANDING SECTION -->
    <section
      id="home"
      class="relative bg-[#faf9f6] py-24 px-6 border-b border-stone-200 overflow-hidden"
    >
      <div class="max-w-7xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-12 items-center">

        <div class="space-y-6">

          <span
            class="inline-block bg-[#4a5d45]/10 text-[#4a5d45] text-xs font-semibold tracking-wider uppercase px-3.5 py-1.5 rounded-full"
          >
            Renovation Portal 2026
          </span>

          <h1 class="text-4xl sm:text-6xl font-serif font-bold text-stone-900 leading-tight">
            Your
            <span class="italic font-normal text-[#4a5d45]">Property</span>,
            Our
            <span class="italic font-normal text-[#4a5d45]">Care</span>
          </h1>

          <p class="text-lg text-stone-600 max-w-xl">
            Powered by Vue.js — tailor your property fit-out,
            review real-time pricing, and finalize your submission seamlessly.
          </p>

          <div class="flex flex-wrap gap-4 pt-2">

            <a
              href="#package-section"
              class="bg-[#4a5d45] hover:bg-[#3b4a37] text-white px-7 py-3.5 rounded-xl font-medium shadow transition"
            >
              Choose Package
            </a>

            <a
              href="#ai-section"
              class="border border-stone-300 hover:border-stone-400 bg-white text-stone-700 px-7 py-3.5 rounded-xl font-medium transition"
            >
              Open 3D Planner
            </a>

          </div>
        </div>

        <div class="relative">
          <div
            class="aspect-[4/3] rounded-2xl overflow-hidden shadow-xl border border-stone-200 bg-stone-100 flex items-center justify-center p-8"
          >
            <img
              src=""
              alt="NextGen Preview"
              class="max-h-24 opacity-80"
            />
          </div>
        </div>

      </div>
    </section>

    <!-- MAIN CONTENT -->
    <main class="py-12 space-y-16">

      <!-- 1. PACKAGE SELECTOR -->
      <section id="package-section" class="scroll-mt-24">
        <PackageSelector />
      </section>

      <!-- 2. AI VISUAL DESIGNER -->
      <section id="ai-section" class="scroll-mt-24">
        <AiVisual />
      </section>

      <!-- 3. A LA CARTE -->
      <section id="alacart-section" class="scroll-mt-24">
        <AlaCart />
      </section>

      <!-- 4. RENOVATION QUOTATION -->
      <section
        id="quotation-section"
        class="scroll-mt-24 px-6"
      >
        <div class="max-w-7xl mx-auto">

          <div class="mb-8">
            <span
              class="inline-block text-xs font-bold tracking-widest uppercase text-[#4a5d45] mb-3"
            >
              Final Step
            </span>

            <h2 class="text-3xl md:text-4xl font-serif font-bold text-stone-900">
              Your Renovation Quotation
            </h2>

            <p class="text-stone-500 mt-3 max-w-2xl">
              Review your selected package and additional items,
              adjust quantities, and submit your quotation request.
            </p>
          </div>

          <RenovationQuotation
            :selected-package="selectedPackage"
            v-model:selected-shop-items="selectedShopItems"
            :package-credit-savings="packageCreditSavings"
            :selected-property="selectedProperty"
            :selected-property-is-custom="selectedPropertyIsCustom"
            :selected-size="selectedSize"
            :selected-bedrooms="selectedBedrooms"
            @reset="resetQuotation"
          />

        </div>
      </section>

    </main>

    <!-- FOOTER -->
    <footer class="bg-stone-900 text-stone-400 py-16 px-6 border-t border-stone-800">

      <div class="max-w-7xl mx-auto grid grid-cols-1 md:grid-cols-3 gap-10 text-sm">

        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">
            Owner Advisory & Support
          </h4>

          <p>
            Dedicated Hotline:
            <strong class="text-white">+603 2181 3455</strong>
          </p>

          <p>
            Concierge Email:
            <a
              href="mailto:inquiry@nextgen.com"
              class="text-white underline underline-offset-4"
            >
              inquiry@nextgen.com
            </a>
          </p>
        </div>

        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">
            Corporate Headquarters
          </h4>

          <p>
            NextGen Sdn Bhd<br />
            Unit 01-01, First Floor, Micasa Shoppes,
            368-A, Jalan Tun Razak, 50400 Kuala Lumpur
          </p>
        </div>

        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">
            Legal
          </h4>

          <p>
            © 2026 NextGen Sdn Bhd. All rights reserved.
            <br />
            <a href="#" class="hover:underline">Terms of Service</a>
            |
            <a href="#" class="hover:underline">Privacy Policy</a>
          </p>
        </div>

      </div>
    </footer>

    <!-- FLOATING AI RENOVATION ASSISTANT -->
    <RenovationAssistant
      webhook-url="https://n8n.srv1102332.hstgr.cloud/webhook/reno-assistants"
      :max-files="3"
      :max-file-m-b="10"
      :history-turns="10"
      storage-key="widebed_reno_chat_v1"
    />

  </div>
</template>