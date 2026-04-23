<script>
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  export let data;
  export let ages;

  let query       = '';
  let suggestions = [];

  function stripAccents(s) {
    return s.normalize('NFD').replace(/[̀-ͯ]/g, '');
  }

  function toTitle(s) {
    return s.charAt(0) + s.slice(1).toLowerCase();
  }

  function getPool() {
    const set = new Set();
    if (data) {
      Object.keys(data.H ?? {}).forEach(n => set.add(n));
      Object.keys(data.M ?? {}).forEach(n => set.add(n));
    }
    if (ages) {
      Object.keys(ages.H ?? {}).forEach(n => set.add(n));
      Object.keys(ages.M ?? {}).forEach(n => set.add(n));
    }
    return [...set];
  }

  function onInput() {
    if (!query.trim()) { suggestions = []; return; }
    const q        = stripAccents(query.trim()).toUpperCase();
    const pool     = getPool();
    const prefix   = pool.filter(n => stripAccents(n).startsWith(q));
    const contains = pool.filter(n => !stripAccents(n).startsWith(q) && stripAccents(n).includes(q));
    suggestions    = [...prefix, ...contains].slice(0, 7);
  }

  function autoSex(name) {
    const freqH = ages?.H?.[name]?.freq ?? (data?.H?.[name] ? 1 : 0);
    const freqM = ages?.M?.[name]?.freq ?? (data?.M?.[name] ? 1 : 0);
    if (freqM > freqH) return 'M';
    if (freqH > 0) return 'H';
    return 'M';
  }

  // Rareza: escala logarítmica sobre el ranking de frecuencia
  // log(rank)/log(total)*100 distribuye bien: top-50 ≈ 0-40, raros ≈ 80-100
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

  function rarityInfo(name) {
    const sx = autoSex(name);
    return freqRanks[sx]?.get(name) ?? null;
  }

  function rarityLabel(logPct) {
    if (logPct === null || logPct === undefined) return null;
    if (logPct >= 92) return 'muy raro';
    if (logPct >= 80) return 'raro';
    if (logPct >= 68) return 'poco común';
    if (logPct >= 55) return 'común';
    if (logPct >= 40) return 'popular';
    return 'muy popular';
  }

  function rarityColor(logPct) {
    if (logPct === null || logPct === undefined) return '#ccc';
    if (logPct >= 68) return '#ff922b';
    if (logPct >= 40) return '#aaa';
    return '#01f3b3';
  }

  function go(name) {
    const n = (name || query).trim().toUpperCase();
    if (!n) return;
    dispatch('discover', { name: n, sex: autoSex(n) });
  }

  function pick(n) {
    suggestions = [];
    go(n);
  }

  function onKeydown(e) {
    if (e.key === 'Enter') {
      if (suggestions.length) pick(suggestions[0]);
      else go('');
    }
    if (e.key === 'Escape') { query = ''; suggestions = []; }
  }
</script>

