<template>
  <div class="shop-wrapper" id="shopWrapper">
    <div class="shop-header">
      <div class="shop-controls">
        <div class="shop-search-wrap">
          <svg class="shop-search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <circle cx="11" cy="11" r="7"></circle>
            <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
          </svg>
          <input type="text" id="shopSearchInput" class="shop-search-input" placeholder="Search catalog..." aria-label="Search catalog" autocomplete="off">
        </div>
        <div id="shopCategoryFilters"></div>
      </div>
    </div>
    <div class="shop-slider-wrap">
      <div class="shop-nav-arrows" id="shopNavArrows" style="visibility: hidden;">
        <button type="button" class="shop-arrow-btn" id="shopPrevBtn" title="Previous">&#10094;</button>
        <button type="button" class="shop-arrow-btn" id="shopNextBtn" title="Next">&#10095;</button>
      </div>
      <div class="shop-grid" id="shopGridContainer">
        <div class="shop-status">
          <span class="shop-spin"></span> Loading catalog…
        </div>
      </div>
      <div class="shop-mobile-pagination" id="shopMobilePagination">
        <button type="button" class="shop-arrow-btn" id="shopMobilePrevBtn" title="Previous Page">&#10094;</button>
        <span class="shop-page-info" id="shopPageInfo">1 / 1</span>
        <button type="button" class="shop-arrow-btn" id="shopMobileNextBtn" title="Next Page">&#10095;</button>
      </div>
    </div>
  </div>
  <div class="shop-modal-backdrop" id="shopImageModal" aria-hidden="true">
    <div class="shop-modal-content">
      <button type="button" class="shop-modal-close" id="shopModalClose" aria-label="Close modal">&times;</button>
      <img id="shopModalImg" src="" alt="Enlarged product view">
      <div class="shop-modal-caption" id="shopModalCaption"></div>
    </div>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'

