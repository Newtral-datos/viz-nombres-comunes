<script>
  import { createEventDispatcher } from 'svelte';
  import BumpChart from './BumpChart.svelte';

  const dispatch = createEventDispatcher();

  export let data;
  export let ages;
  export let name;  // UPPERCASE
  export let sex;   // 'H' | 'M' (initial)

  const PALETTE = [
    '#01f3b3','#305cfa','#ff4757','#eaea40','#a259ff',
    '#ff922b','#40c0ff','#ff6b9d','#82ca00','#494949',
  ];

  let activeSex = sex;
  let compared  = [];

  function toTitle(s) {
    return s.charAt(0) + s.slice(1).toLowerCase();
  }

  function formatNum(n) {
    if (n == null) return '—';
    return Math.round(n).toString().replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  }

  function stripAccents(s) {
    return s.normalize('NFD').replace(/[̀-ͯ]/g, '');
  }

  // Rareza: escala logarítmica sobre ranking de frecuencia
  let freqRanks = {};
  $: if (ages) {
    freqRanks = {};
    for (const sx of ['H', 'M']) {
      const entries = Object.entries(ages[sx] ?? {}).sort((a, b) => b[1].freq - a[1].freq);
      const total   = entries.length;
      freqRanks[sx] = new Map(entries.map(([n], i) => {
        const rank   = i + 1;
        const logPct = rank === 1 ? 0 : Math.round(Math.log(rank) / Math.log(total) * 100);
        return [n, { rank, logPct }];
      }));
    }
  }

  $: rarityInfo    = freqRanks[activeSex]?.get(name) ?? null;
  $: rarityPct     = rarityInfo?.logPct ?? null;
  $: rarityRankNum = rarityInfo?.rank   ?? null;

  function rarityLabel(logPct) {
    if (logPct === null || logPct === undefined) return null;
    if (logPct >= 92) return 'Muy raro';
    if (logPct >= 80) return 'Raro';
    if (logPct >= 68) return 'Poco común';
    if (logPct >= 55) return 'Común';
    if (logPct >= 40) return 'Popular';
    return 'Muy popular';
  }

  function rarityColor(logPct) {
    if (logPct === null || logPct === undefined) return '#aaa';
    if (logPct >= 68) return '#ff922b';
    if (logPct >= 40) return '#aaa';
    return '#01f3b3';
  }

  function compareRarityPct(n) {
    return freqRanks[activeSex]?.get(n)?.logPct ?? null;
  }

  // ¿Existe el nombre en ambos sexos?
  $: inH      = !!(data?.H?.[name] || ages?.H?.[name]);
  $: inM      = !!(data?.M?.[name] || ages?.M?.[name]);
  $: canToggle = inH && inM;

  // Datos según sexo activo
  $: censusData = ages?.[activeSex]?.[name];
  $: birthData  = data?.[activeSex]?.[name];

  // Frecuencia total sumando ambos sexos (el nombre puede existir en H y M)
  $: totalFreq = (ages?.H?.[name]?.freq ?? 0) + (ages?.M?.[name]?.freq ?? 0);
  $: totalAvgAge = (() => {
    const h = ages?.H?.[name]; const m = ages?.M?.[name];
    if (h && m) return Math.round(((h.freq * h.avg_age) + (m.freq * m.avg_age)) / (h.freq + m.freq) * 10) / 10;
    return (h ?? m)?.avg_age ?? null;
  })();
  $: years      = data?.years ?? [];

  // Mejor posición histórica
  $: bestRank = (() => {
    if (!birthData) return null;
    let best = Infinity, bestYear = null, bestCount = null;
    birthData.ranks.forEach((r, i) => {
      if (r != null && r < best) {
        best      = r;
        bestYear  = years[i];
        bestCount = birthData.counts[i];
      }
    });
    return best === Infinity ? null : { rank: best, year: bestYear, count: bestCount };
  })();

  // Último año con datos
  $: lastData = (() => {
    if (!birthData) return null;
    for (let i = years.length - 1; i >= 0; i--) {
      if (birthData.ranks[i] != null)
        return { rank: birthData.ranks[i], count: birthData.counts[i], year: years[i] };
    }
    return null;
  })();

  // Primer año en el top 100
  $: firstYear = (() => {
    if (!birthData) return null;
    for (let i = 0; i < years.length; i++) {
      if (birthData.ranks[i] != null) return years[i];
    }
    return null;
  })();

  // Total nacimientos acumulados
  $: totalBirths = (() => {
    if (!birthData) return null;
    return birthData.counts.reduce((s, c) => s + (c ?? 0), 0);
  })();

  // Selección para el chart
  $: selected = [{ name, color: PALETTE[0] }, ...compared];

  // Compare
  let compareQuery       = '';
  let compareSuggestions = [];

  function onCompareInput() {
    if (!compareQuery.trim()) { compareSuggestions = []; return; }
    const q    = stripAccents(compareQuery.trim()).toUpperCase();
    const pool = Object.keys(data?.[activeSex] ?? {})
      .filter(n => n !== name && !compared.find(c => c.name === n));
    const prefix   = pool.filter(n => stripAccents(n).startsWith(q));
    const contains = pool.filter(n => !stripAccents(n).startsWith(q) && stripAccents(n).includes(q));
    compareSuggestions = [...prefix, ...contains].slice(0, 6);
  }

  function addCompare(n) {
    if (!n || compared.find(c => c.name === n) || n === name) return;
    const color = PALETTE[(compared.length + 1) % PALETTE.length];
    compared    = [...compared, { name: n, color }];
    compareQuery       = '';
    compareSuggestions = [];
  }

  function removeCompare(n) {
    compared = compared.filter(c => c.name !== n);
  }

  function onCompareKeydown(e) {
    if (e.key === 'Enter' && compareSuggestions.length) addCompare(compareSuggestions[0]);
    if (e.key === 'Escape') { compareQuery = ''; compareSuggestions = []; }
  }

  function switchSex(s) {
    activeSex = s;
    compared  = [];
  }
