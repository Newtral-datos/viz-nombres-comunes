<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  export let ages;

  // null = todos, 'H' = solo masculinos dominantes, 'M' = solo femeninos dominantes
  let activeSex    = null;
  let sortBy       = 'freq';
  let sortAsc      = false;
  let rarityFilter = null;
  let query        = '';
  let shown        = 100;

  const RARITY_LEVELS = [
    { label: 'Muy raro',    min: 92, max: 100 },
    { label: 'Raro',        min: 80, max: 91  },
    { label: 'Poco común',  min: 68, max: 79  },
    { label: 'Común',       min: 55, max: 67  },
    { label: 'Popular',     min: 40, max: 54  },
    { label: 'Muy popular', min: 0,  max: 39  },
  ];

  function rarityLabel(logPct) {
    if (logPct >= 92) return 'Muy raro';
    if (logPct >= 80) return 'Raro';
    if (logPct >= 68) return 'Poco común';
    if (logPct >= 55) return 'Común';
    if (logPct >= 40) return 'Popular';
    return 'Muy popular';
  }

  function rarityColor(logPct) {
    if (logPct >= 68) return '#ff922b';
    if (logPct >= 40) return '#aaa';
    return '#01f3b3';
  }

  function stripAccents(s) {
    return s.normalize('NFD').replace(/[̀-ͯ]/g, '');
  }

  function formatNum(n) {
    if (n == null) return '—';
    return Math.round(n).toString().replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  }

  // Construye una entrada por nombre usando los datos del sexo dominante.
  // logPct se calcula sobre el ranking del sexo dominante (no el total fusionado)
  // para que los umbrales coincidan con NameStory y NameEntry.
  $: allMerged = (() => {
    const h = ages?.H ?? {};
    const m = ages?.M ?? {};

    const hSorted = Object.entries(h).sort((a, b) => b[1].freq - a[1].freq);
    const mSorted = Object.entries(m).sort((a, b) => b[1].freq - a[1].freq);
    const hTotal  = hSorted.length;
    const mTotal  = mSorted.length;
    const hRank   = new Map(hSorted.map(([n], i) => [n, i + 1]));
    const mRank   = new Map(mSorted.map(([n], i) => [n, i + 1]));

    const names  = new Set([...Object.keys(h), ...Object.keys(m)]);
    const merged = [];
    for (const name of names) {
      const freqH = h[name]?.freq ?? 0;
      const freqM = m[name]?.freq ?? 0;
      const sex   = freqH >= freqM ? 'H' : 'M';
      const freq  = freqH + freqM;
      const ageH  = h[name]?.avg_age; const ageM = m[name]?.avg_age;
      const avg_age = (ageH != null && ageM != null)
        ? Math.round((freqH * ageH + freqM * ageM) / freq * 10) / 10
        : (ageH ?? ageM ?? null);
      const rank   = sex === 'H' ? hRank.get(name) : mRank.get(name);
      const total  = sex === 'H' ? hTotal : mTotal;
      const logPct = rank === 1 ? 0 : Math.round(Math.log(rank) / Math.log(total) * 100);
      merged.push({ name, freq, avg_age, sex, logPct });
    }
    merged.sort((a, b) => b.freq - a.freq);
    return merged;
  })();

  $: rankedAll = activeSex ? allMerged.filter(e => e.sex === activeSex) : allMerged;

  $: rarityCounts = (() => {
    const counts = {};
    for (const lvl of RARITY_LEVELS) counts[lvl.label] = 0;
    for (const e of rankedAll) counts[rarityLabel(e.logPct)]++;
    return counts;
  })();

  $: filtered = (() => {
    let result = rankedAll;

    if (query.trim()) {
      const q = stripAccents(query.trim().toUpperCase());
      result   = result.filter(e => stripAccents(e.name).includes(q));
    }

    if (rarityFilter !== null) {
      const lvl = RARITY_LEVELS.find(l => l.label === rarityFilter);
      result    = result.filter(e => e.logPct >= lvl.min && e.logPct <= lvl.max);
    }

    const dir = sortAsc ? 1 : -1;
    return [...result].sort((a, b) => {
      if (sortBy === 'rarity') return dir * (a.logPct - b.logPct);
      if (sortBy === 'freq')   return dir * (a.freq - b.freq);
      if (sortBy === 'age')    return dir * ((a.avg_age ?? -1) - (b.avg_age ?? -1));
      if (sortBy === 'name')   return dir * a.name.localeCompare(b.name, 'es');
      return 0;
    });
  })();

  $: visible = filtered.slice(0, shown);

  function setSort(col) {
    if (sortBy === col) sortAsc = !sortAsc;
    else { sortBy = col; sortAsc = col === 'name'; }
    shown = 100;
  }

  function toggleRarity(label) {
    rarityFilter = rarityFilter === label ? null : label;
    shown = 100;
  }

  function switchSex(s) {
    activeSex    = activeSex === s ? null : s;
    rarityFilter = null;
    shown        = 100;
  }

  function onQueryInput() { shown = 100; }

  $: arrow = (col) => {
    if (sortBy !== col) return '';
    return sortAsc ? ' ↑' : ' ↓';
  };

  const SORT_LABEL = { freq: 'frecuencia', rarity: 'rareza', age: 'edad media', name: 'nombre' };