// The original <script> code runs once, after the HTML above is on the page.
onMounted(() => {
  (function () {
    const CONFIG = {
      SHEET_ID: '1aoMPw_jfjZjQ2Q-Wtg4zXkTItbE89hu-P2X8qZwfkyY',
      TAB: 'Products',
      MOBILE_PAGE_SIZE: 6,
      FEATURED_CATEGORY: 'Mini Set'
    };

    let catalogProducts = [];
    let categories = [];
    let currentCategory = 'All';
    let searchQuery = '';
    let currentPage = 1;

    window.selectedShopItems = window.selectedShopItems || [];

    const isMobile = () => window.innerWidth < 640;
    const num = v => Number(String(v == null ? '' : v).replace(/[^0-9.]/g, '')) || 0;
    const esc = s => String(s == null ? '' : s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
    const formatPrice = n => 'RM ' + n.toLocaleString('en-MY', { minimumFractionDigits: 2, maximumFractionDigits: 2 });
    const norm = s => String(s || '').toLowerCase().replace(/[^a-z0-9]/g, '');
    const isFeatured = p => norm(p.category) === norm(CONFIG.FEATURED_CATEGORY) || norm(p.subcategory) === norm(CONFIG.FEATURED_CATEGORY);
    
    function placeholder(text) {
      const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='600' height='400' viewBox='0 0 600 400'><rect width='600' height='400' fill='#F0EFEA'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' font-family='sans-serif' font-size='22' fill='#6C756D'>${esc(text)}</text></svg>`;
      return 'data:image/svg+xml,' + encodeURIComponent(svg);
    }

    function parseCSV(text) {
      const rows = [];
      let row = [];
      let cur = '';
      let q = false;

      for (let i = 0; i < text.length; i++) {
        const c = text[i];
        if (q) {
          if (c === '"' && text[i + 1] === '"') { cur += '"'; i++; }
          else if (c === '"') { q = false; }
          else { cur += c; }
        } else if (c === '"') { q = true; }
        else if (c === ',') { row.push(cur); cur = ''; }
        else if (c === '\n' || c === '\r') {
          if (c === '\r' && text[i + 1] === '\n') i++;
          row.push(cur); rows.push(row); row = []; cur = '';
        } else { cur += c; }
      }
      if (cur !== '' || row.length) { row.push(cur); rows.push(row); }

      const head = (rows.shift() || []).map(h => h.trim());
      return rows.filter(r => r.some(v => v.trim() !== '')).map(r => {
        const o = {};
        head.forEach((h, i) => { o[h] = (r[i] || '').trim(); });
        return o;
      });
    }

    async function fetchTab(tab) {
      const url = `https://docs.google.com/spreadsheets/d/${CONFIG.SHEET_ID}/gviz/tq?tqx=out:csv&headers=1&sheet=${encodeURIComponent(tab)}`;
      const res = await fetch(url);
      if (!res.ok) throw new Error(`Cannot read tab "${tab}" (HTTP ${res.status})`);
      return parseCSV(await res.text());
    }

    function buildCatalog(rows) {
      catalogProducts = rows.map(r => {
        const spec = r['Specification'] || '';
        return {
          id: r['Product ID'],
          category: r['Category'] || 'Other',
          subcategory: r['Subcategory'] || '',
          name: r['Product Name'],
          desc: r['Description'] || '',
          spec: /^(standard|1 piece|1 set)$/i.test(spec) ? '' : spec,
          price: num(r['Price']),
          unit: r['Unit'] || '',
          sold: String(r['Alacart']).trim().toUpperCase() === 'TRUE',
          image: r['Image URL'] || ''
        };
      }).filter(p => p.id && p.name && p.sold);

      categories = [];
      catalogProducts.forEach(p => {
        if (!isFeatured(p) && !categories.includes(p.category)) categories.push(p.category);
      });
      // Mini Set is pinned as the 2nd filter (right after All)
      if (catalogProducts.some(isFeatured)) categories.unshift(CONFIG.FEATURED_CATEGORY);
    }

    function renderFilters() {
      const box = document.getElementById('shopCategoryFilters');
      if (!box) return;
      box.innerHTML = ['All', ...categories].map(cat => {
        const feat = norm(cat) === norm(CONFIG.FEATURED_CATEGORY);
        return `<button type="button" class="shop-cat-btn ${feat ? 'featured' : ''} ${cat === currentCategory ? 'active' : ''}" data-cat="${esc(cat)}">${feat ? '★ ' : ''}${esc(cat)}</button>`;
      }).join('');
    }

    function getFilteredProducts() {
      const q = searchQuery.toLowerCase();
      const list = catalogProducts.filter(p => {
        const matchCat = currentCategory === 'All' ||
          (norm(currentCategory) === norm(CONFIG.FEATURED_CATEGORY) ? isFeatured(p) : p.category === currentCategory);
        const matchSearch = !q || p.name.toLowerCase().includes(q) || p.desc.toLowerCase().includes(q) || p.category.toLowerCase().includes(q);
        return matchCat && matchSearch;
      });
      // in "All", show the recommended Mini Set items first
      return currentCategory === 'All' ? [...list.filter(isFeatured), ...list.filter(p => !isFeatured(p))] : list;
    }

    function renderGrid() {
      const grid = document.getElementById('shopGridContainer');
      const arrows = document.getElementById('shopNavArrows');
      const pageInfo = document.getElementById('shopPageInfo');
      const mobilePrev = document.getElementById('shopMobilePrevBtn');
      const mobileNext = document.getElementById('shopMobileNextBtn');
      if (!grid) return;

      const filtered = getFilteredProducts();

      if (filtered.length === 0) {
        grid.innerHTML = `<div class="shop-status">No products found matching your filter or search.</div>`;
        if (arrows) arrows.style.visibility = 'hidden';
        return;
      }

      let displayProducts = filtered;

      // Apply 6-card limit pagination on mobile
      if (isMobile()) {
        const totalPages = Math.ceil(filtered.length / CONFIG.MOBILE_PAGE_SIZE) || 1;
        if (currentPage > totalPages) currentPage = totalPages;
        if (currentPage < 1) currentPage = 1;

        const start = (currentPage - 1) * CONFIG.MOBILE_PAGE_SIZE;
        displayProducts = filtered.slice(start, start + CONFIG.MOBILE_PAGE_SIZE);

        if (pageInfo) pageInfo.textContent = `${currentPage} / ${totalPages}`;
        if (mobilePrev) mobilePrev.disabled = currentPage === 1;
        if (mobileNext) mobileNext.disabled = currentPage === totalPages;
      }

      grid.innerHTML = displayProducts.map(p => {
        const cartItem = window.selectedShopItems.find(i => i.id === p.id);
        const qty = cartItem ? cartItem.quantity : 0;
        const imgSrc = p.image || placeholder(p.name);

        return `
          <div class="shop-card ${qty > 0 ? 'in-cart' : ''} ${isFeatured(p) ? 'featured' : ''}"data-id="${esc(p.id)}">
            <div class="shop-card-img-wrap shop-img-trigger" data-img="${esc(imgSrc)}" data-title="${esc(p.name)}" title="Click to view full image">
              <img class="shop-card-img" src="${esc(imgSrc)}" alt="${esc(p.name)}" loading="lazy" onError="this.onerror=null;this.src='${placeholder(p.name)}';"/>
              <span class="shop-card-cat-tag">${isFeatured(p) ? '<span class="shop-card-reco">★ Recommended</span>' : ''}${esc(p.category)}</span>
            </div>
            <div class="shop-card-body">
              <h3 class="shop-card-title">${esc(p.name)}</h3>
              <p class="shop-card-desc">${esc(p.desc)} ${p.spec && p.spec !== 'Standard' ? `<br><em>${esc(p.spec)}</em>` : ''}</p>
              <div class="shop-card-footer">
                <div class="shop-card-price">
                  ${formatPrice(p.price)}
                  ${p.unit ? `<small>/ ${esc(p.unit)}</small>` : ''}
                </div>
                ${qty > 0 ? `
                  <div class="shop-qty-control">
                    <button type="button" class="shop-qty-btn shop-dec-btn" data-id="${esc(p.id)}">-</button>
                    <span class="shop-qty-num">${qty}</span>
                    <button type="button" class="shop-qty-btn shop-inc-btn" data-id="${esc(p.id)}">+</button>
                  </div>
                ` : `
                  <button type="button" class="shop-add-btn" data-id="${esc(p.id)}">Add to Order</button>
                `}
              </div>
            </div>
          </div>
        `;
      }).join('');

      if (arrows) arrows.style.visibility = (!isMobile() && filtered.length > 4) ? 'visible' : 'hidden';
    }

    function updateCart(id, change) {
      const product = catalogProducts.find(p => p.id === id);
      if (!product) return;

      let cartItem = window.selectedShopItems.find(i => i.id === id);

      if (!cartItem && change > 0) {
        cartItem = { ...product, quantity: change };
        window.selectedShopItems.push(cartItem);
      } else if (cartItem) {
        cartItem.quantity += change;
        if (cartItem.quantity <= 0) {
          window.selectedShopItems = window.selectedShopItems.filter(i => i.id !== id);
        }
      }

      window.dispatchEvent(new CustomEvent('cart:updated', { detail: { items: window.selectedShopItems } }));
      renderGrid();
    }

    function setupLightbox() {
      const modal = document.getElementById('shopImageModal');
      const modalImg = document.getElementById('shopModalImg');
      const modalCaption = document.getElementById('shopModalCaption');
      const closeBtn = document.getElementById('shopModalClose');
      const grid = document.getElementById('shopGridContainer');

      if (!modal || !grid) return;

      grid.addEventListener('click', e => {
        const trigger = e.target.closest('.shop-img-trigger');
        if (trigger) {
          modalImg.src = trigger.dataset.img;
          modalCaption.textContent = trigger.dataset.title;
          modal.classList.add('open');
          modal.setAttribute('aria-hidden', 'false');
        }
      });

      function closeModal() {
        modal.classList.remove('open');
        modal.setAttribute('aria-hidden', 'true');
        modalImg.src = '';
      }

      if (closeBtn) closeBtn.addEventListener('click', closeModal);
      modal.addEventListener('click', e => {
        if (e.target === modal) closeModal();
      });
      document.addEventListener('keydown', e => {
        if (e.key === 'Escape' && modal.classList.contains('open')) closeModal();
      });
    }

    function setupEvents() {
      const filterBox = document.getElementById('shopCategoryFilters');
      if (filterBox) {
        filterBox.addEventListener('click', e => {
          const btn = e.target.closest('.shop-cat-btn');
          if (!btn) return;
          currentCategory = btn.dataset.cat;
          currentPage = 1;
          renderFilters();
          renderGrid();
        });
      }

      const searchInput = document.getElementById('shopSearchInput');
      if (searchInput) {
        let timer;
        searchInput.addEventListener('input', e => {
          clearTimeout(timer);
          timer = setTimeout(() => {
            searchQuery = e.target.value.trim();
            currentPage = 1;
            renderGrid();
          }, 200);
        });
      }

      const grid = document.getElementById('shopGridContainer');
      if (grid) {
        grid.addEventListener('click', e => {
          const addBtn = e.target.closest('.shop-add-btn');
          const incBtn = e.target.closest('.shop-inc-btn');
          const decBtn = e.target.closest('.shop-dec-btn');

          if (addBtn) updateCart(addBtn.dataset.id, 1);
          if (incBtn) updateCart(incBtn.dataset.id, 1);
          if (decBtn) updateCart(decBtn.dataset.id, -1);
        });
      }

      const prevBtn = document.getElementById('shopPrevBtn');
      const nextBtn = document.getElementById('shopNextBtn');
      if (prevBtn && grid) prevBtn.addEventListener('click', () => grid.scrollBy({ left: -320, behavior: 'smooth' }));
      if (nextBtn && grid) nextBtn.addEventListener('click', () => grid.scrollBy({ left: 320, behavior: 'smooth' }));

      // Mobile Pagination controls
      const mobilePrev = document.getElementById('shopMobilePrevBtn');
      const mobileNext = document.getElementById('shopMobileNextBtn');
      if (mobilePrev) {
        mobilePrev.addEventListener('click', () => {
          if (currentPage > 1) {
            currentPage--;
            renderGrid();
          }
        });
      }
      if (mobileNext) {
        mobileNext.addEventListener('click', () => {
          const totalPages = Math.ceil(getFilteredProducts().length / CONFIG.MOBILE_PAGE_SIZE);
          if (currentPage < totalPages) {
            currentPage++;
            renderGrid();
          }
        });
      }

      window.addEventListener('resize', () => {
        renderGrid();
      });
    }

    async function init() {
      try {
        const rows = await fetchTab(CONFIG.TAB);
        buildCatalog(rows);
        renderFilters();
        renderGrid();
        setupEvents();
        setupLightbox();
      } catch (err) {
        console.error(err);
        const grid = document.getElementById('shopGridContainer');
        if (grid) {
          grid.innerHTML = `<div class="shop-status">Failed to load catalog. <button type="button" onclick="location.reload()">Retry</button></div>`;
        }
      }
    }

    // Quotation changed the cart (qty / remove) -> redraw the cards
    window.addEventListener('cart:external', () => { try { renderGrid(); } catch (e) {} });

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', init);
    } else {
      init();
    }
  })();
})
</script>