<div class="entry-page">
  <div class="entry-inner">

    <h1 class="main-title">¿Cuánta gente se llama como tú en España?</h1>

    <div class="options-grid">

      <!-- Card: Tendencias -->
      <div class="card card-left">
        <h2 class="card-title">Tu nombre</h2>
        <p class="card-desc">
          Descubre cuántas personas comparten tu nombre, su edad media y cómo ha evolucionado a lo largo de los años.
        </p>

        <div class="input-wrap">
          <div class="input-box">
            <input
              bind:value={query}
              on:input={onInput}
              on:keydown={onKeydown}
              on:blur={() => setTimeout(() => suggestions = [], 150)}
              placeholder="Escribe tu nombre…"
              autocomplete="off"
              spellcheck="false"
            />
          </div>
          {#if suggestions.length}
            <div class="dropdown">
              {#each suggestions as name}
                <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
                <div class="dd-item" on:mousedown|preventDefault={() => pick(name)}>
                  <span class="dd-name">{name}</span>
                  {#if rarityLabel(rarityInfo(name)?.logPct)}
                    <span class="dd-rarity" style="color:{rarityColor(rarityInfo(name)?.logPct)}">{rarityLabel(rarityInfo(name)?.logPct)}</span>
                  {/if}
                </div>
              {/each}
            </div>
          {/if}
        </div>

        <button class="cta" on:click={() => go('')}>Descubrir →</button>
      </div>

      <!-- Card: Mapa -->
      <!-- svelte-ignore a11y-click-events-have-key-events a11y-no-static-element-interactions -->
      <div class="card card-right" on:click={() => dispatch('openmap')}>
        <h2 class="card-title">Tu municipio</h2>
        <p class="card-desc">
          Explora qué nombres son más populares en cada municipio de España en un mapa interactivo.
        </p>
        <button class="cta-map" on:click|stopPropagation={() => dispatch('openmap')}>
          Ver el mapa →
        </button>
      </div>

    </div>
  </div>
</div>

<style>
  .entry-page {
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 32px 24px;
  }

  .entry-inner {
    width: 100%;
    max-width: 900px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0;
  }

  @keyframes fadeUp {
    from { opacity: 0; transform: translateY(16px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  @keyframes flyLeft {
    from { opacity: 0; transform: translateX(-28px); }
    to   { opacity: 1; transform: translateX(0); }
  }

  @keyframes flyRight {
    from { opacity: 0; transform: translateX(28px); }
    to   { opacity: 1; transform: translateX(0); }
  }

  .main-title {
    font-size: clamp(2rem, 4vw, 3rem);
    font-weight: 800;
    letter-spacing: -0.03em;
    color: #1a1a1a;
    text-align: center;
    width: 100%;
    margin-bottom: 48px;
    animation: fadeUp 0.5s cubic-bezier(0.22, 1, 0.36, 1) both;
  }

  /* Grid */
  .options-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 14px;
    width: 100%;
  }

  /* Cards base */
  .card {
    border-radius: 18px;
    padding: 36px 32px 32px;
    display: flex;
    flex-direction: column;
  }

  /* Card izquierda: clara */
  .card-left {
    border: 1.5px solid #e8e8e8;
    background: #fff;
    animation: flyLeft 0.55s 0.08s cubic-bezier(0.22, 1, 0.36, 1) both;
    position: relative;
    z-index: 1;
  }

  /* Card derecha */
  .card-right {
    background: #494949;
    cursor: pointer;
    transition: background 0.2s;
    animation: flyRight 0.55s 0.14s cubic-bezier(0.22, 1, 0.36, 1) both;
  }
  .card-right:hover { background: #555; }

  /* Animaciones internas — card izquierda */
  .card-left .card-tag   { animation: fadeUp 0.45s 0.22s cubic-bezier(0.22,1,0.36,1) both; }
  .card-left .card-title { animation: fadeUp 0.45s 0.30s cubic-bezier(0.22,1,0.36,1) both; }
  .card-left .card-desc  { animation: fadeUp 0.45s 0.38s cubic-bezier(0.22,1,0.36,1) both; }
  .card-left .input-wrap { animation: fadeUp 0.45s 0.46s cubic-bezier(0.22,1,0.36,1) both; }
  .card-left .cta        { animation: fadeUp 0.45s 0.54s cubic-bezier(0.22,1,0.36,1) both; }

  /* Animaciones internas — card derecha */
  .card-right .card-tag   { animation: fadeUp 0.45s 0.28s cubic-bezier(0.22,1,0.36,1) both; }
  .card-right .card-title { animation: fadeUp 0.45s 0.36s cubic-bezier(0.22,1,0.36,1) both; }
  .card-right .card-desc  { animation: fadeUp 0.45s 0.44s cubic-bezier(0.22,1,0.36,1) both; }
  .card-right .cta-map    { animation: fadeUp 0.45s 0.52s cubic-bezier(0.22,1,0.36,1) both; }

  /* Card tag */
  .card-tag {
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    margin-bottom: 14px;
  }
  .card-left  .card-tag { color: #bbb; }
  .card-right .card-tag { color: #aaa; }

  /* Card title */
  .card-title {
    font-size: clamp(2rem, 3.5vw, 2.8rem);
    font-weight: 800;
    letter-spacing: -0.04em;
    line-height: 1;
    margin-bottom: 14px;
  }
  .card-left  .card-title { color: #01f3b3; }
  .card-right .card-title { color: #fff; }

  /* Card desc */
  .card-desc {
    font-size: 14px;
    line-height: 1.65;
    margin-bottom: 28px;
    flex: 1;
  }
  .card-left  .card-desc { color: #777; }
  .card-right .card-desc { color: #ccc; }

  /* Input */
  .input-wrap {
    position: relative;
    z-index: 1;
    width: 100%;
    margin-bottom: 12px;
  }

  .input-box {
    width: 100%;
    height: 48px;
    border: 1.5px solid #e0e0e0;
    border-radius: 10px;
    padding: 0 16px;
    display: flex;
    align-items: center;
    transition: border-color 0.15s;
  }
  .input-box:focus-within { border-color: #01f3b3; }

  .input-box input {
    width: 100%;
    border: none;
    outline: none;
    font-size: 16px;
    font-family: var(--font);
    background: transparent;
    letter-spacing: -0.01em;
  }
  .input-box input::placeholder { color: #ccc; }

  .dropdown {
    position: absolute;
    top: calc(100% + 4px);
    left: 0; right: 0;
    background: #fff;
    border: 1px solid #e5e5e5;
    border-radius: 10px;
    box-shadow: 0 4px 24px rgba(0,0,0,0.08);
    z-index: 20;
    overflow: hidden;
  }

  .dd-item {
    padding: 10px 16px;
    font-size: 14px;
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
  .dd-name { flex: 1; }
  .dd-rarity {
    font-size: 11px;
    font-weight: 600;
    letter-spacing: 0.02em;
    flex-shrink: 0;
  }

  /* CTAs */
  .cta {
    height: 44px;
    padding: 0 24px;
    background: #01f3b3;
    color: #1a1a1a;
    border: none;
    border-radius: 9px;
    font-size: 14px;
    font-weight: 700;
    font-family: var(--font);
    cursor: pointer;
    letter-spacing: -0.01em;
    align-self: flex-start;
    transition: background 0.15s, transform 0.1s;
  }
  .cta:hover  { background: #00d9a0; }
  .cta:active { transform: scale(0.97); }

  .cta-map {
    height: 44px;
    padding: 0 24px;
    background: transparent;
    color: #01f3b3;
    border: 1.5px solid #01f3b3;
    border-radius: 9px;
    font-size: 14px;
    font-weight: 700;
    font-family: var(--font);
    cursor: pointer;
    letter-spacing: -0.01em;
    align-self: flex-start;
    transition: background 0.15s, color 0.15s, transform 0.1s;
  }
  .cta-map:hover  { background: #01f3b3; color: #1a1a1a; }
  .cta-map:active { transform: scale(0.97); }

  /* Mobile */
  @media (max-width: 600px) {
    .options-grid { grid-template-columns: 1fr; }
    .card { padding: 28px 24px 24px; }
  }
</style>
