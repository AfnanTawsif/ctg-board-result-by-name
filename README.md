Install Tampermonkey browser extension first. From that extension's UI, add the following script and visit any dataset page (SSC/HSC, YEAR) on https://boardexamrankings.vercel.app/.

You'll see a search icon in the bottom-right corner. Click it to open the script's interface.

```javascript
// ==UserScript==
// @name         boardexamrankings — Name Search Manager v2.2
// @namespace    https://boardexamrankings.vercel.app/
// @version      2.2.0
// @description  Fast name→roll indexing with stable ETA and tolerant name search
// @match        https://boardexamrankings.vercel.app/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
  'use strict';
  if (window.__nsmLoaded) return;
  window.__nsmLoaded = true;

  // ============ constants ============
  const GROUP_SHORT = { science: 'sci', business_studies: 'bs', humanities: 'hum' };
  const GROUP_LABEL = { science: 'Science', business_studies: 'Commerce', humanities: 'Arts' };
  const CONCURRENCY   = 24;
  const SAVE_EVERY    = 40;
  const VERSION       = 2;
  const SEARCH_LIMIT  = 100;
  const REPORTER_MS   = 500;
  const WARMUP_PAGES  = 5;

  // ============ helpers ============
  const cacheKey = (exam, year, group) => `${exam}${year}_${GROUP_SHORT[group]}_index_v${VERSION}`;
  const datasetUrl   = d => `${location.origin}/${d.exam}/${d.year}/${d.group}/`;
  const datasetLabel = d => `${d.exam.toUpperCase()} ${d.year} · ${GROUP_LABEL[d.group]}`;
  const fmtBytes = b => b < 1024 ? `${b} B`
                        : b < 1048576 ? `${(b / 1024).toFixed(1)} KB`
                        : `${(b / 1048576).toFixed(2)} MB`;
  const fmtNum = n => n.toLocaleString('en-US');

  // Coarse-rounded ETA — no flickering last digit.
  const fmtEta = s => {
    if (!isFinite(s) || s < 0) return '';
    if (s < 10)  return `${Math.ceil(s)}s`;
    if (s < 60)  return `${Math.round(s / 5) * 5}s`;
    if (s < 600) {
      const m = Math.floor(s / 60);
      let   r = Math.round((s % 60) / 10) * 10;
      if (r === 60) return `${m + 1}m`;
      return r === 0 ? `${m}m` : `${m}m ${r}s`;
    }
    const m = Math.round(s / 60);
    if (m < 60) return `${m}m`;
    const h = Math.floor(m / 60), mr = m % 60;
    return mr === 0 ? `${h}h` : `${h}h ${mr}m`;
  };

  const escapeHtml = s => String(s).replace(/[&<>"']/g, c =>
    ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const decodeEntities = s => s
    .replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>')
    .replace(/&quot;/g, '"').replace(/&#39;/g, "'").replace(/&nbsp;/g, ' ');

  function parseDatasetFromPath(path) {
    const m = path.match(/^\/(hsc|ssc)\/(\d{4})\/(science|business_studies|humanities)\//);
    return m ? { exam: m[1], year: m[2], group: m[3] } : null;
  }

  function readIndex(key) {
    const raw = localStorage.getItem(key);
    if (!raw) return null;
    try { return JSON.parse(raw); } catch { return null; }
  }

  function listCached() {
    const out = [];
    for (const key of Object.keys(localStorage)) {
      const m = key.match(/^(hsc|ssc)(\d{4})_(sci|bs|hum)_index_v(\d)$/);
      if (!m) continue;
      const [, exam, year, short, ver] = m;
      const group = Object.keys(GROUP_SHORT).find(g => GROUP_SHORT[g] === short);
      if (!group) continue;
      const raw = localStorage.getItem(key);
      let count = 0;
      try { count = Object.keys(JSON.parse(raw)).length; } catch {}
      out.push({ exam, year, group, key, count, bytes: new Blob([raw]).size, version: parseInt(ver) });
    }
    return out.sort((a, b) =>
      a.exam.localeCompare(b.exam) ||
      a.year.localeCompare(b.year) ||
      a.group.localeCompare(b.group));
  }

  // ============ search primitives ============
  function normalize(s) {
    return s.toUpperCase()
      .replace(/[^A-Z0-9\s]/g, ' ')
      .replace(/\s+/g, ' ')
      .trim();
  }

  function levenshtein(a, b, maxDist = 99) {
    if (a === b) return 0;
    if (Math.abs(a.length - b.length) > maxDist) return maxDist + 1;
    const m = a.length, n = b.length;
    if (m === 0) return n;
    if (n === 0) return m;
    let prev = new Array(n + 1), curr = new Array(n + 1);
    for (let j = 0; j <= n; j++) prev[j] = j;
    for (let i = 1; i <= m; i++) {
      curr[0] = i;
      let rowMin = i;
      for (let j = 1; j <= n; j++) {
        const cost = a[i - 1] === b[j - 1] ? 0 : 1;
        curr[j] = Math.min(prev[j] + 1, curr[j - 1] + 1, prev[j - 1] + cost);
        if (curr[j] < rowMin) rowMin = curr[j];
      }
      if (rowMin > maxDist) return maxDist + 1;
      [prev, curr] = [curr, prev];
    }
    return prev[n];
  }

  // ============ in-memory caches ============
  const indexCache     = new Map();  // key → raw parsed index object
  const tokenizedCache = new Map();  // key → [{ name, val, normalized, tokens }]

  function getIndex(key) {
    if (indexCache.has(key)) return indexCache.get(key);
    const idx = readIndex(key);
    if (idx) indexCache.set(key, idx);
    return idx;
  }

  function getTokenized(key) {
    if (tokenizedCache.has(key)) return tokenizedCache.get(key);
    const idx = getIndex(key);
    if (!idx) return null;
    const list = [];
    for (const [name, val] of Object.entries(idx)) {
      const normalized = normalize(name);
      const tokens = normalized.split(' ').filter(Boolean);
      list.push({ name, val, normalized, tokens });
    }
    tokenizedCache.set(key, list);
    return list;
  }

  function invalidateCaches(key) {
    indexCache.delete(key);
    tokenizedCache.delete(key);
  }

  // ============ scoring ============
  function scoreEntry(entry, qNorm, qTokens) {
    // fast path — query appears as a contiguous substring
    const idx = entry.normalized.indexOf(qNorm);
    if (idx >= 0) {
      if (entry.normalized === qNorm) return 2000;
      if (idx === 0 && (qNorm.length === entry.normalized.length ||
                        entry.normalized[qNorm.length] === ' '))
        return 800;
      if (idx > 0 && entry.normalized[idx - 1] === ' ') return 600;
      return 400;
    }
    // token match — every query token must match some name token
    let total = 0;
    for (const qt of qTokens) {
      let best = 0;
      for (const nt of entry.tokens) {
        let s = 0;
        if (nt === qt) s = 100;
        else if (nt.startsWith(qt)) s = 70;
        else if (qt.length === 1 && nt.length > 0 && nt[0] === qt) s = 40;
        else if (qt.length >= 4 && Math.abs(qt.length - nt.length) <= 2) {
          const maxD = qt.length >= 6 ? 2 : 1;
          const d = levenshtein(qt, nt, maxD);
          if (d === 1) s = 50;
          else if (d === 2) s = 25;
        }
        if (s > best) best = s;
        if (best === 100) break;
      }
      if (best === 0) return 0;
      total += best;
    }
    return total;
  }

  function searchAllCaches(q) {
    const qNorm = normalize(q);
    if (!qNorm) return [];
    const qTokens = qNorm.split(' ').filter(Boolean);
    if (!qTokens.length) return [];

    const results = [];
    for (const c of listCached()) {
      const entries = getTokenized(c.key);
      if (!entries) continue;
      for (const entry of entries) {
        const score = scoreEntry(entry, qNorm, qTokens);
        if (score > 0) results.push({ entry, dataset: c, score });
      }
    }
    results.sort((a, b) =>
      b.score - a.score || a.entry.name.localeCompare(b.entry.name));
    return results;
  }

  // ============ scraper ============
  const ROW_RE   = /<tr\s+itemscope[^>]*>([\s\S]*?)<\/tr>/g;
  const ROLL_RE  = /itemprop="identifier">(\d+)</;
  const NAME_RE  = /itemprop="name"[^>]*>([^<]+)</;
  const RANK_RE  = /<td class="rank-cell tabular">(\d+)<\/td>/;
  const MARKS_RE = /class="numeric marks-cell tabular">(\d+)</;
  const GPA_RE   = /gpa-cell tabular"><span itemprop="award">([\d.]+)</;
  const INST_RE  = /itemprop="affiliation">([^<]+)</;

  function parsePage(html, index, rich) {
    ROW_RE.lastIndex = 0;
    let m;
    while ((m = ROW_RE.exec(html))) {
      const row = m[1];
      const roll = row.match(ROLL_RE)?.[1];
      const name = row.match(NAME_RE)?.[1];
      if (!roll || !name) continue;
      const key = name.toUpperCase();
      if (rich) {
        index[key] = {
          roll,
          rank:  row.match(RANK_RE)?.[1]  || '',
          marks: row.match(MARKS_RE)?.[1] || '',
          gpa:   row.match(GPA_RE)?.[1]   || '',
          inst:  decodeEntities(row.match(INST_RE)?.[1] || ''),
        };
      } else {
        index[key] = roll;
      }
    }
  }

  async function buildIndex(d, onProgress, signal, opts = {}) {
    const { rich = true, concurrency = CONCURRENCY } = opts;
    const BASE = datasetUrl(d);
    const key  = cacheKey(d.exam, d.year, d.group);

    onProgress({ phase: 'detect', message: 'Detecting page count…' });

    // Single reference time for the whole run — cumulative math stays smooth.
    const t0 = Date.now();
    const r0 = await fetch(BASE, { cache: 'no-store', signal });
    const html0 = await r0.text();
    const mm = html0.match(/Page\s+1\s+of\s+([\d,]+)/);
    if (!mm) throw new Error('Could not detect page count. Site layout may have changed.');
    const TOTAL_PAGES = parseInt(mm[1].replace(/,/g, ''), 10);

    const index = {};
    parsePage(html0, index, rich);
    let done     = 1;
    let failures = 0;
    let lastSave = 0;

    // Cumulative-progress emitter: rate = done / (now - t0). Jitter shrinks
    // as 1/elapsed — this is what "stable ETA" means mathematically.
    const emit = () => {
      const elapsed   = Math.max((Date.now() - t0) / 1000, 0.001);
      const rate      = done / elapsed;
      const warmingUp = done < WARMUP_PAGES;
      const eta       = warmingUp || rate <= 0
                          ? 0
                          : (TOTAL_PAGES - done) / rate;
      onProgress({
        phase: 'scrape',
        current: done,
        total:   TOTAL_PAGES,
        count:   Object.keys(index).length,
        rate,
        eta,
        elapsed,
        warmingUp,
        failures,
      });
    };

    emit();
    // Time-based reporter — one UI tick every 500 ms, regardless of bursts.
    const reporter = setInterval(emit, REPORTER_MS);

    try {
      const remaining = [];
      for (let p = 2; p <= TOTAL_PAGES; p++) remaining.push(p);
      let cursor = 0;

      async function fetchOne(p) {
        for (let a = 0; a < 3; a++) {
          if (signal.aborted) throw new DOMException('Cancelled', 'AbortError');
          try {
            const r = await fetch(`${BASE}?page=${p}`, { cache: 'no-store', signal });
            if (!r.ok) throw new Error(String(r.status));
            return await r.text();
          } catch (e) {
            if (signal.aborted) throw e;
            if (a === 2) throw e;
            await new Promise(res => setTimeout(res, 500 * (a + 1)));
          }
        }
      }

      const runners = Array.from({ length: concurrency }, async () => {
        while (cursor < remaining.length) {
          if (signal.aborted) throw new DOMException('Cancelled', 'AbortError');
          const p = remaining[cursor++];
          try {
            const html = await fetchOne(p);
            parsePage(html, index, rich);
          } catch (e) {
            if (signal.aborted) throw e;
            failures++;
          }
          done++;
          if (done - lastSave >= SAVE_EVERY) {
            lastSave = done;
            try { localStorage.setItem(key, JSON.stringify(index)); } catch {}
          }
        }
      });

      await Promise.all(runners);
    } finally {
      clearInterval(reporter);
    }

    localStorage.setItem(key, JSON.stringify(index));
    invalidateCaches(key);
    if (failures > 0) console.warn(`[NSM] ${failures} page(s) failed after retries`);
    onProgress({ phase: 'done', count: Object.keys(index).length, total: TOTAL_PAGES, failures });
    return index;
  }

  // ============ UI skeleton ============
  const host = document.createElement('div');
  host.id = 'nsm-host';
  document.body.appendChild(host);
  const root = host.attachShadow({ mode: 'open' });

  root.innerHTML = `
    <style>
      :host { position: fixed; inset: 0; pointer-events: none; z-index: 2147483647; }
      #fab, #panel { pointer-events: auto; }
      *, *::before, *::after { box-sizing: border-box; font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif; }

      #fab {
        position: fixed; bottom: 20px; right: 20px;
        width: 52px; height: 52px; border-radius: 50%; border: 0;
        background: #8FBF6F; color: #fff; font-size: 22px; cursor: pointer;
        box-shadow: 0 4px 16px rgba(0,0,0,.25);
        display: grid; place-items: center;
        transition: transform .15s ease, background .15s;
      }
      #fab[hidden] { display: none; }
      #fab:hover { transform: scale(1.06); background: #7aad5a; }
      #fab.pulse { animation: pulse 1.8s ease-in-out infinite; }
      @keyframes pulse {
        0%,100% { box-shadow: 0 4px 16px rgba(0,0,0,.25), 0 0 0 0 rgba(143,191,111,.6); }
        50%     { box-shadow: 0 4px 16px rgba(0,0,0,.25), 0 0 0 12px rgba(143,191,111,0); }
      }

      #panel {
        position: fixed; bottom: 20px; right: 20px;
        width: 420px; max-width: calc(100vw - 40px);
        max-height: calc(100vh - 40px);
        background: #fdfcf8; border-radius: 14px;
        box-shadow: 0 10px 40px rgba(0,0,0,.18), 0 2px 8px rgba(0,0,0,.08);
        display: flex; flex-direction: column; overflow: hidden;
        color: #2a2a2a; font-size: 13px;
        animation: slideUp .2s ease;
      }
      #panel[hidden] { display: none; }
      @keyframes slideUp {
        from { opacity: 0; transform: translateY(8px); }
        to   { opacity: 1; transform: translateY(0); }
      }

      .header {
        display: flex; align-items: center; justify-content: space-between;
        padding: 12px 14px;
        background: linear-gradient(180deg, #f5f3ec, #f0ede3);
        border-bottom: 1px solid #e5e1d3;
      }
      .header h1 { margin: 0; font-size: 14px; font-weight: 600; }
      .icon-btn {
        border: 0; background: transparent; width: 28px; height: 28px;
        border-radius: 6px; cursor: pointer; color: #666; font-size: 18px;
        display: grid; place-items: center; line-height: 1;
      }
      .icon-btn:hover { background: #e5e1d3; color: #2a2a2a; }

      .body { overflow-y: auto; padding: 12px 14px; flex: 1; }
      .section { margin-bottom: 16px; }
      .section:last-child { margin-bottom: 0; }
      .section-title {
        font-size: 10.5px; font-weight: 600; text-transform: uppercase;
        letter-spacing: .06em; color: #8a8578; margin: 0 0 8px;
      }

      .card { background: #fff; border: 1px solid #e5e1d3; border-radius: 10px; padding: 10px 12px; }

      .dataset-name { font-weight: 600; font-size: 13px; }
      .dataset-meta { font-size: 11.5px; color: #8a8578; margin-top: 3px; }

      .badge { display: inline-block; font-size: 10px; padding: 1px 6px; border-radius: 8px; font-weight: 600; }
      .badge.ok { background: #d8ecb9; color: #4a7014; }
      .badge.empty { background: #f0ede3; color: #8a8578; }
      .badge.current { background: #8FBF6F; color: #fff; margin-left: 4px; }
      .badge.old { background: #f5e3c0; color: #8a6014; margin-left: 4px; }

      .btn {
        border: 1px solid #d3cfc0; background: #fff; border-radius: 7px;
        padding: 6px 11px; font-size: 12px; cursor: pointer; color: #2a2a2a;
        font-weight: 500; transition: background .1s, border-color .1s; white-space: nowrap;
      }
      .btn:hover { background: #f5f3ec; border-color: #bfbaa6; }
      .btn:disabled { opacity: .5; cursor: not-allowed; }
      .btn.primary { background: #8FBF6F; border-color: #7aad5a; color: #fff; }
      .btn.primary:hover { background: #7aad5a; }
      .btn.danger { color: #b53b3b; border-color: #e0c0c0; }
      .btn.danger:hover { background: #fbeeee; border-color: #d09090; }
      .btn.small { padding: 4px 9px; font-size: 11px; }
      .btn-row { display: flex; gap: 6px; flex-wrap: wrap; margin-top: 10px; align-items: center; }

      .opt { display: flex; align-items: center; gap: 6px; font-size: 11.5px; color: #6b6657; }
      .opt input { margin: 0; }

      .progress { margin-top: 10px; height: 6px; background: #f0ede3; border-radius: 3px; overflow: hidden; }
      .progress-bar { height: 100%; background: #8FBF6F; transition: width .2s ease; width: 0%; }
      .progress-text { font-size: 11.5px; color: #8a8578; margin-top: 6px; font-variant-numeric: tabular-nums; }
      .progress-detail { font-size: 10.5px; color: #a5a093; margin-top: 2px; font-variant-numeric: tabular-nums; }
      .progress-detail:empty { display: none; }

      .search-input {
        width: 100%; padding: 9px 11px; border: 1px solid #d3cfc0;
        border-radius: 8px; font-size: 13px; outline: none; background: #fff;
        color: #2a2a2a;
      }
      .search-input:focus { border-color: #8FBF6F; box-shadow: 0 0 0 3px rgba(143,191,111,.18); }

      .search-count { font-size: 11px; color: #8a8578; margin-top: 6px; }
      .search-results { margin-top: 4px; max-height: 320px; overflow-y: auto; }
      .search-result {
        display: block; padding: 8px 10px; border-radius: 7px; text-decoration: none;
        color: inherit; border: 1px solid transparent;
      }
      .search-result:hover { background: #f5f3ec; border-color: #e5e1d3; }
      .sr-top { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
      .sr-name { font-weight: 600; font-size: 12.5px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
      .sr-name mark { background: #d8ecb9; color: inherit; padding: 0 1px; border-radius: 2px; }
      .sr-badge {
        font-size: 9.5px; padding: 2px 6px; background: #f0ede3;
        color: #6b6657; border-radius: 6px; white-space: nowrap; flex-shrink: 0;
      }
      .sr-meta { font-size: 11px; color: #8a8578; margin-top: 2px; font-variant-numeric: tabular-nums; }
      .sr-inst { font-size: 10.5px; color: #a5a093; margin-top: 2px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

      .empty-state { text-align: center; padding: 16px 8px; color: #8a8578; font-size: 12px; line-height: 1.5; }

      .toast {
        position: absolute; bottom: 12px; left: 50%; transform: translateX(-50%);
        background: #2a2a2a; color: #fff; padding: 7px 14px; border-radius: 7px;
        font-size: 12px; opacity: 0; transition: opacity .2s; pointer-events: none;
        white-space: nowrap; max-width: 90%; overflow: hidden; text-overflow: ellipsis;
      }
      .toast.show { opacity: 1; }

      .cache-item {
        display: flex; align-items: center; justify-content: space-between;
        gap: 8px; padding: 8px 0; border-bottom: 1px solid #f0ede3; font-size: 12px;
      }
      .cache-item:last-child { border-bottom: 0; }
      .cache-item-info { min-width: 0; flex: 1; }
      .cache-item-name { font-weight: 500; font-size: 12.5px; }
      .cache-item-meta { font-size: 10.5px; color: #8a8578; margin-top: 2px; font-variant-numeric: tabular-nums; }
    </style>

    <button id="fab" aria-label="Open Name Search Manager">🔍</button>
    <div id="panel" hidden role="dialog" aria-label="Name Search Manager">
      <div class="header">
        <h1>Name Search Manager</h1>
        <button class="icon-btn" id="btn-close" title="Close" aria-label="Close">×</button>
      </div>
      <div class="body" id="body"></div>
      <div class="toast" id="toast"></div>
    </div>
  `;

  const $ = sel => root.querySelector(sel);
  const fab = $('#fab');
  const panel = $('#panel');
  const body = $('#body');
  const toastEl = $('#toast');

  function toast(msg, ms = 2200) {
    toastEl.textContent = msg;
    toastEl.classList.add('show');
    clearTimeout(toast._t);
    toast._t = setTimeout(() => toastEl.classList.remove('show'), ms);
  }

  // ============ state ============
  let currentDataset = parseDatasetFromPath(location.pathname);
  let activeBuild    = null;
  let searchQuery    = '';
  let richMode       = JSON.parse(localStorage.getItem('__nsm_rich') ?? 'true');

  // ============ render ============
  function render() {
    const parts = [];

    if (activeBuild) {
      const isCurrent = currentDataset
        && activeBuild.datasetKey === cacheKey(currentDataset.exam, currentDataset.year, currentDataset.group);
      if (!isCurrent) {
        parts.push(`<div class="section"><h2 class="section-title">Active Build</h2><div class="card">
          <div class="dataset-name">${datasetLabel(activeBuild.datasetInfo)}</div>
          ${renderProgress()}
        </div></div>`);
      }
    }

    parts.push(`<div class="section"><h2 class="section-title">${currentDataset ? 'Current Dataset' : 'No Dataset Detected'}</h2>`);
    parts.push(renderCurrentDataset());
    parts.push(`</div>`);

    parts.push(`<div class="section"><h2 class="section-title">Search by Name</h2>
      <input class="search-input" id="search-input" type="text"
             placeholder="Name — order-agnostic, partial, typo-tolerant…"
             value="${escapeHtml(searchQuery)}" autocomplete="off" spellcheck="false">
      <div class="search-count" id="search-count"></div>
      <div class="search-results" id="search-results"></div>
    </div>`);

    const caches = listCached();
    parts.push(`<div class="section"><h2 class="section-title">Cached Datasets (${caches.length})</h2>`);
    if (!caches.length) {
      parts.push(`<div class="card"><div class="empty-state">No cached datasets yet.<br>Open a dataset page and click "Build Index".</div></div>`);
    } else {
      parts.push(`<div class="card">`);
      for (const c of caches) {
        const isCurrent = currentDataset
          && cacheKey(currentDataset.exam, currentDataset.year, currentDataset.group) === c.key;
        const old = c.version < VERSION;
        parts.push(`<div class="cache-item">
          <div class="cache-item-info">
            <div class="cache-item-name">${datasetLabel(c)}${isCurrent ? '<span class="badge current">current</span>' : ''}${old ? '<span class="badge old">old format</span>' : ''}</div>
            <div class="cache-item-meta">${fmtNum(c.count)} students · ${fmtBytes(c.bytes)}</div>
          </div>
          <button class="btn small danger" data-action="delete-cache"
                  data-key="${c.key}" data-label="${escapeHtml(datasetLabel(c))}">Delete</button>
        </div>`);
      }
      parts.push(`</div>`);
      parts.push(`<div class="btn-row"><button class="btn danger small" data-action="clear-all">Clear All Caches</button></div>`);
    }
    parts.push(`</div>`);

    body.innerHTML = parts.join('');

    body.querySelectorAll('[data-action]').forEach(el => {
      el.addEventListener('click', handleAction);
    });

    const si = body.querySelector('#search-input');
    if (si) {
      si.addEventListener('input', e => {
        searchQuery = e.target.value;
        runSearchDebounced();
      });
      si.focus();
      si.setSelectionRange(searchQuery.length, searchQuery.length);
    }
    runSearchNow();

    const richCb = body.querySelector('#rich-toggle');
    if (richCb) {
      richCb.checked = richMode;
      richCb.addEventListener('change', e => {
        richMode = e.target.checked;
        localStorage.setItem('__nsm_rich', JSON.stringify(richMode));
        toast(richMode ? 'Rich mode: name + rank/marks/GPA/institution'
                       : 'Compact mode: name → roll only');
      });
    }
  }

  function renderCurrentDataset() {
    if (!currentDataset) {
      return `<div class="card"><div class="empty-state">Open a dataset page (HSC/SSC × year × group) to build or update its index.<br>You can still search existing caches above.</div></div>`;
    }
    const key = cacheKey(currentDataset.exam, currentDataset.year, currentDataset.group);
    const idx = getIndex(key);
    const count = idx ? Object.keys(idx).length : 0;
    const isBuildingThis = activeBuild && activeBuild.datasetKey === key;

    let actions;
    if (isBuildingThis) {
      actions = renderProgress();
    } else {
      actions = `<div class="btn-row">
        ${count
          ? `<button class="btn primary" data-action="rebuild">↻ Rebuild</button>
             <button class="btn danger" data-action="delete">Delete Cache</button>`
          : `<button class="btn primary" data-action="build">⚡ Build Index</button>`}
        <label class="opt"><input type="checkbox" id="rich-toggle"> Rich mode</label>
      </div>`;
    }

    return `<div class="card">
      <div class="dataset-name">${datasetLabel(currentDataset)}</div>
      <div class="dataset-meta">${count
        ? `<span class="badge ok">Indexed · ${fmtNum(count)} students</span>`
        : `<span class="badge empty">Not indexed</span>`}</div>
      ${actions}
    </div>`;
  }

  function renderProgress() {
    const p = activeBuild.progress || { phase: 'detect', message: 'Starting…' };
    let barPct = 0;
    let text   = p.message || 'Starting…';
    let detail = '';

    if (p.phase === 'scrape') {
      barPct = p.total ? (p.current / p.total * 100) : 0;
      text   = `${fmtNum(p.current)} / ${fmtNum(p.total)} pages · ${fmtNum(p.count)} students`;

      const bits = [];
      if (p.warmingUp) {
        bits.push('estimating…');
      } else {
        if (p.rate > 0) bits.push(`${p.rate.toFixed(1)} p/s`);
        if (p.eta  > 0) bits.push(`~${fmtEta(p.eta)} left`);
      }
      if (p.elapsed != null) bits.push(`${fmtEta(p.elapsed)} elapsed`);
      if (p.failures)        bits.push(`⚠ ${p.failures} failed`);
      detail = bits.join(' · ');
    }

    return `<div data-progress-wrap>
      <div class="progress"><div class="progress-bar" style="width:${barPct}%"></div></div>
      <div class="progress-text">${text}</div>
      <div class="progress-detail">${detail}</div>
      <div class="btn-row"><button class="btn danger small" data-action="cancel">Cancel</button></div>
    </div>`;
  }

  function updateProgressUI(p) {
    const wrap = body.querySelector('[data-progress-wrap]');
    if (!wrap) return;
    const bar = wrap.querySelector('.progress-bar');
    const txt = wrap.querySelector('.progress-text');
    const det = wrap.querySelector('.progress-detail');

    if (p.phase === 'detect') {
      if (bar) bar.style.width = '0%';
      if (txt) txt.textContent = p.message || 'Detecting…';
      if (det) det.textContent = '';
      return;
    }

    if (p.phase === 'scrape') {
      const pct = p.total ? (p.current / p.total * 100) : 0;
      if (bar) bar.style.width = pct + '%';
      if (txt) txt.textContent =
        `${fmtNum(p.current)} / ${fmtNum(p.total)} pages · ${fmtNum(p.count)} students`;

      if (det) {
        const bits = [];
        if (p.warmingUp) {
          bits.push('estimating…');
        } else {
          if (p.rate > 0) bits.push(`${p.rate.toFixed(1)} p/s`);
          if (p.eta  > 0) bits.push(`~${fmtEta(p.eta)} left`);
        }
        if (p.elapsed != null) bits.push(`${fmtEta(p.elapsed)} elapsed`);
        if (p.failures)        bits.push(`⚠ ${p.failures} failed`);
        det.textContent = bits.join(' · ');
      }
    }
  }

  // ============ search render ============
  let searchTimer = null;
  function runSearchDebounced() {
    clearTimeout(searchTimer);
    searchTimer = setTimeout(runSearchNow, 150);
  }

  function highlight(name, qTokens) {
    let out = escapeHtml(name);
    for (const qt of qTokens) {
      if (qt.length < 2) continue;
      const re = new RegExp('(' + qt.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
      out = out.replace(re, '<mark>$1</mark>');
    }
    return out;
  }

  function runSearchNow() {
    const countEl = body.querySelector('#search-count');
    const srEl    = body.querySelector('#search-results');
    if (!countEl || !srEl) return;

    const q = searchQuery.trim();
    if (!q) {
      countEl.textContent = '';
      srEl.innerHTML = `<div class="empty-state">Type to search across all cached datasets.</div>`;
      return;
    }

    const qNorm   = normalize(q);
    const qTokens = qNorm.split(' ').filter(Boolean);
    const results = searchAllCaches(q);

    if (!results.length) {
      countEl.textContent = '';
      srEl.innerHTML = `<div class="empty-state">No matches for "${escapeHtml(q)}".</div>`;
      return;
    }

    const shown = results.slice(0, SEARCH_LIMIT);
    countEl.textContent = results.length === 1
      ? '1 match'
      : `${fmtNum(results.length)} matches` +
        (results.length > SEARCH_LIMIT ? ` (showing first ${SEARCH_LIMIT})` : '');

    srEl.innerHTML = shown.map(r => {
      const e = r.entry;
      const c = r.dataset;
      const v = e.val;
      const legacy = typeof v === 'string';
      const roll = legacy ? v : v.roll;
      const url  = `${location.origin}/${c.exam}/${c.year}/individual/${roll}/`;
      let meta = `Roll ${roll}`;
      if (!legacy) {
        const bits = [];
        if (v.rank)  bits.push(`Rank ${v.rank}`);
        if (v.marks) bits.push(`${v.marks} marks`);
        if (v.gpa)   bits.push(`GPA ${v.gpa}`);
        if (bits.length) meta += ' · ' + bits.join(' · ');
      }
      const inst = !legacy && v.inst ? `<div class="sr-inst">${escapeHtml(v.inst)}</div>` : '';
      return `<a class="search-result" href="${url}" target="_blank" rel="noopener">
        <div class="sr-top">
          <div class="sr-name">${highlight(e.name, qTokens)}</div>
          <span class="sr-badge">${datasetLabel(c)}</span>
        </div>
        <div class="sr-meta">${meta}</div>
        ${inst}
      </a>`;
    }).join('');

    if (results.length > SEARCH_LIMIT) {
      srEl.innerHTML += `<div class="empty-state">… and ${fmtNum(results.length - SEARCH_LIMIT)} more. Refine your query.</div>`;
    }
  }

  // ============ actions ============
  async function handleAction(e) {
    const btn    = e.currentTarget;
    const action = btn.dataset.action;

    if (action === 'build' || action === 'rebuild') {
      if (!currentDataset) return;
      if (activeBuild) return toast('A build is already running');
      const d = { ...currentDataset };
      const controller = new AbortController();
      activeBuild = {
        controller,
        datasetKey: cacheKey(d.exam, d.year, d.group),
        datasetInfo: d,
        progress: { phase: 'detect', message: 'Starting…' },
      };
      fab.classList.add('pulse');
      render();

      try {
        const index = await buildIndex(d, p => {
          activeBuild.progress = p;
          updateProgressUI(p);
        }, controller.signal, { rich: richMode });
        toast(`✅ Indexed ${fmtNum(Object.keys(index).length)} students in ${datasetLabel(d)}`);
      } catch (err) {
        if (err.name === 'AbortError') toast('Build cancelled');
        else toast(`❌ ${err.message || 'Build failed'}`);
      } finally {
        activeBuild = null;
        fab.classList.remove('pulse');
        render();
      }
      return;
    }

    if (action === 'cancel') {
      if (activeBuild) activeBuild.controller.abort();
      return;
    }

    if (action === 'delete') {
      if (!currentDataset) return;
      const key = cacheKey(currentDataset.exam, currentDataset.year, currentDataset.group);
      if (!confirm(`Delete cache for ${datasetLabel(currentDataset)}?`)) return;
      localStorage.removeItem(key);
      invalidateCaches(key);
      toast('Cache deleted');
      render();
      return;
    }

    if (action === 'delete-cache') {
      const key   = btn.dataset.key;
      const label = btn.dataset.label;
      if (!confirm(`Delete cache for ${label}?`)) return;
      localStorage.removeItem(key);
      invalidateCaches(key);
      toast('Cache deleted');
      render();
      return;
    }

    if (action === 'clear-all') {
      if (!confirm('Delete ALL cached datasets? This cannot be undone.')) return;
      for (const c of listCached()) {
        localStorage.removeItem(c.key);
        invalidateCaches(c.key);
      }
      toast('All caches cleared');
      render();
      return;
    }
  }

  // ============ panel toggle + SPA nav ============
  fab.addEventListener('click', () => {
    panel.hidden = false;
    fab.hidden = true;
    render();
  });
  $('#btn-close').addEventListener('click', () => {
    panel.hidden = true;
    fab.hidden = false;
  });

  let lastPath = location.pathname;
  setInterval(() => {
    if (location.pathname !== lastPath) {
      lastPath = location.pathname;
      currentDataset = parseDatasetFromPath(location.pathname);
      if (!panel.hidden) render();
    }
  }, 500);

  // expose for power users
  window.searchStudent = (q) => {
    const results = searchAllCaches(q);
    console.table(results.slice(0, 50).map(r => ({
      name: r.entry.name,
      roll: typeof r.entry.val === 'string' ? r.entry.val : r.entry.val.roll,
      dataset: datasetLabel(r.dataset),
      score: r.score,
    })));
    return results;
  };

  console.log('[NSM] Name Search Manager v2.2 ready. Click 🔍 (bottom-right).');
})();
```
