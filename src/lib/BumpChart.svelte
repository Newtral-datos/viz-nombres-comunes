<script>
  export let data;
  export let selected;  // [{name, color}]
  export let sex;

  const PAD_TOP    = 24;
  const PAD_BOTTOM = 36;
  const RANK_H     = 20;

  let W = 800;
  let container;

  $: isMobile  = W < 500;
  $: PAD_LEFT  = isMobile ? 28 : 52;
  $: PAD_RIGHT = isMobile ? 72 : 130;

  $: years = data?.years ?? [];

  // Rango dinámico según los nombres seleccionados
  $: rankMin = (() => {
    let min = Infinity;
    for (const { name } of selected) {
      const s = data?.[sex]?.[name];
      if (!s) continue;
      for (const r of s.ranks) { if (r != null && r < min) min = r; }
    }
    return min === Infinity ? 1 : Math.max(1, min - 2);
  })();

  $: rankMax = (() => {
    let max = -Infinity;
    for (const { name } of selected) {
      const s = data?.[sex]?.[name];
      if (!s) continue;
      for (const r of s.ranks) { if (r != null && r > max) max = r; }
    }
    return max === -Infinity ? 20 : Math.min(100, max + 3);
  })();

  $: chartH  = PAD_TOP + (rankMax - rankMin + 1) * RANK_H + PAD_BOTTOM;
  $: chartW  = Math.max(W - PAD_LEFT - PAD_RIGHT, 200);
  $: xScale  = (i) => (i / (years.length - 1)) * chartW;
  $: yScale  = (r) => PAD_TOP + (r - rankMin) * RANK_H + RANK_H / 2;

  // tooltip
  let tooltip = null;

  function getPoints(name) {
    const series = data?.[sex]?.[name];
    if (!series) return [];
    return years.map((y, i) => {
      const r = series.ranks[i];
      const c = series.counts[i];
      return { x: PAD_LEFT + xScale(i), y: r != null ? yScale(r) : null, rank: r, count: c, year: y };
    });
  }

  function buildPath(points) {
    const segments = [];
    let seg = [];
    for (const pt of points) {
      if (pt.rank == null || pt.rank < rankMin || pt.rank > rankMax) {
        if (seg.length > 1) segments.push(seg);
        seg = [];
      } else {
        seg.push(pt);
      }
    }
    if (seg.length > 1) segments.push(seg);

    return segments.map(pts => {
      let d = `M ${pts[0].x} ${pts[0].y}`;
      for (let i = 1; i < pts.length; i++) {
        const cx = (pts[i-1].x + pts[i].x) / 2;
        d += ` C ${cx},${pts[i-1].y} ${cx},${pts[i].y} ${pts[i].x},${pts[i].y}`;
      }
      return d;
    }).join(' ');
  }

  function onMouseMove(e) {
    if (!container) return;
    const rect = container.getBoundingClientRect();
    const mx   = e.clientX - rect.left - PAD_LEFT;
    const yi   = Math.round((mx / chartW) * (years.length - 1));
    if (yi < 0 || yi >= years.length) { tooltip = null; return; }
    const year  = years[yi];
    const items = selected.map(({ name, color }) => {
      const s = data?.[sex]?.[name];
      const rank  = s?.ranks[yi] ?? null;
      const count = s?.counts[yi] ?? null;
      return { name, color, rank, count };
    }).filter(d => d.rank != null).sort((a, b) => a.rank - b.rank);
    tooltip = { year, items, x: e.clientX - rect.left, y: e.clientY - rect.top };
  }

  function toTitle(s) {
    return s.charAt(0) + s.slice(1).toLowerCase();
  }
</script>

