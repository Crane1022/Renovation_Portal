<template>
  <div class="prop-card-wrapper">
    <div class="prop-card">
      <div class="prop-card-header">
        <h3>Which property are you renovating?</h3>
        <p>Select your property from the list or choose 'Other' to type your own property name.</p>
      </div>
      <div class="prop-row">
        <div class="prop-field-group">
          <label for="prop-selector-dropdown">Property Location</label>
          <select id="prop-selector-dropdown" class="prop-select-field" disabled>
            <option value="ALL">Loading available properties...</option>
          </select>
          <input type="text" id="prop-custom-input" class="prop-select-field prop-custom-input" placeholder="Type your property name here..." />
        </div>
        <div class="prop-field-group">
          <label for="prop-bed-select">Bedrooms</label>
          <select id="prop-bed-select" class="prop-select-field">
            <option value="ALL">All Bedroom Types</option>
            <option value="1">1 Bedroom / Studio</option>
            <option value="2">2 Bedrooms</option>
            <option value="3">3 Bedrooms</option>
            <option value="4">4 Bedrooms</option>
            <option value="5">5 Bedrooms</option>
          </select>
        </div>
        <div class="prop-field-group">
          <label for="prop-size-input">Property Size (sqft)</label>
          <input type="number" id="prop-size-input" class="prop-select-field" placeholder="e.g. 551, 841, 1005" min="0" />
        </div>
      </div>
    </div>
  </div>
  <div class="pkg-status" id="pkg-status"><span class="pkg-spin"></span>Loading packages…</div>
  <div class="pkg-layout-wrapper" id="pkg-wrapper" hidden>
    <div class="pkg-split-container">
      <div class="pkg-list-column" id="pkg-list"></div>
      <div class="pkg-detail-pane">
        <div class="pkg-hero-image-wrap" id="pkg-hero-trigger" title="Click to view full image">
          <img id="pkg-demo-img" src="" alt="Package preview">
          <span class="pkg-hero-badge" id="pkg-demo-caption"></span>
        </div>
        <div class="pkg-detail-body">
          <div class="pkg-theme-switch" id="pkg-theme-switch" role="group" aria-label="Choose theme"></div>
          <div class="pkg-detail-header">
            <div>
              <span class="pkg-tag tag-modern" id="pkg-detail-tag"></span>
              <h3 class="pkg-detail-title" id="pkg-detail-title"></h3>
            </div>
            <div class="pkg-detail-price-tag" id="pkg-detail-price"></div>
          </div>
          <div id="pkg-detail-content"></div>
        </div>
      </div>
    </div>
  </div>
  <div class="pkg-modal-backdrop" id="pkg-image-modal" aria-hidden="true">
    <div class="pkg-modal-content">
      <button type="button" class="pkg-modal-close" id="pkg-modal-close" aria-label="Close modal">&times;</button>
      <img id="pkg-modal-img" src="" alt="Enlarged preview">
      <div class="pkg-modal-caption" id="pkg-modal-caption"></div>
    </div>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'

