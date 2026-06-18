// ==UserScript==
// @name         SOS POS Walk-in Sales Loader
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Paste rows from your sheet and bulk-build Walk-in sales in SOS POS — grouped by ticket #, one line item per row. Stops at Checkout so you take payment manually.
// @author       Claude
// @match        https://app.sospos.com.au/*
// @grant        GM_setValue
// @grant        GM_getValue
// ==/UserScript==

(function () {
  'use strict';

  // ─────────────────────────────────────────────────────────────
  // COLUMN MAP (0-based index into a tab-separated pasted row)
  //
  // Your paste looks like:
  // 17.06.26 ⇥ ⇥ A2892 ⇥ Paid & Collected ⇥ 55 ⇥ ⇥ LR ⇥ LR ⇥ LR ⇥ YES ⇥ ⇥ ⇥ 📒 ⇥ PIN ⇥ Walkin 0- Cable + wal plug
  //   0        1   2       3                  4     5  6     7   8    9    10 11  12  13     14
  //
  //   A=0 date | B=1 blank | C=2 TICKET# | D=3 status
  //   E=4 CASH amount | F=5 EFTPOS/card amount
  //   G/H/I = codes (LR/DP) | J=YES | ... | description = LAST cell (after "PIN")
  // ─────────────────────────────────────────────────────────────
  const COL = {
    TICKET: 2,   // grouping key
    CASH:   4,   // cash amount column
    EFTPOS: 5,   // eftpos / card amount column
    // description is resolved dynamically (cell after "PIN", else last non-empty)
  };

  // ─────────────────────────────────────────────────────────────
  // Persistent settings
  // ─────────────────────────────────────────────────────────────
  const DEFAULTS = { stepDelay: 350, stripWalkin: false, priceMode: 'sum' };
  function loadSettings() {
    try { return Object.assign({}, DEFAULTS, JSON.parse(GM_getValue('sos_walkin_cfg', '{}'))); }
    catch { return Object.assign({}, DEFAULTS); }
  }
  function saveSettings(c) { GM_setValue('sos_walkin_cfg', JSON.stringify(c)); }
  let cfg = loadSettings();

  // ─────────────────────────────────────────────────────────────
  // Styles
  // ─────────────────────────────────────────────────────────────
  const style = document.createElement('style');
  style.textContent = `
    #sosw-fab {
      position: fixed; bottom: 20px; left: 224px;
      width: 44px; height: 44px; border-radius: 50%;
      background: #ea580c; box-shadow: 0 3px 14px rgba(234,88,12,.55);
      border: none; cursor: pointer; z-index: 99999;
      display: flex; align-items: center; justify-content: center;
      font-size: 20px; transition: background .15s; user-select: none;
    }
    #sosw-fab:hover { background: #c2410c; }

    #sosw-panel {
      position: fixed; bottom: 72px; left: 20px; width: 400px;
      background: #0f172a; color: #e2e8f0; border-radius: 16px;
      box-shadow: 0 20px 60px rgba(0,0,0,.7);
      font-family: 'Segoe UI',system-ui,sans-serif; font-size: 13px;
      z-index: 99998; border: 1px solid #1e293b; display: none; overflow: hidden;
    }
    #sosw-panel.open { display: block; }

    #sosw-header {
      background: linear-gradient(135deg,#f97316 0%,#ea580c 100%);
      padding: 14px 16px; font-weight: 700; font-size: 15px;
      display: flex; align-items: center; gap: 8px; letter-spacing: .3px;
    }
    #sosw-header .sosw-title { flex: 1; }
    #sosw-close-btn {
      background: rgba(255,255,255,.2); border: none; color: #fff;
      width: 26px; height: 26px; border-radius: 50%; cursor: pointer;
      font-size: 16px; line-height: 1; display: flex; align-items: center; justify-content: center;
    }
    #sosw-close-btn:hover { background: rgba(255,255,255,.35); }

    #sosw-tabs { display: flex; background: #0a1120; border-bottom: 1px solid #1e293b; }
    .sosw-tab {
      flex: 1; padding: 9px 0; text-align: center; font-size: 12px; font-weight: 600;
      cursor: pointer; color: #64748b; border-bottom: 2px solid transparent; user-select: none;
    }
    .sosw-tab.active { color: #f97316; border-bottom-color: #f97316; }

    .sosw-pane { display: none; padding: 14px; }
    .sosw-pane.active { display: block; }

    .sosw-field { margin-bottom: 10px; }
    .sosw-label {
      display: block; font-size: 11px; font-weight: 600; color: #64748b;
      margin-bottom: 4px; text-transform: uppercase; letter-spacing: .5px;
    }
    .sosw-input, .sosw-select {
      width: 100%; box-sizing: border-box; background: #1e293b; border: 1px solid #334155;
      color: #e2e8f0; border-radius: 8px; padding: 7px 10px; font-size: 13px; outline: none;
    }
    .sosw-input:focus, .sosw-select:focus { border-color: #f97316; }
    .sosw-select option { background: #1e293b; }

    .sosw-btn {
      padding: 9px 14px; border-radius: 8px; border: none; cursor: pointer;
      font-weight: 600; font-size: 13px; white-space: nowrap; transition: opacity .15s, transform .1s;
    }
    .sosw-btn:hover { opacity: .88; }
    .sosw-btn:active { transform: scale(.97); }
    .sosw-btn:disabled { opacity: .4; cursor: not-allowed; }
    .sosw-btn-primary { background: linear-gradient(135deg,#f97316,#ea580c); color: #fff; }
    .sosw-btn-success { background: #16a34a; color: #fff; }
    .sosw-btn-muted   { background: #334155; color: #94a3b8; }
    .sosw-btn-sm      { padding: 5px 10px; font-size: 12px; }
    .sosw-btn-row     { display: flex; gap: 6px; margin-top: 8px; }

    #sosw-drop-zone {
      border: 2px dashed #334155; border-radius: 10px; padding: 18px 14px;
      text-align: center; cursor: pointer; margin-bottom: 10px; position: relative;
      transition: border-color .2s, background .2s;
    }
    #sosw-drop-zone:hover { border-color: #f97316; background: rgba(249,115,22,.06); }
    #sosw-drop-zone .dz-icon { font-size: 26px; margin-bottom: 4px; }
    #sosw-drop-zone .dz-main { font-size: 13px; font-weight: 600; color: #cbd5e1; margin-bottom: 2px; }
    #sosw-drop-zone .dz-sub { font-size: 11px; color: #475569; }
    #sosw-drop-zone.has-data { border-style: solid; border-color: #16a34a; background: rgba(22,163,74,.05); padding: 10px 14px; text-align: left; cursor: default; }
    #sosw-drop-zone.has-data .dz-icon, #sosw-drop-zone.has-data .dz-main, #sosw-drop-zone.has-data .dz-sub { display: none; }
    #sosw-paste { position: absolute; opacity: 0; width: 1px; height: 1px; pointer-events: none; top: 0; left: 0; }
    #sosw-paste-summary { display: none; align-items: center; gap: 8px; font-size: 12px; color: #86efac; }
    #sosw-paste-summary .ps-count { background: #166534; color: #86efac; border-radius: 20px; padding: 2px 9px; font-weight: 700; }
    #sosw-paste-summary .ps-clear { margin-left: auto; cursor: pointer; color: #f87171; font-size: 16px; line-height: 1; padding: 2px 4px; }
    #sosw-paste-summary .ps-clear:hover { color: #ef4444; }

    #sosw-preview { max-height: 260px; overflow-y: auto; display: flex; flex-direction: column; gap: 8px; margin-top: 2px; }
    #sosw-preview::-webkit-scrollbar { width: 4px; }
    #sosw-preview::-webkit-scrollbar-thumb { background: #334155; border-radius: 2px; }

    .sosw-group {
      background: #131f2e; border: 1px solid #1e293b; border-radius: 10px; padding: 9px 11px; transition: background .2s;
    }
    .sosw-group.active { background: #1a1206; border-color: #f97316; }
    .sosw-group.done   { background: #0a150a; border-color: #166534; opacity: .65; }
    .sosw-group-head { display: flex; align-items: center; gap: 8px; margin-bottom: 6px; }
    .sosw-group-badge { background: #1e293b; color: #94a3b8; border-radius: 6px; padding: 1px 7px; font-size: 10px; font-weight: 800; }
    .sosw-group.active .sosw-group-badge { background: #7c2d12; color: #fdba74; }
    .sosw-group.done   .sosw-group-badge { background: #14532d; color: #86efac; }
    .sosw-group-ticket { font-size: 12px; font-weight: 700; color: #e2e8f0; }
    .sosw-group-total { margin-left: auto; font-size: 12px; font-weight: 700; color: #4ade80; }
    .sosw-line { display: flex; gap: 6px; font-size: 11px; color: #94a3b8; padding: 2px 0; border-top: 1px solid #1e293b; }
    .sosw-line:first-of-type { border-top: none; }
    .sosw-line .ln-desc { flex: 1; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .sosw-line .ln-price { color: #4ade80; font-weight: 600; }
    .sosw-line .ln-method { color: #64748b; font-size: 10px; background: #0f172a; border-radius: 4px; padding: 0 5px; }

    #sosw-prog-wrap { margin-top: 10px; }
    #sosw-prog-bg { height: 5px; background: #1e293b; border-radius: 3px; overflow: hidden; }
    #sosw-prog-bar { height: 100%; width: 0%; background: linear-gradient(90deg,#f97316,#ea580c); border-radius: 3px; transition: width .4s; }
    #sosw-status { margin-top: 6px; font-size: 11.5px; color: #94a3b8; min-height: 16px; text-align: center; }

    .sosw-divider { border: none; border-top: 1px solid #1e293b; margin: 12px 0; }
    .sosw-note { color: #475569; font-size: 11px; line-height: 1.6; margin: 0; }
    .sosw-note b { color: #fdba74; }
    .sosw-row2 { display: flex; gap: 8px; }
    .sosw-row2 > * { flex: 1; }
  `;
  document.head.appendChild(style);

  // ─────────────────────────────────────────────────────────────
  // FAB + panel
  // ─────────────────────────────────────────────────────────────
  const fab = document.createElement('button');
  fab.id = 'sosw-fab';
  fab.title = 'SOS POS Walk-in Sales Loader';
  fab.innerHTML = '🏷️';
  document.body.appendChild(fab);

  // Sit just to the right of the app's own bottom-left buttons.
  // The app renders them async, so retry a few times + on resize.
  [400, 1200, 2500, 4500].forEach(t => setTimeout(positionFab, t));
  window.addEventListener('resize', () => setTimeout(positionFab, 100));

  const panel = document.createElement('div');
  panel.id = 'sosw-panel';
  panel.innerHTML = `
    <div id="sosw-header">
      <span>🏷️</span>
      <span class="sosw-title">Walk-in Sales Loader</span>
      <button id="sosw-close-btn" title="Close">✕</button>
    </div>

    <div id="sosw-tabs">
      <div class="sosw-tab active" data-tab="build">🛒 Build</div>
      <div class="sosw-tab" data-tab="settings">⚙ Settings</div>
    </div>

    <!-- BUILD TAB -->
    <div class="sosw-pane active" id="tab-build">
      <div id="sosw-drop-zone" tabindex="0" title="Click then Ctrl+V to paste">
        <textarea id="sosw-paste" tabindex="-1" aria-hidden="true"></textarea>
        <div class="dz-icon">📋</div>
        <div class="dz-main">Click here, then paste your rows</div>
        <div class="dz-sub">Copy the rows from your sheet → click this box → Ctrl+V</div>
        <div id="sosw-paste-summary">
          <span class="ps-count" id="sosw-count-badge">0</span>
          <span id="sosw-count-label">tickets ready</span>
          <span class="ps-clear" id="sosw-dz-clear" title="Clear">✕</span>
        </div>
      </div>

      <div class="sosw-btn-row">
        <button class="sosw-btn sosw-btn-success" id="sosw-build-btn" style="display:none;flex:1">▶ Build Ticket 1</button>
        <button class="sosw-btn sosw-btn-muted sosw-btn-sm" id="sosw-clear-btn" style="display:none">Clear</button>
      </div>

      <div id="sosw-preview"></div>

      <div id="sosw-prog-wrap">
        <div id="sosw-prog-bg"><div id="sosw-prog-bar"></div></div>
        <div id="sosw-status"></div>
      </div>
    </div>

    <!-- SETTINGS TAB -->
    <div class="sosw-pane" id="tab-settings">
      <div class="sosw-field">
        <label class="sosw-label">Line item price</label>
        <select class="sosw-select" id="sosw-price-mode">
          <option value="sum">Cash + EFTPOS columns added together</option>
          <option value="eftpos">EFTPOS column only</option>
          <option value="cash">Cash column only</option>
        </select>
      </div>
      <div class="sosw-row2">
        <div class="sosw-field">
          <label class="sosw-label">Step delay (ms)</label>
          <input class="sosw-input" id="sosw-step-delay" type="number" min="100" step="50" />
        </div>
        <div class="sosw-field">
          <label class="sosw-label">Strip "Walkin" prefix</label>
          <select class="sosw-select" id="sosw-strip">
            <option value="no">No — keep text as-is</option>
            <option value="yes">Yes — remove leading Walkin/Walk-in</option>
          </select>
        </div>
      </div>
      <button class="sosw-btn sosw-btn-primary sosw-btn-sm" id="sosw-save-cfg">Save settings</button>
      <hr class="sosw-divider">
      <p class="sosw-note">
        <b>How it works</b><br>
        1. Paste rows, they group by <b>ticket #</b> (col C).<br>
        2. Click <b>Build</b> — it switches to the <b>Sale</b> tab, hits <b>Walk-in</b>, and adds each row as a line item with its price.<br>
        3. It <b>stops at Checkout</b> — you take payment.<br>
        4. After payment, click <b>Next Ticket →</b> for the following group.<br><br>
        <b>Column map</b>: C = ticket# · E = cash · F = eftpos · last cell = description.
        Wrong columns? Edit the <code>COL</code> map at the top of the script.
      </p>
    </div>
  `;
  document.body.appendChild(panel);

  fab.addEventListener('click', () => panel.classList.toggle('open'));
  document.getElementById('sosw-close-btn').addEventListener('click', () => panel.classList.remove('open'));

  document.querySelectorAll('.sosw-tab').forEach(tab => {
    tab.addEventListener('click', () => {
      document.querySelectorAll('.sosw-tab').forEach(t => t.classList.remove('active'));
      document.querySelectorAll('.sosw-pane').forEach(p => p.classList.remove('active'));
      tab.classList.add('active');
      document.getElementById('tab-' + tab.dataset.tab).classList.add('active');
    });
  });

  // Settings wiring
  const $priceMode = document.getElementById('sosw-price-mode');
  const $stepDelay = document.getElementById('sosw-step-delay');
  const $strip     = document.getElementById('sosw-strip');
  $priceMode.value = cfg.priceMode;
  $stepDelay.value = cfg.stepDelay;
  $strip.value     = cfg.stripWalkin ? 'yes' : 'no';
  document.getElementById('sosw-save-cfg').addEventListener('click', () => {
    cfg.priceMode  = $priceMode.value;
    cfg.stepDelay  = Math.max(100, parseInt($stepDelay.value, 10) || DEFAULTS.stepDelay);
    cfg.stripWalkin = $strip.value === 'yes';
    saveSettings(cfg);
    setStatus('✓ Settings saved.');
    if (rawCache) doParse(rawCache); // re-parse with new price rule
  });

  // ─────────────────────────────────────────────────────────────
  // Parse
  // ─────────────────────────────────────────────────────────────
  let groups = [];        // [{ ticket, items:[{desc, price, cash, eftpos, method}], total, status }]
  let currentIdx = 0;
  let rawCache = '';

  const dropZone  = document.getElementById('sosw-drop-zone');
  const pasteArea = document.getElementById('sosw-paste');

  dropZone.addEventListener('click', (e) => {
    if (e.target.id === 'sosw-dz-clear') return;
    if (!dropZone.classList.contains('has-data')) pasteArea.focus();
  });
  pasteArea.addEventListener('paste', (e) => {
    e.preventDefault();
    const raw = (e.clipboardData || window.clipboardData).getData('text');
    pasteArea.value = raw;
    setTimeout(() => doParse(raw), 40);
  });
  document.getElementById('sosw-dz-clear').addEventListener('click', () => clearAll());

  function num(v) { const n = parseFloat(String(v || '').replace(/[^0-9.]/g, '')); return isNaN(n) ? 0 : n; }

  function extractDescription(cols) {
    const pinIdx = cols.findIndex(c => c.trim().toUpperCase() === 'PIN');
    if (pinIdx >= 0 && cols[pinIdx + 1] && cols[pinIdx + 1].trim()) return cols[pinIdx + 1].trim();
    for (let i = cols.length - 1; i >= 0; i--) { if (cols[i] && cols[i].trim()) return cols[i].trim(); }
    return '';
  }

  function cleanDesc(d) {
    if (cfg.stripWalkin) d = d.replace(/^\s*walk[\s-]?in\s*[-–:]?\s*/i, '').trim();
    return d || '(item)';
  }

  function priceFor(cash, eftpos) {
    if (cfg.priceMode === 'cash')   return cash;
    if (cfg.priceMode === 'eftpos') return eftpos;
    return cash + eftpos;
  }

  function methodLabel(cash, eftpos) {
    if (cash > 0 && eftpos > 0) return `Split $${cash}c/$${eftpos}e`;
    if (cash > 0)   return 'Cash';
    if (eftpos > 0) return 'EFTPOS';
    return '—';
  }

  const SKIP = /^(date|ticket|status|description|no\.?)$/i;

  function doParse(raw) {
    rawCache = raw;
    if (!raw) { setStatus('⚠️ Nothing pasted yet.'); return; }

    const map = new Map(); // ticket -> group (preserves first-seen order)
    const lines = raw.split('\n');

    for (const line of lines) {
      if (!line.trim()) continue;
      const cols = line.split('\t');

      const desc = extractDescription(cols);
      if (!desc || SKIP.test(desc)) continue;

      const cash   = num(cols[COL.CASH]);
      const eftpos = num(cols[COL.EFTPOS]);
      const price  = priceFor(cash, eftpos);

      // skip obvious header/blank rows: no description-worthy text AND no money
      if (price === 0 && !/[a-z0-9]/i.test(desc)) continue;

      let ticket = (cols[COL.TICKET] || '').trim();
      if (!ticket) ticket = 'No ticket #';

      if (!map.has(ticket)) map.set(ticket, { ticket, items: [], total: 0, status: 'pending' });
      const g = map.get(ticket);
      g.items.push({ desc: cleanDesc(desc), price, cash, eftpos, method: methodLabel(cash, eftpos) });
      g.total += price;
    }

    groups = Array.from(map.values());
    currentIdx = 0;
    renderPreview();

    const totalRows = groups.reduce((s, g) => s + g.items.length, 0);
    if (groups.length > 0) {
      dropZone.classList.add('has-data');
      const summary = document.getElementById('sosw-paste-summary');
      summary.style.display = 'flex';
      document.getElementById('sosw-count-badge').textContent = groups.length;
      document.getElementById('sosw-count-label').textContent =
        `ticket${groups.length !== 1 ? 's' : ''} · ${totalRows} item${totalRows !== 1 ? 's' : ''}`;
      const buildBtn = document.getElementById('sosw-build-btn');
      buildBtn.style.display = 'block';
      buildBtn.disabled = false;
      buildBtn.textContent = `▶ Build Ticket 1 (${groups[0].items.length} item${groups[0].items.length !== 1 ? 's' : ''})`;
      document.getElementById('sosw-clear-btn').style.display = 'block';
      setStatus('');
    } else {
      dropZone.classList.remove('has-data');
      setStatus('⚠️ No valid rows found — check the column map in Settings.');
    }
  }

  function renderPreview() {
    document.getElementById('sosw-preview').innerHTML = groups.map((g, gi) => `
      <div class="sosw-group ${g.status}" id="sosw-group-${gi}">
        <div class="sosw-group-head">
          <span class="sosw-group-badge">${gi + 1}</span>
          <span class="sosw-group-ticket">${g.ticket}</span>
          <span class="sosw-group-total">$${g.total.toFixed(2)}</span>
        </div>
        ${g.items.map(it => `
          <div class="sosw-line">
            <span class="ln-desc">${esc(it.desc)}</span>
            <span class="ln-method">${it.method}</span>
            <span class="ln-price">$${it.price.toFixed(2)}</span>
          </div>
        `).join('')}
      </div>
    `).join('');
  }

  function esc(s) { return String(s).replace(/[&<>]/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;' }[c])); }

  function clearAll() {
    groups = []; currentIdx = 0; rawCache = '';
    pasteArea.value = '';
    document.getElementById('sosw-preview').innerHTML = '';
    document.getElementById('sosw-build-btn').style.display = 'none';
    document.getElementById('sosw-clear-btn').style.display = 'none';
    document.getElementById('sosw-prog-bar').style.width = '0%';
    dropZone.classList.remove('has-data');
    document.getElementById('sosw-paste-summary').style.display = 'none';
    setStatus('');
  }

  function setStatus(msg) { document.getElementById('sosw-status').textContent = msg; }
  function setGroupStatus(i, s) {
    groups[i].status = s;
    const el = document.getElementById(`sosw-group-${i}`);
    if (el) el.className = `sosw-group ${s}`;
  }
  function setProgress() {
    const done = groups.filter(g => g.status === 'done').length;
    document.getElementById('sosw-prog-bar').style.width = groups.length ? `${Math.round(done / groups.length * 100)}%` : '0%';
  }

  // ─────────────────────────────────────────────────────────────
  // Build (one ticket per click)
  // ─────────────────────────────────────────────────────────────
  const buildBtn = document.getElementById('sosw-build-btn');
  buildBtn.addEventListener('click', onBuildClick);
  document.getElementById('sosw-clear-btn').addEventListener('click', clearAll);

  async function onBuildClick() {
    if (currentIdx >= groups.length) return;
    buildBtn.disabled = true;
    const i = currentIdx;
    setGroupStatus(i, 'active');
    setStatus(`Building ${groups[i].ticket}…`);

    try {
      await buildTicket(groups[i]);
      setGroupStatus(i, 'done');
      setProgress();
      currentIdx++;

      if (currentIdx < groups.length) {
        const n = groups[currentIdx].items.length;
        buildBtn.textContent = `Next Ticket → ${groups[currentIdx].ticket} (${n} item${n !== 1 ? 's' : ''})`;
        buildBtn.disabled = false;
        setStatus(`✓ ${groups[i].ticket} ready — review & Checkout, then click Next.`);
      } else {
        buildBtn.textContent = '✓ All tickets built';
        buildBtn.disabled = true;
        setStatus(`🎉 Done — ${groups.length} ticket${groups.length !== 1 ? 's' : ''} built. Remember to Checkout the last one.`);
      }
    } catch (e) {
      setGroupStatus(i, 'pending');
      buildBtn.disabled = false;
      setStatus('✕ ' + e.message);
      console.error('[SOS Walk-in]', e);
    }
  }

  async function buildTicket(group) {
    // 1. Make sure the Sale tab is active
    const saleTab = findTab('Sale');
    if (saleTab) { saleTab.click(); await sleep(cfg.stepDelay); }

    // 2. Click the Walk-in (no customer details) button
    const walkBtn = findWalkInButton();
    if (!walkBtn) throw new Error('Walk-in button not found');
    walkBtn.click();
    await sleep(cfg.stepDelay + 150);

    // 3. Add each item as a line item
    for (let k = 0; k < group.items.length; k++) {
      const it = group.items[k];
      if (k > 0) {
        const addBtn = findAddItemButton();
        if (!addBtn) throw new Error('"Add another item" button not found');
        addBtn.click();
        await sleep(cfg.stepDelay);
      }
      const descInputs  = getLineInputs('Item description');
      const priceInputs = getLineInputs('0.00');
      const dEl = descInputs[k], pEl = priceInputs[k];
      if (!dEl || !pEl) throw new Error(`Line item row ${k + 1} fields not found`);
      setNativeValue(dEl, it.desc);
      await sleep(80);
      setNativeValue(pEl, String(it.price));
      await sleep(cfg.stepDelay);
    }
    // Intentionally STOP here — user reviews total and clicks Checkout.
  }

  // ─────────────────────────────────────────────────────────────
  // DOM helpers
  // ─────────────────────────────────────────────────────────────
  function findTab(label) {
    return Array.from(document.querySelectorAll('[role="tab"]'))
      .find(t => t.textContent.trim().toLowerCase().includes(label.toLowerCase()));
  }

  function findWalkInButton() {
    // Most reliable: the title attribute seen in the SOS POS markup
    let b = document.querySelector('button[title*="Walk-in" i]');
    if (b) return b;
    // Fallback: button text
    return Array.from(document.querySelectorAll('button'))
      .find(x => /walk[\s-]?in/i.test(x.getAttribute('title') || x.textContent || ''));
  }

  function findAddItemButton() {
    return Array.from(document.querySelectorAll('button'))
      .find(b => /add another item/i.test(b.textContent.trim()));
  }

  function getLineInputs(placeholder) {
    // Exact-placeholder match keeps us off the customer/product search boxes
    return Array.from(document.querySelectorAll(`input[placeholder="${placeholder}"]`));
  }

  function setNativeValue(el, value) {
    const proto = el.tagName === 'TEXTAREA' ? window.HTMLTextAreaElement.prototype : window.HTMLInputElement.prototype;
    const setter = Object.getOwnPropertyDescriptor(proto, 'value')?.set;
    if (setter) setter.call(el, value); else el.value = value;
    el.dispatchEvent(new Event('input',  { bubbles: true }));
    el.dispatchEvent(new Event('change', { bubbles: true }));
  }

  function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

  // Find the app's existing round bottom-left buttons and dock just right of the last one.
  function positionFab() {
    let bestRect = null, bestRight = -1;
    document.querySelectorAll('button, a, div, [role="button"]').forEach(el => {
      if (el.id && el.id.startsWith('sosw')) return;          // ignore our own elements
      if (el === fab || el === panel) return;
      const r = el.getBoundingClientRect();
      // round, FAB-sized
      if (r.width < 32 || r.width > 60) return;
      if (Math.abs(r.width - r.height) > 12) return;
      // anchored to the bottom edge of the viewport
      if (r.bottom < window.innerHeight - 110 || r.bottom > window.innerHeight - 3) return;
      // on the left side
      if (r.left < 2 || r.left > 300) return;
      if (r.right > bestRight) { bestRight = r.right; bestRect = r; }
    });

    if (bestRect && bestRight <= 360) {
      fab.style.left   = Math.round(bestRight + 12) + 'px';
      fab.style.bottom = Math.round(window.innerHeight - bestRect.bottom) + 'px';
    } else {
      // fallback if the app buttons aren't found yet
      fab.style.left   = '224px';
      fab.style.bottom = '20px';
    }
  }

})();
