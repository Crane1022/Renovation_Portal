<template>
  <div class="rp-root" id="room-planner">
    <div class="rp-grid">
      <aside class="rp-card rp-panel rp-catalogue">
        <div class="rp-panel-head">
          <h3>Furniture</h3>
          <span class="rp-sub" data-rp="count"></span>
        </div>
        <div class="rp-search">
          <input type="search" class="rp-input" data-rp="search" placeholder="Search furniture…" aria-label="Search furniture" autocomplete="off">
        </div>
        <div class="rp-chips" data-rp="chips" role="group" aria-label="Filter by category"></div>
        <div class="rp-products" data-rp="products"></div>
      </aside>
      <section class="rp-stage">
        <div class="rp-card rp-roomcfg">
          <div class="rp-roomcfg-title">Room size (metres)</div>
          <div class="rp-field"><label for="rp-w">Width</label><input type="number" id="rp-w" class="rp-input" data-rp="inW" step="0.1" inputmode="decimal"></div>
          <div class="rp-field"><label for="rp-l">Length</label><input type="number" id="rp-l" class="rp-input" data-rp="inL" step="0.1" inputmode="decimal"></div>
          <div class="rp-field"><label for="rp-h">Height</label><input type="number" id="rp-h" class="rp-input" data-rp="inH" step="0.1" inputmode="decimal"></div>
        </div>
        <div class="rp-viewer" data-rp="viewer" tabindex="0" aria-label="3D room preview. Click furniture to select, drag to move.">
          <canvas class="rp-canvas" data-rp="canvas"></canvas>
          <div class="rp-dims" data-rp="dims"></div>
          <div class="rp-overlay-top">
            <button type="button" class="rp-btn rp-btn-sm" data-act="snap" aria-pressed="true">Snap: On</button>
            <button type="button" class="rp-btn rp-btn-sm" data-act="reset-camera">
              <svg viewBox="0 0 24 24"><path d="M3 12a9 9 0 1 0 3-6.7"/><polyline points="3 4 3 9 8 9"/></svg>Reset view
            </button>
          </div>
          <div class="rp-selbar" data-rp="selbar" hidden>
            <span class="rp-selname" data-rp="selname"></span>
            <button type="button" class="rp-btn rp-btn-sm" data-act="rot-left" aria-label="Rotate left 15 degrees" title="Rotate left 15°">
              <svg viewBox="0 0 24 24"><path d="M3 12a9 9 0 1 1 3 6.7"/><polyline points="3 20 3 15 8 15"/></svg>15°
            </button>
            <button type="button" class="rp-btn rp-btn-sm" data-act="rot-right" aria-label="Rotate right 15 degrees" title="Rotate right 15°">
              <svg viewBox="0 0 24 24"><path d="M21 12a9 9 0 1 0-3 6.7"/><polyline points="21 20 21 15 16 15"/></svg>15°
            </button>
            <button type="button" class="rp-btn rp-btn-sm" data-act="rot-90" aria-label="Rotate 90 degrees" title="Rotate 90°">90°</button>
            <button type="button" class="rp-btn rp-btn-sm rp-btn-danger" data-act="delete-selected">
              <svg viewBox="0 0 24 24"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6"/><path d="M10 11v6M14 11v6"/><path d="M9 6V4a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2"/></svg>Delete
            </button>
          </div>
          <div class="rp-toast" data-rp="toast" role="status" aria-live="polite"></div>
          <div class="rp-status" data-rp="status"><div class="rp-spin"></div><div>Loading 3D room…</div></div>
        </div>
        <p class="rp-hint">Drag empty space to rotate · Scroll or pinch to zoom · Click furniture to select, then drag to move</p>
      </section>
      <aside class="rp-card rp-panel rp-selected" data-rp="selected">
        <button type="button" class="rp-selected-toggle" data-act="toggle-selected" aria-expanded="false">
          <h3>Selected Furniture</h3>
          <span class="rp-sub"><span data-rp="selsummary"></span> <span class="rp-caret">▼</span></span>
        </button>
        <div class="rp-selected-body">
          <div class="rp-list" data-rp="list"></div>
          <div class="rp-total-block">
            <div class="rp-total-label">Estimated Total</div>
            <div class="rp-total-val" data-rp="total">RM 0.00</div>
          </div>
          <div class="rp-actions">
            <button type="button" class="rp-btn rp-btn-primary" data-act="save">Save Design</button>
            <button type="button" class="rp-btn" data-act="copy-json">Copy Quotation JSON</button>
            <button type="button" class="rp-btn rp-btn-danger" data-act="reset-room">Reset Room</button>
          </div>
          <div class="rp-saved" data-rp="saved"></div>
        </div>
      </aside>
    </div>
    <div class="rp-modal" data-rp="modal" hidden>
      <div class="rp-modal-box" role="dialog" aria-modal="true" aria-labelledby="rp-modal-title">
        <h4 id="rp-modal-title" data-rp="modal-title">Are you sure?</h4>
        <p data-rp="modal-text"></p>
        <div class="rp-modal-actions">
          <button type="button" class="rp-btn" data-act="modal-cancel">Cancel</button>
          <button type="button" class="rp-btn rp-btn-danger" data-act="modal-ok" data-rp="modal-ok">Confirm</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount } from 'vue'