// The original <script> code runs once, after the HTML above is on the page.
onMounted(() => {
  (function () {
    const SHEET_ID = '1ikb8sfE3h2kMOVmklNoRyDyEo8o8lBUFtjUcU2kGotw';
    const TAB_NAME = 'Packages';

    function parseCSV(text) {
      const rows = []; let row = [], cur = '', q = false;
      for (let i = 0; i < text.length; i++) {
        const c = text[i];
        if (q) {
          if (c === '"' && text[i + 1] === '"') { cur += '"'; i++; }
          else if (c === '"') q = false;
          else cur += c;
        } else if (c === '"') q = true;
        else if (c === ',') { row.push(cur); cur = ''; }
        else if (c === '\n' || c === '\r') {
          if (c === '\r' && text[i + 1] === '\n') i++;
          row.push(cur); rows.push(row); row = []; cur = '';
        } else cur += c;
      }
      if (cur !== '' || row.length) { row.push(cur); rows.push(row); }
      const head = (rows.shift() || []).map(h => h.trim());
      return rows.filter(r => r.some(v => v.trim() !== '')).map(r => {
        const o = {}; head.forEach((h, i) => o[h] = (r[i] || '').trim()); return o;
      });
    }

    async function loadProperties() {
      const dropdown = document.getElementById('prop-selector-dropdown');
      const customInput = document.getElementById('prop-custom-input');
      const bedSelect = document.getElementById('prop-bed-select');
      const sizeInput = document.getElementById('prop-size-input');

      // Fires the filter event. Defined first so it also works if the sheet fails to load.
      function dispatchFilterEvent(source) {
        const isCustom = dropdown.value === 'CUSTOM_OTHER';
        // Only a property picked from the list is used to FILTER packages.
        // When 'Other' is chosen the filter is empty, so ALL packages are shown.
        const listedProp = (isCustom || dropdown.value === 'ALL') ? '' : dropdown.value;
        const customPropName = isCustom ? customInput.value.trim() : '';
        const enteredSize = Number(sizeInput.value) || 0;
        const selectedBeds = bedSelect.value === 'ALL' ? 0 : Number(bedSelect.value);

        // The name to save with the order: typed name for 'Other', listed name otherwise
        window.selectedProperty = isCustom ? customPropName : listedProp;
        window.selectedPropertyIsCustom = isCustom;
        window.selectedSize = enteredSize;
        window.selectedBedrooms = selectedBeds;

        // Typing a custom name must NOT re-filter or reset the chosen package
        if (source === 'custom-name') {
          window.dispatchEvent(new CustomEvent('customPropertyNameChanged', {
            detail: { property: customPropName }
          }));
          return;
        }

        window.dispatchEvent(new CustomEvent('propertySelected', {
          detail: {
            property: listedProp,           // used for filtering ('' = show all)
            customProperty: customPropName, // what the customer typed
            isCustom,
            size: enteredSize,
            bedrooms: selectedBeds
          }
        }));
      }

      function wireEvents() {
        dropdown.addEventListener('change', (e) => {
          if (e.target.value === 'CUSTOM_OTHER') {
            customInput.style.display = 'block';
            customInput.value = '';
            customInput.focus();
          } else {
            customInput.style.display = 'none';
            customInput.value = '';
          }
          dispatchFilterEvent();
        });

        customInput.addEventListener('input', () => dispatchFilterEvent('custom-name'));
        bedSelect.addEventListener('change', () => dispatchFilterEvent());
        sizeInput.addEventListener('input', () => dispatchFilterEvent());
      }

      try {
        const url = `https://docs.google.com/spreadsheets/d/${SHEET_ID}/gviz/tq?tqx=out:csv&headers=1&sheet=${encodeURIComponent(TAB_NAME)}`;
        const res = await fetch(url);
        if (!res.ok) throw new Error('Network error');
        const data = parseCSV(await res.text());

        const propSet = new Set();
        data.forEach(row => {
          const availableVal = (row['Available'] !== undefined && row['Available'] !== '') ? String(row['Available']).trim().toUpperCase() : 'TRUE';
          if (availableVal === 'FALSE' || availableVal === '0' || availableVal === 'NO') return;

          const raw = row['Property Name'] || '';
          raw.split(',').map(s => s.trim()).filter(Boolean).forEach(p => propSet.add(p));
        });

        const properties = Array.from(propSet).sort();

        let options = '<option value="ALL">-- Select your property --</option>';
        properties.forEach(p => {
          options += `<option value="${p.replace(/"/g, '&quot;')}">${p}</option>`;
        });
        options += '<option value="CUSTOM_OTHER">✏️ Other (Type manually)...</option>';

        dropdown.innerHTML = options;
        dropdown.disabled = false;
        wireEvents();

      } catch (err) {
        console.error(err);
        dropdown.innerHTML = '<option value="ALL">All Properties</option><option value="CUSTOM_OTHER">✏️ Other (Type manually)...</option>';
        dropdown.disabled = false;
        wireEvents();
      }
    }

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', loadProperties);
    } else {
      loadProperties();
    }
  })();

  (function () {
  const CONFIG = {
    SHEET_ID: '1ikb8sfE3h2kMOVmklNoRyDyEo8o8lBUFtjUcU2kGotw',
    TABS: { packages: 'Packages', items: 'Package_Items', products: 'Products' }
  };

  const U = id => `https://images.unsplash.com/${id}?auto=format&fit=crop&w=1000&q=80`;
  const FALLBACK_IMAGES = {
    mod: [U('photo-1618221195710-dd6b41faaea6'), U('photo-1600210492486-724fe5c67fb0'), U('photo-1600585154340-be6161a56a0c')],
    ind: [U('photo-1505691938895-1758d7feb511'), U('photo-1513694203232-719a280e022f')],
    other: [U('photo-1586023492125-27b2c045efd7')]
  };

  function parseCSV(text) {
    const rows = []; let row = [], cur = '', q = false;
    for (let i = 0; i < text.length; i++) {
      const c = text[i];
      if (q) {
        if (c === '"' && text[i + 1] === '"') { cur += '"'; i++; }
        else if (c === '"') q = false;
        else cur += c;
      } else if (c === '"') q = true;
      else if (c === ',') { row.push(cur); cur = ''; }
      else if (c === '\n' || c === '\r') {
        if (c === '\r' && text[i + 1] === '\n') i++;
        row.push(cur); rows.push(row); row = []; cur = '';
      } else cur += c;
    }
    if (cur !== '' || row.length) { row.push(cur); rows.push(row); }
    const head = (rows.shift() || []).map(h => h.trim());
    return rows.filter(r => r.some(v => v.trim() !== '')).map(r => {
      const o = {}; head.forEach((h, i) => o[h] = (r[i] || '').trim()); return o;
    });
  }

  async function fetchTab(tab) {
    const m = String(CONFIG.SHEET_ID).match(/\/d\/([\w-]+)/);
    const id = m ? m[1] : String(CONFIG.SHEET_ID).trim();
    const url = `https://docs.google.com/spreadsheets/d/${id}/gviz/tq?tqx=out:csv&headers=1&sheet=${encodeURIComponent(tab)}`;
    const res = await fetch(url);
    if (!res.ok) throw new Error(`Cannot read tab "${tab}" (HTTP ${res.status})`);
    return parseCSV(await res.text());
  }

  const money = n => 'RM ' + Math.round(n).toString().replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  const num = v => Number(String(v == null ? '' : v).replace(/[^0-9.]/g, '')) || 0;
  const esc = s => String(s).replace(/[&<>"]/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' }[c]));

  let pkgData = {};
  let BEDS = [];
  let THEMES = {};
  let currentPropertyFilter = '';      // used ONLY to filter packages ('' = show all)
  let currentPropertyName = '';        // name saved with the order (listed OR typed by customer)
  let currentPropertyIsCustom = false; // true when customer used "Other"
  let currentSizeFilter = 0;
  let currentBedFilter = 0;
  let currentBeds = 0, currentTheme = '';

  const ALACARTE = {
    tagClass: 'tag-alacarte', tagText: 'Custom Selection',
    title: 'Custom À La Carte Fit-Out', price: 'RM 0', numericPrice: 0,
    desc: 'Custom itemized fit-out selected from store catalog',
    image: FALLBACK_IMAGES.other[0], demoImage: '', caption: 'Custom Mix & Match Selection',
    sections: [{ room: 'Flexible Fit-Out Plan', items: [
      'No bundled base package items included',
      'Select individual items freely from the catalog below',
      'Subtotal is calculated directly based on items added to cart',
      'Full customization and layout flexibility'
    ]}]
  };

  function itemLabel(it, prod) {
    const qty = num(it['Qty']) || 1;
    const unit = (prod['Unit'] || '').toLowerCase();
    const spec = (prod['Specification'] || '').trim();
    const name = it['Product Name'] || prod['Product Name'] || it['Product ID'];
    if (unit === 'sq ft') return esc(`${name} (${qty} sq ft)`);
    const showSpec = spec && !/^(standard|1 piece|1 set)$/i.test(spec) && !String(name).toLowerCase().includes(spec.toLowerCase());
    return esc(name) + (qty > 1 ? ` x ${qty}` : '') + (showSpec ? ` <small>${esc(spec)}</small>` : '');
  }

  function buildSections(items, prodMap) {
    const areas = [];
    items.forEach(it => {
      let a = areas.find(x => x.area === it['Area']);
      if (!a) areas.push(a = { area: it['Area'], items: [] });
      a.items.push(itemLabel(it, prodMap[it['Product ID']] || {}));
    });
    const out = [];
    areas.forEach(a => {
      const sig = a.items.join('|');
      const prev = out[out.length - 1];
      if (prev && /^Room \d/i.test(a.area) && prev.sig === sig && prev.rooms) prev.rooms.push(a.area.replace(/^Room\s*/i, ''));
      else out.push({ area: a.area, items: a.items, sig, rooms: /^Room \d/i.test(a.area) ? [a.area.replace(/^Room\s*/i, '')] : null });
    });
    return out.map(o => {
      let room = o.area;
      if (o.rooms) {
        room = 'Room ' + (o.rooms.length === 1 ? o.rooms[0]
          : o.rooms.slice(0, -1).join(', ') + ' & ' + o.rooms[o.rooms.length - 1] + ' (each room)');
      }
      return { room, items: o.items };
    });
  }

  function buildData(packages, items, products) {
    const prodMap = {}; products.forEach(p => prodMap[p['Product ID']] = p);
    const itemsByPkg = {}; items.forEach(i => (itemsByPkg[i['Package ID']] = itemsByPkg[i['Package ID']] || []).push(i));

    pkgData = { alacarte: ALACARTE }; THEMES = {};
    const bedMap = {}; 
    let themeCount = 0;

    packages.forEach(p => {
      const availableVal = (p['Available'] !== undefined && p['Available'] !== '') ? String(p['Available']).trim().toUpperCase() : 'TRUE';
      if (availableVal === 'FALSE' || availableVal === '0' || availableVal === 'NO') return;

      const beds = parseInt(p['Bedrooms'], 10); if (!beds) return;
      const style = p['Style'] || 'Standard';
    
      const rawProps = p['Property Name'] || '';
      const propArray = rawProps.split(',').map(s => s.trim()).filter(Boolean);
      const pkgSize = num(p['Size']);

      const sl = style.toLowerCase();
      const tKey = sl.startsWith('modern') ? 'mod' : sl.startsWith('industrial') ? 'ind' : sl.replace(/[^a-z0-9]+/g, '-');
      if (!THEMES[tKey]) {
        const m = (p['Package Name'] || '').match(/Package\s+([A-Z])\b/i);
        THEMES[tKey] = { key: tKey, name: style, letter: m ? m[1].toUpperCase() : String.fromCharCode(65 + themeCount),
          tagClass: tKey === 'mod' ? 'tag-modern' : tKey === 'ind' ? 'tag-industrial' : 'tag-alacarte' };
        themeCount++;
      }
      const th = THEMES[tKey];
      const label = p['Property Type'] || `${beds} Bedroom`;

      bedMap[beds] = bedMap[beds] || { label, properties: new Set() };
      propArray.forEach(pr => bedMap[beds].properties.add(pr));

      const imgs = FALLBACK_IMAGES[tKey] || FALLBACK_IMAGES.other;
      const price = num(p['Package Price (RM)']);
      const sizeRecommendationStr = pkgSize > 0 ? ` (Ideal for ~${pkgSize} sqft)` : '';
    
      pkgData[`${tKey}-${beds}br-${p['Package ID']}`] = {
        packageId: p['Package ID'], theme: tKey, beds, properties: propArray, size: pkgSize,
        tagClass: th.tagClass, tagText: `Theme ${th.letter} · ${th.name}`,
        title: `${label} Package – Theme ${th.letter} (${th.name})${sizeRecommendationStr}`,
        price: money(price), numericPrice: price,
        desc: `${label} furniture package, ${th.name} style, suitable for ~${pkgSize} sqft`,
        image: p['Image URL'] || imgs[Math.min(beds, imgs.length) - 1],
        demoImage: p['Demo Image URL'] || '',
        caption: `${th.name} · ${label} (~${pkgSize} sqft)`,
        sections: buildSections(itemsByPkg[p['Package ID']] || [], prodMap)
      };
    });

    BEDS = Object.keys(bedMap).map(Number).sort((a, b) => a - b).map(b => ({
      beds: b,
      label: bedMap[b].label,
      properties: Array.from(bedMap[b].properties)
    }));
  }

  function getFilteredPackages() {
    return Object.values(pkgData).filter(pkg => {
      if (pkg.packageId === undefined) return true; // Keep alacarte
      const matchesProp = !currentPropertyFilter || pkg.properties.some(p => p.toLowerCase().includes(currentPropertyFilter.toLowerCase()));
      const matchesBed = !currentBedFilter || pkg.beds === currentBedFilter;
      return matchesProp && matchesBed;
    });
  }

  function getRecommendedPackageId() {
    if (!currentSizeFilter) return null;
    const validPkgs = getFilteredPackages().filter(p => p.packageId && p.size > 0);
    if (!validPkgs.length) return null;

    let closest = validPkgs[0];
    let minDiff = Math.abs(closest.size - currentSizeFilter);

    validPkgs.forEach(pkg => {
      const diff = Math.abs(pkg.size - currentSizeFilter);
      if (diff < minDiff) {
        minDiff = diff;
        closest = pkg;
      }
    });
    return closest.packageId;
  }

  function getFilteredBeds() {
    const validPkgs = getFilteredPackages();
    const bedSet = new Set(validPkgs.filter(p => p.beds).map(p => p.beds));
    return BEDS.filter(b => bedSet.has(b.beds));
  }

  const themesFor = beds => {
    const validPkgs = getFilteredPackages();
    return Object.values(THEMES).filter(t => {
      return validPkgs.some(pkg => pkg.theme === t.key && pkg.beds === beds);
    });
  };

  function renderList() {
    let h = `<div class="pkg-list-heading">Custom</div>
      <button type="button" class="pkg-nav-card" data-beds="0">
        <div class="pkg-nav-info"><span class="pkg-tag tag-alacarte">Custom Selection</span><h4>I Choose À La Carte</h4></div>
        <div class="pkg-nav-from"><b>RM 0</b></div>
      </button>
      <div class="pkg-list-heading">Choose by bedroom</div>`;

    const filteredBeds = getFilteredBeds();
    const recommendedPkgId = getRecommendedPackageId();

    if (filteredBeds.length === 0) {
      h += `<div style="padding: 16px; text-align: center; color: var(--brand-text-muted); font-size: 13px;">No packages match your criteria.</div>`;
    } else {
      filteredBeds.forEach(b => {
        const ts = themesFor(b.beds);
        const matchingPkgs = ts.map(t => getFilteredPackages().find(p => p.theme === t.key && p.beds === b.beds)).filter(Boolean);
      
        const isRecommended = recommendedPkgId && matchingPkgs.some(p => p.packageId === recommendedPkgId);
        const recPkg = matchingPkgs.find(p => p.packageId === recommendedPkgId);
        const prices = matchingPkgs.map(pkg => `${pkg.tagText.split('·')[0].trim()} · ${pkg.price}`).join('<br>');
        const tag = ts.map(t => t.letter).join(' &amp; ');

        h += `<button type="button" class="pkg-nav-card" data-beds="${b.beds}">
          ${isRecommended ? '<span class="pkg-recommended-badge">Recommended for ~' + recPkg.size + ' sqft</span>' : ''}
          <div class="pkg-nav-info">
            <span class="pkg-tag tag-modern">Theme ${tag}</span>
            <h4>${esc(b.label)}</h4>
            ${matchingPkgs[0] && matchingPkgs[0].size ? `<small>Recommended target: ~${matchingPkgs[0].size} sqft</small>` : ''}
          </div>
          <div class="pkg-nav-from">${prices}</div></button>`;
      });
    }

    const list = document.getElementById('pkg-list');
    list.innerHTML = h;
    list.querySelectorAll('.pkg-nav-card').forEach(c => c.addEventListener('click', () => selectBedroom(Number(c.dataset.beds))));
  }

  function renderThemeSwitch() {
    const box = document.getElementById('pkg-theme-switch');
    const ts = currentBeds ? themesFor(currentBeds) : [];
    box.hidden = ts.length === 0;
  
    const validPkgs = getFilteredPackages();
    box.innerHTML = ts.map(t => {
      const pkg = validPkgs.find(p => p.theme === t.key && p.beds === currentBeds);
      return `<button type="button" class="pkg-theme-btn ${t.key === currentTheme ? 'active' : ''}" data-theme="${t.key}" aria-pressed="${t.key === currentTheme}">
        <small>Theme ${esc(t.letter)}</small><strong>${esc(t.name)}</strong><span>${pkg ? pkg.price : ''}</span></button>`;
    }).join('');
  
    box.querySelectorAll('.pkg-theme-btn').forEach(b => b.addEventListener('click', () => selectTheme(b.dataset.theme)));
  }

  function selectBedroom(beds) {
    if (beds === 0) return selectPkg('alacarte');
    const ts = themesFor(beds); if (!ts.length) return;
    const t = ts.find(x => x.key === currentTheme) || ts[0];
    const pkg = getFilteredPackages().find(p => p.theme === t.key && p.beds === beds);
    if (pkg) selectPkgKey(pkg);
  }

  function selectTheme(theme) { 
    if (currentBeds) {
      const pkg = getFilteredPackages().find(p => p.theme === theme && p.beds === currentBeds);
      if (pkg) selectPkgKey(pkg);
    }
  }

  function selectPkgKey(data) {
    currentBeds = data.beds || 0;
    currentTheme = data.theme || '';

    document.querySelectorAll('#pkg-list .pkg-nav-card').forEach(c => c.classList.toggle('active', Number(c.dataset.beds) === currentBeds));
    renderThemeSwitch();

    const activePrice = data.packageId === undefined ? 0 : data.numericPrice;
    window.selectedPackage = {
      key: data.packageId || 'alacarte', packageId: data.packageId || '', title: data.title, price: activePrice, numericPrice: activePrice, amount: activePrice,
      desc: data.desc, img: data.image, image: data.image, demoImage: data.demoImage || '',
      // property = listed name, or the name the customer typed under "Other"
      property: currentPropertyName, propertyIsCustom: currentPropertyIsCustom
    };
    window.dispatchEvent(new CustomEvent('packageSelected', { detail: window.selectedPackage }));
    window.dispatchEvent(new CustomEvent('packageChange', { detail: window.selectedPackage }));

    document.getElementById('pkg-detail-title').innerText = data.title;
    document.getElementById('pkg-detail-price').innerText = data.price;
    const tag = document.getElementById('pkg-detail-tag'); tag.innerText = data.tagText; tag.className = 'pkg-tag ' + data.tagClass;

    const img = document.getElementById('pkg-demo-img');
    img.style.opacity = '0.3';
    setTimeout(() => { img.src = data.image; img.style.opacity = '1'; }, 150);
    document.getElementById('pkg-demo-caption').innerText = data.caption;

    document.getElementById('pkg-detail-content').innerHTML = data.sections.map(sec =>
      `<div class="pkg-room-sec"><h5>${esc(sec.room)}</h5><ul class="pkg-item-list">${
        sec.items.map(i => `<li><span>${i}</span></li>`).join('')}</ul></div>`).join('');

    if (typeof window.updateQuotationSummary === 'function') window.updateQuotationSummary();
  }

  function selectPkg(key) {
    const pkg = pkgData[key];
    if (pkg) selectPkgKey(pkg);
  }

  // Property / size / bedroom changed (dropdown, bedrooms, size, or switching to/from "Other")
  window.addEventListener('propertySelected', (e) => {
    currentPropertyFilter = e.detail.property;   // '' when "Other" -> all packages are shown
    currentPropertyIsCustom = !!e.detail.isCustom;
    currentPropertyName = currentPropertyIsCustom ? (e.detail.customProperty || '') : e.detail.property;
    currentSizeFilter = e.detail.size;
    currentBedFilter = e.detail.bedrooms;

    renderList();

    const recommendedId = getRecommendedPackageId();
    if (recommendedId) {
      const recPkg = Object.values(pkgData).find(p => p.packageId === recommendedId);
      if (recPkg) {
        selectPkgKey(recPkg);
        return;
      }
    }

    const visibleBeds = getFilteredBeds();
    if (visibleBeds.length > 0) {
      selectBedroom(visibleBeds[0].beds);
    } else {
      selectPkg('alacarte');
    }
  });

  // Customer is typing their own property name under "Other":
  // update the name only, do NOT re-filter or reset the selected package
  window.addEventListener('customPropertyNameChanged', (e) => {
    currentPropertyName = e.detail.property;
    if (window.selectedPackage) {
      window.selectedPackage.property = currentPropertyName;
      window.selectedPackage.propertyIsCustom = true;
      window.dispatchEvent(new CustomEvent('packageChange', { detail: window.selectedPackage }));
    }
    if (typeof window.updateQuotationSummary === 'function') window.updateQuotationSummary();
  });

  async function initPackageSelection() {
    const status = document.getElementById('pkg-status'), wrap = document.getElementById('pkg-wrapper');
    if (!status || !wrap) return;
    if (wrap.dataset.pkgReady === '1') return;
    status.hidden = false; wrap.hidden = true;
    status.innerHTML = '<span class="pkg-spin"></span>Loading packages…';
    try {
      const [packages, items, products] = await Promise.all([
        fetchTab(CONFIG.TABS.packages), fetchTab(CONFIG.TABS.items), fetchTab(CONFIG.TABS.products)
      ]);
      buildData(packages, items, products);
      if (!BEDS.length) throw new Error('No packages found');

      renderList();
      wrap.dataset.pkgReady = '1';
      wrap.hidden = false; status.hidden = true;
      selectBedroom(BEDS[0].beds);
    } catch (e) {
      console.error(e);
      status.innerHTML = 'Sorry, we could not load packages.';
    }
  }

  (() => {
    const modal = document.getElementById('pkg-image-modal');
    const modalImg = document.getElementById('pkg-modal-img');
    const modalCaption = document.getElementById('pkg-modal-caption');
    const closeBtn = document.getElementById('pkg-modal-close');
    const heroTrigger = document.getElementById('pkg-hero-trigger');
    const demoImg = document.getElementById('pkg-demo-img');
    const demoCaption = document.getElementById('pkg-demo-caption');

    if (heroTrigger) {
      heroTrigger.addEventListener('click', () => {
        if (demoImg && demoImg.src) {
          modalImg.src = demoImg.src;
          modalCaption.textContent = demoCaption ? demoCaption.textContent : '';
          modal.classList.add('open');
          modal.setAttribute('aria-hidden', 'false');
        }
      });
    }

    function closeModal() {
      modal.classList.remove('open');
      modal.setAttribute('aria-hidden', 'true');
      modalImg.src = '';
    }

    if (closeBtn) closeBtn.addEventListener('click', closeModal);
    modal.addEventListener('click', (e) => { if (e.target === modal) closeModal(); });

    initPackageSelection();
  })();
  })();
})
</script>

<style>
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
  --contentfont: 'Plus Jakarta Sans', system-ui, -apple-system, sans-serif;
  --headlinefont: 'Playfair Display', Georgia, serif;
}

