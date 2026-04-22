<script>
  import { onMount } from 'svelte';
  import { fade, fly } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';
  import NameEntry from './lib/NameEntry.svelte';
  import NameStory from './lib/NameStory.svelte';

  const MAP_URL = 'https://newtral-datos.github.io/mapa-nombres-comunes/';

  let data        = null;
  let ages        = null;
  let screen      = 'entry';  // 'entry' | 'story' | 'map'
  let chosenName  = '';
  let chosenSex   = 'H';

  onMount(async () => {
    [data, ages] = await Promise.all([
      fetch('/data.json').then(r => r.json()),
      fetch('/ages.json').then(r => r.json()),
    ]);
  });

  function handleDiscover(e) {
    chosenName = e.detail.name;
    chosenSex  = e.detail.sex;
    screen     = 'story';
  }

  function handleBack() { screen = 'entry'; }
  function handleOpenMap() { screen = 'map'; }
</script>

<div class="app-wrap">
  {#if !data}
    <div class="loading">Cargando…</div>

  {:else if screen === 'map'}
    <div class="map-screen" in:fly={{ y: 28, duration: 340, easing: cubicOut, delay: 80 }} out:fade={{ duration: 160 }}>
      <header class="map-header">
        <button class="back-btn" on:click={handleBack}>← Tendencias</button>
        <div class="mode-tabs">
          <button class="tab" on:click={handleBack}>Tendencias</button>
          <button class="tab active">Mapa de municipios</button>
        </div>
      </header>
      <iframe src={MAP_URL} title="Mapa de nombres por municipio" allowfullscreen></iframe>
    </div>

  {:else}
    {#key screen}
      <div class="screen"
        in:fly={{ y: 28, duration: 340, easing: cubicOut, delay: 80 }}
        out:fade={{ duration: 160 }}>
        {#if screen === 'entry'}
          <NameEntry {data} {ages} on:discover={handleDiscover} on:openmap={handleOpenMap} />
        {:else}
          <NameStory {data} {ages} name={chosenName} sex={chosenSex} on:back={handleBack} />
        {/if}
      </div>
    {/key}
  {/if}
</div>

<style>
  .app-wrap {
    position: relative;
    height: 100dvh;
    overflow: hidden;
  }

  .screen {
    position: absolute;
    inset: 0;
    overflow: hidden;
  }

  /* Mapa */
  .map-screen {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
  }

  .map-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 10px 20px;
    border-bottom: 1px solid var(--border);
    flex-shrink: 0;
    background: #fff;
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

  .mode-tabs { display: flex; gap: 6px; }

  .tab {
    padding: 6px 16px;
    border-radius: 999px;
    border: 1.5px solid var(--border);
    background: none;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font);
    cursor: pointer;
    color: var(--muted);
    transition: all 0.15s;
  }
  .tab:hover { border-color: #aaa; color: var(--text); }
  .tab.active {
    background: #1a1a1a;
    border-color: #1a1a1a;
    color: #fff;
  }

  .map-screen iframe {
    flex: 1;
    border: none;
    width: 100%;
  }

  .loading {
    height: 100dvh;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #aaa;
    font-size: 14px;
  }
</style>