<div class="chart-wrap" bind:clientWidth={W} bind:this={container}
     on:mousemove={onMouseMove} on:mouseleave={() => tooltip = null} role="img">

  <svg width={W} height={chartH}>
    <!-- Líneas de rango (grid) -->
    {#each Array.from({length: rankMax - rankMin + 1}, (_, i) => rankMin + i) as rank}
      <line
        x1={PAD_LEFT} y1={yScale(rank)}
        x2={PAD_LEFT + chartW} y2={yScale(rank)}
        stroke={rank === 1 ? '#ddd' : '#f0f0f0'}
        stroke-width="1"
      />
      <text x={PAD_LEFT - 5} y={yScale(rank)} text-anchor="end" dominant-baseline="middle"
            font-size={isMobile ? 9 : 11} fill="#bbb">{rank}</text>
    {/each}

    <!-- Etiquetas de año -->
    {#each years as year, i}
      {#if isMobile ? year % 4 === 0 : year % 2 === 0}
        <text
          x={PAD_LEFT + xScale(i)} y={chartH - 8}
          text-anchor="middle" font-size={isMobile ? 9 : 11} fill="#aaa"
        >{year}</text>
      {/if}
    {/each}

    <!-- Líneas de nombres -->
    {#each selected as { name, color }}
      {@const points = getPoints(name)}
      {@const path   = buildPath(points)}
      {#if path}
        <path d={path} fill="none" stroke={color} stroke-width={isMobile ? 2 : 2.5}
              stroke-linecap="round" stroke-linejoin="round" />
      {/if}

      <!-- Puntos -->
      {#each points as pt}
        {#if pt.rank != null && pt.rank >= rankMin && pt.rank <= rankMax}
          <circle cx={pt.x} cy={pt.y} r={isMobile ? 2.5 : 3.5} fill={color} />
        {/if}
      {/each}

      <!-- Etiqueta al final (último año con datos) -->
      {#each [points.filter(p => p.rank != null && p.rank >= rankMin && p.rank <= rankMax).at(-1)].filter(Boolean) as last}
        <text x={last.x + (isMobile ? 6 : 10)} y={last.y} dominant-baseline="middle"
              font-size={isMobile ? 10 : 12} font-weight="600" fill={color}>{toTitle(name)}</text>
        <text x={last.x + (isMobile ? 6 : 10)} y={last.y + (isMobile ? 11 : 13)} dominant-baseline="middle"
              font-size={isMobile ? 9 : 10} fill={color} opacity="0.7">nº{last.rank}</text>
      {/each}
    {/each}

    <!-- Línea vertical de tooltip -->
    {#if tooltip}
      <line
        x1={PAD_LEFT + xScale(years.indexOf(tooltip.year))}
        y1={PAD_TOP}
        x2={PAD_LEFT + xScale(years.indexOf(tooltip.year))}
        y2={chartH - PAD_BOTTOM}
        stroke="#ccc" stroke-width="1" stroke-dasharray="4 3"
      />
    {/if}
  </svg>

  <!-- Tooltip -->
  {#if tooltip && tooltip.items.length}
    <div class="tooltip" style="left:{tooltip.x + 16}px; top:{Math.min(tooltip.y, chartH - 120)}px">
      <div class="tt-year">{tooltip.year}</div>
      {#each tooltip.items as d}
        <div class="tt-row">
          <span class="tt-dot" style="background:{d.color}"></span>
          <span class="tt-name">{toTitle(d.name)}</span>
          <span class="tt-rank">nº{d.rank}</span>
          <span class="tt-count">{d.count.toLocaleString('es')}</span>
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  .chart-wrap {
    position: relative;
    width: 100%;
    flex: 1;
    min-height: 0;
    overflow: hidden;
    cursor: crosshair;
  }
  svg { display: block; width: 100%; }

  .tooltip {
    position: absolute;
    background: rgba(255,255,255,0.97);
    border: 1px solid #e5e5e5;
    border-radius: 8px;
    padding: 8px 12px;
    pointer-events: none;
    box-shadow: 0 4px 16px rgba(0,0,0,0.08);
    font-size: 12px;
    min-width: 160px;
    z-index: 10;
  }
  .tt-year {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: #aaa;
    margin-bottom: 6px;
  }
  .tt-row {
    display: grid;
    grid-template-columns: 10px 1fr auto auto;
    align-items: center;
    gap: 6px;
    padding: 2px 0;
  }
  .tt-dot {
    width: 8px; height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .tt-name { font-weight: 600; }
  .tt-rank { color: #888; font-variant-numeric: tabular-nums; }
  .tt-count { color: #aaa; font-variant-numeric: tabular-nums; font-size: 11px; }
</style>