<style>
:root {
  --shop-primary: #4A5D45;
  --shop-primary-hover: #3b4b37;
  --shop-surface: #FFFFFF;
  --shop-surface-1: #F9F9F7;
  --shop-surface-2: #F0EFEA;
  --shop-border: #E2E0D8;
  --shop-text-main: #2C352B;
  --shop-text-muted: #6C756D;
  --shop-accent: #C18C5D;

  --contentfont: 'Plus Jakarta Sans', system-ui, -apple-system, sans-serif;
  --headlinefont: 'Playfair Display', Georgia, serif;
}

/* FLEXIBLE MAIN WRAPPER */
.shop-wrapper {
  width: 100%;
  max-width: 1200px;
  margin: 0 auto 32px auto;
  font-family: var(--contentfont);
  color: var(--shop-text-main);
  padding: 16px;
  box-sizing: border-box;
  overflow-x: hidden;
}

.shop-wrapper *,
.shop-wrapper *::before,
.shop-wrapper *::after {
  box-sizing: border-box;
}

/* HEADER & CONTROLS */
.shop-header {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  margin-bottom: 20px;
  padding-bottom: 20px;
  border-bottom: 1px solid var(--shop-border);
}

.shop-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  width: 100%;
}

/* SEARCH */
.shop-search-wrap {
  position: relative;
  width: 100%;
  max-width: 420px;
  transition: max-width 0.35s cubic-bezier(0.4, 0, 0.2, 1);
}