.prop-card-wrapper, .pkg-layout-wrapper {
  max-width: 1200px;
  margin: 0 auto 24px;
  padding: 0 16px;
  font-family: var(--contentfont);
  box-sizing: border-box;
}

.prop-card {
  background: var(--brand-surface-1);
  border: 1px solid var(--brand-border);
  border-radius: 16px;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,.02);
}

.prop-card-header h3 {
  margin: 0 0 4px;
  font-family: var(--headlinefont);
  font-size: 20px;
  color: var(--brand-text-main);
}

.prop-card-header p {
  margin: 0;
  font-size: 13px;
  color: var(--brand-text-muted);
}

.prop-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 16px;
}

@media (max-width: 850px) {
  .prop-row { grid-template-columns: 1fr; }
}

.prop-field-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.prop-field-group label {
  font-size: 12px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: .06em;
  color: var(--brand-text-muted);
}

.prop-select-field {
  width: 100%;
  padding: 12px 16px;
  font-family: var(--contentfont);
  font-size: 14px;
  font-weight: 600;
  color: var(--brand-text-main);
  background: var(--brand-surface);
  border: 2px solid var(--brand-border);
  border-radius: 10px;
  outline: none;
  cursor: pointer;
  transition: border-color .2s ease, box-shadow .2s ease;
}

