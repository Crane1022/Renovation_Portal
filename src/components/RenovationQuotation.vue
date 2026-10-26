<script setup>
import { computed, ref, watch, onMounted, onUnmounted, nextTick } from 'vue'

// Parent owns the cart. v-model bindings keep the catalog and quotation in sync.
const props = defineProps({
  selectedPackage: { type: Object, default: null },
  selectedShopItems: { type: Array, default: () => [] },
  packageCreditSavings: { type: Number, default: 0 },
  selectedProperty: { type: String, default: '' },
  selectedPropertyIsCustom: { type: Boolean, default: false },
  selectedSize: { type: [String, Number], default: '' },
  selectedBedrooms: { type: [String, Number], default: '' },
  submitUrl: { type: String, default: '/api/quotation' },
  sheetId: { type: String, default: '' },
  packageTab: { type: String, default: 'Packages' },
  productsTab: { type: String, default: 'Products' },
  resetAfterPrint: { type: Boolean, default: false }
})
const emit = defineEmits(['update:selectedShopItems', 'submitted', 'reset'])
const MAX_QTY = 999
const formatRM = val => 'RM ' + (Number(val) || 0).toLocaleString('en-MY', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
const toNum = v => Number(String(v ?? '').replace(/[^0-9.\-]/g, '')) || 0
const getQty = it => Number(it.quantity ?? it.qty ?? 1) || 0
const pkgTotal = computed(() => toNum(props.selectedPackage?.numericPrice ?? props.selectedPackage?.price))
const visibleItems = computed(() => props.selectedShopItems.map((item, idx) => ({ item, idx, qty: getQty(item), price: toNum(item.price) })).filter(x => x.qty > 0))
const itemsTotal = computed(() => visibleItems.value.reduce((sum, x) => sum + x.qty * x.price, 0))
const itemCount = computed(() => visibleItems.value.reduce((sum, x) => sum + x.qty, 0))
const grandTotal = computed(() => Math.max(0, pkgTotal.value + itemsTotal.value - Number(props.packageCreditSavings || 0)))
const refNo = ref('')
const customer = ref({ name: '', phone: '', email: '' })
const error = ref('')
const status = ref('')
const submitting = ref(false)
const lockOpen = ref(false)
const successOpen = ref(false)
const lockAnswer = ref('')
const lockProduct = ref(null)
const lockFallback = { id: 'P159', name: 'Digital Door Lock + Installation', price: 1200, image: 'https://lh3.googleusercontent.com/d/1YMl6ujN5ZxK2-W5m-dcPJZYXME9pH7Ob', desc: 'Includes installation' }
const lock = computed(() => lockProduct.value || lockFallback)
const printSheet = ref(null)
const lastData = ref(null)
const lastResult = ref(null)
const esc = s => String(s ?? '').replace(/[&<>"']/g, c => ({ '&':'&amp;', '<':'&lt;', '>':'&gt;', '"':'&quot;', "'":'&#39;' }[c]))

function changeQty(idx, raw) {
  const next = props.selectedShopItems.map(x => ({ ...x }))
  if (!next[idx]) return
  let q = Math.floor(Number(raw))
  if (!Number.isFinite(q)) q = getQty(next[idx])
  q = Math.min(MAX_QTY, Math.max(0, q))
  if (q === 0) next.splice(idx, 1)
  else { next[idx].quantity = q; next[idx].qty = q }
  emit('update:selectedShopItems', next)
}
function validate() {
  const c = customer.value
  if (c.name.trim().length < 2) error.value = 'Please enter your name.'
  else if (!/^[+\d][\d\s\-()]{7,24}$/.test(c.phone.trim())) error.value = 'Please enter a valid phone number.'
  else if (c.email && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(c.email.trim())) error.value = 'Please enter a valid email address, or leave it empty.'
  else error.value = ''
  return !error.value
}
function parseCSV(text) {
  const rows = []; let row = [], cur = '', quoted = false
  for (let i = 0; i < text.length; i++) {
    const c = text[i]
    if (quoted) { if (c === '"' && text[i + 1] === '"') { cur += '"'; i++ } else if (c === '"') quoted = false; else cur += c }
    else if (c === '"') quoted = true
    else if (c === ',') { row.push(cur); cur = '' }
    else if (c === '\n' || c === '\r') { if (c === '\r' && text[i + 1] === '\n') i++; row.push(cur); rows.push(row); row = []; cur = '' }
    else cur += c
  }
  if (cur || row.length) { row.push(cur); rows.push(row) }
  const headers = (rows.shift() || []).map(x => x.trim())
  return rows.filter(r => r.some(v => v.trim())).map(r => Object.fromEntries(headers.map((h, i) => [h, (r[i] || '').trim()])))
}
async function fetchTab(tab) {
  if (!props.sheetId) return []
  const url = `https://docs.google.com/spreadsheets/d/${encodeURIComponent(props.sheetId)}/gviz/tq?tqx=out:csv&headers=1&sheet=${encodeURIComponent(tab)}`
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Cannot read ${tab}`)
  return parseCSV(await res.text())
}
async function loadLock() {
  try {
    const products = await fetchTab(props.productsTab)
    const p = products.find(x => x['Product ID']?.trim() === 'P159') || products.find(x => x['Product Name']?.toLowerCase().includes('digital door lock'))
    if (p) lockProduct.value = { id: p['Product ID'] || 'P159', name: p['Product Name'] || lockFallback.name, price: Number(String(p.Price || '').replace(/[^0-9.]/g, '')) || 1200, image: p['Image URL'] || lockFallback.image, desc: p.Specification || lockFallback.desc, category: p.Category || '', subcategory: p.Subcategory || '', unit: p.Unit || 'set' }
  } catch (e) { console.warn('Using fallback digital lock:', e) }
}
function isLock(it) { return String(it.id || it.productId || '') === 'P159' || String(it.name || it.title || '').toLowerCase().includes('digital door lock') }
const hasLock = computed(() => props.selectedShopItems.some(it => isLock(it) && getQty(it) > 0))
let pendingAction = ''
function start(action) {
  if (submitting.value || !validate()) return
  pendingAction = action
  if (!hasLock.value && lockAnswer.value !== 'yes') lockOpen.value = true
  else process(action)
}
function answerLock(answer) {
  lockAnswer.value = answer
  lockOpen.value = false
  if (answer === 'no' && !hasLock.value) {
    const p = lock.value
    emit('update:selectedShopItems', [...props.selectedShopItems, { id: p.id, productId: p.id, name: p.name, price: p.price, image: p.image, quantity: 1, qty: 1, category: p.category || '', subcategory: p.subcategory || '', unit: p.unit || 'set' }])
    nextTick(() => process(pendingAction))
  } else process(pendingAction)
}
async function demoImage(pkg) {
  if (!pkg) return ''
  const own = pkg.demoImage || pkg.demoImageUrl || pkg['Demo Image URL']
  if (own) return own
  const id = pkg.packageId || pkg.key
  if (!id || id === 'alacarte') return ''
  try { const rows = await fetchTab(props.packageTab); const p = rows.find(r => r['Package ID'] === String(id)); return p?.['Demo Image URL'] || p?.['Image URL'] || '' } catch { return '' }
}
async function collectData() {
  const pkg = props.selectedPackage
  const pkgTitle = pkg?.title || pkg?.name || pkg?.packageName || 'Custom Fit-Out Package'
  const lines = pkgTotal.value > 0 ? [{ name: 'Selected Base Package', sub: pkgTitle, qty: 1, price: pkgTotal.value, total: pkgTotal.value }] : []
  const addOns = visibleItems.value.map(({item, qty, price}) => ({ id: item.id || item.productId || '', name: item.name || item.title || item['Product Name'] || 'À La Carte Item', qty, price }))
  for (const a of addOns) lines.push({ name: a.name, sub: '', qty: a.qty, price: a.price, total: a.qty * a.price })
  if (!refNo.value) refNo.value = 'WB-' + Math.floor(100000 + Math.random() * 900000)
  return { refNo: refNo.value, today: new Date().toLocaleDateString('en-CA', {timeZone:'Asia/Kuala_Lumpur'}), customer: {...customer.value}, propertyName: props.selectedProperty || pkg?.property || 'Not specified', propertyIsCustom: props.selectedPropertyIsCustom, sizeSqft: props.selectedSize, bedrooms: props.selectedBedrooms, packageId: pkg?.packageId || '', pkgTitle, pkgTotal: pkgTotal.value, itemsTotal: itemsTotal.value, creditAmount: Number(props.packageCreditSavings || 0), grandTotal: grandTotal.value, lines, addOns, itemsText: addOns.map(a => `${a.name} x${a.qty} @ ${a.price}`).join('; '), digitalLock: lockAnswer.value === 'yes' ? 'Has own' : hasLock.value ? 'Added to quotation' : 'Not asked', demoImgUrl: await demoImage(pkg) }
}

function buildQuotationMarkup(d) {
  const TD = 'padding:8px 10px;border-bottom:1px solid #E2E0D8;font-size:11px;vertical-align:top;';
  const META_TD = 'padding:8px 12px;border:1px solid #E2E0D8;vertical-align:top;width:50%;';
  const LABEL = 'font-size:9px;text-transform:uppercase;font-weight:bold;color:#6C756D;display:block;margin-bottom:2px;';
  const VAL = 'font-size:12px;font-weight:bold;color:#2C352B;';

  const c = d.customer;
  const contact = [c.phone, c.email]
    .filter(Boolean)
    .map(esc)
    .join('<br>');

  // ================================
  // QUOTATION ITEMS
  // ================================
  const rowsHtml = d.lines.length
    ? d.lines.map((l, i) => `
      <tr style="background:${i % 2 ? '#FAFAFA' : '#FFFFFF'};">
        <td style="${TD}text-align:center;">${i + 1}</td>

        <td style="${TD}">
          ${l.sub
            ? `<strong>${esc(l.name)}</strong><br>
               <span style="color:#555555;">${esc(l.sub)}</span>`
            : esc(l.name)}
        </td>

        <td style="${TD}text-align:right;">${l.qty}</td>
        <td style="${TD}text-align:right;">${formatRM(l.price)}</td>
        <td style="${TD}text-align:right;">${formatRM(l.total)}</td>
      </tr>
    `).join('')
    : `
      <tr>
        <td colspan="5" style="padding:20px;text-align:center;color:#6C756D;">
          No package or items selected.
        </td>
      </tr>
    `;

  const hasPage2 = !!d.demoImgUrl;

  return `

  <!-- ==========================================
       PAGE 1 - QUOTATION
       ========================================== -->

  <div class="print-page-1" ${hasPage2 ? 'style="page-break-after:always;"' : ''}>

    <!-- COMPANY HEADER -->
    <table style="width:100%;border-collapse:collapse;border-bottom:2px solid #2C352B;margin-bottom:16px;">
      <tr>
        <td style="padding:0 0 12px 0;vertical-align:top;">

          <div style="font-family:Georgia,serif;font-size:18px;font-weight:bold;color:#4A5D45;letter-spacing:0.5px;">
            1 WORLD SHORT STAY SDN BHD
          </div>

          <div style="font-size:10px;color:#555555;margin-top:2px;line-height:1.35;">
            (Wholly Owned Subsidiary By Widebed Sdn Bhd)<br>
            Unit 01-01, 1st Floor Micasa Shoppes,
            368-A Jalan Tun Razak, 50400 Kuala Lumpur<br>
            Tel: 03-2181 3455 |
            Email: inquiry@widebed.com |
            SST# W10-1808-31027737
          </div>

        </td>

        <td style="padding:0 0 12px 0;vertical-align:top;text-align:right;white-space:nowrap;">

          <div style="font-family:Georgia,serif;font-size:22px;font-weight:bold;color:#2C352B;letter-spacing:1px;">
            QUOTATION
          </div>

          <div style="font-size:11px;margin-top:4px;color:#333333;">
            <strong>Ref No:</strong> ${esc(d.refNo)}<br>
            <strong>Date:</strong> ${esc(d.today)}
          </div>

        </td>
      </tr>
    </table>


    <!-- CUSTOMER & PROPERTY INFORMATION -->
    <table style="width:100%;border-collapse:collapse;margin-bottom:16px;background:#F9F9F7;border:1px solid #E2E0D8;">

      <tr>
        <td style="${META_TD}">
          <span style="${LABEL}">Prepared For</span>
          <span style="${VAL}">${esc(c.name)}</span>
        </td>

        <td style="${META_TD}">
          <span style="${LABEL}">Contact</span>
          <span style="${VAL}">${contact || '-'}</span>
        </td>
      </tr>

      <tr>
        <td style="${META_TD}">
          <span style="${LABEL}">Property Name</span>
          <span style="${VAL}">${esc(d.propertyName)}</span>
        </td>

        <td style="${META_TD}">
          <span style="${LABEL}">Package Selected</span>
          <span style="${VAL}">${esc(d.pkgTitle)}</span>
        </td>
      </tr>

    </table>


    <!-- QUOTATION ITEM TABLE -->
    <table style="width:100%;border-collapse:collapse;margin-bottom:16px;">

      <thead>
        <tr>
          <th style="background:#4A5D45;color:#FFFFFF;text-align:center;padding:8px 10px;font-size:10px;text-transform:uppercase;width:30px;">
            #
          </th>

          <th style="background:#4A5D45;color:#FFFFFF;text-align:left;padding:8px 10px;font-size:10px;text-transform:uppercase;">
            Description
          </th>

          <th style="background:#4A5D45;color:#FFFFFF;text-align:right;padding:8px 10px;font-size:10px;text-transform:uppercase;width:50px;">
            Qty
          </th>

          <th style="background:#4A5D45;color:#FFFFFF;text-align:right;padding:8px 10px;font-size:10px;text-transform:uppercase;width:110px;">
            Unit Price (MYR)
          </th>

          <th style="background:#4A5D45;color:#FFFFFF;text-align:right;padding:8px 10px;font-size:10px;text-transform:uppercase;width:110px;">
            Total (MYR)
          </th>
        </tr>
      </thead>

      <tbody>
        ${rowsHtml}
      </tbody>

    </table>


    <!-- PRICE SUMMARY -->
    <table style="width:100%;border-collapse:collapse;margin-bottom:20px;">

      <tr>
        <td style="width:58%;"></td>

        <td style="width:42%;">

          <table style="width:100%;border-collapse:collapse;">

            <!-- SUBTOTAL -->
            <tr>
              <td style="padding:6px 10px;border-bottom:1px solid #E2E0D8;font-size:11px;">
                Subtotal:
              </td>

              <td style="padding:6px 10px;border-bottom:1px solid #E2E0D8;font-size:11px;text-align:right;">
                ${formatRM(d.pkgTotal + d.itemsTotal)}
              </td>
            </tr>

            <!-- PACKAGE CREDITS -->
            ${d.creditAmount > 0 ? `
              <tr>
                <td style="padding:6px 10px;border-bottom:1px solid #E2E0D8;font-size:11px;">
                  Package Credits:
                </td>

                <td style="padding:6px 10px;border-bottom:1px solid #E2E0D8;font-size:11px;text-align:right;color:#C18C5D;">
                  - ${formatRM(d.creditAmount)}
                </td>
              </tr>
            ` : ''}

            <!-- GRAND TOTAL -->
            <tr>
              <td style="padding:8px 10px;font-family:Georgia,serif;font-size:14px;font-weight:bold;background:#F0EFEA;color:#4A5D45;border-top:2px solid #4A5D45;border-bottom:2px solid #4A5D45;">
                GRAND TOTAL:
              </td>

              <td style="padding:8px 10px;font-family:Georgia,serif;font-size:14px;font-weight:bold;background:#F0EFEA;color:#4A5D45;border-top:2px solid #4A5D45;border-bottom:2px solid #4A5D45;text-align:right;">
                ${formatRM(d.grandTotal)}
              </td>
            </tr>

          </table>
        </td>
      </tr>

    </table>


    <!-- ==========================================
         NEW: TERMS & CONDITIONS
         ========================================== -->

    <div style="
      margin-top:14px;
      margin-bottom:16px;
      padding:12px 14px;
      background:#F9F9F7;
      border:1px solid #E2E0D8;
      font-size:9px;
      line-height:1.5;
      color:#555555;
      page-break-inside:avoid;
    ">

      <strong style="
        display:block;
        font-size:10px;
        color:#2C352B;
        margin-bottom:6px;
        letter-spacing:0.4px;
      ">
        TERMS & CONDITIONS
      </strong>

      <ol style="padding-left:16px;margin:0 0 8px 0;">

        <li>
          This quotation is for reference and estimation purposes only.
        </li>

        <li>
          Prices, quantities and specifications are subject to final verification and site inspection.
        </li>

        <li>
          Actual costs may vary depending on property conditions, item availability and additional requirements.
        </li>

        <li>
          Package images and visuals are for illustration purposes only.
        </li>

        <li>
          Any additional work or changes will be discussed and confirmed before proceeding.
        </li>

        <li>
          Final pricing and payment arrangements are subject to management confirmation.
        </li>

      </ol>

      <div style="
        padding-top:6px;
        border-top:1px dashed #E2E0D8;
        font-size:9px;
        color:#6C756D;
      ">
        <strong>Disclaimer:</strong>
        This quotation does not constitute an order confirmation.
        No renovation work will commence until final arrangements have been mutually agreed upon.
      </div>

    </div>


    <!-- ==========================================
         PAYMENT DETAILS & ORDER CONFIRMATION
         ========================================== -->

    <table style="width:100%;border-collapse:separate;border-spacing:0;border-top:1px solid #E2E0D8;margin-top:20px;page-break-inside:avoid;">

      <tr>

        <!-- PAYMENT DETAILS -->
        <td style="width:50%;padding:12px 8px 0 0;vertical-align:top;">

          <div style="
            background:#F9F9F7;
            border:1px solid #E2E0D8;
            padding:10px 12px;
            font-size:10px;
            line-height:1.4;
          ">

            <strong>Terms &amp; Payment Details:</strong><br>

            1) All cheques made payable to:
            <strong>1 World Short Stay Sdn Bhd</strong><br>

            2) Bank:
            <strong>HONG LEONG BANK</strong> |
            Account No:
            <strong>355 0000 5043</strong><br>

            3) Kindly email bank deposit slip/receipt with your
            Name &amp; Property Name for verification.

          </div>

        </td>


        <!-- ORDER CONFIRMATION -->
        <td style="width:50%;padding:12px 0 0 8px;vertical-align:top;">

          <div style="
            border:1px solid #E2E0D8;
            padding:10px 12px;
            font-size:10px;
          ">

            <strong>ORDER CONFIRMATION</strong>

            <table style="
              width:100%;
              border-collapse:collapse;
              margin-top:34px;
              border-top:1px dashed #6C756D;
            ">

              <tr>
                <td style="padding-top:4px;font-size:9px;color:#6C756D;">
                  Confirmed by Owner / Customer
                </td>

                <td style="padding-top:4px;font-size:9px;color:#6C756D;text-align:right;">
                  Date
                </td>
              </tr>

            </table>

          </div>

        </td>

      </tr>

    </table>

  </div>


  <!-- ==========================================
       PAGE 2 - PACKAGE DEMO VISUAL
       ONLY DISPLAY IF IMAGE EXISTS
       ========================================== -->

  ${hasPage2 ? `

    <div class="print-page-2" style="text-align:center;page-break-before:always;">

      <h2 style="
        font-family:Georgia,serif;
        font-size:20px;
        color:#2C352B;
        margin:0 0 8px 0;
      ">
        Package Demo Visual
      </h2>

      <p style="
        font-size:12px;
        color:#6C756D;
        margin:0 0 16px 0;
      ">
        ${esc(d.pkgTitle)} (${esc(d.propertyName)})
      </p>

      <div class="print-demo-img-container">

        <img
          src="${esc(d.demoImgUrl)}"
          alt="Demo Visual"
          width="640"
          style="max-width:100%;"
        />

      </div>

    </div>

  ` : ''}

  `;
}
async function process(action) {
  if (submitting.value) return
  submitting.value = true; status.value = 'Saving your quotation…'
  try {
    const d = await collectData()
    const payload = { action, refNo: d.refNo, customer: d.customer, propertyName: d.propertyName, propertyIsCustom: d.propertyIsCustom, sizeSqft: d.sizeSqft, bedrooms: d.bedrooms, packageId: d.packageId, packageTitle: d.pkgTitle, packagePrice: d.pkgTotal, itemsText: d.itemsText, itemsTotal: d.itemsTotal, credits: d.creditAmount, grandTotal: d.grandTotal, digitalLock: d.digitalLock, items: d.addOns, pdfName: `Quotation-${d.refNo}-${d.customer.name}`, quotationHtml: '<html><head><meta charset="utf-8"><style>@page{size:A4;margin:10mm 12mm;}body{font-family:Arial,sans-serif;font-size:11px;color:#111;}</style></head><body>' + buildQuotationMarkup(d) + '</body></html>' }
    const controller = new AbortController()
    const timeout = setTimeout(() => controller.abort(), 25000)
    let result
    try {
      const res = await fetch(props.submitUrl, { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify(payload), signal:controller.signal })
      result = await res.json()
      if (!res.ok || !result.ok) throw new Error(result.error || 'Submission failed')
    } finally { clearTimeout(timeout) }
    lastData.value = d; lastResult.value = result
    status.value = ''; successOpen.value = true
    emit('submitted', { action, data: d, result })
  } catch (e) { status.value = 'We could not submit your quotation. Please check your connection and try again.'; console.error(e); submitting.value = false }
}
async function finishSuccess() {
  successOpen.value = false
  if (!lastData.value) return
  const d = lastData.value
  if (printSheet.value) printSheet.value.innerHTML = buildQuotationMarkup(d)
  await nextTick()
  if (d.demoImgUrl) await new Promise(resolve => { const img = new Image(); img.onload = resolve; img.onerror = resolve; img.src = d.demoImgUrl })
  window.print()
}
function afterPrint() {
  if (!lastData.value) return
  submitting.value = false
  if (props.resetAfterPrint) emit('reset')
}
onMounted(() => { loadLock(); window.addEventListener('afterprint', afterPrint) })
onUnmounted(() => window.removeEventListener('afterprint', afterPrint))
</script>

<template>
  <div class="quotation-component">
   <div class="quo-layout">
    <!-- LEFT: property + items -->
    <div class="quo-col-left">
    <div v-if="selectedProperty || selectedSize || selectedBedrooms" class="quo-prop-card">
      <div class="quo-prop-item"><span>Property</span><strong>{{ selectedProperty || '-' }}<em v-if="selectedPropertyIsCustom"> (Custom)</em></strong></div>
      <div class="quo-prop-item"><span>Size</span><strong>{{ selectedSize ? selectedSize + ' sqft' : '-' }}</strong></div>
      <div class="quo-prop-item"><span>Bedrooms</span><strong>{{ selectedBedrooms || '-' }}</strong></div>
    </div>
    <div class="quo-table-card">
      <div class="quo-header"><div class="quo-header-title"><h3>Selected Items &amp; Package</h3><p>Itemized breakdown of selected base package and à la carte items. You can change add-on quantities below.</p></div><div class="quo-ref-badge">Ref: {{ refNo || 'Pending' }}</div></div>
      <div class="quo-table-wrap"><table class="quo-table"><thead><tr><th>Item / Package Description</th><th class="center">Qty</th><th class="right">Unit Price</th><th class="right">Total Price</th></tr></thead><tbody>
        <tr v-if="selectedPackage && pkgTotal > 0"><td><div class="quo-item-info"><img class="quo-item-thumb" :src="selectedPackage.img || selectedPackage.image || 'https://placehold.co/100x100?text=Package'" alt="Package"><div><div class="quo-item-name">{{ selectedPackage.title || selectedPackage.name }} <span class="quo-tag-badge quo-badge-pkg">Base Package</span></div><div class="quo-item-desc">{{ selectedPackage.desc || 'Standard bundle fit-out' }}</div></div></div></td><td class="center quo-qty-fixed">1</td><td class="right">{{ formatRM(pkgTotal) }}</td><td class="right strong">{{ formatRM(pkgTotal) }}</td></tr>
        <tr v-for="x in visibleItems" :key="x.item.id || x.item.productId || x.idx"><td><div class="quo-item-info"><img class="quo-item-thumb" :src="x.item.image || 'https://placehold.co/100x100?text=Item'" alt="Item"><div class="quo-item-name">{{ x.item.name || x.item.title }} <span class="quo-tag-badge quo-badge-item">Add-On</span></div></div></td><td class="center"><div class="quo-qty-control"><button type="button" class="quo-qty-btn" :disabled="x.qty <= 1" @click="changeQty(x.idx, x.qty-1)">−</button><input class="quo-qty-input" type="number" min="1" :max="MAX_QTY" :value="x.qty" @change="changeQty(x.idx, $event.target.value)" @keydown.enter="$event.target.blur()"><button type="button" class="quo-qty-btn" :disabled="x.qty >= MAX_QTY" @click="changeQty(x.idx, x.qty+1)">+</button></div><button type="button" class="quo-remove-btn" @click="changeQty(x.idx, 0)">Remove</button></td><td class="right">{{ formatRM(x.price) }}</td><td class="right strong">{{ formatRM(x.price*x.qty) }}</td></tr>
        <tr v-if="!(selectedPackage && pkgTotal > 0) && !visibleItems.length"><td colspan="4" class="empty">No package or items currently selected. Please choose a package or select items from the catalog above.</td></tr>
      </tbody></table></div>
    </div>
    </div>
    <!-- RIGHT: summary + customer details -->
    <div class="quo-col-right">
    <div class="sticky-quotation-summary-card-body">
      <div class="sticky-quotation-summary-row"><span class="sticky-quotation-summary-label">Base Package</span><span class="sticky-quotation-summary-val">{{ formatRM(pkgTotal) }}</span></div>
      <div v-if="packageCreditSavings > 0" class="sticky-quotation-summary-row"><span class="sticky-quotation-summary-label">Package Credits</span><span class="sticky-quotation-summary-val credit">- {{ formatRM(packageCreditSavings) }}</span></div>
      <div class="sticky-quotation-summary-row"><span class="sticky-quotation-summary-label">À La Carte ({{ itemCount }} Items)</span><span class="sticky-quotation-summary-val addon">+ {{ formatRM(itemsTotal) }}</span></div>
      <div v-if="packageCreditSavings > 0" class="sticky-quotation-summary-badge">You saved {{ formatRM(packageCreditSavings) }} with package credits</div>
      <div class="sticky-quotation-summary-total-block"><div class="sticky-quotation-summary-total-label">Estimated Total</div><div class="sticky-quotation-summary-total-val">{{ formatRM(grandTotal) }}</div></div>
      <div class="quo-cust-form"><div class="quo-cust-title">Your Details</div><input v-model="customer.name" class="quo-cust-input" placeholder="Full name *" maxlength="100" autocomplete="name" @input="error = ''"><input v-model="customer.phone" class="quo-cust-input" type="tel" placeholder="Phone number * (e.g. 012-345 6789)" maxlength="25" autocomplete="tel" @input="error = ''"><input v-model="customer.email" class="quo-cust-input" type="email" placeholder="Email (optional)" maxlength="120" autocomplete="email" @input="error = ''"><div v-if="error" class="quo-cust-error" role="alert">{{ error }}</div></div>
      <div class="sticky-quo-action-bar"><button type="button" class="sticky-quo-btn-secondary" :disabled="submitting" @click="start('Print / PDF')">Print / PDF</button><button type="button" class="sticky-quo-btn-primary" :disabled="submitting" @click="start('Book Consultation')">Book Consultation</button></div><div v-if="status" class="quo-save-status" role="status">{{ status }}</div>
    </div>
    </div>
   </div>
    <div v-if="lockOpen" class="dl-backdrop open" role="presentation" @click.self="lockOpen = false"><div class="dl-modal" role="dialog" aria-modal="true" aria-label="Digital door lock"><button type="button" class="dl-close" @click="lockOpen = false">×</button><h3>Do you already have a digital door lock?</h3><p>If not, we can add one to your quotation. It includes installation.</p><div class="dl-product"><img :src="lock.image" alt="Digital door lock"><div><div class="dl-product-name">{{ lock.name }}</div><div class="dl-product-desc">{{ lock.desc }}</div><div class="dl-product-price">{{ formatRM(lock.price) }}</div></div></div><div class="dl-actions"><button class="dl-btn dl-btn-yes" @click="answerLock('yes')">Yes, I have one<small>Continue without it</small></button><button class="dl-btn dl-btn-no" @click="answerLock('no')">No, add to quotation<small>Add 1 digital lock</small></button></div></div></div>
    <div v-if="successOpen" class="dl-backdrop open"><div class="dl-modal success" role="dialog" aria-modal="true" aria-label="Quotation submitted"><div class="succ-icon">✓</div><h3>Quotation Submitted!</h3><p>Thank you, <strong>{{ customer.name }}</strong>. Your quotation request has been recorded successfully. <strong>Our team will contact you as soon as possible.</strong></p><button type="button" class="sticky-quo-btn-primary" @click="finishSuccess">OK, Got It</button></div></div>
    <Teleport to="body"><div id="printableQuotationSheet" ref="printSheet"></div></Teleport>
  </div>
</template>
<style>

:root {
  --quo-primary: #4A5D45;
  --quo-primary-hover: #3b4b37;
  --quo-surface: #FFFFFF;
  --quo-surface-1: #F9F9F7;
  --quo-surface-2: #F0EFEA;
  --quo-border: #E2E0D8;
  --quo-text-main: #2C352B;
  --quo-text-muted: #6C756D;
  --quo-accent: #C18C5D;
  --quo-danger: #B3433B;
  --contentfont: 'Plus Jakarta Sans', system-ui, -apple-system, sans-serif;
  --headlinefont: 'Playfair Display', Georgia, serif;
}

.quo-table-card {
  background: var(--quo-surface);
  border: 1px solid var(--quo-border);
  border-radius: 16px;
  box-shadow: 0 8px 24px rgba(0,0,0,0.05);
  overflow: hidden;
  font-family: var(--contentfont);
  color: var(--quo-text-main);
  margin-bottom: 20px;
}

.quo-header {
  background: var(--quo-surface-1);
  padding: 20px 24px;
  border-bottom: 1px solid var(--quo-border);
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
}

.quo-header-title h3 {
  font-family: var(--headlinefont);
  font-size: 22px;
  margin: 0 0 4px 0;
  color: var(--quo-text-main);
}

.quo-header-title p {
  font-size: 13px;
  color: var(--quo-text-muted);
  margin: 0;
}

.quo-ref-badge {
  background: var(--quo-surface-2);
  border: 1px solid var(--quo-border);
  padding: 6px 14px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: 600;
  color: var(--quo-primary);
}

.quo-table-wrap {
  width: 100%;
  overflow-x: auto;
}

.quo-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
}

.quo-table th {
  background: var(--quo-surface-2);
  color: var(--quo-text-muted);
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  padding: 14px 24px;
  border-bottom: 1px solid var(--quo-border);
}

.quo-table td {
  padding: 16px 24px;
  border-bottom: 1px solid var(--quo-border);
  font-size: 14px;
  vertical-align: middle;
}

.quo-item-thumb {
  width: 48px;
  height: 48px;
  border-radius: 8px;
  object-fit: cover;
  background: var(--quo-surface-2);
  margin-right: 12px;
  flex-shrink: 0;
}

.quo-item-info {
  display: flex;
  align-items: center;
}

.quo-item-name {
  font-weight: 600;
  color: var(--quo-text-main);
}

.quo-item-desc {
  font-size: 12px;
  color: var(--quo-text-muted);
  margin-top: 2px;
}

.quo-tag-badge {
  font-size: 10px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 10px;
  display: inline-block;
  margin-left: 8px;
}

.quo-badge-pkg {
  background: rgba(74, 93, 69, 0.12);
  color: var(--quo-primary);
}

.quo-badge-item {
  background: rgba(193, 140, 93, 0.15);
  color: var(--quo-accent);
}

/* ---------- Editable quantity ---------- */
.quo-qty-control {
  display: inline-flex;
  align-items: center;
  border: 2px solid var(--quo-border);
  border-radius: 10px;
  background: var(--quo-surface);
  overflow: hidden;
  transition: border-color .2s ease;
}
.quo-qty-control:focus-within,
.quo-qty-control:hover { border-color: var(--quo-primary); }

.quo-qty-btn {
  width: 32px;
  height: 34px;
  border: none;
  background: var(--quo-surface-1);
  color: var(--quo-primary);
  font-size: 18px;
  font-weight: 700;
  line-height: 1;
  cursor: pointer;
  font-family: inherit;
  transition: background .2s ease;
}
.quo-qty-btn:hover { background: var(--quo-surface-2); }
.quo-qty-btn:disabled { color: var(--quo-text-muted); cursor: not-allowed; opacity: .5; }

.quo-qty-input {
  width: 48px;
  height: 34px;
  border: none;
  border-left: 1px solid var(--quo-border);
  border-right: 1px solid var(--quo-border);
  text-align: center;
  font-family: var(--contentfont);
  font-size: 14px;
  font-weight: 600;
  color: var(--quo-text-main);
  background: var(--quo-surface);
  outline: none;
  -moz-appearance: textfield;
  appearance: textfield;
}
.quo-qty-input::-webkit-outer-spin-button,
.quo-qty-input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }

.quo-remove-btn {
  display: block;
  margin: 6px auto 0;
  background: none;
  border: none;
  padding: 0;
  font-family: var(--contentfont);
  font-size: 11px;
  font-weight: 600;
  color: var(--quo-text-muted);
  cursor: pointer;
  text-decoration: underline;
}
.quo-remove-btn:hover { color: var(--quo-danger); }

.quo-qty-fixed { font-weight: 600; }

@media (max-width: 600px) {
  .quo-table th, .quo-table td { padding: 12px 14px; }
}


:root {
  --brand-primary: #4A5D45;
  --brand-primary-hover: #3b4b37;
  --brand-surface: #FFFFFF;
  --brand-surface-1: #F9F9F7;
  --brand-surface-2: #F0EFEA;
  --brand-border: #E2E0D8;
  --brand-text-main: #2C352B;
  --brand-text-muted: #6C756D;
  --brand-accent: #C18C5D;
  --brand-danger: #B3433B;
  --contentfont: 'Plus Jakarta Sans', system-ui, -apple-system, sans-serif;
  --headlinefont: 'Playfair Display', Georgia, serif;
}

.sticky-quotation-summary-card-body {
  width: 100%;
  max-width: 100%;
  overflow: hidden;
  box-sizing: border-box;
  margin-bottom: 20px;
  background: var(--brand-surface);
  border: 1px solid var(--brand-border);
  border-radius: 16px;
  padding: 24px;
  box-shadow: 0 8px 24px rgba(0,0,0,0.05);
  font-family: var(--contentfont);
}

.sticky-quotation-summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  font-size: 13.5px;
}

.sticky-quotation-summary-label { color: var(--brand-text-muted); }
.sticky-quotation-summary-val { color: var(--brand-text-main); font-weight: 600; }
.sticky-quotation-summary-val.credit { color: var(--brand-accent); }
.sticky-quotation-summary-val.addon { color: var(--brand-primary); }

.sticky-quotation-summary-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: rgba(193,140,93,0.12);
  border: 1px solid rgba(193,140,93,0.3);
  padding: 6px 12px;
  border-radius: 20px;
  color: var(--brand-accent);
  font-size: 12.5px;
  font-weight: 600;
  margin: 12px 0 6px 0;
}

.sticky-quotation-summary-badge svg {
  width: 14px;
  height: 14px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
}

.sticky-quotation-summary-total-block {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px dashed var(--brand-border);
}

.sticky-quotation-summary-total-label {
  font-size: 12px;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: var(--brand-text-muted);
}

.sticky-quotation-summary-total-val {
  font-family: var(--headlinefont);
  font-size: 38px;
  font-weight: 600;
  color: var(--brand-text-main);
  line-height: 1.1;
  margin-top: 4px;
}

/* Customer Details Form */
.quo-cust-form {
  margin-top: 18px;
  padding-top: 16px;
  border-top: 1px dashed var(--brand-border);
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.quo-cust-title {
  font-size: 12px;
  letter-spacing: 1px;
  text-transform: uppercase;
  color: var(--brand-text-muted);
  font-weight: 700;
}
.quo-cust-input {
  width: 100%;
  box-sizing: border-box;
  padding: 11px 14px;
  font-family: var(--contentfont);
  font-size: 13.5px;
  font-weight: 500;
  color: var(--brand-text-main);
  background: var(--brand-surface);
  border: 2px solid var(--brand-border);
  border-radius: 10px;
  outline: none;
  transition: border-color .2s ease, box-shadow .2s ease;
}
.quo-cust-input:focus { border-color: var(--brand-primary); box-shadow: 0 2px 8px rgba(74,93,69,.12); }
.quo-cust-input.invalid { border-color: var(--brand-danger); }
.quo-cust-error { display: none; font-size: 12.5px; color: var(--brand-danger); font-weight: 600; }
.quo-save-status { display: none; font-size: 12px; color: var(--brand-text-muted); margin-top: 10px; }

.sticky-quo-action-bar {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.sticky-quo-btn-primary {
  flex: 1;
  background: var(--brand-primary);
  color: #FFFFFF;
  border: none;
  padding: 12px 16px;
  border-radius: 8px;
  font-size: 13.5px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  text-align: center;
}

.sticky-quo-btn-primary:hover { background: var(--brand-primary-hover); }

.sticky-quo-btn-secondary {
  background: var(--brand-surface-1);
  color: var(--brand-text-main);
  border: 1px solid var(--brand-border);
  padding: 12px 14px;
  border-radius: 8px;
  font-size: 13.5px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.sticky-quo-btn-secondary:hover { background: var(--brand-surface-2); }
.sticky-quo-action-bar button:disabled { opacity: .6; cursor: wait; }

/* Modals */
.dl-backdrop {
  position: fixed;
  top: 0; left: 0;
  width: 100vw; height: 100vh;
  background: rgba(30, 38, 29, 0.6);
  backdrop-filter: blur(4px);
  z-index: 99998;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  box-sizing: border-box;
  opacity: 0;
  pointer-events: none;
  transition: opacity .25s ease;
}
.dl-backdrop.open { opacity: 1; pointer-events: auto; }

.dl-modal {
  position: relative;
  width: 100%;
  max-width: 460px;
  max-height: 92vh;
  overflow-y: auto;
  background: var(--brand-surface);
  border: 1px solid var(--brand-border);
  border-radius: 18px;
  box-shadow: 0 20px 50px rgba(0,0,0,.25);
  padding: 26px 24px 22px;
  font-family: var(--contentfont);
  color: var(--brand-text-main);
  box-sizing: border-box;
  transform: translateY(12px) scale(.98);
  transition: transform .25s ease;
}
.dl-backdrop.open .dl-modal { transform: translateY(0) scale(1); }

.dl-close {
  position: absolute;
  top: 10px; right: 12px;
  background: none;
  border: none;
  font-size: 26px;
  line-height: 1;
  color: var(--brand-text-muted);
  cursor: pointer;
  padding: 4px 8px;
}
.dl-close:hover { color: var(--brand-text-main); }

.dl-modal h3 {
  font-family: var(--headlinefont);
  font-size: 21px;
  margin: 0 28px 6px 0;
  color: var(--brand-text-main);
}
.dl-modal > p {
  margin: 0 0 16px;
  font-size: 13px;
  color: var(--brand-text-muted);
  line-height: 1.5;
}

.dl-product {
  display: flex;
  gap: 14px;
  align-items: center;
  background: var(--brand-surface-1);
  border: 1px solid var(--brand-border);
  border-radius: 12px;
  padding: 12px;
  margin-bottom: 18px;
}
.dl-product img {
  width: 84px;
  height: 84px;
  border-radius: 10px;
  object-fit: cover;
  background: var(--brand-surface-2);
  flex-shrink: 0;
}
.dl-product-name { font-weight: 700; font-size: 14px; line-height: 1.3; }
.dl-product-desc { font-size: 12px; color: var(--brand-text-muted); margin-top: 2px; }
.dl-product-price {
  font-family: var(--headlinefont);
  font-size: 20px;
  font-weight: 700;
  color: var(--brand-primary);
  margin-top: 6px;
}

.dl-actions { display: flex; gap: 10px; }
.dl-btn {
  flex: 1;
  padding: 12px 10px;
  border-radius: 10px;
  font-family: inherit;
  font-size: 13.5px;
  font-weight: 600;
  cursor: pointer;
  transition: all .2s ease;
  line-height: 1.3;
}
.dl-btn small { display: block; font-size: 11px; font-weight: 500; opacity: .85; margin-top: 2px; }
.dl-btn-yes { background: var(--brand-surface-1); color: var(--brand-text-main); border: 1px solid var(--brand-border); }
.dl-btn-yes:hover { background: var(--brand-surface-2); }
.dl-btn-no { background: var(--brand-primary); color: #fff; border: 1px solid var(--brand-primary); }
.dl-btn-no:hover { background: var(--brand-primary-hover); }

/* Success Icon */
.succ-icon {
  width: 56px;
  height: 56px;
  background: rgba(74, 93, 69, 0.12);
  color: var(--brand-primary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 16px auto;
}
.succ-icon svg { width: 28px; height: 28px; fill: none; stroke: currentColor; stroke-width: 2.5; stroke-linecap: round; stroke-linejoin: round; }

@media (max-width: 480px) {
  .dl-actions { flex-direction: column; }
}

/* Print Sheet Styles */
#printableQuotationSheet { display: none; }

@page {
  size: A4 portrait;
  margin: 10mm 12mm;
}

@media print {
  body > *:not(#printableQuotationSheet) {
    display: none !important;
  }

  #printableQuotationSheet {
    display: block !important;
    position: relative !important;
    width: 100% !important;
    max-width: 210mm !important;
    margin: 0 auto !important;
    padding: 0 !important;
    background: #FFFFFF !important;
    color: #111111 !important;
    font-family: 'Plus Jakarta Sans', Arial, sans-serif !important;
    font-size: 11px !important;
    line-height: 1.4 !important;
    box-sizing: border-box !important;
    overflow: visible !important;
  }

  .print-page-2 { min-height: 250mm; }

  .print-demo-img-container img {
    max-width: 100% !important;
    max-height: 210mm !important;
    width: auto !important;
    height: auto !important;
    object-fit: contain;
    border-radius: 6px;
    border: 1px solid #E2E0D8;
    display: inline-block;
  }
}

.quotation-component .center{text-align:center}.quotation-component .right{text-align:right}.quotation-component .strong{font-weight:600}.quotation-component .empty{text-align:center;padding:36px;color:var(--quo-text-muted)}.quotation-component .quo-cust-error,.quotation-component .quo-save-status{display:block}.quotation-component .success{text-align:center}.quotation-component .success .sticky-quo-btn-primary{width:100%;margin-top:12px}
@media print { body > *:not(#printableQuotationSheet){display:none!important} #printableQuotationSheet{display:block!important} }

/* ---------- Two-column layout ---------- */
.quotation-component .quo-layout{display:grid;grid-template-columns:minmax(0,1.7fr) minmax(320px,1fr);gap:24px;align-items:start}
.quotation-component .quo-col-left,.quotation-component .quo-col-right{min-width:0}
.quotation-component .quo-col-right{position:sticky;top:96px}
.quotation-component .quo-col-left .quo-table-card,.quotation-component .quo-col-right .sticky-quotation-summary-card-body{margin-bottom:0}
.quotation-component .quo-prop-card{margin-bottom:20px}
@media (max-width:960px){
  .quotation-component .quo-layout{grid-template-columns:1fr}
  .quotation-component .quo-col-right{position:static}
  .quotation-component .quo-col-left .quo-table-card{margin-bottom:20px}
}

/* ---------- Property summary ---------- */
.quo-prop-card{display:grid;grid-template-columns:repeat(auto-fit,minmax(150px,1fr));gap:12px;background:var(--quo-surface-1);border:1px solid var(--quo-border);border-radius:16px;padding:16px 20px;font-family:var(--contentfont)}
.quo-prop-item span{display:block;font-size:11px;text-transform:uppercase;letter-spacing:.06em;color:var(--quo-text-muted);margin-bottom:2px}
.quo-prop-item strong{font-size:14px;color:var(--quo-text-main)}
.quo-prop-item em{font-style:normal;color:var(--quo-accent);font-weight:600}
</style>