</script>

<div class="explore-page">
  <header class="explore-header">
    <button class="back-btn" on:click={() => dispatch('back')}>← Volver</button>
    <span class="explore-title">Explorar nombres</span>
    <div class="sex-toggle">
      <button class:active={activeSex === 'H'} on:click={() => switchSex('H')}>Hombre</button>
      <button class:active={activeSex === 'M'} on:click={() => switchSex('M')}>Mujer</button>
    </div>
  </header>

  <div class="controls">
    <input
      class="search-input"
      bind:value={query}
      on:input={onQueryInput}
      placeholder="Buscar nombre…"
      autocomplete="off"
      spellcheck="false"
    />
    <div class="chips">
      {#each RARITY_LEVELS as lvl}
        <button
          class="chip"
          class:active={rarityFilter === lvl.label}
          on:click={() => toggleRarity(lvl.label)}
        >
          {lvl.label}
          <span class="chip-count">{rarityCounts[lvl.label] ?? 0}</span>
        </button>
      {/each}
    </div>
  </div>

  <div class="table-wrap">
    <table>
      <thead>
        <tr>
          <th on:click={() => setSort('name')}>Nombre{arrow('name')}</th>
          <th on:click={() => setSort('rarity')}>Rareza{arrow('rarity')}</th>
          <th on:click={() => setSort('freq')}>Frecuencia{arrow('freq')}</th>
          <th on:click={() => setSort('age')}>Edad media{arrow('age')}</th>
        </tr>
      </thead>
      <tbody>
        {#each visible as e (e.name)}
          <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
          <tr on:click={() => dispatch('discover', { name: e.name, sex: e.sex })}>
            <td class="td-name">{e.name}</td>
            <td>
              <span class="badge" style="color:{rarityColor(e.logPct)}">{rarityLabel(e.logPct)}</span>
            </td>
            <td class="td-num">{formatNum(e.freq)}</td>
            <td class="td-num">
              {e.avg_age != null ? e.avg_age.toString().replace('.', ',') : '—'}
            </td>
          </tr>
        {/each}
      </tbody>
    </table>

    {#if visible.length === 0}
      <p class="empty">No hay nombres con esos filtros.</p>
    {/if}

    {#if shown < filtered.length}
      <div class="load-more">
        <button on:click={() => shown += 100}>
          Ver {Math.min(100, filtered.length - shown)} más ({filtered.length - shown} restantes)
        </button>
      </div>
    {/if}
  </div>

  <div class="status-bar">
    {filtered.length.toLocaleString('es')} nombres · ordenados por {SORT_LABEL[sortBy]} {sortAsc ? '↑' : '↓'}
  </div>
</div>

<style>
  .explore-page {
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .explore-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 24px;
    border-bottom: 1px solid var(--border);
    flex-shrink: 0;
    gap: 12px;
  }

  .back-btn {
    background: none;
    border: none;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font);
    color: var(--muted);
    cursor: pointer;
    padding: 0;
    white-space: nowrap;
    transition: color 0.1s;
  }
  .back-btn:hover { color: var(--text); }

  .explore-title {
    font-size: 14px;
    font-weight: 700;
    color: var(--text);
    flex: 1;
    text-align: center;
  }

  .sex-toggle { display: flex; gap: 4px; flex-shrink: 0; }
  .sex-toggle button {
    padding: 4px 12px;
    border: 1.5px solid var(--border);
    border-radius: 999px;
    background: none;
    font-size: 12px;
    font-weight: 600;
    font-family: var(--font);
    cursor: pointer;
    color: var(--muted);
    transition: all 0.12s;
  }
  .sex-toggle button.active {
    background: #1a1a1a;
    border-color: #1a1a1a;
    color: #fff;
  }

  .controls {
    padding: 12px 24px;
    border-bottom: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    gap: 10px;
    flex-shrink: 0;
  }

  .search-input {
    height: 38px;
    border: 1.5px solid var(--border);
    border-radius: 8px;
    padding: 0 14px;
    font-size: 14px;
    font-family: var(--font);
    outline: none;
    width: 100%;
    transition: border-color 0.15s;
  }
  .search-input:focus { border-color: #aaa; }
  .search-input::placeholder { color: #ccc; }

  .chips {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  .chip {
    padding: 4px 10px;
    border: 1.5px solid var(--border);
    border-radius: 999px;
    background: none;
    font-size: 12px;
    font-weight: 600;
    font-family: var(--font);
    cursor: pointer;
    color: var(--muted);
    display: flex;
    align-items: center;
    gap: 5px;
    transition: all 0.12s;
  }
  .chip:hover { border-color: #aaa; color: var(--text); }
  .chip.active {
    background: #1a1a1a;
    border-color: #1a1a1a;
    color: #fff;
  }

  .chip-count {
    font-size: 10px;
    font-weight: 500;
    opacity: 0.6;
  }

  .table-wrap {
    flex: 1;
    overflow-y: auto;
    overflow-x: auto;
  }

  table {
    width: 100%;
    border-collapse: collapse;
  }

  thead th {
    position: sticky;
    top: 0;
    background: #fff;
    border-bottom: 1.5px solid var(--border);
    padding: 10px 16px;
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--muted);
    cursor: pointer;
    user-select: none;
    text-align: left;
    white-space: nowrap;
    transition: color 0.1s;
  }
  thead th:hover { color: var(--text); }

  tbody tr {
    border-bottom: 1px solid #f3f3f3;
    cursor: pointer;
    transition: background 0.1s;
  }
  tbody tr:hover { background: #f9f9f9; }

  tbody td {
    padding: 10px 16px;
    font-size: 13px;
  }

  .td-name {
    font-weight: 600;
    color: var(--text);
  }

  .td-num {
    color: #555;
    font-variant-numeric: tabular-nums;
  }

  .badge {
    font-size: 12px;
    font-weight: 600;
  }

  .load-more {
    padding: 20px;
    text-align: center;
  }

  .load-more button {
    padding: 9px 24px;
    border: 1.5px solid var(--border);
    border-radius: 8px;
    background: none;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font);
    cursor: pointer;
    color: var(--muted);
    transition: all 0.12s;
  }
  .load-more button:hover { border-color: #aaa; color: var(--text); }

  .empty {
    text-align: center;
    color: var(--muted);
    font-size: 14px;
    padding: 48px 24px;
  }

  .status-bar {
    border-top: 1px solid var(--border);
    padding: 8px 24px;
    font-size: 11px;
    color: var(--muted);
    flex-shrink: 0;
  }

  @media (max-width: 600px) {
    .explore-title { display: none; }
    .controls { padding: 10px 16px; }
    thead th, tbody td { padding: 9px 10px; font-size: 12px; }
  }
</style>