.shop-search-wrap:focus-within {
  max-width: 560px;
}

.shop-search-icon {
  position: absolute;
  left: 18px;
  top: 50%;
  width: 18px;
  height: 18px;
  transform: translateY(-50%);
  color: var(--shop-text-muted);
  pointer-events: none;
  transition: color 0.25s ease, transform 0.35s cubic-bezier(0.4, 0, 0.2, 1);
}

.shop-search-wrap:focus-within .shop-search-icon {
  color: var(--shop-primary);
  transform: translateY(-50%) scale(1.12);
}

.shop-search-input {
  width: 100%;
  padding: 12px 20px 12px 48px;
  border: 1px solid var(--shop-border);
  border-radius: 999px;
  background: var(--shop-surface);
  color: var(--shop-text-main);
  font-family: var(--contentfont);
  font-size: 14px;
  outline: none;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
  transition: border-color 0.25s ease, box-shadow 0.25s ease;
}

.shop-search-input::placeholder {
  color: var(--shop-text-muted);
}

.shop-search-input:focus {
  border-color: var(--shop-primary);
  box-shadow: 0 0 0 4px rgba(74, 93, 69, 0.12), 0 6px 16px rgba(0, 0, 0, 0.06);
}

/* CATEGORY FILTERS */
#shopCategoryFilters {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 8px;
  max-width: 100%;
}