// The original <script> code runs once, after the HTML above is on the page.
onMounted(() => {
  window.ROOM_PLANNER_PRODUCTS = window.ROOM_PLANNER_PRODUCTS || [
    // Living Room
    { id: "P001", name: "3-Seater Easy Clean Sofa", category: "Living Room", price: 1299, imageUrl: "", modelUrl: "", dimensions: { width: 2.0, depth: 0.85, height: 0.8 }, kind: "sofa", color: "#9aa39a" },
    { id: "P002", name: "Coffee Table", category: "Living Room", price: 299, imageUrl: "", modelUrl: "", dimensions: { width: 1.0, depth: 0.6, height: 0.45 }, kind: "table", color: "#b08968" },
    { id: "P003", name: "TV Console", category: "Living Room", price: 549, imageUrl: "", modelUrl: "", dimensions: { width: 1.6, depth: 0.4, height: 0.5 }, kind: "tvconsole", color: "#8d6e53" },
    { id: "P004", name: "Armchair", category: "Living Room", price: 499, imageUrl: "", modelUrl: "", dimensions: { width: 0.8, depth: 0.8, height: 0.85 }, kind: "armchair", color: "#c18c5d" },
    { id: "P005", name: "Side Cabinet", category: "Living Room", price: 399, imageUrl: "", modelUrl: "", dimensions: { width: 0.9, depth: 0.4, height: 0.9 }, kind: "cabinet", color: "#a68a6d" },
    // Bedroom
    { id: "P012", name: "Queen Bedframe", category: "Bedroom", price: 1299, imageUrl: "", modelUrl: "", dimensions: { width: 1.6, depth: 2.1, height: 1.0 }, kind: "bed", color: "#8a7560" },
    { id: "P014", name: "Bedside Table", category: "Bedroom", price: 199, imageUrl: "", modelUrl: "", dimensions: { width: 0.45, depth: 0.4, height: 0.5 }, kind: "cabinet", color: "#a68a6d" },
    { id: "P015", name: "3-Door Wardrobe", category: "Bedroom", price: 1099, imageUrl: "", modelUrl: "", dimensions: { width: 1.5, depth: 0.55, height: 2.0 }, kind: "cabinet", color: "#d8d2c4" },
    { id: "P016", name: "Study Desk", category: "Bedroom", price: 599, imageUrl: "", modelUrl: "", dimensions: { width: 1.2, depth: 0.6, height: 0.75 }, kind: "desk", color: "#c9b79c" },
    // Dining Room
    { id: "P020", name: "Dining Table", category: "Dining Room", price: 899, imageUrl: "", modelUrl: "", dimensions: { width: 1.5, depth: 0.8, height: 0.75 }, kind: "table", color: "#b08968" },
    { id: "P021", name: "Dining Chair", category: "Dining Room", price: 189, imageUrl: "", modelUrl: "", dimensions: { width: 0.45, depth: 0.5, height: 0.9 }, kind: "chair", color: "#6c756d" },
    // Bathroom
    { id: "P030", name: "Vanity Basin Cabinet", category: "Bathroom", price: 649, imageUrl: "", modelUrl: "", dimensions: { width: 0.8, depth: 0.5, height: 0.85 }, kind: "basin", color: "#cfc7b8" },
    { id: "P031", name: "Toilet Bowl", category: "Bathroom", price: 699, imageUrl: "", modelUrl: "", dimensions: { width: 0.4, depth: 0.7, height: 0.75 }, kind: "toilet", color: "#f4f4f2" },
    // Decoration
    { id: "P040", name: "Area Rug", category: "Decoration", price: 349, imageUrl: "", modelUrl: "", dimensions: { width: 2.3, depth: 1.6, height: 0.02 }, kind: "rug", color: "#c7b9a3" },
    { id: "P041", name: "Artificial Plant", category: "Decoration", price: 129, imageUrl: "", modelUrl: "", dimensions: { width: 0.4, depth: 0.4, height: 1.1 }, kind: "plant", color: "#5b8a55" },
    { id: "P042", name: "Floor Lamp", category: "Decoration", price: 159, imageUrl: "", modelUrl: "", dimensions: { width: 0.35, depth: 0.35, height: 1.5 }, kind: "lamp", color: "#444444" },
    { id: "P043", name: "Decorative Vase", category: "Decoration", price: 69, imageUrl: "", modelUrl: "", dimensions: { width: 0.22, depth: 0.22, height: 0.4 }, kind: "vase", color: "#8fa6a0" }
  ];

  // Optional settings (all have defaults)
  window.ROOM_PLANNER_CONFIG = window.ROOM_PLANNER_CONFIG || {
    storageKey: "widebed_room_planner_v1",
    room: { width: 4, length: 5, height: 3 },
    snapStep: 0.1,        // metres
    rotateStep: 15        // degrees
  };

  (function () {
    'use strict';

    const root = document.getElementById('room-planner');
    if (!root || root.dataset.rpInit === '1') return;
    root.dataset.rpInit = '1';

    const el = name => root.querySelector('[data-rp="' + name + '"]');
    const els = {
      count: el('count'), search: el('search'), chips: el('chips'), products: el('products'),
      inW: el('inW'), inL: el('inL'), inH: el('inH'),
      viewer: el('viewer'), canvas: el('canvas'), dims: el('dims'), selbar: el('selbar'), selname: el('selname'),
      toast: el('toast'), status: el('status'),
      selected: el('selected'), selsummary: el('selsummary'), list: el('list'), total: el('total'), saved: el('saved'),
      modal: el('modal'), modalTitle: el('modal-title'), modalText: el('modal-text'), modalOk: el('modal-ok')
    };

    /* ---------------------------------------------------------
       CONFIG
       --------------------------------------------------------- */
    const THREE_VERSION = '0.147.0';
    const CDN = 'https://cdn.jsdelivr.net/npm/three@' + THREE_VERSION;
    const SCRIPTS = {
      three: CDN + '/build/three.min.js',
      orbit: CDN + '/examples/js/controls/OrbitControls.js',
      gltf: CDN + '/examples/js/loaders/GLTFLoader.js'
    };
    const userCfg = window.ROOM_PLANNER_CONFIG || {};
    const CFG = {
      storageKey: userCfg.storageKey || 'widebed_room_planner_v1',
      room: Object.assign({ width: 4, length: 5, height: 3 }, userCfg.room || {}),
      snapStep: Number(userCfg.snapStep) > 0 ? Number(userCfg.snapStep) : 0.1,
      rotateStep: Number(userCfg.rotateStep) > 0 ? Number(userCfg.rotateStep) : 15,
      limits: { width: [2, 12], length: [2, 12], height: [2.2, 4] }
    };
    const CATEGORIES = ['All', 'Living Room', 'Bedroom', 'Dining Room', 'Bathroom', 'Decoration'];
    const CAT_ICON = { 'Living Room': '🛋️', 'Bedroom': '🛏️', 'Dining Room': '🍽️', 'Bathroom': '🚿', 'Decoration': '🪴' };

    /* ---------------------------------------------------------
       HELPERS
       --------------------------------------------------------- */
    const clamp = (v, a, b) => Math.min(b, Math.max(a, v));
    const round3 = v => Math.round(v * 1000) / 1000;
    const money = n => 'RM ' + Number(n || 0).toLocaleString('en-MY', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
    const esc = s => String(s == null ? '' : s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
    let uidCounter = 0;
    const newUid = () => 'rp_' + Date.now().toString(36) + '_' + (++uidCounter) + Math.random().toString(36).slice(2, 6);
    const normDeg = d => ((Math.round(d) % 360) + 360) % 360;

    /* ---------------------------------------------------------
       STATE
       --------------------------------------------------------- */
    let products = [];
    let productMap = new Map();
    const state = {
      room: { width: CFG.room.width, length: CFG.room.length, height: CFG.room.height },
      items: [],            // { uid, productId, x, z, rotation }   (x,z in metres from room centre, rotation in degrees)
      selected: null,       // uid
      search: '',
      category: 'All',
      snap: true,
      ready: false
    };

    function normalizeProduct(p) {
      const d = p && p.dimensions ? p.dimensions : {};
      const dim = {
        width: Number(d.width) > 0 ? Number(d.width) : 0.5,
        depth: Number(d.depth) > 0 ? Number(d.depth) : 0.5,
        height: Number(d.height) > 0 ? Number(d.height) : 0.5
      };
      return {
        id: String(p.id),
        name: String(p.name || p.id),
        category: CATEGORIES.indexOf(p.category) > 0 ? p.category : 'Decoration',
        price: Number(p.price) >= 0 ? Number(p.price) : 0,
        imageUrl: p.imageUrl || '',
        modelUrl: p.modelUrl || '',
        dimensions: dim,
        kind: p.kind || 'box',
        color: p.color || '#b9b4a8'
      };
    }
    function setProducts(list) {
      const arr = Array.isArray(list) ? list : [];
      products = arr.filter(p => p && p.id != null).map(normalizeProduct);
      productMap = new Map(products.map(p => [p.id, p]));
    }

    /* ---------------------------------------------------------
       TOAST + STATUS + CONFIRM
       --------------------------------------------------------- */
    let toastTimer = 0;
    function toast(msg) {
      els.toast.textContent = msg;
      els.toast.classList.add('show');
      clearTimeout(toastTimer);
      toastTimer = setTimeout(() => els.toast.classList.remove('show'), 2600);
    }
    function setStatus(type, text, retry) {
      if (!type) { els.status.hidden = true; return; }
      els.status.hidden = false;
      els.status.className = 'rp-status' + (type === 'error' ? ' error' : '');
      els.status.innerHTML = (type === 'loading' ? '<div class="rp-spin"></div>' : '') +
        '<div>' + esc(text) + '</div>' +
        (retry ? '<button type="button" class="rp-btn rp-btn-primary" data-act="retry">Try again</button>' : '');
    }
    let confirmResolve = null;
    function askConfirm(title, text, okLabel) {
      els.modalTitle.textContent = title;
      els.modalText.textContent = text;
      els.modalOk.textContent = okLabel || 'Confirm';
      els.modal.hidden = false;
      els.modalOk.focus();
      return new Promise(res => { confirmResolve = res; });
    }
    function closeConfirm(result) {
      els.modal.hidden = true;
      if (confirmResolve) { confirmResolve(result); confirmResolve = null; }
    }

    /* ---------------------------------------------------------
       CATALOGUE UI
       --------------------------------------------------------- */
    function renderChips() {
      els.chips.innerHTML = CATEGORIES.map(c =>
        '<button type="button" class="rp-chip' + (state.category === c ? ' active' : '') + '" data-cat="' + esc(c) + '" aria-pressed="' + (state.category === c) + '">' + esc(c) + '</button>'
      ).join('');
    }
    function filteredProducts() {
      const q = state.search.trim().toLowerCase();
      return products.filter(p =>
        (state.category === 'All' || p.category === state.category) &&
        (!q || p.name.toLowerCase().indexOf(q) !== -1 || p.category.toLowerCase().indexOf(q) !== -1 || p.id.toLowerCase().indexOf(q) !== -1)
      );
    }
    function renderProducts() {
      const list = filteredProducts();
      els.count.textContent = list.length + (list.length === 1 ? ' item' : ' items');
      if (!list.length) {
        els.products.innerHTML = '<div class="rp-empty">No furniture found. Try another search or category.</div>';
        return;
      }
      els.products.innerHTML = list.map(p => {
        const thumb = p.imageUrl
          ? '<img src="' + esc(p.imageUrl) + '" alt="" loading="lazy" onerror="this.parentNode.textContent=\'' + (CAT_ICON[p.category] || '🪑') + '\'">'
          : (CAT_ICON[p.category] || '🪑');
        return '<article class="rp-product">' +
          '<div class="rp-thumb">' + thumb + '</div>' +
          '<div><div class="rp-pname">' + esc(p.name) + '</div><div class="rp-pcat">' + esc(p.category) + '</div><div class="rp-pprice">' + money(p.price) + '</div></div>' +
          '<button type="button" class="rp-btn rp-btn-primary rp-add" data-add="' + esc(p.id) + '"' + (state.ready ? '' : ' disabled') + '>Add to Room</button>' +
          '</article>';
      }).join('');
    }

    /* ---------------------------------------------------------
       SELECTED PANEL + PRICES
       --------------------------------------------------------- */
    function calcTotals() {
      const groups = new Map();
      state.items.forEach(it => {
        const p = productMap.get(it.productId);
        if (!p) return;
        if (!groups.has(p.id)) groups.set(p.id, { product: p, qty: 0, uids: [] });
        const g = groups.get(p.id);
        g.qty += 1;
        g.uids.push(it.uid);
      });
      let totalCents = 0;
      groups.forEach(g => { g.subtotal = Math.round(g.product.price * 100) * g.qty / 100; totalCents += Math.round(g.product.price * 100) * g.qty; });
      return { groups: Array.from(groups.values()), total: totalCents / 100, count: state.items.length };
    }
    function renderSelected() {
      const t = calcTotals();
      els.selsummary.textContent = t.count + (t.count === 1 ? ' item' : ' items') + ' · ' + money(t.total);
      els.total.textContent = money(t.total);
      if (!t.groups.length) {
        els.list.innerHTML = '<div class="rp-empty">No furniture added yet.<br>Choose items from the catalogue to place them in the room.</div>';
        return;
      }
      els.list.innerHTML = t.groups.map(g => {
        const hasSel = g.uids.indexOf(state.selected) !== -1;
        return '<div class="rp-row' + (hasSel ? ' has-selected' : '') + '">' +
          '<div><button type="button" class="rp-row-name" data-focus="' + esc(g.product.id) + '" title="Select in room">' + esc(g.product.name) + '</button>' +
          '<div class="rp-row-cat">' + esc(g.product.category) + '</div></div>' +
          '<div class="rp-qty"><button type="button" data-dec="' + esc(g.product.id) + '" aria-label="Remove one ' + esc(g.product.name) + '">−</button>' +
          '<span aria-label="Quantity">' + g.qty + '</span>' +
          '<button type="button" data-inc="' + esc(g.product.id) + '" aria-label="Add one more ' + esc(g.product.name) + '">+</button></div>' +
          '<div class="rp-row-prices"><span>' + money(g.product.price) + ' each</span><strong>' + money(g.subtotal) + '</strong></div>' +
          '<button type="button" class="rp-btn rp-btn-sm rp-btn-danger rp-remove" data-remove="' + esc(g.product.id) + '">Remove</button>' +
          '</div>';
      }).join('');
    }
    function updateSelbar() {
      const it = getItem(state.selected);
      if (!it) { els.selbar.hidden = true; return; }
      const p = productMap.get(it.productId);
      els.selname.textContent = p ? p.name : '';
      els.selbar.hidden = false;
    }
    function renderRoomInputs() {
      els.inW.value = state.room.width;
      els.inL.value = state.room.length;
      els.inH.value = state.room.height;
      els.dims.textContent = state.room.width.toFixed(1) + ' × ' + state.room.length.toFixed(1) + ' × ' + state.room.height.toFixed(1) + ' m';
    }

    /* ---------------------------------------------------------
       ITEMS + BOUNDS
       --------------------------------------------------------- */
    function getItem(uid) { return uid ? state.items.find(i => i.uid === uid) || null : null; }

    function halfExtents(p, rotDeg) {
      const r = rotDeg * Math.PI / 180, c = Math.abs(Math.cos(r)), s = Math.abs(Math.sin(r));
      const w = p.dimensions.width, d = p.dimensions.depth;
      return { hw: (w * c + d * s) / 2, hd: (w * s + d * c) / 2 };
    }
    // Keeps an item inside the room. Returns true if its position had to change.
    function clampItem(it) {
      const p = productMap.get(it.productId);
      if (!p) return false;
      const { hw, hd } = halfExtents(p, it.rotation);
      const maxX = state.room.width / 2 - hw, maxZ = state.room.length / 2 - hd;
      const nx = maxX < 0 ? 0 : clamp(it.x, -maxX, maxX);
      const nz = maxZ < 0 ? 0 : clamp(it.z, -maxZ, maxZ);
      const moved = Math.abs(nx - it.x) > 0.0005 || Math.abs(nz - it.z) > 0.0005;
      it.x = round3(nx); it.z = round3(nz);
      return moved;
    }
    function snapValue(v) { return state.snap ? Math.round(v / CFG.snapStep) * CFG.snapStep : v; }

    /* ---------------------------------------------------------
       SAVE / LOAD / QUOTATION
       --------------------------------------------------------- */
    function getDesign() {
      return {
        version: 1,
        savedAt: new Date().toISOString(),
        room: { width: state.room.width, length: state.room.length, height: state.room.height },
        items: state.items.map(i => ({ uid: i.uid, productId: i.productId, x: round3(i.x), z: round3(i.z), rotation: normDeg(i.rotation) })),
        totalPrice: calcTotals().total
      };
    }
    function getQuotationPayload() {
      const t = calcTotals();
      return {
        type: 'room_planner_quotation',
        version: 1,
        generated_at: new Date().toISOString(),
        currency: 'MYR',
        room: {
          width_m: state.room.width, length_m: state.room.length, height_m: state.room.height,
          floor_area_sqm: round3(state.room.width * state.room.length)
        },
        items: t.groups.map(g => ({
          product_id: g.product.id,
          name: g.product.name,
          category: g.product.category,
          quantity: g.qty,
          unit_price: g.product.price,
          subtotal: g.subtotal,
          placements: g.uids.map(uid => { const i = getItem(uid); return { instance_id: uid, x_m: i.x, z_m: i.z, rotation_deg: normDeg(i.rotation) }; })
        })),
        summary: { total_items: t.count, estimated_total: t.total }
      };
    }
    let saveTimer = 0;
    function saveNow(manual) {
      try {
        localStorage.setItem(CFG.storageKey, JSON.stringify(getDesign()));
        const hh = new Date().toLocaleTimeString('en-MY', { hour: '2-digit', minute: '2-digit' });
        els.saved.textContent = 'Saved at ' + hh + ' · changes are saved automatically on this device';
        if (manual) toast('Design saved ✓');
        return true;
      } catch (e) {
        els.saved.textContent = 'Could not save in this browser (private mode or storage full).';
        if (manual) toast('Could not save the design in this browser');
        return false;
      }
    }
    function scheduleSave() { clearTimeout(saveTimer); saveTimer = setTimeout(() => saveNow(false), 500); }

    function restoreSaved() {
      let data = null;
      try { data = JSON.parse(localStorage.getItem(CFG.storageKey) || 'null'); } catch (e) { data = null; }
      if (!data || typeof data !== 'object') return;
      const r = data.room || {};
      const lim = CFG.limits;
      if (isFinite(r.width) && isFinite(r.length) && isFinite(r.height)) {
        state.room.width = clamp(Number(r.width), lim.width[0], lim.width[1]);
        state.room.length = clamp(Number(r.length), lim.length[0], lim.length[1]);
        state.room.height = clamp(Number(r.height), lim.height[0], lim.height[1]);
      }
      const seen = new Set();
      (Array.isArray(data.items) ? data.items : []).forEach(i => {
        if (!i || !productMap.has(String(i.productId))) return;
        let uid = typeof i.uid === 'string' && i.uid && !seen.has(i.uid) ? i.uid : newUid();
        seen.add(uid);
        const it = { uid, productId: String(i.productId), x: Number(i.x) || 0, z: Number(i.z) || 0, rotation: normDeg(Number(i.rotation) || 0) };
        clampItem(it);
        state.items.push(it);
      });
    }

    /* ---------------------------------------------------------
       CHANGE HANDLING
       --------------------------------------------------------- */
    function changed() {
      renderSelected();
      updateSelbar();
      scheduleSave();
      try { root.dispatchEvent(new CustomEvent('roomPlanner:change', { bubbles: true, detail: getDesign() })); } catch (e) {}
    }

    /* =========================================================
       3D ENGINE
       ========================================================= */
    let T = null;                         // THREE namespace
    let renderer, scene, camera, controls, roomGroup, furnGroup, hemi, sun, raycaster, floorPlane;
    const fmap = new Map();               // uid -> THREE.Group
    const modelCache = new Map();         // modelUrl -> Promise<scene>
    let raf = 0, visible = true, drag = null, emptyDown = null, hoverQueued = false;
    let resizeObs = null, ioObs = null;
    const ac = new AbortController();     // removes every listener on destroy()
    const SIG = { signal: ac.signal };

    function loadScript(src) {
      return new Promise((resolve, reject) => {
        const existing = Array.prototype.find.call(document.scripts, s => s.src === src);
        if (existing && existing.dataset.rpLoaded === '1') return resolve();
        const s = existing || document.createElement('script');
        s.addEventListener('load', () => { s.dataset.rpLoaded = '1'; resolve(); });
        s.addEventListener('error', () => { if (s.parentNode) s.parentNode.removeChild(s); reject(new Error('Could not load ' + src)); });
        if (!existing) { s.src = src; document.head.appendChild(s); }
      });
    }
    async function loadEngine() {
      if (!window.THREE) await loadScript(SCRIPTS.three);
      if (!window.THREE.OrbitControls) await loadScript(SCRIPTS.orbit);
      if (!window.THREE.GLTFLoader) await loadScript(SCRIPTS.gltf);
      T = window.THREE;
    }

    const rev = () => parseInt(T.REVISION, 10) || 0;
    function col(hex) {
      const c = new T.Color(hex);
      return (rev() < 152 && c.convertSRGBToLinear) ? c.convertSRGBToLinear() : c;
    }
    function shade(hex, f) { const c = new T.Color(hex); c.multiplyScalar(f); return '#' + c.getHexString(); }

    function disposeObject(obj) {
      if (!obj) return;
      obj.traverse(o => {
        if (o.geometry && !o.userData.shared) o.geometry.dispose();
        if (o.material) (Array.isArray(o.material) ? o.material : [o.material]).forEach(m => m && m.dispose && m.dispose());
      });
    }

    /* ---------- Room ---------- */
    function buildRoom() {
      if (roomGroup) { scene.remove(roomGroup); disposeObject(roomGroup); }
      const W = state.room.width, L = state.room.length, H = state.room.height;
      roomGroup = new T.Group();

      const floor = new T.Mesh(new T.PlaneGeometry(W, L), new T.MeshStandardMaterial({ color: col('#ece8de'), roughness: 1 }));
      floor.rotation.x = -Math.PI / 2;
      floor.receiveShadow = true;
      roomGroup.add(floor);

      const wallMat = () => new T.MeshStandardMaterial({ color: col('#f7f5ef'), roughness: 1, side: T.FrontSide });
      const back = new T.Mesh(new T.PlaneGeometry(W, H), wallMat());
      back.position.set(0, H / 2, -L / 2);
      const left = new T.Mesh(new T.PlaneGeometry(L, H), wallMat());
      left.rotation.y = Math.PI / 2;
      left.position.set(-W / 2, H / 2, 0);
      const right = new T.Mesh(new T.PlaneGeometry(L, H), wallMat());
      right.rotation.y = -Math.PI / 2;
      right.position.set(W / 2, H / 2, 0);
      [back, left, right].forEach(w => { w.receiveShadow = true; roomGroup.add(w); });

      // grid (every 0.5 m) for positioning
      const pts = [];
      for (let x = -W / 2; x <= W / 2 + 0.001; x += 0.5) pts.push(x, 0.004, -L / 2, x, 0.004, L / 2);
      for (let z = -L / 2; z <= L / 2 + 0.001; z += 0.5) pts.push(-W / 2, 0.004, z, W / 2, 0.004, z);
      const gg = new T.BufferGeometry();
      gg.setAttribute('position', new T.Float32BufferAttribute(pts, 3));
      const grid = new T.LineSegments(gg, new T.LineBasicMaterial({ color: col('#b9b4a6'), transparent: true, opacity: 0.45 }));
      grid.raycast = function () {};
      roomGroup.add(grid);

      // floor border
      const bg = new T.BufferGeometry().setFromPoints([
        new T.Vector3(-W / 2, 0.006, -L / 2), new T.Vector3(W / 2, 0.006, -L / 2),
        new T.Vector3(W / 2, 0.006, L / 2), new T.Vector3(-W / 2, 0.006, L / 2)
      ]);
      const border = new T.LineLoop(bg, new T.LineBasicMaterial({ color: col('#8b8676') }));
      border.raycast = function () {};
      roomGroup.add(border);

      scene.add(roomGroup);

      // lighting follows the room size
      const span = Math.max(W, L);
      sun.position.set(W * 0.5, H + 2.5, L * 0.6);
      sun.target.position.set(0, 0, 0);
      const sc = sun.shadow.camera;
      sc.left = -span; sc.right = span; sc.top = span; sc.bottom = -span;
      sc.near = 0.5; sc.far = H + span * 3 + 6;
      sc.updateProjectionMatrix();
      renderDims();
    }
    function renderDims() { renderRoomInputs(); }

    function resetCamera() {
      if (!camera) return;
      const W = state.room.width, L = state.room.length, H = state.room.height, span = Math.max(W, L);
      camera.position.set(W * 0.55, Math.max(H * 1.15, span * 0.55) + 0.5, L / 2 + span * 0.95);
      controls.target.set(0, H * 0.25, 0);
      controls.update();
    }

    /* ---------- Placeholder furniture ---------- */
    function mesh(geo, color, opts, noShadow) {
      const m = new T.Mesh(geo, new T.MeshStandardMaterial(Object.assign({ color: col(color), roughness: 0.85, metalness: 0.05 }, opts || {})));
      m.castShadow = !noShadow; m.receiveShadow = true;
      return m;
    }
    function box(g, w, h, d, x, y, z, color, opts, noShadow) {
      const m = mesh(new T.BoxGeometry(w, h, d), color, opts, noShadow);
      m.position.set(x, y, z); g.add(m); return m;
    }
    function cyl(g, rt, rb, h, x, y, z, color, opts) {
      const m = mesh(new T.CylinderGeometry(rt, rb, h, 20), color, opts);
      m.position.set(x, y, z); g.add(m); return m;
    }

    function buildFallback(p) {
      const w = p.dimensions.width, d = p.dimensions.depth, h = p.dimensions.height, c = p.color;
      const g = new T.Group();
      switch (p.kind) {
        case 'sofa':
        case 'armchair': {
          const baseH = h * 0.5, arm = Math.min(0.16, w * 0.12), backD = d * 0.24;
          box(g, w, baseH, d, 0, baseH / 2, 0, shade(c, 0.92));
          box(g, w, h - baseH, backD, 0, baseH + (h - baseH) / 2, -d / 2 + backD / 2, c);
          const armH = baseH + h * 0.2;
          box(g, arm, armH, d - backD, -(w / 2 - arm / 2), armH / 2, backD / 2, shade(c, 0.97));
          box(g, arm, armH, d - backD, (w / 2 - arm / 2), armH / 2, backD / 2, shade(c, 0.97));
          box(g, w - 2 * arm, h * 0.12, d - backD - 0.04, 0, baseH + h * 0.06, backD / 2 + 0.02, shade(c, 1.12));
          break;
        }
        case 'table':
        case 'desk': {
          const top = 0.05, lx = w / 2 - 0.06, lz = d / 2 - 0.06;
          box(g, w, top, d, 0, h - top / 2, 0, c);
          [[-1, -1], [1, -1], [-1, 1], [1, 1]].forEach(s => box(g, 0.05, h - top, 0.05, s[0] * lx, (h - top) / 2, s[1] * lz, shade(c, 0.8)));
          if (p.kind === 'desk') box(g, w * 0.3, 0.2, d - 0.1, w / 2 - w * 0.17, h - top - 0.1, 0, shade(c, 0.9));
          break;
        }
        case 'chair': {
          const seatY = Math.min(0.46, h * 0.5);
          box(g, w, 0.05, d, 0, seatY, 0, c);
          [[-1, -1], [1, -1], [-1, 1], [1, 1]].forEach(s => box(g, 0.04, seatY, 0.04, s[0] * (w / 2 - 0.04), seatY / 2, s[1] * (d / 2 - 0.04), shade(c, 0.8)));
          box(g, w, h - seatY, 0.04, 0, seatY + (h - seatY) / 2, -d / 2 + 0.02, shade(c, 0.92));
          break;
        }
        case 'bed': {
          const frameH = Math.min(0.32, h * 0.4);
          box(g, w, frameH, d, 0, frameH / 2, 0, shade(c, 0.85));
          box(g, w - 0.08, 0.24, d - 0.12, 0, frameH + 0.12, 0.04, '#f1efe9');
          box(g, w, h, 0.08, 0, h / 2, -d / 2 + 0.04, c);
          box(g, w * 0.38, 0.12, 0.3, -w * 0.22, frameH + 0.3, -d / 2 + 0.35, '#ffffff');
          box(g, w * 0.38, 0.12, 0.3, w * 0.22, frameH + 0.3, -d / 2 + 0.35, '#ffffff');
          break;
        }
        case 'plant': {
          const potH = Math.min(0.3, h * 0.3);
          const pot = mesh(new T.CylinderGeometry(w * 0.36, w * 0.28, potH, 18), '#b7a48b');
          pot.position.y = potH / 2; g.add(pot);
          const leaf = mesh(new T.SphereGeometry(w / 2, 16, 12), c);
          leaf.scale.set(1, (h - potH) / w, 1);
          leaf.position.y = potH + (h - potH) / 2; g.add(leaf);
          const leaf2 = mesh(new T.SphereGeometry(w * 0.32, 14, 10), shade(c, 1.1));
          leaf2.position.set(w * 0.12, potH + (h - potH) * 0.78, 0.05); g.add(leaf2);
          break;
        }
        case 'vase': {
          const v = mesh(new T.CylinderGeometry(w * 0.35, w * 0.25, h, 20), c);
          v.position.y = h / 2; g.add(v);
          break;
        }
        case 'lamp': {
          cyl(g, w * 0.4, w * 0.4, 0.03, 0, 0.015, 0, '#333333');
          cyl(g, 0.012, 0.012, h * 0.82, 0, 0.03 + h * 0.41, 0, '#444444');
          cyl(g, w * 0.28, w * 0.45, h * 0.16, 0, h - h * 0.08, 0, '#f4ead2', { emissive: new T.Color(0xffe2a8), emissiveIntensity: 0.35 });
          break;
        }
        case 'rug': {
          box(g, w, 0.02, d, 0, 0.01, 0, c, null, true);
          box(g, w * 0.86, 0.022, d * 0.8, 0, 0.011, 0, shade(c, 1.15), null, true);
          break;
        }
        case 'toilet': {
          box(g, w, 0.4, d * 0.62, 0, 0.2, d * 0.19, c);
          box(g, w * 0.92, h - 0.15, d * 0.26, 0, 0.15 + (h - 0.15) / 2, -d / 2 + d * 0.13, c);
          break;
        }
        case 'basin': {
          box(g, w, h - 0.06, d, 0, (h - 0.06) / 2, 0, c);
          box(g, w, 0.06, d, 0, h - 0.03, 0, '#f4f4f2');
          box(g, w * 0.55, 0.02, d * 0.6, 0, h + 0.005, 0.02, '#dfe3e3');
          cyl(g, 0.015, 0.015, 0.12, 0, h + 0.06, -d / 2 + 0.07, '#b9bdc0');
          break;
        }
        case 'cabinet':
        case 'tvconsole': {
          box(g, w, h, d, 0, h / 2, 0, c);
          const n = clamp(Math.round(w / 0.45), 1, 4);
          for (let i = 1; i < n; i++) box(g, 0.012, h * 0.92, 0.012, -w / 2 + i * w / n, h / 2, d / 2 + 0.004, shade(c, 0.6));
          for (let i = 0; i < n; i++) box(g, 0.02, 0.12, 0.02, -w / 2 + (i + 0.5) * w / n + (w / n) * 0.32, h * 0.55, d / 2 + 0.012, '#444444');
          break;
        }
        default:
          box(g, w, h, d, 0, h / 2, 0, c);
      }
      return g;
    }

    /* ---------- GLB models (optional) ---------- */
    function loadModelScene(url) {
      if (!modelCache.has(url)) {
        modelCache.set(url, new Promise((resolve, reject) => {
          new T.GLTFLoader().load(url, gltf => resolve(gltf.scene), undefined, reject);
        }));
      }
      return modelCache.get(url);
    }
    function fitModel(sceneRoot, dims) {
      const m = sceneRoot.clone(true);
      m.traverse(o => {
        if (o.isMesh) {
          o.userData.shared = true;                 // geometry is shared with the cached model: never dispose it per instance
          o.castShadow = true; o.receiveShadow = true;
          o.material = Array.isArray(o.material) ? o.material.map(x => x.clone()) : (o.material ? o.material.clone() : o.material);
        }
      });
      const bb = new T.Box3().setFromObject(m);
      const size = bb.getSize(new T.Vector3()), ctr = bb.getCenter(new T.Vector3());
      const s = Math.min(
        size.x > 0 ? dims.width / size.x : Infinity,
        size.z > 0 ? dims.depth / size.z : Infinity,
        size.y > 0 ? dims.height / size.y : Infinity
      );
      const k = isFinite(s) ? s : 1;
      m.scale.setScalar(k);
      m.position.set(-ctr.x * k, -bb.min.y * k, -ctr.z * k);
      const wrap = new T.Group();
      wrap.add(m);
      return wrap;
    }

    /* ---------- Furniture instances ---------- */
    function noRaycast(o) { o.raycast = function () {}; }
    function createFurniture(it) {
      const p = productMap.get(it.productId);
      const g = new T.Group();
      g.userData.uid = it.uid;

      const body = buildFallback(p);
      body.userData.isBody = true;
      g.add(body);

      // selection footprint
      const hw = p.dimensions.width / 2 + 0.03, hd = p.dimensions.depth / 2 + 0.03;
      const outline = new T.Group();
      const ring = new T.LineLoop(
        new T.BufferGeometry().setFromPoints([new T.Vector3(-hw, 0.015, -hd), new T.Vector3(hw, 0.015, -hd), new T.Vector3(hw, 0.015, hd), new T.Vector3(-hw, 0.015, hd)]),
        new T.LineBasicMaterial({ color: col('#4A5D45') })
      );
      const fill = new T.Mesh(new T.PlaneGeometry(hw * 2, hd * 2), new T.MeshBasicMaterial({ color: col('#4A5D45'), transparent: true, opacity: 0.18, depthWrite: false }));
      fill.rotation.x = -Math.PI / 2; fill.position.y = 0.012;
      noRaycast(ring); noRaycast(fill);
      outline.add(ring, fill);
      outline.visible = false;
      g.userData.outline = outline;
      g.add(outline);

      applyTransform(g, it);
      furnGroup.add(g);
      fmap.set(it.uid, g);

      if (p.modelUrl) {
        loadModelScene(p.modelUrl).then(sceneRoot => {
          if (fmap.get(it.uid) !== g) return;               // removed while loading
          const model = fitModel(sceneRoot, p.dimensions);
          const old = g.children.find(c => c.userData.isBody);
          if (old) { g.remove(old); disposeObject(old); }
          model.userData.isBody = true;
          g.add(model);
          if (state.selected === it.uid) setHighlight(g, true);
        }).catch(err => {
          console.warn('Room Planner: model failed to load, using placeholder:', p.modelUrl, err);
          toast('Could not load the 3D model for ' + p.name + '. Showing a placeholder.');
        });
      }
      return g;
    }
    function applyTransform(g, it) {
      g.position.set(it.x, 0, it.z);
      g.rotation.y = it.rotation * Math.PI / 180;
    }
    function setHighlight(g, on) {
      g.traverse(o => {
        if (!o.isMesh || !o.material || o.parent === g.userData.outline) return;
        (Array.isArray(o.material) ? o.material : [o.material]).forEach(m => {
          if (!m.emissive) return;
          if (m.userData.__e === undefined) { m.userData.__e = m.emissive.getHex(); m.userData.__ei = m.emissiveIntensity; }
          if (on) { m.emissive.setHex(0x2f6b4a); m.emissiveIntensity = 0.5; }
          else { m.emissive.setHex(m.userData.__e); m.emissiveIntensity = m.userData.__ei; }
        });
      });
      if (g.userData.outline) g.userData.outline.visible = on;
    }

    /* ---------- Actions ---------- */
    function select(uid) {
      if (state.selected === uid) { updateSelbar(); return; }
      const prev = fmap.get(state.selected);
      if (prev) setHighlight(prev, false);
      state.selected = uid || null;
      const cur = fmap.get(state.selected);
      if (cur) setHighlight(cur, true);
      renderSelected();
      updateSelbar();
    }

    function addProduct(pid) {
      const p = productMap.get(pid);
      if (!p) return;
      if (!state.ready) { toast('The 3D room is still loading…'); return; }
      const n = state.items.length;
      const it = { uid: newUid(), productId: pid, x: ((n % 3) - 1) * 0.4, z: (((Math.floor(n / 3)) % 3) - 1) * 0.4, rotation: 0 };
      const tooBig = p.dimensions.width > state.room.width || p.dimensions.depth > state.room.length;
      clampItem(it);
      state.items.push(it);
      createFurniture(it);
      select(it.uid);
      changed();
      toast(tooBig ? p.name + ' is larger than this room. Increase the room size.' : p.name + ' added. Drag it to place it.');
    }

    function removeItem(uid) {
      const idx = state.items.findIndex(i => i.uid === uid);
      if (idx === -1) return;
      state.items.splice(idx, 1);
      const g = fmap.get(uid);
      if (g) { furnGroup.remove(g); disposeObject(g); fmap.delete(uid); }
      if (state.selected === uid) state.selected = null;
      changed();
    }

    function rotateSelected(delta) {
      const it = getItem(state.selected);
      if (!it) return;
      it.rotation = normDeg(it.rotation + delta);
      clampItem(it);
      applyTransform(fmap.get(it.uid), it);
      changed();
    }

    function nudgeSelected(dx, dz) {
      const it = getItem(state.selected);
      if (!it) return;
      it.x += dx; it.z += dz;
      clampItem(it);
      applyTransform(fmap.get(it.uid), it);
      changed();
    }

    function groupUids(pid) { return state.items.filter(i => i.productId === pid).map(i => i.uid); }

    async function resetRoom() {
      if (!state.items.length) { toast('The room is already empty.'); return; }
      const ok = await askConfirm('Clear the room?', 'All furniture will be removed from the room. Your room size stays the same.', 'Clear room');
      if (!ok) return;
      state.items.slice().forEach(i => {
        const g = fmap.get(i.uid);
        if (g) { furnGroup.remove(g); disposeObject(g); fmap.delete(i.uid); }
      });
      state.items = [];
      state.selected = null;
      changed();
      toast('Room cleared');
    }

    function applyRoomSize(w, l, h) {
      const lim = CFG.limits;
      const vals = [w, l, h];
      if (vals.some(v => !isFinite(v))) { toast('Please enter valid room sizes.'); renderRoomInputs(); return; }
      if (w < lim.width[0] || w > lim.width[1] || l < lim.length[0] || l > lim.length[1] || h < lim.height[0] || h > lim.height[1]) {
        toast('Room size limits: width ' + lim.width.join('–') + ' m, length ' + lim.length.join('–') + ' m, height ' + lim.height.join('–') + ' m');
        renderRoomInputs();
        return;
      }
      state.room.width = round3(w); state.room.length = round3(l); state.room.height = round3(h);
      buildRoom();
      let moved = 0;
      state.items.forEach(it => {
        if (clampItem(it)) moved++;
        const g = fmap.get(it.uid);
        if (g) applyTransform(g, it);
      });
      resetCamera();
      changed();
      toast(moved ? moved + ' item' + (moved > 1 ? 's were' : ' was') + ' moved to fit the new room.' : 'Room size updated.');
    }

    /* ---------- Pointer interaction (select / drag) ---------- */
    function setRay(e) {
      const r = els.canvas.getBoundingClientRect();
      const ndc = new T.Vector2(((e.clientX - r.left) / r.width) * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1);
      raycaster.setFromCamera(ndc, camera);
    }
    function pickUid() {
      const hits = raycaster.intersectObjects(furnGroup.children, true);
      if (!hits.length) return null;
      let o = hits[0].object;
      while (o && !o.userData.uid) o = o.parent;
      return o ? o.userData.uid : null;
    }

    function onPointerDown(e) {
      els.viewer.focus({ preventScroll: true });
      if (!state.ready) return;
      if (e.pointerType === 'mouse' && e.button !== 0) return;
      setRay(e);
      const uid = pickUid();
      if (uid) {
        select(uid);
        const g = fmap.get(uid), p = new T.Vector3();
        if (raycaster.ray.intersectPlane(floorPlane, p)) {
          drag = { uid, offX: g.position.x - p.x, offZ: g.position.z - p.z, moved: false, id: e.pointerId, sx: e.clientX, sy: e.clientY };
          controls.enabled = false;                    // furniture drag instead of camera orbit
          try { els.canvas.setPointerCapture(e.pointerId); } catch (err) {}
          els.canvas.style.cursor = 'grabbing';
        }
      } else {
        emptyDown = { x: e.clientX, y: e.clientY };
      }
    }
    function onPointerMove(e) {
      if (drag) {
        if (e.pointerId !== drag.id) return;
        if (!drag.moved && Math.hypot(e.clientX - drag.sx, e.clientY - drag.sy) < 4) return;
        drag.moved = true;
        setRay(e);
        const p = new T.Vector3();
        if (!raycaster.ray.intersectPlane(floorPlane, p)) return;
        const it = getItem(drag.uid);
        if (!it) return;
        it.x = snapValue(p.x + drag.offX);
        it.z = snapValue(p.z + drag.offZ);
        clampItem(it);
        applyTransform(fmap.get(it.uid), it);
        return;
      }
      if (e.pointerType === 'mouse' && e.buttons === 0 && state.ready && !hoverQueued) {
        hoverQueued = true;
        const cx = e.clientX, cy = e.clientY;
        requestAnimationFrame(() => {
          hoverQueued = false;
          if (!state.ready || drag) return;
          setRay({ clientX: cx, clientY: cy });
          els.canvas.style.cursor = pickUid() ? 'grab' : 'default';
        });
      }
    }
    function endDrag(e) {
      if (drag) {
        if (e && e.pointerId !== drag.id) return;
        const moved = drag.moved;
        try { els.canvas.releasePointerCapture(drag.id); } catch (err) {}
        drag = null;
        controls.enabled = true;
        els.canvas.style.cursor = 'default';
        if (moved) changed();
      } else if (emptyDown && e) {
        if (Math.hypot(e.clientX - emptyDown.x, e.clientY - emptyDown.y) < 6) select(null);
        emptyDown = null;
      }
    }
    function onKey(e) {
      if (!state.selected) return;
      const k = e.key;
      if (k === 'Delete' || k === 'Backspace') { e.preventDefault(); removeItem(state.selected); }
      else if (k === 'r' || k === 'R') { e.preventDefault(); rotateSelected(e.shiftKey ? -90 : 90); }
      else if (k === 'Escape') { select(null); }
      else if (k === 'ArrowLeft') { e.preventDefault(); nudgeSelected(-CFG.snapStep, 0); }
      else if (k === 'ArrowRight') { e.preventDefault(); nudgeSelected(CFG.snapStep, 0); }
      else if (k === 'ArrowUp') { e.preventDefault(); nudgeSelected(0, -CFG.snapStep); }
      else if (k === 'ArrowDown') { e.preventDefault(); nudgeSelected(0, CFG.snapStep); }
    }

    /* ---------- Render loop ---------- */
    function startLoop() {
      if (raf || !renderer) return;
      const tick = () => {
        raf = requestAnimationFrame(tick);
        controls.update();
        renderer.render(scene, camera);
      };
      tick();
    }
    function stopLoop() { if (raf) { cancelAnimationFrame(raf); raf = 0; } }

    function resize() {
      if (!renderer) return;
      const w = Math.max(1, els.viewer.clientWidth), h = Math.max(1, els.viewer.clientHeight);
      renderer.setSize(w, h, false);
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
    }

    /* ---------- Engine start ---------- */
    function init3D() {
      try {
        renderer = new T.WebGLRenderer({ canvas: els.canvas, antialias: true, alpha: true });
      } catch (err) {
        throw new Error('Your browser or device does not support 3D graphics (WebGL).');
      }
      renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
      if (rev() < 152) renderer.outputEncoding = T.sRGBEncoding;
      renderer.shadowMap.enabled = true;
      renderer.shadowMap.type = T.PCFSoftShadowMap;
      renderer.setClearColor(0x000000, 0);

      scene = new T.Scene();
      camera = new T.PerspectiveCamera(45, 1, 0.1, 100);
      raycaster = new T.Raycaster();
      floorPlane = new T.Plane(new T.Vector3(0, 1, 0), 0);

      hemi = new T.HemisphereLight(0xffffff, 0xb8b4a6, 0.85);
      scene.add(hemi);
      sun = new T.DirectionalLight(0xffffff, 0.75);
      sun.castShadow = true;
      sun.shadow.mapSize.set(1024, 1024);
      sun.shadow.bias = -0.0005;
      scene.add(sun, sun.target);

      furnGroup = new T.Group();
      scene.add(furnGroup);

      // our pointer handlers are registered BEFORE OrbitControls so we can disable it when dragging furniture
      els.canvas.addEventListener('pointerdown', onPointerDown, SIG);
      els.canvas.addEventListener('pointermove', onPointerMove, SIG);
      els.canvas.addEventListener('pointerup', endDrag, SIG);
      els.canvas.addEventListener('pointercancel', endDrag, SIG);
      els.canvas.addEventListener('lostpointercapture', endDrag, SIG);
      els.viewer.addEventListener('keydown', onKey, SIG);
      els.canvas.addEventListener('webglcontextlost', ev => {
        ev.preventDefault(); stopLoop(); state.ready = false;
        setStatus('error', 'The 3D view was interrupted by your device. Please reload the page.');
      }, SIG);

      controls = new T.OrbitControls(camera, els.canvas);
      controls.enableDamping = true;
      controls.dampingFactor = 0.08;
      controls.minDistance = 2;
      controls.maxDistance = 40;
      controls.maxPolarAngle = Math.PI * 0.495;       // never go below the floor

      buildRoom();
      state.items.forEach(it => { clampItem(it); createFurniture(it); });
      resize();
      resetCamera();

      resizeObs = new ResizeObserver(resize);
      resizeObs.observe(els.viewer);
      ioObs = new IntersectionObserver(entries => {
        visible = entries[0].isIntersecting;
        if (visible && !document.hidden) startLoop(); else stopLoop();
      });
      ioObs.observe(els.viewer);
      document.addEventListener('visibilitychange', () => { if (document.hidden) stopLoop(); else if (visible) startLoop(); }, SIG);

      state.ready = true;
      setStatus(null);
      renderProducts();
      renderSelected();
      startLoop();
    }

    async function boot() {
      setStatus('loading', 'Loading 3D room…');
      try {
        await loadEngine();
        init3D();
      } catch (err) {
        console.error('Room Planner:', err);
        setStatus('error', (err && err.message && err.message.indexOf('WebGL') !== -1) ? err.message : 'The 3D room could not be loaded. Please check your internet connection and try again.', true);
      }
    }

    /* ---------------------------------------------------------
       UI EVENTS
       --------------------------------------------------------- */
    els.search.addEventListener('input', () => { state.search = els.search.value; renderProducts(); }, SIG);

    [els.inW, els.inL, els.inH].forEach(inp => {
      inp.addEventListener('change', () => {
        if (!state.ready) { renderRoomInputs(); toast('The 3D room is still loading…'); return; }
        applyRoomSize(parseFloat(els.inW.value), parseFloat(els.inL.value), parseFloat(els.inH.value));
      }, SIG);
      inp.addEventListener('keydown', e => { if (e.key === 'Enter') inp.blur(); }, SIG);
    });

    async function copyText(text) {
      try { await navigator.clipboard.writeText(text); return true; } catch (e) {}
      try {
        const ta = document.createElement('textarea');
        ta.value = text; ta.style.position = 'fixed'; ta.style.opacity = '0';
        document.body.appendChild(ta); ta.select();
        const ok = document.execCommand('copy');
        document.body.removeChild(ta);
        return ok;
      } catch (e) { return false; }
    }

    root.addEventListener('click', async e => {
      const t = e.target.closest('[data-act],[data-add],[data-cat],[data-inc],[data-dec],[data-remove],[data-focus]');
      if (!t || !root.contains(t)) return;

      if (t.dataset.add) return addProduct(t.dataset.add);
      if (t.dataset.cat) { state.category = t.dataset.cat; renderChips(); renderProducts(); return; }
      if (t.dataset.inc) return addProduct(t.dataset.inc);
      if (t.dataset.dec) {
        const uids = groupUids(t.dataset.dec);
        if (uids.length) removeItem(uids[uids.length - 1]);
        return;
      }
      if (t.dataset.remove) { groupUids(t.dataset.remove).forEach(removeItem); return; }
      if (t.dataset.focus) {
        const uids = groupUids(t.dataset.focus);
        if (!uids.length) return;
        const i = uids.indexOf(state.selected);
        select(uids[(i + 1) % uids.length]);
        return;
      }

      switch (t.dataset.act) {
        case 'snap':
          state.snap = !state.snap;
          t.textContent = 'Snap: ' + (state.snap ? 'On' : 'Off');
          t.setAttribute('aria-pressed', String(state.snap));
          break;
        case 'reset-camera': if (state.ready) resetCamera(); break;
        case 'rot-left': rotateSelected(CFG.rotateStep); break;
        case 'rot-right': rotateSelected(-CFG.rotateStep); break;
        case 'rot-90': rotateSelected(90); break;
        case 'delete-selected': if (state.selected) removeItem(state.selected); break;
        case 'save': saveNow(true); break;
        case 'copy-json': {
          if (!state.items.length) { toast('Add some furniture first.'); break; }
          const ok = await copyText(JSON.stringify(getQuotationPayload(), null, 2));
          toast(ok ? 'Quotation JSON copied ✓' : 'Could not copy. Please try again.');
          break;
        }
        case 'reset-room': resetRoom(); break;
        case 'toggle-selected': {
          const open = els.selected.classList.toggle('rp-open');
          t.setAttribute('aria-expanded', String(open));
          break;
        }
        case 'modal-ok': closeConfirm(true); break;
        case 'modal-cancel': closeConfirm(false); break;
        case 'retry': boot(); break;
      }
    }, SIG);
    els.modal.addEventListener('click', e => { if (e.target === els.modal) closeConfirm(false); }, SIG);
    document.addEventListener('keydown', e => { if (e.key === 'Escape' && !els.modal.hidden) closeConfirm(false); }, SIG);

    /* ---------------------------------------------------------
       PUBLIC API (for later integration)
       --------------------------------------------------------- */
    function destroy() {
      stopLoop();
      ac.abort();
      if (resizeObs) resizeObs.disconnect();
      if (ioObs) ioObs.disconnect();
      clearTimeout(saveTimer); clearTimeout(toastTimer);
      if (controls) controls.dispose();
      if (scene) scene.traverse(o => { if (o.geometry && !o.userData.shared) o.geometry.dispose(); if (o.material) (Array.isArray(o.material) ? o.material : [o.material]).forEach(m => m.dispose && m.dispose()); });
      if (renderer) { renderer.dispose(); if (renderer.forceContextLoss) renderer.forceContextLoss(); }
      fmap.clear(); modelCache.clear();
      state.ready = false;
      root.dataset.rpInit = '';
    }

    window.RoomPlanner = {
      getDesign: getDesign,
      getQuotationPayload: getQuotationPayload,
      setProducts: function (list) {
        setProducts(list);
        // furniture whose product no longer exists is removed
        state.items.filter(i => !productMap.has(i.productId)).forEach(i => removeItem(i.uid));
        renderProducts(); renderSelected();
      },
      save: function () { return saveNow(false); },
      clear: function () { state.items.slice().forEach(i => removeItem(i.uid)); },
      destroy: destroy
    };

    /* ---------------------------------------------------------
       START
       --------------------------------------------------------- */
    setProducts(window.ROOM_PLANNER_PRODUCTS);
    restoreSaved();
    renderChips();
    renderProducts();
    renderRoomInputs();
    renderSelected();
    if (window.matchMedia && window.matchMedia('(min-width: 961px)').matches) els.selected.classList.add('rp-open');
    boot();
  })();
})

// Free the 3D scene when this component leaves the page
onBeforeUnmount(() => {
  if (window.RoomPlanner) window.RoomPlanner.destroy()
})
</script>

<style>
.rp-root {
  --rp-primary: var(--brand-primary, #4A5D45);
  --rp-primary-hover: var(--brand-primary-hover, #3b4b37);
  --rp-surface: #FFFFFF;
  --rp-bg: #F6F6F3;
  --rp-border: var(--brand-border, #E2E0D8);
  --rp-text: var(--brand-text-main, #2C352B);
  --rp-muted: var(--brand-text-muted, #6C756D);
  --rp-accent: var(--brand-accent, #C18C5D);
  --rp-danger: #B3433B;
  --rp-font: var(--contentfont, 'Plus Jakarta Sans', system-ui, -apple-system, 'Segoe UI', sans-serif);
  --rp-head-font: var(--headlinefont, 'Playfair Display', Georgia, serif);

  max-width: 1400px;
  margin: 0 auto 24px;
  padding: 0 16px;
  box-sizing: border-box;
  font-family: var(--rp-font);
  color: var(--rp-text);
  line-height: 1.4;
}
.rp-root *, .rp-root *::before, .rp-root *::after { box-sizing: border-box; }
.rp-root [hidden] { display: none !important; }

/* ---------- Layout ---------- */
.rp-grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 2fr) minmax(0, 1fr);
  gap: 16px;
  align-items: start;
}
.rp-card {
  background: var(--rp-surface);
  border: 1px solid var(--rp-border);
  border-radius: 16px;
  box-shadow: 0 4px 16px rgba(0,0,0,.04);
}
.rp-panel {
  display: flex;
  flex-direction: column;
  max-height: 720px;
  min-height: 0;
  overflow: hidden;
}
.rp-panel-head {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 8px;
  padding: 16px 18px 8px;
}
.rp-panel-head h3 {
  margin: 0;
  font-family: var(--rp-head-font);
  font-size: 19px;
  color: var(--rp-text);
}
.rp-sub { font-size: 12px; color: var(--rp-muted); }

/* ---------- Buttons ---------- */
.rp-btn {
  font-family: inherit;
  font-size: 13px;
  font-weight: 600;
  line-height: 1.2;
  padding: 10px 14px;
  border-radius: 10px;
  border: 1px solid var(--rp-border);
  background: var(--rp-surface);
  color: var(--rp-text);
  cursor: pointer;
  transition: background .2s ease, border-color .2s ease, color .2s ease, box-shadow .2s ease;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}
.rp-btn:hover { background: #F0EFEA; border-color: var(--rp-primary); }
.rp-btn:disabled { opacity: .5; cursor: not-allowed; }
.rp-btn svg { width: 15px; height: 15px; fill: none; stroke: currentColor; stroke-width: 2; stroke-linecap: round; stroke-linejoin: round; flex-shrink: 0; }
.rp-btn-primary { background: var(--rp-primary); border-color: var(--rp-primary); color: #fff; }
.rp-btn-primary:hover { background: var(--rp-primary-hover); border-color: var(--rp-primary-hover); }
.rp-btn-danger { color: var(--rp-danger); }
.rp-btn-danger:hover { border-color: var(--rp-danger); background: #fbeeed; }
.rp-btn-sm { padding: 7px 10px; font-size: 12px; border-radius: 9px; }
.rp-root button:focus-visible, .rp-root input:focus-visible, .rp-viewer:focus-visible {
  outline: 2px solid var(--rp-primary);
  outline-offset: 2px;
}

/* ---------- Catalogue ---------- */
.rp-search { padding: 4px 18px 10px; }
.rp-input {
  width: 100%;
  padding: 10px 14px;
  font-family: inherit;
  font-size: 13.5px;
  font-weight: 500;
  color: var(--rp-text);
  background: var(--rp-surface);
  border: 2px solid var(--rp-border);
  border-radius: 10px;
  outline: none;
  transition: border-color .2s ease;
}
.rp-input:focus { border-color: var(--rp-primary); }
.rp-chips {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  padding: 0 18px 12px;
}
.rp-chip {
  font-family: inherit;
  font-size: 12px;
  font-weight: 600;
  padding: 6px 11px;
  border-radius: 20px;
  border: 1px solid var(--rp-border);
  background: var(--rp-surface);
  color: var(--rp-muted);
  cursor: pointer;
  transition: all .2s ease;
}
.rp-chip:hover { border-color: var(--rp-primary); color: var(--rp-text); }
.rp-chip.active { background: var(--rp-primary); border-color: var(--rp-primary); color: #fff; }
.rp-products {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  padding: 4px 18px 18px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.rp-product {
  display: grid;
  grid-template-columns: 64px minmax(0, 1fr);
  gap: 10px 12px;
  padding: 10px;
  border: 1px solid var(--rp-border);
  border-radius: 12px;
  background: #FCFCFA;
}
.rp-thumb {
  width: 64px;
  height: 64px;
  border-radius: 10px;
  background: #F0EFEA;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
  overflow: hidden;
}
.rp-thumb img { width: 100%; height: 100%; object-fit: cover; display: block; }
.rp-pname { font-weight: 700; font-size: 13.5px; overflow-wrap: anywhere; }
.rp-pcat { font-size: 11.5px; color: var(--rp-muted); margin-top: 1px; }
.rp-pprice { font-family: var(--rp-head-font); font-size: 15px; font-weight: 700; color: var(--rp-primary); margin-top: 3px; }
.rp-add { grid-column: 1 / -1; }
.rp-empty { padding: 28px 12px; text-align: center; color: var(--rp-muted); font-size: 13px; }

/* ---------- Stage (room settings + viewer) ---------- */
.rp-stage { display: flex; flex-direction: column; gap: 12px; min-width: 0; }
.rp-roomcfg {
  display: flex;
  flex-wrap: wrap;
  align-items: flex-end;
  gap: 12px 16px;
  padding: 12px 16px;
}
.rp-roomcfg-title { font-family: var(--rp-head-font); font-size: 16px; font-weight: 700; margin-right: auto; align-self: center; }
.rp-field { display: flex; flex-direction: column; gap: 4px; }
.rp-field label { font-size: 10.5px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--rp-muted); }
.rp-field .rp-input { width: 92px; padding: 8px 10px; }

.rp-viewer {
  position: relative;
  width: 100%;
  height: clamp(420px, 64vh, 620px);
  border-radius: 16px;
  overflow: hidden;
  border: 1px solid var(--rp-border);
  background: linear-gradient(180deg, #F7F7F3 0%, #E9E8E1 100%);
  outline: none;
}
.rp-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%;
  height: 100%;
  display: block;
  touch-action: none;
}
.rp-overlay-top {
  position: absolute;
  top: 10px; right: 10px;
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  justify-content: flex-end;
  max-width: calc(100% - 20px);
}
.rp-overlay-top .rp-btn { background: rgba(255,255,255,.92); backdrop-filter: blur(4px); }
.rp-dims {
  position: absolute;
  top: 10px; left: 10px;
  background: rgba(255,255,255,.92);
  border: 1px solid var(--rp-border);
  border-radius: 20px;
  padding: 6px 12px;
  font-size: 12px;
  font-weight: 600;
  color: var(--rp-muted);
  pointer-events: none;
}
.rp-selbar {
  position: absolute;
  bottom: 12px;
  left: 50%;
  transform: translateX(-50%);
  max-width: calc(100% - 20px);
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 8px 10px;
  background: rgba(255,255,255,.96);
  border: 1px solid var(--rp-border);
  border-radius: 14px;
  box-shadow: 0 8px 24px rgba(0,0,0,.12);
}
.rp-selname { font-size: 12.5px; font-weight: 700; padding: 0 6px; max-width: 180px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.rp-status {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 24px;
  text-align: center;
  background: rgba(255,255,255,.82);
  font-size: 14px;
  color: var(--rp-muted);
  z-index: 5;
}
.rp-status.error { color: var(--rp-danger); font-weight: 600; }
.rp-spin {
  width: 26px; height: 26px;
  border: 3px solid var(--rp-border);
  border-top-color: var(--rp-primary);
  border-radius: 50%;
  animation: rp-spin .8s linear infinite;
}
@keyframes rp-spin { to { transform: rotate(360deg); } }
.rp-toast {
  position: absolute;
  bottom: 68px;
  left: 50%;
  transform: translateX(-50%) translateY(8px);
  max-width: calc(100% - 24px);
  background: rgba(44,53,43,.94);
  color: #fff;
  font-size: 12.5px;
  font-weight: 600;
  padding: 9px 16px;
  border-radius: 22px;
  opacity: 0;
  pointer-events: none;
  transition: opacity .25s ease, transform .25s ease;
  text-align: center;
  z-index: 6;
}
.rp-toast.show { opacity: 1; transform: translateX(-50%) translateY(0); }
.rp-hint { margin: 0; text-align: center; font-size: 12px; color: var(--rp-muted); }

/* ---------- Selected furniture ---------- */
.rp-selected-toggle {
  all: unset;
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 8px;
  width: 100%;
  padding: 16px 18px 8px;
  box-sizing: border-box;
  cursor: default;
}
.rp-selected-toggle h3 { margin: 0; font-family: var(--rp-head-font); font-size: 19px; color: var(--rp-text); }
.rp-selected-toggle .rp-caret { display: none; font-size: 12px; color: var(--rp-muted); transition: transform .2s ease; }
.rp-selected-body { display: flex; flex-direction: column; min-height: 0; flex: 1; }
.rp-list { flex: 1; min-height: 0; overflow-y: auto; padding: 4px 18px 8px; display: flex; flex-direction: column; gap: 8px; }
.rp-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 6px 10px;
  padding: 10px;
  border: 1px solid var(--rp-border);
  border-radius: 12px;
  background: #FCFCFA;
}
.rp-row.has-selected { border-color: var(--rp-primary); background: #F3F6F1; }
.rp-row-name {
  all: unset;
  font-weight: 700;
  font-size: 13px;
  cursor: pointer;
  overflow-wrap: anywhere;
}
.rp-row-name:hover { color: var(--rp-primary); text-decoration: underline; }
.rp-row-cat { font-size: 11px; color: var(--rp-muted); }
.rp-qty { display: inline-flex; align-items: center; gap: 6px; justify-self: end; }
.rp-qty button {
  width: 26px; height: 26px;
  border-radius: 8px;
  border: 1px solid var(--rp-border);
  background: #fff;
  font-size: 15px;
  line-height: 1;
  cursor: pointer;
  color: var(--rp-text);
  font-family: inherit;
}
.rp-qty button:hover { border-color: var(--rp-primary); }
.rp-qty span { min-width: 18px; text-align: center; font-weight: 700; font-size: 13px; }
.rp-row-prices { display: flex; flex-direction: column; font-size: 11.5px; color: var(--rp-muted); }
.rp-row-prices strong { font-size: 13px; color: var(--rp-text); }
.rp-remove { justify-self: end; align-self: end; }
.rp-total-block { padding: 12px 18px 8px; border-top: 1px dashed var(--rp-border); }
.rp-total-label { font-size: 11px; letter-spacing: 1px; text-transform: uppercase; color: var(--rp-muted); }
.rp-total-val { font-family: var(--rp-head-font); font-size: 30px; font-weight: 600; line-height: 1.15; }
.rp-actions { display: flex; flex-direction: column; gap: 8px; padding: 6px 18px 12px; }
.rp-saved { padding: 0 18px 14px; font-size: 11.5px; color: var(--rp-muted); }

/* ---------- Confirm modal ---------- */
.rp-modal {
  position: fixed;
  inset: 0;
  z-index: 99999;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: rgba(30,38,29,.55);
}
.rp-modal-box {
  width: 100%;
  max-width: 380px;
  background: #fff;
  border-radius: 16px;
  padding: 22px;
  box-shadow: 0 20px 50px rgba(0,0,0,.25);
}
.rp-modal-box h4 { margin: 0 0 6px; font-family: var(--rp-head-font); font-size: 19px; }
.rp-modal-box p { margin: 0 0 16px; font-size: 13.5px; color: var(--rp-muted); }
.rp-modal-actions { display: flex; gap: 10px; justify-content: flex-end; }

/* ---------- Mobile ---------- */
@media (max-width: 960px) {
  .rp-grid { grid-template-columns: minmax(0, 1fr); }
  .rp-stage { order: 1; }
  .rp-catalogue { order: 2; }
  .rp-selected { order: 3; }
  .rp-panel { max-height: none; }
  .rp-products { max-height: 460px; }
  .rp-viewer { height: min(68vh, 460px); min-height: 320px; }
  .rp-selected-toggle { cursor: pointer; }
  .rp-selected-toggle .rp-caret { display: inline-block; }
  .rp-selected.rp-open .rp-caret { transform: rotate(180deg); }
  .rp-selected:not(.rp-open) .rp-selected-body { display: none; }
  .rp-list { max-height: 360px; }
  .rp-field .rp-input { width: 80px; }
  .rp-roomcfg-title { width: 100%; }
}
@media (max-width: 480px) {
  .rp-root { padding: 0 10px; }
  .rp-btn-sm { padding: 8px 10px; }
}
</style>