.prop-select-field:focus, .prop-select-field:hover {
  border-color: var(--brand-primary);
  box-shadow: 0 2px 8px rgba(74,93,69,.12);
}

/* Custom plain text input when 'Other' is chosen */
.prop-custom-input {
  display: none;
  margin-top: 8px;
  cursor: text;
}

.pkg-layout-wrapper { padding: 0 16px; margin: 0 auto; color: var(--brand-text-main); }
.pkg-layout-wrapper *, .pkg-layout-wrapper *::before, .pkg-layout-wrapper *::after { box-sizing: border-box; }
.pkg-split-container { display: grid; grid-template-columns: 380px 1fr; gap: 24px; align-items: start; }

.pkg-list-column { 
  display: flex; 
  flex-direction: column; 
  gap: 16px; 
}

.pkg-list-heading { 
  font-size: 11px; 
  font-weight: 700; 
  letter-spacing: .08em; 
  text-transform: uppercase; 
  color: var(--brand-text-muted); 
  margin: 12px 2px 4px; 
}
.pkg-list-heading:first-child {
  margin-top: 0;
}

.pkg-nav-card {
  background: var(--brand-surface); 
  border: 2px solid var(--brand-border); 
  border-radius: 12px;
  padding: 16px 20px; 
  cursor: pointer; 
  transition: all .25s ease;
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  gap: 12px;
  font: inherit; 
  color: inherit; 
  text-align: left; 
  width: 100%;
  margin-bottom: 4px;
  position: relative;
}
.pkg-nav-card:hover { border-color: var(--brand-primary); transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0,0,0,.05); }
.pkg-nav-card.active { border-color: var(--brand-primary); background: var(--brand-surface-1); box-shadow: 0 4px 14px rgba(74,93,69,.12); }
.pkg-nav-info h4 { margin: 4px 0 0; font-family: var(--headlinefont); font-size: 16px; font-weight: 600; color: var(--brand-text-main); }
.pkg-nav-info small { font-size: 11px; color: var(--brand-accent); font-weight: 700; display: block; margin-top: 2px; }
.pkg-nav-from { text-align: right; font-size: 11px; color: var(--brand-text-muted); line-height: 1.5; white-space: nowrap; }
.pkg-nav-from b { font-family: var(--headlinefont); font-size: 14px; color: var(--brand-primary); }