.shop-cat-btn {
  padding: 7px 14px;
  border-radius: 20px;
  border: 1px solid var(--shop-border);
  background: var(--shop-surface);
  color: var(--shop-text-main);
  font-family: var(--contentfont);
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.shop-cat-btn:hover {
  border-color: var(--shop-primary);
  color: var(--shop-primary);
}

.shop-cat-btn.active {
  background: var(--shop-primary);
  color: #FFFFFF;
  border-color: var(--shop-primary);
}

/* CAROUSEL & BASE GRID (> 1000px: 4 COLUMNS) */
.shop-slider-wrap {
  position: relative;
  width: 100%;
  overflow: hidden;
}

.shop-nav-arrows {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-bottom: 12px;
}

.shop-arrow-btn {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  border: 1px solid var(--shop-border);
  background: var(--shop-surface);
  color: var(--shop-text-main);
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.shop-arrow-btn:hover:not(:disabled) {
  background: var(--shop-primary);
  color: #ffffff;
  border-color: var(--shop-primary);
}

.shop-arrow-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.shop-grid {
  --shop-cols: 4;
  --shop-gap: 20px;
  width: 100%;
  display: grid;
  grid-auto-flow: column;
  grid-auto-columns: calc((100% - (var(--shop-cols) - 1) * var(--shop-gap)) / var(--shop-cols));
  gap: var(--shop-gap);
  overflow-x: auto;
  overflow-y: hidden;
  scroll-snap-type: x mandatory;
  padding-bottom: 8px;
  scrollbar-width: none;
}

.shop-grid::-webkit-scrollbar {
  display: none;
}

/* MOBILE PAGINATION BAR */
.shop-mobile-pagination {
  display: none;
  justify-content: center;
  align-items: center;
  gap: 16px;
  margin-top: 16px;
}

.shop-page-info {
  font-size: 13px;
  font-weight: 600;
  color: var(--shop-text-muted);
}

/* PRODUCT CARD STYLES */
.shop-card {
  background: var(--shop-surface);
  border: 1px solid var(--shop-border);
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  transition: all 0.25s ease;
  position: relative;
  scroll-snap-align: start;
  min-width: 0;
}

.shop-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(0,0,0,0.06);
  border-color: var(--shop-primary);
}

.shop-card.in-cart {
  border-color: var(--shop-primary);
  box-shadow: 0 0 0 1px var(--shop-primary);
}

.shop-card-img-wrap {
  width: 100%;
  height: 180px;
  position: relative;
  overflow: hidden;
  background: var(--shop-surface-2);
  cursor: pointer;
}

.shop-card-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s ease;
}