</script>

<div class="story-page">
  <header class="story-header">
    <button class="back-btn" on:click={() => dispatch('back')}>← Otro nombre</button>
    {#if canToggle}
      <div class="sex-toggle">
        <button class:active={activeSex === 'H'} on:click={() => switchSex('H')}>Hombre</button>
        <button class:active={activeSex === 'M'} on:click={() => switchSex('M')}>Mujer</button>
      </div>
    {/if}
  </header>

  <div class="story-content">
  <div class="story-inner">

    <!-- Hero -->
    <section class="hero">
      <h1 class="hero-name">{name}</h1>
      {#if censusData}
        <p class="hero-text">En España hay <strong>{formatNum(totalFreq)}</strong> personas llamadas <strong>{name}</strong>{#if totalAvgAge}, con una edad media de <strong>{String(totalAvgAge).replace('.', ',')} años</strong>{/if}.</p>
      {:else if birthData}
        <p class="hero-text">
          {toTitle(name)} ha aparecido en el top 100 de nombres de recién nacidos en España.
        </p>
      {:else}
        <p class="hero-text muted">
          No hemos encontrado datos para este nombre. Prueba con otro.
        </p>
      {/if}
    </section>

    <!-- Tarjetas de estadísticas -->
    {#if birthData && (bestRank || lastData || firstYear)}
      <section class="stats-grid">
        {#if bestRank}
          <div class="stat-card">
            <div class="stat-value">nº{bestRank.rank}</div>
            <div class="stat-label">Mejor posición</div>
            <div class="stat-sub">{bestRank.year} · {formatNum(bestRank.count)} bebés</div>
          </div>
        {/if}
        {#if lastData}
          <div class="stat-card">
            <div class="stat-value">nº{lastData.rank}</div>
            <div class="stat-label">Posición en {lastData.year}</div>
            <div class="stat-sub">{formatNum(lastData.count)} nacimientos</div>
          </div>
        {/if}
        {#if firstYear}
          <div class="stat-card">
            <div class="stat-value">{firstYear}</div>
            <div class="stat-label">Primera vez en top 100</div>
            <div class="stat-sub">de recién nacidos</div>
          </div>
        {/if}
        {#if totalBirths}
          <div class="stat-card">
            <div class="stat-value">{formatNum(totalBirths)}</div>
            <div class="stat-label">Bebés registrados</div>
            <div class="stat-sub">entre 2002 y 2024</div>
          </div>
        {/if}
        {#if rarityPct !== null}
          <div class="stat-card">
            <div class="stat-value" style="color:{rarityColor(rarityPct)}">{rarityLabel(rarityPct)}</div>
            <div class="stat-label">Rareza del nombre</div>
            <div class="stat-sub">nº{formatNum(rarityRankNum)} más común · 2024</div>
          </div>
        {/if}
      </section>
    {/if}
    {#if !birthData && rarityPct !== null}
      <section class="stats-grid" style="margin-bottom: 32px;">
        <div class="stat-card">
          <div class="stat-value" style="color:{rarityColor(rarityPct)}">{rarityLabel(rarityPct)}</div>
          <div class="stat-label">Rareza del nombre</div>
          <div class="stat-sub">nº{formatNum(rarityRankNum)} más común · 2024</div>
        </div>
      </section>
    {/if}

    <!-- Gráfico de evolución -->
    {#if birthData}
      <section class="chart-section">
        <h2 class="section-title">Evolución en el ranking de nacimientos</h2>
        <p class="section-sub">
          Top {activeSex === 'H' ? 'niños' : 'niñas'} · España · 2002–2024
        </p>
        <BumpChart {data} {selected} sex={activeSex} />
      </section>

      <!-- Comparar -->
      <section class="compare-section">
        <h2 class="section-title">Comparar con</h2>
        <div class="compare-controls">
          <div class="compare-input-wrap">
            <div class="compare-input-box">
              <input
                bind:value={compareQuery}
                on:input={onCompareInput}
                on:keydown={onCompareKeydown}
                on:blur={() => setTimeout(() => compareSuggestions = [], 150)}
                placeholder="Añadir nombre…"
                autocomplete="off"
                spellcheck="false"
              />
            </div>
            {#if compareSuggestions.length}
              <div class="compare-dropdown">
                {#each compareSuggestions as n}
                  <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
                  <div class="dd-item" on:mousedown|preventDefault={() => addCompare(n)}>
                    <span>{toTitle(n)}</span>
                    {#if rarityLabel(compareRarityPct(n))}
                      <span class="dd-rarity" style="color:{rarityColor(compareRarityPct(n))}">{rarityLabel(compareRarityPct(n))}</span>
                    {/if}
                  </div>
                {/each}
              </div>
            {/if}
          </div>

          <div class="chips">
            <div class="chip" style="border-color:{PALETTE[0]}; color:{PALETTE[0]}">
              <span class="chip-dot" style="background:{PALETTE[0]}"></span>
              {toTitle(name)}
            </div>
            {#each compared as { name: n, color }}
              <div class="chip" style="border-color:{color}; color:{color}">
                <span class="chip-dot" style="background:{color}"></span>
                {toTitle(n)}
                <button class="chip-remove" on:click={() => removeCompare(n)}>×</button>
              </div>
            {/each}
          </div>
        </div>
      </section>

    {:else if censusData}
      <section class="no-births">
        <p>
          Este nombre no ha aparecido en el top 100 de nacimientos entre 2002 y 2023.
          {#if totalAvgAge}
            La mayoría de {toTitle(name)} tiene alrededor de {Math.round(totalAvgAge)} años.
          {/if}
        </p>
      </section>
    {/if}

  </div><!-- story-inner -->
  </div><!-- story-content -->
</div>

<style>
  .story-page {
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  /* Header */
  .story-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 14px 24px;
    border-bottom: 1px solid var(--border);
    flex-shrink: 0;
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
    transition: color 0.1s;
  }
  .back-btn:hover { color: var(--text); }

  .sex-toggle { display: flex; gap: 4px; }
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

  /* Content */
  .story-content {
    flex: 1;
    overflow-y: auto;
  }

  .story-inner {
    max-width: 860px;
    margin: 0 auto;
    padding: 40px 24px 64px;
    width: 100%;
  }

  @keyframes fadeUp {
    from { opacity: 0; transform: translateY(14px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  /* Hero */
  .hero { margin-bottom: 32px; }

  .hero-name {
    font-size: clamp(2.5rem, 6vw, 4rem);
    font-weight: 800;
    letter-spacing: -0.04em;
    line-height: 1;
    margin-bottom: 14px;
    color: #01f3b3;
    animation: fadeUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
  }

  .hero-text {
    font-size: 16px;
    line-height: 1.7;
    color: #333;
    animation: fadeUp 0.5s 0.08s cubic-bezier(0.22, 1, 0.36, 1) both;
  }

  .hero-text :global(strong) {
    color: #01f3b3;
    font-weight: 800;
    font-size: 1.15em;
  }

  .muted { color: #aaa; }

  /* Stats */
  .stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(168px, 1fr));
    gap: 12px;
    margin-bottom: 48px;
  }

  .stat-card {
    border: 1.5px solid var(--border);
    border-radius: 12px;
    padding: 18px 20px;
    transition: border-color 0.15s, transform 0.15s;
    animation: fadeUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
  }
  .stat-card:hover {
    border-color: #01f3b3;
    transform: translateY(-2px);
  }

  .stat-card:nth-child(1) { animation-delay: 0.14s; }
  .stat-card:nth-child(2) { animation-delay: 0.20s; }
  .stat-card:nth-child(3) { animation-delay: 0.26s; }
  .stat-card:nth-child(4) { animation-delay: 0.32s; }

  .stat-value {
    font-size: 1.75rem;
    font-weight: 800;
    letter-spacing: -0.04em;
    line-height: 1;
    margin-bottom: 7px;
    color: #01f3b3;
  }

  .stat-label {
    font-size: 12px;
    font-weight: 600;
    color: #333;
    margin-bottom: 3px;
  }

  .stat-sub {
    font-size: 11px;
    color: #aaa;
  }

  /* Chart */
  .chart-section {
    margin-bottom: 36px;
    animation: fadeUp 0.5s 0.38s cubic-bezier(0.22, 1, 0.36, 1) both;
  }

  .section-title {
    font-size: 14px;
    font-weight: 700;
    letter-spacing: -0.01em;
    margin-bottom: 3px;
  }

  .section-sub {
    font-size: 11px;
    color: #aaa;
    margin-bottom: 16px;
  }

  /* Compare */
  .compare-section {
    margin-bottom: 36px;
    animation: fadeUp 0.5s 0.46s cubic-bezier(0.22, 1, 0.36, 1) both;
  }

  .compare-controls {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-top: 12px;
  }

  .compare-input-wrap { position: relative; width: 220px; }

  .compare-input-box {
    height: 36px;
    border: 1.5px solid var(--border);
    border-radius: 8px;
    padding: 0 12px;
    display: flex;
    align-items: center;
    transition: border-color 0.15s;
  }
  .compare-input-box:focus-within { border-color: #aaa; }

  .compare-input-box input {
    width: 100%;
    border: none;
    outline: none;
    font-size: 13px;
    font-family: var(--font);
    background: transparent;
  }
  .compare-input-box input::placeholder { color: #ccc; }

  .compare-dropdown {
    position: absolute;
    top: calc(100% + 4px);
    left: 0;
    min-width: 200px;
    background: #fff;
    border: 1px solid #e5e5e5;
    border-radius: 8px;
    box-shadow: 0 4px 16px rgba(0,0,0,0.08);
    z-index: 20;
    overflow: hidden;
  }

  .dd-item {
    padding: 8px 14px;
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    transition: background 0.1s;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 8px;
  }
  .dd-item:hover { background: #f9f9f9; }
  .dd-item + .dd-item { border-top: 1px solid #f3f3f3; }
  .dd-rarity {
    font-size: 11px;
    font-weight: 600;
    flex-shrink: 0;
  }

  .chips { display: flex; flex-wrap: wrap; gap: 6px; }

  .chip {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 4px 10px 4px 7px;
    border: 1.5px solid;
    border-radius: 999px;
    font-size: 12px;
    font-weight: 600;
  }

  .chip-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .chip-remove {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
    line-height: 1;
    padding: 0 0 0 2px;
    color: inherit;
    opacity: 0.5;
    font-family: var(--font);
    transition: opacity 0.1s;
  }
  .chip-remove:hover { opacity: 1; }

  /* No-births message */
  .no-births {
    padding: 24px;
    background: #f9f9f9;
    border-radius: 12px;
    font-size: 14px;
    color: #555;
    line-height: 1.7;
    margin-top: 8px;
  }
</style>