.pkg-recommended-badge {
  position: absolute;
  top: -10px;
  right: 14px;
  background: var(--brand-accent);
  color: #fff;
  font-size: 9px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: .08em;
  padding: 3px 8px;
  border-radius: 10px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.15);
}

.pkg-tag { font-size: 10px; font-weight: 700; text-transform: uppercase; letter-spacing: .08em; padding: 2px 8px; border-radius: 12px; display: inline-block; }
.tag-modern { background: rgba(74,93,69,.12); color: var(--brand-primary); }
.tag-industrial { background: rgba(193,140,93,.15); color: #9a6a3f; }
.tag-alacarte { background: rgba(108,117,109,.15); color: var(--brand-text-muted); }

.pkg-detail-pane { background: var(--brand-surface); border: 1px solid var(--brand-border); border-radius: 16px; overflow: hidden; box-shadow: 0 6px 20px rgba(0,0,0,.04); position: sticky; top: 20px; }
.pkg-hero-image-wrap { width: 100%; height: 240px; position: relative; overflow: hidden; background: var(--brand-surface-2); cursor: pointer; }
.pkg-hero-image-wrap img { width: 100%; height: 100%; object-fit: cover; transition: opacity .3s ease, transform .3s ease; }
.pkg-hero-image-wrap:hover img { transform: scale(1.02); }
.pkg-hero-badge { position: absolute; bottom: 12px; right: 12px; background: rgba(0,0,0,.65); backdrop-filter: blur(4px); color: #fff; font-size: 11px; padding: 4px 10px; border-radius: 20px; pointer-events: none; }
.pkg-detail-body { padding: 24px; }

.pkg-modal-backdrop {
  position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
  background: rgba(0, 0, 0, 0.85); backdrop-filter: blur(6px); z-index: 99999;
  display: flex; align-items: center; justify-content: center; opacity: 0; pointer-events: none;
  transition: opacity 0.3s ease; padding: 24px; box-sizing: border-box;
}
.pkg-modal-backdrop.open { opacity: 1; pointer-events: auto; }
.pkg-modal-content { position: relative; max-width: 90vw; max-height: 90vh; display: flex; flex-direction: column; align-items: center; }
.pkg-modal-content img { max-width: 100%; max-height: 80vh; object-fit: contain; border-radius: 8px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
.pkg-modal-caption { color: #fff; font-family: var(--contentfont); font-size: 14px; margin-top: 12px; text-align: center; }
.pkg-modal-close { position: absolute; top: -45px; right: 0; background: none; border: none; color: #fff; font-size: 32px; cursor: pointer; padding: 4px 10px; line-height: 1; }
.pkg-modal-close:hover { color: var(--brand-accent); }

.pkg-theme-switch { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 20px; }
.pkg-theme-switch[hidden] { display: none; }
.pkg-theme-btn { font: inherit; text-align: left; background: var(--brand-surface); border: 2px solid var(--brand-border); border-radius: 10px; padding: 10px 14px; cursor: pointer; transition: all .2s ease; color: var(--brand-text-main); }
.pkg-theme-btn small { display: block; font-size: 11px; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--brand-text-muted); }
.pkg-theme-btn strong { display: block; font-family: var(--headlinefont); font-size: 16px; margin-top: 2px; }
.pkg-theme-btn span { font-size: 13px; color: var(--brand-primary); font-weight: 600; }
.pkg-theme-btn:hover { border-color: var(--brand-primary); }
.pkg-theme-btn.active { border-color: var(--brand-primary); background: var(--brand-surface-1); box-shadow: 0 3px 10px rgba(74,93,69,.14); }

.pkg-detail-header { display: flex; justify-content: space-between; align-items: flex-start; gap: 12px; border-bottom: 1px solid var(--brand-border); padding-bottom: 16px; margin-bottom: 20px; }
.pkg-detail-title { font-family: var(--headlinefont); font-size: 24px; margin: 4px 0 0; }
.pkg-detail-price-tag { font-family: var(--headlinefont); font-size: 26px; font-weight: 700; color: var(--brand-primary); text-align: right; white-space: nowrap; }

.pkg-room-sec { margin-bottom: 20px; }
.pkg-room-sec h5 { font-size: 13px; font-weight: 700; text-transform: uppercase; letter-spacing: .05em; color: var(--brand-primary); margin: 0 0 10px; border-bottom: 1px solid var(--brand-border); padding-bottom: 4px; }
.pkg-item-list { list-style: none; padding: 0; margin: 0; display: grid; grid-template-columns: 1fr 1fr; gap: 8px 16px; }
.pkg-item-list li { font-size: 13px; display: flex; align-items: flex-start; gap: 6px; }
.pkg-item-list li::before { content: "•"; color: var(--brand-primary); font-weight: bold; }

@media (max-width: 850px) {
  .pkg-split-container { grid-template-columns: 1fr; }
  .pkg-detail-pane { position: relative; top: 0; }
  .pkg-item-list { grid-template-columns: 1fr; }
  .pkg-hero-image-wrap { height: 200px; }
  .pkg-detail-header { flex-direction: column; }
  .pkg-detail-price-tag { text-align: left; }
}

.pkg-status { max-width: 1200px; margin: 0 auto; padding: 24px 16px; text-align: center; font-family: var(--contentfont); color: var(--brand-text-muted); font-size: 14px; }
.pkg-status[hidden] { display: none; }
.pkg-status .pkg-spin { display: inline-block; width: 14px; height: 14px; border: 2px solid var(--brand-border); border-top-color: var(--brand-primary); border-radius: 50%; animation: pkgspin .8s linear infinite; vertical-align: -2px; margin-right: 8px; }
@keyframes pkgspin { to { transform: rotate(360deg); } }
.pkg-item-list li small { color: var(--brand-text-muted); font-size: 11px; }
</style>