.shop-card:hover .shop-card-img {
  transform: scale(1.04);
}

.shop-card-cat-tag {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(255, 255, 255, 0.88);
  backdrop-filter: blur(4px);
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  padding: 3px 8px;
  border-radius: 12px;
  color: var(--shop-text-main);
  pointer-events: none;
}

.shop-card-body {
  padding: 16px;
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  min-width: 0;
}

.shop-card-title {
  font-size: 15px;
  font-weight: 600;
  margin: 0 0 6px 0;
  color: var(--shop-text-main);
  overflow-wrap: anywhere;
  word-break: break-word;
}

.shop-card-desc {
  font-size: 12px;
  color: var(--shop-text-muted);
  line-height: 1.5;
  margin-bottom: 14px;
  flex-grow: 1;
  overflow-wrap: anywhere;
  word-break: break-word;
  white-space: pre-line; /* Renders line breaks and bullet formatting from sheet */
  max-height: 140px;
  overflow-y: auto; /* Adds a clean inner scroll if item list is extra long */
}

.shop-card-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  border-top: 1px solid var(--shop-border);
  padding-top: 12px;
  margin-top: auto;
}

.shop-card-price {
  font-family: var(--headlinefont);
  font-size: 16px;
  font-weight: 700;
  color: var(--shop-primary);
}

.shop-card-price small {
  display: block;
  font-family: var(--contentfont);
  font-size: 11px;
  font-weight: 500;
  color: var(--shop-text-muted);
}

.shop-qty-control {
  display: flex;
  align-items: center;
  gap: 6px;
  background: var(--shop-surface-1);
  border: 1px solid var(--shop-border);
  border-radius: 6px;
  padding: 2px 4px;
}

