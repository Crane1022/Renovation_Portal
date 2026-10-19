<script setup>
import PackageSelector from './components/PackageSelector.vue'
import AiVisual from './components/AiVisual.vue'
import AlaCart from './components/AlaCart.vue'
import RenovationAssistant from './components/RenovationAssistant.vue'
</script>

<template>
  <div class="min-h-screen bg-[#faf9f6]">
    <!-- NAVBAR (#faf9f6 theme) -->
    <header class="sticky top-0 z-50 bg-[#faf9f6] border-b border-stone-200/80 shadow-sm transition-all">
      <div class="max-w-7xl mx-auto px-6 h-20 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <span class="text-xl font-bold tracking-tight uppercase font-serif text-stone-900">WIDEBED</span>
        </div>
        <nav class="hidden md:flex items-center gap-8 text-sm font-medium text-stone-600">
          <a href="#home" class="hover:text-[#4a5d45] transition">Dashboard</a>
          <a href="#package-section" class="hover:text-[#4a5d45] transition">Packages</a>
          <a href="#ai-section" class="hover:text-[#4a5d45] transition">3D Designer</a>
          <a href="#alacart-section" class="hover:text-[#4a5d45] transition">A La Carte</a>
        </nav>
        <div>
          <a href="#package-section" class="bg-[#4a5d45] hover:bg-[#3b4a37] text-white px-5 py-2.5 rounded-lg text-sm font-medium shadow-sm transition">
            View Packages
          </a>
        </div>
      </div>
    </header>
    <!-- HOME LANDING SECTION -->
    <section id="home" class="relative bg-[#faf9f6] py-24 px-6 border-b border-stone-200 overflow-hidden">
      <div class="max-w-7xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-12 items-center">
        <div class="space-y-6">
          <span class="inline-block bg-[#4a5d45]/10 text-[#4a5d45] text-xs font-semibold tracking-wider uppercase px-3.5 py-1.5 rounded-full">
            Renovation Portal 2026
          </span>
          <h1 class="text-4xl sm:text-6xl font-serif font-bold text-stone-900 leading-tight">
            Your <span class="italic font-normal text-[#4a5d45]">Property</span>, Our <span class="italic font-normal text-[#4a5d45]">Care</span>
          </h1>
          <p class="text-lg text-stone-600 max-w-xl">
            Powered by Vue.js — tailor your property fit-out, review real-time pricing, and finalize your submission seamlessly.
          </p>
          <div class="flex flex-wrap gap-4 pt-2">
            <a href="#package-section" class="bg-[#4a5d45] hover:bg-[#3b4a37] text-white px-7 py-3.5 rounded-xl font-medium shadow transition">
              Choose Package
            </a>
            <a href="#ai-section" class="border border-stone-300 hover:border-stone-400 bg-white text-stone-700 px-7 py-3.5 rounded-xl font-medium transition">
              Open 3D Planner
            </a>
          </div>
        </div>
        <div class="relative">
          <div class="aspect-[4/3] rounded-2xl overflow-hidden shadow-xl border border-stone-200 bg-stone-100 flex items-center justify-center p-8">
            <img src="https://images.leadconnectorhq.com/image/f_webp/q_80/r_1200/u_https://assets.cdn.filesafe.space/urgz3ehyjizBb14KYhgn/media/6a14fef0e05851175c85cb0e.png" alt="Widebed Preview" class="max-h-24 opacity-80">
          </div>
        </div>
      </div>
    </section>
    <!-- MAIN CONTAINER FOR VUE DYNAMIC COMPONENTS -->
    <main class="py-12 space-y-16">
      <div id="package-section"><PackageSelector /></div>
      <div id="ai-section"><AiVisual /></div>
      <div id="alacart-section"><AlaCart /></div>
    </main>
    <!-- FOOTER -->
    <footer class="bg-stone-900 text-stone-400 py-16 px-6 border-t border-stone-800">
      <div class="max-w-7xl mx-auto grid grid-cols-1 md:grid-cols-3 gap-10 text-sm">
        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">Owner Advisory & Support</h4>
          <p>Dedicated Hotline: <strong class="text-white">+603 2181 3455</strong></p>
          <p>Concierge Email: <a href="mailto:inquiry@widebed.com" class="text-white underline underline-offset-4">inquiry@widebed.com</a></p>
        </div>
        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">Corporate Headquarters</h4>
          <p>Widebed Sdn Bhd<br>Unit 01-01, First Floor, Micasa Shoppes, 368-A, Jalan Tun Razak, 50400 Kuala Lumpur</p>
        </div>
        <div class="space-y-3">
          <h4 class="text-white font-semibold uppercase tracking-wider text-xs">Legal</h4>
          <p>© 2026 Widebed Sdn Bhd. All rights reserved.<br><a href="#" class="hover:underline">Terms of Service</a> | <a href="#" class="hover:underline">Privacy Policy</a></p>
        </div>
      </div>
    </footer>

    <!-- AI RENOVATION ASSISTANT -->
    <!-- Keep this outside main so the chat floats above every page section. -->
    <RenovationAssistant
      webhook-url="https://n8n.srv1102332.hstgr.cloud/webhook/reno-assistants"
      :max-files="3"
      :max-file-m-b="10"
      :history-turns="10"
      storage-key="widebed_reno_chat_v1"
    />
  </div>
</template>