.shop-qty-btn {
  background: none;
  border: none;
  width: 24px;
  height: 24px;
  font-size: 14px;
  font-weight: bold;
  color: var(--shop-text-main);
  cursor: pointer;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.shop-qty-num {
  font-size: 13px;
  font-weight: 600;
  min-width: 18px;
  text-align: center;
}

.shop-add-btn {
  padding: 7px 14px;
  border-radius: 6px;
  border: 1px solid var(--shop-primary);
  background: var(--shop-primary);
  color: #FFFFFF;
  font-family: var(--contentfont);
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.shop-add-btn:hover {
  background: var(--shop-primary-hover);
}

.shop-status {
  grid-column: 1 / -1;
  width: 100%;
  text-align: center;
  padding: 40px 16px;
  color: var(--shop-text-muted);
  font-size: 14px;
}

.shop-spin {
  display: inline-block;
  width: 14px;
  height: 14px;
  border: 2px solid var(--shop-border);
  border-top-color: var(--shop-primary);
  border-radius: 50%;
  animation: shopspin .8s linear infinite;
  vertical-align: -2px;
  margin-right: 8px;
}

@keyframes shopspin {
  to { transform: rotate(360deg); }
}

/* LIGHTBOX / IMAGE MODAL STYLES */
.shop-modal-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(6px);
  z-index: 99999;
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
  padding: 24px;
  box-sizing: border-box;
}

.shop-modal-backdrop.open {
  opacity: 1;
  pointer-events: auto;
}

.shop-modal-content {
  position: relative;
  max-width: 90vw;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.shop-modal-content img {
  max-width: 100%;
  max-height: 80vh;
  object-fit: contain;
  border-radius: 8px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

.shop-modal-caption {
  color: #fff;
  font-family: var(--contentfont);
  font-size: 15px;
  font-weight: 600;
  margin-top: 12px;
  text-align: center;
  letter-spacing: 0.03em;
}

.shop-modal-close {
  position: absolute;
  top: -45px;
  right: 0;
  background: none;
  border: none;
  color: #fff;
  font-size: 32px;
  cursor: pointer;
  padding: 4px 10px;
  line-height: 1;
  transition: color 0.2s;
}

.shop-modal-close:hover {
  color: var(--shop-accent);
}

/* 2 COLUMNS GRID & MOBILE ADAPTATION (< 640px) */
/* FEATURED CATEGORY (Mini Set) */
.shop-cat-btn.featured { border-color: var(--shop-accent); color: var(--shop-accent); background: #FBF5EE; }
.shop-cat-btn.featured:hover { border-color: var(--shop-accent); color: #a5733f; }
.shop-cat-btn.featured.active { background: var(--shop-accent); border-color: var(--shop-accent); color: #fff; }
.shop-card.featured { border-color: var(--shop-accent); }
.shop-card-reco {
  position: absolute; top: 10px; right: 10px;
  background: var(--shop-accent); color: #fff;
  font-size: 10px; font-weight: 700; letter-spacing: 0.04em;
  padding: 3px 8px; border-radius: 12px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.15); pointer-events: none;
}

@media screen and (max-width: 639px) {
  .shop-wrapper {
    width: 100% !important;
    max-width: 100vw !important;
    padding: 10px !important;
    margin: 0 !important;
  }

  .shop-header, .shop-controls, .shop-search-wrap {
    width: 100% !important;
    max-width: 100% !important;
  }

  .shop-search-input {
    width: 100% !important;
    font-size: 16px !important;
    padding: 12px 16px 12px 44px !important;
  }

  #shopCategoryFilters {
    width: 100% !important;
    display: flex !important;
    flex-wrap: nowrap !important;
    justify-content: flex-start !important;
    align-items: center !important;
    overflow-x: auto !important;
    gap: 8px !important;
    padding: 8px 12px !important;
    background: var(--shop-surface-1) !important;
    border: 1px solid var(--shop-border) !important;
    border-radius: 12px !important;
    box-sizing: border-box !important;
    scrollbar-width: none !important;
    -webkit-overflow-scrolling: touch;
  }

  #shopCategoryFilters::-webkit-scrollbar {
    display: none !important;
  }

  .shop-cat-btn {
    flex: 0 0 auto !important;
    white-space: nowrap !important;
    padding: 6px 12px !important;
    font-size: 12px !important;
  }

  .shop-nav-arrows {
    display: none !important;
  }

  /* 2 COLUMNS DISPLAY BELOW 640px */
  .shop-grid {
    display: grid !important;
    grid-template-columns: repeat(2, 1fr) !important;
    grid-auto-flow: row !important;
    grid-auto-columns: auto !important;
    width: 100% !important;
    max-width: 100% !important;
    gap: 10px !important;
    overflow-x: hidden !important;
    overflow-y: visible !important;
    scroll-snap-type: none !important;
    padding: 0 !important;
  }

  .shop-card {
    width: 100% !important;
    border-radius: 10px !important;
    scroll-snap-align: none !important;
    min-width: 0 !important;
  }

  .shop-card-img-wrap {
    height: 120px !important;
  }

  .shop-card-body {
    padding: 10px !important;
    min-width: 0 !important;
  }

  .shop-card-title {
    font-size: 13px !important;
    margin-bottom: 4px !important;
  }

  .shop-card-desc {
    font-size: 10px !important;
    line-height: 1.4 !important;
    max-height: 110px !important;
    margin-bottom: 8px !important;
  }

  .shop-card-footer {
    flex-direction: column !important;
    align-items: flex-start !important;
    gap: 6px !important;
    padding-top: 8px !important;
  }

  .shop-card-price {
    font-size: 14px !important;
  }

  .shop-add-btn {
    width: 100% !important;
    padding: 6px 8px !important;
    font-size: 11px !important;
    text-align: center;
  }

  .shop-qty-control {
    width: 100% !important;
    justify-content: space-between;
  }

  .shop-qty-btn {
    width: 24px !important;
    height: 24px !important;
  }

  .shop-mobile-pagination {
    display: flex !important;
  }
}
</style>